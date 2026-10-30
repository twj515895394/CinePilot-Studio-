# 全系统共用一个 SQLite 数据库，项目内容文件独立存储

**状态：accepted**
**日期：2026-10-30**

CinePilot Studio 的管理后台和 Python 业务服务使用**一个系统级 SQLite 数据库**管理全部项目、分集、创意与资产的身份和关系、阶段状态、正式产物版本与审批记录、任务索引等。每个项目仍拥有独立工作目录，其 Markdown、JSON、图片和分集阶段产物保存在该项目下。Python 业务服务按项目归属和版本引用查询 SQLite，再读取对应文件，UI 与 Agent 不直接修改数据库。系统级数据库位于受保护的应用数据区域（建议逻辑位置 `app/cinepilot.sqlite`），**不暴露到任何项目 Agent 沙箱中**。具体物理应用数据根由部署配置决定。

**取舍**：每项目分别设置 SQLite 加一个中心项目注册表有利于整个项目目录直接搬迁，但会增加跨项目索引、数据库升级与管理成本。用户明确要求**共用 SQLite 以支持统一后台管理**，故采用单库模式；项目级备份/迁移需由业务服务把该项目内容文件与相关 SQLite 元数据一起打包导出，不能仅靠复制目录恢复全部版本和审批状态。

**边界**：确认的是 SQLite 的**全系统单实例作用域**，并不要求把 Markdown/JSON 创作正文重复存入数据库。后续可按实际需要建立数据库索引与可重建全文搜索索引；Markdown/JSON 仍按 [ADR-0012](./0012-field-owned-markdown-json-sqlite.md) 保持各自字段权威。跨项目访问权限仍遵守 [ADR-0017](./0017-project-isolated-agent-workspaces-and-skill-projection.md)，单库不代表 Agent 能查询其他项目。
