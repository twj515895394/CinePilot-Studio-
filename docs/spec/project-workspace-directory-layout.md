# CinePilot Studio — 项目工作区目录规范（v0.3，目录分层及系统单 SQLite 已确认）

> 日期：2026-10-30
> 状态：**已确认**目录分层方案 A + 全系统单 SQLite；具体文件名和 Schema 为实施提案。参见 [ADR-0018](../adr/0018-project-stage-artifacts-and-revisions-layout.md)、[ADR-0019](../adr/0019-single-system-sqlite-with-project-content-files.md)。
> 前置：[整体产品总览](./overall-product-workflow-and-system-map.md) / [阶段合同](./stage-contracts-and-artifact-map.md) / [资产模型](./creative-asset-model-and-use.md) / [Agent 工作区隔离](./project-isolated-agent-workspaces-and-skills.md) / [ADR-0012](../adr/0012-field-owned-markdown-json-sqlite.md) / [ADR-0013](../adr/0013-file-first-version-publication.md)

## 1. 为什么现在确定目录结构

Vue/Electron「创建项目」后，Python 应立即创建独立工作区。以后 A1/A2 项目级创意内容、分集 A3–B4 的具体产物、角色/场景与多视图素材、Agent 临时生成文件、项目级启用 Skills，都必须落在可预测的地方。**管理后台不能仅靠文件扫描决定定稿与版本**：它通过 SQLite 的项目/分集/产物/版本关系定位相应 Markdown/JSON/媒体文件，然后显示实际内容。

此前的总览与安全文档都包含目录示意，但没有确定统一目录规范；在 UI 和数据 Schema 之前补齐这一步。

## 2. 已确认项目目录分层与系统级 SQLite

~~~text
cinepilot-data/                        # 由系统配置的总数据根；不是 Agent cwd
├── app/
│   ├── cinepilot.sqlite              # 已确认：系统级关系/版本/审批数据库（物理数据根可配置）
│   └── private/                      # 凭据、系统私有配置；不可让 Agent 读取
├── skill-registry/                   # 系统安装/维护的 Skills 来源（不可全量暴露）
└── projects/
    └── {project_id}/                 # 新建项目时创建，亦是该项目的逻辑工作区根
        ├── creative-pool/            # 项目级；在创建分集之前就存在
        │   ├── ideas/
        │   │   └── {idea_id}/
        │   │       └── idea.json
        │   └── concepts/
        │       └── {concept_id}/
        │           └── concept.json
        │
        ├── assets/                   # 项目基础资产，不跟随某个 B2 阶段
        │   ├── characters/{asset_id}/
        │   │   └── revisions/{revision_id}/asset.json
        │   ├── locations/{asset_id}/
        │   ├── costumes/{asset_id}/
        │   ├── props/{asset_id}/
        │   └── styles/{asset_id}/
        │
        ├── media/                    # 真实图片/参考媒体，media_id 可指向多个资产
        │   └── {media_id}/
        │       └── original.{ext}
        │
        ├── episodes/
        │   └── {episode_id}/         # 分集创建时生成，不需项目创建时预创建空分集
        │       ├── asset-variants/   # 本集专用造型、服装、状态变体
        │       │   └── {variant_id}/
        │       │       └── revisions/{revision_id}/variant.json
        │       └── stages/           # 六个分集级正式阶段
        │           ├── a3-story/
        │           ├── a4-screenplay/
        │           ├── b1-director/
        │           ├── b2-breakdown/
        │           ├── b3-storyboard/
        │           └── b4-prompts/
        │
        ├── .agent-work/              # 任务临时文件；按 run_id 隔离，非正式产物
        │   └── {run_id}/
        └── .agent-skills/            # 逻辑 Skill 入口；受控只读投射/快照
~~~

注：用单独的 \`creative-pool/ideas\` 与 \`creative-pool/concepts\` 而不是硬编码在 \`episodes/\` 中，是因为 **Idea 和 Concept 早于 Episode**。二者的来源关系与选择状态由 SQLite 管理，而不是靠目录嵌套决定；A1/A2 不强制使用分集正式阶段审批规则。

**上图为概念布局，尚未确认确切文件名**。如 \`assets/locations/{asset_id}/\` 等同样采用稳定 ID/版本方式，省略的内部层级参照角色资产示意。

## 3. 分集阶段内的产物与版本分层（已确认原则）

已确认正式阶段按独立产物及草稿/定稿版本组织。推荐以 **artifact_id** 标识产物，以 **draft_id** 或 **revision_id** 隔离草稿和定稿；确切文件命名与内容粒度留待 Schema 设计。

~~~text
episodes/{episode_id}/stages/{stage_code}/
└── artifacts/
    └── {artifact_id}/
        ├── drafts/
        │   └── {draft_id}/            # 当前/历史草稿，具体草稿保存策略待细化
        │       ├── content.md        # 仅当这个产物包含正文
        │       └── structure.json    # 仅当这个产物包含结构化内容
        └── revisions/
            ├── {revision_id_1}/       # 已发布的不可变正式快照
            │   ├── content.md
            │   └── structure.json
            └── {revision_id_2}/       # 第二次定稿，不覆盖 revision_id_1
                ├── content.md
                └── structure.json
~~~

**同一产物不必同时有 Markdown 与 JSON**，按其字段权威选择。例如剧本正文在 \`screenplay.md\`，场次 ID/顺序/元数据在 \`scenes.json\`；提示词包可只存 \`prompts.json\`。不要将可独立修改的同一字段同时写入两种载体。

每个正式阶段推荐的产物：

| 阶段目录 | 主要产物与文件（示例） | 作用 |
|---|---|---|
| \`a3-story\` | \`story.md\`, \`beats.json\` | 完整故事梗概、结构与剧情节点 |
| \`a4-screenplay\` | \`screenplay.md\`, \`scenes.json\` | 剧本正文与场次索引 |
| \`b1-director\` | \`director.md\`, \`visual-rules.json\` | 导演阐述与视听约束 |
| \`b2-breakdown\` | \`breakdown.json\` | 剧本场次→片段/子片段映射、素材需求 |
| \`b3-storyboard\` | \`shots.json\`、分镜图片的媒体引用 | 镜头设计、动作、时长、参考媒体 ID |
| \`b4-prompts\` | \`prompts.json\`, \`package.md\`（可选） | 镜头提示词与导出制作包 |

**颗粒度尚未锁定**：以上推荐「阶段内按可审阅的产物对象创建 artifact_id」，不是所有阶段必须每张卡片、每场戏、每个镜头都有独立磁盘文件。Scene/Segment/Shot 可以保持结构化 JSON 对象，以稳定 ID 精确查找；是否独立拆文件按产物大小和 UI 编辑行为在 Schema 设计中确定。

### A4 剧本真实路径示例

~~~text
projects/P001/episodes/EP03/stages/a4-screenplay/artifacts/SCR03/
├── drafts/DRAFT05/
│   ├── screenplay.md
│   └── scenes.json
└── revisions/REV03/
    ├── screenplay.md
    └── scenes.json
~~~

Vue 的「打开剧本」先让 Python 在 SQLite 查询 \`SCR03\` 的当前已定稿 \`REV03\`，再由业务服务读取上面对应路径的正文和场次结构。之后用户修改时在 \`DRAFT05\` 中保存；系统确认定稿再发布新 \`REV04\`。老版 \`REV03\` 仍可查看，不能被 Agent 或 UI 原位覆盖。

## 4. 项目创建和后续目录生成时机

### 4.1 用户点击「新建项目」

Python 通过受控创建流程登记项目身份并创建 **项目根 + creative-pool + assets + media + episodes + .agent-work**；文件创建与 SQLite 登记不是同一原子事务，失败清理和恢复由开发按常规工程要求处理。按项目启用的 Skills 可以在创建时设置，但不强制预先复制系统所有 Skills。系统记录项目根路径或存储位置映射；不根据用户显示名称直接拼接磁盘路径。

**不需要创建 6 份空的分集阶段目录**：分集尚未创建。用户选择 Concept 并新建 Episode 时，才创建 \`episodes/{episode_id}/\`；该集首次生成正式阶段草稿时，创建相应阶段 \`artifacts/...\` 文件夹（或者提前建立六个空目录，两者不改变产品行为，默认建议按需创建）。

### 4.2 Agent 生成 → 后台可见

1. Vue/Electron 在项目/分集工作台触发任务；Python 选择本项目工作区、当前正式阶段与上游定稿版本。
2. Agent 在受隔离的 \`.agent-work/{run_id}\` 中创建草稿文件，或通过受控 \`submit_draft\` 工具提交结果；**不允许自由覆写正式阶段目录**。
3. Python Artifact Service 根据 \`project_id + episode_id + stage + artifact_id\` 将通过校验的内容保存到对应 \`stages/{stage}/artifacts/{id}/drafts/{draft_id}/\`，同时在 SQLite 中记录身份/草稿状态/位置。
4. Vue 通过业务 API 显示草稿，用户编辑、审查，点击「定稿」时，先写入不可变 \`revisions/{revision_id}/\` 内容，再由 SQLite 发布当前定稿版本指针。
5. 下阶段 Handoff 读取 SQLite 中批准的 version ID，受控检索对应产物目录下必要的内容，不依赖聊天回忆和全目录扫描。

注意「Agent 以项目目录为工作区」与「Agent 可以任意修改整个项目目录」是不同要求：为了满足定稿不可覆盖，建议 Agent 对正式产物只有读权限，仅在任务草稿工作区可写；项目产物通过统一业务服务落盘。

## 5. 素材与分集变体不是同一层级

项目基础角色 \`R01\` 的身份、基础外观和参考视图可以存 \`assets/characters/R01/revisions/REV03/asset.json\`，图片按稳定媒体 ID 存 \`media/{media_id}/original.{ext}\`。分集 EP01/EP02 各自维护造型/服装/状态变体 \`episodes/EP01/asset-variants/VAR01/revisions/.../variant.json\`，再由镜头的 \`shots.json\` 引用 \`R01 + base_revision + VAR01 + media_id\`。

这遵守已确认的「项目基础资产 + 分集显式变体」规则；**不需要在每个镜头文件夹复制角色面部图片，也不让文件名本身作为角色主键**。图片/媒体在项目目录下，访问通过 Python 服务解析而不是向 Agent 暴露其他项目文件。

## 6. 不能忽略的安全边界

- \`projects/P001/\` 是项目的**物理工作区**，但 Agent 的操作系统/容器/沙箱实际可见范围仍须限制为 P001 的允许内容及只读技能；绝不能因此允许跨目录读取兄弟项目。
- 系统级 SQLite 数据库文件、系统私有凭据、未启用的 Skills、其他项目根目录不向 Agent 暴露；数据库只由 Python 业务服务读取和更新，Agent 不得直接操作 SQLite。
- Skill 来自系统 Registry，运行时投射至隔离环境兼容目标 Agent 的发现目录；项目目录中的 \`.agent-skills\` 仅是逻辑位置，实际是否用符号链接或快照按平台实现。
- 使用稳定 ID、服务端安全路径拼接、隔离文件系统；禁止 Agent 输入 \`../\` 或不受检验的绝对路径影响文件定位；业务 API 必须同样按项目权限验证。
- 不能仅凭设置 \`cwd\` 宣称强隔离；实现时必须以跨项目文件/API 禁读测试验收。

## 7. 目录和 SQLite 作用域均已确认

- **项目目录方案 A**：`creative-pool/`、`assets/`、`media/` 与 `episodes/{episode_id}/stages/`，阶段内按产物的草稿和不可覆盖定稿版本组织；见 [ADR-0018](../adr/0018-project-stage-artifacts-and-revisions-layout.md)。
- **SQLite 方案 A**：所有项目共用系统级 SQLite，推荐逻辑位置 `app/cinepilot.sqlite`，只由 Python 统一业务服务访问，不放到 Agent 可访问范围；见 [ADR-0019](../adr/0019-single-system-sqlite-with-project-content-files.md)。
- **查询原则**：SQLite 索引负责找到项目/分集/阶段/产物/定稿版本及其文件位置；Python 按需读取对应 Markdown/JSON/图片原文。后续如果需要跨大量文档搜索，可添加 SQLite 全文检索索引，但原始文件仍为相应正文/结构的权威。
- **项目搬迁提醒**：复制项目文件夹能取得内容与媒体，但不能独立恢复集中 SQLite 中的审批与版本记录；必须提供项目级导出/导入，把该项目相关的关系和版本元数据一起封装。

具体字段 Schema、ID 命名、按需内容片段粒度、性能缓存和常规数据完整性属于后续实现设计。Agent 的操作系统级沙箱仍只暴露本项目工作视图与获准的只读 Skill，不能因全局数据库存在而拥有其他项目访问权。
