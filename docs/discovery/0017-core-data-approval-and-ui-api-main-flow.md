# 核心数据模型确认与 UI/API 主流程设计衔接

> 日期：2026-10-30
> 状态：已确认核心模型及审核粒度；新 UI/API 细节仍为设计草案

## 用户确认的上下文

上一轮已提交 [核心数据模型与存储映射 v0.1](../spec/core-data-model-and-storage-contracts.md)，涵盖 Project、Idea、Concept、Episode、Artifact/Revision/StageApproval、Scene/Segment/Shot、Asset/Variant/Media、AgentRun/SkillBinding、SQLite 与 MD/JSON 的职责。上一轮给出的产品建议是：「阶段整体定稿、单个场次和镜头可独立编辑；不急于给每个镜头再建单独审批流程」。用户回答「OK 确认 继续」。

因此本次将**核心实体与关系映射作为后续 Python 业务服务的数据模型基线**；承认表名/具体字段/DDL/JSON Schema 仍是实施层推荐示例，不视为全部逐字段确认；**确认审批与正式阶段交接以阶段为单位，局部场次、片段或镜头允许独立编辑和 Agent 定向修改，局部修改进入阶段草稿，而不是创建每对象审批流程**。

## 已落档的决定

- [ADR-0020：阶段整体审批，场次/片段/镜头局部编辑](../adr/0020-stage-level-approval-with-localized-editing.md)。
- 不改变此前 SQLite 单库、项目物理目录、不可覆盖 Revision、精简 Handoff、项目 Agent 沙箱或横向资产约束。

## 本轮新设计

已建立 [Python 业务接口与 Vue/Electron 工作台 v0.1](../spec/python-api-and-vue-electron-workbench.md)，按真实用户路径展开：
1. 新建项目与独立目录、选择 CLI/SDK Runtime 及启用 Skill；
2. 项目级创意池 Idea/Concept 与选方向创建 Episode；
3. A3–B4 六个分集级阶段的中栏产物、左侧导航、右侧 Agent；
4. Agent Run 的受控查询/事件流/草稿提交与实时 UI 可见；
5. Scene/Segment/Shot 定点编辑、StageApproval 定稿、依赖待复核；
6. 项目资产、多视图媒体、分集变体、B4 提示词准备包；
7. Python 服务模块划分、接口候选、V1 验收与开发顺序。

**边界**：前端三栏界面、具体 HTTP 路由、SSE/WebSocket 选型、Electron+Python 宿主、首发 Codex/Claude 运行时和 V1 分镜图片处理方式等仍是推荐设计，未得到逐项批准。不再为普通异常情况开设大量小轮 Grill。

## 下一项值得集中评审的产品问题

**Electron 桌面运行形态与 Web 部署的关联**会影响 CLI 位置和项目工作区的物理可见范围：桌面版直接随应用启动本地 Python/Agent Host，还是连接用户部署的远端 Python；Web 则由服务器端受隔离 Agent Host 执行，不能访问浏览器端用户计算机的本地 CLI。建议在 V1 技术架构与发布设计中一次性提出完整推荐，之后再确认，不拆成很多微小问答。
