# Chatwoot 邮件渠道完整收发链路分析（v2 - 修正版）

## 修正说明

v1 报告的关键误差：
- **错误**：将 IMAP 入站的会话查找写成了「通用四策略链」
- **事实**：IMAP 渠道有**独立的简化实现**，实际顺序是 `In-Reply-To → References → 新建会话`

本报告对此进行修正，并补充：
- 归档落库字段的完整映射及其来源
- 端到端示例，演示引用头如何命中同一会话

---

## 1. 概述

本文档详细分析 Chatwoot 邮件渠道（Email Channel）的完整收发链路，包括：
- **入站链路**：从 IMAP 定时抓取到消息解析归档
- **出站链路**：从坐席回复触发到 SMTP 邮件发送
- **会话串联**：邮件引用头（References/In-Reply-To）如何将往来邮件串联为同一会话

---

## 2. 核心数据模型

### 2.1 Channel::Email（邮件渠道配置）

**位置**: `app/models/channel/email.rb`

邮件渠道的核心配置表，存储 IMAP 和 SMTP 连接参数：

| 字段 | 说明 |
|------|------|
| `email` | 邮箱地址（唯一） |
| `forward_to_email` | 自动生成的转发地址（格式：`#{SecureRandom.hex}@#{account.inbound_email_domain}`） |
| `imap_*` 系列 | IMAP 服务器配置（地址、端口、SSL、认证方式、登录凭证） |
| `smtp_*` 系列 | SMTP 服务器配置 |
| `provider` | 邮箱服务商（google/microsoft/空） |
| `provider_config` | OAuth 提供商配置（JSONB） |
| `imap_enabled` | 是否启用 IMAP 抓取 |
| `smtp_enabled` | 是否启用 SMTP 发送 |

---

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

过滤条件：
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

---

## 4. IMAP 入站邮件解析与会话串联（核心修正）

### 4.1 处理入口

**位置**: `app/mailboxes/imap/imap_mailbox.rb`

**处理流程** (`process` 方法):

```
1. load_account → 获取渠道关联的 Account
2. load_inbox   → 获取渠道关联的 Inbox
3. decorate_mail → MailPresenter 包装，提供便捷方法
4. incoming_email_from_valid_email? → 有效性检查（跳过 bounce、auto_reply 等）
    ↓ 事务包裹
5. find_or_create_contact    → 查找/创建联系人
6. find_or_create_conversation → 查找/创建会话（核心：简化的串联逻辑）
7. create_message            → 创建消息记录
8. add_attachments_to_message → 处理附件
```

### 4.2 IMAP 实际会话查找策略（两步法）

**位置**: `app/mailboxes/imap/imap_mailbox.rb:92-110`

**重要**：IMAP 渠道**不使用** `Mailbox::ConversationFinder` 的四策略链。它有自己独立的简化实现：

```ruby
def find_or_create_conversation
  @conversation = 
    find_conversation_by_in_reply_to ||   # 步骤 1: 先查 In-Reply-To
    find_conversation_by_reference_ids || # 步骤 2: 再查 References
    ::Conversation.create!(...)           # 步骤 3: 都找不到则新建
end
```

---

#### 4.2.1 策略一：In-Reply-To 查找

**位置**: `app/mailboxes/imap/imap_mailbox.rb:42-51`

```ruby
def find_conversation_by_in_reply_to
  return if in_reply_to.blank?

  # 方式 A: 通过 Message 的 source_id 查找
  message = @inbox.messages.find_by(source_id: in_reply_to)
  
  if message.nil?
    # 方式 B: 通过 Conversation 的 additional_attributes.in_reply_to 查找
    @inbox.conversations.find_by("additional_attributes->>'in_reply_to' = ?", in_reply_to)
  else
    @inbox.conversations.find(message.conversation_id)
  end
end
```

**查找逻辑**：
1. 用 `In-Reply-To` 的值（原始 Message-ID）去 `messages.source_id` 精确匹配
2. 若未找到消息，尝试直接匹配 `conversations.additional_attributes.in_reply_to`
3. **scoped to inbox**：所有查询都限定在当前 Inbox 范围内

---

#### 4.2.2 策略二：References 查找

**位置**: `app/mailboxes/imap/imap_mailbox.rb:53-90`

```ruby
def find_conversation_by_reference_ids
  return if @inbound_mail.references.blank?

  # 子步骤 A: 通过 Message.source_id 在引用链中查找
  message = find_message_by_references
  if message.present?
    conversation = @inbox.conversations.find_by(id: message.conversation_id)
    return conversation if conversation.present?
  end

  # 子步骤 B: FALLBACK - 从引用中提取会话 UUID（坐席先发消息的场景）
  conversation_id = find_conversation_by_references
  @inbox.conversations.find_by(uuid: conversation_id) if conversation_id.present?
end

def find_message_by_references
  message_to_return = nil
  references = Array.wrap(@inbound_mail.references)
  
  # 遍历整个引用链，找到任意一条匹配的 Message
  references.each do |message_id|
    message = @inbox.messages.find_by(source_id: message_id)
    message_to_return = message if message.present?
  end
  message_to_return
end

def find_conversation_by_references
  references = Array.wrap(@inbound_mail.references)
  references.each do |message_id|
    # 匹配坐席先发邮件时使用的 fallback 格式
    # 例如: <account/1/conversation/abc123-def456@domain.com>
    match = FALLBACK_CONVERSATION_PATTERN.match(message_id)
    return match[2] if match.present?  # 返回 conversation uuid
  end
end
```

**Fallback 正则模式**：
```ruby
FALLBACK_CONVERSATION_PATTERN = %r{account/(\d+)/conversation/([a-zA-Z0-9-]+)@}
```

**References 策略的双路径**：
| 路径 | 场景 | 匹配方式 |
|------|------|----------|
| A | 引用链中有已归档的消息 | `Message.find_by(source_id: ref)` |
| B | 坐席先发邮件（无入站消息） | 正则提取 conversation uuid |

---

#### 4.2.3 策略三：新建会话

**位置**: `app/mailboxes/imap/imap_mailbox.rb:93-109`

当 In-Reply-To 和 References 都未命中时，创建新会话：

```ruby
::Conversation.create!(
  {
    account_id: @account.id,
    inbox_id: @inbox.id,
    contact_id: @contact.id,
    contact_inbox_id: @contact_inbox.id,
    additional_attributes: {
      source: 'email',
      in_reply_to: in_reply_to,           # ← 保存原始 in_reply_to 供后续匹配
      auto_reply: @processed_mail.auto_reply?,
      mail_subject: @processed_mail.subject,
      initiated_at: { timestamp: Time.now.utc }
    }
  }
)
```

**关键保存**：`in_reply_to` 被存入 `additional_attributes`，供后续通过策略一的「方式 B」命中。

---

### 4.3 IMAP vs 通用四策略链对比

| 维度 | IMAP 简化版 | 通用 ConversationFinder |
|------|-------------|------------------------|
| 位置 | `Imap::ImapMailbox` | `Mailbox::ConversationFinder` |
| 策略 1 | In-Reply-To | ReceiverUuidStrategy |
| 策略 2 | References | InReplyToStrategy |
| 策略 3 | — | ReferencesStrategy |
| 兜底 | Conversation.create | NewConversationStrategy |
| 使用场景 | IMAP 渠道入站 | ActionMailbox 路由 + 其他邮件入口 |
| 是否用 UUID 收件人 | ❌ 不支持 | ✅ 支持 |

**设计原因**：IMAP 渠道的入站邮件不经过 ActionMailbox 路由，收件人地址就是渠道邮箱本身（而非 `reply+<uuid>@...`），因此不需要 ReceiverUuidStrategy。

---

## 5. 归档落库字段及其来源

### 5.1 邮件装饰器（数据源）

**位置**: `app/presenters/mail_presenter.rb`

所有落库数据最终来自 `MailPresenter.serialized_data`：

```ruby
def serialized_data
  {
    bcc: bcc,                           # 邮件 Bcc 头
    cc: cc,                             # 邮件 Cc 头
    content_type: content_type,         # 邮件 Content-Type
    date: date,                         # 邮件 Date 头
    from: from,                         # 发件人地址（Reply-To 优先）
    headers: headers_data,              # X-Original-From, X-Original-Sender, X-Forwarded-For
    html_content: html_content,         # HTML 正文（full/reply/quoted）
    in_reply_to: in_reply_to,           # In-Reply-To 头（取数组第一个）
    message_id: message_id,             # Message-ID 头
    multipart: multipart?,              # 是否多部分邮件
    number_of_attachments: number_of_attachments,
    references: references,             # References 头（数组）
    subject: subject,                   # 邮件主题
    text_content: text_content,         # 纯文本正文（full/reply/quoted）
    to: to,                             # 收件人地址
    auto_reply: auto_reply?             # 是否自动回复邮件
  }
end
```

---

### 5.2 Contact 表

**位置**: `app/mailboxes/mailbox_helper.rb:101-114`

| 字段 | 来源 | 值示例 |
|------|------|--------|
| `name` | `MailPresenter.sender_name` 或邮箱前缀 | `"John Doe"` 或 `"john.doe"` |
| `email` | `MailPresenter.original_sender` | `"john.doe@example.com"` |
| `additional_attributes.source_id` | `"email:#{message_id}"` | `"email:<abc123@example.com>"` |

---

### 5.3 Conversation 表

**位置**: `app/mailboxes/imap/imap_mailbox.rb:93-109`

| 字段 | 值来源 | 说明 |
|------|--------|------|
| `account_id` | `@channel.account.id` | 关联账户 |
| `inbox_id` | `@channel.inbox.id` | 关联邮箱 |
| `contact_id` | Contact 记录 id | 关联联系人 |
| `contact_inbox_id` | ContactInbox 记录 id | 关联联系人和邮箱 |
| `uuid` | 自动生成 (`gen_random_uuid()`) | 会话唯一标识 |
| `additional_attributes.source` | 固定值 `'email'` | 渠道来源标记 |
| `additional_attributes.in_reply_to` | `MailPresenter.in_reply_to` | 原始 In-Reply-To 值 |
| `additional_attributes.auto_reply` | `MailPresenter.auto_reply?` | 是否为自动回复 |
| `additional_attributes.mail_subject` | `MailPresenter.subject` | 原始邮件主题 |
| `additional_attributes.initiated_at.timestamp` | `Time.now.utc` | 会话创建时间戳 |

---

### 5.4 Message 表

**位置**: `app/mailboxes/mailbox_helper.rb:6-24`

**表结构** (`db/migrate/20230426130150_init_schema.rb:592-617`):

| 字段 | 类型 | 值来源 | 说明 |
|------|------|--------|------|
| `id` | serial | 自动 | 主键 |
| `account_id` | integer | `@conversation.account_id` | 关联账户 |
| `inbox_id` | integer | `@conversation.inbox_id` | 关联邮箱 |
| `conversation_id` | integer | `@conversation.id` | 关联会话 |
| `message_type` | integer | 固定 `:incoming` (0) | 消息方向 |
| `content_type` | integer | 固定 `:incoming_email` (8) | 消息类型 |
| `sender_type` | string | 固定 `'Contact'` | 发送者类型 |
| `sender_id` | bigint | `@contact.id` | 发送者 ID |
| `source_id` | string | `MailPresenter.message_id` | **核心关联字段** |
| `content` | text | `MailPresenter.text_content[:reply]` 或 `html_content[:reply]`（截断 150k） | 消息正文 |
| `content_attributes` | json | 见下表 | 邮件完整元数据 |
| `private` | boolean | 默认 `false` | 是否私密 |
| `status` | integer | 默认 `:sent` (0) | 发送状态 |
| `created_at` | datetime | 自动 | 创建时间 |
| `updated_at` | datetime | 自动 | 更新时间 |

---

**content_attributes.email 完整结构**（来自 `MailPresenter.serialized_data`）:

| JSON 路径 | 来源 | 说明 |
|-----------|------|------|
| `email.bcc` | `MailPresenter.bcc` | Bcc 收件人 |
| `email.cc` | `MailPresenter.cc` | Cc 收件人 |
| `email.content_type` | `MailPresenter.content_type` | 邮件内容类型 |
| `email.date` | `MailPresenter.date` | 邮件日期 |
| `email.from` | `MailPresenter.from` | 发件人（数组） |
| `email.headers` | `MailPresenter.headers_data` | 扩展头（X-Original-*） |
| `email.html_content.full` | `MailPresenter.html_content[:full]` | 完整 HTML 正文 |
| `email.html_content.reply` | `MailPresenter.html_content[:reply]` | 回复部分 HTML |
| `email.html_content.quoted` | `MailPresenter.html_content[:quoted]` | 引用部分 HTML |
| `email.text_content.full` | `MailPresenter.text_content[:full]` | 完整纯文本 |
| `email.text_content.reply` | `MailPresenter.text_content[:reply]` | 回复部分纯文本 |
| `email.text_content.quoted` | `MailPresenter.text_content[:quoted]` | 引用部分纯文本 |
| `email.in_reply_to` | `MailPresenter.in_reply_to` | In-Reply-To 头 |
| `email.message_id` | `MailPresenter.message_id` | Message-ID |
| `email.multipart` | `MailPresenter.multipart?` | 是否多部分 |
| `email.number_of_attachments` | `MailPresenter.number_of_attachments` | 附件数量 |
| `email.references` | `MailPresenter.references` | References 头（数组） |
| `email.subject` | `MailPresenter.subject` | 邮件主题 |
| `email.to` | `MailPresenter.to` | 收件人（数组） |
| `email.auto_reply` | `MailPresenter.auto_reply?` | 是否自动回复 |

---

**补充字段**：

| JSON 路径 | 来源 | 说明 |
|-----------|------|------|
| `cc_email` | `MailPresenter.cc` | Cc 列表（与 email.cc 重复） |
| `bcc_email` | `MailPresenter.bcc` | Bcc 列表（与 email.bcc 重复） |

---

### 5.5 消息创建代码（带字段映射）

**位置**: `app/mailboxes/mailbox_helper.rb:6-24`

```ruby
@message = @conversation.messages.create!(
  account_id: @conversation.account_id,                # ← Conversation.account_id
  sender: @conversation.contact,                        # ← sender_type='Contact', sender_id=contact.id
  content: mail_content&.truncate(150_000),            # ← text_content[:reply] || html_content[:reply]
  inbox_id: @conversation.inbox_id,                    # ← Conversation.inbox_id
  message_type: 'incoming',                            # ← 固定值
  content_type: 'incoming_email',                      # ← 固定值
  source_id: processed_mail.message_id,                # ← MailPresenter.message_id 【核心】
  content_attributes: {
    email: processed_mail.serialized_data,             # ← 完整邮件数据
    cc_email: processed_mail.cc,                       # ← Cc 列表
    bcc_email: processed_mail.bcc                      # ← Bcc 列表
  }
)
```

---

### 5.6 附件处理

**位置**: `app/mailboxes/mailbox_helper.rb:26-40`

| 处理类型 | 来源 | 存储位置 |
|----------|------|----------|
| **普通附件** | `MailPresenter.attachments` | `ActiveStorage::Blob` + `Message.attachments` |
| **内联图片** | `MailPresenter.attachments` (inline) | 上传后替换 HTML/TXT 中的 `cid:` 引用 |

---

## 6. 端到端示例：引用头如何命中同一会话

### 场景设定

- **客服邮箱**: `support@acme.com`（已配置为 Chatwoot Email Channel）
- **客户邮箱**: `john.doe@example.com`
- **会话 UUID**: `abc123-def456-ghi789`
- **会话 display_id**: `#42`
- **账户 ID**: `1`
- **账户入站域名**: `mailer.chatwoot.com`

---

### 6.1 时序 1：客户首次发邮件（入站）

**客户邮件头**：
```
Message-ID:    <customer-first-msg@example.com>
From:          John Doe <john.doe@example.com>
To:            Support <support@acme.com>
Subject:       我的订单有问题
In-Reply-To:   (空)
References:    (空)
```

**Chatwoot 处理**：

1. IMAP 抓取到邮件
2. `find_or_create_conversation`:
   - `find_conversation_by_in_reply_to` → in_reply_to 为空，返回 nil
   - `find_conversation_by_reference_ids` → references 为空，返回 nil
   - **创建新会话**

**落库结果**：

**Conversation 表**:
| 字段 | 值 |
|------|-----|
| `id` | 100 |
| `uuid` | `abc123-def456-ghi789` |
| `display_id` | 42 |
| `additional_attributes.source` | `'email'` |
| `additional_attributes.mail_subject` | `'我的订单有问题'` |
| `additional_attributes.in_reply_to` | `nil` |

**Message 表**（入站消息 #1）:
| 字段 | 值 |
|------|-----|
| `id` | 1000 |
| `conversation_id` | 100 |
| `message_type` | `:incoming` (0) |
| `content_type` | `:incoming_email` (8) |
| `source_id` | `'<customer-first-msg@example.com>'` |
| `content_attributes.email.message_id` | `'<customer-first-msg@example.com>'` |
| `content_attributes.email.in_reply_to` | `nil` |
| `content_attributes.email.references` | `[]` |
| `content_attributes.email.subject` | `'我的订单有问题'` |

---

### 6.2 时序 2：坐席回复（出站）

**坐席在 Chatwoot 发送消息**: `"您好，请提供订单号，我帮您查询。"`

**ConversationReplyMailer 构建出站邮件头**：

**位置**: `app/mailers/conversation_reply_mailer.rb` + `app/mailers/references_header_builder.rb`

```ruby
# Message-ID（自定义格式）
custom_message_id = "<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>"

# In-Reply-To（优先使用最后一封入站邮件的 Message-ID）
in_reply_to_email = "<customer-first-msg@example.com>"

# References（第一封回复，没有历史引用）
references_header = "<customer-first-msg@example.com>"
```

**发送给客户的邮件头**：
```
Message-ID:    <conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>
From:          Support <support@acme.com>
Reply-To:      Support <support@acme.com>  (IMAP 渠道使用 channel.email)
To:            John Doe <john.doe@example.com>
Subject:       Re: 我的订单有问题
In-Reply-To:   <customer-first-msg@example.com>
References:    <customer-first-msg@example.com>
```

**坐席消息落库**（Message 表，出站消息 #2）:

| 字段 | 值 |
|------|-----|
| `id` | 1001 |
| `conversation_id` | 100 |
| `message_type` | `:outgoing` (1) |
| `content_type` | `:text` (0) |
| `source_id` | `'<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>'` |
| `content` | `'您好，请提供订单号，我帮您查询。'` |
| `content_attributes.email.message_id` | `'<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>'` |
| `content_attributes.email.in_reply_to` | `'<customer-first-msg@example.com>'` |
| `content_attributes.email.references` | `['<customer-first-msg@example.com>']` |

---

### 6.3 时序 3：客户回复坐席的邮件（入站）

**客户邮件客户端自动构建回复头**：

```
Message-ID:    <customer-reply-1@example.com>
From:          John Doe <john.doe@example.com>
To:            Support <support@acme.com>
Subject:       Re: 我的订单有问题
In-Reply-To:   <conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>
References:    <customer-first-msg@example.com>
               <conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>
```

**Chatwoot 处理流程**：

#### 步骤 A：提取引用头

`MailPresenter` 解析：
- `in_reply_to` = `"<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>"`
- `references` = `[
    "<customer-first-msg@example.com>",
    "<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>"
  ]`

#### 步骤 B：查找会话

**策略一：In-Reply-To 查找**（`app/mailboxes/imap/imap_mailbox.rb:42-51`）

```ruby
in_reply_to = "<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>"

# 方式 A: 查 Message.source_id
message = @inbox.messages.find_by(source_id: in_reply_to)
# ⚡ 命中！Message#1001 的 source_id 正好是这个值

if message.nil?  # 不执行
  @inbox.conversations.find_by("additional_attributes->>'in_reply_to' = ?", in_reply_to)
else
  @inbox.conversations.find(message.conversation_id)
  # → Conversation#100 (uuid: abc123-def456-ghi789) ✅ 命中会话
end
```

**命中路径**：
```
客户 In-Reply-To 
    → Message.find_by(source_id: "<conversation/.../messages/1001@...>")
    → Message#1001.conversation_id = 100
    → Conversation#100 ✅
```

---

**落库结果**（Message 表，入站消息 #3）:

| 字段 | 值 |
|------|-----|
| `id` | 1002 |
| `conversation_id` | 100 |
| `message_type` | `:incoming` (0) |
| `content_type` | `:incoming_email` (8) |
| `source_id` | `'<customer-reply-1@example.com>'` |
| `content_attributes.email.message_id` | `'<customer-reply-1@example.com>'` |
| `content_attributes.email.in_reply_to` | `'<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>'` |
| `content_attributes.email.references` | `['<customer-first-msg@example.com>', '<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>']` |

---

### 6.4 时序 4：坐席再次回复（出站）

**ConversationReplyMailer 构建出站邮件头**：

```ruby
# 查找被回复的邮件（入站消息 #3）
replied_to_message = find_replied_to_message(conversation, in_reply_to_message_id)
# → Message#1002

# 提取历史 References
extract_references_from_message(replied_to_message)
# → ['<customer-first-msg@example.com>', '<conversation/.../messages/1001@...>']

# 构建新的 References
references = 历史引用 + in_reply_to_message_id
# → ['<customer-first-msg@example.com>', 
#    '<conversation/.../messages/1001@...>',
#    '<customer-reply-1@example.com>']
```

**发送给客户的邮件头**：
```
Message-ID:    <conversation/abc123-def456-ghi789/messages/1003@mailer.chatwoot.com>
In-Reply-To:   <customer-reply-1@example.com>
References:    <customer-first-msg@example.com>
               <conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>
               <customer-reply-1@example.com>
```

---

### 6.5 时序 5：客户第三次回复（In-Reply-To 丢失但 References 仍在）

假设客户邮件客户端异常，**丢失了 In-Reply-To 头**，但 References 链保留：

**客户邮件头**：
```
Message-ID:    <customer-reply-2@example.com>
In-Reply-To:   (丢失，空)
References:    <customer-first-msg@example.com>
               <conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>
               <customer-reply-1@example.com>
               <conversation/abc123-def456-ghi789/messages/1003@mailer.chatwoot.com>
```

**Chatwoot 处理流程**：

#### 步骤 B：查找会话

**策略一：In-Reply-To 查找**
```ruby
in_reply_to = nil
return if in_reply_to.blank?  # ← 返回 nil
```

**策略二：References 查找**（`app/mailboxes/imap/imap_mailbox.rb:53-90`）

```ruby
references = [
  '<customer-first-msg@example.com>',
  '<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>',
  '<customer-reply-1@example.com>',
  '<conversation/abc123-def456-ghi789/messages/1003@mailer.chatwoot.com>'
]

# 子步骤 A: find_message_by_references
references.each do |message_id|
  message = @inbox.messages.find_by(source_id: message_id)
  
  # 第 1 个: '<customer-first-msg@example.com>' → Message#1000 ✅ 命中！
  # 第 2 个: '<conversation/.../messages/1001@...>' → Message#1001 ✅ 命中！
  # 第 3 个: '<customer-reply-1@example.com>' → Message#1002 ✅ 命中！
  # 第 4 个: '<conversation/.../messages/1003@...>' → Message#1003 ✅ 命中！
  
  message_to_return = message if message.present?
end
# → message_to_return = Message#1003 (最后一个命中的)

# 用 message 找 conversation
@inbox.conversations.find_by(id: message_to_return.conversation_id)
# → Conversation#100 ✅ 命中会话
```

**命中路径**：
```
客户 References 链
    → 遍历每个 reference
    → Message.find_by(source_id: reference)
    → Message#1003.conversation_id = 100
    → Conversation#100 ✅
```

---

### 6.6 示例总结：四层容错的会话串联

| 时序 | 命中策略 | 关键匹配字段 |
|------|----------|-------------|
| 客户首封邮件 | —（新建会话） | — |
| 客户第 1 次回复 | **策略一**：In-Reply-To | `In-Reply-To` → `Message.source_id` |
| 客户第 2 次回复（In-Reply-To 丢失） | **策略二**：References | `References` 链中任意一个 → `Message.source_id` |

**设计意图**：
1. **In-Reply-To 优先**：直接关联，效率最高
2. **References 兜底**：完整引用链，容错性最强
3. **source_id 作为桥梁**：出站邮件的自定义 Message-ID 会被回写为 `source_id`，形成闭环

---

## 7. 出站链路：坐席回复 → SMTP 发送

### 7.1 触发入口

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
  attachments.blank? ? 
    ::SendReplyJob.perform_later(id) : 
    ::SendReplyJob.set(wait: 2.seconds).perform_later(id)
end
```

### 7.2 分发到渠道服务

**位置**: `app/jobs/send_reply_job.rb`

根据渠道类型选择对应的发送服务：

```ruby
CHANNEL_SERVICES = {
  'Channel::Email' => ::Email::SendOnEmailService,
}
```

### 7.3 邮件发送服务

**位置**: `app/services/email/send_on_email_service.rb`

继承 `Base::SendOnChannelService`，核心方法：

```ruby
def perform_reply
  return unless message.email_notifiable_message?
  
  reply_mail = ConversationReplyMailer
    .with(account: message.account)
    .email_reply(message)
    .deliver_now
  
  # 关键：发送成功后，回写邮件的 Message-ID 到 message.source_id
  message.update(source_id: reply_mail.message_id)
rescue StandardError => e
  Messages::StatusUpdateService.new(message, 'failed', e.message).perform
end
```

### 7.4 邮件头构建（核心：会话串联的源头）

**位置**: `app/mailers/conversation_reply_mailer.rb`

#### Message-ID（自定义格式）

```ruby
def custom_message_id
  last_message = @message || @messages&.last
  "<conversation/#{@conversation.uuid}/messages/#{last_message&.id}@#{channel_email_domain}>"
end
```

格式：`<conversation/<会话UUID>/messages/<消息ID>@<域名>>`

**作用**：这个 ID 会被发送到客户邮箱，客户回复时会出现在 In-Reply-To/References 中，形成闭环。

#### In-Reply-To

```ruby
def in_reply_to_email
  conversation_reply_email_id ||  # 优先：最后一封入站邮件的原始 Message-ID
  "<account/#{@account.id}/conversation/#{@conversation.uuid}@#{channel_email_domain}>"  # fallback
end

def conversation_reply_email_id
  content_attributes = @conversation.messages.incoming.last&.content_attributes
  if content_attributes && content_attributes['email'] && content_attributes['email']['message_id']
    return "<#{content_attributes['email']['message_id']}>"
  end
  nil
end
```

**优先级**：
1. 有入站历史 → 使用客户原始邮件的 Message-ID
2. 无入站历史（坐席先发）→ 使用会话级 fallback 格式

#### References Header

**位置**: `app/mailers/references_header_builder.rb`

```ruby
def build_references_header(conversation, in_reply_to_message_id)
  # 1. 从被回复邮件中提取历史 References
  references = get_references_from_replied_message(conversation, in_reply_to_message_id)
  
  # 2. 追加被回复邮件的 Message-ID
  references << in_reply_to_message_id
  
  # 3. 去重 + RFC 5322 换行格式化
  references = references.compact.uniq
  fold_references_header(references)
end
```

**数据来源**：`Message.content_attributes.email.references`

---

### 7.5 SMTP 发送配置

**位置**: `app/mailers/conversation_reply_mailer_helper.rb:63-80`

| 方式 | 场景 | 认证方式 |
|------|------|----------|
| 用户配置 SMTP | 普通邮箱 | PLAIN/LOGIN 等 |
| OAuth SMTP (Google) | provider=google | XOAUTH2 |
| OAuth SMTP (Microsoft) | provider=microsoft | XOAUTH2 |

---

## 8. 完整收发时序图（修正版）

### 8.1 IMAP 入站：客户 → Chatwoot

```
┌──────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  邮件服务器 │     │ FetchImapEmailInboxes │     │   FetchImapEmailsJob   │
│ (IMAP)    │     │     Job (每分钟)  │     │  (单渠道抓取)          │
└────┬─────┘     └────────┬─────────┘     └──────────┬───────────┘
     │                    │                          │
     │                    │ 1. 查询所有 Email Inbox   │
     │                    │─────────────────────────>│
     │                    │                          │
     │<─────────────────────────────────────────────│ 2. SEARCH SINCE <昨天>
     │                    │                          │
     │<─────────────────────────────────────────────│ 3. FETCH BODY.PEEK[HEADER]
     │                    │                          │    (获取 Message-ID 列表，去重)
     │                    │                          │
     │<─────────────────────────────────────────────│ 4. 对新邮件 FETCH RFC822
     │                    │                          │    (获取完整内容)
     │                    │                          │
     │                    │                          │ ┌────────────────────┐
     │                    │                          │ │ Imap::ImapMailbox  │
     │                    │                          │ │ .process(mail)     │
     │                    │                          │ └─────────┬──────────┘
     │                    │                          │           │
     │                    │                          │           │ 5. MailPresenter 解析
     │                    │                          │           │    in_reply_to, references
     │                    │                          │           │
     │                    │                          │           │ 6. IMAP 简化会话查找
     │                    │                          │           │    第一步: find_conversation_by_in_reply_to
     │                    │                          │           │    第二步: find_conversation_by_reference_ids
     │                    │                          │           │    第三步: Conversation.create!
     │                    │                          │           │
     │                    │                          │           │ 7. Contact 查找/创建
     │                    │                          │           │
     │                    │                          │           │ 8. Message 创建
     │                    │                          │           │    source_id = mail.message_id
     │                    │                          │           │    content_attributes.email = serialized_data
     │                    │                          │           │
     │                    │                          │           │ 9. 附件处理（inline/regular）
```

### 8.2 出站：坐席回复 → 客户邮箱

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
     │                    │                          │           │           │    <最后一封入站邮件 Message-ID>
     │                    │                          │           │           │    或 <account/.../conversation/...>
     │                    │                          │           │           │
     │                    │                          │           │           │    References:
     │                    │                          │           │           │    历史引用链 + In-Reply-To
     │                    │                          │           │           │
     │                    │                          │           │           │ 5. SMTP 发送
     │                    │                          │           │           │
     │                    │                          │           │ 6. update(source_id: reply_mail.message_id)
     │                    │                          │           │    ↓ 关键：形成闭环
```

---

## 9. 设计要点总结

### 9.1 IMAP 入站设计
- **定时轮询**：每分钟检查 IMAP 收件箱
- **增量同步**：通过 SINCE 日期和 Message-ID 去重
- **分布式锁**：Redis 锁防止多实例重复处理
- **简化会话查找**：In-Reply-To → References → 新建会话（两步法，非四策略链）

### 9.2 会话串联设计（IMAP 渠道）

| 策略 | 优先级 | 匹配方式 | 数据库字段 |
|------|--------|----------|-----------|
| In-Reply-To | 1 | 精确匹配 | `messages.source_id` / `conversations.additional_attributes.in_reply_to` |
| References | 2 | 遍历引用链 | `messages.source_id` 或正则提取 UUID |

### 9.3 出站设计
- **延迟发送**：附件等待 2 秒确保 ActiveStorage 上传完成
- **动态 SMTP**：每个渠道可独立配置 SMTP 或使用 OAuth
- **状态追踪**：发送成功后回写 `source_id`，失败记录错误信息
- **引用链维持**：References 头完整传递历史上下文（RFC 5322 规范）

### 9.4 数据落库映射

| 表 | 关键字段 | 数据来源 | 作用 |
|----|----------|----------|------|
| `messages.source_id` | string | `MailPresenter.message_id` | **核心关联字段** |
| `messages.content_attributes.email` | json | `MailPresenter.serialized_data` | 完整邮件元数据 |
| `conversations.additional_attributes.in_reply_to` | string | `MailPresenter.in_reply_to` | 会话级 In-Reply-To 备份 |
| `conversations.additional_attributes.mail_subject` | string | `MailPresenter.subject` | 原始邮件主题 |

### 9.5 闭环设计

```
出站: Chatwoot 生成自定义 Message-ID 
    → 存入 Message.source_id
    ↓
客户回复: 邮件客户端自动带入 In-Reply-To/References
    ↓
入站: IMAP 解析 In-Reply-To/References
    → 匹配 Message.source_id
    → 找到对应 Conversation
```

---

## 10. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 渠道模型 | `app/models/channel/email.rb` |
| 定时调度配置 | `config/schedule.yml` |
| IMAP 调度 Job | `app/jobs/inboxes/fetch_imap_email_inboxes_job.rb` |
| IMAP 抓取 Job | `app/jobs/inboxes/fetch_imap_emails_job.rb` |
| IMAP 抓取服务基类 | `app/services/imap/base_fetch_email_service.rb` |
| **IMAP 邮件处理（核心）** | `app/mailboxes/imap/imap_mailbox.rb` |
| 邮件装饰器 | `app/presenters/mail_presenter.rb` |
| 消息创建/附件 | `app/mailboxes/mailbox_helper.rb` |
| References 头构建 | `app/mailers/references_header_builder.rb` |
| 坐席回复触发 | `app/models/message.rb` (send_reply 回调) |
| 回复分发 Job | `app/jobs/send_reply_job.rb` |
| 邮件发送服务 | `app/services/email/send_on_email_service.rb` |
| 回复邮件构建器 | `app/mailers/conversation_reply_mailer.rb` |
| 回复邮件辅助 | `app/mailers/conversation_reply_mailer_helper.rb` |
| 数据库 Schema | `db/migrate/20230426130150_init_schema.rb` |

---

## 11. 附录：通用四策略链（非 IMAP 渠道使用）

**位置**: `app/services/mailbox/conversation_finder.rb`

供 ActionMailbox 路由等其他邮件入口使用，IMAP 渠道**不使用**：

```ruby
DEFAULT_STRATEGIES = [
  Mailbox::ConversationFinderStrategies::ReceiverUuidStrategy,   # 1: 收件人 UUID
  Mailbox::ConversationFinderStrategies::InReplyToStrategy,      # 2: In-Reply-To
  Mailbox::ConversationFinderStrategies::ReferencesStrategy,     # 3: References
  Mailbox::ConversationFinderStrategies::NewConversationStrategy # 4: 新建会话
].freeze
```
