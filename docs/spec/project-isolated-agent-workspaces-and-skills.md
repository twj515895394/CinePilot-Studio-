# CinePilot Studio — 项目隔离 Agent 工作区与 Skill 挂载方案

> 版本：0.1（方案 B 核心约束已确认；实现蓝图为推荐设计）  
> 日期：2026-10-30  
> 状态：**已确认：V1 CLI 优先、可优先采用官方 Agent SDK 控制；创建项目即建立独立 Agent 工作区；选定 Skill 从系统目录链接/投射到项目；Agent 不可访问其他项目。** 具体跨平台安全实现待开发验证。  
> 依据：[ADR-0016](../adr/0016-cli-first-with-official-agent-sdk.md)、[ADR-0017](../adr/0017-project-isolated-agent-workspaces-and-skill-projection.md)、[Agent/Skill 总体设计](./agent-skill-orchestration-and-runtime.md)、[数据权威 ADR-0012](../adr/0012-field-owned-markdown-json-sqlite.md)

## 1. 用户明确要求

1. 创建项目时便有属于该项目的 Agent 工作目录，进入项目执行 Agent 时以此目录作为工作区。
2. Skill 由系统指定的集中目录维护，然后链接到当前项目，让 Codex、Claude Code 等 Agent 自然发现和使用已启用 Skill。
3. **严禁 Agent 访问系统内其他项目的目录**。包括绝对路径、相对路径向上跳转、符号链接跳转、工具/子进程和后台读取接口。
4. V1 首要走 CLI 体系，但如果有官方 Agent SDK 可以更稳定地启动会话、流式接收事件、进行工具调用，就可优先使用 SDK；不是先并行建设所有模型 API 接入。
5. Vue Web / Electron 共享工作台，Python 提供统一业务服务；Markdown、JSON、SQLite 按字段唯一权威，Agent 不可绕过业务服务篡改定稿产物。

## 2. 两个不同层级：项目物理目录与 Agent 隔离执行视图

**重要结论：设置进程 cwd 并不构成安全隔离。** Agent 的 Read/Bash/Python/搜索工具以及 CLI 自身进程仍可能访问同一用户有权限读取的其他目录。把 cwd 设置为项目 A 并不禁止读取 \`../project-B\` 或绝对路径。需要执行环境有明确的只读/可写文件系统授权和项目范围边界。

### 2.1 系统侧建议目录布局（路径示意，未定稿）

~~~text
cinepilot-data/
├── app/
│   ├── cinepilot.sqlite         # 项目/任务/版本/关系，非 Agent 直接可见
│   └── private/                 # 令牌/认证/配置，非 Agent 直接可见
├── skill-registry/              # 系统统一管理的 Skills
│   ├── storyboard-designer/
│   ├── screenwriter/
│   ├── story-architect/
│   └── ...
├── projects/
│   ├── P001/                    # 项目 A 正式工作区及内容
│   │   ├── project-docs/
│   │   ├── creative/
│   │   ├── episodes/
│   │   ├── assets/
│   │   ├── .agent-work/         # 草稿执行/临时文件
│   │   └── .agent-skills/       # 当前项目已启用的 Skill 入口（逻辑）
│   └── P002/                    # 项目 B，不能被 P001 Agent 访问
└── execution/                   # 隔离环境内部实现（非项目 Agent 共享根）
~~~

该目录只是**业务服务在宿主机上的真实存储**，不是给 Agent 整个 \`cinepilot-data/\` 读取。Agent 的运行环境应该只看到当前 P001 所需的投影视图。

### 2.2 Agent 实际可见的隔离环境（推荐）

~~~text
Agent Run for P001
/
├── workspace/                   # Agent cwd = 当前项目工作区
│   ├── project-docs/             # 当前项目的允许读取部分
│   ├── creative/                 # 当前项目的允许读取部分
│   ├── episodes/
│   ├── assets/                   # 允许读取的项目图片
│   ├── .agent-work/              # Agent 可写的非正式草稿输出
│   └── <runtime-specific-skill-link>/
│       └── storyboard-designer -> /skills/storyboard-designer
└── skills/                      # 仅本项目已启用 Skill 的只读投射

不挂载：宿主机 projects/、P002、global SQLite、系统凭据、其余 Skills
~~~

**实现要求**：
- 启动任务时 Python 根据已授权的 \`project_id\` 取得工作区，不接受 Agent 随意传入任意磁盘路径。
- 在 Agent 进程/子进程的文件系统视角里，其他项目目录根本不可见；默认拒绝跨沙箱文件读写。
- 项目资料可读不等于可随意修改：正式已定稿文件必须只读；可写仅限任务草稿、临时工作和统一业务服务授权的提交区域。
- **宿主后端**拥有跨项目管理权限，但此权限不传给 Agent 进程；后端受控工具访问 SQLite/媒体时严格校验 \`project_id\` 与允许的对象 ID，不信任 Agent 自报路径。
- 不让 Agent 的任意 Shell 命令调用一个能绕开项目范围的无约束后端 API。
- 认证信息只按运行必需量提供，不把全局用户 Home、AppData、SSH 文件、API Key 库或整个系统 Skill 源目录直接挂入项目。

**跨平台限制**：Windows、macOS、Linux 的隔离能力不同。实现可以采用特定平台沙箱、受限制进程、受限容器/虚拟化或其他经测试可满足要求的方式；不能假设 Docker / WSL 默认可用，也不能把所有平台一概归结为 \`chdir\`。如果某个执行适配器在目标平台无法满足强隔离，应该阻止其执行并提示环境不支持，而不是悄悄降级为仅提示词警告。

## 3. Skill 集中管理、项目启用和运行时投射

### 3.1 三层职责

1. **系统 Skill Registry**：安装、版本、描述、适用阶段、所需工具、可信来源等元数据；供管理后台选择。
2. **项目 Skill 配置**：某个项目启用了哪些 Skill、使用哪个版本、是否为默认；允许不同项目选不同 Skill 组合。
3. **Agent Runtime Skill Projection**：当前任务启动时，仅把该项目启用、当前任务需要的 Skill 内容以**只读**方式投射到隔离环境的 Agent 可识别目录。

**用户所说的“从系统指定目录链接到项目目录”在产品层完全成立。** 但物理层不能让项目 Agent 通过链接逃逸回可以浏览全部 Skill 或项目的宿主目录。

### 3.2 链接 vs 快照

| 实现形式 | 适用情况 | 风险/建议 |
|---|---|---|
| 直接宿主机绝对符号链接 | 完全受信任、无安全隔离时方便 | **不能单独满足跨项目禁止访问要求**；链接目标位于外部目录、权限可能过大，不推荐作为最终安全方案 |
| **沙箱内只读挂载 + 项目符号链接（推荐，若平台支持）** | 隔离运行环境已有 \`/skills/<selected>\` | 链接目标位于隔离环境内、只读，仅包含当前项目启用 Skill |
| **启动时复制只读 Skill 快照（兼容兜底）** | Windows/Electron 环境不便建立符号链接或目录挂载 | 不依赖系统级链接能力；有少量文件复制成本，但可用缓存和版本钉选控制 |
| 全量复制系统所有 Skill | 简单但占用空间大 | 违背按项目选择和最小权限，不推荐 |

**注意**：各 Agent 对本地 Skill 的自动发现目录、名字和配置加载规则可能不同；\`.agents/skills\`、\`.claude/skills\` 等仅能作为需要验证的运行时映射示例，不应强行宣称所有 Agent 会自动扫描同一目录。使用 \`SkillProjectionAdapter\` 在隔离工作区建立符合当前 Agent 的目录布局。

### 3.3 Skill 目录不是无限信任的代码

Skill 可以包含说明文件、辅助脚本和可能被 Agent 调用的命令。启动时需要审查安装来源/版本，所有 Skill 脚本仍在当前项目沙箱权限内执行；禁止 Skill 安装脚本取得宿主机访问其他项目的文件权限，也禁止注册“绕开项目路径检查”的后台工具。

项目内 Skill 升级应受控登记版本；既有正式作品只记录必要的创作版本来源，不要求所有既有任务在升级后自动重跑。

## 4. Agent Runtime SDK/CLI 与隔离的协作

### 4.1 V1 优先顺序（已确认原则，具体产品待选）

- **优先 CLI 生态的 Agent 执行能力**：首期先跑通一个 SDK 或 CLI 实现，保证同一套 Python 后端可以创建 Run、接收事件、控制取消、接入查询与草稿提交。
- **官方 Agent SDK 可用时优先评估**：例如 Codex Python SDK 或 Claude Agent SDK，本质是让程序控制本地 Agent，不应与普通直接调用模型 API 的 SDK 混为一谈。
- **没有可满足能力或隔离要求的 SDK 时**：可以使用受控 CLI 适配器；但 Agent Runner 的项目隔离要求不因 SDK/CLI 而变化。
- **第二 Agent/模型接入**：保持 Adapter 边界，不为“理论兼容所有模型”提前建设过度通用框架。

### 4.2 为什么不能 SDK 直接覆盖所有隔离问题

SDK 提供的 \`cwd\`、权限参数或自带 sandbox 配置可能帮助约束工作区，但它们是否限制**文件读取**、跨目录符号链接、外部命令、MCP/网络工具、子进程及服务调用，都需按具体运行时实测。若无法保证，不可用 SDK 的高级抽象替代操作系统级隔离。

Web 端的 Python 服务只能执行其所在主机可用的 Agent 运行时；浏览器网页不能天然读取用户本机 CLI。Electron 版可通过本机的 Python 执行服务和本机沙箱使用当前项目 Agent。远端 Web 如要运行，则必须提供独立 Agent Host/沙箱与身份隔离。

### 4.3 Runtime 生命周期（推荐）

~~~text
创建 Project
  → Python 创建项目工作目录与项目配置
  → 选择系统 Skill Registry 中已启用的 Skill
  → 保存项目 Skill 启用清单

用户打开 EP03 / B3 工作台并提交任务
  → Orchestrator 判断阶段准入与允许的产物版本
  → Python 创建 Run（project_id, episode_id, stage_id, task_scope）
  → 准备当前项目沙箱/投射工作区 + 选定 Skill 只读映射
  → Agent Adapter 进入该项目 cwd、启动 CLI 或官方 Agent SDK
  → Handoff Builder 提供最小任务目标和来源版本
  → Agent 使用限制在该项目的文件/业务查询工具
  → Agent 写可控草稿目录或 submit_draft 工具
  → Artifact Service 校验、写入并索引草稿
  → Vue/Electron 展示草稿和任务进度
  → 用户确认定稿 → 系统发布只读正式产物
~~~

## 5. 受控查询和提交：不能只保护文件系统

Agent 可能通过 Python 服务提供的 \`get_asset\`、\`get_scene\`、\`get_media\` 查询其他项目数据。因此每个 Tool Run 必须在服务端绑定不可变的 \`project_id\` 和任务授权范围；Agent 的参数里即使填写其他项目的 ID，也必须拒绝。

推荐规则：
- \`get_artifact_excerpt\` 仅返回当前项目及其确定的上游版本，默认不读取旧稿/其他项目。
- \`get_asset\` 返回当前项目基础资产与该集选用的变体，并检查资产归属。
- \`get_media_ref\` 通过服务解析媒体 ID，权限验证通过后给当前运行时可访问的视图/媒体，不直接返回可读取其他项目路径的宿主绝对路径。
- \`submit_draft\` 强制 \`project_id\`、\`episode_id\`、\`stage_id\` 和 \`run_id\` 关系一致；Agent 不得绕过 Artifact Service 直接修改 SQLite 状态或历史定稿文件。
- 子进程/工具调用、后台任务、MCP 等如获得额外文件或网络权限，必须采用同样的项目隔离要求；不能因为是 SDK 内建工具就默认信任。
- 其他项目不存在“项目外的默认全局搜索”后门；跨项目复制创意仍是用户显式复制内容，不构建跨项目知识库。

## 6. Python 服务与 Vue/Electron UI 的职责

- **Vue**：在当前项目页面显示项目启用的 Skill、Agent 运行模式、执行进度、当前阶段产物，用户可以提交创作/审查任务。
- **Electron**：复用 Vue 界面，负责桌面容器和本机 Agent/Python 启动的外围管理；不在渲染进程里直接暴露系统文件访问权限。
- **Python**：唯一管理项目、资产/版本、Agent Runner/Sandbox、Skill Projection、任务/工具权限、产物提交、审查、发布与数据库。
- **Agent Run**：在本项目受限执行环境中，读取必要的项目输入、专业 Skill 和当前任务，输出草稿；不能读取其他项目或改写已定稿内容。

## 7. 最小验收场景（必须具备）

| 场景 | 预期结果 |
|---|---|
| 新建项目 P001 | 创建独立项目工作区与 Skill 启用配置；启动任务 cwd 定位到 P001 |
| P001 启用 Storyboard Skill | Agent 能在自己的工作区自然发现、读取该 Skill，不能读取未启用 Skill |
| P001 Agent 执行读取 \`../P002\` 或绝对路径访问 P002 | **文件系统级拒绝**，不能靠提示词拒绝模拟成功 |
| P001 Agent 通过 \`get_asset(project=P002)\` 请求其他项目资产 | 业务服务拒绝，无数据泄露 |
| 通过符号链接、子进程或工具间接尝试访问 P002 | 依旧拒绝 |
| P001 Agent 试图改写已定稿文件或全局 SQLite | 文件系统禁止或业务服务拒绝 |
| P001 Agent 查询本项目角色面部/全身/服装参考媒体 | 获得与授权项目及本集变体匹配的媒体；视觉输入按运行时能力判断 |
| 两项目同时创建任务 | 各自 cwd、数据读写、Skill 配置和会话上下文不会串场 |
| 禁用项目 Skill / 更换 Skill 版本 | 下一任务按更新的项目 Skill 清单投射，不自动改写旧定稿 |
| 当前操作系统无法提供要求的隔离 | Agent Runner 明确报不可用，不静默降低隔离等级 |

## 8. 后续顺序

1. **功能与权限基线已确认**：项目独立工作区、选定 Skill 从系统目录只读投射、其他项目严格不可访问、CLI 优先且评估官方 Agent SDK。
2. **下一步先设计 Vue 工作台与 Agent 交互闭环**：项目创建、启用 Skill、每分集阶段对话、实时任务、正文/镜头产物预览与定稿。
3. **再定具体运行时和跨平台隔离实现**：首个 Codex/Claude Adapter、Windows/macOS/Linux 沙箱能力验证、Electron 与 Python 生命周期。
4. **数据/接口设计**：Project/SkillBinding/AgentRun/Artifact/Asset/Media 的稳定身份和相关 JSON/SQLite Schema。
5. 常规异常、日志、恢复和文件校验在开发期完成，不分割成过多用户选择题；但项目隔离是**不能自动降级的安全验收硬要求**。

### 官方参考（供实现阶段核对版本）

- Codex Python SDK：https://github.com/openai/codex/tree/main/sdk/python
- Codex TypeScript SDK：https://github.com/openai/codex/tree/main/sdk/typescript
- Claude Agent SDK：https://platform.claude.com/docs/en/agent-sdk/overview
- 受控 Agent 执行环境与最小权限：https://developers.openai.com/api/docs/guides/agents-api/environments/self-hosted
