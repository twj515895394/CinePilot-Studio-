# 第十一轮 Grill：Handoff 最小必传信息与 Agent 按需检索的责任边界

> 日期：2026-10-30  
> 状态：第十轮 B 已确认；本轮候选设计，尚未批准  
> 工作方式：[grill-with-docs](../process/grill-with-docs/SKILL.md)  
> 依据：[第十轮记录](./0009-agent-handoff-payload.md)、[ADR-0009](../adr/0009-approved-upstream-stage-gates.md)、[ADR-0010](../adr/0010-compact-versioned-agent-handoff.md)

## 背景和用户实际痛点

用户已经用过多种跨阶段对话交接方式，仍因历史聊天与文档太长而造成上下文臃肿、Token 增多、Agent 推理或工具交互执行缓慢。用户提出：大型内容存数据库或知识库，Handoff 只留关键信息与查询依据，让 Agent 自行按需检索。当前没有搭建知识库，且第六轮明确拒绝过度设计全局跨项目共享系统。我们不应该为了实现“按需读取”新增不必要的服务。

**第十轮已确认的产品原则**：精简、有定稿版本来源、可回溯、可按需展开的正式交接；不是“只带索引而什么都不告知”。

## 关键区别：确定性事实查询 vs 不确定语义搜索

- **确定性读取**：如“读取 EP07 剧本 v3 的 S03 场次、角色 R02 当前已定稿设定”，通过 ID + 版本可以准确拿到。
- **非确定性检索**：如“以前是否写过类似时间循环的桥段”，才可能用全文或语义搜索；搜索结果只用于发现候选，再核实对象与版本。
- 无论数据库、向量检索还是知识库，**存储形态本身不会自动解决上下文膨胀**；必须控制主动加载内容和每次检索的返回量。

## 压力测试

EP07 的导演 Agent 被启动，剧本 v3 已定稿，共 18 场，角色有 6 个。用户要求先制作 S03 和 S04 对应的导演片段。当前还保存了旧剧本 v1/v2 和大量被否决的角色设定。

Agent 应如何获取数据：

- 必须知道本次目标、EP07、目标阶段、已批准剧本 v3、当前生效的不可违反约束。
- 无需在开始时读取全部 18 场或所有历史版本。
- 需要 S03/S04 时能按 ID + v3 查询指定片段。
- 如果 Agent 未注意到某项强约束，不能靠它“想到去搜”才避免违反设定。

## 三种方案

### A. 完全被动检索：Handoff 只有最小 ID，所有内容由 Agent 自行查询

优点：默认 Prompt 最短；不多取内容。

缺点：Agent 可能不知道该查什么、漏掉强制人物约束、反复查询造成工具调用延迟；查询错误版本时可能输出不符合定稿的结果。尤其是正式阶段推进，这种风险高。

### B. 系统预取必需事实 + Agent 按需读取详细资料（推荐）

- 系统启动阶段时依据**当前阶段合同**，预先加入一小段不可遗漏的任务目标、版本锚点和强制约束摘要，明确允许的查询边界。
- 大体量上游剧本、历史聊天、非当前任务的角色与场次，不默认放进上下文。
- Agent 通过受控查询工具，按项目/分集、稳定 ID 和**具体定稿版本**读取目标内容；例如获取场次摘要、定位角色设定、读取指定已定稿全文或相关章节。
- 查询需要过滤项目/分集、审批状态与版本，返回内容附带来源 ID、版本、命中片段与截断说明；应设置单次和累计长度上限。
- 一项具体任务必要的场次原文可以由系统**按需预取**，而非强迫 Agent 再走一次检索；最小上下文不意味着所有信息都必须走额外工具调用。
- 保留可重建的交接清单，避免中途上游出现新定稿后 Agent 静默读取不同版本。
- 当需要从海量内容中“发现”未知相关资料时，可后续在 SQLite 上增加全文索引；只有准确匹配不够用且有测试证据，才考虑语义检索或独立 KB。

优点：既减少上下文，又降低漏读必需约束、读错版本和多轮反复查询的风险；不依赖独立知识库。

代价：需要阶段合同和几个受控的查询接口；这属于代码编排层合理承担的职责。

### C. 系统一次性收集所有“可能有关”的上下文

优点：Agent 启动后基本无需查询。

缺点：很快退化为用户已经遇到的过多上下文问题。

## 不必先搭建知识库的推荐实现路线（工程建议，非已确认）

```text
User / UI → Stage Orchestrator
            ├─ validate stage + approved versions
            ├─ build compact pinned Handoff
            └─ Agent + professional Skill
                 └─ controlled read tools (project/episode/version bounded)
                      ├─ metadata/relation/version index: SQLite
                      ├─ approved text & structured artifacts: Markdown / JSON
                      └─ optional content search: SQLite FTS5 or other index
```

具体权威归属未决定，不能把数据库索引和文件同时当成同一字段的唯一事实来源。项目数据模型应与检索索引解耦。尤其中文内容，若采用 SQLite FTS5 应评估分词方案；内置 trigram 可以做子串检索但并不等同于高质量中文语义检索。

### 示例交接概要（说明性，而非最终 Schema）

```yaml
project_id: proj-01
episode_id: ep-07
target_stage: directing
approved_inputs:
  screenplay: {artifact_id: script-07, revision: v3}
task_scope:
  scene_ids: [S03, S04]
mandatory_constraints_summary:
  - 保持已确认角色外观一致
  - 尊重用户确认的喜剧风格
lookup_policy:
  project: proj-01
  episode: ep-07
  allowed_status: approved
  pinned_screenplay_revision: v3
```

## 本轮唯一问题

**是否选择 B：系统确保必需的强制约束、任务目标和准确版本锚点在启动前就位；大段文档由 Agent 通过受控接口按需读取，而不是把一切交给 Agent 自己搜索？**

- A. 完全 Agent 主动检索，Handoff 基本只放 ID。
- B. 系统预取必要事实 + Agent 按需检索细节（推荐）。
- C. 由系统预先装载大量关联文档，尽量不查询。

本轮只确认预取与检索的责任边界。FTS5/向量库/外部知识库选择、实际工具调用 API、Schema、Token 预算与存储权威定义留待后续。
