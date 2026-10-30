# CinePilot Studio — Agent 协作约定

本仓库目前处于**领域建模和架构设计讨论阶段**，所有参与讨论或开发的 Agent 应先阅读：

1. `README.md`
2. `docs/process/grill-with-docs/SKILL.md`
3. `docs/process/grill-with-docs/CONTEXT-FORMAT.md`
4. `docs/process/grill-with-docs/ADR-FORMAT.md`
5. `docs/discovery/` 中的最新讨论记录
6. 若存在，`CONTEXT-MAP.md`、`CONTEXT.md`、`docs/adr/` 的现有内容

## 强制工作方式

- 先给出完整的端到端方案、推荐和依赖关系，再让用户校准关键产品边界；不要为了通用工程细节逐项拆成大量 Grill 选择题。
- 只对影响产品主流程、核心领域关系、用户体验、长期架构锁定或 V1 范围的事项提问；异常处理、数据校验、日志、备份、基本恢复等通用工程健壮性按开发默认要求处理。
- 已能从仓库/已确认文档得知的事实不要重复询问。
- 区分「用户明确要求」「已确认决策」「候选建议」「未决问题」，不要替用户确认。
- 新术语确认后**立刻**更新正确上下文的 CONTEXT.md；此文件只写领域词汇表，不写实现设计。
- 仅当决策难以逆转、脱离上下文容易令人困惑、且有真实替代方案权衡时，按规范新增简洁 ADR。
- 所有重要讨论过程、提议与理由要及时归档到 `docs/discovery/`，方便后续 Agent 恢复上下文。
- 涉及创意编剧、影视制作的产物、数据关系、状态和实现时，明确上下游依赖，避免凭漫长聊天记录隐式传递。
- 未经用户要求或定稿，不要把架构草案直接落成大量实现代码。

当前已确认：创意点子 1:N 派生分集（每集唯一主创意来源）；「独立作品 = 分集」；分集下片段可直接包含镜头，或可选一层子片段再包含镜头（同一片段不可混放）；剧本场次与导演片段独立建模、一对多映射且导演片段仅有一个来源场次；**创意方向独立可保存/筛选/修改，只有用户选中方向并决定进行作品开发才创建分集**。见 `CONTEXT.md` 与 `docs/adr/0001-*.md` 至 `0005-*.md`。

**第六轮已确认**：创意点子、创意方向完全归属各自项目；跨项目使用只允许 UI 复制文本、粘贴给另一项目 Agent 重新发散，**不设计全局创意库、跨项目共享/导入、同步、来源引用机制**。见 `docs/adr/0006-project-local-idea-pools.md`。

**第七轮已确认**：UI 与 Agent 均不得覆盖已定稿产物；修改先产生新草稿，用户确认后才形成新的定稿，旧版本和已存在的下游产物都保留。见 `docs/adr/0007-approved-artifact-revision-policy.md`。

**第八轮已确认**：上游产生新定稿后，系统依据版本依赖标记相关下游待复核，不自动重做；用户决定继续沿用或交给 Agent 修订。见 `docs/adr/0008-review-dependent-artifacts-without-auto-regeneration.md`。

**第九轮已确认**：同一分集只有上游必要创作阶段定稿后才能正式进入下游阶段，所有正式 Handoff 需明确上游定稿版本；不同分集可独立推进或批量推进。点子与创意方向在分集建立之前管理，不得误套分集内部阶段门禁。见 `docs/adr/0009-approved-upstream-stage-gates.md`。

**第十轮已确认**：正式 Handoff 精简、版本明确、按需展开，包含项目/分集、上游定稿版本、已确认约束与任务/产物要求；不默认加载完整聊天或全部文档。见 `docs/adr/0010-compact-versioned-agent-handoff.md`。用户提出数据库/知识库保存大内容并按需检索，但具体检索架构尚未定稿，尤其不能直接把独立知识库当成已批准需求。

**第十一轮已确认**：系统预先提供任务目标、当前有效强制约束和准确的上游定稿版本依据；Agent 通过受控接口按需读取详细原文，不靠自主探索发现必需约束，也不整包加载全部资料。见 `docs/adr/0011-mandatory-handoff-context-controlled-retrieval.md`。

**明确的管理后台需求**：已保存/定稿的创意、剧本、导演、分镜、提示词需要在 UI 中按项目、分集、阶段和版本读取、浏览与管理；分集建立前的项目创意池也必须可见。第十二轮已确定三类数据载体的权威分工；第十三轮已确定文件先行、SQLite 发布版本引用。具体字段 Schema 和恢复实现属于后续技术设计。

**第十二轮已确认**：Markdown 管正文、JSON 管结构化产物、SQLite 管关系和版本审批，写入经统一服务。见 ADR-0012。

**第十三轮已确认**：先准备并校验新的不可变 Markdown/JSON 版本文件，再由 SQLite 发布版本引用；不覆盖旧定稿。见 `docs/adr/0013-file-first-version-publication.md`。

**已确认技术栈**：Web 前端 Vue，业务后端 Python，桌面端 Electron（复用 Vue 工作台）；持久化 Markdown + JSON + SQLite。具体 Vue 状态管理、Python HTTP 框架、Electron 与 Python 的启动/进程通信、桌面安装包形式均未定稿，不得擅自固定为特定实现。

**已确认的 B2 与资产管理边界**：B2 是正式且独立的「制作拆解」阶段，负责剧本场次→导演片段/可选子片段规划及本集资产需求核对；角色、场景、服装、道具等资产贯穿 A3–B4 管理，不能限定到 B2 才能创建。见 `docs/adr/0014-production-breakdown-with-cross-stage-assets.md`。

**已确认 V1 Agent 运行路线与项目隔离**：优先 CLI 生态，官方 Agent SDK 可优先作为 Python 控制接口（并非直接模型 API）；每个项目创建独立 Agent 工作区，任务 cwd 指向该项目，项目已启用 Skill 从系统 Skill Registry 只读链接/投射。Agent 进程、子进程与所有数据查询工具**绝不可访问其他项目目录或数据**；仅设置 cwd/Prompt/普通 symlink 不够，必须满足 OS/执行环境隔离，已定稿内容不可被 Agent 任意改写。见 `docs/adr/0016-cli-first-with-official-agent-sdk.md`、`docs/adr/0017-project-isolated-agent-workspaces-and-skill-projection.md` 与 `docs/spec/project-isolated-agent-workspaces-and-skills.md`。

**已确认目录分层**：Vue/Electron 新建项目立即创建独立项目工作区；项目 `creative-pool/`、`assets/`、`media/`；分集 `episodes/{episode_id}/stages/` 分别存 A3–B4 六个正式阶段；阶段内产物区分草稿与不可覆盖定稿版本。参见 `docs/adr/0018-project-stage-artifacts-and-revisions-layout.md` 与 `docs/spec/project-workspace-directory-layout.md`。具体文件名和 Schema 尚未定稿。Agent 仅写任务草稿，通过业务服务保存正式产物。

**尚待明确的 SQLite 作用域**：此前项目目录草案中 `app/cinepilot.sqlite` 是系统单库方案示例；用户已确认目录分层，不代表已经决定全局单 SQLite 或每项目独立 SQLite。见 `docs/discovery/0016-sqlite-scope-and-project-portability.md`。无论数据库位置均不可让 Agent 直接读写状态/审批数据库。

**当前设计重点**：继续完成整体产品流程、阶段合同与核心产物、素材资产使用关系、Agent/Skill 协作、后台和 V1 范围。总览草案见 `docs/spec/overall-product-workflow-and-system-map.md`；阶段合同见 `docs/spec/stage-contracts-and-artifact-map.md`；资产模型草案见 `docs/spec/creative-asset-model-and-use.md`。**项目内多分集资产复用已确认**：项目基础资产可被同项目多集引用，分集显式选择自己的外观/服装/状态变体，不直接改动项目基础定义，见 `docs/adr/0015-project-assets-with-episode-variants.md`。媒体文件路径/名称不应作为业务关联主键，推荐通过资产/版本/变体/媒体 ID 解析（具体 Schema 未定稿）。Agent/Skill 的推荐路由、任务、查询及运行时适配设计在 `docs/spec/agent-skill-orchestration-and-runtime.md`；CLI/SDK/API 首要运行源尚未确认。常规异常处理不单独 Grill。
