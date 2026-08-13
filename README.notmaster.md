# T8-penguin-canvas 个人开发指南

这份文档只服务于我自己的 fork 和本地开发，不替代上游的 `README.md`。

目标只有三个：快速把项目跑起来、知道代码应该改哪里、能够持续同步上游而不弄乱自己的开发分支。

## 我的仓库布局

```text
本地目录
/Users/jango/code/osc/workspace-T8-penguin-canvas/T8-penguin-canvas

origin
https://github.com/notmaster/T8-penguin-canvas.git

upstream
https://github.com/T8mars/T8-penguin-canvas.git
```

- `main`：保持与上游 `upstream/main` 一致，不放个人功能。
- `dev`：我的个人集成分支，汇总已经验收的功能。
- `codex/task/...`：正常开发任务分支，从最新 `dev` 创建。
- `codex/docs/...`：一次性轻量文档分支。
- 详细规则见 `AGENTS.md` 和 `.nm-workflow/RULES.md`。

不要直接在 `main`、`master` 或 `dev` 上开发功能。

## 首次安装

项目使用 npm 和 `package-lock.json`，不要混用 pnpm 或 yarn。

```bash
cd /Users/jango/code/osc/workspace-T8-penguin-canvas/T8-penguin-canvas
npm install
```

项目没有在 `package.json` 中固定 Node `engines`。当前在 macOS arm64 上实际验证过的环境是 Node `v24.19.0`、npm `11.17.0` 和 Electron `33.4.11`。

### Apple Silicon 的 Electron 原生依赖

项目的 `npm run rebuild:electron` 当前写死为 `--arch x64`，不要在 Apple Silicon Mac 上直接运行它。使用：

```bash
npx electron-rebuild -f -w better-sqlite3 --arch arm64
```

典型报错如下，出现时执行上面的命令后重启后端：

```text
ProjectDatabaseOwnerUnavailableError:
本机 SQLite 原生依赖与当前 Electron 版本不匹配
```

Intel Mac 或 Windows x64 才使用项目原命令：

```bash
npm run rebuild:electron
```

如果 npm 提示某些第三方安装脚本等待批准，不要一次性盲目批准所有脚本。先确认是项目需要的 Electron、esbuild、sharp、better-sqlite3 或 FFmpeg 依赖，再按 npm 提示处理。

## 启动项目

### 1. 先确认不在受保护分支

```bash
git status -sb
npm run worktree:check
npm run worktree:development
```

如果看到下面的错误，说明还在 `dev` 或分支名不符合上游门禁：

```text
development requires a non-release codex/* branch
```

仅查看项目时可以从 `dev` 建本地预览分支：

```bash
git switch dev
git switch -c codex/preview/run-project
```

正式修改功能时，按 `.nm-workflow/RULES.md` 创建带 Task ID 的 `codex/task/...` 分支。

### 2. 启动完整 Web 开发环境

```bash
npm run dev
```

这个命令同时启动：

| 服务         | 地址                                | 说明                                    |
| ------------ | ----------------------------------- | --------------------------------------- |
| Web 前端     | <http://127.0.0.1:11422/>           | Vite + React 开发页面                   |
| 后端         | <http://127.0.0.1:18766/>           | Express，由 Electron 的 Node 运行时承载 |
| 后端状态     | <http://127.0.0.1:18766/api/status> | 返回版本、端口和实例状态                |
| Figma Bridge | <http://127.0.0.1:3845/>            | 后端启动时自动启动                      |

页面顶部显示“后端已连接”才表示前后端都正常。

停止服务时，在运行 `npm run dev` 的终端按 `Ctrl+C`。

### 3. 分开启动前后端

需要单独观察日志时，开两个终端。

终端一，后端热更新：

```bash
npm run dev:backend
```

终端二，前端：

```bash
npm run dev:vite
```

不需要后端热更新时，可将后端命令替换为：

```bash
npm run start:backend
```

### 4. 启动 Electron 桌面端

先确保 Web 开发环境已经运行，再开新终端：

```bash
npm run electron:dev
```

Electron 启动前也会执行 worktree 开发门禁。

## 端口和进程检查

```bash
lsof -nP -iTCP:11422 -sTCP:LISTEN
lsof -nP -iTCP:18766 -sTCP:LISTEN
lsof -nP -iTCP:3845 -sTCP:LISTEN
curl -fsS http://127.0.0.1:18766/api/status
```

Vite 使用 `strictPort: true`。如果 `11422` 被占用，项目不会自动换端口，需要先停止占用者。

## 代码应该改哪里

| 想修改的内容          | 主要位置                                          |
| --------------------- | ------------------------------------------------- |
| 前端入口与整体布局    | `src/main.tsx`、`src/App.tsx`                     |
| 无限画布主体          | `src/components/Canvas.tsx`                       |
| 节点 UI 与行为        | `src/components/nodes/`                           |
| 通用前端组件          | `src/components/`                                 |
| Hooks、状态和服务     | `src/hooks/`、`src/stores/`、`src/services/`      |
| 前端 Provider 适配    | `src/providers/`                                  |
| 样式和主题            | `src/styles/`、`src/theme/`                       |
| 后端入口和路由挂载    | `backend/src/server.js`                           |
| 后端配置与端口        | `backend/src/config.js`                           |
| API 路由              | `backend/src/routes/`                             |
| Provider 与业务服务   | `backend/src/providers/`、`backend/src/services/` |
| Electron 主进程与桥接 | `electron/main.cjs`、`electron/preload.cjs`       |
| 自动化脚本与门禁      | `scripts/`                                        |
| CLI、Figma 和外部工具 | `tools/`                                          |
| 测试                  | `tests/`                                          |
| 产品能力权威记录      | `features.json`                                   |

`src/generated/` 和工具生成的 capability 文件优先通过现有生成脚本更新，不要把生成结果当作唯一源码手工维护。

## 修改一个功能的标准流程

### 1. 从最新 dev 建任务分支

先生成符合规则的 Task ID，再执行：

```bash
git switch dev
git fetch origin
git pull --ff-only origin dev
git switch -c codex/task/T-YYYYMMDD-HHmmss-xxxx-short-slug
```

随后在 `.nm-workflow/0b-tasks/active/` 创建对应 Task 文件，再开始修改。

### 2. 修改前找实现和测试

```bash
rg "界面文案或函数名" src backend tests
rg --files src backend tests | sort
```

先读相关实现和已有测试，再做最小完整改动。不要通过复制整个目录覆盖代码。

### 3. 运行与风险相称的验证

常用基础门禁：

```bash
npm run worktree:check
npm run worktree:development
npm run type-check
npm run lint
```

运行一个精确测试文件：

```bash
node --test tests/example.test.cjs
```

TypeScript 测试需要按仓库现有 Electron loader 方式运行；全量入口是：

```bash
npm run test:electron:ts
```

修改节点、能力清单或生成面时，再运行：

```bash
npm run feature-sync:check
```

`npm run build`、打包、真实 Provider 验证、版本升级、Tag 和 Release 都不是日常默认动作，只在任务范围明确需要并获得授权后执行。

### 4. 检查并提交

```bash
git status -sb
git diff --check
git diff
git add <本次任务的精确文件>
git diff --cached
git commit -m "T-YYYYMMDD-HHmmss-xxxx: 简短说明"
```

不要默认使用 `git add -A`，不要把本地数据、凭据或无关改动提交进去。默认不 push、不合并；需要推送或合并到 `dev` 时再明确执行。

## 同步上游更新

`origin` 是自己的 fork，`upstream` 是原项目。保留这个远端布局，不要互换。

先确认工作树干净，再更新个人集成分支：

```bash
git status -sb
git switch dev
git fetch upstream --prune
git log --oneline --left-right dev...upstream/main
git merge upstream/main
```

完成必要验证后，再同步到自己的 fork：

```bash
git push origin dev
```

如果当前有任务分支，需要把更新后的 `dev` 带回任务：

```bash
git switch codex/task/T-YYYYMMDD-HHmmss-xxxx-short-slug
git merge dev
```

不要把上游变更直接混入一个状态不明或有未提交修改的工作树。发生冲突时逐文件理解两边语义，不使用整树 `ours` 或 `theirs`。

## 本地数据和秘密

以下目录主要是运行时数据或产物，已被 Git 忽略，不应提交：

```text
data/
input/
output/
thumbnails/
userdata/
backend/data/
```

本机还会使用：

```text
/Users/jango/zhenzhen
/Users/jango/zhenzhen/resources
/Users/jango/zhenzhen/theme-templates
```

- 不读取或改动 retained/historical 项目数据库。
- 数据库测试只能在系统临时目录创建测试数据库，并在测试后清理。
- `.env*`、API Key、Cookie、签名 URL、任务原始响应和私有配置不能提交。
- 未明确授权时，不调用真实付费 Provider，不操作生产系统或真实外部数据。

## 明确禁止的操作

- 不覆盖或精简上游官方 `README.md`。
- 不直接在 `main`、`master` 或 `dev` 上开发功能。
- 不使用 `git reset --hard`、`git clean`、checkout 覆盖或整树复制。
- 不编辑或暂存 `tools/ffmpeg-runtime/ffmpeg.exe`。
- 不编辑或暂存 `tools/remove-ai-watermarks-runtime/README.md`。
- 不擅自执行版本升级、正式打包、Tag、GitHub Release 或生产发布。

## 常见问题

### 页面能打开，但显示后端未连接

1. 检查 `18766` 是否监听。
2. 请求 `http://127.0.0.1:18766/api/status`。
3. 查看后端终端的第一条实际异常。
4. 如果是 `ERR_DLOPEN_FAILED`，按当前 Mac 架构重建 `better-sqlite3`。

### 前端启动时报端口被占用

```bash
lsof -nP -iTCP:11422 -sTCP:LISTEN
```

确认进程属于本项目后，在原终端用 `Ctrl+C` 停止。不要对不明进程直接执行强制结束。

### Electron 或原生模块安装不完整

先确认架构：

```bash
uname -m
node -p "process.platform + ' ' + process.arch"
ELECTRON_RUN_AS_NODE=1 ./node_modules/.bin/electron -p "process.platform + ' ' + process.arch"
```

再按“Apple Silicon 的 Electron 原生依赖”一节处理，不要在 arm64 上误用 `--arch x64`。

### 不知道一次修改要跑哪些测试

先运行和改动文件同名或同领域的精确测试，再运行 `type-check` 和相关门禁。只有功能范围确实跨模块时才扩大测试范围；不要用一个无关的全量命令代替精确验证。
