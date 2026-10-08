# moe-copy-ai 项目结构与边界映射

## 1) 仓库总览

- `popup.tsx` / `sidepanel.tsx` / `options.tsx`: UI 入口页面
- `background/`: 扩展生命周期管理与消息处理
- `contents/`: DOM 注入脚本（内容脚本）
- `components/`: React UI 组件
- `hooks/`: React 自定义 hooks
- `utils/`: 核心业务逻辑
- `constants/`: 配置、类型定义、主题
- `parser/`: HTML 解析与转换
- `contexts/`: React Context（全局状态）
- `locales/`: i18n 翻译文件
- `styles/`: 全局样式
- `assets/`: 静态资源
- `docs/`: 文档

## 2) 关键入口文件

- `sidepanel.tsx`
  - 主要 UI 入口，侧边栏面板
  - 渲染核心交互界面
- `popup.tsx`
  - 弹出窗口入口
  - 轻量操作界面
- `options.tsx`
  - 设置页面入口
  - AI 配置、提取设置
- `background/index.ts`
  - 扩展后台服务入口
  - 注册消息监听器
- `background/messages/`
  - Plasmo 消息处理器
  - `getScrapedContent.ts`: 获取抓取内容
  - `scrapeViaTab.ts`: 通过标签页抓取
  - `extractLinksFromPage.ts`: 提取页面链接
  - `openSidePanel.ts` / `openOptionsPage.ts`: 打开面板
  - `clickNextPage.ts`: 翻页操作
- `contents/scraper.ts`
  - 内容脚本抓取逻辑
- `contents/element-selector.tsx`
  - 元素选择器 UI 覆盖层
- `contents/floating-popup.tsx`
  - 浮动操作按钮

## 3) 核心模块关系

### 内容提取链路
1. UI 触发提取 → `hooks/useContentExtraction.ts`
2. Hook 调用 → `utils/extractor.ts`（多层选择器系统）
3. Extractor 使用 → `utils/extractor/`（模块化提取策略）
4. 可选 Readability → `utils/readability-extractor.ts`
5. Pipeline 编排 → `utils/pipeline/`

### AI 处理链路
1. UI 交互 → `components/ai/`
2. 组件使用 → `hooks/useAiPrompt.ts` + `hooks/useAiSettings.ts`
3. Hook 调用 → `utils/ai-service.ts`（xsAI SDK 流式生成）
4. 流处理 → `hooks/useStreamProcessor.ts`

### 批量抓取链路
1. 批量 UI → `components/batch/`
2. Context 管理 → `contexts/BatchScrapeContext`
3. Hook 编排 → `hooks/useBatchScrape.ts`
4. 执行抓取 → `utils/batch-scraper.ts`
5. Worker 通信 → `utils/workers/scrape-worker-client.ts` → `utils/workers/scrape-worker.ts`

### 消息传递链路
1. 前台页面 → `@plasmohq/messaging` sendToBackground
2. 后台处理 → `background/messages/*.ts`
3. 内容脚本 → `contents/*.ts`

## 4) 构建链路

- `pnpm dev`: Plasmo 开发服务器，加载 `build/chrome-mv3-dev`
- `pnpm dev:firefox`: Firefox MV3 开发构建
- `pnpm build`: Chrome 生产构建
- `pnpm build:firefox`: Firefox MV3 生产构建 + 后处理脚本
- `pnpm package` / `pnpm package:firefox`: 商店发布包

## 5) 测试链路

- 框架：Vitest + 浏览器运行器（Playwright, Chromium）
- 测试文件：`utils/__tests__/*.browser.ts`
- 共享 Mock：`utils/__tests__/mocks/`（`plasmo-storage.ts` + `index.ts`）
- 运行：
  - `pnpm test`（watch 模式）
  - `pnpm test:run`（CI 模式）
  - `pnpm test:ui`（UI 模式）
- 截图产物：`vitest-test-results/`

## 6) 常见改动映射

- 修改内容提取逻辑：
  - 改 `utils/extractor.ts` 或 `utils/extractor/`
  - 补/改 `utils/__tests__/extractor.browser.ts`
- 修改 AI 服务：
  - 改 `utils/ai-service.ts`
  - 补/改 `utils/__tests__/ai-service.browser.ts`
- 添加新消息类型：
  - 在 `background/messages/` 添加处理器
  - 在调用端使用 `sendToBackground`
- 修改 UI 组件：
  - 改 `components/` 下对应目录
  - 涉及文本时更新 `locales/en_US.json` + `locales/zh_CN.json`
- 修改存储逻辑：
  - 改 `utils/storage.ts`
  - 补/改 `utils/__tests__/` 相关测试
- 修改主题/样式：
  - 改 `constants/theme.ts` + `constants/theme-colors.ts`
  - 改 `styles/global.css` 或 `utils/theme/`

## 7) 目录自检命令

```bash
rg --files background contents components hooks utils constants parser locales
```
