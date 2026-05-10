# Chatwoot 邮件渠道完整收发链路分析

## 1. 概述

本文档详细分析 Chatwoot 邮件渠道（Email Channel）的完整收发链路，包括：
- **入站链路**：从 IMAP 定时抓取到消息解析归档
- **出站链路**：从坐席回复触发到 SMTP 邮件发送
- **会话串联**：邮件引用头（References/In-Reply-To）如何将往来邮件串联为同一会话

## 2. 核心数据模型

### 2.1 Channel::Email（邮件渠道配置）

**位置**: `app/models/channel/email.rb`

邮件渠道的核心配置表，存储 IMAP 和 SMTP 连接参数：

| 字段 | 说明 |
|------|------|
| `email` | 邮箱地址（唯一） |
| `forward_to_email` | 自动生成的转发地址（用于回复路由） |
| `imap_*` 系列 | IMAP 服务器配置（地址、端口、SSL、认证方式、登录凭证） |
| `smtp_*` 系列 | SMTP 服务器配置 |
| `provider` | 邮箱服务商（google/microsoft/空） |
| `provider_config` | OAuth 提供商配置（JSONB） |
| `imap_enabled` | 是否启用 IMAP 抓取 |
| `smtp_enabled` | 是否启用 SMTP 发送 |

关键方法：
- `ensure_forward_to_email`: 创建时自动生成转发地址，格式为 `#{SecureRandom.hex}@#{account.inbound_email_domain}`

## 3. 入站链路：IMAP 定时抓取 → 消息解析归档

### 3.1 定时调度

**位置**: `config/schedule.yml:24-27`

```yaml
trigger_imap_email_inboxes_job:
  cron: '*/1 * * * *'        # 每分钟执行一次
  class: 'Inboxes::FetchImapEmailInboxesJob'
  queue: scheduled_jobs
```

### 3.2 抓取调度入口

**位置**: `app/jobs/inboxes/fetch_imap_email_inboxes_job.rb`

流程：
1. 查询所有 `channel_type = 'Channel::Email'` 的 Inbox
2. 批量遍历（`find_each`, batch_size=100）
3. 对每个符合条件的渠道，异步执行 `FetchImapEmailsJob`

过滤条件（`should_fetch_emails?`）：
- 账号未被暂停 (`suspended?`)
- IMAP 已启用 (`imap_enabled`)
- 无需重新授权 (`!reauthorization_required?`)
- 云环境下排除默认套餐

### 3.3 单渠道邮件抓取

**位置**: `app/jobs/inboxes/fetch_imap_emails_job.rb`

**关键特性**：继承 `MutexApplicationJob`，使用 Redis 分布式锁防止并发重复处理

处理流程：

```
获取锁 (5分钟)
    ↓
根据 provider 选择抓取服务
    ├── Microsoft → Imap::MicrosoftFetchEmailService
    ├── Google    → Imap::GoogleFetchEmailService
    └── 其他       → Imap::FetchEmailService
    ↓
对每封邮件调用 Imap::ImapMailbox#process
```

### 3.4 IMAP 邮件抓取实现

**位置**: `app/services/imap/base_fetch_email_service.rb`

**核心流程** (`fetch_mail_for_channel`):

1. **获取邮件序列号** (`fetch_available_mail_sequence_numbers`)
   - 使用 IMAP `SEARCH SINCE <date>` 命令
   - 默认抓取过去 1 天的邮件

2. **批量获取 Message-ID** (`fetch_message_ids_with_sequence`)
   - 每批最多 `MAX_MESSAGES_PER_SYNC = 500` 封
   - 使用 `FETCH BODY.PEEK[HEADER]` 仅获取邮件头（避免下载完整内容）
   - 跳过 Chatwoot 自己发出的通知邮件（防止循环）
   - 跳过已存在的邮件（通过 `source_id = message_id` 检查）

3. **获取完整邮件内容** (`process_message_id`)
   - 使用 `FETCH RFC822` 获取完整邮件
   - 通过 `Mail.read_from_string` 解析为 `Mail` 对象

**IMAP 连接管理** (`build_imap_client`):
```ruby
imap = Net::IMAP.new(imap_address, port: imap_port, ssl: imap_enable_ssl)
Imap::Authentication.authenticate!(imap, authentication_type, login, password)
imap.select('INBOX')
```

### 3.5 邮件解析与会话归档

**位置**: `app/mailboxes/imap/imap_mailbox.rb`

**处理流程** (`process` 方法):

```
1. load_account → 获取渠道关联的 Account
2. load_inbox   → 获取渠道关联的 Inbox
3. decorate_mail → MailPresenter 包装，提供便捷方法
4. incoming_email_from_valid_email? → 有效性检查（跳过 bounce、auto_reply 等）
    ↓ 事务包裹
5. find_or_create_contact    → 查找/创建联系人
6. find_or_create_conversation → 查找/创建会话（核心：会话串联逻辑）
7. create_message            → 创建消息记录
8. add_attachments_to_message → 处理附件
```

### 3.6 邮件装饰器（MailPresenter）

**位置**: `app/presenters/mail_presenter.rb`

包装原始 `Mail` 对象，提供：
- `text_content` / `html_content`: 解析正文，提取回复部分（使用 `EmailReplyTrimmer`）
- `serialized_data`: 完整序列化邮件数据（存入 Message 的 `content_attributes.email`）
- `in_reply_to`: 获取回复目标的 Message-ID
- `references`: 获取引用链
- `original_sender`: 解析真实发件人（考虑 X-Original-Sender、Reply-To 等）
- `attachments`: 处理邮件附件，上传到 ActiveStorage

### 3.7 消息创建与附件处理

**位置**: `app/mailboxes/mailbox_helper.rb`

**消息创建** (`create_message`):
```ruby
@conversation.messages.create!(
  message_type: 'incoming',
  content_type: 'incoming_email',
  source_id: processed_mail.message_id,  # 用邮件的 Message-ID 作为 source_id
  content_attributes: {
    email: processed_mail.serialized_data,  # 完整邮件数据
    cc_email: processed_mail.cc,
    bcc_email: processed_mail.bcc
  }
)
```

**附件处理** (`add_attachments_to_message`):
- 区分内联附件（inline images）和普通附件
- 内联图片：上传后替换 `cid:` 引用为真实 URL
- 普通附件：直接关联到 Message

## 4. 会话串联机制：邮件引用头解析

这是邮件渠道最核心的设计：如何将多封往来邮件正确归为同一会话。

### 4.1 会话查找策略链

**位置**: `app/services/mailbox/conversation_finder.rb`

采用**责任链模式**，依次尝试以下策略：

```ruby
DEFAULT_STRATEGIES = [
  Mailbox::ConversationFinderStrategies::ReceiverUuidStrategy,   # 优先级 1
  Mailbox::ConversationFinderStrategies::InReplyToStrategy,      # 优先级 2
  Mailbox::ConversationFinderStrategies::ReferencesStrategy,     # 优先级 3
  Mailbox::ConversationFinderStrategies::NewConversationStrategy # 兜底：创建新会话
].freeze
```

---

### 4.2 策略一：ReceiverUuidStrategy（接收者 UUID）

**位置**: `app/services/mailbox/conversation_finder_strategies/receiver_uuid_strategy.rb`

**原理**：从邮件的收件人地址中提取会话 UUID。

Chatwoot 发出的回复邮件使用特殊的 Reply-To 地址：
```
reply+<conversation-uuid>@<inbound-email-domain>
```

例如：`reply+6bdc3f4d-0bec-4515-a284-5d916fdde489@mailer.chatwoot.com`

**匹配模式**：
```ruby
UUID_PATTERN = /^reply\+([0-9a-f]{8}\b-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-\b[0-9a-f]{12})$/i
```

**流程**：
1. 遍历邮件所有收件人地址（`mail.to` + `X-Forwarded-For`）
2. 提取 `@` 前的 username 部分
3. 尝试匹配 UUID 模式
4. 找到则通过 `Conversation.find_by(uuid: uuid)` 查找

---

### 4.3 策略二：InReplyToStrategy（In-Reply-To 头）

**位置**: `app/services/mailbox/conversation_finder_strategies/in_reply_to_strategy.rb`

**原理**：解析邮件标准 `In-Reply-To` 头，它指向直接回复的那封邮件。

**支持的 Message-ID 格式**：

#### 格式 A：Chatwoot 自定义消息级 ID（出站邮件使用）
```
<conversation/<conversation-uuid>/messages/<message-id>@<domain>
```
例如：`<conversation/6bdc3f4d-0bec-4515-a284-5d916fdde489/messages/12345@mailer.chatwoot.com>`

**匹配模式**：
```ruby
MESSAGE_PATTERN = %r{conversation/([a-zA-Z0-9-]+)/messages/(\d+)@}
```
提取第 1 组作为 conversation uuid。

#### 格式 B：Chatwoot 自定义会话级 ID（Fallback）
```
<account/<account-id>/conversation/<conversation-uuid>@<domain>
```
例如：`<account/1/conversation/6bdc3f4d-0bec-4515-a284-5d916fdde489@mailer.chatwoot.com>`

**匹配模式**：
```ruby
FALLBACK_PATTERN = %r{account/(\d+)/conversation/([a-zA-Z0-9-]+)@}
```
提取第 2 组作为 conversation uuid。

#### 格式 C：外部邮件原始 Message-ID
直接通过 `Message.find_by(source_id: in_reply_to)` 查找。

---

### 4.4 策略三：ReferencesStrategy（References 头）

**位置**: `app/services/mailbox/conversation_finder_strategies/references_strategy.rb`

**原理**：解析邮件 `References` 头，它包含完整的邮件引用链（RFC 5322 标准）。

References 头通常是一个空格或换行分隔的 Message-ID 列表：
```
References: <msg1@domain.com> <msg2@domain.com> <msg3@domain.com>
```

**流程**：
1. 获取 `mail.references` 数组
2. 遍历每个 reference Message-ID
3. 先尝试匹配 Chatwoot 自定义格式（同上）
4. 否则通过 `Message.find_by(source_id: reference, inbox_id: channel.inbox.id)` 查找

**关键限制**：查询时限定 `inbox_id`，防止跨账号/跨邮箱匹配。

---

### 4.5 IMAP 邮箱的简化会话查找

**位置**: `app/mailboxes/imap/imap_mailbox.rb:92-110`

对于 IMAP 渠道，Chatwoot 实现了一个简化版本的会话查找（独立于上述策略链）：

```ruby
def find_or_create_conversation
  @conversation = 
    find_conversation_by_in_reply_to ||   # 先查 In-Reply-To
    find_conversation_by_reference_ids || # 再查 References
    ::Conversation.create!(...)           # 都找不到则新建
end
```

**查找逻辑**：
1. **In-Reply-To 查找**：
   - 通过 `Message.find_by(source_id: in_reply_to)` 找到消息
   - 或通过 `Conversation.find_by("additional_attributes->>'in_reply_to' = ?", in_reply_to)` 查找
   
2. **References 查找**：
   - 遍历 references 数组，通过 `source_id` 匹配消息
   - 或通过 `FALLBACK_CONVERSATION_PATTERN` 从 reference 中提取会话 UUID

---

### 4.6 新会话创建

当所有策略都未找到匹配会话时，创建新会话：

```ruby
Conversation.create!(
  account_id: @account.id,
  inbox_id: @inbox.id,
  contact_id: @contact.id,
  contact_inbox_id: @contact_inbox.id,
  additional_attributes: {
    source: 'email',
    in_reply_to: in_reply_to,           # 保存原始 in_reply_to
    auto_reply: @processed_mail.auto_reply?,
    mail_subject: @processed_mail.subject,
    initiated_at: { timestamp: Time.now.utc }
  }
)
```

## 5. 出站链路：坐席回复 → SMTP 发送

### 5.1 触发入口

**位置**: `app/models/message.rb:324-333, 397-401`

坐席发送消息后，Message 模型的 `after_create_commit` 回调触发：

```ruby
def execute_after_create_commit_callbacks
  reopen_conversation
  mark_pending_conversation_as_open_for_human_response
  set_conversation_activity
  dispatch_create_events
  send_reply              # ← 这里触发出站发送
  execute_message_template_hooks
  update_contact_activity
end

def send_reply
  # 有附件则等待 2 秒，确保 ActiveStorage 上传完成
  attachments.blank? ? 
    ::SendReplyJob.perform_later(id) : 
    ::SendReplyJob.set(wait: 2.seconds).perform_later(id)
end
```

### 5.2 分发到渠道服务

**位置**: `app/jobs/send_reply_job.rb`

根据渠道类型选择对应的发送服务：

```ruby
CHANNEL_SERVICES = {
  'Channel::Email' => ::Email::SendOnEmailService,  # 邮件渠道
  # ... 其他渠道
}
```

### 5.3 邮件发送服务

**位置**: `app/services/email/send_on_email_service.rb`

继承 `Base::SendOnChannelService`，核心方法：

```ruby
def perform_reply
  return unless message.email_notifiable_message?
  
  # 立即发送邮件（deliver_now）
  reply_mail = ConversationReplyMailer
    .with(account: message.account)
    .email_reply(message)
    .deliver_now
  
  # 更新消息的 source_id 为发送邮件的 Message-ID
  message.update(source_id: reply_mail.message_id)
rescue StandardError => e
  Messages::StatusUpdateService.new(message, 'failed', e.message).perform
end
```

**前置检查**（来自 Base 类）：
- 必须是 outgoing 或 template 消息
- 不是私密备注（`private?`）
- 不是从渠道同步过来的消息（`source_id` 为空）

### 5.4 邮件构建（ConversationReplyMailer）

**位置**: `app/mailers/conversation_reply_mailer.rb`
**辅助模块**: `app/mailers/conversation_reply_mailer_helper.rb`

#### 支持的邮件类型

| 方法 | 用途 |
|------|------|
| `email_reply(message)` | 坐席单条回复（邮件渠道） |
| `reply_with_summary` | 汇总回复（含历史消息摘要） |
| `reply_without_summary` | 纯新消息回复 |
| `conversation_transcript` | 会话记录导出 |

#### 邮件头构建（核心：会话串联）

**Message-ID**（自定义格式）:
```ruby
def custom_message_id
  last_message = @message || @messages&.last
  "<conversation/#{@conversation.uuid}/messages/#{last_message&.id}@#{channel_email_domain}>"
end
```
格式：`<conversation/<会话UUID>/messages/<消息ID>@<域名>>`

**In-Reply-To**:
```ruby
def in_reply_to_email
  conversation_reply_email_id ||  # 优先用最后一封入站邮件的 Message-ID
  "<account/#{@account.id}/conversation/#{@conversation.uuid}@#{channel_email_domain}>"
end
```
- 如果能找到之前的入站邮件，使用其原始 Message-ID
- 否则使用会话级 fallback 格式

**References Header**:
```ruby
def references_header
  build_references_header(@conversation, in_reply_to_email)
end
```
委托给 `ReferencesHeaderBuilder` 模块构建。

### 5.5 References Header 构建器

**位置**: `app/mailers/references_header_builder.rb`

**RFC 5322 规范**：
- References 头应包含完整引用链
- 新回复的 References = 原邮件 References + 原邮件 Message-ID
- 超过 998 字符需换行（CRLF + 空格）

**构建流程**:
```ruby
def build_references_header(conversation, in_reply_to_message_id)
  # 1. 获取被回复邮件的 References（从数据库中查找）
  references = get_references_from_replied_message(conversation, in_reply_to_message_id)
  
  # 2. 追加被回复邮件的 Message-ID
  references << in_reply_to_message_id
  
  # 3. 去重 + 按 RFC 5322 格式化换行
  references = references.compact.uniq
  fold_references_header(references)
end
```

**数据来源**：
- 从 Message 的 `content_attributes.email.references` 中提取历史引用链
- 支持多种格式（带尖括号/不带尖括号）

### 5.6 SMTP 发送配置

**位置**: `app/mailers/conversation_reply_mailer_helper.rb:63-80`

根据渠道配置选择发送方式：

#### 方式一：自定义 SMTP（用户配置）
```ruby
smtp_settings = {
  address: @channel.smtp_address,
  port: @channel.smtp_port,
  user_name: @channel.smtp_login,
  password: @channel.smtp_password,
  domain: @channel.smtp_domain,
  tls: @channel.smtp_enable_ssl_tls,
  enable_starttls_auto: @channel.smtp_enable_starttls_auto,
  authentication: @channel.smtp_authentication
}
```

#### 方式二：OAuth SMTP（Google/Microsoft）
```ruby
# Google: smtp.gmail.com:587, authentication: xoauth2
# Microsoft: smtp.office365.com:587, authentication: xoauth2
# password 使用 provider_config['access_token']
```

### 5.7 发件人/回复地址构建

**发件人** (`email_from`):
- 新特性：`Email::FromBuilder` 构建
- 旧逻辑：SMTP/OAuth 渠道使用 `channel.email`，否则使用 `account.support_email`

**回复地址** (`email_reply_to`):
- SMTP 渠道：直接使用 `channel.email`
- 其他：`reply+<conversation-uuid>@<inbound-domain>`（用于入站路由）

**邮件主题** (`mail_subject`):
- 首条消息：原始邮件主题
- 后续消息：`Re: <原始主题>`
- 无主题时：`[#<display_id>] Conversation`

## 6. 完整收发时序图

### 6.1 客户发邮件 → Chatwoot（入站）

```
┌──────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  邮件服务器 │     │ FetchImapEmailInboxes │     │   FetchImapEmailsJob   │
│ (IMAP)    │     │     Job (每分钟)  │     │  (单渠道抓取)          │
└────┬─────┘     └────────┬─────────┘     └──────────┬───────────┘
     │                    │                          │
     │                    │ 1. 查询所有 Email Inbox   │
     │                    │─────────────────────────>│
     │                    │                          │
     │<─────────────────────────────────────────────│ 2. SEARCH SINCE
     │                    │                          │
     │<─────────────────────────────────────────────│ 3. FETCH BODY.PEEK[HEADER]
     │                    │                          │    (获取 Message-ID 列表)
     │                    │                          │
     │<─────────────────────────────────────────────│ 4. 对新邮件 FETCH RFC822
     │                    │                          │    (获取完整内容)
     │                    │                          │
     │                    │                          │ ┌────────────────────┐
     │                    │                          │ │ Imap::ImapMailbox  │
     │                    │                          │ │ .process(mail)     │
     │                    │                          │ └─────────┬──────────┘
     │                    │                          │           │
     │                    │                          │           │ 5. 解析邮件头
     │                    │                          │           │    (In-Reply-To, References)
     │                    │                          │           │
     │                    │                          │           │ 6. 会话查找策略链
     │                    │                          │           │    ReceiverUuid
     │                    │                          │           │    → InReplyTo
     │                    │                          │           │    → References
     │                    │                          │           │    → NewConversation
     │                    │                          │           │
     │                    │                          │           │ 7. 创建/更新 Contact
     │                    │                          │           │
     │                    │                          │           │ 8. 创建 Message
     │                    │                          │           │    source_id = mail.message_id
     │                    │                          │           │    content_attributes.email = { ... }
     │                    │                          │           │
     │                    │                          │           │ 9. 处理附件
```

### 6.2 坐席回复 → 客户邮箱（出站）

```
┌──────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  坐席界面   │     │   Message 模型    │     │    SendReplyJob      │
│ (Vue)     │     │  (after_create)  │     │                      │
└────┬─────┘     └────────┬─────────┘     └──────────┬───────────┘
     │                    │                          │
     │ 1. 发送消息 POST    │                          │
     │───────────────────>│                          │
     │                    │                          │
     │                    │ 2. after_create_commit   │
     │                    │    send_reply()          │
     │                    │─────────────────────────>│
     │                    │                          │
     │                    │                          │ ┌────────────────────┐
     │                    │                          │ │ Email::SendOnEmail │
     │                    │                          │ │   Service          │
     │                    │                          │ └─────────┬──────────┘
     │                    │                          │           │
     │                    │                          │           │ 3. 邮件类型检查
     │                    │                          │           │
     │                    │                          │           │ ┌────────────────────┐
     │                    │                          │           │ │ConversationReply   │
     │                    │                          │           │ │   Mailer           │
     │                    │                          │           │ └─────────┬──────────┘
     │                    │                          │           │           │
     │                    │                          │           │           │ 4. 构建邮件头
     │                    │                          │           │           │
     │                    │                          │           │           │    Message-ID:
     │                    │                          │           │           │    <conversation/<uuid>/
     │                    │                          │           │           │     messages/<id>@domain>
     │                    │                          │           │           │
     │                    │                          │           │           │    In-Reply-To:
     │                    │                          │           │           │    <原邮件 Message-ID>
     │                    │                          │           │           │    或 fallback 格式
     │                    │                          │           │           │
     │                    │                          │           │           │    References:
     │                    │                          │           │           │    历史引用链 + In-Reply-To
     │                    │                          │           │           │
     │                    │                          │           │           │ 5. SMTP 发送
     │                    │                          │           │           │    (用户配置 / OAuth)
     │                    │                          │           │           │
┌────┴────────────────────┴──────────────────────────┴───────────┴───────────┐
│                        邮件服务器 (SMTP)                                    │
│                                                                            │
│ 客户收到邮件：                                                              │
│   From: <channel-email>                                                    │
│   Reply-To: reply+<uuid>@domain  或  channel-email                         │
│   Message-ID: <conversation/...>                                           │
│   In-Reply-To: <原邮件 ID>                                                  │
│   References: <历史引用链>                                                  │
└────────────────────────────────────────────────────────────────────────────┘
```

## 7. 邮件头串联示意图

以下展示一个完整的邮件对话如何通过 References/In-Reply-To 串联：

```
客户首次发邮件 (Message-ID: <customer-1@external.com>)
    ↓
Chatwoot 接收并创建会话 #123
    ↓
坐席回复 (通过 ConversationReplyMailer):
  Message-ID:    <conversation/abc-123/messages/1@chatwoot.com>
  In-Reply-To:  <customer-1@external.com>
  References:    (空，因为是第一封回复)
    ↓
客户回复坐席的邮件:
  Message-ID:    <customer-2@external.com>
  In-Reply-To:  <conversation/abc-123/messages/1@chatwoot.com>
  References:   <customer-1@external.com> <conversation/abc-123/messages/1@chatwoot.com>
    ↓
Chatwoot 接收，通过 InReplyToStrategy 解析:
  In-Reply-To 匹配 MESSAGE_PATTERN → 提取 UUID abc-123 → 找到会话 #123
    ↓
坐席再次回复:
  Message-ID:    <conversation/abc-123/messages/2@chatwoot.com>
  In-Reply-To:  <customer-2@external.com>
  References:   <customer-1@external.com> <conversation/abc-123/messages/1@chatwoot.com> <customer-2@external.com>
```

## 8. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 渠道模型 | `app/models/channel/email.rb` |
| 定时调度配置 | `config/schedule.yml` |
| IMAP 调度 Job | `app/jobs/inboxes/fetch_imap_email_inboxes_job.rb` |
| IMAP 抓取 Job | `app/jobs/inboxes/fetch_imap_emails_job.rb` |
| IMAP 抓取服务基类 | `app/services/imap/base_fetch_email_service.rb` |
| IMAP 抓取服务实现 | `app/services/imap/fetch_email_service.rb` |
| IMAP 邮件处理 | `app/mailboxes/imap/imap_mailbox.rb` |
| 邮件装饰器 | `app/presenters/mail_presenter.rb` |
| 消息创建/附件 | `app/mailboxes/mailbox_helper.rb` |
| 会话查找策略链 | `app/services/mailbox/conversation_finder.rb` |
| 会话查找策略实现 | `app/services/mailbox/conversation_finder_strategies/*.rb` |
| References 头构建 | `app/mailers/references_header_builder.rb` |
| 坐席回复触发 | `app/models/message.rb` (send_reply 回调) |
| 回复分发 Job | `app/jobs/send_reply_job.rb` |
| 邮件发送服务 | `app/services/email/send_on_email_service.rb` |
| 出站发送服务基类 | `app/services/base/send_on_channel_service.rb` |
| 回复邮件构建器 | `app/mailers/conversation_reply_mailer.rb` |
| 回复邮件辅助 | `app/mailers/conversation_reply_mailer_helper.rb` |

## 9. 设计要点总结

### 9.1 入站设计
- **定时轮询**：每分钟检查 IMAP 收件箱
- **增量同步**：通过 SINCE 日期和 Message-ID 去重
- **分布式锁**：防止多实例重复处理
- **多种认证**：支持 PLAIN / LOGIN / XOAUTH2 (Google/Microsoft)

### 9.2 会话串联设计（四层保障）
1. **收件人地址路由**：`reply+<uuid>@domain` 最可靠
2. **In-Reply-To 头**：标准邮件协议，直接关联
3. **References 头**：完整引用链，容错性强
4. **自定义 Message-ID 格式**：嵌入会话 UUID，双向关联

### 9.3 出站设计
- **延迟发送**：附件等待 2 秒确保上传完成
- **动态 SMTP**：每个渠道可独立配置 SMTP / 使用 OAuth
- **状态追踪**：发送成功后回写 `source_id`，失败记录错误信息
- **引用链维持**：References 头完整传递历史上下文

### 9.4 数据持久化
- `Message.source_id` = 邮件 `Message-ID`（用于关联）
- `Message.content_attributes.email` = 完整邮件序列化数据
- `Conversation.additional_attributes.mail_subject` = 原始邮件主题
