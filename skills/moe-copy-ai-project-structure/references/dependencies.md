# moe-copy-ai 依赖落点与复用建议

## 1) 核心框架依赖

- `plasmo` (0.90.3)
  - 用途：Chrome 扩展开发框架，提供 HMR、消息传递、存储抽象
  - 修改建议：遵循 Plasmo 约定（文件命名、导出规范），避免绕过框架直接操作
- `@plasmohq/storage`
  - 用途：Chrome sync/local storage 类型安全封装
  - 使用方式：组件通过 `useStorage` hook，工具层通过 `utils/storage.ts`
- `@plasmohq/messaging`
  - 用途：前台页面与后台脚本之间的消息传递
  - 使用方式：`sendToBackground` / `background/messages/*.ts` handler

## 2) UI 依赖

- `react` (18.2) + `react-dom` (18.2)
  - 用途：UI 渲染框架
- `tailwindcss` (3.4.17)
  - 用途：原子化 CSS，配合 `postcss` 使用
- `motion`
  - 用途：React 动画（原 framer-motion）
- `lucide-react`
  - 用途：图标库
- `markdown-it` + `rehype`
  - 用途：Markdown 渲染与 HTML 转换

## 3) AI 相关依赖

- `@xsai/stream-text` + `@xsai/shared`
  - 用途：AI 流式文本生成 SDK
  - 使用方式：集中在 `utils/ai-service.ts`
- `gpt-tokenizer`
  - 用途：Token 计数（估算 prompt/response 长度）

## 4) 内容处理依赖

- `@mozilla/readability`
  - 用途：Mozilla Readability 算法提取正文
  - 使用方式：`utils/readability-extractor.ts`
- `dompurify`
  - 用途：HTML 清洗（防 XSS）
  - 使用方式：`utils/sanitize-html.ts`
- `turndown`
  - 用途：HTML 转 Markdown
- `jszip`
  - 用途：ZIP 打包导出
  - 使用方式：`utils/zip-exporter.ts`

## 5) 工程协作依赖

- `comlink`
  - 用途：Web Worker 通信（RPC 风格）
  - 使用方式：`utils/workers/`
- `robot3`
  - 用途：有限状态机（Pipeline 编排）
  - 使用方式：`utils/pipeline/`

## 6) 开发依赖

- 代码质量
  - `@biomejs/biome` (2.3.10): Linting + Formatting（替代 ESLint + Prettier）
- 测试
  - `vitest` (4.0.17): 测试框架
  - `@vitest/browser` + `playwright`: 浏览器测试运行器
- 类型
  - `typescript` (5.3): TypeScript strict 模式

## 7) 依赖选择与变更原则

- 优先复用现有依赖，不重复引入同类工具。
- 新增依赖前先评估：
  - 是否有现有依赖可满足需求？
  - 是否影响扩展包体积？（Chrome 扩展对大小敏感）
  - 是否与 Plasmo 构建兼容？
- Chrome 扩展特殊考量：
  - 避免使用 Node.js 专属 API 的依赖
  - 注意 Content Security Policy 对动态执行的限制
  - Web Worker 中使用的依赖需确认 Worker 环境兼容性
- 涉及依赖升级时，至少执行：
  - `pnpm lint`
  - `pnpm build`
  - `pnpm test:run`
