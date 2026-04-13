# Mycel DB 全局概览

```mermaid
mindmap
  root((Mycel DB<br/>5 schema · 39 表))
    identity
      users<br/>human + agent 共表
      accounts<br/>登录凭据
      assets<br/>头像 / 用户文件
      user_settings<br/>KV 偏好配置
      invite_codes<br/>注册邀请码
      model_providers<br/>LLM 提供商
      model_mappings<br/>模型别名 / 激活
    chat
      chats<br/>DM · group · ai
      messages<br/>seq + 幂等 + E2E预留
      messages_for_user<br/>VIEW：per-user软删过滤
      message_attachments
      message_deliveries<br/>多设备投递追踪
      message_reactions<br/>emoji 反馈
      message_pins<br/>消息置顶
      chat_members<br/>水位线 last_read_seq
      contacts<br/>通讯录有向边
      relationships<br/>好友状态机
    agent
      agent_configs<br/>行为定义 1:1 agent user
      agent_rules
      agent_skills
      agent_sub_agents
      threads<br/>对话线程 ★Realtime
      thread_launch_prefs
      run_events<br/>执行日志（无界增长）
      summaries<br/>上下文压缩
      message_queue<br/>待处理消息
      file_operations
      tool_tasks<br/>长任务进度 ★Realtime
      schedules<br/>定时任务
      schedule_runs
      checkpoints<br/>LangGraph · 禁止前端访问
      checkpoint_blobs
      checkpoint_writes
      checkpoint_migrations
    container
      devices<br/>心跳 + 乐观锁 ★Realtime
      environments<br/>双轨状态机 ★Realtime
      workspaces<br/>双轨状态机 ★Realtime
    hub
      publishers<br/>发布者主体
      marketplace_items<br/>全文搜索 tsvector
      marketplace_versions<br/>语义版本 + yanked
```

---

## 跨 schema 依赖

```
identity ←── chat      (sender_user_id, contacts, relationships)
identity ←── agent     (agent_user_id, owner_user_id, model解析)
identity ←── container (owner_user_id 冗余)
identity ←── hub       (publishers.user_id)
container ←── chat     (message_deliveries.device_id → devices)
container ←── agent    (threads.current_workspace_id → workspaces)
```

> identity 是纯根层，零外向依赖。hub 是最孤立叶节点。

---

## 关键设计模式

| 模式 | 用在哪里 | 解决什么 |
|------|---------|---------|
| 双轨状态机 `desired/observed` | container.environments · workspaces | 控制面意图 vs 实际状态分离 |
| 乐观锁 `version` | devices · environments · workspaces · chat_members | 并发写入冲突 |
| 软删除 `status='deleted'` | 全局 | 可恢复 + Realtime UPDATE 事件 |
| 应用层 FK | 所有跨 schema 关联 | 避免跨 schema 锁表 |
| `SECURITY DEFINER` RPC | 心跳·状态上报·seq分配·未读计数 | 绕过 RLS 做原子操作 |
| 部分索引 `WHERE status='active'` | 全局 | 减少索引体积，只扫有效行 |

---

## Realtime 发布（必须开启）

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE
    identity.users,
    identity.model_providers,
    chat.messages,
    chat.relationships,
    agent.threads,
    agent.tool_tasks,
    container.devices,
    container.environments,
    container.workspaces;
```

> ⚠️ `agent.checkpoints` 系列严禁 GRANT 给前端，存有完整对话状态。
> ⚠️ `identity.users` Realtime 广播全体，前端订阅必须加 filter。
