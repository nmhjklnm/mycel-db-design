# agent schema 完整设计

> 2026-04-13，基于 32-schema-decisions.md agent 章节 + fjj runtime database reference
>
> agent 是所有 schema 中工作量最大的：16 张表，零 DDL，需从表名/注释推导完整结构。

---

## 架构前提

```
mycel-agent (service_role 直连 Supabase)
    │ 写 threads / run_events / message_queue / tool_tasks / checkpoints
    ▼
Supabase Postgres (agent schema)

LangGraph (连接参数 search_path=agent)
    │ 读写 checkpoints / checkpoint_blobs / checkpoint_writes / checkpoint_migrations
    ▼
Supabase Postgres (agent schema)

前端 (authenticated JWT)
    │ 读 threads(run_status) / agent_configs / run_events（有限）
    ▼
PostgREST / Realtime（受 RLS 约束）
```

---

## 假设声明

1. **`owner_user_id` 冗余**：agent_configs 和 threads 均加 `owner_user_id`（来自 identity.users 的 agent 行的 owner_user_id），用于 RLS 和快速查询，免 JOIN identity 表。
2. **threads 双状态字段**：`status`（生命周期：active/archived）+ `run_status`（执行状态：idle/running/paused/error），分离"这个 thread 还在用吗"和"agent 现在在不在跑"。
3. **timestamps 统一 TIMESTAMPTZ**：fjj doc 中部分表用 `double precision`（unix epoch），新 schema 统一改 TIMESTAMPTZ，迁移时需 `to_timestamp(col)` 转换。
4. **LangGraph 表结构不改**：checkpoints 系列 4 张表按 LangGraph 要求的精确字段存在，不擅自增减字段，框架通过 `checkpoint_migrations` 自管版本。
5. **`agent.schedules` 的 `next_run_at`**：改为 TIMESTAMPTZ（原 bigint），便于直接与 `now()` 比较。

---

## a) 完整 DDL

```sql
CREATE SCHEMA IF NOT EXISTS agent;
```

---

### 组 1：Agent 定义（4 表）

```sql
-- ================================================================
-- agent.agent_configs
-- agent user 的行为定义，1:1 绑定 identity.users (type='agent')
-- "慢变量"：改了影响该 agent 的所有后续 thread 行为
-- ================================================================

CREATE TABLE agent.agent_configs (
    id              TEXT        PRIMARY KEY,
    -- 归属
    agent_user_id   TEXT        NOT NULL UNIQUE,
        -- → identity.users (type='agent')（应用层 FK）
        -- 每个 agent user 只有一个 config，UNIQUE 约束保证 1:1
    owner_user_id   TEXT        NOT NULL,
        -- → identity.users (type='human')（冗余，便于 RLS + 用户 agent 列表）
        -- 等于 identity.users(agent_user_id).owner_user_id，创建时写入
    -- 基本信息
    name            TEXT        NOT NULL,
        -- agent 显示名，e.g. "我的代码助手"
    description     TEXT        NOT NULL DEFAULT '',
        -- 对外展示的描述
    -- 行为定义
    model           TEXT,
        -- 默认使用的模型 alias，e.g. 'mycel:large'；null 时用系统默认
    tools_json      JSONB       NOT NULL DEFAULT '[]',
        -- 启用的工具列表，e.g. ["bash", "file_read", "web_search"]
    system_prompt   TEXT        NOT NULL DEFAULT '',
        -- agent 的系统提示词
    mcp_json        JSONB       NOT NULL DEFAULT '{}',
        -- MCP server 配置，e.g. {"github": {"url": "..."}}
    runtime_json    JSONB       NOT NULL DEFAULT '{}',
        -- 运行时参数，e.g. {"max_iterations": 50, "timeout_sec": 3600}
    -- 发布状态
    status          TEXT        NOT NULL DEFAULT 'draft',
        -- 'draft' | 'published' | 'archived'
        -- draft: 仅 owner 可用；published: 可被 hub 发现；archived: 停用
    version         TEXT        NOT NULL DEFAULT '0.1.0',
        -- semver，发布到 hub 时用
    -- 元数据
    meta_json       JSONB       NOT NULL DEFAULT '{}',
        -- 任意扩展字段，保持向后兼容
    -- 时间戳
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT agent_configs_status_chk
        CHECK (status IN ('draft', 'published', 'archived'))
);

-- 用户的所有 agent（"我的 agent" 列表）
CREATE INDEX idx_agent_configs_owner
    ON agent.agent_configs(owner_user_id);

-- 已发布 agent（hub 发现、公开浏览）
CREATE INDEX idx_agent_configs_published
    ON agent.agent_configs(owner_user_id, updated_at DESC)
    WHERE status = 'published';


-- ================================================================
-- agent.agent_rules
-- agent 的规则文件，类 CLAUDE.md。单条规则独立 CRUD，不合进 configs JSONB
-- ================================================================

CREATE TABLE agent.agent_rules (
    id               TEXT        PRIMARY KEY,
    agent_config_id  TEXT        NOT NULL,
        -- → agent.agent_configs.id（应用层 FK）
    filename         TEXT        NOT NULL,
        -- 规则文件名，e.g. "CLAUDE.md" / "coding-style.md"
    content          TEXT        NOT NULL,
        -- 规则内容（Markdown）
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT agent_rules_config_filename_uq UNIQUE (agent_config_id, filename)
);

-- 同 config 下所有规则（加载 agent 配置时批量拉取）
CREATE INDEX idx_agent_rules_config
    ON agent.agent_rules(agent_config_id);


-- ================================================================
-- agent.agent_skills
-- agent 的技能定义，类 SKILL.md。单条技能独立 CRUD，支持增删改
-- ================================================================

CREATE TABLE agent.agent_skills (
    id               TEXT        PRIMARY KEY,
    agent_config_id  TEXT        NOT NULL,
        -- → agent.agent_configs.id（应用层 FK）
    name             TEXT        NOT NULL,
        -- 技能名，e.g. "deploy", "code-review"
    content          TEXT        NOT NULL,
        -- 技能内容（Markdown + 触发条件）
    meta_json        JSONB       NOT NULL DEFAULT '{}',
        -- 扩展元数据，e.g. {"triggers": [...], "version": "1.0"}
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT agent_skills_config_name_uq UNIQUE (agent_config_id, name)
);

CREATE INDEX idx_agent_skills_config
    ON agent.agent_skills(agent_config_id);


-- ================================================================
-- agent.agent_sub_agents
-- agent 可调用的子 agent 定义。子 agent 关系是配置时固定的，
-- 运行时的调用关系是临时的（不持久化 parent_thread_id）
-- ================================================================

CREATE TABLE agent.agent_sub_agents (
    id               TEXT        PRIMARY KEY,
    agent_config_id  TEXT        NOT NULL,
        -- → agent.agent_configs.id（应用层 FK）
    name             TEXT        NOT NULL,
        -- 子 agent 名，e.g. "coder", "reviewer"
    description      TEXT,
        -- 何时调用此子 agent
    model            TEXT,
        -- 子 agent 使用的模型（null 时继承父 agent）
    tools_json       JSONB       NOT NULL DEFAULT '[]',
        -- 子 agent 可用工具
    system_prompt    TEXT,
        -- 子 agent 系统提示词
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT agent_sub_agents_config_name_uq UNIQUE (agent_config_id, name)
);

CREATE INDEX idx_agent_sub_agents_config
    ON agent.agent_sub_agents(agent_config_id);
```

---

### 组 2：Thread 运行时（2 表）

```sql
-- ================================================================
-- agent.threads
-- agent 的运行实例。每次用户发起对话产生一个 thread（或复用已有的）
-- thread ↔ workspace 是可切换的连接关系，非 1:1 绑定
-- ================================================================

CREATE TABLE agent.threads (
    id                   TEXT        PRIMARY KEY,
    -- 归属
    agent_user_id        TEXT        NOT NULL,
        -- → identity.users (type='agent')（应用层 FK）
    owner_user_id        TEXT        NOT NULL,
        -- → identity.users (type='human')（冗余，便于 RLS + 前端查询）
        -- 等于 identity.users(agent_user_id).owner_user_id
    -- 工作区关联（可空，agent 可在无工作区模式下运行）
    current_workspace_id TEXT,
        -- → container.workspaces.id（跨 schema，应用层 FK）
        -- 可切换：同一 thread 可先后连不同 workspace
    -- 运行配置（覆盖 agent_config 默认值）
    model                TEXT,
        -- 此 thread 使用的模型 alias（null 时用 agent_config.model）
    cwd                  TEXT,
        -- 当前工作目录（legacy compat，新模式以 workspace_path 为准）
    -- 生命周期状态
    status               TEXT        NOT NULL DEFAULT 'active',
        -- 'active'   — 可正常使用
        -- 'archived' — 已归档，不再接受新消息
    -- 执行状态（Realtime 关注此字段）
    run_status           TEXT        NOT NULL DEFAULT 'idle',
        -- 'idle'    — 等待输入
        -- 'running' — agent 正在执行（LLM 调用 + 工具链）
        -- 'paused'  — 用户主动暂停
        -- 'error'   — 最后一次 run 以错误结束
    -- 分支追踪
    is_main              BOOLEAN     NOT NULL DEFAULT false,
        -- true = 此 agent 的主 thread（通常是最早创建的）
    branch_index         INTEGER     NOT NULL DEFAULT 0,
        -- 分支序号，由 identity.increment_user_thread_seq() 分配
    -- 时间戳
    last_active_at       TIMESTAMPTZ,
        -- 最后一次 run 完成时间，用于会话列表排序
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT threads_status_chk
        CHECK (status IN ('active', 'archived')),
    CONSTRAINT threads_run_status_chk
        CHECK (run_status IN ('idle', 'running', 'paused', 'error')),
    -- 同一 agent 下分支序号唯一
    CONSTRAINT threads_agent_branch_uq UNIQUE (agent_user_id, branch_index)
);

-- 用户的最近活跃 thread（首页加载、thread 列表）
CREATE INDEX idx_threads_owner_active
    ON agent.threads(owner_user_id, last_active_at DESC)
    WHERE status = 'active';

-- 某 agent 的所有活跃 thread
CREATE INDEX idx_threads_agent_active
    ON agent.threads(agent_user_id)
    WHERE status = 'active';

-- 正在运行的 thread（控制面监控、超时检测）
CREATE INDEX idx_threads_running
    ON agent.threads(owner_user_id, updated_at)
    WHERE run_status = 'running';

-- workspace 关联查询（workspace 被删时检查有无关联 thread）
CREATE INDEX idx_threads_workspace
    ON agent.threads(current_workspace_id)
    WHERE current_workspace_id IS NOT NULL;


-- ================================================================
-- agent.thread_launch_prefs
-- 记录"上次成功的启动配置"。是执行历史，不是纯偏好；
-- 启动时自动填充上次成功的参数，减少用户重复配置
-- ================================================================

CREATE TABLE agent.thread_launch_prefs (
    owner_user_id        TEXT        NOT NULL,
        -- → identity.users (human)（应用层 FK）
    agent_user_id        TEXT        NOT NULL,
        -- → identity.users (agent)（应用层 FK）
    last_confirmed_json  JSONB,
        -- 用户最后一次手动确认的启动参数快照
    last_successful_json JSONB,
        -- 最后一次成功启动的参数快照（用于自动填充）
    last_confirmed_at    TIMESTAMPTZ,
    last_successful_at   TIMESTAMPTZ,

    PRIMARY KEY (owner_user_id, agent_user_id)
);
-- PK 已覆盖所有查询模式（按 owner + agent），无需额外索引
```

---

### 组 3：执行状态（5 表）

```sql
-- ================================================================
-- agent.run_events
-- 工具调用、LLM token 用量、agent 行为的审计日志
-- 高频追加写入，seq 由 bigserial 自动分配
-- ================================================================

CREATE TABLE agent.run_events (
    seq         BIGSERIAL   PRIMARY KEY,
        -- 全局自增 seq，保证全表顺序（跨 thread 也有序）
    thread_id   TEXT        NOT NULL,
        -- → agent.threads.id（应用层 FK）
    run_id      TEXT        NOT NULL,
        -- 本次执行的 run ID（一次用户消息触发一个 run，包含多个事件）
    event_type  TEXT        NOT NULL,
        -- 'llm_call' | 'tool_call' | 'tool_result' | 'message' |
        -- 'interrupt' | 'error' | 'usage'
    data        TEXT        NOT NULL,
        -- JSON payload（TEXT 而非 JSONB：避免解析开销，按需解析）
    message_id  TEXT,
        -- 关联的 chat.messages.id（若此 event 触发了聊天消息）
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 加载 thread 的完整事件流（最常用：按 thread + seq 翻页）
CREATE INDEX idx_run_events_thread
    ON agent.run_events(thread_id, seq DESC);

-- 加载某次 run 的所有事件（调试、回放）
CREATE INDEX idx_run_events_run
    ON agent.run_events(thread_id, run_id, seq ASC);

-- 按事件类型过滤（e.g. 只看 tool_call，用量统计）
CREATE INDEX idx_run_events_thread_type
    ON agent.run_events(thread_id, event_type, seq DESC);


-- ================================================================
-- agent.summaries
-- 长对话压缩存档。token 窗口满时压缩历史对话，保留摘要继续运行
-- is_active=true 的摘要是当前 thread 加载对话历史时使用的
-- ================================================================

CREATE TABLE agent.summaries (
    summary_id          TEXT        PRIMARY KEY,
    thread_id           TEXT        NOT NULL,
        -- → agent.threads.id（应用层 FK）
    summary_text        TEXT        NOT NULL,
        -- 压缩后的摘要文本（注入 system prompt 用）
    compact_up_to_index INTEGER     NOT NULL,
        -- 压缩到第几条消息（事件 seq），用于知道从哪里开始续接
    compacted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_split_turn       BOOLEAN     NOT NULL DEFAULT false,
        -- true = 压缩发生在某轮对话中间（LangGraph 需要此标记正确恢复）
    split_turn_prefix   TEXT,
        -- 若 is_split_turn=true，记录被截断的那轮对话前缀
    is_active           BOOLEAN     NOT NULL DEFAULT true,
        -- 每个 thread 只有一条 is_active=true 的摘要（压缩后旧的标 false）
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 获取当前活跃摘要（thread 恢复时加载，极高频）
CREATE INDEX idx_summaries_thread_active
    ON agent.summaries(thread_id)
    WHERE is_active = true;


-- ================================================================
-- agent.message_queue
-- agent 间内部 steering 消息（临时队列，消费后删除）
-- 与 chat.messages 不同：这里是后台指令，不是用户可见的聊天记录
-- ================================================================

CREATE TABLE agent.message_queue (
    id                BIGSERIAL   PRIMARY KEY,
        -- 自增 ID，消费时按序（SKIP LOCKED 避免多消费者冲突）
    thread_id         TEXT        NOT NULL,
        -- 目标 thread（应用层 FK → agent.threads.id）
    content           TEXT        NOT NULL,
        -- 消息内容（JSON 或纯文本 steering 指令）
    notification_type TEXT        NOT NULL DEFAULT 'steer',
        -- 'steer'     — 用户/外部系统向 agent 发送指令
        -- 'interrupt' — 打断当前 run（优先处理）
        -- 'system'    — 系统级通知（工作区重建完成、错误通知等）
    source            TEXT,
        -- 来源标识，e.g. 'user', 'scheduler', 'parent_agent'
    sender_user_id    TEXT,
        -- 发送方 user_id（human 或 agent，应用层 FK）
    sender_name       TEXT,
        -- 发送方名称快照（sender 可能被删，记录快照）
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT message_queue_type_chk
        CHECK (notification_type IN ('steer', 'interrupt', 'system'))
);

-- 消费队列（SKIP LOCKED 模式，按 FIFO 消费）
-- interrupt 优先级高于 steer：ORDER BY notification_type='interrupt' DESC, id ASC
CREATE INDEX idx_message_queue_thread
    ON agent.message_queue(thread_id, id ASC);

-- interrupt 类型快速定位（需要优先处理）
CREATE INDEX idx_message_queue_interrupt
    ON agent.message_queue(thread_id)
    WHERE notification_type = 'interrupt';


-- ================================================================
-- agent.file_operations
-- 文件操作审计，支持"时光机"（undo/redo 文件变更）
-- 每次 agent 修改文件都记录 before/after，不能删
-- ================================================================

CREATE TABLE agent.file_operations (
    id              TEXT        PRIMARY KEY,
    thread_id       TEXT        NOT NULL,
        -- → agent.threads.id（应用层 FK）
    checkpoint_id   TEXT,
        -- 关联的 LangGraph checkpoint ID（用于时间点对齐）
    operation_type  TEXT        NOT NULL,
        -- 'create' | 'update' | 'delete' | 'rename'
    file_path       TEXT        NOT NULL,
        -- 被操作的文件路径（容器内绝对路径）
    before_content  TEXT,
        -- 操作前内容（delete/update 时记录，create 时为 NULL）
    after_content   TEXT,
        -- 操作后内容（create/update 时记录，delete 时为 NULL）
    changes         JSONB,
        -- diff 元数据（行变更统计、patch 格式等，可选）
    status          TEXT        NOT NULL DEFAULT 'applied',
        -- 'applied' | 'reverted' | 'failed'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT file_ops_type_chk
        CHECK (operation_type IN ('create', 'update', 'delete', 'rename')),
    CONSTRAINT file_ops_status_chk
        CHECK (status IN ('applied', 'reverted', 'failed'))
);

-- 按 thread + 时间倒序查看文件操作历史（时光机主查询）
CREATE INDEX idx_file_operations_thread
    ON agent.file_operations(thread_id, created_at DESC);

-- 某个文件的所有操作历史（"这个文件被改了几次？"）
CREATE INDEX idx_file_operations_thread_path
    ON agent.file_operations(thread_id, file_path, created_at DESC);


-- ================================================================
-- agent.tool_tasks
-- 长时间任务的断点恢复机制。任务可能跑一小时，崩溃后通过
-- progress_json 恢复进度，不能放内存
-- ================================================================

CREATE TABLE agent.tool_tasks (
    id            TEXT        PRIMARY KEY,
    thread_id     TEXT        NOT NULL,
        -- 所属 thread（应用层 FK）
    run_id        TEXT        NOT NULL,
        -- 所属 run（同一 run 内可有多个并行 tool_task）
    tool_name     TEXT        NOT NULL,
        -- 工具名，e.g. 'bash', 'web_search', 'code_interpreter'
    status        TEXT        NOT NULL DEFAULT 'running',
        -- 'running' | 'completed' | 'failed' | 'interrupted'
    input_json    JSONB       NOT NULL DEFAULT '{}',
        -- 工具输入参数快照（用于重试）
    output_json   JSONB,
        -- 工具输出（completed 时填入）
    progress_json JSONB       NOT NULL DEFAULT '{}',
        -- 任意进度检查点（e.g. {"step": 3, "partial_result": "..."}）
        -- 崩溃恢复时读此字段，从 step 3 续跑
    error         TEXT,
        -- 失败原因
    started_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at  TIMESTAMPTZ,
    expires_at    TIMESTAMPTZ,
        -- 超过此时间自动清理（completed/failed 任务定时删除）

    CONSTRAINT tool_tasks_status_chk
        CHECK (status IN ('running', 'completed', 'failed', 'interrupted'))
);

-- 某 thread 的运行中任务（崩溃恢复检测）
CREATE INDEX idx_tool_tasks_thread_running
    ON agent.tool_tasks(thread_id, started_at)
    WHERE status = 'running';

-- 某 run 的所有任务
CREATE INDEX idx_tool_tasks_run
    ON agent.tool_tasks(run_id);

-- 过期任务清理（定时任务扫描）
CREATE INDEX idx_tool_tasks_expires
    ON agent.tool_tasks(expires_at)
    WHERE status IN ('completed', 'failed') AND expires_at IS NOT NULL;
```

---

### 组 4：调度（2 表）

```sql
-- ================================================================
-- agent.schedules
-- 定时任务。人或 agent 均可创建（owner_user_id 可以是 human 或 agent user）
-- 原 cron_jobs 改名，语义更清晰
-- ================================================================

CREATE TABLE agent.schedules (
    id              TEXT        PRIMARY KEY,
    owner_user_id   TEXT        NOT NULL,
        -- → identity.users（人或 agent，应用层 FK）
    name            TEXT        NOT NULL,
    description     TEXT        NOT NULL DEFAULT '',
    cron_expression TEXT        NOT NULL,
        -- 标准 cron 格式：'0 9 * * 1'（每周一早 9 点）
    task_template   JSONB       NOT NULL DEFAULT '{}',
        -- 触发时执行什么，e.g.:
        -- {"type": "steer_thread", "thread_id": "...", "content": "做日报"}
        -- {"type": "create_thread", "agent_user_id": "...", "content": "..."}
    enabled         BOOLEAN     NOT NULL DEFAULT true,
    last_run_at     TIMESTAMPTZ,
        -- 最后一次触发时间
    next_run_at     TIMESTAMPTZ,
        -- 预计下次触发时间（每次 run 完成后更新，由调度器读取）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 用户的所有 schedule
CREATE INDEX idx_schedules_owner
    ON agent.schedules(owner_user_id);

-- 调度器轮询：按 next_run_at 排序，取最早到期的 enabled schedule
CREATE INDEX idx_schedules_next_run
    ON agent.schedules(next_run_at ASC)
    WHERE enabled = true;


-- ================================================================
-- agent.schedule_runs
-- 每次 schedule 触发的执行记录。schedules 只记 last_run_at 不够，
-- 需要历史（多少次成功/失败、最后一次错误是什么）
-- ================================================================

CREATE TABLE agent.schedule_runs (
    id          TEXT        PRIMARY KEY,
    schedule_id TEXT        NOT NULL,
        -- → agent.schedules.id（应用层 FK）
    thread_id   TEXT,
        -- 若此次触发创建/复用了 thread，记录 thread_id
    status      TEXT        NOT NULL DEFAULT 'running',
        -- 'running' | 'success' | 'failed' | 'timeout' | 'skipped'
        -- 'skipped'：上一次 run 未完成，跳过本次触发
    error       TEXT,
        -- 失败原因（status='failed' 时填入）
    result_json JSONB,
        -- 执行结果摘要（可选）
    started_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at TIMESTAMPTZ,

    CONSTRAINT schedule_runs_status_chk
        CHECK (status IN ('running', 'success', 'failed', 'timeout', 'skipped'))
);

-- schedule 的执行历史（最近 N 次，用于展示成功率）
CREATE INDEX idx_schedule_runs_schedule
    ON agent.schedule_runs(schedule_id, started_at DESC);

-- 关联 thread（从 thread 找是哪个 schedule 触发的）
CREATE INDEX idx_schedule_runs_thread
    ON agent.schedule_runs(thread_id)
    WHERE thread_id IS NOT NULL;
```

---

### 组 5：LangGraph 状态（4 表）

> 这 4 张表由 LangGraph 框架通过 `search_path=agent` 自动建立和维护。
> 结构必须与 LangGraph PostgreSQL checkpoint saver 的期望完全一致，不可随意增减字段。

```sql
-- ================================================================
-- agent.checkpoints
-- 对话状态存档（LangGraph managed）
-- 每个 run 步骤保存一次，支持从任意历史时间点恢复
-- ================================================================

CREATE TABLE agent.checkpoints (
    thread_id           TEXT    NOT NULL,
    checkpoint_ns       TEXT    NOT NULL DEFAULT '',
        -- namespace，区分同 thread 下不同子图的状态
    checkpoint_id       TEXT    NOT NULL,
    parent_checkpoint_id TEXT,
        -- 父 checkpoint，构成检查点链
    type                TEXT,
        -- checkpoint 序列化类型
    checkpoint          JSONB   NOT NULL,
        -- 完整的 agent 执行状态（channel values、步骤记录等）
    metadata            JSONB   NOT NULL DEFAULT '{}',
        -- 框架级元数据（run_id、step 序号等）

    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);

-- 按 thread 加载最新 checkpoint（恢复时用，LangGraph 自用）
CREATE INDEX idx_checkpoints_thread
    ON agent.checkpoints(thread_id, checkpoint_ns);


-- ================================================================
-- agent.checkpoint_blobs
-- LangGraph channel 状态的二进制存储（大 blob 独立存放）
-- ================================================================

CREATE TABLE agent.checkpoint_blobs (
    thread_id     TEXT    NOT NULL,
    checkpoint_ns TEXT    NOT NULL DEFAULT '',
    channel       TEXT    NOT NULL,
    version       TEXT    NOT NULL,
    type          TEXT    NOT NULL,
    blob          BYTEA,
        -- 二进制序列化的 channel 状态（可为 NULL 表示 channel 为空）

    PRIMARY KEY (thread_id, checkpoint_ns, channel, version)
);


-- ================================================================
-- agent.checkpoint_writes
-- LangGraph 中间写入缓冲（pending writes，run 完成前的临时状态）
-- ================================================================

CREATE TABLE agent.checkpoint_writes (
    thread_id     TEXT    NOT NULL,
    checkpoint_ns TEXT    NOT NULL DEFAULT '',
    checkpoint_id TEXT    NOT NULL,
    task_id       TEXT    NOT NULL,
    idx           INTEGER NOT NULL,
    channel       TEXT    NOT NULL,
    type          TEXT,
    blob          BYTEA   NOT NULL,
    task_path     TEXT    NOT NULL DEFAULT '',

    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);

-- 加载特定 checkpoint 的 pending writes
CREATE INDEX idx_checkpoint_writes_checkpoint
    ON agent.checkpoint_writes(thread_id, checkpoint_ns, checkpoint_id);


-- ================================================================
-- agent.checkpoint_migrations
-- LangGraph 自身的 schema 版本追踪
-- ================================================================

CREATE TABLE agent.checkpoint_migrations (
    v INTEGER PRIMARY KEY
);
```

---

## b) RLS 策略

```sql
-- ================================================================
-- 启用 RLS
-- LangGraph 表不需要 RLS（LangGraph 通过 service_role 访问，前端不直接查）
-- ================================================================

ALTER TABLE agent.agent_configs        ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.agent_rules          ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.agent_skills         ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.agent_sub_agents     ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.threads              ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.thread_launch_prefs  ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.run_events           ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.summaries            ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.message_queue        ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.file_operations      ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.tool_tasks           ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.schedules            ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent.schedule_runs        ENABLE ROW LEVEL SECURITY;
-- checkpoints / checkpoint_blobs / checkpoint_writes / checkpoint_migrations
-- 不开 RLS：仅 service_role + LangGraph 访问，前端零访问
```

---

### agent.agent_configs

```sql
-- SELECT：owner 可见自己创建的所有 agent
CREATE POLICY agent_configs_select_own ON agent.agent_configs
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- INSERT：用户创建 agent（agent_user_id 由服务端先建 identity.users，再调此 INSERT）
-- 实际流程：前端提交 agent info → mycel-agent 创建 identity.users(agent) → 回填 owner/agent_user_id → INSERT agent_config
-- 不开放 authenticated INSERT（需要跨 schema 协调 identity.users 创建）

-- UPDATE：owner 可改自己 agent 的配置
CREATE POLICY agent_configs_update_own ON agent.agent_configs
    FOR UPDATE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

-- DELETE：不开放（agent 删除需要级联清理 identity.users + 所有相关表，走 service_role）

GRANT SELECT ON agent.agent_configs TO authenticated;
GRANT UPDATE (name, description, model, tools_json, system_prompt,
              mcp_json, runtime_json, meta_json, status)
    ON agent.agent_configs TO authenticated;
-- owner_user_id / agent_user_id / version 不开放 authenticated UPDATE
```

---

### agent.agent_rules / agent_skills / agent_sub_agents

```sql
-- 三张表模式相同：owner 通过 agent_config 间接拥有

-- agent_rules
CREATE POLICY agent_rules_own ON agent.agent_rules
    FOR ALL TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.agent_configs c
            WHERE c.id = agent_config_id
              AND c.owner_user_id = (SELECT auth.uid()::text)
        )
    )
    WITH CHECK (
        EXISTS (
            SELECT 1 FROM agent.agent_configs c
            WHERE c.id = agent_config_id
              AND c.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT ALL ON agent.agent_rules TO authenticated;

-- agent_skills
CREATE POLICY agent_skills_own ON agent.agent_skills
    FOR ALL TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.agent_configs c
            WHERE c.id = agent_config_id
              AND c.owner_user_id = (SELECT auth.uid()::text)
        )
    )
    WITH CHECK (
        EXISTS (
            SELECT 1 FROM agent.agent_configs c
            WHERE c.id = agent_config_id
              AND c.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT ALL ON agent.agent_skills TO authenticated;

-- agent_sub_agents
CREATE POLICY agent_sub_agents_own ON agent.agent_sub_agents
    FOR ALL TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.agent_configs c
            WHERE c.id = agent_config_id
              AND c.owner_user_id = (SELECT auth.uid()::text)
        )
    )
    WITH CHECK (
        EXISTS (
            SELECT 1 FROM agent.agent_configs c
            WHERE c.id = agent_config_id
              AND c.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT ALL ON agent.agent_sub_agents TO authenticated;
```

---

### agent.threads

```sql
-- SELECT：owner 可见自己的所有 thread
CREATE POLICY threads_select_own ON agent.threads
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- INSERT：由 mycel-agent 后端创建（service_role），不开放前端

-- UPDATE：owner 可改 current_workspace_id（切换工作区）、model、status（归档）
-- run_status 由后端 RPC 管理，前端不能直接改
CREATE POLICY threads_update_own ON agent.threads
    FOR UPDATE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

GRANT SELECT ON agent.threads TO authenticated;
GRANT UPDATE (current_workspace_id, model, status)
    ON agent.threads TO authenticated;
-- run_status / agent_user_id / owner_user_id 不开放前端 UPDATE
```

---

### agent.thread_launch_prefs

```sql
CREATE POLICY launch_prefs_own ON agent.thread_launch_prefs
    FOR ALL TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

GRANT ALL ON agent.thread_launch_prefs TO authenticated;
```

---

### agent.run_events

```sql
-- SELECT：owner 可见自己 thread 的事件
-- 注意：run_events 高频写入，前端访问需要严格过滤（必须带 thread_id）
CREATE POLICY run_events_select_own ON agent.run_events
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.threads t
            WHERE t.id = thread_id
              AND t.owner_user_id = (SELECT auth.uid()::text)
        )
    );

-- INSERT / UPDATE / DELETE：不开放（仅 mycel-agent 通过 service_role 写入）

GRANT SELECT ON agent.run_events TO authenticated;
```

---

### agent.summaries

```sql
-- SELECT：owner 可见自己 thread 的摘要
CREATE POLICY summaries_select_own ON agent.summaries
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.threads t
            WHERE t.id = thread_id
              AND t.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT SELECT ON agent.summaries TO authenticated;
```

---

### agent.message_queue

```sql
-- message_queue 是后端内部队列，前端不需要直接访问
-- SELECT / INSERT / UPDATE / DELETE 均不开放给 authenticated
-- 全部走 mycel-agent service_role
-- （不创建任何 authenticated policy = 默认全拒绝）
```

---

### agent.file_operations

```sql
-- SELECT：owner 可见自己 thread 的文件操作（时光机 UI）
CREATE POLICY file_ops_select_own ON agent.file_operations
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.threads t
            WHERE t.id = thread_id
              AND t.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT SELECT ON agent.file_operations TO authenticated;
```

---

### agent.tool_tasks

```sql
-- SELECT：owner 可见自己 thread 的 tool_task（进度展示）
CREATE POLICY tool_tasks_select_own ON agent.tool_tasks
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.threads t
            WHERE t.id = thread_id
              AND t.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT SELECT ON agent.tool_tasks TO authenticated;
```

---

### agent.schedules

```sql
CREATE POLICY schedules_select_own ON agent.schedules
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- INSERT：用户可创建自己的 schedule
CREATE POLICY schedules_insert_own ON agent.schedules
    FOR INSERT TO authenticated
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

-- UPDATE：用户可改自己的 schedule（启停、改 cron、改 template）
CREATE POLICY schedules_update_own ON agent.schedules
    FOR UPDATE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

-- DELETE：允许用户删自己的 schedule
CREATE POLICY schedules_delete_own ON agent.schedules
    FOR DELETE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

GRANT ALL ON agent.schedules TO authenticated;
-- last_run_at / next_run_at 由调度器（service_role）写，不开放 UPDATE 给前端
REVOKE UPDATE (last_run_at, next_run_at) ON agent.schedules FROM authenticated;
```

---

### agent.schedule_runs

```sql
-- SELECT：owner 通过 schedule 间接拥有
CREATE POLICY schedule_runs_select_own ON agent.schedule_runs
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM agent.schedules s
            WHERE s.id = schedule_id
              AND s.owner_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT SELECT ON agent.schedule_runs TO authenticated;
```

---

## c) Realtime 发布清单

| 表 | Realtime | 事件 | 理由 |
|----|----------|------|------|
| `agent.threads` | ✅ 必须 | UPDATE | `run_status` 变化（idle→running→idle/error）驱动前端加载动画、状态栏 |
| `agent.tool_tasks` | ✅ 必须 | INSERT, UPDATE | 长任务创建 + 进度更新，驱动前端 loading 状态 |
| `agent.schedules` | ⚠️ 可选 | UPDATE | 调度器更新 `next_run_at` / `last_run_at`，设置页可实时显示 |
| `agent.run_events` | ❌ | — | 高频写入（每次 LLM call/tool call 都写），用 Realtime 会洪水。streaming 走 mycel-agent WS/SSE |
| `agent.message_queue` | ❌ | — | 后端内部队列，前端不订阅 |
| `agent.summaries` | ❌ | — | 低频，无实时需求 |
| LangGraph 4 表 | ❌ | — | 框架内部，前端不访问 |

```sql
ALTER PUBLICATION supabase_realtime
    ADD TABLE agent.threads, agent.tool_tasks;

-- 可选
-- ALTER PUBLICATION supabase_realtime ADD TABLE agent.schedules;
```

### 前端订阅示例

```js
// 监听 thread run_status 变化（驱动加载动画）
supabase
  .channel(`thread-${threadId}`)
  .on('postgres_changes', {
      event: 'UPDATE',
      schema: 'agent',
      table: 'threads',
      filter: `id=eq.${threadId}`
  }, (payload) => {
      const { run_status } = payload.new
      if (run_status === 'running') showSpinner()
      else if (run_status === 'error') showError()
      else hideSpinner()
  })
  .subscribe()

// 监听长任务进度
supabase
  .channel(`thread-tasks-${threadId}`)
  .on('postgres_changes', {
      event: '*',       // INSERT + UPDATE
      schema: 'agent',
      table: 'tool_tasks',
      filter: `thread_id=eq.${threadId}`
  }, (payload) => {
      updateTaskProgress(payload.new)
  })
  .subscribe()
```

---

## d) 安全审查

### 角色权限矩阵

| 角色 | agent_configs | rules/skills/sub | threads | run_events | message_queue | tool_tasks | schedules | LangGraph |
|------|---------------|-----------------|---------|------------|---------------|------------|-----------|-----------|
| `anon` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `authenticated` | SELECT/UPDATE own | ALL own (via config) | SELECT/UPDATE(workspace,model,status) own | SELECT own | ❌ | SELECT own | ALL own (excl.run_at) | ❌ |
| `service_role` | 全部 | 全部 | 全部 | 全部 | 全部 | 全部 | 全部 | 全部 |

---

### 主要风险 + 对策

#### 风险 1：run_status 伪造（最高优先级）

前端直接 UPDATE `threads.run_status` 为 'running'，欺骗其他用户或绕过限速。

**对策**：
```sql
REVOKE UPDATE (run_status) ON agent.threads FROM authenticated;
```
run_status 仅通过 `agent.set_thread_run_status` RPC 修改（见 §e），RPC 内含状态机校验。

#### 风险 2：message_queue 注入

用户直接 INSERT 进别人 thread 的 message_queue，发送恶意 interrupt 指令。

**对策**：
- message_queue 没有任何 authenticated 的 policy，默认全拒绝
- 所有写入通过 mycel-agent API（service_role）

#### 风险 3：agent_config 级联越权

rules/skills/sub_agents 的 RLS 通过 EXISTS 子查询 agent_configs，若攻击者构造 SQL 绕过子查询（不可能，PostgreSQL RLS 在 row 级别强制执行）。需确认的是 agent_config_id 的有效性——不能把规则关联到他人的 config。

**对策**：
INSERT WITH CHECK 的 EXISTS 子查询已包含 `owner_user_id = auth.uid()` 约束，无法插入到他人 config 下。

#### 风险 4：LangGraph checkpoints 含完整对话历史

checkpoints 中存有完整的 agent 执行状态，包括所有工具调用的输入输出（可能含 API key、文件内容）。

**对策**：
- 不对 LangGraph 表 `ENABLE ROW LEVEL SECURITY`，但也不 `GRANT SELECT TO authenticated`
- 前端无任何访问路径：PostgREST 需要 GRANT，没有 GRANT 就访问不到
- 后端仅通过 service_role 访问

#### 风险 5：run_events 数据量爆炸

run_events 是高频追加表，长时间运行的 agent 可能产生百万级行，SELECT 不加 thread_id 过滤会全表扫。

**对策**：
- RLS USING 子句强制 EXISTS 过滤（必须关联到自己的 thread）
- PostgREST 请求必须带 thread_id filter（API 层强制，否则拒绝）
- 考虑分区：`agent.run_events` 按 `created_at` 月分区（超出本 schema 范围，记录为 open 问题）

#### 风险 6：schedules 的 task_template 注入

task_template 是 JSONB，若包含不受控的 thread_id，用户可以调度任务 steer 任意 thread。

**对策**：
- mycel-agent 执行 schedule 时，验证 task_template 中的 thread_id 归属于 schedule.owner_user_id
- `next_run_at` / `last_run_at` 不开放前端 UPDATE（防止用户直接触发执行）

#### 风险 7：tool_tasks.input_json 含敏感数据

input_json 记录工具调用的完整输入，可能包含文件内容、搜索 query、外部 API 参数。

**对策**：
- 前端只有 SELECT（通过 RLS 只能看自己的任务），无法看到他人的 input_json
- 如需要，可在 API 层掩码处理（返回时 input_json 替换为 `{"summarized": "..."}`)

---

## e) RPCs

### RPC 1：agent.set_thread_run_status

状态机管控 run_status 变更，防止非法状态转换（e.g., error → running 必须先 reset）。

```sql
CREATE OR REPLACE FUNCTION agent.set_thread_run_status(
    p_thread_id TEXT,
    p_new_status TEXT,
    p_run_id     TEXT DEFAULT NULL  -- running 时关联 run_id，便于审计
)
RETURNS TABLE(
    ok      BOOLEAN,
    message TEXT
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_current TEXT;
    v_rows    INTEGER;
BEGIN
    -- 校验目标值域
    IF p_new_status NOT IN ('idle', 'running', 'paused', 'error') THEN
        RETURN QUERY SELECT false, 'invalid_status'::text;
        RETURN;
    END IF;

    -- 获取当前状态
    SELECT run_status INTO v_current
    FROM agent.threads WHERE id = p_thread_id;

    IF NOT FOUND THEN
        RETURN QUERY SELECT false, 'thread_not_found'::text;
        RETURN;
    END IF;

    -- 状态机合法转换：
    --   idle    → running (新 run 开始)
    --   running → idle    (run 正常完成)
    --   running → error   (run 出错)
    --   running → paused  (用户中断)
    --   paused  → running (用户恢复)
    --   error   → idle    (用户重置后可重新 run)
    IF NOT (
        (v_current = 'idle'    AND p_new_status = 'running') OR
        (v_current = 'running' AND p_new_status IN ('idle', 'error', 'paused')) OR
        (v_current = 'paused'  AND p_new_status = 'running') OR
        (v_current = 'error'   AND p_new_status = 'idle')
    ) THEN
        RETURN QUERY SELECT false,
            format('invalid_transition: %s -> %s', v_current, p_new_status)::text;
        RETURN;
    END IF;

    UPDATE agent.threads
    SET
        run_status     = p_new_status,
        last_active_at = CASE WHEN p_new_status IN ('idle', 'error') THEN now() ELSE last_active_at END,
        updated_at     = now()
    WHERE id = p_thread_id;

    GET DIAGNOSTICS v_rows = ROW_COUNT;
    RETURN QUERY SELECT (v_rows > 0), 'ok'::text;
END;
$$;

-- mycel-agent 后端调用（service_role），不开放前端
```

---

### RPC 2：agent.dequeue_messages

原子消费 message_queue（SKIP LOCKED 避免多消费者冲突）。

```sql
CREATE OR REPLACE FUNCTION agent.dequeue_messages(
    p_thread_id TEXT,
    p_limit     INTEGER DEFAULT 10
)
RETURNS TABLE(
    id                BIGINT,
    content           TEXT,
    notification_type TEXT,
    source            TEXT,
    sender_user_id    TEXT,
    sender_name       TEXT,
    created_at        TIMESTAMPTZ
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_ids BIGINT[];
BEGIN
    -- 获取并锁定，跳过已被其他 worker 锁定的行
    SELECT array_agg(q.id) INTO v_ids
    FROM (
        SELECT mq.id
        FROM agent.message_queue mq
        WHERE mq.thread_id = p_thread_id
        ORDER BY
            -- interrupt 优先
            CASE WHEN mq.notification_type = 'interrupt' THEN 0 ELSE 1 END,
            mq.id ASC
        LIMIT p_limit
        FOR UPDATE SKIP LOCKED
    ) q;

    IF v_ids IS NULL THEN
        RETURN;
    END IF;

    -- 返回消息内容
    RETURN QUERY
        SELECT mq.id, mq.content, mq.notification_type,
               mq.source, mq.sender_user_id, mq.sender_name, mq.created_at
        FROM agent.message_queue mq
        WHERE mq.id = ANY(v_ids)
        ORDER BY
            CASE WHEN mq.notification_type = 'interrupt' THEN 0 ELSE 1 END,
            mq.id ASC;

    -- 消费即删除
    DELETE FROM agent.message_queue WHERE id = ANY(v_ids);
END;
$$;
```

---

### RPC 3：agent.tick_schedules

调度器心跳：获取到期的 schedule 并原子标记为"处理中"，防止并发重复触发。

```sql
CREATE OR REPLACE FUNCTION agent.tick_schedules(
    p_now   TIMESTAMPTZ DEFAULT now(),
    p_limit INTEGER     DEFAULT 20
)
RETURNS TABLE(
    schedule_id     TEXT,
    owner_user_id   TEXT,
    name            TEXT,
    task_template   JSONB,
    run_id          TEXT  -- 新建的 schedule_run.id
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_schedule RECORD;
    v_run_id   TEXT;
BEGIN
    FOR v_schedule IN
        SELECT s.id, s.owner_user_id, s.name, s.task_template, s.cron_expression
        FROM agent.schedules s
        WHERE s.enabled = true
          AND s.next_run_at <= p_now
          -- 跳过有正在运行的 run 的 schedule（防重入）
          AND NOT EXISTS (
              SELECT 1 FROM agent.schedule_runs sr
              WHERE sr.schedule_id = s.id AND sr.status = 'running'
          )
        ORDER BY s.next_run_at ASC
        LIMIT p_limit
        FOR UPDATE OF s SKIP LOCKED
    LOOP
        -- 创建 run 记录
        v_run_id := gen_random_uuid()::text;
        INSERT INTO agent.schedule_runs (id, schedule_id, status, started_at)
        VALUES (v_run_id, v_schedule.id, 'running', p_now);

        -- 更新 last_run_at（next_run_at 由调度器在 run 完成后根据 cron_expression 计算并更新）
        UPDATE agent.schedules
        SET last_run_at = p_now, updated_at = p_now
        WHERE id = v_schedule.id;

        RETURN QUERY SELECT
            v_schedule.id,
            v_schedule.owner_user_id,
            v_schedule.name,
            v_schedule.task_template,
            v_run_id;
    END LOOP;
END;
$$;
```

---

### RPC 4：agent.complete_schedule_run

标记 schedule_run 完成，同时更新 next_run_at。

```sql
CREATE OR REPLACE FUNCTION agent.complete_schedule_run(
    p_run_id      TEXT,
    p_status      TEXT,         -- 'success' | 'failed' | 'timeout'
    p_next_run_at TIMESTAMPTZ,  -- 由调度器根据 cron_expression 计算
    p_error       TEXT DEFAULT NULL,
    p_result_json JSONB DEFAULT NULL,
    p_thread_id   TEXT DEFAULT NULL
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
    -- 完成 run 记录
    UPDATE agent.schedule_runs
    SET
        status      = p_status,
        error       = p_error,
        result_json = p_result_json,
        thread_id   = p_thread_id,
        finished_at = now()
    WHERE id = p_run_id;

    -- 更新 schedule 的 next_run_at
    UPDATE agent.schedules s
    SET
        next_run_at = p_next_run_at,
        updated_at  = now()
    FROM agent.schedule_runs sr
    WHERE sr.id          = p_run_id
      AND sr.schedule_id = s.id;
END;
$$;
```

---

### RPC 权限汇总

| RPC | 调用方 | 说明 |
|-----|--------|------|
| `agent.set_thread_run_status(TEXT, TEXT, TEXT)` | service_role | 含状态机校验，前端不可直接调 |
| `agent.dequeue_messages(TEXT, INTEGER)` | service_role | SKIP LOCKED 原子消费，内部队列 |
| `agent.tick_schedules(TIMESTAMPTZ, INTEGER)` | service_role | 调度器心跳，SKIP LOCKED 防重入 |
| `agent.complete_schedule_run(TEXT, TEXT, TIMESTAMPTZ, TEXT, JSONB, TEXT)` | service_role | 完成 run + 更新 next_run_at |

---

## 未解决 / 需后续确认

1. **run_events 分区**：高频追加表，随时间增长无界。建议按月分区（`PARTITION BY RANGE (created_at)`），但 Supabase 的 PostgREST 对分区表的行为需验证（特别是 SELECT 查询是否自动路由到正确分区）。

2. **agent_configs INSERT 流程**：创建一个 agent 需要跨 schema 协调（先 INSERT identity.users(type='agent') → 然后 INSERT agent.agent_configs）。两步操作需要在事务内完成，但跨 schema 事务只有 service_role 能执行。需要 mycel-agent 提供一个"创建 agent"的 API endpoint，不能让前端分两步 INSERT。

3. **checkpoints 数据增长**：每次 agent run 的每个步骤都写一次 checkpoint，长期运行会无界增长。需要定期归档/清理策略（保留最新 N 个 checkpoint 或按时间 TTL），LangGraph 本身没有自动清理机制，需要外部定时任务。

4. **threads.owner_user_id 写入时机**：threads 表设计了 `owner_user_id` 冗余字段，来自 `identity.users(agent_user_id).owner_user_id`。需要 mycel-agent 在 INSERT threads 时同步查询并写入，确保数据一致性。后续如果 agent 的 owner 变更（agent transfer，虽然不支持），此字段会失准，需要同步更新。
