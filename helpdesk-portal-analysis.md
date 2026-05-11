# Chatwoot Helpdesk Portal 技术分析报告

## 一、概述

Chatwoot 的帮助中心（Help Center）采用了一个统一的数据模型架构，支持多语言版本管理，并通过不同的控制器和路由为**管理后台**和**公开门户**提供访问同一份内容的能力。

## 二、核心数据模型

### 2.1 Portal（门户）模型

**文件位置**：`app/models/portal.rb`

每个帮助中心是一个 Portal，包含以下关键配置：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 门户名称 |
| `slug` | string | 唯一标识（用于URL） |
| `custom_domain` | string | 自定义域名 |
| `config` | jsonb | 多语言配置 |

**config 中的多语言配置字段**：
- `allowed_locales`: 允许的语言列表
- `default_locale`: 默认语言
- `draft_locales`: 处于草稿状态的语言（不会在公开门户显示）

**关键方法**：
```ruby
def default_locale
  config_value('default_locale').presence || allowed_locale_codes.first || 'en'
end

def allowed_locale_codes
  # 返回允许的语言代码数组
end

def draft_locale_codes
  # 哪些语言版本是草稿状态
end

def public_locale_codes
  allowed_locale_codes - draft_locale_codes  # 对外公开的语言
end

def draft_locale?(locale)
  draft_locale_codes.include?(locale)
end
```

### 2.2 Category（分类）模型

**文件位置**：`app/models/category.rb`

分类的多语言关联机制：

| 字段 | 类型 | 说明 |
|------|------|------|
| `locale` | string | 当前分类的语言代码 |
| `associated_category_id` | bigint | 关联的根分类ID（用于多语言映射） |
| `parent_category_id` | bigint | 父分类（用于层级） |
| `slug` | string | 分类slug（与locale和portal_id联合唯一） |

**关键索引**：
```ruby
# 联合唯一索引：同一portal下，同一locale内slug唯一
add_index :categories, [:slug, :locale, :portal_id], unique: true
```

**关联关系**：
```ruby
has_many :associated_categories,
         class_name: :Category,
         foreign_key: :associated_category_id,
         dependent: :nullify,
         inverse_of: 'root_category'

belongs_to :root_category,
           class_name: :Category,
           foreign_key: :associated_category_id,
           inverse_of: :associated_categories,
           optional: true
```

**语言验证**：
```ruby
validate :allowed_locales

def allowed_locales
  allowed_locales = portal.allowed_locale_codes
  return true if allowed_locales.include?(locale)
  errors.add(:locale, "#{locale} of category is not part of portal's #{allowed_locales}.")
end
```

### 2.3 Article（文章）模型

**文件位置**：`app/models/article.rb`

文章的多语言关联机制：

| 字段 | 类型 | 说明 |
|------|------|------|
| `locale` | string | 当前文章的语言代码 |
| `associated_article_id` | bigint | 关联的根文章ID（用于多语言映射） |
| `slug` | string | 文章slug（全局唯一） |
| `status` | integer | 状态：draft(0), published(1), archived(2) |
| `category_id` | integer | 所属分类 |
| `portal_id` | integer | 所属门户 |

**关联关系**：
```ruby
has_many :associated_articles,
         class_name: :Article,
         foreign_key: :associated_article_id,
         dependent: :nullify,
         inverse_of: 'root_article'

belongs_to :root_article,
           class_name: :Article,
           foreign_key: :associated_article_id,
           inverse_of: :associated_articles,
           optional: true
```

**关键方法 - 多语言关联**：

```ruby
# 确保总是关联到根文章，避免深层嵌套
def associate_root_article(associated_article_id)
  article = portal.articles.find(associated_article_id) if associated_article_id.present?
  return if article.nil?
  
  root_article_id = self.class.find_root_article_id(article)
  update(associated_article_id: root_article_id) if root_article_id.present?
end

# 始终获取最顶层的根文章ID
def self.find_root_article_id(article)
  article.associated_article_id || article.id
end
```

**locale 自动设置**：
```ruby
before_validation :ensure_locale_in_article

def ensure_locale_in_article
  self.locale = if category.present?
                  category.locale  # 继承分类的语言
                else
                  locale.presence || portal.default_locale  # 或使用门户默认语言
                end
end
```

**查询作用域**：
```ruby
scope :search_by_locale, ->(locale) { where(locale: locale) if locale.present? }
scope :search_by_status, ->(status) { where(status: status) if status.present? }
```

## 三、多语言版本管理机制

### 3.1 多语言映射原理

Chatwoot 采用**"根文章 + 关联文章"**的模式管理多语言：

```
root_article (en, id=1)
├── associated_article (zh-CN, id=2, associated_article_id=1)
├── associated_article (ja, id=3, associated_article_id=1)
└── associated_article (es, id=4, associated_article_id=1)
```

### 3.2 语言关联的层级扁平化

系统设计确保不会出现链式关联（如 A→B→C），而是始终扁平化到根文章：

```ruby
# 错误的关联方式（避免）
Article A (id=1) -> Article B (id=2, associated=1) -> Article C (id=3, associated=2)

# 正确的关联方式（实际实现）
Article A (id=1, root) <- Article B (id=2, associated=1)
                       <- Article C (id=3, associated=1)
```

### 3.3 企业版 - AI 自动翻译

**文件位置**：`enterprise/app/jobs/captain/articles/translate_job.rb`

企业版提供了基于 LLM 的自动翻译功能：

```ruby
def find_existing_translation(target_locale)
  root_id = Article.find_root_article_id(@source_article)
  @source_article.portal.articles.find_by(
    associated_article_id: root_id, 
    locale: target_locale
  )
end

def create_translated_article(...)
  @source_article.portal.articles.create!(
    ...
    associated_article_id: Article.find_root_article_id(@source_article),
    status: :draft  # 翻译后默认是草稿状态
  )
end
```

**翻译流程**：
1. 找到源文章的根文章
2. 检查目标语言是否已有翻译（通过 root_id + locale 查询）
3. 如果已有：更新内容
4. 如果没有：创建新的翻译文章（关联到根文章，状态为 draft）

### 3.4 批量翻译

**文件位置**：`enterprise/app/controllers/enterprise/api/v1/accounts/articles/bulk_actions_controller.rb`

```ruby
def find_existing_translations
  root_ids = @articles.map { |a| Article.find_root_article_id(a) }
  @portal.articles.where(associated_article_id: root_ids, locale: @locale)
end
```

## 四、公开门户与管理侧的内容共享

### 4.1 共享的数据层

两者访问的是**同一份数据库表**：
- `portals` 表
- `categories` 表  
- `articles` 表

### 4.2 管理侧（Admin Panel）

**路由前缀**：`/api/v1/accounts/:account_id/portals/:portal_slug/`

**文件位置**：
- 控制器：`app/controllers/api/v1/accounts/portals_controller.rb`
- 控制器：`app/controllers/api/v1/accounts/articles_controller.rb`
- 控制器：`app/controllers/api/v1/accounts/categories_controller.rb`
- 前端：`app/javascript/dashboard/routes/dashboard/helpcenter/`

**管理侧 API 路由**（routes.rb:364-384）：
```ruby
resources :portals do
  member do
    patch :archive
    delete :logo
    post :send_instructions
    get :ssl_status
  end
  
  resources :categories do
    post :reorder, on: :collection
  end
  
  namespace :articles do
    resource :bulk_actions, only: [] do
      post :translate
      patch :update_status
      delete :delete_articles
    end
  end
  
  resources :articles do
    post :reorder, on: :collection
  end
end
```

**管理侧能力**：
- 可以查看所有状态（draft/published/archived）的文章
- 可以编辑所有语言版本
- 可以创建/更新/删除文章
- 可以管理门户配置（allowed_locales, default_locale, draft_locales）
- 企业版可以执行批量翻译

**管理侧查询示例**（articles_controller.rb:7-19）：
```ruby
def index
  @portal_articles = @portal.articles
  set_article_count  # 统计各状态数量
  @articles = @articles.search(list_params)
end
```

**管理侧创建文章时的关联**（articles_controller.rb:24-30）：
```ruby
def create
  params_with_defaults = article_params
  params_with_defaults[:status] ||= :draft
  @article = @portal.articles.create!(params_with_defaults)
  @article.associate_root_article(article_params[:associated_article_id])
end
```

### 4.3 公开门户（Public Portal）

**路由前缀**：`/hc/:portal_slug/:locale/`

**文件位置**：
- 控制器：`app/controllers/public/api/v1/portals_controller.rb`
- 控制器：`app/controllers/public/api/v1/portals/articles_controller.rb`
- 控制器：`app/controllers/public/api/v1/portals/categories_controller.rb`
- 视图：`app/views/public/api/v1/portals/`
- 前端：`app/javascript/portal/`

**公开门户路由**（routes.rb:568-577）：
```ruby
get 'hc/:slug', to: 'public/api/v1/portals#show'
get 'hc/:slug/sitemap.xml', to: 'public/api/v1/portals#sitemap'
get 'hc/:slug/:locale', to: 'public/api/v1/portals#show'
get 'hc/:slug/:locale/articles', to: 'public/api/v1/portals/articles#index'
get 'hc/:slug/:locale/categories', to: 'public/api/v1/portals/categories#index'
get 'hc/:slug/:locale/categories/:category_slug', to: 'public/api/v1/portals/categories#show'
get 'hc/:slug/:locale/categories/:category_slug/articles', to: 'public/api/v1/portals/articles#index'
get 'hc/:slug/articles/:article_slug.png', to: 'public/api/v1/portals/articles#tracking_pixel'
get 'hc/:slug/articles/:article_slug', to: 'public/api/v1/portals/articles#show'
```

**公开侧过滤规则**（articles_controller.rb:11）：
```ruby
def index
  @articles = @portal.articles.published.includes(:category, :author)
  # 只查询 published 状态的文章
end
```

**语言切换机制**（base_controller.rb:24-49）：
```ruby
def set_locale(&)
  switch_locale_with_portal(&) if params[:locale].present?
  switch_locale_with_article(&) if params[:article_slug].present?
  yield
end

def switch_locale_with_article(&)
  article = Article.find_by(slug: params[:article_slug])
  
  article_locale = if article.category.present?
                     article.category.locale
                   else
                     article.portal.default_locale
                   end
  @locale = validate_and_get_locale(article_locale)
  I18n.with_locale(@locale, &)
end
```

**自定义域名支持**（switch_locale.rb:34-44）：
```ruby
def locale_from_custom_domain(&)
  domain = request.host
  return if DomainHelper.chatwoot_domain?(domain)
  
  @portal = Portal.find_by(custom_domain: domain)
  return unless @portal
  
  @portal.default_locale  # 使用自定义域名绑定的门户默认语言
end
```

### 4.4 两侧差异对比

| 维度 | 管理侧 | 公开门户 |
|------|--------|----------|
| **路由** | `/api/v1/accounts/:id/portals/` | `/hc/:slug/` |
| **认证** | 需要登录 + 权限检查 | 无认证（公开访问） |
| **状态过滤** | 显示全部（draft/published/archived） | 仅显示 published |
| **语言过滤** | 全部语言（包含 draft_locales） | 仅 public_locale_codes |
| **操作权限** | CRUD 全部操作 | 仅 GET 读取 |
| **数据来源** | 同一份数据库表 | 同一份数据库表 |
| **前端技术** | Vue SPA（dashboard） | Rails ERB + Vue 组件混合 |

### 4.5 内容发布流程

```
管理侧创建文章 (status: draft)
        ↓
管理侧发布 (status: published)
        ↓
    公开门户可见
        ↓
公开门户仅能看到 published 状态 + 非 draft_locale 的内容
```

**Portal 级别的语言草稿控制**：
- `draft_locales` 配置：某些语言版本整体作为草稿，不会在公开门户显示
- 即使文章是 published 状态，如果其语言在 `draft_locales` 中，也不会公开

## 五、视图渲染机制

### 5.1 管理侧前端

**技术栈**：Vue 3 + Composition API + Vuex/Pinia

**目录结构**：
```
app/javascript/dashboard/routes/dashboard/helpcenter/
├── helpcenter.routes.js
└── pages/
    ├── PortalsIndexPage.vue
    ├── PortalsSettingsIndexPage.vue
    ├── PortalsArticlesIndexPage.vue
    ├── PortalsArticlesNewPage.vue
    └── PortalsArticlesEditPage.vue
```

**API 调用**（dashboard/api/helpCenter/portals.js）：
```javascript
// 账户作用域的 API
super('portals', { accountScoped: true });

getPortal({ portalSlug, locale }) {
  return axios.get(`${this.url}/${portalSlug}?locale=${locale}`);
}
```

### 5.2 公开门户前端

**技术栈**：Rails ERB（服务端渲染）+ 少量 Vue 组件

**目录结构**：
```
app/views/public/api/v1/portals/
├── show.html.erb           # 门户首页
├── sitemap.xml.erb         # SEO 站点地图
├── _header.html.erb        # 头部组件
├── _category-block.html.erb
├── _featured_articles.html.erb
├── categories/
│   ├── index.html.erb
│   └── show.html.erb
└── articles/
    ├── index.html.erb
    └── show.html.erb

app/javascript/portal/
├── api/
│   └── article.js          # 搜索 API
└── components/
    ├── PublicArticleSearch.vue
    ├── PublicSearchInput.vue
    ├── SearchSuggestions.vue
    └── TableOfContents.vue
```

**服务端渲染示例**（articles/show.html.erb）：
```erb
<article id="cw-article-content" class="...">
  <%= @parsed_content %>
</article>
```

**内容解析**（articles_controller.rb:87-89）：
```ruby
def render_article_content(content)
  ChatwootMarkdownRenderer.new(content).render_article
end
```

**公开门户 API 搜索**（portal/api/article.js）：
```javascript
searchArticles(portalSlug, locale, query) {
  let baseUrl = `${this.baseUrl}/hc/${portalSlug}/${locale}/articles.json?query=${query}`;
  return axios.get(baseUrl);
}
```

## 六、URL 与 SEO

### 6.1 URL 结构

**管理侧**：
```
/app/accounts/:account_id/settings/portals/:portal_slug
/app/accounts/:account_id/settings/portals/:portal_slug/articles
/app/accounts/:account_id/settings/portals/:portal_slug/articles/new
```

**公开门户**：
```
/hc/:portal_slug/                          # 自动重定向到默认语言
/hc/:portal_slug/:locale/                  # 门户首页（指定语言）
/hc/:portal_slug/:locale/categories        # 分类列表
/hc/:portal_slug/:locale/categories/:slug  # 分类详情
/hc/:portal_slug/articles/:article_slug    # 文章详情（根据文章自动判断语言）
```

### 6.2 SEO 支持

**文件位置**：`app/views/public/api/v1/portals/articles/show.html.erb`

```erb
<% content_for :head do %>
  <title><%= @article.title %> | <%= @portal.display_title %></title>
  <% if @article.meta["title"].present? %>
    <meta name="title" content="<%= @article.meta["title"] %>">
    <meta property="og:title" content="<%= @article.meta["title"] %>">
  <% end %>
  <% if @article.meta["description"].present? %>
    <meta name="description" content="<%= @article.meta["description"] %>">
  <% end %>
<% end %>
```

**站点地图**：`app/controllers/public/api/v1/portals_controller.rb#sitemap`

## 七、数据访问总结

```
                    ┌─────────────────────────┐
                    │    Database Tables      │
                    │  portals / categories   │
                    │       articles          │
                    └───────────┬─────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
     ┌──────────────────┐ ┌──────────────────┐
     │  Admin API       │ │  Public Portal   │
     │  Controller      │ │  Controller      │
     │                  │ │                  │
     │ - 权限检查        │ │ - 仅读           │
     │ - 全部状态可见    │ │ - 仅 published   │
     │ - 全部语言可见    │ │ - 非 draft_locale│
     └────────┬─────────┘ └────────┬─────────┘
              │                    │
              ▼                    ▼
     ┌──────────────────┐ ┌──────────────────┐
     │ Dashboard Vue    │ │ Rails ERB + Vue  │
     │ (管理后台)        │ │ (公开门户)        │
     └──────────────────┘ └──────────────────┘
```

## 八、关键文件索引

| 功能 | 文件路径 |
|------|----------|
| Portal 模型 | `app/models/portal.rb` |
| Category 模型 | `app/models/category.rb` |
| Article 模型 | `app/models/article.rb` |
| 管理侧 Portal 控制器 | `app/controllers/api/v1/accounts/portals_controller.rb` |
| 管理侧 Article 控制器 | `app/controllers/api/v1/accounts/articles_controller.rb` |
| 公开侧 Portal 控制器 | `app/controllers/public/api/v1/portals_controller.rb` |
| 公开侧 Article 控制器 | `app/controllers/public/api/v1/portals/articles_controller.rb` |
| 公开侧 Base 控制器 | `app/controllers/public/api/v1/portals/base_controller.rb` |
| 语言切换 Concern | `app/controllers/concerns/switch_locale.rb` |
| 路由配置 | `config/routes.rb` |
| 企业版翻译 Job | `enterprise/app/jobs/captain/articles/translate_job.rb` |
| 管理侧前端路由 | `app/javascript/dashboard/routes/dashboard/helpcenter/helpcenter.routes.js` |
| 公开侧前端 API | `app/javascript/portal/api/article.js` |
