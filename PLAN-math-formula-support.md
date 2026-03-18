# Math Formula Extraction & Rendering Plan

## 1. 目标

在网页内容抓取 → Markdown 转换 → 预览渲染的全链路中支持数学公式：

1. **抽取阶段**：识别页面中 KaTeX、MathJax (v2/v3)、原生 MathML 渲染的公式 HTML，提取其 LaTeX 源码
2. **转换阶段**：在 unified pipeline 中将公式 HTML 节点替换为 `$...$`（行内）/ `$$...$$`（块级）LaTeX 定界符
3. **渲染阶段**：Markdown 预览支持 LaTeX 公式渲染

---

## 2. 现状分析

### 当前 Pipeline

```
DOM (element.outerHTML)
  → preprocess-html (移除 script/style、解析 URL)
  → rehype-parse (HTML → HAST)
  → rehypeUnwrapInvalidLinks
  → rehype-remark (HAST → MDAST, 自定义 heading handlers)
  → remark-gfm
  → remark-stringify (MDAST → Markdown string)
```

### 渲染端

```
Markdown string → markdown-it (html: true, linkify, typographer) → innerHTML
```

### 现有问题

| 问题 | 影响 |
|------|------|
| 无数学公式识别 | KaTeX/MathJax 渲染的公式被当作普通 HTML 处理，输出为乱码文本 |
| `preprocess-html` 移除 `<script>` 标签 | MathJax v2 的 `<script type="math/tex">` 源码被丢弃 |
| Readability 可能剥离数学元素 | `@mozilla/readability` 可能移除 `<math>`、`.katex`、`mjx-container` 等节点 |
| markdown-it 无数学插件 | 即使 Markdown 中有 `$...$` 也不会渲染为公式 |

---

## 3. 技术方案

### 3.1 全链路流程

```
DOM (outerHTML)
  → preprocess-html (保留 math/tex script, 新增)
  → rehype-parse
  → [NEW] rehype-extract-math (数学公式抽取插件)
  → rehypeUnwrapInvalidLinks
  → rehype-remark (+ 自定义 math handlers)
  → remark-gfm
  → [NEW] remark-math (数学节点 stringify 支持)
  → remark-stringify
  → Markdown with $...$ / $$...$$

渲染:
  markdown-it
  + [NEW] markdown-it-katex/texmath 插件
  + [NEW] KaTeX CSS
  → HTML with rendered formulas
```

### 3.2 Phase 1 — 数学公式抽取 rehype 插件

**新建文件**: `parser/plugins/math-extractor.ts`

这是核心模块，作为 rehype 插件在 HAST 树上运行，识别并转换三类数学公式源：

#### 3.2.1 KaTeX 渲染识别

KaTeX 的 DOM 结构：

```html
<!-- 行内 -->
<span class="katex">
  <span class="katex-mathml">
    <math xmlns="http://www.w3.org/1998/Math/MathML">
      <semantics>
        <mrow>...</mrow>
        <annotation encoding="application/x-tex">E = mc^2</annotation>
      </semantics>
    </math>
  </span>
  <span class="katex-html" aria-hidden="true">...</span>
</span>

<!-- 块级 -->
<span class="katex-display">
  <span class="katex"><!-- 同上 --></span>
</span>
```

**提取策略**：
1. 匹配 `.katex` 或 `.katex-display` 类名
2. 查找内部 `annotation[encoding="application/x-tex"]` 元素
3. 提取 annotation 的 text content 作为 LaTeX 源码
4. 根据是否有 `.katex-display` 父级判断 inline/block

#### 3.2.2 MathJax v3 渲染识别

MathJax v3 的 DOM 结构：

```html
<!-- CHTML 输出 -->
<mjx-container class="MathJax" jax="CHTML" display="true">
  <mjx-math>...</mjx-math>
  <!-- 辅助 MathML (如开启 assistiveMml) -->
  <mjx-assistive-mml>
    <math xmlns="http://www.w3.org/1998/Math/MathML">
      <semantics>
        <mrow>...</mrow>
        <annotation encoding="application/x-tex">LaTeX source</annotation>
      </semantics>
    </math>
  </mjx-assistive-mml>
</mjx-container>

<!-- SVG 输出 -->
<mjx-container class="MathJax" jax="SVG">
  <svg>...</svg>
  <mjx-assistive-mml><!-- 同上 --></mjx-assistive-mml>
</mjx-container>
```

**提取策略**：
1. 匹配 `mjx-container` 标签或 `.MathJax` 类名
2. 优先查找 `annotation[encoding="application/x-tex"]`
3. 如无 annotation，查找 `mjx-assistive-mml > math` 并用 `mathml-to-latex` 转换
4. `display` 属性判断 inline/block

#### 3.2.3 MathJax v2 渲染识别

MathJax v2 的 DOM 结构：

```html
<!-- 原始 TeX 源码 (在 script 标签中) -->
<script type="math/tex">E = mc^2</script>
<script type="math/tex; mode=display">...</script>

<!-- 渲染输出 -->
<span class="MathJax" id="MathJax-Element-1-Frame">
  <span class="math">...</span>
</span>
<span class="MathJax_Display">
  <span class="MathJax">...</span>
</span>
```

**提取策略**：
1. 匹配 `<script type="math/tex">` 或 `<script type="math/tex; mode=display">`（需在 preprocess 中保留）
2. 直接取 script 的 text content 作为 LaTeX
3. `mode=display` 判断 block
4. 渲染后的 `.MathJax` span 如有相邻 script 则跳过（避免重复）；如无 script，fallback 到 MathML annotation

#### 3.2.4 原生 MathML 识别

```html
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mrow>
    <mi>E</mi><mo>=</mo><mi>m</mi>
    <msup><mi>c</mi><mn>2</mn></msup>
  </mrow>
</math>
```

**提取策略**：
1. 匹配 `<math>` 元素（非 KaTeX/MathJax 内部的）
2. 检查是否有 `<annotation encoding="application/x-tex">`：有则直接取
3. 无 annotation 时，使用 `mathml-to-latex` 库将 MathML 序列化后转换
4. 根据 `display="block"` 属性或上下文判断 inline/block

#### 3.2.5 插件输出

对于每个识别到的数学公式节点，替换为 HAST 节点：

```typescript
// 行内公式 → 替换为包含 data-math 属性的 span
{
  type: 'element',
  tagName: 'span',
  properties: { 'data-math-type': 'inline' },
  children: [{ type: 'text', value: latexSource }]
}

// 块级公式 → 替换为包含 data-math 属性的 div
{
  type: 'element',
  tagName: 'div',
  properties: { 'data-math-type': 'block' },
  children: [{ type: 'text', value: latexSource }]
}
```

### 3.3 Phase 2 — Markdown Pipeline 集成

#### 3.3.1 修改 `parser/plugins/preprocess-html.ts`

**变更**：保留 `<script type="math/tex">` 和 `<script type="math/tex; mode=display">`

```typescript
// 当前逻辑：移除所有 <script> 标签
// 修改为：仅移除非 math/tex 类型的 script
if (tagName === 'script') {
  const type = element.getAttribute('type') || ''
  if (!type.startsWith('math/tex')) {
    element.remove()
  }
}
```

#### 3.3.2 修改 `parser/index.ts`

在 unified 链中插入数学相关插件：

```typescript
import { rehypeExtractMath } from './plugins/math-extractor'
import remarkMath from 'remark-math'

const parserInstance = unified()
  .use(rehypeParse, { fragment: true })
  .use(rehypeExtractMath)                // [NEW] 数学公式抽取
  .use(rehypeUnwrapInvalidLinks)
  .use(rehypeRemark, {
    handlers: {
      ...HEADING_HANDLERS,
      ...MATH_HANDLERS,                  // [NEW] data-math 节点转 mdast math 节点
    }
  })
  .use(remarkGfm)
  .use(remarkMath)                       // [NEW] 数学节点 stringify 支持
  .use(remarkStringify, { /* ... */ })
```

#### 3.3.3 自定义 rehype-remark handler

**新建文件**: `parser/plugins/math-handlers.ts`

为 `data-math-type` 属性的元素注册自定义 handler，将 HAST 节点转为 mdast 的 `inlineMath` / `math` 节点：

```typescript
// 将 <span data-math-type="inline">LaTeX</span> → { type: 'inlineMath', value: 'LaTeX' }
// 将 <div data-math-type="block">LaTeX</div> → { type: 'math', value: 'LaTeX' }
```

`remark-math` 的 stringify 扩展（来自 `mdast-util-math`）会将这些节点输出为 `$...$` 和 `$$\n...\n$$`。

### 3.4 Phase 3 — Markdown 渲染端 LaTeX 支持

#### 3.4.1 修改 `components/ContentDisplay.tsx`

为 markdown-it 实例添加数学渲染插件：

```typescript
import markdownItTexmath from 'markdown-it-texmath'
import katex from 'katex'

const md = new MarkdownIt({ html: true, linkify: true, typographer: true })
  .use(markdownItTexmath, {
    engine: katex,
    delimiters: 'dollars',  // 使用 $ 和 $$ 定界符
  })
```

#### 3.4.2 KaTeX CSS 引入

在扩展的样式中引入 KaTeX 的 CSS：

```typescript
// 方式 1: 直接 import (Plasmo 支持)
import 'katex/dist/katex.min.css'

// 方式 2: 在 styles/ 目录中引入
```

**注意**：KaTeX 字体文件需要通过 Plasmo 的 asset 机制正确打包到扩展中。

#### 3.4.3 方案选型比较

| 方案 | 包 | 大小 | 优势 | 劣势 |
|------|-----|------|------|------|
| **A: markdown-it-texmath + KaTeX** | `markdown-it-texmath` + `katex` | ~300KB (KaTeX) | 灵活定界符、成熟稳定 | 较大体积 |
| **B: @mdit/plugin-katex** | `@mdit/plugin-katex` + `katex` | ~300KB (KaTeX) | 现代 API、TS 原生 | 来自 vuepress 生态 |
| **C: markdown-it + Temml** | `markdown-it-texmath` + `temml` | ~200KB (Temml) | 更轻量、输出纯 MathML | 浏览器 MathML 支持差异 |

**推荐方案 A**：`markdown-it-texmath` + `katex`。理由：
- `markdown-it-texmath` 是 markdown-it 生态最成熟的数学插件
- KaTeX 渲染速度快，Chrome 88+ 全兼容
- 社区资源丰富，问题易排查

---

## 4. 新增依赖

| 包名 | 用途 | 生态 |
|------|------|------|
| `remark-math` | mdast 数学节点定义 + stringify | unified (已有 unified 依赖) |
| `mathml-to-latex` | MathML → LaTeX 转换（无 annotation 时的 fallback） | 独立库 |
| `katex` | LaTeX → HTML 渲染引擎 | 独立库 |
| `markdown-it-texmath` | markdown-it 数学插件 | markdown-it 生态 |

```bash
pnpm add remark-math mathml-to-latex katex markdown-it-texmath
pnpm add -D @types/katex
```

---

## 5. 文件变更清单

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `parser/plugins/math-extractor.ts` | **新建** | 核心：rehype 插件，识别 KaTeX/MathJax/MathML 并提取 LaTeX |
| `parser/plugins/math-handlers.ts` | **新建** | rehype-remark 自定义 handler，data-math 节点 → mdast math 节点 |
| `parser/index.ts` | 修改 | 插入 rehypeExtractMath、remarkMath、MATH_HANDLERS |
| `parser/plugins/preprocess-html.ts` | 修改 | 保留 `<script type="math/tex">` |
| `components/ContentDisplay.tsx` | 修改 | 添加 markdown-it-texmath + KaTeX 渲染 |
| `styles/` 或组件样式 | 修改 | 引入 KaTeX CSS |
| `package.json` | 修改 | 新增 4 个依赖 |

---

## 6. 抽取优先级与 Fallback 链

对每个数学公式节点，按以下顺序尝试提取 LaTeX：

```
1. annotation[encoding="application/x-tex"]   ← 最可靠，KaTeX/MathJax 均会生成
2. <script type="math/tex">                    ← MathJax v2 原始源码
3. data-formula / data-tex 等自定义属性        ← 部分站点自定义
4. mathml-to-latex(<math> 序列化)              ← fallback，质量略低
5. 放弃，保留原始文本                          ← 最终 fallback
```

---

## 7. 风险与注意事项

### 7.1 Readability 模式下数学元素丢失

`@mozilla/readability` 可能将数学相关的 DOM 节点判定为非内容而移除。

**缓解方案**：在 readability 处理前，先在 DOM 层面对数学节点做标记或预转换：
- 遍历 DOM，找到 KaTeX/MathJax/MathML 元素
- 替换为 `<code data-math-preserve="inline">$LaTeX$</code>` 保护节点
- readability 不会移除 `<code>` 标签
- 后续 pipeline 中再还原

**影响文件**：`utils/readability/extractor.ts` 或 `contents/scraper.ts`

### 7.2 MathJax v3 Assistive MathML 可能未开启

部分网站的 MathJax v3 配置未启用 `assistiveMml`，导致无 `<math>` 子树。

**缓解方案**：
- 检查 `mjx-container` 是否有 `data-original` 或类似属性存储原始 TeX
- 如无任何 LaTeX/MathML 来源，fallback 到提取可见文本（质量降级但不丢失内容）

### 7.3 KaTeX CSS 字体打包

KaTeX 依赖自定义字体文件（woff2），在 Chrome Extension 中需要确保字体文件被正确打包。

**缓解方案**：
- Plasmo 支持 static assets，将 KaTeX 字体放入 `assets/` 目录
- 或使用 CSS 中的 `chrome-extension://` URL 引用字体
- 测试方式：检查公式渲染是否正确显示特殊符号（积分号、求和号等）

### 7.4 remark-math 版本兼容性

当前项目 unified v10、remark-stringify v10。`remark-math` v6+ 要求 unified v11+。

**缓解方案**：
- 使用 `remark-math@5.x`（兼容 unified v10）
- 或升级 unified 生态到 v11（需评估其他插件兼容性）
- 也可直接使用底层的 `mdast-util-math` 的 `mathToMarkdown()` 手动注册 stringify 扩展，绕过版本限制

### 7.5 `$` 符号误识别

普通文本中的 `$` 符号（如价格 $100）可能被 `markdown-it-texmath` 误判为公式定界符。

**缓解方案**：
- `markdown-it-texmath` 默认要求 `$` 紧贴内容（`$x$` 识别，`$ 100` 不识别）
- 在抽取阶段，确保只有从数学元素提取的内容才使用 `$` 包裹
- 渲染端可配置 `markdown-it-texmath` 的匹配规则

---

## 8. 验证方案

### 8.1 单元测试

为 `math-extractor.ts` 编写测试，覆盖：

| 测试场景 | 输入 | 期望输出 |
|---------|------|---------|
| KaTeX 行内公式 | `<span class="katex">...<annotation>E=mc^2</annotation>...</span>` | `$E=mc^2$` |
| KaTeX 块级公式 | `<span class="katex-display">...</span>` | `$$\nE=mc^2\n$$` |
| MathJax v3 CHTML | `<mjx-container>...<annotation>...</annotation>...</mjx-container>` | `$...$` |
| MathJax v3 display | `<mjx-container display="true">...</mjx-container>` | `$$\n...\n$$` |
| MathJax v2 script | `<script type="math/tex">x^2</script>` | `$x^2$` |
| MathJax v2 display script | `<script type="math/tex; mode=display">x^2</script>` | `$$\nx^2\n$$` |
| 原生 MathML with annotation | `<math><semantics>...<annotation>...</annotation></semantics></math>` | `$...$` |
| 原生 MathML without annotation | `<math><mrow><mi>x</mi></mrow></math>` | `$x$` (via mathml-to-latex) |
| 混合内容 | 文本 + 公式 + 文本 | 文本 `$公式$` 文本 |
| 无数学内容 | 普通 HTML | 不变 |

### 8.2 集成测试

使用真实网页 HTML 片段测试端到端流程：
- Wikipedia 数学页面（MathML）
- arXiv 论文摘要页（MathJax v3）
- 知乎/CSDN 数学内容（KaTeX）
- Khan Academy（KaTeX）

### 8.3 手动验证

1. 安装扩展 dev 版本
2. 访问包含数学公式的页面（如 Wikipedia 数学条目）
3. 执行抓取
4. 检查 Source 模式：确认 `$...$` / `$$...$$` 正确出现
5. 检查 Preview 模式：确认公式正确渲染
6. 检查复制内容：确认粘贴到 Markdown 编辑器后公式可用

---

## 9. 实施顺序建议

```
Step 1: 安装依赖，确认版本兼容性 (remark-math vs unified v10)
         ↓
Step 2: 实现 math-extractor.ts (核心抽取逻辑)
         ↓
Step 3: 实现 math-handlers.ts (rehype-remark handler)
         ↓
Step 4: 修改 preprocess-html.ts (保留 math/tex script)
         ↓
Step 5: 修改 parser/index.ts (集成到 pipeline)
         ↓
Step 6: 编写抽取阶段单元测试
         ↓
Step 7: 修改 ContentDisplay.tsx (渲染端 KaTeX 支持)
         ↓
Step 8: 处理 KaTeX CSS/字体打包
         ↓
Step 9: Readability 模式兼容处理
         ↓
Step 10: 集成测试 + 真实网页验证
```
