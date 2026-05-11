# Chatwoot SLA 服务等级协议分析报告

## 目录
1. [概述](#1-概述)
2. [核心数据模型](#2-核心数据模型)
3. [SLA 计时机制](#3-sla-计时机制)
4. [营业时间识别](#4-营业时间识别)
5. [升级告警触发条件](#5-升级告警触发条件)
6. [后台任务调度](#6-后台任务调度)
7. [数据反馈到前端](#7-数据反馈到前端)
8. [前端状态计算与显示](#8-前端状态计算与显示)
9. [关键代码位置汇总](#9-关键代码位置汇总)

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

### 3.5 错过处理逻辑

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

### 3.6 对话解决时的最终判定

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

### 6.1 调度层次结构

```
TriggerScheduledItemsJob (定时任务入口)
    ↓
Sla::TriggerSlasForAccountsJob
    ↓ (为每个有 SLA 策略的账户)
Sla::ProcessAccountAppliedSlasJob
    ↓ (为每个活动中的 applied_sla)
Sla::ProcessAppliedSlaJob
    ↓
Sla::EvaluateAppliedSlaService.perform
```

### 6.2 TriggerSlasForAccountsJob

**文件位置**: `enterprise/app/jobs/sla/trigger_slas_for_accounts_job.rb`

```ruby
def perform
  # 找到所有有 SLA 策略的账户
  Account.joins(:sla_policies).distinct.find_each do |account|
    Rails.logger.info "Enqueuing ProcessAccountAppliedSlasJob for account #{account.id}"
    Sla::ProcessAccountAppliedSlasJob.perform_later(account)
  end
end
```

### 6.3 ProcessAccountAppliedSlasJob

**文件位置**: `enterprise/app/jobs/sla/process_account_applied_slas_job.rb`

```ruby
def perform(account)
  # 只处理状态为 active 或 active_with_misses 的记录
  account.applied_slas.where(sla_status: %w[active active_with_misses]).each do |applied_sla|
    Sla::ProcessAppliedSlaJob.perform_later(applied_sla)
  end
end
```

### 6.4 ProcessAppliedSlaJob

**文件位置**: `enterprise/app/jobs/sla/process_applied_sla_job.rb`

```ruby
def perform(applied_sla)
  Sla::EvaluateAppliedSlaService.new(applied_sla: applied_sla).perform
end
```

### 6.5 调度频率

从 `TriggerScheduledItemsJob` 的命名和队列设置 (`queue_as :scheduled_jobs`) 来看，这是一个定时调度的任务。具体的调度间隔由 Sidekiq 定时任务配置决定。

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

## 8. 前端状态计算与显示

### 8.1 SLA 状态评估工具

前端使用 `@chatwoot/utils` 包中的 `evaluateSLAStatus` 函数来计算 SLA 状态。

**引用位置**:
- `app/javascript/dashboard/components/widgets/conversation/components/SLACardLabel.vue:4`
- `app/javascript/dashboard/components-next/Conversation/Sla/SLACardLabel.vue:3`
- `app/javascript/dashboard/components-next/Conversation/ConversationCard/SLACardLabel.vue:3`

### 8.2 前端组件 - SLACardLabel

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

### 8.3 更详细的组件版本

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

### 8.4 前端定时刷新机制

```javascript
const createTimer = () => {
  timer.value = setTimeout(() => {
    updateSlaStatus();  // 重新计算剩余时间
    createTimer();      // 递归设置下一次刷新
  }, REFRESH_INTERVAL);  // 60000ms = 1分钟
};
```

**为什么前端需要定时刷新**:
1. 后端的定时任务有执行间隔
2. 前端需要实时显示倒计时（如 "5m remaining"）
3. 即使后端状态未变，剩余时间也在不断减少

### 8.5 数据流总结

```
┌─────────────────────────────────────────────────────────────────┐
│                        后端数据流                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  定时任务 (TriggerScheduledItemsJob)                            │
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

## 9. 关键代码位置汇总

### 9.1 后端核心文件

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
| `enterprise/app/presenters/enterprise/conversations/event_data_presenter.rb` | SLA 数据序列化 |
| `app/models/working_hour.rb` | 营业时间模型 |
| `app/builders/notification_builder.rb` | 通知构建器 |

### 9.2 前端核心文件

| 文件路径 | 功能描述 |
|---------|---------|
| `app/javascript/dashboard/components-next/Conversation/Sla/SLACardLabel.vue` | 新组件库 SLA 标签 |
| `app/javascript/dashboard/components/widgets/conversation/components/SLACardLabel.vue` | 旧组件库 SLA 标签（带弹层） |
| `app/javascript/dashboard/store/modules/sla.js` | SLA 策略 Vuex 模块 |

### 9.3 API 相关文件

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
