# CinePilot Studio — Agent / Skill 协作与运行架构

> 版本：0.1（整体设计草案）
> 日期：2026-10-30
> 状态：**已有 ADR 和用户约束**与**本文件推荐的 Agent 实现设计**分开；V1 运行时选择未确认
> 依据：[整体设计总览](./overall-product-workflow-and-system-map.md)、[八阶段合同](./stage-contracts-and-artifact-map.md)、[资产模型](./creative-asset-model-and-use.md)、[ADR-0007/0008/0009](../adr/)、[ADR-0010](../adr/0010-compact-versioned-agent-handoff.md)、[ADR-0011](../adr/0011-mandatory-handoff-context-controlled-retrieval.md)、[ADR-0012](../adr/0012-field-owned-markdown-json-sqlite.md)、[ADR-0015](../adr/0015-project-assets-with-episode-variants.md)

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

## 4. Agent 与专业 Skill：谁路由、何时调用

**推荐 V1：一个对话入口，系统路由到专业 Skill，而不是先建固定的 8 个专属 Agent 服务。**

### 4.1 意图类型（草案）

- **对话与探索**：回答问题、讨论创意、比较方案。不自动保存为正式产物。
- **生成/续写**：在当前阶段创作新草稿或扩展候选方案。
- **定点修改**：只修改 Scene S03、Segment P05、Shot SH10 或某一角色变体。
- **评审/解释**：按阶段合同检查已有草稿，给建议和差异。
- **结构化资产操作**：创建角色草稿、添加参考媒体、为分集选择资产变体。
- **阶段/版本操作**：用户在 UI 显式确认后由业务服务执行，如定稿、进入下一阶段、沿用旧下游版本。
- **批量任务**：对一批符合阶段条件的分集逐集执行，同一模型或 Skill 可以复用配置但不混合不同集的版本和结果。

### 4.2 专业 Skill 建议组合

| 业务阶段 | 主 Skill（建议） | 可能组合的辅助 Skill |
|---|---|---|
| A1 点子 | Idea Discovery | 题材/创意评估 |
| A2 创意方向 | Concept Development | 比较/筛选 |
| A3 故事 | Story Architect | 类型剧情技能、叙事审查 |
| A4 剧本 | Screenwriter | 对白、风格、剧本校验 |
| B1 导演 | Director | 表演、视听风格 |
| B2 制作拆解 | Breakdown Planner | 资产需求核对、场次映射校验 |
| B3 分镜 | Storyboard/Shot Designer | 动作连续性、角色一致性 |
| B4 提示词 | Video Prompt Engineer | 模型适配、提示词一致性 |

这些是**能力角色**，不等于必须拆成 8 个文件仓库、8 个 CLI 会话、8 个付费模型或 8 个后台进程。支持根据实际 Skill 质量把相近能力合并。

### 4.3 路由策略

1. 系统先从 UI 上下文定位项目/分集/阶段/当前选中的内容。**用户显式目标优先于自然语言模糊推断**。
2. 分流规则先判断：只读讨论、草稿操作、正式阶段推进、资产操作。
3. 代码校验权限、正式阶段准入与上游版本是否存在。
4. Router 从允许的 Skill 清单选择主 Skill，可附加少量专用检查/风格 Skill。
5. 选取 Agent Runtime Adapter 执行。用户应能在 UI 查看将使用的目标阶段和 Skill，后续可添加手动覆盖选项。
6. Agent 输出必须进入对应草稿/产物服务；纯聊天不等于已持久化定稿。

**不推荐**：每次用户聊天让 Agent 把项目的 8 个 Skill 和每份长文档都装载；也不推荐在 Prompt 里让 Agent 自己决定能否越过阶段门禁。

---

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

### 8.3 推荐 V1 路径（待确认）

优先保证一个真实可用的执行运行时和端到端创作闭环，再把第二个 CLI 或 API Runtime 作为适配器扩展；不要一上来同时维护多种 CLI 会话语义和复杂多 Agent 迁移。项目本身必须保留 Agent Runtime Adapter 层，避免之后替换供应商需要改写阶段代码。

**尚需一个产品级决策**：V1 是以本地 CLI（如 Codex/Claude）为首要 Agent 执行源，还是首期要求本地 CLI 与云端 API 并行。这个决策会明显影响 Web 和 Electron 的可用部署形态，应在技术架构定稿时向用户确认。

---

## 9. Skill 注册与产物合同（推荐）

### 9.1 注册描述建议

~~~yaml
skill_id: storyboard_shot_designer
version: 0.1.0
stages: [B3]
intents: [create_shots, revise_shot, review_shots]
required_approved_sources: [screenplay, director, breakdown]
required_tools: [get_segment, get_scene, get_asset, list_asset_media, submit_draft]
optional_capabilities: [image_input]
output_contract: shot_plan
~~~

Skill 只描述创作工作、输入依据、目标产物和质量提示；是否允许执行由 Python 代码检查，不能依赖 Skill 在提示词里“自觉”。

### 9.2 可观察、可测试的最小链路

- 给 A4 一个已定稿故事版本，能运行剧本 Skill，并在后台出现剧本草稿 + 场次索引。
- 给 B2 已定稿片段 P05，能运行 B3 专业 Skill，查到指定场次/角色资产，只产生作用域内镜头草稿。
- 指定 R01@rev3 + EP03 变体时，Agent 取得的是匹配本集的角色描述/媒体，而不是错误分集的造型。
- 用户点击定稿后，旧定稿仍能打开，下游正式阶段能以新版本 Handoff 启动。
- 上游被修改后，下游标记待复核而不是自动重新生成。
- 重新打开 Vue Web/Electron 工作台，可以从持久化存储恢复正确的内容和任务记录，而非依赖之前的聊天窗口。

---

## 10. 本设计完成后，下一步工作

1. **先确认 Agent 运行源**：V1 首要支持本机 CLI、远端 CLI Agent Host，还是直接 SDK/API；这将影响 Python 执行宿主和 Electron/Web 体验。
2. 将上述能力分解为具体 Python 服务/接口合同（Agent Adapter、Artifact Service、Media Service、Task API）。
3. 设计 Vue 工作台右侧聊天 + 中间产物面板的真实页面状态与交互。
4. 再确定数据实体与 JSON/Markdown Schema、SQLite 关系、任务事件格式。
5. 将 V1 场景验收与开发任务拆解，以可运行闭环检验，而不是只编写大量抽象架构文档。

**提醒**：本文件中“Agent Runtime Adapter”“按阶段逻辑会话”“推荐工具列表”“CLI 优先”等均是工程与产品设计建议；未经过用户确认前不建立新的 ADR 把它们标记为已定稿。
