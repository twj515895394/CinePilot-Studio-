# CinePilot Studio — 领域数据模型、SQLite 索引与 Markdown/JSON 映射

> 版本：0.1（可指导后端接口设计的候选数据合同）
> 日期：2026-10-30
> 状态：**已有 ADR 约束是已确认的产品事实；本文件的表名、字段、SQL、Schema 示例为推荐设计，尚非最终迁移脚本。** 不新增未经用户批准的核心领域关系。
> 依赖：[项目目录规范](./project-workspace-directory-layout.md)、[八阶段合同](./stage-contracts-and-artifact-map.md)、[资产模型](./creative-asset-model-and-use.md)、[Agent/Skill 架构](./agent-skill-orchestration-and-runtime.md)、[ADR-0001~0019](../adr/)

## 0. 设计目标与已确认边界

目标：在 Python 后端定义一套稳定数据合同，同时服务 Vue/Electron、Codex/Claude Agent 查询工具、专业 Skills 和完整 A1–B4 创作流水线。管理后台可以按项目/分集/阶段/版本打开创意、剧本、导演、分镜与提示词；Agent 精确引用已定稿版本且不跨项目。

**已确认：**
- 共享一个系统级 SQLite，建议逻辑位置 \`cinepilot-data/app/cinepilot.sqlite\`；Agent 不直接打开数据库。
- 每个 Project 有独立项目目录，目录的业务分层：\`creative-pool/\`、\`assets/\`、\`media/\`、\`episodes/{episode_id}/stages/{a3..b4}/artifacts/{artifact_id}/{drafts|revisions}/\`，以及 Agent 临时工作区。
- A1 Idea 和 A2 Concept 在分集创建前属于项目。Concept 独立保存/修改/筛选，用户选择方向并决定开发时才创建 Episode。
- 一个 Episode 是一条独立视频，有且仅有一个 Primary Idea。
- A4 文学 Scene 与 B2 导演 Segment 是不同对象；一个 Scene 可映射多个 Segment，每个 Segment 有且只有一个来源 Scene。Segment 直属 Shot 或拥有最多一层 Subsegment→Shot，不能混放。
- 项目基础 Asset 在同项目多集复用；Episode 可持有不回写基础定义的 Variant。
- 正式阶段以上游已批准版本准入；定稿后不可原地覆写；新批准上游只触发下游待复核，不自动重做。
- 每个字段只有一个写入权威：SQLite（身份/跨对象关系/版本/审批/状态）、Markdown（创作正文）、JSON（创作结构和属性）；系统统一写入和校验。
- 先准备不可变内容文件，再通过 SQLite 发布正式版本引用。

**设计原则**：不建重复可写的第二套剧本/镜头实体库；不为每个镜头强制生成物理文件；不提前引入外部知识库、向量数据库或微服务拆分。

---

## 1. 总体关系图（逻辑实体，不等于数据库表）

~~~mermaid
erDiagram
    PROJECT ||--o{ IDEA : contains
    PROJECT ||--o{ CONCEPT : contains
    IDEA ||--o{ CONCEPT : develops
    PROJECT ||--o{ EPISODE : contains
    IDEA ||--o{ EPISODE : primary-source
    CONCEPT ||--o{ EPISODE : selected-from
    EPISODE ||--o{ ARTIFACT : stage-artifact
    ARTIFACT ||--o{ REVISION : revisions
    REVISION ||--o{ REVISION_DEPENDENCY : downstream
    PROJECT ||--o{ ASSET : defines
    ASSET ||--o{ ASSET_VARIANT : variant-base
    EPISODE ||--o{ ASSET_VARIANT : owns
    PROJECT ||--o{ MEDIA : stores
    ASSET ||--o{ ASSET_MEDIA_LINK : references
    MEDIA ||--o{ ASSET_MEDIA_LINK : reused-by
    PROJECT ||--o{ AGENT_RUN : executes
    EPISODE ||--o{ AGENT_RUN : optional-target
    PROJECT ||--o{ PROJECT_SKILL : enables
~~~

- \`EPISODE → CONCEPT\` 只表达“可追溯被选中的来源”，不擅自增加 Concept 与 Episode 唯一绑定的约束；同一 Concept 能否被多集重复选中尚未由用户确认。
- \`EPISODE → AGENT_RUN\` 可选，因为 A1/A2 在项目创意池进行，尚无 Episode。
- \`REVISION_DEPENDENCY\` 表达批准版本之间的依赖；场次→片段→镜头的详细图参见 §5。
- \`STAGE\` 首期建议使用固定代码表/枚举，不要求为每个阶段建立单独数据库。

---

## 2. 字段的唯一事实来源（尤其防止 SQLite 与 JSON 互相打架）

| 字段/事实类别 | 唯一可写权威 | 其他载体允许出现什么 |
|---|---|---|
| Project/Episode/Idea/Concept/Asset/Artifact 的稳定身份和归属 | **SQLite** | JSON 中以 ID 作为引用/自我标识，不能单独改 ID 建立新实体 |
| Episode 的 primary_idea / selected_concept 关系 | **SQLite** | 文件可序列化显示来源 ID，但关系更新必须经过服务 |
| Artifact 所属分集/阶段及当前正式修订 | **SQLite** | 文件只带供人工理解的 ID 与来源引用快照 |
| Draft/Approved、定稿时间、审批人、阶段准入、待复核状态 | **SQLite** | 文件可以有导出摘要，但不可自行变更数据库审批事实 |
| Markdown 文学剧本、故事大纲、导演阐述正文 | **Markdown** | SQLite 仅存内容定位、大小、校验值及可重建全文索引；JSON 不可写另一份正文 |
| JSON 创意细节、节拍结构、场次描述、片段属性、镜头动作/构图/时长、提示词字段 | **JSON** | SQLite 可存由 JSON **派生的只读查询投影**，不作为第二个编辑入口 |
| 不同 Artifact/Revision 间的来源、依赖边 | **SQLite** | JSON/Markdown 可带 ID 作为可携带的引用快照，由服务验证/规范化 |
| Asset 基础身份与分集 Variant 的归属关系 | **SQLite** | JSON 保存基础外观、造型、服装与状态等创作属性 |
| Media 的 ID、归属、文件位置及资产使用关系 | **SQLite** | 图片二进制存于项目 \`media/\`；JSON 可引用 media_id |
| 文档中的对象顺序、文本位置、镜头叙述及创作结构 | **JSON/Markdown** | SQLite 中的顺序/定位字段若存在，须声明为**自动重建的查询投影** |

### 如何避免重复权威的典型错误

\`shots.json\` 里有 \`shot_id, order, action, camera_move\`，它是镜头**创作结构与属性**的权威。SQLite 可以有 \`shot_lookup(project_id, episode_id, revision_id, shot_id, order_cache, json_pointer)\`，但 \`order_cache\` 和 \`json_pointer\` 是服务在文件保存后自动重建、只读的搜索索引，不能在 SQLite 修改它以改变实际镜头顺序。

\`Segment → source Scene\` 和 \`Shot → 叶级 Segment/Subsegment\` 的**正式跨产物引用**应在 SQLite 中有可追溯关系/版本约束；对应 JSON 也可以展示来源 ID，作为由业务服务生成/校验的只读引用副本，不允许 SQLite 和 JSON 各自改关系而悄悄分叉。若用户通过镜头编辑器修改引用关系，业务服务必须统一处理该次修改和关系索引。

**一句话：相同 ID 在多个位置出现是“引用”，不是多份权威。**

---

## 3. 建议的 SQLite 实体组

### 3.1 项目、创意方向、分集

| 建议表 | 核心字段（示意） | 不变量 |
|---|---|---|
| \`projects\` | id, title, workspace_root_key, created_at, archived_at | 项目 ID 不从用户输入名称推导；物理工作区由服务确定 |
| \`ideas\` | id, project_id, content_ref, created_at | 只能属于一个项目；标题/钩子等内容在 JSON |
| \`concepts\` | id, project_id, idea_id, content_ref, selection_state, created_at | 来源 Idea 属于同一项目；选用状态由 SQLite 管 |
| \`episodes\` | id, project_id, primary_idea_id, selected_concept_id, title, created_at | 必须有且只有一个 Primary Idea；创建时需有被选 Concept；三者项目一致 |
| \`stage_instances\` | episode_id, stage_code, progress_state, latest_approval_id | 每个 Episode 每个正式阶段最多一条进度记录；状态属于 SQLite |

**不要强加的规则**：\`selected_concept_id UNIQUE\`、Concept 一旦选中不可修改、每集强制有 6 个已定稿产物。这些都不是用户已确认事实。

A1/A2 不创建 Episode 或 StageInstance 的空记录。创意池保存与多版本编辑可走受控内容版本服务，但不套用 A3–B4 的正式阶段审批门禁。

### 3.2 阶段产物与修订

| 建议表 | 核心字段（示意） | 说明 |
|---|---|---|
| \`artifacts\` | id, project_id, episode_id, stage_code, artifact_type, created_at | 阶段产物长期身份，不等于某一次文件 |
| \`artifact_revisions\` | id, artifact_id, revision_no, lifecycle_state, parent_revision_id, created_at, approved_at, created_by, approved_by | \`draft\` / \`approved\` 由 SQLite 管；已批准修订不可改写 |
| \`revision_files\` | revision_id, file_role, relative_path, sha256, media_type | 具体内容的版本定位；仅存项目相对路径；MD/JSON 为正文/结构权威 |
| \`stage_approvals\` | id, episode_id, stage_code, approved_at, approved_by | 一个“阶段已定稿”的业务锚点，不只等于一个文件存在 |
| \`stage_approval_items\` | stage_approval_id, artifact_revision_id, required_flag | 本次阶段批准到底包含哪些产物修订 |
| \`revision_dependencies\` | downstream_revision_id, upstream_revision_id, dependency_type | 版本级来源边，用户可以回溯批准时使用的具体上游 |
| \`review_flags\` | id, downstream_revision_id, triggering_upstream_revision_id, state, resolved_at | 受上游新定稿影响的提示，不替换、重做或删除旧产物 |

**为什么需要 StageApproval？** 如果某阶段有多个必需产物（如 \`screenplay.md + scenes.json\`，或故事大纲 + 节拍表），不能仅凭某一个修订已定稿就自动判断“整个阶段通过”。即使首期大多数阶段使用一个逻辑 Artifact，保留“阶段审核这一组产物版本”的概念更清晰；具体是否单独建 \`stage_approvals\` 表由实施时决定，不要求一开始就过度拆表。

重要规则：
1. \`artifact_revisions\` 只记录身份与状态，不复制全文。
2. \`revision_files\` 的路径必须限制在本项目内，不得接受来自 Agent 的任意绝对路径。
3. 已批准 Revision 的文件、来源依赖及审批记录不可原地修改；更改产生新草稿。
4. 上游新批准后，按 \`revision_dependencies\` 产生待复核提示，不自动重跑下游。
5. 正式 Stage Handoff 绑定具体 \`stage_approval_id\` 和 Artifact Revision ID，而不是仅引用“最新阶段目录”。

### 3.3 资产、变体、媒体与使用关系

| 建议表 | 核心字段（示意） | 不变量 |
|---|---|---|
| \`assets\` | id, project_id, asset_type, current_revision_id | 项目级基础资产身份，不能跨项目自动共享 |
| \`asset_revisions\` | id, asset_id, revision_no, content_ref, state | 基础描述在 JSON；按需保存不可变版本 |
| \`asset_variants\` | id, project_id, episode_id, base_asset_id, content_ref | 分集造型/状态属于本集，不能直接改基础资产 |
| \`asset_variant_revisions\` | id, variant_id, revision_no, content_ref | 变体在 JSON 中保存不同服装/造型等变化 |
| \`media\` | id, project_id, relative_path, mime_type, checksum | 实际图片位于该项目 \`media/\` |
| \`asset_media_links\` | asset_revision_id 或 variant_revision_id, media_id, image_role | \`face_closeup\`、\`front\`、\`side\`、\`back\`、\`costume_ref\` 等用途标签 |
| \`episode_asset_uses\` | episode_id, base_asset_id, base_revision_id, variant_revision_id? | 分集显式使用哪个基础版本与可选变体 |

**设计注意**：\`asset_media_links\` 最终应通过规范化外键或约束设计避免“基础版本和变体版本都未填/同时错误填”的记录；表中只列逻辑字段。媒体引用会在镜头 JSON 中以 \`media_id\` 形式出现，具体媒体身份与用途仍由 SQLite 验证。

### 3.4 Agent、Skill 与会话

| 建议表 | 核心字段（示意） | 说明 |
|---|---|---|
| \`skill_registry\` | id, version, location_key, capabilities | 系统管理的 Skills 版本和使用能力 |
| \`project_skill_bindings\` | project_id, skill_id, version, enabled | 当前项目启用 Skill；运行时只读投射 |
| \`agent_runs\` | id, project_id, episode_id?, stage_code?, task_type, task_scope_json, runtime_id, state, created_at | A1/A2 允许 episode_id 为空；每次任务显式绑定项目 |
| \`agent_run_inputs\` | run_id, approved_revision_id, input_role | Run 使用哪些批准来源；Project 级创意任务可用合法保存的 Idea/Concept 引用 |
| \`conversations\` | id, project_id, episode_id?, stage_code?, created_at | 完整会话独立于正式产物 |
| \`messages\` | id, conversation_id, role, body_ref?, created_at | 聊天记录可独立存储；不默认全量喂模型 |

建议 V1 可以先只实现必要字段，进度事件可由 \`agent_run_events\` 或运行时事件流承载；**不要求为每个大模型供应商建一组独立业务表**。

---

## 4. 项目目录与数据库记录如何映射

~~~text
cinepilot-data/
├── app/cinepilot.sqlite                   # 所有项目共享，Python 业务服务专用
├── skill-registry/
└── projects/P001/
    ├── creative-pool/
    │   ├── ideas/I001/idea.json
    │   └── concepts/C001/concept.json
    ├── assets/characters/R01/
    │   └── revisions/AR03/asset.json
    ├── media/M001/original.png
    ├── episodes/EP03/
    │   ├── asset-variants/VAR01/
    │   │   └── revisions/VR02/variant.json
    │   └── stages/
    │       ├── a3-story/artifacts/STORY03/revisions/REV02/
    │       │   ├── story.md
    │       │   └── beats.json
    │       ├── a4-screenplay/artifacts/SCR03/revisions/REV03/
    │       │   ├── screenplay.md
    │       │   └── scenes.json
    │       ├── b1-director/
    │       ├── b2-breakdown/artifacts/BRK03/revisions/REV01/
    │       │   └── breakdown.json
    │       ├── b3-storyboard/artifacts/SHT03/revisions/REV02/
    │       │   └── shots.json
    │       └── b4-prompts/artifacts/PRM03/revisions/REV01/
    │           └── prompts.json
    └── .agent-work/RUN001/                 # Agent 临时草稿工作区
~~~

此处 \`REVxx\`、\`ARxx\`、\`VRxx\` 仅为演示，实际需使用不会与其他对象冲突的稳定 ID。项目级创意 \`idea.json\` / \`concept.json\` 的**内部多版本保存规范尚未定稿**；不能因为示例只显示当前文件就否定 Concept 可独立修改或可追溯选中时的内容。建议它们同样可采用文件快照/独立内容修订，但不必执行正式分集阶段审批。

### 4.1 文件路径应是由系统生成的相对定位

推荐 \`revision_files\` 保存：
- \`project_id = P001\`
- \`revision_id = REV03\`
- \`file_role = screenplay_body\`
- \`relative_path = episodes/EP03/stages/a4-screenplay/artifacts/SCR03/revisions/REV03/screenplay.md\`
- \`sha256\`（文件完整性检测）

Python 通过受控的 Project Root + RelativePath 定位文件，并校验真实路径仍属于项目根；不要把请求参数直接当成磁盘绝对路径。Agent 不获得系统 SQLite、其他项目目录或未授权媒体访问权。

---

## 5. Scene → Segment → Subsegment → Shot 的版本关系

已有 ADR 已确认的是**领域关系**，不是要求把每个 Scene、Segment、Shot 各建独立 Markdown 文档。

### 5.1 A4 的 \`scenes.json\`（建议结构）

~~~json
{
  "schema_version": "1.0",
  "screenplay_revision": "REV03",
  "scenes": [
    {
      "scene_id": "S01",
      "order": 1,
      "heading": "办公室 / 日 / 内",
      "summary": "角色发现异常",
      "body_anchor": "scene-S01",
      "character_asset_ids": ["R01"],
      "location_asset_ids": ["L01"]
    },
    {
      "scene_id": "S02",
      "order": 2,
      "heading": "走廊 / 日 / 内",
      "summary": "追逐开始",
      "body_anchor": "scene-S02",
      "character_asset_ids": ["R01"],
      "location_asset_ids": ["L02"]
    }
  ]
}
~~~

\`screenplay.md\` 使用稳定场次锚点或业务服务维护的精准段落定位；源正文不复制进 JSON。若 \`body_anchor\` 格式在重排中失效，受控服务负责刷新；具体编辑器同步方案后续设计。

SQLite 可建立 **只读投影** \`scene_lookup(revision_id, scene_id, order_index, body_locator)\`，用于 \`get_scene(REV03,S02)\` 按 ID 直接返回一个场次，而不是整集剧本。

### 5.2 B2 的 \`breakdown.json\`（建议结构）

~~~json
{
  "schema_version": "1.0",
  "source_screenplay_revision": "REV03",
  "segments": [
    {
      "segment_id": "P01",
      "source_scene_id": "S01",
      "order": 1,
      "title": "发现异常",
      "structure_mode": "direct_shots",
      "subsegments": []
    },
    {
      "segment_id": "P02",
      "source_scene_id": "S02",
      "order": 2,
      "title": "走廊追逐",
      "structure_mode": "subsegments",
      "subsegments": [
        {"subsegment_id": "P02-A", "order": 1, "purpose": "起跑"},
        {"subsegment_id": "P02-B", "order": 2, "purpose": "障碍跨越"}
      ]
    }
  ]
}
~~~

**必须遵守的校验**：
- \`source_scene_id\` 必须存在于指定已定稿剧本修订 \`REV03\`，并属于同一 Episode；每 Segment 仅一个来源 Scene。
- 此处 \`structure_mode\` 是建议显式记录的结构策略，仍由真实 \`Shot\` 归属校验“一个 Segment 不能同时有直属 Shot 与子片段 Shot”。
- 当使用子片段时最多一层；不递归创建无限层级。
- B2 负责定义片段/子片段，**不提前强制要求 B2 已拥有 B3 的镜头内容**。
- \`source_screenplay_revision\` 是跨产物关系的可携带引用快照，正式关系由 SQLite 的版本级来源引用维护。

### 5.3 B3 的 \`shots.json\`（建议结构）

~~~json
{
  "schema_version": "1.0",
  "source_breakdown_revision": "REV_BRK01",
  "shots": [
    {
      "shot_id": "SH01",
      "leaf_type": "segment",
      "leaf_id": "P01",
      "order": 1,
      "duration_sec": 4.0,
      "framing": "近景",
      "camera_move": "缓慢推近",
      "action": "角色抬头发现异常",
      "asset_usage": [
        {
          "asset_id": "R01",
          "asset_revision_id": "AR03",
          "episode_variant_revision_id": "VR02",
          "media_roles": ["face_closeup"]
        }
      ]
    },
    {
      "shot_id": "SH02",
      "leaf_type": "subsegment",
      "leaf_id": "P02-A",
      "order": 2,
      "duration_sec": 3.5,
      "framing": "全景",
      "camera_move": "跟拍",
      "action": "角色冲出办公室",
      "asset_usage": [
        {
          "asset_id": "R01",
          "asset_revision_id": "AR03",
          "episode_variant_revision_id": "VR02",
          "media_roles": ["front"]
        }
      ]
    }
  ]
}
~~~

**重要**：Shot 的 \`leaf_type/leaf_id\` 必须引用本集 B2 已定稿结构中的某个可容纳镜头的叶节点。若 \`P02\` 采用子片段模式，则 Shot 只能放入 \`P02-A\` 或 \`P02-B\`，不能直接放 \`P02\`。镜头与基础资产、分集变体的归属关系须经 SQLite 验证。

**SQLite 索引建议**（服务自动生成的只读投影）：
- \`scene_lookup(revision_id, scene_id, locator)\`
- \`segment_lookup(revision_id, segment_id, source_scene_id, locator)\`
- \`shot_lookup(revision_id, shot_id, leaf_type, leaf_id, locator)\`

这里的 \`source_scene_id\` / \`leaf_id\` 列用于查询和关联校验。它们不能成为绕开 JSON/业务服务单独修改镜头创作结构的入口；正式跨产物来源关系要随 Revision Dependency 一起固定。

**版本说明**：同一 ID 如 \`SH02\` 可在旧修订和新修订中指代同一逻辑镜头，但必须与修订 ID 组合确定当时的具体内容。删除镜头后，旧修订中的 \`SH02\` 仍可查。

### 5.4 B4 的 \`prompts.json\`

~~~json
{
  "schema_version": "1.0",
  "source_shot_revision": "REV_SHOTS02",
  "prompts": [
    {
      "shot_id": "SH01",
      "prompt": "拟人化警犬在办公室抬头，摄影机缓慢推近，保持原面部造型一致",
      "asset_refs": ["R01"],
      "media_roles": ["face_closeup"],
      "notes": "使用镜头 SH01 的已定稿动作与机位"
    }
  ]
}
~~~

Prompt 关联 Shot ID 与明确的 B3 批准版本；在 B3 镜头改版后，旧 B4 仍保留并标记待复核，不能直接替换内容。

---

## 6. 资源版本、阶段批准、Agent Handoff 的一条完整链路

例：P001 / EP03 的故事已经定稿，用户要做「P02 走廊追逐」的镜头设计。

1. 用户在 Vue/Electron 打开 EP03 的 B3，点击「设计 P02 镜头」。
2. Python 在 SQLite 中验证 P001/EP03 的身份和权限，查询 B2 阶段批准记录及 \`BRK03@REV01\`，并读取该修订的来源 \`SCR03@REV03\`。
3. Handoff 预置任务目标、重要视觉约束、具体 Segment P02、以上精确批准修订、当前分集对 R01 选用的版本/变体。
4. Agent 受控查询 \`get_segment(P02, REV01)\`、\`get_scene(S02, REV03)\`、\`get_asset(R01, AR03, VR02)\`；如需图片，通过 \`list_asset_media\` 取得授权媒体引用。
5. Agent 只把草稿文件写到本项目 \`.agent-work/RUN123\`，或调用受控 \`submit_draft\`；SQLite 的项目归属/版本与 JSON Schema 校验通过后，Python 才提交 B3 草稿。
6. UI 显示镜头卡片、用户反馈并定稿后，服务保存不可变 \`shots.json\` 修订，在 SQLite 建立 \`B3_REV02 → B2_REV01\` 等版本级依赖。
7. 后续 B4 仅依据 B3 正式批准修订生成提示词。
8. 如 B2 重新定稿为 \`REV02\`，已有 B3/B4 内容仍可打开，SQLite 中记录相应待复核，不自动重做。

**禁止的捷径**：直接通过 Agent 修改 \`cinepilot.sqlite\`、按当前文件路径猜测最新定稿、从另一个项目读取 R01、在 B2 未批准时正式生成 B3、只让聊天输出“完成”但不保存结构化草稿。

---

## 7. 受控查询 API 的最低数据合同（设计示意）

| 能力 | 输入 | 返回 |
|---|---|---|
| \`list_projects\` | 当前 UI/账户权限 | 项目摘要/阶段数量，不暴露文件绝对路径 |
| \`get_episode_overview\` | project_id, episode_id | 该集阶段状态、已批准产物和待复核提示 |
| \`get_stage_artifacts\` | project_id, episode_id, stage | 该阶段产物清单和草稿/定稿修订索引 |
| \`get_artifact\` | project_id, artifact_id, revision_id | 指定版本正文/结构；根据角色读取必要部分 |
| \`get_scene\` | project_id, episode_id, screenplay_revision_id, scene_id | 指定场次正文/描述，不加载全部剧本 |
| \`get_segment\` | project_id, episode_id, breakdown_revision_id, segment_id | 指定片段/子片段详情及来源场次 |
| \`get_shot\` | project_id, episode_id, shot_revision_id, shot_id | 一个镜头的详细属性、引用 |
| \`get_asset\` | project_id, episode_id?, asset_id, asset_revision_id?, variant_revision_id? | 角色/场景基础设定与本集变体 |
| \`list_asset_media\` | project_id, asset_id, revision_id, role? | 授权的多视图媒体 ID 和用途 |
| \`submit_draft\` | run_id, stage, content payload, base revision ID | 新草稿 ID、校验结果、可在 UI 查看链接 |
| \`approve_stage\` | project_id, episode_id, stage, submitted revision IDs, explicit user action | 新 StageApproval ID + Approved Revision 引用 |

除 \`list_projects\` 外，Agent 专用工具可更严格地自动绑定 Run 的 \`project_id\`，不允许通过传入任意 ID 切换项目。文档读取先按 ID/版本精确定位，不需要预先复制全文到 SQLite；可在后续建立 SQLite 全文索引或结构化字段投影提高搜索体验。

---

## 8. 约束与索引建议（不是最终 SQL DDL）

### 8.1 唯一性/归属/外键

- \`projects.id\` 全局唯一；\`episodes.id\` 全局唯一且带项目外键；Concept/Idea/Asset/Media 所有权必须与目标 Episode 的 project_id 一致。
- 任何 Entity ID 的引用服务端校验归属项目和具体版本，不允许只比对字符串格式。
- \`artifacts\` 的 \`episode_id\` 与 \`project_id\` 一致，\`stage_code\` 只允许 A3、A4、B1、B2、B3、B4 正式阶段。
- 对已批准修订及其来源的修改必须新建修订，不能通过 UPDATE 覆盖事实。
- \`stage_approval_items\` 中的产物都要属于该 Episode/Stage 且具备正式批准状态。
- \`asset_variants\` 的项目/分集与基础资产一致，不允许跨项目挂载。
- \`agent_runs\` 的目录、查询和提交操作都必须被当前 project_id 绑定。

### 8.2 性能索引（实施阶段评估实际查询计划）

- \`episodes(project_id, created_at)\`
- \`concepts(project_id, idea_id, selection_state)\`
- \`artifacts(episode_id, stage_code, artifact_type)\`
- \`artifact_revisions(artifact_id, revision_no)\`
- \`revision_dependencies(upstream_revision_id, downstream_revision_id)\`
- \`agent_runs(project_id, episode_id, stage_code, created_at)\`
- \`scene_lookup(revision_id, scene_id)\`
- \`shot_lookup(revision_id, shot_id)\`
- \`media(project_id, id)\`

**读取策略**：通常只做一次 SQLite 索引/权限/版本查询，然后读取目标 Markdown/JSON 及必要区段。对小文件直接读通常足够快；跨大量文档全文搜索才考虑 FTS，中文分词方案要在实际数据下测试。不要凭“数据库肯定更快”改写正文权威。

### 8.3 项目备份/迁移

因用户已确认系统单 SQLite，拷贝 \`projects/P001/\` **不等于**完整可迁移项目；导出服务应打包该项目的相关业务元数据（身份/关系/版本/审批/依赖）与项目 MD/JSON/媒体，并在新系统重新登记映射，而不是让其他项目的数据库记录随意混入。具体导出格式和 ID 冲突策略属于后续实施规范。

---

## 9. 后端接口/数据库 Schema 开发顺序

1. **Project + Idea + Concept + Episode**：新建项目目录与 SQLite 记录、项目创意池、选择 Concept 创建分集。
2. **Artifact + Revision + StageApproval**：A3/A4 的正文/JSON 文件写入、草稿/正式版本、UI 审查。
3. **Scene + Segment + Shot**：场次索引、制作拆解、分镜镜头与来源校验。
4. **Asset + Variant + Media**：项目角色/场景、多视图图片、分集造型引用。
5. **SkillBinding + AgentRun + 查询工具**：CLI/SDK 受控执行、Handoff、小范围检索、草稿提交。
6. **RevisionDependency + ReviewFlag + 批量任务**：上游更新后的影响提示、跨集任务/审批、最终提示词包导出。
7. 最后补齐导入导出、索引优化、系统管理和数据迁移测试；**安全边界与不可覆盖定稿从最早阶段就需要测试，而非第七阶段才考虑**。

---

## 10. 仍需产品层确认的关键边界（不针对普通异常单独 Grill）

- A1/A2 Idea 与 Concept 的**保存历史/选中快照**需达到哪种回溯程度？可以有独立内容版本，但不应强制复用分集阶段的全部审批规则。
- B3 单个镜头是否只作为 ShotPlan JSON 内的结构化对象，还是产品必须支持镜头级独立审批/定稿？现有决策确认了唯一叶级归属与阶段定稿，但未明确每镜头独立审批。
- 项目媒体引用与镜头多视图的具体编辑和绑定体验，是否必须在 V1 就提供图片上传与参考图预览？已有资产复用/变体原则，UI 细节仍需设计。
- 对话与 Agent 执行日志是否要长久保存全文及可否导出，涉及存储体积和用户使用体验。

**下一项建议**：先把核心实体/关系映射为 Python 服务 API 与 Vue/Electron 管理后台的信息架构和操作闭环，而不是继续围绕普通异常场景逐项提问。若用户要先讨论其中一个可能影响 Schema 的产品边界，应优先讨论「正式审核的最小粒度：阶段整体、产物、还是每个镜头」。
