# Chatwoot Helpdesk Portal 技术分析报告（v4）

## 一、概述

本报告专门核查 **archived 文章**在公开门户直链访问中的行为，与 draft 文章严格区分，避免证据外推。

---

## 二、现有测试覆盖情况

### 2.1 公开门户测试 - 无 archived 覆盖

**文件位置**：`spec/controllers/public/api/v1/portals/articles_controller_spec.rb`

公开门户测试只覆盖了 **draft 文章**：

```ruby
# 第 131-136 行 - 只有 draft 测试
it 'does not increment the view count if the article is not published' do
  draft_article = create(:article, category: category, status: :draft, ...)
  get "/hc/#{portal.slug}/articles/#{draft_article.slug}"
  expect(response).to have_http_status(:success)  # 验证 200 OK
  expect(draft_article.reload.views).to eq 0      # 验证不计数
end
```

**公开门户测试中没有任何 archived 相关测试**。

### 2.2 管理侧测试 - 有 archived 状态设置测试

**文件位置**：`spec/controllers/api/v1/accounts/articles/bulk_actions_controller_spec.rb:47-56`

```ruby
it 'archives multiple articles' do
  patch update_status_url,
        params: { ids: [...], status: 'archived' },
        as: :json
  
  expect(response).to have_http_status(:ok)
  expect(article_one.reload.status).to eq('archived')  # 仅验证状态变更
  expect(article_three.reload.status).to eq('archived')
end
```

**管理侧只验证了"能设置 archived 状态"，没有验证"archived 后公开门户能否访问"**。

### 2.3 测试覆盖总结

| 测试类型 | Draft 文章 | Archived 文章 |
|----------|------------|---------------|
| 公开门户直链访问 | ✅ 有测试（验证 200 OK） | ❌ 无测试 |
| 公开门户 tracking pixel | ✅ 有测试（验证不计数） | ❌ 无测试 |
| 管理侧状态设置 | ✅ 有测试 | ✅ 有测试（仅状态变更） |
| 管理侧访问 | ✅ 有测试 | ⚠️ 无明确验证 |

---

## 三、Article 状态模型定义

### 3.1 Enum 定义

**文件位置**：`app/models/article.rb:67`

```ruby
enum status: { draft: 0, published: 1, archived: 2 }
```

| 状态 | 枚举值 | `published?` 返回值 |
|------|--------|---------------------|
| draft | 0 | `false` |
| published | 1 | `true` |
| archived | 2 | `false` |

**关键**：`archived` 和 `draft` 在 `published?` 检查前是**等价的**——都返回 `false`。

### 3.2 Published Scope 定义

**文件位置**：Rails 自动生成（来自 enum）

```ruby
# 等效于
scope :published, -> { where(status: 1) }
scope :draft, -> { where(status: 0) }
scope :archived, -> { where(status: 2) }
```

**注意**：`articles.published` 只包含 `status=1` 的记录，排除 draft(0) **和** archived(2)。

---

## 四、直链访问路径分析

### 4.1 路径一：HTML 页面 `/hc/:slug/articles/:article_slug`

#### 控制器代码

**文件位置**：`app/controllers/public/api/v1/portals/articles_controller.rb:6,22-24,63-66`

```ruby
before_action :set_article, only: [:show]

def show
  @og_image_url = helpers.set_og_image_url(@portal.name, @article.title)
end

def set_article
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  # 没有 .published 过滤！
  # 没有 .where(status: ...) 过滤！
  @parsed_content = render_article_content(@article.content.to_s)
end
```

#### 已证实的结论

| 维度 | 证据 | 结论 |
|------|------|------|
| **无 status 过滤** | `find_by(slug: ...)` 无 `.published` | archived 会被查询到 |
| **无视图层检查** | 视图代码无 `@article.published?` 或 `@article.status` 检查 | 查到就渲染 |
| **无企业版覆盖** | 企业版扩展只有 `search_articles` | 不影响访问控制 |

#### 与 published scope 的对比

```ruby
# index 方法 - 有过滤
@articles = @portal.articles.published.includes(:category, :author)
# 等同于 where(status: 1)

# set_article 方法 - 无过滤
@article = @portal.articles.find_by(slug: ...)
# 等同于 where(portal_id: ..., slug: ...)
# 不限制 status，可以是 0(draft), 1(published), 或 2(archived)
```

#### 可证实的逻辑推断

```
前提：
- Article 枚举：{ draft: 0, published: 1, archived: 2 }
- index 过滤：articles.published = where(status: 1)
- set_article 查询：find_by(slug: ...) = 无 status 条件

结论：
- find_by(slug: ...) 会匹配 status=0, 1, 2 的任意文章
- draft (status=0) 测试已证实返回 200 OK
- archived (status=2) 与 draft (status=0) 在"无 status 过滤"条件下等价
- 因此 archived 也会返回 200 OK
```

**但这是逻辑推断，不是测试证实**。

### 4.2 路径二：Tracking Pixel `/hc/:slug/articles/:article_slug.png`

#### 控制器代码

**文件位置**：`app/controllers/public/api/v1/portals/articles_controller.rb:26-39`

```ruby
def tracking_pixel
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  # 没有 .published 过滤！
  
  return head :not_found unless @article  # 只检查存在性

  @article.increment_view_count if @article.published?
  # 只有 published? 为 true 才计数
  # archived 和 draft 的 published? 都是 false，所以都不计数

  # 返回图片
  send_file pixel_path, type: 'image/png', disposition: 'inline'
end
```

#### 代码行为分解

```
tracking_pixel 执行流程：

1. find_by(slug: ...) → 无 status 过滤
   ↓
   - 如果 article 不存在 → 404
   - 如果 article 存在（任何 status）→ 继续
   
2. published? 检查 → 用于控制计数
   ↓
   - draft (0) → false → 不计数
   - published (1) → true → 计数
   - archived (2) → false → 不计数
   
3. 无论计数与否，都返回 200 OK + PNG 图片
```

#### 已证实的结论

| 维度 | 证据 | 结论 |
|------|------|------|
| **查询无 status 过滤** | `find_by(slug: ...)` 无 `.published` | archived 会被查询到 |
| **返回 200 的条件** | 只检查 `unless @article` | 查到就返回 200 |
| **计数逻辑** | `if @article.published?` | archived 不计数 |

#### 与 draft 的对比

```ruby
# 测试证实的 draft 行为
it 'serves a PNG image but does not increment view count for draft article' do
  draft_article = create(:article, status: :draft, ...)
  get "/hc/#{portal.slug}/articles/#{draft_article.slug}.png"
  
  expect(response).to have_http_status(:success)  # 200 OK
  expect(draft_article.reload.views).to eq 0      # 不计数
end

# archived 文章的行为推断（基于相同代码逻辑）
# 同一套代码：
# 1. find_by(slug) 无 status 过滤 → archived 会被查到
# 2. 查到就返回 200 OK
# 3. published? 检查 → archived 返回 false → 不计数
```

**但 archived 的 200 OK 行为没有测试直接证实**。

---

## 五、已证实结论 vs 待验证假设

### 5.1 已证实结论（有代码/测试证据）

| 结论 | 证据类型 | 证明强度 |
|------|----------|----------|
| **Draft 文章直链返回 200 OK** | 测试 `spec.rb:131-136` | ✅ 100% 证实 |
| **Draft 文章 tracking pixel 返回 200 但不计数** | 测试 `spec.rb:156-166` | ✅ 100% 证实 |
| **set_article 无 status 过滤** | 代码 `find_by(slug: ...)` | ✅ 100% 证实 |
| **tracking_pixel 查询无 status 过滤** | 代码 `find_by(slug: ...)` | ✅ 100% 证实 |
| **`published?` 只对 status=1 返回 true** | 代码 `enum status: { draft: 0, published: 1, archived: 2 }` | ✅ 100% 证实 |
| **企业版不影响访问控制** | 代码 企业版只有 `search_articles` 扩展 | ✅ 100% 证实 |
| **视图层无 status 检查** | 代码 ERB 模板无 `@article.status` 或 `@article.published?` | ✅ 100% 证实 |

### 5.2 待验证假设（基于逻辑推断）

| 假设 | 推断依据 | 验证方式 |
|------|----------|----------|
| **Archived 文章直链返回 200 OK** | set_article 无 status 过滤，且 archived 与 draft 在查询层等价 | 运行实际测试 |
| **Archived 文章 tracking pixel 返回 200 但不计数** | 同一套代码逻辑，published? 对 archived 返回 false | 运行实际测试 |
| **Archived 文章绕过 draft_locales 限制** | slug 全局唯一，set_article 无 locale 过滤 | 逻辑推断 + 运行测试 |
| **Archived 文章内容会被完整渲染** | 视图层无 status 检查，直接渲染 `@parsed_content` | 逻辑推断 |

### 5.3 为什么不能把 draft 证据外推为 archived 结论

虽然 archived 和 draft 在 `published?` 检查前等价，但存在以下差异：

| 差异点 | Draft | Archived | 影响 |
|--------|-------|----------|------|
| **语义差异** | 工作中/未完成 | 已归档/废弃 | 可能有隐藏的业务逻辑差异 |
| **使用场景** | 可能用于预览链接 | 应该完全不可访问 | 设计意图可能不同 |
| **视图层可能的隐含检查** | 可能有 `if @article.draft?` 或 `unless @article.archived?` 逻辑 | 同上 | 但当前代码没发现 |

**关键发现**：当前代码中**没有**任何视图层或控制器层区分 draft 和 archived 的逻辑。从代码层面看，两者行为一致。但**缺少实际测试验证**。

---

## 六、两条路径的完整行为对比

### 6.1 HTML 页面路径 `/hc/:slug/articles/:article_slug`

```
请求流程：

1. around_action :set_locale
   └── switch_locale_with_article
       ├── article = Article.find_by(slug: ...)  ← 无 status 过滤
       └── 设置 I18n.locale

2. before_action :set_article
   └── @article = @portal.articles.find_by(slug: ...)  ← 无 status 过滤
   └── @parsed_content = render_article_content(@article.content)  ← 直接渲染

3. show action
   └── 无额外检查

4. 视图渲染
   └── <%= @article.title %>
   └── <%= @parsed_content %>
   └── 无 @article.published? 检查
```

**行为预期**：
| 状态 | 预期行为 | 证据强度 |
|------|----------|----------|
| draft | 200 OK，完整渲染 | ✅ 测试证实 |
| archived | 200 OK，完整渲染 | ⚠️ 逻辑推断（与 draft 代码相同） |
| published | 200 OK，完整渲染 | ✅ 测试证实 |

### 6.2 Tracking Pixel 路径 `/hc/:slug/articles/:article_slug.png`

```
请求流程：

1. @article = @portal.articles.find_by(slug: ...)  ← 无 status 过滤
2. return 404 unless @article
3. increment_view_count if @article.published?  ← 仅控制计数
4. send_file pixel_path  ← 无论如何都返回 200 OK + PNG
```

**行为预期**：
| 状态 | HTTP 状态 | 计数 | 证据强度 |
|------|-----------|------|----------|
| draft | 200 OK | 不计数 | ✅ 测试证实 |
| archived | 200 OK | 不计数 | ⚠️ 逻辑推断（与 draft 代码相同） |
| published | 200 OK | 计数 +1 | ✅ 测试证实 |

---

## 七、关键代码证据索引

| 结论 | 代码位置 | 关键代码 |
|------|----------|----------|
| status 枚举定义 | `app/models/article.rb:67` | `enum status: { draft: 0, published: 1, archived: 2 }` |
| set_article 无过滤 | `app/controllers/.../articles_controller.rb:63-66` | `find_by(slug: ...)` |
| index 有过滤 | `app/controllers/.../articles_controller.rb:11` | `articles.published` |
| tracking_pixel 查询 | `app/controllers/.../articles_controller.rb:27` | `find_by(slug: ...)` |
| tracking_pixel 计数 | `app/controllers/.../articles_controller.rb:30` | `if @article.published?` |
| 企业版不影响访问 | `enterprise/app/.../articles_controller.rb:4-10` | 只有 `search_articles` 扩展 |
| draft 测试 | `spec/.../articles_controller_spec.rb:131-136` | `status: :draft` → `expect(response).to have_http_status(:success)` |

---

## 八、建议的验证测试

要完全证实 archived 行为，需补充以下测试：

```ruby
# 建议添加到 spec/controllers/public/api/v1/portals/articles_controller_spec.rb

describe 'GET /hc/:slug/articles/:article_slug (archived article)' do
  let(:archived_article) do
    create(:article, category: category, status: :archived, portal: portal,
                     account_id: account.id, author_id: agent.id, views: 0)
  end

  it 'does not allow access to archived article via direct URL' do
    get "/hc/#{portal.slug}/articles/#{archived_article.slug}"
    # 当前代码预期：200 OK
    # 如果是安全设计：应该是 404 或 redirect
    # 需根据产品意图决定期望值
  end

  it 'serves PNG but does not count views for archived article' do
    get "/hc/#{portal.slug}/articles/#{archived_article.slug}.png"
    expect(response).to have_http_status(:success)  # 当前代码行为
    expect(archived_article.reload.views).to eq 0
  end
end
```

---

## 九、v3 到 v4 的新增内容

1. **archived 与 draft 的严格区分**：
   - ✅ 明确指出枚举定义：`{ draft: 0, published: 1, archived: 2 }`
   - ✅ `published?` 只对 status=1 返回 true，archived 和 draft 都返回 false
   - ✅ 但语义上 archived 应该是"已归档/不可访问"，draft 是"未发布"

2. **现有测试覆盖核查**：
   - ✅ 公开门户测试**只有 draft**，无 archived 覆盖
   - ✅ 管理侧测试只验证"能设置 archived 状态"，不验证访问控制
   - ✅ 企业版扩展只影响搜索，不影响访问控制

3. **已证实结论 vs 待验证假设**：
   - ✅ 已证实：draft 行为、代码无 status 过滤、`published?` 行为、视图层无检查
   - ⚠️ 待验证：archived 的实际 HTTP 行为（虽代码相同但需测试确认）
   - ✅ 明确区分"测试证实"和"逻辑推断"的结论

4. **两条路径的完整行为对比**：
   - ✅ HTML 页面路径：完整流程图 + 行为预期表
   - ✅ Tracking Pixel 路径：完整流程图 + 行为预期表
   - ✅ 建议补充的测试用例
