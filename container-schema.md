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
# container schema 完整审查

> 2026-04-13，针对 devices / environments / workspaces 三张表
> 基于 32-schema-decisions.md + 34-container-core-ddl.md

---

## 架构前提（影响所有设计决策）

```
sandbox-daemon (用户机器)
    │ HTTP/WS
    ▼
mycel-container (中心服务，service_role 直连 Supabase)
    │ service_role（绕过 RLS）
    ▼
Supabase Postgres

前端 (SPA)
    │ anon key + JWT（受 RLS 约束）
    ▼
Supabase PostgREST / Realtime
```

- **daemon 不直连 Supabase**，通过 mycel-container API 汇报状态
- **mycel-container 使用 service_role**，所有后端写入绕过 RLS
- **RLS 约束前端**：保护 PostgREST 直查和 Realtime 订阅

---

## a) RLS 策略

### 启用 RLS

```sql
ALTER TABLE container.devices      ENABLE ROW LEVEL SECURITY;
ALTER TABLE container.environments ENABLE ROW LEVEL SECURITY;
ALTER TABLE container.workspaces   ENABLE ROW LEVEL SECURITY;

-- service_role 自动绕过 RLS，无需额外配置
-- anon 角色：不设任何 GRANT，相当于零访问
```

### container.devices

```sql
-- SELECT：只能看自己的设备
CREATE POLICY devices_select_own ON container.devices
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- INSERT：只能为自己注册设备，且不能直接设为 disabled
-- （disabled 是管理员操作，只有 service_role 能写）
CREATE POLICY devices_insert_own ON container.devices
    FOR INSERT TO authenticated
    WITH CHECK (
        owner_user_id = (SELECT auth.uid()::text)
        AND status != 'disabled'
    );

-- UPDATE：只能改自己的设备，且不能自己把 status 改成 disabled
-- 用户合法操作：改 name、改 push_token、改 capabilities
-- 用户不合法操作：改 status（由后端控制）、改 version（由 RPC 控制）
CREATE POLICY devices_update_own ON container.devices
    FOR UPDATE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (
        owner_user_id = (SELECT auth.uid()::text)
        AND status != 'disabled'
    );

-- DELETE：不允许前端直接删除（防误删，改 status='deleted' 代替）
-- 真实删除由 mycel-container 后端执行
-- 不创建 DELETE policy = 默认拒绝
```

> **注**：`status` 的 online/offline 切换由后端 RPC 管理（见 §d），前端不应直接 UPDATE status。
> 但因为 RLS 无法区分"改哪个字段"，这一约束靠应用层 + column privilege 实现：
> `REVOKE UPDATE (status, version) ON container.devices FROM authenticated;`

```sql
-- 细粒度列权限：前端可改的列
GRANT SELECT ON container.devices TO authenticated;
GRANT INSERT ON container.devices TO authenticated;
-- UPDATE 只开放用户侧字段，status/version/last_heartbeat_at 由后端管理
GRANT UPDATE (name, push_token, push_platform, capabilities)
    ON container.devices TO authenticated;
```

---

### container.environments

```sql
-- SELECT：只能看自己的环境
CREATE POLICY environments_select_own ON container.environments
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- INSERT：前端不能直接创建环境
-- 环境由 mycel-container 后端创建（需要验证 device 归属 + 调用 provider API）
-- 不创建 INSERT policy = 默认拒绝

-- UPDATE：前端只能改 desired_state（用户点启动/停止）和 name
-- observed_state / version / last_error / last_seen_at 由后端写
CREATE POLICY environments_update_own ON container.environments
    FOR UPDATE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

GRANT SELECT ON container.environments TO authenticated;
-- UPDATE 只开放意图字段
GRANT UPDATE (name, desired_state)
    ON container.environments TO authenticated;

-- DELETE：不允许前端直接删，走 desired_state='stopped' + status='deleted' 软删流程
```

> **desired_state 的额外约束**：前端 UPDATE desired_state 时，必须满足当前 status='active'。
> 靠 RLS USING 子句实现：

```sql
-- 替换上面的 environments_update_own，加 status 检查
DROP POLICY IF EXISTS environments_update_own ON container.environments;

CREATE POLICY environments_update_own ON container.environments
    FOR UPDATE TO authenticated
    USING (
        owner_user_id = (SELECT auth.uid()::text)
        AND status = 'active'           -- 不能操作已归档/删除的环境
    )
    WITH CHECK (
        owner_user_id = (SELECT auth.uid()::text)
        AND status = 'active'
        AND desired_state IN ('running', 'stopped')  -- 值域在 CHECK CONSTRAINT 里也有，双保险
    );
```

---

### container.workspaces

```sql
-- SELECT：只能看自己的工作区
CREATE POLICY workspaces_select_own ON container.workspaces
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- INSERT：前端可以在自己的环境里创建工作区
-- 需要验证 environment_id 确实属于当前用户（防止越权关联他人环境）
CREATE POLICY workspaces_insert_own ON container.workspaces
    FOR INSERT TO authenticated
    WITH CHECK (
        owner_user_id = (SELECT auth.uid()::text)
        AND EXISTS (
            SELECT 1 FROM container.environments e
            WHERE e.id = environment_id
              AND e.owner_user_id = (SELECT auth.uid()::text)
              AND e.status = 'active'
        )
    );

-- UPDATE：前端可改 name、desired_state、workspace_path（初始设置时）
CREATE POLICY workspaces_update_own ON container.workspaces
    FOR UPDATE TO authenticated
    USING (
        owner_user_id = (SELECT auth.uid()::text)
        AND status = 'active'
    )
    WITH CHECK (
        owner_user_id = (SELECT auth.uid()::text)
        AND status = 'active'
    );

GRANT SELECT ON container.workspaces TO authenticated;
GRANT INSERT ON container.workspaces TO authenticated;
GRANT UPDATE (name, desired_state, workspace_path)
    ON container.workspaces TO authenticated;
```

---

## b) Realtime 发布清单

### 总表

| 表 | Realtime | 事件 | 理由 |
|----|----------|------|------|
| `container.devices` | ✅ 必须 | UPDATE | 在线/离线状态变更需实时推送到前端（设备面板、推送路由） |
| `container.environments` | ✅ 必须 | INSERT, UPDATE | 环境创建进度 + 状态机变更（starting → running / error） |
| `container.workspaces` | ✅ 必须 | INSERT, UPDATE | 工作区启动状态 + needs_refresh 触发 |

DELETE 三张表均不开 Realtime（统一用软删除 `status='deleted'` 触发 UPDATE 事件，减少订阅复杂度）。

### 加入 publication

```sql
-- 前提：Supabase 默认 publication 是 supabase_realtime
-- 如果 container schema 未加入，需要显式添加

ALTER PUBLICATION supabase_realtime
    ADD TABLE container.devices, container.environments, container.workspaces;
```

### 各表订阅策略

#### container.devices — UPDATE only

```sql
-- 前端订阅：监听自己设备的状态变化
-- Supabase JS:
-- supabase
--   .channel('my-devices')
--   .on('postgres_changes', {
--       event: 'UPDATE',
--       schema: 'container',
--       table: 'devices',
--       filter: `owner_user_id=eq.${userId}`
--   }, handler)
--   .subscribe()
```

- 推送 INSERT 无意义（用户自己创建，本地已知）
- 核心场景：daemon 上线 → status: offline→online，前端设备面板实时更新
- **RLS 过滤**：Supabase Realtime 对 `postgres_changes` 自动应用 RLS，用户只收到自己的设备变更

#### container.environments — INSERT + UPDATE

```sql
-- INSERT：用户发起创建后，后端异步建环境，前端监听创建完成
-- UPDATE：observed_state 变化（detached→creating→running / error）
-- Supabase JS:
-- supabase
--   .channel('my-environments')
--   .on('postgres_changes', {
--       event: '*',                          -- INSERT + UPDATE
--       schema: 'container',
--       table: 'environments',
--       filter: `owner_user_id=eq.${userId}`
--   }, handler)
--   .subscribe()
```

- 关键 UX：用户点"启动环境"后页面不轮询，靠 Realtime 推 observed_state='running' 才解锁操作
- last_error 变化也通过 UPDATE 推送（错误即时展示）

#### container.workspaces — INSERT + UPDATE

```sql
-- 同 environments 模式
-- 额外关注：needs_refresh=true 变为 false（重建完成，前端恢复操作入口）
-- Supabase JS filter 同上，table 改为 'workspaces'
```

---

## c) 安全审查

### 角色权限矩阵

| 角色 | devices | environments | workspaces |
|------|---------|--------------|------------|
| `anon` | ❌ 无任何访问 | ❌ | ❌ |
| `authenticated` | SELECT own / INSERT own / UPDATE(name,push_token,capabilities) own | SELECT own / UPDATE(name,desired_state) own | SELECT own / INSERT own / UPDATE(name,desired_state,workspace_path) own |
| `service_role` (mycel-container) | 全部 | 全部 | 全部 |

### 主要安全风险 + 对策

#### 风险 1：owner_user_id 伪造
用户在 INSERT 时将 `owner_user_id` 设为他人 ID，冒充拥有者。

**对策**：
- `INSERT WITH CHECK (owner_user_id = auth.uid()::text)` 硬拦截
- workspaces INSERT 额外校验 environment 归属（防止关联他人环境）

#### 风险 2：越权操作他人设备/环境
前端直接调 PostgREST UPDATE 修改他人记录。

**对策**：
- 所有 UPDATE policy 均有 `USING (owner_user_id = auth.uid()::text)`
- PostgREST 在 WHERE 子句中自动注入 RLS，无法绕过

#### 风险 3：status 提权（disabled 绕过）
用户通过 UPDATE 将设备 status 从 offline 改为 disabled，或反向。

**对策**：
- `REVOKE UPDATE (status) ON container.devices FROM authenticated`
- INSERT WITH CHECK 拒绝直接注入 `status='disabled'`
- status 变更仅通过 service_role 执行的 RPC 完成

#### 风险 4：心跳 DDoS
daemon 高频上报心跳，写入量爆炸。

**对策**：
- 心跳 RPC 内含节流逻辑：`IF now() - last_heartbeat_at < interval '10 seconds' THEN RETURN`（见 §d）
- mycel-container API 层限流：每设备 1 req/10s
- 不给 authenticated 角色直接 UPDATE last_heartbeat_at 的权限

#### 风险 5：capabilities JSONB 注入
用户上报恶意 JSON 影响 recipe 匹配逻辑。

**对策**：
- mycel-container 应用层 schema 验证（Pydantic model）
- 允许任意 JSON 结构，但提取已知字段时用 `->>'docker'` 而非 eval
- DB 层无法做深度 JSON 校验，应用层是正确的校验边界

#### 风险 6：provider_env_id 伪造
用户提交虚假 provider_env_id 声称拥有某个 Fly machine。

**对策**：
- `provider_env_id` 应由 mycel-container 写入（service_role），不开放给 authenticated
- `REVOKE UPDATE (provider_env_id, provider_name) ON container.environments FROM authenticated`

#### 风险 7：Realtime 数据泄露
Supabase Realtime 使用 anon key，无 RLS = 全表广播。

**对策**：
- 三张表均 ENABLE ROW LEVEL SECURITY（见 §a）
- Supabase `postgres_changes` 在 broadcast 前应用 RLS，消息只到达有权限的订阅者
- **验证**：上线前用不同用户 JWT 各自订阅，确认只收到自己的数据

#### 风险 8：push_token 泄露
push_token 是敏感设备凭证，不应出现在 SELECT * 返回中。

**对策**：
- 应用层 API 返回设备信息时 exclude push_token 字段
- 或创建去敏感化 VIEW：`CREATE VIEW container.devices_public AS SELECT id, owner_user_id, name, device_type, platform, status, ... FROM container.devices;`（不含 push_token）

### 输入校验边界

以下字段必须在 mycel-container 应用层（而非 DB 层）校验：

| 字段 | 校验规则 |
|------|---------|
| `name` | 非空，长度 ≤ 100，无 SQL/XSS 注入字符（Pydantic str 即可） |
| `capabilities` | JSON 对象，已知 key 类型强校验（`docker: bool`，`cpus: int > 0` 等） |
| `workspace_path` | 必须是绝对路径，不含 `..`（路径穿越防护） |
| `image` | 格式 `registry/name:tag`，不允许执行特殊字符 |
| `provider_env_id` | 仅 alphanumeric + `-_`，长度限制 |

---

## d) RPCs

### RPC 1：container.device_heartbeat

daemon 定期（每 30s）调用，原子更新心跳 + 乐观锁 CAS。

```sql
CREATE OR REPLACE FUNCTION container.device_heartbeat(
    p_device_id    TEXT,
    p_version      INTEGER,
    p_capabilities JSONB DEFAULT NULL  -- 可选，daemon 感知能力变化时上报
)
RETURNS TABLE(
    ok         BOOLEAN,
    new_version INTEGER,
    message    TEXT
)
LANGUAGE plpgsql
SECURITY DEFINER  -- 以函数 owner 权限运行，绕过 RLS（daemon 通过 API 调）
SET search_path = container, pg_catalog
AS $$
DECLARE
    v_rows INTEGER;
BEGIN
    -- 节流：10s 内不重复写（防 DDoS）
    IF EXISTS (
        SELECT 1 FROM container.devices
        WHERE id = p_device_id
          AND last_heartbeat_at > now() - interval '10 seconds'
          AND version = p_version
    ) THEN
        RETURN QUERY SELECT true, p_version, 'throttled'::text;
        RETURN;
    END IF;

    -- CAS 更新
    UPDATE container.devices
    SET
        status            = 'online',
        last_heartbeat_at = now(),
        capabilities      = COALESCE(p_capabilities, capabilities),
        version           = version + 1,
        updated_at        = now()
    WHERE id      = p_device_id
      AND version = p_version
      AND status != 'disabled';  -- 被封禁的设备不能通过心跳复活

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    IF v_rows = 0 THEN
        -- 可能是 version 不匹配 或 status=disabled 或 id 不存在
        RETURN QUERY SELECT false, p_version, 'conflict_or_disabled'::text;
    ELSE
        RETURN QUERY SELECT true, p_version + 1, 'ok'::text;
    END IF;
END;
$$;

-- 调用示例（mycel-container 后端）：
-- SELECT * FROM container.device_heartbeat('dev_abc', 42);
-- SELECT * FROM container.device_heartbeat('dev_abc', 42, '{"docker": true, "cpus": 8}');
```

---

### RPC 2：container.environment_observe

daemon 上报 environment 实际状态，后端读回 desired_state 作为指令。

```sql
CREATE OR REPLACE FUNCTION container.environment_observe(
    p_env_id        TEXT,
    p_version       INTEGER,
    p_observed      TEXT,      -- 实际观测到的状态
    p_last_error    TEXT DEFAULT NULL
)
RETURNS TABLE(
    ok            BOOLEAN,
    new_version   INTEGER,
    desired_state TEXT,         -- 返回控制面期望的状态，daemon 据此决定下一步
    message       TEXT
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = container, pg_catalog
AS $$
DECLARE
    v_rows        INTEGER;
    v_desired     TEXT;
BEGIN
    -- 校验 observed 值域
    IF p_observed NOT IN ('running', 'stopped', 'creating', 'detached', 'error') THEN
        RETURN QUERY SELECT false, p_version, NULL::text, 'invalid_observed_state'::text;
        RETURN;
    END IF;

    UPDATE container.environments
    SET
        observed_state = p_observed,
        last_seen_at   = now(),
        last_error     = CASE WHEN p_observed = 'error' THEN p_last_error ELSE NULL END,
        version        = version + 1,
        updated_at     = now()
    WHERE id      = p_env_id
      AND version = p_version
      AND status  = 'active'
    RETURNING desired_state INTO v_desired;

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    IF v_rows = 0 THEN
        RETURN QUERY SELECT false, p_version, NULL::text, 'conflict_or_inactive'::text;
    ELSE
        RETURN QUERY SELECT true, p_version + 1, v_desired, 'ok'::text;
    END IF;
END;
$$;
```

---

### RPC 3：container.workspace_observe

与 environment_observe 同模式，workspace 独立上报。

```sql
CREATE OR REPLACE FUNCTION container.workspace_observe(
    p_workspace_id TEXT,
    p_version      INTEGER,
    p_observed     TEXT,
    p_last_error   TEXT DEFAULT NULL
)
RETURNS TABLE(
    ok            BOOLEAN,
    new_version   INTEGER,
    desired_state TEXT,
    message       TEXT
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = container, pg_catalog
AS $$
DECLARE
    v_rows    INTEGER;
    v_desired TEXT;
BEGIN
    IF p_observed NOT IN ('running', 'stopped', 'creating', 'detached', 'error') THEN
        RETURN QUERY SELECT false, p_version, NULL::text, 'invalid_observed_state'::text;
        RETURN;
    END IF;

    UPDATE container.workspaces
    SET
        observed_state = p_observed,
        last_error     = CASE WHEN p_observed = 'error' THEN p_last_error ELSE NULL END,
        version        = version + 1,
        updated_at     = now()
    WHERE id      = p_workspace_id
      AND version = p_version
      AND status  = 'active'
    RETURNING desired_state INTO v_desired;

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    IF v_rows = 0 THEN
        RETURN QUERY SELECT false, p_version, NULL::text, 'conflict_or_inactive'::text;
    ELSE
        RETURN QUERY SELECT true, p_version + 1, v_desired, 'ok'::text;
    END IF;
END;
$$;
```

---

### RPC 4：container.device_disconnect

daemon 断开连接时原子级联：设备→offline，其下所有环境→detached。

```sql
CREATE OR REPLACE FUNCTION container.device_disconnect(
    p_device_id TEXT
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = container, pg_catalog
AS $$
BEGIN
    -- 设备标记 offline
    UPDATE container.devices
    SET
        status     = 'offline',
        updated_at = now()
    WHERE id = p_device_id
      AND status = 'online';

    -- 该设备下所有活跃环境标记 observed_state = 'detached'
    -- desired_state 不变（用户意图保留，daemon 重连后自动收敛）
    UPDATE container.environments
    SET
        observed_state = 'detached',
        last_seen_at   = now(),
        version        = version + 1,
        updated_at     = now()
    WHERE device_id = p_device_id
      AND status    = 'active'
      AND observed_state != 'detached';

    -- 工作区同步标记 detached（通过 environment_id 关联）
    UPDATE container.workspaces w
    SET
        observed_state = 'detached',
        version        = w.version + 1,
        updated_at     = now()
    FROM container.environments e
    WHERE w.environment_id = e.id
      AND e.device_id      = p_device_id
      AND w.status         = 'active'
      AND w.observed_state != 'detached';
END;
$$;
```

---

### RPC 5：container.get_pending_convergence

控制面轮询：获取所有 desired ≠ observed 的记录（需要协调的工作项）。

```sql
CREATE OR REPLACE FUNCTION container.get_pending_convergence(
    p_limit INTEGER DEFAULT 50
)
RETURNS TABLE(
    entity_type TEXT,   -- 'environment' | 'workspace'
    entity_id   TEXT,
    device_id   TEXT,
    desired     TEXT,
    observed    TEXT,
    version     INTEGER
)
LANGUAGE sql
SECURITY DEFINER
STABLE
SET search_path = container, pg_catalog
AS $$
    SELECT
        'environment'::text,
        e.id,
        e.device_id,
        e.desired_state,
        e.observed_state,
        e.version
    FROM container.environments e
    WHERE e.desired_state != e.observed_state
      AND e.status = 'active'
    ORDER BY e.updated_at ASC
    LIMIT p_limit / 2

    UNION ALL

    SELECT
        'workspace'::text,
        w.id,
        e.device_id,
        w.desired_state,
        w.observed_state,
        w.version
    FROM container.workspaces w
    JOIN container.environments e ON w.environment_id = e.id
    WHERE w.desired_state != w.observed_state
      AND w.status = 'active'
    ORDER BY w.updated_at ASC
    LIMIT p_limit / 2;
$$;
```

---

### RPC 权限汇总

```sql
-- RPCs 均 SECURITY DEFINER，前端不应直接调用
-- mycel-container 后端通过 service_role 调用，无需额外 GRANT
-- 若需要 authenticated 用户直接调（如 device_heartbeat 走前端），则：
GRANT EXECUTE ON FUNCTION container.device_heartbeat(TEXT, INTEGER, JSONB) TO authenticated;
-- 其他 RPCs 不开放给 authenticated，仅 service_role 可用
```

---

## 未解决 / 需后续确认

1. **daemon 认证方式**：daemon 是携带 device token 调 mycel-container API，还是持有 Supabase service_role key 直连？前者是正确架构，但需要 mycel-container 实现 device token 签发和验证机制。

2. **心跳超时自动 offline**：当前 heartbeat RPC 只负责"上线"，"超时 offline"需要定时任务（pg_cron 或 mycel-container 定时检查 `last_heartbeat_at < now() - interval '2 minutes'`）。建议 pg_cron 方案：
   ```sql
   -- 每分钟标记心跳超时设备为 offline
   SELECT cron.schedule(
       'device-timeout-sweep',
       '* * * * *',
       $$
       UPDATE container.devices
       SET status = 'offline', updated_at = now()
       WHERE status = 'online'
         AND last_heartbeat_at < now() - interval '2 minutes'
       $$
   );
   ```

3. **workspaces INSERT RLS 的 subquery 性能**：INSERT policy 中用了 `EXISTS (SELECT 1 FROM container.environments ...)`，高频创建时需要确认 environments(id, owner_user_id) 有覆盖索引（当前设计已有 `idx_environments_owner_active`，可覆盖）。
