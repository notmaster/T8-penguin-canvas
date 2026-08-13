# RunningHub 在线工作流与 ZIP 解压节点设计

## 文档状态

| 项目     | 值                               |
| -------- | -------------------------------- |
| Task     | `T-20260813-231547-a7k2`         |
| 日期     | 2026-08-13                       |
| 状态     | 已按首轮审查修订，待复审         |
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

首版固定使用已经稳定投入使用的 Workflow v1 协议，Provider 适配器版本标识为
`runninghub-workflow-v1`。不得把文档中仍标记为 Developing 的
`/openapi/v2/query` 或 V2 上传接口混入同一次任务生命周期。首版依赖：

| 用途                | 方法与路径                           | 关键数据                                                                                                                                   |
| ------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 获取 API 格式工作流 | `POST /api/openapi/getJsonApiFormat` | `apiKey`、`workflowId`                                                                                                                     |
| 上传工作流输入      | `POST /task/openapi/upload`          | `apiKey`、文件；官方 v1 上限 30 MiB，返回站点内可用的 `fileName`                                                                           |
| 创建工作流任务      | `POST /task/openapi/create`          | `apiKey`、`workflowId`、`nodeInfoList`、可选 `workflow`、`webhookUrl`、`instanceType`、`usePersonalQueue`、`addMetadata`、`accessPassword` |
| 查询状态            | `POST /task/openapi/status`          | `apiKey`、任务 ID                                                                                                                          |
| 查询输出            | `POST /task/openapi/outputs`         | `apiKey`、任务 ID；返回 `fileUrl`、`fileType`、`nodeId` 等                                                                                 |
| 取消任务            | `POST /task/openapi/cancel`          | `apiKey`、任务 ID                                                                                                                          |

`outputs` 是首版任务查询的权威来源，适配器必须覆盖官方业务码 `0`、`804`、
`813` 和 `805`；`status` 只作兼容诊断，不能与 V2 查询结果拼接出新的状态语义。

`runninghub.ai` 和 `runninghub.cn` 使用相同调用形态，只切换站点基址和对应 API Key。
本功能默认 `intl`，但一次运行在首次上传前冻结站点，后续上传、创建、查询、输出、取消
全部使用同一站点和对应密钥，禁止自动跨站回退。

### 2.3 当前项目差距

- `src/components/nodes/RunningHubNode.tsx` 以 `webappId` 为核心，面向已发布的 AI App。
- `src/services/generation.ts` 和 `backend/src/routes/proxy.js` 当前调用 `/task/openapi/ai-app/run` 相关链路，而不是普通 Workflow API。
- 当前 RunningHub 查询物化只识别图片、视频、音频和文本；ZIP 不属于允许媒体，会被拒绝并可能停留在 `MATERIALIZING`。
- 当前画布端口没有 `archive` 类型，`runninghub` 的输出也只有图片和视频。
- `extract-zip`、`yauzl` 不能满足通用密码 ZIP，间接依赖中的 `7zip-bin` 当前还被 Electron 打包规则排除，不能视为可用产品运行时。
- `backend/src/services/subflowPackage.js` 已有路径穿越、符号链接、数量、大小和压缩比防护思路，但其契约面向 `.t8flow` 且明确拒绝加密包，只能复用安全原则，不能直接复用入口。
- `backend/src/services/materializedOutputStore.js` 已提供哈希、清单和原子提交能力，但大归档需要新增流式写入，而不是把整个文件读入内存。
- 现有 `commitMaterializedOutputBuffer` 以 `Buffer` 为入口，不能承载本设计 1/2/4 GiB 上限；归档下载、校验、分类和最终提交都必须增加文件流接口。
- 现有 RunningHub 候选站点逻辑会在 `.ai` 与 `.cn` 间自动尝试；新节点不得复用该 fallback，必须使用精确站点 resolver。
- 画布保存会整体持久化 `node.data`；因此把密码写入节点 React state 之外的普通数据结构仍会泄漏，必须使用独立的会话秘密存储。
- `src/utils/comfyuiWorkflow.ts` 可作为字段识别基础，但当前会跳过部分输出节点的普通输入；保存视频和压缩节点参数需要补充识别规则。

## 3. 目标与非目标

### 3.1 目标

- 用户可以用工作流 ID 加载 RunningHub 在线工作流，选择需要暴露的参数，连接本地图片、视频或音频并提交任务。
- 工作流返回 ZIP 时，画布能保留并下载原始 ZIP，并通过类型明确的连线交给解压节点。
- 用户可以输入密码解压 ZIP，得到图片、视频、音频、文本和完整清单。
- 取得 `taskId` 的 Provider 任务在重启后只能查询恢复，不能因恢复而重复提交；提交结果不明时停止自动重试并明确提示潜在重复计费。
- 密钥和密码不出现在画布 JSON、项目数据库、URL、日志、RunEvent、错误文本或遥测中。
- 现有 RunningHub AI App 节点和旧画布保持原行为。

### 3.2 非目标

- 不在首版支持创建、编辑或发布 RunningHub 在线工作流。
- 不在首版自动递归解压嵌套归档。
- 不在首版把本地 ZIP 上传为 RunningHub 工作流输入；这需要独立的流式上传契约。
- 不在首版提供 ZIP 节点本地文件选择器；首版只接受已安全物化的 `archive` 端口。
- 不使用仍在 Developing 的 RunningHub V2 查询或上传接口。
- 不承诺 Provider 未提供幂等键时仍能实现跨网络故障的严格 exactly-once 创建。
- 不把解压节点做成任意文件执行器或通用文件管理器。
- 不在本轮方案阶段安装依赖、打包运行时或发起计费任务。
- 不把 `accessPassword`、ZIP 密码或 API Key 放在节点边或普通 metadata 中传递。

## 4. 核心架构决策

| 编号 | 决策                                                        | 原因                                                    |
| ---- | ----------------------------------------------------------- | ------------------------------------------------------- |
| D1   | 新节点类型建议为 `runninghub-workflow` 与 `archive-extract` | 与现有 `runninghub` AI App 节点分离，旧画布零迁移       |
| D2   | 新增共享端口种类 `archive`                                  | ZIP 是一等归档物，不能冒充媒体或 3D 模型                |
| D3   | 海外站 `intl` 为默认值，仍支持 `cn`，但使用精确 resolver    | 符合主要域名且避免继承现有自动 fallback                 |
| D4   | 新建 Workflow 专用后端路由                                  | 避免对 AI App 提交和查询行为产生隐式回归                |
| D5   | ZIP 使用独立、流式、失败关闭的物化器                        | 现有媒体校验器按可预览媒体设计，不应全局放宽            |
| D6   | 密码 ZIP 使用受校验的 7-Zip 运行时                          | 需要兼容常见 ZipCrypto/AES；现有 JS 依赖能力不足        |
| D7   | 工作流访问密码与归档密码是两个独立秘密                      | 两者用途、生命周期和错误模型不同                        |
| D8   | 原始 ZIP 与解压结果都持久化                                 | 便于下载、校验、重试解压和问题追踪                      |
| D9   | 先列目录和预检，再解压到临时目录，最后原子提交              | 防止路径穿越、炸弹包和部分成功污染正式素材目录          |
| D10  | 首版不自动跨节点传递明文 ZIP 密码                           | 端口和持久化契约中不应出现秘密；缺密码时由解压节点询问  |
| D11  | 一次任务冻结精确站点，禁止 `.ai`/`.cn` fallback             | 上传句柄、任务和密钥都具有站点范围，跨站会失败或误计费  |
| D12  | 首版固定 `runninghub-workflow-v1`                           | 避免混用仍在开发中的 V2 接口和不同状态语义              |
| D13  | 提交结果不明进入 `SUBMISSION_UNKNOWN`，绝不自动重试         | create 没有远端幂等键，本地 request ID 无法消除重复任务 |
| D14  | 原始工作流只在后端有界解析并脱敏，秘密只进会话存储          | 工作流 JSON 可能包含压缩密码，画布会持久化全部节点数据  |
| D15  | 归档全程流式处理，解压结果按批次事务发布                    | 避免大 Buffer 和部分结果进入正式素材库                  |
| D16  | 归档运行时按 OS/架构分别随包交付并校验                      | macOS、Windows、Linux 的二进制和打包路径不同            |

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

- 可读名称、节点 ID、字段名、当前值、推断类型和推断置信度。
- 本地类型：图片、视频、音频、文本、整数、小数、布尔、枚举或敏感文本。
- 默认值及“是否覆盖远端默认值”开关；未覆盖字段不进入 `nodeInfoList`。
- 媒体字段可选择“节点上传”或接受同类型端口。
- 文件名前缀、视频时长、比例、质量、FPS、CRF 等只在有权威范围时按范围约束；没有范围时展示当前值、推断类型和“范围未知”。
- 明确的节点类/字段规则、敏感关键词或用户手工标记命中的字段必须遮罩且不持久化值；未知字段保守处理。

`getJsonApiFormat` 返回的 `data.prompt` 是 API-format prompt，不是完整控件 schema。字段映射分三层：

1. 权威层：读取 `nodeId`、`fieldName`、当前原始值和连接关系。
2. 推断层：根据原始值、节点类和既有规则推断媒体、文本、数值、布尔及常见枚举。
3. 确认层：用户可以修正显示名、类型、枚举、范围和敏感性；低置信度、枚举或范围未知的字段在首次使用前必须确认。

除非 RunningHub 后续提供可验证的 object-info/schema 接口，否则不得声称恢复了在线
ComfyUI 控件的完整类型、枚举和 min/max。原始 prompt 只在后端按字节、深度和节点数上限解析；
后端先基于原始规范化 prompt 计算 canonical SHA-256 指纹，再删除敏感默认值，只把脱敏后的
字段模型和指纹返回前端。原始 JSON、其缓存和解析错误片段不得进入前端、数据库、日志、
RunEvent 或错误文本。

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
| 输入 | `video`    | `video`    | 视频输入；由字段映射选择目标                     |
| 输入 | `audio`    | `audio`    | 音频输入；由字段映射选择目标                     |
| 输入 | `text`     | `text`     | 可绑定一个或多个文本字段                         |
| 输入 | `config`   | `config`   | 兼容现有 `RhConfigNode` 的非秘密结构化运行覆盖值 |
| 输出 | `image`    | `image`    | 第一项图片，同时在 metadata 中保留全部列表       |
| 输出 | `video`    | `video`    | 第一项视频，同时在 metadata 中保留全部列表       |
| 输出 | `audio`    | `audio`    | 第一项音频，同时在 metadata 中保留全部列表       |
| 输出 | `text`     | `text`     | 文本输出或工作流返回文本                         |
| 输出 | `archive`  | `archive`  | 第一项已物化归档；支持下载和连接解压节点         |
| 输出 | `metadata` | `metadata` | 任务、输出清单、站点、工作流和追踪信息，不含秘密 |

画布现有单端口 UI 若暂时不能表达同类型多结果，节点主卡片展示全部结果列表，端口输出第一项；`metadata.outputs` 保存有序的全部非秘密描述。不得丢弃第二项及后续结果。

`config` 继续使用现有 `RhConfigNode.nodeInfoList` 结构及
`text | number | image | video | audio` 值类型。每个目标在当前工作流指纹下都必须存在且类型匹配；
敏感目标拒绝从 `config` 赋值。覆盖优先级固定为：运行时媒体/文本端口绑定 > 节点内显式覆盖 >
上游 `RhConfigNode` > 远端默认值。同一优先级出现多个不同值或多个上游配置提供者时失败关闭；
不得沿用现有节点“先遇到者静默获胜”的行为。

### 5.4 提交流程

```text
读取节点配置
  -> 冻结站点和 runninghub-workflow-v1 协议
  -> 校验工作流 ID、指纹、字段映射、冲突和秘密可用性
  -> 检查输入类型和 v1 30 MiB 上限
  -> 向同一站点上传本地媒体并得到 RunningHub fileName
  -> 只生成用户明确覆盖的 nodeInfoList
  -> 写入非秘密提交尝试记录，然后创建 Workflow 任务
  -> 得到 taskId 后立即持久化恢复描述符
  -> 若请求已发送但无法确认 taskId，进入 SUBMISSION_UNKNOWN 且不自动重试
  -> 查询状态；重启后有 taskId 时只从查询继续
  -> 成功后查询 outputs
  -> 按声明和文件签名分类，分别物化媒体与 archive
  -> 批次原子提交结果并结束 RunEvent
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
    | "SUBMISSION_UNKNOWN"
    | "QUEUED"
    | "RUNNING"
    | "MATERIALIZING"
    | "SUCCEEDED"
    | "FAILED"
    | "CANCELLED";
  taskId?: string;
  outputs?: WorkflowArtifact[];
  retryAfterMs?: number;
  error?: { code: string; message: string; retryable: boolean };
};
```

远端状态名称需在 v1 Provider 适配器内映射，未知状态失败关闭并保留可诊断的非秘密摘要。
HTTP 200 不等于任务成功；必须同时检查官方业务码、任务状态和输出数组。请求采用当前兼容所需的
body `apiKey`/Bearer 形式时，两处都由后端注入且都不能记录。

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
  workflowRevisionFingerprint: string;
  createdAt: string;
};
```

恢复描述符不得包含 API Key、访问密码、ZIP 密码、签名下载 URL、输入正文或完整
`nodeInfoList`。站点一经记录不可切换；恢复时只使用该站点当前配置的密钥。如果密钥轮换后
无法访问任务，返回明确的认证/恢复错误，不得跨站寻找。

提交开始前另写一条短小的非秘密尝试记录，包含 `site`、`workflowId`、指纹、`requestId`、
开始时间和阶段。`requestId` 只是本地诊断标识，不是 Provider 幂等键。当 create 请求尚未发送
时可以安全重试；一旦请求字节可能已被远端接收却没有拿到 `taskId`，记录转为
`SUBMISSION_UNKNOWN`，恢复后仍然停止。用户只能通过带“可能重复任务及计费”提示的显式操作
发起新任务。只有未来存在可按本地标识对账的官方接口时，才能自动消解该状态。

## 6. 归档物契约

### 6.1 新端口类型

`archive` 需要进入前端端口定义、Canvas Node Schema、Agent 公共视图和连线校验的允许集合，但不自动加入所有现有媒体能力集合。只有明确声明接受归档的节点能连接。

建议持久化结构：

```ts
type ArchiveArtifact = {
  version: 1;
  kind: "archive";
  format: "zip";
  artifactId: string; // 本地提交 manifest 的稳定身份
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
  retentionClass: "source-archive";
};
```

禁止字段：明文密码、远端 API Key、长期远端签名 URL 和用户工作流访问密码。

### 6.2 下载与物化

归档下载走专用、纯文件流的 `remoteArchiveMaterializer`：

- 只接受 Provider 输出记录中的 HTTPS URL；按官方 CDN host policy、公共 DNS 和 IP 重绑定防护校验，不假设单个固定 CDN 域名永不变化。
- 逐跳验证重定向，禁止本机、内网、link-local、metadata IP 和非 HTTP(S) 协议。
- 以流式方式写入随机临时文件，同时计算 SHA-256 和字节数；超限立即中止并删除临时文件。
- 使用文件头和流式探测校验 ZIP 签名、扩展名与声明类型；HTML、JSON 错误页和空文件必须拒绝，禁止退回整文件 `Buffer` 校验。
- 下载前按内容长度或硬上限预留归档、解压 staging、发布 staging 和安全余量；磁盘空间不足时在写入前失败。
- fsync/关闭后纳入节点输出批次事务，全部产物完成才发布稳定本地 URL。
- Provider 查询重试以 `site + taskId + nodeId + Provider output identity` 幂等；远端 URL 变化不能制造重复本地归档。

不要修改 `validateProxyMediaBuffer` 让它全局接受 ZIP，因为那会降低现有图片、视频和音频的严格类型校验。

### 6.3 批次事务与保留策略

同一节点运行的媒体和归档先写入受控 staging，并生成包含来源、哈希、大小、分类和目标 ID 的
batch manifest。所有成员完成校验和容量检查后，才在单一提交边界发布素材记录；任一成员失败
必须回滚本批次已经发布或暂存的成员，不能暴露半成品。ZIP 解压同样先完成整批分类和验证，再
发布全部展开结果；发布失败保留原始 ZIP，但不保留部分展开素材。

`source-archive` 与 `expanded-output` 使用独立保留类别：原始 ZIP 默认随画布/素材引用保留，
展开结果按正常素材引用保留；无引用的下载 staging、解压 staging、失败批次和崩溃孤儿由启动时
和周期 GC 按年龄清理。GC 只处理 manifest 明确归属的临时目录，不扫描或删除任意用户路径。
1/2/4 GiB 是硬上限候选，不等于每次都可用；实际准入还必须满足实时可用空间、并发预留和
固定安全余量。

## 7. 节点二：ZIP 解压

### 7.1 节点职责与界面

| 字段         | 行为                                                       |
| ------------ | ---------------------------------------------------------- |
| 归档输入     | 首版只接受 `archive` 连线；不提供本地文件选择器            |
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

### 7.3 跨平台解压运行时

7-Zip 不再是 Windows 专用程序。官方当前同时发布 Windows、Linux 和 macOS 控制台版本；
[官方下载页](https://www.7-zip.org/download.html)明确提供覆盖 macOS arm64/x86-64 的包。
2026-08-13 已在本机 Apple Silicon macOS 临时目录验证官方 26.02 `7zz`：下载包 SHA-256 为
`1cf6760579502f87e591ff5c73a005ec50b3e4d6f507e8b038382d563c3175b9`，二进制为
arm64/x86-64 Mach-O universal，实际启动报告 `7-Zip 26.02 (arm64)`。该验证只证明候选运行时
能在当前 macOS 启动，不等于已经把它加入产品或完成安全验收。

首版候选为官方 7-Zip console runtime；不能依赖系统预装、Homebrew、PATH 或当前偶然存在的
间接 npm 依赖。正式实现前必须为目标平台固定同一审查版本的独立资产：

| 平台               | 候选资产形态                         | 最低验证                                       |
| ------------------ | ------------------------------------ | ---------------------------------------------- |
| macOS arm64/x86-64 | 官方 universal `7zz`                 | 两种架构启动、签名/公证策略、加密 ZIP 安全矩阵 |
| Windows x64/arm64  | 官方对应架构 console executable      | unpacked、NSIS 安装版、加密 ZIP 安全矩阵       |
| Linux x64/arm64    | 官方对应架构 `7zz`（如产品发布该端） | 目标发行环境启动、权限和加密 ZIP 安全矩阵      |

官方许可证说明除 `7z.dll` 外的文件使用 GNU LGPL，并要求二进制再分发时附带相关许可证信息；
实现阶段必须由依赖/发布审查确认所选资产和随包许可证文本，不能只记录网址。

实施要求：

- 运行时版本、平台、架构、来源、许可证、文件大小和 SHA-256 写入独立 manifest。
- manifest 按 `darwin-arm64`、`darwin-x64`、`win32-x64`、`win32-arm64` 等目标精确选择；未声明的平台失败关闭。
- 开发态和 Electron packaged 态通过单一 resolver 定位，找不到、不可执行、架构不符或哈希不符即失败关闭；不得静默改用系统 `7z`。
- 打包前、打包后都验证二进制哈希、架构和最小功能；macOS packaged app 与 Windows 安装包都必须做真实密码 ZIP 冒烟和攻击样本测试。
- 使用 `spawn` 参数数组而不是拼接 shell 命令；日志完全删除密码参数。
- 7-Zip 的密码参数可能在进程存活期间被同机高权限进程看到。首版把这项列为本地信任边界，并通过短生命周期、无 shell、无日志、运行后清空内存引用降低暴露；若审查要求抵御同机进程观察，需要改选支持安全密码输入的库/运行时。

### 7.4 安全解压流程

```text
校验本地 archive 引用和 SHA-256
  -> 在受控、权限收紧且父链无链接的 staging 根目录创建随机批次目录
  -> 7zz 以固定模板列目录，不写磁盘
  -> 用结构化状态机解析 entry 属性并验证完整清单
  -> 预留归档、展开、发布和安全余量所需磁盘空间
  -> 7zz 以固定模板解压到空 staging，禁止 shell 与覆盖
  -> 从 staging 根开始逐级 lstat，复核链接、文件签名、大小和清单
  -> 生成 batch manifest 并整批原子登记到项目素材存储
  -> 发布成功后清理 staging；失败则回滚整批并保留原 ZIP
```

候选调用模板必须在实现中以常量和快照测试固定。列表模板至少固定 `l`、技术列表格式、UTF-8
输出、禁止响应文件展开、`--` 参数终止符和归档路径；解压模板至少固定 `x`、空输出目录、
非交互、禁止覆盖、UTF-8、`--` 和归档路径。密码只能作为独立参数传给 `spawn` 且在任何日志、
错误和进程诊断包装中删除。最终参数要以所固定版本的官方手册和跨平台实测为准，不允许运行时
拼接用户选项。

技术列表不能用简单的 `Path =` 文本切割。解析器必须识别 entry 边界、类型、Attributes、
Size、Packed Size 和 link/reparse 信息，并拒绝缺字段、重复字段或未知结构。预检必须拒绝
symlink、hardlink、junction、reparse 和设备条目；解压后的每一级父目录也必须 `lstat` 为真实
目录。攻击矩阵必须包含“链接条目在前、其子路径文件在后”，以证明解压器不会先创建链接再向
其外部目标写入。如果所选 7-Zip 版本/参数无法证明解压阶段不会跟随链接或 reparse point，
运行时选型验收失败，必须改用能逐 entry 受控写出的库或隔离 worker，不能只依赖事后扫描。

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

解压前的空间准入按最坏情况计算，不只比较 ZIP 大小：至少覆盖声明总展开大小、归档/展开
staging、批次发布复制或重命名开销、并发预留和固定安全余量。运行中实际字节数接近预留上限
时立即取消子进程并回滚；不能等磁盘写满后再处理。

### 7.5 文件分类

分类依据文件签名和可信探测结果，不能只看扩展名。探测器必须接受受控文件路径/流和大小上限，
不得为大文件调用现有整文件 `Buffer` validator。首版输出：

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

首版增加进程内 `WorkflowSecretSessionStore`，以 `canvasId + nodeId + fieldId` 为键，只存
当前应用会话的值。节点 `data`、React Flow 复制负载、撤销历史、导出、协作消息、RunEvent、
恢复描述符和 Agent 公共视图只能出现“需要/当前会话已提供”的占位状态，不能出现值或可逆引用。
删除节点、关闭画布、退出应用或显式清除时立即销毁对应值；后台使用后缩短引用生命周期。

工作流 JSON 的秘密隔离在后端完成：先有界解析和计算仅返回摘要的 canonical fingerprint，再以
明确节点类/字段规则、敏感词和保守 fallback 删除值，最后才构造前端字段模型。用户手工取消
“敏感”只能影响展示分类，不能覆盖服务端明确敏感规则。任何无法安全规范化的 prompt 整体拒绝，
不得把原文塞进错误消息帮助调试。

首版不通过 `archive` 边传递密码。真实样例工作流的压缩密码已经配置在远端，用户只需在本地解压节点输入同一密码；如用户覆盖远端压缩节点密码，则在两个节点分别输入。后续可设计只存在于后端内存的短期 opaque secret reference 来减少重复输入，但不属于本方案首版。

## 9. 错误模型与可观测性

稳定错误码建议：

| 范围       | 错误码示例                                                                            |
| ---------- | ------------------------------------------------------------------------------------- |
| 工作流读取 | `RH_WORKFLOW_NOT_FOUND`、`RH_WORKFLOW_ACCESS_REQUIRED`、`RH_WORKFLOW_SCHEMA_CHANGED`  |
| 提交与查询 | `RH_AUTH_FAILED`、`RH_QUOTA_EXCEEDED`、`RH_SUBMISSION_UNKNOWN`、`RH_TASK_FAILED`      |
| 归档物化   | `ARCHIVE_REMOTE_UNSAFE`、`ARCHIVE_TOO_LARGE`、`ARCHIVE_TYPE_MISMATCH`                 |
| 解压       | `ZIP_PASSWORD_REQUIRED`、`ZIP_PASSWORD_INVALID`、`ZIP_UNSUPPORTED_ENCRYPTION`         |
| 安全边界   | `ZIP_PATH_UNSAFE`、`ZIP_ENTRY_FORBIDDEN`、`ZIP_LIMIT_EXCEEDED`、`ZIP_RUNTIME_INVALID` |
| 本地事务   | `ARCHIVE_DISK_INSUFFICIENT`、`ARCHIVE_BATCH_COMMIT_FAILED`、`ARCHIVE_GC_FAILED`       |

RunEvent 记录节点类型、站点、工作流 ID、任务 ID、阶段、耗时、结果数量、字节数、归档/结果 SHA-256 和稳定错误码。所有自由文本先走现有日志脱敏器；不得记录请求体、密码、API Key、远端签名 URL、原始工作流 JSON 或 7-Zip 完整 argv。

## 10. 兼容性与迁移

- 现有节点类型 `runninghub` 和 `runninghub-wallet` 不改名、不迁移、不增加 ZIP 行为。
- 旧 Canvas Node Schema 能继续读取旧画布；新节点由 schema 版本升级明确引入。
- 新增 `archive` 后，所有穷举端口种类的前后端集合与测试必须同步；不能只改前端类型。
- `archive` 默认只能连到 `archive-extract` 或未来明确声明的归档节点。
- 新应用打开未知或未来节点类型时必须失败关闭。包含本次新节点的文档记录最低 schema/app capability；已经发布的旧应用无法补发“不支持”UI，旧版打开新文档属于不支持场景，不作兼容承诺。
- Provider 输出含媒体和 ZIP 时分别物化；单个输出失败时整个批次失败，只保留非秘密诊断摘要，不提交半成品结果。

## 11. 预期代码影响面

以下是实施阶段的预期范围，不是本轮修改清单：

| 层            | 预计变更                                                                                             |
| ------------- | ---------------------------------------------------------------------------------------------------- |
| 节点与画布    | 新增两个节点组件、节点注册、默认数据、端口定义、连线规则、完成通知和上下文菜单                       |
| Workflow 解析 | 后端有界解析、指纹和脱敏；前端扩展字段推断、置信度、用户确认与敏感字段标记                           |
| 前端服务      | 增加 Workflow JSON/submit/query/cancel 和 archive extract 客户端类型                                 |
| 后端路由      | 增加 Workflow 专用代理路由和本地 ZIP 解压路由                                                        |
| 后端服务      | 增加精确站点/v1 适配、流式归档物化、容量预留、批次事务、GC、跨平台 runtime resolver 与解压服务       |
| 秘密会话      | 增加节点数据之外的会话秘密存储，并覆盖复制、导出、协作、恢复和 Agent 视图泄漏边界                    |
| 恢复与事件    | 增加 `runninghub-workflow` 恢复描述符、状态映射和 RunEvent 数据                                      |
| Schema/Agent  | Canvas Node Schema、端口种类、Agent 工具/公共视图和生成产物同步                                      |
| Electron      | 固定版本的 7-Zip runtime、manifest、打包复制及 post-build 校验                                       |
| 测试          | 节点、端口、Provider、恢复、SSRF、物化、ZIP 攻击样本、macOS/Windows Electron 打包与冒烟测试          |
| 文档          | `features.json` 生成源、README/节点帮助、`PROJECT_STRUCTURE.md`、项目 Skill 和发布说明按最终能力同步 |

实施时先修改权威源，再运行项目既有生成器；不得手工修补生成能力产物来绕过 `feature-sync`。

## 12. 分阶段实施方案

### WP1：契约与迁移边界

- 固化节点 ID、`archive` 结构、端口和 schema 版本。
- 固化 v1 Provider 协议、精确站点、`SUBMISSION_UNKNOWN` 和字段来源契约。
- 固化会话秘密存储、流式文件接口、批次 manifest、容量预留和保留/GC 契约。
- 补齐所有端口穷举集合和连接测试。
- 加入秘密字段序列化拒绝测试。

完成门：旧画布测试不变，新节点可被安全保存但尚不调用 Provider。

### WP2：RunningHub Workflow 节点

- 实现 Workflow JSON 获取、字段映射和缓存指纹。
- 实现后端有界解析/脱敏、字段确认、v1 上传、提交、查询、取消、恢复和错误映射。
- 实现媒体结果与 ZIP 结果分流；ZIP 使用流式物化。

完成门：mock/录制契约测试全部通过；取得 `taskId` 后恢复不再 create，提交不明停止；现有 AI App 测试无回归。

### WP3：ZIP 解压节点

- 为 macOS/Windows 目标分别固化并校验 7-Zip runtime、架构、许可证和打包路径。
- 实现结构化列表预检、链接安全证明、受控解压、事后核验、批次登记和清理。
- 实现密码交互、分类输出、manifest 和重试。

完成门：安全样本矩阵通过，错误和取消不留下临时文件，原 ZIP 可重复解压。

### WP4：产品集成与能力同步

- 完成节点视觉、帮助文本、Agent 可见性、RunEvent 和恢复 UI。
- 更新权威能力资料并运行生成器。
- 完成全量 TypeScript/CJS、lint、schema、writer/lifecycle 和打包门。

### WP5：真实环境验证

- 在用户明确授权和预算范围内，用海外站执行一次低成本真实工作流。
- 验证工作流 ID、字段覆盖、ZIP 返回、密码解压和本地下载。
- 在当前 macOS Electron packaged app 验证 universal runtime、中文路径、攻击样本和重启恢复。
- 在真实 Windows Electron 安装版验证 7-Zip 运行时、中文路径和重启恢复。

真实 Provider 验证和正式安装包构建都需单独授权；localhost/mock 不能替代这些结论。

## 13. 测试矩阵

### 13.1 RunningHub Workflow

- `.ai` 与 `.cn` 精确路由、密钥隔离、默认 `.ai`。
- 同一运行的上传/create/outputs/cancel 站点粘连，禁止现有候选站点 fallback。
- v1 协议固定、`0/804/813/805` 映射以及拒绝混用 V2。
- 工作流不存在、需要访问密码、密码错误、字段 JSON 不完整和字段版本漂移。
- prompt 中含密码时后端脱敏；前端、缓存、错误、数据库和画布均无原始 JSON/秘密。
- 权威字段、启发式推断、低置信度人工确认、未知范围和手工映射。
- 图片/视频/音频上传、30 MiB 边界、中文文件名、多输入字段、仅发送显式覆盖值。
- config 目标/类型校验、覆盖优先级、重复冲突和敏感字段拒绝。
- 提交成功、排队、运行、超时、失败、取消、额度不足和未知状态。
- create 发送前断线可重试；发送后响应丢失进入 `SUBMISSION_UNKNOWN` 且重启不自动重试。
- 已取得 taskId 后网络断开、应用重启和查询恢复不重复提交。
- 单媒体、多媒体、纯 ZIP、媒体加 ZIP、空输出和过期 URL。
- 远端 URL SSRF、重定向、HTML 伪装、类型漂移、超限和碰撞幂等。
- 归档全程不进入大 Buffer，磁盘准入、并发预留、批次提交失败回滚和孤儿 GC。
- 日志、错误、数据库、RunEvent 和画布 JSON 的秘密扫描。
- 复制、撤销、导出、协作消息、恢复描述符和 Agent 公共视图的秘密扫描。

### 13.2 ZIP 解压

- 无密码 ZIP、ZipCrypto、AES、空密码、缺密码、错误密码和不支持加密算法。
- 中文、空格、Emoji、长文件名、嵌套目录和大小写差异。
- `../`、绝对路径、盘符、UNC、ADS、设备名、NUL 和规范化碰撞。
- symlink、hardlink、junction、reparse、FIFO、device、可执行文件和嵌套 ZIP。
- “链接条目后跟其子路径”以及 staging 父目录链被替换为链接的竞态攻击。
- 空包、截断包、CRC 错误、伪 ZIP、条目数/大小/深度/比例边界和炸弹包。
- 取消、7-Zip 崩溃、磁盘已满、并发解压、重复执行和应用重启。
- 事后出现未列出文件、类型与扩展名不符、单项登记失败和原子回滚。
- 批次中最后一项登记失败时零展开结果可见，原始 ZIP 仍可再次解压。
- 图片、视频、音频、UTF-8 文本分类和完整 manifest 顺序。

### 13.3 产品与打包

- 两个节点的新增、复制、删除、保存、恢复、连线和不兼容提示。
- 现有 `runninghub` AI App、媒体节点、3D ZIP 上传和 `.t8flow` 导入无回归。
- 开发态、macOS arm64/x64 packaged app、Windows unpacked/安装版的 runtime 路径、架构、执行权限和哈希验证。
- 打包缺二进制、版本不匹配或 manifest 漂移时必须失败关闭。

## 14. 复审时必须确认的决策

首轮审查已经收敛为以下推荐基线；复审不是重新打开已发现的安全缺口，而是确认产品取舍：

1. 新增共享 `archive` 端口，首版 ZIP 节点只接受该端口；本地 ZIP 选择和 RunningHub archive 输入延期。
2. 首版支持 RunningHub 图片、视频、音频、文本和 config 输入；远端 v1 上传按 30 MiB 提前拒绝。
3. 固定 `runninghub-workflow-v1`、默认海外站并保持整次运行精确站点粘连，禁止跨站 fallback。
4. 接受 create 无幂等键的事实：只保证取得 `taskId` 后不重复 create；`SUBMISSION_UNKNOWN` 必须人工决定是否新建并提示重复计费风险。
5. 字段采用“API prompt 权威标识 + 本地启发式 + 用户确认/手工兜底”，不承诺不存在的完整控件范围。
6. 密码只在 `WorkflowSecretSessionStore` 当前会话内存在；首版不通过边或 opaque reference 自动传递。
7. 首版只输出图片、视频、音频和受限文本，拒绝通用文件、可执行文件和嵌套归档。
8. 1/2/4 GiB、512 条、16 层和 200:1 作为服务端候选硬上限；是否满足实时磁盘准入仍按每次运行计算。
9. 候选运行时采用官方跨平台 `7zz` 并按平台随包；只有结构化列表、链接攻击矩阵、macOS packaged 和 Windows 安装版全部通过才正式选定。
10. 接受密码作为子进程参数的本机进程观察边界，或要求改用安全密码通道的原生库/隔离 worker；这是剩余的明确安全取舍。

第 9 项不是“macOS 以后再看”：官方 universal binary 已在当前 Apple Silicon macOS 启动验证，
但产品采用仍以攻击样本和 packaged app 测试为门。若 7-Zip 不能满足链接写入安全证明，必须更换
实现，不得降低预检/事后检查标准。

## 15. 实施验收标准

审查通过并完成后续开发时，能力必须同时满足：

- 真实海外站 Workflow API 可运行指定工作流并取得 ZIP，不依赖 Web App 发布。
- ZIP 以 `archive` 一等产物保存、下载和连线，远端签名 URL 不成为长期数据。
- 加密 ZIP 在 macOS packaged app 与真实 Windows 安装版均可解密；密码错误和安全拒绝有稳定、可理解的错误。
- 路径、链接、压缩炸弹、SSRF、磁盘上限、秘密泄漏和部分提交测试失败关闭。
- 原始工作流 JSON 和会话秘密不进入前端或任何持久化/可观测载荷。
- 一次任务全程使用同一站点和 v1 协议；取得 `taskId` 后恢复不重复提交，提交不明不自动重试。
- 下载、探测、解压和素材发布不依赖大 Buffer；容量预留、批次回滚、保留和孤儿 GC 可验证。
- 现有 RunningHub AI App、旧画布、其他媒体节点和 `.t8flow` 功能无回归。
- 权威 schema、能力资料、文档、测试和 Electron 打包校验同步完成。

本修订完成文档校验后 Task 恢复为 `ready`，等待第 14 节复审。复审明确通过前，不得进入产品实现、
引入 runtime、发起真实 Provider 提交或构建正式安装包。
