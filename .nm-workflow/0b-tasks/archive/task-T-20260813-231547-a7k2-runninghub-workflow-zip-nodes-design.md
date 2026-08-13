---
id: T-20260813-231547-a7k2
status: done
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
- [x] 完成首轮方案审查并记录阻断项。
- [x] 按审查意见修订提交幂等、字段来源、秘密隔离、站点粘连、流式事务和安全解压契约。
- [x] 补充 macOS、Windows 与 Linux 的跨平台归档运行时契约和实机可用性依据。
- [x] 重新执行文档与仓库校验，将 Task 恢复为 `ready` 并提交修订。
- [x] 方案审查通过，获授权合入 `dev` 并归档 Task。

## 首版方案验证

- 通过：Prettier 格式化和 `--check` 覆盖本 Task、正式设计与项目结构 Markdown。
- 通过：`npx markdownlint` 覆盖本 Task、正式设计与项目结构 Markdown。
- 通过：`npm run worktree:check` 与 `npm run worktree:development`。
- 通过：`npm run feature-sync:check`，77/77 节点已被当前能力产物覆盖，文档变更未造成漂移。
- 通过：`node -e` 解析 `features.json`、`package.json` 和 `backend/src/shared/canvasNodeSchema.json`。
- 通过：`git diff --check` 与精确差异复核。
- 未运行产品测试、构建或真实 RunningHub 任务：本分支只有方案和工作流文档，没有产品实现。

## 首轮审查修订验证

- 通过：RunningHub create 无幂等键、API prompt 非完整 schema、站点范围和大文件流式边界均已转为显式失败关闭契约。
- 通过：原始 workflow JSON、会话秘密、config 冲突和复制/导出/协作/恢复泄漏边界已纳入设计与测试矩阵。
- 通过：官方 7-Zip 下载页确认提供 macOS arm64/x86-64 universal console runtime；本机 Apple Silicon 临时运行 26.02 `7zz` 成功，未安装系统软件或修改仓库。
- 通过：`npm run worktree:check`、`npm run worktree:development` 和 `npm run feature-sync:check`。
- 通过：Prettier、markdownlint、JSON/schema 解析与 `git diff --check`。
- 未运行产品测试、Electron 构建、归档攻击样本或真实 RunningHub 任务；这些属于后续开发和独立授权范围。

## 合并结论

- 2026-08-13：用户确认方案审查通过并授权合入 `dev`、以非强制方式推送 `origin/dev`。
- 合并前重新 fetch 并确认本地 `dev` 与 `origin/dev` 同为 `006f8b011f9b6a85ee4647fadd245b69db3f8943`，且该提交是任务分支祖先。
- 受保护文件在任务分支、本地 `dev` 与 `origin/dev` 的 Git blob 一致，本任务未修改或暂存它们。
