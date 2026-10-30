# SQLite 放在哪里：全局索引数据库 vs 每项目独立数据库

> 日期：2026-10-30
> 状态：**已确认 A：全系统共用一个 SQLite**；项目/分集产物目录方案 A 也已确认，参见 [ADR-0018](../adr/0018-project-stage-artifacts-and-revisions-layout.md) 与 [ADR-0019](../adr/0019-single-system-sqlite-with-project-content-files.md)
> 依据：[项目目录规范](../spec/project-workspace-directory-layout.md)、[字段权威 ADR-0012](../adr/0012-field-owned-markdown-json-sqlite.md)、[项目 Agent 隔离 ADR-0017](../adr/0017-project-isolated-agent-workspaces-and-skill-projection.md)

## 问题由来

用户已确认：在 UI 创建项目时创建项目目录，阶段产物按分集/阶段放在项目内部，对话与生成过程经 Python 统一业务服务持久化。看到目录树中的 \`app/cinepilot.sqlite\` 后问“那 SQLite 呢？”，因此需要明确：数据库只是位于系统级的数据关系和审批状态存储，还是每个项目必须自带独立数据库。**之前只确认 Markdown+JSON+SQLite 的各自业务权威，不等于已经确认 SQLite 的物理作用域**。

## SQLite 到底保存什么

SQLite 的权威字段（依 ADR-0012）：身份、关系、阶段、审批、定稿版本与查询索引；具体 Schema 尚待后续定义。

| 类别 | SQLite 存储示意 | 相关文件 |
|---|---|---|
| 项目/分集 | Project、Episode、主创意来源、当前阶段 | 项目目录与分集目录 |
| 创意池 | Idea、Concept 的 ID 和源关系、选择/归档状态 | \`creative-pool/ideas\`, \`creative-pool/concepts\` 的 JSON |
| 阶段产物 | artifact_id、stage、episode_id、内容位置索引 | \`stages/{stage}/artifacts/{id}/\` |
| 草稿与定稿 | revision_id、审批状态、已定稿指针、来源版本 | \`drafts/\`, \`revisions/\` 的 MD/JSON |
| 场次与镜头关系 | Scene/Segment/Shot 的稳定身份及来源定位 | \`scenes.json\`, \`breakdown.json\`, \`shots.json\` |
| 角色素材 | 资产、分集变体、media_id 与各级引用 | \`assets/\`, \`asset-variants/\`, \`media/\` |
| Agent 任务 | run_id、project_id、stage、产物引用、执行状态 | \`.agent-work/\` 为临时工作文件，不是定稿事实来源 |
| 阶段依赖 | 哪个正式版本依赖哪个上游版本、待复核 | Markdown/JSON 原始产物保持独立 |

**不要**将同一份可编辑剧本全文同时写到 SQLite 和 Markdown，也不要将镜头结构全文在 SQLite 和 JSON 里维护两份可独立修改的权威。少量派生的搜索索引/展示摘要可存在 SQLite，但以各字段指定的权威为准。

## 方案 A：系统共用一个 SQLite（V1 推荐）

~~~text
cinepilot-data/
├── app/
│   └── cinepilot.sqlite     ← 只允许 Python 业务服务访问
├── skill-registry/
└── projects/
    ├── P001/               ← 全部 P001 创作正文、JSON、图片与任务工作区
    └── P002/               ← 全部 P002 创作正文、JSON、图片与任务工作区
~~~

- 一个数据库存多个项目的元数据；关系/版本/审批查询必须有明确 \`project_id\` 范围，由 Python API 与权限层**服务端强制验证**。
- Agent 仅看到隔离后的当前项目文件和有限受控查询，不直接挂载或打开这个 SQLite，也不能用 Agent 参数绕开作用域。
- Vue/Electron 项目列表、跨多个分集的进度/待复核/任务汇总，只需一个 SQLite 查询源。
- 创建项目时：服务创建 P001 文件目录，并在 SQLite 中登记 Project 的身份、逻辑存储位置；A3–B4 产物正文仍全部在 P001 下面。
- 读取 EP01 剧本：服务按 \`project_id/episode_id/artifact_id/revision_id\` 在 SQLite 中找已批准的修订和对应内容位置，再读取 \`projects/P001/episodes/EP01/stages/a4-screenplay/...\`。
- 项目备份/迁移须由系统导出该项目的 MD/JSON/媒体和与之相关的 SQLite 元数据（例如可转存 JSON manifest）；**仅复制 P001 文件夹不能完整恢复审批/版本关系**。
- 优点：符合当前轻量单体 Python 后端、一套 UI 和版本控制，无需跨多个项目数据库管理与同步。
- 代价：单数据库是业务元数据集中存储点；需要常规备份、权限测试和按项目导入导出功能。Python 业务服务要保证所有按 ID 查询都验证项目归属，不能只要求调用者传 \`project_id\`。

## 方案 B：每个项目各有 SQLite，系统另有轻量项目注册表

~~~text
cinepilot-data/
├── app/
│   └── registry.sqlite        ← 只管理系统设置、项目列表/项目目录索引
└── projects/
    ├── P001/
    │   ├── .cinepilot/
    │   │   └── project.sqlite ← P001 身份、版本、审批、任务等元数据
    │   ├── creative-pool/
    │   ├── assets/
    │   ├── episodes/
    │   └── media/
    └── P002/
        └── .cinepilot/project.sqlite
~~~

- 每个项目具有自己的数据库和 MD/JSON/图片，适合项目整体导出/搬迁/独立备份、将来不同存储卷或独立部署。
- 全系统项目列表至少还需一个注册表（可以是中心轻量 SQLite）；跨项目统计需要遍历/汇总多个数据库。
- **非常重要**：虽然项目数据库在 P001 文件夹内，Agent 不应直接得到该文件的读写权。Agent 的隔离运行视图必须排除 \`.cinepilot/\` 或仅暴露经授权的项目内容；否则 Agent 可以绕开审核流程修改项目审批和定稿记录。
- 优点：项目文件夹更接近独立的可迁移工作包。
- 代价：双层元数据管理、项目级 DB 升级、连接池、备份一致性和运行时排除 DB 的隔离机制更复杂。SQLite 文件系统权限必须适应 Electron/本机/远端模式。

## 本轮建议及确认范围

**推荐方案 A**：V1 一个系统级 \`app/cinepilot.sqlite\`，所有项目创作文件按已确认目录结构保存。它是业务服务内部的数据关系与版本目录，不是给 Agent 自己扫描的公共资料。此方案最契合用户希望先完整跑通 Agent 创作和管理后台、不为通用异常设计过度架构的方向。

但 **如果最重要的目标是拷贝整个项目文件夹即可在另一台电脑上恢复完整项目（包括版本/审批）**，方案 B 更直接。二者区别在于是否将关系和状态也作为项目自包含内容；无论选择哪个方案，Agent 都应被禁止直接修改 SQLite。

### 唯一需要用户确认的架构边界

- **A（推荐）**：一个系统级 SQLite，项目 MD/JSON/图片各自独立存放；系统后续做项目级导出/导入。
- **B**：每项目一个 SQLite，另有轻量项目注册表，项目尽量自包含、易单独搬迁。

## 本轮正式结论（2026-10-30）

**用户明确确认 A：全系统共用一个 SQLite 数据库。** 用户认为这也是后台管理系统集中查询项目、分集和创作状态的自然选择。已形成 [ADR-0019](../adr/0019-single-system-sqlite-with-project-content-files.md)。

**用户追问查询速度**：读取文档原文时，SQLite 和文件的速度没有绝对胜者；对于通过已知 ID 定位版本、跨项目/跨分集的列表过滤，带索引的 SQLite 更合适；对于直接读取一个已知 Markdown 文件的全文，文件读取已经很高效；对于跨大量文档搜索文本，可后续使用 SQLite FTS5 构建可重建索引，再按结果 ID 读取原文。**索引和缓存是派生加速层，不能成为一份可独立编辑的创作正文权威**。

**推荐读取链路**：UI/Agent → Python 业务服务 → SQLite 查询对象/权限/版本/文件定位 → 按需读取 Markdown/JSON 或媒体 → 返回当前作用域需要的文档/片段；先实现稳定 ID 定位和字段索引，不为了“数据库读取可能更快”重新复制全部创作正文到 SQLite。

**补充说明**：由于当前选择中心数据库，用户若想将某个项目单独备份或迁移，系统需要导出该项目的内容文件和关联的 SQLite 关系/版本元数据；复制 P001 文件夹本身并不包含全部审批和版本事实。

字段表结构、文件缓存、FTS5 的中文检索方法、索引维护及备份恢复属于开发和性能验收细节，不在此轮追加异常情况 Grill。
