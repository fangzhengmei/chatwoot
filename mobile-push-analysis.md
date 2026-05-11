# Chatwoot 移动端推送通知分析报告

## 1. 概述

本报告基于 Chatwoot 源码分析移动端推送通知系统的架构、路由机制、客户端注册链路和失败处理策略。所有结论均有源码直接证据支持。

---

## 2. 核心组件

| 组件 | 文件路径 |
|------|---------|
| PushNotificationService | `app/services/notification/push_notification_service.rb` |
| FcmService | `app/services/notification/fcm_service.rb` |
| PushNotificationJob | `app/jobs/notification/push_notification_job.rb` |
| ChatwootHub | `lib/chatwoot_hub.rb` |
| NotificationSubscription | `app/models/notification_subscription.rb` |
| NotificationSubscriptionBuilder | `app/builders/notification_subscription_builder.rb` |
| NotificationSubscriptionsController | `app/controllers/api/v1/notification_subscriptions_controller.rb` |
| Notification | `app/models/notification.rb` |

---

## 3. 通知触发入口

### 3.1 通知创建后触发推送

**源码位置**：`app/models/notification.rb:155-164`

```ruby
def process_notification_delivery
  Notification::PushNotificationJob.perform_later(self) if user_subscribed_to_notification?('push')
  
  Notification::EmailNotificationJob.perform_later(self) if user_subscribed_to_notification?('email')
  
  Notification::RemoveDuplicateNotificationJob.perform_later(self)
end
```

**证据**：
- 通知创建后，`process_notification_delivery` 方法被调用（通过 `after_create_commit` 回调）
- 该方法检查用户是否订阅了推送通知，若是则异步执行 `PushNotificationJob`

### 3.2 用户通知设置检查

**源码位置**：`app/services/notification/push_notification_service.rb:22-27`

```ruby
def user_subscribed_to_notification?
  notification_setting = notification_settings.find_by(account_id: notification.account.id)
  return true if notification_setting.public_send("push_#{notification.notification_type}?")
  false
end
```

**证据**：
- 推送服务在执行前检查用户对该类型通知的订阅设置
- 不同通知类型（如 `conversation_creation`、`conversation_assignment`）有独立的开关

---

## 4. 客户端订阅注册链路

### 4.1 API 端点

**源码位置**：`app/controllers/api/v1/notification_subscriptions_controller.rb:1-25`

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
    notification_subscription = NotificationSubscription.where(
      ["subscription_attributes->>'push_token' = ?", params[:push_token]]
    ).first
    notification_subscription.destroy! if notification_subscription.present?
    head :ok
  end

  private

  def set_user
    @user = current_user
  end

  def notification_subscription_params
    params.require(:notification_subscription).permit(
      :subscription_type, 
      subscription_attributes: {}
    )
  end
end
```

**证据**：
- `POST /api/v1/notification_subscriptions`：创建/更新订阅
- `DELETE /api/v1/notification_subscriptions`：通过 `push_token` 删除订阅
- 参数包含 `subscription_type` 和 `subscription_attributes`

### 4.2 订阅构建器逻辑

**源码位置**：`app/builders/notification_subscription_builder.rb:1-34`

```ruby
class NotificationSubscriptionBuilder
  pattr_initialize [:params, :user!]

  def perform
    move_subscription_to_user if identifier_subscription && identifier_subscription.user_id != user.id
    identifier_subscription.blank? ? build_identifier_subscription : update_identifier_subscription
    identifier_subscription
  end

  private

  def identifier
    @identifier ||= params[:subscription_attributes][:endpoint] if params[:subscription_type] == 'browser_push'
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

**证据**：

| 逻辑 | 源码位置 |
|------|---------|
| identifier 生成规则 | `notification_subscription_builder.rb:13-17` |
| 已存在订阅判断 | `notification_subscription_builder.rb:19-21` |
| 订阅迁移到新用户 | `notification_subscription_builder.rb:23-25` |
| 创建新订阅 | `notification_subscription_builder.rb:27-29` |
| 更新已有订阅 | `notification_subscription_builder.rb:31-33` |

**identifier 字段规则**：
- `browser_push` 类型：使用 `subscription_attributes[:endpoint]`
- `fcm` 类型：使用 `subscription_attributes[:device_id]`

### 4.3 订阅数据模型

**源码位置**：`app/models/notification_subscription.rb:1-29`

```ruby
class NotificationSubscription < ApplicationRecord
  belongs_to :user
  validates :identifier, presence: true

  SUBSCRIPTION_TYPES = {
    browser_push: 1,
    fcm: 2
  }.freeze

  enum subscription_type: SUBSCRIPTION_TYPES
end
```

**数据库 schema 证据**：`db/schema.rb:1044-1052`

| 字段 | 类型 | 约束 |
|------|------|------|
| `user_id` | bigint | NOT NULL |
| `subscription_type` | integer | NOT NULL |
| `subscription_attributes` | jsonb | NOT NULL, default: {} |
| `identifier` | text | 唯一索引 |
| `created_at` | datetime | NOT NULL |
| `updated_at` | datetime | NOT NULL |

**证据**：
- 只有两种订阅类型：`browser_push` (1) 和 `fcm` (2)
- `identifier` 字段有唯一索引，用于去重
- `subscription_attributes` 是 JSONB 字段，存储推送所需的具体参数

### 4.4 subscription_attributes 字段内容

**源码证据**：

| 订阅类型 | 使用的字段 | 源码位置 |
|---------|-----------|---------|
| `browser_push` | `endpoint`, `p256dh`, `auth` | `push_notification_service.rb:49-64` |
| `fcm` | `push_token` | `push_notification_service.rb:126-137` |

**browser_push 使用示例**（`push_notification_service.rb:49-64`）：
```ruby
def browser_push_payload(subscription)
  {
    message: JSON.generate(push_message),
    endpoint: subscription.subscription_attributes['endpoint'],
    p256dh: subscription.subscription_attributes['p256dh'],
    auth: subscription.subscription_attributes['auth'],
    ...
  }
end
```

**fcm 使用示例**（`push_notification_service.rb:126-137`）：
```ruby
def fcm_options(subscription)
  {
    'token': subscription.subscription_attributes['push_token'],
    ...
  }
end
```

---

## 5. device_id 和 push_token 的作用

### 5.1 两个字段的职责

**源码位置**：`notification_subscription_builder.rb:13-17`

| 字段 | 来源 | 作用 |
|------|------|------|
| `device_id` | 客户端传入 | 作为 `identifier` 字段的值，用于订阅去重和更新 |
| `push_token` | 客户端传入 | 作为 FCM 发送时的 `token` 参数 |

**证据**：
- `device_id` 用于 `identifier` 字段：`notification_subscription_builder.rb:15`
- `push_token` 用于 FCM 发送：`push_notification_service.rb:128`

### 5.2 identifier 的使用场景

**场景 1：防止重复注册**
- 源码：`notification_subscription_builder.rb:19-21`
- 通过 `identifier` 查询已存在的订阅，避免重复创建

**场景 2：更新已有订阅**
- 源码：`notification_subscription_builder.rb:31-33`
- 当 `identifier` 已存在时，更新而非创建

**场景 3：用户切换账号**
- 源码：`notification_subscription_builder.rb:23-25`
- 当 `identifier` 已存在但属于其他用户时，将订阅迁移到当前用户

---

## 6. FCM 与 APNs 路由机制

### 6.1 订阅类型识别

**源码位置**：`app/models/notification_subscription.rb:23-28`

```ruby
SUBSCRIPTION_TYPES = {
  browser_push: 1,
  fcm: 2
}.freeze
```

**证据**：
- 系统只有两种订阅类型，没有单独的 `apns` 类型
- 所有移动端推送统一归类为 `fcm` 类型

### 6.2 推送路由决策

**源码位置**：`app/services/notification/push_notification_service.rb:6-14`

```ruby
def perform
  return unless user_subscribed_to_notification?

  notification_subscriptions.each do |subscription|
    send_browser_push(subscription)
    send_fcm_push(subscription)
    send_push_via_chatwoot_hub(subscription)
  end
end
```

**三种推送方式的条件**：

| 推送方式 | 源码位置 | 条件 |
|---------|---------|------|
| `send_browser_push` | `push_notification_service.rb:45-47` | VAPID 公钥存在 且 `subscription.browser_push?` |
| `send_fcm_push` | `push_notification_service.rb:90-92` | Firebase 凭证存在 且 `subscription.fcm?` |
| `send_push_via_chatwoot_hub` | `push_notification_service.rb:102-105` | Firebase 凭证不存在 且 Hub 启用 且 `subscription.fcm?` |

### 6.3 FCM 直接发送

**源码位置**：`app/services/notification/push_notification_service.rb:90-100`

```ruby
def send_fcm_push(subscription)
  return unless firebase_credentials_present?
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

**Firebase 凭证检查**（`push_notification_service.rb:110-112`）：
```ruby
def firebase_credentials_present?
  GlobalConfigService.load('FIREBASE_PROJECT_ID', nil) && GlobalConfigService.load('FIREBASE_CREDENTIALS', nil)
end
```

### 6.4 Chatwoot Hub 中继

**源码位置**：`app/services/notification/push_notification_service.rb:102-108`

```ruby
def send_push_via_chatwoot_hub(subscription)
  return if firebase_credentials_present?
  return unless chatwoot_hub_enabled?
  return unless subscription.fcm?

  ChatwootHub.send_push(fcm_options(subscription))
end
```

**Hub 启用检查**（`push_notification_service.rb:114-116`）：
```ruby
def chatwoot_hub_enabled?
  ActiveModel::Type::Boolean.new.cast(ENV.fetch('ENABLE_PUSH_RELAY_SERVER', true))
end
```

**ChatwootHub 发送实现**（`lib/chatwoot_hub.rb:108-119`）：
```ruby
def self.send_push(fcm_options)
  send_push_with_response(fcm_options)
rescue *ExceptionList::REST_CLIENT_EXCEPTIONS => e
  Rails.logger.error "Exception: #{e.message}"
rescue StandardError => e
  ChatwootExceptionTracker.new(e).capture_exception
end

def self.send_push_with_response(fcm_options)
  info = { fcm_options: fcm_options }
  RestClient.post(push_notification_url, info.merge(instance_config).to_json, { content_type: :json, accept: :json })
end
```

### 6.5 FCM 消息体格式

**源码位置**：`app/services/notification/push_notification_service.rb:126-171`

```ruby
def fcm_options(subscription)
  {
    'token': subscription.subscription_attributes['push_token'],
    'data': fcm_data,
    'notification': fcm_notification,
    'android': fcm_android_options,
    'apns': fcm_apns_options,
    'fcm_options': {
      analytics_label: 'Label'
    }
  }
end

def fcm_android_options
  {
    priority: 'high'
  }
end

def fcm_apns_options
  {
    payload: {
      aps: {
        sound: 'default',
        category: Time.zone.now.to_i.to_s
      }
    }
  }
end
```

**证据**：
- FCM 消息同时包含 `android` 和 `apns` 两个平台配置块
- 发送时使用 `push_token` 作为目标标识
- `apns` 配置包含 `aps` 标准字段（`sound`、`category`）

### 6.6 FCM 服务认证

**源码位置**：`app/services/notification/fcm_service.rb:1-40`

```ruby
class Notification::FcmService
  SCOPES = ['https://www.googleapis.com/auth/firebase.messaging'].freeze

  def initialize(project_id, credentials)
    @project_id = project_id
    @credentials = credentials
    @token_info = nil
  end

  def fcm_client
    FCM.new(current_token, credentials_path, @project_id)
  end

  private

  def current_token
    @token_info = generate_token if @token_info.nil? || token_expired?
    @token_info[:token]
  end

  def token_expired?
    Time.zone.now >= @token_info[:expires_at]
  end

  def generate_token
    authorizer = Google::Auth::ServiceAccountCredentials.make_creds(
      json_key_io: credentials_path,
      scope: SCOPES
    )
    token = authorizer.fetch_access_token!
    {
      token: token['access_token'],
      expires_at: Time.zone.now + token['expires_in'].to_i
    }
  end

  def credentials_path
    StringIO.new(@credentials)
  end
end
```

**证据**：
- 使用 Google 服务账号凭证进行认证
- Token 会缓存并在过期时刷新
- 认证失败会抛出异常（`fetch_access_token!`）

---

## 7. 失败处理策略

### 7.1 浏览器推送失败处理

**源码位置**：`app/services/notification/push_notification_service.rb:66-88`

```ruby
def send_browser_push(subscription)
  return unless can_send_browser_push?(subscription)

  WebPush.payload_send(**browser_push_payload(subscription))
  Rails.logger.info("Browser push sent to #{user.email} with title #{push_message[:title]}")
rescue StandardError => e
  handle_browser_push_error(e, subscription)
end

def handle_browser_push_error(error, subscription)
  case error
  when WebPush::ExpiredSubscription, WebPush::InvalidSubscription, WebPush::Unauthorized
    Rails.logger.info "WebPush subscription expired: #{error.message}"
    subscription.destroy!
  when WebPush::TooManyRequests
    Rails.logger.warn "WebPush rate limited for #{user.email} on account #{notification.account.id}: #{error.message}"
  when Errno::ECONNRESET, Net::OpenTimeout, Net::ReadTimeout, Socket::ResolutionError
    Rails.logger.error "WebPush operation error: #{error.message}"
  else
    ChatwootExceptionTracker.new(error, account: notification.account).capture_exception
    true
  end
end
```

**处理策略汇总**：

| 错误类型 | 源码位置 | 处理方式 |
|---------|---------|---------|
| `ExpiredSubscription` | `push_notification_service.rb:77-79` | 删除订阅 |
| `InvalidSubscription` | `push_notification_service.rb:77-79` | 删除订阅 |
| `Unauthorized` | `push_notification_service.rb:77-79` | 删除订阅 |
| `TooManyRequests` | `push_notification_service.rb:80-81` | 记录警告日志 |
| `ECONNRESET` | `push_notification_service.rb:82-83` | 记录错误日志 |
| `OpenTimeout` | `push_notification_service.rb:82-83` | 记录错误日志 |
| `ReadTimeout` | `push_notification_service.rb:82-83` | 记录错误日志 |
| `ResolutionError` | `push_notification_service.rb:82-83` | 记录错误日志 |
| 其他 | `push_notification_service.rb:84-86` | 异常追踪 |

### 7.2 FCM 直接发送失败处理

**源码位置**：`app/services/notification/push_notification_service.rb:118-124`

```ruby
def remove_subscription_if_error(subscription, response)
  if JSON.parse(response[:body])['results']&.first&.keys&.include?('error')
    subscription.destroy!
  else
    Rails.logger.info("FCM push sent to #{user.email} with title #{push_message[:title]}")
  end
end
```

**证据**：
- 检查 FCM 响应 `results[0]` 是否包含 `error` 字段
- 只要存在 `error`，立即删除订阅
- 没有区分不同类型的 FCM 错误

**注意**：`send_fcm_push` 方法（`push_notification_service.rb:90-100`）没有 `rescue` 块，网络异常或认证异常会向上传播。

### 7.3 Chatwoot Hub 中继失败处理

**源码位置**：`lib/chatwoot_hub.rb:108-114`

```ruby
def self.send_push(fcm_options)
  send_push_with_response(fcm_options)
rescue *ExceptionList::REST_CLIENT_EXCEPTIONS => e
  Rails.logger.error "Exception: #{e.message}"
rescue StandardError => e
  ChatwootExceptionTracker.new(e).capture_exception
end
```

**ExceptionList 定义**（`lib/exception_list.rb:1-19`）：
```ruby
module ExceptionList
  REST_CLIENT_EXCEPTIONS = [
    RestClient::NotFound, RestClient::GatewayTimeout, RestClient::BadRequest,
    RestClient::MethodNotAllowed, RestClient::Forbidden, RestClient::InternalServerError,
    RestClient::Exceptions::OpenTimeout, RestClient::Exceptions::ReadTimeout,
    RestClient::TemporaryRedirect, RestClient::SSLCertificateNotVerified, RestClient::PaymentRequired,
    RestClient::BadGateway, RestClient::Unauthorized, RestClient::PayloadTooLarge,
    RestClient::MovedPermanently, RestClient::ServiceUnavailable, Errno::ECONNREFUSED, SocketError
  ].freeze
end
```

**证据**：
- `REST_CLIENT_EXCEPTIONS` 类型异常：仅记录错误日志
- 其他 `StandardError`：上报异常追踪
- 所有异常被捕获后不重新抛出
- 不会删除订阅

---

## 8. 重试机制

### 8.1 Sidekiq 全局配置

**源码位置**：`config/sidekiq.yml:1-9`

```yaml
:verbose: false
:concurrency: <%= ENV.fetch("SIDEKIQ_CONCURRENCY", 10) %>
:timeout: 25
:max_retries: 3
```

**证据**：
- 全局 `max_retries: 3`
- Job 失败后最多重试 3 次

### 8.2 ApplicationJob 基础配置

**源码位置**：`app/jobs/application_job.rb:1-8`

```ruby
class ApplicationJob < ActiveJob::Base
  discard_on ActiveJob::DeserializationError do |job, error|
    Rails.logger.info("Skipping #{job.class} with #{
      job.instance_variable_get(:@serialized_arguments)
    } because of ActiveJob::DeserializationError (#{error.message})")
  end
end
```

**证据**：
- `ActiveJob::DeserializationError` 直接丢弃，不重试
- 没有配置 `retry_on`，依赖 Sidekiq 全局配置

### 8.3 PushNotificationJob 配置

**源码位置**：`app/jobs/notification/push_notification_job.rb:1-7`

```ruby
class Notification::PushNotificationJob < ApplicationJob
  queue_as :default

  def perform(notification)
    Notification::PushNotificationService.new(notification: notification).perform
  end
end
```

**证据**：
- 使用 `default` 队列
- 没有额外配置 `retry_on` 或 `discard_on`
- 依赖 `ApplicationJob` 的配置和 Sidekiq 全局配置

### 8.4 重试行为分析

基于源码的实际重试行为：

| 推送方式 | 异常是否被捕获 | 重试行为 |
|---------|---------------|---------|
| `send_browser_push` | ✅ 有 `rescue` 块 | ❌ 不重试（异常被处理） |
| `send_fcm_push` | ❌ 无 `rescue` 块 | ✅ Sidekiq 全局重试 3 次（网络/认证异常） |
| `send_push_via_chatwoot_hub` | ✅ ChatwootHub 内部捕获 | ❌ 不重试（异常被处理） |

**注意**：`send_fcm_push` 中 FCM 返回的业务错误（HTTP 200 但包含 `error` 字段）会触发订阅删除，但不会触发异常，因此不会重试。

---

## 9. 问题分析

### 9.1 通知设置缺失时的空对象异常

#### 9.1.1 结论

**结论**：`PushNotificationService` 和 `EmailNotificationService` 的 `user_subscribed_to_notification?` 方法在通知设置记录不存在时，会对 `nil` 调用 `public_send`，抛出 `NoMethodError`。

#### 9.1.2 源码位置

**PushNotificationService** (`app/services/notification/push_notification_service.rb:22-27`):

```ruby
def user_subscribed_to_notification?
  notification_setting = notification_settings.find_by(account_id: notification.account.id)
  return true if notification_setting.public_send("push_#{notification.notification_type}?")
  false
end
```

**EmailNotificationService** (`app/services/notification/email_notification_service.rb:26-31`):

```ruby
def user_subscribed_to_notification?
  notification_setting = notification.user.notification_settings.find_by(account_id: notification.account.id)
  return true if notification_setting.public_send("email_#{notification.notification_type}?")
  false
end
```

**通知设置创建机制** (`app/models/account_user.rb:39, 45-50`):

```ruby
after_create_commit :notify_creation, :create_notification_setting

def create_notification_setting
  setting = user.notification_settings.new(account_id: account.id)
  setting.selected_email_flags = [:email_conversation_assignment]
  setting.selected_push_flags = [:push_conversation_assignment]
  setting.save!
end
```

**Sidekiq 重试配置** (`config/sidekiq.yml:7`):

```yaml
:max_retries: 3
```

**证据**：
- 两个服务的 `user_subscribed_to_notification?` 方法都使用 `find_by` 查询通知设置
- `find_by` 找不到记录时返回 `nil`
- 返回值 `nil` 直接用于调用 `public_send`
- `AccountUser` 创建后通过 `after_create_commit` 回调创建通知设置
- Sidekiq 全局配置 `max_retries: 3`

#### 9.1.3 影响范围

| 影响对象 | 说明 |
|---------|------|
| PushNotificationJob | 异常抛出后 Job 失败，Sidekiq 重试最多 3 次，重试耗尽后进入 Dead Job 队列 |
| EmailNotificationJob | 同上，邮件通知 Job 失败 |
| 用户体验 | 目标用户缺少通知设置时，整个通知任务失败 |

---

### 9.2 订阅删除接口的越权风险

#### 9.2.1 结论

**结论**：`DELETE /api/v1/notification_subscriptions` 接口在查询订阅时只过滤 `push_token`，未限制 `user_id = current_user.id`，导致任意已认证用户可删除他人的 FCM 订阅。

#### 9.2.2 源码位置

**删除接口** (`app/controllers/api/v1/notification_subscriptions_controller.rb:10-14`):

```ruby
def destroy
  notification_subscription = NotificationSubscription.where(
    ["subscription_attributes->>'push_token' = ?", params[:push_token]]
  ).first
  notification_subscription.destroy! if notification_subscription.present?
  head :ok
end
```

**创建接口** (`app/controllers/api/v1/notification_subscriptions_controller.rb:4-8`):

```ruby
def create
  notification_subscription = NotificationSubscriptionBuilder.new(
    user: @user,  # 明确使用 current_user
    params: notification_subscription_params
  ).perform
  render json: notification_subscription
end
```

**删除接口测试** (`spec/controllers/api/v1/notification_subscriptions_controller_spec.rb:96-108`):

```ruby
it 'delete existing notification subscription if subscription exists' do
  subscription = create(:notification_subscription, 
                        subscription_type: 'fcm', 
                        subscription_attributes: { push_token: 'bUvZo8AYGGmCMr' },
                        user: agent)
  delete '/api/v1/notification_subscriptions',
         params: { push_token: subscription.subscription_attributes['push_token'] },
         headers: agent.create_new_auth_token,
         as: :json

  expect(response).to have_http_status(:success)
  expect { subscription.reload }.to raise_exception(ActiveRecord::RecordNotFound)
end
```

**证据**：
- 删除接口的 SQL 查询条件：`subscription_attributes->>'push_token' = ?`
- 查询未包含 `user_id = current_user.id` 条件
- 创建接口使用 `@user`（即 `current_user`）作为创建者
- 删除接口与创建接口的权限模型不一致
- 现有测试仅验证"用户可以删除自己的订阅"，未验证"用户不能删除他人的订阅"

#### 9.2.3 影响范围

| 影响对象 | 说明 |
|---------|------|
| 所有 FCM 类型订阅 | 任意已认证用户只要知道目标用户的 `push_token`，即可删除其订阅 |
| 接口响应 | 无论订阅是否属于当前用户，删除成功均返回 `200 OK` |
| 测试覆盖 | 现有测试用例未覆盖越权场景 |

---

## 10. 关键配置项

| 配置项 | 源码位置 | 说明 |
|--------|---------|------|
| `FIREBASE_PROJECT_ID` | `push_notification_service.rb:95` | Firebase 项目 ID |
| `FIREBASE_CREDENTIALS` | `push_notification_service.rb:95` | Firebase 服务账号凭证 JSON |
| `ENABLE_PUSH_RELAY_SERVER` | `push_notification_service.rb:115` | 是否启用 Chatwoot Hub 中继，默认 `true` |
| `SIDEKIQ_CONCURRENCY` | `config/sidekiq.yml:7` | Sidekiq 并发数，默认 10 |

---

## 11. 代码引用索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 订阅 API 控制器 | `app/controllers/api/v1/notification_subscriptions_controller.rb` | 1-25 |
| 订阅构建器 | `app/builders/notification_subscription_builder.rb` | 1-34 |
| 订阅模型 | `app/models/notification_subscription.rb` | 1-29 |
| 通知触发入口 | `app/models/notification.rb` | 155-164 |
| 推送服务主逻辑 | `app/services/notification/push_notification_service.rb` | 1-172 |
| 推送用户设置检查 | `app/services/notification/push_notification_service.rb` | 22-27 |
| 推送路由决策 | `app/services/notification/push_notification_service.rb` | 6-14 |
| 浏览器推送 | `app/services/notification/push_notification_service.rb` | 66-88 |
| FCM 直接发送 | `app/services/notification/push_notification_service.rb` | 90-100 |
| Hub 中继发送 | `app/services/notification/push_notification_service.rb` | 102-108 |
| FCM 错误处理 | `app/services/notification/push_notification_service.rb` | 118-124 |
| FCM 消息体 | `app/services/notification/push_notification_service.rb` | 126-171 |
| FCM 认证服务 | `app/services/notification/fcm_service.rb` | 1-40 |
| 邮件通知服务 | `app/services/notification/email_notification_service.rb` | 1-32 |
| 邮件用户设置检查 | `app/services/notification/email_notification_service.rb` | 26-31 |
| ChatwootHub 客户端 | `lib/chatwoot_hub.rb` | 1-133 |
| Hub 推送发送 | `lib/chatwoot_hub.rb` | 108-119 |
| 异常分类列表 | `lib/exception_list.rb` | 1-19 |
| 通知设置模型 | `app/models/notification_setting.rb` | 1-35 |
| AccountUser 关联 | `app/models/account_user.rb` | 27-86 |
| 通知设置创建回调 | `app/models/account_user.rb` | 39, 45-50 |
| 推送 Job | `app/jobs/notification/push_notification_job.rb` | 1-7 |
| 基础 Job | `app/jobs/application_job.rb` | 1-8 |
| Sidekiq 配置 | `config/sidekiq.yml` | 1-39 |
| 订阅 API 测试 | `spec/controllers/api/v1/notification_subscriptions_controller_spec.rb` | 84-110 |
