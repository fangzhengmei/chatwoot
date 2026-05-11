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

### 5.4 Rack::Attack 限流规则分析

**文件**: `config/initializers/rack_attack.rb`

#### 5.4.1 全局配置

| 配置项 | 代码位置 | 配置值 | 说明 |
|-------|---------|--------|------|
| **开关** | `rack_attack.rb:280` | 生产环境默认 `true`，受 `ENABLE_RACK_ATTACK` 控制 | 可通过环境变量禁用 |
| **Cache Store** | `rack_attack.rb:19` | `RedisCacheStore` 使用 `$velma` Redis 连接 | 限流计数存储在 Redis |
| **白名单** | `rack_attack.rb:48` | `127.0.0.1`, `::1` + `RACK_ATTACK_ALLOWED_IPS` | 本地和配置的 IP 绕过所有限流 |

#### 5.4.2 Widget API 限流规则

Widget API 限流受 `ENABLE_RACK_ATTACK_WIDGET_API` 控制（默认 `true`），可通过环境变量关闭。

**代码结构** (`rack_attack.rb:176-191`):

```ruby
if ActiveModel::Type::Boolean.new.cast(ENV.fetch('ENABLE_RACK_ATTACK_WIDGET_API', true))
  ## Prevent Conversation Bombing on Widget APIs ###
  throttle('api/v1/widget/conversations', limit: 6, period: 12.hours) do |req|
    req.ip if req.path_without_extentions == '/api/v1/widget/conversations' && req.post?
  end

  ## Prevent Contact update Bombing in Widget API ###
  throttle('api/v1/widget/contacts', limit: 60, period: 1.hour) do |req|
    req.ip if req.path_without_extentions == '/api/v1/widget/contacts' && (req.patch? || req.put?)
  end

  ## Prevent Conversation Bombing through multiple sessions
  throttle('widget?website_token={website_token}&cw_conversation={x-auth-token}', limit: 5, period: 1.hour) do |req|
    req.ip if req.path_without_extentions == '/widget' && ActionDispatch::Request.new(req.env).params['cw_conversation'].blank?
  end
end
```

**逐规则核对**:

| 限流规则 | 代码位置 | 路径匹配 | HTTP 方法 | 计数键 | 阈值 | 窗口 | 是否覆盖 `/api/v1/widget/events` | 是否覆盖 `/api/v1/widget/config` |
|---------|---------|---------|-----------|--------|------|------|-------------------------------|-------------------------------|
| **req/ip** (全局) | `rack_attack.rb:70` | **所有路径** | 所有 | `req.ip` | 3000 (可通过 `RACK_ATTACK_LIMIT` 配置) | 1 分钟 | ✅ **覆盖**（作为全局 fallback） | ✅ **覆盖**（作为全局 fallback） |
| **api/v1/widget/conversations** | `rack_attack.rb:178-180` | `/api/v1/widget/conversations` | POST | `req.ip` | 6 | 12 小时 | ❌ **不覆盖** | ❌ **不覆盖** |
| **api/v1/widget/contacts** | `rack_attack.rb:183-185` | `/api/v1/widget/contacts` | PATCH/PUT | `req.ip` | 60 | 1 小时 | ❌ **不覆盖** | ❌ **不覆盖** |
| **widget load** | `rack_attack.rb:188-190` | `/widget` | 所有 | `req.ip` | 5 | 1 小时 | ❌ **不覆盖** | ❌ **不覆盖** |

**关键发现**: 
- `/api/v1/widget/events` **没有专门的限流规则**
- `/api/v1/widget/config` **没有专门的限流规则**
- 两者都只受最宽松的全局 `req/ip` 规则限制：

#### 5.4.2.1 `/api/v1/widget/config` 接口限流分析

**路由**: `config/routes.rb:422` → `resource :config, only: [:create]`

**控制器**: `app/controllers/api/v1/widget/configs_controller.rb`

**核心逻辑**:

```ruby
def create
  build_contact   # 创建新 contact_inbox（如果不存在）
  set_token       # 下发 X-Auth-Token (JWT)
end

def build_contact
  return if @contact.present?
  @contact_inbox = @web_widget.create_contact_inbox(additional_attributes)
  @contact = @contact_inbox.contact
end

def set_token
  payload = { source_id: @contact_inbox.source_id, inbox_id: @web_widget.inbox.id }
  @token = ::Widget::TokenService.new(payload: payload).generate_token
end
```

**Token 有效期**: `app/services/widget/token_service.rb:24-26`

```ruby
DEFAULT_EXPIRY_DAYS = 180

def expire_in
  token_expiry_value = InstallationConfig.find_by(name: 'WIDGET_TOKEN_EXPIRY')&.value
  (token_expiry_value.presence || DEFAULT_EXPIRY_DAYS).to_i
end
```

**`/api/v1/widget/config` 限流评估**:

| 维度 | 评估结果 | 说明 |
|-----|---------|------|
| **是否被限流** | ✅ 是，但非常宽松 | 仅受 `req/ip` 全局规则限制 |
| **专门规则** | ❌ 无 | 没有针对 config 接口的专门限流 |
| **计数维度** | ❌ 仅按 IP | 不按 website_token 维度 |
| **阈值** | 3000 req/min (默认) | 每次调用都可能创建新的 contact_inbox |
| **影响** | ⚠️ 高 | 可被用于批量创建 contact_inbox，为后续 campaign 触发铺路 |

#### 5.4.2.2 `ENABLE_RACK_ATTACK_WIDGET_API` 关闭时的规则影响

**修正结论**: 当 `ENABLE_RACK_ATTACK_WIDGET_API=false` 时：

| 限流规则 | 是否失效 | 原因 |
|---------|---------|------|
| `req/ip` (全局) | ❌ **仍生效** | 在 `if` 块**外部**定义，不受此开关控制 |
| `api/v1/widget/conversations` | ✅ **失效** | 在 `if` 块**内部**定义 |
| `api/v1/widget/contacts` | ✅ **失效** | 在 `if` 块**内部**定义 |
| `widget?website_token=...` | ✅ **失效** | 在 `if` 块**内部**定义 |
| 所有认证相关限流 (login/reset_password 等) | ❌ **仍生效** | 在另一个独立的 `if` 块中 |
| 所有应用 API 限流 (reports/upload 等) | ❌ **仍生效** | 在另一个独立的 `if` 块中 |

**关键修正**: 之前的结论认为关闭此开关会完全关闭 Widget 限流，这是**错误的**。实际上：
- `ENABLE_RACK_ATTACK_WIDGET_API` **只控制 4 条 Widget 专用规则**
- **全局 `req/ip` 规则（3000/min）不受影响**，始终生效
- 这意味着即使关闭 Widget 专用限流，仍有 3000 req/min 的兜底保护

#### 5.4.2.3 环境开关组合风险矩阵

| `ENABLE_RACK_ATTACK` | `ENABLE_RACK_ATTACK_WIDGET_API` | 全局 `req/ip` 规则 | Widget 专用规则 | 风险等级 | 风险说明 |
|---------------------|--------------------------------|------------------|---------------|---------|---------|
| `true` (默认) | `true` (默认) | ✅ 3000 req/min | ✅ 4 条规则生效 | ⚠️ 中等 | 阈值过高，但有多层保护 |
| `true` (默认) | `false` | ✅ 3000 req/min | ❌ 全部失效 | ⚠️ 高 | Widget 专用保护消失，仅剩全局兜底 |
| `false` | `true` | ❌ 完全失效 | ⚠️ 条件依赖 | 🔴 **极高** | Rack::Attack 整体关闭，所有规则失效 |
| `false` | `false` | ❌ 完全失效 | ❌ 完全失效 | 🔴 **极高** | 无任何限流保护 |

**注意**: `ENABLE_RACK_ATTACK` 在生产环境默认为 `true`，可完全禁用整个 Rack::Attack。当 `ENABLE_RACK_ATTACK=false` 时，**所有限流规则都不生效**，包括全局 `req/ip` 规则。

#### 5.4.2.4 真实攻击路径：config → events 完整链路

**正常用户流程**:

```
1. GET /widget?website_token=xxx
   ↓
   WidgetsController#index (服务端渲染)
   ↓
   build_contact (创建 contact_inbox)
   ↓
   set_token (下发 X-Auth-Token)
   ↓
2. 前端加载后：
   - 等待 time_on_page 秒
   - 检查 URL 匹配
   - 检查营业时间
   ↓
3. POST /api/v1/widget/events
   ↓
   Dispatcher → CampaignListener → CampaignConversationBuilder
   ↓
   创建 Conversation + Message
```

**攻击者流程（绕过前端）**:

```
1. POST /api/v1/widget/config?website_token=xxx
   └─ 限流门槛：req/ip (3000/min)
   └─ 每次调用创建新的 contact_inbox
   └─ 返回 X-Auth-Token (JWT，有效期 180 天)
   ↓
2. POST /api/v1/widget/events?website_token=xxx
   body: { name: 'campaign.triggered', event_info: {...} }
   Headers: X-Auth-Token: <token_from_step_1>
   └─ 限流门槛：req/ip (3000/min)
   └─ 使用新的 contact_inbox 触发 campaign
   └─ 行级锁 + 幂等检查：对新 contact_inbox 无效
   ↓
3. 重复步骤 1-2，每次使用不同的 source_id
   └─ 每分钟可创建 3000 个 contact_inbox + 3000 个 campaign 触发
```

**每一步的限流门槛分析**:

| 步骤 | 接口 | 限流规则 | 阈值 | 是否可绕过 | 攻击成本 |
|-----|------|---------|------|-----------|---------|
| 1. 获取 token | `POST /api/v1/widget/config` | `req/ip` (全局) | 3000 req/min | 分布式攻击可绕过 | 低 |
| 2. 触发 campaign | `POST /api/v1/widget/events` | `req/ip` (全局) | 3000 req/min | 分布式攻击可绕过 | 低 |
| 2a. 后端幂等 | `CampaignConversationBuilder` | 行级锁 + 对话存在检查 | 1 per contact_inbox | 使用新 contact_inbox 绕过 | 零 |

**关键发现**: 
- 攻击者可以**每秒创建 50 个新 contact_inbox**（3000/min）
- 每个新 contact_inbox 可以**成功触发一次 campaign**
- **行级锁和幂等检查对新用户无效**
- **分布式攻击可将容量扩大 N 倍**（N = IP 数量）

```ruby
# rack_attack.rb:70
throttle('req/ip', limit: ENV.fetch('RACK_ATTACK_LIMIT', '3000').to_i, period: 1.minute, &:ip)
```

- **计数键**: `"rack::attack:#{Time.now.to_i/:period}:req/ip:#{req.ip}"`
- **仅按 IP 计数**: 不区分 `website_token`、`contact_id`、`campaign_id`
- **默认阈值**: 3000 次/分钟（约 50 次/秒）
- **可配置**: 可通过 `RACK_ATTACK_LIMIT` 环境变量调整

#### 5.4.3 `/api/v1/widget/events` 限流评估

| 维度 | 评估结果 | 说明 |
|-----|---------|------|
| **是否被限流** | ✅ 是，但非常宽松 | 受 `req/ip` 全局规则限制 |
| **计数维度** | ❌ 仅按 IP | 不按 website_token、用户、campaign 维度 |
| **阈值** | 3000 req/min (默认) | 对于 campaign 触发场景过于宽松 |
| **绕过风险** | ⚠️ 中等 | 可通过 `ENABLE_RACK_ATTACK_WIDGET_API=false` 完全关闭 Widget 限流 |
| **分布式攻击** | ❌ 无法防御 | IP 分散的攻击无法通过 IP 限流阻止 |

### 5.5 分层防护表

| 防护层 | 实现位置 | 防护能力 | 能防什么 | 防不了什么 | 风险漏洞 |
|-------|---------|---------|---------|-----------|---------|
| **L1: 前端触发延迟** | `campaignTimer.js:11` | `setTimeout(() => {...}, timeOnPage * 1000)` | 正常用户不会立即触发，需要在页面停留 N 秒 | 攻击者可直接调用 API，绕过前端延迟 | 前端保护可被完全绕过 |
| **L2: 前端 URL 匹配** | `campaignHelper.js:33-46` | `filterCampaigns()` 检查 URL 模式和营业时间 | 只有匹配配置的页面才会触发 | 攻击者可直接调用 API，绕过 URL 检查 | 前端保护可被完全绕过 |
| **L3: Rack::Attack 全局限流** | `rack_attack.rb:70` | 3000 req/min per IP | 单 IP 高频攻击 | 分布式攻击（多 IP）、同 IP 不同 website_token 无法区分 | 阈值过高，计数维度单一 |
| **L4: 后端行级锁** | `campaign_conversation_builder.rb:9` | `@contact_inbox.lock!` | 同一 contact_inbox 的并发请求串行化 | 不同 contact_inbox 的请求不受影响 | 仅防止同一用户重复创建，不防高频 |
| **L5: 后端幂等检查** | `campaign_conversation_builder.rb:12` | `if @contact_inbox.reload.conversations.present?` | 同一 contact_inbox 已创建对话则不再创建 | 新用户（新 contact_inbox）每次都能创建 | 仅去重，不防高频新用户 |

**分层防护效果图**:

```
正常用户流：
  L1(延迟) → L2(URL匹配) → L3(RackAttack) → L4(锁) → L5(幂等) → ✅ 创建成功

攻击者流（直接调用 API）：
  ❌ 绕过 L1+L2 → L3(3000/min) → 大量新 contact_inbox → L4+L5 无法阻挡 → ❌ 超量投递
```

### 5.6 Website 渠道风险分析

| 风险点 | 现状 | 风险等级 |
|-------|------|---------|
| **前端频率控制** | 有 `time_on_page` 延迟 + URL 匹配 + 营业时间 | 中等（可被绕过） |
| **后端幂等** | 有行级锁 + 对话存在检查 | 低（已实现） |
| **Rack::Attack 限流** | 有但过于宽松（3000 req/min per IP），无专门规则 | 高 |
| **计数维度** | 仅按 IP，不按 website_token/contact/campaign | 高 |
| **可被完全关闭** | `ENABLE_RACK_ATTACK_WIDGET_API=false` 可关闭 Widget 限流 | 高 |
| **同一用户重复触发** | 依赖 contact_inbox.conversations.present? 检查 | 低（已实现） |

**关键发现**: Website 渠道的幂等约束在后端通过数据库锁和存在检查实现，但缺少应用层的频率限制（如每 N 分钟最多触发一次）。Rack::Attack 虽然有全局限流，但阈值过高且维度单一，无法有效防止 campaign 超量投递。

---

## 5.7 高并发触发场景时序推演

### 5.7.1 场景描述

**场景**: 恶意攻击者或配置错误导致在短时间内大量触发同一个 Website Campaign

**前提条件**:
- 网站有大量匿名访客（每个访客创建新的 `contact_inbox`）
- `time_on_page = 5`（前端延迟 5 秒）
- `RACK_ATTACK_LIMIT = 3000`（默认值）
- Campaign 配置为所有页面触发

### 5.7.2 时序推演

```
时间轴: T0 到 T10（单位：秒）

T0:
  - 攻击者脚本启动，同时创建 1000 个浏览器实例
  - 或：CDN 缓存失效，大量真实用户同时访问

T1:
  - 正常用户流：等待 `time_on_page` 5 秒
  - 攻击者流：直接调用 API，绕过前端

T1 - T60（第一个 1 分钟窗口）:
  
  [攻击者流 - 直接调用 API]
  ┌─────────────────────────────────────────────────────┐
  │ POST /api/v1/widget/events (name: 'campaign.triggered')
  │ body: { website_token, event, page, ... }
  │
  │ T1: 第 1 个请求 → 进入 EventsController
  │         ↓
  │      Dispatcher.dispatch('campaign.triggered', ...)
  │         ↓
  │      CampaignListener#campaign_triggered
  │         ↓
  │      CampaignConversationBuilder#perform
  │         ↓
  │      ContactInbox.find(...)  ← 新 contact_inbox，无锁冲突
  │         ↓
  │      @contact_inbox.lock!     ← 获取锁（无竞争，立即成功）
  │         ↓
  │      contact_inbox.conversations.present? → false
  │         ↓
  │      Conversation.create!    ← ✅ 创建成功
  │
  │ T1: 第 2 个请求（不同 contact_inbox）
  │         ↓
  │      同样的流程...            ← ✅ 创建成功
  │
  │ ...
  │
  │ T1: 第 N 个请求（不同 contact_inbox）
  │         ↓
  │      同样的流程...            ← ✅ 创建成功
  │
  │ Rack::Attack 计数:
  │   每秒钟发送 50 次 → 1 分钟 3000 次 → 刚好到达阈值
  │   第 3001 次请求被拦截（返回 429）
  └─────────────────────────────────────────────────────┘

  [正常用户流 - 经过前端]
  ┌─────────────────────────────────────────────────────┐
  │ T5: 第一批用户的 time_on_page 延迟结束
  │         ↓
  │      executeCampaign() → POST /api/v1/widget/events
  │         ↓
  │      同上流程，但用户量通常可控
  └─────────────────────────────────────────────────────┘

T61（第二个 1 分钟窗口）:
  - Rack::Attack 计数器重置
  - 攻击者可以继续发送 3000 次请求
  - 理论上无限循环...
```

### 5.7.3 超量投递路径分析

| 路径编号 | 攻击场景 | 超量机制 | 是否可防御 |
|---------|---------|---------|-----------|
| **路径 1** | 单 IP 高频攻击 | 利用 Rack::Attack 3000/min 的高阈值 | ❌ 无法完全防御，1 分钟内可创建 3000 个对话 |
| **路径 2** | 分布式攻击（多 IP） | 每个 IP 独立计数，10 个 IP = 30000/min | ❌ 完全无法防御 |
| **路径 3** | 关闭 Widget 限流 | `ENABLE_RACK_ATTACK_WIDGET_API=false` | ❌ 完全无法防御，无任何限制 |
| **路径 4** | 白名单 IP 绕过 | IP 在 `RACK_ATTACK_ALLOWED_IPS` 中 | ❌ 白名单 IP 不受任何限流 |
| **路径 5** | 大量真实新用户 | 网站流量突发，正常用户行为 | ❌ 无法区分正常/异常，全部放行 |

### 5.7.4 数据库压力分析

```
每次触发的数据库操作（CampaignConversationBuilder#perform）:

1. SELECT * FROM contact_inboxes WHERE id = ?    (1 query)
2. SELECT * FROM campaigns WHERE inbox_id = ? AND display_id = ?  (1 query)
3. BEGIN TRANSACTION
4. SELECT * FROM contact_inboxes WHERE id = ? FOR UPDATE  (加锁，1 query)
5. SELECT COUNT(*) FROM conversations WHERE contact_inbox_id = ?  (1 query)
6. INSERT INTO conversations (...)                (1 query)
7. INSERT INTO messages (...)                     (1 query)
8. COMMIT

总计: 约 5-7 个 SQL 查询 + 2 个 INSERT
```

**压力估算**（3000 req/min）:
- 约 18000-21000 SQL 查询/分钟
- 约 6000 INSERT/分钟
- 数据库连接池可能耗尽

### 5.8 最小改造建议

#### 方案 A：新增 Rack::Attack 专门规则（最小改动，推荐）

**改动文件**: `config/initializers/rack_attack.rb`

**新增规则**:

```ruby
# 防止 Website Campaign 高频触发
throttle('api/v1/widget/events/website_token', limit: 100, period: 1.hour) do |req|
  if req.path_without_extentions == '/api/v1/widget/events' && req.post?
    params = ActionDispatch::Request.new(req.env).params
    "#{params['website_token']}:#{params.dig('event', 'name')}"
  end
end

# 按 contact_inbox 维度限流（如果有 contact_inbox_id）
throttle('api/v1/widget/events/contact_inbox', limit: 1, period: 1.day) do |req|
  if req.path_without_extentions == '/api/v1/widget/events' && req.post?
    params = ActionDispatch::Request.new(req.env).params
    "#{params['website_token']}:#{params['contact_inbox_id']}" if params['contact_inbox_id'].present?
  end
end
```

**改动量**: ~15 行代码

**防护效果**:
- 每个 website_token 每小时最多触发 100 次 campaign 事件
- 同一 contact_inbox 每天最多触发 1 次

**风险**: 无

#### 方案 B：应用层限流（更精准，中等改动）

**改动文件**:
- `app/controllers/api/v1/widget/events_controller.rb` 或 `app/builders/campaigns/campaign_conversation_builder.rb`

**新增限流逻辑**:

```ruby
# 在 CampaignConversationBuilder 中
def perform
  # 新增：应用层限流检查
  return nil unless within_campaign_rate_limit?

  # 原有逻辑...
end

private

def within_campaign_rate_limit?
  # 按 campaign + contact_inbox 维度，每 24 小时最多 1 次
  key = "campaign_rate_limit:#{@campaign.id}:#{@contact_inbox.id}"
  $velma.with do |redis|
    if redis.exists(key)
      Rails.logger.info("[Campaign Throttle] Blocked: #{key}")
      return false
    end
    redis.setex(key, 24.hours.to_i, '1')
    true
  end
end
```

**改动量**: ~20-30 行代码

**防护效果**:
- 同一 contact_inbox 在 24 小时内不会被同一 campaign 重复触发
- 比幂等检查更早生效，减少数据库压力

**风险**: 低

#### 方案 C：综合方案（推荐）

| 改动 | 文件 | 改动量 | 防护目标 |
|-----|------|-------|---------|
| 新增 Rack::Attack 规则 | `config/initializers/rack_attack.rb` | ~15 行 | 按 website_token 限流，防止单网站高频触发 |
| 新增应用层限流 | `app/builders/campaigns/campaign_conversation_builder.rb` | ~20 行 | 按 contact_inbox 限流，防止重复创建 |
| 增加日志监控 | 新增 `Rack::Attack` throttle 日志（已有） | 0 行 | 监控限流触发情况 |

**防护效果矩阵（改造后）**:

| 攻击场景 | 方案 A | 方案 B | 方案 C |
|---------|-------|-------|-------|
| 单 IP 高频攻击 | ✅ 100/h per website_token | ❌ 不限 IP | ✅ 双重防护 |
| 分布式攻击（多 IP） | ✅ 100/h per website_token | ✅ 1/day per contact_inbox | ✅ 完全防护 |
| 关闭 Widget 限流 | ❌ 规则被禁用 | ✅ 应用层仍生效 | ⚠️ 应用层生效 |
| 白名单 IP 绕过 | ❌ Rack::Attack 被跳过 | ✅ 应用层仍生效 | ⚠️ 应用层生效 |
| 大量真实新用户 | ⚠️ 可能误杀正常用户 | ✅ 仅限制重复 | ✅ 可调整阈值 |

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
