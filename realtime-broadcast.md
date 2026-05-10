# Chatwoot 实时消息广播实现分析

## 概述

Chatwoot 实现了一个完整的实时消息广播系统，能够在服务端数据变更时，将更新实时推送到多个客服坐席的浏览器。该系统基于 Rails ActionCable 框架，配合 Redis Pub/Sub 机制实现跨实例消息分发。

---

## 架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据变更层 (Models/Services)                        │
│  Message / Conversation / Contact / Notification / Assignment 等模型变更      │
│         ↓ 触发 after_create_commit / after_update_commit 回调                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                              事件分发层 (Dispatcher)                          │
│  Rails.configuration.dispatcher.dispatch(event_name, timestamp, data)        │
│         ↓ SyncDispatcher / AsyncDispatcher 分别处理                            │
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

**Message 模型回调定义** (`app/models/message.rb:137-140`):
```ruby
after_create_commit :execute_after_create_commit_callbacks
after_update_commit :dispatch_update_event
after_commit :reindex_for_search, if: :should_index?, on: [:create, :update]
```

**关键点**：
- 使用 `after_create_commit` 而非 `after_create`，确保**数据库事务提交后**再触发事件
- 使用 `after_update_commit` 而非 `after_update`
- 注释明确说明这是为了解决 Rails issue #20911（回调执行顺序问题）

**Message 模型创建后回调链** (`app/models/message.rb:324-333`):
```ruby
def execute_after_create_commit_callbacks
  # rails issue with order of active record callbacks being executed https://github.com/rails/rails/issues/20911
  reopen_conversation
  mark_pending_conversation_as_open_for_human_response
  set_conversation_activity
  dispatch_create_events
  send_reply
  execute_message_template_hooks
  update_contact_activity
end
```

**回调执行顺序**：
1. `reopen_conversation` - 如果会话已 snoozed/resolved 则重新打开
2. `mark_pending_conversation_as_open_for_human_response` - Capatain 待处理会话标记
3. `set_conversation_activity` - 更新会话最后活动时间
4. **`dispatch_create_events` - 触发 MESSAGE_CREATED 事件**
5. `send_reply` - 异步发送消息到外部渠道
6. `execute_message_template_hooks` - 执行消息模板钩子
7. `update_contact_activity` - 更新联系人最后活动时间

**dispatch_create_events 实现** (`app/models/message.rb:378-387`):
```ruby
def dispatch_create_events
  Rails.configuration.dispatcher.dispatch(MESSAGE_CREATED, Time.zone.now, message: self, performed_by: Current.executed_by)

  if valid_first_reply?
    Rails.configuration.dispatcher.dispatch(FIRST_REPLY_CREATED, Time.zone.now, message: self, performed_by: Current.executed_by)
    conversation.update(first_reply_created_at: created_at, waiting_since: nil)
  else
    update_waiting_since
  end
end
```

**消息更新回调** (`app/models/message.rb:389-395`):
```ruby
def dispatch_update_event
  # ref: https://github.com/rails/rails/issues/44500
  # we want to skip the update event if the message is not updated
  return if previous_changes.blank?

  send_update_event
end
```

**Conversation 模型** (`app/models/conversation.rb:312-315`):
```ruby
def dispatcher_dispatch(event_name, changed_attributes = nil)
  Rails.configuration.dispatcher.dispatch(event_name, Time.zone.now, conversation: self, changed_attributes: changed_attributes, performed_by: Current.executed_by)
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
  REPLY_CREATED = 'reply.created'
  
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
  
  # Copilot 事件（企业版）
  COPILOT_MESSAGE_CREATED = 'copilot.message.created'
end
```

---

### 3. 事件名到监听方法的映射机制

#### 3.1 事件名转换规则

事件名通过 `Events::Base#method_name` 方法转换为监听方法名：

`lib/events/base.rb`:
```ruby
class Events::Base
  attr_accessor :data
  attr_reader :name, :timestamp

  def initialize(name, timestamp, data)
    @name = name
    @data = data
    @timestamp = timestamp
  end

  def method_name
    name.to_s.tr('.', '_')
  end
end
```

**转换规则**：将事件名字符串中的点号 `.` 替换为下划线 `_`

**映射示例**：
| 事件常量 | 事件名字符串 | 监听方法名 |
|---------|-------------|-----------|
| `MESSAGE_CREATED` | `'message.created'` | `message_created` |
| `CONVERSATION_TYPING_ON` | `'conversation.typing_on'` | `conversation_typing_on` |
| `ACCOUNT_CACHE_INVALIDATED` | `'account.cache_invalidated'` | `account_cache_invalidated` |
| `FIRST_REPLY_CREATED` | `'first.reply.created'` | `first_reply_created` |

#### 3.2 Wisper 发布-订阅机制

Chatwoot 使用 Wisper gem 实现发布-订阅模式。分发器通过继承 `BaseDispatcher` 获得 `Wisper::Publisher` 能力。

`app/dispatchers/base_dispatcher.rb`:
```ruby
class BaseDispatcher
  include Wisper::Publisher

  def listeners
    []
  end

  def load_listeners
    listeners.each { |listener| subscribe(listener) }
  end
end
```

**工作流程**：
1. `SyncDispatcher#dispatch` 创建 `Events::Base` 对象
2. 调用 `publish(event_object.method_name, event_object)`
3. Wisper 查找所有已订阅 listener 中同名方法
4. 调用 `listener.message_created(event_object)`

---

### 4. 事件分发系统

#### 4.1 Dispatcher 单例

`app/dispatchers/dispatcher.rb`:
```ruby
class Dispatcher
  include Singleton

  attr_reader :async_dispatcher, :sync_dispatcher

  def self.dispatch(event_name, timestamp, data, async = false)
    Rails.configuration.dispatcher.dispatch(event_name, timestamp, data, async)
  end

  def initialize
    @sync_dispatcher = SyncDispatcher.new
    @async_dispatcher = AsyncDispatcher.new
  end

  def dispatch(event_name, timestamp, data, _async = false)
    @sync_dispatcher.dispatch(event_name, timestamp, data)
    @async_dispatcher.dispatch(event_name, timestamp, data)
  end

  def load_listeners
    @sync_dispatcher.load_listeners
    @async_dispatcher.load_listeners
  end
end
```

#### 4.2 同步分发器 (SyncDispatcher)

`app/dispatchers/sync_dispatcher.rb`:
```ruby
class SyncDispatcher < BaseDispatcher
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
- 继承 `BaseDispatcher`，通过继承获得 `Wisper::Publisher` 能力
- 同步执行，实时性高
- `ActionCableListener` 注册在这里，确保消息广播的低延迟

#### 4.3 异步分发器 (AsyncDispatcher)

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
      CsatSurveyListener.instance,
      HookListener.instance,
      InstallationWebhookListener.instance,
      NotificationListener.instance,
      ParticipationListener.instance,
      ReportingEventListener.instance,
      WebhookListener.instance
    ]
  end
end

AsyncDispatcher.prepend_mod_with('AsyncDispatcher')
```

**关键点**：
- 通过 `EventDispatcherJob` 异步执行
- 不阻塞主请求流程
- 用于 Webhook、自动化规则等耗时操作

---

### 5. 监听器加载时机

监听器在 Rails 应用启动时通过 `config.to_prepare` 加载：

`config/initializers/event_handlers.rb`:
```ruby
Rails.application.configure do
  config.to_prepare do
    Rails.configuration.dispatcher = Dispatcher.instance
    Rails.configuration.dispatcher.load_listeners
  end
end
```

**加载顺序**：
1. Rails 应用启动
2. `to_prepare` 回调执行（开发环境下每次代码重载都会执行）
3. `Dispatcher.instance` 获取单例
4. `Dispatcher#load_listeners` 调用：
   - `SyncDispatcher#load_listeners` → 订阅 `ActionCableListener`、`AgentBotListener`
   - `AsyncDispatcher#load_listeners` → 订阅 `WebhookListener`、`AutomationRuleListener` 等

**监听器列表**：
- **同步监听器** (`SyncDispatcher`):
  - `ActionCableListener` - 实时消息广播
  - `AgentBotListener` - Agent Bot 集成

- **异步监听器** (`AsyncDispatcher`):
  - `AutomationRuleListener` - 自动化规则
  - `CampaignListener` - 营销活动
  - `CsatSurveyListener` - CSAT 满意度调查
  - `HookListener` - 内部钩子
  - `InstallationWebhookListener` - 安装级 Webhook
  - `NotificationListener` - 通知系统
  - `ParticipationListener` - 参与追踪
  - `ReportingEventListener` - 统计报表
  - `WebhookListener` - 账户级 Webhook

---

### 6. 同步分发与消息回调真实触发点

#### 6.1 Message 模型完整时序图（消息创建场景）

```
HTTP 请求
    ↓
MessagesController#create
    ↓
Message.save!
    ↓
数据库事务开始
    ↓
before_validation / before_save 回调
    ↓
INSERT INTO messages
    ↓
数据库事务提交 ←── 关键点：数据已持久化
    ↓
after_create_commit 触发
    ↓
execute_after_create_commit_callbacks
    ├── reopen_conversation
    ├── mark_pending_conversation_as_open_for_human_response
    ├── set_conversation_activity
    ├── dispatch_create_events
    │   └── Rails.configuration.dispatcher.dispatch(MESSAGE_CREATED, ...)
    │       ├── SyncDispatcher.dispatch  ←── 同步执行
    │       │   ├── ActionCableListener.message_created(event)
    │       │   │   └── ActionCableBroadcastJob.perform_later(...)
    │       │   └── AgentBotListener.message_created(event)
    │       └── AsyncDispatcher.dispatch ←── 异步执行
    │           └── EventDispatcherJob.perform_later(...)
    ├── send_reply
    ├── execute_message_template_hooks
    └── update_contact_activity
    ↓
HTTP 响应返回
```

#### 6.2 重要结论

1. **同步分发在 HTTP 请求线程内执行**，但 `ActionCableListener` 只创建 `perform_later` Job，实际广播由 Sidekiq 异步处理
2. **事件触发在事务提交之后**，确保监听器看到的数据是已持久化的
3. **`dispatch_create_events` 可能触发多个事件**：
   - `MESSAGE_CREATED` - 无条件触发
   - `FIRST_REPLY_CREATED` - 首条回复时触发
   - `REPLY_CREATED` - 人类回复时由 `update_waiting_since` 触发

#### 6.3 update_waiting_since 额外事件

`app/models/message.rb:344-360`:
```ruby
def clear_waiting_since_on_outgoing_response
  if human_response?
    Rails.configuration.dispatcher.dispatch(
      REPLY_CREATED, Time.zone.now, waiting_since: conversation.waiting_since, message: self
    )
    conversation.update(waiting_since: nil)
    return
  end

  conversation.update(waiting_since: nil) if bot_response? && !preserve_waiting_since
end

def set_waiting_since_on_incoming_message
  conversation.update(waiting_since: created_at) if incoming? && conversation.waiting_since.blank?
end
```

---

### 7. 企业版扩展监听

Chatwoot 企业版通过 `prepend` 机制扩展 OSS 版的监听器功能。

#### 7.1 企业版模块注入机制

`config/initializers/01_inject_enterprise_edition_module.rb`:
```ruby
Module.prepend(InjectEnterpriseEditionModule)
```

这会给所有 Ruby Module 添加 `prepend_mod_with` 方法：

```ruby
module InjectEnterpriseEditionModule
  def prepend_mod_with(constant_name, namespace: Object, with_descendants: false)
    each_extension_for(constant_name, namespace) do |constant|
      prepend_module(constant, with_descendants)
    end
  end

  private

  def prepend_module(mod, with_descendants)
    prepend(mod)
    descendants.each { |descendant| descendant.prepend(mod) } if with_descendants
  end

  def each_extension_for(constant_name, namespace)
    ChatwootApp.extensions.each do |extension_name|
      extension_namespace = const_get_maybe_false(namespace, extension_name.camelize)
      extension_module = const_get_maybe_false(extension_namespace, constant_name)
      yield(extension_module) if extension_module
    end
  end
end
```

#### 7.2 ActionCableListener 企业版扩展

OSS 版在文件末尾声明：
`app/listeners/action_cable_listener.rb:224-225`:
```ruby
end

ActionCableListener.prepend_mod_with('ActionCableListener')
```

企业版扩展模块：
`enterprise/app/listeners/enterprise/action_cable_listener.rb`:
```ruby
module Enterprise::ActionCableListener
  include Events::Types
  def copilot_message_created(event)
    copilot_message = event.data[:copilot_message]
    copilot_thread = copilot_message.copilot_thread
    account = copilot_thread.account
    user = copilot_thread.user

    broadcast(account, [user.pubsub_token], COPILOT_MESSAGE_CREATED, copilot_message.push_event_data)
  end
end
```

**扩展原理**：
1. 启动时 `ActionCableListener.prepend_mod_with('ActionCableListener')` 被调用
2. 查找 `Enterprise::ActionCableListener` 模块
3. 使用 Ruby `prepend` 将企业版模块插入祖先链
4. 效果：`ActionCableListener.ancestors` 变为：
   ```
   [Enterprise::ActionCableListener, ActionCableListener, BaseListener, ...]
   ```
5. OSS 版的 `broadcast` 方法可被企业版调用

**企业版扩展模式的优势**：
- **无侵入修改**：不需要修改 OSS 代码
- **方法可叠加**：企业版可以新增方法，也可以通过 `super` 覆盖 OSS 方法
- **延迟加载**：`to_prepare` 确保代码重载后重新注入

---

### 8. ActionCableListener 核心实现

`app/listeners/action_cable_listener.rb` 是连接事件系统和 ActionCable 广播的核心组件。

#### 8.1 事件处理方法示例

```ruby
def message_created(event)
  message, account = extract_message_and_account(event)
  conversation = message.conversation
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

#### 8.2 Token 计算逻辑

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

#### 8.3 核心广播方法

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

### 9. ActionCableBroadcastJob

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

### 10. Redis 在跨实例消息分发中的角色

#### 10.1 ActionCable Redis 适配器配置

`config/cable.yml`:
```yaml
default: &default
  adapter: redis
  url: <%= ENV.fetch('REDIS_URL', 'redis://127.0.0.1:6379') %>
  password: <%= ENV.fetch('REDIS_PASSWORD', nil).presence %>
  channel_prefix: <%= "chatwoot_#{Rails.env}_action_cable" %>
```

#### 10.2 自定义 Redis 连接

`config/initializers/actioncable.rb`:
```ruby
require 'action_cable/subscription_adapter/redis'

ActionCable::SubscriptionAdapter::Redis.redis_connector = lambda do |config|
  config[:id] = nil if ENV['REDIS_DISABLE_CLIENT_COMMAND'].present?
  Redis.new(config.except(:adapter, :channel_prefix))
end
```

#### 10.3 Redis Pub/Sub 工作原理

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

### 11. ActionCable 频道 (RoomChannel)

`app/channels/room_channel.rb` 是客户端订阅的唯一频道。

```ruby
class RoomChannel < ApplicationCable::Channel
  def subscribed
    # TODO: should we only do ensure stream  if current account is present?
    # for now going ahead with guard clauses in update_subscription and broadcast_presence
    current_user
    current_account
    ensure_stream
    update_subscription
    broadcast_presence
  end

  def update_presence
    update_subscription
    broadcast_presence
  end

  private

  def broadcast_presence
    return if @current_account.blank?

    data = { account_id: @current_account.id, users: ::OnlineStatusTracker.get_available_users(@current_account.id) }
    data[:contacts] = ::OnlineStatusTracker.get_available_contacts(@current_account.id) if @current_user.is_a? User
    ActionCable.server.broadcast(pubsub_token, { event: 'presence.update', data: data })
  end

  def ensure_stream
    stream_from pubsub_token
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

  def current_account
    return if current_user.blank?

    @current_account ||= if @current_user.is_a? Contact
                           @current_user.account
                         else
                           @current_user.accounts.find(params[:account_id])
                         end
  end
end
```

**Stream 订阅策略**：
- **所有用户**：订阅 `pubsub_token`（个人 stream）
- **客服（User）**：额外订阅 `account_{account_id}`（账户级 stream）
- **访客（Contact）**：只订阅个人 stream

---

### 12. 在线状态更新机制

#### 12.1 OnlineStatusTracker 核心实现

`lib/online_status_tracker.rb`:
```ruby
class OnlineStatusTracker
  # NOTE: You can customise the environment variable to keep your agents/contacts as online for longer
  PRESENCE_DURATION = ENV.fetch('PRESENCE_DURATION', 20).to_i.seconds
  # Widget pings every 60s, so contacts need a longer presence window
  CONTACT_PRESENCE_DURATION = ENV.fetch('CONTACT_PRESENCE_DURATION', 90).to_i.seconds

  # presence : sorted set with timestamp as the score & object id as value
  # online status : online | busy | offline
  # redis hash with obj_id key && status as value

  def self.update_presence(account_id, obj_type, obj_id)
    ::Redis::Alfred.zadd(presence_key(account_id, obj_type), Time.now.to_i, obj_id)
  end

  def self.get_available_users(account_id)
    user_ids = get_available_user_ids(account_id)
    return {} if user_ids.blank?

    user_availabilities = ::Redis::Alfred.hmget(status_key(account_id), user_ids)
    user_ids.map.with_index { |id, index| [id, (user_availabilities[index] || get_availability_from_db(account_id, id))] }.to_h
  end

  def self.get_available_user_ids(account_id)
    account = Account.find(account_id)
    range_start = (Time.zone.now - PRESENCE_DURATION).to_i
    user_ids = ::Redis::Alfred.zrangebyscore(presence_key(account_id, 'User'), range_start, '+inf')
    # since we are dealing with redis items as string, casting to string
    user_ids += account.account_users.where(auto_offline: false)&.map(&:user_id)&.map(&:to_s)
    user_ids.uniq
  end

  def self.get_available_contacts(account_id)
    get_available_contact_ids(account_id).index_with { |_id| 'online' }
  end

  def self.presence_key(account_id, type)
    case type
    when 'Contact'
      format(::Redis::Alfred::ONLINE_PRESENCE_CONTACTS, account_id: account_id)
    else
      format(::Redis::Alfred::ONLINE_PRESENCE_USERS, account_id: account_id)
    end
  end

  def self.status_key(account_id)
    format(::Redis::Alfred::ONLINE_STATUS, account_id: account_id)
  end
end
```

**Redis 数据结构**：
| Key 类型 | Redis 结构 | 用途 |
|---------|-----------|------|
| `ONLINE_PRESENCE_USERS::{account_id}` | Sorted Set | 客服在线 presence（score=timestamp, member=user_id） |
| `ONLINE_PRESENCE_CONTACTS::{account_id}` | Sorted Set | 访客在线 presence |
| `ONLINE_STATUS::{account_id}` | Hash | 客服可用性状态（online/busy/offline） |

**时间窗口配置**：
- **客服 (User)**：`PRESENCE_DURATION = 20s`（前端每 20s ping 一次）
- **访客 (Contact)**：`CONTACT_PRESENCE_DURATION = 90s`（Widget 每 60s ping 一次）

#### 12.2 前端定时心跳

`app/javascript/shared/helpers/BaseActionCableConnector.js`:
```javascript
const PRESENCE_INTERVAL = 20000;  // 20秒

class BaseActionCableConnector {
  constructor(...) {
    // ...
    this.triggerPresenceInterval = () => {
      setTimeout(() => {
        this.subscription.updatePresence();  // 调用 RoomChannel#update_presence
        this.triggerPresenceInterval();
      }, presenceInterval);
    };
    this.triggerPresenceInterval();
  }
}
```

**访客 Widget** 使用 60 秒间隔（`WIDGET_PRESENCE_INTERVAL = 60000`）。

#### 12.3 在线状态对多坐席同步的影响

**完整在线状态同步流程**：
```
客服A登录，建立WebSocket连接
    ↓
RoomChannel#subscribed
    ├── update_subscription → Redis ZADD 更新 presence
    └── broadcast_presence → 向客服A自己发送 presence.update
    ↓
客服A每20秒触发一次 updatePresence
    ├── update_subscription → 刷新 Redis 中的时间戳
    └── broadcast_presence → 向个人 stream 广播最新在线列表
    ↓
客服B收到 presence.update
    ↓
Store 更新 agents/updatePresence
    ↓
UI 显示客服A在线
```

**跨实例在线状态共享**：
```
客服A连接到 Rails 实例 A
    ↓
实例 A: ZADD ONLINE_PRESENCE_USERS::1 1715000000 123
    ↓
客服B连接到 Rails 实例 B
    ↓
实例 B: ZRANGEBYSCORE ONLINE_PRESENCE_USERS::1 (20秒前) +inf
    ↓
获取到 [123] (客服A的ID)
    ↓
客服B的 presence.update 包含客服A在线
```

**关键点**：
- 在线状态完全通过 Redis 共享，不依赖 Rails 实例
- 每个用户定时心跳刷新自己的时间戳
- `broadcast_presence` 是向 **个人 stream** 发送，不是全局广播
- 每个用户看到的在线列表是自己查询 Redis 后组装的

**潜在问题**：
1. **心跳间隔 vs 超时窗口**：客服心跳 20s，超时窗口 20s，网络延迟可能导致短暂显示离线
2. **无全局广播**：客服A下线后，客服B不会立即收到通知，需等待自己的下一次心跳（最多 20s）
3. **Redis Sorted Set 清理**：`get_available_user_ids` 每次查询时清理过期数据

---

### 13. 前端 WebSocket 连接

#### 13.1 基础连接器

`app/javascript/shared/helpers/BaseActionCableConnector.js`:
```javascript
import { createConsumer } from '@rails/actioncable';

const PRESENCE_INTERVAL = 20000;
const RECONNECT_INTERVAL = 1000;

class BaseActionCableConnector {
  static isDisconnected = false;

  constructor(
    app,
    pubsubToken,
    websocketHost = '',
    presenceInterval = PRESENCE_INTERVAL
  ) {
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
          BaseActionCableConnector.isDisconnected = true;
          this.onDisconnected();
          this.initReconnectTimer();
        },
      }
    );
    this.app = app;
    this.events = {};
    this.reconnectTimer = null;
    this.isAValidEvent = () => true;
    this.triggerPresenceInterval = () => {
      setTimeout(() => {
        this.subscription.updatePresence();
        this.triggerPresenceInterval();
      }, presenceInterval);
    };
    this.triggerPresenceInterval();
  }

  checkConnection() {
    const isConnectionActive = this.consumer.connection.isOpen();
    const isReconnected =
      BaseActionCableConnector.isDisconnected && isConnectionActive;
    if (isReconnected) {
      this.clearReconnectTimer();
      this.onReconnect();
      BaseActionCableConnector.isDisconnected = false;
    } else {
      this.initReconnectTimer();
    }
  }

  clearReconnectTimer = () => {
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = null;
    }
  };

  initReconnectTimer = () => {
    this.clearReconnectTimer();
    this.reconnectTimer = setTimeout(() => {
      this.checkConnection();
    }, RECONNECT_INTERVAL);
  };

  onReconnect = () => {};
  onDisconnected = () => {};

  disconnect() {
    this.consumer.disconnect();
  }

  onReceived = ({ event, data } = {}) => {
    if (this.isAValidEvent(data)) {
      if (this.events[event] && typeof this.events[event] === 'function') {
        this.events[event](data);
      }
    }
  };
}

export default BaseActionCableConnector;
```

#### 13.2 客服面板连接器

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
      'notification.created': this.onNotificationCreated,
      'conversation.mentioned': this.onConversationMentioned,
      'account.cache_invalidated': this.onCacheInvalidate,
      'copilot.message.created': this.onCopilotMessageCreated,
    };
  }

  onReconnect = () => {
    emitter.emit(BUS_EVENTS.WEBSOCKET_RECONNECT);
  };

  onDisconnected = () => {
    emitter.emit(BUS_EVENTS.WEBSOCKET_DISCONNECT);
  };

  onMessageCreated = data => {
    DashboardAudioNotificationHelper.onNewMessage(data);
    this.app.$store.dispatch('addMessage', data);
    this.app.$store.dispatch('updateConversationLastActivity', {
      lastActivityAt: data.conversation.last_activity_at,
      conversationId: data.conversation_id,
    });
  };

  onPresenceUpdate = data => {
    if (isImpersonating.value) return;
    this.app.$store.dispatch('contacts/updatePresence', data.contacts);
    this.app.$store.dispatch('agents/updatePresence', data.users);
    this.app.$store.dispatch('setCurrentUserAvailability', data.users);
  };
}
```

#### 13.3 访客 Widget 连接器

`app/javascript/widget/helpers/actionCable.js`:
```javascript
const WIDGET_PRESENCE_INTERVAL = 60000;

class ActionCableConnector extends BaseActionCableConnector {
  constructor(app, pubsubToken) {
    super(app, pubsubToken, '', WIDGET_PRESENCE_INTERVAL);
    
    this.events = {
      'message.created': this.onMessageCreated,
      'message.updated': this.onMessageUpdated,
      'conversation.typing_on': this.onTypingOn,
      'conversation.typing_off': this.onTypingOff,
      'conversation.status_changed': this.onStatusChange,
      'presence.update': this.onPresenceUpdate,
      'contact.merged': this.onContactMerge,
    };
  }

  onDisconnected = () => {
    this.setLastMessageId();
  };

  onReconnect = () => {
    this.syncLatestMessages();
  };

  setLastMessageId = () => {
    this.app.$store.dispatch('conversation/setLastMessageId');
  };

  syncLatestMessages = () => {
    this.app.$store.dispatch('conversation/syncLatestMessages');
  };
}
```

---

### 14. 前端重连机制

#### 14.1 基础连接器重连逻辑

**重连流程**：
```
WebSocket 连接断开
    ↓
disconnected 回调触发
    ↓
isDisconnected = true
    ↓
initReconnectTimer()
    ↓
setTimeout(checkConnection, 1000ms)
    ↓
checkConnection()
    ├── isConnectionActive?
    │   ├── YES → 调用 onReconnect() → 结束
    │   └── NO  → initReconnectTimer() → 循环
    ↓
ActionCable 内部自动重连 (由 @rails/actioncable 库处理)
```

#### 14.2 客服面板 vs 访客 Widget 重连策略

| 角色 | 断开时处理 | 重连后处理 | 消息丢失风险 |
|-----|-----------|-----------|-------------|
| 客服面板 | 发送 `WEBSOCKET_DISCONNECT` 事件 | 发送 `WEBSOCKET_RECONNECT` 事件 | 可能需要手动刷新 |
| 访客 Widget | 保存最后消息 ID (`setLastMessageId`) | HTTP API 补全 (`syncLatestMessages`) | **不会丢失** |

**客服面板重连影响**：
- 发送 `WEBSOCKET_RECONNECT` 事件通知全局
- 在线状态会通过 `triggerPresenceInterval` 自动恢复
- 但断开期间的消息可能丢失，需要页面刷新或其他机制补全

**访客 Widget 重连策略**：
1. **断开时**：保存当前最后一条消息 ID
2. **重连后**：调用 API 同步断开期间的消息
3. **保证消息不丢失**：通过 HTTP API 补全 WebSocket 断开期间的消息

**重连对多坐席同步的潜在问题**：
```
客服A发送消息
    ↓
客服B的WebSocket短暂断开 (网络抖动)
    ↓
Redis Pub/Sub 广播 MESSAGE_CREATED
    ↓
客服B的Rails实例收到消息，但找不到WebSocket连接
    ↓
消息丢失
    ↓
客服B重连成功
    ↓
客服B需要重新获取最新数据
```

---

### 15. 完整消息链路示例

让我们以"客服发送一条消息"为例，追踪完整的消息链路：

#### 步骤 1：客服发送消息

客服 A 在浏览器点击发送，消息通过 HTTP API 发送到后端。

#### 步骤 2：消息创建触发事件

`app/models/message.rb:137`
```ruby
after_create_commit :execute_after_create_commit_callbacks
```

事务提交后，`dispatch_create_events` 被调用：

```ruby
def dispatch_create_events
  Rails.configuration.dispatcher.dispatch(MESSAGE_CREATED, Time.zone.now, message: self, performed_by: Current.executed_by)
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

#### 步骤 4：SyncDispatcher 发布事件

`app/dispatchers/sync_dispatcher.rb:2-5`
```ruby
def dispatch(event_name, timestamp, data)
  event_object = Events::Base.new(event_name, timestamp, data)
  publish(event_object.method_name, event_object)  # publish(:message_created, event_object)
end
```

#### 步骤 5：ActionCableListener 处理

`app/listeners/action_cable_listener.rb:41-47`
```ruby
def message_created(event)
  message, account = extract_message_and_account(event)
  conversation = message.conversation
  tokens = user_tokens(account, conversation.inbox.members) + contact_tokens(conversation.contact_inbox, message)
  
  broadcast(account, tokens, MESSAGE_CREATED, message.push_event_data)
end
```

#### 步骤 6：创建广播 Job

`app/listeners/action_cable_listener.rb:221`
```ruby
::ActionCableBroadcastJob.perform_later(tokens.uniq, event_name, payload)
```

#### 步骤 7：ActionCableBroadcastJob 执行

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

#### 步骤 8：Redis Pub/Sub 分发

```
Rails 实例 A → Redis PUBLISH "chatwoot_production_action_cable:token_xxx" → Rails 实例 A, B, C (全部收到)
```

#### 步骤 9：各实例向本地 WebSocket 连接推送

- **实例 A**：有客服 B 的 WebSocket 连接 → 推送消息
- **实例 B**：有客服 C 的 WebSocket 连接 → 推送消息
- **实例 C**：有访客的 WebSocket 连接 → 推送消息

#### 步骤 10：浏览器接收并处理

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

### 16. 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `lib/events/types.rb` | 定义所有事件类型常量 |
| `lib/events/base.rb` | 事件对象基类，包含 method_name 转换 |
| `config/initializers/event_handlers.rb` | 监听器加载初始化 |
| `config/initializers/01_inject_enterprise_edition_module.rb` | 企业版模块注入机制 |
| `app/dispatchers/dispatcher.rb` | 事件分发中心（单例） |
| `app/dispatchers/sync_dispatcher.rb` | 同步事件分发器 |
| `app/dispatchers/async_dispatcher.rb` | 异步事件分发器 |
| `app/dispatchers/base_dispatcher.rb` | 分发器基类（集成 Wisper） |
| `app/listeners/action_cable_listener.rb` | ActionCable 事件监听器（OSS） |
| `enterprise/app/listeners/enterprise/action_cable_listener.rb` | ActionCable 监听器企业版扩展 |
| `app/jobs/action_cable_broadcast_job.rb` | ActionCable 广播后台任务 |
| `app/channels/room_channel.rb` | ActionCable 频道定义 |
| `lib/online_status_tracker.rb` | 在线状态追踪器（Redis 操作） |
| `config/cable.yml` | ActionCable 配置（Redis 适配器） |
| `config/initializers/actioncable.rb` | ActionCable Redis 连接器定制 |
| `app/javascript/shared/helpers/BaseActionCableConnector.js` | WebSocket 连接基础类（含重连逻辑） |
| `app/javascript/dashboard/helper/actionCable.js` | 客服面板 WebSocket 连接器 |
| `app/javascript/widget/helpers/actionCable.js` | 访客 Widget WebSocket 连接器 |

---

### 17. 设计亮点与最佳实践

#### 17.1 after_commit 确保数据一致性

- 使用 `after_create_commit` 而非 `after_create`
- 确保事件触发时数据已持久化到数据库
- 避免监听器看到未提交的数据

#### 17.2 同步 vs 异步分离

- **实时广播**（ActionCableListener）使用同步分发
- **耗时操作**（Webhook、自动化规则）使用异步分发，不阻塞请求

#### 17.3 防止事件乱序

`ActionCableBroadcastJob` 中对会话更新类事件重新获取最新数据：
```ruby
def prepare_broadcast_data(event_name, data)
  return data unless CONVERSATION_UPDATE_EVENTS.include?(event_name)
  
  account = Account.find(data[:account_id])
  conversation = account.conversations.find_by!(display_id: data[:id])
  conversation.push_event_data.merge(account_id: data[:account_id])
end
```

#### 17.4 Stream 粒度控制

- 个人 Stream：`pubsub_token`（精准推送）
- 账户 Stream：`account_{id}`（广播给所有客服）
- 访客只能订阅个人 Stream（安全隔离）

#### 17.5 Redis Pub/Sub 实现水平扩展

- 所有 Rails 实例共享同一个 Redis
- 任何实例触发的广播都会通过 Redis 到达所有实例
- 每个实例只负责向自己的 WebSocket 连接推送

#### 17.6 企业版无侵入扩展

- 使用 `prepend_mod_with` + Ruby `prepend` 实现
- OSS 代码无需修改
- 方法可叠加，支持新增和覆盖

#### 17.7 访客消息不丢失保障

- Widget 断开时记录最后消息 ID
- 重连后通过 HTTP API 同步缺失消息
- 不依赖 WebSocket 持久化

---

### 18. 总结

Chatwoot 的实时消息广播系统是一个精心设计的多层架构：

#### 核心链路

1. **事件驱动**：通过 `after_create_commit` 等回调触发事件（确保数据已持久化）
2. **事件名映射**：`'message.created'` → `message_created` 方法名转换
3. **监听器加载**：`config.to_prepare` 时订阅 Wisper listener
4. **发布-订阅**：使用 Wisper 实现灵活的事件监听（通过继承 `BaseDispatcher` 获得能力）
5. **分层分发**：同步（ActionCable/AgentBot）+ 异步（Webhook/自动化）两种策略
6. **企业扩展**：`prepend_mod_with` 注入企业版模块
7. **ActionCable**：Rails 原生 WebSocket 框架
8. **Redis Pub/Sub**：实现跨实例消息分发和在线状态共享
9. **精细流控**：基于 pubsub_token 的精准推送
10. **前端重连**：1秒轮询检测，Widget 支持消息补全
11. **在线状态**：Redis Sorted Set + 定时心跳，跨实例共享

#### 这种架构能够支持：

- 多客服坐席实时同步（消息、状态、输入状态）
- 水平扩展多个 Rails 实例（Redis 解耦）
- 高并发下的消息可靠性（事件乱序防护）
- 灵活的事件扩展（Wisper + 企业版 prepend）
- 网络不稳定环境下的消息不丢失（Widget 消息补全）

