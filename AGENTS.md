# moe-copy-ai — Agent Collaboration Guide

Unified project context, skill-loading protocol, and collaboration guidelines for all AI agents (Claude Code, Cursor, Windsurf, Copilot, and general-purpose LLM agents).

---

## 1. Project Overview

**moe-copy-ai** is a Plasmo-based Chrome extension for AI-powered web content extraction with mobile browser support.

Core architecture:
- **UI Entry Points** (`popup.tsx`, `sidepanel.tsx`, `options.tsx`): React-based user interfaces
- **Background** (`background/`): Extension lifecycle, message handlers via Plasmo messaging
- **Content Scripts** (`contents/`): DOM injection, element selection, scraping
- **Components** (`components/`): React UI organized by feature (ai, batch, extraction, sidepanel, popup, option, ui)
- **Hooks** (`hooks/`): React custom hooks for state management and business logic orchestration
- **Utils** (`utils/`): Core business logic (extractor, AI service, storage, pipeline, workers)
- **Constants** (`constants/`): Configuration, types, theme definitions

Data flow: UI → Hooks → Utils → Chrome APIs / AI Service / Workers

---

## 2. Skill Catalog

The project maintains structured AI skill packs under `skills/`. Each skill contains a `SKILL.md` (main instructions), `references/` (supporting docs), and optionally `agents/` (agent interface config).

### Available Skills

| Skill | Path | Trigger Scenarios |
|-------|------|-------------------|
| **Code Review Expert** | `skills/code-review-expert/SKILL.md` | Code review, PR review, security scan, SOLID checks |
| **Software Design Philosophy** | `skills/software-design-philosophy/SKILL.md` | Module design, API complexity, refactoring, architecture review |
| **Project Spec** | `skills/moe-copy-ai-project-spec/SKILL.md` | Feature development, interface refactoring, quality gates, test acceptance |
| **Project Structure** | `skills/moe-copy-ai-project-structure/SKILL.md` | Path lookup, module navigation, dependency management, build pipeline |

### Skill-Loading Protocol

Agents should proactively load the appropriate `SKILL.md` based on user intent and task type.

1. **Code review** — When the user requests a review, audit, security check, or pre-PR validation:
   - Load `skills/code-review-expert/SKILL.md`
   - Load checklists from `references/` as needed

2. **Design & refactoring** — When the user mentions module design, API complexity, shallow modules, complexity management, or references *A Philosophy of Software Design*:
   - Load `skills/software-design-philosophy/SKILL.md`
   - Load design principle docs from `references/` as needed

3. **Feature development & quality verification** — When the user works on features, interface refactoring, performance optimization, or needs quality gate confirmation:
   - Load `skills/moe-copy-ai-project-spec/SKILL.md`
   - Always read `references/global-prompt.md` first as the global prompt

4. **Path lookup & structure navigation** — When the user asks "where is this file", "which module to modify", "dependency relationships", or makes cross-module changes:
   - Load `skills/moe-copy-ai-project-structure/SKILL.md`
   - Read `references/project-structure.md` first for entry-point mapping

**Combined loading**: Complex tasks may require multiple skills. For example, a refactoring task should load Design Philosophy + Project Spec + Project Structure together.

---

## 3. Project Constraints

### Architecture Boundaries

- **Dependency flow**: `popup/options/sidepanel → components → hooks → utils → constants` — no circular deps
- **Chrome APIs isolation**: Only `background/` and `contents/` access `chrome.*` directly
- **AI service encapsulation**: All AI calls go through `utils/ai-service.ts`; components use hooks
- **Storage encapsulation**: Chrome storage wrapped in `utils/storage.ts`; components use `useStorage` hook
- **Worker isolation**: Web Worker logic in `utils/workers/`; communication via Comlink
- **i18n**: Always use `useI18n()` hook; update both `locales/en_US.json` and `locales/zh_CN.json`
- **No feature flags**: App is unreleased — directly modify code

### Quality Gates (Required After Code Changes)

```bash
pnpm lint              # Biome checks
pnpm build             # Build verification
```

### Test Acceptance

```bash
pnpm test              # Vitest browser tests (watch mode)
pnpm test:run          # CI mode (single run)
pnpm test:ui           # Vitest UI mode
```

- Test files: `*.browser.ts` or `*.browser.tsx`
- Shared mocks: `utils/__tests__/mocks/`
- Always `resetMockStorage()` in `beforeEach` for isolation
- Use `vi.useFakeTimers()` for time-dependent tests

### Change Impact Map

| Change Scope | Affected Files | Required Verification |
|--------------|---------------|----------------------|
| Content extraction logic | `utils/extractor.ts`, `utils/extractor/` | Tests + build |
| AI service | `utils/ai-service.ts` | Tests + build |
| Message handlers | `background/messages/` | Build + manual test |
| UI components | `components/` | Build + visual check |
| Storage logic | `utils/storage.ts` | Tests + build |
| i18n | `locales/*.json` | Build |
| Worker logic | `utils/workers/` | Tests + build |

---

## 4. Coding Standards

### Design Principles

Follow the core principles from *A Philosophy of Software Design* (see `skills/software-design-philosophy/SKILL.md` for details):

- Modules should be deep: simple interface, powerful implementation
- Information hiding: encapsulate design decisions within a single module
- General-purpose over special-purpose: find the simplest interface that covers all current needs
- Strategic programming: invest 10-20% extra effort in design improvement

### Development Workflow

1. **Identify boundaries**: Determine whether the requirement belongs to UI entry, background, content script, component, hook, util, or constant
2. **Diagnose complexity**: Locate complexity sources (component responsibility mixing, hook dependency weight, utils coupling, message protocol, Worker communication, storage race)
3. **Interface first, then implementation**: Modify interfaces and abstractions before changing implementations
4. **Incremental changes**: Each change addresses only one class of core complexity
5. **Quality verification**: Run quality gates and test acceptance after each change

---

## 5. Skill Extension Guide

When adding a new skill, follow this structure:

```
skills/<skill-name>/
├── SKILL.md                 # Main instruction file (required)
├── README.md                # Skill description (optional)
├── agents/
│   └── agent.yaml           # Agent interface config (optional)
└── references/              # Supporting documents (optional)
    └── *.md
```

After adding a new skill, update:
1. This file (`AGENTS.md`) — add to skill catalog
2. `.claude/skills.md` — add loading entry for Claude Code
3. `.agents/skills.md` — add loading entry for general agents
