# CinePilot Studio — Agent 协作约定

本仓库目前处于**领域建模和架构设计讨论阶段**，所有参与讨论或开发的 Agent 应先阅读：

1. `README.md`
2. `docs/process/grill-with-docs/SKILL.md`
3. `docs/process/grill-with-docs/CONTEXT-FORMAT.md`
4. `docs/process/grill-with-docs/ADR-FORMAT.md`
5. `docs/discovery/` 中的最新讨论记录
6. 若存在，`CONTEXT-MAP.md`、`CONTEXT.md`、`docs/adr/` 的现有内容

## 强制工作方式

- 每次**只提出一个**关键架构问题，附带有依据的推荐与具体边界场景，等待用户反馈后再推进。
- 已能从仓库/已确认文档得知的事实不要重复询问。
- 区分「用户明确要求」「已确认决策」「候选建议」「未决问题」，不要替用户确认。
- 新术语确认后**立刻**更新正确上下文的 CONTEXT.md；此文件只写领域词汇表，不写实现设计。
- 仅当决策难以逆转、脱离上下文容易令人困惑、且有真实替代方案权衡时，按规范新增简洁 ADR。
- 所有重要讨论过程、提议与理由要及时归档到 `docs/discovery/`，方便后续 Agent 恢复上下文。
- 涉及创意编剧、影视制作的产物、数据关系、状态和实现时，明确上下游依赖，避免凭漫长聊天记录隐式传递。
- 未经用户要求或定稿，不要把架构草案直接落成大量实现代码。

当前已确认：创意点子 1:N 派生分集（每集唯一主创意来源），且「独立作品 = 分集」，不再独立建立作品层级。见 `CONTEXT.md`、`docs/adr/0001-idea-to-work-derivation.md`、`docs/adr/0002-episode-is-independent-work.md`。

**当前单个待讨论问题**：分集内片段、子片段与镜头采用哪种层级组织，见 `docs/discovery/0002-episode-and-shot-hierarchy.md`。
