# 快捷回复与宏命令服务端架构分析报告

## 1. 概览

本文档分析 Chatwoot 中快捷回复（Canned Responses）和宏命令（Macros）在服务端的存储机制、触发流程以及跨多个对话时的执行上下文隔离策略。

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

### 3.1.6 执行前的完整校验链路

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

**完整执行校验链路：**

```
POST /macros/:id/execute
    ↓
1. fetch_macro: Current.account.macros.find_by(id: params[:id])
   ├── 限制在 Current.account 内查询（租户隔离）
   └── 找不到返回 nil，后续 authorize 不报错但 @macro 为 nil
    ↓
2. check_authorization: authorize(@macro) if @macro.present?
   └── 调用 MacroPolicy#execute?
       ├── global? → true（全局宏任何人可执行）
       └── author? → created_by == current_user（个人宏仅作者可执行）
    ↓
3. 若授权通过
   └── MacrosExecutionJob.perform_later(@macro, conversation_ids, Current.user)
    ↓
4. MacrosExecutionJob#perform
   └── account.conversations.where(display_id: conversation_ids)
       └── 二次验证对话属于同一账户
```

**三层校验保障：**

| 层次 | 校验点 | 代码位置 | 失败表现 |
|------|--------|---------|---------|
| 第1层 | 租户级查询限制 | `Current.account.macros.find_by` | @macro = nil，不执行 |
| 第2层 | Pundit 权限校验 | `MacroPolicy#execute?` | 抛出 Pundit::NotAuthorizedError → 403 |
| 第3层 | 对话归属验证 | `account.conversations.where` | 不在同一账户的对话被过滤 |

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

### 5.4 数据流图示

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

### 6.2 前端实现

| 文件路径 | 说明 |
|----------|------|
| `app/javascript/dashboard/components/widgets/conversation/CannedResponse.vue` | 快捷回复组件 |
| `app/javascript/dashboard/routes/dashboard/conversation/Macros/MacroItem.vue` | 宏执行组件 |
| `app/javascript/dashboard/api/macros.js` | 宏 API 调用封装 |
| `app/javascript/dashboard/store/modules/macros.js` | 宏 Vuex store |

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
