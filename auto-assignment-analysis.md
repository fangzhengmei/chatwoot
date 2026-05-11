# Chatwoot 会话自动分配机制分析报告

## 1. 概述

Chatwoot 的会话自动分配系统包含两个核心版本：
- **V1（传统系统）**：基于 `AgentAssignmentService` 的简单轮询
- **V2（新系统）**：基于 `AssignmentService`，支持高级分配策略

本文重点分析 V2 系统中的轮询（Round Robin）与负载均衡（Balanced）选择机制，以及跨团队的工作量分配规则。

---

## 2. 轮询 vs 负载均衡：选择机制

### 2.1 选择条件

系统在 `enterprise/app/services/enterprise/auto_assignment/assignment_service.rb:26` 中决定使用哪种分配策略：

```ruby
selector = policy&.balanced? && account.feature_enabled?('advanced_assignment') ? balanced_selector : round_robin_selector
```

**选择规则：**

| 条件 | 使用策略 |
|------|---------|
| 非企业版 | 仅轮询 |
| 企业版但 `advanced_assignment` 功能未开启 | 仅轮询 |
| 企业版且 `advanced_assignment` 已开启，但策略为 `round_robin` | 轮询 |
| 企业版且 `advanced_assignment` 已开启，且策略为 `balanced` | 负载均衡 |

### 2.2 策略配置位置

策略选择在 `AssignmentPolicy` 模型中定义：

- **OSS 版** (`app/models/assignment_policy.rb:34`)：
  ```ruby
  enum assignment_order: { round_robin: 0 } unless ChatwootApp.enterprise?
  ```

- **企业版扩展** (`enterprise/app/models/enterprise/concerns/assignment_policy.rb:5`)：
  ```ruby
  enum assignment_order: { round_robin: 0, balanced: 1 } if ChatwootApp.enterprise?
  ```

### 2.3 两种策略的实现差异

#### 轮询策略 (Round Robin)

**核心文件**：
- `app/services/auto_assignment/round_robin_selector.rb`
- `app/services/auto_assignment/inbox_round_robin_service.rb`

**工作原理**：
1. 使用 Redis 维护一个循环队列（`ROUND_ROBIN_AGENTS` key）
2. 每次分配从队列头部取出一个 agent
3. 将该 agent 移至队列尾部
4. 确保分配均匀轮换

```ruby
# InboxRoundRobinService 核心逻辑
def get_member_from_allowed_agent_ids(allowed_agent_ids)
  user_id = queue.intersection(allowed_agent_ids).pop
  pop_push_to_queue(user_id)  # 移出并重新加入尾部
  user_id
end
```

**优点**：
- 简单公平，确保每个 agent 获得相同数量的分配机会
- 不依赖当前工作量历史

#### 负载均衡策略 (Balanced)

**核心文件**：
- `enterprise/app/services/enterprise/auto_assignment/balanced_selector.rb`

**工作原理**：
1. 统计每个可用 agent 当前在该 inbox 中**已打开的会话数量**
2. 选择会话数量最少的 agent

```ruby
# BalancedSelector 核心逻辑
def select_agent(available_agents)
  agent_users = available_agents.map(&:user)
  assignment_counts = fetch_assignment_counts(agent_users)
  # 选择当前打开会话最少的 agent
  agent_users.min_by { |user| assignment_counts[user.id] || 0 }
end

def fetch_assignment_counts(users)
  user_ids = users.map(&:id)
  counts = inbox.conversations
                .open
                .where(assignee_id: user_ids)
                .group(:assignee_id)
                .count
  Hash.new(0).merge(counts)
end
```

**优点**：
- 基于实际工作负载智能分配
- 避免某些 agent 过载

**适用场景对比**：

| 场景 | 推荐策略 |
|------|---------|
| 会话复杂度相似，希望绝对公平 | 轮询 |
| 会话处理时间差异大，需负载均衡 | Balanced |
| 企业版高级功能已启用 | 两者皆可，根据业务选择 |

### 2.4 Balanced 并列最小负载时的 Tie-Break 与公平性影响

**核心代码** (`enterprise/app/services/enterprise/auto_assignment/balanced_selector.rb:10`)：

```ruby
agent_users.min_by { |user| assignment_counts[user.id] || 0 }
```

**Tie-Break 机制**：

当多个 agent 具有相同的最小负载时，Ruby 的 `Enumerable#min_by` 行为是：

1. **顺序敏感性**：返回数组中**第一个**遇到的最小值
2. **无二次排序**：没有任何二级排序条件（如 ID、加入时间、最近分配时间等）
3. **依赖迭代顺序**：最终选择依赖 `available_agents.map(&:user)` 的数组顺序

**数组顺序来源**：

```ruby
# app/services/auto_assignment/assignment_service.rb:48
agents = filter_agents_by_team(inbox.available_agents, conversation)

# app/models/concerns/inbox_agent_availability.rb:8-11
inbox_members
  .joins(:user)
  .where(users: { id: online_agent_ids })
  .includes(:user)
```

由于 ActiveRecord 查询未显式 `order`，顺序取决于：
- 数据库的默认返回顺序（通常是 `id ASC` 或插入顺序）
- `filter_agents_by_team` 的 `where(user_id: team_member_ids)` 过滤结果顺序

**公平性影响**：

| 影响类型 | 具体表现 |
|---------|---------|
| **短期偏差** | 连续多个并列负载的会话可能全部分配给数组中的第一个 agent |
| **长期不公平** | 如果 agent 列表顺序长期稳定，靠前的 agent 会系统性地获得更多并列负载下的分配 |
| **与 Round Robin 对比** | 轮询通过 Redis 队列确保轮换，而 balanced 在并列时缺乏此保障 |

**测试确认** (`spec/enterprise/services/enterprise/auto_assignment/balanced_selector_spec.rb:48-58`)：

```ruby
it 'selects any agent when agents have equal workload' do
  # All agents have same number of conversations
  [member1, member2, member3].each do |member|
    create(:conversation, inbox: inbox, assignee: member.user, status: 'open')
  end

  selected_agent = selector.select_agent(available_agents)

  # Should select one of the agents (when equal, min_by returns the first one it finds)
  expect([agent1, agent2, agent3]).to include(selected_agent)
end
```

---

## 3. 跨团队工作量分配规则

### 3.1 团队过滤机制

当会话属于某个团队时，分配范围会被限制在该团队成员内：

**核心逻辑** (`app/services/auto_assignment/assignment_service.rb:57-65`)：

```ruby
def filter_agents_by_team(agents, conversation)
  return agents if conversation&.team_id.blank?

  team = conversation.team
  return nil if team.blank? || team.allow_auto_assign.blank?

  team_member_ids = team.members.ids
  agents.where(user_id: team_member_ids)
end
```

**分配流程**：
1. 检查会话是否关联了 `team_id`
2. 如果关联了团队，检查该团队的 `allow_auto_assign` 标志
3. 如果允许自动分配，过滤出同时属于 inbox 和该 team 的成员
4. 仅在过滤后的成员列表中进行轮询/负载均衡选择

### 3.2 团队自动分配开关

**Team 模型** (`app/models/team.rb:6`)：
```ruby
#  allow_auto_assign :boolean          default(TRUE)
```

每个团队都有 `allow_auto_assign` 配置：
- `true`（默认）：会话可以自动分配给团队成员
- `false`：跳过该会话，不进行自动分配

### 3.3 团队切换时 Assignee 失效、清空与重新分配的完整链路

#### 3.3.1 Conversation 模型中的两个独立 Handler

Conversation 模型同时 include 了两个 concern，它们在**不同生命周期阶段**处理团队切换：

```ruby
# app/models/conversation.rb:57-58
include AssignmentHandler      # ← 处理团队切换的 assignee 校验 + 同步重新分配
include AutoAssignmentHandler  # ← 处理状态变更触发的自动分配
```

| Handler | 触发时机 | 触发条件 | 主要职责 |
|---------|---------|---------|---------|
| `AssignmentHandler` | `before_save` | `team_id_changed?` | 校验 assignee 是否属于新团队 + 尝试同步重新分配 |
| `AutoAssignmentHandler` | `after_save` | 状态变更（open/resolved/snoozed） | 触发 V1 同步分配或 V2 异步 Job |

**关键点**：`team_id_changed?` 本身**不会**触发 `AutoAssignmentHandler`，只有状态变更才会触发。

---

#### 3.3.2 AssignmentHandler：团队切换的完整链路

**核心文件**：`app/models/concerns/assignment_handler.rb`

```ruby
included do
  before_save :ensure_assignee_is_from_team
  after_commit :notify_assignment_change, :process_assignment_changes
end

def ensure_assignee_is_from_team
  return unless team_id_changed?  # ← 仅当 team_id 变化时执行

  validate_current_assignee_team
  self.assignee ||= find_assignee_from_team
end
```

**完整执行流程（保存前）**：

```
conversation.team = new_team (或 conversation.update!(team: new_team))
    ↓
before_save 回调触发
    ↓
ensure_assignee_is_from_team
    ├── 1. validate_current_assignee_team（清空失效 assignee）
    └── 2. find_assignee_from_team（尝试从新团队重新分配）
    ↓
save（保存到数据库）
    ↓
after_commit
    ├── notify_assignment_change（派发 TEAM_CHANGED / ASSIGNEE_CHANGED 事件）
    └── process_assignment_changes（创建活动记录）
```

##### 3.3.2.1 第一步：Assignee 失效与清空

```ruby
# app/models/concerns/assignment_handler.rb:19-21
def validate_current_assignee_team
  self.assignee_id = nil if team&.members&.exclude?(assignee)
end
```

**清空条件**：`team&.members&.exclude?(assignee)`

| 切换场景 | 旧 assignee 在新团队中？ | 结果 |
|---------|------------------------|------|
| 无团队 → 有团队 A | 取决于 assignee 是否在 A 中 | 不在 → 清空 |
| 团队 A → 团队 B | 取决于 assignee 是否在 B 中 | 不在 → 清空 |
| 有团队 → 无团队 | `team` 为 nil，条件不触发 | **保留原 assignee** |
| 团队内成员间切换（不切换 team） | `team_id_changed?` 为 false | 不进入此逻辑 |

**注意**：从有团队切换到无团队时，`team` 变为 `nil`，`team&.members` 返回 `nil`，`nil&.exclude?(assignee)` 返回 `nil`（falsy），因此条件不满足，**不会清空 assignee**。

##### 3.3.2.2 第二步：重新分配尝试

如果上一步清空了 assignee（`self.assignee` 变为 `nil`），则尝试从新团队中自动分配：

```ruby
# app/models/concerns/assignment_handler.rb:23-28
def find_assignee_from_team
  return if team&.allow_auto_assign.blank?  # ← 团队禁用自动分配则跳过

  team_members_with_capacity = inbox.member_ids_with_assignment_capacity & team.members.ids
  ::AutoAssignment::AgentAssignmentService.new(conversation: self, allowed_agent_ids: team_members_with_capacity).find_assignee
end
```

**分配逻辑**：
1. 检查新团队的 `allow_auto_assign` 是否为 `true`
2. 计算候选 agent 集合：`inbox.member_ids_with_assignment_capacity` ∩ `team.members.ids`
3. 调用 `AgentAssignmentService#find_assignee`（**V1 逻辑，仅轮询**）

**AgentAssignmentService 的选择逻辑**：

```ruby
# app/services/auto_assignment/agent_assignment_service.rb:7-9
def find_assignee
  round_robin_manage_service.available_agent(allowed_agent_ids: allowed_online_agent_ids)
end

# 过滤出在线 agent
def allowed_online_agent_ids
  @allowed_online_agent_ids ||= online_agent_ids & allowed_agent_ids&.map(&:to_s)
end
```

**关键注意**：
- 此处使用的是 **V1 的 `AgentAssignmentService`**（Redis 轮询）
- 不经过 V2 的 `AssignmentService`
- 不应用 `RateLimiter`（fair_distribution_limit/window）
- 不应用 `CapacityService`（agent 容量限制）
- 不支持 Balanced 策略

---

#### 3.3.3 V1 与 V2 的触发条件与处理路径差异

##### 触发条件对比

**判断 V1/V2 的唯一标准**：

```ruby
# app/models/concerns/auto_assignment_handler.rb:17
if inbox.auto_assignment_v2_enabled?
  # V2 路径
else
  # V1 路径
end

# app/models/inbox.rb:206-208
def auto_assignment_v2_enabled?
  account.feature_enabled?('assignment_v2')
end
```

**团队切换场景的完整路径差异**：

| 阶段 | V1 路径（assignment_v2 未启用） | V2 路径（assignment_v2 已启用） |
|------|------------------------------|------------------------------|
| **before_save（团队切换）** | 同 V2：`AssignmentHandler` 清空 assignee + `find_assignee_from_team` 同步调用 `AgentAssignmentService` | 同 V1 |
| **after_save（状态变更）** | `AutoAssignmentHandler` 同步调用 `AgentAssignmentService` | `AutoAssignmentHandler` 触发 `AssignmentJob.perform_later`（异步） |
| **策略选择** | 始终轮询（Redis 队列） | before_save：轮询；异步 Job：根据策略选择轮询或 Balanced |
| **RateLimiter** | 不应用 | 异步 Job 中应用 |
| **CapacityService** | 不应用 | 异步 Job 中应用（企业版） |

##### 团队切换时的 V1 完整流程图

```
conversation.update!(team: new_team)
    ↓
before_save: AssignmentHandler
    ├── validate_current_assignee_team
    │    └── assignee 不在新团队中 → assignee_id = nil
    └── find_assignee_from_team
         ├── team.allow_auto_assign?
         ├── 计算交集: inbox.member_ids_with_assignment_capacity ∩ team.members.ids
         └── AgentAssignmentService#find_assignee（同步，仅轮询）
    ↓
save
    ↓
after_commit: AssignmentHandler
    ├── TEAM_CHANGED 事件（如果 team_id 变化）
    └── ASSIGNEE_CHANGED 事件（如果 assignee 变化）
    ↓
after_save: AutoAssignmentHandler（仅当状态变更时才触发）
    ├── conversation_status_changed_to_open?
    │    └── AgentAssignmentService#perform（同步）
    └── 注意：纯 team_id 变更（无状态变更）不会触发此路径
```

##### 团队切换时的 V2 完整流程图

```
conversation.update!(team: new_team)
    ↓
before_save: AssignmentHandler（与 V1 完全相同）
    ├── validate_current_assignee_team
    │    └── assignee 不在新团队中 → assignee_id = nil
    └── find_assignee_from_team
         └── AgentAssignmentService#find_assignee（同步，仅轮询）
    ↓
save
    ↓
after_commit: AssignmentHandler（与 V1 完全相同）
    ├── TEAM_CHANGED 事件
    └── ASSIGNEE_CHANGED 事件（如发生）
    ↓
after_save: AutoAssignmentHandler（仅当状态变更时才触发）
    ├── conversation_status_changed_to_open? || conversation_status_changed_to_resolved_or_snoozed?
    │    └── AutoAssignment::AssignmentJob.perform_later（异步）
    │         └── AssignmentService#perform_bulk_assignment
    │              ├── filter_agents_by_team
    │              ├── filter_agents_by_rate_limit（应用 fair_distribution）
    │              ├── filter_agents_by_capacity（企业版）
    │              └── 轮询 / Balanced 选择（根据策略）
    └── 注意：纯 team_id 变更（无状态变更）不会触发此路径
```

##### V1 vs V2 关键差异总结表

| 维度 | V1（assignment_v2 未启用） | V2（assignment_v2 已启用） |
|------|--------------------------|--------------------------|
| **团队切换的同步重新分配** | `AgentAssignmentService`（轮询） | 同 V1 |
| **状态变更的后续分配** | 同步 `AgentAssignmentService` | 异步 `AssignmentJob` |
| **支持的分配策略** | 仅轮询 | before_save：轮询；异步 Job：轮询或 Balanced |
| **fair_distribution 限速** | 不应用 | 异步 Job 中应用 |
| **Agent 容量限制** | 不应用 | 异步 Job 中应用（企业版） |
| **触发频率** | 同步执行 | 同步 + 异步可能再次执行 |

#### 3.3.4 特殊情况：Inbox 自动分配禁用时的团队分配

**测试确认** (`spec/enterprise/models/conversation_spec.rb:83-111`)：

```ruby
it 'does not enforce max_assignment_limit for team assignment when inbox auto-assignment is disabled' do
  conversation = create(:conversation, inbox: inbox, account: account, assignee: nil, status: :open)
  conversation.update!(team: team)
  expect(conversation.reload.assignee).to be_present
end
```

**原因**：`find_assignee_from_team` 使用 `inbox.member_ids_with_assignment_capacity`，而在 `Enterprise::Inbox` 中：

```ruby
# enterprise/app/models/enterprise/inbox.rb:2-9
def member_ids_with_assignment_capacity
  return super unless enable_auto_assignment?  # ← 关键检查
  return filter_by_capacity(available_agents).map(&:user_id) if auto_assignment_v2_enabled?

  max_assignment_limit = auto_assignment_config['max_assignment_limit']
  overloaded_agent_ids = max_assignment_limit.present? ? get_agent_ids_over_assignment_limit(max_assignment_limit) : []
  super - overloaded_agent_ids
end
```

当 `enable_auto_assignment?` 为 `false` 时，直接返回 `super`（所有成员 ID），**不应用任何容量限制**。

这意味着：**即使 inbox 禁用了自动分配，团队切换时仍会尝试分配，且不考虑容量限制**。

---

### 3.4 过滤条件叠加

跨团队分配时，会经过多层过滤：

```
inbox.available_agents
    ↓ filter_agents_by_team
    ↓ filter_agents_by_rate_limit
    ↓ filter_agents_by_capacity (企业版)
    ↓ 轮询/负载均衡选择
```

### 3.4 Agent 容量限制（企业版高级功能）

**核心文件**：
- `enterprise/app/services/enterprise/auto_assignment/capacity_service.rb`
- `enterprise/app/models/agent_capacity_policy.rb`
- `enterprise/app/models/inbox_capacity_limit.rb`

**容量检查逻辑**：

```ruby
# CapacityService
def agent_has_capacity?(user, inbox)
  account_user = user.account_users.find_by(account: inbox.account)
  return true unless account_user&.agent_capacity_policy

  policy = account_user.agent_capacity_policy
  inbox_limit = policy.inbox_capacity_limits.find_by(inbox: inbox)

  # 没有配置 inbox 限制 = 无限容量
  return true unless inbox_limit

  # 统计当前该 agent 在此 inbox 的打开会话数
  current_count = user.assigned_conversations
                      .where(inbox: inbox, status: :open)
                      .count

  # 必须小于限制值才继续分配
  current_count < inbox_limit.conversation_limit
end
```

**容量配置模型**：

```
AgentCapacityPolicy (每个 account_user 可关联一个)
  └── InboxCapacityLimit (每个 inbox 可配置一个限制)
         └── conversation_limit (整数，最大同时打开会话数)
```

**启用条件** (`enterprise/app/services/enterprise/auto_assignment/assignment_service.rb:37-40`)：
```ruby
def capacity_filtering_enabled?
  account.feature_enabled?('advanced_assignment') &&
    account.account_users.joins(:agent_capacity_policy).exists?
end
```

### 3.5 公平分配限速（Rate Limiting）

**核心文件**：`app/services/auto_assignment/rate_limiter.rb`

即使使用轮询或负载均衡，系统还会通过 Redis 进行时间窗口内的分配限速：

```ruby
def within_limit?
  current_count < limit
end

def limit
  config&.fair_distribution_limit.present? ? config.fair_distribution_limit.to_i : 5
end

def window
  config&.fair_distribution_window&.to_i || 5.minutes.to_i
end

def config
  @config ||= inbox.assignment_policy
end
```

**工作原理**：
1. 每次分配在 Redis 中记录一个带过期时间的 key
2. 分配前检查该 agent 在当前时间窗口内的分配记录数
3. 超过限制则跳过该 agent

### 3.6 fair_distribution_limit/window 在有策略与无策略下的真实默认值链路

#### 3.6.1 默认值来源的三层结构

系统存在**三层默认值**，优先级从高到低：

```
用户显式配置值
    ↓ (无)
数据库 schema 默认值 (AssignmentPolicy 字段)
    ↓ (无策略或字段为空)
代码硬编码 fallback (RateLimiter)
```

#### 3.6.2 有策略时（Inbox 关联了 AssignmentPolicy）

**数据库 Schema 默认值** (`db/schema.rb` + `app/models/assignment_policy.rb`)：

```ruby
# Schema:
#  fair_distribution_limit  :integer          default(100), not null
#  fair_distribution_window :integer          default(3600), not null
```

当创建 `AssignmentPolicy` 时，如果未显式指定，数据库会使用：

| 参数 | Schema 默认值 | 说明 |
|------|--------------|------|
| `fair_distribution_limit` | 100 | 每小时最多 100 次分配 |
| `fair_distribution_window` | 3600 | 窗口 = 3600 秒 = 1 小时 |

**代码获取链路**：

```ruby
# RateLimiter
def config
  @config ||= inbox.assignment_policy  # ← 存在策略
end

def limit
  # config&.fair_distribution_limit.present? → true（数据库有值）
  # 使用 config.fair_distribution_limit.to_i → 100
  config&.fair_distribution_limit.present? ? config.fair_distribution_limit.to_i : 5
end

def window
  # config&.fair_distribution_window&.to_i → 3600
  config&.fair_distribution_window&.to_i || 5.minutes.to_i  # 3600
end
```

#### 3.6.3 无策略时（Inbox 未关联 AssignmentPolicy）

当 `inbox.assignment_policy` 为 `nil` 时：

```ruby
# RateLimiter
def config
  @config ||= inbox.assignment_policy  # ← nil
end

def limit
  # config&.fair_distribution_limit.present? → nil&. → nil → false
  # 使用 fallback 值
  config&.fair_distribution_limit.present? ? ... : 5  # ← 5
end

def window
  # config&.fair_distribution_window&.to_i → nil
  # 使用 fallback 值
  config&.fair_distribution_window&.to_i || 5.minutes.to_i  # ← 300 秒
end
```

**代码硬编码 fallback**：

| 参数 | 代码 fallback | 说明 |
|------|--------------|------|
| `fair_distribution_limit` | 5 | 每 5 分钟最多 5 次分配 |
| `fair_distribution_window` | 300 秒 (5 分钟) | 窗口 = 5 分钟 |

#### 3.6.4 默认值对比表

| 场景 | `fair_distribution_limit` | `fair_distribution_window` | 限制速率 |
|------|--------------------------|---------------------------|---------|
| **有策略（默认创建）** | 100 | 3600 秒 (1 小时) | ~1.67 次/分钟 |
| **无策略** | 5 | 300 秒 (5 分钟) | ~1 次/分钟 |
| **差异比例** | 20× | 12× | 约 1.67× |

#### 3.6.5 注意事项

1. **不要混淆两套默认值**：
   - Schema 默认值是 `AssignmentPolicy` 记录创建时的数据库默认
   - 代码 fallback 是 `RateLimiter` 在**无策略或策略缺失**时的兜底

2. **两种限速机制并存**：
   - **V1**：`inbox.auto_assignment_config['max_assignment_limit']`（企业版）
     ```ruby
     # enterprise/app/models/enterprise/inbox.rb:6-8
     max_assignment_limit = auto_assignment_config['max_assignment_limit']
     overloaded_agent_ids = max_assignment_limit.present? ? get_agent_ids_over_assignment_limit(max_assignment_limit) : []
     ```
   - **V2**：`fair_distribution_limit/window`（RateLimiter）

   两者是**不同层次**的限速：
   - `max_assignment_limit`：基于当前**已打开会话数**的硬限制
   - `fair_distribution_limit/window`：基于**时间窗口内分配次数**的软限制

3. **测试中的 mock**：

在测试中经常直接 mock `auto_assignment_config` 返回特定值，绕过真实默认值链路：

```ruby
# spec/services/auto_assignment/assignment_service_spec.rb:193-196
allow(inbox).to receive(:auto_assignment_config).and_return({
  'fair_distribution_limit' => 2,
  'fair_distribution_window' => 3600
})
```

注意：实际生产代码中 `auto_assignment_config` 是 Inbox 的 JSONB 字段（用于 V1 的 `max_assignment_limit`），V2 的 `fair_distribution_*` 参数来自 `AssignmentPolicy` 模型。

---

## 4. 排除规则（企业版）

企业版还支持基于条件排除某些会话不参与自动分配：

**核心逻辑** (`enterprise/app/services/enterprise/auto_assignment/assignment_service.rb:75-97`)：

### 4.1 按标签排除
```ruby
def apply_label_exclusions(scope, excluded_labels)
  scope.tagged_with(excluded_labels, exclude: true, on: :labels)
end
```

### 4.2 按时间排除
```ruby
def apply_age_exclusions(scope, hours_threshold)
  scope.where('conversations.created_at >= ?', hours.hours.ago)
end
```

**配置位置**：`AgentCapacityPolicy.exclusion_rules` (JSONB 字段)

---

## 5. 完整分配流程总结

### 5.1 触发时机

`app/models/concerns/auto_assignment_handler.rb` 定义了触发自动分配的时机：

1. **会话状态变为 open**
2. **会话状态变为 resolved 或 snoozed**（仅 V2，用于重新平衡容量）

### 5.2 V2 完整流程图

```
会话状态变更
    ↓
inbox.auto_assignment_v2_enabled?
    ↓ (是)
AutoAssignment::AssignmentJob.perform_later
    ↓
AssignmentService#perform_bulk_assignment
    ↓
获取 unassigned_conversations (按策略排序)
    ├── 应用排除规则 (企业版)
    └── 按 conversation_priority 排序
         ├── earliest_created: created_at ASC
         └── longest_waiting: last_activity_at ASC, created_at ASC
    ↓
对每个会话执行:
    ├── 检查: open 且 assignee 为空
    ├── find_available_agent
    │    ├── filter_agents_by_team (团队过滤)
    │    ├── filter_agents_by_rate_limit (公平分配限速)
    │    └── filter_agents_by_capacity (企业版容量限制)
    ├── 选择策略
    │    ├── 轮询: RoundRobinSelector (Redis 队列)
    │    └── 负载均衡: BalancedSelector (最少当前会话)
    └── assign_conversation
         ├── 更新 assignee
         ├── 记录 rate limiter
         └── 派发 ASSIGNEE_CHANGED 事件
```

---

## 6. 关键配置参数汇总

### AssignmentPolicy 参数
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `assignment_order` | `round_robin` | 分配策略 |
| `conversation_priority` | `earliest_created` | 会话优先级排序 |
| `fair_distribution_limit` | 100 | 每窗口最大分配数 |
| `fair_distribution_window` | 3600 | 时间窗口（秒） |
| `enabled` | true | 策略是否启用 |

### Team 参数
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `allow_auto_assign` | true | 是否允许自动分配给团队成员 |

### AgentCapacityPolicy 参数（企业版）
| 参数 | 说明 |
|------|------|
| `exclusion_rules` | JSONB，排除规则（标签、时间等） |

### InboxCapacityLimit 参数（企业版）
| 参数 | 说明 |
|------|------|
| `conversation_limit` | 每个 agent 在该 inbox 的最大同时打开会话数 |

---

## 7. 代码位置索引

| 功能 | 文件路径 |
|------|---------|
| 基础分配服务 | `app/services/auto_assignment/assignment_service.rb` |
| 企业版分配扩展 | `enterprise/app/services/enterprise/auto_assignment/assignment_service.rb` |
| 轮询选择器 | `app/services/auto_assignment/round_robin_selector.rb` |
| 轮询 Redis 队列 | `app/services/auto_assignment/inbox_round_robin_service.rb` |
| 负载均衡选择器 | `enterprise/app/services/enterprise/auto_assignment/balanced_selector.rb` |
| 容量检查服务 | `enterprise/app/services/enterprise/auto_assignment/capacity_service.rb` |
| 公平分配限速 | `app/services/auto_assignment/rate_limiter.rb` |
| 分配策略模型 | `app/models/assignment_policy.rb` |
| 企业版策略扩展 | `enterprise/app/models/enterprise/concerns/assignment_policy.rb` |
| 容量策略模型 | `enterprise/app/models/agent_capacity_policy.rb` |
| Inbox 容量限制 | `enterprise/app/models/inbox_capacity_limit.rb` |
| 团队模型 | `app/models/team.rb` |
| 自动分配触发 | `app/models/concerns/auto_assignment_handler.rb` |

---

## 8. 功能开关矩阵

| 功能 | OSS 版 | 企业版（需开启 advanced_assignment） |
|------|--------|-------------------------------------|
| 轮询分配 | ✅ | ✅ |
| 负载均衡分配 | ❌ | ✅ |
| 团队成员过滤 | ✅ | ✅ |
| 团队自动分配开关 | ✅ | ✅ |
| 公平分配限速 | ✅ | ✅ |
| Agent 容量限制 | ❌ | ✅ |
| 会话排除规则 | ❌ | ✅ |
| 会话优先级排序 | ✅ | ✅ |
