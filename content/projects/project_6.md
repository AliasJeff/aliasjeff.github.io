+++
title = "ACLIx: CLI Agent Tool"
description = "一款命令行（CLI）AI 助手，能够智能执行 Shell 命令、编辑和管理代码库，并实现端到端的工作流自动化。"
weight = 1

[extra]
local_image = ""
github = "https://github.com/AliasJeff/ACLIx"
demo = "https://www.npmjs.com/package/@aliasjeff/acli"
tags = ["AI", "大模型应用", "TypeScript"]
+++

### 1. Agent 编排

采用 Master Orchestrator + ReAct 模式，将复杂任务拆分为多个专用子 Agent（如 Planner、Explorer、Executor）。各子 Agent 在隔离上下文中运行，并具备读写权限控制，避免冲突并优化 Token 使用。

### 2. 工具（Tools）

提供一组原生工具用于与环境交互：
shell、python、file_read、file_write、file_edit、glob、grep、ask_user、web_search、read_skill。

### 3. 分层记忆系统

多层次上下文管理机制：
- 长期记忆基于 Markdown 文件存储用户和项目级的持久信息。
- 短期记忆基于 SQLite 记录会话历史并支持跨会话恢复。
- 压缩记忆在超出 Token 限制时自动摘要历史内容以压缩上下文。
- BM25 检索根据查询动态召回最相关的 Top 3 记忆片段以提升效率与相关性。

### 4. 可扩展 Skills 与 Rules

基于文件系统的模块化插件架构：
- Skills 通过 SKILL.md 定义可复用的标准操作流程。
- Rules 通过 RULE.md 动态注入行为约束。
- 渐进式加载机制仅在需要时引入相关规则或流程以避免上下文冗余。

### 5. 安全机制与 HITL
- 命令风险评估通过 AST 分析命令并划分低中高风险且对中高风险操作要求用户确认。
- 自动快照在文件修改前创建 SQLite 快照并支持通过 /undo 一键回滚。
- 注入防护与数据脱敏通过 <untrusted_data> 包裹不可信输入并自动处理敏感信息。
