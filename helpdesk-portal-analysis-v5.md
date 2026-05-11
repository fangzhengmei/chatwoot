# Chatwoot Helpdesk Portal 技术分析报告（v5）

## 一、概述

本报告专门聚焦 **archived 文章**在两条公开路径的行为核查：

1. **HTML 页面路径**：`GET /hc/:slug/articles/:article_slug`
2. **Tracking Pixel 路径**：`GET /hc/:slug/articles/:article_slug.png`

报告严格区分：
- **代码可证实结论**：基于现有代码逻辑和测试设计可 100% 确定
- **运行时验证缺口**：需补充测试才能完全确认
- **建议补测方案**：最小化的测试用例

---

## 二、测试环境限制说明

### 2.1 当前环境限制

- **操作系统**：Windows
- **Ruby 环境**：不可用（`ruby` 和 `bundle` 命令不在 PATH 中）
- **rbenv**：需要初始化但当前终端未配置

因此**无法直接运行测试**来验证 archived 的实际行为。本报告基于代码分析和现有测试设计推断。

### 2.2 替代分析方法

既然无法直接运行测试，采用以下分析策略：

| 分析维度 | 方法 |
|----------|------|
| **代码逻辑** | 逐行审查控制器代码，确认无 status 过滤 |
| **测试设计意图** | 分析现有测试用例名称和结构，推断 'not published' 的范围 |
| **枚举定义** | 确认 `published?` 对 draft 和 archived 都返回 false |
| **等价性分析** | 证明 draft 和 archived 在访问控制代码中完全等价 |

---

## 三、现有测试覆盖情况

### 3.1 公开门户测试结构

**文件位置**：`spec/controllers/public/api/v1/portals/articles_controller_spec.rb`

#### 测试用例清单

| 测试用例 | 状态 | 验证内容 |
|----------|------|----------|
| 124-129 | published | HTML 页面返回 200，渲染内容 |
| **131-136** | **draft** | **200 OK，不计数** |
| 138-142 | published（不同 locale） | 200 OK |
| 146-154 | published | PNG 返回 200，计数 +1 |
| **156-166** | **draft** | **PNG 返回 200，不计数** |
| 168-172 | 不存在 | 404 |

### 3.2 关键测试用例分析

#### HTML 路径 - Draft 测试（第 131-136 行）

```ruby
it 'does not increment the view count if the article is not published' do
  # 测试用例名称："如果文章不是 published，不增加视图计数"
  # 注意：名称用的是 "not published"，不是 "draft"
  
  draft_article = create(:article, category: category, status: :draft, ...)
  get "/hc/#{portal.slug}/articles/#{draft_article.slug}"
  
  expect(response).to have_http_status(:success)  # 200 OK
  expect(draft_article.reload.views).to eq 0       # 不计数
end
```

#### Tracking Pixel - Draft 测试（第 156-166 行）

```ruby
it 'serves a PNG image but does not increment view count for draft article' do
  # 这个测试用例明确说是 "draft article"
  
  draft_article = create(:article, category: category, status: :draft, ...)
  get "/hc/#{portal.slug}/articles/#{draft_article.slug}.png"
  
  expect(response).to have_http_status(:success)  # 200 OK
  expect(draft_article.reload.views).to eq 0      # 不计数
end
```

### 3.3 测试设计意图推断

| 测试用例名称 | 推断 |
|--------------|------|
| `'does not increment the view count if the article is not published'` | "not published" 暗示包括 **所有非 published 状态**（draft 和 archived） |
| `'serves a PNG image but does not increment view count for draft article'` | 这个用 draft 作为 "not published" 的**代表**进行测试 |

**结论**：测试设计者知道 `published?` 对 draft 和 archived 都返回 false，因此选择 draft 作为代表进行测试。但**只测试了 draft，没测试 archived**。

---

## 四、代码层面的等价性分析

### 4.1 Article 状态枚举

**文件位置**：`app/models/article.rb:67`

```ruby
enum status: { draft: 0, published: 1, archived: 2 }
```

| 状态 | 枚举值 | `article.status` | `article.published?` |
|------|--------|------------------|---------------------|
| draft | 0 | `'draft'` | `false` |
| published | 1 | `'published'` | `true` |
| archived | 2 | `'archived'` | `false` |

**关键**：`published?` 是 Rails 基于 enum 自动生成的方法，只对 `status == 1` 返回 `true`。draft(0) 和 archived(2) **在 `published?` 检查前完全等价**。

### 4.2 路径一：HTML 页面 `/hc/:slug/articles/:article_slug`

#### 控制器代码

**文件位置**：`app/controllers/public/api/v1/portals/articles_controller.rb:6,22-24,63-66`

```ruby
before_action :set_article, only: [:show]

def show
  @og_image_url = helpers.set_og_image_url(@portal.name, @article.title)
  # 无任何 status 检查
end

def set_article
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  # ↑ 关键：只有两个条件
  # 1. portal_id = @portal.id
  # 2. slug = params[:article_slug]
  # 没有 .published 过滤！
  # 没有 .where(status: ...) 过滤！
  
  @parsed_content = render_article_content(@article.content.to_s)
  # 直接渲染内容，不检查状态
end
```

#### 代码分析结论

| 检查点 | 代码证据 | 结论 |
|--------|----------|------|
| **查询条件** | `find_by(slug: ...)` | 无 status 过滤 |
| **show 动作** | 空方法，只有 OG 图片设置 | 无状态检查 |
| **视图层** | `<%= @article.title %>`, `<%= @parsed_content %>` | 无 `@article.published?` 检查 |
| **企业版扩展** | 只有 `search_articles` | 不影响访问控制 |

**代码可证实结论**：
- 只要 `@portal.articles` 中有该 slug 的文章（无论 status=0, 1, 2），就会被查到并渲染
- **没有任何代码区分 draft 和 archived**

### 4.3 路径二：Tracking Pixel `/hc/:slug/articles/:article_slug.png`

#### 控制器代码

**文件位置**：`app/controllers/public/api/v1/portals/articles_controller.rb:26-39`

```ruby
def tracking_pixel
  @article = @portal.articles.find_by(slug: permitted_params[:article_slug])
  # ↑ 同样无 status 过滤
  
  return head :not_found unless @article
  # 只检查存在性，不检查状态
  
  @article.increment_view_count if @article.published?
  # ↑ 关键：published? 只对 status=1 返回 true
  # draft(0) 和 archived(2) 都返回 false → 都不计数
  
  send_file pixel_path, type: 'image/png', disposition: 'inline'
  # 无论计数与否，都返回 200 OK + PNG
end
```

#### 代码分析结论

| 检查点 | 代码证据 | 结论 |
|--------|----------|------|
| **查询条件** | `find_by(slug: ...)` | 无 status 过滤 |
| **404 条件** | `unless @article` | 只有不存在才 404 |
| **计数条件** | `if @article.published?` | draft 和 archived 都不计数 |
| **返回条件** | `send_file` 无条件执行（只要 @article 存在） | 查到就返回 200 |

**代码可证实结论**：
- 只要文章存在（无论 status），就返回 200 OK + PNG
- 只有 `published?` 检查用于控制计数，draft 和 archived 都返回 `false` → 都不计数
- **没有任何代码区分 draft 和 archived**

---

## 五、代码可证实结论

### 5.1 路径一：HTML 页面

| 结论 | 证据类型 | 证明强度 |
|------|----------|----------|
| **控制器查询无 status 过滤** | `find_by(slug: ...)` | ✅ 100% 证实 |
| **show 动作无状态检查** | 空方法 | ✅ 100% 证实 |
| **视图层无状态检查** | ERB 模板无 `@article.published?` | ✅ 100% 证实 |
| **企业版不影响访问控制** | 只有 `search_articles` 扩展 | ✅ 100% 证实 |
| **draft 测试返回 200 OK** | 测试用例 131-136 | ✅ 100% 证实 |
| **draft 和 archived 在代码中无区别** | 没有 `if @article.archived?` 或 `unless @article.draft?` | ✅ 100% 证实 |

**基于以上，可逻辑推断**：
> 既然 draft 返回 200，且代码没有任何区分 draft 和 archived 的逻辑，archived 也应该返回 200。

但这仍是**逻辑推断**，不是**运行时验证**。

### 5.2 路径二：Tracking Pixel

| 结论 | 证据类型 | 证明强度 |
|------|----------|----------|
| **控制器查询无 status 过滤** | `find_by(slug: ...)` | ✅ 100% 证实 |
| **404 只在文章不存在时返回** | `unless @article` | ✅ 100% 证实 |
| **`published?` 只对 status=1 返回 true** | `enum status: { draft: 0, published: 1, archived: 2 }` | ✅ 100% 证实 |
| **draft 测试返回 200 且不计数** | 测试用例 156-166 | ✅ 100% 证实 |
| **draft 和 archived 的 `published?` 都返回 false** | 枚举定义 + Rails 自动方法 | ✅ 100% 证实 |
| **没有代码区分 draft 和 archived** | 同上 | ✅ 100% 证实 |

**基于以上，可逻辑推断**：
> 既然 draft 返回 200 且不计数，且代码没有任何区分，archived 也应该返回 200 且不计数。

---

## 六、运行时验证缺口

### 6.1 缺口定义

**缺口**：代码逻辑显示 draft 和 archived 完全等价，但**只有 draft 被实际测试过**。

| 缺口类型 | 描述 |
|----------|------|
| **HTML 页面** | archived 文章是否真的返回 200？ |
| **Tracking Pixel** | archived 文章是否真的返回 200 且不计数？ |
| **边缘情况** | 有没有隐藏的中间件或 Rack 层拦截 archived？ |

### 6.2 为什么代码等价性不等于运行时验证

即使代码中没有 `if @article.archived?` 检查，理论上仍可能存在：

1. **Rails 中间件**：可能有未被审查的中间件拦截特定状态
2. **数据库约束**：可能有触发器或视图影响查询结果
3. **Rails 魔术方法**：可能有 `archived?` 相关的回调或钩子
4. **配置差异**：可能有基于环境的行为差异

**但根据代码审查，以上可能性极低**（代码中没有任何迹象）。

---

## 七、最小补测方案

### 7.1 需补充的测试用例

**文件**：`spec/controllers/public/api/v1/portals/articles_controller_spec.rb`

在 `describe 'GET /public/api/v1/portals/:slug/articles/:id'` 块中添加：

```ruby
# 在第 136 行后添加
it 'does not increment the view count for archived article' do
  archived_article = create(:article, category: category, status: :archived, portal: portal,
                            account_id: account.id, author_id: agent.id, views: 0)
  
  get "/hc/#{portal.slug}/articles/#{archived_article.slug}"
  
  expect(response).to have_http_status(:success)
  expect(response.body).to include(ChatwootMarkdownRenderer.new(archived_article.content).render_article)
  expect(archived_article.reload.views).to eq 0
end
```

在 `describe 'GET /public/api/v1/portals/:slug/articles/:slug.png'` 块中添加：

```ruby
# 在第 166 行后添加
it 'serves a PNG image but does not increment view count for archived article' do
  archived_article = create(:article, category: category, status: :archived, portal: portal,
                            account_id: account.id, author_id: agent.id, views: 0)

  get "/hc/#{portal.slug}/articles/#{archived_article.slug}.png"

  expect(response).to have_http_status(:success)
  expect(response.headers['Content-Type']).to eq('image/png')
  expect(archived_article.reload.views).to eq 0
end
```

### 7.2 测试执行命令

```bash
# 运行所有公开文章测试
bundle exec rspec spec/controllers/public/api/v1/portals/articles_controller_spec.rb

# 或只运行特定行
bundle exec rspec spec/controllers/public/api/v1/portals/articles_controller_spec.rb:137  # archived HTML
bundle exec rspec spec/controllers/public/api/v1/portals/articles_controller_spec.rb:168  # archived PNG
```

### 7.3 预期测试结果

| 测试 | 预期结果 | 原因 |
|------|----------|------|
| archived HTML 200 | ✅ 200 OK | 代码无 status 过滤 |
| archived HTML 渲染内容 | ✅ 包含内容 | 视图无状态检查 |
| archived HTML 计数 | ✅ 不计数 | `published?` 返回 false |
| archived PNG 200 | ✅ 200 OK | 代码无 status 过滤 |
| archived PNG 计数 | ✅ 不计数 | `published?` 返回 false |

如果以上都通过，archived 行为即 100% 证实。

---

## 八、最终结论分栏

### 8.1 代码可证实结论（100% 确定）

| 结论 | 证据代码位置 |
|------|--------------|
| **控制器查询无 status 过滤** | `articles_controller.rb:64` `find_by(slug: ...)` |
| **show 动作无状态检查** | `articles_controller.rb:22-24` 空方法 |
| **tracking_pixel 只有存在性检查** | `articles_controller.rb:28` `unless @article` |
| **`published?` 只对 status=1 返回 true** | `article.rb:67` `enum status: { draft: 0, published: 1, archived: 2 }` |
| **视图层无状态检查** | ERB 模板无 `@article.published?` |
| **企业版不影响访问控制** | 企业版只有 `search_articles` 扩展 |
| **draft HTML 返回 200** | `articles_controller_spec.rb:134` `expect(response).to have_http_status(:success)` |
| **draft PNG 返回 200 且不计数** | `articles_controller_spec.rb:161,165` |
| **没有代码区分 draft 和 archived** | 全局搜索无 `if @article.archived?` 或 `unless @article.draft?` |

### 8.2 仍需运行时验证的结论

| 结论 | 逻辑依据 | 验证方法 |
|------|----------|----------|
| **archived HTML 返回 200** | 代码无区别，draft 返回 200 | 运行补充测试 |
| **archived HTML 渲染内容** | 视图无状态检查 | 运行补充测试 |
| **archived PNG 返回 200** | 代码无区别，draft 返回 200 | 运行补充测试 |
| **archived PNG 不计数** | `published?` 返回 false | 运行补充测试 |

---

## 九、关键代码证据索引

| 结论 | 代码位置 | 关键代码 |
|------|----------|----------|
| status 枚举定义 | `app/models/article.rb:67` | `enum status: { draft: 0, published: 1, archived: 2 }` |
| set_article 无过滤 | `app/controllers/.../articles_controller.rb:63-66` | `find_by(slug: ...)` |
| tracking_pixel 查询 | `app/controllers/.../articles_controller.rb:27` | `find_by(slug: ...)` |
| tracking_pixel 404 条件 | `app/controllers/.../articles_controller.rb:28` | `unless @article` |
| tracking_pixel 计数条件 | `app/controllers/.../articles_controller.rb:30` | `if @article.published?` |
| draft HTML 测试 | `spec/.../articles_controller_spec.rb:131-136` | `status: :draft` → `expect(response).to have_http_status(:success)` |
| draft PNG 测试 | `spec/.../articles_controller_spec.rb:156-166` | `status: :draft` → `expect(response).to have_http_status(:success)` |

---

## 十、v4 到 v5 的新增内容

1. **代码可证实结论的收敛**：
   - ✅ 明确列出了所有 100% 可证实的结论
   - ✅ 每条结论都有具体的代码位置和关键代码

2. **运行时验证缺口的明确定义**：
   - ✅ 区分了"代码逻辑等价"和"运行时验证"
   - ✅ 说明了为什么代码等价性不等于运行时验证（中间件、触发器等边缘情况）

3. **最小补测方案**：
   - ✅ 提供了两个最小化的测试用例（HTML 路径 + PNG 路径）
   - ✅ 提供了执行命令
   - ✅ 给出了预期结果和原因

4. **最终结论分栏**：
   - ✅ **代码可证实结论**（100% 确定）- 左栏
   - ✅ **仍需运行时验证的结论** - 右栏
   - ✅ 严格区分，避免逻辑推断被当作已证实事实
