# Chatwoot 出站通知与 Webhook 投递机制分析

## 1. 架构概览

Chatwoot 采用**异步作业队列 + 事件监听**的架构模式处理出站通知，主要包括：
- **Slack 集成通知**：通过 `SendOnSlackJob` 处理
- **外部 Webhook 通知**：通过 `WebhookJob` 和 `AgentBots::WebhookJob` 处理
- **多渠道消息投递**：通过 `SendReplyJob` 路由到各渠道服务

### 核心组件关系图

```
消息创建/更新
    ↓
事件分发 (ActiveRecord callbacks + Wisper)
    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                           事件监听器层                                    │
├────────────────────────────┬────────────────────────────┬────────────────┤
│ WebhookListener            │ AgentBotListener           │ HookListener   │
│ (外部 Webhook: 账户/API)    │ (Agent Bot 回调)            │ (集成 Hooks)   │
└────────────────────────────┴────────────────────────────┴────────────────┘
    ↓                                    ↓                          ↓
WebhookJob                    AgentBots::WebhookJob            HookJob
(medium queue)                (high queue)                    (medium queue)
    ↓                                    ↓                          ↓
Webhooks::Trigger.execute    Webhooks::Trigger.execute    ┌───────────────┐
(HTTP POST)                   (HTTP POST, 带重试)           │ SendOnSlackJob│
                                                           │ (medium queue)│
                                                           │ + 分布式锁    │
                                                           └───────────────┘
                                                                  ↓
                                                     Integrations::Slack::SendOnSlackService
```

---

## 2. 外部回调分层详解

### 2.1 回调类型分层总览

Chatwoot 存在三种独立的外部回调机制，各自有不同的触发源、数据模型和失败处理边界：

| 回调类型 | 数据模型 | 监听器 | 作业类 | 队列 | 核心用途 |
|----------|----------|--------|--------|------|----------|
| **账户级 Webhook** | `Webhook` (webhook_type: account_type) | `WebhookListener` | `WebhookJob` | medium | 账户级别事件广播（CRM/数据分析） |
| **API 渠道回调** | `Channel::Api` (webhook_url 字段) | `WebhookListener` | `WebhookJob` | medium | API 渠道的消息事件通知 |
| **Agent Bot 回调** | `AgentBot` (outgoing_url 字段) | `AgentBotListener` | `AgentBots::WebhookJob` | high | 智能体机器人交互（对话自动化） |

---

### 2.2 分层一：账户级 Webhook (Account/Inbox Webhook)

#### 数据模型
- **表**：`webhooks`
- **关键字段**：
  - `webhook_type`：`account_type`(0) 或 `inbox_type`(1)
  - `url`：回调 URL
  - `secret`：签名密钥（自动生成，支持加密存储）
  - `subscriptions`：订阅的事件类型数组（JSONB）
  - `account_id` / `inbox_id`：作用域

- 文件位置：`app/models/webhook.rb:21-44`

#### 触发机制
- **监听器**：`WebhookListener`（`app/listeners/webhook_listener.rb`）
- **事件订阅**：
  ```ruby
  ALLOWED_WEBHOOK_EVENTS = %w[
    conversation_status_changed conversation_updated conversation_created
    contact_created contact_updated message_created message_updated
    webwidget_triggered inbox_created inbox_updated
    conversation_typing_on conversation_typing_off
  ].freeze
  ```
  - 文件位置：`app/models/webhook.rb:32-34`

- **入队逻辑**：
  ```ruby
  def deliver_account_webhooks(payload, account)
    account.webhooks.account_type.each do |webhook|
      next unless webhook.subscriptions.include?(payload[:event])
      WebhookJob.perform_later(webhook.url, payload, :account_webhook,
                               secret: webhook.secret,
                               delivery_id: SecureRandom.uuid)
    end
  end
  ```
  - 文件位置：`app/listeners/webhook_listener.rb:110-118`

#### 失败处理边界
| 处理维度 | 行为 | 边界说明 |
|----------|------|----------|
| **自动重试** | ❌ 无 | 完全不重试，无论任何错误 |
| **状态更新** | ❌ 无 | 不更新任何业务数据状态 |
| **消息状态** | ❌ 不影响 | Webhook 失败不影响消息本身的发送状态 |
| **对话流转** | ❌ 不影响 | 不会触发对话状态变更 |
| **日志记录** | ✅ 有 | `Rails.logger.warn "Exception: Invalid webhook URL #{url} : #{error.message}"` |
| **异常上报** | ❌ 无 | 不调用 Sentry/异常追踪 |
| **死信队列** | ❌ 无 | Sidekiq 默认重试可能介入，但非主动设计 |

**边界结论**：账户级 Webhook 是"尽力而为"的通知机制，失败即丢弃，业务系统需要自行保证数据一致性或实现补偿逻辑。

---

### 2.3 分层二：API 渠道回调 (API Inbox Webhook)

#### 数据模型
- **表**：`channel_api`
- **关键字段**：
  - `webhook_url`：回调 URL
  - `secret`：签名密钥
  - `hmac_token`：HMAC 验证令牌（用于入站消息验证）
  - `hmac_mandatory`：是否强制 HMAC 验证
  - `identifier`：渠道唯一标识

- 文件位置：`app/models/channel/api.rb:22-46`

#### 触发机制
- **监听器**：`WebhookListener`（同一监听器，不同分支）
- **入队条件**：
  ```ruby
  def deliver_api_inbox_webhooks(payload, inbox)
    return unless inbox.channel_type == 'Channel::Api'
    return if inbox.channel.webhook_url.blank?

    WebhookJob.perform_later(inbox.channel.webhook_url, payload, :api_inbox_webhook,
                             secret: inbox.channel.secret, delivery_id: SecureRandom.uuid)
  end
  ```
  - 文件位置：`app/listeners/webhook_listener.rb:120-126`

- **触发事件**：与账户级 Webhook 相同的 12 种事件

#### 失败处理边界
| 处理维度 | 行为 | 边界说明 |
|----------|------|----------|
| **自动重试** | ❌ 无 | 与账户级相同，无重试机制 |
| **状态更新** | ⚠️ 部分 | 仅对 `message_created` / `message_updated` 事件更新消息状态 |
| **消息状态** | ⚠️ 条件更新 | 调用 `Messages::StatusUpdateService.new(message, 'failed', error.message).perform` |
| **对话流转** | ❌ 不影响 | 不会触发对话状态变更 |
| **日志记录** | ✅ 有 | 同账户级 Webhook |
| **异常上报** | ❌ 无 | 不调用 Sentry |

**关键代码**：
```ruby
def handle_error(error)
  return unless SUPPORTED_ERROR_HANDLE_EVENTS.include?(@payload[:event])
  return unless message

  case @webhook_type
  when :agent_bot_webhook
    update_conversation_status(message)
  when :api_inbox_webhook
    update_message_status(error)  # ← API 渠道特有
  end
end
```
- 文件位置：`lib/webhooks/trigger.rb:65-75`

**边界结论**：API 渠道回调在消息事件失败时会标记消息为 `failed`，但其他事件（如对话创建、联系人更新）失败时无任何状态追溯。消息状态更新仅作为"已尝试"标记，不触发重试。

---

### 2.4 分层三：Agent Bot 回调 (Agent Bot Webhook)

#### 数据模型
- **表**：`agent_bots`
- **关键字段**：
  - `outgoing_url`：回调 URL（注意：字段名不是 webhook_url）
  - `secret`：签名密钥
  - `bot_type`：仅支持 `webhook`(0)
  - `bot_config`：机器人配置（JSONB）
  - `account_id`：可空，空表示系统级机器人

- 文件位置：`app/models/agent_bot.rb:21-69`

#### 触发机制
- **监听器**：`AgentBotListener`（`app/listeners/agent_bot_listener.rb`）
- **触发条件**：
  - Inbox 关联的激活机器人：`inbox.agent_bot_inbox&.active?`
  - 对话指定的机器人：`conversation.assignee_agent_bot`
- **入队逻辑**：
  ```ruby
  def process_webhook_bot_event(agent_bot, payload)
    return if agent_bot.outgoing_url.blank?

    AgentBots::WebhookJob.perform_later(agent_bot.outgoing_url, payload, :agent_bot_webhook,
                                        secret: agent_bot.secret, delivery_id: SecureRandom.uuid)
  end
  ```
  - 文件位置：`app/listeners/agent_bot_listener.rb:85-90`

#### 失败处理边界（最复杂的分层）

**A. 可重试错误（主动设计）**
```ruby
RETRYABLE_AGENT_BOT_STATUSES = [429, 500].freeze
```
- 文件位置：`lib/webhooks/trigger.rb:3`

**重试配置**：
```ruby
retry_on Webhooks::Trigger::RetryableError, wait: 3.seconds, attempts: 3 do |job, error|
  url, payload, webhook_type = job.arguments
  kwargs = job.arguments.last.is_a?(Hash) ? job.arguments.last : {}
  Webhooks::Trigger.new(url, payload, webhook_type || :agent_bot_webhook,
                        secret: kwargs[:secret], delivery_id: kwargs[:delivery_id]).handle_failure(error)
end
```
- 文件位置：`app/jobs/agent_bots/webhook_job.rb:3-8`

**B. 重试耗尽后的降级处理**
```ruby
def update_conversation_status(message)
  conversation = message.conversation
  return unless conversation&.pending?
  return if conversation&.account&.keep_pending_on_bot_failure

  conversation.open!
  create_agent_bot_error_activity(conversation)
end
```
- 文件位置：`lib/webhooks/trigger.rb:77-84`

**完整边界矩阵**：

| 处理维度 | 行为 | 边界说明 |
|----------|------|----------|
| **自动重试** | ✅ 有 | 仅对 HTTP 429（限流）和 500（服务器错误）重试 |
| **重试策略** | 固定间隔 | 等待 3 秒，最多 3 次 |
| **状态更新** | ⚠️ 条件 | 重试耗尽后，仅对 pending 状态的对话执行降级 |
| **对话流转** | ✅ 有条件 | `conversation.open!`（可通过 `keep_pending_on_bot_failure` 禁用） |
| **活动记录** | ✅ 有 | 创建活动消息：`conversations.activity.agent_bot.error_moved_to_open` |
| **日志记录** | ✅ 有 | 每次重试失败都记录警告日志 |
| **异常上报** | ❌ 无 | 不调用 Sentry |

**账户级开关**：
```ruby
'keep_pending_on_bot_failure': { 'type': %w[boolean null] }
```
- 文件位置：`app/models/concerns/account_settings_schema.rb:13`

**边界结论**：Agent Bot 回调是唯一具有完整重试 + 降级策略的分层。重试仅针对限流和服务器错误（网络错误/超时不在此列），重试耗尽后通过对话状态流转实现业务降级，确保用户不会永远卡在机器人待处理状态。

---

### 2.5 三种回调的失败边界对比

| 维度 | 账户级 Webhook | API 渠道回调 | Agent Bot 回调 |
|------|---------------|-------------|----------------|
| **重试机制** | 无 | 无 | HTTP 429/500，3次 |
| **消息状态影响** | 无 | 消息事件失败时标记 failed | 无（影响对话状态） |
| **对话状态影响** | 无 | 无 | pending → open（可配置） |
| **活动记录** | 无 | 无 | 机器人错误活动消息 |
| **业务降级** | 无 | 无 | ✅ 完整降级策略 |
| **队列优先级** | medium | medium | high |
| **数据一致性** | 最终一致靠接收方 | 消息状态可追溯 | 对话状态保证不阻塞 |

---

## 3. 投递机制详解

### 3.1 Slack 通知投递机制

#### 触发流程
1. **事件监听**：`HookListener` 监听 `message.created` 和 `message.updated` 事件
   - 文件位置：`app/listeners/hook_listener.rb:58-71`

2. **作业入队**：`HookJob` 根据 `app_id` 路由到具体集成
   - 文件位置：`app/jobs/hook_job.rb:9-18`
   - 带附件的消息延迟 2 秒发送

3. **分布式锁机制**：`SendOnSlackJob` 继承自 `MutexApplicationJob`
   - 文件位置：`app/jobs/send_on_slack_job.rb:1-10`
   - 锁键格式：`Redis::Alfred::SLACK_MESSAGE_MUTEX`

4. **核心发送服务**：`Integrations::Slack::SendOnSlackService`
   - 文件位置：`lib/integrations/slack/send_on_slack_service.rb:1-215`

#### Slack API 错误处理（非重试型致命错误）
```ruby
rescue Slack::Web::Api::Errors::IsArchived,
       Slack::Web::Api::Errors::AccountInactive,
       Slack::Web::Api::Errors::MissingScope,
       Slack::Web::Api::Errors::InvalidAuth,
       Slack::Web::Api::Errors::ChannelNotFound,
       Slack::Web::Api::Errors::NotInChannel => e
  Rails.logger.error e
  hook.prompt_reauthorization!
  hook.disable
```
- 文件位置：`lib/integrations/slack/send_on_slack_service.rb:105-111`

---

### 3.2 Webhook 请求规范（通用）

#### 三层回调共享的 HTTP 规范
- **HTTP 方法**：POST
- **Content-Type**：application/json
- **超时设置**：默认 5 秒，可通过 `WEBHOOK_TIMEOUT` 全局配置
- **HTTP 客户端**：`SafeFetch`（内置 SSRF 防护）

#### 请求头
```ruby
headers = {
  'Content-Type' => 'application/json',
  'Accept' => 'application/json',
  'X-Chatwoot-Delivery' => @delivery_id,  # UUID，每次请求唯一
  'X-Chatwoot-Timestamp' => ts,           # 当前时间戳
  'X-Chatwoot-Signature' => "sha256=#{HMAC}"  # 仅配置 secret 时
}
```
- 文件位置：`lib/webhooks/trigger.rb:54-63`

#### 签名算法
```ruby
headers['X-Chatwoot-Signature'] = "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', secret, "#{ts}.#{body}")}"
```
- 文件位置：`lib/webhooks/trigger.rb:58-61`

---

### 3.3 多渠道消息投递机制

#### 统一入口：SendReplyJob
- 文件位置：`app/jobs/send_reply_job.rb:1-39`
- 队列优先级：`:high`

#### 支持的渠道服务映射
```ruby
CHANNEL_SERVICES = {
  'Channel::TwitterProfile' => ::Twitter::SendOnTwitterService,
  'Channel::TwilioSms'      => ::Twilio::SendOnTwilioService,
  'Channel::Line'           => ::Line::SendOnLineService,
  'Channel::Telegram'       => ::Telegram::SendOnTelegramService,
  'Channel::Whatsapp'       => ::Whatsapp::SendOnWhatsappService,
  'Channel::Sms'            => ::Sms::SendOnSmsService,
  'Channel::Instagram'      => ::Instagram::SendOnInstagramService,
  'Channel::Tiktok'         => ::Tiktok::SendOnTiktokService,
  'Channel::Email'          => ::Email::SendOnEmailService,
  'Channel::WebWidget'      => ::Messages::SendEmailNotificationService,
  'Channel::Api'            => ::Messages::SendEmailNotificationService
}.freeze
```
- 文件位置：`app/jobs/send_reply_job.rb:4-16`

---

## 4. 现有失败重试策略

### 4.1 队列优先级设计

| 优先级 | 队列名 | 使用场景 |
|--------|--------|----------|
| **high** | `:high` | `SendReplyJob`（渠道消息投递）、`AgentBots::WebhookJob`（机器人 Webhook）、`ActivityMessageJob` |
| **medium** | `:medium` | `SendOnSlackJob`、`WebhookJob`、`HookJob`、`UpdateSlackMessageJob` |
| **low** | `:low` | 投递状态回调、数据同步、后台清理任务 |

### 4.2 现有重试策略汇总

#### A. 分布式锁获取失败重试
| 作业类 | 等待时间 | 最大重试次数 |
|--------|----------|--------------|
| `SendOnSlackJob` | 1 秒 | 8 次 |
| `UpdateSlackMessageJob` | 1 秒 | 8 次 |
| `HookJob` | 3 秒 | 3 次 |
| `FacebookEventsJob` | 1 秒 | 8 次 |
| `InstagramEventsJob` | 1 秒 | 8 次 |
| `WhatsappEventsJob` | 2 秒 | 20 次 |
| `TiktokEventsJob` | 2 秒 | 8 次 |

#### B. Agent Bot Webhook 重试
- HTTP 429 / 500：等待 3 秒，最多 3 次
- 文件位置：`app/jobs/agent_bots/webhook_job.rb:3-8`

### 4.3 消息状态流转模型

**Message 状态枚举**：
```ruby
enum status: { sent: 0, delivered: 1, read: 2, failed: 3 }
```
- 文件位置：`app/models/message.rb:103`

**状态更新服务**：`Messages::StatusUpdateService`
- 文件位置：`app/services/messages/status_update_service.rb:1-34`
- 仅允许 `failed` 状态时设置 `external_error`

---

## 5. 多渠道重试设计方案（建议）

基于现有架构的不足，设计统一的多渠道重试框架。

### 5.1 设计目标
1. **统一抽象**：所有出站操作共享一致的重试语义
2. **分类处理**：按错误类型区分可重试/不可重试
3. **幂等保障**：接收方可以安全处理重复投递
4. **可观测性**：完整的监控、告警、追踪链路
5. **渐进降级**：重试耗尽后有明确的业务降级策略

---

### 5.2 错误分类：可重试 vs 不可重试

#### 分类矩阵

| 错误类别 | 错误类型 | 可重试？ | 重试策略 | 降级策略 |
|----------|----------|----------|----------|----------|
| **网络传输层** | `Net::OpenTimeout` | ✅ 是 | 指数退避 | 无 |
| | `Net::ReadTimeout` | ✅ 是 | 指数退避 | 无 |
| | `SocketError` | ✅ 是 | 指数退避 | 无 |
| | `OpenSSL::SSL::SSLError` | ⚠️ 条件 | 固定间隔（3次） | 证书错误则放弃 |
| | `SafeFetch::FetchError` | ✅ 是 | 指数退避 | 无 |
| **HTTP 4xx** | 400 Bad Request | ❌ 否 | - | 记录，丢弃 |
| | 401 Unauthorized | ❌ 否 | - | 禁用集成/提示重授权 |
| | 403 Forbidden | ❌ 否 | - | 禁用集成/提示重授权 |
| | 404 Not Found | ❌ 否 | - | 记录，丢弃 |
| | 408 Request Timeout | ✅ 是 | 指数退避 | 无 |
| | 429 Too Many Requests | ✅ 是 | 带抖动的指数退避 | 无（读取 Retry-After） |
| **HTTP 5xx** | 500 Internal Server Error | ✅ 是 | 指数退避 | 无 |
| | 502 Bad Gateway | ✅ 是 | 指数退避 | 无 |
| | 503 Service Unavailable | ✅ 是 | 指数退避 | 无 |
| | 504 Gateway Timeout | ✅ 是 | 指数退避 | 无 |
| **业务逻辑层** | 无效签名/认证 | ❌ 否 | - | 提示重新配置 |
| | 资源不存在（渠道侧） | ❌ 否 | - | 禁用集成 |
| | 配额耗尽 | ⚠️ 条件 | 长时间延迟重试 | 告警通知管理员 |
| **并发控制层** | 锁获取失败 | ✅ 是 | 固定间隔 | 无（8次后进入死信） |

---

### 5.3 退避策略设计

#### 策略选型

| 策略类型 | 算法 | 适用场景 | 示例（最大重试 5 次） |
|----------|------|----------|----------------------|
| **指数退避** | `wait = base * (2^attempt)` | 网络抖动、服务限流 | 1s, 2s, 4s, 8s, 16s |
| **指数退避 + 抖动** | `wait = base * (2^attempt) * random(0.5, 1.5)` | 大规模并发场景（避免重试风暴） | 0.8s, 1.7s, 3.5s, 7.8s, 14.2s |
| **固定间隔** | `wait = constant` | 锁竞争、简单场景 | 3s, 3s, 3s, 3s, 3s |
| **线性退避** | `wait = base * attempt` | 逐步增加等待 | 1s, 2s, 3s, 4s, 5s |

#### 建议配置

```ruby
# 统一重试配置（建议新增 config/initializers/outbound_retries.rb）

RETRY_CONFIG = {
  network: {
    strategy: :exponential_backoff_with_jitter,
    base: 1.second,
    max_attempts: 5,
    max_delay: 30.seconds
  },
  http_429: {
    strategy: :exponential_backoff_with_jitter,
    base: 2.seconds,
    max_attempts: 8,
    max_delay: 60.seconds,
    respect_retry_after: true  # 读取响应头 Retry-After
  },
  http_5xx: {
    strategy: :exponential_backoff,
    base: 1.second,
    max_attempts: 5,
    max_delay: 30.seconds
  },
  lock_acquisition: {
    strategy: :fixed,
    delay: 1.second,
    max_attempts: 8
  },
  ssl_error: {
    strategy: :fixed,
    delay: 5.seconds,
    max_attempts: 3
  }
}.freeze
```

#### 指数退避 + 抖动实现
```ruby
def exponential_backoff_with_jitter(attempt, base = 1.second, max_delay = 30.seconds)
  exponential_wait = base * (2 ** attempt)
  jitter = rand(0.5..1.5)
  [exponential_wait * jitter, max_delay].min
end
```

---

### 5.4 幂等键设计

#### 现有幂等标识

Chatwoot 已部分实现幂等支持：
- **`X-Chatwoot-Delivery`**：UUID，每次 Webhook 请求唯一
- **`message.id`**：消息唯一标识
- **`conversation.id`**：对话唯一标识

#### 建议增强：统一幂等键规范

**幂等键生成策略**：
```ruby
# 建议新增 lib/outbound/idempotency_key.rb

module Outbound
  class IdempotencyKey
    # Webhook 幂等键：delivery_id（已存在，保持兼容）
    def self.for_webhook(delivery_id)
      "webhook:#{delivery_id}"
    end

    # 渠道消息幂等键：消息 ID + 渠道类型
    def self.for_channel_message(message_id, channel_type)
      "channel:#{channel_type}:#{message_id}"
    end

    # Slack 消息幂等键：对话 ID + Hook 引用 ID
    def self.for_slack(conversation_id, reference_id)
      "slack:#{conversation_id}:#{reference_id}"
    end

    # Agent Bot 幂等键：机器人 ID + 事件类型 + 实体 ID
    def self.for_agent_bot(bot_id, event, entity_id)
      "bot:#{bot_id}:#{event}:#{entity_id}"
    end
  end
end
```

#### 接收方幂等处理建议

```
接收方实现步骤：
1. 读取请求头 X-Chatwoot-Delivery
2. 查询本地幂等表/缓存是否已处理该 ID
3. 如已处理，返回 200 OK 但不重复执行业务逻辑
4. 如未处理，执行业务逻辑并记录幂等 ID
5. 幂等记录 TTL：建议 24 小时（覆盖最长重试周期）

数据库表设计（可选）：
CREATE TABLE idempotent_deliveries (
  id BIGSERIAL PRIMARY KEY,
  idempotency_key VARCHAR(255) UNIQUE NOT NULL,
  status VARCHAR(20) NOT NULL,  -- pending, completed, failed
  response_code INTEGER,
  created_at TIMESTAMP NOT NULL,
  processed_at TIMESTAMP
);
```

#### Redis 分布式锁 + 幂等双重保障
```ruby
# 发送前检查
def with_idempotency_guarantee(idempotency_key)
  # 1. 检查是否已成功投递
  if Rails.cache.exist?("outbound:success:#{idempotency_key}")
    Rails.logger.info "[Idempotency] Skipping duplicate: #{idempotency_key}"
    return
  end

  # 2. 获取发送锁（防止并发重复发送）
  lock_key = "outbound:lock:#{idempotency_key}"
  lock_manager = Redis::LockManager.new

  if lock_manager.lock(lock_key, 5.minutes)
    begin
      yield
      # 3. 成功后标记（TTL 24 小时）
      Rails.cache.write("outbound:success:#{idempotency_key}", true, expires_in: 24.hours)
    ensure
      lock_manager.unlock(lock_key)
    end
  else
    Rails.logger.info "[Idempotency] Lock held, skipping: #{idempotency_key}"
  end
end
```

---

### 5.5 告警落点设计

#### 告警层次

```
┌─────────────────────────────────────────────────────────────┐
│                        告警层次架构                            │
├─────────────────────────────────────────────────────────────┤
│  Level 1: 实时告警 (P1)                                       │
│  ├── 连续失败 > 阈值（如 10 次/5 分钟）                          │
│  ├── 关键渠道可用性下降（如 WhatsApp 5xx 率 > 5%）              │
│  └── 死信队列新增高优先级消息                                   │
├─────────────────────────────────────────────────────────────┤
│  Level 2: 小时级告警 (P2)                                      │
│  ├── Webhook 失败率 > 10%                                     │
│  ├── 平均重试次数 > 3 次                                       │
│  └── Agent Bot 降级触发次数 > 阈值                              │
├─────────────────────────────────────────────────────────────┤
│  Level 3: 日报告警 (P3)                                       │
│  ├── 各渠道投递成功率汇总                                       │
│  ├── 死信队列规模                                              │
│  └── 重试资源消耗分析                                          │
└─────────────────────────────────────────────────────────────┘
```

#### 告警指标定义

| 指标名 | 类型 | 告警阈值 | 告警级别 | 标签维度 |
|--------|------|----------|----------|----------|
| `outbound_webhook_failure_rate` | Gauge | > 10% (5m) | P2 | account_id, webhook_type |
| `outbound_channel_delivery_success_rate` | Gauge | < 95% (5m) | P1 | channel_type, account_id |
| `outbound_retry_count_total` | Counter | - | - | channel_type, error_type |
| `outbound_dlq_size` | Gauge | > 100 | P1 | priority |
| `outbound_dead_letter_total` | Counter | - | - | job_type, reason |
| `outbound_bot_downgrade_total` | Counter | > 10/hour | P2 | account_id, bot_id |
| `outbound_lock_contention_total` | Counter | - | - | lock_type |

#### 告警实现示例（Prometheus + AlertManager）

```yaml
# alertmanager.yml 示例规则
groups:
  - name: outbound_notifications
    rules:
      - alert: CriticalChannelDeliveryFailure
        expr: outbound_channel_delivery_success_rate{channel_type=~"whatsapp|twilio"} < 0.95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "关键渠道投递成功率低于 95%"
          description: "渠道 {{ $labels.channel_type }} 成功率: {{ $value | humanizePercentage }}"

      - alert: WebhookHighFailureRate
        expr: outbound_webhook_failure_rate > 0.10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Webhook 失败率超过 10%"
          description: "类型 {{ $labels.webhook_type }} 失败率: {{ $value | humanizePercentage }}"

      - alert: DeadLetterQueueGrowing
        expr: outbound_dlq_size{priority="high"} > 100
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "高优先级死信队列堆积"
          description: "当前队列大小: {{ $value }}"
```

#### 告警落点矩阵

| 失败场景 | 日志 | Metrics | Sentry | 业务告警 | 死信队列 |
|----------|------|---------|--------|----------|----------|
| 网络超时/连接错误 | ✅ | `outbound_retry_count_total` | ❌ | 小时级汇总 | 重试耗尽后 |
| HTTP 429 限流 | ✅ | `outbound_retry_count_total` | ❌ | 阈值触发 | 重试耗尽后 |
| HTTP 401/403 认证错误 | ✅ | `outbound_auth_failure_total` | ✅ | 实时告警 | 立即进入 |
| HTTP 5xx 服务端错误 | ✅ | `outbound_retry_count_total` | ❌ | 小时级汇总 | 重试耗尽后 |
| Slack 集成禁用 | ✅ | `outbound_integration_disabled_total` | ✅ | 实时告警 | - |
| Agent Bot 降级 | ✅ | `outbound_bot_downgrade_total` | ❌ | 阈值触发 | - |
| 死信队列新增 | ✅ | `outbound_dead_letter_total` | - | 实时告警 | - |

---

### 5.6 死信队列（DLQ）设计

#### DLQ 触发条件
```
1. 重试次数耗尽（按策略达到 max_attempts）
2. 遇到不可重试的致命错误（401, 403, 404 等）
3. 作业反序列化失败（ActiveJob::DeserializationError）
4. 人工手动移入（运营操作）
```

#### DLQ 数据模型
```ruby
# 建议新增 app/models/outbound_dead_letter.rb

class OutboundDeadLetter < ApplicationRecord
  enum priority: { high: 0, medium: 1, low: 2 }
  enum status: { pending: 0, resolved: 1, discarded: 2 }

  # 序列化的作业参数
  store :job_payload, accessors: [:job_class, :arguments, :executions], coder: JSON

  # 错误信息
  store :error_info, accessors: [:error_class, :error_message, :backtrace], coder: JSON

  # 分类标签
  store :tags, accessors: [:channel_type, :webhook_type, :account_id, :inbox_id], coder: JSON
end
```

#### DLQ 运营流程
```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  重试耗尽    │────▶│  进入 DLQ     │────▶│  人工审核     │
└─────────────┘     └──────────────┘     └──────┬───────┘
                                                │
                        ┌───────────────────────┼───────────────────────┐
                        ▼                       ▼                       ▼
                  ┌──────────┐           ┌──────────┐           ┌──────────┐
                  │ 重新投递  │           │  修改参数  │           │  丢弃     │
                  │ (retry)  │           │ (edit)   │           │ (discard) │
                  └──────────┘           └──────────┘           └──────────┘
```

---

## 6. 重试方案实现建议

### 6.1 对现有代码的最小改动

#### 改动点 1：增强 Webhooks::Trigger
```ruby
# lib/webhooks/trigger.rb（修改建议）

class Webhooks::Trigger
  RETRYABLE_HTTP_STATUSES = [408, 429, 500, 502, 503, 504].freeze

  def execute
    perform_request
  rescue SafeFetch::FetchError => e
    # 网络错误：可重试
    raise RetryableError.new(status: nil, message: e.message)
  rescue SafeFetch::HttpError => e
    status = http_status(e)
    if RETRYABLE_HTTP_STATUSES.include?(status)
      raise RetryableError.new(status: status, message: e.message)
    else
      handle_failure(e)  # 不可重试，直接处理
    end
  rescue StandardError => e
    handle_failure(e)
  end
end
```

#### 改动点 2：增强 WebhookJob 重试
```ruby
# app/jobs/webhook_job.rb（修改建议）

class WebhookJob < ApplicationJob
  queue_as :medium

  RETRY_CONFIG = {
    network: { wait: :exponential, attempts: 5 },
    http_retryable: { wait: :exponential, attempts: 5 },
    http_429: { wait: :exponential_with_jitter, attempts: 8 }
  }.freeze

  retry_on Webhooks::Trigger::RetryableError do |job, error|
    if error.status == 429
      # 限流：检查 Retry-After 头
      retry_after = extract_retry_after(error.message)
      retry_job wait: retry_after if retry_after
    end
    # 其他情况由 ActiveJob 按策略重试
  end

  after_retry do |job, error|
    Metrics.increment('outbound_retry_count_total', tags: {
      job_class: job.class.name,
      error_class: error.class.name
    })
  end

  discard_on ActiveJob::DeserializationError do |job, error|
    DeadLetterQueue.push(job, error, reason: :deserialization)
  end

  def perform(url, payload, webhook_type = :account_webhook, secret: nil, delivery_id: nil)
    Webhooks::Trigger.execute(url, payload, webhook_type, secret: secret, delivery_id: delivery_id)
  rescue StandardError => e
    # 重试耗尽后
    DeadLetterQueue.push(self, e, reason: :retries_exhausted, tags: { webhook_type: webhook_type })
    Metrics.increment('outbound_dead_letter_total', tags: { webhook_type: webhook_type })
    raise
  end
end
```

#### 改动点 3：统一渠道服务异常处理
```ruby
# 建议新增 app/services/concerns/channel_retryable.rb

module ChannelRetryable
  extend ActiveSupport::Concern

  included do
    class_attribute :retry_config, default: {
      network_errors: [Net::OpenTimeout, Net::ReadTimeout, SocketError],
      max_attempts: 5,
      backoff_strategy: :exponential
    }
  end

  def with_retry(&block)
    attempt = 0
    begin
      yield
    rescue *retry_config[:network_errors] => e
      attempt += 1
      if attempt <= retry_config[:max_attempts]
        wait = calculate_backoff(attempt)
        Metrics.increment('outbound_retry_count_total', tags: {
          channel: channel_name,
          error: e.class.name
        })
        sleep(wait)
        retry
      else
        handle_permanent_failure(e)
      end
    rescue StandardError => e
      handle_permanent_failure(e)
    end
  end

  private

  def calculate_backoff(attempt)
    case retry_config[:backoff_strategy]
    when :exponential
      [1.second * (2**attempt), 30.seconds].min
    when :exponential_with_jitter
      base = 1.second * (2**attempt)
      [base * rand(0.5..1.5), 30.seconds].min
    else
      retry_config[:backoff_strategy].call(attempt)
    end
  end

  def handle_permanent_failure(error)
    Messages::StatusUpdateService.new(message, 'failed', error.message).perform
    ChatwootExceptionTracker.new(error, account: message.account).capture_exception
    Metrics.increment('outbound_channel_delivery_failed', tags: { channel: channel_name })
  end
end
```

---

## 7. 关键设计模式

### 7.1 模板方法模式
- `Base::SendOnChannelService` 定义渠道发送服务的骨架
- 子类实现 `channel_class` 和 `perform_reply`

### 7.2 事件驱动架构
- 使用 ActiveRecord callbacks + Wisper 事件系统
- 监听器解耦业务逻辑和通知逻辑

### 7.3 分布式锁
- 使用 Redis 实现互斥锁
- 防止并发消息导致的重复投递或时序问题

### 7.4 分层错误处理
- **传输层**：网络错误、超时
- **业务层**：HTTP 状态码、API 错误码
- **策略层**：区分可重试和不可重试错误

---

## 8. 总结

### 8.1 现有架构核心特点
1. **异步优先**：所有出站操作均通过 Sidekiq 异步处理
2. **三层回调**：账户级、API 渠道、Agent Bot 各自独立
3. **差异化重试**：Agent Bot 有重试，其他基本没有
4. **分布式锁**：关键路径使用 Redis 锁保证顺序性
5. **可观测性不足**：缺少统一的 metrics 和告警体系

### 8.2 现有重试策略矩阵

| 通知类型 | 作业队列 | 重试机制 | 最大重试 | 失败后处理 |
|----------|----------|----------|----------|------------|
| 渠道消息 (SendReplyJob) | high | 无自动重试 | - | 更新消息状态为 failed |
| Slack 通知 | medium | 锁获取失败重试 | 8 次 | 致命错误禁用集成 |
| 账户级 Webhook | medium | 无重试 | - | 仅记录日志 |
| API 渠道回调 | medium | 无重试 | - | 消息事件失败时标记 failed |
| Agent Bot Webhook | high | HTTP 429/500 重试 | 3 次 | 对话转 open，记录活动 |
| Hook 集成 | medium | 锁获取失败重试 | 3 次 | 捕获异常记录日志 |

### 8.3 建议方案核心改进

#### 错误分类
- **可重试**：网络错误（超时、连接失败）、HTTP 408/429/5xx
- **不可重试**：HTTP 400/401/403/404、认证错误、配置错误

#### 退避策略
- **网络/5xx**：指数退避（1s, 2s, 4s, 8s, 16s），最大 5 次
- **429 限流**：指数退避 + 抖动，最大 8 次，支持 `Retry-After` 头
- **锁竞争**：固定间隔 1 秒，最大 8 次

#### 幂等设计
- **发送端**：`X-Chatwoot-Delivery` + Redis 锁 + 成功标记缓存
- **接收端**：建议实现幂等表或缓存检查

#### 告警体系
- **P1 实时**：关键渠道成功率 < 95%、死信队列堆积
- **P2 小时级**：Webhook 失败率 > 10%、Bot 降级频繁
- **P3 日报**：成功率汇总、重试资源消耗分析

#### 死信队列
- 重试耗尽、致命错误、反序列化失败时进入
- 支持人工审核、重新投递、修改参数、丢弃操作

---

## 9. 附录

### 9.1 关键代码位置索引

| 组件 | 文件路径 |
|------|----------|
| Webhook 模型 | `app/models/webhook.rb` |
| API 渠道模型 | `app/models/channel/api.rb` |
| Agent Bot 模型 | `app/models/agent_bot.rb` |
| Webhook 监听器 | `app/listeners/webhook_listener.rb` |
| Agent Bot 监听器 | `app/listeners/agent_bot_listener.rb` |
| Webhook 作业 | `app/jobs/webhook_job.rb` |
| Agent Bot 作业 | `app/jobs/agent_bots/webhook_job.rb` |
| Webhook 触发器 | `lib/webhooks/trigger.rb` |
| SendReplyJob | `app/jobs/send_reply_job.rb` |
| 消息状态服务 | `app/services/messages/status_update_service.rb` |
| 账户设置 Schema | `app/models/concerns/account_settings_schema.rb` |
| Sidekiq 配置 | `config/initializers/sidekiq.rb` |

### 9.2 消息状态流转

```
sent (0) ──▶ delivered (1) ──▶ read (2)
   │
   └──▶ failed (3)  [可由 Webhook/渠道失败触发]
```

### 9.3 对话状态流转（Agent Bot 相关）

```
pending (机器人处理中)
    │
    ├──▶ bot 响应成功 ──▶ 继续等待或 resolved
    │
    └──▶ 重试耗尽 + keep_pending_on_bot_failure = false
          │
          └──▶ open (转人工处理) + 活动消息记录
```
