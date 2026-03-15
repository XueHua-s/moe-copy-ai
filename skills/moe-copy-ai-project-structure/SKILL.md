---
name: moe-copy-ai-project-structure
description: "moe-copy-ai 项目结构与模块地图。用于定位代码、理解 Chrome 扩展各层（popup/sidepanel/options/background/contents）与业务逻辑层（components/hooks/utils/constants）的协作关系、选择正确的修改目标，或在目录迁移后同步路径映射时。"
---

# moe-copy-ai 项目结构技能

## 快速开始

- 先读 `references/project-structure.md` 获取入口、模块边界与关键链路映射。
- 需要选型时读 `references/dependencies.md`，优先复用现有依赖与封装。
- 用户提到"目录变更/路径失效/找不到文件/不确定改哪个模块"时，先执行 `rg --files background contents components hooks utils constants parser locales` 做路径自检。

## 导航原则

- 先判断模块边界：`popup.tsx`/`sidepanel.tsx`/`options.tsx`（UI 入口）、`background/`（扩展生命周期与消息）、`contents/`（DOM 注入）、`components/`（React UI）、`hooks/`（React hooks）、`utils/`（核心业务逻辑）、`constants/`（配置与类型）。
- 先找入口再追深模块：`sidepanel.tsx` → `components/sidepanel/` → `hooks/useContentExtraction.ts` → `utils/extractor.ts`。
- 依赖流向：`popup/options/sidepanel → components → hooks → utils → constants`，不可反向。
- Chrome 扩展 API 只在 `background/` 和 `contents/` 中直接使用。
- AI 相关调用链：`components/ai/ → hooks/useAiPrompt.ts → utils/ai-service.ts → @xsai/stream-text`。
- 内容提取调用链：`components/extraction/ → hooks/useContentExtraction.ts → utils/extractor.ts → utils/extractor/`。
- 批量抓取调用链：`contexts/BatchScrapeContext → hooks/useBatchScrape.ts → utils/batch-scraper.ts → utils/workers/scrape-worker-client.ts`。

## 执行步骤

1) 判断需求属于哪个模块层：UI 入口、后台脚本、内容脚本、组件、hooks、utils、constants。
2) 路径不确定时先做目录自检并确认关键入口文件存在。
3) 从入口沿调用链追踪到具体模块，不在未确认目录下盲改。
4) 增删依赖前先确认 `package.json` 所属模块。
5) 若发现目录变更，先同步更新 `references/project-structure.md` 与 `references/dependencies.md` 再继续实现。

## 质量命令选择

- 代码质量检查：
  - `pnpm lint`（Biome 检查）
  - `pnpm lint:fix`（自动修复）
- 构建验证（改动任意源码时）：
  - `pnpm build`（Chrome 生产构建）
  - `pnpm build:firefox`（Firefox 构建）

## 测试命令选择

- `pnpm test`（Vitest watch 模式）
- `pnpm test:run`（CI 单次运行）
- `pnpm test:ui`（Vitest UI 模式）
- 测试文件命名：`*.browser.ts` 或 `*.browser.tsx`
- 共享 mock：`utils/__tests__/mocks/`

## 需要时加载的参考

- `references/project-structure.md`: 目录、入口、调用链路与改动映射。
- `references/dependencies.md`: 依赖落点、用途与复用建议。
