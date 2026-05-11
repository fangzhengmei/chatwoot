# Chatwoot Helpdesk Portal 技术分析报告（v2）

## 一、概述

Chatwoot 的帮助中心（Help Center）采用**统一数据模型 + 双路访问控制器**的架构：
- **管理侧**（Admin Panel）：完整 CRUD 权限，可编辑所有状态和语言版本
- **公开门户**（Public Portal）：只读访问，有多层过滤逻辑（status + draft_locales）

两者共享同一份数据库表（`portals` / `categories` / `articles`），差异在控制器层、序列化层和视图层。

---

## 二、草稿语言在公开门户的访问行为

### 2.1 核心概念

Portal 的 `config` 字段中有三个关键语言配置：

```ruby
# app/models/portal.rb:67-83
def allowed_locale_codes
  # 门户允许的所有语言
end

def draft_locale_codes
  # 被标记为"草稿"的语言（不会在语言切换菜单中显示）
end

def public_locale_codes
  allowed_locale_codes - draft_locale_codes  # 对外公开的语言
end
```

### 2.2 语言切换菜单的行为

**文件位置**：`app/views/public/api/v1/portals/_header.html.erb:81-99`

```erb
<%# Locale switcher section %>
<% if @portal.public_locale_codes.length > 1 %>
  <select class="locale-switcher">
    <%# 如果当前在草稿语言中，先添加一个 disabled 的当前语言选项 %>
    <% if @portal.draft_locale?(@locale) %>
      <option selected disabled value="<%= @locale %>">
        <%= "#{language_name(@locale)} (#{@locale})" %>
      </option>
    <% end %>
    <%# 只遍历 public_locale_codes（排除 draft_locales） %>
    <% @portal.public_locale_codes.each do |locale| %>
      <option <%= locale == @locale ? 'selected': '' %> value="<%= locale %>">
        <%= "#{language_name(locale)} (#{locale})" %>
      </option>
    <% end %>
  </select>
<% end %>
```

**行为总结**：
| 场景 | 语言切换菜单行为 |
|------|------------------|
| 当前在公开语言（public_locale） | 只显示 `public_locale_codes`，不包含 draft 语言 |
| 当前在草稿语言（draft_locale） | 显示当前 draft 语言（disabled） + 其他 public_locales |
| 只有一个公开语言 | 不显示语言切换器（条件 `public_locale_codes.length > 1`） |

### 2.3 直接 URL 访问的行为

**路由**：`/hc/:portal_slug/:locale/...`

**测试用例验证**（`spec/controllers/public/api/v1/portals_controller_spec.rb:76-82`）：

```ruby
it 'allows direct access to drafted locale pages' do
  portal.update!(config: { 
    allowed_locales: %w[en es], 
    draft_locales: ['es'],  # es 是草稿语言
    default_locale: 'en' 
  })

  get "/hc/#{portal.slug}/es"  # 直接访问草稿语言

  expect(response).to have_http_status(:success)  # ✅ 可以访问！
end
```

**结论**：
- **语言切换菜单**：草稿语言被隐藏（除非当前就在该语言页面）
- **直接 URL 访问**：草稿语言**完全可访问**，没有任何拦截

### 2.4 测试用例完整覆盖

**文件位置**：`spec/controllers/public/api/v1/portals_controller_spec.rb:66-106`

| 测试用例 | 期望行为 | 验证结果 |
|----------|----------|----------|
| `hides drafted locales from the public locale switcher` | 草稿语言不出现在切换器选项中 | ✅ `response.body.not_to include('value="es"')` |
| `allows direct access to drafted locale pages` | 直接 URL 可访问草稿语言 | ✅ `http_status(:success)` |
| `shows the active drafted locale in the switcher state` | 已在草稿语言时，切换器显示 disabled 的当前语言 | ✅ `selected && disabled` 同时存在 |

---

## 三、Locale 回退优先级

### 3.1 公开门户的 Locale 逻辑

**文件位置**：`app/controllers/public/api/v1/portals/base_controller.rb`

公开门户使用**独立的 locale 切换逻辑**（不使用通用的 `SwitchLocale` concern）：

```ruby
# base_controller.rb:24-49
around_action :set_locale

def set_locale(&)
  switch_locale_with_portal(&) if params[:locale].present?
  switch_locale_with_article(&) if params[:article_slug].present?
  yield
end

def switch_locale_with_portal(&)
  @locale = validate_and_get_locale(params[:locale])  # 直接使用 URL 路由参数
  I18n.with_locale(@locale, &)
end

def switch_locale_with_article(&)
  article = Article.find_by(slug: params[:article_slug])
  
  article_locale = if article.category.present?
                     article.category.locale  # 优先使用文章所在分类的语言
                   else
                     article.portal.default_locale  # 否则使用门户默认语言
                   end
  @locale = validate_and_get_locale(article_locale)
  I18n.with_locale(@locale, &)
end
```

### 3.2 公开门户的 Locale 优先级

**优先级从高到低**：

```
1. URL 路由参数 (:locale)
   └── /hc/:slug/:locale/... 路径中的 :locale 段

2. 文章 slug 推断（仅在访问 /hc/:slug/articles/:article_slug 时）
   ├── 2a. 文章所属分类的 locale（category.locale）
   └── 2b. 门户默认语言（portal.default_locale）

3. 无语言时的重定向（portals_controller.rb:25-30）
   └── /hc/:slug → 重定向到 /hc/:slug/:default_locale
```

**关键点**：
- 公开门户 **不使用** 用户设置（`@user.ui_settings['locale']`）
- 公开门户 **不使用** 通用 SwitchLocale concern 中的自定义域名回退
- 自定义域名仅用于**查找门户**（`locale_from_custom_domain`），但不用于设置 I18n.locale

### 3.3 通用 SwitchLocale Concern（管理侧和其他模块使用）

**文件位置**：`app/controllers/concerns/switch_locale.rb:6-79`

```ruby
def switch_locale(&)
  # Priority is for locale set in query string (mostly for widget/from js sdk)
  locale ||= params[:locale]                    # 1. 路由/查询参数
  
  # Use the user's locale if available
  locale ||= locale_from_user                   # 2. 用户设置（@user.ui_settings['locale']）
  
  # Use the locale from a custom domain if applicable
  locale ||= locale_from_custom_domain          # 3. 自定义域名绑定的门户默认语言
  
  # if locale is not set in account, let's use DEFAULT_LOCALE env variable
  locale ||= ENV.fetch('DEFAULT_LOCALE', nil)   # 4. 环境变量
  
  set_locale(locale, &)
end

def locale_from_custom_domain(&)
  domain = request.host
  return if DomainHelper.chatwoot_domain?(domain)
  
  @portal = Portal.find_by(custom_domain: domain)
  return unless @portal
  
  @portal.default_locale  # 使用自定义域名绑定的门户默认语言
end
```

### 3.4 管理侧的 Locale 逻辑

**文件位置**：`app/controllers/api/v1/accounts/base_controller.rb:5`

```ruby
around_action :switch_locale_using_account_locale
```

**switch_locale_using_account_locale**（switch_locale.rb:22-30）：
```ruby
def switch_locale_using_account_locale(&)
  locale = locale_from_user                    # 1. 用户设置
  locale ||= locale_from_account(@current_account)  # 2. 账户设置
  set_locale(locale, &)
end
```

### 3.5 Locale 优先级对比表

| 优先级 | 通用 SwitchLocale | 管理侧 API | 公开门户 |
|--------|-------------------|-----------|----------|
| 1 | `params[:locale]`（查询/路由） | 用户设置 | URL 路由参数 `:locale` |
| 2 | 用户设置 | 账户设置 | 文章分类 locale（仅文章页） |
| 3 | 自定义域名默认语言 | - | 门户默认语言（仅文章页） |
| 4 | `ENV['DEFAULT_LOCALE']` | - | 自动重定向（仅门户首页） |

---

## 四、管理侧写入与公开侧读取的完整链路

### 4.1 共享的数据层

**数据库表**（`db/schema.rb`）：

```ruby
# portals 表
create_table "portals" do |t|
  t.integer "account_id", null: false
  t.string "name", null: false
  t.string "slug", null: false
  t.string "custom_domain"
  t.jsonb "config", default: {"allowed_locales" => ["en"]}
  # config 包含: allowed_locales, default_locale, draft_locales
end

# categories 表
create_table "categories" do |t|
  t.integer "account_id", null: false
  t.integer "portal_id", null: false
  t.string "locale", default: "en"
  t.string "slug", null: false
  t.bigint "associated_category_id"  # 多语言关联
end

# articles 表
create_table "articles" do |t|
  t.integer "account_id", null: false
  t.integer "portal_id", null: false
  t.integer "category_id"
  t.string "locale", default: "en"
  t.string "slug", null: false
  t.integer "status"  # draft(0), published(1), archived(2)
  t.bigint "associated_article_id"  # 多语言关联
  t.jsonb "meta", default: {}
end
```

### 4.2 管理侧写入链路

**路由**：`/api/v1/accounts/:account_id/portals/:portal_slug/...`

#### 4.2.1 控制器层

**Portal 控制器**（`app/controllers/api/v1/accounts/portals_controller.rb`）：
```ruby
before_action :fetch_portal, except: [:index, :create]
before_action :check_authorization

def create
  @portal = Current.account.portals.build(portal_params)
  @portal.save!
end

def update
  @portal.update!(portal_params)
end

def portal_params
  params.require(:portal).permit(
    :name, :slug, :custom_domain,
    { config: [:default_locale, { allowed_locales: [] }, { draft_locales: [] }] }
  )
end
```

**Article 控制器**（`app/controllers/api/v1/accounts/articles_controller.rb`）：
```ruby
before_action :portal
before_action :check_authorization
before_action :fetch_article, except: [:index, :create, :reorder]

def index
  @portal_articles = @portal.articles
  # 无状态过滤 - 可查看所有 status
  @articles = @articles.search(list_params)
end

def create
  @article = @portal.articles.create!(article_params)
  @article.associate_root_article(article_params[:associated_article_id])
end

def article_params
  params.require(:article).permit(
    :title, :slug, :content, :description, :status,
    :locale, :category_id, :associated_article_id,
    meta: [:title, :description, { tags: [] }]
  )
end
```

#### 4.2.2 序列化层（管理侧）

**Portal 序列化**（`app/views/api/v1/accounts/portals/_portal.json.jbuilder`）：
```ruby
json.config do
  json.allowed_locales do
    json.array! portal.allowed_locale_codes.each do |locale|
      json.partial! 'api/v1/models/portal_config', 
                    locale: locale, portal: portal
    end
  end
end

json.meta do
  json.all_articles_count articles.try(:size)
  json.archived_articles_count articles.try(:archived).try(:size)
  json.published_count articles.try(:published).try(:size)
  json.draft_articles_count articles.try(:draft).try(:size)
  json.default_locale portal.default_locale
end
```

**Portal Config 序列化**（`app/views/api/v1/models/_portal_config.json.jbuilder`）：
```ruby
json.code locale
json.articles_count portal.articles.search({ locale: locale }).size
json.categories_count portal.categories.search_by_locale(locale).size
json.draft portal.draft_locale?(locale)  # 管理侧可看到 draft 标记
```

**Article 序列化**（`app/views/api/v1/accounts/articles/_article.json.jbuilder`）：
```ruby
json.id article.id
json.slug article.slug
json.title article.title
json.content article.content        # 完整内容
json.status article.status          # 所有状态
json.associated_articles do ... end # 关联文章
json.views article.views
```

### 4.3 公开侧读取链路

**路由**：`/hc/:portal_slug/:locale/...`

#### 4.3.1 控制器层

**Portal 控制器**（`app/controllers/public/api/v1/portals_controller.rb`）：
```ruby
before_action :ensure_custom_domain_request, only: [:show]
before_action :redirect_to_portal_with_locale, only: [:show]
before_action :portal
before_action :ensure_portal_feature_enabled

def show
  # 无特殊过滤，由视图层处理
end

def redirect_to_portal_with_locale
  return if params[:locale].present?
  redirect_to "/hc/#{@portal.slug}/#{@portal.default_locale}"
end
```

**Articles 控制器**（`app/controllers/public/api/v1/portals/articles_controller.rb`）：
```ruby
before_action :ensure_custom_domain_request, only: [:show, :index]
before_action :portal
before_action :ensure_portal_feature_enabled
before_action :set_category, except: [:index, :show, :tracking_pixel]
before_action :set_article, only: [:show]

def index
  @search_query = list_params[:query]
  @articles = @portal.articles.published.includes(:category, :author)
  # ↑ 关键：只查询 published 状态的文章
  
  @articles = @articles.where(locale: permitted_params[:locale]) if permitted_params[:locale].present?
  # ↑ 按语言过滤
  
  search_articles
  order_by_sort_param
  limit_results
end

def show
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  # 注意：这里没有 status 过滤！
  # 但实际测试表明 tracking_pixel 方法检查了 published
end

def tracking_pixel
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  @article.increment_view_count if @article.published?  # 仅 published 计数
end
```

**Categories 控制器**（`app/controllers/public/api/v1/portals/categories_controller.rb`）：
```ruby
def set_category
  @category = @portal.categories.find_by!(
    locale: params[:locale],
    slug: params[:category_slug]
  )
  # 按 locale 和 slug 查找，但无 draft 检查
end
```

#### 4.3.2 视图层过滤

**Portal 首页视图**（`app/views/public/api/v1/portals/show.html.erb:9-16`）：
```erb
<%# 只显示有 published 文章的分类 %>
<% @portal.categories.where(locale: @locale)
  .joins(:articles)
  .where(articles:{ status: :published })  # 关键：只显示有 published 文章的分类
  .order(position: :asc)
  .group('categories.id').each do |category| %>
  <%= render "category-block", category: category %>
<% end %>

<%# 未分类文章也只显示 published %>
<% if @portal.articles.where(status: :published, category_id: nil, locale: @locale).count > 0 %>
  <%= render "uncategorized-block" %>
<% end %>
```

#### 4.3.3 序列化层（公开侧）

**Portal 序列化**（`app/views/public/api/v1/models/hc/_portal.json.jbuilder`）：
```ruby
# 公开侧的 portal 序列化不包含 draft_locales 等内部信息
```

**Article 序列化**（`app/views/public/api/v1/models/_article.json.jbuilder`）：
```ruby
json.id article.id
json.title article.title
json.content article.content
json.status article.status  # 仍返回 status，但控制器已过滤
json.link "hc/#{article.portal.slug}/articles/#{article.slug}"
```

### 4.4 完整链路对比

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         数据库层（共享）                                  │
│   portals / categories / articles 表（同一份数据）                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                                 ▼
┌─────────────────────────────┐    ┌─────────────────────────────────────┐
│      管理侧写入链路          │    │         公开侧读取链路                │
├─────────────────────────────┤    ├─────────────────────────────────────┤
│ 路由:                        │    │ 路由:                              │
│ /api/v1/accounts/:id/...    │    │ /hc/:slug/:locale/...               │
├─────────────────────────────┤    ├─────────────────────────────────────┤
│ 控制器层过滤:                │    │ 控制器层过滤:                       │
│ ┌─────────────────────────┐ │    │ ┌─────────────────────────────────┐ │
│ │ - 权限检查               │ │    │ │ - 自定义域名验证                 │ │
│ │ - 无 status 过滤         │ │    │ │ - articles.published (仅列表)   │ │
│ │ - 无 draft_locale 过滤   │ │    │ │ - 无单篇文章的 status 检查      │ │
│ └─────────────────────────┘ │    │ └─────────────────────────────────┘ │
├─────────────────────────────┤    ├─────────────────────────────────────┤
│ 视图层:                     │    │ 视图层过滤:                         │
│ ┌─────────────────────────┐ │    │ ┌─────────────────────────────────┐ │
│ │ - Vue SPA (dashboard)   │ │    │ │ - Rails ERB (服务端渲染)        │ │
│ │ - 显示所有 status        │ │    │ │ - 仅显示有 published 文章的分类  │ │
│ │ - 显示 draft 标记        │ │    │ │ - 语言切换器只显示 public_locales│ │
│ └─────────────────────────┘ │    │ └─────────────────────────────────┘ │
├─────────────────────────────┤    ├─────────────────────────────────────┤
│ 序列化层:                   │    │ 序列化层:                           │
│ ┌─────────────────────────┐ │    │ ┌─────────────────────────────────┐ │
│ │ - _portal_config.json   │ │    │ │ - 不包含 draft_locales 元数据    │ │
│ │   包含 draft 标记        │ │    │ │ - 不包含 articles_count 等统计  │ │
│ │ - 包含完整统计数据        │ │    │ └─────────────────────────────────┘ │
│ └─────────────────────────┘ │    │                                     │
└─────────────────────────────┘    └─────────────────────────────────────┘
```

### 4.5 关键过滤点总结

| 过滤维度 | 管理侧 | 公开门户 |
|----------|--------|----------|
| **权限** | 需要登录 + 权限检查 | 无 |
| **Article.status** | 无过滤（全部可见） | 列表页：仅 `published`；单篇：无检查 |
| **Portal.draft_locales** | 可见（含 `draft: true` 标记） | 语言切换器隐藏，但 URL 可直接访问 |
| **数据完整性** | 完整内容 + 关联 + 统计 | 仅必要字段 |
| **操作权限** | CRUD | 只读 GET |

---

## 五、多语言版本管理机制（补充）

### 5.1 根文章 + 关联文章模式

```ruby
# app/models/article.rb:38-48
has_many :associated_articles,
         class_name: :Article,
         foreign_key: :associated_article_id,
         inverse_of: 'root_article'

belongs_to :root_article,
           class_name: :Article,
           foreign_key: :associated_article_id,
           inverse_of: :associated_articles,
           optional: true
```

### 5.2 关联扁平化确保

```ruby
# app/models/article.rb:110-123
def associate_root_article(associated_article_id)
  article = portal.articles.find(associated_article_id) if associated_article_id.present?
  return if article.nil?
  
  root_article_id = self.class.find_root_article_id(article)
  update(associated_article_id: root_article_id) if root_article_id.present?
end

def self.find_root_article_id(article)
  article.associated_article_id || article.id  # 始终指向最顶层
end
```

### 5.3 企业版自动翻译

**文件位置**：`enterprise/app/jobs/captain/articles/translate_job.rb:37-53`

```ruby
def find_existing_translation(target_locale)
  root_id = Article.find_root_article_id(@source_article)
  @source_article.portal.articles.find_by(
    associated_article_id: root_id,  # 通过根文章查找
    locale: target_locale
  )
end

def create_translated_article(...)
  @source_article.portal.articles.create!(
    ...
    associated_article_id: Article.find_root_article_id(@source_article),
    status: :draft  # 翻译后默认为草稿
  )
end
```

---

## 六、关键文件索引

| 功能 | 文件路径 |
|------|----------|
| Portal 模型 | `app/models/portal.rb` |
| Category 模型 | `app/models/category.rb` |
| Article 模型 | `app/models/article.rb` |
| SwitchLocale Concern | `app/controllers/concerns/switch_locale.rb` |
| 公开门户 Base 控制器 | `app/controllers/public/api/v1/portals/base_controller.rb` |
| 公开门户 Portal 控制器 | `app/controllers/public/api/v1/portals_controller.rb` |
| 公开门户 Articles 控制器 | `app/controllers/public/api/v1/portals/articles_controller.rb` |
| 管理侧 Portal 控制器 | `app/controllers/api/v1/accounts/portals_controller.rb` |
| 管理侧 Articles 控制器 | `app/controllers/api/v1/accounts/articles_controller.rb` |
| 管理侧 Base 控制器 | `app/controllers/api/v1/accounts/base_controller.rb` |
| 公开门户头部视图（含语言切换器） | `app/views/public/api/v1/portals/_header.html.erb` |
| 公开门户首页视图 | `app/views/public/api/v1/portals/show.html.erb` |
| 管理侧 Portal 序列化 | `app/views/api/v1/accounts/portals/_portal.json.jbuilder` |
| 管理侧 Article 序列化 | `app/views/api/v1/accounts/articles/_article.json.jbuilder` |
| 公开侧 Article 序列化 | `app/views/public/api/v1/models/_article.json.jbuilder` |
| 草稿语言测试 | `spec/controllers/public/api/v1/portals_controller_spec.rb` |

---

## 七、v1 到 v2 的修正内容

1. **草稿语言访问行为**：
   - ✅ 明确区分了语言切换菜单（隐藏 draft 语言）与直接 URL（可访问 draft 语言）
   - ✅ 添加了测试用例验证
   - ✅ 补充了视图代码分析（`_header.html.erb`）

2. **Locale 回退优先级**：
   - ✅ 明确了公开门户使用独立的 `set_locale` 逻辑
   - ✅ 区分了通用 SwitchLocale 与公开门户的不同优先级
   - ✅ 列出了完整的优先级对比表

3. **完整数据链路**：
   - ✅ 补充了数据库 schema 定义
   - ✅ 分析了控制器层的过滤差异
   - ✅ 对比了管理侧与公开侧的序列化层（jbuilder）
   - ✅ 补充了视图层的过滤逻辑（ERB 模板中的 `where(status: :published)`）
