---
id: T-20260813-231547-a7k2
status: ready
depends_on: []
---

# RunningHub 在线工作流与 ZIP 解压节点设计

## 目标与验收

在不开发产品代码的前提下，基于真实 RunningHub 工作流、官方 API 文档和当前 v2.9.0 实现，形成一份可审查、可直接指导后续开发的正式设计方案。

验收条件：

- 明确新增 `RunningHub 在线工作流` 与 `ZIP 解压` 两个节点的职责、端口和用户流程。
- 明确 RunningHub 海外站 Workflow API、结果归档物化和任务恢复契约。
- 明确密码 ZIP 的运行时选择、安全边界、错误模型和 Electron 打包要求。
- 明确与现有 AI App、ComfyUI、RunEvent、素材持久化和能力清单的兼容策略。
- 给出分阶段实施范围、文件影响面、测试矩阵和审查决策项。
- 方案分支仅包含文档和工作流记录，不包含节点实现或 Provider 真实提交。

## TODO

- [x] 核对真实 RunningHub 工作流、官方 Workflow API 与当前节点实现差距。
- [x] 编写正式设计方案。
- [x] 同步项目结构文档。
- [x] 执行 Markdown、结构、JSON 与差异检查。
- [x] 将 Task 标记为 `ready` 并准备本地提交，等待方案审查。

## 验证

- 通过：Prettier 格式化和 `--check` 覆盖本 Task、正式设计与项目结构 Markdown。
- 通过：`npx markdownlint` 覆盖本 Task、正式设计与项目结构 Markdown。
- 通过：`npm run worktree:check` 与 `npm run worktree:development`。
- 通过：`npm run feature-sync:check`，77/77 节点已被当前能力产物覆盖，文档变更未造成漂移。
- 通过：`node -e` 解析 `features.json`、`package.json` 和 `backend/src/shared/canvasNodeSchema.json`。
- 通过：`git diff --check` 与精确差异复核。
- 未运行产品测试、构建或真实 RunningHub 任务：本分支只有方案和工作流文档，没有产品实现。
