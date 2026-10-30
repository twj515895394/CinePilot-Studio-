# CinePilot Studio — Skill Registry 与阶段 Skill 编排规格

> 版本：0.1
> 日期：2026-10-30
> 状态：**用户已确认需要每个阶段具备独立 Skill 配置能力，并允许一阶段组合多个 Skill、一个 Skill 跨阶段复用；用户提供的 8 个候选项目作为第一批研究来源。** Registry、Profile、Router、页面与数据结构为推荐设计。候选 Skill 的能力描述来自用户提供的初始清单，仓库文件/License/Commit 状态为本轮 GitHub 核验结果；是否导入、如何改造尚未批准。
> 关联：[Agent 接入与模型管理](./agent-integration-and-model-management.md) · [Agent/Skill 运行时协同](./agent-skill-orchestration-and-runtime.md) · [产品页面功能](./product-information-architecture-and-page-functions.md) · [核心数据模型](./core-data-model-and-storage-contracts.md) · [阶段合同](./stage-contracts-and-artifact-map.md) · [项目隔离工作区](./project-isolated-agent-workspaces-and-skills.md)

---

## 0. 目标与核心原则

本规格定义 CinePilot Studio 如何登记、治理、组合和执行专业创作 Skill，覆盖 A1–A4 创意编剧与 B1–B4 影视制作八个阶段。

**不是一阶段固定一个 Skill。** 正确模型是：

- 一个阶段有一份 Stage Skill Profile；
- Profile 可以组合主 Skill、辅助 Skill、条件/题材 Skill 和审查 Skill；
- 一个 Skill 可以被多个阶段复用；
- 同一阶段可以根据题材、任务目标和局部对象选择不同 Skill；
- Skill 不绑定 Agent 或 Model，具体由 Agent Adapter 执行；
- Skill Router 不替代 Python 阶段门禁、版本、权限和产物校验。

### 0.1 已确认的产品约束

- A1 点子发现、A2 创意发散是项目级创意池能力；A3–B4 是分集正式阶段。
- 项目可以启用系统 Registry 中的 Skill；Skill 由系统集中维护，并只读投射到当前项目隔离工作区。
- Agent 不可访问其他项目目录、全局 SQLite、系统凭据或未授权 Skill 源目录。
- Handoff 精简并固定上游 Revision；Skill 按需加载，不能把所有 Skill 和资料整包塞进上下文。
- 用户可以通过 Agent 对话产出草稿，统一 ArtifactService 校验保存；阶段整体定稿，局部 Scene/Segment/Shot 可修改。
- V1 不接视频生成 API。
- 用户明确：当前其他阶段缺失的 Skill 后续由用户补充；本文只登记已提供的八个候选，不自行虚构或替用户选择 B1/B2/B4 的外部 Skill。

### 0.2 本规格的关键目标

1. 建立系统级 Skill Registry；
2. 记录外部来源、版本、许可证、作者、上游 Commit、文件摘要和改造来源；
3. 为 A1–B4 分别配置 Stage Skill Profile；
4. 支持项目配置、分集覆盖和单次任务覆盖；
5. 通过 Skill Router 生成可解释的 Skill Plan；
6. 按任务只加载必要 Skill 与 references；
7. 将外部 Skill 的输出适配 CinePilot 阶段产物合同；
8. 固定版本并建立回归评测，防止升级后创作质量或数据合同退化。

---

## 1. Skill 的分类和职责模型

### 1.1 Skill 类型

| 类型 | 职责 | 示例 |
|---|---|---|
| Primary / 主 Skill | 对本次任务的核心创作流程和主要输出负责 | A3 故事深化主创作能力 |
| Supporting / 辅助 Skill | 提供局部方法、结构、研究或创作建议 | 叙事结构工具箱 |
| Conditional / 条件插件 | 满足题材、风格或任务条件时加载 | 喜剧、悬疑、动作、无厘头喜剧 |
| Review / 审查 Skill | 检查已有草稿、提出问题或建议 | 剧本逻辑审查、镜头连续性审查 |
| Utility / 工具 Skill | 辅助结构转换、素材检查、字段校验等 | 提示词格式检查、资产引用核验 |

类型表示 Skill 在本次任务中的职责，不一定是外部仓库原始分类。一个已登记 Skill 可以在不同阶段承担不同角色，但每次 Run 的角色必须明确记录。

### 1.2 多 Skill 协作原则

- 一个 Run 默认只有一个主 Skill 负责生成主要草稿，避免多个 Skill 同时争夺同一产物的主导权。
- Supporting Skill 只提供指定维度的方法或补充，不应无边界重写主 Skill 输出。
- Conditional Skill 只有触发条件满足时才加载。
- Review Skill 默认先给诊断结果；除非用户明确要求，不自动覆盖或重写草稿。
- 阶段合同、用户明确约束和系统安全规则高于任何外部 Skill 的提示词要求。
- 多个 Skill 对同一字段或规则冲突时，按本节优先级解决；无法确定时停止自动组合并要求用户选择。

### 1.3 Skill 与 Agent / Model 分离

Skill 描述“如何完成专业工作”，Agent Runtime 提供执行能力，Model 提供推理能力。

同一个 Skill 可以在不同 Agent 上执行，但每种 Agent 需要通过 Adapter 验证：
- Skill 文件格式和发现目录；
- 引用文件加载方式；
- 辅助脚本运行能力及依赖；
- 工具调用和结构化输出能力；
- 图片输入等多模态能力；
- 沙箱中脚本和文件权限。

Skill 不得把 Codex、Claude Code 或某个模型名称写成永久运行前提。若外部 Skill 原本面向单一 Runtime，应通过 Skill Compatibility Adapter 处理，不兼容时不得假定可以直接使用。

---

## 2. 系统级 Skill Registry

### 2.1 Registry 负责什么

系统 Skill Registry 是所有项目可选择的专业技能目录，负责登记来源、版本、能力、适用阶段、依赖、兼容性、许可证和启用状态。

系统级“已登记”不代表每个项目自动启用。项目必须显式选择允许使用的 Skill 版本。

### 2.2 Skill 元数据

每个 Skill 版本建议包含：

| 字段 | 说明 |
|---|---|
| skill_id | 稳定的内部标识 |
| display_name | 展示名称 |
| skill_version | CinePilot 管理的版本 |
| source_type | external / internal / derived |
| upstream_repository | 原始 GitHub 仓库 URL |
| upstream_path | 仓库内 Skill 目录或入口文件路径 |
| upstream_commit | 导入时固定的 Commit SHA |
| upstream_file_sha | 原始入口文件 Blob SHA |
| upstream_author | 原作者/维护者（按来源记录） |
| license_id / license_status | License 标识及审核状态 |
| attribution / notice | 必须保留的署名和 NOTICE |
| capabilities | 能力标签 |
| supported_stages | 候选/已验证适用阶段 |
| required_inputs | 所需输入类型 |
| output_contracts | 可产出的 CinePilot Artifact 类型 |
| required_tools | 依赖的受控工具 |
| dependencies | references、脚本、模型或外部依赖 |
| runtime_compatibility | 已验证的 Agent Runtime/版本 |
| security_review | 脚本/外部命令/网络访问审查状态 |
| evaluation_status | 未评估、评估中、通过、未通过 |
| lifecycle_status | candidate、reviewed、adapted、enabled、deprecated |

Skill 的来源 URL、Commit、License、署名和文件摘要应在导入时固化，不能只保留一个会变化的 main 分支链接。

### 2.3 Registry 状态

建议状态：
- Candidate：已登记来源，尚未评估；
- Reviewing：正在检查能力、依赖和 License；
- Approved for Adaptation：允许提炼/改造；
- Adapted：已有 CinePilot 适配版本；
- Enabled：可被项目启用；
- Deprecated：不再推荐新任务使用，历史 Run 仍可回溯。

状态变化需要记录时间、操作者、原因和所依据的来源版本。

---

## 3. 外部 Skill 调研、复制与改造治理

### 3.1 不能无差别复制整仓库

外部项目可能包含 Skill 指令、辅助脚本、参考文档、数据集、第三方素材、示例作品和产品代码。它们的许可证与权利范围可能不同。

每个候选至少检查：

1. 仓库是否存在，目标 Skill 路径是否存在；
2. Skill 入口文件和所有引用文件；
3. LICENSE、NOTICE、作者署名及第三方来源声明；
4. 辅助脚本的运行命令、依赖、文件和网络访问；
5. 是否包含第三方课程、影视台词、广告案例、图片或其他受限素材；
6. Skill 是否适合 CinePilot 的短视频叙事，而不是品牌营销、口播或其他特定内容形态；
7. 是否与 CinePilot 阶段拆分、产物合同、版本机制和 Handoff 冲突。

### 3.2 三种处置方式

| 处置方式 | 使用条件 | 处理要求 |
|---|---|---|
| 完整复制/导入 | License 明确允许，依赖和内容范围可控，整体结构适合 | 保留原 License、版权声明、NOTICE、作者和来源 Commit |
| 模块提炼/改造 | 仅部分方法、Prompt、工作流或校验机制有价值 | 记录提炼来源、保留所需署名，标记修改内容，删除不适用模块 |
| 参考方法后独立实现 | License 缺失/不明确、不允许复制或原结构不适合 | 不复制原文/代码/模板；只记录高层方法参考与独立实现说明 |

### 3.3 License 处理规则

- MIT：复制或分发相关代码/文档时保留版权声明和许可证文本；检查第三方内容是否另有许可。
- CC BY 4.0：按要求署名、提供许可证链接、标注修改；保留 NOTICE；不得暗示原作者背书。
- License 缺失或范围不明确：不得把“公开可访问”理解为“允许复制”；先取得许可或仅做不复制表达的独立实现。
- 仓库根 License 不一定覆盖第三方数据集、图片、课程、案例库或嵌入内容；必须逐项核验。
- 本文不是法律意见；高风险商业分发场景应进行额外许可审查。

### 3.4 导入后的来源追溯

每个 CinePilot 派生 Skill 版本至少保存：
- upstream URL、上游 Commit SHA、入口文件 SHA；
- License 文件 SHA、NOTICE 和作者；
- 导入/提炼日期；
- 保留了哪些文件/模块；
- 修改/删除/新增了哪些模块；
- CinePilot 版本号；
- 评估报告与回归测试结果。

禁止只复制内容而丢失来源。升级上游时先创建候选版本，评估通过后由用户/维护者决定是否升级；不自动覆盖已启用版本或历史 Run。

---

## 4. 第一批八个外部 Skill 候选登记

> **核验时间：2026-10-30。** 本轮通过 GitHub 接口确认八个仓库均存在，并定位了各自的 Skill 入口文件。表中 Commit SHA 是核验时各仓库 main 分支的 HEAD；Skill 文件 SHA 是对应入口文件 Blob SHA。它们是来源快照线索，不代表已经导入 CinePilot，也不代表经过质量评估。
>
> 用户最初给出的“核心能力”列为用户原始候选描述；“实际仓库观察”是本轮读取 README/SKILL.md/LICENSE 后的补充判断。两者不能混为一谈。

| # | 候选 Skill | 用户提供的核心能力 | 仓库 / Skill 路径 | License 核验 |
|---|---|---|---|---|
| 1 | Story Idea Generator | 多题材原创故事构思、情绪驱动、脑洞发散 | [jwynia/agent-skills](https://github.com/jwynia/agent-skills/tree/main/skills/creative/fiction/core/story-idea-generator) · skills/creative/fiction/core/story-idea-generator/SKILL.md | Skill frontmatter 声明 MIT；仓库根目录未发现 LICENSE 文件，需确认声明覆盖范围 |
| 2 | Short Film Development | 独立短片创意、冲突、反转、完整故事开发 | [0xhughs/director-skills](https://github.com/0xhughs/director-skills/tree/main/skills/short-film-development) · skills/short-film-development/SKILL.md | 根目录 LICENSE 为 MIT |
| 3 | Comedy Short Video Skill | 喜剧创意、人物反差、笑点升级、剧本 | [heran1225-byte/comedy-short-video-skill](https://github.com/heran1225-byte/comedy-short-video-skill/tree/main/skills/creating-comedy-short-videos) · skills/creating-comedy-short-videos/SKILL.md | 根目录 LICENSE 为 MIT；README 明确课程内容不包含在许可证范围 |
| 4 | Stephen Chow Skill | 无厘头喜剧、荒诞逻辑、地位反转 | [chnjames/stephen-chow-skill](https://github.com/chnjames/stephen-chow-skill) · SKILL.md | 根目录 LICENSE 为 MIT |
| 5 | Creative Director | 跨领域创意碰撞、突破常规、创意评审 | [smixs/creative-director-skill](https://github.com/smixs/creative-director-skill/tree/main/creative-director) · creative-director/SKILL.md | 根目录 LICENSE 为 CC BY 4.0；存在 NOTICE，署名和修改说明必须保留 |
| 6 | AI ShortVideo Story Engine | 故事引擎、钩子、人生逆袭、反转、分镜 | [slad-dev/AI-ShortVideo-Story-Engine](https://github.com/slad-dev/AI-ShortVideo-Story-Engine/tree/main/skills/douyin-script) · skills/douyin-script/SKILL.md | 根目录 LICENSE 为 MIT |
| 7 | Wind Comic Screenwriter | 从点子到完整剧本、镜头与角色一致性 | [ChrisChen667788/wind-comic](https://github.com/ChrisChen667788/wind-comic/tree/main/skills/screenwriter) · skills/screenwriter/SKILL.md | 根目录 LICENSE 为 MIT |
| 8 | Storytelling Frameworks | 六种短视频叙事结构 | [advisorita/storytelling-frameworks](https://github.com/advisorita/storytelling-frameworks) · SKILL.md | 仓库根目录没有 LICENSE 文件，Skill frontmatter 未声明 License；未经许可不得复制其文件/模板 |

### 4.1 上游版本快照

以下 SHA 用于未来下载/复查时固定来源。不要把 main 分支自动更新当成稳定版本。

| Skill | main HEAD Commit SHA | SKILL.md Blob SHA |
|---|---|---|
| Story Idea Generator | e02ec7e226a6e4f8419fd3b88a1d8e472d421b32 | 5e75fd8e97b987c36864c960cf0cbe749c8aaac2 |
| Short Film Development | 35ca0a2b4cd55b0668aa9ea0f40273324b774bb4 | 768f1cb2cecf83c9f136d12df7d66afe32347e7d |
| Comedy Short Video Skill | de6096c5e452d74c36a691e80aacb3c255aa20cc | 87f65f887194073c79fbe72caa60ef6b2ebce715 |
| Stephen Chow Skill | 7fbd2daa020fb55d0dc3d2ec626d7f8718e55b03 | 166e9cda1006f3b2e50684a55626723484b3ed41 |
| Creative Director | ac06eb255371da35d1db0a146dbfcca8c8857543 | ca9d3189277d5cd51524672b01d71a0b36f1065c |
| AI ShortVideo Story Engine | 616b46921a7265ddfede48d27ec4eca499e9c8e6 | 71896b00bf769680206fa8053d8bafb4e56cd33c |
| Wind Comic Screenwriter | fc69a03a1e907e28fa3cacecfcd50e8230e5c684 | abfcb603be01b7c8d5c3d0b55b9edef643b859fd |
| Storytelling Frameworks | b252ccb17b3a3b870a72a4598eafc15839e47568 | d11c2041e17a4994e33b2ed7a5ff433f5cb4f711 |

### 4.2 初步内容观察与适配风险

#### 1. Story Idea Generator

实际入口采用 genre-first 方法：先确定观众希望获得的情绪体验，再决定题材设置、人物和情节；包含 11 种情绪/类型方向，并引用多个 setting modules 和 implementation guide。

**候选价值**：A1 创意种子生成、情绪承诺、题材与故事元素关联。

**适配风险**：入口 Skill 引用多个支持文件，不能只复制单个 SKILL.md；需检查各模块是否都在同一许可证范围内。仓库根未发现 LICENSE，虽然 frontmatter 声明 MIT，仍应确认声明适用范围后再批量导入。

#### 2. Short Film Development

实际入口从原始点子发展到 logline、premise、theme、character engine、conflict、structure、treatment、scene list 和 revision plan；提供多条差异化方向、短片结构选择与可拍性检查。

**候选价值**：A2/A3 的完整短片开发主流程，尤其是冲突、人物目标、结构、场次目的和结局回收。

**适配风险**：其产物是通用开发资料，需映射到 CinePilot A2 Concept 与 A3 Story Artifact，不得把外部 Skill 的场次列表直接当成 A4 正式剧本。

#### 3. Comedy Short Video Skill

实际入口覆盖选题与创意发散、喜剧人物、故事节拍、喜剧机制、台词与可拍脚本、脚本诊断、原创性与安全边界；提供模板和 Node.js 辅助工具。

**候选价值**：作为题材条件插件，覆盖人物反差、笑点机制、铺垫/升级/反转/回收和喜剧脚本诊断。

**适配风险**：辅助工具依赖 Node.js 22+，需逐项审查是否真的需要；README 明确该仓库许可证不包含其提到的第三方课程内容，禁止复制课程材料或误导为官方合作。

#### 4. Stephen Chow Skill

实际入口将相关作品作为评论/研究对象，强调通过可分析的高层喜剧机制创作原创内容；包括需求路由、创作工作流、原创性审查和权利安全检查。

**候选价值**：喜剧题材的荒诞机制、规则碰撞、地位反转、视觉升级和情绪回落等方法研究。

**强制适配要求**：不得模仿现实创作者的独特个人风格，不得复刻其受保护角色、故事、标志性台词、声音、肖像或具体场景；将用户的风格要求转化为通用、非专属的喜剧机制后独立创作。保留原 Skill 的 rights/safety gate。

#### 5. Creative Director

实际项目面向广告和 Campaign 创意：brief intake、insight、ideation、evaluation/refine、articulate，使用多种创意方法、评分机制和案例库。

**候选价值**：A1/A2 的创意方法论、洞察提炼、跨领域联想、创意比较和反向评审。

**适配风险**：
- 它不是影视导演 Skill，不能作为 B1 主 Skill；
- 其案例库规模在 README 与 Skill 元数据中存在 569/571 的数量差异，且 README 自述部分案例来源 URL 未验证；
- 不应把整个广告案例库自动复制进 CinePilot；先评估方法论，第三方案例/图片/来源逐项审核；
- CC BY 4.0 要求署名、许可证链接和修改说明，保留 NOTICE，不暗示原作者背书。

#### 6. AI ShortVideo Story Engine

本轮定位到的核心 Skill 是 douyin-script，针对“人生副本”类抖音/短视频故事：第二人称沉浸、普通人困境、人生逆袭/商业反转、黄金 3 秒钩子、故事先导、5–8 秒镜头节奏、漫画视觉规范、剪辑与声画对齐。仓库还包含 explain-commercial-models，用于商业模式和商业案例题材。

**候选价值**：A3 的钩子/压力阶梯/反转回收方法，以及 B3 的镜头节奏和视觉化提示参考。

**适配风险**：这是明显带有账号定位和题材偏好的专项 Skill，不是通用短片剧本引擎。其 16:9、漫画风、第二人称、5–8 秒镜头节奏不能成为 CinePilot 所有作品的全局默认；必须改成可配置规则。其镜头/提示词输出也不能绕过 A4/B1/B2/B3 的阶段边界。

#### 7. Wind Comic Screenwriter

实际入口采用 McKee 三幕结构、Story Bible、角色 Voice Fingerprints、场次预算、两阶段生成（先规划后 JSON）和可选 Critic-Rewrite Loop；输出结构中同时包含 scenes 与 shots。

**候选价值**：A4 剧本结构、角色事实一致性、对白声音区分、场次预算、分步生成和定向批评修订；B3 可研究其镜头预算与结构化输出方式。

**适配风险**：
- 它所在仓库是一个较大的完整应用项目，不等于只有一个独立 Skill；
- 入口描述涉及仓库内辅助代码/模块，需要先梳理实际依赖，不能只复制 SKILL.md 就假设可运行；
- 其 Script JSON 同时包含 scenes 与 shots，而 CinePilot 明确将 A4 Screenplay、B2 Breakdown、B3 Shot Plan 分开；必须适配输出，不允许一个 A4 Run 直接发布 B3 镜头成果。

#### 8. Storytelling Frameworks

实际入口提供六种短视频叙事框架：Big Goal/Dream、Challenge、Breakthrough、About Me、Hero's Journey、Man in a Hole；强调按故事类型选择框架、访谈填充真实信息和按节拍组织口播脚本。

**候选价值**：A2/A3 的结构参考，尤其是个人经历、品牌故事、客户案例等内容。

**适配风险**：它主要针对真实经历、个人品牌和社交媒体口播故事，并非通用影视虚构剧情结构；只能在题材匹配时作为辅助参考。仓库没有发现 LICENSE 文件，也没有 Skill frontmatter License 声明；在取得许可前不得复制其 SKILL.md、references 或模板。可以另行设计 CinePilot 自己的叙事结构工具，但应避免照搬原文表达和模板。

---

## 5. A1–B4 阶段 Skill Profile 设计

### 5.1 每个阶段的 Skill 角色

每个阶段都拥有独立 Profile，至少可以配置：

| 配置项 | 说明 |
|---|---|
| Primary Skill | 本阶段主要生成方法；允许暂时为空并标记能力缺口 |
| Supporting Skills | 提供指定的辅助方法 |
| Conditional Skills | 由题材、风格、任务意图或用户明确选择触发 |
| Review Skills | 对本阶段草稿进行结构/质量检查 |
| Execution Order | 指定先后顺序和角色 |
| Invocation Policy | required / recommended / on_demand / disabled |
| Compatibility | 与项目允许的 Agent Runtime 的兼容验证状态 |
| Output Mapping | Skill 原始输出到 CinePilot Artifact 合同的映射 |
| Version Pins | 每个 Skill 固定使用的 CinePilot Skill Version |

### 5.2 第一批候选的阶段映射

> 下表是推荐候选，不是用户逐项确认的最终绑定，也不代表候选已经导入或通过质量测试。其他阶段缺失 Skill 由用户后续补充。

| 阶段 | 主 Skill 候选 | 辅助 Skill 候选 | 条件/题材插件 | 当前判断 |
|---|---|---|---|---|
| A1 点子发现 | Story Idea Generator | Creative Director；Storytelling Frameworks（仅题材匹配时） | 用户后续补充题材 Skill | 有第一批主候选 |
| A2 创意发散 | Short Film Development | Creative Director；Storytelling Frameworks（仅真实经历/品牌故事等适用） | Comedy Short Video；Stephen Chow | 有主候选与喜剧插件 |
| A3 故事深化 | Short Film Development | AI ShortVideo Story Engine；Storytelling Frameworks（仅适用题材） | Comedy Short Video；Stephen Chow | 有主候选；需适配短片与题材 |
| A4 文学剧本 | **待补充通用 Screenwriter** | Wind Comic Screenwriter（待评估）；Comedy Short Video（条件） | 题材 Skill 后续补充 | 通用主 Skill 缺失 |
| B1 导演阐述与视听设计 | **待用户补充** | 暂不指定 | 暂不指定 | 缺失，不以 Creative Director 替代 |
| B2 制作拆解 | **待用户补充** | 暂不指定 | 暂不指定 | 缺失 |
| B3 分镜与镜头规划 | **待用户补充通用 Shot Designer** | AI ShortVideo Story Engine；Wind Comic Screenwriter（均待评估） | Comedy Short Video（仅喜剧镜头节奏辅助） | 候选有重叠，需验证阶段边界 |
| B4 视频生成提示词 | **待用户补充** | 暂不指定 | 未来模型适配插件 | 缺失 |

### 5.3 阶段边界必须保持

- A1/A2 管项目级 Idea/Concept；选定 Concept 并决定开发后创建 Episode。
- A3 负责故事结构和故事正文，不等同文学剧本。
- A4 负责文学剧本与 Scene，不应提前生成正式 Shot Plan。
- B1 负责导演阐述和视听原则。
- B2 负责 Scene → Segment/Subsegment 制作拆解和资产需求。
- B3 负责叶级片段下的 Shot 规划。
- B4 负责基于已定稿 Shot 的视频生成提示词。
- 外部 Skill 若横跨多个阶段，必须拆分其能力模块并映射到正确阶段，不能因为它能输出整套全案就绕过 CinePilot 阶段审批和产物版本合同。

---

## 6. Profile 作用域、继承和覆盖

### 6.1 四级配置

推荐配置优先级（高到低）：

~~~text
单次任务 Skill Override
        ↓
分集 + 阶段 Skill Override
        ↓
项目 + 阶段 Stage Skill Profile
        ↓
系统推荐 Stage Skill Template
~~~

项目阶段配置是用户主要维护入口。系统模板提供默认建议；分集覆盖用于特殊题材；任务覆盖用于一次性修改。

### 6.2 继承和合并规则

- 系统模板提供阶段默认角色、Skill 候选和调用策略；
- 项目阶段 Profile 复制/继承模板后，项目可以自行调整；
- 分集覆盖只影响当前 Episode 的指定阶段，不影响同项目其他分集；
- 单次任务覆盖只影响本次 Run，不回写项目配置；
- required Skill 默认不可被低层覆盖静默移除；确需禁用时必须显式确认并显示阶段能力缺口；
- 同一 Skill 的不同版本不能在同一个 Run 中以同一 skill_id 重复加载；
- 版本升级只影响新 Run；历史 Run 固定原 Skill 版本。

### 6.3 Profile 建议字段

- profile_id、scope_type、scope_id、stage_code、profile_version；
- primary_skill_version_id（允许为空，但需显示缺失原因）；
- supporting_skill_items；
- conditional_skill_rules；
- review_skill_items；
- execution_order；
- invocation_policy；
- output_contract_mapping；
- compatibility_constraints；
- enabled / archived 状态；
- 变更人、变更时间和配置来源。

### 6.4 题材标签

题材标签用于条件 Skill 候选过滤，不等于必须自动启用某个 Skill。

例如 comedy、mystery、action、romance、sci-fi、slice-of-life 等标签由项目/分集的创作元数据提供；用户明确选择优先于模型推断。题材未确定时不加载风格插件，避免创作方向被过早锁死。

---

## 7. Skill Router 与 Skill Plan

### 7.1 Router 不是一个只靠 Prompt 的入口 Skill

Skill Router 是 Python 编排层中的规则化解析能力，可调用轻量意图分类模型辅助识别，但最终选择必须受系统配置和代码约束。

输入：
- Project / Idea / Concept / Episode / Stage；
- 当前任务类型（探索、生成、续写、定点修改、审查、整理、导出）；
- 用户明确指定的 Skill；
- 目标对象范围（Artifact / Scene / Segment / Shot）；
- 项目与分集题材/风格标签；
- Stage Skill Profile；
- Agent Runtime 的 Skill 兼容能力；
- 当前阶段产物合同和上游 Revision。

### 7.2 路由步骤

1. 从 UI 和 Run 请求确定项目、阶段、对象范围，不让模型猜项目身份。
2. 检查阶段准入和允许的输入版本。
3. 解析系统模板、项目 Profile、分集覆盖和任务覆盖。
4. 筛选项目已启用、版本固定、许可证/安全审核通过的 Skill。
5. 根据任务类型和题材条件决定必选、推荐、按需和禁用项。
6. 检查所选 Agent Runtime 是否能发现和运行这些 Skill；不兼容时移除候选或阻止任务，不静默假设兼容。
7. 生成 Skill Plan，说明每个 Skill 的角色、版本、调用理由、输入和预期贡献。
8. 将主 Skill 的必要指令与当前任务相关的辅助资料按需加载；不把所有 Skill references 一次性装进上下文。
9. Agent 执行并提交草稿。
10. ArtifactService 按 CinePilot 合同校验输出；Review Skill 可以提供诊断，但最终是否修改/定稿由用户决定。

### 7.3 Skill Plan 示例

~~~yaml
run_id: RUN_EP03_A3_001
stage: A3
task_type: revise_story_turn
scope:
  artifact_id: STORY_EP03
  section_id: midpoint
profile_version: SSP_A3_V4
skills:
  - skill_id: short_film_development
    version: 0.1.0
    role: primary
    invocation: required
    reason: "A3 故事深化主流程"
  - skill_id: storytelling_frameworks
    version: 0.1.0
    role: supporting
    invocation: on_demand
    reason: "仅当故事类型与六种框架适配时使用"
  - skill_id: creating_comedy_short_videos
    version: 0.1.0
    role: conditional
    invocation: disabled
    reason: "当前作品未标记为喜剧"
~~~

示例中的 CinePilot 版本号仅表示字段形态，不表示这些 Skill 已导入。

### 7.4 用户可见性与覆盖

- 普通任务可默认按 Profile 自动解析，不要求用户每次手动选择所有 Skill。
- 工作台提供“本次使用的 Skill”预览，显示主/辅/条件 Skill 和原因。
- 用户可以在当前 Run 启动前添加、移除可选 Skill 或选择其他兼容 Skill。
- 用户明确指定某 Skill 时优先考虑，但仍需检查项目启用、License、安全和 Runtime 兼容。
- Run 启动后 Skill Plan 固定；执行中若需要新增 Skill，必须创建明确的计划变更事件并记录新版本，不得静默改变。

### 7.5 冲突与缺失处理

优先级：
1. 法律、安全、隐私与项目隔离约束；
2. 用户明确要求与已确认产品约束；
3. CinePilot 阶段合同和 Artifact Schema；
4. 本次任务的主 Skill；
5. 辅助 Skill；
6. 条件题材/风格 Skill；
7. 审查 Skill 的建议。

出现以下情况应停止或降级为普通 Agent 对话并说明原因：
- 阶段缺少必要主 Skill 且没有可接受替代；
- Skill License/安全审核未通过；
- Skill 与 Runtime 不兼容；
- Skill 输出合同无法映射到 CinePilot 产物；
- 多个 Skill 对关键规则存在无法自动解决的冲突。

缺失 Skill 不代表系统不能让用户讨论创意；但正式生成阶段产物时应显示“专业 Skill 配置未完成”，由用户决定是否使用明确标注的通用 Agent 能力继续。

---

## 8. Skill 输入/输出合同与 Handoff

### 8.1 输入合同

Skill 不应自行探索项目全目录。由系统提供：
- 当前项目/阶段/任务/对象范围；
- 必须遵守的用户约束；
- 上游已定稿 Artifact Revision ID；
- 允许读取的资产/媒体 ID；
- 当前阶段输出合同；
- 当前 Skill Plan 与本 Skill 的职责范围。

具体正文、Scene、Segment、Shot、角色资产和媒体由受控工具按需读取。

### 8.2 输出合同

外部 Skill 可以有自己的 Markdown、JSON 或脚本输出；CinePilot 不直接把外部输出当作正式产物。

需要定义 Adapter Mapping：
- 外部字段 → CinePilot 字段；
- 外部产物 → CinePilot Artifact 类型；
- 外部场次/镜头 ID → CinePilot 稳定对象 ID；
- 缺失字段、额外字段和冲突字段处理；
- 校验错误如何反馈给 Agent；
- 不允许 Skill 改写上游 Revision 或绕过 StageApproval。

例如 Wind Comic Screenwriter 的输出包含 scenes 与 shots；CinePilot 应将 A4 的 screenplay/scenes 与 B3 的 shots 分别处理，不允许一个 A4 Run 直接发布 B3 镜头成果。

### 8.3 按需加载，控制上下文

- 先加载 Skill 入口与本次任务必需的最小规则；
- references 按 Skill 内路由指示和任务需要单独读取；
- 脚本/案例库只在当前任务确实需要时使用；
- 对大案例库优先使用受控索引和少量相关案例，不整库加载；
- Run 记录实际读取的 Skill 文件和参考资料版本；
- 下一阶段不默认继承上一阶段所有 Skill 上下文，只按新阶段 Profile 重新解析。

---

## 9. 评测、版本升级和质量治理

### 9.1 Skill 导入评估表

每个候选都要评估：

| 维度 | 核心问题 |
|---|---|
| 创作质量 | 是否能产生有差异、具体、可拍摄的短视频内容？ |
| 适用范围 | 是否适用于独立短视频，而不是长剧/品牌口播/广告 Campaign？ |
| 阶段匹配 | 是否能清晰映射 A1–B4 某个阶段？ |
| 方法透明度 | 是否有清楚的流程、选择规则和质量检查？ |
| 结构输出 | 能否稳定映射 CinePilot Markdown/JSON 合同？ |
| 上下文效率 | 是否能按需读取，避免冗长重复 Prompt？ |
| Runtime 兼容 | 能否在目标 Agent Runtime 正常发现、加载和调用？ |
| 安全性 | 是否运行脚本、访问网络、写入文件或调用外部工具？ |
| License | 是否允许计划中的复制、改造和分发？ |
| 维护成本 | 依赖、案例库和上游升级是否可控？ |

### 9.2 回归测试

每个启用 Skill 至少有：
- 代表性任务测试集；
- 输入样例和固定上游 Revision；
- 预期产物结构和 Schema；
- 质量评分维度（创意差异、因果、人物动机、冲突升级、反转回收、可拍性、角色连续性等按阶段选择）；
- 不允许发生的越界行为；
- Token/上下文消耗基线；
- Runtime/Model 组合记录。

升级 Skill 或 Model 时使用同一测试集对比，不能只凭一次主观体验宣布更好。

### 9.3 版本升级

- 上游 Commit 变化只产生新的待评估来源快照；
- CinePilot 适配版本有自己的语义版本；
- 新版本通过 License、安全、结构、质量和 Token 回归后，才可启用；
- 项目 Profile 可固定旧版本或显式升级；
- 历史 Run 始终保留原 Skill 版本和文件摘要；
- 已定稿产物不因 Skill 更新自动重做。

---

## 10. 页面与管理功能

### 10.1 系统设置 → Skill Registry

列表展示：
- Skill 名称、类型、阶段适用范围；
- 来源仓库、上游 Commit；
- License 状态、作者/NOTICE；
- CinePilot 版本；
- 安全/兼容/评测状态；
- Candidate/Reviewing/Adapted/Enabled/Deprecated 状态。

操作：登记来源、查看来源详情、检查 License、导入候选、建立改造版本、运行测试、启用/停用、查看历史版本。

### 10.2 Skill 详情与来源审计

展示：
- 原始仓库与 Skill 文件路径；
- 原始作者、License、NOTICE；
- 固定 Commit 和文件 SHA；
- 实际导入文件清单与依赖；
- CinePilot 改造说明、差异和署名；
- 适用阶段、输入/输出合同；
- Runtime 兼容测试；
- 回归评测历史。

### 10.3 项目设置 → Stage Skill Profiles

显示 A1–B4 八个阶段，每个阶段可设置：
- 主 Skill；
- 辅助 Skill；
- 条件/题材 Skill 及触发条件；
- Review Skill；
- Skill 版本；
- 执行顺序和调用策略；
- 当前缺失能力提示。

支持从系统推荐模板初始化、项目级复制和自定义；不同项目的配置互不影响。

### 10.4 分集 → 阶段 Skill Override

允许某一分集的特定阶段增加或替换可选 Skill。例如某集是喜剧，用户可以给该集 A3/A4/B3 增加喜剧插件；不改变项目其他分集。

### 10.5 Agent 工作台 → Skill Plan 预览

在 Run 启动前显示：
- 本次主 Skill；
- 辅助 Skill；
- 条件 Skill 是否触发及原因；
- Review Skill；
- Skill 版本和兼容状态；
- 用户临时覆盖入口；
- 缺失或冲突告警。

Run 启动后，在任务详情中保留不可变 Skill Plan 快照。

---

## 11. V1 实施边界

### V1 必须

1. 系统 Skill Registry 元数据和来源追溯；
2. 外部 Skill 候选登记与 License 状态；
3. 项目启用 Skill 与版本固定；
4. 八个阶段的 Stage Skill Profile 数据结构，即使某些阶段暂时为空；
5. 项目阶段配置与分集/任务覆盖；
6. Skill Router 规则化解析；
7. Skill Plan 预览和 AgentRun 快照；
8. 当前项目的只读 Skill 投射；
9. 至少一个真实 Runtime 的 Skill 加载验证；
10. 输出适配、Artifact Schema 校验和阶段合同检查；
11. 按需读取 references，避免全量加载。

### 后续扩展

- 自动化 Skill 质量基准和多模型对比；
- 更丰富的题材 Skill 市场/导入体验；
- Skill 自动推荐与学习型路由；
- 多 Skill 并行协作；
- 远端 Skill Registry 镜像；
- 自动上游升级 PR。

**当前不做**：自动导入全部八个仓库、自动执行所有外部脚本、自动启用所有题材 Skill、让外部 Skill 直接写入正式产物。

---

## 12. 当前候选与待用户补充事项

- 八个候选来源已登记并定位到入口文件，但尚未逐项完成深度质量评测和 License 范围审查。
- A1/A2/A3 有第一批候选；A4 有 Wind Comic Screenwriter 待评估，但通用主 Screenwriter Skill 仍待确认。
- B1/B2/B4 由用户后续补充 Skill 来源；当前不擅自选择外部仓库。
- B3 的 AI ShortVideo Story Engine / Wind Comic 只作为待评估候选，必须先解决与 B2/A4 阶段合同的重叠。
- Storytelling Frameworks 目前未找到 License 声明；不得复制其文本和模板。
- Creative Director 的 CC BY 4.0 署名、NOTICE 和案例库第三方来源需要单独审核。

本规格和 [Agent 接入与模型管理规格](./agent-integration-and-model-management.md) 一起，为 [Agent/Skill 运行时协同规格](./agent-skill-orchestration-and-runtime.md)提供输入；开发任务拆解见 [开发启动路线图](./development-kickoff-and-implementation-roadmap.md)。