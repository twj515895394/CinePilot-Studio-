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
- [第五轮待讨论：创意方向与分集建立时机](docs/discovery/0004-idea-concept-episode-lifecycle.md)
- [领域词汇表：CONTEXT.md](CONTEXT.md)
- [ADR-0001：创意点子与分集派生关系](docs/adr/0001-idea-to-work-derivation.md)
- [ADR-0002：独立作品与分集同一领域对象](docs/adr/0002-episode-is-independent-work.md)
- [ADR-0003：片段可选一级子片段](docs/adr/0003-optional-single-level-subsegments.md)
- [ADR-0004：剧本场次与导演片段一对多映射](docs/adr/0004-screenplay-scene-to-director-segment.md)
- [grill-with-docs 规范](docs/process/grill-with-docs/SKILL.md)
- [CONTEXT.md 术语格式](docs/process/grill-with-docs/CONTEXT-FORMAT.md)
- [ADR 决策记录格式](docs/process/grill-with-docs/ADR-FORMAT.md)

## 文档约定

- `docs/discovery/`：持续记录用户明确要求、未决问题、方案比较与逐轮结论；必须标注哪些已确认、哪些仍是假设。
- `CONTEXT.md` 或未来的 `CONTEXT-MAP.md`：只收录经确认的项目领域术语，不装实现方案；首次术语确认时再创建。
- `docs/adr/`：只在实际权衡后的重大、难逆转架构决策确定时创建。
- `docs/spec/`（按需）：领域、数据、接口、交互和阶段产物的详细设计规格；在决策逐步清晰后扩展。

目前仓库以设计讨论为主。不要把讨论草案误读为已经批准的开发规范。
