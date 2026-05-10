# Chatwoot 实时消息广播实现分析

## 概述

Chatwoot 实现了一个完整的实时消息广播系统，能够在服务端数据变更时，将更新实时推送到多个客服坐席的浏览器。该系统基于 Rails ActionCable 框架，配合 Redis Pub/Sub 机制实现跨实例消息分发。

---

## 架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据变更层 (Models/Services)                        │
│  Message / Conversation / Contact / Notification / Assignment 等模型变更      │
│         ↓ 触发 after_create / after_update / after_destroy 回调               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                              事件分发层 (Dispatcher)                          │
│  Rails.configuration.dispatcher.dispatch(event_name, timestamp, data)        │
│         ↓ Wisper::Publisher 发布事件                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
         ┌──────────────────────────────────────────────────────┐
         │                  事件监听器层 (Listeners)              │
         │  ┌─────────────────┐    ┌─────────────────────────┐   │
         │  │ SyncDispatcher  │    │    AsyncDispatcher      │   │
         │  │ (同步执行)        │    │    (异步 Job)            │   │
         │  └────────┬────────┘    └───────────┬─────────────┘   │
         │           │                        │                  │
         │  ┌────────▼────────┐    ┌───────────▼─────────────┐   │
         │  │ActionCableListener│   │其他监听器(Webhook/Automation等)│
         │  └────────┬────────┘    └─────────────────────────┘   │
         └───────────┼───────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           广播任务层 (Jobs)                                   │
│              ActionCableBroadcastJob.perform_later(tokens, event, data)      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ActionCable 服务器层                                      │
│         ActionCable.server.broadcast(stream_name, {event, data})             │
│              ↓ (通过 Redis Pub/Sub)                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
         ┌──────────────────────────────────────────────────────┐
         │                   Redis 层 (跨实例分发)                 │
         │  ┌────────────────────────────────────────────────┐  │
         │  │ PUBLISH "chatwoot_production_action_cable:..."  │  │
         │  │ ↓                                              │  │
         │  │ 所有 Rails 实例订阅相同频道，接收消息              │  │
         │  └────────────────────────────────────────────────┘  │
         └──────────────────────────────────────────────────────┘
                                    ↓
         ┌──────────────────────────────────────────────────────┐
         │               WebSocket 连接层                         │
         │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
         │  │ 客服A    │  │ 客服B    │  │ 访客1    │  │ 访客2    │ │
         │  │ 浏览器   │  │ 浏览器   │  │ 浏览器   │  │ 浏览器   │ │
         │  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │
         └──────────────────────────────────────────────────────┘
```

---

## 详细实现分析

### 1. 事件触发机制

#### 1.1 模型层面的事件触发

服务端的数据变更通过 ActiveRecord 回调触发事件。主要在以下模型中实现：

**Message 模型** (`app/models/message.rb:378-383`):
```ruby
after_create :dispatch_create_events

def dispatch_create_events
  Rails.configuration.dispatcher.dispatch(MESSAGE_CREATED, Time.zone.now, message: self, performed_by: Current.executed_by)
  
  if valid_first_reply?
    Rails.configuration.dispatcher.dispatch(FIRST_REPLY_CREATED, Time.zone.now, message: self, performed_by: Current.executed_by)
  end
end

after_update :send_update_event

def send_update_event
  Rails.configuration.dispatcher.dispatch(MESSAGE_UPDATED, Time.zone.now, message: self, previous_changes: previous_changes)
end
```

**Conversation 模型** (`app/models/conversation.rb:312-315`):
```ruby
def dispatcher_dispatch(event_name, changed_attributes = nil)
  Rails.configuration.dispatcher.dispatch(event_name, Time.zone.now, conversation: self, changed_attributes: changed_attributes)
end
```

**Notification 模型** (`app/models/notification.rb:174-186`):
```ruby
after_create :dispatch_create_event
after_update :dispatch_update_event
after_destroy :dispatch_destroy_event

def dispatch_create_event
  Rails.configuration.dispatcher.dispatch(NOTIFICATION_CREATED, Time.zone.now, notification: self)
end
```

#### 1.2 服务层面的事件触发

除了模型回调，一些服务也会直接触发事件：

**TypingStatusManager** (`app/services/conversations/typing_status_manager.rb:12-13`):
```ruby
def trigger_typing_event(event, is_private)
  Rails.configuration.dispatcher.dispatch(event, Time.zone.now, conversation: @conversation, user: @user, is_private: is_private)
end
```

**AssignmentService** (`app/services/auto_assignment/assignment_service.rb:88-92`):
```ruby
def dispatch_assignment_event(conversation, agent)
  Rails.configuration.dispatcher.dispatch(
    Events::Types::ASSIGNEE_CHANGED,
    Time.zone.now,
    conversation: conversation
  )
end
```

---

### 2. 事件类型定义

所有事件类型定义在 `lib/events/types.rb` 中：

```ruby
module Events::Types
  # 会话事件
  CONVERSATION_CREATED = 'conversation.created'
  CONVERSATION_UPDATED = 'conversation.updated'
  CONVERSATION_READ = 'conversation.read'
  CONVERSATION_STATUS_CHANGED = 'conversation.status_changed'
  CONVERSATION_TYPING_ON = 'conversation.typing_on'
  CONVERSATION_TYPING_OFF = 'conversation.typing_off'
  ASSIGNEE_CHANGED = 'assignee.changed'
  TEAM_CHANGED = 'team.changed'
  
  # 消息事件
  MESSAGE_CREATED = 'message.created'
  MESSAGE_UPDATED = 'message.updated'
  FIRST_REPLY_CREATED = 'first.reply.created'
  
  # 通知事件
  NOTIFICATION_CREATED = 'notification.created'
  NOTIFICATION_UPDATED = 'notification.updated'
  NOTIFICATION_DELETED = 'notification.deleted'
  
  # 联系人事件
  CONTACT_CREATED = 'contact.created'
  CONTACT_UPDATED = 'contact.updated'
  CONTACT_DELETED = 'contact.deleted'
  
  # 账户事件
  ACCOUNT_CACHE_INVALIDATED = 'account.cache_invalidated'
  # ... 更多事件类型
end
```

---

### 3. 事件分发系统

Chatwoot 使用 Wisper gem 实现发布-订阅模式，结合同步和异步两种分发机制。

#### 3.1 Dispatcher 单例

`app/dispatchers/dispatcher.rb`:
```ruby
class Dispatcher
  include Singleton
  
  def initialize
    @sync_dispatcher = SyncDispatcher.new
    @async_dispatcher = AsyncDispatcher.new
  end
  
  def dispatch(event_name, timestamp, data, _async = false)
    @sync_dispatcher.dispatch(event_name, timestamp, data)
    @async_dispatcher.dispatch(event_name, timestamp, data)
  end
end
```

#### 3.2 同步分发器 (SyncDispatcher)

`app/dispatchers/sync_dispatcher.rb`:
```ruby
class SyncDispatcher < BaseDispatcher
  include Wisper::Publisher
  
  def dispatch(event_name, timestamp, data)
    event_object = Events::Base.new(event_name, timestamp, data)
    publish(event_object.method_name, event_object)
  end
  
  def listeners
    [ActionCableListener.instance, AgentBotListener.instance]
  end
end
```

**关键点**：
- 同步执行，实时性高
- ActionCableListener 注册在这里，确保消息广播的低延迟

#### 3.3 异步分发器 (AsyncDispatcher)

`app/dispatchers/async_dispatcher.rb`:
```ruby
class AsyncDispatcher < BaseDispatcher
  def dispatch(event_name, timestamp, data)
    EventDispatcherJob.perform_later(event_name, timestamp, data)
  end
  
  def publish_event(event_name, timestamp, data)
    event_object = Events::Base.new(event_name, timestamp, data)
    publish(event_object.method_name, event_object)
  end
  
  def listeners
    [
      AutomationRuleListener.instance,
      CampaignListener.instance,
      WebhookListener.instance,
      NotificationListener.instance,
      # ... 其他监听器
    ]
  end
end
```

**关键点**：
- 通过 ActiveJob 异步执行
- 不阻塞主请求流程
- 用于 Webhook、自动化规则等耗时操作

---

### 4. ActionCableListener 核心实现

`app/listeners/action_cable_listener.rb` 是连接事件系统和 ActionCable 广播的核心组件。

#### 4.1 事件处理方法示例

```ruby
def message_created(event)
  message, account = extract_message_and_account(event)
  conversation = message.conversation
  # 确定需要接收消息的用户 tokens
  tokens = user_tokens(account, conversation.inbox.members) + contact_tokens(conversation.contact_inbox, message)
  
  broadcast(account, tokens, MESSAGE_CREATED, message.push_event_data)
end

def conversation_typing_on(event)
  conversation = event.data[:conversation]
  account = conversation.account
  user = event.data[:user]
  tokens = typing_event_listener_tokens(account, conversation, user)
  
  broadcast(account, tokens, CONVERSATION_TYPING_ON, {
    conversation: conversation.push_event_data,
    user: user.push_event_data,
    is_private: event.data[:is_private] || false
  })
end
```

#### 4.2 Token 计算逻辑

```ruby
def user_tokens(account, agents)
  agent_tokens = agents.pluck(:pubsub_token)
  admin_tokens = account.administrators.pluck(:pubsub_token)
  (agent_tokens + admin_tokens).uniq
end

def contact_tokens(contact_inbox, message)
  return [] if message.private?
  return [] if message.activity?
  return [] if contact_inbox.nil?
  
  contact_inbox_tokens(contact_inbox)
end

def contact_inbox_tokens(contact_inbox)
  contact = contact_inbox.contact
  contact_inbox.hmac_verified? ? 
    contact.contact_inboxes.where(hmac_verified: true).filter_map(&:pubsub_token) : 
    [contact_inbox.pubsub_token]
end
```

**Token 计算策略**：
- **客服/管理员**：使用 `User.pubsub_token`
- **访客**：使用 `ContactInbox.pubsub_token`
- **私有消息**：不会广播给访客
- **HMAC 验证**：已验证的访客会广播到所有验证过的 contact_inboxes

#### 4.3 核心广播方法

`app/listeners/action_cable_listener.rb:213-222`:
```ruby
def broadcast(account, tokens, event_name, data)
  return if tokens.blank?
  
  payload = data.merge(account_id: account.id)
  payload[:performer] = Current.user&.push_event_data if Current.user.present?
  
  ::ActionCableBroadcastJob.perform_later(tokens.uniq, event_name, payload)
end
```

---

### 5. ActionCableBroadcastJob

`app/jobs/action_cable_broadcast_job.rb` 负责将消息通过 ActionCable 广播出去。

```ruby
class ActionCableBroadcastJob < ApplicationJob
  queue_as :critical
  
  CONVERSATION_UPDATE_EVENTS = [
    CONVERSATION_READ,
    CONVERSATION_UPDATED,
    TEAM_CHANGED,
    ASSIGNEE_CHANGED,
    CONVERSATION_STATUS_CHANGED
  ].freeze
  
  def perform(members, event_name, data)
    return if members.blank?
    
    broadcast_data = prepare_broadcast_data(event_name, data)
    broadcast_to_members(members, event_name, broadcast_data)
  end
  
  private
  
  def prepare_broadcast_data(event_name, data)
    return data unless CONVERSATION_UPDATE_EVENTS.include?(event_name)
    
    # 对于会话更新事件，重新获取最新数据
    # 防止高流量下事件乱序导致 UI 问题
    account = Account.find(data[:account_id])
    conversation = account.conversations.find_by!(display_id: data[:id])
    conversation.push_event_data.merge(account_id: data[:account_id])
  end
  
  def broadcast_to_members(members, event_name, broadcast_data)
    members.each do |member|
      ActionCable.server.broadcast(
        member,
        {
          event: event_name,
          data: broadcast_data
        }
      )
    end
  end
end
```

**重要设计**：
- 使用 `:critical` 队列，确保实时性
- 对于会话更新类事件，会重新从数据库获取最新数据，防止事件乱序

---

### 6. Redis 在跨实例消息分发中的角色

#### 6.1 ActionCable Redis 适配器配置

`config/cable.yml`:
```yaml
default: &default
  adapter: redis
  url: <%= ENV.fetch('REDIS_URL', 'redis://127.0.0.1:6379') %>
  password: <%= ENV.fetch('REDIS_PASSWORD', nil).presence %>
  channel_prefix: <%= "chatwoot_#{Rails.env}_action_cable" %>
```

#### 6.2 自定义 Redis 连接

`config/initializers/actioncable.rb`:
```ruby
require 'action_cable/subscription_adapter/redis'

ActionCable::SubscriptionAdapter::Redis.redis_connector = lambda do |config|
  config[:id] = nil if ENV['REDIS_DISABLE_CLIENT_COMMAND'].present?
  Redis.new(config.except(:adapter, :channel_prefix))
end
```

#### 6.3 Redis Pub/Sub 工作原理

当调用 `ActionCable.server.broadcast(stream_name, message)` 时：

1. **发布消息**：当前 Rails 实例向 Redis 发布消息
   - 频道名格式：`chatwoot_{env}_action_cable:{stream_name}`
   - 例如：`chatwoot_production_action_cable:user_token_123`

2. **订阅消息**：所有 Rails 实例都订阅了相同的 Redis 频道模式

3. **分发连接**：当某个 Rails 实例收到 Redis 消息后
   - 查找该实例上订阅了对应 stream 的 WebSocket 连接
   - 将消息通过 WebSocket 发送给客户端

**架构示意图**：

```
                    ┌───────────────┐
                    │   Redis       │
                    │  Pub/Sub      │
                    └───────┬───────┘
                            │ PUBLISH / SUBSCRIBE
            ┌───────────────┼───────────────┐
            │               │               │
   ┌────────▼──────┐ ┌─────▼──────┐ ┌──────▼────────┐
   │  Rails 实例 A  │ │ Rails 实例 B│ │  Rails 实例 C  │
   │               │ │            │ │               │
   │ WS: 客服1, 访客1│ │ WS: 客服2,3 │ │ WS: 访客2,3   │
   └────────┬──────┘ └─────┬──────┘ └──────┬────────┘
            │              │               │
   ┌────────▼──────┐ ┌─────▼──────┐ ┌──────▼────────┐
   │ 浏览器连接    │ │ 浏览器连接  │ │ 浏览器连接    │
   └───────────────┘ └────────────┘ └───────────────┘
```

**优势**：
- **水平扩展**：可以部署任意数量的 Rails 实例
- **负载均衡**：WebSocket 连接可以分布到不同实例
- **消息一致性**：所有实例通过 Redis 收到相同的广播消息

---

### 7. ActionCable 频道 (RoomChannel)

`app/channels/room_channel.rb` 是客户端订阅的唯一频道。

```ruby
class RoomChannel < ApplicationCable::Channel
  def subscribed
    current_user
    current_account
    ensure_stream
    update_subscription
    broadcast_presence
  end
  
  private
  
  def ensure_stream
    # 订阅个人 stream
    stream_from pubsub_token
    # 客服额外订阅账户级别 stream
    stream_from "account_#{@current_account.id}" if @current_account.present? && @current_user.is_a?(User)
  end
  
  def update_subscription
    return if @current_account.blank?
    ::OnlineStatusTracker.update_presence(@current_account.id, @current_user.class.name, @current_user.id)
  end
  
  def pubsub_token
    @pubsub_token ||= params[:pubsub_token]
  end
  
  def current_user
    @current_user ||= if params[:user_id].blank?
                        ContactInbox.find_by!(pubsub_token: pubsub_token).contact
                      else
                        User.find_by!(pubsub_token: pubsub_token, id: params[:user_id])
                      end
  end
end
```

**Stream 订阅策略**：
- **所有用户**：订阅 `pubsub_token`（个人 stream）
- **客服（User）**：额外订阅 `account_{account_id}`（账户级 stream）
- **访客（Contact）**：只订阅个人 stream

---

### 8. 前端 WebSocket 连接

#### 8.1 基础连接器

`app/javascript/shared/helpers/BaseActionCableConnector.js`:
```javascript
import { createConsumer } from '@rails/actioncable';

class BaseActionCableConnector {
  constructor(app, pubsubToken, websocketHost = '') {
    const websocketURL = websocketHost ? `${websocketHost}/cable` : undefined;
    
    this.consumer = createConsumer(websocketURL);
    this.subscription = this.consumer.subscriptions.create(
      {
        channel: 'RoomChannel',
        pubsub_token: pubsubToken,
        account_id: app.$store.getters.getCurrentAccountId,
        user_id: app.$store.getters.getCurrentUserID,
      },
      {
        updatePresence() {
          this.perform('update_presence');
        },
        received: this.onReceived,
        disconnected: () => {
          this.onDisconnected();
          this.initReconnectTimer();
        },
      }
    );
    
    this.triggerPresenceInterval();
  }
  
  onReceived = ({ event, data } = {}) => {
    if (this.isAValidEvent(data)) {
      if (this.events[event] && typeof this.events[event] === 'function') {
        this.events[event](data);
      }
    }
  };
}
```

#### 8.2 客服面板连接器

`app/javascript/dashboard/helper/actionCable.js`:
```javascript
class ActionCableConnector extends BaseActionCableConnector {
  constructor(app, pubsubToken) {
    super(app, pubsubToken, websocketURL);
    
    this.events = {
      'message.created': this.onMessageCreated,
      'message.updated': this.onMessageUpdated,
      'conversation.created': this.onConversationCreated,
      'conversation.status_changed': this.onStatusChange,
      'assignee.changed': this.onAssigneeChanged,
      'conversation.typing_on': this.onTypingOn,
      'conversation.typing_off': this.onTypingOff,
      'presence.update': this.onPresenceUpdate,
      // ... 更多事件
    };
  }
  
  onMessageCreated = data => {
    DashboardAudioNotificationHelper.onNewMessage(data);
    this.app.$store.dispatch('addMessage', data);
    this.app.$store.dispatch('updateConversationLastActivity', {
      lastActivityAt: data.conversation.last_activity_at,
      conversationId: data.conversation_id,
    });
  };
  
  onTypingOn = ({ conversation, user }) => {
    this.app.$store.dispatch('conversationTypingStatus/create', {
      conversationId: conversation.id,
      user,
    });
  };
}
```

#### 8.3 访客 Widget 连接器

`app/javascript/widget/helpers/actionCable.js`:
```javascript
class ActionCableConnector extends BaseActionCableConnector {
  constructor(app, pubsubToken) {
    super(app, pubsubToken);
    
    this.events = {
      'message.created': this.onMessageCreated,
      'conversation.typing_on': this.onTypingOn,
      'conversation.status_changed': this.onStatusChange,
      'presence.update': this.onPresenceUpdate,
      // ... 访客相关事件
    };
  }
}
```

---

### 9. 完整消息链路示例

让我们以"客服发送一条消息"为例，追踪完整的消息链路：

#### 步骤 1：客服发送消息

客服 A 在浏览器点击发送，消息通过 HTTP API 发送到后端。

#### 步骤 2：消息创建触发事件

`app/models/message.rb:378-380`
```ruby
after_create :dispatch_create_events

def dispatch_create_events
  Rails.configuration.dispatcher.dispatch(MESSAGE_CREATED, Time.zone.now, message: self)
end
```

#### 步骤 3：Dispatcher 分发事件

`app/dispatchers/dispatcher.rb:15-17`
```ruby
def dispatch(event_name, timestamp, data, _async = false)
  @sync_dispatcher.dispatch(event_name, timestamp, data)
  @async_dispatcher.dispatch(event_name, timestamp, data)
end
```

#### 步骤 4：ActionCableListener 处理

`app/listeners/action_cable_listener.rb:41-47`
```ruby
def message_created(event)
  message, account = extract_message_and_account(event)
  conversation = message.conversation
  # 计算需要接收消息的用户 tokens
  # - 客服 B, C (同一 inbox 的其他成员)
  # - 访客 (如果不是私有消息)
  tokens = user_tokens(account, conversation.inbox.members) + contact_tokens(conversation.contact_inbox, message)
  
  broadcast(account, tokens, MESSAGE_CREATED, message.push_event_data)
end
```

#### 步骤 5：创建广播 Job

`app/listeners/action_cable_listener.rb:221`
```ruby
::ActionCableBroadcastJob.perform_later(tokens.uniq, event_name, payload)
```

#### 步骤 6：ActionCableBroadcastJob 执行

`app/jobs/action_cable_broadcast_job.rb:33-43`
```ruby
def broadcast_to_members(members, event_name, broadcast_data)
  members.each do |member|
    ActionCable.server.broadcast(
      member,
      {
        event: event_name,
        data: broadcast_data
      }
    )
  end
end
```

#### 步骤 7：Redis Pub/Sub 分发

```
Rails 实例 A → Redis PUBLISH → Rails 实例 A, B, C (全部收到)
```

#### 步骤 8：各实例向本地 WebSocket 连接推送

- **实例 A**：有客服 B 的 WebSocket 连接 → 推送消息
- **实例 B**：有客服 C 的 WebSocket 连接 → 推送消息
- **实例 C**：有访客的 WebSocket 连接 → 推送消息

#### 步骤 9：浏览器接收并处理

客服 B 的浏览器：
```javascript
onMessageCreated = data => {
  this.app.$store.dispatch('addMessage', data);
  // UI 自动更新，显示新消息
};
```

访客浏览器：
```javascript
onMessageCreated = data => {
  this.app.$store.dispatch('conversation/addOrUpdateMessage', data);
  // Widget 显示新消息
  playNewMessageNotificationInWidget();
};
```

---

### 10. 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `lib/events/types.rb` | 定义所有事件类型常量 |
| `app/dispatchers/dispatcher.rb` | 事件分发中心（单例） |
| `app/dispatchers/sync_dispatcher.rb` | 同步事件分发器 |
| `app/dispatchers/async_dispatcher.rb` | 异步事件分发器 |
| `app/dispatchers/base_dispatcher.rb` | 分发器基类（集成 Wisper） |
| `app/listeners/action_cable_listener.rb` | ActionCable 事件监听器 |
| `app/jobs/action_cable_broadcast_job.rb` | ActionCable 广播后台任务 |
| `app/channels/room_channel.rb` | ActionCable 频道定义 |
| `config/cable.yml` | ActionCable 配置（Redis 适配器） |
| `config/initializers/actioncable.rb` | ActionCable Redis 连接器定制 |
| `app/javascript/shared/helpers/BaseActionCableConnector.js` | WebSocket 连接基础类 |
| `app/javascript/dashboard/helper/actionCable.js` | 客服面板 WebSocket 连接器 |
| `app/javascript/widget/helpers/actionCable.js` | 访客 Widget WebSocket 连接器 |

---

### 11. 设计亮点与最佳实践

#### 11.1 同步 vs 异步分离

- **实时广播**（ActionCableListener）使用同步分发，确保低延迟
- **耗时操作**（Webhook、自动化规则）使用异步分发，不阻塞请求

#### 11.2 防止事件乱序

`ActionCableBroadcastJob` 中对会话更新类事件重新获取最新数据：
```ruby
def prepare_broadcast_data(event_name, data)
  return data unless CONVERSATION_UPDATE_EVENTS.include?(event_name)
  
  account = Account.find(data[:account_id])
  conversation = account.conversations.find_by!(display_id: data[:id])
  conversation.push_event_data.merge(account_id: data[:account_id])
end
```

#### 11.3 Stream 粒度控制

- 个人 Stream：`pubsub_token`（精准推送）
- 账户 Stream：`account_{id}`（广播给所有客服）
- 访客只能订阅个人 Stream（安全隔离）

#### 11.4 Redis Pub/Sub 实现水平扩展

- 所有 Rails 实例共享同一个 Redis
- 任何实例触发的广播都会通过 Redis 到达所有实例
- 每个实例只负责向自己的 WebSocket 连接推送

#### 11.5 Token 安全策略

- 客服使用 `User.pubsub_token`
- 访客使用 `ContactInbox.pubsub_token`
- HMAC 验证的访客有额外的安全机制

---

### 12. 总结

Chatwoot 的实时消息广播系统是一个精心设计的多层架构：

1. **事件驱动**：通过 ActiveRecord 回调和服务层触发事件
2. **发布-订阅**：使用 Wisper 实现灵活的事件监听
3. **分层分发**：同步+异步两种分发策略
4. **ActionCable**：Rails 原生 WebSocket 框架
5. **Redis Pub/Sub**：实现跨实例消息分发
6. **精细流控**：基于 pubsub_token 的精准推送

这种架构能够支持：
- 多客服坐席实时同步
- 水平扩展多个 Rails 实例
- 高并发下的消息可靠性
- 灵活的事件扩展

