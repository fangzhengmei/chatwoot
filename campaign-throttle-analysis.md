# Chatwoot 主动触达活动（Campaign）跨渠道调度与限流机制分析

## 1. 概述

Chatwoot 的主动触达活动（Campaign）系统支持通过多个渠道向目标受众发送消息。本报告详细分析其跨渠道调度架构和限流机制，重点补充了 Website 渠道的频率控制、各渠道超配额行为，以及完整的渠道对照表。

## 2. 整体架构

### 2.1 Campaign 模型核心

**文件**: `app/models/campaign.rb`

Campaign 是主动触达的核心模型，包含以下关键属性：

```ruby
enum campaign_type: { ongoing: 0, one_off: 1 }
enum campaign_status: { active: 0, completed: 1 }
```

**核心调度逻辑** (`campaign.rb:66-75`):

```ruby
def execute_campaign
  case inbox.inbox_type
  when 'Twilio SMS'
    Twilio::OneoffSmsCampaignService.new(campaign: self).perform
  when 'Sms'
    Sms::OneoffSmsCampaignService.new(campaign: self).perform
  when 'Whatsapp'
    Whatsapp::OneoffCampaignService.new(campaign: self).perform if account.feature_enabled?(:whatsapp_campaign)
  end
end
```

### 2.2 触发机制

**定时触发**: `app/jobs/trigger_scheduled_items_job.rb`

- **队列**: `scheduled_jobs`
- **触发条件**: 查找 `scheduled_at` 在 3 天前到当前时间之间的活动
- **批量处理**: `find_each(batch_size: 100)`

**单次活动触发**: `app/jobs/campaigns/trigger_oneoff_campaign_job.rb`

- **队列**: `low`
- **执行**: 调用 `campaign.trigger!`

## 3. 支持的渠道

Campaign 支持以下渠道 (`campaign.rb:84`):

| 渠道类型 | 渠道名称 | 活动类型 | 触发方式 |
|---------|---------|---------|---------|
| Website | Widget | ongoing | 前端事件触发 |
| Twilio SMS | Twilio SMS | one_off | 后台 Job 批量发送 |
| Sms | Bandwidth SMS | one_off | 后台 Job 批量发送 |
| Whatsapp | WhatsApp | one_off | 后台 Job 批量发送 |

### 3.1 渠道自动分类

**文件**: `app/models/campaign.rb:88-97`

Campaign 会根据 `inbox.inbox_type` 自动设置活动类型：

- **Twilio SMS / Sms / Whatsapp**: 自动设为 `one_off`，`scheduled_at` 默认为当前时间
- **Website**: 自动设为 `ongoing`，无 `scheduled_at`

## 4. 各渠道调度实现

### 4.1 WhatsApp 渠道

**文件**: `app/services/whatsapp/oneoff_campaign_service.rb`

**执行流程**:

1. **验证阶段** (`validate_campaign!`):
   - 验证活动类型必须是 `one_off`
   - 验证活动状态不是 `completed`
   - 验证 provider 必须是 `whatsapp_cloud`
   - 验证账户启用了 `whatsapp_campaign` 功能

2. **状态更新**: 立即将 campaign 标记为 `completed!`

3. **受众提取**:
   ```ruby
   audience_label_ids = campaign.audience.select { |audience| audience['type'] == 'Label' }.pluck('id')
   ```

4. **消息发送**:
   - 使用 `campaign.account.contacts.tagged_with(audience_labels, any: true)` 获取目标联系人
   - **同步遍历**: `contacts.each { |contact| process_contact(contact) }`
   - 必须使用模板消息 (`template_params` 必填)
   - 支持 Liquid 模板变量 (`{{contact.name}}`, `{{contact.email}}` 等)

**发送方法**: `channel.send_template()`

**Provider 服务**: `app/services/whatsapp/providers/whatsapp_cloud_service.rb`

- 直接调用 Meta WhatsApp Cloud API
- 无内部限流逻辑
- 错误捕获后继续处理后续联系人

### 4.2 Twilio SMS 渠道

**文件**: `app/services/twilio/oneoff_sms_campaign_service.rb`

**执行流程**:

1. **验证**: 检查渠道类型和活动状态
2. **状态更新**: 立即标记为 `completed!`
3. **受众筛选**: 同样基于标签
4. **消息发送**:
   - 支持 Liquid 模板渲染消息内容
   - 调用 `channel.send_message(to: contact.phone_number, body: content)`
   - 捕获 `Twilio::REST::TwilioError` / `Twilio::REST::RestError` 后继续

**Channel 实现**: `app/models/channel/twilio_sms.rb:57-62`

```ruby
def send_message(to:, body:, media_url: nil)
  params = send_message_from.merge(to: to, body: body)
  params[:media_url] = media_url if media_url.present?
  params[:status_callback] = twilio_delivery_status_index_url
  client.messages.create(**params)
end
```

### 4.3 Bandwidth SMS 渠道

**文件**: `app/services/sms/oneoff_sms_campaign_service.rb`

**执行流程**: 与 Twilio 类似

**Channel 实现**: `app/models/channel/sms.rb:36-73`

- 使用 Bandwidth API (`https://messaging.bandwidth.com/api/v2`)
- 无内部限流
- 错误处理后记录日志并继续

### 4.4 Website Widget 渠道

**类型**: `ongoing`

**触发**: 通过 `CampaignListener` 监听前端事件触发

**文件**: `app/listeners/campaign_listener.rb`

使用 `Campaigns::CampaignConversationBuilder` 创建对话

---

## 5. Website 渠道：事件触发链路与频率控制

### 5.1 完整触发链路

```
前端 Widget
    ↓
JavaScript: executeCampaign()
    ↓
POST /api/v1/widget/events
    ↓
Api::V1::Widget::EventsController#create
    ↓
Rails.configuration.dispatcher.dispatch('campaign.triggered', ...)
    ↓
Dispatcher → SyncDispatcher + AsyncDispatcher
    ↓
CampaignListener#campaign_triggered
    ↓
Campaigns::CampaignConversationBuilder#perform
    ↓
创建 Conversation + Message
```

### 5.2 前端频率控制

**文件**: `app/javascript/widget/helpers/campaignTimer.js`

```javascript
class CampaignTimer {
  initTimers = ({ campaigns }, websiteToken) => {
    this.clearTimers();
    campaigns.forEach(campaign => {
      const { timeOnPage, id: campaignId } = campaign;
      this.campaignTimers[campaignId] = setTimeout(() => {
        store.dispatch('campaign/startCampaign', { campaignId, websiteToken });
      }, timeOnPage * 1000);  // 基于 time_on_page 的延迟触发
    });
  };
}
```

**控制机制**:

| 控制维度 | 实现位置 | 代码证据 | 控制逻辑 |
|---------|---------|---------|---------|
| **延迟触发** | `campaignTimer.js:10-14` | `setTimeout(() => {...}, timeOnPage * 1000)` | 根据 `trigger_rules.time_on_page` 配置，用户在页面停留 N 秒后才触发 |
| **URL 匹配** | `campaignHelper.js:33-46` | `filterCampaigns()` 中检查 `isPatternMatchingWithURL` | 只有匹配 `trigger_rules.url` 的页面才会触发 |
| **营业时间** | `campaignHelper.js:42-44` | `campaign.triggerOnlyDuringBusinessHours` | 仅在配置的营业时间内触发 |
| **Widget 打开检测** | `campaign.js:114-116` | `if (!isWidgetOpen)` | Widget 已打开时不执行 campaign |
| **缓存控制** | `campaign.js:48-63` | `CACHE_EXPIRY = 60 * 60 * 1000` | Campaign 列表缓存 1 小时，避免重复请求 |
| **单次获取** | `campaign.js:84-95` | `if (!uiFlags.hasFetched)` | SPA 页面切换时不重复获取 campaign |

### 5.3 后端幂等约束

**文件**: `app/builders/campaigns/campaign_conversation_builder.rb`

```ruby
def perform
  @contact_inbox = ContactInbox.find(@contact_inbox_id)
  @campaign = @contact_inbox.inbox.campaigns.find_by!(display_id: campaign_display_id)

  ActiveRecord::Base.transaction do
    @contact_inbox.lock!  # 行级锁

    # 幂等检查：如果 contact_inbox 已有对话，不再创建
    raise 'Conversation already present' if @contact_inbox.reload.conversations.present?

    @conversation = ::Conversation.create!(conversation_params)
    Messages::MessageBuilder.new(@campaign.sender, @conversation, message_params).perform
  end
  @conversation
rescue StandardError => e
  Rails.logger.info(e.message)
  nil  # 失败返回 nil，不抛异常
end
```

**幂等机制分析**:

| 机制 | 代码位置 | 代码证据 | 作用 |
|-----|---------|---------|------|
| **数据库行级锁** | `campaign_conversation_builder.rb:9` | `@contact_inbox.lock!` | 防止并发请求同时创建 |
| **事务包裹** | `campaign_conversation_builder.rb:8` | `ActiveRecord::Base.transaction do` | 原子性操作 |
| **重复检查** | `campaign_conversation_builder.rb:12` | `if @contact_inbox.reload.conversations.present?` | 同一 contact_inbox 只创建一次对话 |
| **异常捕获** | `campaign_conversation_builder.rb:18-20` | `rescue StandardError => e; nil` | 重复创建时静默返回，不抛错 |

**测试验证** (`spec/builders/campaigns/campaign_conversation_builder_spec.rb:22-30`):

```ruby
it 'will not create a conversation with campaign id if another conversation exists' do
  create(:conversation, contact_inbox_id: contact_inbox.id, inbox: inbox, account: account)
  campaign_conversation = described_class.new(
    contact_inbox_id: contact_inbox.id,
    campaign_display_id: campaign.display_id
  ).perform

  expect(campaign_conversation).to be_nil  # 返回 nil，不创建
end
```

### 5.4 Website 渠道风险分析

| 风险点 | 现状 | 风险等级 |
|-------|------|---------|
| **前端频率控制** | 有 `time_on_page` 延迟 + URL 匹配 + 营业时间 | 中等 |
| **后端幂等** | 有行级锁 + 对话存在检查 | 低（已实现） |
| **高频恶意触发** | 无后端频率限制（Rack::Attack 可能有限制） | 高 |
| **同一用户重复触发** | 依赖 contact_inbox.conversations.present? 检查 | 低（已实现） |

**关键发现**: Website 渠道的幂等约束在后端通过数据库锁和存在检查实现，但缺少应用层的频率限制（如每 N 分钟最多触发一次）。

---

## 6. 超配额后系统行为分析

### 6.1 消息状态模型

**文件**: `app/models/message.rb:103`

```ruby
enum status: { sent: 0, delivered: 1, read: 2, failed: 3 }
```

Message 支持 `failed` 状态和 `external_error` 字段，但 **campaign 发送流程中大部分场景不会使用这些字段**。

### 6.2 各渠道超配额行为对照表

#### 6.2.1 WhatsApp 渠道

**正常对话消息的错误处理** (`app/services/whatsapp/providers/base_service.rb:34-55`):

```ruby
def process_response(response, message)
  parsed_response = response.parsed_response
  if response.success? && parsed_response['error'].blank?
    parsed_response['messages'].first['id']
  else
    handle_error(response, message)
    nil
  end
end

def handle_error(response, message)
  Rails.logger.error response.body
  return if message.blank?

  error_message = error_message(response)
  return if error_message.blank?

  message.external_error = error_message
  message.status = :failed
  message.save!
end
```

**但是！Campaign 中的调用** (`app/services/whatsapp/oneoff_campaign_service.rb:88-110`):

```ruby
def send_whatsapp_template_message(to:, template_params:)
  # ...
  channel.send_template(to, {
    name: name,
    namespace: namespace,
    lang_code: lang_code,
    parameters: processed_parameters
  }, nil)  # 第三个参数 message 传入 nil！

rescue StandardError => e
  Rails.logger.error "Failed to send WhatsApp template message to #{to}: #{e.message}"
  Rails.logger.error "Backtrace: #{e.backtrace.first(5).join('\n')}"
  nil  # 继续处理后续联系人
end
```

**代码证据**: `oneoff_campaign_service.rb:98` 中 `send_template` 调用的第三个参数是 `nil`

**结论**:
- Campaign 发送的消息 **不会创建 Message 记录**，因此 `status = :failed` 和 `external_error` 不会被设置
- 错误仅记录到 Rails 日志
- 发生错误后 **继续发送** 下一个联系人

#### 6.2.2 Twilio SMS 渠道

**文件**: `app/services/twilio/oneoff_sms_campaign_service.rb:21-34`

```ruby
def process_audience(audience_labels)
  campaign.account.contacts.tagged_with(audience_labels, any: true).each do |contact|
    next if contact.phone_number.blank?

    content = Liquid::CampaignTemplateService.new(campaign: campaign, contact: contact).call(campaign.message)

    begin
      channel.send_message(to: contact.phone_number, body: content)
    rescue Twilio::REST::TwilioError, Twilio::REST::RestError => e
      Rails.logger.error("[Twilio Campaign #{campaign.id}] Failed to send to #{contact.phone_number}: #{e.message}")
      next  # 继续下一个
    end
  end
end
```

**代码证据**:
- 第 27-31 行：`begin...rescue` 捕获 Twilio 错误
- 第 30 行：仅记录日志
- 第 31 行：`next` 继续处理后续联系人

**结论**:
- **不创建 Message 记录**
- 错误仅记录到 Rails 日志
- 发生错误后 **继续发送** 下一个联系人
- **无重试机制**

#### 6.2.3 Bandwidth SMS 渠道

**文件**: `app/services/sms/oneoff_sms_campaign_service.rb:21-34`

```ruby
def process_audience(audience_labels)
  campaign.account.contacts.tagged_with(audience_labels, any: true).each do |contact|
    next if contact.phone_number.blank?

    content = Liquid::CampaignTemplateService.new(campaign: campaign, contact: contact).call(campaign.message)
    send_message(to: contact.phone_number, content: content)
  end
end

def send_message(to:, content:)
  channel.send_text_message(to, content)
rescue StandardError => e
  Rails.logger.error("[SMS Campaign #{campaign.id}] Failed to send to #{to}: #{e.message}")
end
```

**代码证据**:
- 第 30-34 行：`rescue StandardError` 捕获所有错误
- 第 32 行：仅记录日志
- 无 `retry` 或 `next`（但在 `each` 循环中，异常被捕获后自然继续）

**结论**:
- **不创建 Message 记录**
- 错误仅记录到 Rails 日志
- 发生错误后 **继续发送** 下一个联系人
- **无重试机制**

#### 6.2.4 行为汇总

| 行为维度 | WhatsApp | Twilio SMS | Bandwidth SMS | Website |
|---------|---------|-----------|--------------|---------|
| **创建 Message 记录** | ❌ 否 | ❌ 否 | ❌ 否 | ✅ 是 |
| **设置 failed 状态** | ❌ 否 | ❌ 否 | ❌ 否 | N/A |
| **记录 external_error** | ❌ 否 | ❌ 否 | ❌ 否 | N/A |
| **记录错误日志** | ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 |
| **超配额后继续发送** | ✅ 是 | ✅ 是 | ✅ 是 | N/A |
| **自动重试** | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 |
| **可手动重试** | ❌ 无接口 | ❌ 无接口 | ❌ 无接口 | N/A |

### 6.3 超配额风险分析

| 风险场景 | 影响 |
|---------|------|
| **API 限流/配额耗尽** | 后续消息全部失败，但 campaign 状态仍为 `completed` |
| **账户封禁** | 所有消息失败，无自动恢复机制 |
| **部分成功部分失败** | 无法追踪哪些联系人收到了消息 |
| **无法重试** | 失败的消息无法单独重试，只能重新发送整个 campaign |

---

## 7. 渠道对照表（代码证据-结论-风险）

### 7.1 调度与触发

| 维度 | Website | WhatsApp | Twilio SMS | Bandwidth SMS |
|-----|---------|---------|-----------|--------------|
| **活动类型** | ongoing | one_off | one_off | one_off |
| **代码证据** | `campaign.rb:95` `self.campaign_type = 'ongoing'` | `campaign.rb:92` `self.campaign_type = 'one_off'` | `campaign.rb:92` | `campaign.rb:92` |
| **触发方式** | 前端事件 → Dispatcher → Listener → Builder | Job → Service → 逐个调用 API | Job → Service → 逐个调用 API | Job → Service → 逐个调用 API |
| **代码证据** | `events_controller.rb:5` `dispatcher.dispatch(...)` | `trigger_oneoff_campaign_job.rb:5` `campaign.trigger!` | 同 WhatsApp | 同 WhatsApp |
| **触发入口** | `POST /api/v1/widget/events` | `Campaigns::TriggerOneoffCampaignJob` | 同 WhatsApp | 同 WhatsApp |
| **结论** | 事件驱动，实时触发 | 后台批量处理 | 后台批量处理 | 后台批量处理 |
| **风险** | 可被高频恶意调用 | 无 Job 拆分，可能超时 | 无 Job 拆分，可能超时 | 无 Job 拆分，可能超时 |

### 7.2 频率控制与幂等

| 维度 | Website | WhatsApp | Twilio SMS | Bandwidth SMS |
|-----|---------|---------|-----------|--------------|
| **前端频率控制** | ✅ time_on_page 延迟 + URL 匹配 + 营业时间 | N/A | N/A | N/A |
| **代码证据** | `campaignTimer.js:11` `setTimeout(..., timeOnPage * 1000)` | - | - | - |
| **后端频率控制** | ❌ 无（仅幂等） | ❌ 无 | ❌ 无 | ❌ 无 |
| **幂等机制** | ✅ 行级锁 + 对话存在检查 | ❌ 无（逐个发送，跳过错误） | ❌ 无（逐个发送，跳过错误） | ❌ 无（逐个发送，跳过错误） |
| **代码证据** | `campaign_conversation_builder.rb:9` `@contact_inbox.lock!` | `oneoff_campaign_service.rb:105-109` `rescue... nil` | `twilio/oneoff_sms_campaign_service.rb:29-31` `rescue... next` | `sms/oneoff_sms_campaign_service.rb:30-33` `rescue...` |
| **结论** | 后端有幂等，前端有频率控制 | 无任何限流/幂等保护 | 无任何限流/幂等保护 | 无任何限流/幂等保护 |
| **风险** | 可被恶意高频触发 | 超配额后继续浪费请求 | 超配额后继续浪费请求 | 超配额后继续浪费请求 |

### 7.3 超配额行为

| 维度 | Website | WhatsApp | Twilio SMS | Bandwidth SMS |
|-----|---------|---------|-----------|--------------|
| **创建 Message 记录** | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 |
| **代码证据** | `campaign_conversation_builder.rb:14-15` `Conversation.create! + MessageBuilder` | `oneoff_campaign_service.rb:98` `send_template(..., nil)` | `twilio/oneoff_sms_campaign_service.rb:28` `channel.send_message(...)` 无返回处理 | 同 Twilio |
| **失败状态追踪** | ✅ 有（通过 Message 状态） | ❌ 无 | ❌ 无 | ❌ 无 |
| **错误记录位置** | Message.external_error | Rails.logger | Rails.logger | Rails.logger |
| **代码证据** | `message.rb:103` `enum status: { ... failed: 3 }` | `oneoff_campaign_service.rb:106` `Rails.logger.error` | `twilio/oneoff_sms_campaign_service.rb:30` `Rails.logger.error` | `sms/oneoff_sms_campaign_service.rb:32` `Rails.logger.error` |
| **超配额后行为** | 抛出异常，返回 nil | 记录日志，继续发送 | 记录日志，继续发送 | 记录日志，继续发送 |
| **代码证据** | `campaign_conversation_builder.rb:18-20` `rescue... nil` | `oneoff_campaign_service.rb:71` `contacts.each { ... }` + `rescue... nil` | `twilio/oneoff_sms_campaign_service.rb:22-33` `.each do ... rescue ... next ... end` | `sms/oneoff_sms_campaign_service.rb:22-27` `.each do ... end` |
| **自动重试** | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 |
| **结论** | 有状态追踪，无重试 | 无状态追踪，继续发送 | 无状态追踪，继续发送 | 无状态追踪，继续发送 |
| **风险** | 失败可追踪但无法重试 | 失败不可追踪，静默失败 | 失败不可追踪，静默失败 | 失败不可追踪，静默失败 |

### 7.4 外部平台限制

| 维度 | Website | WhatsApp | Twilio SMS | Bandwidth SMS |
|-----|---------|---------|-----------|--------------|
| **平台限制类型** | 无（自托管） | messaging_limit_tier + throughput | 平台侧限流 | 平台侧限流 |
| **代码中是否查询** | N/A | ✅ 是（仅查询显示） | ❌ 否 | ❌ 否 |
| **代码证据** | - | `health_service.rb:44` `messaging_limit_tier` | - | - |
| **是否用于限流决策** | N/A | ❌ 否（仅显示，不使用） | ❌ 否 | ❌ 否 |
| **代码证据** | - | `oneoff_campaign_service.rb` 无相关检查 | - | - |
| **结论** | 无外部限制 | 有查询但不使用 | 完全依赖平台 | 完全依赖平台 |
| **风险** | 无 | 超量可能被平台降质/封禁 | 超量可能被平台限流 | 超量可能被平台限流 |

---

## 8. 限流机制现状总结

### 8.1 现有限流组件（均不用于 Campaign）

#### 8.1.1 AutoAssignment::RateLimiter

**文件**: `app/services/auto_assignment/rate_limiter.rb`

**用途**: 自动分配的限流，**不用于 campaign**

```ruby
def within_limit?
  current_count < limit
end

def limit
  config&.fair_distribution_limit.present? ? config.fair_distribution_limit.to_i : 5
end

def window
  config&.fair_distribution_window&.to_i || 5.minutes.to_i
end
```

#### 8.1.2 AccountEmailRateLimitable

**文件**: `app/models/concerns/account_email_rate_limitable.rb`

**用途**: 邮件通知限流，**不用于 campaign**

```ruby
OUTBOUND_EMAIL_TTL = 25.hours.to_i

def within_email_rate_limit?
  return true unless ChatwootApp.chatwoot_cloud?
  return true if emails_sent_today < email_rate_limit
  # ...
end
```

### 8.2 Campaign 限流现状

**重要发现**: **Chatwoot 内部对 Campaign 没有实现任何限流机制**

#### 8.2.1 发送方式

所有 `oneoff_campaign_service` 均采用 **同步顺序发送**:

```ruby
# Whatsapp::OneoffCampaignService
contacts.each { |contact| process_contact(contact) }

# Twilio::OneoffSmsCampaignService
campaign.account.contacts.tagged_with(...).each do |contact|
  channel.send_message(...)
end
```

- **无批量发送**: 逐个调用 API
- **无延迟间隔**: 无 `sleep` 或 `delay`
- **无队列拆分**: 所有联系人在同一个 Job 中处理
- **无限流检查**: 发送前不检查任何限制

---

## 9. 建议改进

### 9.1 应用层限流

1. **引入 Campaign 专用限流服务**:
   ```ruby
   class Campaign::RateLimiter
     def can_send?(campaign)
       # 检查渠道每日配额
       # 检查发送速率
     end
   end
   ```

2. **利用 WhatsApp `messaging_limit_tier`**:
   - 发送前检查当前等级
   - 超过限制后暂停发送

3. **账户级配额**:
   - 每日发送上限
   - 可配置的发送速率

4. **Website 渠道频率限制**:
   - 每 contact_inbox 每 N 分钟最多触发一次
   - 使用 Redis 实现滑动窗口限流

### 9.2 分批发送

1. **分批处理联系人**:
   ```ruby
   contacts.in_batches(of: 100).each_with_index do |batch, index|
     Campaigns::SendBatchJob.set(wait: index * 1.minute).perform_later(campaign, batch.ids)
   end
   ```

2. **引入延迟**:
   - 批量间添加延迟
   - 平滑发送速率

### 9.3 状态追踪与可观测性

1. **创建 CampaignMessage 记录**:
   - 追踪每个联系人的发送状态
   - 记录成功/失败/错误信息

2. **精细的活动状态**:
   - `processing`: 正在发送
   - `completed`: 全部完成
   - `failed`: 部分/全部失败
   - `paused`: 暂停

3. **发送进度追踪**:
   - 已发送/总数量
   - 成功/失败统计

4. **可重试机制**:
   - 支持单独重试失败的消息
   - 支持暂停/恢复 campaign

### 9.4 幂等与去重

1. **基于 contact_id + campaign_id 的唯一约束**:
   - 防止同一 campaign 向同一联系人重复发送
   - 数据库唯一索引

---

## 10. 核心结论汇总

| 维度 | Website | WhatsApp | Twilio SMS | Bandwidth SMS |
|-----|---------|---------|-----------|--------------|
| **调度架构** | 事件驱动（前端→API→Dispatcher→Listener→Builder） | Job 批量（TriggerOneoffCampaignJob → OneoffCampaignService） | 同 WhatsApp | 同 WhatsApp |
| **频率控制** | 前端有，后端仅幂等 | 无 | 无 | 无 |
| **幂等机制** | ✅ 行级锁 + 对话存在检查 | ❌ 无 | ❌ 无 | ❌ 无 |
| **超配额行为** | 创建 Message，可追踪失败 | 仅记录日志，继续发送 | 仅记录日志，继续发送 | 仅记录日志，继续发送 |
| **失败重试** | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 |
| **平台限制感知** | N/A | 有查询但不使用 | 完全依赖平台 | 完全依赖平台 |
| **主要风险** | 可被高频恶意触发 | 超量后平台可能降质/封禁，失败不可追踪 | 同 WhatsApp | 同 WhatsApp |

**核心结论**:
1. **Website 渠道**：前端有频率控制，后端有幂等保护，但缺少应用层限流
2. **WhatsApp/SMS 渠道**：无任何应用层限流，超配额后继续发送，失败不可追踪
3. **所有 one_off 渠道**：不创建 Message 记录，无法追踪单个联系人的发送状态
4. **改进优先级**：
   - P0: 为 one_off campaign 引入消息状态追踪
   - P1: 实现分批发送 + 延迟机制
   - P2: 利用平台限制信息进行主动限流
   - P3: 实现失败重试机制
