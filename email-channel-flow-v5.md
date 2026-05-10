# Chatwoot 邮件渠道完整收发链路分析（v5 - 代码可证据对照表 + 确定性表述修正）

## 修正说明

v4 报告的不准确之处：
- **错误**：使用概率化表述（100%/80%），代码中无支撑概率的证据
- **事实**：所有会话串联命中路径都是**条件化确定性**的，可通过代码条件精确推导

本报告在 v4 基础上修正：
- 将概率化表述改为条件化确定性表述（可命中/不可命中/依赖引用头）
- 新增「代码可证据对照表」，每条结论都对应具体代码位置和条件

---

## 1. 概述

本文档详细分析 Chatwoot 邮件渠道（Email Channel）的完整收发链路，包括：
- **入站链路**：从 IMAP 定时抓取到消息解析归档
- **出站链路**：从坐席回复触发到 SMTP 邮件发送
- **会话串联**：邮件引用头（References/In-Reply-To）如何将往来邮件串联为同一会话
- **Feature Flag**：`reply_mailer_migration` 开关对 From/Reply-To 的影响

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
| `verified_for_sending` | 仅使用 Chatwoot SMTP 时有效，标识是否已验证发件权限 |

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

## 4. IMAP 入站邮件解析与会话串联

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

  message = @inbox.messages.find_by(source_id: in_reply_to)
  
  if message.nil?
    @inbox.conversations.find_by("additional_attributes->>'in_reply_to' = ?", in_reply_to)
  else
    @inbox.conversations.find(message.conversation_id)
  end
end
```

**查找逻辑**：
1. 用 `In-Reply-To` 的值去 `messages.source_id` 精确匹配
2. 若未找到消息，尝试直接匹配 `conversations.additional_attributes.in_reply_to`
3. 所有查询都限定在当前 Inbox 范围内

---

#### 4.2.2 策略二：References 查找

**位置**: `app/mailboxes/imap/imap_mailbox.rb:53-90`

```ruby
def find_conversation_by_reference_ids
  return if @inbound_mail.references.blank?

  message = find_message_by_references
  if message.present?
    conversation = @inbox.conversations.find_by(id: message.conversation_id)
    return conversation if conversation.present?
  end

  conversation_id = find_conversation_by_references
  @inbox.conversations.find_by(uuid: conversation_id) if conversation_id.present?
end

def find_message_by_references
  message_to_return = nil
  references = Array.wrap(@inbound_mail.references)
  
  references.each do |message_id|
    message = @inbox.messages.find_by(source_id: message_id)
    message_to_return = message if message.present?
  end
  message_to_return
end

def find_conversation_by_references
  references = Array.wrap(@inbound_mail.references)
  references.each do |message_id|
    match = FALLBACK_CONVERSATION_PATTERN.match(message_id)
    return match[2] if match.present?
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
      in_reply_to: in_reply_to,
      auto_reply: @processed_mail.auto_reply?,
      mail_subject: @processed_mail.subject,
      initiated_at: { timestamp: Time.now.utc }
    }
  }
)
```

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

---

## 5. 归档落库字段及其来源

### 5.1 邮件装饰器（数据源）

**位置**: `app/presenters/mail_presenter.rb`

所有落库数据最终来自 `MailPresenter.serialized_data`：

```ruby
def serialized_data
  {
    bcc: bcc,
    cc: cc,
    content_type: content_type,
    date: date,
    from: from,
    headers: headers_data,
    html_content: html_content,
    in_reply_to: in_reply_to,
    message_id: message_id,
    multipart: multipart?,
    number_of_attachments: number_of_attachments,
    references: references,
    subject: subject,
    text_content: text_content,
    to: to,
    auto_reply: auto_reply?
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
| `uuid` | 自动生成 | 会话唯一标识 |
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
| `content` | text | `text_content[:reply]` 或 `html_content[:reply]` | 消息正文 |
| `content_attributes` | json | 见下表 | 邮件完整元数据 |

---

**content_attributes.email 完整结构**:

| JSON 路径 | 来源 |
|-----------|------|
| `email.bcc` | `MailPresenter.bcc` |
| `email.cc` | `MailPresenter.cc` |
| `email.content_type` | `MailPresenter.content_type` |
| `email.date` | `MailPresenter.date` |
| `email.from` | `MailPresenter.from` |
| `email.headers` | `MailPresenter.headers_data` |
| `email.html_content.full` | `MailPresenter.html_content[:full]` |
| `email.html_content.reply` | `MailPresenter.html_content[:reply]` |
| `email.html_content.quoted` | `MailPresenter.html_content[:quoted]` |
| `email.text_content.full` | `MailPresenter.text_content[:full]` |
| `email.text_content.reply` | `MailPresenter.text_content[:reply]` |
| `email.text_content.quoted` | `MailPresenter.text_content[:quoted]` |
| `email.in_reply_to` | `MailPresenter.in_reply_to` |
| `email.message_id` | `MailPresenter.message_id` |
| `email.multipart` | `MailPresenter.multipart?` |
| `email.number_of_attachments` | `MailPresenter.number_of_attachments` |
| `email.references` | `MailPresenter.references` |
| `email.subject` | `MailPresenter.subject` |
| `email.to` | `MailPresenter.to` |
| `email.auto_reply` | `MailPresenter.auto_reply?` |

---

### 5.5 消息创建代码（带字段映射）

**位置**: `app/mailboxes/mailbox_helper.rb:6-24`

```ruby
@message = @conversation.messages.create!(
  account_id: @conversation.account_id,
  sender: @conversation.contact,
  content: mail_content&.truncate(150_000),
  inbox_id: @conversation.inbox_id,
  message_type: 'incoming',
  content_type: 'incoming_email',
  source_id: processed_mail.message_id,
  content_attributes: {
    email: processed_mail.serialized_data,
    cc_email: processed_mail.cc,
    bcc_email: processed_mail.bcc
  }
)
```

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

**策略一：In-Reply-To 查找**

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

**策略二：References 查找**

```ruby
references = [
  '<customer-first-msg@example.com>',
  '<conversation/abc123-def456-ghi789/messages/1001@mailer.chatwoot.com>',
  '<customer-reply-1@example.com>',
  '<conversation/abc123-def456-ghi789/messages/1003@mailer.chatwoot.com>'
]