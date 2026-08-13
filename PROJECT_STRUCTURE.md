# 项目结构

```text
T8-penguin-canvas/
├── .agents/                         # 上游画布 Agent skill 与参考资料
├── .nm-workflow/
│   ├── 0a-docs/                     # 长期有效的需求、设计、决策和报告
│   ├── 0b-tasks/
│   │   ├── active/                  # active、blocked 或 ready Task
│   │   └── archive/                 # done 或 cancelled Task
│   ├── 0c-work-packages/
│   │   ├── active/                  # 未归档 Work Package
│   │   └── archive/                 # 已归档 Work Package
│   ├── templates/                   # 可复用工作流模板
│   └── RULES.md                     # notmaster fork 的 Rules-mini 工作流规则
├── backend/                         # 后端服务、路由与后端测试
├── deploy/                          # 部署配置
├── docs/                            # 项目文档
├── electron/                        # Electron 主进程、预加载与打包逻辑
├── extension/                       # 浏览器扩展
├── public/                          # Web 静态资源
├── release-notes/                   # 版本发布说明
├── resources/                       # Electron/产品资源
├── scripts/                         # 构建、审计、验证与发布脚本
├── shared/                          # 前后端共享数据与契约
├── src/                             # React/Vite 前端源码
├── tests/                           # Node、前端、后端及集成测试
├── tools/                           # CLI、桥接器与本地运行时工具
├── .gitignore                       # Git 忽略规则
├── .markdownlint.json               # Markdown lint 规则配置
├── .markdownlintignore              # Markdown lint 排除边界
├── .prettierignore                  # Markdown 格式化排除边界
├── AGENTS.md                        # 上游项目规则与 fork 路由入口
├── features.json                    # 产品能力与发布事实的权威记录
├── package.json                     # npm 依赖、命令与构建配置
├── package-lock.json                # npm 依赖锁文件
├── PROJECT_STRUCTURE.md             # 本文件
├── README.notmaster.md              # notmaster fork 的个人运行与开发指南
└── README.md                        # 项目定位、安装、使用和发布入口
```

这里只记录主要职责边界；功能级源码和测试位置以 `features.json`、相关实现及测试为准。
