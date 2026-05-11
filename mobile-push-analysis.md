# Chatwoot 移动端推送通知分析报告

## 1. 概述

本报告分析 Chatwoot 系统中移动端推送通知的路由机制（FCM vs APNs）、客户端注册链路、token/device_id 的作用，以及投递失败时的处理策略和重试机制。

---

## 2. 移动端推送架构概览

Chatwoot 的移动端推送通知系统采用 **FCM 作为统一推送网关** 的架构，不直接与 APNs 通信，而是通过 FCM 中继处理 iOS 设备的推送。

### 核心组件

| 组件 | 文件路径 | 职责 |
|------|---------|------|
| PushNotificationService | `app/services/notification/push_notification_service.rb` | 推送通知的核心路由和分发服务 |
| FcmService | `app/services/notification/fcm_service.rb` | FCM 客户端初始化和认证 |
| PushNotificationJob | `app/jobs/notification/push_notification_job.rb` | 异步任务调度 |
| ChatwootHub | `lib/chatwoot_hub.rb` | 推送中继服务（无自建 Firebase 时使用） |
| NotificationSubscription | `app/models/notification_subscription.rb` | 用户订阅模型 |
| NotificationSubscriptionBuilder | `app/builders/notification_subscription_builder.rb` | 订阅构建器（处理注册/更新逻辑） |
| NotificationSubscriptionsController | `app/controllers/api/v1/notification_subscriptions_controller.rb` | 订阅 API 控制器 |

---

## 3. 客户端注册到服务端订阅的完整链路

### 3.1 注册链路概览

```
移动端 App / 浏览器
        ↓
获取 FCM token / WebPush endpoint
        ↓
调用 POST /api/v1/notification_subscriptions
        ↓
NotificationSubscriptionsController#create
        ↓
NotificationSubscriptionBuilder#perform
        ↓
创建/更新 NotificationSubscription 记录
```

### 3.2 API 层：控制器处理

**位置**：`app/controllers/api/v1/notification_subscriptions_controller.rb:1-25`

```ruby
class Api::V1::NotificationSubscriptionsController < Api::BaseController
  before_action :set_user

  def create
    notification_subscription = NotificationSubscriptionBuilder.new(
      user: @user, 
      params: notification_subscription_params
    ).perform

    render json: notification_subscription
  end

  def destroy
    # 通过 push_token 查找并删除订阅
    notification_subscription = NotificationSubscription.where(
      ["subscription_attributes->>'push_token' = ?", params[:push_token]]
    ).first
    notification_subscription.destroy! if notification_subscription.present?
    head :ok
  end

  private

  def notification_subscription_params
    params.require(:notification_subscription).permit(
      :subscription_type, 
      subscription_attributes: {}
    )
  end
end
```

**API 端点**：
- `POST /api/v1/notification_subscriptions` - 创建/更新订阅
- `DELETE /api/v1/notification_subscriptions` - 删除订阅（通过 push_token）

### 3.3 业务层：订阅构建器

**位置**：`app/builders/notification_subscription_builder.rb:1-34`

```ruby
class NotificationSubscriptionBuilder
  pattr_initialize [:params, :user!]

  def perform
    # 处理多账号登录同一浏览器/设备的情况
    move_subscription_to_user if identifier_subscription && identifier_subscription.user_id != user.id
    
    # 存在则更新，不存在则创建
    identifier_subscription.blank? ? build_identifier_subscription : update_identifier_subscription
    identifier_subscription
  end

  private

  def identifier
    # browser_push 类型使用 endpoint 作为唯一标识
    @identifier ||= params[:subscription_attributes][:endpoint] if params[:subscription_type] == 'browser_push'
    # fcm 类型使用 device_id 作为唯一标识
    @identifier ||= params[:subscription_attributes][:device_id] if params[:subscription_type] == 'fcm'
    @identifier
  end

  def identifier_subscription
    @identifier_subscription ||= NotificationSubscription.find_by(identifier: identifier)
  end

  def move_subscription_to_user
    @identifier_subscription.update(user_id: user.id)
  end

  def build_identifier_subscription
    @identifier_subscription = user.notification_subscriptions.create!(
      params.merge(identifier: identifier)
    )
  end

  def update_identifier_subscription
    identifier_subscription.update(params.merge(identifier: identifier))
  end
end
```

### 3.4 订阅数据结构

**数据库表结构** (`db/schema.rb:1044-1052`)：
```ruby
create_table "notification_subscriptions", force: :cascade do |t|
  t.bigint "user_id", null: false
  t.integer "subscription_type", null: false  # 1: browser_push, 2: fcm
  t.jsonb "subscription_attributes", default: {}, null: false
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.text "identifier"  # 唯一索引，用于去重
end
```

**subscription_attributes 字段内容**：

| 订阅类型 | 字段 | 说明 |
|---------|------|------|
| `browser_push` | `endpoint` | WebPush 服务端点 URL |
| `browser_push` | `p256dh` | 椭圆曲线公钥 |
| `browser_push` | `auth` | 认证密钥 |
| `fcm` | `push_token` | FCM 推送令牌（核心） |
| `fcm` | `device_id` | 设备唯一标识（作为 identifier） |

### 3.5 客户端请求示例

**移动端 FCM 订阅请求**：
```json
POST /api/v1/notification_subscriptions
{
  "notification_subscription": {
    "subscription_type": "fcm",
    "subscription_attributes": {
      "push_token": "dU8_Z...FCM_TOKEN",
      "device_id": "iPhone14,3-ABC123DEF456"
    }
  }
}
```

**浏览器 WebPush 订阅请求**：
```json
POST /api/v1/notification_subscriptions
{
  "notification_subscription": {
    "subscription_type": "browser_push",
    "subscription_attributes": {
      "endpoint": "https://fcm.googleapis.com/fcm/send/...",
      "p256dh": "BP12345...",
      "auth": "ABCDEF..."
    }
  }
}
```

---

## 4. device_id 和 push_token 对 FCM 到 APNs 投递的影响

### 4.1 两个关键字段的职责划分

#### push_token（推送令牌）

**来源**：
- **Android**：由 FCM SDK 生成
- **iOS**：由 Firebase iOS SDK 生成（内部封装了 APNs token）

**作用**：
1. **决定投递目标平台**：FCM 根据 token 的类型/前缀自动判断目标是 Android 还是 iOS
2. **定位具体设备**：FCM 使用 token 在其注册中心查找目标设备
3. **iOS 场景**：FCM 将 token 映射到对应的 APNs token，然后转发到 APNs

**在代码中的使用** (`push_notification_service.rb:128`)：
```ruby
def fcm_options(subscription)
  {
    'token': subscription.subscription_attributes['push_token'],  # 核心：目标设备标识
    'data': fcm_data,
    'notification': fcm_notification,
    'android': fcm_android_options,
    'apns': fcm_apns_options,
    ...
  }
end
```

#### device_id（设备标识）

**来源**：
- **Android**：通常使用 `Settings.Secure.ANDROID_ID` 或 Firebase Installation ID
- **iOS**：通常使用 `identifierForVendor` 或 `UIDevice.current.identifierForVendor`

**作用**：
1. **作为订阅的唯一标识符**（`identifier` 字段），用于：
   - 防止同一设备重复注册
   - 更新已存在的订阅（如 token 刷新时）
   - 处理用户切换账号的场景（将订阅移到新用户）
2. **不直接参与 FCM 投递决策**，仅用于服务端订阅管理

**在代码中的使用** (`notification_subscription_builder.rb:13-17`)：
```ruby
def identifier
  # browser_push 使用 endpoint 作为 identifier
  @identifier ||= params[:subscription_attributes][:endpoint] if params[:subscription_type] == 'browser_push'
  # fcm 使用 device_id 作为 identifier
  @identifier ||= params[:subscription_attributes][:device_id] if params[:subscription_type] == 'fcm'
  @identifier
end
```

### 4.2 FCM 如何根据 token 决定投递到 Android 还是 APNs

#### FCM token 的结构特征

虽然 FCM token 的具体格式未公开文档化，但根据 Firebase 文档和社区实践：

| 平台 | token 特征（推测） | FCM 处理方式 |
|------|-------------------|-------------|
| Android | 通常以 `dU`、`eU` 等开头 | 直接投递到 Android 设备 |
| iOS | 通常以 `cU`、`fU` 等开头 | 中继到 APNs |
| 通用 | 长度约 140-180 字符 | FCM 内部注册表决定 |

#### FCM 到 APNs 的投递流程

```
Chatwoot 后端
    ↓ 发送到 FCM (包含 token + apns payload)
FCM 服务
    ↓ 解析 token，判断是 iOS token
    ↓ 查找对应的 APNs token（Firebase 内部映射）
    ↓ 使用配置的 APNs 证书/密钥
APNs 服务
    ↓ 投递到 iOS 设备
iPhone / iPad
```

#### 消息体中的平台配置

**位置**：`push_notification_service.rb:126-171`

Chatwoot 在发送到 FCM 时，**同时包含 Android 和 iOS 的配置**，由 FCM 根据 token 选择适用的部分：

```ruby
def fcm_options(subscription)
  {
    'token': subscription.subscription_attributes['push_token'],  # 决定目标平台
    'data': fcm_data,                                              # 两个平台通用数据
    'notification': fcm_notification,                              # 两个平台通用通知
    'android': { priority: 'high' },                               # Android 专用配置
    'apns': {                                                      # iOS 专用配置
      payload: {
        aps: {
          sound: 'default',
          category: Time.zone.now.to_i.to_s
        }
      }
    },
    'fcm_options': { analytics_label: 'Label' }
  }
end
```

### 4.3 关键场景分析

#### 场景 1：Token 刷新

**问题**：FCM token 可能会过期或刷新（如应用重装、系统重置等）

**处理流程**：
1. 移动端检测到 token 变化
2. 使用相同的 `device_id` 重新调用注册 API
3. `NotificationSubscriptionBuilder` 通过 `identifier`（即 device_id）找到旧订阅
4. 更新订阅的 `push_token` 和其他属性

**代码** (`notification_subscription_builder.rb:31-33`)：
```ruby
def update_identifier_subscription
  identifier_subscription.update(params.merge(identifier: identifier))
end
```

#### 场景 2：用户切换账号（同一设备）

**问题**：同一台设备上用户 A 登出，用户 B 登入

**处理流程**：
1. 用户 B 登录后，移动端发送注册请求（携带相同 device_id）
2. `NotificationSubscriptionBuilder` 发现订阅已存在但属于用户 A
3. 将订阅的 `user_id` 更新为用户 B

**代码** (`notification_subscription_builder.rb:23-25`)：
```ruby
def move_subscription_to_user
  @identifier_subscription.update(user_id: user.id)
end
```

#### 场景 3：iOS 设备的 APNs 凭证问题

**问题**：Firebase 项目的 APNs 证书/密钥过期或配置错误

**影响**：
- FCM 尝试投递到 APNs 时会失败
- 返回 `InvalidApnsCredential` 错误
- Chatwoot 当前实现会**删除订阅**（可能误删有效订阅）

**相关代码** (`push_notification_service.rb:198-204`)：
```ruby
def remove_subscription_if_error(subscription, response)
  if JSON.parse(response[:body])['results']&.first&.keys&.include?('error')
    subscription.destroy!  # 任何错误都删除
  end
end
```

---

## 5. FCM 与 APNs 路由机制

### 5.1 订阅类型识别

`NotificationSubscription` 模型定义了两种订阅类型：

```ruby
SUBSCRIPTION_TYPES = {
  browser_push: 1,  # 浏览器推送
  fcm: 2            # 移动端推送（Android + iOS）
}.freeze
```

**关键发现**：系统没有单独的 `apns` 订阅类型，所有移动端推送统一归类为 `fcm` 类型。

### 5.2 FCM 统一网关策略

Chatwoot 使用 **FCM 作为统一的移动端推送平台**，通过 FCM 的 `send_v1` API 同时支持 Android 和 iOS 设备。

### 5.3 平台特定配置

**Android 配置** (`fcm_android_options`):
```ruby
def fcm_android_options
  {
    priority: 'high'  # 高优先级推送
  }
end
```

**iOS 配置** (`fcm_apns_options`):
```ruby
def fcm_apns_options
  {
    payload: {
      aps: {
        sound: 'default',           # 默认提示音
        category: Time.zone.now.to_i.to_s  # 通知类别标识
      }
    }
  }
end
```

**架构说明**：
- FCM 收到包含 `apns` 字段的消息后，会自动将其转发到 APNs
- 设备注册时获取的 token 决定了最终投递到哪个平台
- 后端无需区分 Android/iOS，统一发送到 FCM 即可

### 5.4 推送路由决策树

```
通知创建
    ↓
PushNotificationJob 入队
    ↓
PushNotificationService#perform
    ↓
检查用户通知设置 (user_subscribed_to_notification?)
    ↓
遍历用户所有 notification_subscriptions
    ↓
┌─────────────────────────────────────────────────────┐
│ 对每个 subscription：                               │
│                                                     │
│  1. send_browser_push(subscription)                 │
│     - 条件：VAPID 配置存在 且 subscription.browser_push? │
│                                                     │
│  2. send_fcm_push(subscription)                     │
│     - 条件：Firebase 凭证配置存在 且 subscription.fcm? │
│     - 路径：直接调用 FCM API                         │
│                                                     │
│  3. send_push_via_chatwoot_hub(subscription)        │
│     - 条件：无 Firebase 凭证 且 ENABLE_PUSH_RELAY_SERVER=true │
│     - 路径：通过 Chatwoot Hub 中继服务器发送          │
└─────────────────────────────────────────────────────┘
```

### 5.5 FCM 推送的双重路由

**路由一：直接 FCM 发送**

```ruby
def send_fcm_push(subscription)
  return unless firebase_credentials_present?  # 检查 FIREBASE_PROJECT_ID 和 FIREBASE_CREDENTIALS
  return unless subscription.fcm?

  fcm_service = Notification::FcmService.new(
    GlobalConfigService.load('FIREBASE_PROJECT_ID', nil), 
    GlobalConfigService.load('FIREBASE_CREDENTIALS', nil)
  )
  fcm = fcm_service.fcm_client
  response = fcm.send_v1(fcm_options(subscription))
  remove_subscription_if_error(subscription, response)
end
```

**路由二：Chatwoot Hub 中继**

```ruby
def send_push_via_chatwoot_hub(subscription)
  return if firebase_credentials_present?  # 已配置 Firebase 则跳过
  return unless chatwoot_hub_enabled?      # 检查 ENABLE_PUSH_RELAY_SERVER (默认 true)
  return unless subscription.fcm?

  ChatwootHub.send_push(fcm_options(subscription))
end
```

**路由优先级**：
1. **首选**：直接 FCM 发送（需要配置 Firebase 凭证）
2. **备选**：Chatwoot Hub 中继（无需配置 Firebase，使用 Chatwoot 官方服务）

---

## 6. 重试机制与异常分层处理

### 6.1 Sidekiq 全局配置

**位置**：`config/sidekiq.yml:1-39`

```yaml
:verbose: false
:concurrency: <%= ENV.fetch("SIDEKIQ_CONCURRENCY", 10) %>
:timeout: 25
:max_retries: 3    # 全局最大重试次数

:queues:
  - critical
  - high
  - medium
  - default        # PushNotificationJob 使用 default 队列
  - mailers
  ...
```

**关键点**：
- 全局 `max_retries: 3`，意味着 Job 失败后最多重试 3 次
- 使用 Sidekiq 默认的指数退避算法：第 1 次失败后约 15 秒，第 2 次约 16 分钟，第 3 次约 6 小时
- `PushNotificationJob` 使用 `default` 队列

### 6.2 ApplicationJob 基础异常处理

**位置**：`app/jobs/application_job.rb:1-8`

```ruby
class ApplicationJob < ActiveJob::Base
  # 反序列化错误直接丢弃，不重试
  discard_on ActiveJob::DeserializationError do |job, error|
    Rails.logger.info("Skipping #{job.class} with #{
      job.instance_variable_get(:@serialized_arguments)
    } because of ActiveJob::DeserializationError (#{error.message})")
  end
end
```

**处理策略**：
- `ActiveJob::DeserializationError`：参数反序列化失败（如关联记录已删除），直接丢弃不重试
- 其他异常：遵循 Sidekiq 全局配置，最多重试 3 次

### 6.3 PushNotificationJob 的重试行为

**位置**：`app/jobs/notification/push_notification_job.rb:1-7`

```ruby
class Notification::PushNotificationJob < ApplicationJob
  queue_as :default

  def perform(notification)
    Notification::PushNotificationService.new(notification: notification).perform
  end
end
```

**关键点**：
- 继承自 `ApplicationJob`，没有额外配置 `retry_on` 或 `discard_on`
- **依赖 Sidekiq 全局重试机制**（最多 3 次）
- 但 `PushNotificationService` 内部的异常处理可能影响重试

### 6.4 项目中的异常分层模式（参考其他 Job）

Chatwoot 项目中其他 Job 使用了更精细的异常处理策略：

#### 示例 1：AgentBots::WebhookJob

**位置**：`app/jobs/agent_bots/webhook_job.rb:1-16`

```ruby
class AgentBots::WebhookJob < WebhookJob
  queue_as :high
  # 对可重试异常自定义重试策略
  retry_on Webhooks::Trigger::RetryableError, wait: 3.seconds, attempts: 3 do |job, error|
    url, payload, webhook_type = job.arguments
    kwargs = job.arguments.last.is_a?(Hash) ? job.arguments.last : {}
    Webhooks::Trigger.new(...).handle_failure(error)
  end

  def perform(...)
    super(...)
  rescue Webhooks::Trigger::RetryableError => e
    Rails.logger.warn("[AgentBots::WebhookJob] attempt #{executions} failed #{e.class.name}")
    raise  # 重新抛出以触发 retry_on
  end
end
```

**模式特点**：
1. 定义**自定义异常类型**（如 `RetryableError`）区分可重试和不可重试错误
2. 使用 `retry_on` 指定重试次数和等待时间
3. 在 `perform` 中捕获异常、记录日志、重新抛出
4. 重试耗尽后执行回调块

#### 示例 2：DataImportJob

**位置**：`app/jobs/data_import_job.rb:4-7`

```ruby
class DataImportJob < ApplicationJob
  queue_as :low
  retry_on ActiveStorage::FileNotFoundError, wait: 1.minute, attempts: 3
  ...
end
```

**模式特点**：
- 对特定异常（如文件未找到）使用自定义重试策略
- 等待时间和重试次数可以根据业务需求调整

### 6.5 ExceptionList：异常分类定义

**位置**：`lib/exception_list.rb:1-19`

```ruby
module ExceptionList
  REST_CLIENT_EXCEPTIONS = [
    RestClient::NotFound, 
    RestClient::GatewayTimeout, 
    RestClient::BadRequest,
    RestClient::MethodNotAllowed, 
    RestClient::Forbidden, 
    RestClient::InternalServerError,
    RestClient::Exceptions::OpenTimeout, 
    RestClient::Exceptions::ReadTimeout,
    RestClient::TemporaryRedirect, 
    RestClient::SSLCertificateNotVerified, 
    RestClient::PaymentRequired,
    RestClient::BadGateway, 
    RestClient::Unauthorized, 
    RestClient::PayloadTooLarge,
    RestClient::MovedPermanently, 
    RestClient::ServiceUnavailable, 
    Errno::ECONNREFUSED, 
    SocketError
  ].freeze
  
  SMTP_EXCEPTIONS = [Net::SMTPSyntaxError].freeze
  IMAP_EXCEPTIONS = [...]
end
```

**异常分层分类**：

| 异常类别 | 包含异常 | 典型场景 | 可重试？ |
|---------|---------|---------|---------|
| 网络/超时 | `OpenTimeout`, `ReadTimeout`, `ECONNREFUSED`, `SocketError` | 网络波动、DNS 问题 | ✅ 通常可重试 |
| 服务端错误 | `InternalServerError`, `BadGateway`, `ServiceUnavailable`, `GatewayTimeout` | 服务端过载、维护 | ✅ 通常可重试 |
| 客户端错误 | `BadRequest`, `NotFound`, `Forbidden`, `Unauthorized` | 参数错误、权限问题 | ⚠️ 需判断 |
| 业务错误 | `MovedPermanently`, `PayloadTooLarge` | 永久重定向、请求过大 | ❌ 不可重试 |

### 6.6 推送通知的异常处理分层

#### 当前实现的问题

**PushNotificationService 内部异常处理不一致**：

```
send_browser_push (有异常捕获)
    ↓ 异常
    → handle_browser_push_error (分类处理，部分删除订阅)

send_fcm_push (无异常捕获)
    ↓ 异常
    → 向上抛出
    → Sidekiq 全局重试 (最多 3 次)
    → 但 FCM 返回的错误响应会删除订阅

send_push_via_chatwoot_hub (有异常捕获)
    ↓ 异常
    → 仅记录日志，不删除订阅
    → 不向上抛出
    → Sidekiq 认为成功，不会重试
```

#### 三种推送方式的异常处理对比

| 推送方式 | 异常捕获 | 重试行为 | 订阅清理 |
|---------|---------|---------|---------|
| browser_push | ✅ 精细分类 | ❌ 捕获后不重试 | ✅ 智能（按错误类型） |
| fcm (直接) | ❌ 仅处理 HTTP 响应 | ✅ Sidekiq 全局重试 | ⚠️ 任何错误都删除 |
| fcm (Hub 中继) | ✅ 全部捕获 | ❌ 捕获后不重试 | ❌ 不清理 |

#### ChatwootHub 中继的异常处理

**位置**：`lib/chatwoot_hub.rb:108-114`

```ruby
def self.send_push(fcm_options)
  send_push_with_response(fcm_options)
rescue *ExceptionList::REST_CLIENT_EXCEPTIONS => e
  Rails.logger.error "Exception: #{e.message}"
  # 问题：捕获后不重新抛出，Sidekiq 不会重试
rescue StandardError => e
  ChatwootExceptionTracker.new(e).capture_exception
  # 问题：同上，不重试
end
```

**问题**：
- 所有异常被捕获后**静默处理**，不重新抛出
- Sidekiq 认为 Job 执行成功，**不会触发重试**
- 临时性网络错误导致的推送失败无法恢复

### 6.7 理想的异常分层处理建议

参考项目中其他 Job 的模式，建议的推送异常处理架构：

```
┌─────────────────────────────────────────────────────────────────┐
│ 异常分层架构建议                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: 不可恢复错误 (立即丢弃)                                 │
│  ├── InvalidSubscription / NotRegistered (token 永久失效)        │
│  ├── MismatchSenderId (配置错误)                                 │
│  └── ActiveJob::DeserializationError                            │
│                                                                 │
│  Layer 2: 可重试错误 (指数退避)                                  │
│  ├── Network 层: OpenTimeout, ReadTimeout, ECONNREFUSED         │
│  ├── Server 层: InternalServerError, Unavailable, BadGateway    │
│  └── Rate Limit: TooManyRequests, QuotaExceeded                 │
│                                                                 │
│  Layer 3: 业务决策 (不重试，记录日志)                             │
│  └── 配置缺失: Firebase 凭证未配置                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 投递失败处理策略

### 7.1 浏览器推送失败处理

**位置**：`push_notification_service.rb:75-88`

```ruby
def handle_browser_push_error(error, subscription)
  case error
  when WebPush::ExpiredSubscription, WebPush::InvalidSubscription, WebPush::Unauthorized
    Rails.logger.info "WebPush subscription expired: #{error.message}"
    subscription.destroy!  # 永久删除无效订阅
  when WebPush::TooManyRequests
    Rails.logger.warn "WebPush rate limited for #{user.email} on account #{notification.account.id}: #{error.message}"
    # 仅记录警告，不删除订阅（限流是临时性的）
  when Errno::ECONNRESET, Net::OpenTimeout, Net::ReadTimeout, Socket::ResolutionError
    Rails.logger.error "WebPush operation error: #{error.message}"
    # 网络错误，记录日志，不删除订阅
  else
    ChatwootExceptionTracker.new(error, account: notification.account).capture_exception
    true  # 未知错误，上报异常追踪
  end
end
```

**处理策略**：

| 错误类型 | 处理方式 | 理由 |
|---------|---------|------|
| ExpiredSubscription | 删除订阅 | 订阅已过期，永久无效 |
| InvalidSubscription | 删除订阅 | 订阅格式无效 |
| Unauthorized | 删除订阅 | 权限被撤销 |
| TooManyRequests | 记录警告 | 限流是临时性的，稍后可重试 |
| 网络超时/连接错误 | 记录错误 | 网络问题是临时性的 |
| 其他错误 | 异常追踪 | 需要进一步分析 |

### 7.2 FCM 推送失败处理

**位置**：`push_notification_service.rb:118-124`

```ruby
def remove_subscription_if_error(subscription, response)
  if JSON.parse(response[:body])['results']&.first&.keys&.include?('error')
    subscription.destroy!  # FCM 返回错误，删除订阅
  else
    Rails.logger.info("FCM push sent to #{user.email} with title #{push_message[:title]}")
  end
end
```

**处理策略**：
- 检查 FCM 响应 `results[0]` 是否包含 `error` 字段
- **只要 FCM 返回任何错误，立即删除订阅**
- 没有区分可恢复错误和不可恢复错误

**FCM 常见错误码**（根据 FCM API 文档）：

| 错误码 | 含义 | 可恢复？ | 建议处理 |
|--------|------|---------|---------|
| `InvalidRegistration` | Token 格式无效 | ❌ | 删除订阅 |
| `NotRegistered` | Token 已注销 | ❌ | 删除订阅 |
| `MismatchSenderId` | Sender ID 不匹配 | ❌ | 配置错误，需检查 |
| `InvalidApnsCredential` | APNs 凭证无效 | ⚠️ | 配置问题，非订阅问题 |
| `QuotaExceeded` | 配额超限 | ✅ | 稍后重试 |
| `DeviceMessageRateExceeded` | 设备频率超限 | ✅ | 稍后重试 |
| `InternalServerError` | FCM 服务端错误 | ✅ | 稍后重试 |
| `Unavailable` | FCM 不可用 | ✅ | 稍后重试 |

**潜在问题**：
- 当前实现将所有 FCM 错误视为不可恢复，直接删除订阅
- 对于临时性错误（如限流、服务端不可用、APNs 凭证过期），这会导致用户订阅被误删

### 7.3 Chatwoot Hub 中继失败处理

**位置**：`lib/chatwoot_hub.rb:108-114`

```ruby
def self.send_push(fcm_options)
  send_push_with_response(fcm_options)
rescue *ExceptionList::REST_CLIENT_EXCEPTIONS => e
  Rails.logger.error "Exception: #{e.message}"
  # 仅记录日志，不做其他处理
rescue StandardError => e
  ChatwootExceptionTracker.new(e).capture_exception
  # 上报异常追踪
end
```

**处理策略**：
- REST 客户端异常：仅记录错误日志
- 其他异常：上报异常追踪器
- **不会删除用户订阅**
- **没有重试机制**（异常被捕获后不重新抛出）

### 7.4 FCM 服务认证失败处理

**位置**：`app/services/notification/fcm_service.rb`

```ruby
def generate_token
  authorizer = Google::Auth::ServiceAccountCredentials.make_creds(
    json_key_io: credentials_path,
    scope: SCOPES
  )
  token = authorizer.fetch_access_token!  # 认证失败会抛出异常
  {
    token: token['access_token'],
    expires_at: Time.zone.now + token['expires_in'].to_i
  }
end
```

**处理策略**：
- 认证失败时直接抛出异常
- 异常会向上传播到 `PushNotificationService#send_fcm_push`
- 但 `send_fcm_push` 方法**没有异常捕获**
- 最终异常会由 Sidekiq 的异常处理机制捕获（全局重试 3 次）

---

## 8. 推送通知生命周期

### 8.1 通知创建流程

```ruby
# notification.rb:155-164
def process_notification_delivery
  Notification::PushNotificationJob.perform_later(self) if user_subscribed_to_notification?('push')
  
  Notification::EmailNotificationJob.perform_later(self) if user_subscribed_to_notification?('email')
  
  Notification::RemoveDuplicateNotificationJob.perform_later(self)
end
```

### 8.2 用户通知设置检查

```ruby
# push_notification_service.rb:22-27
def user_subscribed_to_notification?
  notification_setting = notification_settings.find_by(account_id: notification.account.id)
  return true if notification_setting.public_send("push_#{notification.notification_type}?")
  false
end
```

**支持的通知类型**（`notification.rb:38-47`）：
- `conversation_creation` - 新会话创建
- `conversation_assignment` - 会话分配
- `assigned_conversation_new_message` - 已分配会话新消息
- `conversation_mention` - @提及
- `participating_conversation_new_message` - 参与的会话新消息
- `sla_missed_first_response` - SLA 首次响应超时
- `sla_missed_next_response` - SLA 后续响应超时
- `sla_missed_resolution` - SLA 解决超时

---

## 9. 关键配置项

| 配置项 | 环境变量 | 用途 |
|--------|---------|------|
| Firebase Project ID | `FIREBASE_PROJECT_ID` | 直接 FCM 发送必需 |
| Firebase Credentials | `FIREBASE_CREDENTIALS` | 直接 FCM 发送必需（JSON 字符串） |
| Push Relay Server | `ENABLE_PUSH_RELAY_SERVER` | 是否启用 Chatwoot Hub 中继（默认 true） |
| VAPID Public Key | （通过 VapidService） | 浏览器推送必需 |
| Chatwoot Hub URL | `CHATWOOT_HUB_URL`（企业版开发环境） | 中继服务器地址 |
| Sidekiq 并发数 | `SIDEKIQ_CONCURRENCY` | Sidekiq 并发处理数（默认 10） |

---

## 10. 总结与建议

### 10.1 架构特点

1. **FCM 统一网关**：通过 FCM 同时处理 Android 和 iOS，简化了后端架构
2. **双重路由**：支持自建 Firebase 或使用 Chatwoot 官方中继服务
3. **异步处理**：所有推送通过 Sidekiq 异步执行，不阻塞主流程
4. **设备维度去重**：使用 `device_id` 作为订阅唯一标识，支持 token 刷新和用户切换

### 10.2 关键字段职责总结

| 字段 | 来源 | 作用 | 影响投递决策 |
|------|------|------|-------------|
| `push_token` | FCM SDK | 标识目标设备，FCM 据此判断平台 | ✅ 直接影响 |
| `device_id` | 操作系统 | 服务端订阅去重和更新 | ❌ 不影响 |
| `identifier` | 构建器生成 | 数据库唯一索引 | ❌ 不影响 |

### 10.3 失败处理策略总结

| 推送类型 | 失败处理策略 | 订阅清理 | 重试机制 |
|---------|-------------|---------|---------|
| 浏览器推送 | 分类处理：过期/无效订阅删除，限流/网络错误仅记录 | ✅ 智能清理 | ❌ 无（捕获后不抛出） |
| FCM 直接发送 | 任何 FCM 错误都删除订阅 | ⚠️ 可能误删 | ✅ Sidekiq 全局 3 次 |
| Chatwoot Hub 中继 | 仅记录日志/上报异常 | ❌ 不清理 | ❌ 无（捕获后不抛出） |

### 10.4 异常分层处理现状

| 层级 | 处理方式 | 推送 Job 应用情况 |
|------|---------|-----------------|
| `discard_on` | 不可恢复错误立即丢弃 | ✅ 仅 DeserializationError |
| `retry_on` | 可重试错误自定义策略 | ❌ PushNotificationJob 未配置 |
| Sidekiq 全局 | 统一重试 3 次 | ✅ 作为兜底 |
| 业务层捕获 | 精细分类处理 | ⚠️ 不一致 |

### 10.5 潜在改进点

#### 改进点 1：FCM 错误精细化处理

**当前问题**：所有 FCM 错误都删除订阅

**建议**：
```ruby
# 建议的改进实现
def remove_subscription_if_error(subscription, response)
  error = JSON.parse(response[:body])['results']&.first&.[]('error')
  return if error.nil?

  # 不可恢复错误：删除订阅
  unrecoverable_errors = %w[InvalidRegistration NotRegistered MismatchSenderId]
  if unrecoverable_errors.include?(error)
    subscription.destroy!
    Rails.logger.info("FCM subscription removed: #{error}")
  else
    # 可恢复错误：记录日志，依赖 Sidekiq 重试
    Rails.logger.warn("FCM retryable error: #{error}")
    raise "FCM #{error}"  # 抛出异常触发重试
  end
end
```

#### 改进点 2：统一重试机制

**当前问题**：
- 直接 FCM：依赖 Sidekiq 全局重试
- Hub 中继：异常被捕获，不重试
- 浏览器推送：异常被捕获，不重试

**建议**：
参考 `AgentBots::WebhookJob` 的模式，在 `PushNotificationJob` 中添加：
```ruby
class Notification::PushNotificationJob < ApplicationJob
  queue_as :default
  
  # 网络和服务端错误：重试 3 次，指数退避
  retry_on Net::OpenTimeout, wait: :exponentially_longer, attempts: 3
  retry_on Net::ReadTimeout, wait: :exponentially_longer, attempts: 3
  retry_on SocketError, wait: :exponentially_longer, attempts: 3
  
  def perform(notification)
    Notification::PushNotificationService.new(notification: notification).perform
  end
end
```

#### 改进点 3：ChatwootHub 异常处理改进

**当前问题**：Hub 中继的异常被静默捕获，不触发重试

**建议**：
```ruby
# lib/chatwoot_hub.rb 改进
def self.send_push(fcm_options)
  send_push_with_response(fcm_options)
rescue *ExceptionList::REST_CLIENT_EXCEPTIONS => e
  Rails.logger.error "ChatwootHub push failed: #{e.message}"
  # 可重试错误重新抛出
  retryable_errors = [
    RestClient::GatewayTimeout,
    RestClient::InternalServerError,
    RestClient::BadGateway,
    RestClient::ServiceUnavailable,
    RestClient::Exceptions::OpenTimeout,
    RestClient::Exceptions::ReadTimeout,
    Errno::ECONNREFUSED,
    SocketError
  ]
  raise if retryable_errors.include?(e.class)
rescue StandardError => e
  ChatwootExceptionTracker.new(e).capture_exception
  raise  # 其他异常也抛出，让 Sidekiq 决定是否重试
end
```

#### 改进点 4：订阅状态追踪

- 增加 `failure_count` 字段记录连续失败次数
- 增加 `last_failed_at` 字段记录最后失败时间
- 连续多次失败后再删除订阅，避免单次临时性错误导致订阅丢失

#### 改进点 5：监控和告警

- 推送失败率监控
- FCM/APNs 服务健康检查
- 订阅清理频率监控
- 异常追踪告警

---

## 11. 代码引用索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 订阅 API 控制器 | `app/controllers/api/v1/notification_subscriptions_controller.rb` | 1-25 |
| 订阅构建器 | `app/builders/notification_subscription_builder.rb` | 1-34 |
| 订阅模型 | `app/models/notification_subscription.rb` | 1-29 |
| 推送路由主逻辑 | `app/services/notification/push_notification_service.rb` | 6-14 |
| FCM 直接发送 | `app/services/notification/push_notification_service.rb` | 90-100 |
| Chatwoot Hub 中继 | `app/services/notification/push_notification_service.rb` | 102-108 |
| FCM 错误处理 | `app/services/notification/push_notification_service.rb` | 118-124 |
| 浏览器推送错误处理 | `app/services/notification/push_notification_service.rb` | 75-88 |
| APNs 配置（通过 FCM） | `app/services/notification/push_notification_service.rb` | 162-171 |
| FCM 认证服务 | `app/services/notification/fcm_service.rb` | 1-40 |
| Chatwoot Hub 客户端 | `lib/chatwoot_hub.rb` | 108-119 |
| 异常分类列表 | `lib/exception_list.rb` | 1-19 |
| 推送 Job | `app/jobs/notification/push_notification_job.rb` | 1-7 |
| 基础 Job | `app/jobs/application_job.rb` | 1-8 |
| Webhook Job 重试模式参考 | `app/jobs/agent_bots/webhook_job.rb` | 1-16 |
| Sidekiq 全局配置 | `config/sidekiq.yml` | 1-39 |
| 通知触发入口 | `app/models/notification.rb` | 155-164 |
| 数据库 schema | `db/schema.rb` | 1044-1052 |
