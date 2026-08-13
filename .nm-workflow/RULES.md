# Rules-mini-v1.2.1

本文件由个人规则模板同步而来，并针对当前 fork 的 npm 生态、远端和分支布局做了最小适配。若本文件与根目录 `AGENTS.md` 的项目安全、产品版本或发布约束冲突，以更严格的约束为准；本 fork 的本地路径、远端和分支路由以根目录 `AGENTS.md` 的“notmaster fork 本地工作流”为准。

## 通用原则

- 沿用本项目现有的 `npm` 和 `package-lock.json`；包管理器迁移必须作为独立任务处理。
- 默认使用简体中文沟通；时间默认使用 UTC+8。
- 先检查仓库事实并遵循现有约定；只有无法自行确认且会显著影响目标、范围、验收或安全时才询问。
- 在授权范围内自主推进至完成；只做必要且完整的改动，不触碰无关内容，不覆盖无法解释的既有改动。
- 修改后执行与风险相称的验证并报告结果；无法验证时明确说明缺口、原因和影响。
- 未经明确授权，不改变生产环境、外部系统或真实数据，不外发敏感信息，不执行不可逆操作。

## Git 与分支

- `main`、`master` 和 `dev` 是受保护分支；除非管理员明确要求，不直接在其上开发。
- 开发任务从最新可用的 `dev` 创建独立任务分支或 worktree；先 fetch 并核对 `origin/dev`，同步上游时另行 fetch 并核对 `upstream/main`。不存在 `dev`、无法确定正确远程或基线状态异常时，不自行创建或替代集成分支，先询问管理员。
- 一个 Task 对应一个分支或 worktree；先生成 Task ID，再从 `dev` 创建 `codex/task/<Task-ID>-<slug>`，随后在该分支创建 Task 文件并开始实施。`codex/` 前缀用于兼容上游现有的 worktree 开发保护门。
- 启用并行 Work Package 时，正式 Task 分支同时作为集成分支；内部工作分支使用 `codex/wp/<Task-ID>/<WP-ID>-<slug>`，只能从依赖已合入后的 Task 集成分支创建，不直接从 `dev` 创建。
- 每个写入型 Work Package 使用独立 worktree；同一 worktree 同时只能有一个写入者。主 Agent 只将验收通过的 Work Package 合入 Task 集成分支，不把中间成果合入受保护分支。
- 验证通过后默认创建本地 commit；提交前检查差异，只提交当前 Task 可解释的改动，commit message 以 Task ID 开头。用户明确要求不提交时除外。
- 默认不合并、不 push；合并到受保护分支、push、改写历史或删除远程引用必须获得管理员明确授权。获授权集成前必须重新 fetch 并核对目标远程 SHA。
- 未合并的任务分支或 worktree 必须保留；只有 Task 已合并或管理员明确取消后才能清理。

## 工作流文档

- 工作流产物统一存放在 `.nm-workflow/`；`0a-docs/` 保存长期有效的需求、设计、决策和报告，`0b-tasks/active/` 保存进行中、阻塞或待合并的 Task，`0b-tasks/archive/` 保存已合并或取消的 Task，`0c-work-packages/` 保存 Task 内部的并行工作包，`templates/` 保存可复用工作流模板。
- 讨论、咨询和只读检查不创建 Task。开发变更和需要跨会话跟踪的工作必须创建 Task；一次性轻量文档修正可使用独立 `codex/docs/<slug>` 分支而不创建 Task，除非用户另有要求。
- 长期文档命名为 `<type>-YYYYMMDD-<slug>.md`，`type` 使用 `req | design | decision | report`；不使用版本后缀，版本由 Git 记录。

## Task 规范

- Task ID 为 `T-YYYYMMDD-HHmmss-xxxx`，`xxxx` 是四位小写字母或数字；创建前扫描 `active/` 和 `archive/`，冲突时重新生成。ID 创建后不变且不得重用。
- Task 文件名为 `task-<Task-ID>-<slug>.md`，`slug` 使用小写 ASCII kebab-case，文件在状态目录间移动时不改名。
- frontmatter 只保留 `id`、`status` 和 `depends_on`；`status` 为 `active | blocked | ready | done | cancelled`，`depends_on` 使用 Task ID 列表。
- `ready` 表示开发、验证和提交已完成但尚未合并；`done` 仅表示已合并到 `dev`。
- 正文至少包含“目标与验收”“TODO”和“验证”；TODO 只使用 `[ ]` 和 `[x]`，阻塞、迁移或高风险任务按需增加必要章节。
- Task 只有在 `depends_on` 全部为 `done` 后才能开始；依赖不得自指、重复或成环。
- `active`、`blocked` 和 `ready` 保留在 `active/`；获授权合并时将 Task 设为 `done` 并把原文件移入 `archive/`，`cancelled` 也移入 `archive/`。

## Work Package 规范

- Work Package 是单个正式 Task 内部的可并行执行单元，不是独立 Task，不使用 Task 状态或替代 Task 依赖；只在工作确实能安全并行时启用。
- Work Package ID 在所属 Task 内使用 `WP-NN`；文件名为 `wp-<NN>-<slug>.md`，存放在 `.nm-workflow/0c-work-packages/active/<Task-ID>/`。frontmatter 只保留 `id`、`task_id` 和 `depends_on`。
- 正文至少包含“目标与边界”“输入与依赖”“路径所有权”“验收与测试”和“交付回报”。路径所有权必须明确可写范围、只读共享契约和禁止触碰的共享热点。
- 依赖只有在前置 Work Package 已验证、提交并合入 Task 集成分支后才算满足；工作分支和 worktree 按 ready queue 即时创建，避免基于过期基线提前铺开。
- 主 Agent 是编排器，不编写业务代码或交付文档，也不亲自解决内容冲突；它负责 DAG、分支与 worktree、原生 Agent 协调、证据检查、合并和通知。实现、测试、文档、缺陷修复及冲突解决均交给有明确所有权的子 Agent。
- 每个普通 Work Package 必须设置精确的非 E2E 测试门槛；开发阶段不运行 E2E。最终验收 Work Package 依赖全部功能终点，先跑构建和全量非 E2E 门禁，再运行仓库规定的可重复 E2E 套件与当前 Agent 平台的浏览器自动化黑盒验收；本项目的可重复 E2E 套件为 Playwright。
- 测试失败由主 Agent 退回原 Work Package 或派发独立修复包，修复后从受影响门禁重跑。只有权限、凭据、外部系统、权威文档互斥或同一根因经过至少三种有效修复仍不收敛时才通知管理员介入。
- Task 进入 `done` 或 `cancelled` 时，将其 Work Package 目录原样移入 `.nm-workflow/0c-work-packages/archive/<Task-ID>/`；此前保留相关分支和 worktree。

## 文档格式化与同步

- 将 `prettier` 和 `markdownlint-cli` 安装到 `devDependencies`；在根目录维护 `.markdownlintignore`、`.markdownlint.json` 和 `.prettierignore`，其中 `.prettierignore` 必须排除 `**/AGENTS.md`。

```json
{
  "default": true,
  "MD013": false,
  "MD033": false,
  "MD025": {
    "front_matter_title": ""
  }
}
```

- `package.json` 提供以下 scripts：

```json
{
  "scripts": {
    "fm": "prettier \"**/*.md\" --write",
    "lm": "markdownlint --dot \"**/*.md\""
  }
}
```

- Markdown 变更必须执行覆盖实际变更文件的格式化和 lint；仓库级命令只有在确认覆盖目标文件时才能作为验收依据。`AGENTS.md` 只执行 Markdown lint 并人工复核，不由 Prettier 改写。
- 文件或目录的增删、移动、重命名或职责变化时同步 `PROJECT_STRUCTURE.md`；项目定位、安装、使用、命令、配置、入口或用户可见行为变化时同步 `README.md`。Task 归档前必须检查二者，无需更新时在最终交付中说明原因。

---

## 发送通知

- 本节构成对以下飞书通知的预授权。仅在 `$nm-notify-feishu` SKILL 可用且配置有效时执行；通知失败不改变 Task 状态，不递归发送失败通知，只在最终回复中说明。
- 通知以用户发起的整项工作为单位；内部 TODO、测试阶段、工具调用或重试不单独通知。
- 整项工作无法安全继续或确实需要人工介入时，使用 `warning` scene 发送一次通知，由配置路由到 attention 频道。
- 本次工作创建了 Task，或整项工作持续超过 4 分钟时，在最终目标完成且验证结束后使用 `success` scene 发送一次通知。
- Task 处于 `ready` 时，通知必须注明“开发完成，待合并”；只有进入 `done` 后才能表述为“已合并完成”。

## 本项目规则

- 根目录 `AGENTS.md` 与 `.agents/skills/zhenzhen-canvas/SKILL.md` 是必读入口；`features.json`、`package.json`、相关源码、测试和当前交接记录按任务范围读取。
- 开发前执行 `npm run worktree:check` 与 `npm run worktree:development`。`dev` 是受保护集成分支，开发门在 `dev` 上拒绝执行属于预期行为；新 Task、Work Package 和轻量文档分支必须使用上述 `codex/` 前缀。若上游脚本仍不识别本 fork 的 macOS 路径或这些开发分支，应把适配作为独立 Task，不能绕过或删除保护门。
- 不读取 retained/historical 项目数据库；数据库测试只能使用系统临时目录并在测试后清理。
- 未经明确授权，不执行生产构建、打包、版本升级、Tag、GitHub Release 或真实 Provider 调用。
