# Chatwoot 移动端推送通知分析报告

## 1. 概述

本报告分析 Chatwoot 系统中移动端推送通知的路由机制（FCM vs APNs）以及投递失败时的处理策略。

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

---

## 3. FCM 与 APNs 路由机制

### 3.1 订阅类型识别

`NotificationSubscription` 模型定义了两种订阅类型：

```ruby
SUBSCRIPTION_TYPES = {
  browser_push: 1,  # 浏览器推送
  fcm: 2            # 移动端推送（Android + iOS）
}.freeze
```

**关键发现**：系统没有单独的 `apns` 订阅类型，所有移动端推送统一归类为 `fcm` 类型。

### 3.2 FCM 统一网关策略

Chatwoot 使用 **FCM 作为统一的移动端推送平台**，通过 FCM 的 `send_v1` API 同时支持 Android 和 iOS 设备。

在 `push_notification_service.rb:126-137` 中，`fcm_options` 方法构建的消息体同时包含平台特定配置：

```ruby
def fcm_options(subscription)
  {
    'token': subscription.subscription_attributes['push_token'],
    'data': fcm_data,
    'notification': fcm_notification,
    'android': fcm_android_options,  # Android 特定配置
    'apns': fcm_apns_options,        # iOS 特定配置（通过 FCM 中继到 APNs）
    'fcm_options': {
      analytics_label: 'Label'
    }
  }
end
```

### 3.3 平台特定配置

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

### 3.4 推送路由决策树

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

### 3.5 FCM 推送的双重路由

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

## 4. 投递失败处理策略

### 4.1 浏览器推送失败处理

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

### 4.2 FCM 推送失败处理

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
- `InvalidRegistration` / `NotRegistered` - token 无效或已注销
- `MismatchSenderId` - 发送者 ID 不匹配
- `InvalidApnsCredential` - APNs 凭证无效（iOS 特定）
- `QuotaExceeded` / `DeviceMessageRateExceeded` - 限流
- `InternalServerError` / `Unavailable` - 服务端错误

**潜在问题**：
- 当前实现将所有 FCM 错误视为不可恢复，直接删除订阅
- 对于临时性错误（如限流、服务端不可用），这会导致用户订阅被误删

### 4.3 Chatwoot Hub 中继失败处理

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
- **没有重试机制**

### 4.4 FCM 服务认证失败处理

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
- 最终异常会由 Sidekiq 的异常处理机制捕获

---

## 5. 推送通知生命周期

### 5.1 通知创建流程

```ruby
# notification.rb:155-164
def process_notification_delivery
  Notification::PushNotificationJob.perform_later(self) if user_subscribed_to_notification?('push')
  
  Notification::EmailNotificationJob.perform_later(self) if user_subscribed_to_notification?('email')
  
  Notification::RemoveDuplicateNotificationJob.perform_later(self)
end
```

### 5.2 用户通知设置检查

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

## 6. 关键配置项

| 配置项 | 环境变量 | 用途 |
|--------|---------|------|
| Firebase Project ID | `FIREBASE_PROJECT_ID` | 直接 FCM 发送必需 |
| Firebase Credentials | `FIREBASE_CREDENTIALS` | 直接 FCM 发送必需（JSON 字符串） |
| Push Relay Server | `ENABLE_PUSH_RELAY_SERVER` | 是否启用 Chatwoot Hub 中继（默认 true） |
| VAPID Public Key | （通过 VapidService） | 浏览器推送必需 |
| Chatwoot Hub URL | `CHATWOOT_HUB_URL`（企业版开发环境） | 中继服务器地址 |

---

## 7. 总结与建议

### 7.1 架构特点

1. **FCM 统一网关**：通过 FCM 同时处理 Android 和 iOS，简化了后端架构
2. **双重路由**：支持自建 Firebase 或使用 Chatwoot 官方中继服务
3. **异步处理**：所有推送通过 Sidekiq 异步执行，不阻塞主流程

### 7.2 失败处理策略总结

| 推送类型 | 失败处理策略 | 订阅清理 |
|---------|-------------|---------|
| 浏览器推送 | 分类处理：过期/无效订阅删除，限流/网络错误仅记录 | ✅ 智能清理 |
| FCM 直接发送 | 任何 FCM 错误都删除订阅 | ⚠️ 可能误删 |
| Chatwoot Hub 中继 | 仅记录日志/上报异常 | ❌ 不清理 |

### 7.3 潜在改进点

1. **FCM 错误精细化处理**：
   - 当前实现将所有 FCM 错误视为不可恢复
   - 建议区分 `NotRegistered`（应删除）与 `Unavailable`/`QuotaExceeded`（应重试）

2. **重试机制**：
   - 对于临时性错误（网络超时、服务端不可用、限流），应实现指数退避重试
   - 可考虑使用 Sidekiq 的重试机制

3. **订阅状态追踪**：
   - 增加订阅失败计数器
   - 连续多次失败后再删除订阅，避免单次临时性错误导致订阅丢失

4. **监控和告警**：
   - 推送失败率监控
   - FCM/APNs 服务健康检查
   - 订阅清理频率监控

---

## 8. 代码引用索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 推送路由主逻辑 | `app/services/notification/push_notification_service.rb` | 6-14 |
| FCM 直接发送 | `app/services/notification/push_notification_service.rb` | 90-100 |
| Chatwoot Hub 中继 | `app/services/notification/push_notification_service.rb` | 102-108 |
| FCM 错误处理 | `app/services/notification/push_notification_service.rb` | 118-124 |
| 浏览器推送错误处理 | `app/services/notification/push_notification_service.rb` | 75-88 |
| APNs 配置（通过 FCM） | `app/services/notification/push_notification_service.rb` | 162-171 |
| 订阅类型定义 | `app/models/notification_subscription.rb` | 23-28 |
| FCM 认证服务 | `app/services/notification/fcm_service.rb` | 1-40 |
| Chatwoot Hub 客户端 | `lib/chatwoot_hub.rb` | 108-119 |
| 通知触发入口 | `app/models/notification.rb` | 155-164 |
