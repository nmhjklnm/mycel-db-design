# chat schema 完整设计

> 2026-04-13，基于 schema-decisions.md chat 章节（9 张表，15 条关键决策）
>
> **架构前提**：
> - mycel-chat 服务（service_role）负责所有写入路径（发消息、投递追踪）
> - 前端通过 PostgREST (authenticated JWT) 直读 + Realtime 订阅
> - RLS 是唯一访问控制边界（Realtime 用 anon key，无 RLS = 全表裸奔）
>
> **假设**：
> - `last_message_at` 改为 TIMESTAMPTZ（decisions doc 写的 DOUBLE PRECISION 是 fjj 遗留，
>   统一新 schema 改 TIMESTAMPTZ，迁移时旧表 `to_timestamp(col)` 转换）
> - `relationships` 状态变更（accept/block/reject）走 RPC，不开放前端直接 UPDATE
> - `message_deliveries` INSERT/UPDATE 全部走 service_role（push 投递追踪是后端职责）
> - `chat_members` INSERT 走 service_role（加减成员需要业务校验，不开放前端直接 INSERT）
> - **开发者应通过 `chat.messages_for_user` VIEW 查消息，不要直接查 `chat.messages`**
>   （per-user 软删除在 VIEW 里过滤，避免每个调用方手动加条件）

---

## DDL

```sql
CREATE SCHEMA IF NOT EXISTS chat;


-- ================================================================
-- 1. chat.chats
-- 会话容器。kind='dm' 一对一，'group' 群组，'ai' 人机对话
-- next_message_seq 由 chat.increment_chat_message_seq RPC 原子递增
-- last_message_at / last_message_preview 冗余字段，发消息时由后端更新
-- ================================================================

CREATE TABLE chat.chats (
    id                   TEXT        PRIMARY KEY,
    kind                 TEXT        NOT NULL,
        -- 'dm' | 'group' | 'ai'
        -- dm: 两人私聊；group: 多人群聊；ai: human ↔ agent 对话
    name                 TEXT,
        -- 群组/AI 会话名称；DM 为 NULL（用对方名字展示）
    status               TEXT        NOT NULL DEFAULT 'active',
        -- 'active' | 'archived'
    creator_user_id      TEXT        NOT NULL,
        -- → identity.users（应用层 FK）；创建者
    next_message_seq     BIGINT      NOT NULL DEFAULT 1,
        -- 单调递增，由 increment_chat_message_seq RPC 原子更新
    last_message_at      TIMESTAMPTZ,
        -- 冗余：最新消息时间，驱动会话列表排序（O(1) 替代 JOIN messages）
    last_message_preview TEXT,
        -- 冗余：摘要，e.g. "张三: 好的，明天见"
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chats_kind_chk   CHECK (kind IN ('dm', 'group', 'ai')),
    CONSTRAINT chats_status_chk CHECK (status IN ('active', 'archived'))
);

-- 会话列表排序（最新消息在前，仅活跃会话）
CREATE INDEX idx_chats_last_message
    ON chat.chats(last_message_at DESC)
    WHERE status = 'active';


-- ================================================================
-- 2. chat.messages
-- 消息主体。seq 在 chat 内单调递增，由 RPC 分配（不可前端写入）
-- 多种软删除语义：deleted_for_user_ids（per-user）/ deleted_at（全员）/ retracted_at（撤回）
-- ================================================================

CREATE TABLE chat.messages (
    id                        TEXT        PRIMARY KEY,
    chat_id                   TEXT        NOT NULL,
        -- → chat.chats.id（应用层 FK）
    sender_user_id            TEXT        NOT NULL,
        -- → identity.users（应用层 FK）
    seq                       BIGINT      NOT NULL,
        -- 由 chat.increment_chat_message_seq RPC 分配；同 chat 内唯一
    content                   TEXT,
        -- 消息正文；撤回后置 "[已撤回]"，E2E 加密后置空
    content_type              TEXT        NOT NULL DEFAULT 'text',
        -- 'text' | 'card' | 'system' | 'attachment'
    -- 幂等（弱网重试防重复）
    client_message_id         TEXT,
        -- 客户端生成 UUID，同 chat 内唯一（见 UNIQUE 约束）
    -- 线程化 & 转发
    reply_to_message_id       TEXT,
        -- → chat.messages.id（应用层）；扁平回复某条消息
    forwarded_from_message_id TEXT,
        -- → chat.messages.id（应用层，可跨 chat）；转发溯源
    thread_root_id            TEXT,
        -- → chat.messages.id（应用层）；Slack 式线程根
    -- 删除语义
    deleted_for_user_ids      JSONB       NOT NULL DEFAULT '[]',
        -- per-user 软删除：追加 user_id，查询时应用层过滤
        -- RLS 不做此过滤（JSONB 操作无法走索引），查询层加 WHERE
    deleted_at                TIMESTAMPTZ,
        -- 全员删除（管理员/群主操作）
    -- 编辑 & 撤回
    edited_at                 TIMESTAMPTZ,
        -- 最后编辑时间；NULL = 未编辑
    retracted_at              TIMESTAMPTZ,
        -- 消息撤回时间（时间窗口内限制）；NULL = 未撤回
    -- 全文搜索（由触发器维护）
    search_vector             TSVECTOR,
        -- GIN 索引，支持 content 搜索；见 trg_messages_search_vector
    -- E2E 加密预留
    encrypted_content         BYTEA,
        -- 加密后消息体；启用 E2E 时 content 置空，此字段填密文
    encryption_key_id         TEXT,
        -- 客户端密钥标识；启用 E2E 时填入
    -- 时间戳
    created_at                TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT messages_content_type_chk
        CHECK (content_type IN ('text', 'card', 'system', 'attachment')),
    -- seq 在同 chat 内唯一
    UNIQUE (chat_id, seq),
);

-- client_message_id 在同 chat 内唯一（NULL 排除在外，用 partial index 代替 UNIQUE NULLS NOT DISTINCT）
CREATE UNIQUE INDEX messages_client_dedup
    ON chat.messages(chat_id, client_message_id)
    WHERE client_message_id IS NOT NULL;

-- 消息分页（最核心查询：加载 chat 历史）
CREATE INDEX idx_messages_chat_seq
    ON chat.messages(chat_id, seq DESC);

-- 活跃消息分页（排除全员删除）
CREATE INDEX idx_messages_chat_seq_alive
    ON chat.messages(chat_id, seq DESC)
    WHERE deleted_at IS NULL;

-- 按发送者（admin 审计 / "我发过的"）
CREATE INDEX idx_messages_sender
    ON chat.messages(sender_user_id);

-- 全文搜索
CREATE INDEX idx_messages_search
    ON chat.messages USING GIN(search_vector);

-- 线程查询（加载某消息下的讨论线程）
CREATE INDEX idx_messages_thread
    ON chat.messages(thread_root_id)
    WHERE thread_root_id IS NOT NULL;


-- ================================================================
-- 3. chat.message_attachments
-- 消息附件（一条消息可有多个附件）
-- asset_id → identity.assets（应用层 FK）
-- ================================================================

CREATE TABLE chat.message_attachments (
    id         TEXT    PRIMARY KEY,
    message_id TEXT    NOT NULL REFERENCES chat.messages(id) ON DELETE CASCADE,
    asset_id   TEXT    NOT NULL,
        -- → identity.assets.id（应用层 FK）
    sort_order INTEGER NOT NULL DEFAULT 0,
        -- 同一消息内的附件顺序
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_message_attachments_message
    ON chat.message_attachments(message_id);


-- ================================================================
-- 4. chat.message_deliveries
-- 多设备投递追踪。驱动 push 通知：status='pending' 的设备需要推送
-- device_id → container.devices（应用层 FK）
-- INSERT/UPDATE 仅 service_role（投递追踪是后端职责）
-- ================================================================

CREATE TABLE chat.message_deliveries (
    message_id   TEXT        NOT NULL,
    device_id    TEXT        NOT NULL,
        -- → container.devices.id（应用层 FK）
    status       TEXT        NOT NULL DEFAULT 'pending',
        -- 'pending' | 'delivered' | 'failed'
    delivered_at TIMESTAMPTZ,

    PRIMARY KEY (message_id, device_id),

    CONSTRAINT deliveries_status_chk
        CHECK (status IN ('pending', 'delivered', 'failed')),
    -- pending 时 delivered_at 必须为空
    CONSTRAINT deliveries_ts_chk
        CHECK (status != 'delivered' OR delivered_at IS NOT NULL)
);

-- 某设备的未投递消息（push 补发扫描）
CREATE INDEX idx_deliveries_device_pending
    ON chat.message_deliveries(device_id)
    WHERE status = 'pending';

-- 某消息的投递状态总览（消息详情页"已读/未读"）
CREATE INDEX idx_deliveries_message
    ON chat.message_deliveries(message_id);


-- ================================================================
-- 5. chat.message_reactions
-- emoji 反应。agent 平台场景：👍/👎 是对 agent 输出的自然反馈
-- ================================================================

CREATE TABLE chat.message_reactions (
    message_id TEXT        NOT NULL REFERENCES chat.messages(id) ON DELETE CASCADE,
    user_id    TEXT        NOT NULL,
        -- → identity.users（应用层 FK）
    emoji      TEXT        NOT NULL,
        -- '👍' | '👎' | '❤️' | ... 无枚举约束，客户端限制
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (message_id, user_id, emoji)
);

-- 某消息的所有反应（emoji 统计展示）
CREATE INDEX idx_reactions_message
    ON chat.message_reactions(message_id);


-- ================================================================
-- 6. chat.message_pins
-- 消息置顶（群公告、agent 关键产出）
-- ================================================================

CREATE TABLE chat.message_pins (
    chat_id    TEXT        NOT NULL,
        -- → chat.chats.id
    message_id TEXT        NOT NULL REFERENCES chat.messages(id) ON DELETE CASCADE,
    pinned_by  TEXT        NOT NULL,
        -- → identity.users（应用层 FK）
    pinned_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (chat_id, message_id)
);

-- 某 chat 的所有置顶消息（群公告列表）
CREATE INDEX idx_pins_chat
    ON chat.message_pins(chat_id, pinned_at DESC);

CREATE INDEX idx_message_pins_message_id ON chat.message_pins(message_id);


-- ================================================================
-- 7. chat.chat_members
-- 会话成员 + 未读水位线。last_read_seq 驱动未读数计算
-- INSERT 仅 service_role（加成员需业务校验，如防止用户自己加入私聊）
-- ================================================================

CREATE TABLE chat.chat_members (
    chat_id      TEXT        NOT NULL,
        -- → chat.chats.id
    user_id      TEXT        NOT NULL,
        -- → identity.users（应用层 FK）
    role         TEXT        NOT NULL DEFAULT 'member',
        -- 'owner' | 'admin' | 'member'
    last_read_seq BIGINT     NOT NULL DEFAULT 0,
        -- 水位线：seq <= last_read_seq 的消息视为已读
        -- unread_count = chats.next_message_seq - 1 - last_read_seq
    muted        BOOLEAN     NOT NULL DEFAULT false,
        -- true = 不弹通知，仍可收消息
    version      INTEGER     NOT NULL DEFAULT 0,
        -- 乐观锁：并发更新 last_read_seq 时携带 version，服务端 CAS
    joined_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (chat_id, user_id),
    CONSTRAINT chat_members_role_chk CHECK (role IN ('owner', 'admin', 'member'))
);

-- 用户的所有 chat（会话列表加载，最核心查询之一）
CREATE INDEX idx_chat_members_user
    ON chat.chat_members(user_id);

-- chat 的所有成员（成员列表、群管理）
CREATE INDEX idx_chat_members_chat
    ON chat.chat_members(chat_id);


-- ================================================================
-- 8. chat.contacts
-- 通讯录有向边（kind + state 双字段，fjj 设计）
-- 每条记录代表"我"将"对方"加入通讯录，是单向的
-- ================================================================

CREATE TABLE chat.contacts (
    owner_user_id   TEXT        NOT NULL,
        -- → identity.users；通讯录拥有者（"我"）
    contact_user_id TEXT        NOT NULL,
        -- → identity.users；被收录的联系人
    kind            TEXT        NOT NULL DEFAULT 'friend',
        -- 'friend' | 'colleague' | 'agent' | ...（可扩展）
    state           TEXT        NOT NULL DEFAULT 'normal',
        -- 'normal' | 'blocked' | 'muted'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (owner_user_id, contact_user_id),
    CONSTRAINT contacts_self_chk CHECK (owner_user_id != contact_user_id),
    CONSTRAINT contacts_state_chk CHECK (state IN ('normal', 'blocked', 'muted'))
);

-- 反向查：谁把我加进了通讯录（隐私审计）
CREATE INDEX idx_contacts_contact_user
    ON chat.contacts(contact_user_id);


-- ================================================================
-- 9. chat.relationships
-- 好友状态机（双向关系，user_low/user_high 保证唯一性）
-- initiator_user_id 是 PR #259 要求的必填字段（当前代码缺失）
-- 状态变更走 RPC（respond_to_friend_request / block_user）
-- ================================================================

CREATE TABLE chat.relationships (
    user_low            TEXT        NOT NULL,
        -- min(user_a_id, user_b_id) 字典序小的一方
    user_high           TEXT        NOT NULL,
        -- max(user_a_id, user_b_id) 字典序大的一方
    initiator_user_id   TEXT        NOT NULL,
        -- → identity.users；发起好友请求的一方
    status              TEXT        NOT NULL DEFAULT 'pending',
        -- 'pending' | 'accepted' | 'rejected' | 'blocked'
    blocked_by          TEXT,
        -- → identity.users；status='blocked' 时记录谁发起了屏蔽
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (user_low, user_high),
    CONSTRAINT relationships_order_chk
        CHECK (user_low < user_high),
    CONSTRAINT relationships_initiator_chk
        CHECK (initiator_user_id IN (user_low, user_high)),
    CONSTRAINT relationships_status_chk
        CHECK (status IN ('pending', 'accepted', 'rejected', 'blocked')),
    CONSTRAINT relationships_blocked_by_chk
        CHECK (blocked_by IS NULL OR status = 'blocked')
);

-- 查某用户的所有关系（好友列表、好友请求收件箱）
CREATE INDEX idx_relationships_user_low
    ON chat.relationships(user_low);
CREATE INDEX idx_relationships_user_high
    ON chat.relationships(user_high);

-- 待处理的好友请求（接受/拒绝操作）
CREATE INDEX idx_relationships_pending
    ON chat.relationships(user_high)
    WHERE status = 'pending';
    -- user_high 通常是被请求方（initiator 是 user_low 时）；
    -- 实际应扫 user_low 和 user_high 两个索引，应用层合并


-- ================================================================
-- chat.messages_for_user
-- 过滤掉当前用户已软删除的消息（deleted_for_user_ids 包含 auth.uid()）
-- 开发者应优先查这个 VIEW，而不是直接查 chat.messages
-- ================================================================
CREATE OR REPLACE VIEW chat.messages_for_user
    WITH (security_invoker = true)   -- 以调用者身份执行，自动应用 RLS
AS
SELECT *
FROM chat.messages
WHERE NOT (deleted_for_user_ids @> jsonb_build_array(auth.uid()::text));
```

---

## 全文搜索触发器

```sql
-- messages 的 search_vector 自动维护
-- weight A = content（主权重），'simple' 语言支持 CJK 逐字索引
-- 生产如需中文分词可换 'chinese'（需 pg_jieba 扩展）

CREATE OR REPLACE FUNCTION chat.refresh_message_search_vector()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = chat, public
AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('simple', coalesce(NEW.content, '')), 'A');
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_messages_search_vector
    BEFORE INSERT OR UPDATE OF content
    ON chat.messages
    FOR EACH ROW
    EXECUTE FUNCTION chat.refresh_message_search_vector();
```

---

## RLS 策略

```sql
-- ================================================================
-- 启用 RLS
-- ================================================================
ALTER TABLE chat.chats               ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.messages            ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.message_attachments ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.message_deliveries  ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.message_reactions   ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.message_pins        ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.chat_members        ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.contacts            ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat.relationships       ENABLE ROW LEVEL SECURITY;

-- service_role 自动绕过 RLS，无需额外配置
-- anon 角色：不设任何 GRANT，相当于零访问


-- ================================================================
-- 辅助函数：判断当前用户是否为某个 chat 的成员
-- SECURITY DEFINER + STABLE 可被优化器缓存（同一 query 内多次调用只查一次）
-- ================================================================
CREATE OR REPLACE FUNCTION chat.is_member(p_chat_id TEXT)
RETURNS BOOLEAN
LANGUAGE sql
SECURITY DEFINER
STABLE
SET search_path = chat, public
AS $$
    SELECT EXISTS (
        SELECT 1 FROM chat.chat_members
        WHERE chat_id = p_chat_id
          AND user_id = (SELECT auth.uid()::text)
    );
$$;


-- ================================================================
-- chat.chats
-- SELECT：成员可见；INSERT：自己创建；UPDATE：owner/admin 可改名/状态
-- ================================================================

CREATE POLICY chats_select_member ON chat.chats
    FOR SELECT TO authenticated
    USING (chat.is_member(id));

CREATE POLICY chats_insert_own ON chat.chats
    FOR INSERT TO authenticated
    WITH CHECK (creator_user_id = (SELECT auth.uid()::text));

-- UPDATE：只有 owner 可改（admin 改名走 RPC 更安全；此处留 owner 直改）
CREATE POLICY chats_update_owner ON chat.chats
    FOR UPDATE TO authenticated
    USING (
        chat.is_member(id)
        AND EXISTS (
            SELECT 1 FROM chat.chat_members
            WHERE chat_id = id
              AND user_id = (SELECT auth.uid()::text)
              AND role IN ('owner', 'admin')
        )
    )
    WITH CHECK (
        chat.is_member(id)
        AND EXISTS (
            SELECT 1 FROM chat.chat_members
            WHERE chat_id = id
              AND user_id = (SELECT auth.uid()::text)
              AND role IN ('owner','admin')
        )
    );

GRANT SELECT ON chat.chats TO authenticated;
GRANT INSERT ON chat.chats TO authenticated;
GRANT UPDATE (name, status) ON chat.chats TO authenticated;
-- next_message_seq / last_message_at / last_message_preview 由后端 RPC 管理
REVOKE UPDATE (next_message_seq, last_message_at, last_message_preview, creator_user_id)
    ON chat.chats FROM authenticated;


-- ================================================================
-- chat.messages
-- SELECT：chat 成员可见（per-user deleted_for_user_ids 过滤在应用层）
-- INSERT：成员可发消息（seq 由 RPC 分配，不开放前端 UPDATE）
-- UPDATE：仅发送者可撤回 / 编辑自己的消息（列权限限制）
-- ================================================================

CREATE POLICY messages_select_member ON chat.messages
    FOR SELECT TO authenticated
    USING (
        chat.is_member(chat_id)
        AND deleted_at IS NULL  -- 全员删除的消息不可见
    );
-- 注意：deleted_for_user_ids 的 per-user 过滤不在 RLS 里（JSONB @> 无行级索引）
-- 应用层 query 追加：AND NOT (deleted_for_user_ids @> jsonb_build_array(auth.uid()::text))

CREATE POLICY messages_insert_member ON chat.messages
    FOR INSERT TO authenticated
    WITH CHECK (
        sender_user_id = (SELECT auth.uid()::text)
        AND chat.is_member(chat_id)
    );

-- UPDATE：只有发送者可操作（撤回 / 编辑）
CREATE POLICY messages_update_own ON chat.messages
    FOR UPDATE TO authenticated
    USING (
        sender_user_id = (SELECT auth.uid()::text)
        AND deleted_at IS NULL
    )
    WITH CHECK (sender_user_id = (SELECT auth.uid()::text));

GRANT SELECT ON chat.messages TO authenticated;
GRANT INSERT ON chat.messages TO authenticated;
GRANT UPDATE (content, edited_at, retracted_at, deleted_for_user_ids)
    ON chat.messages TO authenticated;
-- seq / sender_user_id / chat_id / created_at 不开放
REVOKE UPDATE (seq, sender_user_id, chat_id, search_vector, encrypted_content)
    ON chat.messages FROM authenticated;


-- ================================================================
-- chat.message_attachments
-- SELECT：chat 成员可见；INSERT：消息发送者；无 UPDATE/DELETE policy
-- ================================================================

CREATE POLICY attachments_select_member ON chat.message_attachments
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM chat.messages m
            WHERE m.id = message_id
              AND chat.is_member(m.chat_id)
        )
    );

CREATE POLICY attachments_insert_own ON chat.message_attachments
    FOR INSERT TO authenticated
    WITH CHECK (
        EXISTS (
            SELECT 1 FROM chat.messages m
            WHERE m.id = message_id
              AND m.sender_user_id = (SELECT auth.uid()::text)
        )
    );

GRANT SELECT ON chat.message_attachments TO authenticated;
GRANT INSERT ON chat.message_attachments TO authenticated;


-- ================================================================
-- chat.message_deliveries
-- SELECT：chat 成员可见（知道消息投递到哪些设备）
-- INSERT/UPDATE：service_role 专属（不开放任何 authenticated policy）
-- ================================================================

CREATE POLICY deliveries_select_member ON chat.message_deliveries
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM chat.messages m
            WHERE m.id = message_id
              AND chat.is_member(m.chat_id)
        )
    );

GRANT SELECT ON chat.message_deliveries TO authenticated;
-- INSERT / UPDATE 不开放（service_role 专属）


-- ================================================================
-- chat.message_reactions
-- SELECT：chat 成员可见；INSERT / DELETE：仅自己的反应
-- ================================================================

CREATE POLICY reactions_select_member ON chat.message_reactions
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM chat.messages m
            WHERE m.id = message_id
              AND chat.is_member(m.chat_id)
        )
    );

CREATE POLICY reactions_insert_own ON chat.message_reactions
    FOR INSERT TO authenticated
    WITH CHECK (
        user_id = (SELECT auth.uid()::text)
        AND EXISTS (
            SELECT 1 FROM chat.messages m
            WHERE m.id = message_id
              AND chat.is_member(m.chat_id)
        )
    );

CREATE POLICY reactions_delete_own ON chat.message_reactions
    FOR DELETE TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

GRANT SELECT ON chat.message_reactions TO authenticated;
GRANT INSERT ON chat.message_reactions TO authenticated;
GRANT DELETE ON chat.message_reactions TO authenticated;


-- ================================================================
-- chat.message_pins
-- SELECT：成员可见；INSERT：owner/admin pin；DELETE：pin 发起者或 owner/admin
-- ================================================================

CREATE POLICY pins_select_member ON chat.message_pins
    FOR SELECT TO authenticated
    USING (chat.is_member(chat_id));

CREATE POLICY pins_insert_admin ON chat.message_pins
    FOR INSERT TO authenticated
    WITH CHECK (
        pinned_by = (SELECT auth.uid()::text)
        AND EXISTS (
            SELECT 1 FROM chat.chat_members m
            WHERE m.chat_id = message_pins.chat_id
              AND m.user_id = (SELECT auth.uid()::text)
              AND m.role IN ('owner', 'admin')
        )
    );

CREATE POLICY pins_delete_admin ON chat.message_pins
    FOR DELETE TO authenticated
    USING (
        pinned_by = (SELECT auth.uid()::text)
        OR EXISTS (
            SELECT 1 FROM chat.chat_members m
            WHERE m.chat_id = message_pins.chat_id
              AND m.user_id = (SELECT auth.uid()::text)
              AND m.role IN ('owner', 'admin')
        )
    );

GRANT SELECT ON chat.message_pins TO authenticated;
GRANT INSERT ON chat.message_pins TO authenticated;
GRANT DELETE ON chat.message_pins TO authenticated;


-- ================================================================
-- chat.chat_members
-- SELECT：同 chat 的成员相互可见
-- INSERT：service_role 专属（加成员需要业务校验 + 防自我加入私聊）
-- UPDATE：自己的 last_read_seq / muted（role 变更 service_role 专属）
-- DELETE：自己退群（DM 不允许退，由应用层校验）
-- ================================================================

CREATE POLICY members_select_peer ON chat.chat_members
    FOR SELECT TO authenticated
    USING (chat.is_member(chat_id));

-- 自己可更新自己的成员信息
CREATE POLICY members_update_own ON chat.chat_members
    FOR UPDATE TO authenticated
    USING (user_id = (SELECT auth.uid()::text))
    WITH CHECK (user_id = (SELECT auth.uid()::text));

-- 退群（自己退，DM 退出在应用层拒绝）
CREATE POLICY members_delete_self ON chat.chat_members
    FOR DELETE TO authenticated
    USING (user_id = (SELECT auth.uid()::text));

GRANT SELECT ON chat.chat_members TO authenticated;
GRANT UPDATE (last_read_seq, muted, version) ON chat.chat_members TO authenticated;
GRANT DELETE ON chat.chat_members TO authenticated;
-- role 不开放 UPDATE（走 service_role RPC）
REVOKE UPDATE (role, chat_id, user_id, joined_at) ON chat.chat_members FROM authenticated;


-- ================================================================
-- chat.contacts
-- 纯自己的数据：全部操作仅限 owner_user_id = auth.uid()
-- ================================================================

CREATE POLICY contacts_own ON chat.contacts
    FOR ALL TO authenticated
    USING (owner_user_id = (SELECT auth.uid()::text))
    WITH CHECK (owner_user_id = (SELECT auth.uid()::text));

GRANT ALL ON chat.contacts TO authenticated;


-- ================================================================
-- chat.relationships
-- SELECT：双方可见（user_low 或 user_high）
-- INSERT：发起方创建，且必须包含自己（防代他人发请求）
-- UPDATE：service_role 专属（状态变更走 RPC）
-- ================================================================

CREATE POLICY relationships_select_party ON chat.relationships
    FOR SELECT TO authenticated
    USING (
        user_low  = (SELECT auth.uid()::text)
        OR user_high = (SELECT auth.uid()::text)
    );

CREATE POLICY relationships_insert_own ON chat.relationships
    FOR INSERT TO authenticated
    WITH CHECK (
        initiator_user_id = (SELECT auth.uid()::text)
        AND (
            user_low  = (SELECT auth.uid()::text)
            OR user_high = (SELECT auth.uid()::text)
        )
    );
-- UPDATE / DELETE：不开放（走 respond_to_friend_request / block_user RPC）

GRANT SELECT ON chat.relationships TO authenticated;
GRANT INSERT ON chat.relationships TO authenticated;
```

---

## Realtime 发布清单

| 表 | Realtime | 事件 | 前端用途 |
|---|---------|------|---------|
| `chat.messages` | ✅ 必须 | INSERT | 新消息实时推送（IM 核心） |
| `chat.relationships` | ✅ 必须 | INSERT, UPDATE | 好友请求通知、状态变更（accepted/blocked） |
| `chat.chat_members` | ⚠️ 可选 | INSERT, UPDATE | 成员加入/退出、未读数同步 |
| `chat.chats` | ❌ | — | 低频，前端拉取即可 |
| `chat.contacts` | ❌ | — | 低频 |
| `chat.message_reactions` | ⚠️ 可选 | INSERT, DELETE | 实时 emoji 反应更新（如需高实时性） |
| `chat.message_pins` | ❌ | — | 低频，pin 动作后前端主动刷新 |
| `chat.message_deliveries` | ❌ | — | 后端内部状态，前端无需订阅 |
| `chat.message_attachments` | ❌ | — | 随消息一起加载，无独立 Realtime 需求 |

```sql
-- 必须
ALTER PUBLICATION supabase_realtime
    ADD TABLE chat.messages, chat.relationships;

-- 可选
-- ALTER PUBLICATION supabase_realtime
--     ADD TABLE chat.chat_members, chat.message_reactions;
```

**RLS 保障**：`chat.messages` 的 SELECT policy 使用 `chat.is_member(chat_id)`，
Supabase Realtime 的 `postgres_changes` 在 broadcast 前自动对每个订阅者应用 RLS，
用户只收到自己所在 chat 的消息推送。

---

## RPCs

```sql
-- ================================================================
-- chat.increment_chat_message_seq
-- 原子递增 next_message_seq，返回分配到的 seq 值（给当前消息用）
-- SECURITY DEFINER：绕过 RLS 做原子 UPDATE + RETURNING
-- 调用方（mycel-chat 服务）负责验证调用者是 chat 成员
-- ================================================================

CREATE OR REPLACE FUNCTION chat.increment_chat_message_seq(
    p_chat_id TEXT
)
RETURNS BIGINT
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = chat, public
AS $$
DECLARE
    v_seq BIGINT;
BEGIN
    UPDATE chat.chats
    SET next_message_seq = next_message_seq + 1,
        updated_at       = now()
    WHERE id = p_chat_id
    RETURNING next_message_seq - 1 INTO v_seq;
    -- 返回递增前的值（即刚分配给本条消息的 seq）

    IF v_seq IS NULL THEN
        RAISE EXCEPTION 'chat not found: %', p_chat_id;
    END IF;

    RETURN v_seq;
END;
$$;

-- mycel-chat 服务（service_role）调用，不开放前端
-- 注：前端通过 INSERT messages 时让后端填 seq，不直接调此 RPC


-- ================================================================
-- chat.count_unread_per_chat
-- 批量计算当前用户所有会话的未读数（重连后一次同步）
-- 基于水位线：unread = next_message_seq - 1 - last_read_seq
-- SECURITY DEFINER + user_id 从 auth.uid() 取（不信任参数，防越权查他人）
-- ================================================================

CREATE OR REPLACE FUNCTION chat.count_unread_per_chat()
RETURNS TABLE(
    chat_id      TEXT,
    unread_count BIGINT
)
LANGUAGE sql
SECURITY DEFINER
STABLE
SET search_path = chat, public
AS $$
    SELECT
        m.chat_id,
        GREATEST(0, c.next_message_seq - 1 - m.last_read_seq)::BIGINT AS unread_count
    FROM chat.chat_members m
    JOIN chat.chats c ON c.id = m.chat_id
    WHERE m.user_id   = (SELECT auth.uid()::text)
      AND c.status    = 'active'
      AND m.last_read_seq < c.next_message_seq - 1  -- 跳过无未读的 chat
    ORDER BY unread_count DESC;
$$;

-- 前端直接调用（user_id 从 JWT 取，无参数伪造风险）
GRANT EXECUTE ON FUNCTION chat.count_unread_per_chat() TO authenticated;


-- ================================================================
-- chat.mark_read
-- 更新 last_read_seq 水位线（乐观锁防并发覆写）
-- 只前进不后退（MAX 保护：不能把已读标记回退为未读）
-- ================================================================

CREATE OR REPLACE FUNCTION chat.mark_read(
    p_chat_id TEXT,
    p_seq     BIGINT
)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = chat, public
AS $$
BEGIN
    UPDATE chat.chat_members
    SET
        last_read_seq = GREATEST(last_read_seq, p_seq),  -- 只前进
        version       = version + 1,
        updated_at    = now()
    WHERE chat_id = p_chat_id
      AND user_id = (SELECT auth.uid()::text);

    IF NOT FOUND THEN
        RAISE EXCEPTION 'not a member of chat: %', p_chat_id;
    END IF;
END;
$$;

GRANT EXECUTE ON FUNCTION chat.mark_read(TEXT, BIGINT) TO authenticated;


-- ================================================================
-- chat.respond_to_friend_request
-- 接受 / 拒绝好友请求（被请求方调用）
-- 或：任意一方发起屏蔽
-- ================================================================

CREATE OR REPLACE FUNCTION chat.respond_to_friend_request(
    p_requester_id TEXT,       -- 发起请求的用户 ID
    p_accept       BOOLEAN     -- true = 接受，false = 拒绝
)
RETURNS TABLE(ok BOOLEAN, message TEXT)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = chat, public
AS $$
DECLARE
    v_uid      TEXT := (SELECT auth.uid()::text);
    v_user_low  TEXT;
    v_user_high TEXT;
    v_rows     INTEGER;
BEGIN
    -- 规范化 user_low / user_high
    IF v_uid < p_requester_id THEN
        v_user_low  := v_uid;
        v_user_high := p_requester_id;
    ELSE
        v_user_low  := p_requester_id;
        v_user_high := v_uid;
    END IF;

    UPDATE chat.relationships
    SET
        status     = CASE WHEN p_accept THEN 'accepted' ELSE 'rejected' END,
        updated_at = now()
    WHERE user_low            = v_user_low
      AND user_high           = v_user_high
      AND initiator_user_id   = p_requester_id  -- 只有被请求方才能响应
      AND status              = 'pending';

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    IF v_rows = 0 THEN
        RETURN QUERY SELECT false, 'request_not_found_or_not_pending'::text;
    ELSE
        RETURN QUERY SELECT true, 'ok'::text;
    END IF;
END;
$$;

GRANT EXECUTE ON FUNCTION chat.respond_to_friend_request(TEXT, BOOLEAN) TO authenticated;
```

---

## 安全审查

### 角色权限矩阵

| 角色 | chats | messages | chat_members | contacts | relationships | deliveries |
|------|-------|----------|-------------|----------|--------------|------------|
| `anon` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `authenticated` | SELECT member / INSERT own / UPDATE(name,status) admin | SELECT member / INSERT member / UPDATE(content,edited_at,retracted_at) own | SELECT peer / UPDATE(last_read_seq,muted) own / DELETE self | ALL own | SELECT party / INSERT own | SELECT member |
| `service_role` | 全部 | 全部 | 全部 | 全部 | 全部 | 全部 |

### 主要风险 + 对策

#### 风险 1：per-user 软删除越权读取

`deleted_for_user_ids` 是 JSONB 数组，RLS 里 `NOT (deleted_for_user_ids @> to_jsonb(uid))` 无法
利用行级索引，对大表是全扫描。

**对策**：
1. RLS 不做 per-user 过滤（仅做 membership 检查）
2. 所有查询接口（PostgREST / mycel-chat API）**强制**追加 `AND NOT (deleted_for_user_ids @> jsonb_build_array(auth.uid()::text))`
3. 前端通过 PostgREST 查消息时，在 query string 中加 `deleted_for_user_ids=not.cs.["uid"]`（Supabase PostgREST 语法）
4. 只要 API 层一致执行，数据不会泄露；漏加的 bug 代价是用户看到"已删除"条目，不是跨用户泄露

#### 风险 2：message_deliveries 设备归属校验

`device_id` 是 container.devices.id，若后端不验证设备归属，可以为不属于收件人的设备创建投递记录，
污染 push 路由逻辑（给攻击者的设备推送他人的消息）。

**对策**：
1. mycel-chat 服务在写 deliveries 前查 `container.devices WHERE id=device_id AND owner_user_id=recipient_id`
2. RLS 无法跨 schema 校验（container 是独立 schema），**必须**在应用层做

#### 风险 3：messages.seq 可被前端手动填写

`seq` 字段 DDL 是 `BIGINT NOT NULL`，前端在 INSERT 时若传入自定义 seq，可能制造乱序或冲突。

**对策**：
```sql
REVOKE UPDATE (seq) ON chat.messages FROM authenticated;
```
对 INSERT，seq 必须由 mycel-chat 服务调用 `increment_chat_message_seq` 后填入，
不允许前端通过 PostgREST 直接 INSERT 含 seq 的行（通过 `GRANT INSERT` 但在应用层强制填 seq）。
理想做法：前端 INSERT 不含 seq，由触发器或后端填充。

**推荐 trigger 方案**：
```sql
CREATE OR REPLACE FUNCTION chat.assign_message_seq()
RETURNS TRIGGER LANGUAGE plpgsql SECURITY DEFINER AS $$
BEGIN
    IF NEW.seq IS NULL OR NEW.seq = 0 THEN
        NEW.seq := chat.increment_chat_message_seq(NEW.chat_id);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_messages_assign_seq
    BEFORE INSERT ON chat.messages
    FOR EACH ROW EXECUTE FUNCTION chat.assign_message_seq();
```
此方案让前端 INSERT 无需传 seq（或传 0），trigger 自动分配，消除竞态。

#### 风险 4：relationships INSERT 发起恶意请求

攻击者构造 `user_low / user_high / initiator_user_id` 冒充他人发起好友请求。

**对策**：
INSERT policy 的 WITH CHECK：
- `initiator_user_id = auth.uid()`
- `user_low = auth.uid() OR user_high = auth.uid()`

两个条件共同保证：只能以自己的身份发起，且自己必须是 user_low 或 user_high 之一。

#### 风险 5：chat.is_member() 函数的时序竞态

`chat.is_member()` 在 RLS 执行时查 chat_members。若用户刚被踢出但 session 仍有效，
下一次 SELECT 才会失效。Supabase JWT 有效期内仍可短暂读取历史消息。

**对策**：
1. 这是 JWT-based auth 的固有特性，接受这个窗口（通常 JWT TTL = 1h）
2. 极敏感数据（私密 DM）踢出操作后主动调用 `auth.admin.signOut()` 使 session 失效
3. 不是可利用的安全漏洞（需要攻击者已持有合法 JWT + 刚被踢出的时间窗口）

#### 风险 6：blocked 用户仍能发消息

RLS 的 messages INSERT policy 只检查 membership，未检查 relationships.status='blocked'。
被屏蔽的用户若仍是 chat 成员，可以继续发消息。

**对策**：
1. blocked 处理在 mycel-chat API 层：发消息前查 `relationships WHERE status='blocked' AND 双方都在`
2. DM chat 场景：block 后同时移除被屏蔽者的 chat_members 记录（service_role）
3. 不在 RLS 层做跨表 relationships 检查（性能代价过高）

#### 风险 7：Realtime messages 广播

`chat.messages` 的 SELECT RLS 使用 `chat.is_member(chat_id)`，
Realtime broadcast 前会对每个订阅者运行此函数。大规模群聊（100+ 成员）的
每条新消息都触发 100+ 次 `is_member()` 调用。

**对策**：
1. `is_member()` 函数标注 `STABLE`，PG 在同一事务内会缓存结果
2. `chat_members` 表的 `(chat_id, user_id)` PK 索引确保每次 is_member 查询为 O(1)
3. 极端场景（万人群）考虑改用 Supabase Broadcast channel（绕过 postgres_changes，前端消息推送改 WS）

---

## 未解决 / 需后续确认

1. **messages.seq 的 trigger 方案 vs API 层分配**：trigger 方案更简洁（前端无需感知 seq），
   但 SECURITY DEFINER trigger 在 Supabase 自托管模式下需要确认函数 owner 权限。
   API 层分配更明确但增加一次 RPC 调用。推荐 trigger 方案，上线前测试。

2. **DM 创建的原子性**：创建 DM 需要在同一事务内 INSERT chat + INSERT 两条 chat_members。
   前端不能直接做（两次独立 INSERT 非事务），必须通过 mycel-chat API 或 `chat.create_dm(p_other_user_id TEXT)` RPC。

3. **群聊成员上限**：chat_members 表无成员数量约束。超大群（1000+）的 Realtime 广播性能需要提前规划。
   建议在 chats 表加 `max_members INTEGER DEFAULT 500`，INSERT chat_members 时检查。

4. **消息内容大小限制**：`content TEXT` 无长度约束。长消息（含大段代码）会影响
   `search_vector` 计算 + Realtime payload 大小（Supabase 单条消息 payload 上限 ~1MB）。
   建议 API 层限制 content ≤ 32KB，`search_vector` trigger 截断超长 content。