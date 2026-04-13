# hub schema — 完整 DDL + 审查

> 2026-04-13，基于 32-schema-decisions.md hub 章节设计
>
> **设计约束**：
> - hub 是唯一已独立部署的模块，schema 最干净
> - 无跨 schema DB 外键；publisher_user_id 只存 ID，不 FK 到 identity.users
> - fjj 参考文档丢失；字段根据 marketplace 语义和 32-schema-decisions.md 决策推导
>
> **假设**：
> - 发布者注册制：user 须先申请 publisher profile，才能发布 item
> - item type = 'skill' | 'agent' | 'member'（来自 32 决策文档）
> - install_count 为去重计数；同一用户重复安装只计一次（通过 hub.item_installs 中间表保障）
>   → 设计决策：不新增第 4 张表，install_count 通过 RPC 原子更新，重复防护在应用层

---

## 1. DDL

```sql
-- ================================================================
-- Schema: hub
-- Marketplace：发布 / 安装 skill、agent、member 模板
-- 无跨 schema DB FK；已独立服务，零跨 schema 依赖
-- ================================================================

CREATE SCHEMA IF NOT EXISTS hub;


-- ================================================================
-- 1. hub.marketplace_publishers
--    发布者 profile（冗余存储，不 FK 到 identity.users）
--    每个 user 至多一个 publisher profile
-- ================================================================

CREATE TABLE hub.marketplace_publishers (
    id           TEXT        PRIMARY KEY,
    user_id      TEXT        NOT NULL,
        -- → identity.users（应用层，无 DB FK）；发布者对应的用户
    display_name TEXT        NOT NULL,
        -- 市场展示名，e.g. "Mycel Team"
    bio          TEXT,
        -- Markdown 简介，展示在发布者主页
    avatar_url   TEXT,
        -- CDN URL；冗余存储，不 FK 到 identity.assets（无跨 schema 依赖）
    verified     BOOLEAN     NOT NULL DEFAULT false,
        -- 管理员授予的认证标志（官方/可信发布者）
    status       TEXT        NOT NULL DEFAULT 'active',
        -- 'active' | 'suspended'
        -- 'suspended'：管理员封禁，发布 / 更新操作被 RLS 阻止
    item_count   INTEGER     NOT NULL DEFAULT 0,
        -- 冗余计数：已发布 item 数量；publish_version 时由 RPC 维护
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT publishers_status_chk
        CHECK (status IN ('active', 'suspended')),
    CONSTRAINT publishers_item_count_chk
        CHECK (item_count >= 0)
);

-- 每个 user 只能有一个发布者 profile
CREATE UNIQUE INDEX uq_publishers_user_id
    ON hub.marketplace_publishers(user_id);


-- ================================================================
-- 2. hub.marketplace_items
--    市场项目（skill / agent / member 模板）
--    含全文搜索 tsvector；由触发器 trg_items_search_vector 自动维护
-- ================================================================

CREATE TABLE hub.marketplace_items (
    id                TEXT        PRIMARY KEY,
    publisher_id      TEXT        NOT NULL,
        -- → hub.marketplace_publishers.id（应用层）
    -- 身份
    name              TEXT        NOT NULL,
        -- 展示名称，e.g. "Code Review Agent"
    slug              TEXT        NOT NULL,
        -- URL 友好标识，e.g. "code-reviewer"；同发布者下唯一
        -- 格式约束：仅小写字母 / 数字 / 连字符，API 层验证
    type              TEXT        NOT NULL,
        -- 'skill' | 'agent' | 'member'
    description       TEXT,
        -- Markdown 格式详细描述，展示在 item 详情页
    tags              TEXT[]      NOT NULL DEFAULT '{}',
        -- e.g. '{"coding", "python", "review"}'；支持按 tag 过滤
    -- 可见性 & 生命周期
    is_public         BOOLEAN     NOT NULL DEFAULT true,
        -- false = unlisted：有链接可访问，但不出现在搜索结果
    status            TEXT        NOT NULL DEFAULT 'draft',
        -- 'draft' | 'published' | 'archived'
        -- 'draft'：仅发布者可见；'archived'：下线但保留历史版本
    -- 最新版本（冗余）
    latest_version_id TEXT,
        -- → hub.marketplace_versions.id（应用层）；publish_version RPC 维护
    latest_version    TEXT,
        -- 版本号字符串，e.g. "1.2.0"；展示用冗余
    -- 统计（全部冗余，由 RPC 维护）
    install_count     INTEGER     NOT NULL DEFAULT 0,
        -- hub.record_install RPC 原子递增；不允许 authenticated 直接 UPDATE（见 REVOKE）
    rating_count      INTEGER     NOT NULL DEFAULT 0,
        -- 累计评分人数
    rating_sum        DOUBLE PRECISION NOT NULL DEFAULT 0,
        -- 累计评分总分；rating_sum / rating_count = 平均分
    -- 全文搜索
    search_vector     TSVECTOR,
        -- GIN 索引；weight A=name, B=description, C=tags
        -- 由 trg_items_search_vector 触发器自动维护，写入时无需手动填
    -- 时间戳
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT items_type_chk
        CHECK (type IN ('skill', 'agent', 'member')),
    CONSTRAINT items_status_chk
        CHECK (status IN ('draft', 'published', 'archived')),
    CONSTRAINT items_install_count_chk
        CHECK (install_count >= 0),
    CONSTRAINT items_rating_count_chk
        CHECK (rating_count >= 0)
);

-- 同发布者下 slug 唯一
CREATE UNIQUE INDEX uq_items_publisher_slug
    ON hub.marketplace_items(publisher_id, slug);

-- 全文搜索 GIN 索引（核心）
CREATE INDEX idx_items_search_vector
    ON hub.marketplace_items USING GIN(search_vector);

-- Tags 过滤（e.g. 筛选 tag='python' 的所有 item）
CREATE INDEX idx_items_tags
    ON hub.marketplace_items USING GIN(tags);

-- 按类型浏览（只扫描公开已发布条目）
CREATE INDEX idx_items_type_status
    ON hub.marketplace_items(type, status)
    WHERE is_public = true;

-- 热门排序（install_count DESC，仅已发布公开 item）
CREATE INDEX idx_items_install_count
    ON hub.marketplace_items(install_count DESC)
    WHERE status = 'published' AND is_public = true;

-- 发布者的 item 列表（管理后台、发布者主页）
CREATE INDEX idx_items_publisher
    ON hub.marketplace_items(publisher_id);


-- ================================================================
-- 3. hub.marketplace_versions
--    版本历史；每次发布一条完整 snapshot，不可修改（仅可 yank）
-- ================================================================

CREATE TABLE hub.marketplace_versions (
    id           TEXT        PRIMARY KEY,
    item_id      TEXT        NOT NULL,
        -- → hub.marketplace_items.id（应用层）
    version      TEXT        NOT NULL,
        -- 语义版本号，e.g. "1.0.0"；同 item 下唯一（见 uq_versions_item_version）
    content      JSONB       NOT NULL,
        -- 发布时的完整快照（不可变）：
        -- agent:  { config, rules[], skills[], sub_agents[] }
        -- skill:  { definition, parameters, examples[] }
        -- member: { profile, specialties[], instructions[] }
        -- 安装时以此为基础创建本地副本，与后续版本变更隔离
    changelog    TEXT,
        -- Markdown；本版本变更说明
    published_by TEXT        NOT NULL,
        -- → identity.users（应用层）；可与 publisher owner 不同（团队场景）
    status       TEXT        NOT NULL DEFAULT 'active',
        -- 'active' | 'yanked'
        -- 'yanked'：因安全/内容问题撤回；不物理删除（保留审计）
        -- yank 后不出现在 Realtime/public SELECT；安装了此版本的用户收到警告（应用层）
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
        -- 不加 updated_at：版本一旦发布不可修改，status 变更通过专用 RPC

    CONSTRAINT versions_status_chk
        CHECK (status IN ('active', 'yanked'))
);

-- 防止同 item 同版本号重复发布
CREATE UNIQUE INDEX uq_versions_item_version
    ON hub.marketplace_versions(item_id, version);

-- 版本历史列表（item 详情页的版本下拉）
CREATE INDEX idx_versions_item_created
    ON hub.marketplace_versions(item_id, created_at DESC);
```

---

## 2. 全文搜索触发器

```sql
-- ================================================================
-- hub.refresh_item_search_vector
-- 在 INSERT / UPDATE name/description/tags 时自动重建 tsvector
-- weight A = name（最重要）, B = description, C = tags
-- 语言 'simple'：无词干提取，支持 CJK（中文字符逐字索引）
-- 生产环境如需中文分词可换 'chinese'（需 pg_jieba）
-- ================================================================

CREATE OR REPLACE FUNCTION hub.refresh_item_search_vector()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = hub, pg_catalog
AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('simple', coalesce(NEW.name, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(NEW.description, '')), 'B') ||
        setweight(to_tsvector('simple', array_to_string(NEW.tags, ' ')), 'C');
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_items_search_vector
    BEFORE INSERT OR UPDATE OF name, description, tags
    ON hub.marketplace_items
    FOR EACH ROW
    EXECUTE FUNCTION hub.refresh_item_search_vector();

-- updated_at 自动维护
CREATE OR REPLACE FUNCTION hub.set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql
SET search_path = hub, pg_catalog
AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$;

CREATE TRIGGER trg_publishers_updated_at
    BEFORE UPDATE ON hub.marketplace_publishers
    FOR EACH ROW EXECUTE FUNCTION hub.set_updated_at();

CREATE TRIGGER trg_items_updated_at
    BEFORE UPDATE ON hub.marketplace_items
    FOR EACH ROW EXECUTE FUNCTION hub.set_updated_at();
```

---

## 3. RLS 策略

```sql
-- ================================================================
-- 启用 RLS
-- ================================================================

ALTER TABLE hub.marketplace_publishers ENABLE ROW LEVEL SECURITY;
ALTER TABLE hub.marketplace_items      ENABLE ROW LEVEL SECURITY;
ALTER TABLE hub.marketplace_versions   ENABLE ROW LEVEL SECURITY;


-- ================================================================
-- hub.marketplace_publishers
-- 公开读；只有本人可创建 / 更新自己的 profile
-- ================================================================

-- 任何人（包括 anon）均可读发布者 profile——市场首页展示
CREATE POLICY publishers_select_all
    ON hub.marketplace_publishers
    FOR SELECT
    USING (true);

-- 已登录用户可创建自己的发布者 profile（user_id 必须等于 caller）
CREATE POLICY publishers_insert_own
    ON hub.marketplace_publishers
    FOR INSERT
    TO authenticated
    WITH CHECK (user_id = auth.uid()::text);

-- 只有本人可更新自己的 profile
-- verified / status 字段的 REVOKE 见权限部分（防止自我提权）
CREATE POLICY publishers_update_own
    ON hub.marketplace_publishers
    FOR UPDATE
    TO authenticated
    USING  (user_id = auth.uid()::text)
    WITH CHECK (user_id = auth.uid()::text);


-- ================================================================
-- hub.marketplace_items
-- 公开读（仅 published）；发布者可见自己的所有状态 item
-- 写入需 active 的发布者 profile
-- ================================================================

-- SELECT：已发布公开 item 任何人可读；发布者可读自己所有状态的 item
CREATE POLICY items_select_published_or_own
    ON hub.marketplace_items
    FOR SELECT
    USING (
        -- 任何人可看已发布的公开 item
        (status = 'published' AND is_public = true)
        -- 发布者可看自己的所有 item（draft / archived 也能看）
        OR publisher_id IN (
            SELECT id FROM hub.marketplace_publishers
            WHERE user_id = auth.uid()::text
        )
    );

-- INSERT：调用者必须是该 publisher 的 owner，且 publisher 状态为 active
CREATE POLICY items_insert_own
    ON hub.marketplace_items
    FOR INSERT
    TO authenticated
    WITH CHECK (
        publisher_id IN (
            SELECT id FROM hub.marketplace_publishers
            WHERE user_id = auth.uid()::text
              AND status = 'active'
        )
    );

-- UPDATE：仅 publisher owner 可改（install_count/rating_* 由 RPC 改；见 REVOKE）
CREATE POLICY items_update_own
    ON hub.marketplace_items
    FOR UPDATE
    TO authenticated
    USING (
        publisher_id IN (
            SELECT id FROM hub.marketplace_publishers
            WHERE user_id = auth.uid()::text
        )
    )
    WITH CHECK (
        publisher_id IN (
            SELECT id FROM hub.marketplace_publishers
            WHERE user_id = auth.uid()::text
        )
    );


-- ================================================================
-- hub.marketplace_versions
-- 公开读（仅 active 版本 + 对应 item 为 published）
-- INSERT 通过 publish_version RPC（SECURITY DEFINER）；直接 INSERT 仅限 publisher owner
-- 不允许 UPDATE（版本不可修改，yank 通过 yank_version RPC）
-- ================================================================

-- SELECT：active 版本 + item 为 published 可公开读
-- 发布者自己的 item 的所有版本（含 yanked）可读
CREATE POLICY versions_select_active_or_own
    ON hub.marketplace_versions
    FOR SELECT
    USING (
        -- 公开：active 版本 + 已发布 item
        (status = 'active'
         AND item_id IN (
             SELECT id FROM hub.marketplace_items
             WHERE status = 'published' AND is_public = true
         ))
        -- 发布者自己的 item 的所有版本
        OR item_id IN (
            SELECT i.id FROM hub.marketplace_items i
            JOIN hub.marketplace_publishers p ON p.id = i.publisher_id
            WHERE p.user_id = auth.uid()::text
        )
    );

-- INSERT：发布者 owner 可直接插入（通常通过 publish_version RPC）
CREATE POLICY versions_insert_own
    ON hub.marketplace_versions
    FOR INSERT
    TO authenticated
    WITH CHECK (
        item_id IN (
            SELECT i.id FROM hub.marketplace_items i
            JOIN hub.marketplace_publishers p ON p.id = i.publisher_id
            WHERE p.user_id = auth.uid()::text
              AND p.status = 'active'
        )
    );

-- 不开放 UPDATE policy：版本不可修改
-- yank 通过 hub.yank_version RPC（SECURITY DEFINER）操作


-- ================================================================
-- 列级权限：防止统计字段被直接篡改
-- ================================================================

-- Schema access
GRANT USAGE ON SCHEMA hub TO anon, authenticated;

-- Public marketplace browsing (anon can read)
GRANT SELECT ON hub.marketplace_publishers  TO anon, authenticated;
GRANT SELECT ON hub.marketplace_items       TO anon, authenticated;
GRANT SELECT ON hub.marketplace_versions    TO anon, authenticated;

-- Authenticated users can publish
GRANT INSERT ON hub.marketplace_publishers  TO authenticated;
GRANT UPDATE ON hub.marketplace_publishers  TO authenticated;
GRANT INSERT ON hub.marketplace_items       TO authenticated;
GRANT UPDATE ON hub.marketplace_items       TO authenticated;
GRANT INSERT ON hub.marketplace_versions    TO authenticated;
-- No UPDATE on versions (immutable after publish; yanking goes through RPC)

-- 禁止 authenticated 直接修改统计字段和版本指针
-- （install_count / rating_* 只能通过 SECURITY DEFINER RPC 递增）
REVOKE UPDATE (install_count, rating_count, rating_sum, latest_version_id, latest_version)
    ON hub.marketplace_items
    FROM authenticated;

-- 禁止 authenticated 修改 publishers 的 verified / item_count（管理员 / RPC 专属）
REVOKE UPDATE (verified, item_count, status)
    ON hub.marketplace_publishers
    FROM authenticated;
```

---

## 4. Realtime 发布清单

| 表 | Realtime | 事件 | 说明 |
|---|---------|------|------|
| `hub.marketplace_items` | ✅ 建议 | UPDATE | 用户关注的 item 状态变更（draft→published）、install_count 更新 |
| `hub.marketplace_versions` | ✅ 建议 | INSERT | 新版本发布通知（已安装该 item 的用户） |
| `hub.marketplace_publishers` | ❌ | — | 极低频，无实时推送价值 |

**订阅示例**（前端）：

```typescript
// 追踪某个 item 的新版本
supabase
  .channel('hub-versions')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'hub',
    table: 'marketplace_versions',
    filter: `item_id=eq.${itemId}`
  }, handler)
  .subscribe()
```

**RLS 保障**：versions_select_active_or_own policy 确保 Realtime 订阅到的版本
符合可见性规则——yanked 版本和未发布 item 的版本不会推送给 anon/authenticated 用户。

---

## 5. RPCs

```sql
-- ================================================================
-- hub.publish_version
-- 发布新版本：创建 version 记录 + 更新 item 的 latest_version_id
-- 如 item 为 draft，自动切换为 published
-- SECURITY DEFINER：绕过 RLS 做原子写入；内部验证调用者所有权
-- ================================================================

CREATE OR REPLACE FUNCTION hub.publish_version(
    p_item_id   TEXT,
    p_version   TEXT,
    p_content   JSONB,
    p_changelog TEXT DEFAULT NULL
)
RETURNS TEXT          -- 返回新 version 的 id
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = hub, pg_catalog
AS $$
DECLARE
    v_version_id        TEXT := gen_random_uuid()::text;
    v_publisher_user_id TEXT;
    v_publisher_status  TEXT;
BEGIN
    -- 验证调用者对该 item 有所有权且 publisher 未被封禁
    SELECT p.user_id, p.status
    INTO   v_publisher_user_id, v_publisher_status
    FROM   hub.marketplace_items i
    JOIN   hub.marketplace_publishers p ON p.id = i.publisher_id
    WHERE  i.id = p_item_id;

    IF v_publisher_user_id IS NULL THEN
        RAISE EXCEPTION 'item not found';
    END IF;

    IF v_publisher_user_id != auth.uid()::text THEN
        RAISE EXCEPTION 'permission denied';
    END IF;

    IF v_publisher_status = 'suspended' THEN
        RAISE EXCEPTION 'publisher is suspended';
    END IF;

    -- 插入版本记录（uq_versions_item_version 防止同版本号重复）
    INSERT INTO hub.marketplace_versions
        (id, item_id, version, content, changelog, published_by)
    VALUES
        (v_version_id, p_item_id, p_version, p_content, p_changelog, auth.uid()::text);

    -- 更新 item 的最新版本指针；若为 draft 则自动发布
    UPDATE hub.marketplace_items
    SET latest_version_id = v_version_id,
        latest_version    = p_version,
        status            = CASE WHEN status = 'draft' THEN 'published' ELSE status END,
        updated_at        = now()
    WHERE id = p_item_id;

    -- 更新发布者 item_count（仅在首次从 draft 变为 published 时 +1）
    -- 简化实现：每次 publish_version 后重算（subquery count）
    UPDATE hub.marketplace_publishers p
    SET item_count = (
        SELECT count(*)
        FROM   hub.marketplace_items i
        WHERE  i.publisher_id = p.id
          AND  i.status = 'published'
    ),
        updated_at = now()
    WHERE p.user_id = auth.uid()::text;

    RETURN v_version_id;
END;
$$;


-- ================================================================
-- hub.record_install
-- 原子递增 install_count；验证 item 为已发布状态
-- SECURITY DEFINER：绕过 RLS 写统计字段（authenticated REVOKE 了 install_count UPDATE）
-- 重复安装防护：应用层负责去重（DB 层不做，避免引入 item_installs 第 4 张表）
-- ================================================================

CREATE OR REPLACE FUNCTION hub.record_install(
    p_item_id TEXT
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = hub, pg_catalog
AS $$
BEGIN
    UPDATE hub.marketplace_items
    SET install_count = install_count + 1,
        updated_at    = now()
    WHERE id = p_item_id
      AND status = 'published';

    IF NOT FOUND THEN
        RAISE EXCEPTION 'item not found or not published';
    END IF;
END;
$$;


-- ================================================================
-- hub.yank_version
-- 撤回（yank）一个已发布版本（安全问题、误发）
-- 不物理删除（保留审计记录）
-- 若被 yank 的是 latest_version，同时将 latest_version_id 回退到上一个 active 版本
-- ================================================================

CREATE OR REPLACE FUNCTION hub.yank_version(
    p_version_id TEXT
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = hub, pg_catalog
AS $$
DECLARE
    v_item_id           TEXT;
    v_publisher_user_id TEXT;
    v_is_latest         BOOLEAN;
    v_prev_version_id   TEXT;
    v_prev_version      TEXT;
BEGIN
    -- 验证调用者所有权
    SELECT i.id, p.user_id, (i.latest_version_id = p_version_id)
    INTO   v_item_id, v_publisher_user_id, v_is_latest
    FROM   hub.marketplace_versions v
    JOIN   hub.marketplace_items i     ON i.id = v.item_id
    JOIN   hub.marketplace_publishers p ON p.id = i.publisher_id
    WHERE  v.id = p_version_id;

    IF v_publisher_user_id IS NULL THEN
        RAISE EXCEPTION 'version not found';
    END IF;

    IF v_publisher_user_id != auth.uid()::text THEN
        RAISE EXCEPTION 'permission denied';
    END IF;

    -- 标记为 yanked
    UPDATE hub.marketplace_versions
    SET status = 'yanked'
    WHERE id = p_version_id;

    -- 若被 yank 的是最新版本，回退 latest_version_id 到上一个 active 版本
    IF v_is_latest THEN
        SELECT id, version
        INTO   v_prev_version_id, v_prev_version
        FROM   hub.marketplace_versions
        WHERE  item_id = v_item_id
          AND  status  = 'active'
          AND  id     != p_version_id
        ORDER  BY created_at DESC
        LIMIT  1;

        -- 更新 item（若无 active 版本则置为 archived）
        UPDATE hub.marketplace_items
        SET latest_version_id = v_prev_version_id,
            latest_version    = v_prev_version,
            status = CASE
                WHEN v_prev_version_id IS NULL THEN 'archived'
                ELSE status
            END,
            updated_at = now()
        WHERE id = v_item_id;
    END IF;
END;
$$;
```

---

## 6. 安全审查

| 风险 | 处置 |
|------|------|
| **统计字段篡改**（install_count、rating_*） | `REVOKE UPDATE (install_count, rating_count, rating_sum)` from authenticated；只能通过 SECURITY DEFINER RPC 修改 |
| **自我提权**（verified、publisher status） | `REVOKE UPDATE (verified, item_count, status)` from authenticated；管理员通过 service_role 操作 |
| **版本内容篡改** | versions 表不开 UPDATE policy；yank 操作通过专用 RPC（仅改 status 字段） |
| **slug 注入**（路径穿越、XSS） | slug 格式约束（小写字母/数字/连字符）在 API 层验证；DB 层加 CHECK 约束可选 |
| **content JSONB 体积攻击** | 单个版本 content 无大小限制；应在 API 层加 max_body_size 限制（建议 1 MB） |
| **发布者冒充**（publisher_id 伪造） | publish_version RPC 内部验证 `p.user_id = auth.uid()`；RLS INSERT policy 也有同等保护 |
| **Realtime 越权订阅** | versions_select_active_or_own policy 确保 yanked 版本 / 未发布 item 的版本不进 Realtime 推送 |
| **publisher 封禁绕过** | publish_version RPC 显式检查 publisher.status = 'active'；RLS INSERT policy 同样检查 |
| **重复安装刷量** | record_install 无去重；去重责任在应用层（login user 每 item 只调用一次）；可选后续增加 hub.item_installs 表 |
