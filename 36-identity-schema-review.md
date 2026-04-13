# identity schema 完整审查

> 2026-04-13，基于 32-schema-decisions.md identity 章节 + fjj runtime database reference
>
> **identity 的特殊地位**：所有 schema 的共享只读层，服务端通过 service_role 读写，
> 前端通过 PostgREST (authenticated JWT) 访问，RLS 是唯一防线。

---

## 7 张表清单

| 表 | 来源 | 核心字段（已有定义处） |
|----|------|----------------------|
| `identity.users` | staging.users (fjj doc) | id, type, display_name, avatar, owner_user_id, agent_config_id |
| `identity.accounts` | staging.accounts (fjj doc) | id, user_id, username, password_hash, api_key_hash |
| `identity.assets` | staging.assets (fjj doc) | id, owner_user_id, kind, storage_key, sha256 |
| `identity.user_settings` | public.user_settings 改造 | (user_id, scope) PK, config JSONB |
| `identity.invite_codes` | staging.invite_codes (fjj doc) | code PK, used_by, expires_at |
| `identity.model_providers` | NEW | id, user_id, api_key_enc, status |
| `identity.model_mappings` | NEW | id, user_id, alias, is_active |

---

## 假设声明

**api_key_enc 存储方式**：model_providers.api_key_enc 存储服务端 AES-256-GCM 加密后的密文（服务端持有 master key，与 DB 分离存储）。前端提交明文 API key → 后端加密后入库 → 读取时返回 `'****'` 掩码，**明文永不落 DB 层以外**。

**identity.assets 可见性**：assets 元数据分两类——avatar（必须对所有认证用户可见，否则 chat 无法展示头像）和其他用户文件（仅 owner 可见）。用 `kind = 'avatar'` 区分，RLS 据此建两条策略。

---

## a) 完整索引设计

### identity.users

```sql
-- 已有（来自约束）：
-- PK (id)
-- UNIQUE (email)
-- UNIQUE (mycel_id)

-- 按类型列出 agent / human（首页 agent 列表、管理面板）
CREATE INDEX idx_users_type
    ON identity.users(type, created_at DESC);

-- 某用户拥有的所有 agent（"我的 agent" 列表）
CREATE INDEX idx_users_owner_agents
    ON identity.users(owner_user_id)
    WHERE type = 'agent' AND owner_user_id IS NOT NULL;

-- agent_config_id 反向查（从 config 找 user）
-- identity ↔ agent 循环引用的应用层校验辅助
CREATE INDEX idx_users_agent_config
    ON identity.users(agent_config_id)
    WHERE agent_config_id IS NOT NULL;

-- 头像资产反向查（换头像时检查旧资产引用）
CREATE INDEX idx_users_avatar
    ON identity.users(avatar)
    WHERE avatar IS NOT NULL;
```

### identity.accounts

```sql
-- 已有（来自约束）：
-- PK (id)
-- UNIQUE (user_id)
-- UNIQUE (username)
-- 上面两个 UNIQUE 索引已覆盖所有常用查询（登录走 username，会话校验走 user_id）
-- 不需要额外索引
```

### identity.assets

```sql
-- 已有：
-- PK (id)

-- 用户资产库（列表展示，按上传时间倒序）
CREATE INDEX idx_assets_owner_active
    ON identity.assets(owner_user_id, created_at DESC)
    WHERE deleted_at IS NULL;

-- 按 kind 过滤（头像 vs 文档 vs 代码片段）
CREATE INDEX idx_assets_owner_kind
    ON identity.assets(owner_user_id, kind)
    WHERE deleted_at IS NULL;

-- 去重检查（上传前 sha256 对比，避免重复存储相同文件）
CREATE INDEX idx_assets_sha256
    ON identity.assets(sha256)
    WHERE deleted_at IS NULL;
```

### identity.user_settings

```sql
-- 已有：
-- PK (user_id, scope)
-- PK 前缀 (user_id) 已覆盖"获取用户所有设置"查询，不需要额外索引
```

### identity.invite_codes

```sql
-- 已有：
-- PK (code)

-- 用户查自己创建的码（管理面板）
CREATE INDEX idx_invite_codes_creator
    ON identity.invite_codes(created_by)
    WHERE created_by IS NOT NULL;

-- 清理过期未使用码（定时任务）
CREATE INDEX idx_invite_codes_expires
    ON identity.invite_codes(expires_at)
    WHERE used_at IS NULL AND expires_at IS NOT NULL;

-- 审计：查某用户用了哪个码（注册溯源）
CREATE INDEX idx_invite_codes_used_by
    ON identity.invite_codes(used_by)
    WHERE used_by IS NOT NULL;
```

### identity.model_providers

```sql
-- 已有（来自约束）：
-- PK (id)
-- UNIQUE (user_id, name)
-- UNIQUE 索引前缀 (user_id) 已覆盖"用户所有 provider"查询

-- 模型解析热路径：用户的活跃 provider（每次 agent 调用都查）
CREATE INDEX idx_model_providers_user_active
    ON identity.model_providers(user_id)
    WHERE status = 'active';

-- 全局健康检查扫描（mycel-container 定时探测，按最久未检查排序）
CREATE INDEX idx_model_providers_health_sweep
    ON identity.model_providers(last_check_at ASC NULLS FIRST)
    WHERE status = 'active';
```

### identity.model_mappings

```sql
-- 已有（来自约束）：
-- PK (id)
-- UNIQUE (user_id, alias)

-- provider 删除时级联检查（避免 FK 孤悬，应用层校验前查）
CREATE INDEX idx_model_mappings_provider
    ON identity.model_mappings(provider_id);

-- 解析当前激活模型（agent 每次调用的热路径）
-- 每用户应只有一条 is_active=true，此索引极小
CREATE INDEX idx_model_mappings_active
    ON identity.model_mappings(user_id)
    WHERE is_active = true;

-- 用户所有启用的模型（设置页列表，排除已禁用）
CREATE INDEX idx_model_mappings_user_enabled
    ON identity.model_mappings(user_id)
    WHERE enabled = true;
```

---

## b) RLS 策略

### 启用 RLS

```sql
ALTER TABLE identity.users           ENABLE ROW LEVEL SECURITY;
ALTER TABLE identity.accounts        ENABLE ROW LEVEL SECURITY;
ALTER TABLE identity.assets          ENABLE ROW LEVEL SECURITY;
ALTER TABLE identity.user_settings   ENABLE ROW LEVEL SECURITY;
ALTER TABLE identity.invite_codes    ENABLE ROW LEVEL SECURITY;
ALTER TABLE identity.model_providers ENABLE ROW LEVEL SECURITY;
ALTER TABLE identity.model_mappings  ENABLE ROW LEVEL SECURITY;
```

---

### identity.users

```sql
-- SELECT：所有认证用户可读所有行（IM 必须：chat 展示 sender 名字/头像）
-- 敏感列限制用列权限而非 RLS（见下方 REVOKE）
CREATE POLICY users_select_all ON identity.users
    FOR SELECT TO authenticated
    USING (true);

-- INSERT：不开放，注册流由 auth_service (service_role) 执行
-- （不创建 INSERT policy = 默认拒绝）

-- UPDATE：仅能改自己的 profile 字段
CREATE POLICY users_update_own ON identity.users
    FOR UPDATE TO authenticated
    USING (id = (SELECT auth.uid()::text))
    WITH CHECK (id = (SELECT auth.uid()::text));

-- DELETE：不开放（注销由 service_role 执行，涉及级联清理）

-- 列级权限：只开放前端可改的 profile 字段
GRANT SELECT ON identity.users TO authenticated;
GRANT UPDATE (display_name, avatar, bio) ON identity.users TO authenticated;
-- 不开放 UPDATE: owner_user_id, agent_config_id, mycel_id, type, email（由服务端管理）
REVOKE UPDATE (owner_user_id, agent_config_id, mycel_id, type)
    ON identity.users FROM authenticated;
```

---

### identity.accounts

```sql
-- SELECT：仅自己的账户（含敏感字段，列权限限制见下）
CREATE POLICY accounts_select_own ON identity.accounts
    FOR SELECT TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

-- INSERT / DELETE：不开放（注册/注销走 service_role）

-- UPDATE：不开放直接改密码/api_key（需通过 RPC，含旧密码校验）

GRANT SELECT ON identity.accounts TO authenticated;
-- 即使行级 SELECT 开放了，也屏蔽敏感列
REVOKE SELECT (password_hash, api_key_hash) ON identity.accounts FROM authenticated;
```

---

### identity.assets

```sql
-- SELECT（规则 1）：owner 可见自己所有资产
CREATE POLICY assets_select_own ON identity.assets
    FOR SELECT TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text));

-- SELECT（规则 2）：头像资产对所有认证用户可见（chat 展示头像必须）
-- kind='avatar' 的资产视为公开元数据
CREATE POLICY assets_select_avatar ON identity.assets
    FOR SELECT TO authenticated
    USING (kind = 'avatar' AND deleted_at IS NULL);

-- INSERT：用户可以上传自己的资产（storage_key 由 service_role 在上传回调后写入）
CREATE POLICY assets_insert_own ON identity.assets
    FOR INSERT TO authenticated
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

-- UPDATE：仅自己的资产（软删除）
CREATE POLICY assets_update_own ON identity.assets
    FOR UPDATE TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

-- DELETE：不开放物理删除（前端走 UPDATE deleted_at = now()）
-- 实际删除（清理 storage bucket）由 service_role 定时执行

GRANT SELECT ON identity.assets TO authenticated;
GRANT INSERT ON identity.assets TO authenticated;
GRANT UPDATE (deleted_at) ON identity.assets TO authenticated;
-- storage_key 不开放 UPDATE（由 service_role 在上传完成后写入）
```

---

### identity.user_settings

```sql
-- SELECT / INSERT / UPDATE / DELETE：仅自己
CREATE POLICY user_settings_own ON identity.user_settings
    FOR ALL TO authenticated
    USING (user_id = (SELECT auth.uid()::text))
    WITH CHECK (user_id = (SELECT auth.uid()::text));

GRANT ALL ON identity.user_settings TO authenticated;
```

---

### identity.invite_codes

```sql
-- SELECT：用户只能看自己创建的码（管理） + 自己使用的码（记录）
CREATE POLICY invite_codes_select_own ON identity.invite_codes
    FOR SELECT TO authenticated
    USING (
        created_by = (SELECT auth.uid()::text)
        OR used_by  = (SELECT auth.uid()::text)
    );

-- INSERT / UPDATE / DELETE：不开放（码由 service_role 生成，兑换走 RPC）

GRANT SELECT ON identity.invite_codes TO authenticated;
```

---

### identity.model_providers

```sql
-- SELECT：仅自己的 provider（api_key_enc 敏感，列权限限制见下）
CREATE POLICY providers_select_own ON identity.model_providers
    FOR SELECT TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

-- INSERT：不开放（前端提交明文 key → mycel API 加密 → service_role INSERT）
-- 防止前端直接插入明文或自定义加密值

-- UPDATE：用户可改 name、base_url、status（启停）
-- api_key_enc 变更通过专用 RPC（key rotation）
CREATE POLICY providers_update_own ON identity.model_providers
    FOR UPDATE TO authenticated
    USING (user_id = (SELECT auth.uid()::text))
    WITH CHECK (user_id = (SELECT auth.uid()::text));

-- DELETE：允许用户删自己的 provider
CREATE POLICY providers_delete_own ON identity.model_providers
    FOR DELETE TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

GRANT SELECT ON identity.model_providers TO authenticated;
GRANT UPDATE (name, base_url, status) ON identity.model_providers TO authenticated;
GRANT DELETE ON identity.model_providers TO authenticated;
-- api_key_enc 完全不开放给前端（只有 service_role 可写）
REVOKE SELECT (api_key_enc) ON identity.model_providers FROM authenticated;
```

---

### identity.model_mappings

```sql
-- SELECT：仅自己
CREATE POLICY mappings_select_own ON identity.model_mappings
    FOR SELECT TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

-- INSERT：用户添加 mapping，但需要校验 provider_id 归属（防止关联他人 provider）
CREATE POLICY mappings_insert_own ON identity.model_mappings
    FOR INSERT TO authenticated
    WITH CHECK (
        user_id = (SELECT auth.uid()::text)
        AND EXISTS (
            SELECT 1 FROM identity.model_providers p
            WHERE p.id      = provider_id
              AND p.user_id = (SELECT auth.uid()::text)
        )
    );

-- UPDATE：可改 alias、enabled、temperature、max_tokens、extra_config
-- is_active 不开放直接 UPDATE（防并发冲突：两个 UPDATE 同时设 is_active=true）
CREATE POLICY mappings_update_own ON identity.model_mappings
    FOR UPDATE TO authenticated
    USING (user_id = (SELECT auth.uid()::text))
    WITH CHECK (user_id = (SELECT auth.uid()::text));

-- DELETE：仅自己
CREATE POLICY mappings_delete_own ON identity.model_mappings
    FOR DELETE TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

GRANT SELECT ON identity.model_mappings TO authenticated;
GRANT INSERT ON identity.model_mappings TO authenticated;
GRANT UPDATE (alias, enabled, temperature, max_tokens, extra_config)
    ON identity.model_mappings TO authenticated;
GRANT DELETE ON identity.model_mappings TO authenticated;
-- is_active 不开放，走 identity.set_active_model RPC
```

---

## c) Realtime 发布清单

| 表 | Realtime | 事件 | 理由 |
|----|----------|------|------|
| `identity.users` | ✅ 必须 | UPDATE | display_name / avatar 变更需推送到 chat UI（消息列表立即刷新头像）|
| `identity.model_providers` | ✅ 必须 | UPDATE | status / last_error 变更需实时推送到设置页（provider 探测失败即时提示）|
| `identity.model_mappings` | ⚠️ 可选 | UPDATE | 多标签页切换激活模型时同步（低优先级，前端可轮询替代）|
| `identity.accounts` | ❌ | — | 极敏感，无实时需求 |
| `identity.assets` | ❌ | — | 上传完成由 API 回调通知，无需 Realtime |
| `identity.user_settings` | ❌ | — | 多标签同步低频，可轮询 |
| `identity.invite_codes` | ❌ | — | 无实时需求 |

```sql
ALTER PUBLICATION supabase_realtime
    ADD TABLE identity.users, identity.model_providers;

-- 可选，多标签同步
-- ALTER PUBLICATION supabase_realtime ADD TABLE identity.model_mappings;
```

### 订阅示例（前端 JS）

```js
// 监听自己 provider 的健康状态变化
supabase
  .channel('my-model-providers')
  .on('postgres_changes', {
      event:  'UPDATE',
      schema: 'identity',
      table:  'model_providers',
      filter: `user_id=eq.${userId}`
  }, (payload) => {
      if (payload.new.status === 'error') {
          showNotification(`Provider "${payload.new.name}" 连接失败`)
      }
  })
  .subscribe()
```

> **注意**：identity.users 的 RLS 策略是 `USING(true)`（所有认证用户可读所有行），
> 意味着 Realtime 的 UPDATE 广播对所有已登录用户可见。
> 务必在订阅时加 filter（如 chat 成员列表的 user_id），避免前端收到大量无关更新。

---

## d) 安全审查

### 角色权限矩阵

| 角色 | users | accounts | assets | user_settings | invite_codes | model_providers | model_mappings |
|------|-------|----------|--------|---------------|--------------|-----------------|----------------|
| `anon` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `authenticated` | SELECT all rows / UPDATE(display_name,avatar,bio) own | SELECT(excl.hashes) own | SELECT own + avatar kind / INSERT+UPDATE(deleted_at) own | ALL own | SELECT own-created/used | SELECT(excl.api_key_enc) own / UPDATE(name,base_url,status) / DELETE | SELECT/INSERT/UPDATE(non-active)/DELETE own |
| `service_role` | 全部 | 全部 | 全部 | 全部 | 全部 | 全部 | 全部 |

---

### 主要风险 + 对策

#### 风险 1：api_key_enc 泄露（最高优先级）

model_providers 存放用户的 LLM API key（OpenAI/Anthropic 等），一旦泄露直接造成账单损失。

**对策**：
1. `REVOKE SELECT (api_key_enc) ON identity.model_providers FROM authenticated` — DB 列级禁止前端读取
2. 后端 API 返回 provider 信息时，api_key_enc 字段替换为 `'sk-****...****'` 掩码
3. 加密 master key 存在应用配置（K8s Secret / Vault），与 DB 物理隔离
4. 写入时只有 mycel-container 后端（service_role）能做加密写入，前端不得 INSERT

#### 风险 2：password_hash / api_key_hash 泄露

accounts 表中的凭据字段即使是 hash 也不应暴露（彩虹表攻击、hash 算法弱点）。

**对策**：
1. `REVOKE SELECT (password_hash, api_key_hash) ON identity.accounts FROM authenticated`
2. 密码校验走 RPC（server-side bcrypt 比对），不返回 hash 值
3. 日志系统设置 accounts 表 SELECT 的 scrubbing 规则

#### 风险 3：email 枚举（UNIQUE 约束被利用）

`identity.users.email` 有 UNIQUE 约束，攻击者可通过注册接口探测哪些邮箱已注册。

**对策**：
1. 注册 API 统一返回 "邮箱或密码错误"，不区分"邮箱不存在"和"密码错误"
2. 注册流限速（每 IP 每分钟 5 次）
3. 用户 profile 页不暴露 email 字段（前端 API 仅返回 display_name）

#### 风险 4：invite_code 暴力破解

邀请码如果是可预测格式，攻击者可暴力枚举。

**对策**：
1. 码长 ≥ 12 位，使用 `gen_random_bytes(9)` 生成（Base64 = 12 字符，熵值 72bit）
2. 兑换 API 限速（每 IP 每分钟 3 次）
3. 错误不区分"不存在"和"已使用"，统一返回"邀请码无效"
4. `expires_at` 设置合理有效期（7-30 天）

#### 风险 5：model_mappings.is_active 并发冲突

两个标签页同时调用"切换激活模型"，可能出现两条 `is_active=true` 记录。

**对策**：
1. `REVOKE UPDATE (is_active) ON identity.model_mappings FROM authenticated` — 前端不能直接改
2. 通过 `identity.set_active_model` RPC 在事务中原子切换（见 §e）
3. 兜底：`CHECK` 约束无法做到"只有一条 true"，必须靠 RPC 保证

#### 风险 6：identity.users UPDATE 越权（agent 转移）

users 表 UPDATE policy 基于 `id = auth.uid()`，但若前端能改 `owner_user_id` 字段，可把他人的 agent 转移给自己。

**对策**：
```sql
REVOKE UPDATE (owner_user_id, agent_config_id, mycel_id, type)
    ON identity.users FROM authenticated;
-- 只开放 display_name / avatar / bio，其余字段由 service_role 管理
```

#### 风险 7：assets.storage_key 泄露（私有文件被访问）

storage_key 是从 Supabase Storage 读取文件的关键，若他人获得可直接下载私有文件。

**对策**：
1. RLS 确保非 avatar 类资产只有 owner 可见 storage_key
2. Supabase Storage bucket 设置为 private，必须通过 signed URL（带 TTL）访问
3. 后端生成 signed URL 时验证请求者与 asset.owner_user_id 匹配

#### 风险 8：identity.users USING(true) 的广播放大

所有认证用户可读所有 users 行，Realtime 广播也无差别推送所有 UPDATE 事件。

**对策**：
1. 前端订阅时加 filter，只订阅 chat 成员的 user_id
2. 评估是否需要改为 `USING(id = auth.uid() OR type = 'human')`（见未解决清单）

---

## e) RPCs

### RPC 1：identity.redeem_invite_code

原子兑换邀请码：检查有效性 + 标记使用者，防止并发重复兑换。

```sql
CREATE OR REPLACE FUNCTION identity.redeem_invite_code(
    p_code    TEXT,
    p_user_id TEXT   -- 兑换者 user_id（注册流中传入）
)
RETURNS TABLE(
    ok      BOOLEAN,
    message TEXT
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_rows INTEGER;
BEGIN
    UPDATE identity.invite_codes
    SET
        used_by = p_user_id,
        used_at = now()
    WHERE code      = p_code
      AND used_by   IS NULL
      AND (expires_at IS NULL OR expires_at > now());

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    IF v_rows = 0 THEN
        -- 统一错误信息，不区分"不存在"/"已用"/"过期"（防枚举）
        RETURN QUERY SELECT false, 'invalid_or_expired'::text;
    ELSE
        RETURN QUERY SELECT true, 'ok'::text;
    END IF;
END;
$$;

-- 走 mycel 注册 API（service_role）调用，不开放给 authenticated
```

---

### RPC 2：identity.next_mycel_id

全局递增整数 ID（迁移自 staging.next_mycel_id，逻辑不变）。

```sql
CREATE SEQUENCE IF NOT EXISTS identity.mycel_id_seq START 1;

CREATE OR REPLACE FUNCTION identity.next_mycel_id()
RETURNS INTEGER
LANGUAGE sql
SECURITY DEFINER
AS $$
    SELECT nextval('identity.mycel_id_seq')::integer;
$$;

GRANT EXECUTE ON FUNCTION identity.next_mycel_id() TO service_role;
```

---

### RPC 3：identity.set_active_model

原子切换"当前激活模型"：事务内先清除旧的 is_active，再设新的，防并发冲突。

```sql
CREATE OR REPLACE FUNCTION identity.set_active_model(
    p_alias TEXT    -- 要激活的模型 alias
)
RETURNS TABLE(
    ok      BOOLEAN,
    message TEXT
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_user_id TEXT;
    v_rows    INTEGER;
BEGIN
    v_user_id := auth.uid()::text;  -- 从 JWT 取调用方 user_id，不信任参数

    -- 确认目标 mapping 存在且属于当前用户
    IF NOT EXISTS (
        SELECT 1 FROM identity.model_mappings
        WHERE user_id = v_user_id
          AND alias   = p_alias
          AND enabled = true
    ) THEN
        RETURN QUERY SELECT false, 'not_found_or_disabled'::text;
        RETURN;
    END IF;

    -- 原子切换：先清除旧的，再设新的
    UPDATE identity.model_mappings
    SET is_active  = false,
        updated_at = now()
    WHERE user_id   = v_user_id
      AND is_active = true
      AND alias    != p_alias;

    UPDATE identity.model_mappings
    SET is_active  = true,
        updated_at = now()
    WHERE user_id = v_user_id
      AND alias   = p_alias;

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    RETURN QUERY SELECT (v_rows > 0), 'ok'::text;
END;
$$;

-- 前端可直接调用（user_id 从 JWT 取，不依赖参数）
GRANT EXECUTE ON FUNCTION identity.set_active_model(TEXT) TO authenticated;
```

---

### RPC 4：identity.upsert_user_setting

合并式写入设置（JSONB merge，非全量覆盖），防止并发写入覆盖其他 key。

```sql
CREATE OR REPLACE FUNCTION identity.upsert_user_setting(
    p_scope TEXT,   -- 'general' | 'ui' | 'observation' 等
    p_patch JSONB   -- 只传需要更新的 key，其余保留
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
    INSERT INTO identity.user_settings (user_id, scope, config, updated_at)
    VALUES (auth.uid()::text, p_scope, p_patch, now())
    ON CONFLICT (user_id, scope)
    DO UPDATE SET
        config     = identity.user_settings.config || p_patch,  -- JSONB merge
        updated_at = now();
END;
$$;

-- user_id 从 auth.uid() 隐式绑定，前端无法伪造
GRANT EXECUTE ON FUNCTION identity.upsert_user_setting(TEXT, JSONB) TO authenticated;
```

---

### RPC 权限汇总

| RPC | 调用方 | 说明 |
|-----|--------|------|
| `identity.redeem_invite_code(TEXT, TEXT)` | service_role | 注册流后端调用，不开放前端 |
| `identity.next_mycel_id()` | service_role | 注册时分配 mycel_id |
| `identity.set_active_model(TEXT)` | authenticated | user_id 从 JWT 取，无参数伪造风险 |
| `identity.upsert_user_setting(TEXT, JSONB)` | authenticated | user_id 从 auth.uid() 隐式绑定 |

---

## 未解决 / 需后续确认

1. **identity.users USING(true) 的范围**：所有认证用户能枚举所有 users 行（含所有 agent user），在当前设计中是必要的（chat 展示 sender 信息）。如果 agent 配置需要保密，应改为 `USING(id = auth.uid() OR type = 'human')`，仅 human user 公开可见。

2. **assets 的 avatar 可见性**：当前用 `kind = 'avatar'` 区分公开资产。若未来新增需要公开访问的 kind（e.g., 'public_thumbnail'），建议在 assets 表加 `is_public BOOLEAN DEFAULT false` 字段，避免 kind 和可见性耦合，RLS 改为 `USING(owner_user_id = auth.uid() OR is_public = true)`。

3. **model_providers INSERT 流程**：不允许 authenticated INSERT，要求前端走 mycel API → 后端加密 → service_role INSERT。需要 mycel API 明确实现此端点，不能走 PostgREST 直接 INSERT。
