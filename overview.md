# Mycel DB 全局概览

## 思维导图

```mermaid
mindmap
  root((Mycel DB<br/>5 schema · 39 表))

    identity<br/>身份根层<br/>所有 schema 共享
      users<br/>👤 human + agent 共表<br/>type 区分身份类型
      accounts<br/>🔑 登录凭据
      assets<br/>🖼 头像与用户文件<br/>kind 区分用途
      user_settings<br/>⚙️ KV 配置<br/>user_id + scope 为主键
      invite_codes<br/>📨 注册邀请码
      model_providers<br/>🤖 LLM 提供商<br/>api_key 加密存储 ★RT
      model_mappings<br/>🗺 模型别名<br/>is_active 唯一激活

    chat<br/>即时通讯<br/>9表+1视图
      chats<br/>💬 会话容器<br/>dm · group · ai
      messages<br/>📩 消息主体<br/>seq幂等 E2E预留 ★RT
      messages_for_user<br/>👁 VIEW<br/>per-user软删过滤
      message_attachments<br/>📎 消息附件
      message_deliveries<br/>📲 多设备投递追踪
      message_reactions<br/>👍 emoji反馈
      message_pins<br/>📌 消息置顶
      chat_members<br/>👥 成员水位线<br/>last_read_seq未读计数
      contacts<br/>📇 通讯录有向边
      relationships<br/>🤝 好友状态机 ★RT

    agent<br/>AI对话与调度<br/>17张表
      agent_configs<br/>🧠 行为定义<br/>1:1 绑定 agent user
      agent_rules<br/>📋 行为规则
      agent_skills<br/>🛠 技能定义
      agent_sub_agents<br/>子agent编排
      threads<br/>🧵 对话线程<br/>双状态字段 ★RT
      thread_launch_prefs<br/>🚀 启动偏好
      run_events<br/>📊 执行日志<br/>无界增长需分区
      summaries<br/>📝 上下文压缩<br/>agent重启必查
      message_queue<br/>📥 待处理消息 FIFO
      file_operations<br/>📁 文件操作记录
      tool_tasks<br/>长任务追踪 ★RT
      schedules<br/>⏰ 定时任务
      schedule_runs<br/>调度执行记录
      checkpoints<br/>💾 LangGraph状态<br/>🔴 严禁前端访问
      checkpoint_blobs<br/>💾 LangGraph Blob
      checkpoint_writes<br/>💾 LangGraph写入
      checkpoint_migrations<br/>💾 LangGraph迁移

    container<br/>计算基础设施<br/>三层模型
      devices<br/>💻 用户计算端点<br/>心跳+乐观锁 ★RT
      environments<br/>🐳 隔离执行环境<br/>desired与observed双轨 ★RT
      workspaces<br/>📂 项目工作区<br/>desired与observed双轨 ★RT

    hub<br/>Agent市场<br/>匿名可浏览
      publishers<br/>🏢 发布者主体
      marketplace_items<br/>📦 市场商品<br/>全文搜索tsvector
      marketplace_versions<br/>🔖 语义版本<br/>yanked下架标记
```

> `★RT` = 已加入 Realtime publication · `🔴` = 安全敏感，禁止前端直接访问

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

## 各 Schema ER 图

### identity — 身份根层（7 表）

```mermaid
erDiagram
    users {
        text id PK
        text type "human 或 agent"
        text display_name
        text email UK
        text mycel_id UK
        text owner_user_id FK "agent 的归属人"
        text agent_config_id FK
        timestamptz created_at
    }
    accounts {
        text id PK
        text user_id FK
        text username UK
        text password_hash
        text api_key_hash
    }
    assets {
        text id PK
        text owner_user_id FK
        text kind "avatar · file · etc"
        text storage_key
        text sha256
        text status
    }
    user_settings {
        text user_id PK
        text scope PK "general · ui · etc"
        jsonb config
        timestamptz updated_at
    }
    invite_codes {
        text code PK
        text created_by FK
        text used_by FK
        timestamptz expires_at
    }
    model_providers {
        text id PK
        text user_id FK
        text name
        text api_key_enc "AES-256-GCM 密文"
        text endpoint
        text status
    }
    model_mappings {
        text id PK
        text user_id FK
        text provider_id FK
        text alias UK
        boolean is_active "同用户唯一激活"
        jsonb params
    }

    users ||--o{ accounts : "登录凭据"
    users ||--o{ assets : "拥有"
    users ||--|| user_settings : "偏好配置"
    users ||--o{ model_providers : "LLM 提供商"
    model_providers ||--o{ model_mappings : "模型别名"
    users ||--o{ users : "创建 agent user"
```

---

### chat — 即时通讯（9 表 + 1 视图）

```mermaid
erDiagram
    chats {
        text id PK
        text kind "dm · group · ai"
        text creator_user_id FK
        bigint next_message_seq "原子递增"
        timestamptz last_message_at "冗余排序字段"
        text last_message_preview
        text status
    }
    messages {
        text id PK
        text chat_id FK
        text sender_user_id FK
        bigint seq UK "chat 内唯一"
        text client_message_id UK "幂等防重"
        text content
        text content_type
        text reply_to_message_id FK
        text thread_root_id FK
        jsonb deleted_for_user_ids "per-user 软删"
        timestamptz deleted_at "全员删除"
        timestamptz retracted_at
        tsvector search_vector
    }
    message_attachments {
        text id PK
        text message_id FK
        text asset_id FK
        int sort_order
    }
    message_deliveries {
        text message_id PK
        text device_id PK "关联 container.devices"
        text status "pending · delivered · failed"
        timestamptz delivered_at
    }
    message_reactions {
        text message_id PK
        text user_id PK
        text emoji PK
    }
    message_pins {
        text id PK
        text chat_id FK
        text message_id FK
        text pinned_by FK
        timestamptz pinned_at
    }
    chat_members {
        text chat_id PK
        text user_id PK
        text role "owner · admin · member"
        bigint last_read_seq "未读水位线"
        boolean muted
        int version
    }
    contacts {
        text id PK
        text owner_user_id FK
        text target_user_id FK
        text kind
        text state
    }
    relationships {
        text id PK
        text user_low FK "user_id 较小者"
        text user_high FK "user_id 较大者"
        text initiator_user_id FK
        text status "pending · accepted · blocked"
    }

    chats ||--|{ chat_members : "成员"
    chats ||--o{ messages : "消息"
    chats ||--o{ message_pins : "置顶"
    messages ||--o{ message_attachments : "附件"
    messages ||--o{ message_deliveries : "投递追踪"
    messages ||--o{ message_reactions : "反馈"
    messages ||--o{ messages : "回复线程"
```

---

### agent — AI 对话与调度（17 表）

```mermaid
erDiagram
    agent_configs {
        text id PK
        text agent_user_id FK UK "1:1 绑定 identity.users"
        text owner_user_id FK
        text model "默认模型 alias"
        jsonb tools_json
        text system_prompt
        jsonb mcp_json
    }
    agent_rules { text id PK; text config_id FK }
    agent_skills { text id PK; text config_id FK }
    agent_sub_agents { text id PK; text config_id FK; text sub_agent_user_id FK }

    threads {
        text id PK
        text agent_user_id FK
        text owner_user_id FK
        text current_workspace_id FK "关联 container.workspaces"
        text status "active · archived"
        text run_status "idle · running · paused · error"
        timestamptz last_active_at
        int version
    }
    thread_launch_prefs { text id PK; text thread_id FK; jsonb prefs }

    run_events {
        text id PK
        text thread_id FK
        text type
        jsonb data
        timestamptz created_at "无界增长，建议按月分区"
    }
    summaries {
        text id PK
        text thread_id FK
        text content
        boolean is_active "当前有效摘要"
        int token_count
    }
    message_queue {
        text id PK
        text thread_id FK
        text role
        text content
        timestamptz created_at "id ASC 顺序消费"
    }
    file_operations { text id PK; text thread_id FK; text operation; text path }
    tool_tasks {
        text id PK
        text thread_id FK
        text tool_name
        text status
        jsonb progress
        jsonb result
    }

    schedules {
        text id PK
        text owner_user_id FK
        text agent_user_id FK
        text cron_expr
        boolean enabled
        timestamptz next_run_at
    }
    schedule_runs { text id PK; text schedule_id FK; text status; timestamptz ran_at }

    checkpoints {
        text thread_id PK "LangGraph checkpoint_id"
        text checkpoint_id PK
        jsonb metadata
    }
    checkpoint_blobs { text thread_id PK; text checkpoint_id PK; text channel PK }
    checkpoint_writes { text thread_id PK; text checkpoint_id PK; int idx PK }
    checkpoint_migrations { int v PK }

    agent_configs ||--o{ agent_rules : ""
    agent_configs ||--o{ agent_skills : ""
    agent_configs ||--o{ agent_sub_agents : ""
    threads ||--o{ run_events : "执行日志"
    threads ||--o{ summaries : "上下文压缩"
    threads ||--o{ message_queue : "待处理消息"
    threads ||--o{ file_operations : ""
    threads ||--o{ tool_tasks : "长任务"
    threads ||--o{ checkpoints : "LangGraph 状态"
    schedules ||--o{ schedule_runs : "执行记录"
```

---

### container — 计算基础设施（3 表）

```mermaid
erDiagram
    devices {
        text id PK
        text owner_user_id FK
        text name
        text device_type "desktop · server · cloud_vm"
        text platform "macos · linux · windows"
        text arch
        jsonb capabilities "docker · gpu · cpus 等"
        text push_token
        text push_platform "apns · fcm · web_push"
        text status "online · offline · disabled"
        timestamptz last_heartbeat_at
        int version "乐观锁"
    }
    environments {
        text id PK
        text device_id FK
        text owner_user_id FK
        text provider_name "local · docker · fly · e2b"
        text provider_env_id UK "同设备下唯一"
        text image
        text desired_state "running · stopped"
        text observed_state "running · stopped · creating · detached · error"
        text status "active · archived · deleted"
        timestamptz last_seen_at
        text last_error
        int version "乐观锁"
    }
    workspaces {
        text id PK
        text environment_id FK
        text owner_user_id FK
        text workspace_path UK "同环境下唯一"
        text recipe_id FK
        jsonb recipe_snapshot "创建时快照，重建用"
        text volume_id FK
        text desired_state "running · stopped"
        text observed_state "running · stopped · creating · detached · error"
        text status "active · archived · deleted"
        boolean needs_refresh
        timestamptz refresh_hint_at
        int version "乐观锁"
    }

    devices ||--o{ environments : "托管"
    environments ||--o{ workspaces : "包含"
```

---

### hub — Agent 市场（3 表）

```mermaid
erDiagram
    publishers {
        text id PK
        text user_id FK UK "1:1 绑定 identity.users"
        text display_name
        text status "active · suspended"
        timestamptz created_at
    }
    marketplace_items {
        text id PK
        text publisher_id FK
        text name
        text slug UK
        text visibility "public · unlisted · private"
        int install_count
        jsonb tags
        tsvector search_vector "GIN 索引全文搜索"
        timestamptz published_at
    }
    marketplace_versions {
        text id PK
        text item_id FK
        text version "语义版本 semver"
        text published_by FK
        boolean yanked "true = 已下架"
        jsonb manifest
        timestamptz created_at
    }

    publishers ||--o{ marketplace_items : "发布"
    marketplace_items ||--o{ marketplace_versions : "版本历史"
```

---

## 5 个核心 RPC 概览

| RPC | 所属 schema | 功能 |
|-----|-----------|------|
| `device_heartbeat(device_id, version, capabilities?)` | container | 心跳 + CAS 更新，含 10s 节流防 DDoS |
| `device_disconnect(device_id)` | container | 原子级联：设备→offline，环境→detached，工作区→detached |
| `environment_observe / workspace_observe` | container | daemon 上报实际状态，返回 desired_state 作指令 |
| `count_unread_per_chat()` | chat | 基于水位线批量计算未读数，user_id 从 auth.uid() 取 |
| `mark_read(chat_id, seq)` | chat | 更新 last_read_seq 水位线，只前进不后退 |
