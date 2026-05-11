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
┌─────────────────────────────────────────────────────────────┐
│                     事件监听器层                              │
├────────────────────────────┬────────────────────────────────┤
│ WebhookListener            │ HookListener                   │
│ (外部 Webhook)             │ (集成 Hooks: Slack/Dialogflow  │
│                            │ /GoogleTranslate/LeadSquared)  │
└────────────────────────────┴────────────────────────────────┘
    ↓                                    ↓
WebhookJob                          HookJob
(medium queue)                      (medium queue)
    ↓                                    ↓
Webhooks::Trigger.execute        ┌─────────────────────┐
(HTTP POST 到外部 URL)            │ SendOnSlackJob     │
                                 │ (medium queue)     │
                                 │ + 分布式锁         │
                                 └─────────────────────┘
                                          ↓
                              Integrations::Slack::SendOnSlackService
```

---

## 2. 投递机制详解

### 2.1 Slack 通知投递机制

#### 触发流程
1. **事件监听**：`HookListener` 监听 `message.created` 和 `message.updated` 事件
   - 文件位置：`app/listeners/hook_listener.rb:58-71`
   - 支持的事件类型：
     - `message.created`：新消息创建
     - `message.updated`：消息更新（仅特定内容类型）

2. **作业入队**：`HookJob` 根据 `app_id` 路由到具体集成
   - 文件位置：`app/jobs/hook_job.rb:9-18`
   - Slack 集成处理：`app/jobs/hook_job.rb:25-45`
   - 带附件的消息延迟 2 秒发送，确保附件已上传

3. **分布式锁机制**：`SendOnSlackJob` 继承自 `MutexApplicationJob`
   - 文件位置：`app/jobs/send_on_slack_job.rb:1-10`
   - 锁键格式：`Redis::Alfred::SLACK_MESSAGE_MUTEX`（基于 conversation_id + reference_id）
   - 防止同一对话的重复投递

4. **核心发送服务**：`Integrations::Slack::SendOnSlackService`
   - 文件位置：`lib/integrations/slack/send_on_slack_service.rb:1-215`
   - 功能：
     - 消息内容格式化（去除 HTML、处理 @ 提及）
     - 文本消息发送（`chat_postMessage`）
     - 附件上传（`files_upload_v2`）
     - 线程消息管理（基于 `conversation.identifier`）

#### Slack 消息更新机制
- 作业：`UpdateSlackMessageJob`（`app/jobs/update_slack_message_job.rb`）
- 服务：`Integrations::Slack::UpdateSlackMessageService`
- 适用场景：交互式机器人消息的表单提交响应

---

### 2.2 外部 Webhook 投递机制

#### 触发流程
1. **事件监听**：`WebhookListener` 监听多种业务事件
   - 文件位置：`app/listeners/webhook_listener.rb:1-132`
   - 支持的事件类型（12 种）：
     ```ruby
     ALLOWED_WEBHOOK_EVENTS = %w[
       conversation_status_changed conversation_updated conversation_created
       contact_created contact_updated message_created message_updated
       webwidget_triggered inbox_created inbox_updated
       conversation_typing_on conversation_typing_off
     ].freeze
     ```
   - 文件位置：`app/models/webhook.rb:32-34`

2. **Webhook 类型**：
   - **Account Webhook**：账户级别的 Webhook
   - **Inbox Webhook**：收件箱级别的 Webhook
   - **API Inbox Webhook**：API 渠道的回调通知
   - **Agent Bot Webhook**：智能体机器人 Webhook

3. **作业入队**：
   - 普通 Webhook：`WebhookJob`（medium 队列）
   - Agent Bot：`AgentBots::WebhookJob`（high 队列）

4. **核心执行**：`Webhooks::Trigger.execute`
   - 文件位置：`lib/webhooks/trigger.rb:1-134`

#### Webhook 请求规范
- **HTTP 方法**：POST
- **Content-Type**：application/json
- **超时设置**：默认 5 秒，可通过 `WEBHOOK_TIMEOUT` 全局配置
- **请求头**：
  - `X-Chatwoot-Delivery`：唯一投递 ID（UUID）
  - `X-Chatwoot-Timestamp`：时间戳
  - `X-Chatwoot-Signature`：HMAC SHA256 签名（当配置 secret 时）

#### 签名验证机制
```ruby
headers['X-Chatwoot-Timestamp'] = ts
headers['X-Chatwoot-Signature'] = "sha256=#{OpenSSL::HMAC.hexdigest('SHA256', secret, "#{ts}.#{body}")}"
```
- 文件位置：`lib/webhooks/trigger.rb:58-61`

---

### 2.3 多渠道消息投递机制

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

#### 触发时机
- `Message` 模型的 `after_create_commit` 回调
- 文件位置：`app/models/message.rb:137, 324-401`
- 带附件的消息延迟 2 秒发送

---

## 3. 失败重试策略

### 3.1 队列优先级设计

| 优先级 | 队列名 | 使用场景 |
|--------|--------|----------|
| **high** | `:high` | `SendReplyJob`（渠道消息投递）、`AgentBots::WebhookJob`（机器人 Webhook）、`ActivityMessageJob` |
| **medium** | `:medium` | `SendOnSlackJob`、`WebhookJob`、`HookJob`、`UpdateSlackMessageJob` |
| **low** | `:low` | 投递状态回调、数据同步、后台清理任务 |

### 3.2 重试策略分类

#### A. 分布式锁获取失败重试
**适用场景**：Slack 通知、Hook 处理、多渠道 Webhook 事件

**实现机制**：
- 基类：`MutexApplicationJob`（`app/jobs/mutex_application_job.rb`）
- 异常类型：`LockAcquisitionError`
- 重试配置：

| 作业类 | 等待时间 | 最大重试次数 |
|--------|----------|--------------|
| `SendOnSlackJob` | 1 秒 | 8 次 |
| `UpdateSlackMessageJob` | 1 秒 | 8 次 |
| `HookJob` | 3 秒 | 3 次 |
| `FacebookEventsJob` | 1 秒 | 8 次 |
| `InstagramEventsJob` | 1 秒 | 8 次 |
| `WhatsappEventsJob` | 2 秒 | 20 次 |
| `TiktokEventsJob` | 2 秒 | 8 次 |

**设计意图**：
- 防止并发操作导致的数据竞争
- Slack 等渠道的消息顺序性保证
- 指数级或固定间隔重试，避免锁竞争加剧

#### B. Agent Bot Webhook 可重试错误
**适用场景**：智能体机器人 Webhook 调用

**实现机制**：
- 作业：`AgentBots::WebhookJob`（`app/jobs/agent_bots/webhook_job.rb`）
- 异常类型：`Webhooks::Trigger::RetryableError`
- 重试配置：等待 3 秒，最多 3 次

**可重试的 HTTP 状态码**：
```ruby
RETRYABLE_AGENT_BOT_STATUSES = [429, 500].freeze
```
- 文件位置：`lib/webhooks/trigger.rb:3`

**重试失败后的处理**：
1. 记录警告日志
2. 如果对话状态为 `pending` 且账户未配置 `keep_pending_on_bot_failure`，将对话转为 `open`
3. 创建活动消息记录机器人错误

#### C. Slack API 错误处理（非重试型）
**适用场景**：Slack 集成的致命错误

**错误类型及处理**：
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

**处理策略**：
- **不重试**：这些错误表示集成配置失效或权限问题
- **禁用 Hook**：自动禁用 Slack 集成
- **提示重新授权**：触发重新授权流程

#### D. 渠道消息投递失败处理
**适用场景**：WhatsApp、Twilio、Email 等渠道

**实现机制**：
- **WhatsApp**：更新消息 `status` 为 `failed`，记录 `external_error`
- **Twilio**：调用 `Messages::StatusUpdateService` 更新状态
- **Email**：捕获异常，调用 `Messages::StatusUpdateService`，并通过 `ChatwootExceptionTracker` 上报

**特点**：
- **无自动重试**：依赖人工或业务逻辑重试
- **状态可追溯**：消息状态字段记录失败信息

#### E. 普通 Webhook 失败处理
**适用场景**：Account/Inbox Webhook

**实现机制**：
- **无重试**：直接记录警告日志
- **事件特定处理**：仅对 `message_created` 和 `message_updated` 事件有额外处理
- **API Inbox Webhook**：更新消息状态为 `failed`

---

## 4. 关键设计模式

### 4.1 模板方法模式
- `Base::SendOnChannelService` 定义渠道发送服务的骨架
- 子类实现 `channel_class` 和 `perform_reply`

### 4.2 事件驱动架构
- 使用 ActiveRecord callbacks + Wisper 事件系统
- 监听器解耦业务逻辑和通知逻辑

### 4.3 分布式锁
- 使用 Redis 实现互斥锁
- 防止并发消息导致的重复投递或时序问题

### 4.4 分层错误处理
- **传输层**：网络错误、超时
- **业务层**：HTTP 状态码、API 错误码
- **策略层**：区分可重试和不可重试错误

---

## 5. 队列与基础设施

### Sidekiq 配置
- 文件位置：`config/initializers/sidekiq.rb`
- Redis 配置：`Redis::Config.app`
- 支持 Cron 定时任务（`sidekiq-cron`）
- 生产环境使用 JSON 日志格式

### 作业基类继承关系
```
ActiveJob::Base
    ↓
ApplicationJob (discard_on ActiveJob::DeserializationError)
    ↓
MutexApplicationJob (with_lock 机制)
    ↓
SendOnSlackJob, HookJob 等
```

---

## 6. 总结

### 核心特点
1. **异步优先**：所有出站操作均通过 Sidekiq 异步处理
2. **分层重试**：不同类型错误采用不同重试策略
3. **分布式锁**：关键路径使用 Redis 锁保证顺序性
4. **事件驱动**：业务逻辑与通知逻辑解耦
5. **可观测性**：丰富的日志记录和状态追踪

### 重试策略矩阵

| 通知类型 | 作业队列 | 重试机制 | 最大重试 | 失败后处理 |
|----------|----------|----------|----------|------------|
| 渠道消息 (SendReplyJob) | high | 无自动重试 | - | 更新消息状态为 failed |
| Slack 通知 | medium | 锁获取失败重试 | 8 次 | 致命错误禁用集成 |
| 普通 Webhook | medium | 无重试 | - | 记录日志 |
| Agent Bot Webhook | high | 429/500 重试 | 3 次 | 对话转 open，记录活动 |
| Hook 集成 (Slack/Dialogflow) | medium | 锁获取失败重试 | 3 次 | 捕获异常记录日志 |

### 建议改进点
1. **普通 Webhook 重试**：当前 Account/Inbox Webhook 无重试机制，可考虑引入指数退避重试
2. **死信队列**：重试耗尽后的消息处理机制
3. **监控告警**：Webhook 失败率监控和告警
4. **幂等性保证**：Webhook 接收方需要处理重复投递
