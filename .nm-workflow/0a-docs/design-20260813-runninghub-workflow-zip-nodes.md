# RunningHub 在线工作流与 ZIP 解压节点设计

## 文档状态

| 项目     | 值                               |
| -------- | -------------------------------- |
| Task     | `T-20260813-231547-a7k2`         |
| 日期     | 2026-08-13                       |
| 状态     | 待审查，不得据此直接发布         |
| 目标版本 | 审查后确定                       |
| 实施状态 | 未开发、未提交真实 Provider 任务 |

本文是两个新节点的推荐开发方案：`RunningHub 在线工作流` 负责调用
RunningHub Workflow API 并交付归档物，`ZIP 解压` 负责在本地安全解密、解包和分类产物。
本轮只固化设计，不修改产品代码、配置、依赖、能力清单或发布版本。

## 1. 结论

项目当前没有等价工装。已有 `runninghub` 节点封装的是 RunningHub Web App API，结果链路只接受可直接展示的图片、视频、音频和文本；它既不能提交普通在线工作流，也会拒绝或持续等待 ZIP 结果。项目也没有通用 ZIP 解压节点，现有 ZIP 相关代码不能满足密码 ZIP 的需求。

推荐新增两个彼此独立、可以连线组合的节点：

1. `RunningHub 在线工作流`：通过 Workflow API 读取、配置、提交和恢复工作流任务，默认使用海外站 `runninghub.ai`，输出媒体或 `archive` 归档。
2. `ZIP 解压`：接收 `archive`，使用随应用校验和打包的 7-Zip 运行时解密和解包，在严格资源与路径边界内生成可继续连线的素材。

不改造现有 `runninghub` AI App 节点，不放宽现有媒体下载校验器，也不把 ZIP 伪装成 `video`、`model3d` 或 `any`。这能将兼容风险限制在新增能力内。

## 2. 事实依据

### 2.1 已核对的真实工作流

- 页面：<https://www.runninghub.ai/zh-cn/workflow/2085806474898464770>
- 名称：`(Lightx2v T8转化版-4步极速)Minimax双时钟图生视频V1`
- 在线图包含 21 个节点，能够接收用户上传图片。
- 页面中可配置的代表性参数包括画面比例、时长、视频质量以及保存视频的帧率和 CRF。
- 核对时可见的示例配置为 `16:9`、`0.5MP`、`6 秒`、`24 FPS`、`CRF 19`。
- 最终归档节点可配置文件名前缀、文件格式和密码；任务历史中存在成功的 `.zip` 结果。

上述值只作为接口和交互设计的真实样本，不作为新节点的硬编码默认值。工作流中的提示词、访问密码和归档密码均不写入本文。

### 2.2 RunningHub 官方 API

官方文档入口：<https://www.runninghub.ai/runninghub-api-doc-en/>

本设计依赖以下 Workflow API：

| 用途                | 方法与路径                           | 关键数据                                                                                                                                   |
| ------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 获取 API 格式工作流 | `POST /api/openapi/getJsonApiFormat` | `apiKey`、`workflowId`                                                                                                                     |
| 创建工作流任务      | `POST /task/openapi/create`          | `apiKey`、`workflowId`、`nodeInfoList`、可选 `workflow`、`webhookUrl`、`instanceType`、`usePersonalQueue`、`addMetadata`、`accessPassword` |
| 查询状态            | `POST /task/openapi/status`          | `apiKey`、任务 ID                                                                                                                          |
| 查询输出            | `POST /task/openapi/outputs`         | `apiKey`、任务 ID；返回 `fileUrl`、`fileType`、`nodeId` 等                                                                                 |
| 取消任务            | `POST /task/openapi/cancel`          | `apiKey`、任务 ID                                                                                                                          |

`runninghub.ai` 和 `runninghub.cn` 使用相同调用形态，只切换站点基址和对应 API Key。本功能默认 `intl`，但继续复用现有双站路由，不跨站回退密钥。

### 2.3 当前项目差距

- `src/components/nodes/RunningHubNode.tsx` 以 `webappId` 为核心，面向已发布的 AI App。
- `src/services/generation.ts` 和 `backend/src/routes/proxy.js` 当前调用 `/task/openapi/ai-app/run` 相关链路，而不是普通 Workflow API。
- 当前 RunningHub 查询物化只识别图片、视频、音频和文本；ZIP 不属于允许媒体，会被拒绝并可能停留在 `MATERIALIZING`。
- 当前画布端口没有 `archive` 类型，`runninghub` 的输出也只有图片和视频。
- `extract-zip`、`yauzl` 不能满足通用密码 ZIP，间接依赖中的 `7zip-bin` 当前还被 Electron 打包规则排除，不能视为可用产品运行时。
- `backend/src/services/subflowPackage.js` 已有路径穿越、符号链接、数量、大小和压缩比防护思路，但其契约面向 `.t8flow` 且明确拒绝加密包，只能复用安全原则，不能直接复用入口。
- `backend/src/services/materializedOutputStore.js` 已提供哈希、清单和原子提交能力，但大归档需要新增流式写入，而不是把整个文件读入内存。
- `src/utils/comfyuiWorkflow.ts` 可作为字段识别基础，但当前会跳过部分输出节点的普通输入；保存视频和压缩节点参数需要补充识别规则。

## 3. 目标与非目标

### 3.1 目标

- 用户可以用工作流 ID 加载 RunningHub 在线工作流，选择需要暴露的参数，连接本地图片并提交任务。
- 工作流返回 ZIP 时，画布能保留并下载原始 ZIP，并通过类型明确的连线交给解压节点。
- 用户可以输入密码解压 ZIP，得到图片、视频、音频、文本和完整清单。
- Provider 任务在重启后可以查询恢复，已创建任务不能因恢复而重复提交。
- 密钥和密码不出现在画布 JSON、项目数据库、URL、日志、RunEvent、错误文本或遥测中。
- 现有 RunningHub AI App 节点和旧画布保持原行为。

### 3.2 非目标

- 不在首版支持创建、编辑或发布 RunningHub 在线工作流。
- 不在首版自动递归解压嵌套归档。
- 不把解压节点做成任意文件执行器或通用文件管理器。
- 不在本轮方案阶段安装依赖、打包运行时或发起计费任务。
- 不把 `accessPassword`、ZIP 密码或 API Key 放在节点边或普通 metadata 中传递。

## 4. 核心架构决策

| 编号 | 决策                                                        | 原因                                                   |
| ---- | ----------------------------------------------------------- | ------------------------------------------------------ |
| D1   | 新节点类型建议为 `runninghub-workflow` 与 `archive-extract` | 与现有 `runninghub` AI App 节点分离，旧画布零迁移      |
| D2   | 新增共享端口种类 `archive`                                  | ZIP 是一等归档物，不能冒充媒体或 3D 模型               |
| D3   | 海外站 `intl` 为新节点默认值，仍显式支持 `cn`               | 符合当前主要使用域名，且复用现有站点/API Key 路由      |
| D4   | 新建 Workflow 专用后端路由                                  | 避免对 AI App 提交和查询行为产生隐式回归               |
| D5   | ZIP 使用独立、流式、失败关闭的物化器                        | 现有媒体校验器按可预览媒体设计，不应全局放宽           |
| D6   | 密码 ZIP 使用受校验的 7-Zip 运行时                          | 需要兼容常见 ZipCrypto/AES；现有 JS 依赖能力不足       |
| D7   | 工作流访问密码与归档密码是两个独立秘密                      | 两者用途、生命周期和错误模型不同                       |
| D8   | 原始 ZIP 与解压结果都持久化                                 | 便于下载、校验、重试解压和问题追踪                     |
| D9   | 先列目录和预检，再解压到临时目录，最后原子提交              | 防止路径穿越、炸弹包和部分成功污染正式素材目录         |
| D10  | 首版不自动跨节点传递明文 ZIP 密码                           | 端口和持久化契约中不应出现秘密；缺密码时由解压节点询问 |

## 5. 节点一：RunningHub 在线工作流

### 5.1 节点职责

节点负责：读取工作流 API JSON、建立可审查的参数映射、上传本地输入、创建任务、轮询或恢复任务、读取输出列表，以及安全物化返回文件。它不负责解压归档。

### 5.2 用户界面

固定字段：

| 字段           | 行为                                                                      |
| -------------- | ------------------------------------------------------------------------- |
| 站点           | 默认“海外站 · runninghub.ai”；可选国内站                                  |
| 工作流 ID      | 必填；本例为 `2085806474898464770`                                        |
| 工作流访问密码 | 可选、遮罩、仅本次会话；不是 ZIP 密码                                     |
| 加载/刷新字段  | 调用 API 格式接口，展示工作流名称、字段和上次刷新状态                     |
| 输出策略       | 默认“全部可识别输出”；可按输出节点筛选                                    |
| 高级设置       | `instanceType`、`usePersonalQueue`、`addMetadata`；采用官方默认且不臆造值 |

动态字段来自工作流 API JSON，每项必须显示：

- 可读名称、节点 ID、字段名和 RunningHub/ComfyUI 原始类型。
- 本地类型：图片、文本、整数、小数、布尔、枚举或敏感文本。
- 默认值及“是否覆盖远端默认值”开关；未覆盖字段不进入 `nodeInfoList`。
- 输入图片字段可选择“节点上传”或接受 `image` 端口。
- 文件名前缀、视频时长、比例、质量、FPS、CRF 等按原始范围约束展示。
- 名称含 `password`、`token`、`secret`、`key` 或被用户标记敏感的字段必须遮罩且不持久化值。

字段识别不应只靠显示名称。唯一稳定标识为：

```text
workflowId + workflowRevisionFingerprint + nodeId + fieldName
```

当刷新后的指纹变化时：保留仍能精确匹配的映射；把消失、改类型或范围冲突的字段标为“需要确认”，禁止静默提交旧值。

### 5.3 端口

建议端口契约：

| 方向 | ID         | 类型       | 说明                                             |
| ---- | ---------- | ---------- | ------------------------------------------------ |
| 输入 | `image`    | `image`    | 默认图片输入；多个图片字段由字段映射选择目标     |
| 输入 | `text`     | `text`     | 可绑定一个或多个文本字段                         |
| 输入 | `config`   | `config`   | 非秘密的结构化运行覆盖值                         |
| 输出 | `image`    | `image`    | 第一项图片，同时在 metadata 中保留全部列表       |
| 输出 | `video`    | `video`    | 第一项视频，同时在 metadata 中保留全部列表       |
| 输出 | `audio`    | `audio`    | 第一项音频，同时在 metadata 中保留全部列表       |
| 输出 | `text`     | `text`     | 文本输出或工作流返回文本                         |
| 输出 | `archive`  | `archive`  | 第一项已物化归档；支持下载和连接解压节点         |
| 输出 | `metadata` | `metadata` | 任务、输出清单、站点、工作流和追踪信息，不含秘密 |

画布现有单端口 UI 若暂时不能表达同类型多结果，节点主卡片展示全部结果列表，端口输出第一项；`metadata.outputs` 保存有序的全部非秘密描述。不得丢弃第二项及后续结果。

### 5.4 提交流程

```text
读取节点配置
  -> 校验站点、工作流 ID、字段映射和秘密可用性
  -> 上传本地图片并得到 RunningHub fileName
  -> 只生成用户明确覆盖的 nodeInfoList
  -> 创建 Workflow 任务并立即持久化恢复描述符
  -> 查询状态；重启后从查询继续，不重复 create
  -> 成功后查询 outputs
  -> 按声明和文件签名分类，分别物化媒体与 archive
  -> 原子提交结果并结束 RunEvent
```

请求必须由后端注入站点对应 API Key。前端不能收到 API Key，也不能通过 query string 发送工作流访问密码。

### 5.5 推荐后端接口

为避免和现有 AI App 混用，建议增加：

| 本地接口                                     | 作用                                       |
| -------------------------------------------- | ------------------------------------------ |
| `POST /api/proxy/runninghub/workflow-json`   | 获取并规范化 API 格式工作流                |
| `POST /api/proxy/runninghub/workflow-submit` | 上传引用解析完成后创建任务                 |
| `GET /api/proxy/runninghub/workflow-query`   | 合并状态、输出读取和物化进度               |
| `POST /api/proxy/runninghub/workflow-cancel` | 取消 Workflow 任务；底层可复用公共取消实现 |

本地接口响应采用稳定 envelope：

```ts
type RunningHubWorkflowQuery = {
  status:
    | "QUEUED"
    | "RUNNING"
    | "MATERIALIZING"
    | "SUCCEEDED"
    | "FAILED"
    | "CANCELLED";
  taskId: string;
  outputs?: WorkflowArtifact[];
  retryAfterMs?: number;
  error?: { code: string; message: string; retryable: boolean };
};
```

远端状态名称需在后端映射，未知状态失败关闭并保留可诊断的非秘密 Provider 片段。HTTP 200 不等于任务成功；必须同时检查官方业务码、任务状态和输出数组。

### 5.6 任务恢复

新增恢复描述符版本，建议：

```ts
type RunningHubWorkflowRecovery = {
  version: 1;
  kind: "runninghub-workflow";
  site: "intl" | "cn";
  workflowId: string;
  taskId: string;
  requestId: string;
  createdAt: string;
};
```

恢复描述符不得包含 API Key、访问密码、ZIP 密码、签名下载 URL、输入正文或完整 `nodeInfoList`。提交成功但前端断线时，以 `taskId` 恢复；只有在确定远端未创建任务时才能重试创建。

## 6. 归档物契约

### 6.1 新端口类型

`archive` 需要进入前端端口定义、Canvas Node Schema、Agent 公共视图和连线校验的允许集合，但不自动加入所有现有媒体能力集合。只有明确声明接受归档的节点能连接。

建议持久化结构：

```ts
type ArchiveArtifact = {
  version: 1;
  kind: "archive";
  format: "zip";
  url: string; // 本地受控 /api/files URL，不是远端签名 URL
  fileName: string;
  byteSize: number;
  sha256: string;
  source: {
    provider: "runninghub" | "local";
    taskId?: string;
    workflowId?: string;
    nodeId?: string;
  };
};
```

禁止字段：明文密码、远端 API Key、长期远端签名 URL 和用户工作流访问密码。

### 6.2 下载与物化

归档下载走专用 `remoteArchiveMaterializer`：

- 仅允许受 Provider allowlist 和 DNS/IP 重绑定防护校验的 HTTPS URL。
- 逐跳验证重定向，禁止本机、内网、link-local、metadata IP 和非 HTTP(S) 协议。
- 以流式方式写入随机临时文件，同时计算 SHA-256 和字节数；超限立即中止并删除临时文件。
- 校验 ZIP 签名、扩展名与声明类型；HTML、JSON 错误页和空文件必须拒绝。
- fsync/关闭后通过已有物化存储的原子提交边界登记，生成稳定本地 URL。
- Provider 查询重试以 `taskId + output identity` 幂等，不能重复占用磁盘。

不要修改 `validateProxyMediaBuffer` 让它全局接受 ZIP，因为那会降低现有图片、视频和音频的严格类型校验。

## 7. 节点二：ZIP 解压

### 7.1 节点职责与界面

| 字段         | 行为                                                       |
| ------------ | ---------------------------------------------------------- |
| 归档输入     | 接受 `archive` 连线，也可选择本地 `.zip`                   |
| 解压密码     | 遮罩、默认仅本次会话、可为空                               |
| 允许的结果   | 默认图片、视频、音频、文本；首版不允许可执行文件和嵌套归档 |
| 保留目录结构 | 默认开启；输出素材名保留安全的相对路径                     |
| 同名策略     | 默认失败，不静默覆盖或自动合并                             |
| 原始 ZIP     | 始终保留，可从节点结果下载                                 |

节点状态至少区分：等待归档、等待密码、预检、解压、登记素材、完成、密码错误、安全拒绝、资源超限、运行时缺失、已取消。

### 7.2 端口

| 方向 | ID         | 类型       | 说明                                     |
| ---- | ---------- | ---------- | ---------------------------------------- |
| 输入 | `archive`  | `archive`  | 必填 ZIP 归档                            |
| 输出 | `image`    | `image`    | 第一项图片                               |
| 输出 | `video`    | `video`    | 第一项视频                               |
| 输出 | `audio`    | `audio`    | 第一项音频                               |
| 输出 | `text`     | `text`     | 第一项允许的文本                         |
| 输出 | `metadata` | `metadata` | 完整安全清单、分类列表、哈希和拒绝项统计 |

原始 `archive` 已由上游节点持久化，无需从解压节点重复输出；节点卡片仍提供“下载原始 ZIP”。若后续需要级联归档操作，应单独审查后再增加 passthrough 端口。

### 7.3 解压运行时

首版选择官方 7-Zip 命令行运行时或经过许可证与来源审查的等价可再分发二进制，不能依赖当前偶然存在的间接 npm 依赖。

实施要求：

- 运行时版本、平台、架构、来源、许可证、文件大小和 SHA-256 写入独立 manifest。
- 开发态和 Electron packaged 态通过单一 resolver 定位，找不到或哈希不符即失败关闭。
- 打包前、打包后都验证二进制哈希和最小功能；Windows 安装包必须做真实加密 ZIP 冒烟测试。
- 使用 `spawn` 参数数组而不是拼接 shell 命令；日志完全删除密码参数。
- 7-Zip 的密码参数可能在进程存活期间被同机高权限进程看到。首版把这项列为本地信任边界，并通过短生命周期、无 shell、无日志、运行后清空内存引用降低暴露；若审查要求抵御同机进程观察，需要改选支持安全密码输入的库/运行时。

### 7.4 安全解压流程

```text
校验本地 archive 引用和 SHA-256
  -> 7-Zip 列目录，不写磁盘
  -> 解析并验证每个 entry
  -> 在系统临时目录创建单次随机目录
  -> 7-Zip 解压到该目录
  -> lstat + 文件签名 + 大小二次核验
  -> 把允许文件逐项原子登记到项目素材存储
  -> 写入 manifest，删除临时目录
```

预检和事后检查均必须执行。最低拒绝条件：

- 绝对路径、`..`、空路径、NUL、Windows 盘符、UNC、ADS、保留设备名或超长路径。
- 规范化后重复路径、Unicode/大小写折叠冲突、文件与目录同名冲突。
- 符号链接、硬链接、junction、reparse point、设备文件、FIFO 和 socket。
- 可执行文件、脚本、快捷方式、安装包以及首版不支持的嵌套归档。
- 声明大小、实际大小、文件数量、总展开大小、压缩比或目录深度超限。
- 解压后新增未列出文件、条目缺失、哈希读取失败或临时目录逃逸。

建议首版默认限制，最终值是审查项：

| 限制           | 建议默认值 |
| -------------- | ---------- |
| ZIP 下载大小   | 1 GiB      |
| 单文件展开大小 | 2 GiB      |
| 总展开大小     | 4 GiB      |
| 文件条目数     | 512        |
| 目录深度       | 16         |
| 单条规范化路径 | 240 字符   |
| 最大压缩比     | 200:1      |

所有限制应有服务端硬上限；前端配置只能收紧，不能绕过。取消、失败、密码错误和进程崩溃都必须清理临时目录，但不删除已持久化的原始 ZIP。

### 7.5 文件分类

分类依据文件签名和可信探测结果，不能只看扩展名。首版输出：

- `image`：沿用现有图片魔数与子类型校验。
- `video`：沿用现有视频探测和可用性验证。
- `audio`：沿用现有音频探测。
- `text`：只允许有大小上限的 UTF-8 纯文本、JSON、字幕或清单；不得自动执行或渲染 HTML。
- `other`：默认不登记到可连线输出；manifest 记录拒绝原因。若用户确需通用文件，另开设计审查。

解压结果 manifest 必须包括相对路径、分类、本地 URL、字节数、SHA-256、MIME、来源归档 SHA-256 和顺序，不包含密码。

## 8. 密钥与密码生命周期

| 秘密               | 来源       | 使用位置                    | 持久化策略                                 |
| ------------------ | ---------- | --------------------------- | ------------------------------------------ |
| RunningHub API Key | 应用设置   | 后端 Provider 请求          | 沿用现有安全配置，不到前端                 |
| 工作流访问密码     | 工作流节点 | 创建任务的 `accessPassword` | 默认仅本次会话，不进入画布/日志/恢复描述符 |
| 工作流中的敏感参数 | 动态字段   | `nodeInfoList`              | 遮罩且仅本次会话；刷新或重启后要求重填     |
| ZIP 解压密码       | 解压节点   | 7-Zip 进程                  | 默认仅本次会话，不进入画布/日志/manifest   |

保存画布时，秘密字段只保存 `required: true/false`、`hasSessionValue: false` 等非秘密状态。刷新页面或恢复运行时如果缺秘密，任务停在“需要用户输入”，不能把密码错误伪装为 Provider 失败。

首版不通过 `archive` 边传递密码。真实样例工作流的压缩密码已经配置在远端，用户只需在本地解压节点输入同一密码；如用户覆盖远端压缩节点密码，则在两个节点分别输入。后续可设计只存在于后端内存的短期 opaque secret reference 来减少重复输入，但不属于本方案首版。

## 9. 错误模型与可观测性

稳定错误码建议：

| 范围       | 错误码示例                                                                            |
| ---------- | ------------------------------------------------------------------------------------- |
| 工作流读取 | `RH_WORKFLOW_NOT_FOUND`、`RH_WORKFLOW_ACCESS_REQUIRED`、`RH_WORKFLOW_SCHEMA_CHANGED`  |
| 提交与查询 | `RH_AUTH_FAILED`、`RH_QUOTA_EXCEEDED`、`RH_TASK_FAILED`、`RH_OUTPUT_EXPIRED`          |
| 归档物化   | `ARCHIVE_REMOTE_UNSAFE`、`ARCHIVE_TOO_LARGE`、`ARCHIVE_TYPE_MISMATCH`                 |
| 解压       | `ZIP_PASSWORD_REQUIRED`、`ZIP_PASSWORD_INVALID`、`ZIP_UNSUPPORTED_ENCRYPTION`         |
| 安全边界   | `ZIP_PATH_UNSAFE`、`ZIP_ENTRY_FORBIDDEN`、`ZIP_LIMIT_EXCEEDED`、`ZIP_RUNTIME_INVALID` |

RunEvent 记录节点类型、站点、工作流 ID、任务 ID、阶段、耗时、结果数量、字节数、归档/结果 SHA-256 和稳定错误码。所有自由文本先走现有日志脱敏器；不得记录请求体、密码、API Key、远端签名 URL、原始工作流 JSON 或 7-Zip 完整 argv。

## 10. 兼容性与迁移

- 现有节点类型 `runninghub` 和 `runninghub-wallet` 不改名、不迁移、不增加 ZIP 行为。
- 旧 Canvas Node Schema 能继续读取旧画布；新节点由 schema 版本升级明确引入。
- 新增 `archive` 后，所有穷举端口种类的前后端集合与测试必须同步；不能只改前端类型。
- `archive` 默认只能连到 `archive-extract` 或未来明确声明的归档节点。
- 新版打开包含新节点的画布后，旧版应显示“不支持的节点版本”，而不是把它降级成 `any` 后继续运行。
- Provider 输出含媒体和 ZIP 时分别物化；单个输出失败时整个节点失败并保留已下载临时证据的诊断摘要，但不提交半成品结果。

## 11. 预期代码影响面

以下是实施阶段的预期范围，不是本轮修改清单：

| 层            | 预计变更                                                                                             |
| ------------- | ---------------------------------------------------------------------------------------------------- |
| 节点与画布    | 新增两个节点组件、节点注册、默认数据、端口定义、连线规则、完成通知和上下文菜单                       |
| Workflow 解析 | 扩展 `comfyuiWorkflow` 字段发现，覆盖输出节点的可配置输入和敏感字段标记                              |
| 前端服务      | 增加 Workflow JSON/submit/query/cancel 和 archive extract 客户端类型                                 |
| 后端路由      | 增加 Workflow 专用代理路由和本地 ZIP 解压路由                                                        |
| 后端服务      | 增加流式归档物化、归档安全检查、7-Zip resolver 与解压事务服务                                        |
| 恢复与事件    | 增加 `runninghub-workflow` 恢复描述符、状态映射和 RunEvent 数据                                      |
| Schema/Agent  | Canvas Node Schema、端口种类、Agent 工具/公共视图和生成产物同步                                      |
| Electron      | 固定版本的 7-Zip runtime、manifest、打包复制及 post-build 校验                                       |
| 测试          | 节点、端口、Provider、恢复、SSRF、物化、ZIP 攻击样本、Electron 打包和 Windows 冒烟测试               |
| 文档          | `features.json` 生成源、README/节点帮助、`PROJECT_STRUCTURE.md`、项目 Skill 和发布说明按最终能力同步 |

实施时先修改权威源，再运行项目既有生成器；不得手工修补生成能力产物来绕过 `feature-sync`。

## 12. 分阶段实施方案

### WP1：契约与迁移边界

- 固化节点 ID、`archive` 结构、端口和 schema 版本。
- 补齐所有端口穷举集合和连接测试。
- 加入秘密字段序列化拒绝测试。

完成门：旧画布测试不变，新节点可被安全保存但尚不调用 Provider。

### WP2：RunningHub Workflow 节点

- 实现 Workflow JSON 获取、字段映射和缓存指纹。
- 实现上传、提交、查询、取消、恢复和错误映射。
- 实现媒体结果与 ZIP 结果分流；ZIP 使用流式物化。

完成门：mock/录制契约测试全部通过；恢复不会重复创建任务；现有 AI App 测试无回归。

### WP3：ZIP 解压节点

- 固化并校验 7-Zip runtime。
- 实现列表预检、受控解压、事后核验、素材登记和清理。
- 实现密码交互、分类输出、manifest 和重试。

完成门：安全样本矩阵通过，错误和取消不留下临时文件，原 ZIP 可重复解压。

### WP4：产品集成与能力同步

- 完成节点视觉、帮助文本、Agent 可见性、RunEvent 和恢复 UI。
- 更新权威能力资料并运行生成器。
- 完成全量 TypeScript/CJS、lint、schema、writer/lifecycle 和打包门。

### WP5：真实环境验证

- 在用户明确授权和预算范围内，用海外站执行一次低成本真实工作流。
- 验证工作流 ID、字段覆盖、ZIP 返回、密码解压和本地下载。
- 在真实 Windows Electron 安装版验证 7-Zip 运行时、中文路径和重启恢复。

真实 Provider 验证和正式安装包构建都需单独授权；localhost/mock 不能替代这些结论。

## 13. 测试矩阵

### 13.1 RunningHub Workflow

- `.ai` 与 `.cn` 精确路由、密钥隔离、默认 `.ai`。
- 工作流不存在、需要访问密码、密码错误、字段 JSON 不完整和字段版本漂移。
- 图片上传、中文文件名、多输入字段、仅发送显式覆盖值。
- 提交成功、排队、运行、超时、失败、取消、额度不足和未知状态。
- 网络断开、应用重启和查询恢复不重复提交。
- 单媒体、多媒体、纯 ZIP、媒体加 ZIP、空输出和过期 URL。
- 远端 URL SSRF、重定向、HTML 伪装、类型漂移、超限和碰撞幂等。
- 日志、错误、数据库、RunEvent 和画布 JSON 的秘密扫描。

### 13.2 ZIP 解压

- 无密码 ZIP、ZipCrypto、AES、空密码、缺密码、错误密码和不支持加密算法。
- 中文、空格、Emoji、长文件名、嵌套目录和大小写差异。
- `../`、绝对路径、盘符、UNC、ADS、设备名、NUL 和规范化碰撞。
- symlink、hardlink、junction、reparse、FIFO、device、可执行文件和嵌套 ZIP。
- 空包、截断包、CRC 错误、伪 ZIP、条目数/大小/深度/比例边界和炸弹包。
- 取消、7-Zip 崩溃、磁盘已满、并发解压、重复执行和应用重启。
- 事后出现未列出文件、类型与扩展名不符、单项登记失败和原子回滚。
- 图片、视频、音频、UTF-8 文本分类和完整 manifest 顺序。

### 13.3 产品与打包

- 两个节点的新增、复制、删除、保存、恢复、连线和不兼容提示。
- 现有 `runninghub` AI App、媒体节点、3D ZIP 上传和 `.t8flow` 导入无回归。
- 开发态、Windows unpacked、Windows 安装版的 runtime 路径和哈希验证。
- 打包缺二进制、版本不匹配或 manifest 漂移时必须失败关闭。

## 14. 审查前必须确认的决策

1. 接受新增共享端口类型 `archive`，还是首版只做节点内部自动解压。推荐新增端口，以保留组合能力和原始 ZIP。
2. 接受首版密码只在当前会话内存在、刷新后重填。推荐接受，避免明文写入项目。
3. 接受首版只输出图片、视频、音频和受限文本，拒绝通用文件。推荐接受。
4. 接受表中的 1/2/4 GiB、512 条、16 层和 200:1 默认限制，或给出产品目标上限。
5. 接受默认海外站且不自动跨站回退。推荐接受，避免意外使用另一站密钥或资源。
6. 接受 7-Zip 的本机进程参数暴露边界，还是要求改用支持安全密码通道的原生库。
7. 首版动态参数采用“自动识别后由用户确认”，还是仅允许手工添加 `nodeId + fieldName`。推荐自动识别加手工兜底。
8. ZIP 密码是否需要后续增加后端内存态短期 secret reference，以实现相连节点一次输入。

## 15. 实施验收标准

审查通过并完成后续开发时，能力必须同时满足：

- 真实海外站 Workflow API 可运行指定工作流并取得 ZIP，不依赖 Web App 发布。
- ZIP 以 `archive` 一等产物保存、下载和连线，远端签名 URL 不成为长期数据。
- 加密 ZIP 在真实 Windows 安装版可解密；密码错误和安全拒绝有稳定、可理解的错误。
- 路径、链接、压缩炸弹、SSRF、磁盘上限、秘密泄漏和部分提交测试失败关闭。
- Provider 任务可恢复且不重复提交，取消和重试保持幂等。
- 现有 RunningHub AI App、旧画布、其他媒体节点和 `.t8flow` 功能无回归。
- 权威 schema、能力资料、文档、测试和 Electron 打包校验同步完成。

在第 14 节审查项得到确认前，Task 保持 `ready`，不得进入产品实现或真实 Provider 提交。
