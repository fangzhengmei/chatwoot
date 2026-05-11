# Chatwoot 满意度调查 (CSAT) 完整流程分析

## 概述

本文档详细分析 Chatwoot 中满意度调查（CSAT）的生成、发送、客户提交评价以及跨域回调写回系统的完整技术流程。

---

## 一、CSAT 链接的生成与发送

### 1.1 触发时机

CSAT 调查在对话被标记为"已解决"（resolved）时触发：

**监听器**：`app/listeners/csat_survey_listener.rb:1-15`

```ruby
class CsatSurveyListener < BaseListener
  def conversation_status_changed(event)
    conversation = extract_conversation_and_account(event)[0]
    return unless conversation.resolved?
    CsatSurveyService.new(conversation: conversation).perform
  end

  def message_updated(event)
    message = extract_message_and_account(event)[0]
    return unless message.input_csat?
    CsatSurveys::ResponseBuilder.new(message: message).perform
  end
end
```

**事件订阅**：监听 `conversation_status_changed` 事件，当对话状态变为 `resolved` 时触发 CSAT 发送流程。

**监听器注册**：`app/dispatchers/async_dispatcher.rb:15`

```ruby
def listeners
  [
    AutomationRuleListener.instance,
    CampaignListener.instance,
    CsatSurveyListener.instance,  # 注册到异步分发器
    HookListener.instance,
    # ...
  ]
end
```

---

### 1.2 CSAT 发送条件检查

**服务类**：`app/services/csat_survey_service.rb:1-180`

CSAT 发送需要满足以下条件：

1. **对话允许 CSAT**：对话已解决且不是 Twitter 推文
   ```ruby
   def conversation_allows_csat?
     conversation.resolved? && !conversation.tweet?
   end
   ```

2. **Inbox 已启用 CSAT**：
   ```ruby
   def csat_enabled?
     inbox.csat_survey_enabled?
   end
   ```

3. **CSAT 尚未发送**：
   ```ruby
   def csat_already_sent?
     conversation.messages.where(content_type: :input_csat).present?
   end
   ```

4. **符合调查规则**（可选）：
   ```ruby
   def csat_allowed_by_survey_rules?
     return true unless survey_rules_configured?
     labels = conversation.label_list
     return true if rule_values.empty?
     case rule_operator
     when 'contains'
       rule_values.any? { |label| labels.include?(label) }
     when 'does_not_contain'
       rule_values.none? { |label| labels.include?(label) }
     else
       true
     end
   end
   ```

---

### 1.3 CSAT 发送策略

根据渠道类型选择不同的发送方式：

| 场景 | 发送方式 |
|------|---------|
| WhatsApp 渠道 + 已审批模板 | 发送 WhatsApp 模板消息 |
| Twilio WhatsApp + 已审批模板 | 发送 Twilio 模板消息 |
| 在消息窗口内（可回复） | 发送普通消息模板 |
| 超出消息窗口 | 创建活动消息记录（不实际发送） |

**普通消息模板创建**：`app/services/message_templates/template/csat_survey.rb:1-40`

```ruby
class MessageTemplates::Template::CsatSurvey
  def perform
    ActiveRecord::Base.transaction do
      conversation.messages.create!(csat_survey_message_params)
    end
  end

  private

  def csat_survey_message_params
    {
      account_id: @conversation.account_id,
      inbox_id: @conversation.inbox_id,
      message_type: :template,
      content_type: :input_csat,  # 关键标识
      content: message_content,
      content_attributes: content_attributes
    }
  end

  def content_attributes
    {
      display_type: csat_config['display_type'] || 'emoji'  # emoji 或 star
    }
  end
end
```

---

### 1.4 CSAT 外链 URL 生成

#### 1.4.1 外链页面路由

**路由配置**：`config/routes.rb:34-36`

```ruby
namespace :survey do
  resources :responses, only: [:show]  # GET /survey/responses/:id
end
```

**页面控制器**：`app/controllers/survey/responses_controller.rb:1-10`

```ruby
class Survey::ResponsesController < ActionController::Base
  before_action :set_global_config
  def show; end  # 渲染 SPA 页面

  private

  def set_global_config
    @global_config = GlobalConfig.get('LOGO_THUMBNAIL', 'BRAND_NAME', 'WIDGET_BRAND_URL', 'INSTALLATION_NAME')
  end
end
```

#### 1.4.2 URL 生成逻辑

**对话模型**：`app/models/conversation.rb:211-213`

```ruby
def csat_survey_link
  "#{ENV.fetch('FRONTEND_URL', nil)}/survey/responses/#{uuid}"
end
```

**消息内容 Presenter**：`app/presenters/message_content_presenter.rb:1-33`

```ruby
class MessageContentPresenter < SimpleDelegator
  def outgoing_content
    Messages::MarkdownRendererService.new(
      content_with_survey_link,
      conversation.inbox.channel_type,
      conversation.inbox.channel
    ).render
  end

  def webhook_content
    Messages::WebhookContentNormalizer.normalize(content_with_survey_link)
  end

  private

  def content_with_survey_link
    if should_append_survey_link?
      survey_link = survey_url(conversation.uuid)
      custom_message = inbox.csat_config&.dig('message')
      custom_message.present? ? "#{custom_message} #{survey_link}" : I18n.t('conversations.survey.response', link: survey_link)
    else
      content
    end
  end

  def should_append_survey_link?
    input_csat? && !inbox.web_widget?  # 非 Web Widget 渠道才附加外链
  end

  def survey_url(conversation_uuid)
    "#{ENV.fetch('FRONTEND_URL', nil)}/survey/responses/#{conversation_uuid}"
  end
end
```

**邮件模板**：`app/views/mailers/conversation_reply_mailer/reply_with_summary.html.erb:13-14`

```erb
<% if (message.content_type == 'input_csat' && message.message_type == 'template') %>
  <p>Click <a href="<%= message.conversation.csat_survey_link %>" _target="blank">here</a> to rate the conversation.</p>
<% end %>
```

#### 1.4.3 URL 生成规则总结

| 场景 | URL 格式 | 示例 |
|------|---------|------|
| 外链评分页 | `{FRONTEND_URL}/survey/responses/{conversation_uuid}` | `https://app.chatwoot.com/survey/responses/abc123-def456` |
| Web Widget | 无外链，直接在会话内显示评分组件 | - |
| WhatsApp 模板 | `{base_url}/survey/responses/{{1}}`（模板变量） | `https://app.chatwoot.com/survey/responses/{conversation_uuid}` |

---

### 1.5 WhatsApp 模板发送

#### 1.5.1 官方 WhatsApp 渠道

**发送逻辑**：`app/services/csat_survey_service.rb:107-155`

```ruby
def send_whatsapp_template_survey
  template_config = inbox.csat_config&.dig('template')
  template_name = template_config['name'] || CsatTemplateNameService.csat_template_name(inbox.id)
  phone_number = conversation.contact_inbox.source_id
  template_info = build_template_info(template_name, template_config)
  message = build_csat_message
  message_id = inbox.channel.provider_service.send_template(phone_number, template_info, message)
  message.update!(source_id: message_id) if message_id.present?
end

def build_template_info(template_name, template_config)
  {
    name: template_name,
    lang_code: template_config['language'] || 'en',
    parameters: [
      {
        type: 'button',
        sub_type: 'url',
        index: '0',
        parameters: [{ type: 'text', text: conversation.uuid }]  # 传入对话 UUID 作为 URL 后缀
      }
    ]
  }
end
```

#### 1.5.2 模板创建

**模板服务**：`app/services/whatsapp/csat_template_service.rb:61-96`

```ruby
def build_template_components(template_config)
  [
    build_body_component(template_config[:message]),
    build_buttons_component(template_config)
  ]
end

def build_buttons_component(template_config)
  {
    type: 'BUTTONS',
    buttons: [
      {
        type: 'URL',
        text: template_config[:button_text] || DEFAULT_BUTTON_TEXT,
        url: "#{template_config[:base_url]}/survey/responses/{{1}}",  # 模板 URL 格式
        example: ['12345']
      }
    ]
  }
end
```

---

## 二、CSAT 评价表单展示（两条路径）

### 路径 A：Web Widget 会话内评分

适用于 Web Widget 渠道，用户在聊天窗口内直接评分。

#### 2.1 前端组件结构

**消息气泡组件**：`app/javascript/widget/components/AgentMessageBubble.vue:61-150`

```javascript
isCSAT() {
  return this.contentType === 'input_csat';
}

// 渲染 CSAT 组件
<CustomerSatisfaction
  v-if="isCSAT"
  :message-content-attributes="messageContentAttributes.submitted_values"
  :display-type="messageContentAttributes.display_type"
  :message="message"
  :message-id="messageId"
/>
```

#### 2.2 CSAT 评价组件

**核心组件**：`app/javascript/shared/components/CustomerSatisfaction.vue:1-209`

组件特性：
- 支持两种显示方式：emoji（表情）和 star（星级）
- 评分范围：1-5
- 可选反馈消息输入
- 提交后显示已提交状态

```javascript
methods: {
  async onSubmit() {
    this.isUpdating = true;
    try {
      await this.$store.dispatch('message/update', {
        submittedValues: {
          csat_survey_response: {
            rating: this.selectedRating,
            feedback_message: this.feedback,
          },
        },
        messageId: this.messageId,
      });
    } catch (error) {
      // Ignore error
    } finally {
      this.isUpdating = false;
    }
  },
}
```

---

### 路径 B：外链独立评分页

适用于 WhatsApp、Email、SMS 等渠道，用户点击链接跳转到独立页面评分。

#### 2.3 外链页面组件

**页面视图**：`app/javascript/survey/views/Response.vue:1-204`

```javascript
// 从 URL 获取 survey ID（conversation UUID）
computed: {
  surveyId() {
    const pageURL = window.location.href;
    return pageURL.substring(pageURL.lastIndexOf('/') + 1);
  },
},

// 挂载时获取调查详情
async mounted() {
  this.getSurveyDetails();
},

methods: {
  async getSurveyDetails() {
    this.isLoading = true;
    try {
      const result = await getSurveyDetails({ uuid: this.surveyId });
      this.logo = result.data.inbox_avatar_url;
      this.inboxName = result.data.inbox_name;
      this.surveyDetails = result?.data?.csat_survey_response;
      this.selectedRating = this.surveyDetails?.rating;
      this.feedbackMessage = this.surveyDetails?.feedback_message || '';
      this.displayType = result.data.display_type || CSAT_DISPLAY_TYPES.EMOJI;
      this.setLocale(result.data.locale);
    } catch (error) {
      const errorMessage = error?.response?.data?.message;
      this.errorMessage = errorMessage || this.$t('SURVEY.API.ERROR_MESSAGE');
    } finally {
      this.isLoading = false;
    }
  },

  async updateSurveyDetails() {
    this.isUpdating = true;
    try {
      const data = {
        message: {
          submitted_values: {
            csat_survey_response: {
              rating: this.selectedRating,
              feedback_message: this.feedbackMessage,
            },
          },
        },
      };
      await updateSurvey({
        uuid: this.surveyId,
        data,
      });
      this.surveyDetails = {
        rating: this.selectedRating,
        feedback_message: this.feedbackMessage,
      };
    } catch (error) {
      const errorMessage = error?.response?.data?.error;
      this.errorMessage = errorMessage || this.$t('SURVEY.API.ERROR_MESSAGE');
      useAlert(this.errorMessage);
    } finally {
      this.isUpdating = false;
    }
  },
}
```

**页面组件特性**：
- 独立的 SPA 页面，通过 `survey/responses/:id` 路由访问
- 从 URL 提取 `conversation_uuid`
- 支持本地化（locale）
- 显示品牌 Logo 和 Inbox 名称
- 支持 emoji 和 star 两种评分方式

---

## 三、跨域回调与数据提交（两条路径对比）

### 3.1 CORS 配置

**配置文件**：`config/initializers/cors.rb:1-35`

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    # 允许所有公共 API 端点的跨域访问
    resource '/public/api/*', headers: :any, methods: :any

    if ActiveModel::Type::Boolean.new.cast(ENV.fetch('CW_API_ONLY_SERVER', false)) || Rails.env.development?
      resource '*', headers: :any, methods: :any, expose: %w[access-token client uid expiry]
    end

    if ActiveModel::Type::Boolean.new.cast(ENV.fetch('ENABLE_API_CORS', false))
      resource '/api/*', headers: :any, methods: :any, expose: %w[access-token client uid expiry]
    end
  end
end
```

**关键配置说明**：
- `origins '*'`：允许来自任何域名的请求
- `resource '/public/api/*'`：所有 `/public/api/` 下的接口都支持跨域
- `methods: :any`：允许所有 HTTP 方法（GET, POST, PATCH, PUT, DELETE 等）
- `headers: :any`：允许任何请求头

### 3.2 CSRF 处理

**公共控制器基类**：`app/controllers/public_controller.rb:1-28`

```ruby
class PublicController < ActionController::Base
  include RequestExceptionHandler
  skip_before_action :verify_authenticity_token  # 跳过 CSRF 验证
end
```

---

## 四、评价数据写回系统（两条回写路径详细对比）

### 4.0 两条路径总览对比

| 维度 | 路径 A：Web Widget 会话内评分 | 路径 B：外链独立评分页 |
|------|-----------------------------|---------------------|
| **适用渠道** | Web Widget | WhatsApp、Email、SMS 等 |
| **前端入口** | `CustomerSatisfaction.vue`（共享组件） | `Response.vue`（独立页面） |
| **身份识别** | Message ID（数据库自增 ID） | Conversation UUID |
| **API 端点** | `/api/v1/widget/messages/:id` | `/public/api/v1/csat_survey/:uuid` |
| **HTTP 方法** | PATCH | PUT |
| **控制器** | `Public::Api::V1::Inboxes::MessagesController` | `Public::Api::V1::CsatSurveyController` |
| **参数结构** | `submitted_values: {...}` | `message: { submitted_values: {...} }` |
| **后续流程** | 相同（事件驱动 → ResponseBuilder） | 相同（事件驱动 → ResponseBuilder） |

---

### 路径 A：Web Widget 会话内评分回写

#### 4.1.1 前端 Store Action

**Store Action**：`app/javascript/widget/store/modules/message.js:13-44`

```javascript
export const actions = {
  update: async (
    { commit, dispatch, getters: { getUIFlags: uiFlags } },
    { email, messageId, submittedValues }
  ) => {
    if (uiFlags.isUpdating) {
      return;
    }
    commit('toggleUpdateStatus', true);
    try {
      await MessageAPI.update({
        email,
        messageId,
        values: submittedValues,
      });
      commit(
        'conversation/updateMessage',
        {
          id: messageId,
          content_attributes: {
            submitted_email: email,
            submitted_values: email ? null : submittedValues,
          },
        },
        { root: true }
      );
      dispatch('contacts/get', {}, { root: true });
    } catch (error) {
      // Ignore error
    }
    commit('toggleUpdateStatus', false);
  },
};
```

#### 4.1.2 Widget API 客户端

**API 客户端**：`app/javascript/widget/api/message.js:1-12`

```javascript
import authEndPoint from 'widget/api/endPoints';
import { API } from 'widget/helpers/axios';

export default {
  update: ({ messageId, email, values }) => {
    const urlData = authEndPoint.updateMessage(messageId);
    return API.patch(urlData.url, {
      contact: { email },
      message: { submitted_values: values },
    });
  },
};
```

**端点定义**：`app/javascript/widget/api/endPoints.js:86-88`

```javascript
const updateMessage = id => ({
  url: `/api/v1/widget/messages/${id}${window.location.search}`,
});
```

#### 4.1.3 后端控制器处理

**Messages 控制器**：`app/controllers/public/api/v1/inboxes/messages_controller.rb:1-73`

```ruby
class Public::Api::V1::Inboxes::MessagesController < Public::Api::V1::InboxesController
  before_action :set_message, only: [:update]

  def update
    # 14 天后锁定，无法修改
    render json: { error: 'You cannot update the CSAT survey after 14 days' }, status: :unprocessable_entity and return if check_csat_locked

    @message.update!(message_update_params)
  rescue StandardError => e
    render json: { error: @contact.errors, message: e.message }.to_json, status: :internal_server_error
  end

  private

  def message_update_params
    params.permit(submitted_values: [:name, :title, :value, { csat_survey_response: [:feedback_message, :rating] }])
  end

  def set_message
    @message = @conversation.messages.find(params[:id])  # 通过 Message ID 查找
  end

  def check_csat_locked
    (Time.zone.now.to_date - @message.created_at.to_date).to_i > 14 and @message.content_type == 'input_csat'
  end
end
```

**路由配置**：`config/routes.rb:557`

```ruby
resources :messages, only: [:index, :create, :update]  # PATCH /api/v1/widget/messages/:id
```

**请求示例**：
```bash
PATCH /api/v1/widget/messages/123?website_token=xxx
Content-Type: application/json

{
  "contact": { "email": "user@example.com" },
  "message": {
    "submitted_values": {
      "csat_survey_response": {
        "rating": 5,
        "feedback_message": "服务很好！"
      }
    }
  }
}
```

---

### 路径 B：外链独立评分页回写

#### 4.2.1 外链 API 客户端

**Survey API**：`app/javascript/survey/api/survey.js:1-15`

```javascript
import endPoints from 'survey/api/endPoints';
import { API } from 'survey/helpers/axios';

const getSurveyDetails = async ({ uuid }) => {
  const urlData = endPoints.getSurvey({ uuid });
  const result = await API.get(urlData.url, { params: urlData.params });
  return result;
};

const updateSurvey = async ({ uuid, data }) => {
  const urlData = endPoints.updateSurvey({ data, uuid });
  await API.put(urlData.url, { ...urlData.data });  // 使用 PUT 方法
};

export { getSurveyDetails, updateSurvey };
```

**端点定义**：`app/javascript/survey/api/endPoints.js:1-13`

```javascript
const updateSurvey = ({ uuid, data }) => ({
  url: `/public/api/v1/csat_survey/${uuid}`,
  data,
});

const getSurvey = ({ uuid }) => ({
  url: `/public/api/v1/csat_survey/${uuid}`,
});

export default {
  getSurvey,
  updateSurvey,
};
```

#### 4.2.2 后端控制器处理

**CSAT 控制器**：`app/controllers/public/api/v1/csat_survey_controller.rb:1-32`

```ruby
class Public::Api::V1::CsatSurveyController < PublicController
  before_action :set_conversation
  before_action :set_message

  def show; end

  def update
    # 14 天后锁定，无法修改
    render json: { error: 'You cannot update the CSAT survey after 14 days' }, status: :unprocessable_entity and return if check_csat_locked

    @message.update!(message_update_params[:message])
  end

  private

  def set_conversation
    return if params[:id].blank?
    @conversation = Conversation.find_by!(uuid: params[:id])  # 通过 UUID 查找
  end

  def set_message
    @message = @conversation.messages.find_by!(content_type: 'input_csat')
  end

  def message_update_params
    params.permit(message: [{ submitted_values: [:name, :title, :value, { csat_survey_response: [:feedback_message, :rating] }] }])
  end

  def check_csat_locked
    (Time.zone.now.to_date - @message.created_at.to_date).to_i > 14
  end
end
```

**路由配置**：`config/routes.rb:563`

```ruby
resources :csat_survey, only: [:show, :update]  # PUT /public/api/v1/csat_survey/:id
```

**请求示例**：
```bash
PUT /public/api/v1/csat_survey/abc123-def456-uuid
Content-Type: application/json

{
  "message": {
    "submitted_values": {
      "csat_survey_response": {
        "rating": 5,
        "feedback_message": "客服服务非常专业！"
      }
    }
  }
}
```

---

### 4.3 公共流程：消息更新事件触发（两条路径汇合点）

无论使用哪条路径，当 Message 更新后都会触发相同的事件流程。

#### 4.3.1 Message 模型回调

**Message 模型**：`app/models/message.rb:137-140, 389-395, 246-249`

```ruby
after_update_commit :dispatch_update_event  # 更新后提交触发

def dispatch_update_event
  # ref: https://github.com/rails/rails/issues/44500
  # we want to skip the update event if the message is not updated
  return if previous_changes.blank?

  send_update_event
end

def send_update_event
  Rails.configuration.dispatcher.dispatch(
    MESSAGE_UPDATED,
    Time.zone.now,
    message: self,
    performed_by: Current.executed_by,
    previous_changes: previous_changes
  )
end
```

#### 4.3.2 事件监听与处理

**事件监听器**：`app/listeners/csat_survey_listener.rb:10-15`

```ruby
def message_updated(event)
  message = extract_message_and_account(event)[0]
  return unless message.input_csat?  # 只处理 CSAT 消息
  CsatSurveys::ResponseBuilder.new(message: message).perform
end
```

**响应构建器**：`app/builders/csat_surveys/response_builder.rb:1-28`

```ruby
class CsatSurveys::ResponseBuilder
  pattr_initialize [:message]

  def perform
    raise 'Invalid Message' unless message.input_csat?

    conversation = message.conversation
    rating = message.content_attributes.dig('submitted_values', 'csat_survey_response', 'rating')
    feedback_message = message.content_attributes.dig('submitted_values', 'csat_survey_response', 'feedback_message')

    return if rating.blank?

    process_csat_response(conversation, rating, feedback_message)
  end

  private

  def process_csat_response(conversation, rating, feedback_message)
    csat_survey_response = message.csat_survey_response || CsatSurveyResponse.new(
      message_id: message.id,
      account_id: message.account_id,
      conversation_id: message.conversation_id,
      contact_id: conversation.contact_id,
      assigned_agent: conversation.assignee
    )
    csat_survey_response.rating = rating
    csat_survey_response.feedback_message = feedback_message
    csat_survey_response.save!
    csat_survey_response
  end
end
```

---

### 4.4 CSAT 响应数据模型

**数据模型**：`app/models/csat_survey_response.rb:1-47`

```ruby
class CsatSurveyResponse < ApplicationRecord
  belongs_to :account
  belongs_to :conversation
  belongs_to :contact
  belongs_to :message
  belongs_to :assigned_agent, class_name: 'User', optional: true, inverse_of: :csat_survey_responses
  belongs_to :review_notes_updated_by, class_name: 'User', optional: true

  validates :rating, presence: true, inclusion: { in: [1, 2, 3, 4, 5] }
  validates :account_id, presence: true
  validates :contact_id, presence: true
  validates :conversation_id, presence: true

  scope :filter_by_created_at, ->(range) { where(created_at: range) if range.present? }
  scope :filter_by_assigned_agent_id, ->(user_ids) { where(assigned_agent_id: user_ids) if user_ids.present? }
  scope :filter_by_inbox_id, ->(inbox_id) { joins(:conversation).where(conversations: { inbox_id: inbox_id }) if inbox_id.present? }
  scope :filter_by_team_id, ->(team_id) { joins(:conversation).where(conversations: { team_id: team_id }) if team_id.present? }
  scope :filter_by_rating, ->(rating) { where(rating: rating) if rating.present? }
end
```

**数据库字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `rating` | integer | 评分（1-5）必填 |
| `feedback_message` | text | 反馈消息 |
| `csat_review_notes` | text | 内部评审备注 |
| `account_id` | bigint | 账户 ID |
| `conversation_id` | bigint | 对话 ID |
| `contact_id` | bigint | 联系人 ID |
| `message_id` | bigint | 消息 ID（唯一） |
| `assigned_agent_id` | bigint | 分配的客服 ID |
| `review_notes_updated_at` | datetime | 评审备注更新时间 |
| `review_notes_updated_by_id` | bigint | 评审备注更新人 ID |

---

## 五、完整流程时序图

### 5.1 总览时序图

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│ 对话状态变更 │────▶│CsatSurvey    │────▶│CsatSurvey    │────▶│ 创建 CSAT    │────▶│ 消息存储        │
│ (resolved)  │     │Listener      │     │Service       │     │ 消息 (input_csat) ││ (messages表)    │
└─────────────┘     └──────────────┘     └──────────────┘     └──────────────┘     └──────────────────┘
                                                                      │
                                                                      ▼
                                                           ┌─────────────────────────┐
                                                           │ 判断渠道类型             │
                                                           │ 是 Web Widget?          │
                                                           └───────────┬─────────────┘
                                                                       │
                        ┌──────────────────────────────────────────────┼──────────────────────────────────────────────┐
                        │                                              │                                              │
                        ▼                                              ▼                                              ▼
              ┌─────────────────┐                        ┌─────────────────────────┐                    ┌─────────────────────────┐
              │ Web Widget 渠道 │                        │ WhatsApp/Email/SMS 渠道 │                    │ 邮件通知               │
              └────────┬────────┘                        └───────────┬─────────────┘                    └──────────┬──────────────┘
                       │                                              │                                              │
                       ▼                                              ▼                                              ▼
              ┌─────────────────┐                        ┌─────────────────────────┐                    ┌─────────────────────────┐
              │ 直接嵌入组件    │                        │ 生成外链 URL            │                    │ 邮件模板渲染           │
              │ CustomerSatis- │                        │ /survey/responses/:uuid │                    │ csat_survey_link        │
              │ faction.vue    │                        │ 附加到消息内容          │                    │                         │
              └────────┬────────┘                        └───────────┬─────────────┘                    └──────────┬──────────────┘
                       │                                              │                                              │
                       ▼                                              ▼                                              ▼
              ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
              │                        用户评分提交（两条路径分叉）                                                        │
              └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 路径 A：Web Widget 会话内评分时序图

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ 用户选择评分    │────▶│CustomerSatis-    │────▶│Vuex Store:       │────▶│Axios PATCH       │────▶│/api/v1/widget/   │
│ (emoji/star)   │     │faction.vue       │     │message/update     │     │请求              │     │messages/:id      │
└─────────────────┘     └────────┬─────────┘     └────────┬─────────┘     └──────────────────┘     └────────┬─────────┘
                                 │                        │                                                 │
                                 │                        │                                                 ▼
                                 │                        │                                     ┌─────────────────────────┐
                                 │                        │                                     │MessagesController#update │
                                 │                        │                                     │@message.update!          │
                                 │                        │                                     └───────────┬─────────────┘
                                 │                        │                                                 │
                                 │                        │                                                 ▼
                                 │                        │                                     ┌─────────────────────────┐
                                 │                        │                                     │after_update_commit       │
                                 │                        │                                     │dispatch_update_event    │
                                 │                        │                                     └───────────┬─────────────┘
                                 │                        │                                                 │
                                 │                        │                                                 ▼
                                 │                        │                                     ┌─────────────────────────┐
                                 │                        │                                     │MESSAGE_UPDATED 事件    │
                                 │                        │                                     │CsatSurveyListener        │
                                 │                        │                                     └───────────┬─────────────┘
                                 │                        │                                                 │
                                 │                        │                                                 ▼
                                 │                        │                                     ┌─────────────────────────┐
                                 │                        └────────────────────────────────────▶│ResponseBuilder#perform  │
                                 │                                                              │CsatSurveyResponse.save!  │
                                 │                                                              └───────────┬─────────────┘
                                 │                                                                          │
                                 │                                                                          ▼
                                 │                                                              ┌─────────────────────────┐
                                 └─────────────────────────────────────────────────────────────▶│ 数据持久化               │
                                                                │ csat_survey_responses 表  │
                                                                └─────────────────────────┘
```

### 5.3 路径 B：外链独立评分页时序图

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ 用户点击链接    │────▶│GET /survey/      │────▶│Response.vue      │────▶│GET /public/api/  │────▶│CsatSurvey        │
│ (WhatsApp/Email)│     │responses/:uuid  │     │mounted()         │     │v1/csat_survey/   │     │Controller#show   │
└─────────────────┘     └──────────────────┘     └────────┬─────────┘     │:uuid             │     └────────┬─────────┘
                                                          │               └──────────────────┘              │
                                                          │                                               ▼
                                                          │                                    ┌─────────────────────────┐
                                                          │                                    │ 返回调查详情            │
                                                          │                                    │ inbox_avatar_url        │
                                                          │                                    │ inbox_name              │
                                                          │                                    │ display_type            │
                                                          │                                    │ csat_survey_response    │
                                                          │                                    └───────────┬─────────────┘
                                                          │                                                │
                                                          ▼                                                │
                                               ┌─────────────────────────┐                                  │
                                               │ 渲染评分页面             │                                  │
                                               │ Rating / Feedback 组件  │                                  │
                                               └───────────┬─────────────┘                                  │
                                                           │                                                │
                                                           ▼                                                │
                                               ┌─────────────────────────┐                                  │
                                               │ 用户选择评分/提交反馈    │                                  │
                                               └───────────┬─────────────┘                                  │
                                                           │                                                │
                                                           ▼                                                │
                                               ┌─────────────────────────┐                                  │
                                               │Response.vue             │                                  │
                                               │updateSurveyDetails()    │                                  │
                                               └───────────┬─────────────┘                                  │
                                                           │                                                │
                                                           ▼                                                │
                                               ┌─────────────────────────┐                                  │
                                               │Axios PUT 请求           │                                  │
                                               │/public/api/v1/          │                                  │
                                               │csat_survey/:uuid        │                                  │
                                               └───────────┬─────────────┘                                  │
                                                           │                                                │
                                                           ▼                                                │
                                               ┌─────────────────────────┐                                  │
                                               │CsatSurveyController     │◀─────────────────────────────────┘
                                               │#update                  │
                                               │@message.update!        │
                                               └───────────┬─────────────┘
                                                           │
                                                           ▼
                                               ┌─────────────────────────┐
                                               │after_update_commit       │
                                               │dispatch_update_event    │
                                               └───────────┬─────────────┘
                                                           │
                                                           ▼
                                               ┌─────────────────────────┐
                                               │MESSAGE_UPDATED 事件    │
                                               │CsatSurveyListener        │
                                               └───────────┬─────────────┘
                                                           │
                                                           ▼
                                               ┌─────────────────────────┐
                                               │ResponseBuilder#perform  │
                                               │CsatSurveyResponse.save!  │
                                               └───────────┬─────────────┘
                                                           │
                                                           ▼
                                               ┌─────────────────────────┐
                                               │ 数据持久化               │
                                               │ csat_survey_responses 表  │
                                               └─────────────────────────┘
```

---

## 六、关键技术点总结

### 6.1 两条回写路径详细对比

| 对比项 | 路径 A：Web Widget 会话内 | 路径 B：外链独立评分页 |
|--------|------------------------|---------------------|
| **适用渠道** | Web Widget 网站插件 | WhatsApp、Email、SMS、API 渠道 |
| **前端组件** | `shared/components/CustomerSatisfaction.vue` | `survey/views/Response.vue` |
| **页面路由** | 无（嵌入 Widget） | `GET /survey/responses/:id` |
| **身份识别** | Message 数据库 ID（`params[:id]`） | Conversation UUID（`params[:id]`） |
| **API 端点** | `/api/v1/widget/messages/:id` | `/public/api/v1/csat_survey/:uuid` |
| **HTTP 方法** | **PATCH** | **PUT** |
| **控制器** | `Public::Api::V1::Inboxes::MessagesController` | `Public::Api::V1::CsatSurveyController` |
| **请求参数包装** | `message: { submitted_values: {...} }` | `message: { submitted_values: {...} }` |
| **参数结构差异** | 外层有 `contact: { email }` | 无 contact 参数 |
| **前置中间件** | `set_inbox_channel`, `set_contact_inbox`, `set_conversation` | `set_conversation`, `set_message` |
| **查找逻辑** | `@conversation.messages.find(params[:id])` | `Conversation.find_by!(uuid: params[:id])` |
| **14天锁检查** | `check_csat_locked` (需同时满足 content_type == input_csat) | `check_csat_locked` |
| **更新方式** | `@message.update!(message_update_params)` | `@message.update!(message_update_params[:message])` |
| **事件触发** | `after_update_commit :dispatch_update_event` | 相同 |
| **事件处理** | `CsatSurveyListener#message_updated` | 相同 |
| **数据落库** | `CsatSurveys::ResponseBuilder` | 相同 |

### 6.2 URL 完整生成链路

```
对话状态变更为 resolved
    ↓
CsatSurveyListener#conversation_status_changed
    ↓
CsatSurveyService#perform
    ↓
MessageTemplates::Template::CsatSurvey#perform
    ↓
创建 Message (content_type: :input_csat)
    ↓
根据渠道类型决定是否附加外链:
  ├── Web Widget 渠道: 不附加外链，前端直接渲染组件
  └── 其他渠道: MessageContentPresenter#content_with_survey_link
                    ↓
              survey_url(conversation.uuid)
                    ↓
              "#{FRONTEND_URL}/survey/responses/#{uuid}"
                    ↓
              例如: https://app.chatwoot.com/survey/responses/abc123-def456
```

### 6.3 安全性设计

| 机制 | 实现位置 | 说明 |
|------|---------|------|
| 跨域支持 | `config/initializers/cors.rb` | `/public/api/*` 允许所有源 |
| CSRF 跳过 | `app/controllers/public_controller.rb` | 公共接口不验证 CSRF Token |
| 身份识别 - 外链 | `Conversation.find_by!(uuid: params[:id])` | 使用 UUID 而非自增 ID |
| 身份识别 - Widget | `@conversation.messages.find(params[:id])` | 依赖会话上下文 |
| 时间锁 | `check_csat_locked` | 14 天后无法修改评价 |
| 强参数 | `message_update_params` | 仅允许白名单字段 |

### 6.4 事件落库完整流程

```
用户提交评价
    ↓
API 控制器处理
    ↓
Message.update!(content_attributes)  # 更新 messages 表
    ↓
ActiveRecord 回调触发: after_update_commit
    ↓
Message#dispatch_update_event
    ↓
Message#send_update_event
    ↓
Rails.configuration.dispatcher.dispatch(MESSAGE_UPDATED, ...)
    ↓
AsyncDispatcher 分发到所有已注册 listeners
    ↓
CsatSurveyListener#message_updated(event)
    ↓
检查 message.input_csat?
    ↓
CsatSurveys::ResponseBuilder#perform
    ↓
从 message.content_attributes 提取 rating 和 feedback_message
    ↓
CsatSurveyResponse.create / update
    ↓
数据持久化到 csat_survey_responses 表
```

### 6.5 核心文件索引

| 功能 | 文件路径 |
|------|---------|
| CSAT 发送服务 | `app/services/csat_survey_service.rb` |
| CSAT 消息模板 | `app/services/message_templates/template/csat_survey.rb` |
| CSAT 事件监听 | `app/listeners/csat_survey_listener.rb` |
| CSAT 响应构建 | `app/builders/csat_surveys/response_builder.rb` |
| CSAT 控制器（外链） | `app/controllers/public/api/v1/csat_survey_controller.rb` |
| Messages 控制器（Widget） | `app/controllers/public/api/v1/inboxes/messages_controller.rb` |
| 外链评分页控制器 | `app/controllers/survey/responses_controller.rb` |
| CSAT 数据模型 | `app/models/csat_survey_response.rb` |
| 消息内容 Presenter（URL 生成） | `app/presenters/message_content_presenter.rb` |
| 外链页面组件 | `app/javascript/survey/views/Response.vue` |
| 共享评分组件 | `app/javascript/shared/components/CustomerSatisfaction.vue` |
| Widget API 客户端 | `app/javascript/widget/api/message.js` |
| 外链 API 客户端 | `app/javascript/survey/api/survey.js` |
| CORS 配置 | `config/initializers/cors.rb` |
| 公共控制器基类 | `app/controllers/public_controller.rb` |
| 事件分发器 | `app/dispatchers/async_dispatcher.rb` |

---

## 七、API 示例

### 7.1 路径 A：Web Widget 会话内提交 CSAT 评价

**请求**：
```bash
PATCH /api/v1/widget/messages/123?website_token=abc-def-123
Content-Type: application/json

{
  "contact": {
    "email": "user@example.com"
  },
  "message": {
    "submitted_values": {
      "csat_survey_response": {
        "rating": 5,
        "feedback_message": "客服服务非常专业，问题解决得很快！"
      }
    }
  }
}
```

**成功响应**（200 OK）：
```json
{}
```

**失败响应**（422 Unprocessable Entity）：
```json
{
  "error": "You cannot update the CSAT survey after 14 days"
}
```

---

### 7.2 路径 B：外链独立评分页提交 CSAT 评价

**请求**：
```bash
PUT /public/api/v1/csat_survey/{conversation_uuid}
Content-Type: application/json

{
  "message": {
    "submitted_values": {
      "csat_survey_response": {
        "rating": 5,
        "feedback_message": "客服服务非常专业，问题解决得很快！"
      }
    }
  }
}
```

**成功响应**（200 OK）：
```json
{}
```

**失败响应**（422 Unprocessable Entity）：
```json
{
  "error": "You cannot update the CSAT survey after 14 days"
}
```

---

### 7.3 获取 CSAT 调查信息（外链页面使用）

**请求**：
```bash
GET /public/api/v1/csat_survey/{conversation_uuid}
```

**响应**（通过 JBuilder 序列化）：
- 参考 `app/views/public/api/v1/models/_csat_survey.json.jbuilder`

返回数据结构：
```json
{
  "inbox_avatar_url": "https://...",
  "inbox_name": "Support Inbox",
  "display_type": "emoji",
  "content": "Please rate this conversation",
  "locale": "en",
  "csat_survey_response": {
    "rating": null,
    "feedback_message": null
  }
}
```

---

## 八、数据流转总览

### 8.1 会话内评分数据流

```
用户选择评分
    ↓
CustomerSatisfaction.vue (onSubmit)
    ↓
Vuex Store: message/update
    ↓
Axios: PATCH /api/v1/widget/messages/:id
    ↓
Public::Api::V1::Inboxes::MessagesController#update
    ↓
Message.update!(content_attributes)
    ↓
after_update_commit → dispatch_update_event
    ↓
MESSAGE_UPDATED 事件
    ↓
CsatSurveyListener#message_updated
    ↓
CsatSurveys::ResponseBuilder#perform
    ↓
CsatSurveyResponse.create / update
    ↓
数据持久化到 csat_survey_responses 表
```

### 8.2 外链评分数据流

```
用户点击链接: /survey/responses/:uuid
    ↓
Response.vue 加载
    ↓
Axios: GET /public/api/v1/csat_survey/:uuid
    ↓
获取调查详情
    ↓
用户选择评分
    ↓
Response.vue (updateSurveyDetails)
    ↓
Axios: PUT /public/api/v1/csat_survey/:uuid
    ↓
Public::Api::V1::CsatSurveyController#update
    ↓
Message.update!(content_attributes)
    ↓
after_update_commit → dispatch_update_event
    ↓
MESSAGE_UPDATED 事件
    ↓
CsatSurveyListener#message_updated
    ↓
CsatSurveys::ResponseBuilder#perform
    ↓
CsatSurveyResponse.create / update
    ↓
数据持久化到 csat_survey_responses 表
```
