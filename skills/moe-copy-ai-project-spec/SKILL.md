---
name: moe-copy-ai-project-spec
description: "moe-copy-ai 全局架构与改造规范。用于跨 popup、sidepanel、background、contents、components、hooks、utils 的功能迭代、接口重构、性能优化、Chrome 扩展治理，并按《软件设计的哲学》原则直接实施代码改造时。默认全局提示词见 references/global-prompt.md。"
---

# moe-copy-ai 项目规范技能

## 快速开始

- 读取 `references/global-prompt.md`，将其作为当前任务的默认全局提示词并贯彻执行。
- 涉及路径定位或跨模块改动时，先读取 `moe-copy-ai-project-structure` 技能中的 `references/project-structure.md`。
- 路径不确定时，先执行 `rg --files background contents components hooks utils constants parser locales` 确认最新目录映射。

## 注意事项

- 依赖流向严格遵守：`popup/options/sidepanel → components → hooks → utils → constants`，禁止循环依赖。
- Chrome 扩展 API 调用集中在 `background/` 和 `contents/`，组件层不直接调用 `chrome.*`。
- AI 服务（xsAI SDK）的调用集中在 `utils/ai-service.ts`，组件通过 hooks 间接使用。
- Storage 操作通过 `utils/storage.ts` 封装，组件通过 Plasmo `useStorage` hook 访问。
- Web Worker 相关逻辑集中在 `utils/workers/`，通过 Comlink 进行通信。
- i18n 必须同时更新 `locales/en_US.json` 和 `locales/zh_CN.json`，使用 `useI18n()` hook。
- 应用未发布，直接修改代码，不使用 feature flag 或向后兼容 shim。

## 执行步骤

1) 先判断需求边界属于 UI 入口（`popup.tsx`/`sidepanel.tsx`/`options.tsx`）、后台脚本（`background/`）、内容脚本（`contents/`）、组件层（`components/`）、逻辑层（`hooks/`/`utils/`）或配置层（`constants/`）。
2) 识别复杂性来源（组件职责混杂、hook 依赖过重、utils 间耦合、消息传递协议、Worker 通信、存储竞态），确定最小且有效的改造目标。
3) 先改接口与抽象，再改实现；优先让常见场景（单页提取、AI 处理、批量抓取）保持简单。
4) 小步增量改造：每次变更只解决一类核心复杂性，避免同时改动 UI、逻辑、后台与内容脚本的多个高风险点。
5) 改造完成后补齐必要测试，并更新结构映射或依赖映射文档。

## 质量门禁（必须执行）

- 代码质量检查：
  - `pnpm lint`（Biome 检查）
- 构建验证（代码有改动时）：
  - `pnpm build`

## 测试验收（必须执行）

- 测试命令：
  - `pnpm test`（Vitest 浏览器测试 watch 模式）
  - `pnpm test:run`（CI 模式，单次运行）
- 测试文件命名：
  - `*.browser.ts` 或 `*.browser.tsx`
- 测试基础设施：
  - 使用 `vi.useFakeTimers()` + `vi.setSystemTime()` 处理时间依赖
  - 共享 mock 在 `utils/__tests__/mocks/`
  - 每个测试前 `resetMockStorage()` 确保隔离

## 需要时加载的参考（加载关键词：`《软件设计的哲学》`）

- `references/global-prompt.md`: 项目全局角色设定、警示信号与设计原则。
