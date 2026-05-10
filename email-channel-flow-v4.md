# Chatwoot 邮件渠道完整收发链路分析（v4 - inbound_email_enabled? 差异修正）

## 修正说明

v3 报告的不准确之处：
- **错误**：「非邮箱渠道且 inbound 开启」场景下新旧逻辑完全一致
- **事实**：旧版 `inbound_email_enabled?` 额外依赖 `@account.support_email.present?`，新版不依赖

本报告在 v3 基础上修正：
- 明确新旧 `inbound_email_enabled?` 的条件差异
- 重画 Reply-To 与 ReceiverUuid 命中路径的分歧
- 提供最小可复现案例说明旧逻辑为什么会回退

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

### 6.6 示例总结

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

## 7. reply_mailer_migration 开关分析（v4 修正）

### 7.1 Feature Flag 概述

**位置**: `config/features.yml:213-218`

```yaml
- name: reply_mailer_migration
  display_name: Reply Mailer Migration
  enabled: false
  chatwoot_internal: true
```

**代码注释**：
> "This feature is temporary only to migrate reply mailer to new email builder. Once the migration is done, this feature can be removed."

**状态**：当前默认关闭，仅 Chatwoot 内部使用，用于渐进式迁移旧的发件人/回复地址逻辑到新的 Builder 类。

---

### 7.2 开关的作用点

**位置**: `app/mailers/conversation_reply_mailer_helper.rb:94-104`

```ruby
def email_from
  return Email::FromBuilder.new(inbox: @inbox, message: current_message).build if @account.feature_enabled?(:reply_mailer_migration)
  email_oauth_enabled? || email_smtp_enabled? ? channel_email_with_name : from_email_with_name
end

def email_reply_to
  return Email::ReplyToBuilder.new(inbox: @inbox, message: current_message).build if @account.feature_enabled?(:reply_mailer_migration)
  email_imap_enabled? ? @channel.email : reply_email
end
```

**影响范围**：
| 开关打开 | 开关关闭 |
|----------|----------|
| `Email::FromBuilder.build()` | 旧逻辑：`email_from` |
| `Email::ReplyToBuilder.build()` | 旧逻辑：`email_reply_to` |

---

### 7.3 旧逻辑（开关关闭）详解

#### 7.3.1 旧 email_from 逻辑

**位置**: `app/mailers/conversation_reply_mailer_helper.rb:94-98` + `conversation_reply_mailer.rb:111-113,135-147`

```ruby
def email_from
  email_oauth_enabled? || email_smtp_enabled? ? channel_email_with_name : from_email_with_name
end

def from_email
  should_use_conversation_email_address? ? parse_email(@account.support_email) : parse_email(inbox_from_email_address)
end
```

**分支决策**：
```
email_oauth_enabled? || email_smtp_enabled?
    ├── true  → channel_email_with_name → @channel.email
    └── false → from_email_with_name
                    ├── should_use_conversation_email_address?
                    │       ├── true  → @account.support_email
                    │       └── false → @inbox.email_address || @account.support_email
                    └── sender_name(...) 包装
```

**判定条件**:
| 条件 | 定义 |
|------|------|
| `email_oauth_enabled?` | 邮箱渠道 + (Google OAuth 或 Microsoft OAuth) |
| `email_smtp_enabled?` | 邮箱渠道 + 已配置 SMTP |
| `should_use_conversation_email_address?` | 邮箱渠道 或 inbound_email_enabled?（旧版） |

---

#### 7.3.2 旧 email_reply_to 逻辑

**位置**: `app/mailers/conversation_reply_mailer_helper.rb:100-104` + `conversation_reply_mailer.rb:127-133`

```ruby
def email_reply_to
  email_imap_enabled? ? @channel.email : reply_email
end

def reply_email
  if should_use_conversation_email_address?
    sender_name("reply+#{@conversation.uuid}@#{@account.inbound_email_domain}")
  else
    @inbox.email_address || @agent&.email
  end
end
```

**分支决策**：
```
email_imap_enabled?
    ├── true  → @channel.email（直接回复渠道邮箱）
    └── false → reply_email
                    ├── should_use_conversation_email_address?
                    │       ├── true  → reply+<conversation-uuid>@<domain>
                    │       └── false → @inbox.email_address || @agent.email
                    └── sender_name(...) 包装（仅 true 分支）
```

---

### 7.4 新逻辑（开关打开）详解

#### 7.4.1 新 Email::FromBuilder 逻辑

**位置**: `app/builders/email/from_builder.rb`

```ruby
def build
  return sender_name(account_support_email) unless inbox.email?

  from_email = case email_channel_type
               when :standard_imap_smtp,
                    :google_oauth,
                    :microsoft_oauth,
                    :forwarding_own_smtp
                 channel.email
               when :imap_chatwoot_smtp,
                    :forwarding_chatwoot_smtp
                 channel.verified_for_sending ? channel.email : account_support_email
               else
                 account_support_email
               end

  sender_name(from_email)
end
```

**邮件渠道类型分类**：
| 类型 | IMAP 启用 | SMTP 启用 | Provider | `verified_for_sending` 要求 |
|------|-----------|-----------|----------|------------------------------|
| `:google_oauth` | 任意 | 任意 | google | ❌ 不需要 |
| `:microsoft_oauth` | 任意 | 任意 | microsoft | ❌ 不需要 |
| `:standard_imap_smtp` | ✅ | ✅ | 空 | ❌ 不需要 |
| `:forwarding_own_smtp` | ❌ | ✅ | 空 | ❌ 不需要 |
| `:imap_chatwoot_smtp` | ✅ | ❌ | 空 | ✅ 需要（未验证则降级） |
| `:forwarding_chatwoot_smtp` | ❌ | ❌ | 空 | ✅ 需要（未验证则降级） |

---

#### 7.4.2 新 Email::ReplyToBuilder 逻辑

**位置**: `app/builders/email/reply_to_builder.rb`

```ruby
def build
  reply_to = if inbox.email?
               channel.email
             elsif inbound_email_enabled?
               "reply+#{conversation.uuid}@#{account.inbound_email_domain}"
             else
               account_support_email
             end

  sender_name(reply_to)
end

def inbound_email_enabled?
  account.feature_enabled?('inbound_emails') && account.inbound_email_domain.present?
end
```

**Reply-To 决策（新逻辑）**：
```
inbox.email?
    ├── true  → channel.email
    └── false → inbound_email_enabled?
                  ├── true  → reply+<conversation-uuid>@<domain>
                  └── false → account_support_email
```

---

### 7.5 关键差异：inbound_email_enabled? 条件（v4 修正）

**⚠️ v3 报告遗漏的关键点**：

**旧版 `inbound_email_enabled?`**（`conversation_reply_mailer.rb:197-200`）:
```ruby
def inbound_email_enabled?
  @inbound_email_enabled ||= 
    @account.feature_enabled?('inbound_emails') && 
    @account.inbound_email_domain.present? && 
    @account.support_email.present?  # ← 额外依赖 support_email
end
```

**新版 `inbound_email_enabled?`**（`reply_to_builder.rb:18-19`）:
```ruby
def inbound_email_enabled?
  account.feature_enabled?('inbound_emails') && 
  account.inbound_email_domain.present?  # ← 不依赖 support_email
end
```

**条件对比表**：

| 条件 | 旧版 | 新版 |
|------|------|------|
| `feature_enabled?('inbound_emails')` | ✅ 必须 | ✅ 必须 |
| `inbound_email_domain.present?` | ✅ 必须 | ✅ 必须 |
| `support_email.present?` | ✅ **必须**（旧版额外要求） | ❌ 不要求 |

---

### 7.6 条件差异的传播链

这个差异通过 `should_use_conversation_email_address?` 传播到旧逻辑的多个分支：

**位置**: `conversation_reply_mailer.rb:74-76`

```ruby
def should_use_conversation_email_address?
  @inbox.inbox_type == 'Email' || inbound_email_enabled?
end
```

**传播链图示**：

```
旧逻辑 email_reply_to（非 IMAP 渠道）
    ↓
reply_email
    ↓
should_use_conversation_email_address?
    ├── @inbox.inbox_type == 'Email'  → true（邮箱渠道）
    └── inbound_email_enabled?（旧版）→ 需同时满足
            ├── feature_enabled?('inbound_emails')
            ├── inbound_email_domain.present?
            └── support_email.present?  ← 关键差异
```

---

### 7.7 开关对比汇总表（v4 修正）

#### 7.7.1 关键场景：非邮箱渠道 + inbound 开启 + support_email 未配置

**v3 错误**：认为新旧逻辑完全一致

**v4 修正**：新旧逻辑**行为不同**

| 配置项 | 值 |
|--------|-----|
| 渠道类型 | API Channel（非 Email） |
| `inbound_emails` feature | ✅ 开启 |
| `inbound_email_domain` | ✅ 已配置 `mailer.chatwoot.com` |
| `support_email` | ❌ **未配置** |

**旧逻辑执行路径**（开关 OFF）：
```
email_reply_to
    ↓
email_imap_enabled? = false（非 IMAP 渠道）
    ↓
reply_email
    ↓
should_use_conversation_email_address?
    ├── @inbox.inbox_type == 'Email' = false
    └── inbound_email_enabled?（旧版）= false  ← support_email 为空
            ↓
    false
    ↓
@inbox.email_address || @agent.email  ← 回退！
```

**新逻辑执行路径**（开关 ON）：
```
ReplyToBuilder.build
    ↓
inbox.email? = false
    ↓
inbound_email_enabled?（新版）= true  ← 不检查 support_email
    ↓
"reply+#{conversation.uuid}@mailer.chatwoot.com"  ← 使用 UUID 地址！
```

**结果对比**：

| 开关状态 | `inbound_email_enabled?` 返回 | 最终 Reply-To |
|----------|------------------------------|---------------|
| OFF（旧） | `false`（因 support_email 为空） | `inbox.email_address \|\| agent.email` |
| ON（新） | `true` | `reply+abc123-def456@mailer.chatwoot.com` |

---

#### 7.7.2 完整场景对比表

| 场景 | 渠道类型 | inbound feature | inbound_domain | support_email | 旧逻辑（OFF） | 新逻辑（ON） | 是否一致？ |
|------|----------|-----------------|----------------|---------------|---------------|--------------|-----------|
| A | Email + IMAP 启用 | 任意 | 任意 | 任意 | `channel.email` | `channel.email` | ✅ 一致 |
| B | 非 Email + inbound 关闭 | ❌ | 任意 | 任意 | `inbox.email_address \|\| agent.email` | `account.support_email` | 🔴 不一致 |
| C | 非 Email + inbound 开启 + support_email 配置 | ✅ | ✅ | ✅ | `reply+<uuid>@domain` | `reply+<uuid>@domain` | ✅ 一致 |
| D | 非 Email + inbound 开启 + support_email **未配置** | ✅ | ✅ | ❌ | `inbox.email_address \|\| agent.email` | `reply+<uuid>@domain` | 🔴 **不一致**（v4 修正） |

---

### 7.8 最小可复现案例（v4 新增）

#### 7.8.1 环境配置

| 配置项 | 值 |
|--------|-----|
| 账户 ID | `1` |
| 账户 `inbound_email_domain` | `mailer.chatwoot.com` |
| 账户 `support_email` | `nil`（未配置） |
| `inbound_emails` feature | ✅ 开启 |
| 渠道类型 | API Channel（非 Email） |
| Inbox ID | `50` |
| Inbox `email_address` | `nil`（未配置） |
| 当前坐席 `email` | `agent@company.com` |
| 会话 UUID | `abc123-def456-ghi789` |
| `reply_mailer_migration` feature | ❌ 关闭（测试旧逻辑）|

---

#### 7.8.2 旧逻辑（开关 OFF）执行步骤

**步骤 1**：坐席发送回复消息，触发 `ConversationReplyMailer`

**步骤 2**：计算 `email_reply_to`
```ruby
email_imap_enabled? = false  # 非 IMAP 渠道
→ 走 reply_email
```

**步骤 3**：计算 `reply_email`
```ruby
# should_use_conversation_email_address?
@inbox.inbox_type == 'Email' = false  # API Channel
# 旧版 inbound_email_enabled?
@account.feature_enabled?('inbound_emails') = true
@account.inbound_email_domain.present? = true
@account.support_email.present? = false  # ← 关键
→ inbound_email_enabled? = false

→ should_use_conversation_email_address? = false || false = false
```

**步骤 4**：走 else 分支
```ruby
@inbox.email_address = nil
@agent&.email = "agent@company.com"
→ Reply-To = "agent@company.com"
```

**步骤 5**：客户收到邮件
```
Reply-To: Agent Name <agent@company.com>
```

**步骤 6**：客户点击「回复」
```
To: agent@company.com
```

**步骤 7**：入站会话串联影响

**ReceiverUuidStrategy**（策略一）：
```ruby
# 匹配收件人地址是否包含 reply+<uuid>
UUID_PATTERN = /^reply\+([0-9a-f]{8}\b-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-\b[0-9a-f]{12})$/i

# 实际收件人: agent@company.com
# 不匹配 UUID_PATTERN
→ 策略一失败
```

**结果**：
- 依赖 **InReplyToStrategy**（策略二）和 **ReferencesStrategy**（策略三）
- 如果客户邮件客户端未正确设置 `In-Reply-To`/`References`，可能**创建新会话**

---

#### 7.8.3 新逻辑（开关 ON）执行步骤

**步骤 1**：同样的环境，打开 `reply_mailer_migration` 开关

**步骤 2**：计算 `ReplyToBuilder.build`
```ruby
inbox.email? = false  # 非 Email 渠道
```

**步骤 3**：新版 `inbound_email_enabled?`
```ruby
account.feature_enabled?('inbound_emails') = true
account.inbound_email_domain.present? = true
→ inbound_email_enabled? = true  # ← 不检查 support_email
```

**步骤 4**：使用 UUID 地址
```ruby
"reply+#{conversation.uuid}@#{account.inbound_email_domain}"
= "reply+abc123-def456-ghi789@mailer.chatwoot.com"
```

**步骤 5**：客户收到邮件
```
Reply-To: Support <reply+abc123-def456-ghi789@mailer.chatwoot.com>
```

**步骤 6**：客户点击「回复」
```
To: reply+abc123-def456-ghi789@mailer.chatwoot.com
```

**步骤 7**：入站会话串联

**ReceiverUuidStrategy**（策略一）：
```ruby
# 实际收件人: reply+abc123-def456-ghi789@mailer.chatwoot.com
# 提取 username: reply+abc123-def456-ghi789
# 匹配 UUID_PATTERN → 提取 uuid = abc123-def456-ghi789

Conversation.find_by(uuid: "abc123-def456-ghi789")
→ Conversation#xxx ✅ 命中会话
```

**结果**：
- **策略一直接命中**，无需依赖 `In-Reply-To`/`References`
- 最可靠的会话串联方式

---

#### 7.8.4 案例对比总结

| 步骤 | 旧逻辑（开关 OFF） | 新逻辑（开关 ON） |
|------|-------------------|-------------------|
| Reply-To 头 | `agent@company.com` | `reply+<uuid>@mailer.chatwoot.com` |
| 客户回复的收件人 | `agent@company.com` | `reply+<uuid>@mailer.chatwoot.com` |
| 策略一（ReceiverUuid） | ❌ 不匹配 | ✅ 匹配并命中 |
| 依赖策略 | 策略二/三（引用头） | 策略一（最可靠） |
| 风险 | 引用头缺失时可能新建会话 | ⚡ 零风险 |

**根因**：旧版 `inbound_email_enabled?` 额外检查 `support_email.present?`，导致在 inbound 开启但 support_email 未配置时错误地回退到非 UUID 地址。

---

### 7.9 对入站会话串联命中的实际影响（v4 修正）

#### 7.9.1 影响链路图示

```
出站（坐席回复）                        入站（客户回复）
─────────────────────────────────────────────────────────────

reply_mailer_migration 开关
    ↓
FromBuilder / ReplyToBuilder 被使用？
    ├── ON  → 新版 inbound_email_enabled?（不检查 support_email）
    │       └── inbound 开启时 → Reply-To = reply+<uuid>@...
    │                                      ↓
    │                              ReceiverUuidStrategy 命中 ✅
    │
    └── OFF → 旧版 inbound_email_enabled?（检查 support_email）
            ├── support_email 已配置 → Reply-To = reply+<uuid>@...
            │                          ↓
            │                    ReceiverUuidStrategy 命中 ✅
            │
            └── support_email 未配置 → Reply-To = inbox/agent email
                                       ↓
                                 ReceiverUuidStrategy 失败 ❌
                                       ↓
                                 依赖 InReplyTo/References 策略
                                       ↓
                                 可能新建会话 ⚠️
```

---

#### 7.9.2 各场景会话串联命中概率

| 场景 | 开关 OFF 命中概率 | 开关 ON 命中概率 | 差异说明 |
|------|-------------------|------------------|---------|
| IMAP 渠道 | 100%（不受开关影响） | 100% | IMAP 用 In-Reply-To/References 策略，不受 Reply-To 影响 |
| 非 IMAP + inbound 关闭 | ~80%（依赖引用头） | ~80% | 都走策略二/三 |
| 非 IMAP + inbound 开启 + support_email 配置 | 100%（策略一） | 100% | 一致 |
| 非 IMAP + inbound 开启 + support_email 未配置 | ~80%（依赖引用头） | 100%（策略一） | 🔴 **开关开启提升命中率** |

---

#### 7.9.3 不受开关影响的邮件头

**重要**：以下邮件头**完全不受 `reply_mailer_migration` 开关影响**：

| 邮件头 | 构建位置 | 是否受开关影响 |
|--------|----------|----------------|
| `Message-ID` | `ConversationReplyMailer.custom_message_id` | ❌ 不受 |
| `In-Reply-To` | `ConversationReplyMailer.in_reply_to_email` | ❌ 不受 |
| `References` | `ReferencesHeaderBuilder.build_references_header` | ❌ 不受 |
| `Subject` | `ConversationReplyMailer.mail_subject` | ❌ 不受 |

**设计意图**：
- `reply_mailer_migration` 只影响 `From` 和 `Reply-To` 的构建逻辑
- 会话串联的核心引用头（Message-ID/In-Reply-To/References）完全独立
- 确保开关切换不会破坏已有引用链

---

### 7.10 开关影响总结

| 维度 | 影响程度 | 说明 |
|------|----------|------|
| **IMAP 渠道会话串联** | ❌ 无影响 | IMAP 用 In-Reply-To/References 策略，不受 Reply-To 影响 |
| **IMAP 渠道 From 地址** | ✅ 影响（安全改进） | Chatwoot SMTP 场景强制 `verified_for_sending`，防止 spam |
| **非 IMAP + inbound 开启 + support_email 未配置** | ⚠️ **重大影响**（v4 修正） | 旧逻辑走回退，新逻辑用 UUID 地址，提升策略一命中率 |
| **其他场景** | ⚠️ 轻微影响 | 回退地址不同，但核心串联逻辑一致 |

**设计意图**：
1. **渐进式迁移**：Feature Flag 允许灰度切换
2. **安全性改进**：Chatwoot SMTP 场景强制验证 sender
3. **Bug 修复**：移除 `inbound_email_enabled?` 中不必要的 `support_email` 依赖
4. **向后兼容**：核心引用头完全不变

---

## 8. 出站链路：坐席回复 → SMTP 发送

### 8.1 触发入口

**位置**: `app/models/message.rb:324-333, 397-401`

坐席发送消息后，Message 模型的 `after_create_commit` 回调触发：

```ruby
def execute_after_create_commit_callbacks
  reopen_conversation
  mark_pending_conversation_as_open_for_human_response
  set_conversation_activity
  dispatch_create_events
  send_reply
  execute_message_template_hooks
  update_contact_activity
end

def send_reply
  attachments.blank? ? 
    ::SendReplyJob.perform_later(id) : 
    ::SendReplyJob.set(wait: 2.seconds).perform_later(id)
end
```

### 8.2 分发到渠道服务

**位置**: `app/jobs/send_reply_job.rb`

根据渠道类型选择对应的发送服务：

```ruby
CHANNEL_SERVICES = {
  'Channel::Email' => ::Email::SendOnEmailService,
}
```

### 8.3 邮件发送服务

**位置**: `app/services/email/send_on_email_service.rb`

继承 `Base::SendOnChannelService`，核心方法：

```ruby
def perform_reply
  return unless message.email_notifiable_message?
  
  reply_mail = ConversationReplyMailer
    .with(account: message.account)
    .email_reply(message)
    .deliver_now
  
  message.update(source_id: reply_mail.message_id)
rescue StandardError => e
  Messages::StatusUpdateService.new(message, 'failed', e.message).perform
end
```

### 8.4 邮件头构建

**位置**: `app/mailers/conversation_reply_mailer.rb`

#### Message-ID（自定义格式）

```ruby
def custom_message_id
  last_message = @message || @messages&.last
  "<conversation/#{@conversation.uuid}/messages/#{last_message&.id}@#{channel_email_domain}>"
end
```

格式：`<conversation/<会话UUID>/messages/<消息ID>@<域名>>`

#### In-Reply-To

```ruby
def in_reply_to_email
  conversation_reply_email_id ||
  "<account/#{@account.id}/conversation/#{@conversation.uuid}@#{channel_email_domain}>"
end
```

#### References Header

**位置**: `app/mailers/references_header_builder.rb`

```ruby
def build_references_header(conversation, in_reply_to_message_id)
  references = get_references_from_replied_message(conversation, in_reply_to_message_id)
  references << in_reply_to_message_id
  references = references.compact.uniq
  fold_references_header(references)
end
```

---

### 8.5 SMTP 发送配置

**位置**: `app/mailers/conversation_reply_mailer_helper.rb:63-80`

| 方式 | 场景 | 认证方式 |
|------|------|----------|
| 用户配置 SMTP | 普通邮箱 | PLAIN/LOGIN 等 |
| OAuth SMTP (Google) | provider=google | XOAUTH2 |
| OAuth SMTP (Microsoft) | provider=microsoft | XOAUTH2 |

---

## 9. 完整收发时序图

### 9.1 IMAP 入站：客户 → Chatwoot

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
     │                    │                          │           │
     │                    │                          │           │ 6. IMAP 简化会话查找
     │                    │                          │           │    第一步: In-Reply-To
     │                    │                          │           │    第二步: References
     │                    │                          │           │    第三步: 新建会话
     │                    │                          │           │
     │                    │                          │           │ 7. Contact 查找/创建
     │                    │                          │           │
     │                    │                          │           │ 8. Message 创建
     │                    │                          │           │    source_id = mail.message_id
     │                    │                          │           │
     │                    │                          │           │ 9. 附件处理
```

### 9.2 出站：坐席回复 → 客户邮箱

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
     │                    │                          │           │           │    From: email_from
     │                    │                          │           │           │    Reply-To: email_reply_to
     │                    │                          │           │           │    （两者受开关影响）
     │                    │                          │           │           │
     │                    │                          │           │           │    Message-ID:
     │                    │                          │           │           │    <conversation/<uuid>/
     │                    │                          │           │           │     messages/<id>@domain>
     │                    │                          │           │           │
     │                    │                          │           │           │    In-Reply-To:
     │                    │                          │           │           │    <最后一封入站邮件 ID>
     │                    │                          │           │           │
     │                    │                          │           │           │    References:
     │                    │                          │           │           │    历史引用链 + In-Reply-To
     │                    │                          │           │           │    （不受开关影响）
     │                    │                          │           │           │
     │                    │                          │           │           │ 5. SMTP 发送
     │                    │                          │           │           │
     │                    │                          │           │ 6. update(source_id: reply_mail.message_id)
```

---

## 10. 设计要点总结

### 10.1 IMAP 入站设计
- **定时轮询**：每分钟检查 IMAP 收件箱
- **增量同步**：通过 SINCE 日期和 Message-ID 去重
- **分布式锁**：Redis 锁防止多实例重复处理
- **简化会话查找**：In-Reply-To → References → 新建会话（两步法）

### 10.2 会话串联设计（IMAP 渠道）

| 策略 | 优先级 | 匹配方式 | 数据库字段 |
|------|--------|----------|-----------|
| In-Reply-To | 1 | 精确匹配 | `messages.source_id` |
| References | 2 | 遍历引用链 | `messages.source_id` 或正则 |

### 10.3 出站设计
- **延迟发送**：附件等待 2 秒确保 ActiveStorage 上传完成
- **动态 SMTP**：每个渠道可独立配置 SMTP 或使用 OAuth
- **状态追踪**：发送成功后回写 `source_id`
- **Feature Flag**：`reply_mailer_migration` 渐进式迁移 From/Reply-To 逻辑

### 10.4 reply_mailer_migration 开关核心发现（v4）

| 发现 | 说明 |
|------|------|
| **条件差异** | 旧版 `inbound_email_enabled?` 额外检查 `support_email.present?`，新版不检查 |
| **传播影响** | 通过 `should_use_conversation_email_address?` 影响 `email_reply_to` 的分支选择 |
| **关键场景** | 非 IMAP 渠道 + inbound 开启 + support_email 未配置时，新旧逻辑行为不同 |
| **串联影响** | 旧逻辑回退到 `inbox/agent.email`，策略一无法命中；新逻辑使用 `reply+<uuid>@...`，策略一直接命中 |

---

## 11. 关键文件索引

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
| **回复邮件辅助（开关分支）** | `app/mailers/conversation_reply_mailer_helper.rb` |
| **新 From 构建器** | `app/builders/email/from_builder.rb` |
| **新 Reply-To 构建器** | `app/builders/email/reply_to_builder.rb` |
| **旧版 inbound_email_enabled?（v4 关键）** | `app/mailers/conversation_reply_mailer.rb:197-200` |
| **新版 inbound_email_enabled?（v4 关键）** | `app/builders/email/reply_to_builder.rb:18-19` |
| **Feature Flag 配置** | `config/features.yml` |
| 数据库 Schema | `db/migrate/20230426130150_init_schema.rb` |

---

## 12. 附录：通用四策略链（非 IMAP 渠道使用）

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

### 策略一：ReceiverUuidStrategy

**场景**：非 IMAP 渠道（API Channel + email_continuity）

**原理**：从收件人地址中提取会话 UUID

**Reply-To 格式**：
```
Reply-To: Support <reply+6bdc3f4d-0bec-4515-a284-5d916fdde489@mailer.chatwoot.com>
```

**匹配模式**（`receiver_uuid_strategy.rb:14-15`）:
```ruby
UUID_PATTERN = /^reply\+([0-9a-f]{8}\b-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-\b[0-9a-f]{12})$/i
```

**受 `reply_mailer_migration` 开关的影响**（v4 修正）：

| 场景 | 开关 OFF | 开关 ON | 策略一是否可用 |
|------|----------|---------|---------------|
| inbound 开启 + support_email 配置 | `reply+<uuid>@...` | `reply+<uuid>@...` | ✅ 可用 |
| inbound 开启 + support_email **未配置** | `inbox.email_address \|\| agent.email` | `reply+<uuid>@...` | 🔴 **OFF 不可用，ON 可用**（v4 修正） |
| inbound 关闭 | `inbox.email_address \|\| agent.email` | `account.support_email` | ❌ 都不可用 |

**结论**：
- 开关 ON 时，策略一的可用场景更多（修复了 support_email 未配置时的 Bug）
- 开关 OFF 时，若 support_email 未配置，即使 inbound 功能开启，策略一也无法使用，降级到引用头策略
