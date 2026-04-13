# container schema — 三核心表 DDL

> 2026-04-13，基于 32-schema-decisions.md 设计
>
> **命名决策**：中间层采用 `environments`（非 `containers`）。
> 32-schema-decisions.md 中写的是 `container.containers`，但会造成 schema/table 同名混淆；
> 33-open-questions.md 和任务 prompt 均使用 `environments`，语义上也更准确（隔离执行环境）。
> 其他依赖此表的表（terminals、terminal_sessions 等）字段名应从 `container_id` 改为 `environment_id`，需后续对齐。

---

```sql
-- ================================================================
-- Schema: container
-- 三层模型: devices → environments → workspaces / terminals
-- 跨 schema FK 统一用应用层校验，不加 DB 外键约束
-- ================================================================

CREATE SCHEMA IF NOT EXISTS container;


-- ================================================================
-- 1. container.devices
--    用户的计算端点，持久存在，有心跳 / 在线状态
--    push_token 归此表（设备属性，chat.message_deliveries 通过 device_id 关联）
-- ================================================================

CREATE TABLE container.devices (
    id                TEXT        PRIMARY KEY,
    -- 归属
    owner_user_id     TEXT        NOT NULL,
        -- → identity.users（应用层，无 DB FK）
    -- 设备身份
    name              TEXT        NOT NULL,
        -- 用户自定义名称，e.g. "我的 MacBook Pro"；同用户下唯一
    device_type       TEXT        NOT NULL,
        -- 'desktop' | 'server' | 'cloud_vm'
    platform          TEXT        NOT NULL,
        -- 'macos' | 'linux' | 'windows'
    arch              TEXT,
        -- CPU 架构，'arm64' | 'x86_64'；可选，便于 recipe 匹配
    hostname          TEXT,
        -- 主机名，辅助识别，不唯一（同一主机可能重装多次）
    -- 能力声明（动态结构，避免 ALTER TABLE）
    capabilities      JSONB       NOT NULL DEFAULT '{}',
        -- e.g. {"docker": true, "gpu": "A100", "cpus": 8, "ram_gb": 16}
        -- sandbox-daemon 注册时上报，控制面用于 recipe 匹配
    -- 推送（驱动 chat.message_deliveries 中离线设备的 push 通知）
    push_token        TEXT,
        -- APNs / FCM / Web Push 推送 token
    push_platform     TEXT,
        -- 'apns' | 'fcm' | 'web_push'；token 存在时必填
    -- 在线状态
    status            TEXT        NOT NULL DEFAULT 'offline',
        -- 'online' | 'offline' | 'disabled'
        -- 'disabled'：管理员封禁，不允许心跳激活
    last_heartbeat_at TIMESTAMPTZ,
        -- daemon 定期上报心跳；用于判断设备是否超时掉线
    -- 并发控制
    version           INTEGER     NOT NULL DEFAULT 0,
        -- 乐观锁：心跳 / 状态更新前须携带当前 version，服务端 CAS 更新
    -- 时间戳
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- 值域约束
    CONSTRAINT devices_status_chk
        CHECK (status IN ('online', 'offline', 'disabled')),
    CONSTRAINT devices_type_chk
        CHECK (device_type IN ('desktop', 'server', 'cloud_vm')),
    CONSTRAINT devices_platform_chk
        CHECK (platform IN ('macos', 'linux', 'windows')),
    CONSTRAINT devices_push_platform_chk
        CHECK (push_platform IS NULL
            OR push_platform IN ('apns', 'fcm', 'web_push')),
    -- push_token 和 push_platform 必须同时有或同时无
    CONSTRAINT devices_push_pair_chk
        CHECK ((push_token IS NULL) = (push_platform IS NULL))
);

-- 同用户下设备名唯一（用户侧识别）
CREATE UNIQUE INDEX uq_devices_owner_name
    ON container.devices(owner_user_id, name);

-- 用户的所有设备（设备列表、注册）
CREATE INDEX idx_devices_owner
    ON container.devices(owner_user_id);

-- 在线设备（心跳路由、推送路由；过滤非在线行）
CREATE INDEX idx_devices_online
    ON container.devices(owner_user_id, last_heartbeat_at DESC)
    WHERE status = 'online';


-- ================================================================
-- 2. container.environments
--    设备上的隔离执行环境（原 staging.sandbox_instances 重构）
--    注：32-schema-decisions.md 中此表名为 containers；
--        33-open-questions.md 和任务 prompt 均用 environments，
--        采用后者以避免 container.containers 同名混淆。
-- ================================================================

CREATE TABLE container.environments (
    id               TEXT        PRIMARY KEY,
    -- 归属
    device_id        TEXT        NOT NULL,
        -- → container.devices.id（应用层）；环境属于设备
    owner_user_id    TEXT        NOT NULL,
        -- → identity.users（冗余，便于 RLS 和跨设备查询）
    -- 环境身份
    name             TEXT,
        -- 用户可选命名，e.g. "Python 3.12 dev"；可为空
    provider_name    TEXT        NOT NULL,
        -- 执行提供商：'local' | 'docker' | 'fly' | 'e2b' | 'ssh'
    provider_env_id  TEXT,
        -- 提供商侧唯一标识（Docker container ID、Fly machine ID 等）
        -- 同设备下同提供商 env 不重复注册（见 uq_environments_device_provider）
    image            TEXT,
        -- 基础镜像，e.g. "ubuntu:24.04" / "python:3.12-slim"
        -- local 类型可为空
    -- 双轨状态机：desired（意图）由控制面写；observed（实际）由 daemon 上报
    desired_state    TEXT        NOT NULL DEFAULT 'running',
        -- 'running' | 'stopped'
    observed_state   TEXT        NOT NULL DEFAULT 'detached',
        -- 'running' | 'stopped' | 'creating' | 'detached' | 'error'
        -- 'detached'：daemon 尚未建立连接 / 未上报
    -- 记录生命周期（独立于运行状态）
    status           TEXT        NOT NULL DEFAULT 'active',
        -- 'active' | 'archived' | 'deleted'
        -- 'archived'：已停用但保留数据；'deleted'：软删除
    -- 健康监控
    last_seen_at     TIMESTAMPTZ,
        -- 最后一次存活探针时间（provider 上报或 daemon ping）
    last_error       TEXT,
        -- 最后一次错误原因，展示给用户
    -- 并发控制
    version          INTEGER     NOT NULL DEFAULT 0,
        -- 乐观锁：daemon 上报状态时携带 version，防止并发覆写
    -- 时间戳
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- 值域约束
    CONSTRAINT environments_desired_state_chk
        CHECK (desired_state IN ('running', 'stopped')),
    CONSTRAINT environments_observed_state_chk
        CHECK (observed_state IN ('running', 'stopped', 'creating', 'detached', 'error')),
    CONSTRAINT environments_status_chk
        CHECK (status IN ('active', 'archived', 'deleted'))
);

-- 同设备下同提供商环境不重复注册
CREATE UNIQUE INDEX uq_environments_device_provider
    ON container.environments(device_id, provider_env_id)
    WHERE provider_env_id IS NOT NULL;

-- 设备的所有环境（最常用：展示设备下环境列表）
CREATE INDEX idx_environments_device
    ON container.environments(device_id)
    WHERE status = 'active';

-- 用户的所有活跃环境（RLS / 跨设备查询）
CREATE INDEX idx_environments_owner_active
    ON container.environments(owner_user_id)
    WHERE status = 'active';

-- 状态不一致的环境（控制面轮询：desired ≠ observed 需要协调）
CREATE INDEX idx_environments_state_mismatch
    ON container.environments(updated_at)
    WHERE desired_state != observed_state AND status = 'active';


-- ================================================================
-- 3. container.workspaces
--    环境内的项目工作区（原 staging.sandbox_leases 重构）
--    agent.threads 通过 threads.current_workspace_id 连接此表
--    thread ↔ workspace 是可切换的连接关系，非 1:1 绑定
-- ================================================================

CREATE TABLE container.workspaces (
    id               TEXT        PRIMARY KEY,
    -- 归属
    environment_id   TEXT        NOT NULL,
        -- → container.environments.id（应用层）；工作区在环境内
    owner_user_id    TEXT        NOT NULL,
        -- → identity.users（冗余，便于 RLS 和直接查用户工作区）
    -- 工作区身份
    name             TEXT,
        -- 用户可选命名，e.g. "mycel-backend"
    workspace_path   TEXT,
        -- 容器内绝对路径，e.g. "/workspace/mycel-backend"
        -- 同环境下路径唯一（见 uq_workspaces_env_path）
    -- 启动模板（创建时可基于 recipe，快照存档以支持重建）
    recipe_id        TEXT,
        -- → container.workspace_recipes.id（应用层）；可为空（手动创建）
    recipe_snapshot  JSONB,
        -- 创建时的 recipe 快照，recipe 后续变动不影响已创建工作区
        -- 触发重建时以此为参考
    -- 存储绑定
    volume_id        TEXT,
        -- → container.volumes.id（应用层）；数据卷归 device，workspace 可重建
    -- 双轨状态机（与 environments 对齐）
    desired_state    TEXT        NOT NULL DEFAULT 'running',
        -- 'running' | 'stopped'
    observed_state   TEXT        NOT NULL DEFAULT 'detached',
        -- 'running' | 'stopped' | 'creating' | 'detached' | 'error'
    -- 记录生命周期
    status           TEXT        NOT NULL DEFAULT 'active',
        -- 'active' | 'archived' | 'deleted'
    -- 刷新机制（recipe 更新、环境重建后触发）
    needs_refresh    BOOLEAN     NOT NULL DEFAULT false,
        -- true 时控制面应在 refresh_hint_at 之后调度重建
    refresh_hint_at  TIMESTAMPTZ,
        -- 建议重建时间（可延迟执行，允许用户手动推迟）
    -- 错误追踪
    last_error       TEXT,
        -- 最后一次操作失败原因，展示给用户
    -- 并发控制
    version          INTEGER     NOT NULL DEFAULT 0,
        -- 乐观锁：所有状态更新携带 version，服务端 CAS 更新
    -- 时间戳
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- 值域约束
    CONSTRAINT workspaces_desired_state_chk
        CHECK (desired_state IN ('running', 'stopped')),
    CONSTRAINT workspaces_observed_state_chk
        CHECK (observed_state IN ('running', 'stopped', 'creating', 'detached', 'error')),
    CONSTRAINT workspaces_status_chk
        CHECK (status IN ('active', 'archived', 'deleted'))
);

-- 同环境下路径唯一（排除已删除，允许路径复用）
CREATE UNIQUE INDEX uq_workspaces_env_path
    ON container.workspaces(environment_id, workspace_path)
    WHERE workspace_path IS NOT NULL AND status != 'deleted';

-- 环境的所有工作区（展示工作区列表、terminal 查询）
CREATE INDEX idx_workspaces_environment
    ON container.workspaces(environment_id)
    WHERE status = 'active';

-- 用户的所有活跃工作区（agent.threads 连接查询、RLS）
CREATE INDEX idx_workspaces_owner_active
    ON container.workspaces(owner_user_id)
    WHERE status = 'active';

-- 待刷新工作区（控制面调度扫描）
CREATE INDEX idx_workspaces_needs_refresh
    ON container.workspaces(refresh_hint_at ASC)
    WHERE needs_refresh = true AND status = 'active';

-- 状态不一致的工作区（控制面轮询，与 environments 同模式）
CREATE INDEX idx_workspaces_state_mismatch
    ON container.workspaces(updated_at)
    WHERE desired_state != observed_state AND status = 'active';
```
