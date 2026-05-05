# Chatwoot 多渠道入站消息处理机制分析报告

## 1. 概述

Chatwoot 作为一个多渠道客户支持平台，支持接收来自 WhatsApp、Facebook Messenger、Twilio (SMS/WhatsApp)、Instagram、Telegram、Line 等多种渠道的消息。本报告深入分析 Chatwoot 如何将这些不同渠道的入站 webhook 消息归一化为统一的对话模型，以及各渠道的校验机制和联系人身份合并策略。

### 核心架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                    外部渠道 Webhook 入口                       │
│  WhatsApp │ Facebook Messenger │ Twilio │ Instagram │ ...   │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                   Webhook 控制器层                            │
│  WhatsappController │ InstagramController                    │
│  (Facebook 使用 facebook-messenger gem 挂载)                 │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    异步 Job 处理层                            │
│  WhatsappEventsJob │ FacebookEventsJob │ TwilioEventsJob   │
│  (带分布式锁机制)                                             │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                   渠道特定消息处理服务                         │
│  Whatsapp::IncomingMessageService                            │
│  Twilio::IncomingMessageService                              │
│  Messages::Facebook::MessageBuilder                          │
└───────────────────────┬─────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              统一模型层 (归一化核心)                           │
│  ContactInboxWithContactBuilder → Contact + ContactInbox    │
│  ConversationBuilder → Conversation                          │
│  MessageBuilder → Message + Attachment                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 各渠道入站消息处理流程分析

### 2.1 WhatsApp 渠道

#### 2.1.1 Webhook 入口

**文件位置**: `app/controllers/webhooks/whatsapp_controller.rb`

WhatsApp 渠道支持两种提供商：
- WhatsApp Cloud API (Meta 官方)
- 360Dialog (第三方提供商)

**路由配置** (`config/routes.rb:583-584`):
```ruby
get  'webhooks/whatsapp/:phone_number', to: 'webhooks/whatsapp#verify'
post 'webhooks/whatsapp/:phone_number', to: 'webhooks/whatsapp#process_payload'
```

**处理流程**:
1. 接收 webhook 请求，URL 路径中包含 `phone_number` 参数标识具体的 WhatsApp 号码
2. 校验是否为 inactive 号码（通过 `INACTIVE_WHATSAPP_NUMBERS` 全局配置）
3. 将参数异步投递到 `Webhooks::WhatsappEventsJob`
4. 立即返回 `200 OK`

#### 2.1.2 Job 层处理 (带锁机制)

**文件位置**: `app/jobs/webhooks/whatsapp_events_job.rb`

```ruby
class Webhooks::WhatsappEventsJob < MutexApplicationJob
  retry_on LockAcquisitionError, wait: 2.seconds, attempts: 20

  def perform(params = {})
    # ... 渠道查找逻辑 ...
    
    key = format(::Redis::Alfred::WHATSAPP_MESSAGE_MUTEX, 
                 inbox_id: channel.inbox.id, sender_id: sender_id)
    with_lock(key, 30.seconds) do  # 对话级锁，30秒超时
      process_events(channel, params)
    end
  end
end
```

#### 2.1.3 消息解析与处理

**文件位置**: 
- `app/services/whatsapp/incoming_message_base_service.rb` (基类)
- `app/services/whatsapp/incoming_message_service.rb` (360Dialog)
- `app/services/whatsapp/incoming_message_whatsapp_cloud_service.rb` (WhatsApp Cloud)

**核心处理步骤**:

```ruby
# 来自 incoming_message_base_service.rb:9-17
def perform
  processed_params  # 标准化不同提供商的参数格式
  
  if processed_params.try(:[], :statuses).present?
    process_statuses  # 处理消息状态更新 (sent/delivered/read/failed)
  elsif messages_data.present?
    process_messages   # 处理实际消息内容
  end
end
```

**消息类型过滤**:
- 跳过 `reaction` (消息回应)
- 跳过 `ephemeral` (阅后即焚消息)
- 跳过 `unsupported` 等不支持的消息类型

#### 2.1.4 WhatsApp 消息去重机制 (三层防护)

**⚠️ 重要修正**: 消息去重**不是**通过数据库唯一约束实现的！

WhatsApp 采用**三层去重策略**，是所有渠道中最完整的：

```
┌─────────────────────────────────────────────────────────────────────┐
│  第一层: Job 级对话锁 (防止并发创建重复对话)                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 锁 Key: WHATSAPP_MESSAGE_MUTEX::<inbox_id>::<sender_id>     │   │
│  │ 实现: Redis::LockManager (with_lock)                         │   │
│  │ 超时: 30 秒                                                   │   │
│  │ 目的: 防止同一用户在同一 inbox 的并发消息创建重复对话          │   │
│  │ 场景: 相册上传 (多张图片并发投递)                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  第二层: 消息级数据库查询检查 (防止重复处理已入库的消息)              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 实现: Message.find_by(source_id: message_id)                 │   │
│  │ 注意: 只是普通查询，source_id 没有唯一约束！                   │   │
│  │ 索引: index_messages_on_source_id (普通索引，非唯一)          │   │
│  │ 目的: 快速检查该消息是否已被处理过                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  第三层: 消息级 Redis 原子锁 (防止并发 workers 重复处理同一条消息)   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 锁 Key: MESSAGE_SOURCE_KEY::<message_id>                     │   │
│  │ 实现: Redis SET NX EX (原子操作)                              │   │
│  │ 类: Whatsapp::MessageDedupLock                               │   │
│  │ TTL: 1 天 (DEFAULT_TTL = 1.day.to_i)                        │   │
│  │ 目的: Meta 可能多次投递同一个 webhook，用 Redis 锁保证只处理一次│   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**代码实现** (`app/services/whatsapp/incoming_message_base_service.rb:35-36`):
```ruby
def process_messages
  # ... 类型过滤 ...
  
  # 第二层: 数据库查询检查
  return if find_message_by_source_id(messages_data.first[:id])
  
  # 第三层: Redis 原子锁 (真正的防并发核心)
  return unless lock_message_source_id!
  
  # ... 后续处理 ...
end
```

**Redis 锁实现** (`app/services/whatsapp/message_dedup_lock.rb`):
```ruby
class Whatsapp::MessageDedupLock
  KEY_PREFIX = Redis::RedisKeys::MESSAGE_SOURCE_KEY  # "MESSAGE_SOURCE_KEY::%<id>s"
  DEFAULT_TTL = 1.day.to_i

  def acquire!
    # SET NX = 只在 key 不存在时设置
    # SET EX = 设置过期时间
    # 这是一个原子操作，保证只有一个 worker 能获取锁
    ::Redis::Alfred.set(@key, true, nx: true, ex: @ttl)
  end
end
```

**数据库检查实现** (`app/services/whatsapp/incoming_message_service_helpers.rb:66-76`):
```ruby
def find_message_by_source_id(source_id)
  return unless source_id
  # 只是普通的 find_by 查询，没有唯一约束！
  @message = Message.find_by(source_id: source_id)
end

def lock_message_source_id!
  return false if messages_data.blank?
  # 真正的去重核心：Redis SET NX 原子锁
  Whatsapp::MessageDedupLock.new(messages_data.first[:id]).acquire!
end
```

#### 2.1.5 支持的消息类型

| 消息类型 | 处理方式 | 目标字段 |
|---------|---------|---------|
| `text` | 文本消息 | `message.content` |
| `image` | 图片附件 | `Attachment` + 可选 caption |
| `audio` | 音频附件 | `Attachment` |
| `video` | 视频附件 | `Attachment` |
| `document` | 文档附件 | `Attachment` |
| `location` | 位置消息 | `Attachment` (坐标存储) |
| `contacts` | 联系人卡片 | 多条 `Attachment` |
| `button` | 按钮回复 | `message.content` |
| `interactive` | 交互消息 (列表/模板) | `message.content` |
| `statuses` | 状态更新 | 更新 `message.status` |

---

### 2.2 Facebook Messenger 渠道 (⚠️ 之前报告遗漏的完整链路)

#### 2.2.1 Webhook 入口与挂载方式

**⚠️ 重要修正**: Facebook Messenger **不使用** `MetaTokenVerifyConcern`，也没有独立的控制器！

**实际实现**: 使用 `facebook-messenger` Ruby gem，直接挂载到 Rails 路由。

**路由配置** (`config/routes.rb:577`):
```ruby
mount Facebook::Messenger::Server, at: 'bot'
```

这意味着：
- Webhook URL: `https://your-server/bot`
- 所有请求由 `facebook-messenger` gem 的 `Server` 类处理
- Chatwoot 通过 gem 提供的回调机制注册处理器

#### 2.2.2 订阅机制 (自动 webhook 配置)

**文件位置**: `app/models/channel/facebook_page.rb:49-60`

当创建 `Channel::FacebookPage` 记录时，**自动**调用 `subscribe` 方法：

```ruby
class Channel::FacebookPage < ApplicationRecord
  after_create_commit :subscribe  # 创建后自动订阅
  before_destroy :unsubscribe     # 删除前取消订阅

  def subscribe
    # ref https://developers.facebook.com/docs/messenger-platform/reference/webhook-events
    Facebook::Messenger::Subscriptions.subscribe(
      access_token: page_access_token,
      subscribed_fields: %w[
        messages           # 普通消息
        message_deliveries # 送达状态
        message_echoes     # 回声消息 (从 FB 页面直接发送的)
        message_reads      # 已读状态
        standby            # 备用模式 (handover protocol)
        messaging_handovers # 交接协议
      ]
    )
  end
end
```

#### 2.2.3 Webhook 校验链路 (由 facebook-messenger gem 处理)

Facebook Messenger 的 webhook 校验分为**两个阶段**：

**阶段 1: 订阅时的 Hub Challenge 验证**

当你在 Facebook Developer Console 配置 webhook URL 时，Facebook 会发送一个 GET 请求：

```
GET /bot?hub.mode=subscribe&hub.challenge=CHALLENGE_CODE&hub.verify_token=YOUR_TOKEN
```

`facebook-messenger` gem 会：
1. 检查 `hub.verify_token` 是否匹配配置的 `verify_token`
2. 如果匹配，返回 `hub.challenge` 的值
3. 如果不匹配，返回错误

**Chatwoot 配置**：
- Verify Token 通过 `FB_VERIFY_TOKEN` 环境变量配置
- App Secret 通过 `FB_APP_SECRET` 环境变量配置（用于 HMAC 签名验证）

**阶段 2: 实际事件的 HMAC 签名验证**

每个 POST 请求（实际消息事件）都会包含 `X-Hub-Signature` 头：

```
X-Hub-Signature: sha1=HMAC_HEX_VALUE
```

`facebook-messenger` gem 会：
1. 使用 `FB_APP_SECRET` 计算请求 body 的 HMAC-SHA1 签名
2. 与 `X-Hub-Signature` 头中的值比较
3. 如果不匹配，拒绝请求

**⚠️ 关键点**: 
- WhatsApp 和 Instagram 使用 `MetaTokenVerifyConcern`（Chatwoot 自己实现的）
- Facebook Messenger 使用 `facebook-messenger` gem（第三方实现）
- 这就是为什么之前的报告"遗漏"了 Facebook Messenger 的校验链路——它根本不在 Chatwoot 的控制器代码里！

#### 2.2.4 事件处理流程

**文件位置**: `app/jobs/webhooks/facebook_events_job.rb`

```ruby
class Webhooks::FacebookEventsJob < MutexApplicationJob
  retry_on LockAcquisitionError, wait: 1.second, attempts: 8

  def perform(message)
    response = ::Integrations::Facebook::MessageParser.new(message)

    # 对话级锁（注意：不是消息级去重！）
    key = format(::Redis::Alfred::FACEBOOK_MESSAGE_MUTEX, 
                 sender_id: response.sender_id, 
                 recipient_id: response.recipient_id)
    with_lock(key) do
      process_message(response)
    end
  end
end
```

#### 2.2.5 Facebook Messenger 去重机制分析

**⚠️ 重要发现**: Facebook Messenger **只有对话级锁，没有消息级去重！**

| 去重层级 | WhatsApp | Facebook Messenger |
|---------|----------|-------------------|
| 对话级 Redis 锁 | ✅ 有 | ✅ 有 |
| 消息级 DB 查询 | ✅ 有 (`find_by`) | ❌ 无 |
| 消息级 Redis 锁 | ✅ 有 (`SET NX`) | ❌ 无 |

**锁 Key 对比**:
- WhatsApp 消息锁: `MESSAGE_SOURCE_KEY::<message_id>` (按消息 ID)
- Facebook 对话锁: `FB_MESSAGE_CREATE_LOCK::<sender_id>::<recipient_id>` (按对话双方)

**这意味着**:
- Facebook Messenger 只能防止**同一对话的并发处理**（避免创建重复对话）
- 但如果**同一条消息被 Facebook 多次投递**，可能会创建**重复消息**！
- 这是 WhatsApp 比 Facebook Messenger 更健壮的地方

#### 2.2.6 消息解析器

**文件位置**: `lib/integrations/facebook/message_parser.rb`

解析的关键字段：
- `sender_id` - 发送者 PSID (Page Scoped ID)
- `recipient_id` - 接收者 Page ID
- `content` - 消息文本 (`message.text`)
- `attachments` - 附件数组 (图片/视频/音频/位置/文件)
- `identifier` - 消息 ID (`message.mid`)
- `echo?` - 是否为 echo 消息（从 Facebook 页面直接发送的消息）
- `in_reply_to_external_id` - 回复的消息 ID

#### 2.2.7 消息构建器

**文件位置**: `app/builders/messages/facebook/message_builder.rb`

**核心特点**:
1. 区分 `echo` 消息（从 Facebook 页面发送的消息）和普通用户消息
2. 调用 Facebook Graph API 获取用户 profile 信息（姓名、头像）
3. 处理 `standby` 模式消息（当 handover protocol 控制时）

**Echo 消息处理逻辑**:
```ruby
# 来自 message_creator.rb:24-29
def agent_message_via_echo?
  # 判断是否为从 Facebook 页面直接发送的消息（非 Chatwoot 发送）
  response.echo? && !response.sent_from_chatwoot_app?
end
```

---

### 2.3 Twilio 渠道

Twilio 渠道支持两种媒介：
- **SMS** - 短信
- **WhatsApp** - 通过 Twilio 的 WhatsApp Business API

#### 2.3.1 Webhook 入口

**文件位置**: `app/controllers/webhooks/sms_controller.rb`

```ruby
def process_payload
  Webhooks::SmsEventsJob.perform_later(params['_json']&.first&.to_unsafe_hash)
  head :ok
end
```

#### 2.3.2 Job 层处理 (⚠️ 无锁机制)

**文件位置**: `app/jobs/webhooks/twilio_events_job.rb`

```ruby
class Webhooks::TwilioEventsJob < ApplicationJob
  queue_as :low

  def perform(params = {})
    # ⚠️ 注意：没有继承 MutexApplicationJob！
    # ⚠️ 没有 with_lock 调用！
    
    # 直接调用服务，无任何锁保护
    ::Twilio::IncomingMessageService.new(params: params).perform
  end
end
```

#### 2.3.3 消息处理服务

**文件位置**: `app/services/twilio/incoming_message_service.rb`

**渠道识别**:
```ruby
def twilio_channel
  # 优先通过 Messaging Service SID 查找
  @twilio_channel ||= ::Channel::TwilioSms.find_by(messaging_service_sid: params[:MessagingServiceSid])
  # 其次通过 Account SID + To 号码查找
  if params[:AccountSid].present? && params[:To].present?
    @twilio_channel ||= ::Channel::TwilioSms.find_by(account_sid: params[:AccountSid], phone_number: params[:To])
  end
end
```

#### 2.3.4 Twilio 去重机制分析 (⚠️ 完全没有去重!)

**让我们验证一下**：

```ruby
# 在 Twilio::IncomingMessageService 中搜索:
# - find_message_by_source_id? ❌ 没有
# - lock_message_source_id?   ❌ 没有
# - 任何形式的去重检查?       ❌ 没有

# 消息创建代码 (第 11-21 行):
def perform
  return if twilio_channel.blank?

  set_contact
  set_conversation
  @message = @conversation.messages.build(
    content: message_body,
    account_id: @inbox.account_id,
    inbox_id: @inbox.id,
    message_type: :incoming,
    sender: @contact,
    source_id: params[:SmsSid]  # 只是保存，没有去重检查！
  )
  attach_files
  attach_location if location_message?
  @message.save!
end
```

**各渠道去重能力对比**:

| 渠道 | 对话级锁 | 消息级 DB 检查 | 消息级 Redis 锁 | 去重完整性 |
|-----|---------|---------------|----------------|-----------|
| WhatsApp | ✅ | ✅ | ✅ | 完整 |
| Facebook Messenger | ✅ | ❌ | ❌ | 仅对话级 |
| Instagram | ✅ | ❌ | ❌ | 仅对话级 |
| TikTok | ✅ | ❌ | ❌ | 仅对话级 |
| Twilio | ❌ | ❌ | ❌ | 无 |
| SMS (其他) | ❌ | ❌ | ❌ | 无 |

#### 2.3.5 支持的消息类型

- **文本消息**: `params[:Body]`
- **媒体附件**: 通过 `NumMedia` 和 `MediaUrl{N}` 处理
- **位置消息**: 通过 `MessageType=location` + `Latitude`/`Longitude` 处理
- **Profile Name**: WhatsApp 用户的显示名称 (`params[:ProfileName]`)

**SMS 附加属性**:
- `FromZip` - 发件人邮编
- `FromCountry` - 发件人国家
- `FromState` - 发件人省份/州

---

### 2.4 Instagram 渠道 (简要对比)

**路由配置** (`config/routes.rb:585-586`):
```ruby
get  'webhooks/instagram', to: 'webhooks/instagram#verify'
post 'webhooks/instagram', to: 'webhooks/instagram#events'
```

**校验机制**: 使用 `MetaTokenVerifyConcern`（与 WhatsApp 相同的模式）

**去重机制**: 
- 对话级锁: `IG_MESSAGE_MUTEX::<sender_id>::<ig_account_id>`
- 消息级去重: ❌ 无（与 Facebook Messenger 相同）

---

## 3. 消息去重机制深度分析 (修正版)

### 3.1 核心概念澄清

**之前报告的错误**:
- ❌ 错误描述: "数据库层: `Message.source_id` 唯一索引"
- ✅ 实际情况: `Message.source_id` 只是**普通索引**，不是唯一约束！

**Schema 验证** (来自 `app/models/message.rb`):
```ruby
# Indexes
# ...
#  index_messages_on_source_id                          (source_id)  ← 普通索引！
# ...
```

注意：没有 `UNIQUE` 关键字！

### 3.2 各渠道去重实现详解

#### 3.2.1 WhatsApp (最完整的三层去重)

```
Webhook 到达
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: 对话级锁 (WhatsappEventsJob)                         │
│ 目的: 防止同一用户的并发消息创建重复对话                        │
│ 场景: 相册上传 (多张图片同时投递)                               │
│ 实现: Redis::LockManager#with_lock                            │
│ Key: WHATSAPP_MESSAGE_MUTEX::<inbox_id>::<sender_id>         │
│ TTL: 30 秒                                                     │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 2: 消息级数据库检查 (IncomingMessageBaseService)         │
│ 目的: 快速检查该消息是否已被处理过                              │
│ 实现: Message.find_by(source_id: message_id)                  │
│ 注意: 只是查询，不是唯一约束！                                   │
│ 返回: 已存在 → 跳过处理                                         │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 3: 消息级 Redis 原子锁 (MessageDedupLock)               │
│ 目的: 防止多个并发 workers 同时处理同一条消息                   │
│ 场景: Meta 多次投递同一个 webhook                              │
│ 实现: Redis SET NX EX (原子操作)                               │
│ Key: MESSAGE_SOURCE_KEY::<message_id>                          │
│ TTL: 1 天                                                       │
│ 返回: 锁获取失败 → 跳过处理                                      │
└──────────────────────────────────────────────────────────────┘
```

**关键代码位置**:
- Layer 1: `app/jobs/webhooks/whatsapp_events_job.rb:23-26`
- Layer 2: `app/services/whatsapp/incoming_message_service_helpers.rb:66-70`
- Layer 3: `app/services/whatsapp/incoming_message_service_helpers.rb:72-76`
- Lock 实现: `app/services/whatsapp/message_dedup_lock.rb`

#### 3.2.2 Facebook Messenger / Instagram / TikTok (仅对话级锁)

```
Webhook 到达
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: 对话级锁                                             │
│ 目的: 防止同一对话的并发处理                                   │
│ 实现: Redis::LockManager#with_lock                            │
│                                                                │
│ 各渠道 Key 格式:                                               │
│ - Facebook: FB_MESSAGE_CREATE_LOCK::<sender_id>::<recipient_id>│
│ - Instagram: IG_MESSAGE_CREATE_LOCK::<sender_id>::<ig_account_id>│
│ - TikTok: TIKTOK_MESSAGE_CREATE_LOCK::<business_id>::<conversation_id>│
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼ (直接创建消息，无消息级检查)
┌──────────────────────────────────────────────────────────────┐
│ ⚠️ 风险: 如果同一条消息被多次投递，会创建重复消息！             │
│ 原因: 没有 find_by(source_id) 检查，也没有 Redis 消息锁        │
└──────────────────────────────────────────────────────────────┘
```

**关键代码位置**:
- Facebook: `app/jobs/webhooks/facebook_events_job.rb:8-11`
- Instagram: `app/jobs/webhooks/instagram_events_job.rb:11-14`
- TikTok: `app/jobs/webhooks/tiktok_events_job.rb:13-16`

#### 3.2.3 Twilio / SMS (完全无锁)

```
Webhook 到达
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ ⚠️ 无任何锁机制！直接创建消息                                   │
│                                                                │
│ 风险:                                                          │
│ 1. 同一对话的并发消息可能创建重复对话                           │
│ 2. 同一条消息的多次投递会创建重复消息                           │
└──────────────────────────────────────────────────────────────┘
```

**关键代码位置**:
- Twilio: `app/jobs/webhooks/twilio_events_job.rb` (无锁)
- SMS: `app/services/sms/incoming_message_service.rb` (无去重检查)

### 3.3 去重机制对比总结

| 维度 | WhatsApp | Facebook/Instagram/TikTok | Twilio/SMS |
|-----|----------|--------------------------|------------|
| **对话级锁** | ✅ | ✅ | ❌ |
| **消息级 DB 检查** | ✅ (`find_by`) | ❌ | ❌ |
| **消息级 Redis 锁** | ✅ (`SET NX`) | ❌ | ❌ |
| **防止重复对话** | ✅ | ✅ | ❌ (风险) |
| **防止重复消息** | ✅ | ❌ (风险) | ❌ (风险) |
| **去重完整性** | 完整 | 部分 | 无 |

### 3.4 Redis 锁 Key 定义汇总

**文件位置**: `lib/redis/redis_keys.rb`

```ruby
module Redis::RedisKeys
  # 消息去重锁 (仅 WhatsApp 使用)
  MESSAGE_SOURCE_KEY = 'MESSAGE_SOURCE_KEY::%<id>s'.freeze
  
  # 对话级锁 (各渠道)
  FACEBOOK_MESSAGE_MUTEX = 'FB_MESSAGE_CREATE_LOCK::%<sender_id>s::%<recipient_id>s'.freeze
  IG_MESSAGE_MUTEX = 'IG_MESSAGE_CREATE_LOCK::%<sender_id>s::%<ig_account_id>s'.freeze
  TIKTOK_MESSAGE_MUTEX = 'TIKTOK_MESSAGE_CREATE_LOCK::%<business_id>s::%<conversation_id>s'.freeze
  WHATSAPP_MESSAGE_MUTEX = 'WHATSAPP_MESSAGE_CREATE_LOCK::%<inbox_id>s::%<sender_id>s'.freeze
  SLACK_MESSAGE_MUTEX = 'SLACK_MESSAGE_LOCK::%<conversation_id>s::%<reference_id>s'.freeze
  EMAIL_MESSAGE_MUTEX = 'EMAIL_CHANNEL_LOCK::%<inbox_id>s'.freeze
end
```

---

## 4. Webhook 校验机制 (修正版)

### 4.1 校验机制分类

Chatwoot 的 webhook 校验分为**三种模式**:

| 模式 | 实现方式 | 使用渠道 |
|-----|---------|---------|
| **模式 A**: Chatwoot 自定义控制器 + `MetaTokenVerifyConcern` | 自己实现 Hub Challenge | WhatsApp, Instagram |
| **模式 B**: `facebook-messenger` gem 挂载 | 第三方 gem 处理校验和路由 | Facebook Messenger |
| **模式 C**: 无显式校验 / 隐式校验 | 通过参数查找渠道 | Twilio, SMS |

### 4.2 模式 A: MetaTokenVerifyConcern (WhatsApp / Instagram)

#### 4.2.1 验证 Concern 实现

**文件位置**: `app/controllers/concerns/meta_token_verify_concern.rb`

```ruby
module MetaTokenVerifyConcern
  def verify
    service = is_a?(Webhooks::WhatsappController) ? 'whatsapp' : 'instagram'
    if valid_token?(params['hub.verify_token'])
      Rails.logger.info("#{service.capitalize} webhook verified")
      render json: params['hub.challenge']  # 返回 challenge 完成验证
    else
      render status: :unauthorized, json: { error: 'Error; wrong verify token' }
    end
  end
end
```

#### 4.2.2 WhatsApp 令牌验证

**文件位置**: `app/controllers/webhooks/whatsapp_controller.rb:17-21`

```ruby
def valid_token?(token)
  channel = Channel::Whatsapp.find_by(phone_number: params[:phone_number])
  whatsapp_webhook_verify_token = channel.provider_config['webhook_verify_token'] if channel.present?
  token == whatsapp_webhook_verify_token if whatsapp_webhook_verify_token.present?
end
```

**特点**: 
- 每个 WhatsApp 号码 (`phone_number`) 有独立的 verify token
- Token 存储在 `Channel::Whatsapp.provider_config['webhook_verify_token']`
- URL 路径中包含 `phone_number`，用于定位具体的 channel

#### 4.2.3 Instagram 令牌验证

**文件位置**: `app/controllers/webhooks/instagram_controller.rb:36-41`

```ruby
def valid_token?(token)
  # 支持两种验证令牌:
  # 1. IG_VERIFY_TOKEN - 通过 Facebook Page 连接的 Instagram
  # 2. INSTAGRAM_VERIFY_TOKEN - 直接 Instagram 登录连接
  token == GlobalConfigService.load('IG_VERIFY_TOKEN', '') ||
    token == GlobalConfigService.load('INSTAGRAM_VERIFY_TOKEN', '')
end
```

**特点**:
- 使用全局配置而非 per-channel 配置
- 支持两种令牌以兼容不同的集成方式

### 4.3 模式 B: facebook-messenger Gem (Facebook Messenger)

#### 4.3.1 完整校验链路 (之前报告遗漏)

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 1: 订阅验证 (Hub Challenge)                                          │
│                                                                           │
│ Facebook Developer Console                                                │
│         │                                                                 │
│         ▼ GET /bot?hub.mode=subscribe&hub.challenge=XXX&hub.verify_token=YYY│
│         │                                                                 │
│         ▼                                                                 │
│ ┌─────────────────────────────────────────────────────────────────┐     │
│ │ facebook-messenger gem 的 Server 类                              │     │
│ │ 1. 解析 hub.verify_token 参数                                     │     │
│ │ 2. 与配置的 FB_VERIFY_TOKEN 比较                                  │     │
│ │ 3. 匹配 → 返回 hub.challenge                                      │     │
│ │ 4. 不匹配 → 返回 403 Forbidden                                    │     │
│ └─────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 阶段 2: 事件签名验证 (HMAC-SHA1)                                          │
│                                                                           │
│ 每个实际事件 POST 请求都携带:                                             │
│   Header: X-Hub-Signature: sha1=<HMAC_HEX>                              │
│                                                                           │
│ facebook-messenger gem 的验证流程:                                       │
│ 1. 获取请求原始 body                                                      │
│ 2. 使用 FB_APP_SECRET 作为 key，计算 HMAC-SHA1 签名                      │
│ 3. 与 X-Hub-Signature 头中的值比较                                       │
│ 4. 匹配 → 继续处理                                                        │
│ 5. 不匹配 → 返回 403 Forbidden                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 Chatwoot 的配置方式

**环境变量**:
- `FB_VERIFY_TOKEN` - 用于 Hub Challenge 验证
- `FB_APP_SECRET` - 用于 HMAC 签名验证
- `FB_APP_ID` - 用于判断 echo 消息来源

**自动订阅机制** (`app/models/channel/facebook_page.rb:49-60`):
```ruby
def subscribe
  Facebook::Messenger::Subscriptions.subscribe(
    access_token: page_access_token,
    subscribed_fields: %w[
      messages message_deliveries message_echoes 
      message_reads standby messaging_handovers
    ]
  )
end
```

### 4.4 模式 C: 隐式校验 (Twilio / SMS)

#### 4.4.1 当前实现分析

通过代码搜索 (`X-Twilio-Signature`, `validate.*signature`)，Chatwoot 当前**没有显式实现 Twilio 请求签名验证**。

**Twilio 官方推荐的验证方式**:
1. **HTTP Basic Auth**: Twilio 支持在 webhook URL 中包含用户名密码
2. **请求签名验证**: 通过 `X-Twilio-Signature` 头 + `auth_token` 计算 HMAC-SHA1

**当前实现的替代验证方式**:
```ruby
# 来自 app/services/twilio/incoming_message_service.rb:26-34
def twilio_channel
  # 通过以下组合查找渠道，作为隐式验证:
  # 1. MessagingServiceSid (如果存在)
  # 2. AccountSid + To 号码组合
  @twilio_channel ||= ::Channel::TwilioSms.find_by(messaging_service_sid: params[:MessagingServiceSid])
  if params[:AccountSid].present? && params[:To].present?
    @twilio_channel ||= ::Channel::TwilioSms.find_by(account_sid: params[:AccountSid], phone_number: params[:To])
  end
end
```

**⚠️ 安全风险**: 这意味着任何知道 `AccountSid` 和 `To` 号码的人都可以伪造 webhook 请求。生产环境建议增强安全性。

### 4.5 校验机制汇总 (修正版)

| 渠道 | 校验模式 | 验证方式 | Token 存储 |
|-----|---------|---------|-----------|
| WhatsApp | **模式 A** | Hub Challenge (per-channel) | `Channel::Whatsapp.provider_config` |
| Instagram | **模式 A** | Hub Challenge (全局) | `IG_VERIFY_TOKEN` / `INSTAGRAM_VERIFY_TOKEN` |
| Facebook Messenger | **模式 B** | Hub Challenge + HMAC-SHA1 签名 (gem 处理) | `FB_VERIFY_TOKEN` / `FB_APP_SECRET` |
| Twilio | **模式 C** | 隐式验证 (通过 AccountSid + To 查找 channel) | 无显式签名验证 |
| SMS (其他) | **模式 C** | 类似 Twilio | 无显式签名验证 |
| Telegram | 路径中包含 Bot Token | URL 路径: `webhooks/telegram/:bot_token` | 路径参数验证 |
| Line | 签名验证 (HMAC-SHA256) | `X-Line-Signature` 头 | Channel Secret |

### 4.6 Facebook Messenger vs WhatsApp 校验对比

| 维度 | WhatsApp | Facebook Messenger |
|-----|----------|-------------------|
| **控制器** | `Webhooks::WhatsappController` (自定义) | 无 (gem 挂载) |
| **Concern** | ✅ 使用 `MetaTokenVerifyConcern` | ❌ 不使用 |
| **路由** | `webhooks/whatsapp/:phone_number` | `mount ... at: 'bot'` |
| **Hub Challenge** | ✅ 自己实现 | ✅ gem 处理 |
| **HMAC 签名验证** | ❌ 无 | ✅ gem 处理 |
| **Per-channel Token** | ✅ 每个号码独立 | ❌ 全局配置 |

---

## 5. 消息归一化到统一对话模型

### 5.1 核心数据模型

#### 5.1.1 模型关系图

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────┐
│   Contact    │◄──────│   ContactInbox   │──────►│    Inbox     │
│  (统一联系人) │       │  (渠道关联层)     │       │  (渠道入口)   │
└──────────────┘       └──────────────────┘       └──────────────┘
       │                        │
       │                        │
       │                        ▼
       │               ┌──────────────────┐
       │               │  Conversation    │
       │               │   (对话)         │
       │               └──────────────────┘
       │                        │
       │                        ▼
       │               ┌──────────────────┐
       └──────────────►│     Message      │
                       │    (消息)         │
                       └──────────────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │    Attachment    │
                       │    (附件)         │
                       └──────────────────┘
```

#### 5.1.2 关键模型说明

| 模型 | 文件位置 | 核心作用 |
|-----|---------|---------|
| `Contact` | `app/models/contact.rb` | 统一的联系人实体，跨渠道共享 |
| `ContactInbox` | `app/models/contact_inbox.rb` | 联系人与特定渠道 Inbox 的关联，存储渠道特定的 source_id |
| `Inbox` | `app/models/inbox.rb` | 渠道入口，关联特定的 Channel (如 `Channel::Whatsapp`) |
| `Conversation` | `app/models/conversation.rb` | 统一的对话会话 |
| `Message` | `app/models/message.rb` | 统一的消息实体 |
| `Attachment` | `app/models/attachment.rb` | 消息附件 (图片/音频/视频/文件/位置等) |

### 5.2 统一消息处理模式

所有渠道的消息处理都遵循相同的**四步归一化模式**：

```
┌──────────────────────────────────────────────────────────────┐
│  步骤 1: 解析渠道特定参数                                      │
│  - WhatsApp: 处理 contacts/messages/statuses 数组            │
│  - Facebook: 解析 messaging/standby 事件                     │
│  - Twilio: 解析 Body/MediaUrl/ProfileName 等参数            │
└───────────────────────┬──────────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  步骤 2: 设置/查找联系人 (ContactInboxWithContactBuilder)     │
│  - 提取渠道特定的身份标识符                                     │
│  - 按优先级查找现有联系人                                       │
│  - 创建或复用 Contact + ContactInbox                          │
└───────────────────────┬──────────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  步骤 3: 设置/查找对话 (Conversation)                          │
│  - 检查是否存在未 resolve 的活跃对话                           │
│  - 支持 lock_to_single_conversation 模式                       │
│  - 新建或复用 Conversation                                     │
└───────────────────────┬──────────────────────────────────────┘
                        ▼
┌──────────────────────────────────────────────────────────────┐
│  步骤 4: 创建统一消息 (Message + Attachment)                   │
│  - 标准化消息内容到 message.content                            │
│  - 处理附件下载和存储                                           │
│  - 保存 source_id 用于追溯 (⚠️ 注意：不是唯一约束！)           │
└──────────────────────────────────────────────────────────────┘
```

### 5.3 消息归一化核心字段映射

#### 5.3.1 Message 模型核心字段

```ruby
# 来自 app/models/message.rb (Schema 注释)
# Table name: messages
#
#  id                        :integer          # 主键
#  content                   :text              # 归一化的消息文本内容
#  content_type              :integer          # 内容类型枚举 (text/input_*/cards/form/...)
#  message_type              :integer          # 消息方向: incoming(0)/outgoing(1)/activity(2)/template(3)
#  status                    :integer          # 发送状态: sent(0)/delivered(1)/read(2)/failed(3)
#  source_id                 :text              # 渠道原始消息ID (用于追溯和去重查询)
#  account_id                :integer          # 所属账户
#  conversation_id           :integer          # 关联对话
#  inbox_id                  :integer          # 所属渠道入口
#  sender_id                 :bigint            # 发送者 (Contact 或 User)
#  content_attributes        :json              # 扩展属性 (回复引用/外部echo/翻译等)
#  additional_attributes     :jsonb             # 附加属性 (活动ID等)
```

**⚠️ 重要提醒**: `source_id` 有索引 (`index_messages_on_source_id`)，但这是**普通索引**，不是唯一约束！去重依赖应用层逻辑（Redis 锁 + `find_by` 查询）。

#### 5.3.2 各渠道 source_id 生成规则

**文件位置**: `app/builders/contact_inbox_builder.rb:14-29`

| 渠道类型 | source_id 格式 | 示例 |
|---------|---------------|------|
| `Channel::Whatsapp` | E.164 格式去除 `+` | `8613800138000` |
| `Channel::TwilioSms` (SMS) | E.164 格式 | `+8613800138000` |
| `Channel::TwilioSms` (WhatsApp) | `whatsapp:` + E.164 | `whatsapp:+8613800138000` |
| `Channel::FacebookPage` | PSID (Page Scoped ID) | `1234567890123456` |
| `Channel::Instagram` | Instagram 业务 ID | 类似 PSID |
| `Channel::Email` | 邮箱地址 | `user@example.com` |
| `Channel::Sms` | 手机号 | `+8613800138000` |
| `Channel::Api` / `Channel::WebWidget` | UUID | `550e8400-e29b-41d4-a716-446655440000` |

### 5.4 对话复用策略

所有渠道都遵循相同的对话查找逻辑：

**文件位置**: 各渠道服务中的 `set_conversation` 方法

```ruby
# 通用模式 (以 WhatsApp 为例)
def set_conversation
  # 策略1: 锁定到单一会话模式 - 始终使用最后一个对话
  if @inbox.lock_to_single_conversation
    @conversation = @contact_inbox.conversations.last
  else
    # 策略2: 标准模式 - 查找未 resolved 的活跃对话
    @conversation = @contact_inbox.conversations
                                  .where.not(status: :resolved).last
  end
  
  # 如果没有找到合适的对话，创建新对话
  return if @conversation
  @conversation = ::Conversation.create!(conversation_params)
end
```

**对话状态枚举** (`conversation.rb:75`):
- `open` (0) - 进行中
- `resolved` (1) - 已解决 (新消息将创建新对话)
- `pending` (2) - 待处理
- `snoozed` (3) - 已延后 (新消息会重新打开)

---

## 6. 联系人身份合并机制

### 6.1 核心设计理念

Chatwoot 的联系人合并机制基于 **"一个联系人，多个渠道身份"** 的设计：

```
┌─────────────────────────────────────────────────────────────┐
│                      Contact (统一联系人)                      │
│  - name: "张三"                                               │
│  - email: zhangsan@example.com                               │
│  - phone_number: +8613800138000                             │
│  - identifier: external_id_123                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ContactInbox 1│ │ContactInbox 2│ │ContactInbox 3│
    │ WhatsApp     │ │ Facebook     │ │ Email        │
    │ source_id:   │ │ source_id:   │ │ source_id:   │
    │ 8613800138000│ │ 1234567890   │ │ zhangsan@... │
    └──────────────┘ └──────────────┘ └──────────────┘
```

### 6.2 ContactInboxWithContactBuilder 详解

**文件位置**: `app/builders/contact_inbox_with_contact_builder.rb`

这是联系人身份合并的核心组件。

#### 6.2.1 主流程

```ruby
# 第 8-25 行
def perform
  find_or_create_contact_and_contact_inbox
rescue ActiveRecord::RecordNotUnique
  # 处理并发竞态条件
  find_or_create_contact_and_contact_inbox
end

def find_or_create_contact_and_contact_inbox
  # 步骤 1: 优先通过 source_id 查找现有 ContactInbox
  @contact_inbox = inbox.contact_inboxes.find_by(source_id: source_id) if source_id.present?
  return @contact_inbox if @contact_inbox

  # 步骤 2: 如果没有，查找或创建 Contact + ContactInbox
  ActiveRecord::Base.transaction(requires_new: true) do
    build_contact_with_contact_inbox
  end
  update_contact_avatar(@contact) unless @contact.avatar.attached?
  @contact_inbox
end
```

#### 6.2.2 联系人查找优先级 (合并策略)

**文件位置**: `contact_inbox_with_contact_builder.rb:62-69`

```ruby
def find_contact
  # 优先级 1: 通过 identifier (外部系统ID) 查找
  contact = find_contact_by_identifier(contact_attributes[:identifier])
  # 优先级 2: 通过 email 查找
  contact ||= find_contact_by_email(contact_attributes[:email])
  # 优先级 3: 通过 phone_number 查找
  contact ||= find_contact_by_phone_number(contact_attributes[:phone_number])
  # 优先级 4: Instagram 特殊处理 (跨 Facebook Page 合并)
  contact ||= find_contact_by_instagram_source_id(source_id) if instagram_channel?

  contact
end
```

**查找优先级图示**:

```
                    ┌──────────────────┐
                    │   新消息到达      │
                    │  携带身份信息     │
                    └────────┬─────────┘
                             ▼
              ┌──────────────────────────────┐
              │  1. 检查 source_id 是否存在  │
              │  (inbox.contact_inboxes)     │
              └──────────────┬───────────────┘
                             │ 不存在
                             ▼
              ┌──────────────────────────────┐
              │  2. 按优先级查找现有 Contact  │
              │                              │
              │   ┌─────────────────────┐   │
              │   │ Priority 1:         │   │
              │   │ identifier          │   │
              │   │ (外部系统唯一ID)      │   │
              │   └──────────┬──────────┘   │
              │              │ 未找到        │
              │              ▼               │
              │   ┌─────────────────────┐   │
              │   │ Priority 2:         │   │
              │   │ email               │   │
              │   └──────────┬──────────┘   │
              │              │ 未找到        │
              │              ▼               │
              │   ┌─────────────────────┐   │
              │   │ Priority 3:         │   │
              │   │ phone_number        │   │
              │   └──────────┬──────────┘   │
              │              │ 未找到        │
              │              ▼               │
              │   ┌─────────────────────┐   │
              │   │ Priority 4:         │   │
              │   │ Instagram 特殊逻辑   │   │
              │   │ (跨 Facebook Page)  │   │
              │   └─────────────────────┘   │
              └──────────────┬───────────────┘
                             │
              ┌──────────────┴───────────────┐
              │         找到 / 新建           │
              ▼                                ▼
    ┌──────────────────┐            ┌──────────────────┐
    │  复用现有 Contact │            │  创建新 Contact   │
    │  新建 ContactInbox│            │  新建 ContactInbox│
    └──────────────────┘            └──────────────────┘
```

#### 6.2.3 Instagram 跨渠道合并特殊逻辑

**文件位置**: `contact_inbox_with_contact_builder.rb:79-91`

```ruby
# Instagram 和 Facebook Messenger 可能共享同一个真实用户
# 当用户从 Instagram 发消息时，检查是否有对应的 Facebook Contact

def find_contact_by_instagram_source_id(instagram_id)
  return if instagram_id.blank?

  # 在同一账户下，查找使用相同 source_id 的 Facebook Page 渠道的 ContactInbox
  existing_contact_inbox = ContactInbox.joins(:inbox)
                                       .where(source_id: instagram_id)
                                       .where(
                                         'inboxes.channel_type = ? AND inboxes.account_id = ?',
                                         'Channel::FacebookPage',  # 注意：是 FacebookPage，不是 Instagram
                                         account.id
                                       ).first

  existing_contact_inbox&.contact  # 复用找到的 Contact
end
```

**设计意图**:
- Instagram 和 Facebook Messenger 同属 Meta 生态
- 同一个用户可能在两个渠道有相同的身份标识 (PSID 可能关联)
- 通过这种方式实现跨 Meta 渠道的联系人合并

### 6.3 ContactInbox 的唯一性约束

**文件位置**: `app/models/contact_inbox.rb` (Schema 注释)

```ruby
# 关键索引
#  index_contact_inboxes_on_inbox_id_and_source_id  (inbox_id,source_id) UNIQUE
```

**业务规则**:
- 同一个 `inbox_id` + `source_id` 组合只能有一个 `ContactInbox`
- 这保证了：
  - 同一用户在同一渠道不会重复创建身份
  - 不同渠道的相同 source_id 是允许的 (通过 inbox_id 区分)

### 6.4 竞态条件处理

**文件位置**: `app/builders/contact_inbox_builder.rb:68-104`

当多个请求同时尝试创建相同的 `(inbox_id, source_id)` 时，会触发 `ActiveRecord::RecordNotUnique` 异常。

```ruby
def create_contact_inbox
  attrs = { contact_id: @contact.id, inbox_id: @inbox.id, source_id: @source_id }
  ::ContactInbox.where(attrs).first_or_create!(hmac_verified: hmac_verified || false)
rescue ActiveRecord::RecordNotUnique
  Rails.logger.info("[ContactInboxBuilder] RecordNotUnique #{@source_id} #{@contact.id} #{@inbox.id}")
  update_old_contact_inbox  # 处理冲突
  retry
end

def update_old_contact_inbox
  # 冲突场景: 同一个 source_id 已存在，但属于不同的 contact
  # 可能原因:
  # 1. 代理更新了联系人的 email/phone
  # 2. 联系人合并操作
  #
  # 解决策略: 将旧的 ContactInbox 的 source_id 改为随机值，
  #           让新的 ContactInbox 能够正常创建
  
  raise ActiveRecord::RecordNotUnique unless allowed_channels?

  contact_inbox = ::ContactInbox.find_by(inbox_id: @inbox.id, source_id: @source_id)
  return if contact_inbox.blank?

  contact_inbox.update!(source_id: new_source_id)  # 旧记录"让位"
end
```

### 6.5 联系人合并机制总结

| 维度 | 说明 |
|-----|------|
| **合并触发点** | `ContactInboxWithContactBuilder.find_contact` 方法 |
| **合并优先级** | identifier → email → phone_number → Instagram 特殊逻辑 |
| **跨渠道合并** | 支持 (通过 email/phone/identifier 匹配) |
| **Meta 生态合并** | Instagram ↔ Facebook Page (通过特殊逻辑) |
| **唯一性保证** | `(inbox_id, source_id)` 数据库唯一索引 |
| **竞态处理** | `RecordNotUnique` 捕获 + 旧记录 source_id 置换 |
| **身份标识** | `ContactInbox.source_id` (渠道特定) + `Contact` 全局属性 |

---

## 7. 关键代码位置速查

### 7.1 去重相关代码

| 功能模块 | 文件路径 |
|---------|---------|
| WhatsApp 消息去重锁 | `app/services/whatsapp/message_dedup_lock.rb` |
| WhatsApp 去重检查 helper | `app/services/whatsapp/incoming_message_service_helpers.rb` |
| WhatsApp Job 对话锁 | `app/jobs/webhooks/whatsapp_events_job.rb` |
| Facebook Job 对话锁 | `app/jobs/webhooks/facebook_events_job.rb` |
| Instagram Job 对话锁 | `app/jobs/webhooks/instagram_events_job.rb` |
| TikTok Job 对话锁 | `app/jobs/webhooks/tiktok_events_job.rb` |
| Twilio Job (无锁) | `app/jobs/webhooks/twilio_events_job.rb` |
| 锁基类 | `app/jobs/mutex_application_job.rb` |
| Redis Key 定义 | `lib/redis/redis_keys.rb` |

### 7.2 校验相关代码

| 功能模块 | 文件路径 |
|---------|---------|
| Meta 系列验证 Concern | `app/controllers/concerns/meta_token_verify_concern.rb` |
| WhatsApp 控制器 | `app/controllers/webhooks/whatsapp_controller.rb` |
| Instagram 控制器 | `app/controllers/webhooks/instagram_controller.rb` |
| Facebook Page 订阅 | `app/models/channel/facebook_page.rb` |

### 7.3 归一化相关代码

| 功能模块 | 文件路径 |
|---------|---------|
| WhatsApp 入站消息基类 | `app/services/whatsapp/incoming_message_base_service.rb` |
| Twilio 入站消息服务 | `app/services/twilio/incoming_message_service.rb` |
| Facebook 消息解析器 | `lib/integrations/facebook/message_parser.rb` |
| Facebook 消息创建器 | `lib/integrations/facebook/message_creator.rb` |
| Facebook 消息构建器 | `app/builders/messages/facebook/message_builder.rb` |
| 联系人+Inbox 构建器 (核心合并) | `app/builders/contact_inbox_with_contact_builder.rb` |
| ContactInbox 构建器 | `app/builders/contact_inbox_builder.rb` |

---

## 8. 架构亮点与设计启示

### 8.1 亮点设计

1. **统一的归一化模式**
   - 所有渠道都遵循 `解析 → 联系人查找/创建 → 对话查找/创建 → 消息创建` 的四步模式
   - 便于新增渠道的开发和维护

2. **ContactInbox 中间层**
   - 优雅地解决了"一个联系人，多个渠道身份"的问题
   - `source_id` 的设计使渠道特定标识与全局联系人解耦

3. **分层锁设计 (WhatsApp)**
   - 对话级锁: 防止并发创建重复对话
   - 消息级锁: 防止重复处理同一条消息
   - 数据库查询: 快速检查已处理消息

4. **灵活的对话策略**
   - 支持 `lock_to_single_conversation` 模式 (适用于客服场景)
   - 支持标准模式 (每轮新对话，适用于营销场景)

### 8.2 潜在风险与改进点

#### 8.2.1 去重不一致问题

**当前状态**:
- WhatsApp: 完整三层去重 ✅
- Facebook/Instagram/TikTok: 仅对话级锁 ⚠️
- Twilio/SMS: 完全无锁 ❌

**风险**:
- 如果 Facebook 多次投递同一条消息，可能创建重复消息
- Twilio 不仅可能有重复消息，还可能有重复对话

**建议改进**:
```ruby
# 可以考虑为所有渠道统一实现消息级去重
# 类似于 WhatsApp 的 MessageDedupLock 模式
```

#### 8.2.2 Twilio 签名验证缺失

**风险**: 任何知道 `AccountSid` 和 `To` 号码的人都可以伪造 webhook 请求

**建议改进**:
```ruby
# 实现 X-Twilio-Signature 验证
# 参考: https://www.twilio.com/docs/usage/tutorials/how-to-secure-your-twilio-webhooks-ruby
```

#### 8.2.3 ContactInbox 冲突处理

**当前策略**: 将旧记录的 `source_id` 改为随机值

**风险**: 可能导致历史消息追溯问题

**建议改进**: 考虑更优雅的合并策略，保留历史关联

---

## 9. 附录

### A. 支持的渠道列表 (基于代码分析)

| 渠道名称 | Channel 模型 | 主要文件位置 | 去重级别 |
|---------|-------------|-------------|---------|
| WhatsApp (Cloud/360Dialog) | `Channel::Whatsapp` | `app/models/channel/whatsapp.rb` | 完整 |
| Facebook Messenger | `Channel::FacebookPage` | `app/models/channel/facebook_page.rb` | 仅对话级 |
| Instagram | `Channel::Instagram` | `app/models/channel/instagram.rb` | 仅对话级 |
| Twilio (SMS/WhatsApp) | `Channel::TwilioSms` | `app/models/channel/twilio_sms.rb` | 无 |
| SMS (其他提供商) | `Channel::Sms` | `app/models/channel/sms.rb` | 无 |
| TikTok | `Channel::Tiktok` | `app/models/channel/tiktok.rb` | 仅对话级 |
| Email | `Channel::Email` | `app/models/channel/email.rb` | 仅对话级 (有锁) |
| API | `Channel::Api` | `app/models/channel/api.rb` | 无 |
| Web Widget | `Channel::WebWidget` | `app/models/channel/web_widget.rb` | 无 |
| Line | `Channel::Line` | `app/models/channel/line.rb` | 未知 |
| Telegram | - | `app/services/telegram/` | 未知 |

### B. 消息类型枚举 (Message)

```ruby
# app/models/message.rb:87-103
enum message_type: { incoming: 0, outgoing: 1, activity: 2, template: 3 }
enum content_type: {
  text: 0,
  input_text: 1,
  input_textarea: 2,
  input_email: 3,
  input_select: 4,
  cards: 5,
  form: 6,
  article: 7,
  incoming_email: 8,
  input_csat: 9,
  integrations: 10,
  sticker: 11,
  voice_call: 12
}
enum status: { sent: 0, delivered: 1, read: 2, failed: 3 }
```

### C. 对话状态枚举 (Conversation)

```ruby
# app/models/conversation.rb:75-76
enum status: { open: 0, resolved: 1, pending: 2, snoozed: 3 }
enum priority: { low: 0, medium: 1, high: 2, urgent: 3 }
```

### D. 关键修正记录

| 修正项 | 之前错误描述 | 正确实现 |
|-------|------------|---------|
| Facebook Messenger 校验 | 遗漏，暗示使用 MetaTokenVerifyConcern | 使用 facebook-messenger gem 挂载在 `/bot`，有独立的订阅和 HMAC 签名验证 |
| 消息去重机制 | "数据库唯一约束" | 多层应用层策略：对话级锁 + 消息级 DB 查询 + 消息级 Redis 锁 (仅 WhatsApp) |
| source_id 索引 | 暗示是唯一约束 | 普通索引 (`index_messages_on_source_id`)，用于查询加速，不是唯一约束 |
| 各渠道去重差异 | 未区分 | WhatsApp 完整三层 / FB/IG/TikTok 仅对话级 / Twilio/SMS 完全无锁 |

---

**报告生成日期**: 2026-05-05  
**报告版本**: v2 (修正版)  
**修正内容**: 
1. 补充 Facebook Messenger webhook 完整校验链路
2. 修正消息去重机制描述（从"数据库唯一约束"改为"多层应用层策略"）
3. 详细分析各渠道去重差异
4. 澄清 `source_id` 是普通索引而非唯一约束

**分析基于代码版本**: Chatwoot (当前工作目录版本)  
**分析范围**: WhatsApp, Facebook Messenger, Twilio, Instagram 渠道的入站消息处理
