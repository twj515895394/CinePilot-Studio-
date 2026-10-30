# Agent 执行源选择：V1 运行时路线（待确认）

> 日期：2026-10-30
> 状态：产品/技术重要边界，**未由用户确认**
> 来源：[Agent / Skill 协作与运行架构](../spec/agent-skill-orchestration-and-runtime.md)、[整体产品流程](../spec/overall-product-workflow-and-system-map.md)

## 已确认

Vue Web + Electron 复用前端、Python 负责后端业务服务。Agent 负责内容创作、Skill 是专业能力，代码负责正式阶段准入、精简 Handoff、权限、数据查询、提交草稿和定稿。用户希望把 CLI 创作体验放进 UI，但当前尚未决定 Agent 的 V1 首要执行源。

## 一个直接影响架构的例子

用户打开 Electron 应用，电脑上已安装 Codex CLI，希望在右侧对话生成 EP01 的分镜。Python 可以在受控条件下管理本机 CLI 执行并将结果保存；同一用户如果通过 Web 远程访问，浏览器**不可能直接调用用户电脑上的 CLI**，执行必须发生在 Python 服务所在机器/指定 Agent Host，或使用明确授权的远程代理。把 CLI 与 Web/Electron 的运行位置说清楚，关系到部署形态和任务隔离。

## 候选方案

### A. V1 优先本机/指定 Agent Host 的 CLI，API 往后扩展（推荐）

- Python 通过 Agent Runtime Adapter 对接优先选定的一种 CLI（候选 Codex CLI 或 Claude Code CLI）；将第二种 CLI 当成兼容目标而非同时首发承诺。
- Electron 本机 Python 服务与 CLI 同机运行；Web 若要执行任务，服务器/Agent Host 必须有可用 CLI 及允许访问的工作目录。
- CLI 版本、认证方式、图像输入、结构化输出、工具调用桥接和会话恢复能力需实测；不得默认所有 CLI 功能完全一致。
- 优点：符合用户此前希望将 Agent CLI 直接带进项目 UI 的路径，首期只需实现一个稳定适配器。
- 局限：Web 远端部署与多用户隔离需要额外注意，不能直接复用本机 CLI 会话。

### B. V1 优先云端 SDK/API，CLI 以后扩展

- Python 直接通过统一模型 API/SDK 调用并管理工具。
- 优点：服务端模型与工具调用能力更易统一；远程 Web 架构较直接。
- 局限：需要 API 认证和调用费用，不满足“已有本机 CLI 体验优先”的使用习惯。

### C. V1 同时完整支持 CLI + SDK/API

- 早期就承担两套运行时的多模态、工具调用、权限和会话差异测试。
- 优点：选型自由。
- 局限：V1 开发与验收范围膨胀，违背先做稳固闭环的原则。

## 推荐 A 的边界

不把选 A 等同于必须 Codex CLI 独家支持，也不等同于 Web 可自动复用用户本机账户。需要再评估具体 CLI 的能力与首发次序，先按最小可用运行时完成 A4→B1→B2→B3→B4 的闭环。

**当前唯一核心问题**：首期 Agent 执行源是否以 **A：CLI 优先、API 后续** 为准？若有不同偏好，可选 B 或 C。本问题只决定优先路线，不进入具体终端命令、异常重试或供应商版本细节。
