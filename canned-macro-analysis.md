# 快捷回复与宏命令服务端架构分析报告

## 1. 概览

本文档分析 Chatwoot 中快捷回复（Canned Responses）和宏命令（Macros）在服务端的存储机制、触发流程、权限校验以及跨多个对话时的执行上下文隔离策略。

---

## 2. 快捷回复（Canned Responses）

### 2.1 数据模型与存储

快捷回复用于存储预定义的消息模板，通过短代码（short_code）快速引用。

#### 2.1.1 数据库表结构

表名：`canned_responses`

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | integer | PRIMARY KEY | 主键 |
| `account_id` | integer | NOT NULL | 所属账户 ID |
| `short_code` | string | - | 短代码（如 `/greeting`） |
| `content` | text | - | 快捷回复内容 |
| `created_at` | datetime | NOT NULL | 创建时间 |
| `updated_at` | datetime | NOT NULL | 更新时间 |

#### 2.1.2 模型定义

位置：`app/models/canned_response.rb`

```ruby
class CannedResponse < ApplicationRecord
  validates :content, presence: true
  validates :short_code, presence: true
  validates :account, presence: true
  validates :short_code, uniqueness: { scope: :account_id }

  belongs_to :account
end
```

**关键特性：**
- `short_code` 在同一账户内唯一
- 与 Account 模型关联，通过 `account_id` 进行租户隔离
- 包含自定义排序作用域 `order_by_search`，支持按短代码和内容的匹配程度排序

### 2.2 触发机制

#### 2.2.1 后端 API

位置：`app/controllers/api/v1/accounts/canned_responses_controller.rb`

**CRUD 操作：**

| 动作 | HTTP 方法 | 路由 | 说明 |
|------|-----------|------|------|
| `index` | GET | `/api/v1/accounts/:account_id/canned_responses` | 列表查询（支持搜索） |
| `create` | POST | `/api/v1/accounts/:account_id/canned_responses` | 创建快捷回复 |
| `update` | PATCH/PUT | `/api/v1/accounts/:account_id/canned_responses/:id` | 更新快捷回复 |
| `destroy` | DELETE | `/api/v1/accounts/:account_id/canned_responses/:id` | 删除快捷回复 |

**搜索功能：**
```ruby
def canned_responses
  if params[:search]
    Current.account.canned_responses
           .where('short_code ILIKE :search OR content ILIKE :search', search: "%#{params[:search]}%")
           .order_by_search(params[:search])
  else
    Current.account.canned_responses
  end
end
```

#### 2.2.2 前端触发流程

位置：`app/javascript/dashboard/components/widgets/conversation/CannedResponse.vue`

1. **加载阶段**：组件挂载时通过 Vuex store 调用 `getCannedResponse` 接口获取数据
2. **搜索阶段**：用户输入触发搜索，更新 `searchKey` 并重新请求后端
3. **选择阶段**：用户点击某个快捷回复，通过 `replace` 事件将内容插入编辑器

**关键点**：
- 快捷回复本质是**纯文本模板**，不包含复杂的业务逻辑
- 触发是**同步的、前端驱动的**，直接将内容替换到消息输入框
- **不涉及后台异步任务**，也不跨越多个对话执行

---

### 2.2.3 "服务端仅存取不展开 short_code"的完整反证链路

**命题**：服务端仅存储和检索快捷回复数据，**不负责**根据 `short_code` 展开为 `content`。展开逻辑完全在前端完成。

以下从三个维度提供完整证据链。

---

#### 证据一：控制器参数接收与响应 — 只有 CRUD，无"展开/解析"动作

位置：`app/controllers/api/v1/accounts/canned_responses_controller.rb`

**完整控制器代码分析：**

```ruby
class Api::V1::Accounts::CannedResponsesController < Api::V1::Accounts::BaseController
  def index
    @canned_responses = canned_responses
  end

  def create
    @canned_response = Current.account.canned_responses.new(canned_response_params)
    @canned_response.save!
    render json: @canned_response
  end

  def update
    @canned_response.update!(canned_response_params)
    render json: @canned_response
  end

  def destroy
    @canned_response.destroy!
    head :ok
  end

  private

  def canned_response_params
    params.require(:canned_response).permit(:short_code, :content)
  end

  def canned_responses
    if params[:search]
      Current.account.canned_responses
             .where('short_code ILIKE :search OR content ILIKE :search', search: "%#{params[:search]}%")
             .order_by_search(params[:search])
    else
      Current.account.canned_responses
    end
  end
end
```

**分析点：**

| 接口 | 接收参数 | 返回内容 | 是否涉及 "short_code → content" 展开 |
|------|---------|---------|-----------------------------------|
| `index` | `search`（可选） | 匹配的 `canned_response` 数组（包含 `short_code` 和 `content` 字段） | **否** — 只是按 search 过滤，返回完整记录 |
| `create` | `canned_response[short_code]`, `canned_response[content]` | 保存后的记录 | **否** — 只是存储，不做解析 |
| `update` | 同上 | 更新后的记录 | **否** — 只是更新 |
| `destroy` | 无 | `head :ok` | **否** — 只是删除 |

**关键观察：**
- 没有 `expand`、`resolve`、`apply` 之类的额外接口
- `index` 的 `search` 参数用于**列表过滤**，不是"根据 short_code 找 content"
- 响应中总是同时返回 `short_code` 和 `content`，由客户端自行选择使用哪个

---

#### 证据二：模型字段存储 — `short_code` 和 `content` 是两个独立字段，服务端只做持久化

位置：`app/models/canned_response.rb`

```ruby
class CannedResponse < ApplicationRecord
  validates :content, presence: true
  validates :short_code, presence: true
  validates :account, presence: true
  validates :short_code, uniqueness: { scope: :account_id }

  belongs_to :account
end
```

**数据库 schema：**

```ruby
create_table "canned_responses", force: :cascade do |t|
  t.integer "account_id", null: false
  t.string "short_code"
  t.text "content"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.index ["account_id", "short_code"], unique: true
end
```

**分析点：**

| 方面 | 结论 |
|------|------|
| **字段独立性** | `short_code` 和 `content` 是两个独立列，没有 `content` 是通过 `short_code` 计算得出的迹象 |
| **唯一性约束** | `unique: true` 加在 `[account_id, short_code]` 上，保证同一账户内 short_code 唯一，但这是**存储约束**，不是"展开约束" |
| **模型方法** | 没有任何实例方法或类方法接受 `short_code` 参数并返回对应的 `content` |
| **回调/钩子** | 没有 `before_save`、`after_find` 等回调处理 short_code 的解析 |

**关键结论：**
- 服务端将 `short_code` 和 `content` 作为**两个独立数据项**存储
- 没有任何数据库层面或模型层面的"展开逻辑"
- 如果服务端要做展开，必然需要一个方法如 `CannedResponse.find_by(short_code: '/greeting').content`，但这个查询由**谁来发起**？后续证据表明发起者不是服务端。

---

#### 证据三：发送消息路径中没有 short_code 解析步骤

这是最关键的反证：**消息从前端发出到服务端存入数据库的完整链路中，没有任何一步解析 short_code 模式。**

##### 3.1 前端发送消息的参数

位置：`app/javascript/dashboard/helper/editorHelper.js`

```javascript
cannedResponse: (editorView, content, from, to, variables) => {
  const updatedMessage = replaceVariablesInMessage({
    message: content,  // ← 这里的 content 已经是展开后的完整消息
    variables,
  });
  const node = createNode(editorView, 'cannedResponse', updatedMessage);
  return {
    node,
    from: node.textContent === updatedMessage ? from : from - 1,
    to,
  };
},
```

**关键观察：**
- 插入编辑器时，使用的已经是 `content`（展开后的完整消息）
- 编辑器中最终显示的是完整文本，不是 `/greeting`

##### 3.2 后端消息接收控制器

位置：`app/controllers/api/v1/accounts/conversations/messages_controller.rb`

```ruby
def create
  user = Current.user || @resource
  mb = Messages::MessageBuilder.new(user, @conversation, params)
  @message = mb.perform
rescue StandardError => e
  render_could_not_create_error(e.message)
end
```

**分析：**
- 直接将 `params` 传给 `Messages::MessageBuilder`
- 没有对 `params[:content]` 做任何 short_code 模式匹配或替换

##### 3.3 MessageBuilder 消息构建逻辑

位置：`app/builders/messages/message_builder.rb`

```ruby
def perform
  @message = @conversation.messages.build(message_params)
  process_attachments
  process_emails
  process_email_content
  @message.save!
  @message
end

def message_params
  {
    account_id: @conversation.account_id,
    inbox_id: @conversation.inbox_id,
    message_type: message_type,
    content: @params[:content],  # ← 直接使用传入的 content，不做解析
    private: @private,
    sender: sender,
    content_type: @params[:content_type],
    content_attributes: content_attributes.presence,
    items: @items,
    in_reply_to: @in_reply_to,
    echo_id: @params[:echo_id],
    source_id: @params[:source_id]
  }.merge(external_created_at).merge(automation_rule_id).merge(campaign_id).merge(template_params)
end
```

**关键证据：**

```ruby
content: @params[:content]  # 直接使用，不做任何 short_code → content 的解析
```

**搜索验证：**

全局搜索 Ruby 代码中是否存在 short_code 解析模式：

```bash
rg -n "short_code.*content|content.*short_code|expand.*short|resolve.*short" app lib --type ruby
# 无匹配结果（除了测试和模型定义本身）
```

全局搜索 JavaScript 代码中是否存在替换逻辑：

```bash
rg -n "canned.*content|shortCode|short_code" app/javascript
# 前端有逻辑，但后端完全没有
```

##### 3.4 完整消息链路对比

**如果服务端展开 short_code，链路应该是：**

```
用户输入 "/greeting"
    ↓
前端发送 { content: "/greeting" }
    ↓
后端 messages#create 接收
    ↓
后端查找 CannedResponse.find_by(short_code: "/greeting")
    ↓
后端替换 content 为 "Hello! How can I help you?"
    ↓
存入数据库
```

**但实际链路是：**

```
用户输入 "/greet"
    ↓
前端拦截 "/" 前缀
    ↓
前端调用 GET /canned_responses?search=greet
    ↓
后端返回 [{ short_code: "/greeting", content: "Hello! ..." }]
    ↓
前端展示列表
    ↓
用户选择某一项
    ↓
前端将 content 直接插入编辑器
    ↓
用户发送消息
    ↓
前端发送 { content: "Hello! How can I help you?" }
    ↓
后端 messages#create 接收
    ↓
后端直接存入数据库（不做任何解析）
```

**关键差异：**

| 步骤 | 假设的"服务端展开" | 实际行为 |
|------|------------------|---------|
| 发送到后端的 `content` | `/greeting` | `"Hello! How can I help you?"` |
| 后端是否需要 `canned_responses` 表参与消息创建 | 是 | **否** |
| `Messages::MessageBuilder` 是否处理 short_code | 是 | **否** |
| 短代码在哪个阶段被"消失" | 后端 | **前端（插入编辑器时）** |

---

#### 结论：服务端仅做存取，不参与展开

| 证据维度 | 结论 |
|---------|------|
| 控制器接口 | 只有 CRUD + 搜索，没有 `expand/apply/resolve` |
| 参数接收 | `canned_response_params` 只接收 `short_code` 和 `content`，不接收需要解析的消息 |
| 消息创建路径 | `messages#create` → `MessageBuilder` 直接使用 `params[:content]`，不做任何 short_code 解析 |
| 全局搜索验证 | Ruby 代码中没有 short_code → content 的替换逻辑 |
| 前端代码 | 前端 `CannedResponse.vue` 和 `editorHelper.js` 明确展示了完整的展开流程 |

**一句话总结：**
> 服务端的 `canned_responses` 表只是一个**键值存储**（key: short_code, value: content）。
> 这个存储的读取由前端在**消息发送之前**完成，然后前端直接把展开后的 `content` 发给消息创建接口。
> 消息创建接口完全不知道 `short_code` 的存在，它只处理已展开的文本。

---

## 3. 宏命令（Macros）

### 3.1 数据模型与存储

宏命令是一组预定义的自动化操作集合，可对一个或多个对话批量执行。

#### 3.1.1 数据库表结构

表名：`macros`

| 字段 | 类型 | 约束 | 默认值 | 说明 |
|------|------|------|--------|------|
| `id` | bigint | PRIMARY KEY | - | 主键 |
| `account_id` | bigint | NOT NULL | - | 所属账户 ID |
| `name` | string | NOT NULL | - | 宏名称 |
| `visibility` | integer | - | 0 (personal) | 可见性：0=个人，1=全局 |
| `created_by_id` | bigint | - | - | 创建者用户 ID |
| `updated_by_id` | bigint | - | - | 更新者用户 ID |
| `actions` | jsonb | NOT NULL | {} | 动作列表（JSON 格式） |
| `created_at` | datetime | NOT NULL | - | 创建时间 |
| `updated_at` | datetime | NOT NULL | - | 更新时间 |

#### 3.1.2 模型定义

位置：`app/models/macro.rb`

```ruby
class Macro < ApplicationRecord
  belongs_to :account
  belongs_to :created_by, class_name: :User, optional: true, inverse_of: :macros
  belongs_to :updated_by, class_name: :User, optional: true
  has_many_attached :files

  enum visibility: { personal: 0, global: 1 }

  validate :json_actions_format

  ACTIONS_ATTRS = %w[
    send_message add_label assign_team assign_agent mute_conversation 
    change_status remove_label remove_assigned_agent remove_assigned_team 
    resolve_conversation snooze_conversation change_priority 
    send_email_transcript send_attachment add_private_note send_webhook_event
  ].freeze
end
```

**关键特性：**
- 使用 PostgreSQL `jsonb` 类型存储 `actions`，支持高效查询和索引
- 支持文件附件（`has_many_attached :files`），用于发送附件类动作
- 两种可见性：
  - `personal`：仅创建者可见
  - `global`：账户内所有用户可见
- 严格的动作格式验证，确保 `actions` 数组中的 `action_name` 必须在白名单内

#### 3.1.3 可见性过滤（List 级别）

位置：`app/models/macro.rb`

```ruby
def self.with_visibility(user, _params)
  records = Current.account.macros.global
  records = records.or(personal.where(created_by_id: user.id, account_id: Current.account.id))
  records.order(:id)
end
```

**查询逻辑：**
- 基础集合：`Current.account.macros.global`（所有全局宏）
- 叠加集合：`personal.where(created_by_id: user.id)`（当前用户创建的个人宏）
- 使用 `or` 连接，返回用户有权查看的所有宏

**可见性规则汇总：**

| 宏类型 | 谁可以在列表中看到 |
|--------|-------------------|
| `global` | 账户内所有用户 |
| `personal` | 仅创建者本人 |

---

### 3.1.4 可见性设置限制

位置：`app/models/macro.rb`

```ruby
def set_visibility(user, params)
  self.visibility = params[:visibility]
  self.visibility = :personal if user.agent?
end
```

**规则：**
- **管理员（administrator）**：可以自由设置 `personal` 或 `global`
- **客服（agent）**：无论参数传入什么，强制设为 `personal`
- 客服无法创建全局宏

---

### 3.1.5 权限策略（Pundit）

位置：`app/policies/macro_policy.rb`

```ruby
class MacroPolicy < ApplicationPolicy
  def index?
    true
  end

  def create?
    true
  end

  def show?
    @record.global? || author?
  end

  def update?
    return @account_user.administrator? if @record.global?

    author?
  end

  def destroy?
    return @account_user.administrator? if @record.global?

    author?
  end

  def execute?
    @record.global? || author?
  end

  private

  def author?
    @record.created_by == @account_user.user
  end
end
```

**权限规则矩阵：**

| 操作 | 条件 | 授权判定 |
|------|------|---------|
| `index?` | 无特殊条件 | 始终 `true`（列表级别的过滤由 `with_visibility` 处理） |
| `create?` | 无特殊条件 | 始终 `true`（但创建时的可见性由 `set_visibility` 限制） |
| `show?` | `global?` 或 `author?` | 全局宏对所有人可见；个人宏仅作者可见 |
| `execute?` | `global?` 或 `author?` | 与 `show?` 规则相同 |
| `update?` | global → admin；personal → author | 全局宏仅管理员可改；个人宏仅作者可改 |
| `destroy?` | global → admin；personal → author | 全局宏仅管理员可删；个人宏仅作者可删 |

**特殊边界情况：**

1. **孤儿全局宏**（作者已被删除）：
   - `global?` 为 `true`，`created_by` 为 `nil`
   - `author?` 返回 `false`
   - 但 `global?` 为 `true`，所以 `show?` 和 `execute?` 仍返回 `true`
   - `update?` 和 `destroy?` 要求 `administrator?`，所以仅管理员可操作

2. **作者降级为 agent 的全局宏**：
   - 宏仍是 `global`
   - `update?` 要求 `administrator?`
   - 即使原作者曾是 admin，现在降级为 agent 后也无法修改/删除自己创建的全局宏

---

### 3.1.6 执行前的完整校验链路与权限失败路径

位置：`app/controllers/api/v1/accounts/macros_controller.rb`

```ruby
before_action :fetch_macro, only: [:show, :update, :destroy, :execute]
before_action :check_authorization, only: [:show, :update, :destroy, :execute]

def fetch_macro
  @macro = Current.account.macros.find_by(id: params[:id])
end

def check_authorization
  authorize(@macro) if @macro.present?
end

def execute
  ::MacrosExecutionJob.perform_later(@macro, conversation_ids: params[:conversation_ids], user: Current.user)
  head :ok
end
```

---

#### 3.1.6.1 三层校验的顺序与条件

```
POST /api/v1/accounts/:account_id/macros/:id/execute
    │
    ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 1 层：fetch_macro (租户级查询限制)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
查询逻辑：Current.account.macros.find_by(id: params[:id])
    │
    ├─ 宏存在 且 属于 Current.account ──→ @macro = 宏对象
    │
    └─ 宏不存在 OR 属于其他账户 ──→ @macro = nil
              │
              ▼
              后续 check_authorization 中：
              authorize(@macro) if @macro.present?
              因为 @macro = nil，所以 authorize 被跳过
              直接进入 execute action
              │
              ▼
              execute 中：
              MacrosExecutionJob.perform_later(nil, conversation_ids, user)
              │
              ▼
              Rails 会尝试序列化 nil 到队列
              实际执行时会出错（见下文详细分析）
```

```
    │
    ▼ (假设 @macro 存在)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 2 层：check_authorization (Pundit 权限校验)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
调用：authorize(@macro)
    │
    ├─ @macro.global? = true ──→ 任何用户都可执行
    │
    ├─ @macro.personal? 且 @macro.created_by == Current.user ──→ 仅作者可执行
    │
    └─ @macro.personal? 且 不是作者 ──→ 抛出 Pundit::NotAuthorizedError
              │
              ▼
              ApplicationController 中的异常处理：
              include Pundit::Authorization
              include RequestExceptionHandler
              │
              ▼
              RequestExceptionHandler 中：
              rescue Pundit::NotAuthorizedError => e
                log_handled_error(e)
                render_unauthorized('You are not authorized to do this action')
              ensure
                Current.reset
              end
              │
              ▼
              HTTP 401 Unauthorized
              Body: { "error": "You are not authorized to do this action" }
```

```
    │
    ▼ (假设通过了前两层)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 3 层：execute 入队 + MacrosExecutionJob 对话验证
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Controller:
  MacrosExecutionJob.perform_later(@macro, conversation_ids: params[:conversation_ids], user: Current.user)
  head :ok
    │
    ▼ (异步任务在 Sidekiq 中执行)
MacrosExecutionJob#perform(macro, conversation_ids:, user:)
  account = macro.account
  conversations = account.conversations.where(display_id: conversation_ids.to_a)
    │
    ├─ 所有 conversation_ids 都属于 macro.account
    │   └─ conversations = 匹配的对话集合
    │       └─ 执行每个对话
    │
    ├─ 部分 conversation_ids 属于其他账户
    │   └─ conversations = 只包含属于 macro.account 的对话
    │       └─ 其他对话被静默过滤，无任何报错
    │
    └─ 所有 conversation_ids 都不属于 macro.account (或无效 ID)
        └─ conversations = [] (empty)
            └─ return if conversations.blank?
            └─ 任务静默结束，不执行任何操作
```

---

#### 3.1.6.2 三种失败场景的详细分析

**场景 A：@macro 不存在（或跨租户）**

| 项目 | 值 |
|------|-----|
| **触发条件** | `params[:id]` 不存在，或该宏属于其他 `account_id` |
| **第 1 层结果** | `@macro = nil` |
| **第 2 层结果** | `check_authorization` 中 `@macro.present?` 为 `false`，跳过 `authorize` |
| **第 3 层结果** | `execute` 调用 `MacrosExecutionJob.perform_later(nil, ...)` |
| **HTTP 响应** | `head :ok` → **200 OK** |
| **实际执行** | **任务入队时传 nil，执行时可能出错或静默失败** |

**代码级分析：**

```ruby
# macros_controller.rb
def fetch_macro
  @macro = Current.account.macros.find_by(id: params[:id])  # find_by 返回 nil，不抛异常
end

def check_authorization
  authorize(@macro) if @macro.present?  # @macro = nil，不执行 authorize
end

def execute
  ::MacrosExecutionJob.perform_later(@macro, conversation_ids: params[:conversation_ids], user: Current.user)
  head :ok  # 立即返回 200
end
```

```ruby
# macros_execution_job.rb
def perform(macro, conversation_ids:, user:)
  account = macro.account  # macro = nil → NoMethodError: undefined method `account' for nil:NilClass
  # ...
end
```

**最终表现：**
- 前端收到 HTTP **200 OK**（因为 controller 先返回了）
- Sidekiq 任务执行时抛出 `NoMethodError`
- 任务可能被重试或丢弃（取决于 Sidekiq 配置）
- **这是一个潜在的 bug：宏不存在时不应该返回 200**

---

**场景 B：Pundit 未授权（个人宏但非作者）**

| 项目 | 值 |
|------|-----|
| **触发条件** | 宏存在且是 `personal`，但 `created_by != Current.user` |
| **第 1 层结果** | `@macro = 宏对象` |
| **第 2 层结果** | `MacroPolicy#execute?` 返回 `false` |
| **第 3 层结果** | `execute` action **根本不执行** |
| **HTTP 响应** | **401 Unauthorized**（由 `RequestExceptionHandler` 处理） |
| **实际执行** | **任务未入队，完全不执行** |

**异常处理链路：**

位置：`app/controllers/concerns/request_exception_handler.rb`

```ruby
module RequestExceptionHandler
  extend ActiveSupport::Concern

  included do
    rescue_from ActiveRecord::RecordInvalid, with: :render_record_invalid
  end

  private

  def handle_with_exception
    yield
  rescue ActiveRecord::RecordNotFound => e
    log_handled_error(e)
    render_not_found_error('Resource could not be found')
  rescue Pundit::NotAuthorizedError => e
    log_handled_error(e)
    render_unauthorized('You are not authorized to do this action')
  rescue ActionController::ParameterMissing => e
    log_handled_error(e)
    render_could_not_create_error(e.message)
  ensure
    Current.reset
  end

  def render_unauthorized(message)
    render json: { error: message }, status: :unauthorized  # ← 返回 401
  end
end
```

注意：`rescue_from Pundit::NotAuthorizedError` 不在 `included do` 块中，而是在 `handle_with_exception` 方法内。这意味着只有使用 `handle_with_exception` 包裹的代码才会捕获 Pundit 异常。

**但 MacrosController 是标准的 Rails 控制器 + Pundit：**

位置：`app/controllers/application_controller.rb`

```ruby
class ApplicationController < ActionController::Base
  include DeviseTokenAuth::Concerns::SetUserByToken
  include RequestExceptionHandler
  include Pundit::Authorization  # ← Pundit 的默认行为是抛出异常后由 Rails 处理
  # ...
end
```

Pundit 的 `authorize` 方法抛出 `Pundit::NotAuthorizedError` 后，Rails 默认会返回 **403 Forbidden**。但 `RequestExceptionHandler` 中有 `handle_with_exception` 方法可以返回 401。

**实际行为取决于是否使用了 `handle_with_exception` 包裹。** 从代码看，`macros_controller.rb` 的 `execute` 方法是直接执行，没有 `handle_with_exception` 包裹。

**修正后的场景 B 结论：**

| 项目 | 值 |
|------|-----|
| **HTTP 响应** | **403 Forbidden**（Pundit 默认行为） |
| **Body** | 取决于 Rails 的异常处理配置 |

---

**场景 C：conversation_ids 跨租户或无效**

| 项目 | 值 |
|------|-----|
| **触发条件** | 宏存在且授权通过，但 `conversation_ids` 包含不属于 `macro.account` 的 ID，或不存在的 ID |
| **第 1 层结果** | `@macro = 宏对象` |
| **第 2 层结果** | 授权通过 |
| **第 3 层结果** | `MacrosExecutionJob` 中过滤对话 |
| **HTTP 响应** | **200 OK** |
| **实际执行** | **取决于过滤后的 conversations 是否为空** |

**代码级分析：**

```ruby
# macros_execution_job.rb
def perform(macro, conversation_ids:, user:)
  account = macro.account
  # WHERE display_id IN (conversation_ids) 只返回属于这个 account 的对话
  conversations = account.conversations.where(display_id: conversation_ids.to_a)

  return if conversations.blank?  # 如果所有 ID 都无效/跨租户，直接返回

  conversations.each do |conversation|
    ::Macros::ExecutionService.new(macro, conversation, user).perform
  end
end
```

**细分情况：**

| conversation_ids 内容 | 过滤后的 conversations | 执行结果 |
|----------------------|---------------------|---------|
| `[1, 2, 3]` 都有效且属于同一账户 | `[Conv1, Conv2, Conv3]` | 3 个对话都执行 |
| `[1, 2]` 有效，`[999]` 不存在 | `[Conv1, Conv2]` | 2 个对话执行，999 被静默忽略 |
| `[1]` 属于当前账户，`[2]` 属于其他账户 | `[Conv1]` | 1 个对话执行，跨租户的 2 被静默过滤 |
| `[999, 888]` 都不存在 | `[]` | 任务静默结束，不执行任何操作 |
| `[3]` 属于其他账户 | `[]` | 任务静默结束 |

**关键特性：**
- 没有任何报错，也没有返回"部分成功/部分失败"的指示
- 前端收到 200 后无法知道实际执行了多少个对话
- 跨租户的对话 ID 被静默过滤（安全但不够可观测）

---

#### 3.1.6.3 三种失败场景汇总表

| 失败场景 | HTTP 状态码 | 是否入队 | 实际执行 | 错误反馈 |
|---------|------------|---------|---------|---------|
| **A. @macro 不存在/跨租户** | **200 OK** | **是**（但传 nil） | Sidekiq 任务执行时抛出 `NoMethodError` | **前端无反馈**（潜在 bug） |
| **B. Pundit 未授权** | **403 Forbidden** | **否** | 任务未入队，完全不执行 | **有明确反馈**（Pundit 异常） |
| **C. conversation_ids 无效/跨租户** | **200 OK** | **是** | 过滤后为空则不执行，部分有效则执行有效部分 | **前端无反馈**（静默忽略） |

**问题代码定位：**

```ruby
# 问题 1：fetch_macro 使用 find_by，失败时返回 nil 而不是抛异常
def fetch_macro
  @macro = Current.account.macros.find_by(id: params[:id])  # 应该用 find! 并在异常处理中返回 404
end

# 问题 2：check_authorization 在 @macro 为 nil 时跳过
def check_authorization
  authorize(@macro) if @macro.present?  # 导致场景 A 绕过权限校验
end

# 问题 3：conversation_ids 过滤后没有任何反馈
conversations = account.conversations.where(display_id: conversation_ids.to_a)
return if conversations.blank?  # 静默返回，前端不知道发生了什么
```

---

### 3.2 支持的动作类型

宏命令支持以下动作（定义在 `ACTIONS_ATTRS` 常量中）：

| 动作名称 | 说明 | 参数类型 |
|----------|------|----------|
| `send_message` | 发送公开消息 | 消息内容字符串 |
| `add_private_note` | 添加私密备注 | 备注内容字符串 |
| `send_attachment` | 发送附件 | Blob ID 数组 |
| `add_label` | 添加标签 | 标签名称数组 |
| `remove_label` | 移除标签 | 标签名称数组 |
| `assign_agent` | 分配客服 | 客服 ID 数组（支持 `self`） |
| `remove_assigned_agent` | 移除分配的客服 | 无参数 |
| `assign_team` | 分配团队 | 团队 ID 数组 |
| `remove_assigned_team` | 移除分配的团队 | 无参数 |
| `change_status` | 更改对话状态 | 状态值数组 |
| `resolve_conversation` | 解决对话 | 无参数 |
| `snooze_conversation` | 延迟处理对话 | 无参数 |
| `mute_conversation` | 静音对话 | 无参数 |
| `change_priority` | 更改优先级 | 优先级值数组 |
| `send_email_transcript` | 发送对话记录邮件 | 邮箱地址数组 |
| `send_webhook_event` | 发送 Webhook 事件 | Webhook URL 数组 |

### 3.3 触发机制

#### 3.3.1 后端 API

位置：`app/controllers/api/v1/accounts/macros_controller.rb`

| 动作 | HTTP 方法 | 路由 | 说明 |
|------|-----------|------|------|
| `index` | GET | `/api/v1/accounts/:account_id/macros` | 列表查询（按可见性过滤） |
| `show` | GET | `/api/v1/accounts/:account_id/macros/:id` | 获取单个宏 |
| `create` | POST | `/api/v1/accounts/:account_id/macros` | 创建宏 |
| `update` | PATCH/PUT | `/api/v1/accounts/:account_id/macros/:id` | 更新宏 |
| `destroy` | DELETE | `/api/v1/accounts/:account_id/macros/:id` | 删除宏 |
| `execute` | POST | `/api/v1/accounts/:account_id/macros/:id/execute` | 执行宏 |

**执行接口：**
```ruby
def execute
  ::MacrosExecutionJob.perform_later(@macro, conversation_ids: params[:conversation_ids], user: Current.user)
  head :ok
end
```

#### 3.3.2 前端触发流程

**单对话场景**（位置：`app/javascript/dashboard/routes/dashboard/conversation/Macros/MacroItem.vue`）：

1. 用户在对话侧边栏的宏列表中点击执行按钮
2. 前端调用 Vuex action `macros/execute`，传入 `macroId` 和 `conversationIds`（当前对话 ID）
3. `macros.js` store 调用 API：`POST /macros/:id/execute`

**批量场景**：
- 支持传入多个 `conversation_ids`，一次 API 调用触发对多个对话执行

**关键特性：**
- 触发是**异步的**：API 立即返回 200 OK，实际执行由后台作业处理
- 支持**跨多个对话**批量执行
- 前端只需提供 `macro_id` 和 `conversation_ids` 数组

### 3.4 执行流程

宏的执行涉及多层架构：

```
前端请求
    ↓
MacrosController#execute
    ↓
MacrosExecutionJob.perform_later  (ActiveJob 异步任务)
    ↓
MacrosExecutionJob#perform
    ├── 查询账户下的目标对话
    └── 对每个对话调用 Macros::ExecutionService
            ↓
        Macros::ExecutionService#perform
            ├── 遍历 @macro.actions 数组
            ├── 对每个动作调用对应的方法（如 send_message, add_label）
            └── 单个动作失败不影响其他动作（异常捕获）
```

#### 3.4.1 MacrosExecutionJob

位置：`app/jobs/macros_execution_job.rb`

```ruby
class MacrosExecutionJob < ApplicationJob
  queue_as :medium

  def perform(macro, conversation_ids:, user:)
    account = macro.account
    conversations = account.conversations.where(display_id: conversation_ids.to_a)

    return if conversations.blank?

    conversations.each do |conversation|
      ::Macros::ExecutionService.new(macro, conversation, user).perform
    end
  end
end
```

**职责：**
1. 接收 `macro`、`conversation_ids` 和 `user` 参数
2. 验证对话属于同一账户（`account.conversations.where(...)`）
3. 遍历每个对话，逐个调用执行服务

#### 3.4.2 Macros::ExecutionService

位置：`app/services/macros/execution_service.rb`

```ruby
class Macros::ExecutionService < ActionService
  def initialize(macro, conversation, user)
    super(conversation)
    @macro = macro
    @account = macro.account
    @user = user
    Current.user = user
  end

  def perform
    @macro.actions.each do |action|
      action = action.with_indifferent_access
      begin
        send(action[:action_name], action[:action_params])
      rescue StandardError => e
        ChatwootExceptionTracker.new(e, account: @account).capture_exception
      end
    end
  ensure
    Current.reset
  end

  # ... 具体动作方法实现
end
```

**继承关系：**
- `Macros::ExecutionService` 继承自 `ActionService`
- `ActionService` 提供基础对话操作方法（状态变更、分配、标签等）

**关键特性：**
1. 初始化时设置 `Current.user = user`，为后续操作提供执行上下文
2. 使用 `begin/rescue` 包裹每个动作，单个动作失败不中断整个宏执行
3. 使用 `ensure` 块保证 `Current.reset` 始终执行，防止上下文泄漏

#### 3.4.3 ActionService

位置：`app/services/action_service.rb`

提供的核心方法：
- `assign_agent`：分配客服
- `assign_team`：分配团队
- `add_label` / `remove_label`：标签管理
- `change_status` / `resolve_conversation` / `snooze_conversation`：状态管理
- `change_priority`：优先级管理
- `send_email_transcript`：发送邮件

---

## 4. 执行上下文隔离机制

### 4.1 Current 模块：线程级上下文存储

位置：`lib/current.rb`

```ruby
module Current
  thread_mattr_accessor :user
  thread_mattr_accessor :account
  thread_mattr_accessor :account_user
  thread_mattr_accessor :executed_by
  thread_mattr_accessor :contact

  def self.reset
    Current.user = nil
    Current.account = nil
    Current.account_user = nil
    Current.executed_by = nil
    Current.contact = nil
  end
end
```

**技术实现：**
- 使用 Rails 的 `thread_mattr_accessor`
- 数据存储在**线程本地存储（Thread Local Storage, TLS）**
- 每个线程有独立的 Current 变量副本

### 4.2 上下文生命周期

#### 4.2.1 请求处理流程中的上下文管理

在 Web 请求中：
1. 控制器中间件/过滤器设置 `Current.user`、`Current.account`
2. 业务逻辑执行期间使用这些上下文
3. 请求结束时通过 `ensure` 或异常处理器调用 `Current.reset`

#### 4.2.2 宏执行流程中的上下文管理

**单个对话执行流程：**

```
Macros::ExecutionService.new(macro, conversation, user)
    ↓
initialize: Current.user = user
    ↓
perform: 遍历执行所有动作
    ↓  (每个动作可能依赖 Current.user)
例如: Messages::MessageBuilder.new(@user, @conversation, params)
    ↓
ensure: Current.reset
```

**关键代码分析：**

```ruby
def initialize(macro, conversation, user)
  super(conversation)
  @macro = macro
  @account = macro.account
  @user = user
  Current.user = user  # 设置上下文
end

def perform
  @macro.actions.each do |action|
    # ... 执行动作
  end
ensure
  Current.reset  # 确保清理
end
```

### 4.3 跨多个对话的隔离策略

当宏在多个对话上执行时（`MacrosExecutionJob`），隔离通过以下方式实现：

#### 4.3.1 逐对话独立执行

```ruby
conversations.each do |conversation|
  ::Macros::ExecutionService.new(macro, conversation, user).perform
end
```

每个对话：
1. 创建**独立**的 `Macros::ExecutionService` 实例
2. 每个实例有自己的 `@conversation`、`@user` 等实例变量
3. 每个实例独立设置和清理 `Current.user`

#### 4.3.2 实例变量隔离

每个 `ExecutionService` 实例包含：
- `@macro`：共享的宏定义（只读）
- `@conversation`：**每个对话独立**
- `@account`：共享的账户对象
- `@user`：执行用户（同一宏执行中通常相同）

#### 4.3.3 Current 变量的设置与清理

```
对话1: new() → Current.user = user → perform() → ensure: Current.reset
                                                          ↓
对话2: new() → Current.user = user → perform() → ensure: Current.reset
                                                          ↓
对话3: ...
```

即使在同一线程中顺序执行：
1. 对话1 执行完毕后，`Current.reset` 清理所有变量
2. 对话2 开始时重新设置 `Current.user`
3. 对话间的 `Current` 状态完全隔离

---

### 4.3.4 两层隔离边界的细分

宏执行时存在**两个层次**的隔离：**线程级隔离**和**对话实例级隔离**。

#### 线程级隔离（Thread-Level Isolation）

**基础机制**：
位置：`lib/current.rb`

```ruby
module Current
  thread_mattr_accessor :user
  thread_mattr_accessor :account
  thread_mattr_accessor :account_user
  thread_mattr_accessor :executed_by
  thread_mattr_accessor :contact
end
```

**线程级隔离的是：**

| Current 变量 | 说明 | 何时使用 |
|-------------|------|---------|
| `Current.user` | 当前操作用户 | 模型回调、审计日志、消息发送者 |
| `Current.account` | 当前账户 | 租户隔离、权限校验 |
| `Current.account_user` | 用户-账户关联 | 角色判断（admin/agent） |
| `Current.executed_by` | 执行者标识 | 特殊操作追踪 |
| `Current.contact` | 当前联系人 | 联系人相关操作 |

**线程级隔离的保证：**

1. **线程本地存储（TLS）**：使用 `thread_mattr_accessor`，数据绑定到 `Thread.current`
2. **不同线程天然隔离**：Sidekiq  worker 线程、Web 请求线程各有独立副本
3. **同一线程内的时序隔离**：通过 `Current.reset` 在每次执行前后清理

**场景分析：**

```
Sidekiq 进程（多线程）
├── Thread 1: 执行 Job A (用户 A 的宏)
│   └── Current.user = User_A  ← 只在 Thread 1 可见
│
├── Thread 2: 执行 Job B (用户 B 的宏)
│   └── Current.user = User_B  ← 只在 Thread 2 可见，与 Thread 1 无关
│
└── Thread 3: 执行 Job C (批量处理 10 个对话)
    ├── 对话 1: Current.user = User_C → 执行 → Current.reset
    ├── 对话 2: Current.user = User_C → 执行 → Current.reset
    └── ...
```

**线程级隔离不解决的问题：**
- 同一线程内顺序执行多个对话时，`Current` 变量可能被前一个对话"污染"
- 需要配合对话实例级隔离

---

#### 对话实例级隔离（Conversation-Instance Level Isolation）

**基础机制**：`MacrosExecutionJob` 对每个对话创建独立的 `ExecutionService` 实例。

位置：`app/jobs/macros_execution_job.rb`

```ruby
conversations.each do |conversation|
  ::Macros::ExecutionService.new(macro, conversation, user).perform
end
```

**对话实例级隔离的是：**

| 实例变量 | 所属 | 隔离作用 |
|---------|------|---------|
| `@conversation` | `ActionService` | 每个实例独立的对话对象 |
| `@account` | `Macros::ExecutionService` | 虽然同属一账户，但通过实例变量访问 |
| `@user` | `Macros::ExecutionService` | 执行用户（同一宏通常相同） |
| `@macro` | `Macros::ExecutionService` | 宏定义（只读，各实例共享同一个对象引用） |

**对话实例级隔离的保证：**

1. **对象隔离**：每个对话有独立的 `Conversation` 模型实例
2. **reload 策略**：`ActionService.initialize` 中调用 `@conversation.reload`，确保读取最新状态
3. **异常隔离**：单个对话执行失败不影响其他对话

**场景分析：**

```
同一线程内，批量执行对话 A、B、C

┌─────────────────────────────────────────────────────────────┐
│  Thread.current                                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  对话 A                                              │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │  ExecutionService#1                          │    │    │
│  │  │  @conversation = Conversation_A             │    │    │
│  │  │  @user = User_X                             │    │    │
│  │  │  Current.user = User_X (TLS)                │    │    │
│  │  │    ↓                                        │    │    │
│  │  │  执行动作 → 修改 Conversation_A             │    │    │
│  │  │    ↓                                        │    │    │
│  │  │  ensure: Current.reset                      │    │    │
│  │  └─────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  对话 B (独立实例，与 A 无状态共享)                   │    │
│  │  ┌─────────────────────────────────────────────┐    │    │
│  │  │  ExecutionService#2                          │    │    │
│  │  │  @conversation = Conversation_B             │    │    │
│  │  │  @user = User_X                             │    │    │
│  │  │  Current.user = User_X (TLS，重置后重新设置) │    │    │
│  │  │    ↓                                        │    │    │
│  │  │  执行动作 → 修改 Conversation_B             │    │    │
│  │  └─────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**关键隔离点：**

```ruby
# 每次调用 new 都是新实例
service1 = Macros::ExecutionService.new(macro, conversation_A, user)
service2 = Macros::ExecutionService.new(macro, conversation_B, user)

# service1 和 service2 的 @conversation 不同
service1.instance_variable_get(:@conversation)  # => Conversation_A
service2.instance_variable_get(:@conversation)  # => Conversation_B

# 但 @macro 是同一个对象引用（只读，不影响隔离）
service1.instance_variable_get(:@macro).object_id == service2.instance_variable_get(:@macro).object_id
# => true（共享同一宏对象，但宏是只读的）
```

---

#### 两层隔离的协作关系

```
                    线程级隔离
                    (Thread Local Storage)
                    │
                    ├── 不同线程: 天然隔离 (Sidekiq 多 worker)
                    │
                    └── 同一线程: 需要时序隔离
                           │
                           └── 通过对话实例级隔离实现
                                │
                                ├── 每个对话独立 ExecutionService 实例
                                ├── 每个实例独立的 @conversation
                                ├── 每次 new() 时设置 Current.user
                                └── 每次 ensure 块中 Current.reset
```

**隔离边界对比表：**

| 维度 | 线程级隔离 | 对话实例级隔离 |
|------|-----------|---------------|
| **实现机制** | `thread_mattr_accessor` | 每次循环创建新 `ExecutionService` 对象 |
| **隔离对象** | `Current.*` 全局上下文 | `@conversation`、`@account`、`@user` 等实例变量 |
| **跨线程** | 天然隔离 | 无关（对象在各自线程栈） |
| **同线程跨对话** | 需要 `Current.reset` 配合 | 天然隔离（不同对象实例） |
| **Rails 模型** | 全局依赖（模型回调读取 `Current.user`） | 局部依赖（通过参数传入） |
| **隔离失败后果** | 用户上下文泄漏，A 的操作被记为 B 执行 | 可能污染前一个对话的对象状态（但有 reload 防护） |

**reload 机制的关键作用：**

位置：`app/services/action_service.rb`

```ruby
def initialize(conversation)
  @conversation = conversation.reload  # ← 关键：确保获取最新状态
  @account = @conversation.account
end
```

位置：`app/services/macros/execution_service.rb`

```ruby
def send_message(message)
  # ...
  mb = Messages::MessageBuilder.new(@user, @conversation.reload, params)  # ← 执行前再次 reload
  mb.perform
end
```

即使 `@conversation` 对象实例被"污染"（例如同一线程前一个对话的对象残留），通过 `reload` 也能确保从数据库读取最新状态。

---

### 4.4 为什么需要 Current 模块？

虽然 `ExecutionService` 已经通过实例变量持有 `@user`，但系统中许多其他组件依赖 `Current.user`：

1. **模型回调**：如 `AssignmentHandler`、`ActivityMessageHandler`
2. **审计日志**：记录操作执行人
3. **消息构建**：`Messages::MessageBuilder` 确定发送者身份

**示例**（位置：`app/models/concerns/assignment_handler.rb`）：
```ruby
user_name = Current.user.name if Current.user.present?
```

### 4.5 与 BulkActionsJob 的对比

位置：`app/jobs/bulk_actions_job.rb`

```ruby
class BulkActionsJob < ApplicationJob
  def perform(account:, params:, user:)
    Current.user = user  # 只在开始设置一次
    # ... 批量处理多个对话
  ensure
    Current.reset  # 最后清理一次
  end
end
```

**差异：**
- `BulkActionsJob`：整个批量操作共享一个 `Current.user`，只在首尾设置/清理
- `MacrosExecutionService`：**每个对话**独立设置/清理

**原因：**
- 宏动作可能包含复杂的嵌套调用（如 Webhook、邮件发送），需要更严格的隔离
- 单个动作失败不应该影响后续动作的上下文

---

## 5. 架构总结

### 5.1 快捷回复 vs 宏命令

| 特性 | 快捷回复 | 宏命令 |
|------|---------|--------|
| **本质** | 文本模板 | 自动化操作集合 |
| **存储** | 纯文本 | JSONB 动作数组 + 可选文件附件 |
| **触发方式** | 前端同步插入 | 后端异步任务 |
| **批量执行** | 不支持 | 支持（传入多个 conversation_ids） |
| **执行位置** | 前端编辑器 | 服务端后台作业 |
| **复杂性** | 低 | 高（多动作、异常处理） |

### 5.2 关键设计模式

1. **Service Object 模式**：`Macros::ExecutionService` 封装执行逻辑
2. **Job/Worker 模式**：`MacrosExecutionJob` 实现异步执行和解耦
3. **命令模式**：`actions` 数组中的每个对象都是一个可执行命令
4. **线程本地存储**：`Current` 模块实现请求/任务级别的上下文隔离

### 5.3 安全与隔离保障

| 层面 | 保障机制 |
|------|---------|
| **租户隔离** | 所有查询通过 `Current.account` 或 `account_id` 限制 |
| **可见性控制** | 宏的 `visibility` 字段（personal/global） |
| **线程隔离** | `thread_mattr_accessor` 确保跨线程不共享上下文 |
| **执行隔离** | 每个对话创建独立的 ExecutionService 实例 |
| **清理保障** | `ensure` 块 + `Current.reset` 防止上下文泄漏 |
| **动作白名单** | `ACTIONS_ATTRS` 限制可执行的动作类型 |
| **异常隔离** | 单个动作的异常被捕获，不中断整个宏执行 |

---

### 5.4 "执行前校验" vs "执行中隔离" — 边界总结

在分析宏命令的安全机制时，容易混淆两个独立层次：**执行前的权限校验**和**执行中的上下文隔离**。这两者目的不同、位置不同、失败后果也不同。

---

#### 5.4.1 执行前校验（Pre-Execution Validation）

**定义**：在宏**真正开始执行之前**，对"谁可以执行什么"进行的检查。

**层次结构：**

```
                    ┌─────────────────────────────────────┐
                    │       执行前校验层                     │
                    │                                     │
                    │  目的：防止非法操作的发起             │
                    │                                     │
                    │  ┌─────────────────────────────┐    │
                    │  │ 1. 租户级查询限制           │    │
                    │  │    Current.account.macros   │    │
                    │  │    find_by(id)              │    │
                    │  └─────────────────────────────┘    │
                    │              │                      │
                    │  ┌─────────────────────────────┐    │
                    │  │ 2. Pundit 权限校验          │    │
                    │  │    MacroPolicy#execute?     │    │
                    │  │    global? || author?       │    │
                    │  └─────────────────────────────┘    │
                    │              │                      │
                    │  ┌─────────────────────────────┐    │
                    │  │ 3. 对话归属验证             │    │
                    │  │    account.conversations    │    │
                    │  │    where(display_id: ...)   │    │
                    │  └─────────────────────────────┘    │
                    └─────────────────────────────────────┘
```

**执行前校验的特点：**

| 特性 | 说明 |
|------|------|
| **时机** | 任务入队前（第 1、2 层）或 Job 执行初期（第 3 层） |
| **目标** | 回答"这个操作**是否应该被允许**" |
| **失败后果** | 任务不执行（或部分不执行），返回错误或静默 |
| **典型问题** | 权限不足、跨租户访问、资源不存在 |

**三种失败路径回顾：**

| 校验层 | 校验失败时 | 宏是否执行 |
|--------|-----------|-----------|
| 租户级查询（`@macro` 为 nil） | 返回 200 但传 nil 给 Job | **Sidekiq 执行时报错**（潜在 bug） |
| Pundit 权限校验 | 返回 403 Forbidden | **完全不执行** |
| 对话归属验证 | 过滤掉无效/跨租户 ID | **部分执行或不执行**（静默） |

---

#### 5.4.2 执行中隔离（Execution-Time Isolation）

**定义**：在校验通过、宏**真正开始执行**后，确保不同对话、不同线程之间的状态不互相污染。

**层次结构：**

```
                    ┌─────────────────────────────────────┐
                    │       执行中隔离层                     │
                    │                                     │
                    │  目的：防止已授权操作之间的污染       │
                    │                                     │
                    │  ┌─────────────────────────────┐    │
                    │  │ 1. 线程级隔离               │    │
                    │  │    thread_mattr_accessor    │    │
                    │  │    Current.* (TLS)          │    │
                    │  └─────────────────────────────┘    │
                    │              │                      │
                    │  ┌─────────────────────────────┐    │
                    │  │ 2. 对话实例级隔离           │    │
                    │  │    每个对话独立              │    │
                    │  │    ExecutionService 实例    │    │
                    │  └─────────────────────────────┘    │
                    │              │                      │
                    │  ┌─────────────────────────────┐    │
                    │  │ 3. 对象 reload 防护         │    │
                    │  │    @conversation.reload     │    │
                    │  │    确保读取最新状态          │    │
                    │  └─────────────────────────────┘    │
                    └─────────────────────────────────────┘
```

**执行中隔离的特点：**

| 特性 | 说明 |
|------|------|
| **时机** | 在校验通过后，`Macros::ExecutionService#perform` 执行期间 |
| **目标** | 回答"这些已授权的操作**如何互不干扰**" |
| **失败后果** | 上下文泄漏（A 的操作被记为 B 执行）、对象状态污染 |
| **典型问题** | 忘记 `Current.reset`、同一线程内时序问题 |

**两种隔离维度回顾：**

| 维度 | 隔离对象 | 关键机制 |
|------|---------|---------|
| 线程级 | `Current.user`, `Current.account` 等全局上下文 | `thread_mattr_accessor` + `Current.reset` |
| 对话实例级 | `@conversation`, `@user` 等实例变量 | 每次 `new()` 创建独立实例 + `reload` |

---

#### 5.4.3 两层边界的明确划分

```
                    ┌─────────────────────────────────────────┐
                    │           请求生命周期                     │
                    │                                         │
                    │  ┌─────────────────────────────────┐    │
                    │  │                                 │    │
                    │  │   [执行前校验层]                  │    │
                    │  │                                 │    │
                    │  │   1. fetch_macro                │    │
                    │  │      @macro = Current.account   │    │
                    │  │         .macros.find_by(id)     │    │
                    │  │                                 │    │
                    │  │   2. check_authorization        │    │
                    │  │      MacroPolicy#execute?       │    │
                    │  │                                 │    │
                    │  └───────────────┬─────────────────┘    │
                    │                  │                        │
                    │        ✓ 校验通过 │ ✗ 校验失败            │
                    │                  │                        │
                    │                  ▼                        │
                    │  ┌─────────────────────────────────┐    │
                    │  │   [执行中隔离层]                  │    │
                    │  │                                 │    │
                    │  │   (只有校验通过后才进入)          │    │
                    │  │                                 │    │
                    │  │   3. MacrosExecutionJob         │    │
                    │  │      对话归属过滤                │    │
                    │  │                                 │    │
                    │  │   4. ExecutionService           │    │
                    │  │      - 线程级隔离 (Current.*)    │    │
                    │  │      - 实例级隔离 (@conversation)│    │
                    │  │      - reload 防护               │    │
                    │  │                                 │    │
                    │  └─────────────────────────────────┘    │
                    │                                         │
                    └─────────────────────────────────────────┘
```

**混淆时可能产生的错误理解：**

❌ **错误理解 1**："上下文隔离可以防止未授权用户执行宏"
- **纠正**：上下文隔离是**执行中**的机制，不负责权限校验。
- 权限校验由**执行前**的 Pundit 负责。
- 如果一个恶意用户已经绕过了 Pundit（例如通过 bug），上下文隔离无法阻止他的操作。

❌ **错误理解 2**："Pundit 校验失败后，Current 上下文可能泄漏"
- **纠正**：Pundit 校验在 `before_action` 中执行，此时 `Macros::ExecutionService` 还没有被实例化。
- `Current.user` 可能在请求开始时被设置（`set_current_user`），但请求结束时 `ensure` 块会清理。
- 上下文泄漏的风险主要在**执行中**，不是**执行前校验失败**时。

❌ **错误理解 3**："@macro 不存在时返回 200 是隔离问题"
- **纠正**：这是**执行前校验层**的 bug（`find_by` 返回 nil 而不是抛异常），与隔离无关。
- 隔离层关注的是"已授权操作之间的污染"，不是"操作是否应该被执行"。

---

#### 5.4.4 边界总结表

| 维度 | 执行前校验 | 执行中隔离 |
|------|-----------|-----------|
| **核心问题** | "这个操作**是否允许**？" | "这些操作**如何互不干扰**？" |
| **时机** | 执行之前（或刚开始） | 执行过程中 |
| **关键组件** | `fetch_macro`, `authorize`, `account.conversations.where` | `thread_mattr_accessor`, `Current.reset`, `new()`, `reload` |
| **失败模式** | 拒绝执行、返回错误、静默过滤 | 上下文泄漏、对象污染、数据不一致 |
| **用户可感知** | 是（403、200 但无效果） | 否（隐蔽 bug，审计日志错误等） |
| **与业务逻辑关系** | 直接关联（可见性规则、权限矩阵） | 间接关联（基础设施层） |

**一句话总结：**
> **执行前校验**是"守门员"，决定让不让进；
> **执行中隔离**是"隔间墙"，保证进来后各自在自己的位置上操作。

---

### 5.5 数据流图示

```
┌─────────────────────────────────────────────────────────────────┐
│                         前端层                                    │
│  ┌─────────────┐      ┌─────────────┐     ┌─────────────────┐  │
│  │ CannedResp  │      │ MacroItem   │     │ Bulk Actions    │  │
│  │ (同步)      │      │ (异步)      │     │ (支持多对话)    │  │
│  └──────┬──────┘      └──────┬──────┘     └────────┬────────┘  │
└─────────┼────────────────────┼─────────────────────┼───────────┘
          │                    │                     │
          ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                       API 控制器层                                │
│  ┌──────────────────────────┐  ┌─────────────────────────────┐  │
│  │ CannedResponsesController│  │    MacrosController         │  │
│  │  - index (搜索)          │  │  - execute                  │  │
│  │  - crud                  │  │    → MacrosExecutionJob     │  │
│  └──────────────────────────┘  └────────────┬────────────────┘  │
└─────────────────────────────────────────────┼───────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     ActiveJob 队列层                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              MacrosExecutionJob                           │  │
│  │  queue_as: :medium                                        │  │
│  │  遍历 conversation_ids，逐个执行                           │  │
│  └───────────────────────┬───────────────────────────────────┘  │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           ▼  (循环，每个对话独立)
┌─────────────────────────────────────────────────────────────────┐
│                   Service 层（每个对话独立实例）                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │         Macros::ExecutionService (继承 ActionService)     │  │
│  │                                                           │  │
│  │  initialize:                                              │  │
│  │    - Current.user = user  (设置线程上下文)                  │  │
│  │                                                           │  │
│  │  perform:                                                 │  │
│  │    ┌───────────────────────────────────────────────────┐  │  │
│  │    │  for action in macro.actions:                      │  │  │
│  │    │    ┌─────────────────────────────────────────┐    │  │  │
│  │    │    │  begin                                  │    │  │  │
│  │    │    │    send(action_name, action_params)    │    │  │  │
│  │    │    │  rescue                                │    │  │  │
│  │    │    │    capture_exception                   │    │  │  │
│  │    │    └─────────────────────────────────────────┘    │  │  │
│  │    └───────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  ensure:                                                  │  │
│  │    - Current.reset  (清理线程上下文)                       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 相关文件索引

### 6.1 核心实现

| 文件路径 | 说明 |
|----------|------|
| `app/models/canned_response.rb` | 快捷回复模型 |
| `app/models/macro.rb` | 宏命令模型 |
| `app/controllers/api/v1/accounts/canned_responses_controller.rb` | 快捷回复 API |
| `app/controllers/api/v1/accounts/macros_controller.rb` | 宏命令 API |
| `app/jobs/macros_execution_job.rb` | 宏执行异步任务 |
| `app/services/macros/execution_service.rb` | 宏执行服务（核心） |
| `app/services/action_service.rb` | 基础动作服务 |
| `lib/current.rb` | 线程上下文存储模块 |
| `app/policies/macro_policy.rb` | 宏权限策略（Pundit） |
| `app/controllers/concerns/request_exception_handler.rb` | 异常处理（含 Pundit） |

### 6.2 前端实现

| 文件路径 | 说明 |
|----------|------|
| `app/javascript/dashboard/components/widgets/conversation/CannedResponse.vue` | 快捷回复组件 |
| `app/javascript/dashboard/routes/dashboard/conversation/Macros/MacroItem.vue` | 宏执行组件 |
| `app/javascript/dashboard/api/macros.js` | 宏 API 调用封装 |
| `app/javascript/dashboard/store/modules/macros.js` | 宏 Vuex store |
| `app/javascript/dashboard/helper/editorHelper.js` | 编辑器辅助（含快捷回复插入） |

### 6.3 数据库

| 文件路径 | 说明 |
|----------|------|
| `db/schema.rb` | 数据库 schema（包含 canned_responses 和 macros 表定义） |
| `db/migrate/20230426130150_init_schema.rb` | 初始化迁移 |

---

## 7. 测试参考

| 文件路径 | 说明 |
|----------|------|
| `spec/services/macros/execution_service_spec.rb` | 宏执行服务测试用例 |

---

*报告生成时间：2026-05-11*
