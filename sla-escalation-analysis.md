# Chatwoot SLA 服务等级协议分析报告

## 目录
1. [概述](#1-概述)
2. [核心数据模型](#2-核心数据模型)
3. [SLA 计时机制](#3-sla-计时机制)
4. [营业时间识别](#4-营业时间识别)
5. [升级告警触发条件](#5-升级告警触发条件)
6. [后台任务调度](#6-后台任务调度)
7. [数据反馈到前端](#7-数据反馈到前端)
8. [管理端 SLA 报表数据链路](#8-管理端-sla-报表数据链路)
9. [前端状态计算与显示](#9-前端状态计算与显示)
10. [关键代码位置汇总](#10-关键代码位置汇总)

---

## 1. 概述

Chatwoot 的 SLA (Service Level Agreement，服务等级协议) 是一个企业级功能，用于监控和保证客户服务的响应时间和解决时间。系统通过三个核心阈值来衡量服务质量：

- **FRT (First Response Time)**: 首次响应时间 - 从客户发起对话到客服首次回复的时间
- **NRT (Next Response Time)**: 后续响应时间 - 客户后续消息发送后的客服响应时间
- **RT (Resolution Time)**: 解决时间 - 对话从创建到标记为已解决的时间

---

## 2. 核心数据模型

### 2.1 SlaPolicy - SLA 策略模型

**文件位置**: `enterprise/app/models/sla_policy.rb`

```ruby
# 核心字段
- name: SLA 策略名称
- description: 策略描述
- first_response_time_threshold: 首次响应时间阈值（秒）
- next_response_time_threshold: 后续响应时间阈值（秒）
- resolution_time_threshold: 解决时间阈值（秒）
- only_during_business_hours: 是否仅在营业时间内计算（布尔值，默认 false）
- account_id: 所属账户
```

**关联关系**:
- 一个账户可拥有多个 SLA 策略
- 一个 SLA 策略可应用于多个对话
- 策略被删除时，已应用的 SLA 记录 (`applied_slas`) 会异步销毁

### 2.2 AppliedSla - 已应用的 SLA 记录

**文件位置**: `enterprise/app/models/applied_sla.rb`

```ruby
# 核心字段
- sla_status: SLA 状态（枚举）
  - active: 0 - 活动中
  - hit: 1 - 已达成
  - missed: 2 - 已错过
  - active_with_misses: 3 - 活动中但有错过项
- account_id: 账户 ID
- conversation_id: 对话 ID
- sla_policy_id: SLA 策略 ID
```

**状态流转**:
- `active` → `active_with_misses`: 当任一阈值被错过时
- `active` → `hit`: 对话成功解决且无任何错过项
- `active_with_misses` → `missed`: 对话解决时存在错过项

**过滤 Scope** (applied_sla.rb:32-42):
```ruby
scope :filter_by_date_range, ->(range) { where(created_at: range) if range.present? }
scope :filter_by_inbox_id, ->(inbox_id) { ... }
scope :filter_by_team_id, ->(team_id) { ... }
scope :filter_by_sla_policy_id, ->(sla_policy_id) { ... }
scope :filter_by_label_list, lambda { |label_list| ... }
scope :filter_by_assigned_agent_id, lambda { |assigned_agent_id| ... }
scope :missed, -> { where(sla_status: %i[missed active_with_misses]) }
```

### 2.3 SlaEvent - SLA 事件记录

**文件位置**: `enterprise/app/models/sla_event.rb`

```ruby
# 核心字段
- event_type: 事件类型（枚举）
  - frt: 0 - 首次响应时间错过
  - nrt: 1 - 后续响应时间错过
  - rt: 2 - 解决时间错过
- meta: JSONB 字段，存储额外元数据（如 nrt 事件中的 message_id）
- applied_sla_id: 关联的应用 SLA 记录
- conversation_id: 对话 ID
- account_id: 账户 ID
- inbox_id: 收件箱 ID
- sla_policy_id: SLA 策略 ID
```

**事件触发后的动作** (sla_event.rb:64-86):
1. 通知对话参与者 (`conversation_participants`)
2. 通知账户所有管理员 (`account.administrators`)
3. 确保通知对话负责人 (`conversation.assignee`)
4. 根据事件类型创建不同类型的通知：
   - `frt` → `sla_missed_first_response`
   - `nrt` → `sla_missed_next_response`
   - `rt` → `sla_missed_resolution`

### 2.4 Conversation - 对话模型 (Enterprise 扩展)

**文件位置**: `enterprise/app/models/enterprise/concerns/conversation.rb`

Enterprise 版为对话模型添加了 SLA 相关关联：
```ruby
belongs_to :sla_policy, optional: true
has_one :applied_sla, dependent: :destroy_async
has_many :sla_events, dependent: :destroy_async
```

**关键逻辑**:
- 验证 SLA 策略是否属于同一账户
- 当对话被分配 SLA 策略时，自动创建 `AppliedSla` 记录
- 在事务中确保数据一致性

---

## 3. SLA 计时机制

### 3.1 核心评估服务

**文件位置**: `enterprise/app/services/sla/evaluate_applied_sla_service.rb`

#### 评估流程 (`perform` 方法，第 4-12 行):

```ruby
def perform
  check_sla_thresholds  # 检查所有 SLA 阈值
  
  # 仅在对话已解决时进行最终判定
  return unless applied_sla.conversation.resolved?
  
  handle_hit_sla(applied_sla)  # 标记为 hit 或 missed
end
```

#### 阈值检查逻辑 (`check_sla_thresholds` 方法，第 16-22 行):

系统会依次检查三个阈值，每个阈值有对应的检查方法。

### 3.2 首次响应时间 (FRT) 计算

**方法**: `check_first_response_time_threshold` (第 28-34 行)

```ruby
def check_first_response_time_threshold(applied_sla, conversation, sla_policy)
  # 计算截止时间：对话创建时间 + FRT 阈值
  threshold = conversation.created_at.to_i + sla_policy.first_response_time_threshold.to_i
  
  # 如果已经在阈值内回复了，直接返回
  return if first_reply_was_within_threshold?(conversation, threshold)
  
  # 如果当前时间还没到截止时间，直接返回
  return if still_within_threshold?(threshold)
  
  # 否则，标记为错过 FRT
  handle_missed_sla(applied_sla, 'frt')
end
```

**判断逻辑**:
1. `first_reply_was_within_threshold?`: 检查 `first_reply_created_at` 是否存在且在阈值内
2. `still_within_threshold?`: `Time.zone.now.to_i < threshold`

### 3.3 后续响应时间 (NRT) 计算

**方法**: `check_next_response_time_threshold` (第 40-50 行)

```ruby
def check_next_response_time_threshold(applied_sla, conversation, sla_policy)
  # 如果还没有首次回复，归属于 FRT 检查
  return if conversation.first_reply_created_at.blank?
  
  # 如果不是等待客服回复状态，不需要检查
  return if conversation.waiting_since.blank?
  
  # 计算截止时间：等待开始时间 + NRT 阈值
  threshold = conversation.waiting_since.to_i + sla_policy.next_response_time_threshold.to_i
  
  return if still_within_threshold?(threshold)
  
  handle_missed_sla(applied_sla, 'nrt')
end
```

**关键概念 - `waiting_since`**:
- 表示对话处于"等待客服回复"状态的开始时间
- 当客户发送新消息时更新
- 客服回复后置为 `nil`

**NRT 事件元数据**: 记录最后一条客户消息的 ID，用于防止重复通知

### 3.4 解决时间 (RT) 计算

**方法**: `check_resolution_time_threshold` (第 61-68 行)

```ruby
def check_resolution_time_threshold(applied_sla, conversation, sla_policy)
  # 如果对话已解决，不进行 RT 错过标记
  # 注意：对话解决时由 handle_hit_sla 统一处理最终状态
  return if conversation.resolved?
  
  threshold = conversation.created_at.to_i + sla_policy.resolution_time_threshold.to_i
  return if still_within_threshold?(threshold)
  
  handle_missed_sla(applied_sla, 'rt')
end
```

### 3.5 RT 边界条件分析

#### 3.5.1 关键代码路径

从 `evaluate_applied_sla_service.rb` 的代码结构来看，RT 检查存在两条独立路径：

**路径一：阈值检查（对话未解决时）**
```ruby
# check_resolution_time_threshold (第 61-68 行)
return if conversation.resolved?  # ← 关键点：已解决则跳过
```

**路径二：最终判定（对话解决时）**
```ruby
# perform (第 4-12 行)
check_sla_thresholds  # 会跳过 RT 检查
return unless applied_sla.conversation.resolved?
handle_hit_sla(applied_sla)  # 只检查 active? 状态
```

#### 3.5.2 RT 错过事件创建条件

`SlaEvent` (RT 类型) 只会在以下场景创建：

| 场景 | 是否创建 RT 事件 | AppliedSla 最终状态 |
|-----|-----------------|--------------------|
| 对话未解决 + RT 已超时 | ✅ 是 | `active_with_misses` |
| 对话已解决（在 RT 超时前解决） | ❌ 否 | `hit` |
| 对话已解决（在 RT 超时后才解决，但解决前未被后台任务检查到） | ❌ 否 | `hit`（**异常**） |
| 对话已解决 + 解决前已有其他 SLA 错过 | ❌ 不检查 RT | `missed`（基于已有错过） |

#### 3.5.3 边界场景："小幅度超时但未创建 RT 事件"

**规格说明验证** (spec/enterprise/services/sla/evaluate_applied_sla_service_spec.rb:87-99):

```ruby
# 规格描述
# We will mark resolved miss only if while processing the SLA
# if the conversation is resolved and the resolution time is missed
# by small margins then we will not mark it as missed

# 测试场景：
# 1. 对话 6 小时前创建
# 2. RT 阈值设置为 1 小时（实际已超时 5 小时）
# 3. 对话先被标记为 resolved
# 4. 然后执行 SLA 评估

it 'does not update the SLA status to missed' do
  conversation.resolved!
  applied_sla.sla_policy.update(resolution_time_threshold: 1.hour)
  described_class.new(applied_sla: applied_sla).perform
  expect(applied_sla.reload.sla_status).to eq('hit')  # ← 注意：结果是 hit
end
```

**场景重现**:
```
时间线：
T0: 对话创建 (RT 阈值 = 1小时，截止时间 = T0+1h)
T0+2h: 后台任务执行，检查到 RT 超时 → 创建 RT 事件 → 状态变为 active_with_misses
    (这是正常路径)

但如果：
T0: 对话创建 (RT 阈值 = 1小时)
T0+30min: 客服仍在处理中...
T0+2h: 客服标记对话为 resolved (此时已超时 1 小时)
T0+2h+5min: 后台任务执行评估
    → check_resolution_time_threshold 看到 conversation.resolved? = true，直接跳过
    → handle_hit_sla 检查 applied_sla.active? = true
    → 最终状态为 hit！(不会创建 RT 事件)
```

#### 3.5.4 影响分析

**业务影响**:
1. **RT 统计不完整**: 管理端报表的 SLA 事件列表中不会包含此类"解决时才发现的 RT 超时"
2. **告警遗漏**: 不会触发 `sla_missed_resolution` 类型的通知
3. **Hit Rate 偏高**: `(total - misses) / total` 的计算结果可能高于实际情况
4. **事件一致性**: FRT 和 NRT 在对话解决后仍可能检查到超时（通过之前的事件记录），但 RT 不会

**技术原因**:
- `check_resolution_time_threshold` 中的 `return if conversation.resolved?` 过早返回
- `handle_hit_sla` 只判断 `applied_sla.active?`，不重新验证 RT 阈值

**设计意图推测** (从规格注释):
> "if the conversation is resolved and the resolution time is missed by small margins then we will not mark it as missed"

这似乎是一个有意设计的"宽容"策略：
- 如果对话已经解决，即使 RT 略微超时，也不计为错过
- 但这个"小幅度"没有量化标准，实际上是**完全不检查**

### 3.6 错过处理逻辑

**方法**: `handle_missed_sla` (第 70-80 行)

```ruby
def handle_missed_sla(applied_sla, type, meta = {})
  # NRT 事件需要记录关联的消息 ID
  meta = { message_id: get_last_message_id(applied_sla.conversation) } if type == 'nrt'
  
  # 防止重复创建同一事件
  return if already_missed?(applied_sla, type, meta)
  
  # 创建 SLA 事件记录（触发通知）
  create_sla_event(applied_sla, type, meta)
  
  # 记录警告日志
  Rails.logger.warn "SLA #{type} missed for conversation #{applied_sla.conversation.id}..."
  
  # 更新状态为 active_with_misses
  applied_sla.update!(sla_status: 'active_with_misses') if applied_sla.sla_status != 'active_with_misses'
end
```

**防重复机制**:
- `already_missed?` 方法检查是否已存在相同 `event_type` 和 `meta` 的事件
- 对于 NRT，`message_id` 确保每条客户消息的错过只通知一次

### 3.7 对话解决时的最终判定

**方法**: `handle_hit_sla` (第 82-94 行)

```ruby
def handle_hit_sla(applied_sla)
  if applied_sla.active?
    # 如果一直是 active 状态，说明所有阈值都在时限内
    applied_sla.update!(sla_status: 'hit')
    Rails.logger.info "SLA hit for conversation ..."
  else
    # 如果有过错过记录，最终标记为 missed
    applied_sla.update!(sla_status: 'missed')
    Rails.logger.info "SLA missed for conversation ..."
  end
end
```

---

## 4. 营业时间识别

### 4.1 WorkingHour 模型

**文件位置**: `app/models/working_hour.rb`

```ruby
# 核心字段
- day_of_week: 星期几 (0-6，对应周日到周六)
- open_hour: 开门时间（小时，0-23）
- open_minutes: 开门时间（分钟，0-59）
- close_hour: 关门时间（小时，0-23）
- close_minutes: 关门时间（分钟，0-59）
- open_all_day: 全天开放
- closed_all_day: 全天关闭
- inbox_id: 关联的收件箱
- account_id: 关联的账户
```

**关键方法**:
- `open_at?(time)`: 检查指定时间是否在营业时间内
- `open_now?`: 检查当前时间是否在营业时间内
- `closed_now?`: 检查当前是否非营业时间

```ruby
def open_at?(time)
  return false if closed_all_day?
  
  # 考虑收件箱所在时区
  open_time = Time.zone.now.in_time_zone(inbox.timezone).change({ hour: open_hour, min: open_minutes })
  close_time = Time.zone.now.in_time_zone(inbox.timezone).change({ hour: close_hour, min: close_minutes })
  
  time.between?(open_time, close_time)
end
```

### 4.2 SLA 策略中的营业时间设置

SlaPolicy 有一个 `only_during_business_hours` 字段（默认 `false`）。

**当前代码分析**:

通过查看 `evaluate_applied_sla_service.rb` 的实现，发现当前版本的 SLA 计时逻辑**并没有真正实现**基于营业时间的计算。

**证据**:
```ruby
# evaluate_applied_sla_service.rb 中的阈值计算
threshold = conversation.created_at.to_i + sla_policy.first_response_time_threshold.to_i
# 直接使用 created_at + 阈值，没有考虑 only_during_business_hours
```

**字段存在但未使用**:
- 字段在数据库和模型中定义 (`sla_policy.rb:10`)
- 在 API 中可配置 (`sla_policies_controller.rb:26`)
- 推送到前端 (`applied_sla.rb:57`)
- 但在核心评估服务中未被引用

**结论**:
当前实现的 SLA 计时是**全天候**的，无论营业时间设置如何。`only_during_business_hours` 字段是一个预留功能，尚未在业务逻辑中实现。

---

## 5. 升级告警触发条件

### 5.1 告警触发流程

当 SLA 阈值被错过时，系统按以下流程触发告警：

```
时间点错过阈值
    ↓
handle_missed_sla() 被调用
    ↓
创建 SlaEvent 记录
    ↓
SlaEvent after_create_commit 回调
    ↓
create_notifications() 方法
    ↓
为相关用户创建 Notification 记录
```

### 5.2 通知接收者 (sla_event.rb:64-70)

```ruby
def create_notifications
  notify_users = conversation.conversation_participants.map(&:user)
  # 所有对话参与者
  
  notify_users += account.administrators
  # 账户所有管理员
  
  notify_users += [conversation.assignee] if conversation.assignee.present?
  # 对话负责人（确保被通知）

  # 去重后通知每个用户
  notify_users.uniq.each do |user|
    NotificationBuilder.new(...).perform
  end
end
```

### 5.3 通知类型映射

| SLA 事件类型 | 通知类型 |
|-------------|---------|
| `frt` (首次响应错过) | `sla_missed_first_response` |
| `nrt` (后续响应错过) | `sla_missed_next_response` |
| `rt` (解决时间错过) | `sla_missed_resolution` |

### 5.4 通知构建器

**文件位置**: `app/builders/notification_builder.rb`

```ruby
def build_notification
  return if notification_type == 'conversation_creation' && !user_subscribed_to_notification?
  
  user.notifications.create!(
    notification_type: notification_type,
    account: account,
    primary_actor: conversation,      # 对话作为主角色
    secondary_actor: sla_policy        # SLA 策略作为次角色
  )
end
```

### 5.5 防重复机制

**already_missed? 检查** (第 57-59 行):
```ruby
def already_missed?(applied_sla, type, meta = {})
  SlaEvent.exists?(applied_sla: applied_sla, event_type: type, meta: meta)
end
```

- **FRT**: 每个对话只会触发一次（首次回复时间只有一个）
- **NRT**: 每条客户消息可能触发一次（通过 `message_id` 区分）
- **RT**: 每个对话只会触发一次（在对话解决前的超时）

---

## 6. 后台任务调度

### 6.1 调度配置出处

**配置文件**: `config/schedule.yml` (第 12-15 行)

```yaml
# executed At every 5th minute..
trigger_scheduled_items_job:
  cron: '*/5 * * * *'
  class: 'TriggerScheduledItemsJob'
  queue: scheduled_jobs
```

**配置验证**:
- Cron 表达式: `*/5 * * * *` → **每 5 分钟执行一次**
- 使用 `sidekiq-cron` 实现定时调度
- 验证配置在 `/spec/configs/schedule_spec.rb`

### 6.2 调度层次结构

```
config/schedule.yml (Cron 配置: 每 5 分钟)
    ↓
TriggerScheduledItemsJob (OSS 版本入口)
    ↓ prepend_mod_with
Enterprise::TriggerScheduledItemsJob (Enterprise 扩展)
    ↓
Sla::TriggerSlasForAccountsJob
    ↓ (为每个有 SLA 策略的账户)
Sla::ProcessAccountAppliedSlasJob
    ↓ (为每个活动中的 applied_sla)
Sla::ProcessAppliedSlaJob
    ↓
Sla::EvaluateAppliedSlaService.perform
```

### 6.3 OSS 与 Enterprise 的调度关系

**OSS 入口** (`app/jobs/trigger_scheduled_items_job.rb`):
```ruby
class TriggerScheduledItemsJob < ApplicationJob
  queue_as :scheduled_jobs

  def perform
    # 触发活动营销任务
    # 重新打开已暂停的对话
    # 自动解决到期对话
    # 同步 WhatsApp 模板
  end
end

TriggerScheduledItemsJob.prepend_mod_with('TriggerScheduledItemsJob')
```

**Enterprise 扩展** (`enterprise/app/jobs/enterprise/trigger_scheduled_items_job.rb`):
```ruby
module Enterprise::TriggerScheduledItemsJob
  def perform
    super  # 先执行 OSS 版本的逻辑

    ## Triggers Enterprise specific jobs
    ####################################

    # Triggers Account Sla jobs
    Sla::TriggerSlasForAccountsJob.perform_later
  end
end
```

**机制说明**:
- 使用 `prepend_mod_with` 实现模块扩展
- Enterprise 版本通过 `prepend` 方式在 OSS 任务基础上追加 SLA 相关任务
- 这种设计保持了 OSS 和 Enterprise 版本的解耦

### 6.4 TriggerSlasForAccountsJob

**文件位置**: `enterprise/app/jobs/sla/trigger_slas_for_accounts_job.rb`

```ruby
class Sla::TriggerSlasForAccountsJob < ApplicationJob
  queue_as :scheduled_jobs

  def perform
    # 找到所有有 SLA 策略的账户（去重）
    Account.joins(:sla_policies).distinct.find_each do |account|
      Rails.logger.info "Enqueuing ProcessAccountAppliedSlasJob for account #{account.id}"
      Sla::ProcessAccountAppliedSlasJob.perform_later(account)
    end
  end
end
```

**队列**: `scheduled_jobs`（优先级较低，见 sidekiq.yml）

### 6.5 ProcessAccountAppliedSlasJob

**文件位置**: `enterprise/app/jobs/sla/process_account_applied_slas_job.rb`

```ruby
class Sla::ProcessAccountAppliedSlasJob < ApplicationJob
  queue_as :medium

  def perform(account)
    # 只处理状态为 active 或 active_with_misses 的记录
    account.applied_slas.where(sla_status: %w[active active_with_misses]).each do |applied_sla|
      Sla::ProcessAppliedSlaJob.perform_later(applied_sla)
    end
  end
end
```

**队列**: `medium`（优先级高于 scheduled_jobs）

**过滤条件**:
- 只处理 `active` 和 `active_with_misses` 状态
- `hit` 和 `missed` 是终止状态，不再参与评估

### 6.6 ProcessAppliedSlaJob

**文件位置**: `enterprise/app/jobs/sla/process_applied_sla_job.rb`

```ruby
class Sla::ProcessAppliedSlaJob < ApplicationJob
  queue_as :medium

  def perform(applied_sla)
    Sla::EvaluateAppliedSlaService.new(applied_sla: applied_sla).perform
  end
end
```

**队列**: `medium`

### 6.7 Sidekiq 队列优先级

**配置文件**: `config/sidekiq.yml` (第 17-33 行)

```yaml
:queues:
  - critical
  - high
  - medium              # ← SLA 处理任务
  - default
  - mailers
  - action_mailbox_routing
  - low
  - scheduled_jobs      # ← SLA 触发任务
  - deferred
  - purgable
  - housekeeping
  - async_database_migration
  - bulk_reindex_low
  - active_storage_analysis
  - active_storage_purge
  - action_mailbox_incineration
```

**调度时序示例**:

```
时间线 (每 5 分钟一个周期):

T=00:00: Cron 触发 TriggerScheduledItemsJob
  ↓
T=00:00: TriggerSlasForAccountsJob 入队 (scheduled_jobs)
  ↓ (假设 Sidekiq 空闲)
T=00:00: 扫描 Account.joins(:sla_policies)
  ↓
T=00:00: 为账户 A 入队 ProcessAccountAppliedSlasJob (medium)
T=00:00: 为账户 B 入队 ProcessAccountAppliedSlasJob (medium)
  ↓
T=00:01: 账户 A: 遍历 active/active_with_misses 的 applied_sla
  ↓
T=00:01: 为每个 applied_sla 入队 ProcessAppliedSlaJob (medium)
  ↓
T=00:02: 执行 EvaluateAppliedSlaService
  - 检查 FRT/NRT/RT 阈值
  - 如错过，创建 SlaEvent → 触发通知
  - 更新 sla_status

T=05:00: 下一个周期开始...
```

**延迟影响**:
- 最坏情况下，SLA 错过可能在**最多 5 分钟 + 任务执行时间**后才被检测到
- 这解释了为什么前端组件需要自己做 60 秒的定时刷新（用于倒计时显示）
- 告警通知的延迟取决于 Sidekiq 的队列积压情况

---

## 7. 数据反馈到前端

### 7.1 状态变更事件推送

**文件位置**: `enterprise/app/models/applied_sla.rb` (第 64-71 行)

```ruby
after_update_commit :push_conversation_event

def push_conversation_event
  # 目前使用 CONVERSATION_UPDATED 事件通知前端
  # 未来可使用专门的 CONVERSATION_SLA_UPDATED 事件
  
  return unless saved_change_to_sla_status?
  
  conversation.dispatch_conversation_updated_event
end
```

**机制**:
- 当 `sla_status` 字段变更时触发
- 通过 `CONVERSATION_UPDATED` 事件通过 WebSocket 推送到前端
- 前端收到事件后刷新对话数据

### 7.2 对话数据序列化 (API 响应)

**文件位置**: `enterprise/app/views/enterprise/api/v1/conversations/partials/_conversation.json.jbuilder`

```ruby
json.applied_sla do
  json.partial! 'api/v1/models/applied_sla', resource: conversation.applied_sla if conversation.applied_sla.present?
end

json.sla_events do
  json.array! conversation.sla_events do |sla_event|
    json.partial! 'api/v1/models/sla_event', sla_event: sla_event
  end
end
```

### 7.3 AppliedSla 序列化格式

**文件位置**: `enterprise/app/views/api/v1/models/_applied_sla.json.jbuilder`

```ruby
json.id resource.id
json.sla_id resource.sla_policy_id
json.sla_status resource.sla_status              # "active", "hit", "missed", "active_with_misses"
json.created_at resource.created_at.to_i
json.updated_at resource.updated_at.to_i
json.sla_description resource.sla_policy.description
json.sla_name resource.sla_policy.name
json.sla_first_response_time_threshold resource.sla_policy.first_response_time_threshold
json.sla_next_response_time_threshold resource.sla_policy.next_response_time_threshold
json.sla_only_during_business_hours resource.sla_policy.only_during_business_hours
json.sla_resolution_time_threshold resource.sla_policy.resolution_time_threshold
```

### 7.4 SlaEvent 序列化格式

**文件位置**: `enterprise/app/views/api/v1/models/_sla_event.json.jbuilder`

```ruby
json.id sla_event.id
json.event_type sla_event.event_type    # "frt", "nrt", "rt"
json.meta sla_event.meta                 # 额外元数据
json.updated_at sla_event.updated_at.to_i
json.created_at sla_event.created_at.to_i
```

### 7.5 WebSocket 推送数据

**文件位置**: `enterprise/app/presenters/enterprise/conversations/event_data_presenter.rb`

```ruby
def push_data
  if account.feature_enabled?('sla')
    super.merge(
      applied_sla: applied_sla&.push_event_data,
      sla_events: sla_events.map(&:push_event_data),
      sla_policy_id: sla_policy_id
    )
  else
    super
  end
end
```

**特性开关**:
- 只有当账户启用了 `'sla'` 功能时才推送 SLA 数据
- 未启用时，`applied_sla` 和 `sla_events` 字段不会出现在推送数据中

### 7.6 推送事件数据格式

**AppliedSla#push_event_data** (applied_sla.rb:46-60):
```ruby
{
  id: id,
  sla_id: sla_policy_id,
  sla_status: sla_status,
  created_at: created_at.to_i,
  updated_at: updated_at.to_i,
  sla_description: sla_policy.description,
  sla_name: sla_policy.name,
  sla_first_response_time_threshold: sla_policy.first_response_time_threshold,
  sla_next_response_time_threshold: sla_policy.next_response_time_threshold,
  sla_only_during_business_hours: sla_policy.only_during_business_hours,
  sla_resolution_time_threshold: sla_policy.resolution_time_threshold
}
```

**SlaEvent#push_event_data** (sla_event.rb:36-44):
```ruby
{
  id: id,
  event_type: event_type,
  meta: meta,
  created_at: created_at.to_i,
  updated_at: updated_at.to_i
}
```

---

## 8. 管理端 SLA 报表数据链路

### 8.1 报表功能概览

管理端 SLA 报表提供三条数据链路：
1. **指标数据** (Metrics): 汇总统计（总数、错过数、命中率）
2. **列表数据** (List): 分页展示 SLA 错过的对话详情
3. **导出数据** (Download): CSV 格式导出错过记录

### 8.2 路由配置

**文件位置**: `config/routes.rb` (第 222-226 行)

```ruby
resources :applied_slas, only: [:index] do
  collection do
    get :metrics
    get :download
  end
end
```

**生成的路由**:
- `GET /api/v1/accounts/:id/applied_slas` → 列表
- `GET /api/v1/accounts/:id/applied_slas/metrics` → 指标
- `GET /api/v1/accounts/:id/applied_slas/download` → CSV 导出

### 8.3 后端 API 控制器

**文件位置**: `enterprise/app/controllers/api/v1/accounts/applied_slas_controller.rb`

```ruby
class Api::V1::Accounts::AppliedSlasController < Api::V1::Accounts::EnterpriseAccountsController
  include Sift
  include DateRangeHelper

  RESULTS_PER_PAGE = 25

  before_action :set_applied_slas, only: [:index, :metrics, :download]
  before_action :set_current_page, only: [:index]
  before_action :check_admin_authorization?  # 仅管理员可访问

  sort_on :created_at, type: :datetime

  # 三个主要端点
  def index; end
  def metrics; end
  def download; end
end
```

**权限控制**:
- 继承自 `EnterpriseAccountsController`
- `check_admin_authorization?` 确保只有管理员可以访问

### 8.4 共享的过滤逻辑

```ruby
private

def set_applied_slas
  initial_query = Current.account.applied_slas.includes(:conversation)
  @applied_slas = apply_filters(initial_query)
end

def apply_filters(query)
  query.filter_by_date_range(range)           # 日期范围 (since/until)
       .filter_by_inbox_id(params[:inbox_id])
       .filter_by_team_id(params[:team_id])
       .filter_by_sla_policy_id(params[:sla_policy_id])
       .filter_by_label_list(params[:label_list])
       .filter_by_assigned_agent_id(params[:assigned_agent_id])
end

def missed_applied_slas
  @missed_applied_slas ||= @applied_slas.missed
  # scope :missed, -> { where(sla_status: %i[missed active_with_misses]) }
end
```

**注意**:
- 列表和导出只显示 `missed` 状态的记录（`missed` + `active_with_misses`）
- 指标计算则基于所有 `applied_slas`

### 8.5 指标数据链路

#### 8.5.1 后端计算

```ruby
def metrics
  @total_applied_slas = total_applied_slas      # 所有应用 SLA 的对话数
  @number_of_sla_misses = number_of_sla_misses  # 有错过的对话数
  @hit_rate = hit_rate                           # 命中率百分比
end

private

def total_applied_slas
  @total_applied_slas ||= @applied_slas.count
end

def number_of_sla_misses
  @number_of_sla_misses ||= missed_applied_slas.count
end

def hit_rate
  number_of_sla_misses.zero? ? '100%' : "#{hit_rate_percentage}%"
end

def hit_rate_percentage
  ((total_applied_slas - number_of_sla_misses) / total_applied_slas.to_f * 100).round(2)
end
```

**计算公式**:
```
Hit Rate = (Total - Misses) / Total × 100%

其中:
- Total = 符合过滤条件的所有 applied_slas 数量
- Misses = sla_status 为 missed 或 active_with_misses 的数量
```

#### 8.5.2 序列化

**文件位置**: `enterprise/app/views/api/v1/accounts/applied_slas/metrics.json.jbuilder`

```ruby
json.total_applied_slas @total_applied_slas
json.number_of_sla_misses @number_of_sla_misses
json.hit_rate @hit_rate
```

**响应示例**:
```json
{
  "total_applied_slas": 100,
  "number_of_sla_misses": 15,
  "hit_rate": "85.0%"
}
```

### 8.6 列表数据链路

#### 8.6.1 后端分页查询

```ruby
def index
  @count = number_of_sla_misses  # 总错过数（用于分页）
  @applied_slas = @missed_applied_slas.page(@current_page).per(RESULTS_PER_PAGE)
end
```

#### 8.6.2 序列化

**文件位置**: `enterprise/app/views/api/v1/accounts/applied_slas/index.json.jbuilder`

```ruby
json.payload do
  json.array! @applied_slas do |applied_sla|
    json.applied_sla applied_sla.push_event_data
    
    json.conversation do
      conversation = applied_sla.conversation
      json.id conversation.display_id
      json.contact do
        json.name conversation.contact.name if conversation.contact
      end
      json.labels conversation.cached_label_list
      json.assignee conversation.assignee.push_event_data if conversation.assignee
    end
    
    json.sla_events applied_sla.sla_events do |sla_event|
      json.partial! 'api/v1/models/sla_event', formats: [:json], sla_event: sla_event
    end
  end
end

json.meta do
  json.count @count
  json.current_page @current_page
end
```

**响应结构**:
```json
{
  "payload": [
    {
      "applied_sla": { ... },
      "conversation": {
        "id": 123,
        "contact": { "name": "Customer" },
        "labels": "support,urgent",
        "assignee": { ... }
      },
      "sla_events": [
        { "id": 1, "event_type": "frt", "created_at": 1234567890 }
      ]
    }
  ],
  "meta": {
    "count": 15,
    "current_page": 1
  }
}
```

### 8.7 导出数据链路

#### 8.7.1 后端 CSV 生成

```ruby
def download
  @missed_applied_slas = missed_applied_slas
  response.headers['Content-Type'] = 'text/csv'
  response.headers['Content-Disposition'] = 'attachment; filename=breached_conversation.csv'
  render layout: false, formats: [:csv]
end
```

#### 8.7.2 CSV 模板

**文件位置**: `enterprise/app/views/api/v1/accounts/applied_slas/download.csv.erb`

```erb
<% headers = [
  I18n.t('reports.sla_csv.conversation_id'),
  I18n.t('reports.sla_csv.sla_policy_breached'),
  I18n.t('reports.sla_csv.assignee'),
  I18n.t('reports.sla_csv.team'),
  I18n.t('reports.sla_csv.inbox'),
  I18n.t('reports.sla_csv.labels'),
  I18n.t('reports.sla_csv.conversation_link'),
  I18n.t('reports.sla_csv.breached_events')
] %>
<%= CSV.generate_line headers %>

<% @missed_applied_slas.each do |sla| %>
  <% missed_events = sla.sla_events.map(&:event_type).join(', ') %>
  <% conversation = sla.conversation %>
  <%= CSV.generate_line([
    conversation.display_id,
    sla.sla_policy.name,
    conversation.assignee&.name,
    conversation.team&.name,
    conversation.inbox&.name,
    conversation.cached_label_list,
    app_account_conversation_url(account_id: conversation.account_id, id: conversation.display_id),
    missed_events
  ]) %>
<% end %>
```

**CSV 列映射**:

| 列名 | 数据源 | 说明 |
|-----|-------|------|
| Conversation ID | `conversation.display_id` | 对话显示 ID |
| SLA Policy Breached | `sla.sla_policy.name` | 违反的 SLA 策略名称 |
| Assignee | `conversation.assignee&.name` | 负责人 |
| Team | `conversation.team&.name` | 所属团队 |
| Inbox | `conversation.inbox&.name` | 所属收件箱 |
| Labels | `conversation.cached_label_list` | 标签列表 |
| Conversation Link | `app_account_conversation_url(...)` | 直接链接 |
| Breached Events | `sla_events.map(&:event_type)` | 错过的事件类型 (frt,nrt,rt) |

### 8.8 前端 API 层

**文件位置**: `app/javascript/dashboard/api/slaReports.js`

```javascript
class SLAReportsAPI extends ApiClient {
  constructor() {
    super('applied_slas', { accountScoped: true });
  }

  // 获取列表（分页 + 过滤）
  get({ from, to, assigned_agent_id, inbox_id, team_id, sla_policy_id, label_list, page } = {}) {
    return axios.get(this.url, {
      params: { since: from, until: to, assigned_agent_id, inbox_id, team_id, sla_policy_id, label_list, page }
    });
  }

  // 导出 CSV
  download({ from, to, assigned_agent_id, inbox_id, team_id, sla_policy_id, label_list } = {}) {
    return axios.get(`${this.url}/download`, {
      params: { since: from, until: to, assigned_agent_id, inbox_id, team_id, label_list, sla_policy_id }
    });
  }

  // 获取指标
  getMetrics({ from, to, assigned_agent_id, inbox_id, team_id, label_list, sla_policy_id } = {}) {
    return axios.get(`${this.url}/metrics`, {
      params: { since: from, until: to, assigned_agent_id, inbox_id, label_list, team_id, sla_policy_id }
    });
  }
}
```

### 8.9 前端 Vuex Store

**文件位置**: `app/javascript/dashboard/store/modules/SLAReports.js`

```javascript
export const state = {
  records: [],           // 列表数据
  metrics: {
    numberOfConversations: 0,   // total_applied_slas
    numberOfSLAMisses: 0,       // number_of_sla_misses
    hitRate: '0%'               // hit_rate
  },
  uiFlags: {
    isFetching: false,
    isFetchingMetrics: false,
  },
  meta: {
    count: 0,
    currentPage: 1,
  },
};

// Actions: get, getMetrics, download
// Mutations: SET_SLA_REPORTS, SET_SLA_REPORTS_METRICS, SET_SLA_REPORTS_META
```

**字段映射**:
```javascript
// 后端响应 → 前端 state
{
  total_applied_slas     → metrics.numberOfConversations
  number_of_sla_misses   → metrics.numberOfSLAMisses
  hit_rate               → metrics.hitRate
}
```

### 8.10 前端报表页面

**文件位置**: `app/javascript/dashboard/routes/dashboard/settings/reports/SLAReports.vue`

```javascript
mounted() {
  // 预加载筛选所需数据
  this.$store.dispatch('agents/get');
  this.$store.dispatch('inboxes/get');
  this.$store.dispatch('teams/get');
  this.$store.dispatch('labels/get');
  this.$store.dispatch('sla/get');
  
  // 加载报表数据
  this.fetchSLAMetrics();
  this.fetchSLAReports();
}

methods: {
  fetchSLAReports({ pageNumber } = {}) {
    this.$store.dispatch('slaReports/get', {
      page: pageNumber || this.pageNumber,
      ...this.activeFilter,
    });
  },
  
  fetchSLAMetrics() {
    this.$store.dispatch('slaReports/getMetrics', this.activeFilter);
  },
  
  onFilterChange(params) {
    this.activeFilter = params;
    this.fetchSLAReports();
    this.fetchSLAMetrics();
  },
  
  downloadReports() {
    this.$store.dispatch('slaReports/download', {
      fileName: generateFileName({ type, to }),
      ...this.activeFilter,
    });
  },
}
```

### 8.11 前端组件结构

```
SLAReports.vue (页面容器)
├── ReportHeader.vue (标题 + 导出按钮)
├── SLAReportFilters.vue (筛选条件)
│   ├── 日期范围选择
│   ├── 收件箱筛选
│   ├── 团队筛选
│   ├── SLA 策略筛选
│   ├── 标签筛选
│   └── 负责人筛选
├── SLAMetrics.vue (三个指标卡片)
│   └── SLAMetricCard.vue × 3
│       ├── Hit Rate (%)
│       ├── Number of Breaches
│       └── Number of Conversations
└── SLATable.vue (错过列表)
    ├── TableHeaderCell.vue (表头)
    ├── SLAReportItem.vue × N (每行记录)
    │   ├── 对话信息
    │   ├── SLA 策略名
    │   ├── 负责人
    │   └── 详情按钮
    └── TableFooter.vue (分页)
```

### 8.12 报表数据流完整链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         用户操作触发                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  进入报表页面 / 更改筛选条件 / 切换页码 / 点击导出                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           前端 Vuex Store                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  slaReports/get        → SLAReportsAPI.get()                           │
│  slaReports/getMetrics → SLAReportsAPI.getMetrics()                    │
│  slaReports/download   → SLAReportsAPI.download()                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ HTTP
┌─────────────────────────────────────────────────────────────────────────┐
│                           后端 API 控制器                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  GET /applied_slas           → index action                            │
│  GET /applied_slas/metrics   → metrics action                          │
│  GET /applied_slas/download  → download action                         │
│                                                                         │
│  共同前置: set_applied_slas → apply_filters                             │
│    filter_by_date_range                                                 │
│    filter_by_inbox_id                                                   │
│    filter_by_team_id                                                    │
│    filter_by_sla_policy_id                                              │
│    filter_by_label_list                                                 │
│    filter_by_assigned_agent_id                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              数据查询层                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  metrics:                                                               │
│    total = @applied_slas.count                                          │
│    misses = @applied_slas.missed.count                                  │
│    hit_rate = ((total - misses) / total * 100).round(2)                 │
│                                                                         │
│  list:                                                                  │
│    @missed_applied_slas.page(@current_page).per(25)                     │
│    includes(:conversation) → N+1 优化                                    │
│                                                                         │
│  download:                                                              │
│    @missed_applied_slas (全量，无分页)                                   │
│    CSV.generate_line 逐行渲染                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              数据返回层                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  metrics:  JSON → { total_applied_slas, number_of_sla_misses, hit_rate }│
│  list:     JSON → { payload: [...], meta: { count, current_page } }    │
│  download: CSV  → Content-Type: text/csv, attachment                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.13 报表统计与 RT 边界条件的关联

**报表统计口径**:

从 `applied_slas_controller.rb` 的 `missed_applied_slas` 实现：

```ruby
scope :missed, -> { where(sla_status: %i[missed active_with_misses]) }
```

**这意味着**:

| 场景 | 报表中的状态 | 是否计入 misses | RT 事件记录 |
|-----|------------|-----------------|------------|
| 对话未解决 + RT 超时被后台检测到 | `active_with_misses` | ✅ 计入 | ✅ 有 |
| 对话解决 + FRT/NRT 有超时 | `missed` | ✅ 计入 | 取决于是否曾被检测 |
| 对话解决 + 仅 RT 超时 + 未被后台检测到 | `hit` (异常) | ❌ 不计入 | ❌ 无 |
| 对话解决 + 所有阈值都合规 | `hit` | ❌ 不计入 | ❌ 无 |

**报表数据一致性问题**:

由于 RT 边界条件的存在，报表可能出现以下不一致：

1. **Hit Rate 虚高**: 某些 RT 超时的对话被计入 `hit`，导致命中率高于实际
2. **CSV 导出不完整**: 上述对话不会出现在 `breached_conversation.csv` 中
3. **指标与事件表不一致**: `number_of_sla_misses` 可能小于 `sla_events` 中 RT 类型的预期数量

---

## 9. 前端状态计算与显示

### 9.1 SLA 状态评估工具

前端使用 `@chatwoot/utils` 包中的 `evaluateSLAStatus` 函数来计算 SLA 状态。

**引用位置**:
- `app/javascript/dashboard/components/widgets/conversation/components/SLACardLabel.vue:4`
- `app/javascript/dashboard/components-next/Conversation/Sla/SLACardLabel.vue:3`
- `app/javascript/dashboard/components-next/Conversation/ConversationCard/SLACardLabel.vue:3`

### 9.2 前端组件 - SLACardLabel

**文件位置**: `app/javascript/dashboard/components-next/Conversation/Sla/SLACardLabel.vue`

```vue
<script setup>
const REFRESH_INTERVAL = 60000;  // 每分钟刷新一次

const slaStatus = ref({
  threshold: null,      // 剩余时间显示（如 "5m", "1h"）
  isSlaMissed: false,   // 是否已错过
  type: null,           // "frt", "nrt", "rt"
  icon: null,           // 图标名称
});

const updateSlaStatus = () => {
  slaStatus.value = evaluateSLAStatus({
    appliedSla: appliedSLA.value || {},
    chat: props.chat,
  });
};

// 挂载时开始定时刷新
onMounted(() => {
  updateSlaStatus();
  createTimer();  // 每 60 秒重新计算
});

// 对话数据变化时重新计算
watch(() => props.chat, updateSlaStatus);
</script>

<template>
  <Label
    :label="slaStatus.threshold"
    :color="isSlaMissed ? 'ruby' : 'amber'"
  >
    <template #icon>
      <Icon icon="i-lucide-flame" />
    </template>
  </Label>
</template>
```

### 9.3 更详细的组件版本

**文件位置**: `app/javascript/dashboard/components/widgets/conversation/components/SLACardLabel.vue`

这个版本提供了更丰富的 UI：
- 显示 SLA 状态文本（如 "FRT MISSED" 或 "NRT DUE"）
- 悬停时显示 SLA 错过事件的弹层 (`SLAPopoverCard`)
- 根据宽度自适应显示信息

```vue
const slaStatusText = computed(() => {
  const upperCaseType = slaStatus.value?.type?.toUpperCase();
  const statusKey = isSlaMissed.value ? 'MISSED' : 'DUE';
  
  return t(`CONVERSATION.HEADER.SLA_STATUS.${upperCaseType}`, {
    status: t(`CONVERSATION.HEADER.SLA_STATUS.${statusKey}`),
  });
});

const showSlaPopoverCard = computed(
  () => props.showExtendedInfo && slaEvents.value?.length > 0
);
```

### 9.4 前端定时刷新机制

```javascript
const createTimer = () => {
  timer.value = setTimeout(() => {
    updateSlaStatus();  // 重新计算剩余时间
    createTimer();      // 递归设置下一次刷新
  }, REFRESH_INTERVAL);  // 60000ms = 1分钟
};
```

**为什么前端需要定时刷新**:
1. 后端的定时任务有执行间隔（5 分钟）
2. 前端需要实时显示倒计时（如 "5m remaining"）
3. 即使后端状态未变，剩余时间也在不断减少

### 9.5 数据流总结

```
┌─────────────────────────────────────────────────────────────────┐
│                        后端数据流                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  定时任务 (每 5 分钟: TriggerScheduledItemsJob)                  │
│         ↓                                                       │
│  评估服务 (EvaluateAppliedSlaService)                           │
│         ↓                                                       │
│  阈值检查 → 错过 → 创建 SlaEvent → 通知用户                     │
│         ↓                                                       │
│  AppliedSla 状态更新 → CONVERSATION_UPDATED 事件                │
│         ↓                                                       │
│  WebSocket 推送 (含 applied_sla 和 sla_events)                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ WebSocket
┌─────────────────────────────────────────────────────────────────┐
│                        前端数据流                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  收到 CONVERSATION_UPDATED 事件                                 │
│         ↓                                                       │
│  更新 Vuex/Pinia store 中的对话数据                             │
│         ↓                                                       │
│  SLACardLabel 组件 watch 到 props.chat 变化                     │
│         ↓                                                       │
│  调用 evaluateSLAStatus() 计算显示状态                          │
│         ↓                                                       │
│  每 60 秒重新计算剩余时间（倒计时）                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. 关键代码位置汇总

### 10.1 后端核心文件

| 文件路径 | 功能描述 |
|---------|---------|
| `enterprise/app/models/sla_policy.rb` | SLA 策略模型 |
| `enterprise/app/models/applied_sla.rb` | 已应用 SLA 记录模型 |
| `enterprise/app/models/sla_event.rb` | SLA 事件模型 |
| `enterprise/app/models/enterprise/concerns/conversation.rb` | 对话模型的 SLA 扩展 |
| `enterprise/app/services/sla/evaluate_applied_sla_service.rb` | SLA 评估核心服务 |
| `enterprise/app/jobs/sla/trigger_slas_for_accounts_job.rb` | SLA 触发器任务 |
| `enterprise/app/jobs/sla/process_account_applied_slas_job.rb` | 账户级 SLA 处理任务 |
| `enterprise/app/jobs/sla/process_applied_sla_job.rb` | 单个 SLA 处理任务 |
| `enterprise/app/jobs/enterprise/trigger_scheduled_items_job.rb` | Enterprise 定时任务扩展 |
| `app/jobs/trigger_scheduled_items_job.rb` | OSS 定时任务入口 |
| `enterprise/app/presenters/enterprise/conversations/event_data_presenter.rb` | SLA 数据序列化 |
| `app/models/working_hour.rb` | 营业时间模型 |
| `app/builders/notification_builder.rb` | 通知构建器 |
| `config/schedule.yml` | Sidekiq Cron 调度配置 |
| `config/sidekiq.yml` | Sidekiq 队列配置 |

### 10.2 管理端报表文件

| 文件路径 | 功能描述 |
|---------|---------|
| `enterprise/app/controllers/api/v1/accounts/applied_slas_controller.rb` | SLA 报表 API 控制器 |
| `enterprise/app/views/api/v1/accounts/applied_slas/index.json.jbuilder` | 列表数据序列化 |
| `enterprise/app/views/api/v1/accounts/applied_slas/metrics.json.jbuilder` | 指标数据序列化 |
| `enterprise/app/views/api/v1/accounts/applied_slas/download.csv.erb` | CSV 导出模板 |

### 10.3 前端核心文件

| 文件路径 | 功能描述 |
|---------|---------|
| `app/javascript/dashboard/components-next/Conversation/Sla/SLACardLabel.vue` | 新组件库 SLA 标签 |
| `app/javascript/dashboard/components/widgets/conversation/components/SLACardLabel.vue` | 旧组件库 SLA 标签（带弹层） |
| `app/javascript/dashboard/store/modules/sla.js` | SLA 策略 Vuex 模块 |
| `app/javascript/dashboard/api/slaReports.js` | SLA 报表 API 封装 |
| `app/javascript/dashboard/store/modules/SLAReports.js` | SLA 报表 Vuex Store |
| `app/javascript/dashboard/routes/dashboard/settings/reports/SLAReports.vue` | SLA 报表主页面 |
| `app/javascript/dashboard/routes/dashboard/settings/reports/components/SLA/SLAMetrics.vue` | SLA 指标组件 |
| `app/javascript/dashboard/routes/dashboard/settings/reports/components/SLA/SLATable.vue` | SLA 列表表格 |
| `app/javascript/dashboard/routes/dashboard/settings/reports/components/SLA/SLAFilter.vue` | SLA 报表筛选器 |

### 10.4 API 相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `enterprise/app/controllers/api/v1/accounts/sla_policies_controller.rb` | SLA 策略 CRUD API |
| `enterprise/app/views/api/v1/models/_sla_policy.json.jbuilder` | SLA 策略序列化 |
| `enterprise/app/views/api/v1/models/_applied_sla.json.jbuilder` | AppliedSla 序列化 |
| `enterprise/app/views/api/v1/models/_sla_event.json.jbuilder` | SlaEvent 序列化 |
| `enterprise/app/views/enterprise/api/v1/conversations/partials/_conversation.json.jbuilder` | 对话数据中的 SLA 字段 |

---

## 附录：SLA 状态机

```
                              ┌─────────────────┐
                              │     active      │
                              │   (活动中)      │
                              └────────┬────────┘
                                       │
                                       │ 任一阈值错过
                                       │ (frt/nrt/rt)
                                       ▼
                              ┌─────────────────┐
                              │active_with_misses│
                              │ (活动中但有错过) │
                              └────────┬────────┘
                                       │
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
            │  对话解决   │    │  对话解决   │    │  继续错过   │
            │  无新错过  │    │  有新错过  │    │ (保持状态)  │
            └──────┬──────┘    └──────┬──────┘    └─────────────┘
                   │                  │
                   ▼                  ▼
            ┌─────────────┐    ┌─────────────┐
            │     hit     │    │   missed    │
            │   (已达成)  │    │  (已错过)   │
            └─────────────┘    └─────────────┘
```

**说明**:
- `active` 和 `active_with_misses` 是活跃状态，会持续被后台任务评估
- `hit` 和 `missed` 是终止状态，不再参与评估
- 一旦进入 `active_with_misses`，对话解决后必然是 `missed` 状态
- ⚠️ **边界情况**: 如果对话在 `active` 状态下直接解决（即使 RT 实际上已超时），只要后台任务未在解决前检测到，最终状态会是 `hit`
