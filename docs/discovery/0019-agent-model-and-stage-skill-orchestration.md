# Discovery 0019 — Agent/Model 管理与 A1–B4 阶段 Skill 编排

> 日期：2026-10-30
> 状态：**用户已确认补齐 Agent 接入/模型管理和阶段 Skill 编排设计；本记录与相关 Spec 已落档。** 本记录中的配置结构、路由细节和开发顺序是推荐方案；具体 Agent Runtime、模型、Skill 导入和技术框架仍需在实现时验证。

## 1. 用户确认的方向

1. 系统需要独立的 Agent 接入配置：用户选择使用哪个 Agent Runtime，并选择该 Agent 实际支持的模型。
2. 模型选择不能硬编码成所有 Agent 共用的固定列表；应根据 Agent Adapter 检测到的模型目录、参数和能力动态呈现。
3. 每个 A1–B4 阶段都需要 Skill 配置能力；不能采用一阶段固定一个 Skill 的一对一关系。
4. 一个阶段可组合主 Skill、辅助 Skill、条件/题材 Skill 和审查 Skill；一个 Skill 可跨阶段复用。
5. 用户最初提供的 8 个创意/编剧 Skill 是第一批研究来源；其他阶段缺失 Skill 由用户后续补充，不由系统设计者擅自补全。
6. 用户希望设计文档补齐后，能够使用 Codex 开始工程开发；无需等待所有外部 Skill 都完成导入，也无需先完成全部高保真 UIUX。

## 2. 已确认架构约束保持不变

- Vue + Python + Electron；
- 全系统共用一个 SQLite，项目内容分别存储为 Markdown / JSON / 媒体；
- 新建项目即建立独立 Agent 工作区，Agent 不可访问其他项目、全局 SQLite 和系统凭据；
- V1 CLI 生态优先，官方 Agent SDK 可用时优先评估；
- A1/A2 是项目级 Idea/Concept；选定 Concept 后才创建 Episode；
- 每个 Episode 是一条独立短视频；
- A3–B4 是分集正式阶段；Scene 与 Segment 独立映射，Segment 最多一层 Subsegment；
- 正式审批以阶段整体为单位，局部 Scene/Segment/Shot 可编辑；
- 定稿不可覆盖；上游变化只标记下游待复核，不自动重做；
- Handoff 精简并按需读取；
- V1 不接视频生成 API。

## 3. 新增设计规格

- [Agent 接入与模型管理](../spec/agent-integration-and-model-management.md)：Agent Registry、Connection Profile、Adapter、动态模型发现、Agent Model Profile、配置继承、凭据和隔离、Run Snapshot。
- [Skill Registry 与阶段编排](../spec/stage-skill-orchestration-and-registry.md)：Skill 来源治理、License/Provenance、八个候选登记、A1–B4 Stage Skill Profile、项目/分集/任务覆盖、Skill Router 和 Skill Plan。
- [开发启动与实施路线图](../spec/development-kickoff-and-implementation-roadmap.md)：Codex 可执行的 M0–M9 里程碑、第一条垂直切片、验收和非 V1 前置事项。

## 4. 八个外部 Skill 来源的 GitHub 核验结果

核验日期：2026-10-30。通过 GitHub 接口确认八个仓库存在，并找到对应 Skill 入口文件。Commit SHA 为核验时 main 分支 HEAD；Skill SHA 为入口文件 Blob SHA。该结果只证明仓库/路径存在和文件可读，不等于完成质量评估或批准导入。

| Skill | 仓库与入口文件 | License 观察 | 核验时 main Commit |
|---|---|---|---|
| Story Idea Generator | [jwynia/agent-skills](https://github.com/jwynia/agent-skills/tree/main/skills/creative/fiction/core/story-idea-generator) · skills/creative/fiction/core/story-idea-generator/SKILL.md | SKILL.md frontmatter 声明 MIT；根目录未发现 LICENSE，范围待确认 | e02ec7e226a6e4f8419fd3b88a1d8e472d421b32 |
| Short Film Development | [0xhughs/director-skills](https://github.com/0xhughs/director-skills/tree/main/skills/short-film-development) · skills/short-film-development/SKILL.md | 根目录 MIT | 35ca0a2b4cd55b0668aa9ea0f40273324b774bb4 |
| Comedy Short Video Skill | [heran1225-byte/comedy-short-video-skill](https://github.com/heran1225-byte/comedy-short-video-skill/tree/main/skills/creating-comedy-short-videos) · skills/creating-comedy-short-videos/SKILL.md | 根目录 MIT；README 说明许可证不包含第三方课程材料 | de6096c5e452d74c36a691e80aacb3c255aa20cc |
| Stephen Chow Skill | [chnjames/stephen-chow-skill](https://github.com/chnjames/stephen-chow-skill) · SKILL.md | 根目录 MIT | 7fbd2daa020fb55d0dc3d2ec626d7f8718e55b03 |
| Creative Director | [smixs/creative-director-skill](https://github.com/smixs/creative-director-skill/tree/main/creative-director) · creative-director/SKILL.md | 根目录 CC BY 4.0；有 NOTICE，署名和修改说明必须保留 | ac06eb255371da35d1db0a146dbfcca8c8857543 |
| AI ShortVideo Story Engine | [slad-dev/AI-ShortVideo-Story-Engine](https://github.com/slad-dev/AI-ShortVideo-Story-Engine/tree/main/skills/douyin-script) · skills/douyin-script/SKILL.md | 根目录 MIT | 616b46921a7265ddfede48d27ec4eca499e9c8e6 |
| Wind Comic Screenwriter | [ChrisChen667788/wind-comic](https://github.com/ChrisChen667788/wind-comic/tree/main/skills/screenwriter) · skills/screenwriter/SKILL.md | 根目录 MIT | fc69a03a1e907e28fa3cacecfcd50e8230e5c684 |
| Storytelling Frameworks | [advisorita/storytelling-frameworks](https://github.com/advisorita/storytelling-frameworks) · SKILL.md | 根目录未发现 LICENSE，Skill frontmatter 未声明 License；不得复制文件/模板，除非获得许可 | b252ccb17b3a3b870a72a4598eafc15839e47568 |

对应入口文件 SHA 和详细适配风险记录在 [Skill Registry 规格 §4](../spec/stage-skill-orchestration-and-registry.md)。

### 4.1 内容适配观察

- Creative Director 主要是广告/Campaign 创意流程，不等于影视导演 Skill；README 与 Skill 元数据中的案例库数量存在 569/571 差异，且部分案例 URL 未验证。
- AI ShortVideo Story Engine 的 douyin-script 偏“人生副本”账号、第二人称、漫画视觉、黄金钩子和特定镜头节奏，不是所有独立短片的通用默认规范。
- Storytelling Frameworks 主要针对个人经历、品牌故事和社交口播，不等同通用影视虚构故事结构。
- Wind Comic Screenwriter 输出同时包含 scenes 与 shots；CinePilot 必须拆分 A4/B3，不能让外部 Skill 绕过阶段边界。
- Stephen Chow Skill 强调基于高层喜剧机制创作原创内容；不得模仿创作者个人风格或复刻受保护表达。
- Story Idea Generator 的 Skill frontmatter 声明 MIT，但仓库根无 LICENSE；需要确认声明覆盖范围。
- Comedy Short Video README 明确第三方课程内容不在其 MIT 授权范围内。

## 5. 推荐阶段映射（候选，非最终绑定）

| 阶段 | 主候选 | 辅助/条件候选 | 状态 |
|---|---|---|---|
| A1 | Story Idea Generator | Creative Director；Storytelling Frameworks 仅适用题材 | 候选 |
| A2 | Short Film Development | Creative Director；Storytelling Frameworks；Comedy/Stephen Chow 条件加载 | 候选 |
| A3 | Short Film Development | AI ShortVideo Story Engine；Storytelling Frameworks；喜剧插件按题材 | 候选 |
| A4 | 通用 Screenwriter 待补充 | Wind Comic Screenwriter 待评估；Comedy 条件插件 | 能力缺口 |
| B1 | 用户后续补充 | 不指定 | 待补充 |
| B2 | 用户后续补充 | 不指定 | 待补充 |
| B3 | 通用 Shot Designer 待补充 | AI ShortVideo Story Engine / Wind Comic 待评估 | 能力缺口 |
| B4 | 用户后续补充 | 不指定 | 待补充 |

## 6. 开发实施建议

按 [开发启动与实施路线图](../spec/development-kickoff-and-implementation-roadmap.md) 推进：

1. M0 工程骨架和项目沙箱技术 Spike；
2. M1 Project / Workspace / SQLite；
3. M2 Idea / Concept / Episode；
4. M3 Artifact / Revision / StageApproval；
5. M4 Agent Adapter / Model Profile / 安全隔离；
6. M5 Skill Registry / Stage Skill Profile / Skill Router；
7. M6 A1/A2 创意编剧闭环；
8. M7 A3/A4；
9. M8 B1–B4；
10. M9 资产、审查、批量和导出完善。

其他阶段缺失 Skill 不阻塞工程骨架、数据服务、Agent Adapter 和 Profile 机制的开发。

## 7. 尚未完成的工作

- 对八个候选 Skill 做完整依赖、脚本安全、License 范围和输出质量评测；
- 用户后续提供 A4 通用编剧、B1 导演、B2 制作拆解、B3 通用分镜、B4 视频提示词等缺失 Skill 来源；
- 选择首个实际 Agent Runtime 并完成 Windows/Electron 项目沙箱安全验证；
- 在 M0 技术 Spike 中确定 Vue/Python/Electron 具体框架和服务启动方式。

本轮不新增 ADR：当前配置层级和 Skill Profile 方案可迭代，尚无必须单独锁定的不可逆技术决策；现有 ADR-0016/0017 已覆盖 CLI 优先和项目隔离原则。
