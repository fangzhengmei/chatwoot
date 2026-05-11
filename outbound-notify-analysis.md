# Chatwoot 出站通知与 Webhook 投递机制分析

## 1. 核心发现：失败处理的三重分类

Chatwoot 的出站通知失败处理存在**三种截然不同的模式**，这是理解整个系统的关键：

| 分类 | 定义 | 触发条件 | 最大次数 | 终态行为 |
|------|------|----------|----------|----------|
| **A. 业务层显式重试** | 通过 `retry_on` 明确定义的重试策略 | 特定异常类型 + 业务判断 | 代码定义的 attempts | 执行 exhaustion block 或重新抛出 |
| **B. 队列默认重试** | Sidekiq/ActiveJob 的默认重试机制 | 任何未被捕获的异常 | Sidekiq 默认 25 次 | 进入 Dead Set（死信队列） |
| **C. 吞错不重试** | 异常被 `rescue` 捕获且未重新抛出 | 代码中的 `rescue => e` 无 `raise` | 0 次 | 记录日志/更新状态，静默失败 |

---

## 2. 各通道失败处理分类详解

### 2.1 分类总览矩阵

| 通知类型 | 业务层显式重试 | 队列默认重试 | 吞错不重试 | 终态 |
|----------|----------------|--------------|------------|------|
| **普通 Webhook** (account/api) | ❌ 无 | ❌ 无 | ✅ 是 | 记录日志，消息事件更新为 failed |
| **Agent Bot Webhook** | ✅ 部分 | ❌ 无 | ⚠️ 部分 | 重试耗尽后对话转 open（可配置） |
| **Slack 发送** | ⚠️ 仅锁竞争 | ❌ 无 | ✅ 是 | 致命错误禁用集成，其他静默失败 |
| **多渠道消息** (SendReplyJob) | ❌ 无 | ⚠️ 部分 | ✅ 多数是 | 更新消息 status = failed |

---

### 2.2 分类 A：业务层显式重试（代码定义）

#### 定义
通过 `retry_on` 宏明确定义的重试策略，重试逻辑由开发者控制。

#### 完整清单

| 作业类 | 重试异常 | 等待策略 | 最大次数 | 触发条件 | 终态处理 | 文件位置 |
|--------|----------|----------|----------|----------|----------|----------|
| `AgentBots::WebhookJob` | `Webhooks::Trigger::RetryableError` | 固定 3s | 3 | HTTP 429/500（仅 Agent Bot） | 执行 `handle_failure` | `app/jobs/agent_bots/webhook_job.rb:3-8` |
| `SendOnSlackJob` | `LockAcquisitionError` | 固定 1s | 8 | Redis 锁获取失败 | 重新抛出（无 exhaustion block） | `app/jobs/send_on_slack_job.rb:3` |
| `UpdateSlackMessageJob` | `LockAcquisitionError` | 固定 1s | 8 | Redis 锁获取失败 | 重新抛出 | `app/jobs/update_slack_message_job.rb:3` |
| `HookJob` | `LockAcquisitionError` | 固定 3s | 3 | Redis 锁获取失败 | 重新抛出 | `app/jobs/hook_job.rb:2` |
| `Webhooks::WhatsappEventsJob` | `LockAcquisitionError` | 固定 2s | 20 | Redis 锁获取失败 | 重新抛出 | `app/jobs/webhooks/whatsapp_events_job.rb:6` |
| `Webhooks::FacebookEventsJob` | `LockAcquisitionError` | 固定 1s | 8 | Redis 锁获取失败 | 重新抛出 | `app/jobs/webhooks/facebook_events_job.rb:3` |
| `Webhooks::InstagramEventsJob` | `LockAcquisitionError` | 固定 1s | 8 | Redis 锁获取失败 | 重新抛出 | `app/jobs/webhooks/instagram_events_job.rb:3` |
| `Webhooks::TiktokEventsJob` | `LockAcquisitionError` | 固定 2s | 8 | Redis 锁获取失败 | 重新抛出 | `app/jobs/webhooks/tiktok_events_job.rb:4` |

#### Agent Bot 重试的关键限制
```ruby
# lib/webhooks/trigger.rb:3
RETRYABLE_AGENT_BOT_STATUSES = [429, 500].freeze

# lib/webhooks/trigger.rb:125-127
def retryable_agent_bot_error?(error)
  @webhook_type == :agent_bot_webhook && RETRYABLE_AGENT_BOT_STATUSES.include?(http_status(error))
end

# lib/webhooks/trigger.rb:129-133
def http_status(error)
  return unless error.is_a?(SafeFetch::HttpError)  # ← 关键！只有 HttpError 才提取状态码
  error.message.to_s[/\A(\d{3})\b/, 1]&.to_i
end
```

**触发条件详解**：
1. ✅ **HTTP 429** (Too Many Requests) → 触发重试
2. ✅ **HTTP 500** (Internal Server Error) → 触发重试
3. ❌ **HTTP 502** (Bad Gateway) → **不触发**
4. ❌ **HTTP 503** (Service Unavailable) → **不触发**
5. ❌ **HTTP 504** (Gateway Timeout) → **不触发**
6. ❌ **网络超时** (`Net::OpenTimeout`, `Net::ReadTimeout`) → **不触发**（因为是 `SafeFetch::FetchError`，不是 `HttpError`）
7. ❌ **连接错误** (`SocketError`) → **不触发**
8. ❌ **SSL 错误** (`OpenSSL::SSL::SSLError`) → **不触发**

---

### 2.3 分类 B：队列默认重试（Sidekiq 默认）

#### 定义
异常未被任何 `rescue` 捕获，向上抛出到 Sidekiq，触发 Sidekiq 的默认重试机制。

#### Sidekiq 默认行为
- **最大重试次数**：25 次
- **退避策略**：指数退避 `(count^4) + 15` 秒
  - 第 1 次：16 秒
  - 第 2 次：31 秒
  - 第 3 次：96 秒
  - ...
  - 第 25 次：约 15 天
- **终态**：25 次后进入 `Sidekiq::DeadSet`（死信队列，保留 6 个月）

#### 可能触发队列默认重试的场景

在 Chatwoot 当前代码中，**几乎没有通道会触发队列默认重试**，因为：

1. **WebhookJob**：`Webhooks::Trigger.execute` 捕获所有 `StandardError`，不重新抛出
2. **HookJob**：`perform` 方法末尾有 `rescue StandardError => e; Rails.logger.error e; end`
3. **SendReplyJob**：各 `SendOn*Service` 自己有 `rescue`

**唯一可能的例外**：
- 某些 `SendOn*Service` 的 `rescue` 可能不完整，导致特定异常漏网

#### 代码证据
```ruby
# app/jobs/hook_job.rb:19-21
rescue StandardError => e
  Rails.logger.error e
  # ← 没有重新抛出！异常被吞掉
end
```

```ruby
# lib/webhooks/trigger.rb:26-32
def execute
  perform_request
rescue StandardError => e
  raise RetryableError.new(...) if retryable_agent_bot_error?(e)
  handle_failure(e)  # ← 其他情况不重新抛出！
end
```

---

### 2.4 分类 C：吞错不重试（静默失败）

#### 定义
异常被 `rescue` 捕获，记录日志或更新状态，但**不重新抛出异常**，作业被标记为成功。

#### 完整清单

| 通知类型 | 捕获位置 | 捕获的异常 | 终态处理 | 文件位置 |
|----------|----------|------------|----------|----------|
| **普通 Webhook** (account) | `Webhooks::Trigger#execute` | 所有 `StandardError` | 记录 warn 日志 | `lib/webhooks/trigger.rb:28-32` |
| **普通 Webhook** (api_inbox) | `Webhooks::Trigger#execute` | 所有 `StandardError` | 记录日志 + 消息 status=failed | `lib/webhooks/trigger.rb:72-74` |
| **Slack 发送** (致命错误) | `SendOnSlackService#send_message` | 特定 Slack API 错误 | 禁用集成 + 记录 error 日志 | `lib/integrations/slack/send_on_slack_service.rb:105-111` |
| **Slack 附件上传** | `SendOnSlackService#upload_files` | `Slack::Web::Api::Errors::SlackError` | 记录 error 日志 | `lib/integrations/slack/send_on_slack_service.rb:136-138` |
| **HookJob (所有集成)** | `HookJob#perform` | 所有 `StandardError` | 记录 error 日志 | `app/jobs/hook_job.rb:19-21` |
| **Twilio 消息发送** | `Twilio::SendOnTwilioService#perform_reply` | `Twilio::REST::TwilioError/RestError` | 消息 status=failed | `app/services/twilio/send_on_twilio_service.rb:35-37` |
| **Tiktok 消息发送** | `Tiktok::SendOnTiktokService#perform` | 所有 `StandardError` | 消息 status=failed | `app/services/tiktok/send_on_tiktok_service.rb:17-19` |
| **Facebook 消息发送** | `Facebook::SendOnFacebookService` | 多种异常 | 消息 status=failed | `app/services/facebook/send_on_facebook_service.rb` |
| **Email 发送** | `Email::SendOnEmailService#perform_reply` | 所有 `StandardError` | 消息 status=failed + Sentry | `app/services/email/send_on_email_service.rb:14-17` |

#### 吞错代码示例
```ruby
# app/jobs/hook_job.rb:19-21
# 这个 rescue 会吞掉所有异常，包括 Slack 发送失败！
rescue StandardError => e
  Rails.logger.error e
  # 没有 raise，异常被吞掉
end
```

```ruby
# lib/webhooks/trigger.rb:28-32
# 普通 Webhook 的所有异常都在这里被吞掉
rescue StandardError => e
  raise RetryableError.new(...) if retryable_agent_bot_error?(e)  # 仅 Agent Bot 的 429/500
  handle_failure(e)  # 其他情况：记录日志，不重新抛出
end
```

---

## 3. 三个关键通道的深度对比

### 3.1 普通 Webhook vs Agent Bot vs Slack：网络超时与 5xx 差异

#### 对比矩阵

| 错误类型 | 普通 Webhook | Agent Bot Webhook | Slack 发送 |
|----------|-------------|-------------------|------------|
| **网络超时** (OpenTimeout) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **读取超时** (ReadTimeout) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **连接错误** (SocketError) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **SSL 错误** | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **HTTP 408** (Timeout) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **HTTP 429** (限流) | 吞错不重试 ❌ | 业务层重试 ✅ (3次) | 吞错不重试 ❌ |
| **HTTP 500** (服务器错误) | 吞错不重试 ❌ | 业务层重试 ✅ (3次) | 吞错不重试 ❌ |
| **HTTP 502** (Bad Gateway) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **HTTP 503** (Service Unavailable) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |
| **HTTP 504** (Gateway Timeout) | 吞错不重试 ❌ | 吞错不重试 ❌ | 吞错不重试 ❌ |

#### 原因分析

**普通 Webhook**：
```ruby
# lib/webhooks/trigger.rb:28-32
rescue StandardError => e
  raise RetryableError.new(...) if retryable_agent_bot_error?(e)
  # retryable_agent_bot_error? 检查 @webhook_type == :agent_bot_webhook
  # 普通 Webhook 永远不满足这个条件！
  handle_failure(e)  # → 直接吞错
end
```

**Agent Bot Webhook**：
```ruby
# lib/webhooks/trigger.rb:125-127
def retryable_agent_bot_error?(error)
  @webhook_type == :agent_bot_webhook && 
    RETRYABLE_AGENT_BOT_STATUSES.include?(http_status(error))
    # RETRYABLE_AGENT_BOT_STATUSES = [429, 500].freeze
    # 注意：502/503/504 不在列表中！
end

# lib/webhooks/trigger.rb:129-133
def http_status(error)
  return unless error.is_a?(SafeFetch::HttpError)
  # 网络超时是 SafeFetch::FetchError，不是 HttpError！
  # 所以网络超时永远不会触发重试！
  error.message.to_s[/\A(\d{3})\b/, 1]&.to_i
end
```

**Slack 发送**：
```ruby
# app/jobs/hook_job.rb:19-21
# HookJob 捕获所有异常，所以 Slack 发送的任何异常都会被吞掉
rescue StandardError => e
  Rails.logger.error e
end

# lib/integrations/slack/send_on_slack_service.rb:105-111
# SendOnSlackService 只捕获特定的致命错误
rescue Slack::Web::Api::Errors::IsArchived, ... => e
  hook.disable
  # 其他 Slack 错误（如 429、5xx）会向上抛出
  # 但最终会被 HookJob 的 rescue 捕获
end
```

#### 结论：当前系统的"伪重试"问题

| 表象 | 实际 |
|------|------|
| Agent Bot 有重试 | 仅对 HTTP 429/500，网络超时和其他 5xx 都不重试 |
| 普通 Webhook 应该重试 | 实际上完全不重试，静默失败 |
| Slack 发送应该重试 | HookJob 的 rescue 吞掉所有异常，完全不重试 |
| 队列默认重试兜底 | 由于到处都是 rescue，几乎不会触发 |

**结果**：当前系统中，网络抖动、网关超时等临时性错误导致的通知失败，**几乎没有任何重试机制**！

---

## 4. 避免双重重试风暴的治理方案

### 4.1 问题根源：双重重试风险

#### 什么是双重重试？

```
异常发生
    ↓
┌─────────────────────────────────────────────────────────┐
│  业务层重试（retry_on）                                   │
│  - 等待 3s，重试 3 次                                     │
│  - 第 3 次失败 → exhaustion block                         │
│  - 如果 exhaustion block 没有吸收异常 → 异常继续向上抛出    │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│  Sidekiq 默认重试                                         │
│  - 等待 16s, 31s, 96s...                                  │
│  - 重试 25 次（约 15 天）                                  │
│  - 最终进入 Dead Set                                      │
└─────────────────────────────────────────────────────────┘

问题：业务层重试了 3 次，Sidekiq 又重试了 25 次
     总共 28 次重试！接收方可能被打爆！
```

#### Chatwoot 中的实际风险

**当前代码中的安全处理**（好的例子）：
```ruby
# enterprise/app/jobs/captain/documents/perform_sync_job.rb:11-13
# 注释明确说明：吸收最终异常，避免 Sidekiq 叠加重试
retry_on StandardError, wait: 5.seconds, attempts: 3 do |job, error|
  # exhaustion block 吸收异常
  ChatwootExceptionTracker.new(error, account: document.account).capture_exception
  job.send(:log_sync_outcome, document, result: :unexpected_retry_exhausted, ...)
  # ← 没有重新抛出异常！
end
```

**当前代码中的风险处理**（Agent Bot Webhook）：
```ruby
# app/jobs/agent_bots/webhook_job.rb:3-8
retry_on Webhooks::Trigger::RetryableError, wait: 3.seconds, attempts: 3 do |job, error|
  # exhaustion block 调用 handle_failure
  Webhooks::Trigger.new(...).handle_failure(error)
  # handle_failure 内部没有 raise
  # 所以异常被吸收了 ✓
end
```

---

### 4.2 治理方案：三层重试协作模型

#### 方案原则

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        重试分层治理原则                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  Layer 1: 业务层重试（retry_on）                                          │
│  ├── ✅ 保留：针对已知可重试异常（网络错误、429、5xx）                       │
│  ├── ✅ 明确指定：异常类型、等待策略、最大次数                               │
│  └── ✅ 必须有 exhaustion block 吸收异常（避免 Layer 2 介入）               │
├─────────────────────────────────────────────────────────────────────────┤
│  Layer 2: 队列默认重试（Sidekiq）                                         │
│  ├── 🚫 关闭：对于有业务层重试的作业                                        │
│  ├── ⚠️ 保留：仅作为完全意外的兜底                                          │
│  └── ⚠️ 配置：减少默认重试次数（如 3 次而非 25 次）                          │
├─────────────────────────────────────────────────────────────────────────┤
│  Layer 3: 吞错不重试（rescue 无 raise）                                    │
│  ├── 🚫 避免：不应该用 rescue 吞掉所有异常                                   │
│  ├── ✅ 替代：区分可重试/不可重试，可重试的抛出，不可重试的记录状态             │
│  └── ✅ 例外：已知的致命错误（认证失败、配置错误）                            │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 具体方案：按作业类型配置

| 作业类型 | Layer 1 业务层重试 | Layer 2 队列默认重试 | Layer 3 吞错 | 幂等键 | 告警 |
|----------|-------------------|---------------------|-------------|--------|------|
| **AgentBot WebhookJob** | ✅ 保留并增强 | 🚫 关闭（sidekiq_options retry: 0） | 🚫 移除 | `X-Chatwoot-Delivery` | 重试耗尽 → P2 |
| **普通 WebhookJob** | ✅ 新增 | 🚫 关闭 | 🚫 移除 | `X-Chatwoot-Delivery` | 失败率 > 10% → P2 |
| **SendOnSlackJob** | ✅ 新增 | 🚫 关闭 | 🚫 从 HookJob 移除 | `slack:{conversation_id}:{reference_id}` | 连续失败 → P1 |
| **SendReplyJob** | ✅ 新增（各渠道） | 🚫 关闭 | 🚫 仅致命错误 | `channel:{type}:{message_id}` | 成功率 < 95% → P1 |
| **HookJob** | ⚠️ 仅保留锁竞争 | 🚫 关闭 | 🚫 移除 rescue | - | 子作业失败透传 |

---

### 4.3 方案一：Agent Bot WebhookJob 增强

#### 当前问题
- 仅重试 HTTP 429/500
- 网络超时、502/503/504 都不重试
- 但这些才是最常见的临时性错误！

#### 改进方案

```ruby
# 建议修改 app/jobs/agent_bots/webhook_job.rb

class AgentBots::WebhookJob < WebhookJob
  queue_as :high
  
  # 关闭 Sidekiq 默认重试，避免双重重试
  sidekiq_options retry: 0
  
  # 可重试的 HTTP 状态码
  RETRYABLE_HTTP_STATUSES = [408, 429, 500, 502, 503, 504].freeze
  
  # 网络错误类型
  NETWORK_ERRORS = [
    Net::OpenTimeout,
    Net::ReadTimeout,
    SocketError,
    OpenSSL::SSL::SSLError,
    SafeFetch::FetchError
  ].freeze
  
  # HTTP 429：限流，指数退避 + 抖动，最多 8 次
  retry_on Webhooks::Trigger::RateLimitError, 
           wait: ->(executions) { [2.seconds * (2**executions) * rand(0.5..1.5), 60.seconds].min },
           attempts: 8 do |job, error|
    # 限流：记录 metric，继续后续处理
    Metrics.increment('outbound_webhook_rate_limited', tags: { bot_id: job.bot_id })
    Webhooks::Trigger.new(...).handle_failure(error)
  end
  
  # 网络错误和其他 5xx：指数退避，最多 5 次
  retry_on Webhooks::Trigger::TransientError,
           wait: ->(executions) { [1.second * (2**executions), 30.seconds].min },
           attempts: 5 do |job, error|
    # 重试耗尽：记录告警
    Metrics.increment('outbound_webhook_retry_exhausted', tags: { bot_id: job.bot_id })
    Alerting.trigger(:webhook_retry_exhausted, job_id: job.job_id, error: error.message)
    Webhooks::Trigger.new(...).handle_failure(error)
  end
  
  # 不可重试的错误：直接丢弃
  discard_on Webhooks::Trigger::PermanentError do |job, error|
    Metrics.increment('outbound_webhook_permanent_failure', tags: { bot_id: job.bot_id })
    Rails.logger.error "Agent Bot permanent failure: #{error.message}"
  end
  
  def perform(url, payload, webhook_type = :agent_bot_webhook, secret: nil, delivery_id: nil)
    super(url, payload, webhook_type, secret: secret, delivery_id: delivery_id)
  rescue Webhooks::Trigger::RetryableError => e
    Rails.logger.warn("[AgentBots::WebhookJob] attempt #{executions} failed")
    raise  # 重新抛出让 retry_on 处理
  end
end
```

```ruby
# 建议修改 lib/webhooks/trigger.rb

class Webhooks::Trigger
  # 新增异常分类
  class RateLimitError < StandardError; end      # 429
  class TransientError < StandardError; end      # 网络错误、5xx
  class PermanentError < StandardError; end      # 401、403、404
  
  def execute
    perform_request
  rescue StandardError => e
    error_type = classify_error(e)
    
    case error_type
    when :rate_limit
      raise RateLimitError, e.message
    when :transient
      raise TransientError, e.message
    when :permanent
      raise PermanentError, e.message
    end
  end
  
  private
  
  def classify_error(error)
    # Agent Bot 的限流：业务层重试
    if @webhook_type == :agent_bot_webhook
      status = http_status(error)
      
      if status == 429
        :rate_limit
      elsif [408, 500, 502, 503, 504].include?(status)
        :transient
      elsif NETWORK_ERRORS.any? { |e| error.is_a?(e) }
        :transient
      elsif [401, 403, 404].include?(status)
        :permanent
      else
        # 其他错误：记录日志，不重试
        handle_failure(error)
        nil
      end
    else
      # 普通 Webhook：当前方案是吞错不重试
      # 改进方案：也应该有重试逻辑
      handle_failure(error)
      nil
    end
  end
end
```

---

### 4.4 方案二：普通 WebhookJob 新增重试

#### 当前问题
- 完全不重试，网络抖动导致的通知直接丢失
- 业务系统需要自行补偿

#### 改进方案

```ruby
# 建议修改 app/jobs/webhook_job.rb

class WebhookJob < ApplicationJob
  queue_as :medium
  
  # 关闭 Sidekiq 默认重试
  sidekiq_options retry: 0
  
  # 指数退避：1s, 2s, 4s, 8s, 16s，最多 5 次
  retry_on Webhooks::Trigger::TransientError,
           wait: ->(executions) { [1.second * (2**executions), 30.seconds].min },
           attempts: 5 do |job, error|
    url, payload, webhook_type = job.arguments
    Metrics.increment('outbound_webhook_retry_exhausted', tags: { webhook_type: webhook_type })
    
    # 重试耗尽：记录告警
    if webhook_type == :account_webhook
      Alerting.trigger(:account_webhook_failure, url: url, error: error.message)
    end
    
    Webhooks::Trigger.new(...).handle_failure(error)
  end
  
  # 不可重试的错误
  discard_on Webhooks::Trigger::PermanentError do |job, error|
    Metrics.increment('outbound_webhook_permanent_failure')
    Rails.logger.error "Webhook permanent failure: #{error.message}"
  end
  
  def perform(url, payload, webhook_type = :account_webhook, secret: nil, delivery_id: nil)
    Webhooks::Trigger.execute(url, payload, webhook_type, secret: secret, delivery_id: delivery_id)
  end
end
```

```ruby
# Webhooks::Trigger 需要修改以支持普通 Webhook 的重试分类
def classify_error(error)
  # 所有 Webhook 类型都应该区分可重试/不可重试
  status = http_status(error)
  
  if [408, 429, 500, 502, 503, 504].include?(status)
    :transient
  elsif NETWORK_ERRORS.any? { |e| error.is_a?(e) }
    :transient
  elsif [401, 403, 404].include?(status)
    :permanent
  else
    :transient  # 未知错误视为临时性错误，给一次重试机会
  end
end
```

---

### 4.5 方案三：Slack 发送修复

#### 当前问题
- HookJob 的 `rescue StandardError => e` 吞掉所有异常
- Slack 发送失败（如网络超时、429、5xx）完全不重试
- 用户无法知道 Slack 通知失败了

#### 改进方案

```ruby
# 建议修改 app/jobs/hook_job.rb

class HookJob < MutexApplicationJob
  retry_on LockAcquisitionError, wait: 3.seconds, attempts: 3
  
  queue_as :medium
  
  # 关闭 Sidekiq 默认重试
  sidekiq_options retry: 0
  
  def perform(hook, event_name, event_data = {})
    return if hook.disabled?

    case hook.app_id
    when 'slack'
      process_slack_integration(hook, event_name, event_data)
    when 'dialogflow'
      process_dialogflow_integration(hook, event_name, event_data)
    when 'google_translate'
      google_translate_integration(hook, event_name, event_data)
    when 'leadsquared'
      process_leadsquared_integration_with_lock(hook, event_name, event_data)
    end
  # 移除 rescue StandardError！让异常透传到子作业
  end
  
  # ...
end
```

```ruby
# 建议修改 app/jobs/send_on_slack_job.rb

class SendOnSlackJob < MutexApplicationJob
  queue_as :medium
  
  # 关闭 Sidekiq 默认重试
  sidekiq_options retry: 0
  
  # 锁竞争：固定间隔重试
  retry_on LockAcquisitionError, wait: 1.second, attempts: 8
  
  # Slack API 限流：指数退避 + 抖动
  retry_on Slack::Web::Api::Errors::TooManyRequestsError,
           wait: ->(executions) { [2.seconds * (2**executions) * rand(0.5..1.5), 120.seconds].min },
           attempts: 10 do |job, error|
    Metrics.increment('outbound_slack_rate_limited')
    Alerting.trigger(:slack_integration_failing, hook_id: job.hook_id)
  end
  
  # 网络错误和 5xx：指数退避
  retry_on Slack::Web::Api::Errors::ServiceUnavailable,
           Slack::Web::Api::Errors::InternalError,
           Faraday::TimeoutError,
           Faraday::ConnectionFailed,
           wait: ->(executions) { [1.second * (2**executions), 30.seconds].min },
           attempts: 5 do |job, error|
    Metrics.increment('outbound_slack_retry_exhausted')
    Alerting.trigger(:slack_integration_failing, hook_id: job.hook_id)
  end
  
  # 致命错误：直接禁用集成
  discard_on Slack::Web::Api::Errors::IsArchived,
             Slack::Web::Api::Errors::AccountInactive,
             Slack::Web::Api::Errors::MissingScope,
             Slack::Web::Api::Errors::InvalidAuth,
             Slack::Web::Api::Errors::ChannelNotFound,
             Slack::Web::Api::Errors::NotInChannel do |job, error|
    message, hook = job.arguments
    hook.disable
    hook.prompt_reauthorization!
    Rails.logger.error "Slack hook disabled: #{error.message}"
    Alerting.trigger(:slack_integration_disabled, hook_id: hook.id)
  end
  
  def perform(message, hook)
    key = format(::Redis::Alfred::SLACK_MESSAGE_MUTEX, conversation_id: message.conversation_id, reference_id: hook.reference_id)
    with_lock(key) do
      Integrations::Slack::SendOnSlackService.new(message: message, hook: hook).perform
    end
  end
end
```

---

### 4.6 方案四：幂等键设计（配合重试）

#### 原则
重试必须配合幂等，否则可能产生副作用。

#### 幂等键规范

| 通知类型 | 幂等键格式 | 存储位置 | TTL |
|----------|-----------|----------|-----|
| Webhook (所有类型) | `webhook:{delivery_id}` | 请求头 `X-Chatwoot-Delivery` | 24h |
| Slack 发送 | `slack:{conversation_id}:{hook_reference_id}` | Redis 锁 + 成功标记 | 24h |
| 渠道消息 | `channel:{channel_type}:{message_id}` | Redis 成功标记 | 24h |
| Agent Bot | `bot:{bot_id}:{event}:{entity_id}` | Redis 成功标记 | 24h |

#### 发送端幂等保障
```ruby
# 建议新增 lib/outbound/idempotent_sender.rb

module Outbound
  module IdempotentSender
    def with_idempotency_guarantee(idempotency_key, &block)
      # 1. 检查是否已成功投递
      success_key = "outbound:success:#{idempotency_key}"
      if Rails.cache.exist?(success_key)
        Rails.logger.info "[Idempotency] Skipping duplicate: #{idempotency_key}"
        return
      end
      
      # 2. 获取发送锁（防止并发重复发送）
      lock_key = "outbound:lock:#{idempotency_key}"
      lock_manager = Redis::LockManager.new
      
      if lock_manager.lock(lock_key, 5.minutes)
        begin
          yield
          # 3. 成功后标记
          Rails.cache.write(success_key, true, expires_in: 24.hours)
        ensure
          lock_manager.unlock(lock_key)
        end
      else
        Rails.logger.info "[Idempotency] Lock held, skipping: #{idempotency_key}"
      end
    end
  end
end
```

#### 接收端幂等处理（文档建议）
```
接收方实现步骤：
1. 读取请求头 X-Chatwoot-Delivery
2. 检查本地幂等表/缓存：已处理？
3. 已处理 → 返回 200 OK，不执行业务逻辑
4. 未处理 → 执行业务逻辑 + 记录幂等 ID
5. 幂等记录 TTL：建议 24 小时

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

---

### 4.7 方案五：告警体系

#### 告警指标

| 指标名 | 类型 | 阈值 | 级别 | 标签 |
|--------|------|------|------|------|
| `outbound_webhook_retry_exhausted_total` | Counter | > 10/5min | P2 | webhook_type, account_id |
| `outbound_webhook_failure_rate` | Gauge | > 10% | P2 | webhook_type, account_id |
| `outbound_slack_retry_exhausted_total` | Counter | > 5/10min | P1 | hook_id, account_id |
| `outbound_channel_delivery_success_rate` | Gauge | < 95% | P1 | channel_type, account_id |
| `outbound_bot_downgrade_total` | Counter | > 10/hour | P2 | bot_id, account_id |
| `outbound_double_retry_detected` | Counter | > 0 | P1 | job_type |

#### 告警规则（Prometheus）
```yaml
groups:
  - name: outbound_notifications
    rules:
      - alert: SlackIntegrationFailing
        expr: increase(outbound_slack_retry_exhausted_total[10m]) > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Slack 集成连续失败"
          description: "Hook {{ $labels.hook_id }} 重试耗尽 {{ $value }} 次"

      - alert: WebhookHighFailureRate
        expr: outbound_webhook_failure_rate > 0.10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Webhook 失败率超过 10%"
          description: "类型 {{ $labels.webhook_type }} 失败率: {{ $value | humanizePercentage }}"

      - alert: ChannelDeliveryDegraded
        expr: outbound_channel_delivery_success_rate < 0.95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "渠道投递成功率低于 95%"
          description: "渠道 {{ $labels.channel_type }} 成功率: {{ $value | humanizePercentage }}"

      - alert: DoubleRetryDetected
        expr: increase(outbound_double_retry_detected_total[5m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "检测到双重重试！"
          description: "作业 {{ $labels.job_type }} 存在业务层+队列层双重重试"
```

---

### 4.8 方案六：双重重试检测机制

#### 问题
如何确保业务层重试耗尽后，Sidekiq 不会再次重试？

#### 检测方案
```ruby
# 建议新增 config/initializers/sidekiq_retry_monitor.rb

# 监控是否有作业既配置了 retry_on 又没关闭 Sidekiq 默认重试
Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    chain.add DoubleRetryDetectionMiddleware
  end
end

class DoubleRetryDetectionMiddleware
  def call(worker, job, queue)
    # 检测条件：
    # 1. 作业有 retry_on 配置（ActiveJob）
    # 2. Sidekiq retry 没有被关闭（retry != 0）
    
    if job['class'].safe_constantize&.respond_to?(:retry_on_callbacks) &&
       job['class'].safe_constantize&.retry_on_callbacks&.any? &&
       job['retry'] != 0
      
      Rails.logger.warn "[DoubleRetry] Detected potential double retry: #{job['class']}"
      Metrics.increment('outbound_double_retry_detected', tags: { job_type: job['class'] })
      Alerting.trigger(:double_retry_risk, job_class: job['class'])
    end
    
    yield
  end
end
```

---

## 5. 治理方案总结

### 5.1 实施优先级

| 优先级 | 方案 | 改动量 | 风险 | 收益 |
|--------|------|--------|------|------|
| P0 | 移除 HookJob 的 `rescue StandardError` | 小 | 中 | Slack 发送失败可被观察 |
| P0 | Agent Bot 重试增强（网络错误 + 5xx） | 中 | 低 | 修复最关键的重试缺口 |
| P1 | 普通 Webhook 新增重试 | 中 | 低 | 减少通知丢失 |
| P1 | 关键作业配置 `sidekiq_options retry: 0` | 小 | 低 | 避免双重重试 |
| P1 | 告警体系建设 | 中 | 低 | 可观测性提升 |
| P2 | 幂等键完善 | 大 | 中 | 安全重试的基础 |
| P2 | 双重重试检测中间件 | 小 | 低 | 预防未来问题 |

### 5.2 重试配置规范（实施后）

```ruby
# 所有出站作业的标准配置模板

class OutboundJobTemplate < ApplicationJob
  # 1. 明确关闭 Sidekiq 默认重试
  sidekiq_options retry: 0
  
  # 2. 按异常类型分层重试
  retry_on TransientError, 
           wait: ->(n) { [1.second * (2**n), 30.seconds].min },
           attempts: 5 do |job, error|
    # 3. exhaustion block 必须吸收异常
    Metrics.increment('retry_exhausted')
    Alerting.trigger(:job_failing, job_id: job.job_id)
    handle_permanent_failure(error)
    # 没有 raise！
  end
  
  retry_on RateLimitError,
           wait: ->(n) { [2.seconds * (2**n) * rand(0.5..1.5), 120.seconds].min },
           attempts: 8 do |job, error|
    Metrics.increment('rate_limited')
    handle_permanent_failure(error)
  end
  
  # 4. 不可重试的错误直接丢弃
  discard_on PermanentError do |job, error|
    Metrics.increment('permanent_failure')
    disable_integration if auth_error?(error)
  end
  
  def perform(*args)
    # 5. 幂等保障
    with_idempotency_guarantee(idempotency_key(args)) do
      do_work(*args)
    end
  end
end
```

---

## 6. 附录

### 6.1 关键代码位置索引

| 组件 | 文件路径 |
|------|----------|
| Webhook 触发器（核心异常处理） | `lib/webhooks/trigger.rb` |
| Agent Bot Webhook 作业 | `app/jobs/agent_bots/webhook_job.rb` |
| 普通 Webhook 作业 | `app/jobs/webhook_job.rb` |
| Hook 作业（Slack 入口） | `app/jobs/hook_job.rb` |
| Slack 发送作业 | `app/jobs/send_on_slack_job.rb` |
| Slack 发送服务 | `lib/integrations/slack/send_on_slack_service.rb` |
| 多渠道发送入口 | `app/jobs/send_reply_job.rb` |
| 应用作业基类 | `app/jobs/application_job.rb` |
| 互斥锁作业基类 | `app/jobs/mutex_application_job.rb` |
| 双重重试示例（好的实践） | `enterprise/app/jobs/captain/documents/perform_sync_job.rb` |

### 6.2 SafeFetch 异常分类

```ruby
# lib/safe_fetch.rb:17-24
class Error < StandardError; end
class InvalidUrlError < Error; end
class UnsafeUrlError < Error; end
class FetchError < Error; end       # ← 网络超时、连接错误、SSL 错误
class HttpError < Error; end        # ← HTTP 非 2xx 响应
class FileTooLargeError < Error; end
class UnsupportedContentTypeError < Error; end
class UnsupportedMethodError < Error; end
```

### 6.3 消息状态流转

```
sent (0) ──▶ delivered (1) ──▶ read (2)
   │
   └──▶ failed (3)  [可由 Webhook 失败/渠道失败触发]
```

### 6.4 当前重试策略全景图（修订前）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        出站通知重试全景（修订前）                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Agent Bot Webhook                                                        │
│  ├── retry_on: HTTP 429/500 → 3 次（固定 3s）                             │
│  ├── 网络超时/502/503/504 → 吞错不重试 ❌                                 │
│  └── exhaustion block: 对话转 open（可配置）                               │
│                                                                          │
│  普通 Webhook (account/api)                                               │
│  ├── retry_on: 无 ❌                                                      │
│  ├── 所有异常 → Webhooks::Trigger#handle_failure → 吞掉                    │
│  └── 终态: 记录日志（消息事件会更新 status=failed）                         │
│                                                                          │
│  Slack 发送                                                                │
│  ├── retry_on: LockAcquisitionError → 8 次（锁竞争）                       │
│  ├── 其他异常 → HookJob#rescue StandardError → 吞掉 ❌                     │
│  └── 致命错误（IsArchived 等）→ 禁用集成                                   │
│                                                                          │
│  多渠道消息 (SendReplyJob)                                                 │
│  ├── retry_on: 无 ❌                                                       │
│  ├── 各 SendOn*Service 自己 rescue → 吞掉                                  │
│  └── 终态: 消息 status=failed                                              │
│                                                                          │
│  队列默认重试 (Sidekiq)                                                    │
│  ├── 理论上: 25 次指数退避                                                 │
│  └── 实际上: 由于到处都是 rescue，几乎不会触发 😱                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
