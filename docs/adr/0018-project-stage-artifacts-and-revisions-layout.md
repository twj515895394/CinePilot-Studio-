# 项目级创意/资产、分集阶段目录与独立产物版本

**状态：accepted**
**日期：2026-10-30**

CinePilot Studio 用户在 Vue/Electron 中创建项目时，系统为其创建独立项目工作区。项目级的创意点子/方向位于 `creative-pool/`，项目基础角色/场景等资产和媒体位于 `assets/` 与 `media/`；选定创意方向创建分集后，该分集在 `episodes/{episode_id}/stages/` 下分别组织 A3 故事、A4 剧本、B1 导演、B2 制作拆解、B3 分镜、B4 提示词。阶段内产物以稳定 `artifact_id` 区分，并分别存放可修改草稿与不可覆盖的定稿版本，以支持 UI 和 Agent 按来源查阅及历史回溯。Agent 执行的临时内容位于专属工作目录，已启用的系统 Skill 则由受控投射提供。

**取舍**：不把全部产物混入项目根的统一无类别目录（不利于用户直观定位各阶段文件）；也不把当前可变文件直接放在阶段根、全部旧版另挪到统一历史目录（易与已批准不可变快照混淆）。项目级创意池独立于分集目录，符合选中 Concept 后才创建 Episode 的已确认规则。

**边界**：此决定确认的是**物理目录的业务分层与产物/版本组织**，不锁死所有文件名、每个镜头是否单独成为文件、磁盘迁移格式或数据库表。此前示例的 `cinepilot-data/app/cinepilot.sqlite` 为**单一中心数据库的推荐布局，SQLite 实际按系统共享还是按项目独立仍待用户明确选择**；不以本 ADR 把数据库作用域冒充已确认。无论 SQLite 的位置如何，Agent 不能直接访问全局应用数据或其他项目，必须经受控服务查询。阶段产物和数据库版本仍遵守 [ADR-0012](./0012-field-owned-markdown-json-sqlite.md)、[ADR-0013](./0013-file-first-version-publication.md) 与 [ADR-0017](./0017-project-isolated-agent-workspaces-and-skill-projection.md)。
