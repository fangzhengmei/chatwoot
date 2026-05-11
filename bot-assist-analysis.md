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

## 11. 总结

Chatwoot 的机器人协作机制具有以下核心特点:

1. **多接入方式**: 支持 Webhook 自定义机器人、Dialogflow 集成、企业版 Captain LLM 机器人
2. **事件驱动**: 所有消息和会话变化通过事件系统分发，实现松耦合
3. **状态驱动**: `pending` 状态表示机器人控制，`open` 状态表示人工控制
4. **互斥分配**: 人工和机器人通过 `assignee_id` 和 `assignee_agent_bot_id` 实现互斥控制
5. **灵活切换**: 支持机器人主动触发 (`bot_handoff!`) 和人工主动接管两种切换方式
6. **企业级增强**: Captain 提供工具化的 LLM 机器人，支持结构化的 handoff 和 resolve 操作

这种设计使得 Chatwoot 能够灵活地在自动回复和人工介入之间切换，为客户提供无缝的客服体验。
