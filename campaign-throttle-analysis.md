# Chatwoot 主动触达活动（Campaign）跨渠道调度与限流机制分析

## 1. 概述

Chatwoot 的主动触达活动（Campaign）系统支持通过多个渠道向目标受众发送消息。本报告详细分析其跨渠道调度架构和限流机制。

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

| 渠道类型 | 渠道名称 | 活动类型 |
|---------|---------|---------|
| Website | Widget | ongoing |
| Twilio SMS | Twilio SMS | one_off |
| Sms | Bandwidth SMS | one_off |
| Whatsapp | WhatsApp | one_off |

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

- **类型**: `ongoing`
- **触发**: 通过 `CampaignListener` 监听前端事件触发
- **文件**: `app/listeners/campaign_listener.rb`
- 使用 `Campaigns::CampaignConversationBuilder` 创建对话

## 5. 限流机制分析

### 5.1 现有限流组件

#### 5.1.1 AutoAssignment::RateLimiter

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

#### 5.1.2 AccountEmailRateLimitable

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

### 5.2 Campaign 限流现状

**重要发现**: **Chatwoot 内部对 Campaign 没有实现任何限流机制**

#### 5.2.1 发送方式

所有 `oneoff_campaign_service` 均采用 **同步顺序发送**:

```ruby
# Whatsapp::OneoffCampaig
```

- **无批量发送**: 逐个调用 API
- **无延迟间隔**: 无 `sleep` 或 `delay`
- **无队列拆分**: 所有联系人在同一个 Job 中处理
- **无限流检查**: 发送前不检查任何限制

#### 5.2.2 外部平台限制

##### WhatsApp (Meta Cloud API)

**文件**: `app/services/whatsapp/health_service.rb:40-56`

Chatwoot 会查询 Meta 平台的限制信息：

```ruby
def health_fields
  %w[
    id
    quality_rating
    messaging_limit_tier      # 消息限制等级
    code_verification_status
    account_mode
    display_phone_number
    name_status
    verified_name
    webhook_configuration
    throughput                  # 吞吐量
    last_onboarded_time
    platform_type
    certificate
  ].join(',')
end
```

**Messaging Limit Tier** (来自 Meta 平台):

- `TIER_1000`: 1000 条/24 小时
- `TIER_10000`: 10000 条/24 小时
- 更高等级...

**关键问题**: Chatwoot **仅查询显示**这些信息，**不使用它们进行限流**

##### Twilio / Bandwidth

- 限流完全由平台侧控制
- Chatwoot 不实现任何应用层限流
- 平台返回的错误被捕获并记录，但不影响后续发送

### 5.3 风险分析

#### 超量投递风险

1. **同一 Job 内同步发送大量消息**:
   - 没有分批
   - 没有延迟
   - 可能导致 Job 执行时间过长

2. **无平台限制感知**:
   - 超过 WhatsApp `messaging_limit_tier` 后继续发送
   - 可能导致 Meta 平台封禁或降质

3. **账户级无保护**:
   - 无每日/每小时发送配额
   - 无账户级发送频率限制

#### 错误处理

- **单条失败不影响整体**: 捕获异常后继续处理后续联系人
- **无重试机制**: 失败的消息不会重试
- **状态不一致**: campaign 立即标记为 `completed`，即使部分消息发送失败

## 6. 建议改进

### 6.1 应用层限流

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

### 6.2 分批发送

1. **分批处理联系人**:
   ```ruby
   contacts.in_batches(of: 100).each_with_index do |batch, index|
     Campaigns::SendBatchJob.set(wait: index * 1.minute).perform_later(campaign, batch.ids)
   end
   ```

2. **引入延迟**:
   - 批量间添加延迟
   - 平滑发送速率

### 6.3 状态追踪

1. **精细的活动状态**:
   - `processing`: 正在发送
   - `completed`: 全部完成
   - `failed`: 部分/全部失败
   - `paused`: 暂停

2. **发送进度追踪**:
   - 已发送/总数量
   - 成功/失败统计

## 7. 总结

| 维度 | 现状 | 风险 |
|-----|------|------|
| 跨渠道调度 | 基于 `inbox.inbox_type` 路由 | 架构清晰，但扩展性有限 |
| WhatsApp 限流 | 仅查询 `messaging_limit_tier`，不使用 | 超量后平台可能封禁 |
| SMS 限流 | 无应用层限流 | 依赖平台侧限制 |
| 批量发送 | 同步逐个发送 | Job 可能超时，API 可能限流 |
| 错误处理 | 单条失败不影响整体 | 无重试，状态不一致 |

**核心结论**: Chatwoot 的 Campaign 系统缺少应用层限流机制，大量发送时可能触发外部平台限制。建议引入分批发送和应用层限流策略。
