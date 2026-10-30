# CinePilot Studio — Agent / Skill 协作与运行架构

> 版本：0.3（补齐 Agent/Model Profile、Stage Skill Profile、Skill Router 与 Run 快照合同）
> 日期：2026-10-30
> 状态：**V1 CLI 生态优先、可优先通过官方 Agent SDK 控制，以及项目工作区强隔离均已确认**；首个具体 Agent/SDK 和执行沙箱实现未确认
> 依据：[Agent 接入与模型管理](./agent-integration-and-model-management.md)、[Skill Registry 与阶段编排](./stage-skill-orchestration-and-registry.md)、[整体设计总览](./overall-product-workflow-and-system-map.md)、[八阶段合同](./stage-contracts-and-artifact-map.md)、[资产模型](./creative-asset-model-and-use.md)、[ADR-0007/0008/0009](../adr/)、[ADR-0010](../adr/0010-compact-versioned-agent-handoff.md)、[ADR-0011](../adr/0011-mandatory-handoff-context-controlled-retrieval.md)、[ADR-0012](../adr/0012-field-owned-markdown-json-sqlite.md)、[ADR-0015](../adr/0015-project-assets-with-episode-variants.md)

---

## 1. 目标与非目标

让用户在 Vue Web / Electron 工作台中通过一个自然的聊天入口调用创意、剧本、导演、拆解、分镜、提示词专业能力。Agent 创作内容，**Python 系统编排器负责阶段准入、版本钉选、资料按需读取、结果校验、草稿/定稿和依赖关系**。不再把长聊天历史当作项目数据的事实来源。

V1 目标：
- A1–A4、B1–B4 的 Agent + Skill 创作、审阅、继续修改、定稿 Handoff。
- 支持单分集、局部场次、片段和镜头任务，也能对多个分集提交批量任务。
- 读取用户已有项目、角色、场景、参考图/多视图与版本；草稿写入后管理后台立即可预览。
- Web/Electron 复用主要 Vue 页面、同一 Python 服务逻辑。
- 预留 Agent CLI/SDK/API 适配器，不在主业务代码写死供应商命令格式。

**V1 非目标**：执行实际图像/视频生成 API、通用多 Agent 自治平台、独立知识库部署、自动替用户批准产物、复杂跨项目共享。

**新增明确要求（已确认）**：每创建一个 Project 就为该项目建立独立 Agent 工作区，Agent 的执行 cwd 定位该项目；所需专业 Skill 来自系统集中注册目录，只允许将当前项目启用的 Skill 只读链接/投射到隔离工作区；**严格禁止 Agent 访问其他项目文件目录**。设置 cwd 不是隔离机制，系统必须提供真正的文件系统与工具查询权限边界。实现详见 [项目隔离 Agent 工作区与 Skill 挂载](./project-isolated-agent-workspaces-and-skills.md)、[ADR-0016](../adr/0016-cli-first-with-official-agent-sdk.md) 与 [ADR-0017](../adr/0017-project-isolated-agent-workspaces-and-skill-projection.md)。


---

## 2. 已确认的职责与原则

1. 技术基线：Vue + Python + Electron；Markdown 长篇正文、JSON 结构化产物、SQLite 关系/版本/审批，字段唯一权威。
2. 正式下游阶段必须引用上游**已定稿的确切版本**。项目创意池 A1/A2 在分集建立前，不受同一分集阶段门禁约束。
3. 定稿不可覆盖。上游新定稿导致依赖下游待复核，但不得自动重做。
4. 系统生成精简 Handoff：目标、作用域、确切定稿版本、当前有效关键约束、目标产物合同；其余从受控查询按需读取。
5. 项目基础角色/场景等资产可被同项目多分集复用；分集可选择自己的造型、服装、状态变体，不直接改写基础资产。
6. B2 是独立制作拆解关卡；资产库贯穿 A3–B4。
7. Web UI 和 Agent 走相同的业务服务读写、审批与版本发布规则。
8. 本文其余执行架构是**推荐设计**，不是用户已确认的实现选型。

---

## 3. 逻辑架构

~~~text
Vue Web UI                       Electron (复用 Vue)
  ├─ 项目创意池                       ├─ 分集工作台
  ├─ 分集阶段导航                     ├─ 资产选择器
  ├─ Agent 对话 + 任务进度              └─ 产物审查 / 编辑 / 定稿
  └─ 产物预览 / 版本审查
                  │ API + 流式事件（协议待定）
                  ▼
              Python Business Server
  ┌──────────────────────────────────────────────────┐
  │ Request Context / Auth & Scope                    │
  │ Stage Orchestrator（阶段准入、任务、状态、审批）       │
  │ Intent Router + Skill Registry（能力路由）           │
  │ Handoff Builder（小上下文、版本固定）                 │
  │ Agent Runtime Adapter（CLI/SDK/API 可替换）          │
  │ Controlled Read Tools（项目/分集/产物/媒体读取）      │
  │ Artifact Submission & Validation（草稿/审查/版本）    │
  │ Event/Task Stream（UI 进度与结果通知）                │
  └──────────────────────────────────────────────────┘
       │                      │                    │
       ▼                      ▼                    ▼
  SQLite 身份/状态/关系      Markdown/JSON        媒体文件/缩略图
                         版本化内容正文/结构      角色多视图/分镜参考
~~~

**特别注意**：Electron 不是直接调用 Python 中的某个对象，而是通过稳定业务服务接口交互；桌面端 Python 是随 Electron 本机启动，还是连接远端 Python 服务，需要后续统一定稿。Web 部署也无法凭空控制**浏览器用户电脑**上的 Codex/Claude CLI；CLI 若选为 Agent 运行时，必须安装并运行在 Python 执行所在的机器或有明确的远程 Agent Host。

### 推荐的模块划分（不是进程划分）

| 模块 | 主要职责 | 不应承担 |
|---|---|---|
| Stage Orchestrator | 进入阶段、任务作用域、确定输入、定稿门禁、任务状态 | 剧情创作质量判断 |
| Intent Router | 依据工作台/用户意图选择专业技能和任务类型 | 绕过门禁、自动更换正式来源版本 |
| Skill Registry | Skill 元数据、能力、使用阶段、产物合同、版本 | 将专业 Skill 与特定模型永久绑定 |
| Handoff Builder | 精简任务摘要、强约束、来源版本钉选、查询许可范围 | 塞入全部聊天与所有大文件 |
| Agent Runtime Adapter | 启动、交互、事件流、工具调用桥接、取消/恢复能力映射 | 保存数据库权威状态 |
| Controlled Tools | 查询场次、片段、角色/场景/图片、草稿提交 | 开放不受限的 SQL 或任意路径读写 |
| Artifact Service | Markdown/JSON 解析、结构校验、草稿保存、版本与定稿 | 用聊天回复替代正式产物 |
| Media Service | 图片登记、标签/视图、按媒体 ID 获取预览与授权资源 | 让 Agent 自由拼接本地路径 |
| UI API/Event | 聊天事件、产物变化、审查确认和进度 | 直接改变阶段审批规则 |

---

## 4. Agent、Model 与 Skill 的职责分离和配置解析

本节只定义运行时协同边界，具体配置对象、页面和字段以两份专项规格为准：

- [Agent 接入与模型管理](./agent-integration-and-model-management.md)：Runtime Registry、Adapter、Connection Profile、动态模型发现、Agent Model Profile、配置继承和执行快照。
- [Skill Registry 与阶段 Skill 编排](./stage-skill-orchestration-and-registry.md)：外部来源治理、系统 Skill Registry、A1–B4 Stage Skill Profile、项目/分集/任务覆盖、Skill Router 和 Skill Plan。

### 4.1 三个独立维度

| 维度 | 解决的问题 | 由谁负责 |
|---|---|---|
| Agent Runtime | 谁来执行任务、如何启动/接收事件/调用工具 | Agent Registry + Agent Adapter |
| Model | 该 Runtime 本次使用哪个模型和支持的参数 | Model Discovery + Agent Model Profile |
| Skill | 当前阶段采用什么创作方法、辅助模块和质量检查 | Skill Registry + Stage Skill Profile + Skill Router |

Skill 不绑定特定 Agent/Model；Agent Model Profile 不包含 Skill。Stage Orchestrator 在启动 Run 时分别解析 Agent/Model Profile 与 Stage Skill Profile，再组合成 Run Plan。

### 4.2 Stage Skill Profile 与候选映射

每个 A1–B4 阶段拥有独立 Stage Skill Profile，可配置主 Skill、辅助 Skill、条件/题材 Skill、审查 Skill、版本、调用策略和执行顺序。一个 Skill 可跨阶段复用，一个阶段可组合多个 Skill。

用户提供的第一批八个候选及其阶段映射、License、上游 Commit 与适配风险，统一记录在 [Skill Registry 与阶段编排规格 §4–5](./stage-skill-orchestration-and-registry.md)。该映射仍是候选评估方案；B1/B2/B4 的缺失 Skill 由用户后续补充。能力角色名称不等于已导入 Skill。

### 4.3 Skill Router 与执行计划

Skill Router 是 Python 编排层的规则化能力，不是仅靠一个大 Prompt 或让 Agent 自己决定。它读取当前项目/阶段/任务范围、有效 Stage Skill Profile、题材条件、已启用 Skill 版本和 Agent Runtime 兼容性，输出可解释 Skill Plan。

每个 Run 默认一个 Primary Skill 负责主要草稿；Supporting Skill 提供限定范围的方法；Conditional Skill 按条件加载；Review Skill 默认诊断而不擅自改写。阶段合同、用户约束、安全规则优先于外部 Skill 指令。

### 4.4 Agent/Model Profile 解析

Agent/Model 配置解析顺序为：Task Override → Episode Stage Override → Project Stage Route → Project Default → System Default。Skill 配置解析顺序为：Task Skill Override → Episode Stage Override → Project Stage Skill Profile → System Stage Template。两条配置链分别解析、分别快照，不相互覆盖。

Adapter 必须动态声明 Runtime 能力、模型目录和参数 Schema；不支持显式模型选择时使用 Runtime Default，不得硬编码或虚构通用模型列表。详细规则见 Agent 接入规格。

## 5. 会话与任务：避免长期上下文膨胀

要区分三个东西：

- **Conversation（聊天记录）**：完整可查的交流历史，是产品审计与用户回看资料，不意味着每次都全部发送给模型。
- **Task/Run（任务/执行）**：一次生成、局部修改、审查或批量执行。绑定项目、分集、阶段、作用域、输入版本和执行 Skill/Runtime。
- **Artifact Revision（产物版本）**：通过业务服务提交的 Markdown/JSON 草稿或定稿，与聊天消息不等同。

**推荐**：同一分集每一正式阶段拥有独立的逻辑创作工作区和历史会话；前端可以看起来始终是同一个 Agent 聊天侧栏，但底层按照项目 + 分集 + 阶段 + 任务检索近期对话，并使用 Handoff 和明确产物版本交接。不要让一条持续增长的单一模型上下文横跨整个 A3→B4 流程。

可选的任务示例：

- A4 任务：以已定稿 Story v3 改写剧本场次 S05 的对白。
- B3 任务：根据 Breakdown v2 的 P05 生成 6 个镜头。
- B4 任务：为 SH01–SH04 编写保持连续性的提示词。

同一逻辑会话内也需控制历史摘要/最近对话长度，但具体 Token 预算是性能测试和实现细节，而非当前产品决策。

---

## 6. 精简 Handoff：系统准备什么，Agent 查询什么

### 6.1 Handoff 概念示例（非最终 Schema）

~~~yaml
project_id: PROJ01
episode_id: EP03
stage: B3
task: design_shots
scope:
  segment_ids: [P05]
approved_inputs:
  screenplay: {artifact_id: SCR03, revision: v4}
  director: {artifact_id: DIR03, revision: v2}
  breakdown: {artifact_id: BRK03, revision: v1}
mandatory_constraints:
  - 保持已确认的拟人角色基础身份和视觉风格
asset_refs:
  - {asset_id: R01, asset_revision: rev3, episode_variant_id: VAR_EP03_OFFICE}
expected_output: shot_plan_draft
allowed_reads:
  - screenplay_scene
  - segment
  - asset
  - asset_media
~~~

注意：引用已定稿版本不可在运行途中被隐式替换为最新版本。对话阶段讨论可更自由，但**正式下游产物生成**要使用已批准且可追溯的输入。

### 6.2 推荐的最小受控查询能力

| 接口概念 | 请求依据 | 返回结果 |
|---|---|---|
| get_stage_context | project/episode/stage | 当前阶段、目标、准入和约束摘要 |
| list_approved_artifacts | project/episode/stage | 可用的正式产物 ID 和版本 |
| get_artifact_excerpt | artifact_id/revision/section_id | 指定场次、章节或片段的正文/结构，不返回全项目 |
| get_scene | screenplay_revision/scene_id | 指定剧本场次结构及对应原文 |
| get_segment | breakdown_revision/segment_id | 片段/子片段及镜头规划上下文 |
| get_asset | asset_id/revision/variant_id | 角色、场景基础定义和当前分集变体 |
| list_asset_media | asset_id/revision/variant_id | 角色四视图、面部特写、服装图等媒体 ID 与用途标签 |
| get_media_ref | media_id | 可用缩略图/授权下载/运行时可访问引用，受模型能力限制 |
| submit_draft | task_id/output_contract/payload | 草稿产物 ID、校验结果、可预览引用 |
| request_review | artifact_id/draft_revision | 审核结果或供用户审查的 UI 入口 |

这些是业务能力名称，不要求一定通过 MCP 实现；可以由受控 Python tool gateway、Agent SDK function calling 或 CLI 适配器支持的桥接方式实现。**若某个 CLI 无法稳定调用这些工具，必须在 Agent Adapter 中桥接，不得宣称安装任意 Skill 后天然就支持完整工具调用。**

### 6.3 图片与多模态资产读取（关键）

以用户关心的角色图片为例：

1. 用户在项目素材页上传 R01 的 face_closeup、front、side、back 图片，并为 EP03 建立 office 造型变体。Python Media Service 登记每张图片的媒体 ID、角色视图用途与实际位置。
2. B3 Agent 接收 R01 + 基础版本 + EP03 变体的引用；无需将该角色所有图像自动塞进每次对话。
3. Agent 需要脸部镜头时，先查角色资料/图片列表，再精确请求 face_closeup 对应媒体；全身或背面镜头再请求所需视图。**图片元数据检索**与**模型实际看到图片像素**是两步：只有当前运行时/模型支持视觉输入且 Adapter 正确传输媒体时，才算真正完成看图。
4. 如果模型/CLI 只支持文本输入，则返回媒体引用、文字描述和不可用提示，不能把“拿到文件路径”当成模型已理解图片。可在支持多模态的 Adapter 中切换或延后这项功能。
5. 正式镜头/提示词保留资产版本、分集变体与媒体 ID 引用，Python 负责实际路径/URL 解析。重命名文件不会改变角色引用身份。

**不推荐**让 Agent 拿到任意项目目录的系统文件路径自行扫描、让镜头数据直接依赖字符串文件名、每次装载所有角色参考图，或让浏览器直接访问未经业务服务控制的本地磁盘路径。

---

## 7. 草稿提交、UI 预览、审查与定稿

### 7.1 推荐的执行时序

~~~text
User @ Vue/Electron
   │「为 EP03 P05 设计 6 个镜头」
   ▼
Python API + Orchestrator
   ├─ 根据当前工作区确定项目/阶段/作用域
   ├─ 验证上游已定稿
   ├─ 锁定输入版本 / 创建 Task Run
   ├─ Router → Storyboard Skill
   └─ 组装精简 Handoff
                  │
                  ▼
           Agent Runtime Adapter
             ├─ 按需查询 P05 与来源剧本场次
             ├─ 按需查询 R01 与本集变体、对应媒体
             └─ 输出结构化镜头草稿
                  │
                  ▼
           Artifact Submission Service
             ├─ 解析与 Schema 校验
             ├─ 必要来源 ID / 版本关系核对
             ├─ 写新 Markdown/JSON 草稿
             └─ SQLite 记录草稿与任务引用
                  │
                  ▼
       Vue/Electron 镜头卡片实时更新
             ├─ 用户逐镜头查看/批注
             ├─ 用户修改或让 Agent 定点优化
             └─ 用户点击「定稿」→ 系统发布正式版本
                                      └─ 开放下一阶段
~~~

Chat 流和结构化产物流要分开处理。模型回答“完成了六个镜头”不等于系统已成功提交六个符合合同的 Shot；只有业务服务返回可定位草稿，UI 才呈现“已保存”。

### 7.2 任务体验建议

Vue 工作台右侧显示聊天与执行事件，例如「正在读取 P05」「读取 R01 参考图」「正在生成镜头草稿」「已提交 6 条镜头待审查」；中间页面显示已保存的对应镜头卡片。用户可以点击其中某镜头，在对话中指定“只改 SH03 的动作，不改变其他镜头”，任务作用域由系统明确限定。

用户在 UI 编辑与 Agent 修改应通过相同的草稿版本服务；Agent 无权直接覆盖已定稿内容。点击「定稿」属于明确的用户操作，代码完成版本发布和阶段状态推进。

### 7.3 批量创作

可以批量给 EP01–EP10 创建剧本或分镜任务，但每个执行单元都要有独立 Task、输入版本、局部 Handoff 和草稿产物。建议 V1 采用有限并发的顺序/并行执行调度，不让单次 Agent 上下文同时包含十集的全文。批量结果按每集列表展示、分别定稿。

---

## 8. Agent Runtime Adapter：供应商与部署边界

### 8.1 为什么建议做 Adapter

Python Orchestrator 不应该把业务逻辑写成“拼接一条 Codex CLI 终端命令并解析终端彩色文字”。推荐一个逻辑适配接口：

~~~text
start_run(task, handoff, skills, allowed_tools) -> run_id
stream_events(run_id) -> typed events
provide_tool_result(run_id, call_id, result) -> optional
cancel_run(run_id)
get_capabilities(runtime) -> tool_calling, image_input, session_resume, structured_output...
~~~

这里是设计目标，不承诺所有 CLI/SDK 都有完全相同的能力；无法支持的能力必须明确显示或由适配层替代。

### 8.2 候选运行方式

- **本机 CLI 执行**（如用户机器安装的 Codex CLI / Claude Code CLI）：适合 Electron 本地桌面场景，复用现有账户和本机工具能力的可能性需按具体 CLI 版本和实际授权方式验证；不默认保证共享 ChatGPT 网页端额度。CLI 的工具调用、图片、结构化输出、会话恢复与自动化模式需要单独做能力测试。
- **远端 Agent Host/CLI**：Web 用户经 Python 服务提交任务，CLI 在受控服务器/Agent Host 上执行；不能把远端网站操作理解为浏览器能直接调用终端用户的本地 CLI。多用户时还需要任务/文件/凭证隔离。
- **SDK/API**：可以作为同一 Adapter 接口下的另一实现，便于服务端部署和模型能力管理，但用户尚未批准 V1 必须接入外部 API。

### 8.3 已确认的 V1 路线与未定细节

**已确认**：V1 优先 CLI 运行时生态，只要求优先跑通一个可靠的 Agent；如目标 Agent 有官方 Agent SDK，优先评估 SDK 作为 Python 的程序控制接口，必要时再通过 CLI 或其他受控协议接入；直接模型 API 后续扩展。SDK 与 CLI 是否支持 Skill 加载、项目 cwd、多模态媒体和工具权限，需要按具体版本验证。项目必须具备统一 Runtime Adapter，让后续供应商切换不需要改写阶段业务。

**仍未定稿**：先实际支持 Codex 或 Claude、具体 SDK/CLI 版本、凭据与权限隔离实现、Web/桌面进程宿主。Web 端若运行于远端，必须由 Python 服务所在主机或独立 Agent Host 执行，而非直接调用浏览器用户电脑上的 CLI。严格的按项目文件权限是已确认要求，具体隔离机制不能自动降级。

---

## 9. Skill 输入/输出合同与 ArtifactService

Skill Registry 负责来源、版本、能力、兼容性和阶段绑定；ArtifactService 负责 CinePilot 正式产物的结构与保存。外部 Skill 的输出不能直接视为 CinePilot 定稿。

### 9.1 Skill 执行合同

每个 Skill Version 应声明：
- 能力标签、适用阶段和任务意图；
- 必需输入、可选输入、受控工具需求；
- 输出 Artifact 类型和字段映射；
- 依赖 references、脚本、模型或外部工具；
- 运行时兼容性与安全审核；
- License、作者、来源 Commit、NOTICE 和改造说明。

具体 Registry 字段和阶段 Profile 见 [Skill Registry 与阶段编排规格](./stage-skill-orchestration-and-registry.md)。

### 9.2 输出适配和校验

- Agent 按 Skill Plan 执行，提交候选 Markdown/JSON 草稿。
- Skill Adapter 将外部 Skill 的原始输出映射为 CinePilot 阶段合同。
- ArtifactService 校验 Schema、对象归属、来源 Revision、修改范围和引用关系。
- 校验成功后才写入草稿版本；模型回复“完成”不等于已保存。
- StageService 仍负责阶段整体审查和用户定稿；Skill 不能直接批准阶段。
- Wind Comic 等外部 Skill 即使输出 scenes + shots，也必须按 CinePilot A4/B2/B3 阶段边界拆分，不允许跳过中间阶段。

### 9.3 版本和回归

每次 Run 固定 Skill Plan 与每个 Skill 的 CinePilot Version、上游 Commit 和入口文件 SHA。升级产生候选版本，经过 License、安全、Runtime 兼容、结构和质量回归后再启用；历史 Run 和已定稿产物不因 Skill 更新自动重做。

## 10. 开发启动与下一步

完整的开发顺序和可执行验收见 [开发启动与实施路线图](./development-kickoff-and-implementation-roadmap.md)。

建议先完成最小垂直闭环：Project/Workspace/SQLite → Artifact Draft/Revision/StageApproval → Agent Adapter/项目隔离 → Skill Registry/Stage Profile/Skill Plan → 单一 A1 或 A3 任务端到端生成并保存草稿。然后逐步扩展八阶段编辑器、资产、批量任务和导出。

**确认/草案边界**：CLI 优先、官方 Agent SDK 可用时优先评估、项目隔离、统一服务写入、阶段整体定稿、上游变化只标记下游待复核均为已确认约束；具体首个 Runtime、模型目录能力、沙箱实现、Skill 质量评测和各阶段缺失 Skill 仍需验证/补充。
