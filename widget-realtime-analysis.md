# Chatwoot Widget 实时通信机制分析报告

## 1. 概述

本文档分析 Chatwoot 访客 Widget 从脚本加载到实时消息推送的完整技术架构，包括跨域会话建立、令牌认证机制、以及消息实时推送到坐席仪表盘的实现原理。

---

## 2. Widget 脚本加载与初始化流程

### 2.1 脚本嵌入方式

访客网站通过嵌入 Chatwoot 提供的 SDK 脚本来加载 Widget：

```javascript
<script>
  (function(d,t) {
    var BASE_URL="https://your-chatwoot-instance.com";
    var g=d.createElement(t),s=d.getElementsByTagName(t)[0];
    g.src=BASE_URL+"/packs/js/sdk.js";
    g.async = true;
    s.parentNode.insertBefore(g,s);
    g.onload=function(){
      window.chatwootSDK.run({
        websiteToken: 'your-website-token',
        baseUrl: BASE_URL
      })
    }
  })(document,"script");
</script>
```

**关键点**：
- `websiteToken`：唯一标识 WebWidget 渠道，由 Chatwoot 后台生成
- SDK 脚本异步加载，加载完成后调用 `chatwootSDK.run()` 初始化

### 2.2 Iframe 创建与 URL 构建

SDK 初始化时创建 iframe 并构建 Widget URL：

**文件**: `app/javascript/sdk/IFrameHelper.js:54-91`

```javascript
getUrl({ baseUrl, websiteToken }) {
  return `${baseUrl}/widget?website_token=${websiteToken}`;
},

createFrame: ({ baseUrl, websiteToken }) => {
  const cwCookie = Cookies.get('cw_conversation');
  let widgetUrl = IFrameHelper.getUrl({ baseUrl, websiteToken });
  if (cwCookie) {
    widgetUrl = `${widgetUrl}&cw_conversation=${cwCookie}`;
  }
  iframe.src = widgetUrl;
  // ...
}
```

**关键点**：
- 检查 `cw_conversation` cookie 是否存在
- 如果存在，将其作为 `cw_conversation` 参数附加到 URL
- 该 cookie 存储的是 JWT 格式的 `authToken`

### 2.3 后端初始化流程

WidgetsController 处理 `/widget` 路由请求：

**文件**: `app/controllers/widgets_controller.rb:1-97`

```ruby
class WidgetsController < ActionController::Base
  include WidgetHelper

  before_action :set_global_config
  before_action :set_web_widget
  before_action :ensure_account_is_active
  before_action :set_token
  before_action :set_contact
  before_action :build_contact
  after_action :allow_iframe_requests

  def set_web_widget
    @web_widget = ::Channel::WebWidget.find_by!(website_token: permitted_params[:website_token])
  end

  def set_token
    @token = permitted_params[:cw_conversation]
    @auth_token_params = if @token.present?
                           ::Widget::TokenService.new(token: @token).decode_token
                         else
                           {}
                         end
  end

  def set_contact
    return if @auth_token_params[:source_id].nil?
    @contact_inbox = ::ContactInbox.find_by(
      inbox_id: @web_widget.inbox.id,
      source_id: @auth_token_params[:source_id]
    )
    @contact = @contact_inbox&.contact
  end

  def build_contact
    return if @contact.present?
    @contact_inbox, @token = build_contact_inbox_with_token(@web_widget, additional_attributes)
    @contact = @contact_inbox.contact
  end
end
```

**初始化流程图**：
```
访客请求 /widget?website_token=xxx&cw_conversation=yyy
                    ↓
          set_web_widget (通过 website_token 查找 WebWidget)
                    ↓
            set_token (解码 cw_conversation JWT)
                    ↓
       set_contact (通过 source_id 查找 contact_inbox)
                    ↓
         ┌──────────┴──────────┐
         ↓                      ↓
    找到 contact_inbox      未找到或无 token
         ↓                      ↓
    使用现有身份        build_contact_inbox_with_token
         ↓                      ↓
         └──────────┬──────────┘
                    ↓
           渲染 show.html.erb
           注入全局变量:
           - window.chatwootWebChannel
           - window.authToken
           - window.chatwootPubsubToken
```

### 2.4 前端初始化

**文件**: `app/views/widgets/show.html.erb:1-43`

```erb
<script>
  window.chatwootWebChannel = {
    websiteToken: '<%= @web_widget.website_token %>',
    // ... 其他配置
  }
  window.chatwootPubsubToken = '<%= @contact_inbox.pubsub_token %>'
  window.authToken = '<%= @token %>'
  window.globalConfig = <%= raw @global_config.to_json %>
</script>
```

**文件**: `app/javascript/widget/App.vue:81-102`

```javascript
mounted() {
  const { websiteToken, locale, widgetColor } = window.chatwootWebChannel;
  this.setLocale(locale);
  this.setWidgetColor(widgetColor);
  this.setWidgetColorVariable(widgetColor);
  setHeader(window.authToken);  // 设置 API 请求头
  if (this.isIFrame) {
    this.registerListeners();
    this.sendLoadedEvent();
  } else {
    this.fetchOldConversations();
    this.fetchAvailableAgents(websiteToken);
    this.setLocale(getLocale(window.location.search));
  }
  // ...
}
```

---

## 3. 跨域会话建立机制

### 3.1 CORS 配置

**文件**: `config/initializers/cors.rb:1-35`

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '/packs/*', headers: :any, methods: [:get, :options]
    resource '/audio/*', headers: :any, methods: [:get, :options]
    resource '/public/api/*', headers: :any, methods: :any

    if ActiveModel::Type::Boolean.new.cast(ENV.fetch('CW_API_ONLY_SERVER', false)) || Rails.env.development?
      resource '*', headers: :any, methods: :any, expose: %w[access-token client uid expiry]
    end

    if ActiveModel::Type::Boolean.new.cast(ENV.fetch('ENABLE_API_CORS', false))
      resource '/api/*', headers: :any, methods: :any, expose: %w[access-token client uid expiry]
    end
  end
end
```

**关键点**：
- 静态资源 (`/packs/*`, `/audio/*`) 允许所有来源跨域访问
- 公共 API (`/public/api/*`) 允许所有来源
- `/api/*` 路径需要 `ENABLE_API_CORS` 环境变量启用 CORS

### 3.2 Iframe 嵌入权限控制

**文件**: `app/controllers/widgets_controller.rb:79-96`

```ruby
def allow_iframe_requests
  if @web_widget.allowed_domains.blank? || embedded_from_non_web_origin?
    response.headers.delete('X-Frame-Options')
  else
    domains = @web_widget.allowed_domains.split(',').map(&:strip).join(' ')
    response.headers['Content-Security-Policy'] = "frame-ancestors #{domains}"
  end
end

def embedded_from_non_web_origin?
  return false unless @web_widget.allow_mobile_webview?
  origin = request.headers['Origin']
  origin.blank? || origin == 'null' || origin&.start_with?('file://')
end
```

**关键点**：
- 如果配置了 `allowed_domains`，则使用 CSP `frame-ancestors` 限制可嵌入的域名
- 支持移动端 WebView（`file://`、`null` origin）
- 未配置时删除 `X-Frame-Options`，允许所有域名嵌入

### 3.3 JWT 令牌认证机制

Chatwoot 使用 **JWT (JSON Web Token)** 进行无状态认证，不依赖 Cookie（跨域 Cookie 受限）。

#### 3.3.1 令牌生成

**文件**: `app/helpers/widget_helper.rb:1-9`

```ruby
module WidgetHelper
  def build_contact_inbox_with_token(web_widget, additional_attributes = {})
    contact_inbox = web_widget.create_contact_inbox(additional_attributes)
    payload = { source_id: contact_inbox.source_id, inbox_id: web_widget.inbox.id }
    token = ::Widget::TokenService.new(payload: payload).generate_token

    [contact_inbox, token]
  end
end
```

**文件**: `app/services/widget/token_service.rb:1-27`

```ruby
class Widget::TokenService < BaseTokenService
  DEFAULT_EXPIRY_DAYS = 180

  def generate_token
    JWT.encode(token_payload, secret_key, algorithm)
  end

  private

  def token_payload
    (payload || {}).merge(exp: exp, iat: iat)
  end

  def exp
    iat + expire_in.days.to_i
  end

  def expire_in
    token_expiry_value = InstallationConfig.find_by(name: 'WIDGET_TOKEN_EXPIRY')&.value
    (token_expiry_value.presence || DEFAULT_EXPIRY_DAYS).to_i
  end
end
```

**文件**: `app/services/base_token_service.rb:1-27`

```ruby
class BaseTokenService
  pattr_initialize [:payload, :token]

  def generate_token
    JWT.encode(token_payload, secret_key, algorithm)
  end

  def decode_token
    JWT.decode(token, secret_key, true, algorithm: algorithm).first.symbolize_keys
  rescue JWT::ExpiredSignature, JWT::DecodeError
    {}
  end

  private

  def secret_key
    Rails.application.secret_key_base
  end

  def algorithm
    'HS256'
  end
end
```

**JWT Payload 结构**：
```json
{
  "source_id": "contact_inbox_source_id",
  "inbox_id": 123,
  "exp": 1777986000,
  "iat": 1762434000
}
```

**关键点**：
- 使用 HS256 算法，密钥为 `Rails.application.secret_key_base`
- 默认有效期 180 天，可通过 `WIDGET_TOKEN_EXPIRY` 配置修改
- 解码失败或过期时返回空 Hash，后端会创建新的访客身份

#### 3.3.2 API 请求认证

**文件**: `app/controllers/concerns/website_token_helper.rb:1-26`

```ruby
module WebsiteTokenHelper
  def auth_token_params
    @auth_token_params ||= ::Widget::TokenService.new(token: request.headers['X-Auth-Token']).decode_token
  end

  def set_web_widget
    @web_widget = ::Channel::WebWidget.find_by!(website_token: permitted_params[:website_token])
    @current_account = @web_widget.inbox.account
    # ...
  end

  def set_contact
    @contact_inbox = @web_widget.inbox.contact_inboxes.find_by(
      source_id: auth_token_params[:source_id]
    )
    @contact = @contact_inbox&.contact
    raise ActiveRecord::RecordNotFound unless @contact
    Current.contact = @contact
  end
end
```

**文件**: `app/javascript/widget/helpers/axios.js`（示意）

```javascript
import { setHeader } from 'widget/helpers/axios';

// 在 App.vue mounted 中调用
setHeader(window.authToken);

// 效果：所有 API 请求携带 X-Auth-Token 请求头
// headers: { 'X-Auth-Token': 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...' }
```

**认证流程**：
```
前端 API 请求
     ↓
请求头携带:
- X-Auth-Token: JWT authToken
- website_token: URL 参数或请求参数
     ↓
后端 WebsiteTokenHelper
     ↓
1. 通过 website_token 找到 WebWidget
2. 解码 X-Auth-Token 获取 source_id
3. 通过 source_id 找到 ContactInbox
4. 设置 Current.contact
     ↓
返回请求结果
```

### 3.4 Cookie 持久化

**文件**: `app/javascript/sdk/IFrameHelper.js:40-51`

```javascript
const updateAuthCookie = (cookieContent, baseDomain = '') =>
  setCookieWithDomain('cw_conversation', cookieContent, {
    baseDomain,
  });

// 在 loaded 事件中调用
events: {
  loaded: message => {
    updateAuthCookie(message.config.authToken, window.$chatwoot.baseDomain);
    // ...
  }
}
```

**关键点**：
- `cw_conversation` Cookie 存储在访客网站域名下（不是 Chatwoot 域名）
- 这样可以在访客刷新页面或导航到其他页面时保持身份
- Cookie 值就是 JWT authToken

---

## 4. 实时推送机制（WebSocket / Action Cable）

### 4.1 技术栈

- **Rails Action Cable**: Rails 内置的 WebSocket 框架
- **Pub/Sub 模式**: 通过 `pubsub_token` 标识订阅者

### 4.2 Pubsub Token 机制

**文件**: `app/models/concerns/pubsubable.rb:1-26`

```ruby
module Pubsubable
  extend ActiveSupport::Concern

  included do
    has_secure_token :pubsub_token
    before_save :rotate_pubsub_token
  end

  def rotate_pubsub_token
    return unless is_a?(User)
    # User 修改密码时轮换 pubsub_token
    self.pubsub_token = self.class.generate_unique_secure_token if will_save_change_to_encrypted_password?
  end

  def pubsub_token
    # 为现有记录回填 token
    regenerate_pubsub_token if self[:pubsub_token].blank? && persisted?
    self[:pubsub_token]
  end
end
```

**文件**: `app/models/contact_inbox.rb:23-34`

```ruby
class ContactInbox < ApplicationRecord
  include Pubsubable
  # ...
  belongs_to :contact
  belongs_to :inbox
  has_many :conversations, dependent: :destroy_async
end
```

**关键点**：
- `ContactInbox` 和 `User` 模型都包含 `Pubsubable` concern
- `pubsub_token` 是唯一的安全令牌，用于标识 WebSocket 订阅者
- 访客通过 `ContactInbox.pubsub_token` 订阅
- 坐席通过 `User.pubsub_token` 订阅

### 4.3 前端 WebSocket 连接

**文件**: `app/javascript/shared/helpers/BaseActionCableConnector.js:1-96`

```javascript
import { createConsumer } from '@rails/actioncable';

class BaseActionCableConnector {
  constructor(app, pubsubToken, websocketHost = '', presenceInterval = PRESENCE_INTERVAL) {
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
    // ...
    this.triggerPresenceInterval();  // 定期更新在线状态
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

**文件**: `app/javascript/widget/helpers/actionCable.js:1-143`

```javascript
class ActionCableConnector extends BaseActionCableConnector {
  constructor(app, pubsubToken) {
    super(app, pubsubToken, '', WIDGET_PRESENCE_INTERVAL);
    this.events = {
      'message.created': this.onMessageCreated,
      'message.updated': this.onMessageUpdated,
      'conversation.typing_on': this.onTypingOn,
      'conversation.typing_off': this.onTypingOff,
      'conversation.status_changed': this.onStatusChange,
      'conversation.created': this.onConversationCreated,
      'presence.update': this.onPresenceUpdate,
      'contact.merged': this.onContactMerge,
    };
  }

  onMessageCreated = data => {
    if (isMessageInActiveConversation(this.app.$store.getters, data)) {
      return;
    }
    this.app.$store
      .dispatch('conversation/addOrUpdateMessage', data)
      .then(() => emitter.emit(ON_AGENT_MESSAGE_RECEIVED));
    // 播放通知音等...
  };
  // ... 其他事件处理
}
```

**连接参数说明**：
| 参数 | 访客 Widget | 坐席 Dashboard |
|------|-------------|----------------|
| `channel` | `'RoomChannel'` | `'RoomChannel'` |
| `pubsub_token` | `ContactInbox.pubsub_token` | `User.pubsub_token` |
| `account_id` | 当前账户 ID | 当前账户 ID |
| `user_id` | 空 | 当前用户 ID |

### 4.4 服务端频道订阅

**文件**: `app/channels/room_channel.rb:1-59`

```ruby
class RoomChannel < ApplicationCable::Channel
  def subscribed
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

  def ensure_stream
    stream_from pubsub_token
    # 坐席额外订阅账户级别的广播
    stream_from "account_#{@current_account.id}" if @current_account.present? && @current_user.is_a?(User)
  end

  def pubsub_token
    @pubsub_token ||= params[:pubsub_token]
  end

  def current_user
    @current_user ||= if params[:user_id].blank?
                        # 访客：通过 pubsub_token 查找 ContactInbox，再获取 Contact
                        ContactInbox.find_by!(pubsub_token: pubsub_token).contact
                      else
                        # 坐席：通过 pubsub_token + user_id 查找 User
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

**订阅流程图**：
```
前端 WebSocket 连接
     ↓
连接参数:
- channel: 'RoomChannel'
- pubsub_token: 'xxx'
- (可选) user_id, account_id
     ↓
RoomChannel#subscribed
     ↓
current_user 识别:
├─ 有 user_id → User.find_by!(pubsub_token, user_id)
└─ 无 user_id → ContactInbox.find_by!(pubsub_token).contact
     ↓
ensure_stream:
├─ stream_from pubsub_token           # 个人订阅流
└─ 坐席额外: stream_from "account_#{id}"  # 账户级广播流
     ↓
订阅成功，等待消息
```

### 4.5 事件广播机制

#### 4.5.1 事件监听器

**文件**: `app/listeners/action_cable_listener.rb:1-225`

```ruby
class ActionCableListener < BaseListener
  include Events::Types

  def message_created(event)
    message, account = extract_message_and_account(event)
    conversation = message.conversation
    # 收集需要广播的 token
    tokens = user_tokens(account, conversation.inbox.members) + contact_tokens(conversation.contact_inbox, message)
    broadcast(account, tokens, MESSAGE_CREATED, message.push_event_data)
  end

  def conversation_typing_on(event)
    conversation = event.data[:conversation]
    account = conversation.account
    user = event.data[:user]
    tokens = typing_event_listener_tokens(account, conversation, user)
    broadcast(account, tokens, CONVERSATION_TYPING_ON, ...)
  end
  # ... 其他事件处理

  private

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
    # HMAC 验证的访客需要广播到所有已验证的 contact_inbox
    contact_inbox.hmac_verified? ? contact.contact_inboxes.where(hmac_verified: true).filter_map(&:pubsub_token) : [contact_inbox.pubsub_token]
  end

  def broadcast(account, tokens, event_name, data)
    return if tokens.blank?
    payload = data.merge(account_id: account.id)
    payload[:performer] = Current.user&.push_event_data if Current.user.present?
    # 异步广播
    ::ActionCableBroadcastJob.perform_later(tokens.uniq, event_name, payload)
  end
end
```

#### 4.5.2 消息创建时的广播流程

以 `message.created` 事件为例：

```
访客/坐席发送消息
     ↓
Message 创建
     ↓
触发事件: 'message.created'
     ↓
ActionCableListener#message_created
     ↓
收集广播目标 tokens:
├─ user_tokens: 对话所属 inbox 的所有成员（坐席）+ 管理员
└─ contact_tokens: 访客的 ContactInbox.pubsub_token（非私信、非活动消息）
     ↓
ActionCableBroadcastJob.perform_later(tokens, 'message.created', data)
     ↓
ActionCable.server.broadcast(token, { event: 'message.created', data: ... })
     ↓
所有订阅了该 token 的 WebSocket 连接收到消息
     ↓
前端 onMessageCreated 处理:
├─ Widget: 更新消息列表，播放通知音
└─ Dashboard: 更新对话列表，显示新消息提示
```

#### 4.5.3 事件类型汇总

| 事件类型 | 触发场景 | 广播目标 |
|---------|---------|---------|
| `message.created` | 新消息创建 | 坐席成员 + 访客 |
| `message.updated` | 消息更新 | 坐席成员 + 访客 |
| `conversation.created` | 新对话创建 | 坐席成员 + 访客 |
| `conversation.status_changed` | 对话状态变化 | 坐席成员 + 访客 |
| `conversation.typing_on` | 输入中 | 对方（排除自己） |
| `conversation.typing_off` | 输入结束 | 对方（排除自己） |
| `presence.update` | 在线状态更新 | 相关订阅者 |
| `assignee_changed` | 分配人变更 | 坐席成员 |
| `team_changed` | 团队变更 | 坐席成员 |

### 4.6 坐席仪表盘实时更新

坐席端的连接与 Widget 类似，但有以下区别：

1. **订阅参数**: 包含 `user_id`，后端识别为 `User` 而非 `Contact`
2. **额外订阅**: 坐席订阅 `account_#{account_id}` 流，接收账户级别的广播
3. **事件处理**: 坐席端处理更多事件类型（如分配变更、团队变更等）

---

## 5. 令牌与对话绑定机制

### 5.1 数据模型关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                           数据模型关系图                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Channel::WebWidget                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ website_token: 'abc123'  (唯一标识 Widget 渠道)              │   │
│  │ inbox_id: 1                  ───────────────────┐            │   │
│  │ hmac_token: 'xyz789'        (HMAC 验证密钥)    │            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                          │                            │
│                                          │ belongs_to                 │
│                                          ▼                            │
│  Inbox                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ id: 1                                                         │   │
│  │ account_id: 1                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                                             │
│         │ has_many :contact_inboxes                                  │
│         ▼                                                             │
│  ContactInbox  (关键: 连接 Contact 和 Inbox)                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ id: 101                                                       │   │
│  │ inbox_id: 1                  ───────────────────────────┐   │   │
│  │ contact_id: 200            ──────────────────────────┐ │   │   │
│  │ source_id: 'random_uuid_123'  (authToken 中的标识)   │ │   │   │
│  │ pubsub_token: 'pub_abc456'    (WebSocket 订阅令牌)   │ │   │   │
│  │ hmac_verified: false            (是否 HMAC 验证)      │ │   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│         │                                      │                      │
│         │ belongs_to :contact                  │ has_many             │
│         ▼                                      ▼                      │
│  Contact                              Conversations                  │
│  ┌─────────────────────┐            ┌─────────────────────────┐   │
│  │ id: 200             │            │ id: 301                 │   │
│  │ account_id: 1       │            │ contact_inbox_id: 101   │   │
│  │ name: 'Visitor'     │◄───────────│ inbox_id: 1             │   │
│  │ email: '...'        │            │ status: 'open'          │   │
│  └─────────────────────┘            └─────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 令牌类型说明

| 令牌名称 | 存储位置 | 用途 | 有效期 |
|---------|---------|------|--------|
| `website_token` | `channel_web_widgets` 表 | 标识 Widget 渠道（inbox） | 永久（可重置） |
| `authToken` (JWT) | 不存储，动态生成 | API 请求认证，包含 `source_id` | 默认 180 天 |
| `source_id` | `contact_inboxes` 表 | ContactInbox 的唯一标识 | 永久 |
| `pubsub_token` | `contact_inboxes` / `users` 表 | WebSocket 订阅标识 | 永久（User 改密码时轮换） |
| `hmac_token` | `channel_web_widgets` 表 | HMAC 身份验证签名密钥 | 永久（可重置） |
| `cw_conversation` Cookie | 访客浏览器 | 存储 `authToken`，跨页面保持身份 | 同 `authToken` |

### 5.3 令牌流转详解

#### 5.3.1 新访客首次访问

```
访客首次访问网站，无 cw_conversation Cookie
                    ↓
SDK 创建 iframe，URL: /widget?website_token=abc123
                    ↓
WidgetsController#set_token
  @auth_token_params = {}  (无 cw_conversation 参数)
                    ↓
WidgetsController#set_contact
  @contact = nil  (无 source_id)
                    ↓
WidgetsController#build_contact
  @contact_inbox, @token = build_contact_inbox_with_token(@web_widget, ...)
                    ↓
WidgetHelper#build_contact_inbox_with_token:
  1. 创建新的 Contact (访客)
  2. 创建新的 ContactInbox:
     - source_id: 自动生成的 UUID
     - pubsub_token: 自动生成的安全令牌
  3. 生成 JWT authToken:
     payload = { source_id: contact_inbox.source_id, inbox_id: inbox.id }
     token = JWT.encode(payload, secret_key_base, 'HS256')
                    ↓
渲染 show.html.erb，注入:
  window.authToken = 'eyJhbGciOiJIUzI1NiIs...'
  window.chatwootPubsubToken = 'pub_abc456'
                    ↓
SDK 收到 loaded 事件:
  updateAuthCookie(authToken, baseDomain)
  → 设置 cw_conversation Cookie
```

#### 5.3.2 回访客（有 Cookie）

```
访客再次访问，有 cw_conversation Cookie
                    ↓
SDK 创建 iframe，URL: /widget?website_token=abc123&cw_conversation=eyJ...
                    ↓
WidgetsController#set_token:
  @token = params[:cw_conversation]
  @auth_token_params = TokenService.decode_token(@token)
    → { source_id: 'random_uuid_123', inbox_id: 1, exp: ..., iat: ... }
                    ↓
WidgetsController#set_contact:
  @contact_inbox = ContactInbox.find_by(
    inbox_id: @web_widget.inbox.id,
    source_id: @auth_token_params[:source_id]
  )
  @contact = @contact_inbox.contact
                    ↓
找到现有身份，无需创建
                    ↓
渲染页面，使用现有的 contact_inbox.pubsub_token
                    ↓
访客可以看到历史对话记录
```

#### 5.3.3 API 请求认证流程

```
访客发送消息: POST /api/v1/widget/messages
                    ↓
请求头:
  X-Auth-Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  (或 URL 参数: website_token=abc123)
                    ↓
Api::V1::Widget::BaseController (before_action):
  1. set_web_widget: 通过 website_token 找到 WebWidget
  2. set_contact:
     - auth_token_params = decode X-Auth-Token → { source_id: ... }
     - @contact_inbox = find_by(source_id: auth_token_params[:source_id])
     - @contact = @contact_inbox.contact
                    ↓
认证成功，Current.contact 已设置
                    ↓
MessagesController#create:
  @conversation = conversation  (属于 @contact_inbox)
  @message = @conversation.messages.new(message_params)
  @message.save!
                    ↓
触发 message.created 事件
                    ↓
ActionCableListener 广播到:
  - 坐席: conversation.inbox.members 的 pubsub_token
  - 访客: @contact_inbox.pubsub_token
                    ↓
实时推送到相关客户端
```

### 5.4 对话创建与绑定

**文件**: `app/controllers/api/v1/widget/base_controller.rb:19-46`

```ruby
def conversations
  if @contact_inbox.hmac_verified?
    verified_contact_inbox_ids = @contact.contact_inboxes.where(inbox_id: auth_token_params[:inbox_id], hmac_verified: true).map(&:id)
    @conversations = @contact.conversations.where(contact_inbox_id: verified_contact_inbox_ids)
  else
    @conversations = @contact_inbox.conversations.where(inbox_id: auth_token_params[:inbox_id])
  end
end

def conversation
  @conversation ||= conversations.last
end

def create_conversation
  ::Conversation.create!(conversation_params)
end

def conversation_params
  {
    account_id: inbox.account_id,
    inbox_id: inbox.id,
    contact_id: @contact.id,
    contact_inbox_id: @contact_inbox.id,  # 关键绑定
    additional_attributes: { ... },
    custom_attributes: permitted_params[:custom_attributes].presence || {}
  }
end
```

**关键点**：
- 新对话的 `contact_inbox_id` 绑定到当前的 `@contact_inbox`
- 这建立了对话与访客身份的关联
- HMAC 验证的访客可以访问同一 Contact 下所有已验证的 ContactInbox 的对话

### 5.5 HMAC 身份验证（可选增强）

当 Widget 启用 `hmac_mandatory` 时，需要使用 HMAC 签名来验证访客身份：

**文件**: `app/models/channel/web_widget.rb:47-48`

```ruby
has_secure_token :website_token
has_secure_token :hmac_token
```

**流程**：
```
客户网站后端:
  user_identifier = 'user@example.com'  # 或其他唯一标识
  hmac_hash = OpenSSL::HMAC.hexdigest(
    'sha256',
    web_widget.hmac_token,
    user_identifier
  )

前端 SDK:
  window.chatwootSDK.run({
    websiteToken: 'abc123',
    baseUrl: '...',
    user: {
      identifier: 'user@example.com',
      identifier_hash: hmac_hash  # HMAC 签名
    }
  })

Chatwoot 后端:
  验证 identifier_hash 是否正确
  正确则标记 contact_inbox.hmac_verified = true
```

**优势**：
- 防止访客身份冒用
- 可以安全地关联访客与客户系统的用户账户
- 支持跨设备、跨浏览器的身份同步

---

## 6. 关键代码位置汇总

### 6.1 后端代码

| 功能 | 文件路径 |
|-----|---------|
| Widget 页面控制器 | `app/controllers/widgets_controller.rb` |
| Widget API 基础控制器 | `app/controllers/api/v1/widget/base_controller.rb` |
| 消息 API 控制器 | `app/controllers/api/v1/widget/messages_controller.rb` |
| Token 认证辅助 | `app/controllers/concerns/website_token_helper.rb` |
| JWT 令牌服务 | `app/services/widget/token_service.rb` |
| 基础令牌服务 | `app/services/base_token_service.rb` |
| Widget 辅助方法 | `app/helpers/widget_helper.rb` |
| WebSocket 频道 | `app/channels/room_channel.rb` |
| 实时事件监听器 | `app/listeners/action_cable_listener.rb` |
| CORS 配置 | `config/initializers/cors.rb` |
| Pubsub Token 机制 | `app/models/concerns/pubsubable.rb` |
| ContactInbox 模型 | `app/models/contact_inbox.rb` |
| WebWidget 模型 | `app/models/channel/web_widget.rb` |

### 6.2 前端代码

| 功能 | 文件路径 |
|-----|---------|
| Widget 主应用 | `app/javascript/widget/App.vue` |
| SDK Iframe 管理 | `app/javascript/sdk/IFrameHelper.js` |
| Widget Action Cable 连接 | `app/javascript/widget/helpers/actionCable.js` |
| 基础 Action Cable 连接器 | `app/javascript/shared/helpers/BaseActionCableConnector.js` |
| Axios 配置（请求头） | `app/javascript/widget/helpers/axios.js` |
| Widget 视图模板 | `app/views/widgets/show.html.erb` |

---

## 7. 安全考虑

### 7.1 令牌安全

1. **JWT 签名**: 使用 `Rails.application.secret_key_base` 作为 HS256 密钥
2. **有效期**: 默认 180 天，可配置缩短
3. **解码失败处理**: 过期或无效的 token 返回 `{}`，后端创建新身份

### 7.2 跨域安全

1. **CORS 配置**: 仅允许必要的路径跨域
2. **Iframe 嵌入**: 支持 `allowed_domains` 白名单配置
3. **不依赖 Cookie**: 使用 JWT 请求头避免跨域 Cookie 限制

### 7.3 HMAC 增强验证

- 可选但推荐的安全增强
- 防止访客身份冒用
- 适合需要身份关联的场景

---

## 8. 总结

Chatwoot Widget 的实时通信架构采用了以下核心设计：

1. **无状态认证**: JWT authToken 包含 `source_id` 和 `inbox_id`，不依赖服务器会话
2. **双令牌机制**:
   - `authToken`: 用于 HTTP API 认证
   - `pubsub_token`: 用于 WebSocket 实时订阅
3. **Iframe + PostMessage**: 解决跨域 DOM 交互问题
4. **Action Cable**: Rails 原生 WebSocket 方案，实现实时消息推送
5. **事件驱动**: 通过监听器模式将数据变更广播到相关订阅者

这种架构既解决了跨域通信的技术挑战，又保证了实时消息的可靠传递，同时通过令牌机制实现了访客身份的持久化和安全认证。
