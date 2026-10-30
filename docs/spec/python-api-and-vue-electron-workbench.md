# CinePilot Studio — Python 服务接口与 Vue/Electron 创作工作台设计

> 版本：0.3（补齐 Agent/Model 与 Skill 编排服务边界；完整后台页面见独立规格）
> 日期：2026-10-30
> 状态：**已经确认的产品边界**与**候选 UI/API 细节**分别记录。此文档不代表最终 OpenAPI 或 Vue 组件实现。
> 关联：[整体产品流程](./overall-product-workflow-and-system-map.md) · [核心数据模型](./core-data-model-and-storage-contracts.md) · [项目目录规范](./project-workspace-directory-layout.md) · [Agent/Skill](./agent-skill-orchestration-and-runtime.md) · [项目隔离](./project-isolated-agent-workspaces-and-skills.md) · [阶段合同](./stage-contracts-and-artifact-map.md) · [ADR-0020](../adr/0020-stage-level-approval-with-localized-editing.md)

---

## 1. 目标、已确认规则与本次设计重点

### V1 主目标

用户从 Vue Web 或 Electron 中创建项目，使用同一创作工作台和 Agent 对话，按 A1–A4 / B1–B4 完成创意、剧本、导演、制作拆解、分镜、提示词生成，最后拿到可查看、可编辑、可导出的影视生成准备包。V1 不调用真正的视频生成平台。

**本次优先回答**：每个页面显示什么、用户如何从一个阶段走到下一个阶段、如何与 CLI/Agent SDK 交互、Python 后端需要提供哪些业务接口、如何保证 UI 展示的产物与项目目录真实文件及 SQLite 记录一致。

### 已确认硬边界

1. 技术栈为 Vue 前端、Python 后端、Electron 桌面应用；Web/Electron 应复用业务 UI。
2. 新建项目必须创建独立项目目录；项目级创意池和资产、分集级阶段、阶段产物草稿/定稿分别有明确目录；所有项目共用系统级 SQLite。
3. 正式阶段产物由用户定稿，旧定稿不可覆盖，下一阶段使用确定的上游已定稿版本；上游新定稿标记下游待复核，不自动重做。
4. **阶段是正式定稿与正式 Handoff 单元**；Scene/Segment/Shot 可局部编辑或定向让 Agent 修改，但不单独执行每镜头审批。参见 ADR-0020。
5. A1/A2 属于项目级创意池；用户选中 Concept 并决定创作才创建 Episode，每个 Episode 是一条独立视频。
6. B2 是独立制作拆解阶段，资产管理贯穿 A3–B4；项目基础角色/场景可以在同项目多集复用，每集允许独立造型/服装/状态变体。
7. Agent 运行优先 CLI 生态、可由官方 Agent SDK 控制；每次 Run 只能访问当前项目工作区的受隔离视图，已启用 Skill 从系统注册目录只读投射，**不能访问其他项目**。
8. Markdown 管长篇创作正文，JSON 管结构化创作字段，SQLite 管项目关系、版本、状态、审批；UI 和 Agent 均经 Python 业务服务，不直接操作文件或数据库审批。
9. 首期讨论的页面路由、按钮名、具体 REST URL、事件协议与 Vue 组件库等都是**推荐草案**，并未由用户逐项确认。

---

## 2. 推荐的用户主路径

~~~text
进入系统 / 项目列表
       ↓
新建 Project
  ├─ 输入基本项目资料与目标
  ├─ Python 在系统单 SQLite 登记 Project
  ├─ 创建 projects/{project_id}/ 工作区
  ├─ 选择本项目启用的 Agent 运行时（如已配置）
  └─ 从系统 Skill Registry 勾选此项目可用的 Skill
       ↓
打开项目创意池（A1/A2，不需要 Episode）
  ├─ 用户聊天生成/导入 Idea
  ├─ Idea 卡片 → 发散多个独立 Concept
  ├─ 用户筛选、改写、比较、保留/归档
  └─ 选择 Concept → 「创建分集」
       ↓
创建 Episode
  ├─ SQLite 记录唯一 Primary Idea / 已选 Concept 来源
  ├─ 创建 episodes/{episode_id}/ 工作区
  └─ 进入分集 A3 故事工作台
       ↓
A3 故事 → A4 剧本 → B1 导演 → B2 制作拆解 → B3 分镜 → B4 提示词
  ├─ 进入阶段之前检查必需上游定稿
  ├─ 左侧显示阶段与状态、中间显示结构化产物、右侧 Agent 对话
  ├─ Agent 创建草稿，Python 服务保存到对应项目/分集/阶段
  ├─ 用户可手改，也可选中具体 Scene/Segment/Shot 让 Agent 定点修改
  ├─ 用户审查本阶段必需产物 → 阶段定稿
  └─ 系统固定该阶段的 Approved Revision / 来源引用并开放正式下游
       ↓
B4 定稿并导出：镜头、角色/场景引用、提示词和生产准备文件
       ↓
未来 V2：ComfyUI / RunningHub / Seedance 等生成任务（本期不做）
~~~

**要点**：自由聊天不自动改变阶段状态；Agent 生成“完成”不等于产物已保存；用户的正式定稿是显式操作。项目无需在新建时预生成空分集；只有有选定 Concept 才创建 Episode。

---

## 3. Vue / Electron 信息架构

> **范围说明**：本文件重点描述 Agent 创作工作台的任务闭环、Python 业务模块和 API 候选。Agent 接入/模型配置和 Skill Registry/阶段编排的完整业务规则分别见 [Agent 接入与模型管理](./agent-integration-and-model-management.md) 与 [Skill Registry 与阶段编排](./stage-skill-orchestration-and-registry.md)。此前 3.1 的 App 层级只是早期导航草案，不能视为完整的后台页面设计。完整的项目管理后台、创意池、分集管理、产物库、资产库、审查中心、Agent 任务中心、系统设置，以及它们与创作工作台的关系，统一以 [产品信息架构与页面功能规格](./product-information-architecture-and-page-functions.md) 为准。视觉样式/UIUX 不在上述规格范围内。

### 3.1 推荐应用层级

~~~text
App Shell
├─ Projects / 项目列表
│   ├─ 新建项目
│   └─ 打开项目
├─ Project Workspace / 项目空间
│   ├─ Overview / 项目概览
│   ├─ Creative Pool / 创意池
│   │   ├─ Ideas
│   │   ├─ Concepts / 候选方向比较
│   │   └─ 选中方向 → 创建 Episode
│   ├─ Episodes / 分集列表
│   ├─ Project Assets / 基础资产
│   ├─ Project Skills / 已启用技能
│   ├─ Runs / Agent 执行与待处理任务
│   └─ Activity / 项目内版本/定稿记录
├─ Episode Workspace / 分集工作台
│   ├─ Stage Navigator / A3 → B4
│   ├─ Stage Artifact Editor / 本阶段正式产物
│   ├─ Review and Versions / 审查、草稿/定稿历史
│   ├─ Episode Assets / 本集资产与变体
│   └─ Contextual Agent / 当前项目分集阶段的对话和任务
└─ Settings / 应用级设置
    ├─ Agent Runtime 配置（本地 CLI / 官方 SDK 适配器）
    ├─ 系统 Skill Registry
    ├─ 数据根和项目信息
    └─ 运行环境隔离检测结果
~~~

**V1 优先导航**：项目列表、项目概览、创意池、分集、资产、Skills；任务、版本可用抽屉或页面实现，不必为了菜单数量创建一套复杂管理门户。

### 3.2 分集工作台布局（推荐三栏，适应窗口）

~~~text
┌────────────────────────────────────────────────────────────────────────┐
│ CinePilot Studio / Project P001 / EP03                          [设置] │
│ 当前：B3 分镜   │ 上游：B2 REV01 已定稿  │ 产物：草稿   │ [审查/定稿]   │
├───────────────┬────────────────────────────────┬───────────────────────┤
│ 阶段导航       │ 当前阶段产物工作区             │ Agent 对话 / 任务       │
│               │                                │                       │
│ A3 故事  ✓     │ [镜头卡] [镜头表] [片段树]      │ 当前作用域 P05          │
│ A4 剧本  ✓     │                                │                       │
│ B1 导演  ✓     │ P05 ─ SH01 / SH02 / SH03       │ 「把 P05 拆为 6 镜头」   │
│ B2 拆解  ✓     │ ┌───────────────────────┐      │                       │
│ B3 分镜  ●     │ │ SH03 动作、机位、运镜 │      │ 读取 B2 / P05 ...      │
│ B4 提示词      │ │ 角色 R01 / 变体 VAR03 │      │ 生成草稿 ...           │
│               │ │ 参考图 / 图片视图     │      │ 已保存镜头草稿          │
│               │ └───────────────────────┘      │                       │
│               │                                │ [输入对话/修改要求]     │
├───────────────┴────────────────────────────────┴───────────────────────┤
│ [版本历史] [来源与依赖] [待复核提醒] [阶段审查结果]                   │
└────────────────────────────────────────────────────────────────────────┘
~~~

这是桌面宽屏参考。Web 窄屏或 Electron 小窗口可以将右侧 Agent 和版本审查切为抽屉，保持产物编辑区可用；**不是三栏永远固定**。

### 3.3 页面职责、输入与动作

| 页面 | 主要展示 | 关键操作 | Python 业务来源 |
|---|---|---|---|
| 项目列表 | 项目名称、更新时间、分集数、最近任务 | 新建、打开、归档 | ProjectService + SQLite |
| 新建项目 | 项目资料、可选 Agent Runtime、Skill 选项 | 创建工作区与项目 | ProjectService + WorkspaceService |
| 项目概览 | 项目目标、分集进度、最新产物、活动 | 进入创意/分集 | Project/Episode/Artifact 查询 |
| 创意池 | Idea 卡、对应多个 Concept、方向比较 | 生成、编辑、保留、归档、选方向立项 | CreativeService + AgentTask |
| 分集列表 | 来源 Concept、当前阶段、待审查/待复核 | 打开某集、按阶段筛选 | Episode/StageService |
| A3 故事 | 梗概、结构节拍、故事正文 | 改写、审查、定稿 | Artifact + Revision |
| A4 剧本 | Markdown 正文、Scene 目录/索引 | 定点编辑场次、版本对比 | Artifact + SceneQuery |
| B1 导演 | 导演阐述、视听规范、重点场次 | 编辑局部说明、审查定稿 | Artifact + DirectorRules |
| B2 制作拆解 | 剧本场次与导演片段树、资产需求 | 拆分/排序片段、核对资产 | Breakdown + AssetQuery |
| B3 分镜 | Segment/Subsegment/Shot、镜头卡、参考图 | 逐镜头改动作/运镜、局部 Agent 修改 | ShotQuery + MediaService |
| B4 提示词 | Shot 对应 Prompt、角色/风格约束 | 定点改提示词、复制、导出制作包 | Prompt + PackageExport |
| 项目资产 | 基础角色/地点、图片组与多视图 | 导入媒体、编辑基础资产、查看引用 | Asset + Media |
| 分集资产 | 本集引用版本、服装与造型变体 | 选择基础资产/创建变体 | AssetVariant + EpisodeAssetUse |
| Skills | 系统可用 Skill 与项目已启用清单 | 选择/禁用某项目的 Skill | SkillRegistry + SkillBinding |
| 任务/审查 | AgentRun、阶段审批、待复核、版本比较 | 停止任务、打开草稿、定稿、查看来源 | AgentRun + StageApproval + ReviewFlag |

**产品展示原则**：各阶段不是把 JSON 文件原样放在一个文本框里，而是根据 Schema 采用正文编辑器、列表、树、镜头卡等适合的形式；底层仍由统一服务正确读写 MD/JSON。

---

## 4. Python 业务服务模块（推荐单体模块化，而非微服务）

| 业务模块 | 对外核心职责 | 依赖/限制 |
|---|---|---|
| ProjectService | 新建/读取项目、工作区定位、项目元信息 | SQLite + WorkspaceService |
| WorkspaceService | 创建项目/分集路径、授权沙箱视图、任务草稿目录 | 不向 Agent 暴露其他项目路径 |
| CreativeService | Idea/Concept 创建、读取、修改、选择并立项 | A1/A2 项目级；Episode 不提前创建 |
| EpisodeService | 列表、来源、阶段概览、推进条件 | Episode 的唯一主 Idea |
| StageService | 正式阶段门禁、来源批准版本、审核/定稿、待复核 | 阶段整体审查，不能提前授权 |
| ArtifactService | MD/JSON 草稿保存、版本快照、来源定位、局部编辑 | 单字段唯一权威；定稿不可覆盖 |
| StoryStructureService | Scene/Segment/Subsegment/Shot 精确查询与结构校验 | ID + Revision ID；JSON 结构权威 |
| AssetService | 项目基础资产、分集变体、使用清单 | 禁止直接改写其他分集或跨项目引用 |
| MediaService | 文件上传、缩略图、视图用途、按媒体 ID 读取 | 服务端验证项目归属 |
| AgentRegistryService / AgentConnectionService | Runtime 注册、安装/认证检测、Connection Profile | 不保存明文凭据 |
| AgentCapabilityService / ModelDiscovery | 动态发现 Runtime 能力、模型目录和参数 Schema | 不硬编码跨 Agent 模型列表 |
| AgentModelProfileService / ExecutionProfileResolver | Agent+Model Profile 与系统/项目/阶段/任务配置解析 | 不混入 Skill 配置 |
| AgentTaskService / AgentRunSnapshotService | 创建任务、Handoff、Run 状态和不可变执行快照 | 绑定 Run 的 project_id/episode/stage |
| AgentRuntimeAdapter | 控制一个 CLI 或官方 Agent SDK、转事件与工具调用 | 不直接写数据库/修改定稿 |
| SkillRegistryService / StageSkillProfileService | 系统 Skill 来源/版本、项目 A1–B4 Profile、分集覆盖 | 不把系统登记等同于项目启用 |
| SkillRouter / SkillPlanService | 按任务、阶段、题材和兼容性生成 Skill Plan | 不绕过阶段门禁或 License/安全检查 |
| SkillProjectionAdapter | 将当前 Run 所需 Skill 只读投射到项目隔离工作区 | 不暴露全局 Skill 源目录 |
| ExportService | 影视生成准备包导出，项目级备份/迁移 | 项目导出要附 SQLite 元数据 |
| EventGateway | 前端订阅 Run 和产物变化事件 | SSE / WebSocket 方案后续落技术规范 |

Python 服务**可以是一个进程和模块集合**；不需要一开始部署十几个后台服务或队列系统。长任务的持久化 Run 和进度事件作为受控业务能力即可。

---

## 5. 主要 API 草案（逻辑路由，不是最终 OpenAPI）

全部路径建议以前缀 /api/v1 开始，HTTP 状态码、具体请求 JSON、分页/认证签名由开发规范细化。**一个统一服务入口**同时供 Vue 浏览器、Electron 和 Agent 的受控工具网关使用；Agent 不应直接调用高权限后台管理 API。

### 5.1 Project / Creative Pool / Episode

| 方法 | 路由草案 | 功能 |
|---|---|---|
| GET | /projects | 项目列表、简要进度 |
| POST | /projects | 创建项目、工作区、SQLite 元信息 |
| GET | /projects/{project_id} | 项目详情 |
| GET | /projects/{project_id}/ideas | 查询项目点子 |
| POST | /projects/{project_id}/ideas | 保存 Idea |
| GET | /projects/{project_id}/concepts?idea_id= | 某 Idea 下的多方向 |
| POST | /projects/{project_id}/concepts | 保存 Concept |
| POST | /projects/{project_id}/episodes | 选定 Concept 立项成为新分集 |
| GET | /projects/{project_id}/episodes | 分集列表 |
| GET | /projects/{project_id}/episodes/{episode_id} | 分集概览和来源 |
| GET | /projects/{project_id}/episodes/{episode_id}/stages | A3–B4 进度及可进入条件 |

### 5.2 Artifact / Revision / Stage Review

| 方法 | 路由草案 | 功能 |
|---|---|---|
| GET | /projects/{project_id}/episodes/{episode_id}/stages/{stage_code}/artifacts | 该阶段的产物、草稿与定稿清单 |
| GET | /projects/{project_id}/artifacts/{artifact_id}/revisions/{revision_id} | 读取指定内容版本、文件片段或结构化对象 |
| POST | /projects/{project_id}/artifacts/{artifact_id}/drafts | 从指定基础版本创建草稿 |
| PATCH | /projects/{project_id}/artifacts/{artifact_id}/drafts/{draft_id} | 编辑草稿字段或局部对象；不覆盖定稿 |
| GET | /projects/{project_id}/episodes/{episode_id}/stages/{stage_code}/review | 检查该阶段必需产物、来源版本、是否可定稿 |
| POST | /projects/{project_id}/episodes/{episode_id}/stages/{stage_code}/approve | **用户显式触发整阶段定稿**，固定产物 Revision 集合 |
| GET | /projects/{project_id}/artifacts/{artifact_id}/history | 查看产物修订、差异与来源 |
| GET | /projects/{project_id}/episodes/{episode_id}/review-flags | 查看待复核及影响来源 |

**局部编辑模式**：编辑器可提交操作内容如“把 SH03 的 action 改为……”；但真正的更新仍落到当前阶段 Artifact Draft。不能让 PATCH 直接修改已定稿 Revision；若没有草稿，先基于历史定稿创建新草稿。

### 5.3 结构化内容、资产与媒体

| 方法 | 路由草案 | 功能 |
|---|---|---|
| GET | /projects/{project_id}/episodes/{episode_id}/scenes/{scene_id}?revision= | 获取准确剧本版本的单场次内容 |
| GET | /projects/{project_id}/episodes/{episode_id}/segments/{segment_id}?revision= | 获取片段与来源 |
| GET | /projects/{project_id}/episodes/{episode_id}/shots/{shot_id}?revision= | 获取某镜头，按指定版本 |
| GET | /projects/{project_id}/assets | 项目角色/地点/服装/道具等 |
| POST | /projects/{project_id}/assets | 保存新资产草稿 |
| GET | /projects/{project_id}/episodes/{episode_id}/asset-uses | 本集引用的基础资产和变体 |
| POST | /projects/{project_id}/episodes/{episode_id}/asset-variants | 新建本集造型/服装/状态变体 |
| POST | /projects/{project_id}/media | 上传并登记图片媒体、角色视图 |
| GET | /projects/{project_id}/media/{media_id} | 授权媒体下载/预览 |
| GET | /projects/{project_id}/skills | 本项目已启用 Skill |
| PUT | /projects/{project_id}/skills | 修改本项目 Skill 选择 |

**按版本定位**：Scene/Segment/Shot 的 ID 必须与指定 Artifact Revision 配对；读取“当前最新”仅用于方便 UI 选项，正式 Agent Handoff 永远使用确定的版本锚点。

### 5.4 AgentRun / Conversation / Output

| 方法 | 路由草案 | 功能 |
|---|---|---|
| POST | /projects/{project_id}/agent-runs | 在当前项目/可选 Episode/阶段创建任务 |
| GET | /projects/{project_id}/agent-runs/{run_id} | 任务状态、输入版本、使用 Skill、目标产物 |
| POST | /projects/{project_id}/agent-runs/{run_id}/cancel | 用户取消运行 |
| GET | /projects/{project_id}/agent-runs/{run_id}/events | 流式任务事件（SSE 或其他协议待定） |
| GET | /projects/{project_id}/conversations | 查询项目/分集/阶段会话 |
| GET | /projects/{project_id}/conversations/{id}/messages | 会话记录 |
| POST | /projects/{project_id}/conversations/{id}/messages | 用户对话或指定内容修改要求 |
| POST | /agent-tools/submit-draft | **Agent 内部受限工具**：提交当前 Run 的草稿；由服务器从 Run 确定项目作用域 |

内部 Agent 工具建议不暴露可自由指定 project_id 的跨项目 API；系统从不可变 Run 的授权上下文自动绑定，并验证产物和媒体均属于当前项目。

---

### 5.5 Agent / Model / Skill 配置

| 方法 | 路由草案 | 功能 |
|---|---|---|
| GET | /system/agent-runtimes | 系统 Runtime Registry |
| POST | /system/agent-connections/detect | 检测本机 Agent |
| POST | /system/agent-connections/{id}/test | 连接、能力和隔离测试 |
| GET | /system/agent-connections/{id}/capabilities | Adapter 能力声明 |
| POST | /system/agent-connections/{id}/discover-models | 动态发现模型 |
| GET/POST/PATCH | /system/agent-model-profiles | Agent + Model Profile |
| GET/PUT | /projects/{project_id}/stage-agent-routes | 项目 A1–B4 阶段 Agent/Model 路由 |
| GET | /system/skills | 系统 Skill Registry |
| GET/PUT | /projects/{project_id}/stage-skill-profiles | 项目八阶段 Skill 配置 |
| GET/PUT | /projects/{project_id}/episodes/{episode_id}/stage-skill-overrides | 分集阶段 Skill 覆盖 |
| POST | /projects/{project_id}/agent-runs/preview | 预览 Agent/Model/Skill Plan |
| GET | /projects/{project_id}/agent-runs/{run_id}/execution-snapshot | 查看实际执行配置快照 |

模型目录与参数由 Agent Adapter 动态提供；Skill Router 生成的 Skill Plan 与 Agent Model Profile 分开解析，Run 启动后一起写入快照。

## 6. 一次任务的实时状态与 UI 反馈（推荐）

推荐将 Agent 输出区分为 **对话流**、**任务事件流**、**正式产物变更**，不把模型自然语言完成提示等同于保存成功。

### 6.1 任务事件类型

~~~json
{
  "run_id": "RUN123",
  "project_id": "P001",
  "episode_id": "EP03",
  "stage": "B3",
  "event_type": "draft_saved",
  "sequence": 12,
  "payload": {
    "artifact_id": "SHT03",
    "draft_id": "DRAFT05",
    "summary": "已提交 6 个镜头草稿"
  }
}
~~~

建议事件：
- run_created：Run 已登记及输入版本固定。
- run_started：在当前项目沙箱启动 Agent。
- skill_selected：已选主 Skill（可选附加能力）。
- context_read：从允许的场次/片段/角色读取资料（仅公开 UI 可安全展示的摘要）。
- agent_message_delta：聊天正文增量。
- draft_submitted：收到候选草稿，但不宣称已保存。
- draft_saved：经过 Python 校验并可由 UI 持久化读取。
- review_needed：产物可进入用户审阅。
- run_completed / run_failed / run_cancelled：执行最终状态。

**任务事件不应该包括**：项目文件绝对路径、密钥、其他项目对象内容、未经授权的原始模型工具参数。UI 显示“已保存”的前提是服务真正返回持久化草稿引用。

### 6.2 推荐视觉反馈

- 右栏按阶段显示 Agent 会话和任务步骤；可查看是否正在读取 P05 或 R01 参考图。
- 中栏在收到 draft_saved 后局部更新对应镜头/场次卡片，**不要每一字增量都改写正式 JSON 文件**。
- 用户离开页面、再次进入后，通过 SQLite Run/Artifact 状态恢复任务和已保存产物，不依赖旧页面内存。
- 用户选择“仅改 SH03”，系统明确限定任务作用域，完成后高亮变更镜头及变更字段供用户审阅。
- 运行中不意味着获得阶段定稿权限，审批按钮必须由 StageService 检查必要产物和版本。

---

## 7. 重点交互一：新建项目 + Skill 启用

1. 用户打开项目列表，点击「新建项目」。
2. 输入项目名称和创作方向（可选）：例如“拟人化狗狗职场短片”。
3. 可选选择一个已配置的 Agent 执行源；无执行源时可创建项目，但启动 Agent 任务时应明确提示未配置，而不是无声失败。
4. 系统显示来自 Skill Registry 的可用专业 Skill，用户为项目勾选；建议有覆盖 A1–B4 的默认配置，避免初学用户自己理解几十个 Skill。
5. 点击创建：Python 生成稳定 Project ID、登记 SQLite、创建 projects/{project_id} 下必要目录和项目 Skill 启用清单。
6. 以后每次启动 Run，Python 按项目 Skill 启用清单准备只读 Skill 投射，并将 Agent cwd 设置为**当前项目隔离视图**；不能直接让 Agent 看到兄弟项目文件。
7. 跳转项目创意池；不自动创建 Episode，不制造空白剧本。

项目设置页应能查看运行时与 Skill 启用状态，允许后续更换。系统级 Skill 安装和项目级 Skill 启用是两种权限不同的操作。

---

## 8. 重点交互二：创意池 → 创建 Episode → A3

1. 项目创意池无分集时也可以创建和编辑 Idea。
2. 选中 Idea，右侧 Agent 对话发起“给这个点子 4 个差异明显的故事方向”；Agent 由项目工作区运行，受控提交 Concept 草稿。
3. 中栏以 Concept 卡展示方向的核心钩子、冲突、人物目标及结局建议。用户可修改、比较、保留、归档。
4. 用户明确点击“选择此方向并创建分集”，系统读取该 Concept 的当前被选内容，创建 Episode，记录唯一 Primary Idea 和选中方向来源。
5. 新分集展示 A3 故事工作台；正式 A3 产物可引用选中 Concept 的准确保存版本/内容快照。A1/A2 本身不被套入分集的 StageApproval。
6. 尚未创建 Episode 时，不显示 B1–B4 的空审批任务。

**未额外决定**：同一 Concept 能否创建多集；V1 暂不为此强加唯一索引，待产品行为明确后细化。

---

## 9. 重点交互三：分集 B3 局部镜头生成 → 审查定稿

场景：EP03 的 A4 / B1 / B2 已定稿，用户在 B3 选择片段 P05。

1. UI 显示 B2 来源 Revision ID、P05 的来源剧本场次与相关角色/场景资产。
2. 用户在右侧输入：“将 P05 拆为 6 个镜头，保持角色 R01 的脸部与衣着一致”。
3. StageService 先检查 B2 正式准入并锁定所需上游 Revision，AgentTaskService 创建 RUN123。
4. 项目隔离 Runner 读取已启用的分镜 Skill、受控查询 P05 / 来源 Scene / R01 基础资产与当前分集变体。
5. Agent 输出镜头结构；ArtifactService 校验 ShotID、顺序、叶级所属片段、来源关系和资产引用，再写入 B3 对应 Artifact Draft。
6. 中栏出现六个镜头卡；用户点开 SH03，手动修改动作或在右侧说“只改 SH03 的第二个动作，不调整其他镜头”。
7. 修改只进入 B3 现有草稿/新草稿；若已有 B3 旧定稿，先基于旧定稿建立新草稿，原定稿保持不动。
8. 阶段审查显示“本次 B3 草稿有哪些镜头变化、来源是否有效、资产引用是否缺失”，用户明确点击“定稿 B3”。
9. Python 写入不可变 B3 Revision，SQLite 发布这次 StageApproval/来源依赖；B4 此时才可以作为正式下游阶段使用该 B3 Revision。
10. 未来 B2 改版不触发 B3 自动重做，旧 B3 保留并在 UI 显示待复核标记。

**确认的审批粒度**：一次审查针对**B3 阶段所采用的整套镜头产物版本**；不是要求用户逐个批准 6 个镜头。局部编辑仍然充分支持。

---

## 10. 重点交互四：项目资产、图片与分集变体

- 在项目资产页，用户建立 R01 的角色基础设定，上传面部特写、正面、侧面、背面等媒体并标记用途。
- 在 EP03 分集资产页选择 R01 的基础版本，建立本集办公室造型变体 VAR03（西装/领带）。
- B2 显示 P05 需要 R01 角色和某地点；B3 镜头卡显示使用 R01/VAR03 及选定角色视图；B4 Prompt 读取相同的引用锚点。
- Vue 显示资产卡和图片预览；Agent 查询工具取得的是当前项目授权的资产版本和媒体引用。**只有 Runtime/模型支持图像输入并正确传入像素，Agent 才真正“看到图片”**；路径不等于视觉输入。
- 用户可以从镜头卡跳转查看所用资产，却不应在修改镜头时悄悄改变项目级 R01 的身份或默认基础外观。

建议 UI 默认继承分集所选变体，镜头可以显式展示/选择特殊造型状态（实际覆盖优先级后续细化）；所有正式来源可回溯。

---

## 11. 审批、修订与影响提示的统一体验

### 11.1 一个阶段内多个产物

用户审阅的是一个阶段的**必需产物集合**（StageApproval）。例如 A4 可以含 screenplay Markdown + scenes JSON；B3 是镜头计划和必要媒体引用。只有所有必需产物通过校验和用户确认后，StageService 才发布本阶段的定稿锚点。

### 11.2 版本对比

- 阶段页顶部显示当前草稿基于哪个已批准 Revision。
- 用户可以切换“查看旧定稿 / 查看当前草稿 / 变化比较”。
- 故事/剧本正文可进行文本差异，镜头和片段可按稳定 ID 对比新增/删除/修改；实际 diff 算法是后续开发细节。
- 用户审批的是新内容快照，不是自动修改旧 Revision；上游依赖的版本必须可查看。

### 11.3 待复核提醒

上游修订定稿后，系统依据已保存版本依赖标记相关下游待复核。UI 展示触发原因和旧来源版本，并提供查看差异、沿用旧成果或新建修订的操作入口；具体“沿用”后的记录结构留待业务状态设计，不默认自动重做。

---

## 12. V1 API 与 UI 验收场景

| 测试场景 | 验收结果 |
|---|---|
| 新建项目 P001 | SQLite 有 Project 身份，项目工作区创建，启用 Skill 状态可回查 |
| P001 启用技能并启动 Agent | Agent 只在 P001 隔离环境运行，能读已启用 Skill，**无法读取 P002** |
| 项目创意池生成 Idea / Concept | 不需要空 Episode；方向在后台可编辑、可比较与回查 |
| 选择 Concept 创建 EP03 | EP03 对应唯一 Primary Idea，具备分集阶段工作台 |
| 生成 A4 剧本草稿 | screenplay.md / scenes.json 有实际持久化引用，界面能按 Scene ID 查正文 |
| A4 审查定稿 | StageApproval 固定具体产物修订，旧定稿不覆盖，B1 正式准入开放 |
| B2 拆解剧本 S02 | Segment 来源场次明确；结构层数合法，镜头归属约束可校验 |
| B3 局部修改 SH03 | 仅目标内容进入当前 B3 草稿，其他镜头与旧 B3 定稿保持不变 |
| B3 整阶段定稿 | 用户只需审查整套镜头产物一次，无需 SH01..SH06 分别审批 |
| 资产 R01 + EP03 变体 | B3/B4 可查询到匹配的基础形象与本集造型；媒体不跨项目 |
| Agent 自然语言说“已完成”但无正式草稿 | UI 不显示成“已保存产物” |
| 上游重新定稿 | 旧下游仍可见，关联下游显示待复核，不自动重新生成 |
| B4 导出 | 可从已保存镜头、资产引用和提示词组成完整准备包，不执行视频生成 |
| Electron 重新打开与 Web 重新进入 | 从 Python/SQLite/文件持久化恢复相同项目、阶段、产物与审批状态；**执行宿主差异仍待技术选型** |

---

## 13. 与 V1 开发计划的衔接

建议从一条**真正可运行的主路径**开始开发，而不是把所有页面先做成只有展示的空壳：

1. **项目外壳**：Project + Workspace + 系统 SQLite + Skill 启用列表。
2. **创意池**：Idea / Concept 编辑保存，选择 Concept 创建 Episode。
3. **阶段产物闭环**：Artifact 草稿 → UI 展示/编辑 → StageApproval → 旧 Revision 可查。
4. **一个真实 Agent 闭环**：选定单一 CLI/Agent SDK，以项目隔离沙箱读取一个授权内容并提交草稿到 Vue。
5. **完整 A3–B4**：复用阶段合同，逐步实现剧本、导演、拆解、镜头、提示词专用编辑器。
6. **资产与分集变体**：参考图、多视图、媒体引用、风格一致性。
7. **版本依赖 / 批量 / 导出**：成熟后补齐效率能力；前述定稿/隔离/文件真实性从第一条闭环就需要落实。

**可保留的重大待确认项**：
- Electron 本地 Python 与 CLI/Agent SDK 是“桌面一起启动”还是“桌面连接已有远端服务”，以及 Web 服务如何调用同一 Agent 执行宿主；
- V1 首个实测 Agent Runtime（Codex 或 Claude）及图片传递/沙箱能力；
- 分镜图片在 V1 的最低范围：文本镜头卡 + 上传已有图片，还是必须内建分镜图片生成（当前 V1 不包含实际图片/视频生成 API）。

这些会影响具体发布形态与 V1 价值，可以在技术架构与验收阶段集中讨论。**不为普通异常、重试策略、细节 API 命名逐项 Grill**。

## 14. 开发衔接

本文件的 API 仍是逻辑路由候选，实施时按统一业务服务和阶段合同细化 OpenAPI/Schema。Agent/Model 和 Skill 配置合同以对应专项规格为准。

**可以开始 V1 开发**：建议先执行 [开发启动与实施路线图](./development-kickoff-and-implementation-roadmap.md) 的 M0 技术 Spike，再实现 Project/Workspace、Artifact/Revision/StageApproval、Agent Adapter、Skill Registry 和第一条 A1/A2 创作垂直切片。视觉样式和高保真 UIUX 不构成开工前置。
