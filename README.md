# Mycel DB Design

Mycel 模块化数据库设计文档。基于 Supabase / PostgreSQL，5 个 schema，覆盖完整 DDL、索引、RLS、Realtime 和 RPC。

## 从哪里开始

**1. 先看总览** → [`overview.md`](overview.md)

- Mermaid 思维导图：5 个 schema 的结构和 Realtime 标记一览
- 每个 schema 的 ER 图（需要支持 Mermaid 的 Markdown 渲染器）
- 跨 schema 依赖图（ASCII）
- 核心 RPC 汇总表

> VSCode 推荐安装 [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) 插件后按 `Cmd+Shift+V` 预览。GitHub 原生支持 Mermaid，直接在线查看即可。

**2. 看设计决策** → [`schema-decisions.md`](schema-decisions.md)

- 5 个 schema 的完整表清单（49 张表）
- 34 条设计 FAQ（为什么这么设计，为什么不那么设计）
- 全局依赖矩阵、Realtime 发布清单、RLS 一致性审查、全局索引策略

**3. 按需查具体 schema**

| 文件 | Schema | 内容 |
|------|--------|------|
| [`identity-schema.md`](identity-schema.md) | 用户 & 认证 | users / accounts / assets / settings / invite_codes / model_providers |
| [`chat-schema.md`](chat-schema.md) | 即时通讯 | chats / messages / reactions / pins / members / contacts，含水位线未读计数 |
| [`container-schema.md`](container-schema.md) | 设备 & 运行时 | devices / environments / workspaces，双状态机 + 心跳 RPC |
| [`agent-schema.md`](agent-schema.md) | AI Agent | threads / runs / steps + LangGraph checkpoint 表（严格隔离，禁止前端访问） |
| [`hub-schema.md`](hub-schema.md) | Agent 市场 | publishers / marketplace_items（全文搜索）/ marketplace_versions |

## 关键设计模式

- **双状态机**：`desired_state`（控制面意图） vs `observed_state`（daemon 上报实际状态），用于 container schema
- **软删除**：`status = 'deleted'` 代替 `DELETE`，保留 Realtime UPDATE 事件
- **乐观锁**：`version INTEGER` + CAS 更新，防止并发覆写
- **应用层 FK**：跨 schema 不使用数据库外键约束，避免锁竞争
- **水位线未读数**：`next_message_seq - 1 - last_read_seq`，O(1) 计算
- **per-user 软删除视图**：`messages_for_user` VIEW 封装 `deleted_for_user_ids` 过滤，绕过 RLS JSONB 索引限制
