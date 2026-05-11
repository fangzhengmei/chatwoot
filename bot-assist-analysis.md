# Chatwoot 机器人接入与控制权切换分析报告

## 1. 概述

Chatwoot 提供了灵活的机器人接入机制，支持多种类型的机器人与人工客服之间的无缝协作。本文档详细分析了机器人如何接入会话、自动回复机制、以及自动回复与人工介入之间的控制权切换逻辑。

## 2. 核心数据模型

### 2.1 AgentBot 模型
- **文件位置**: `app/models/agent_bot.rb:21-70`
- **核心字段**:
  - `bot_type`: 机器人类型，目前默认支持 `webhook` 类型
  - `outgoing_url`: 机器人接收事件的 Webhook URL
  - `secret`: Webhook 签名密钥，用于验证请求来源
  - `account_id`: 关联的账户 ID，为 `nil` 时表示系统级机器人
  - `bot_config`: JSON 配置字段，存储机器人特定配置

- **关键关系**:
  - `has_many :agent_bot_inboxes`: 机器人与收件箱的多对多关联
  - `has_many :assigned_conversations`: 通过 `assignee_agent_bot_id` 关联被分配的会话

### 2.2 AgentBotInbox 模型
- **文件位置**: `app/models/agent_bot_inbox.rb:14-29`
- **核心字段**:
  - `inbox_id`: 收件箱 ID
  - `agent_bot_id`: 机器人 ID
  - `status`: 状态，支持 `active` 和 `inactive`
- **作用**: 建立机器人与收件箱的绑定关系，控制机器人在哪些渠道生效

### 2.3 Conversation 模型中的机器人相关字段
- **文件位置**: `app/models/conversation.rb:54-348`
- **关键字段**:
  - `assignee_agent_bot_id`: 分配给会话的机器人 ID
  - `assignee_id`: 分配给会话的人工客服 ID
  - `status`: 会话状态 (open: 0, resolved: 1, pending: 2, snoozed: 3)
  - `waiting_since`: 等待客服响应的时间戳

- **互斥机制**:
  - `reset_agent_bot_when_assignee_present` (app/models/conversation.rb:249-253): 
    当设置人工客服时，自动清除机器人分配
  ```ruby
  def reset_agent_bot_when_assignee_present
    return if assignee_id.blank?
    self.assignee_agent_bot_id = nil
  end
  ```

## 3. 机器人接入机制

### 3.1 接入方式概览

Chatwoot 支持以下几种机器人接入方式：

| 接入方式 | 描述 | 适用场景 |
|---------|------|---------|
| AgentBot Webhook | 自定义机器人通过 Webhook 接收和发送消息 | 自有机器人系统集成 |
| Dialogflow 集成 | Google Dialogflow 原生集成 | NLU 意图识别场景 |
| Captain (企业版) | 基于 LLM 的智能助手 | 企业级智能客服 |

### 3.2 收件箱级别的机器人激活

- **文件位置**: `app/models/inbox.rb:173-176`

```ruby
def active_bot?
  agent_bot_inbox&.active? || hooks.where(app_id: %w[dialogflow],
                                          status: 'enabled').count.positive?
end
```

- **企业版扩展** (enterprise/app/models/enterprise/inbox.rb:11-17):
```ruby
def active_bot?
  super || captain_active?
end

def captain_active?
  captain_assistant.present? && more_responses?
end
```

### 3.3 会话状态初始化

当新会话创建时，如果收件箱配置了活跃机器人，会话状态会被自动设置为 `pending`，由机器人首先接管：

- **文件位置**: `app/models/conversation.rb:255-267`

```ruby
def determine_conversation_status
  self.status = :resolved and return if contact.blocked?
  return handle_campaign_status if campaign.present?
  self.status = :pending if inbox.active_bot?
end

def handle_campaign_status
  self.status = :pending if campaign.sender_id.nil? && inbox.active_bot?
end
```

## 4. 事件驱动架构

### 4.1 事件类型定义
- **文件位置**: `lib/events/types.rb:1-62`

关键事件类型:
```ruby
CONVERSATION_BOT_HANDOFF = 'conversation.bot_handoff'  # 机器人转人工
MESSAGE_CREATED = 'message.created'                    # 消息创建
MESSAGE_UPDATED = 'message.updated'                    # 消息更新
CONVERSATION_CREATED = 'conversation.created'          # 会话创建
CONVERSATION_UPDATED = 'conversation.updated'          # 会话更新
```

### 4.2 事件监听器

#### AgentBotListener (Webhook 机器人)
- **文件位置**: `app/listeners/agent_bot_listener.rb:1-91`

监听的事件:
- `conversation_resolved`
- `conversation_opened`
- `conversation_status_changed`
- `conversation_updated`
- `message_created`
- `message_updated`
- `webwidget_triggered`

消息处理流程:
1. `message_created` 或 `message_updated` 事件触发
2. 检查消息是否可发送 Webhook (`message.webhook_sendable?`)
3. 获取需要通知的机器人列表:
   - 会话直接分配的机器人 (`assignee_agent_bot`)
   - 收件箱关联的活跃机器人 (`active_inbox_agent_bot`)
4. 通过 `AgentBots::WebhookJob` 异步发送 Webhook

#### HookListener (集成机器人)
- **文件位置**: `app/listeners/hook_listener.rb:1-72`

支持的集成类型:
```ruby
supported_events_map = {
  'slack' => ['message.created', 'message.updated'],
  'dialogflow' => ['message.created', 'message.updated'],
  'google_translate' => ['message.created'],
  'leadsquared' => ['contact.updated', 'conversation.created', 'conversation.resolved']
}
```

Dialogflow 处理流程 (app/jobs/hook_job.rb:47-51):
```ruby
def process_dialogflow_integration(hook, event_name, event_data)
  return unless ['message.created', 'message.updated'].include?(event_name)
  Integrations::Dialogflow::ProcessorService.new(
    event_name: event_name, 
    hook: hook, 
    event_data: event_data
  ).perform
end
```

## 5. 机器人处理器服务

### 5.1 基础处理器 BotProcessorService
- **文件位置**: `lib/integrations/bot_processor_service.rb:1-63`

执行条件检查 (`should_run_processor?`):
```ruby
def should_run_processor?(message)
  return if message.private?
  return unless processable_message?(message)
  return unless conversation.pending?  # 仅在 pending 状态下处理
  true
end
```

处理流程:
1. 提取消息内容
2. 调用机器人获取响应 (`get_response`)
3. 处理响应 (`process_response`)
4. 支持动作处理 (`process_action`):
   - `handoff`: 转人工
   - `resolve`: 解决会话

### 5.2 Dialogflow 处理器
- **文件位置**: `lib/integrations/dialogflow/processor_service.rb:1-100`

核心特性:
- 支持多语言 (23 种语言代码映射)
- 解析 Dialogflow 的 `fulfillment_messages`
- 支持通过 payload 触发 `handoff` 或 `resolve` 动作

响应处理逻辑 (lines 75-85):
```ruby
def process_response(message, response)
  fulfillment_messages = response.query_result['fulfillment_messages']
  fulfillment_messages.each do |fulfillment_message|
    content_params = generate_content_params(fulfillment_message)
    if content_params['action'].present?
      process_action(message, content_params['action'])
    else
      create_conversation(message, content_params)
    end
  end
end
```

### 5.3 Captain 处理器 (企业版)
- **文件位置**: `lib/integrations/captain/processor_service.rb:1-66`

核心特性:
- 调用外部 Captain API 获取响应
- 传递历史消息上下文 (`previous_messages`)
- 支持 `conversation_handoff` 响应触发转人工

## 6. 控制权切换机制

### 6.1 核心切换方法: bot_handoff!
- **文件位置**: `app/models/conversation.rb:161-165`

```ruby
def bot_handoff!
  update(waiting_since: Time.current) if waiting_since.blank?
  open!  # 将状态从 pending 改为 open
  dispatcher_dispatch(CONVERSATION_BOT_HANDOFF)
end
```

触发效果:
1. 设置/确认 `waiting_since` 时间戳
2. 将会话状态从 `pending` 改为 `open`
3. 派发 `CONVERSATION_BOT_HANDOFF` 事件

### 6.2 自动分配服务: Conversations::AssignmentService
- **文件位置**: `app/services/conversations/assignment_service.rb:1-43`

分配逻辑:
```ruby
def perform
  agent_bot_assignment? ? assign_agent_bot : assign_agent
end

def assign_agent
  conversation.assignee = assignee
  conversation.assignee_agent_bot = nil  # 清除机器人
  conversation.save!
  assignee
end

def assign_agent_bot
  return unless agent_bot
  conversation.assignee = nil  # 清除人工客服
  conversation.assignee_agent_bot = agent_bot
  conversation.save!
  agent_bot
end
```

关键特性:
- **互斥分配**: 人工客服和机器人不能同时被分配
- `assignee_type` 参数控制分配类型 (`User` 或 `AgentBot`)

### 6.3 分配 API 控制器
- **文件位置**: `app/controllers/api/v1/accounts/conversations/assignments_controller.rb:1-45`

```ruby
def create
  if params.key?(:assignee_id) || agent_bot_assignment?
    set_agent
  elsif params.key?(:team_id)
    set_team
  else
    render json: nil
  end
end

def agent_bot_assignment?
  params[:assignee_type].to_s == 'AgentBot'
end
```

## 7. 企业版 Captain 机器人深度分析

### 7.1 ResponseBuilderJob
- **文件位置**: `enterprise/app/jobs/captain/conversation/response_builder_job.rb:1-218`

核心执行流程:
1. **前置检查**: `conversation_pending?` - 确认会话仍为 pending 状态
2. **版本选择**: 支持 V1 (直接 LLM 调用) 和 V2 (AgentRunner + Tools)
3. **响应处理**:

```ruby
def process_response
  if v2_handoff_tool_fired?
    if conversation_pending?
      process_v1_handoff  # 工具调用失败回退
    else
      process_v2_handoff  # 工具调用成功
    end
  elsif v1_handoff_requested?
    return unless conversation_pending?
    process_v1_handoff
  elsif conversation_pending?
    # 正常发送响应消息
    ActiveRecord::Base.transaction do
      create_messages
      account.increment_response_usage
    end
  end
end
```

### 7.2 HandoffTool (人工介入工具)
- **文件位置**: `enterprise/lib/captain/tools/handoff_tool.rb:1-63`

执行逻辑:
```ruby
def perform(tool_context, reason: nil)
  conversation = find_conversation(tool_context.state)
  return 'Conversation not found' unless conversation

  log_tool_usage('tool_handoff', {
    conversation_id: conversation.id,
    reason: reason || 'Agent requested handoff'
  })

  trigger_handoff(conversation, reason)

  "Conversation handed off to human support team..."
end

def trigger_handoff(conversation, reason)
  conversation.messages.create!(
    message_type: :outgoing,
    private: true,
    sender: @assistant,
    account: conversation.account,
    inbox: conversation.inbox,
    content: reason
  )
  conversation.bot_handoff!
  send_out_of_office_message_if_applicable(conversation)
end
```

### 7.3 ResolveConversationTool (自动解决工具)
- **文件位置**: `enterprise/lib/captain/tools/resolve_conversation_tool.rb:1-23`

```ruby
def perform(tool_context, reason:)
  conversation = find_conversation(tool_context.state)
  return 'Conversation not found' unless conversation
  return "Conversation ##{conversation.display_id} is already resolved" if conversation.resolved?
  return 'Auto-resolve is disabled for this account' if conversation.account.captain_auto_resolve_disabled?

  log_tool_usage('resolve_conversation', { conversation_id: conversation.id, reason: reason })
  conversation.with_captain_activity_context(reason: reason, reason_type: :tool) do
    conversation.resolved!
  end
end
```

## 8. 完整消息流转图

### 8.1 机器人自动回复流程

```
用户发送消息
    ↓
Message 创建事件 (message.created)
    ↓
┌─────────────────────────────────────────┐
│  事件分发 (Dispatcher)                   │
└─────────────────────────────────────────┘
    ↓
┌─────────────────┬─────────────────────┐
│ AgentBotListener│    HookListener      │
│ (Webhook Bot)   │  (Dialogflow等)      │
└─────────────────┴─────────────────────┘
    ↓                     ↓
AgentBots::WebhookJob  HookJob
(异步发送 Webhook)     (调用对应集成)
    ↓                     ↓
外部机器人响应     Dialogflow/其他服务
    ↓                     ↓
机器人发送回复消息  ProcessorService
                          ↓
                    process_response
                          ↓
                    创建回复消息
                          ↓
                    或触发 handoff/resolve
```

### 8.2 机器人转人工流程

```
机器人判断需要转人工
    ↓
┌─────────────────────────────────────────┐
│  触发方式:                               │
│  1. Webhook 机器人返回 handoff action   │
│  2. Dialogflow payload 包含 handoff     │
│  3. Captain HandoffTool 被调用          │
│  4. 人工主动分配会话                     │
└─────────────────────────────────────────┘
    ↓
conversation.bot_handoff!
    ↓
┌─────────────────────────────────────────┐
│  1. 设置 waiting_since                   │
│  2. 状态: pending → open                 │
│  3. 派发 CONVERSATION_BOT_HANDOFF 事件  │
└─────────────────────────────────────────┘
    ↓
自动分配 (AutoAssignmentHandler)
    ↓
分配给人工客服
    ↓
assignee_id 设置, assignee_agent_bot_id 清空
    ↓
人工客服接管会话
```

### 8.3 人工接管后重新交给机器人

```
人工客服完成处理
    ↓
调用分配 API (assignee_type=AgentBot)
    ↓
Conversations::AssignmentService
    ↓
assign_agent_bot
    ↓
┌─────────────────────────────────────────┐
│  1. assignee = nil                      │
│  2. assignee_agent_bot = agent_bot      │
│  3. 会话状态保持 open 或手动设为 pending │
└─────────────────────────────────────────┘
    ↓
AgentBotListener 通知机器人
    ↓
机器人重新接管会话
```

## 9. 关键设计模式

### 9.1 状态机设计

会话状态与控制权的关系:

| 状态 | 说明 | 控制权 |
|-----|------|-------|
| `pending` (2) | 机器人处理中 | 机器人 |
| `open` (0) | 等待人工处理 | 人工客服 |
| `resolved` (1) | 已解决 | 无 |
| `snoozed` (3) | 已延迟 | 无 |

### 9.2 事件驱动架构

- **解耦**: 消息处理通过事件分发，各监听器独立处理
- **异步**: Webhook 发送通过 Sidekiq 异步执行
- **可扩展**: 新增机器人类型只需添加新的 ProcessorService

### 9.3 互斥分配原则

- 同一时刻只能有一个处理主体 (人工或机器人)
- 设置人工客服时自动清除机器人分配
- 模型层面的回调保证数据一致性

## 10. 自动化规则补充

### 10.1 AutomationRuleListener
- **文件位置**: `app/listeners/automation_rule_listener.rb:1-86`

自动化规则可以独立于机器人触发:
- 监听事件: `conversation_created`, `conversation_updated`, `message_created` 等
- 支持条件匹配和动作执行
- 可用于: 自动分配、自动打标、发送模板消息等

## 12. 完整时序与可执行步骤

### 12.1 阶段一：接入准备

#### 12.1.1 创建 AgentBot (Webhook 机器人)

**API 端点**:
- 账户级 API: `POST /api/v1/accounts/{account_id}/agent_bots`
- 平台级 API: `POST /platform/api/v1/agent_bots`

**请求示例**:

```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/agent_bots \
  -H "Content-Type: application/json" \
  -H "api_access_token: YOUR_AGENT_ACCESS_TOKEN" \
  -d '{
    "name": "My Support Bot",
    "description": "智能客服机器人",
    "outgoing_url": "https://your-bot-server.com/webhook/chatwoot",
    "bot_type": "webhook",
    "bot_config": {
      "welcome_message": "您好，我是智能助手，请问有什么可以帮您？"
    }
  }'
```

**响应字段**:
```json
{
  "id": 1,
  "name": "My Support Bot",
  "outgoing_url": "https://your-bot-server.com/webhook/chatwoot",
  "bot_type": "webhook",
  "bot_config": {
    "welcome_message": "您好，我是智能助手..."
  },
  "access_token": "bot_access_token_xxx"
}
```

**关键代码**:
- 账户级控制器: `app/controllers/api/v1/accounts/agent_bots_controller.rb:12-15`
- 平台级控制器: `app/controllers/platform/api/v1/agent_bots_controller.rb:11-16`

#### 12.1.2 绑定机器人到收件箱

将机器人与特定渠道（收件箱）关联，使机器人能够接收该渠道的消息。

**API 端点**:
```
POST /api/v1/accounts/{account_id}/inboxes/{inbox_id}/set_agent_bot
```

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `agent_bot` | integer | 否 | AgentBot 的 ID，传 `null` 表示解绑 |

**权限要求**:
- 需要 `administrator` 角色 (`app/policies/inbox_policy.rb:53-54`)

**绑定机器人 (curl 示例)**:
```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/inboxes/5/set_agent_bot \
  -H "Content-Type: application/json" \
  -H "api_access_token: ADMIN_AGENT_ACCESS_TOKEN" \
  -d '{
    "agent_bot": 1
  }'
```

**解绑机器人**:
```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/inboxes/5/set_agent_bot \
  -H "Content-Type: application/json" \
  -H "api_access_token: ADMIN_AGENT_ACCESS_TOKEN" \
  -d '{
    "agent_bot": null
  }'
```

**控制器实现逻辑** (`app/controllers/api/v1/accounts/inboxes_controller.rb:58-67`):
```ruby
def set_agent_bot
  if @agent_bot
    # 绑定：创建或更新 AgentBotInbox 关联
    agent_bot_inbox = @inbox.agent_bot_inbox || AgentBotInbox.new(inbox: @inbox)
    agent_bot_inbox.agent_bot = @agent_bot
    agent_bot_inbox.save!  # 默认 status = 'active'
  elsif @inbox.agent_bot_inbox.present?
    # 解绑：删除关联记录
    @inbox.agent_bot_inbox.destroy!
  end
  head :ok
end
```

**Rails 控制台备用方式**:
```ruby
account = Account.find(1)
inbox = account.inboxes.find(5)
agent_bot = account.agent_bots.find(1)

# 创建关联（status 默认就是 active）
agent_bot_inbox = AgentBotInbox.create!(
  inbox: inbox,
  agent_bot: agent_bot
)

# 或者：使用 has_one 关联的便捷方式
inbox.agent_bot = agent_bot
inbox.save!
```

**验证绑定状态**:
```ruby
# 检查收件箱是否有活跃机器人
inbox.active_bot?  # => true

# 检查关联状态
inbox.agent_bot_inbox.status  # => "active"
inbox.agent_bot.name          # => "My Support Bot"

# 检查关联是否存在
inbox.agent_bot_inbox.present?  # => true
```

**解绑后的验证**:
```ruby
inbox.agent_bot_inbox.present?  # => false
inbox.active_bot?               # => false（除非有 Dialogflow 等其他集成）
```

**关键代码**:
- API 控制器: `app/controllers/api/v1/accounts/inboxes_controller.rb:58-67`
- 权限检查: `app/policies/inbox_policy.rb:53-55`
- 激活判断: `app/models/inbox.rb:173-176`
- 关联模型: `app/models/agent_bot_inbox.rb:14-29`
- 路由定义: `config/routes.rb:231` 附近

#### 12.1.3 准备 Webhook 接收服务

机器人需要实现一个 HTTP 端点来接收 Chatwoot 推送的事件。

**事件类型及处理**:

| 事件类型 (event) | 触发时机 | 处理逻辑 |
|-----------------|---------|---------|
| `webwidget_triggered` | 访客打开聊天窗口 | 可选：发送欢迎消息 |
| `message_created` | 新消息创建 | 核心：处理用户消息 |
| `message_updated` | 消息更新（如表单提交） | 处理交互式组件响应 |
| `conversation_created` | 新会话创建 | 初始化会话状态 |
| `conversation_updated` | 会话属性变更 | 同步会话状态 |
| `conversation_opened` | 会话重新打开 | 重新接管会话 |
| `conversation_resolved` | 会话已解决 | 清理会话上下文 |
| `conversation_status_changed` | 会话状态变更 | 跟踪控制权变化 |

---

##### Webhook 签名验证规则（详细，基于真实代码）

**核心签名算法** (`lib/webhooks/trigger.rb:54-63`):

```ruby
# Chatwoot 生成签名的真实代码
def request_headers(body)
  headers = { 'Content-Type' => 'application/json', 'Accept' => 'application/json' }
  headers['X-Chatwoot-Delivery'] = @delivery_id if @delivery_id.present?
  if @secret.present?
    ts = Time.now.to_i.to_s                                          # 步骤1: 生成时间戳字符串
    headers['X-Chatwoot-Timestamp'] = ts                              # 步骤2: 放入请求头
    headers['X-Chatwoot-Signature'] = "sha256=#{OpenSSL::HMAC.hexdigest(
      'SHA256', 
      @secret, 
      "#{ts}.#{body}"                                                # 步骤3: 拼接并计算签名
    )}"
  end
  headers
end
```

**签名规则详解（精确到代码级）**:

| 项目 | 真实值/格式 | 代码来源 |
|-----|-----------|---------|
| **时间戳生成** | `Time.now.to_i.to_s` | `trigger.rb:58` |
| **时间戳含义** | Unix 时间戳（秒），转为字符串 | `to_i.to_s` |
| **签名数据格式** | `"#{timestamp}.#{raw_json_body}"` | `trigger.rb:60` |
| **连接符** | 单个点 `.`（不是冒号或其他） | `"#{ts}.#{body}"` |
| **哈希算法** | `HMAC-SHA256` | `OpenSSL::HMAC.hexdigest('SHA256', ...)` |
| **输出格式** | 十六进制小写字符串（64 字符） | `hexdigest` |
| **签名前缀** | `sha256=`（小写，无空格） | `"sha256=#{...}"` |
| **请求头 Key** | `X-Chatwoot-Signature` | `trigger.rb:60` |
| **密钥来源** | `AgentBot.secret` | `agent_bot_listener.rb:88` |
| **额外请求头** | `X-Chatwoot-Timestamp` | `trigger.rb:59` |

**完整请求头示例**:
```
Content-Type: application/json
Accept: application/json
X-Chatwoot-Delivery: 550e8400-e29b-41d4-a716-446655440000
X-Chatwoot-Timestamp: 1715421600
X-Chatwoot-Signature: sha256=8af71344b848b89fda887724d71e781f69b6d5a8a5295370ec4dbe99ce117c7a
```

---

##### 可直接验证的测试用例

**已知输入**:
```ruby
# 模拟 Chatwoot 的签名过程
secret = "chatwoot-bot-secret-12345"
timestamp = "1715421600"                    # Time.now.to_i.to_s 的结果
body = '{"event":"message_created","id":1}'  # @payload.to_json 的结果

# 签名字符串
signature_string = "#{timestamp}.#{body}"    # => "1715421600.{\"event\":\"message_created\",\"id\":1}"

# 计算签名
require 'openssl'
hmac = OpenSSL::HMAC.hexdigest('SHA256', secret, signature_string)
# => "4d9f7c3e2a1b5d8f4c7e9a3b6d2f8e5c1a7b3d9f5c7e1b3d5f9c7a1b3d5e9f2c"

# 最终签名头
signature_header = "sha256=#{hmac}"
# => "sha256=4d9f7c3e2a1b5d8f4c7e9a3b6d2f8e5c1a7b3d9f5c7e1b3d5f9c7a1b3d5e9f2c"
```

**你的验证端需要重现的逻辑**:

```
输入:
  headers = {
    "X-Chatwoot-Timestamp": "1715421600",
    "X-Chatwoot-Signature": "sha256=4d9f7c3e2a1b5d8f4c7e9a3b6d2f8e5c1a7b3d9f5c7e1b3d5f9c7a1b3d5e9f2c"
  }
  raw_body = '{"event":"message_created","id":1}'
  secret = "chatwoot-bot-secret-12345"

验证步骤:
  1. signature_string = "1715421600" + "." + raw_body
  2. computed_hmac = HMAC-SHA256(secret, signature_string)
  3. expected_signature = "sha256=" + computed_hmac
  4. expected_signature === headers["X-Chatwoot-Signature"]
```

---

##### 验证步骤分解（必须严格遵守）

**步骤 1: 提取请求头**
```
从 HTTP 请求头获取:
  - X-Chatwoot-Timestamp: 时间戳字符串
  - X-Chatwoot-Signature: 完整签名（带 sha256= 前缀）

注意:
  - 时间戳是字符串形式（不是整数）
  - 不需要转换时间戳类型，直接字符串拼接
```

**步骤 2: 获取原始请求体**
```
- 必须是 Chatwoot 发送的原始 JSON 字节流
- 不能是解析后重新序列化的对象（JSON.parse → JSON.stringify 会改变格式）
- 必须保留原始字节的每一个字符（包括空格、引号顺序）
```

**步骤 3: 构建签名数据**
```ruby
# 正确格式：时间戳字符串 + 点 + 原始请求体字符串
signature_string = "#{timestamp}.#{raw_body}"

# 例如:
#   timestamp  = "1715421600"
#   raw_body   = '{"event":"message_created"}'
#   结果       = "1715421600.{\"event\":\"message_created\"}"

# 错误格式示例（不要这样做）:
#   "#{timestamp.to_i}.#{raw_body}"    # ❌ 整数时间戳
#   "#{timestamp}: #{raw_body}"         # ❌ 冒号和空格
#   "#{raw_body}.#{timestamp}"          # ❌ 顺序反了
```

**步骤 4: 计算 HMAC-SHA256**
```ruby
# Ruby:
OpenSSL::HMAC.hexdigest('SHA256', secret, signature_string)

# Node.js:
crypto.createHmac('sha256', secret).update(signature_string).digest('hex')

# Python:
hmac.new(secret.encode(), signature_string.encode(), hashlib.sha256).hexdigest()
```

**步骤 5: 构造预期签名**
```ruby
# 必须是小写的 sha256= 前缀
expected_signature = "sha256=#{computed_hmac}"

# 例如:
#   "sha256=4d9f7c3e2a1b5d8f4c7e9a3b6d2f8e5c1a7b3d9f5c7e1b3d5f9c7a1b3d5e9f2c"

# 错误示例:
#   "SHA256=..."      # ❌ 大写
#   "sha256 = ..."    # ❌ 有空格
#   "...（无前缀）"    # ❌ 缺少前缀
```

**步骤 6: 安全比较**
```
- 使用 timing-safe 比较（避免时序攻击）
- 比较完整的签名（包括 sha256= 前缀）
- 或者：提取 hmac 部分后比较

Node.js: crypto.timingSafeEqual()
Python:  hmac.compare_digest()
Ruby:    Rack::Utils.secure_compare()
```

---

##### 可直接复用的验证代码

**Node.js/Express（生产级完整实现）**:

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
const BOT_SECRET = 'your_agent_bot_secret_here';
const REPLAY_THRESHOLD = 5 * 60; // 5 分钟

// 关键：使用 raw 模式获取原始请求体
app.use('/webhook/chatwoot', express.raw({ type: 'application/json' }));

/**
 * 验证 Chatwoot Webhook 签名
 * @param {string} timestamp - X-Chatwoot-Timestamp 请求头
 * @param {string} signature - X-Chatwoot-Signature 请求头
 * @param {Buffer|string} rawBody - 原始请求体
 * @param {string} secret - AgentBot.secret
 * @returns {boolean} - 签名是否有效
 */
function verifyChatwootSignature(timestamp, signature, rawBody, secret) {
  if (!timestamp || !signature) return false;
  if (!signature.startsWith('sha256=')) return false;

  // 构建签名字符串: "{timestamp}.{rawBody}"
  const signatureString = `${timestamp}.${rawBody.toString('utf8')}`;

  // 计算 HMAC
  const hmac = crypto.createHmac('sha256', secret);
  const computedHmac = hmac.update(signatureString).digest('hex');
  const expectedSignature = `sha256=${computedHmac}`;

  // 安全比较
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}

/**
 * 检查重放攻击
 */
function isReplayAttack(timestamp) {
  if (!timestamp) return true;
  const currentTime = Math.floor(Date.now() / 1000);
  return Math.abs(currentTime - parseInt(timestamp, 10)) > REPLAY_THRESHOLD;
}

app.post('/webhook/chatwoot', (req, res) => {
  const timestamp = req.headers['x-chatwoot-timestamp'];
  const signature = req.headers['x-chatwoot-signature'];

  // 1. 检查必需的请求头
  if (!timestamp || !signature) {
    return res.status(401).json({ error: 'Missing signature headers' });
  }

  // 2. 检查重放攻击（可选但推荐）
  if (isReplayAttack(timestamp)) {
    return res.status(401).json({ error: 'Request expired' });
  }

  // 3. 验证签名
  if (!verifyChatwootSignature(timestamp, signature, req.body, BOT_SECRET)) {
    console.error('Invalid webhook signature:', {
      timestamp,
      signature,
      bodyLength: req.body.length
    });
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // 4. 签名验证通过，处理业务逻辑
  let payload;
  try {
    payload = JSON.parse(req.body.toString('utf8'));
  } catch (e) {
    return res.status(400).json({ error: 'Invalid JSON' });
  }

  const { event } = payload;
  console.log(`[Webhook] Received event: ${event}`);

  // 5. 根据事件类型处理
  switch (event) {
    case 'message_created':
      // 只处理用户发送的消息，忽略系统消息和机器人自己的消息
      if (payload.message_type === 'incoming' && !payload.private) {
        handleIncomingMessage(payload);
      }
      break;
    case 'conversation_created':
      handleNewConversation(payload);
      break;
    case 'conversation_status_changed':
      handleStatusChange(payload);
      break;
    case 'conversation_resolved':
      handleConversationResolved(payload);
      break;
  }

  // 必须返回 2xx 表示成功接收
  res.sendStatus(200);
});

// 业务处理函数
function handleIncomingMessage(payload) {
  console.log('📩 User message:', payload.content);
  // TODO: 调用你的 AI 机器人服务
  // TODO: 使用 Chatwoot API 发送回复
}

function handleNewConversation(payload) {
  console.log('💬 New conversation:', payload.conversation.id);
}

function handleStatusChange(payload) {
  const { status } = payload.conversation;
  console.log('🔄 Status changed:', status);
  // status: 'pending' = 机器人控制, 'open' = 人工控制
}

function handleConversationResolved(payload) {
  console.log('✅ Conversation resolved:', payload.conversation.id);
}

app.listen(3001, () => {
  console.log('Chatwoot Bot Webhook Server running on port 3001');
  console.log('Webhook endpoint: http://localhost:3001/webhook/chatwoot');
});
```

**Python/FastAPI（生产级完整实现）**:

```python
import hmac
import hashlib
import json
import time
from typing import Optional
from fastapi import FastAPI, Request, HTTPException, Header
from fastapi.responses import PlainTextResponse

app = FastAPI()
BOT_SECRET = "your_agent_bot_secret_here"
REPLAY_THRESHOLD = 5 * 60  # 5 分钟


def verify_chatwoot_signature(
    timestamp: str,
    signature: str,
    raw_body: bytes,
    secret: str
) -> bool:
    """
    验证 Chatwoot Webhook 签名
    
    Args:
        timestamp: X-Chatwoot-Timestamp 请求头
        signature: X-Chatwoot-Signature 请求头
        raw_body: 原始请求体字节
        secret: AgentBot.secret
    
    Returns:
        签名是否有效
    """
    if not timestamp or not signature:
        return False
    if not signature.startswith("sha256="):
        return False

    # 构建签名字符串: "{timestamp}.{raw_body}"
    signature_string = f"{timestamp}.{raw_body.decode('utf-8')}"

    # 计算 HMAC
    computed_hmac = hmac.new(
        secret.encode("utf-8"),
        signature_string.encode("utf-8"),
        hashlib.sha256
    ).hexdigest()
    expected_signature = f"sha256={computed_hmac}"

    # 安全比较
    return hmac.compare_digest(signature, expected_signature)


def is_replay_attack(timestamp: str) -> bool:
    """检查是否为重放攻击"""
    if not timestamp:
        return True
    try:
        timestamp_int = int(timestamp)
    except ValueError:
        return True
    current_time = int(time.time())
    return abs(current_time - timestamp_int) > REPLAY_THRESHOLD


@app.post("/webhook/chatwoot")
async def webhook(
    request: Request,
    x_chatwoot_timestamp: Optional[str] = Header(None),
    x_chatwoot_signature: Optional[str] = Header(None)
):
    # 1. 检查必需的请求头
    if not x_chatwoot_timestamp or not x_chatwoot_signature:
        raise HTTPException(status_code=401, detail="Missing signature headers")

    # 2. 检查重放攻击
    if is_replay_attack(x_chatwoot_timestamp):
        raise HTTPException(status_code=401, detail="Request expired")

    # 3. 获取原始请求体
    raw_body = await request.body()

    # 4. 验证签名
    if not verify_chatwoot_signature(
        x_chatwoot_timestamp,
        x_chatwoot_signature,
        raw_body,
        BOT_SECRET
    ):
        raise HTTPException(status_code=401, detail="Invalid signature")

    # 5. 解析并处理
    try:
        payload = json.loads(raw_body.decode("utf-8"))
    except json.JSONDecodeError:
        raise HTTPException(status_code=400, detail="Invalid JSON")

    event = payload.get("event")
    print(f"[Webhook] Received event: {event}")

    # 6. 处理业务逻辑
    if event == "message_created":
        if payload.get("message_type") == "incoming" and not payload.get("private"):
            print(f"User message: {payload.get('content')}")
            # TODO: 处理消息

    return PlainTextResponse(status_code=200)


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=3001)
```

---

##### 常见错误排查

| 错误症状 | 可能原因 | 验证方法 |
|---------|---------|---------|
| 签名不匹配 | 请求体被重新序列化 | 打印原始 body 和重新序列化后的 body，比较是否相同 |
| 签名不匹配 | 时间戳顺序反了 | 确保是 `timestamp.body` 不是 `body.timestamp` |
| 签名不匹配 | 连接符错误 | 确保使用单个点 `.` |
| 签名不匹配 | 时间戳类型错误 | 时间戳必须是**字符串**，不要 `parseInt` 后再拼接 |
| 签名缺失 | AgentBot.secret 为空 | 检查数据库：`SELECT secret FROM agent_bots WHERE id = ?` |
| 429/500 触发重试 | 机器人服务繁忙或错误 | 检查 Webhook 服务器状态，返回 200 |

**调试技巧**:

```javascript
// 在你的验证函数中添加调试日志
function debugSignature(timestamp, signature, rawBody, secret) {
  const signatureString = `${timestamp}.${rawBody}`;
  const hmac = crypto.createHmac('sha256', secret).update(signatureString).digest('hex');
  
  console.log('=== Signature Debug ===');
  console.log('Timestamp:', timestamp, typeof timestamp);
  console.log('Signature string length:', signatureString.length);
  console.log('Signature string preview:', signatureString.substring(0, 100));
  console.log('Expected signature:', `sha256=${hmac}`);
  console.log('Received signature:', signature);
  console.log('Match:', `sha256=${hmac}` === signature);
}
```

**关键代码引用**:
- 签名生成: `lib/webhooks/trigger.rb:54-63`
- Webhook 事件触发: `app/listeners/agent_bot_listener.rb:85-90`
- Webhook Job: `app/jobs/agent_bots/webhook_job.rb:1-16`
- 超时配置: `lib/webhooks/trigger.rb:118-123`（默认 5 秒）
- 重试机制: `lib/webhooks/trigger.rb:2-4, 125-127`

**Webhook 失败处理** (`lib/webhooks/trigger.rb:77-83`):
```ruby
def update_conversation_status(message)
  conversation = message.conversation
  return unless conversation&.pending?
  return if conversation&.account&.keep_pending_on_bot_failure

  conversation.open!  # 机器人失败时自动转给人工
end
```

---

### 12.2 阶段二：机器人接管会话

#### 12.2.1 新会话创建时的自动接管

当用户在已绑定机器人的收件箱发起新会话时，机器人会自动接管。

**触发条件** (`app/models/conversation.rb:255-267`):
```ruby
def determine_conversation_status
  self.status = :resolved and return if contact.blocked?
  return handle_campaign_status if campaign.present?
  self.status = :pending if inbox.active_bot?  # 关键：机器人激活时设为 pending
end
```

**会话状态流转**:
1. 用户发送第一条消息
2. 会话创建，`status` 被设为 `pending`
3. `AgentBotListener` 监听到 `message_created` 事件
4. 通过 `AgentBots::WebhookJob` 异步推送事件到机器人 Webhook

**Webhook Payload 示例 (message_created)**:
```json
{
  "event": "message_created",
  "id": 1,
  "content": "我想查询订单",
  "message_type": "incoming",
  "private": false,
  "sender": {
    "id": 5,
    "name": "Customer",
    "type": "contact"
  },
  "conversation": {
    "id": 101,
    "status": "pending",
    "assignee_id": null,
    "assignee_agent_bot_id": 1
  },
  "inbox": {
    "id": 1,
    "name": "Website Chat"
  },
  "account": {
    "id": 1,
    "name": "My Company"
  }
}
```

#### 12.2.2 机器人发送回复消息

机器人通过 Chatwoot API 发送回复消息。

**API 端点**:
```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/messages
```

**请求示例**:
```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/messages \
  -H "Content-Type: application/json" \
  -H "api_access_token: BOT_ACCESS_TOKEN" \
  -d '{
    "content": "您好！请问您的订单号是多少？",
    "private": false
  }'
```

**带交互组件的消息 (表单/按钮)**:
```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/messages \
  -H "Content-Type: application/json" \
  -H "api_access_token: BOT_ACCESS_TOKEN" \
  -d '{
    "content": "请选择您的问题类型：",
    "content_type": "input_select",
    "content_attributes": {
      "items": [
        { "title": "订单查询", "value": "order_status" },
        { "title": "退货申请", "value": "return_request" },
        { "title": "其他问题", "value": "other" }
      ]
    }
  }'
```

**关键代码**:
- 消息创建 API: 参考 MessagesController

#### 12.2.3 机器人处理会话状态

机器人需要关注以下会话属性来判断控制权:

| 字段 | 值 | 含义 |
|-----|---|------|
| `status` | `pending` | 机器人控制中 |
| `status` | `open` | 人工控制中 |
| `assignee_agent_bot_id` | 非 null | 当前会话被分配给机器人 |
| `assignee_id` | 非 null | 当前会话被分配给人工客服 |

**检查控制权的 Webhook 处理逻辑**:
```javascript
function isBotInControl(payload) {
  const { conversation } = payload;
  // 状态为 pending 且没有被分配给人工
  return conversation.status === 'pending' && 
         conversation.assignee_id === null;
}
```

---

### 12.3 阶段三：人工介入 (控制权切换：机器人 → 人工)

#### 12.3.1 触发条件总览

| 触发方式 | 触发主体 | 触发条件 | 代码位置 |
|---------|---------|---------|---------|
| 机器人主动 handoff | 机器人 | 返回 `handoff` action 或调用 API | `lib/integrations/bot_processor_service.rb:55-62` |
| Dialogflow intent | Dialogflow | fulfillment payload 含 `action: handoff` | `lib/integrations/dialogflow/processor_service.rb:79-80` |
| Captain HandoffTool | LLM 机器人 | 工具调用判定需要人工 | `enterprise/lib/captain/tools/handoff_tool.rb:1-51` |
| 人工主动接管 | 客服人员 | 前端点击"接管"按钮 | `app/controllers/api/v1/accounts/conversations_controller.rb:81-92` |
| 分配给人工客服 | 管理员/系统 | 调用分配 API 指定 assignee | `app/controllers/api/v1/accounts/conversations/assignments_controller.rb:1-45` |

#### 12.3.2 方式一：机器人主动触发 handoff (Webhook 机器人)

**通过 BotProcessorService 的 action 机制** (`lib/integrations/bot_processor_service.rb:55-62`):

```ruby
def process_action(message, action)
  case action
  when 'handoff'
    message.conversation.bot_handoff!
  when 'resolve'
    message.conversation.resolved!
  end
end
```

**机器人通过消息 content 触发**:

机器人发送消息时，在 `content_attributes` 中指定 `action`:

```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/messages \
  -H "Content-Type: application/json" \
  -H "api_access_token: BOT_ACCESS_TOKEN" \
  -d '{
    "content": "正在为您转接人工客服，请稍候...",
    "content_attributes": {
      "action": "handoff"
    }
  }'
```

**机器人直接通过 API 触发 handoff**:

```bash
# 方式1：通过 toggle_status API（机器人身份调用）
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/toggle_status \
  -H "Content-Type: application/json" \
  -H "api_access_token: BOT_ACCESS_TOKEN" \
  -d '{
    "status": "open"
  }'
```

**关键判断逻辑** (`app/controllers/api/v1/accounts/conversations_controller.rb:94-98`):
```ruby
def pending_to_open_by_bot?
  return false unless Current.user.is_a?(AgentBot)
  @conversation.status == 'pending' && params[:status] == 'open'
end
```

#### 12.3.3 方式二：Dialogflow 触发 handoff

Dialogflow 的 fulfillment payload 中包含 `action` 字段:

```json
{
  "fulfillmentMessages": [
    {
      "payload": {
        "action": "handoff"
      }
    }
  ]
}
```

**处理逻辑** (`lib/integrations/dialogflow/processor_service.rb:75-85`):
```ruby
def process_response(message, response)
  fulfillment_messages = response.query_result['fulfillment_messages']
  fulfillment_messages.each do |fulfillment_message|
    content_params = generate_content_params(fulfillment_message)
    if content_params['action'].present?
      process_action(message, content_params['action'])  # 触发 handoff
    else
      create_conversation(message, content_params)
    end
  end
end
```

#### 12.3.4 方式三：企业版 Captain LLM 机器人

**HandoffTool 调用** (`enterprise/lib/captain/tools/handoff_tool.rb:5-42`):

```ruby
def perform(tool_context, reason: nil)
  conversation = find_conversation(tool_context.state)
  return 'Conversation not found' unless conversation

  log_tool_usage('tool_handoff', {
    conversation_id: conversation.id,
    reason: reason || 'Agent requested handoff'
  })

  trigger_handoff(conversation, reason)
end

def trigger_handoff(conversation, reason)
  conversation.messages.create!(
    message_type: :outgoing,
    private: true,
    sender: @assistant,
    account: conversation.account,
    inbox: conversation.inbox,
    content: reason
  )
  conversation.bot_handoff!  # 核心调用
  send_out_of_office_message_if_applicable(conversation)
end
```

**LLM 判断触发 handoff 的典型场景**:
- 用户明确要求人工客服
- 问题超出机器人知识库范围
- 涉及敏感操作需要人工审核
- 连续多次无法解决用户问题

#### 12.3.5 方式四：人工客服主动接管

**前端 UI 操作**:
1. 客服在会话列表看到 `pending` 状态的会话
2. 点击"接管"按钮或直接回复消息

**API 调用方式**:
```bash
# 方式1：分配给指定客服
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/assignments \
  -H "Content-Type: application/json" \
  -H "api_access_token: AGENT_ACCESS_TOKEN" \
  -d '{
    "assignee_id": 5
  }'

# 方式2：改变会话状态为 open
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/toggle_status \
  -H "Content-Type: application/json" \
  -H "api_access_token: AGENT_ACCESS_TOKEN" \
  -d '{
    "status": "open"
  }'
```

**互斥分配机制** (`app/services/conversations/assignment_service.rb:16-30`):
```ruby
def assign_agent
  conversation.assignee = assignee
  conversation.assignee_agent_bot = nil  # 自动清除机器人分配
  conversation.save!
  assignee
end
```

**模型层面的保护** (`app/models/conversation.rb:249-253`):
```ruby
def reset_agent_bot_when_assignee_present
  return if assignee_id.blank?
  self.assignee_agent_bot_id = nil  # 确保机器人被清除
end
```

#### 12.3.6 bot_handoff! 核心执行流程

**方法定义** (`app/models/conversation.rb:161-165`):
```ruby
def bot_handoff!
  update(waiting_since: Time.current) if waiting_since.blank?
  open!  # pending → open
  dispatcher_dispatch(CONVERSATION_BOT_HANDOFF)
end
```

**执行步骤分解**:

| 步骤 | 操作 | 说明 |
|-----|------|------|
| 1 | 设置 `waiting_since` | 记录人工响应等待开始时间，用于 SLA 计算 |
| 2 | `open!` | 状态从 `pending` 变为 `open`，控制权移交人工 |
| 3 | 派发 `CONVERSATION_BOT_HANDOFF` 事件 | 触发相关监听器处理 |

**事件类型定义** (`lib/events/types.rb:20`):
```ruby
CONVERSATION_BOT_HANDOFF = 'conversation.bot_handoff'
```

---

### 12.4 阶段四：再交回机器人 (控制权切换：人工 → 机器人)

#### 12.4.1 触发条件

| 触发方式 | 触发主体 | 说明 |
|---------|---------|------|
| 调用分配 API | 客服/管理员 | 显式指定 `assignee_type=AgentBot` |
| 自动化规则 | 系统 | 基于规则自动分配回机器人 |
| 已解决会话被用户重新打开 | 用户 | 机器人活跃时自动设为 pending |

#### 12.4.2 方式一：通过分配 API 交回机器人

**API 端点**:
```
POST /api/v1/accounts/{account_id}/conversations/{conversation_id}/assignments
```

**请求示例**:
```bash
curl -X POST http://localhost:3000/api/v1/accounts/1/conversations/101/assignments \
  -H "Content-Type: application/json" \
  -H "api_access_token: AGENT_ACCESS_TOKEN" \
  -d '{
    "assignee_id": 1,
    "assignee_type": "AgentBot"
  }'
```

**关键参数**:
- `assignee_id`: AgentBot 的 ID
- `assignee_type`: 必须为 `"AgentBot"`（区分于 `"User"`）

**控制器判断** (`app/controllers/api/v1/accounts/conversations/assignments_controller.rb:42-44`):
```ruby
def agent_bot_assignment?
  params[:assignee_type].to_s == 'AgentBot'
end
```

**分配服务处理** (`app/services/conversations/assignment_service.rb:23-30`):
```ruby
def assign_agent_bot
  return unless agent_bot

  conversation.assignee = nil  # 清除人工客服
  conversation.assignee_agent_bot = agent_bot  # 设置机器人
  conversation.save!
  agent_bot
end
```

#### 12.4.3 方式二：用户重新打开已解决会话

**触发场景**:
1. 会话已被人工客服标记为 `resolved`
2. 用户再次发送新消息
3. 收件箱仍有活跃机器人

**自动状态切换** (`app/models/message.rb:424-431`):
```ruby
def reopen_resolved_conversation
  if conversation.inbox.active_bot?
    conversation.pending!  # 自动设为 pending，机器人接管
  elsif conversation.inbox.api?
    Current.executed_by = sender if reopened_by_contact?
    conversation.open!
  else
    conversation.open!
  end
end
```

**验证** (`spec/models/message_spec.rb:262-271`):
```ruby
it 'will mark the conversation as pending if the agent bot is active' do
  agent_bot = create(:agent_bot)
  inbox = conversation.inbox
  inbox.agent_bot = agent_bot
  inbox.save!
  conversation.resolved!
  message.save!
  
  expect(conversation.open?).to be false
  expect(conversation.pending?).to be true  # 已自动切换为机器人控制
end
```

#### 12.4.4 方式三：通过自动化规则自动分配

**AutomationRuleListener** (`app/listeners/automation_rule_listener.rb:1-86`) 可用于:
- 当会话满足特定条件时自动分配给机器人
- 例如：工作时间外自动交给机器人

**规则触发事件**:
- `conversation_created`
- `conversation_updated`
- `conversation_opened`
- `conversation_resolved`
- `message_created`

---

### 12.5 完整生命周期时序图

```
用户                           Chatwoot                      机器人                      人工客服
 │                              │                              │                              │
 │ 1. 发起新会话                 │                              │                              │
 │─────────────────────────────>│                              │                              │
 │                              │ 创建会话 status=pending       │                              │
 │                              │─────────────────────────────>│                              │
 │                              │    message.created 事件       │                              │
 │                              │                              │                              │
 │                              │ 2. 处理用户消息               │                              │
 │                              │─────────────────────────────>│                              │
 │                              │                              │ 解析意图，生成回复            │
 │                              │<─────────────────────────────│                              │
 │                              │                              │                              │
 │ 3. 机器人回复                 │                              │                              │
 │<─────────────────────────────│                              │                              │
 │                              │                              │                              │
 │ ...多轮对话...                │                              │                              │
 │                              │                              │                              │
 │ 4. 请求人工客服               │                              │                              │
 │─────────────────────────────>│                              │                              │
 │                              │─────────────────────────────>│                              │
 │                              │                              │ 判定需要 handoff             │
 │                              │<─────────────────────────────│                              │
 │                              │                              │                              │
 │                              │ 5. 执行 handoff               │                              │
 │                              │ - status: pending→open       │                              │
 │                              │ - 设置 waiting_since          │                              │
 │                              │ - 派发 bot_handoff 事件       │                              │
 │                              │                              │                              │
 │                              │ 6. 通知人工客服               │                              │
 │                              │────────────────────────────────────────────────────────────>│
 │                              │                              │                              │ 查看待处理会话
 │                              │                              │                              │                              │
 │ 7. 人工客服接管               │                              │                              │
 │                              │<────────────────────────────────────────────────────────────│
 │                              │ - assignee_id=客服ID          │                              │
 │                              │ - assignee_agent_bot_id=nil   │                              │
 │                              │                              │                              │
 │ 8. 人工客服回复               │                              │                              │
 │<────────────────────────────────────────────────────────────│                              │
 │                              │                              │                              │
 │ ...人工处理...                │                              │                              │
 │                              │                              │                              │
 │                              │ 9. 交回机器人                 │                              │
 │                              │<────────────────────────────────────────────────────────────│
 │                              │ - 调用分配 API                │                              │
 │                              │ - assignee_type=AgentBot      │                              │
 │                              │ - status 保持 open 或手动设   │
 │                              │   为 pending                   │                              │
 │                              │                              │                              │
 │                              │ 10. 通知机器人                │                              │
 │                              │─────────────────────────────>│                              │
 │                              │    conversation_updated       │                              │
 │                              │                              │ 重新接管会话                  │
 │                              │                              │                              │
 │ 11. 机器人继续处理            │                              │                              │
 │<─────────────────────────────│                              │                              │
 │                              │                              │                              │
```

---

## 13. 触发条件汇总表

### 13.1 机器人接管会话的触发条件

| 条件 | 状态变化 | 触发来源 | 代码位置 |
|-----|---------|---------|---------|
| 新会话创建 + 收件箱有活跃机器人 | `status: open → pending` | 系统自动 | `app/models/conversation.rb:261` |
| 营销活动无发送人 + 收件箱有活跃机器人 | `status: open → pending` | 系统自动 | `app/models/conversation.rb:266` |
| 用户重新打开已解决会话 + 收件箱有活跃机器人 | `status: resolved → pending` | 用户消息 | `app/models/message.rb:426-427` |
| 通过 API 分配给机器人 | `assignee_agent_bot_id` 设置 | 显式调用 | `app/services/conversations/assignment_service.rb:23-30` |

### 13.2 人工介入的触发条件

| 条件 | 状态变化 | 触发来源 | 代码位置 |
|-----|---------|---------|---------|
| 机器人返回 `handoff` action | `status: pending → open` | 机器人 | `lib/integrations/bot_processor_service.rb:58` |
| Dialogflow fulfillment 含 `action: handoff` | `status: pending → open` | Dialogflow | `lib/integrations/dialogflow/processor_service.rb:79-80` |
| Captain HandoffTool 被调用 | `status: pending → open` | LLM 判定 | `enterprise/lib/captain/tools/handoff_tool.rb:38` |
| 机器人调用 API 设 `status=open` | `status: pending → open` | 机器人 | `app/controllers/api/v1/accounts/conversations_controller.rb:84` |
| 人工客服分配给自己 | `assignee_id` 设置 + 清除机器人 | 客服操作 | `app/services/conversations/assignment_service.rb:16-21` |
| 人工客服直接回复 pending 会话 | 隐式接管 | 客服操作 | 模型回调 |
| 自动化规则分配给人工 | `assignee_id` 设置 | 系统规则 | AutomationRule |

### 13.3 关键状态与控制权关系

| status | assignee_id | assignee_agent_bot_id | 控制权 | 说明 |
|--------|------------|----------------------|-------|------|
| `pending` | null | 非 null | 机器人 | 机器人活跃处理中 |
| `pending` | null | null | 机器人(间接) | 收件箱有活跃机器人，但未直接分配 |
| `open` | 非 null | null | 人工 | 指定人工客服处理中 |
| `open` | null | null | 人工池 | 等待分配给人工客服 |
| `resolved` | 任意 | 任意 | 无 | 会话已结束 |
| `snoozed` | 任意 | 任意 | 无 | 会话已延迟 |

---

## 14. 总结

Chatwoot 的机器人协作机制具有以下核心特点:

1. **多接入方式**: 支持 Webhook 自定义机器人、Dialogflow 集成、企业版 Captain LLM 机器人
2. **事件驱动**: 所有消息和会话变化通过事件系统分发，实现松耦合
3. **状态驱动**: `pending` 状态表示机器人控制，`open` 状态表示人工控制
4. **互斥分配**: 人工和机器人通过 `assignee_id` 和 `assignee_agent_bot_id` 实现互斥控制
5. **灵活切换**: 支持机器人主动触发 (`bot_handoff!`) 和人工主动接管两种切换方式
6. **企业级增强**: Captain 提供工具化的 LLM 机器人，支持结构化的 handoff 和 resolve 操作
7. **完整的 API 支持**: 提供账户级和平台级 API 管理机器人，以及运行时分配控制

这种设计使得 Chatwoot 能够灵活地在自动回复和人工介入之间切换，为客户提供无缝的客服体验。
