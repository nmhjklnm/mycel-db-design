# Mycel DB 全局概览

## 思维导图

```mermaid
mindmap
  root((Mycel DB<br/>━━━━━━━━━━<br/>5 schema<br/>39 表 + 1 视图))

    **identity**<br/>身份与账号根层<br/>所有 schema 的共享只读层
      users<br/>👤 human + agent 共表<br/>type 字段区分<br/>owner_user_id 指向创建者
      accounts<br/>🔑 登录凭据<br/>username · password_hash<br/>api_key_hash
      assets<br/>🖼 用户静态资源<br/>头像 · 用户上传文件<br/>kind 字段区分用途
      user_settings<br/>⚙️ KV 配置存储<br/>(user_id, scope) PK<br/>config JSONB
      invite_codes<br/>📨 注册邀请码<br/>控制谁能注册<br/>used_by · expires_at
      model_providers<br/>🤖 LLM 提供商连接<br/>api_key_enc 加密存储<br/>状态 · 健康检查 ★Realtime
      model_mappings<br/>🗺 模型别名映射<br/>alias → 实际模型<br/>is_active 唯一激活

    **chat**<br/>即时通讯<br/>9 表 + 1 视图<br/>所有写入走 mycel-chat 服务
      chats<br/>💬 会话容器<br/>kind: dm · group · ai<br/>next_message_seq 原子递增
      messages<br/>📩 消息主体<br/>seq 单调递增 · 幂等防重<br/>E2E 加密预留 ★Realtime
      messages_for_user<br/>👁 VIEW<br/>自动过滤 per-user 软删除<br/>开发者应查此视图
      message_attachments<br/>📎 消息附件<br/>关联 identity.assets<br/>一条消息多个附件
      message_deliveries<br/>📲 多设备投递追踪<br/>关联 container.devices<br/>驱动 push 通知
      message_reactions<br/>👍 emoji 反馈<br/>agent 反馈信号<br/>可用于 RLHF 训练
      message_pins<br/>📌 消息置顶<br/>群公告 · agent 关键产出<br/>记录 pin 操作人
      chat_members<br/>👥 成员 + 权限<br/>last_read_seq 水位线<br/>未读数基于水位线计算
      contacts<br/>📇 通讯录有向边<br/>kind + state 双字段<br/>fjj 设计：非单枚举
      relationships<br/>🤝 好友状态机<br/>user_low / user_high 规范化<br/>initiator_user_id 记录发起方 ★Realtime

    **agent**<br/>AI 对话与调度<br/>17 张表<br/>LangGraph 深度集成
      agent_configs<br/>🧠 agent 行为定义<br/>1:1 绑定 identity.users<br/>tools · prompt · mcp 配置
      agent_rules<br/>📋 行为规则
      agent_skills<br/>🛠 技能定义
      agent_sub_agents<br/>🤖 子 agent 编排
      threads<br/>🧵 对话线程<br/>status: 生命周期<br/>run_status: 运行状态 ★Realtime
      thread_launch_prefs<br/>🚀 启动偏好<br/>默认 workspace · 模型
      run_events<br/>📊 执行日志<br/>⚠️ 无界增长<br/>建议按月分区
      summaries<br/>📝 上下文压缩<br/>is_active 标识当前有效<br/>agent 重启必查
      message_queue<br/>📥 待处理消息<br/>thread 内 FIFO<br/>id ASC 顺序消费
      file_operations<br/>📁 文件操作记录
      tool_tasks<br/>⚙️ 长任务追踪<br/>进度 · 状态 · 结果 ★Realtime
      schedules<br/>⏰ 定时任务<br/>cron 表达式<br/>next_run_at ASC 索引
      schedule_runs<br/>📅 调度执行记录
      checkpoints<br/>💾 LangGraph 状态<br/>🔴 严禁前端访问<br/>含完整对话状态
      checkpoint_blobs<br/>💾 LangGraph Blob
      checkpoint_writes<br/>💾 LangGraph 写入
      checkpoint_migrations<br/>💾 LangGraph 迁移

    **container**<br/>计算基础设施<br/>三层模型<br/>daemon 通过 mycel-container 写入
      devices<br/>💻 用户计算端点<br/>心跳 + 乐观锁 version<br/>push_token 归此 ★Realtime
      environments<br/>🐳 隔离执行环境<br/>desired ↔ observed 双轨<br/>provider: local·docker·fly ★Realtime
      workspaces<br/>📂 项目工作区<br/>desired ↔ observed 双轨<br/>recipe_snapshot 创建时快照 ★Realtime

    **hub**<br/>Agent 市场<br/>3 张表<br/>允许匿名浏览
      publishers<br/>🏢 发布者主体<br/>关联 identity.users<br/>status: active · suspended
      marketplace_items<br/>📦 市场商品<br/>全文搜索 tsvector<br/>GIN 索引 · tags 数组
      marketplace_versions<br/>🔖 语义版本<br/>yanked 下架标记<br/>install_count 统计
```

---

## 全局设计模式

| 模式 | 用在哪里 | 解决什么问题 |
|------|---------|------------|
| **双轨状态机** `desired_state / observed_state` | container.environments · workspaces | 用户意图（控制面写）与实际状态（daemon 上报）分离，差异驱动收敛 |
| **乐观锁** `version INTEGER` | devices · environments · workspaces · chat_members | 心跳/状态上报高并发时防覆写，CAS 更新，冲突则调用方重试 |
| **软删除** `status = 'deleted'` | 全局 | 数据可恢复；Realtime 触发 UPDATE 事件而非 DELETE，前端拿到完整行再处理 |
| **冗余 `owner_user_id`** | agent · container（跨 schema 查询） | 避免跨表 JOIN，直接按 owner 过滤，也用于 RLS 策略 |
| **应用层 FK（无 DB 外键约束）** | 所有跨 schema 关联 | 避免跨 schema 锁表；允许未来按 schema 拆分数据库 |
| **`SECURITY DEFINER` RPC** | 心跳·状态上报·seq 分配·未读计数 | 绕过 RLS 做原子操作，业务校验在 RPC 内部显式执行 |
| **部分索引** `WHERE status = 'active'` | 全局 | 只索引有效行，减少索引体积，提升扫描效率 |
| **`is_member()` 辅助函数** | chat schema 所有 RLS 策略 | `STABLE` 缓存优化，同一 query 内多次 RLS 判断只查一次 |
| **recipe_snapshot 快照** | container.workspaces | 创建时固化 recipe，recipe 后续变更不影响已有工作区的重建 |
| **水位线未读计数** | chat.chat_members.last_read_seq | O(1) 计算未读数：`next_message_seq - 1 - last_read_seq`，不扫消息表 |

---

## 跨 schema 依赖关系

```
                    ┌─────────────────────────────────┐
                    │           identity               │
                    │  users · accounts · assets       │
                    │  user_settings · invite_codes    │
                    │  model_providers · model_mappings│
                    └──────────────┬──────────────────┘
                                   │ 所有 schema 依赖 identity.users
           ┌───────────────────────┼───────────────────────┐
           ▼                       ▼                       ▼
  ┌────────────────┐    ┌─────────────────┐    ┌──────────────────┐
  │     chat       │    │     agent       │    │    container     │
  │ messages_for_  │    │ threads ──────────────→ workspaces      │
  │ user(VIEW)     │    │ run_events      │    │ environments     │
  │ chat_members   │    │ tool_tasks      │    │ devices ◄────────┤
  │ relationships  │    │ schedules       │    └──────────────────┘
  └───────┬────────┘    └─────────────────┘             │
          │                                              │
          └──── message_deliveries.device_id ────────────┘
                    (push 投递追踪)

  ┌──────────────┐
  │     hub      │  → identity.users（publishers.user_id）
  │  publishers  │  孤立叶节点，无其他 schema 依赖
  │  items       │
  │  versions    │
  └──────────────┘
```

---

## Realtime 发布配置

```sql
-- ★ 必须开启（核心实时体验）
ALTER PUBLICATION supabase_realtime ADD TABLE
    identity.users,           -- chat 列表头像 / 名字实时刷新
    identity.model_providers, -- 设置页：provider 探测失败即时提示
    chat.messages,            -- 新消息实时推送（IM 核心）
    chat.relationships,       -- 好友请求通知
    agent.threads,            -- AI 运行状态 → 前端进度条
    agent.tool_tasks,         -- 长任务进度实时更新
    container.devices,        -- 设备在线 / 离线状态
    container.environments,   -- 环境启动进度
    container.workspaces;     -- 工作区状态 + 重建进度

-- ⚠ 可选（低频，可轮询替代）
-- ALTER PUBLICATION supabase_realtime ADD TABLE
--     chat.chat_members,    -- 成员变更
--     agent.schedules,      -- 定时任务更新
--     hub.marketplace_items;
```

> **⚠️ 安全红线**
> - `agent.checkpoints` 系列（4 张表）**严禁** GRANT 给 authenticated / anon：存有完整对话状态，可能含 API key、文件内容
> - `identity.users` Realtime 广播全体已认证用户，前端订阅**必须加 filter**（如只订阅当前 chat 成员的 ID 集合）
> - `chat.messages` per-user 软删除过滤在应用层，**必须查 `messages_for_user` 视图**，不要直接查 `chat.messages`

---

## 5 个核心 RPC 概览

| RPC | 所属 schema | 功能 |
|-----|-----------|------|
| `device_heartbeat(device_id, version, capabilities?)` | container | 心跳 + CAS 更新，含 10s 节流防 DDoS |
| `device_disconnect(device_id)` | container | 原子级联：设备→offline，环境→detached，工作区→detached |
| `environment_observe / workspace_observe` | container | daemon 上报实际状态，返回 desired_state 作指令 |
| `count_unread_per_chat()` | chat | 基于水位线批量计算未读数，user_id 从 auth.uid() 取 |
| `mark_read(chat_id, seq)` | chat | 更新 last_read_seq 水位线，只前进不后退 |
