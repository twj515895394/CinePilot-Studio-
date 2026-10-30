# CinePilot Studio

**Agent-first 的独立短视频创意、剧本与影视制作规划平台（设计讨论阶段）。**

## 当前范围

一个产品内设置两大业务系统：

- **创意编剧**：点子发现、创意发散、故事深化、剧本编写与审查。
- **影视制作**：导演与视听设计、片段/镜头/分镜规划、生成提示词与制作准备包。

创作过程以 Agent + Skill 与用户对话协作为主；系统负责项目、分集、片段、镜头、素材、阶段产物、版本、审批与回溯，并通过 Web 页面浏览和操作。

本期**不实现**视频生成；后续考虑 ComfyUI、RunningHub、即梦/Seedance 等适配。

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
- [第十三轮待讨论：文件与 SQLite 的保存一致性](docs/discovery/0012-cross-store-publish-consistency.md)
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
- [grill-with-docs 规范](docs/process/grill-with-docs/SKILL.md)
- [CONTEXT.md 术语格式](docs/process/grill-with-docs/CONTEXT-FORMAT.md)
- [ADR 决策记录格式](docs/process/grill-with-docs/ADR-FORMAT.md)

## 文档约定

- `docs/discovery/`：持续记录用户明确要求、未决问题、方案比较与逐轮结论；必须标注哪些已确认、哪些仍是假设。
- `CONTEXT.md` 或未来的 `CONTEXT-MAP.md`：只收录经确认的项目领域术语，不装实现方案；首次术语确认时再创建。
- `docs/adr/`：只在实际权衡后的重大、难逆转架构决策确定时创建。
- `docs/spec/`（按需）：领域、数据、接口、交互和阶段产物的详细设计规格；在决策逐步清晰后扩展。

目前仓库以设计讨论为主。不要把讨论草案误读为已经批准的开发规范。
