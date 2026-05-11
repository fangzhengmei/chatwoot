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

### 3.3 过滤条件叠加

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
```

**配置参数**（在 `AssignmentPolicy` 中）：
- `fair_distribution_limit`：每个时间窗口内最多分配的会话数（默认 100）
- `fair_distribution_window`：时间窗口秒数（默认 3600 = 1 小时）

**工作原理**：
1. 每次分配在 Redis 中记录一个带过期时间的 key
2. 分配前检查该 agent 在当前时间窗口内的分配记录数
3. 超过限制则跳过该 agent

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
