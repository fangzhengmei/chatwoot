# Chatwoot Helpdesk Portal 技术分析报告（v3）

## 一、单篇文章直链读取链路深度分析

### 1.1 路由定义

**文件位置**：`config/routes.rb:575-576`

```ruby
get 'hc/:slug/articles/:article_slug.png', to: 'public/api/v1/portals/articles#tracking_pixel'
get 'hc/:slug/articles/:article_slug', to: 'public/api/v1/portals/articles#show'
```

**关键发现**：
- 文章直链路由 **不包含** `:locale` 参数
- 与之对比，其他公开门户路由都包含 `:locale`：
  ```ruby
  get 'hc/:slug/:locale/articles', to: 'public/api/v1/portals/articles#index'
  get 'hc/:slug/:locale/categories', to: 'public/api/v1/portals/categories#index'
  ```

### 1.2 控制器过滤器执行顺序

**ArticlesController**（`app/controllers/public/api/v1/portals/articles_controller.rb`）：

```ruby
class Public::Api::V1::Portals::ArticlesController < Public::Api::V1::Portals::BaseController
  before_action :ensure_custom_domain_request, only: [:show, :index]
  before_action :portal
  before_action :ensure_portal_feature_enabled
  before_action :set_category, except: [:index, :show, :tracking_pixel]
  before_action :set_article, only: [:show]
  # ...
end
```

**BaseController**（`app/controllers/public/api/v1/portals/base_controller.rb`）：

```ruby
class Public::Api::V1::Portals::BaseController < PublicController
  include SwitchLocale
  
  before_action :show_plain_layout
  before_action :set_color_scheme
  before_action :set_global_config
  around_action :set_locale   # ← 关键点：around_action
  # ...
end
```

**Rails 过滤器执行顺序**：
```
1. before_action（父类）：show_plain_layout, set_color_scheme, set_global_config
2. around_action（父类）：set_locale（开始）
3. before_action（子类）：ensure_custom_domain_request, portal, ensure_portal_feature_enabled, set_article
4. action：show
5. around_action（父类）：set_locale（结束）
```

### 1.3 直链的 Locale 设置逻辑

**文件位置**：`app/controllers/public/api/v1/portals/base_controller.rb:24-49`

```ruby
def set_locale(&)
  switch_locale_with_portal(&) if params[:locale].present?  # ← 直链不满足此条件
  switch_locale_with_article(&) if params[:article_slug].present?  # ← 直链走这里
  
  yield
end

def switch_locale_with_article(&)
  article = Article.find_by(slug: params[:article_slug])  # ← 先查一次文章
  render_404 && return if article.blank?

  article_locale = if article.category.present?
                     article.category.locale  # 使用分类的 locale
                   else
                     article.portal.default_locale  # 或门户默认语言
                   end
  @locale = validate_and_get_locale(article_locale)
  I18n.with_locale(@locale, &)
end
```

### 1.4 set_article 方法（文章查询）

**文件位置**：`app/controllers/public/api/v1/portals/articles_controller.rb:63-66`

```ruby
def set_article
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  # 没有 .published 过滤！
  # 没有 .where(locale: ...) 过滤！
  @parsed_content = render_article_content(@article.content.to_s)
end
```

**对比 index 方法**（`articles_controller.rb:9-20`）：

```ruby
def index
  @search_query = list_params[:query]
  @articles = @portal.articles.published.includes(:category, :author)
  # ↑ 有 .published 过滤！
  
  @articles = @articles.where(locale: permitted_params[:locale]) if permitted_params[:locale].present?
  # ↑ 有 locale 过滤！
  # ...
end
```

## 二、直链读取的关键问题分析

### 2.1 问题一：完全绕过 status 过滤

**问题描述**：
- `set_article` 方法只按 `slug` 查询，没有任何 `status` 过滤
- `index` 方法使用 `@portal.articles.published`，但 `show` 方法没有
- **Draft 和 Archived 文章可通过直链完全访问**

**代码证据**：

```ruby
# set_article 方法 - 无状态过滤
@article = @portal.articles.find_by(slug: permitted_params[:article_slug])

# 对比 index - 有状态过滤
@articles = @portal.articles.published.includes(:category, :author)
```

**测试用例验证**（`spec/controllers/public/api/v1/portals/articles_controller_spec.rb:131-136`）：

```ruby
it 'does not increment the view count if the article is not published' do
  draft_article = create(:article, category: category, status: :draft, portal: portal, 
                         account_id: account.id, author_id: agent.id, views: 0)
  
  get "/hc/#{portal.slug}/articles/#{draft_article.slug}"
  
  expect(response).to have_http_status(:success)  # ✅ 200 OK，完全可访问！
  expect(draft_article.reload.views).to eq 0      # 只是不增加视图计数
end
```

**tracking_pixel 方法的部分防护**（`articles_controller.rb:26-39`）：

```ruby
def tracking_pixel
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  return head :not_found unless @article
  
  @article.increment_view_count if @article.published?  # 仅 published 文章计数
  # ...
end
```

### 2.2 问题二：绕过 Portal 级别的 draft_locales 过滤

**问题描述**：
- 文章直链路由没有 `:locale` 参数
- Locale 由文章本身决定（`switch_locale_with_article`）
- 即使文章语言在 `draft_locales` 中，只要知道 slug 就能访问

**对比门户首页的保护**（`_header.html.erb:81-99`）：

```erb
<% if @portal.public_locale_codes.length > 1 %>
  <%# 语言切换器只显示 public_locale_codes %>
  <% @portal.public_locale_codes.each do |locale| %>
    <option value="<%= locale %>">...</option>
  <% end %>
<% end %>
```

### 2.3 问题三：绕过 locale 查询过滤

**问题描述**：
- 列表页使用 `@articles.where(locale: permitted_params[:locale])` 限制语言
- 直链 `set_article` 没有 locale 过滤，因为 slug 是**全局唯一**的

**数据库索引**（`db/schema.rb`）：

```ruby
add_index :articles, [:slug], unique: true  # slug 全局唯一
```

**对比 Category 的 locale 过滤**（`categories_controller.rb:68-75`）：

```ruby
def set_category
  @category = @portal.categories.find_by!(
    slug: permitted_params[:category_slug],
    locale: permitted_params[:locale]  # ← 有 locale 过滤
  )
end
```

## 三、Invalid Locale 参数与 I18n 不一致分析

### 3.1 validate_and_get_locale 方法

**文件位置**：`app/controllers/concerns/switch_locale.rb:59-72`

```ruby
def validate_and_get_locale(locale)
  return I18n.default_locale.to_s if locale.blank?

  available_locales = I18n.available_locales.map(&:to_s)
  locale_without_variant = locale.split('_')[0]

  if available_locales.include?(locale)
    locale
  elsif available_locales.include?(locale_without_variant)
    locale_without_variant
  else
    I18n.default_locale.to_s  # ← 无效 locale 回退到默认语言
  end
end
```

**验证逻辑**：
1. 如果 `locale` 在 `I18n.available_locales` 中 → 直接使用
2. 如果 `locale_without_variant`（如 `zh` 从 `zh-CN`）在 available_locales 中 → 使用变体
3. 否则 → **回退到 `I18n.default_locale`**（通常是 `'en'`）

### 3.2 直链场景下的不一致分析

对于直链 `/hc/:slug/articles/:article_slug`：

```
请求流程：
1. 路由没有 :locale 参数 → params[:locale] = nil
2. 不执行 switch_locale_with_portal
3. 执行 switch_locale_with_article：
   ├── 查询文章（不检查 status/locale）
   ├── 根据文章分类或门户确定 article_locale
   ├── validate_and_get_locale(article_locale)
   └── 设置 I18n.with_locale

4. 执行 set_article：
   └── 按 slug 查询文章（无过滤）

5. 渲染视图：
   ├── 文章内容：来自 @article（文章实际的语言）
   ├── I18n.t() 翻译：来自 I18n.locale（validate 后的语言）
```

**潜在不一致场景**：

| 场景 | 文章 locale | 验证结果 | I18n.locale | 行为 |
|------|-------------|----------|-------------|------|
| 正常情况 | 'es' | valid | 'es' | 一致 |
| 门户配置了未安装的语言 | 'xx-XX' | invalid | 'en' | **不一致** |

**不一致的具体表现**：
- 文章标题/内容：显示原文（西班牙文或自定义语言）
- UI 翻译（`I18n.t('public_portal.common.home')`）：显示英文（default_locale）
- 语言切换器：可能显示错误的当前语言

### 3.3 带 locale 参数路由的不一致分析

对于带 locale 参数的路由 `/hc/:slug/:locale/articles`：

```ruby
def switch_locale_with_portal(&)
  @locale = validate_and_get_locale(params[:locale])  # 验证并可能回退
  I18n.with_locale(@locale, &)
end
```

**潜在问题**：
```
请求：/hc/my-portal/zh-Hant/articles（假设 zh-Hant 不在 I18n.available_locales 中）

实际行为：
1. validate_and_get_locale('zh-Hant') → 'zh'（如果 'zh' 有效）或 'en'
2. 设置 I18n.locale = 'zh'
3. 但 params[:locale] 仍然是 'zh-Hant'（未修改）
4. 文章查询：where(locale: 'zh-Hant') ← 使用原始参数！
```

**代码证据**（`articles_controller.rb:13`）：

```ruby
@articles = @articles.where(locale: permitted_params[:locale]) if permitted_params[:locale].present?
# 使用的是 params[:locale] = 'zh-Hant'，不是 @locale = 'zh'
```

**可能的结果**：
- I18n.locale = 'zh' → UI 翻译是中文
- 文章查询：`where(locale: 'zh-Hant')` → 可能查不到任何文章
- **用户看到的是中文 UI，但文章列表为空**

### 3.4 不一致影响总结

| 影响维度 | 描述 |
|----------|------|
| **UI 翻译不一致** | 文章内容是一种语言，页面 UI 是另一种语言 |
| **文章查询为空** | I18n.locale 已回退，但文章查询仍使用原始无效 locale |
| **语言切换器混乱** | 当前语言显示可能与实际 UI 不一致 |
| **SEO 问题** | meta 标签使用的 locale 与实际内容可能不匹配 |

## 四、完整的读取链路对比

### 4.1 列表页读取链路（带 locale 参数）

```
GET /hc/:slug/:locale/articles

1. around_action :set_locale
   └── switch_locale_with_portal
       ├── @locale = validate_and_get_locale(params[:locale])
       └── I18n.with_locale(@locale)

2. before_action :portal
3. before_action :set_category
   └── @portal.categories.find_by!(slug: ..., locale: params[:locale])
       ↑ 使用 params[:locale]，可能与 @locale 不一致

4. index action
   └── @articles = @portal.articles.published.where(locale: params[:locale])
       ↑ 有 .published 过滤，但 locale 用 params[:locale]
```

### 4.2 直链读取链路（无 locale 参数）

```
GET /hc/:slug/articles/:article_slug

1. around_action :set_locale
   └── switch_locale_with_article
       ├── article = Article.find_by(slug: ...)  ← 不检查 status！
       ├── article_locale = article.category.locale || portal.default_locale
       ├── @locale = validate_and_get_locale(article_locale)
       └── I18n.with_locale(@locale)

2. before_action :portal
3. before_action :set_article
   └── @article = @portal.articles.find_by(slug: ...)
       ↑ 没有 .published！
       ↑ 没有 locale 过滤！

4. show action - 无额外检查
```

### 4.3 过滤行为对比表

| 过滤维度 | 列表页 | 文章直链 |
|----------|--------|----------|
| **Article.status** | ✅ `.published` 过滤 | ❌ 无过滤 |
| **Locale 查询** | ✅ `where(locale: params[:locale])` | ❌ 无过滤（slug 全局唯一） |
| **Portal.draft_locales** | ⚠️ 语言切换器隐藏，但 URL 可访问 | ⚠️ 完全绕过 |
| **I18n.locale 一致性** | ⚠️ 可能不一致（见 3.3） | ⚠️ 取决于文章 locale 有效性 |

## 五、关键发现总结

### 5.1 已确认的绕过问题

| 问题 | 严重程度 | 代码位置 |
|------|----------|----------|
| Draft/Archived 文章直链可访问 | 高 | `articles_controller.rb:63-66` |
| 直链绕过 locale 查询过滤 | 中 | 同上 |
| 直链绕过 draft_locales 限制 | 中 | 同上 |
| Invalid locale 可能导致 I18n 与查询不一致 | 中 | `switch_locale.rb:59-72` |

### 5.2 设计意图推测

从测试用例来看，部分行为可能是**有意设计**：

```ruby
it 'does not increment the view count if the article is not published' do
  # 验证的是"不增加视图计数"，而不是"404"
  expect(response).to have_http_status(:success)
end
```

这暗示：
- Draft 文章可访问可能是有意的（例如预览链接）
- 但缺少明确的认证/授权检查
- 且没有与 `draft_locales` 机制配合

### 5.3 视图层的 locale 依赖

**文件位置**：`app/views/public/api/v1/portals/show.html.erb:5,9,15`

```erb
<%# 首页视图依赖 @locale 进行过滤 %>
@portal.categories.where(locale: @locale)...
@portal.articles.where(..., locale: @locale)...
```

**文件位置**：`_header.html.erb:89,92-93`

```erb
<% if @portal.draft_locale?(@locale) %>
  <option selected disabled value="<%= @locale %>">...</option>
<% end %>
<% @portal.public_locale_codes.each do |locale| %>
  ...
<% end %>
```

**文件位置**：`_article_header.html.erb:13`

```erb
href="<%= generate_home_link(@portal.slug, @article.category&.locale, ...) %>"
```

注意：面包屑导航使用 `@article.category&.locale`（文章实际语言），而不是 `@locale`（I18n 当前语言）。

## 六、关键文件索引（v3 新增）

| 功能 | 文件路径 |
|------|----------|
| 公开 Articles 控制器 | `app/controllers/public/api/v1/portals/articles_controller.rb` |
| 公开 Base 控制器 | `app/controllers/public/api/v1/portals/base_controller.rb` |
| SwitchLocale Concern | `app/controllers/concerns/switch_locale.rb` |
| 路由配置 | `config/routes.rb` |
| 文章页视图 | `app/views/public/api/v1/portals/articles/show.html.erb` |
| 文章头部视图 | `app/views/public/api/v1/portals/articles/_article_header.html.erb` |
| 公开文章测试 | `spec/controllers/public/api/v1/portals/articles_controller_spec.rb` |

---

## 七、v2 到 v3 的新增内容

1. **直链读取链路深度分析**：
   - ✅ 确认了 `/hc/:slug/articles/:article_slug` 路由**不包含** `:locale` 参数
   - ✅ 分析了 around_action 与 before_action 的执行顺序
   - ✅ 确认 `set_article` 方法**完全绕过** `status` 和 `locale` 过滤

2. **Draft/Archived 访问确认**：
   - ✅ 测试用例显式验证 draft 文章直链返回 200 OK
   - ✅ 只有 `tracking_pixel` 检查了 `@article.published?`（仅用于视图计数）
   - ✅ `show` 动作没有任何状态检查

3. **Invalid Locale 不一致分析**：
   - ✅ `validate_and_get_locale` 会将无效 locale 回退到 `I18n.default_locale`
   - ✅ 但 `params[:locale]` 不会被修改，导致后续查询可能使用无效值
   - ✅ 分析了带 locale 参数和不带 locale 参数两种场景的不一致
   - ✅ 列出了 UI 翻译、文章查询、SEO 等多个受影响维度

4. **完整链路对比**：
   - ✅ 绘制了列表页和直链的完整读取流程图
   - ✅ 对比表清晰展示各过滤维度的行为差异
