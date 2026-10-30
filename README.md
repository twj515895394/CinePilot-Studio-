# CinePilot Studio

**Agent-first 的独立短视频创意、剧本与影视制作规划平台（设计讨论阶段）。**

## 当前范围

一个产品内设置两大业务系统：

- **创意编剧**：点子发现、创意发散、故事深化、剧本编写与审查。
- **影视制作**：导演与视听设计、片段/镜头/分镜规划、生成提示词与制作准备包。

创作过程以 Agent + Skill 与用户对话协作为主；系统负责项目、分集、片段、镜头、素材、阶段产物、版本、审批与回溯，并通过 Web 页面浏览和操作。

本期**不实现**视频生成；后续考虑 ComfyUI、RunningHub、即梦/Seedance 等适配。

## 已确认技术栈

- 前端：**Vue**（Web 管理后台与创作工作台）。
- 后端：**Python**（业务服务、Agent/Skill 编排、版本与数据管理）。
- 桌面端：**Electron**（复用 Vue 前端，提供桌面应用）。
- 数据持久化：**Markdown + JSON + SQLite**（按字段唯一权威，统一服务读写）。

Vue 的配套组件库、Python Web 框架、Electron 与 Python 服务的运行/打包方式尚未定稿，不得在未经确认时将推荐方案写成已批准选型。

**Agent 运行与隔离（已确认）**：项目创建即拥有独立工作区，任务以该项目目录作为工作目录；系统集中维护 Skills，按项目选择并只读链接/投射；Agent 禁止访问其他项目目录。V1 以本地 CLI 体系为先，官方 Agent SDK 可用时优先用于程序控制。**cwd 本身不构成隔离**，必须由真实执行环境和业务工具边界保证。

## 设计讨论入口

- [需求基线与架构讨论树（第一、二轮定稿）](docs/discovery/0001-initial-scope-and-design-tree.md)
- [第二、三轮定稿：分集、片段、镜头层级](docs/discovery/0002-episode-and-shot-hierarchy.md)
- [第四轮定稿：剧本场次与导演片段映射](docs/discovery/0003-screenplay-scene-to-segment-mapping.md)
- [第五轮定稿：独立创意方向与分集建立时机](docs/discovery/0004-idea-concept-episode-lifecycle.md)
- [第六轮定稿：项目独立创意池，跨项目仅复制内容](docs/discovery/0005-idea-concept-library-scope.md)
- [第七轮定稿：已定稿产物的版本保留与重新确认](docs/discovery/0006-approved-artifact-revision-policy.md)
- [第八轮定稿：上游版本变化后标记下游待复核](docs/discovery/0007-upstream-change-impact-policy.md)
- [第九轮定稿：分集上游定稿才能正式交接](docs/discovery/0008-stage-gate-and-handoff.md)
- [第十轮已定稿：精简且版本明确的 Agent Handoff](docs/discovery/0009-agent-handoff-payload.md)
- [第十一轮定稿：系统预备必需事实、Agent 受控按需检索](docs/discovery/0010-controlled-on-demand-retrieval.md)
- [第十二轮定稿：SQLite + Markdown/JSON 的权威分工](docs/discovery/0011-storage-authority-and-admin-ui.md)
- [第十三轮定稿：文件先行、SQLite 发布版本引用](docs/discovery/0012-cross-store-publish-consistency.md)
- [整体产品流程与系统设计总览（0.2 草案，已纳入 Vue/Python/Electron）](docs/spec/overall-product-workflow-and-system-map.md)
- [八阶段输入输出、产物及正式交接设计（B2 已确认）](docs/spec/stage-contracts-and-artifact-map.md)
- [创作资产体系：项目基础资产 + 分集变体（方案 B 已确认）](docs/spec/creative-asset-model-and-use.md)
- [Agent / Skill 协作与运行架构（详细草案）](docs/spec/agent-skill-orchestration-and-runtime.md)
- [Agent V1 运行路线：CLI 生态优先、官方 Agent SDK 可用时优先评估（已确认）](docs/discovery/0015-agent-runtime-priority.md)
- [项目隔离 Agent 工作区与 Skill 只读投射（设计草案）](docs/spec/project-isolated-agent-workspaces-and-skills.md)
- [项目基础资产与分集变体确认记录](docs/discovery/0014-project-asset-episode-variants.md)
- [B2 与横向资产管理的确认记录](docs/discovery/0013-production-breakdown-and-asset-lifecycle.md)
- [领域词汇表：CONTEXT.md](CONTEXT.md)
- [ADR-0001：创意点子与分集派生关系](docs/adr/0001-idea-to-work-derivation.md)
- [ADR-0002：独立作品与分集同一领域对象](docs/adr/0002-episode-is-independent-work.md)
- [ADR-0003：片段可选一级子片段](docs/adr/0003-optional-single-level-subsegments.md)
- [ADR-0004：剧本场次与导演片段一对多映射](docs/adr/0004-screenplay-scene-to-director-segment.md)
- [ADR-0005：创意方向独立保存，选中后创建分集](docs/adr/0005-independent-concepts-before-episode.md)
- [ADR-0006：创意库仅项目级，复制内容即可复用](docs/adr/0006-project-local-idea-pools.md)
- [ADR-0007：定稿产物不可覆盖，修订需新草稿与再次确认](docs/adr/0007-approved-artifact-revision-policy.md)
- [ADR-0008：依据版本依赖标记下游待复核，不自动重做](docs/adr/0008-review-dependent-artifacts-without-auto-regeneration.md)
- [ADR-0009：正式阶段必须依赖上游已定稿产物](docs/adr/0009-approved-upstream-stage-gates.md)
- [ADR-0010：正式交接采用精简版本锚定资料](docs/adr/0010-compact-versioned-agent-handoff.md)
- [ADR-0011：交接必需事实预置与 Agent 按需读取](docs/adr/0011-mandatory-handoff-context-controlled-retrieval.md)
- [ADR-0012：分字段权威分工](docs/adr/0012-field-owned-markdown-json-sqlite.md)
- [ADR-0013：先写入不可变版本文件，再发布 SQLite 引用](docs/adr/0013-file-first-version-publication.md)
- [ADR-0014：B2 制作拆解独立，创作资产横向贯穿多个阶段](docs/adr/0014-production-breakdown-with-cross-stage-assets.md)
- [ADR-0015：项目基础资产可复用，分集可建立独立变体](docs/adr/0015-project-assets-with-episode-variants.md)
- [ADR-0016：CLI 生态优先，可通过官方 Agent SDK 控制](docs/adr/0016-cli-first-with-official-agent-sdk.md)
- [ADR-0017：项目独立 Agent 工作区、Skill 按项目投射，禁止访问其他项目](docs/adr/0017-project-isolated-agent-workspaces-and-skill-projection.md)
- [grill-with-docs 规范](docs/process/grill-with-docs/SKILL.md)
- [CONTEXT.md 术语格式](docs/process/grill-with-docs/CONTEXT-FORMAT.md)
- [ADR 决策记录格式](docs/process/grill-with-docs/ADR-FORMAT.md)

## 文档约定

- `docs/discovery/`：持续记录用户明确要求、未决问题、方案比较与逐轮结论；必须标注哪些已确认、哪些仍是假设。
- `CONTEXT.md` 或未来的 `CONTEXT-MAP.md`：只收录经确认的项目领域术语，不装实现方案；首次术语确认时再创建。
- `docs/adr/`：只在实际权衡后的重大、难逆转架构决策确定时创建。
- `docs/spec/`（按需）：领域、数据、接口、交互和阶段产物的详细设计规格；在决策逐步清晰后扩展。

目前仓库以设计讨论为主。不要把讨论草案误读为已经批准的开发规范。
