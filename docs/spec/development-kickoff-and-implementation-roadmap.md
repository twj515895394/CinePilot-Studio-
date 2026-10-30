# CinePilot Studio — 开发启动与实施路线图

> 版本：0.1
> 日期：2026-10-30
> 状态：**可作为 Codex 开发启动基线。** Vue + Python + Electron、系统单 SQLite、项目文件独立、项目 Agent 工作区隔离、阶段整体定稿和 V1 不接视频生成均为已确认约束。里程碑顺序、模块拆分和建议代码结构是推荐实施方案；具体框架/库/进程部署在首个技术 Spike 中验证后记录。
> 设计基线：[整体产品总览](./overall-product-workflow-and-system-map.md) · [页面功能规格](./product-information-architecture-and-page-functions.md) · [核心数据模型](./core-data-model-and-storage-contracts.md) · [项目目录规范](./project-workspace-directory-layout.md) · [阶段合同](./stage-contracts-and-artifact-map.md) · [Agent 接入与模型管理](./agent-integration-and-model-management.md) · [Skill Registry 与阶段编排](./stage-skill-orchestration-and-registry.md) · [Agent/Skill 运行时协同](./agent-skill-orchestration-and-runtime.md)

---

## 0. 开工结论

CinePilot Studio 的产品主流程和核心数据约束已经足以开始 V1 工程开发。**不需要等所有 Skill 都补齐，也不需要先把所有页面设计成高保真 UI 才开工。**

Codex 可以从项目骨架、SQLite/Workspace、统一 ArtifactService、Agent Adapter/隔离、Skill Registry/Router 等基础能力开始，再按真实创作闭环逐步扩展页面。

### 0.1 已确认、开发时不得擅自改变的决策

1. 前端 Vue，后端 Python，桌面端 Electron；Web/Electron 复用核心业务服务与 Vue 工作台。
2. 全系统共用一个 SQLite；Markdown 管创作正文，JSON 管结构化创作产物，SQLite 管身份、关系、版本、审批和状态；同一字段只有一个权威来源，统一服务读写。
3. 新建项目即创建独立项目目录与项目 Agent Workspace；Agent 只能访问当前项目授权内容，严禁访问其他项目、全局 SQLite、全局凭据和未授权系统 Skill 源。
4. A1 Idea / A2 Concept 属于项目级创意池；用户选择 Concept 并决定进入视频开发时才创建 Episode。
5. 一个 Episode 对应一条独立短视频，只有一个 Primary Idea。
6. A3 故事、A4 剧本、B1 导演、B2 制作拆解、B3 分镜、B4 提示词为分集正式阶段；同一分集上游定稿后才正式进入下游。
7. A4 Screenplay Scene 与 B2 Director Segment 独立建模；一个 Scene 可映射多个 Segment，每个 Segment 只有一个来源 Scene。
8. Segment 可直接包含 Shot，或通过最多一层 Subsegment 组织 Shot；同一 Segment 不混用直属 Shot 和 Subsegment。
9. 项目基础资产可被同项目多集复用；Episode 可有独立变体，不直接改写基础资产。
10. 正式审批以阶段整体为单位；Scene/Segment/Shot 可局部编辑，但不设置逐镜头独立审批。
11. 已定稿 Revision 不可覆盖；修改生成新草稿，用户再次确认后产生新定稿。
12. 上游新定稿只标记依赖下游待复核，保留旧结果，不自动重做。
13. Handoff 精简、版本明确；Agent 按需读取授权资料，不默认加载全部聊天或文档。
14. V1 输出到 B4 视频提示词/制作准备包，不接实际图片或视频生成 API。

### 0.2 推荐的开发原则

- 先打通数据与业务闭环，再做完整 UI 页面；不要先堆空壳页面。
- 先做模块化单体 Python 服务，不提前拆微服务。
- 先支持一个真实 Agent Runtime；Adapter 边界稳定后再加第二个。
- 模型目录和参数从 Agent Adapter 动态发现；不硬编码统一模型清单。
- Skill Registry 和阶段 Profile 先建立机制；缺失阶段允许空 Profile，不阻塞系统开发。
- 外部 Skill 先做来源/License/安全/质量评估，不一次性复制全部仓库。
- 安全、项目隔离、不可覆盖定稿、文件/数据库一致性从第一条垂直切片开始测试，不留到后期补救。

---

## 1. 推荐代码库初始结构

具体框架和库在技术 Spike 中选择；以下是模块边界建议，不是用户已批准的最终目录：

~~~text
CinePilot-Studio/
├── apps/
│   ├── web/                 # Vue Web 管理后台 + 创作工作台
│   └── desktop/             # Electron 宿主，复用 Web UI
├── services/
│   └── api/                 # Python 模块化单体业务服务
│       ├── app/
│       ├── domain/
│       ├── services/
│       ├── repositories/
│       ├── adapters/
│       │   ├── agents/
│       │   └── skills/
│       ├── security/
│       └── tests/
├── contracts/
│   ├── api/
│   ├── artifacts/
│   └── events/
├── docs/
│   ├── adr/
│   ├── discovery/
│   ├── process/
│   └── spec/
├── scripts/
└── tests/
    ├── integration/
    ├── e2e/
    ├── fixtures/
    └── skill-evaluations/
~~~

初期不要求把 contracts 做成独立包；关键是避免前端、Python、Agent 三边各自维护互相冲突的字段定义。

### 1.1 技术选型待验证，不是产品 Grill 阻塞

Vue、Python、Electron 已确认；以下由 Codex 在 M0 Spike 给出推荐并记录：

- Vue 版本、Vite、TypeScript、路由、状态管理；
- Python Web 框架、数据访问方式、迁移工具、Schema 校验；
- Electron 主进程/渲染进程通信和安全设置；
- Python 服务随 Electron 启动、健康检查、退出和打包；
- Web 部署形态与本地 Agent Runtime 的关系；
- Windows/macOS 项目隔离技术可行性。

优先选择成熟、简单、可测试的方案。不要为未来多租户、微服务或复杂分布式任务系统过度设计。

---

## 2. 里程碑总览

| 里程碑 | 目标 | 主要交付 | 完成后可做什么 |
|---|---|---|---|
| M0 工程骨架与技术 Spike | 确认跨平台开发和安全基础 | Monorepo、Vue/Python/Electron 最小启动、SQLite 迁移、隔离 Spike | 能运行空应用并验证部署路径 |
| M1 Project / Workspace / Storage | 项目创建与数据隔离 | 单 SQLite、项目目录、WorkspaceService、项目管理基础页 | 创建/打开/归档项目 |
| M2 Creative Pool | Idea/Concept 生命周期 | Idea/Concept 文件、关系、历史版本、创建 Episode | 管理创意并从 Concept 立项 |
| M3 Artifact / Revision / Approval | 正式产物和定稿机制 | ArtifactService、MD/JSON 权威、Draft/Revision/StageApproval | 脱离聊天查看、编辑、审查、定稿 |
| M4 Agent Runtime / Model | 一个真实 Agent 端到端执行 | Adapter、Connection、动态模型发现、Run Snapshot、隔离执行 | Agent 能在当前项目生成并提交草稿 |
| M5 Skill Registry / Stage Profiles | 专业技能注册和路由 | Skill Source/Version、Profile、Router、Skill Plan、只读投射 | 每阶段可配置并执行 Skill 组合 |
| M6 A1/A2 创作闭环 | 先交付创意编剧系统前半段 | Idea Discovery、Concept Development 工作台 | Agent 生成、比较、修改和定稿创意方向 |
| M7 A3/A4 正式创作 | 故事与文学剧本 | Story/Screenplay 编辑器、Scene 结构和阶段审批 | 一集故事和剧本可定稿 |
| M8 B1/B2/B3/B4 制作规划 | 导演到提示词 | Director、Breakdown、Shot、Prompt 专用视图 | 完整影视制作准备包 |
| M9 资产、审查、批量和导出完善 | 提升长期使用和批量生产能力 | Asset/Variant/Media、Reviews、Run Center、Export | 多集生产、回溯和交付 |

里程碑可并行开发 UI 壳、领域服务和测试，但不可跳过底层安全/版本合同。

---

## 3. 各里程碑实施要求

### M0 — 工程骨架与技术 Spike

目标：验证项目能在目标 Windows 开发机运行 Vue、Python、Electron，并为后续 Agent 沙箱选定真实可行路径。

交付：
- 前端、后端、桌面端最小启动；
- Python 健康检查 API；
- SQLite 初始化和第一条迁移；
- Electron 能启动/停止 Python 服务，处理异常退出；
- 项目路径定位原型；
- Agent 子进程文件系统隔离 Spike：尝试读取兄弟项目、系统 SQLite、系统凭据时必须失败；
- 一个 Agent CLI/SDK 最小无副作用任务原型，评估认证方式和模型选择能力。

验收：
- Web 和 Electron 都能连接同一类 Python 业务接口；
- 服务重启后 SQLite 持久化；
- 明确 Web 部署能否访问本机 Agent；不能时给出清楚部署边界；
- 隔离测试有自动化证据，不接受“cwd 已切到项目目录”作为通过。

### M1 — Project / Workspace / Storage

目标：创建项目时就创建独立项目工作区，系统级 SQLite 统一管理。

核心模块：
- ProjectService；
- WorkspaceService；
- Database/Migration；
- Project Repository；
- Path Resolver / Project Boundary Guard；
- Project List / Project Overview 基础页面。

必须做到：
- 系统只有一个 SQLite 数据库；
- 项目内容写入各自项目目录；
- 业务服务只接受 project_id，不接受 Agent 自报绝对路径；
- 文件真实路径解析后校验仍在项目根内；
- 项目 A 的服务请求不能读取项目 B；
- Agent 不能直接连接 SQLite。

### M2 — Creative Pool

目标：实现项目级 Idea / Concept，选中 Concept 后创建 Episode。

核心模块：
- IdeaService、ConceptService、EpisodeService；
- Idea/Concept JSON 内容与版本定位；
- 项目创意池列表、详情、比较和编辑；
- Concept 选择与 Episode 创建事务。

验收：
- 新项目可以没有 Episode；
- Idea 可派生多个 Concept；
- Concept 可独立修改和归档；
- 用户明确选定 Concept 后才创建 Episode；
- Episode 保存唯一 Primary Idea 和选中 Concept 的来源版本；
- 不建立跨项目自动共享关系。

### M3 — Artifact / Revision / Stage Approval

这是系统长期可靠性的核心里程碑，优先于复杂 Agent UI。

核心模块：
- ArtifactService；
- RevisionService；
- StageService / StageApprovalService；
- Markdown/JSON 内容解析与 Schema 校验；
- File-first immutable publication；
- Revision Dependency / ReviewFlag；
- Artifact Library 和通用内容查看/编辑器。

验收：
- 新草稿基于指定旧 Revision；
- 先写入不可变内容文件，再由 SQLite 发布版本引用；
- 已定稿文件不被覆盖；
- 同一阶段多份必需产物以一个 StageApproval 整体定稿；
- 局部 Scene/Segment/Shot 编辑进入阶段草稿；
- 上游变化只标记下游待复核，不自动重做；
- 管理后台不打开聊天也能阅读和编辑已保存产物。

### M4 — Agent Runtime / Model / Security

核心模块：
- AgentRegistryService；
- AgentConnectionService；
- AgentAdapter Protocol；
- AgentCapabilityService；
- ModelDiscovery；
- AgentModelProfileService；
- ExecutionProfileResolver；
- AgentRun / Event；
- CredentialBroker；
- Project Sandbox / Tool Gateway。

V1 限定：
- 先完成一个经隔离验证的本机 Agent Runtime；
- 优先 CLI，官方 Agent SDK 可用时优先评估；
- 模型和参数由 Adapter 动态发现；
- 运行时不支持显式模型选择时使用 Runtime Default；
- 不接直接模型 API 作为前置；
- 每个 Run 固定 Agent/Model/参数和输入版本快照。

验收：
- Agent 在项目隔离环境运行；
- 无法读取其他项目、全局 SQLite 和系统凭据；
- 能通过受控工具读取当前项目指定对象；
- 能提交草稿到 ArtifactService；
- 运行状态和事件可从 UI 恢复；
- Agent 的“完成”文本不能伪装成保存成功。

### M5 — Skill Registry / Stage Skill Profiles

核心模块：
- SkillRegistryService；
- SkillSource / License / Provenance；
- SkillVersion；
- SkillProjectionAdapter；
- StageSkillProfileResolver；
- SkillRouter；
- SkillPlan；
- Skill Compatibility Check；
- Skill evaluation harness。

V1 限定：
- 系统登记八个候选来源；
- 只有 License、安全、质量评估通过的版本才能启用；
- 每个项目有 A1–B4 八个 Stage Skill Profile，允许部分阶段为空；
- 支持项目、分集、任务三级覆盖；
- Skill Plan 启动前可预览，Run 后固定快照；
- 当前任务只加载必要 Skill 和 references；
- 不自动复制八个仓库，不自动运行所有外部脚本。

验收：
- 同一个 Skill 可以在多个阶段引用；
- 一个阶段可以组合多个 Skill；
- 题材插件按条件加载；
- 不兼容 Runtime 或未审核 Skill 不进入 Run；
- Skill 更新不改变历史 Run 或已定稿内容。

### M6 — A1/A2 创意编剧闭环

建议先把用户最常用的创意工作流做完整：

- A1 Idea 列表、详情、生成和修改；
- A2 Concept 多方向生成、比较、深化和选择；
- Agent 工作台与产物视图联动；
- Skill Plan 展示；
- 创意历史和版本；
- 选择 Concept 创建 Episode。

第一批 Skill 候选按评估结果逐步导入。不要为了演示而将未经许可/评估的 Skill 直接复制到生产 Registry。

### M7 — A3/A4 故事与剧本

A3：
- 故事梗概、人物目标/关系、冲突升级、节拍、转折、结局；
- 正文与结构化节拍视图；
- 选定节拍局部修改；
- 阶段整体审查和定稿。

A4：
- 文学剧本 Markdown；
- Scene JSON 索引；
- 场次列表和详情；
- 角色/地点/道具引用；
- 单场次定点修改；
- Scene 与正文锚点一致性校验；
- 阶段整体定稿。

### M8 — B1/B2/B3/B4 影视制作规划

B1 导演：
- 导演阐述、视听基调、表演方向、摄影原则和声音方向。

B2 制作拆解：
- Scene → Segment 一对多映射；
- Segment 最多一层 Subsegment；
- 每个 Segment 只有一个来源 Scene；
- 片段树、场次映射、资产需求核对。

B3 分镜：
- 叶级单元下 Shot 列表/卡片/序列；
- 动作、景别、机位、运镜、时长、构图、连续性；
- 局部 Shot 编辑；
- 用户上传图片/参考图预览和引用。

B4 提示词：
- 每个 Shot 对应 Prompt；
- 角色/场景/服装/道具引用；
- 连续性约束；
- 全集提示词包导出。

B1/B2/B4 的外部 Skill 来源等待用户后续补充；不阻止先搭建空 Profile、编辑器和阶段合同。

### M9 — 资产、审查、批量与导出完善

- 项目基础资产、Asset Revision；
- Episode Variant 和 Episode Asset Use；
- Media 上传、用途标签、多视图；
- 全局 Reviews；
- Agent Run Center；
- 版本差异与 ReviewFlag；
- 多集批量任务（每集独立 Run/Handoff/草稿/审批）；
- 项目导入导出（项目文件 + 对应 SQLite 元数据）；
- B4 制作准备包导出。

---

## 4. 最小垂直切片：第一条必须真实跑通的用户路径

在铺开所有页面前，至少跑通以下一条端到端路径：

1. 用户在 Vue/Electron 新建项目 P001；
2. 系统创建共享 SQLite Project 记录和独立项目目录；
3. 用户在项目创意池创建 Idea；
4. 用户为项目启用一个已审核 Skill Version，并设置 A1 Stage Skill Profile；
5. 用户在 Agent 工作台发起 A1 点子任务；
6. Python 解析当前 Agent Model Profile 和 Skill Plan；
7. Agent 在 P001 隔离环境读取允许的 Idea/约束，不可访问 P002；
8. Agent 提交候选 Idea/Concept 内容；
9. ArtifactService 校验并保存为草稿/创意版本；
10. UI 能独立查看和编辑该版本，不依赖聊天历史；
11. 用户继续生成多个 Concept 并比较；
12. 用户选择 Concept 并创建 Episode；
13. 后续 A3 可读取选定 Concept 的准确来源版本。

这条路径证明系统不只是聊天壳，而是真正的项目化创作系统。

---

## 5. 测试与质量门禁

### 5.1 必须从第一阶段建立的测试

- SQLite migration / repository tests；
- 项目目录归属和路径穿越测试；
- Agent Sandbox cross-project denial tests；
- Agent 不可读取 SQLite/凭据的测试；
- File-first publication 与不可覆盖 Revision 测试；
- StageApproval 必需产物完整性测试；
- Handoff 输入 Revision 固定测试；
- Skill Plan 配置解析优先级测试；
- Adapter 动态模型和参数校验测试；
- Artifact Schema 与跨对象关系校验；
- Web/Electron API 契约测试；
- 任务失败、取消、重启后的状态恢复测试。

### 5.2 外部 Skill 回归

每个导入 Skill 建立固定测试集：
- 创作任务样例；
- 固定输入和约束；
- 预期 Artifact Schema；
- 质量评分维度；
- 不得发生的阶段越界；
- 上下文/Token 消耗；
- Runtime + Model 组合。

升级 Skill 或 Model 后对比同一测试集。不能只看一条样例输出。

### 5.3 完成定义（Definition of Done）

一个里程碑只有满足以下条件才算完成：
- 功能可在 Web/Electron 中真实操作；
- 数据持久化且重启可恢复；
- 服务层规则和数据权限有测试；
- UI 状态准确区分流式输出、已保存草稿、已定稿；
- 文档和实现一致；
- 不破坏已确认 ADR；
- 关键安全测试通过；
- 没有用假数据/静态 UI 冒充已完成后端能力。

---

## 6. 明确不作为 V1 开工前置的内容

- B1/B2/B4 缺失 Skill 的补充；
- 八个外部 Skill 全部深度改造完成；
- 所有 Agent Runtime 全部接入；
- 直接模型 API Provider；
- 远端 Agent Host；
- ComfyUI / RunningHub / Seedance / 即梦视频生成接口；
- 自动图像/视频生成；
- 多用户组织权限与团队协作；
- 独立知识库、向量数据库或微服务化；
- 全部页面高保真 UIUX。

这些项目可在相应里程碑逐步补充，不阻塞基础工程开始。

---

## 7. Codex 开工指令与工作约定

Codex 开始实现前应：

1. 阅读 README.md、AGENTS.md、CONTEXT.md、grill-with-docs 规范和本路线图；
2. 阅读本路线图“已确认决策”，不得擅自改变；
3. 阅读相关 Spec/ADR，不以聊天摘要替代仓库文档；
4. 对尚未确定的框架/库/运行方式先做最小 Spike，给出推荐和测试证据；
5. 每个里程碑先实现端到端可验收切片，再扩展；
6. 发现设计文档冲突时先指出并提出修正，不直接选择对自己最方便的版本；
7. 更新对应 Discovery/Spec/ADR，保持实现和文档一致；
8. 不要在尚未通过项目隔离测试前启用真实 Agent 执行任意项目任务。

**建议 Codex 第一项开发任务**：完成 M0 工程骨架与技术 Spike，输出具体框架/目录建议、启动方式、SQLite migration 原型和项目沙箱验证报告；经用户检查后进入 M1。