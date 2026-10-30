# CinePilot Studio — Agent 接入与模型管理规格

> 版本：0.1
> 日期：2026-10-30
> 状态：**产品需求已确认：需要独立 Agent 接入配置，并能选择 Agent 支持的模型；CLI 优先、官方 Agent SDK 可用时优先评估已确认。** 本文的领域对象、配置层级、页面和 Adapter 合同为推荐设计，具体首个 Runtime、SDK、沙箱实现尚未锁定。
> 关联：[整体产品流程](./overall-product-workflow-and-system-map.md) · [Agent/Skill 运行时协同](./agent-skill-orchestration-and-runtime.md) · [Skill Registry 与阶段编排](./stage-skill-orchestration-and-registry.md) · [产品页面功能](./product-information-architecture-and-page-functions.md) · [核心数据模型](./core-data-model-and-storage-contracts.md) · [项目隔离工作区](./project-isolated-agent-workspaces-and-skills.md) · [ADR-0016](../adr/0016-cli-first-with-official-agent-sdk.md) · [ADR-0017](../adr/0017-project-isolated-agent-workspaces-and-skill-projection.md)

---

## 0. 目标与边界

建立一个系统级 Agent 接入和模型配置中心，让用户能够：

1. 接入/检测本机或受控远端的 Agent Runtime；
2. 查看该 Runtime 实际支持的能力和可用模型；
3. 为 Agent 选择其支持的模型与推理/运行参数；
4. 将 Agent + Model + 参数保存为可复用的执行配置；
5. 在系统、项目、阶段和单次任务层级选择或覆盖执行配置；
6. 在每次 Agent Run 中追溯实际使用的 Agent、版本、模型、参数、Skill 和输入版本。

本模块不负责创作方法和 Skill 组合；Skill 的安装、版本、阶段绑定由 [Skill Registry 与阶段编排规格](./stage-skill-orchestration-and-registry.md)管理。

### 0.1 已确认的约束

- 技术栈 Vue + Python + Electron；Web/Electron 共用 Python 业务服务。
- V1 优先接入 CLI 生态；存在官方 Agent SDK 时，优先评估其作为程序控制接口。官方 Agent SDK 不等于直接模型 API。
- 不预先建设所有厂商的直接模型 API；API Provider 直连属于后续扩展。
- 每个项目有独立 Agent 工作区；Agent 只能访问当前项目授权内容，不得访问其他项目目录、系统级 SQLite、全局 Skill 源目录或系统凭据。
- Agent 不直接修改正式产物；所有内容通过受控工具和 ArtifactService 保存草稿。
- Agent/Model 配置与 Skill 配置分离；Skill 不绑定特定 Agent 或模型。

### 0.2 本规格的核心原则

**先选 Agent Runtime，再从该 Runtime 实际支持的模型和参数中选择。** 不建立一个假设所有 Agent 都支持相同模型的全局硬编码下拉框。

某些 Runtime 可以动态列出模型、切换模型和推理等级；另一些可能只支持自己的默认模型，或只能通过配置文件选择。界面必须忠实展示 Adapter 检测结果，不能伪造统一能力。

---

## 1. 核心概念与职责边界

| 概念 | 定义 | 示例/说明 |
|---|---|---|
| Agent Runtime | 负责执行 Agent 任务的产品/运行时 | Codex CLI、Claude Code、Gemini CLI 等候选 |
| Agent Adapter | Python 侧对某个 Runtime 的统一控制适配器 | 检测、启动、流式事件、取消、工具桥接 |
| Connection Profile | 某个 Runtime 的安装位置、运行宿主、认证引用和连接状态 | 本机 CLI、受控远端 Agent Host |
| Model Provider | 为 Runtime 提供模型服务的提供方或订阅/网关来源 | 可能是 Runtime 自带账户，也可能是未来的自定义 API Provider |
| Model Catalog Entry | Adapter 从 Runtime 发现或验证的模型条目 | 使用 Runtime 返回的准确 model_id，不自行推测 |
| Agent Model Profile | 将某个 Runtime Connection、模型选择及其支持的参数组合成可复用配置 | Codex + Runtime 可用模型 ID + 支持的推理等级 |
| Execution Policy | 系统/项目/阶段/任务选择 Agent Model Profile 的规则 | 项目默认、阶段覆盖、单次覆盖 |
| Agent Run Snapshot | 某次运行时实际解析出的 Runtime、模型、参数、Skill 与输入版本快照 | 供回溯、复现和审计 |

**不要混淆：**

- Agent Runtime 是执行器，不等于模型；
- Model Provider 是模型来源，不等于 Agent；
- Model 是 Runtime 可识别的具体模型 ID，不等于 Skill；
- Skill 是创作方法和工作流程，不决定使用哪个 Agent 或模型；
- Stage Orchestrator 决定阶段准入和任务范围，Agent 不拥有阶段审批权限。

---

## 2. 系统级 Agent Registry 与 Connection Profile

### 2.1 Agent Registry

系统级注册表描述当前应用认识哪些 Agent Runtime 及其 Adapter：

| 字段 | 说明 |
|---|---|
| runtime_id | 稳定标识，例如 codex_cli |
| display_name | 用户看到的名称 |
| adapter_key | Python Adapter 实现标识 |
| integration_type | cli / official_sdk / remote_agent_host |
| supported_platforms | Windows/macOS/Linux 等实际验证结果 |
| discovery_status | 未检测、可用、不可用、需要认证、能力不完整 |
| detected_version | 最近检测到的 Runtime 版本 |
| capabilities | 最近一次能力发现结果 |
| documentation_url | 官方文档入口 |
| enabled | 系统是否允许项目使用 |

Registry 是应用内适配器清单，不等于把所有 Agent 都安装进应用。

### 2.2 Connection Profile

一个 Runtime 可以有多个 Connection Profile，例如本机不同安装位置或不同受控 Agent Host。

推荐信息：
- Profile 名称、Runtime 类型、执行宿主；
- CLI 可执行文件定位方式（优先自动检测，允许用户校正）；
- Runtime 版本和最后检测时间；
- 认证状态及认证方式（只显示状态，不显示密钥）；
- 当前宿主是否满足项目隔离要求；
- 可用/不可用原因、连接测试结果；
- 默认模型配置引用（若 Runtime 支持）。

连接测试至少检查：可执行程序存在、版本可读、认证状态可识别、能力发现可完成、最小无副作用任务可运行、隔离策略可用。

### 2.3 V1 接入顺序

1. 先实现通用 AgentAdapter Protocol 与一个真实 Runtime 的端到端适配；
2. 首个 Runtime 根据本机安装和隔离能力验证结果选择，不在设计文档中预先假定 Codex 或 Claude 必然先上；
3. 若官方 SDK 能稳定控制会话、工具调用、事件流和取消，优先评估 SDK；否则采用受控 CLI Adapter；
4. 第二种 Runtime 作为验证 Adapter 边界的后续里程碑；
5. 远端 Agent Host 仅保留接口，不作为首个本地 V1 的前置依赖。

---

## 3. Runtime 能力发现与模型目录

### 3.1 Capability Discovery

每个 Adapter 应返回标准化能力描述，而不是由 UI 根据 Runtime 名称猜能力。

建议能力项：

| 能力 | 含义 |
|---|---|
| model_discovery | 是否能动态列出可用模型 |
| explicit_model_selection | 是否支持任务级明确指定模型 |
| runtime_default_model | 是否可使用 Runtime 当前默认模型 |
| reasoning_effort | 是否支持推理强度设置；允许值由 Adapter 返回 |
| verbosity / output_style | 是否支持输出风格参数 |
| temperature / top_p | 仅在 Runtime 真实支持时暴露 |
| image_input | 是否能把媒体像素传给模型，而非仅传路径 |
| tool_calling | 是否支持受控工具调用或桥接 |
| streaming | 是否支持增量消息/事件 |
| cancellation | 是否可取消运行 |
| resume | 是否可恢复会话或任务 |
| skill_discovery | 是否能按约定目录发现 Skill |
| sandbox_controls | 是否有可验证的文件系统/进程隔离能力 |
| structured_output | 是否支持结构化输出约束 |

每个能力都应携带 supported / unsupported / unknown、检测来源和检测时间。unknown 不能当作 supported。

### 3.2 Model Discovery

模型列表按以下顺序获取：

1. Runtime 官方 SDK/API 提供的模型目录；
2. Runtime CLI 提供的模型列表或诊断命令；
3. Runtime 官方配置中可解析的模型条目；
4. 若以上均不支持，则仅提供“使用 Runtime 默认模型”；不自行猜测模型名称。

每个模型条目建议记录：
- Runtime 返回的精确 model_id；
- 展示名称（若有）；
- Provider/来源标识（若可识别）；
- 模型能力（图像输入、上下文能力等，只有来源可验证时才填写）；
- 是否当前可用、发现来源、检测时间；
- Runtime 是否允许用户选择。

模型 ID 是 Runtime 的原样标识，不做自行规范化后再传回 Runtime。

### 3.3 参数 Schema 由 Adapter 声明

模型参数不是全局固定字段。Adapter 返回参数定义、类型、允许值、默认值、作用范围和限制。

例如某个 Runtime 返回 reasoning_effort，另一个 Runtime 可能返回 thinking_budget，还有的完全不支持。UI 只渲染当前 Adapter 声明的参数。

**不支持的参数不展示，不支持的值不提交。** 不要为了统一 UI 而把参数硬编码到所有 Agent。

---

## 4. Agent Model Profile（Agent + Model 组合配置）

### 4.1 配置内容

一个 Agent Model Profile 代表“使用哪个 Runtime，以什么模型/参数执行”：

- profile_id、名称、用途说明；
- connection_profile_id；
- model_selection：明确模型 ID 或 runtime_default；
- 参数对象：仅保存 Adapter 声明且通过校验的参数；
- 可选运行选项：超时、并发策略等由后端运行策略管理，不与模型参数混为一谈；
- 创建/修改时间、Profile 版本；
- 最近验证状态。

示例仅用于表达结构，不代表任何 Agent 的真实模型或参数：

~~~yaml
profile_id: AMP_CODING_HIGH
connection_profile_id: CONN_LOCAL_CODEX
model_selection:
  mode: explicit
  model_id: <adapter-discovered-model-id>
parameters:
  <adapter-declared-parameter>: <validated-value>
~~~

当用户选择 Runtime 默认模型时，保存 mode: runtime_default，运行时仍记录当时实际解析出的模型（如果 Runtime 可报告）。

### 4.2 Profile 与 Agent Runtime 的关系

- 一个 Runtime 可以有多个 Agent Model Profile；
- 一个项目可以引用不同 Profile；
- 阶段可引用不同 Profile；
- Skill 不写入 Agent Model Profile；
- 切换模型不应导致项目 Skill 配置发生变化；
- 删除/禁用某个 Profile 时，历史 Agent Run 快照仍可读。

### 4.3 不支持显式模型选择时

若 Adapter 报告 explicit_model_selection = unsupported：
- UI 显示 Runtime 当前默认模型（若可读取），或“由 Agent 自身配置决定”；
- 不显示虚假的模型下拉框；
- 允许用户使用该 Runtime 默认配置；
- AgentRun 保存 model_selection=runtime_default 和实际可观测信息；
- 不得静默伪造一个模型 ID。

---

## 5. 配置作用域与继承优先级

用户可在不同范围指定执行配置。推荐解析优先级（从高到低）：

~~~text
单次任务显式覆盖（Task Override）
        ↓
分集 + 阶段覆盖（Episode Stage Override）
        ↓
项目 + 阶段配置（Project Stage Route）
        ↓
项目默认 Agent Model Profile（Project Default）
        ↓
系统默认 Agent Model Profile（System Default）
~~~

### 5.1 每层可配置什么

| 作用域 | 可配置内容 | 典型用途 |
|---|---|---|
| 系统级 | 默认 Connection/Profile、允许的 Runtime、全局默认值 | 新项目默认执行环境 |
| 项目级 | 项目默认 Agent Model Profile | 某项目整体偏好 |
| 项目阶段级 | A1–B4 各阶段 Agent Model Profile | A3 用高推理，B4 用快速模型 |
| 分集阶段级 | 单集某阶段覆盖 | 特殊作品/特殊质量要求 |
| 单次任务级 | 本次 Runtime/Profile/参数覆盖 | 临时切换模型或 Agent |

### 5.2 配置解析规则

- 用户明确选择的高优先级配置覆盖低优先级配置；
- 未设置的字段继承下一层；
- 运行前由 Python ExecutionProfileResolver 解析为完整有效配置；
- 所选 Runtime 不可用、模型不再支持或参数无效时，任务应阻止启动并给出原因；
- 不得在未告知用户的情况下自动换成另一个 Agent 或模型；
- 用户可以显式选择“使用 Runtime 默认模型”作为回退方案；
- 变更项目默认值不改变历史 Run 的实际配置快照。

### 5.3 与 Skill 配置解耦

Agent Model Profile 只决定执行器、模型和参数。Stage Skill Profile 只决定专业技能组合。Stage Orchestrator 在任务启动时分别解析两者，再合并成一个 Run Plan。

---

## 6. Agent Run 执行配置快照

每次 Agent Run 在启动前创建不可变的执行配置快照，至少记录：

- runtime_id、connection_profile_id、Adapter 版本；
- Agent CLI/SDK 版本、执行宿主标识；
- model_selection 模式、实际 model_id（若可报告）、Provider（若可识别）；
- Adapter 参数及有效值；
- 配置解析来源：系统/项目/阶段/分集/任务；
- 项目 Stage Skill Profile 版本和实际 Skill Plan；
- 每个 Skill 的稳定 ID、CinePilot 版本、来源 Commit；
- 项目/分集/阶段/目标对象范围；
- Handoff 版本与输入 Artifact Revision ID；
- 沙箱策略版本和校验结果；
- 任务开始时间、完成状态及 Runtime 可返回的诊断摘要。

历史快照用于回溯，不代表能够保证模型服务永远提供完全相同的输出。凭据、Token、完整认证文件、敏感环境变量绝不能进入 Run Snapshot 或普通日志。

---

## 7. 安全、认证与项目隔离

### 7.1 凭据原则

- 系统级连接凭据由宿主侧安全存储管理；优先使用操作系统凭据库或 Runtime 自身安全认证机制。
- SQLite 只存凭据引用/状态，不存明文密钥。
- 项目配置只引用 Connection Profile，不复制认证数据。
- Agent 不得读取全局凭据目录、用户完整 Home/AppData、SSH 私钥、系统环境变量全集或 SQLite。
- 不把全局系统配置目录挂载到项目 Agent 沙箱。
- Run 日志必须过滤 Token、Authorization Header、Cookie 和敏感环境变量。

### 7.2 CLI 认证是强制验证项

某些 CLI 会从用户 Home 下的配置/认证文件读取凭据。不能因为 CLI 已登录，就把整个 Home 目录挂入 Agent 沙箱。

Adapter 必须证明：
- Agent Runtime 能在最小授权环境内完成认证；
- Agent 运行时的 Shell/子进程不能直接读取全局认证文件；
- 认证材料不会通过环境变量、调试输出或工具回显泄露；
- 其他项目和系统凭据在 Agent 文件系统视图中不可见。

如果某个 CLI/SDK 无法满足要求，则该接入方式不能被标记为“安全可用”；应评估官方 SDK 的受控凭据通道、独立 Agent Host 或其他隔离方案。**不能把 cwd、Prompt 提醒或普通 symlink 当作安全边界。**

### 7.3 项目隔离

- Run 由后端根据不可变的 project_id 解析工作区；Agent 不可自行指定磁盘根路径。
- 当前项目的文件/媒体和当前任务所需 Skill 只读投射进入隔离视图。
- 不挂载其他项目、系统 SQLite、完整 Skill Registry 或全局配置。
- 所有工具调用从 Run 的授权上下文获取项目范围，校验对象归属。
- 如果目标平台无法提供经验证的隔离能力，应拒绝启动该 Runtime，而不是静默降级。

详细隔离机制见 [项目隔离 Agent 工作区规格](./project-isolated-agent-workspaces-and-skills.md)。

---

## 8. 页面功能与操作流程

### 8.1 系统设置 → Agent 接入管理

列表显示：
- Runtime 名称、接入类型、安装/连接状态；
- 版本、执行宿主、最近检测时间；
- 认证状态；
- 隔离资格状态；
- 可用模型发现状态；
- 是否启用。

操作：添加/检测 Runtime、编辑 Connection Profile、认证、测试连接、查看能力、启用/停用。

### 8.2 Agent 详情 → 能力与模型

分区：
1. 基本信息与版本；
2. 支持能力清单；
3. 模型目录（来自 Adapter 动态发现）；
4. 参数 Schema；
5. 连接测试与隔离测试结果；
6. 已使用该 Runtime 的项目/任务统计。

### 8.3 模型配置 → Agent Model Profiles

用户先选 Runtime Connection，再选择其支持的模型/Runtime 默认选项，最后设置当前 Adapter 支持的参数。保存为可复用 Profile。

### 8.4 项目设置 → Agent 执行策略

- 项目默认 Agent Model Profile；
- A1–A4、B1–B4 阶段级 Profile；
- 可选允许哪些 Runtime；
- 查看配置继承来源；
- 分集阶段覆盖入口。

该页面与 Stage Skill Profile 页面分离，但在任务预览时合并展示。

### 8.5 Agent Run 详情

显示实际执行快照：
- Agent/Runtime/版本；
- 实际模型与参数；
- 配置来自哪个层级；
- Skill Plan 和版本；
- Handoff 的输入 Revision；
- 运行事件、结果草稿、失败原因；
- 不显示密钥和敏感配置。

---

## 9. Python 服务模块与逻辑接口

建议模块：
- AgentRegistryService：系统支持的 Runtime/Adapter 注册；
- AgentConnectionService：Connection Profile、检测、认证状态；
- AgentCapabilityService：能力和模型目录发现；
- AgentModelProfileService：Profile CRUD 与参数校验；
- ExecutionProfileResolver：按优先级解析系统/项目/阶段/分集/任务配置；
- AgentRuntimeAdapter：启动、事件、取消、恢复、工具桥接；
- AgentRunSnapshotService：固化执行快照；
- CredentialBroker：宿主侧凭据引用和最小授权注入。

逻辑 API 候选（非最终 OpenAPI）：

| 方法 | 路由 | 功能 |
|---|---|---|
| GET | /system/agent-runtimes | Runtime Registry |
| POST | /system/agent-connections/detect | 检测本机 Runtime |
| POST | /system/agent-connections | 建立 Connection Profile |
| POST | /system/agent-connections/{id}/test | 连接/认证/隔离测试 |
| GET | /system/agent-connections/{id}/capabilities | 能力声明 |
| POST | /system/agent-connections/{id}/discover-models | 动态发现模型 |
| GET | /system/agent-connections/{id}/models | 已发现模型目录 |
| GET/POST/PATCH | /system/agent-model-profiles | Agent Model Profile 管理 |
| GET/PUT | /projects/{project_id}/agent-policy | 项目默认策略 |
| GET/PUT | /projects/{project_id}/stage-agent-routes | 项目 A1–B4 阶段路由 |
| GET/PUT | /projects/{project_id}/episodes/{episode_id}/stage-agent-overrides | 分集阶段覆盖 |
| POST | /projects/{project_id}/agent-runs/preview | 预览解析后的 Runtime/Model 配置 |
| GET | /projects/{project_id}/agent-runs/{run_id}/execution-snapshot | 查看实际运行快照 |

Agent 内部受控工具不暴露系统设置 API，也不允许 Agent 修改 Runtime/Model/凭据配置。

---

## 10. V1 范围与后续扩展

### V1 必须

1. Agent Registry + Adapter Protocol；
2. 一个经验证的本机 Agent Runtime；
3. 安装/版本/认证/连接状态检测；
4. 动态能力发现；
5. 若 Runtime 支持则动态模型发现和显式选择，否则提供 Runtime 默认模型；
6. Adapter 参数 Schema 与参数校验；
7. 系统默认 + 项目默认 + 阶段配置 + 单次覆盖；
8. Agent Run 执行快照；
9. 项目隔离与凭据安全验证；
10. Agent 任务可创建、接收事件、取消（若 Runtime 支持）和提交草稿。

### 后续扩展

- 第二/第三种 Agent Runtime；
- 受控远端 Agent Host；
- 直接 API Provider（OpenAI-compatible 或其他）；
- Provider 级模型目录与成本统计；
- 模型能力基准和阶段质量路由；
- 自动回退/多 Agent 协作（必须显式策略，不默认开启）。

**V1 不要求**先建设统一的多供应商模型网关，也不要求把所有 Runtime 的参数强行标准化为相同字段。

---

## 11. 验收标准

1. 用户能添加/检测一个本机 Agent，并看到真实版本与认证状态。
2. 系统能展示该 Agent 实际支持的能力，不把未知标成支持。
3. 用户能从 Adapter 返回的模型清单中选择模型；不支持模型发现时能明确使用 Runtime 默认模型。
4. 不同 Agent 显示不同的参数字段和可选值，不出现虚假的统一参数。
5. 用户能建立 Agent Model Profile，并在项目和阶段中引用。
6. 单次任务覆盖能改变本次执行配置，但不修改项目默认配置。
7. Agent Run 能回溯实际 Runtime、模型、参数、Skill Plan、输入版本和配置来源。
8. 不可用模型或失效参数会阻止任务启动并解释原因，不静默切换。
9. Agent 进程无法读取其他项目、系统 SQLite 和全局凭据；隔离验证失败时不能标记 Runtime 为可安全运行。
10. Web/Electron 均通过同一 Python 服务解析执行配置；UI 不直接拼接任意 CLI 命令或注入系统路径。

---

## 12. 与开发的关系

建议先实现 Adapter Protocol、Runtime Registry、Connection Profile、Capability Discovery 和 Run Snapshot，再做模型配置 UI。首个端到端 Agent Run 能够正确绑定项目隔离工作区、准确模型配置和 Skill Plan 后，再增加第二种 Runtime。

该规格与 [开发启动路线图](./development-kickoff-and-implementation-roadmap.md) 一起作为 Codex 开发依据。