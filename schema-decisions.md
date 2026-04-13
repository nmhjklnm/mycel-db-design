# Schema 设计讨论决策记录

> 2026-04-08, 逐 schema 交互式审查

---

## Schema 1: identity (7 表) ✅ 已确认

### 表清单

| 表 | 来源 | 说明 |
|---|------|------|
| `identity.users` | staging.users (fjj doc) | 根身份，human + agent 共表 |
| `identity.accounts` | staging.accounts | 登录凭据 |
| `identity.assets` | staging.assets | 静态资源（头像、用户上传文件）。agent 产出物不进这里，走 container |
| `identity.user_settings` | public.user_settings 改造 | KV 模式：`(user_id, scope, config, updated_at)` |
| `identity.invite_codes` | staging.invite_codes | 邀请码，注册流使用 |
| `identity.model_providers` | NEW | 用户 LLM 提供商（连接 + 健康状态），有生命周期的资产 |
| `identity.model_mappings` | NEW | 模型配置（别名 + 参数 + 启用/禁用 + 当前激活） |

### 关键决策

1. **assets 放 identity** — 只存用户主动管理的静态资源，agent 产出物不进此表
2. **invite_codes 放 identity** — 控制"谁能成为用户"，是身份域关切
3. **user_settings 改 KV** — `(user_id, scope, config JSONB)`，scope 如 'general' | 'ui' | 'observation'
4. **sandbox_configs 不放 user_settings** — 沙箱配置随 device 注册走（container.devices）
5. **model_providers 独立表** — 有生命周期（状态、健康检查、错误），不是纯偏好
6. **model_mappings 独立表** — 需要 CRUD（启用/禁用、调参、切换激活），不适合 JSONB
7. **虚拟模型命名** — `leon:large` → `mycel:large`，`leon:mini` → `mycel:mini`，全量改名

### 表结构

```sql
-- identity.user_settings (KV 模式)
CREATE TABLE identity.user_settings (
    user_id    TEXT NOT NULL,
    scope      TEXT NOT NULL,           -- 'general' | 'ui' | 'observation'
    config     JSONB NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, scope)
);

-- identity.model_providers (LLM 提供商资产)
CREATE TABLE identity.model_providers (
    id            TEXT PRIMARY KEY,
    user_id       TEXT NOT NULL,
    name          TEXT NOT NULL,         -- "我的 302.ai"
    provider_type TEXT NOT NULL,         -- 'openai_compatible' | 'anthropic' | 'openrouter'
    base_url      TEXT NOT NULL,
    api_key_enc   TEXT NOT NULL,         -- 加密存储
    status        TEXT DEFAULT 'active', -- 'active' | 'error' | 'disabled'
    last_check_at TIMESTAMPTZ,
    last_error    TEXT,
    created_at    TIMESTAMPTZ DEFAULT now(),
    updated_at    TIMESTAMPTZ DEFAULT now(),
    UNIQUE (user_id, name)
);

-- identity.model_mappings (模型配置)
CREATE TABLE identity.model_mappings (
    id           TEXT PRIMARY KEY,
    user_id      TEXT NOT NULL,
    alias        TEXT NOT NULL,          -- 'mycel:large' 或 'claude-opus-4-6'
    provider_id  TEXT NOT NULL,          -- → identity.model_providers.id (应用层)
    model_name   TEXT NOT NULL,          -- 提供商侧真实模型名
    enabled      BOOLEAN DEFAULT true,
    is_active    BOOLEAN DEFAULT false,  -- 当前激活
    temperature  DOUBLE PRECISION,
    max_tokens   INTEGER,
    extra_config JSONB DEFAULT '{}',
    created_at   TIMESTAMPTZ DEFAULT now(),
    updated_at   TIMESTAMPTZ DEFAULT now(),
    UNIQUE (user_id, alias)
);
```

### 跨 schema 规则
- **所有模块可读 identity** — 获取用户名、头像、类型
- **写入仅限** auth_service（注册创建 user/account）、member_service（更新 profile）、settings router（偏好）
- **跨 schema FK 不加 DB 约束** — 应用层校验

### 命名约定
- 数据库: `container.devices` / `container.workspaces`
- 用户界面: 设备 / 工作区
- 虚拟模型: `mycel:large` / `mycel:medium` / `mycel:mini` / `mycel:max`

---

## Schema 2: chat (9 表) ✅ 已确认

### 表清单

| 表 | 来源 | 说明 |
|---|------|------|
| `chat.chats` | staging.chats (fjj doc) | 会话，含 next_message_seq + last_message_at + last_message_preview |
| `chat.messages` | staging.messages (fjj doc) | 消息，seq + 幂等 + per-user 删除 + 转发 + 线程 + 搜索 + E2E 预备 |
| `chat.message_attachments` | NEW | 消息附件，关联 identity.assets |
| `chat.message_deliveries` | NEW | 多设备投递追踪 |
| `chat.message_reactions` | NEW | emoji 反应（agent 反馈 + RLHF 信号） |
| `chat.message_pins` | NEW | 消息置顶（群公告、agent 关键产出） |
| `chat.chat_members` | staging.chat_members (fjj doc) | 成员 + last_read_seq 水位线 + muted + version |
| `chat.contacts` | staging.contacts (fjj doc) | 通讯录有向边（kind + state 双字段） |
| `chat.relationships` | staging.relationships (fjj doc) | 好友状态机（user_low/user_high + initiator_user_id） |

### 关键决策

1. **以 fjj 文档为准，非当前代码** — 代码尚未实现 seq 模型、initiator_user_id 等，设计按目标态走
2. **messages 加 `deleted_for_user_ids JSONB`** — per-user 软删除，IM 标配
3. **messages 加 `client_message_id TEXT` + UNIQUE(chat_id, client_message_id)** — 消息幂等，弱网重试防重复
4. **message_attachments 独立表** — 一条消息多个附件，关联 identity.assets
5. **message_deliveries 表** — Mycel 天然多设备（web + 本机 Electron + 未来移动端），单个 delivered_at 无法追踪每设备投递状态。推送通知也依赖此表
6. **message_reactions 表** — agent 平台场景 👍/👎 是自然反馈，也可用于训练信号
7. **chats 加 last_message_at + last_message_preview** — 会话列表排序必须，不加则前端每次 JOIN messages 取最新
8. **count_unread_per_chat RPC** — 用户重连后一次拉所有会话未读数，基于水位线
9. **push_token 归 container.devices** — 推送 token 是设备属性，不在 chat 单独建表
10. **message_reads 不要** — 被 chat_members.last_read_seq 水位线替代（PR #259 决策）
11. **contacts 用 kind + state 双字段** — fjj 设计，非代码中的单 relation 枚举
12. **relationships 必须有 initiator_user_id** — PR #259 要求，当前代码缺失
13. **RLS 策略必须** — Supabase Realtime 用 anon key，无 RLS = 任何人能订阅任何 chat
14. **关键索引** — messages(chat_id, seq DESC)、chat_members(user_id)、messages(sender_user_id)
15. **API 层安全** — blocked 用户检查、角色提权防护、消息内容大小限制

### 新增表结构

```sql
-- chat.message_attachments (消息附件)
CREATE TABLE chat.message_attachments (
    id         TEXT PRIMARY KEY,
    message_id TEXT NOT NULL REFERENCES chat.messages(id),
    asset_id   TEXT NOT NULL,         -- → identity.assets.id (应用层)
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- chat.message_deliveries (多设备投递追踪)
-- Mycel 天然多设备（web + Electron daemon + 未来移动端）
-- 追踪每设备投递状态 + 驱动推送通知（未投递设备 → push via container.devices.push_token）
CREATE TABLE chat.message_deliveries (
    message_id   TEXT NOT NULL,
    device_id    TEXT NOT NULL,         -- → container.devices（应用层）
    status       TEXT DEFAULT 'pending', -- 'pending' | 'delivered' | 'failed'
    delivered_at TIMESTAMPTZ,
    PRIMARY KEY (message_id, device_id)
);

-- chat.message_reactions (emoji 反应)
-- agent 平台场景：👍/👎 是对 agent 输出的自然反馈，可用作训练信号
CREATE TABLE chat.message_reactions (
    message_id TEXT NOT NULL REFERENCES chat.messages(id),
    user_id    TEXT NOT NULL,           -- → identity.users（应用层）
    emoji      TEXT NOT NULL,           -- '👍' | '👎' | '❤️' 等
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (message_id, user_id, emoji)
);

-- chat.message_pins (消息置顶)
-- 群聊里的重要消息（公告、决策、关键链接）需要置顶，否则被新消息淹没
-- agent 场景：agent 产出的关键结论/代码可以被用户 pin 住方便回看
CREATE TABLE chat.message_pins (
    chat_id       TEXT NOT NULL,           -- → chat.chats(id)
    message_id    TEXT NOT NULL REFERENCES chat.messages(id),
    pinned_by     TEXT NOT NULL,           -- → identity.users（应用层）
    pinned_at     TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (chat_id, message_id)
);
```

### chats 新增字段（相对 fjj 文档）

```sql
last_message_at      DOUBLE PRECISION   -- 会话列表排序（冗余，发消息时更新）
last_message_preview TEXT               -- "张三: 你好..." 摘要（冗余）
```

### messages 新增字段（相对 fjj 文档）

```sql
-- 幂等
client_message_id        TEXT             -- 客户端生成的 UUID，防弱网重发
-- UNIQUE (chat_id, client_message_id)    -- 同 chat 内去重

-- per-user 软删除
deleted_for_user_ids     JSONB DEFAULT '[]'

-- 转发溯源：用户转发消息到另一个 chat 时记录来源
-- agent 场景：agent 在一个 chat 里产出结论，用户转发给其他人或其他 agent
forwarded_from_message_id TEXT           -- → chat.messages(id)（应用层，可跨 chat）

-- 线程化回复：支持 Slack 式嵌套讨论
-- reply_to_message_id 是扁平回复（"回复某条"），thread_root_id 是线程根（"这条属于哪个讨论线"）
-- agent 场景：一个复杂问题下 agent 多轮展开，形成独立讨论线程而不污染主聊天流
thread_root_id           TEXT            -- → chat.messages(id)（应用层）

-- 全文搜索：agent 产出大量内容（代码、分析、日志），用户需要搜历史
-- PostgreSQL tsvector + GIN 索引，支持中文需 pg_jieba 扩展
search_vector            TSVECTOR        -- 由触发器自动维护

-- E2E 加密预备：为端到端加密预留字段
-- 加密后 content 置空，密文存 encrypted_content，解密需要客户端持有密钥
-- 元数据（sender_id, chat_id, created_at）保持明文以支持索引和排序
encrypted_content        BYTEA           -- 加密后的消息体
encryption_key_id        TEXT            -- 加密密钥标识，客户端用于查找解密密钥
```

### 索引设计

```sql
-- 消息分页（最常用查询：加载 chat 历史）
CREATE INDEX idx_messages_chat_seq ON chat.messages(chat_id, seq DESC);

-- 已删除过滤（排除全员删除的消息）
CREATE INDEX idx_messages_chat_seq_alive ON chat.messages(chat_id, seq DESC) WHERE deleted_at IS NULL;

-- 按发送者查（admin 审计、用户搜"我发过的"）
CREATE INDEX idx_messages_sender ON chat.messages(sender_user_id);

-- 全文搜索
CREATE INDEX idx_messages_search ON chat.messages USING GIN(search_vector);

-- 线程查询（加载某条消息下的讨论线程）
CREATE INDEX idx_messages_thread ON chat.messages(thread_root_id) WHERE thread_root_id IS NOT NULL;

-- 会话列表排序
CREATE INDEX idx_chats_last_message ON chat.chats(last_message_at DESC) WHERE status = 'active';

-- 用户的所有 chat
CREATE INDEX idx_chat_members_user ON chat.chat_members(user_id);
```

### 消息生命周期

```
发送 → created_at, client_message_id（幂等）
投递 → message_deliveries 按设备记录
已读 → chat_members.last_read_seq >= msg.seq
编辑 → edited_at, content 更新
撤回 → retracted_at, content → "[已撤回]"（时间窗口限制）
转发 → 新消息，forwarded_from_message_id 记录来源
置顶 → message_pins 插入
反应 → message_reactions 插入
删除(仅自己) → deleted_for_user_ids 追加 user_id
删除(全员) → deleted_at
加密 → content 清空，encrypted_content + encryption_key_id 填入
```

### Realtime 发布清单

| 表 | Realtime | 说明 |
|---|---------|------|
| `chat.messages` | ✅ 必须 | 新消息推送 |
| `chat.relationships` | ✅ 必须 | 好友请求通知 |
| `chat.chat_members` | ⚠️ 可选 | 成员变更 |
| `chat.chats` | ❌ | 低频 |
| `chat.contacts` | ❌ | 低频 |

### RPCs

```sql
chat.increment_chat_message_seq(p_chat_id TEXT)   -- 原子递增消息 seq
chat.count_unread_per_chat(p_user_id TEXT)         -- 一次拉所有会话未读数（重连后批量同步）
```

### 跨 schema 规则
- **chat 读 identity** — 获取 sender 名字/头像（允许）
- **chat 读 container.devices** — message_deliveries 需要设备列表
- **agent 读 chat** — 发消息到 chat（通过 API，不直连）
- **chat 不读 agent** — 解耦 thread_repo 依赖（用回调或 identity.users 解析）

---

## Schema 3: agent (16 表) ✅ 已确认

### 表清单（每表附决策注释）

**组 1: Agent 定义 (4 表)**
```
agent.agent_configs       — 行为定义，1:1 绑定 agent user。PR #259 核心："慢变量"，改了影响所有 thread
agent.agent_rules         — 单条规则需独立增删改，不适合合进 configs 做 JSONB
agent.agent_skills        — 同上，技能需独立 CRUD
agent.agent_sub_agents    — 同上，子 agent 定义需独立 CRUD
```

**组 2: Thread 运行时 (2 表)**
```
agent.threads             — 纯运行时实例。不加 parent_thread_id（子 agent 关系是临时的，不持久化）
agent.thread_launch_prefs — 记录"上次成功的启动配置"，是执行历史不是纯偏好，不合并进 user_settings

~~thread_config~~ 删除 — 代码分析证实：6 个字段中 4 个和 threads 重复，queue_mode 从未读取，agent 字段注释 "never pass it"。历史遗留中间层，从未真正集成
```

**组 3: 执行状态 (5 表)**
```
agent.run_events          — 工具调用、token 用量追踪
agent.summaries           — 对话压缩。长对话必须，不能删
agent.message_queue       — agent 间内部 steering 消息（临时，消费后删）。不同于 chat.messages（永久聊天记录）
agent.file_operations     — 文件操作审计，支持时光回溯。不能删
agent.tool_tasks          — 长时间任务断点恢复。任务可能跑一小时，崩溃后需要从 DB 恢复进度，不能放内存
```

**组 4: 调度 (2 表)**
```
agent.schedules           — 原 cron_jobs 改名。人或 agent 均可创建定时任务
agent.schedule_runs       — 每次触发的执行记录（成功/失败/超时）。schedules 只记 last_run_at 不够，需要历史
```

**组 5: LangGraph (4 表)**
```
agent.checkpoints         — 对话状态存档。通过连接参数 search_path=agent 引导 LangGraph 建表到此 schema
agent.checkpoint_blobs    — LangGraph 管理
agent.checkpoint_writes   — LangGraph 管理
agent.checkpoint_migrations — LangGraph 版本追踪
```

### 删除的表及理由

| 删除的表 | 理由 |
|---------|------|
| ~~`panel_tasks`~~ | 不需要任务面板。schedules 覆盖定时触发需求 |
| ~~`agent_registry`~~ | 子 agent 运行时追踪放内存。threads.agent_user_id 已能追踪"哪个 agent 在跑" |
| ~~`thread_config`~~ | 代码从未读取此表（创建时不读、恢复时不读、运行时不读）。所有字段要么和 threads 重复，要么从未使用 |
| ~~`eval_*` ×4~~ | 开发工具，不进生产。本地 eval 需求用 SQLite 即可 |
| ~~`library_recipes`~~ | 重构为 container.sandbox_templates（沙盒创建蓝图属于 container 域） |

### 关键决策

1. **identity ↔ agent 循环引用** — `users.agent_config_id ↔ agent_configs.agent_user_id`，跨 schema 不加 DB FK，应用层校验。顺序：先建 user → 建 agent_config → 回填 user.agent_config_id
2. **一个 user 可创建多个 agent** — agent user 是 identity.users 中 type='agent' 的行，owner_user_id 指向创建者
3. **以 fjj 文档为准** — 代码中 agent_config_repo 仍用旧 member_id，需跟进 PR #259

---

## Schema 4: container (14 表) ✅ 已确认

### 实体层级

```
device ←── container ←── workspace ←── thread (连接，可切换)
                    ←── terminal

用户视角：设备 → 工作区（container 对用户不可见）
```

- device: 用户的机器，持久
- container: 设备上的隔离环境，可启停
- workspace: 容器内的项目目录，agent 连接到这里
- terminal: 容器内的 shell，和 workspace 是兄弟（都属于 container）
- thread ↔ workspace 是连接关系，可切换

### 表清单

**三层核心 (4)**
```
container.devices             — NEW。用户的计算端点，持久，有心跳/在线状态（TODO: 表结构）
container.sandbox_templates   — NEW。可复用的沙盒创建蓝图
container.sandboxes           — 原 sandbox_instances 重构。sandbox.device_id → devices
container.workspaces          — 原 sandbox_leases 重构。workspace.sandbox_id → sandboxes
```

**终端层 (5)**
```
container.terminals           — 原 abstract_terminals。terminal.sandbox_id → sandboxes（指向沙盒，非 workspace）
container.terminal_pointers   — 原 thread_terminal_pointers
container.terminal_sessions   — 原 chat_sessions 改名，避免 chat 域混淆
container.terminal_commands   — 命令历史
container.terminal_command_chunks — 流式输出，高频写入需批量缓冲
```

**存储 (2)**
```
container.volumes             — 原 sandbox_volumes。归 device（设备持久，workspace 可重建）
container.sync_files          — 文件同步 SHA256 校验
```

**监控 (2)**
```
container.resource_snapshots  — 原 lease_resource_snapshots
container.provider_events     — 提供商事件审计
```

**模板 (1)**
```
container.sandbox_templates   — 原 library_recipes 概念重构，沙盒创建蓝图
```

### 关键决策

1. **三层模型 device → sandbox → workspace/terminal** — 替代旧 lease/instance 二层
2. **terminal 指向 sandbox** — 终端是沙盒里的 shell，可在不同 workspace 间 cd
3. **thread ↔ workspace 连接关系** — agent.threads.current_workspace_id，可切换，非 1:1 绑定
4. **volumes 归 device** — 设备持久，workspace 可重建
5. **DB vs 用户命名** — DB: devices/sandboxes/workspaces；用户: 设备/工作区

### 旧表映射

| 旧表 | → 新表 |
|------|--------|
| sandbox_leases | → container.sandboxes |
| sandbox_instances | → 吸收进 sandboxes.provider_env_id（无独立表） |
| abstract_terminals | → terminals |
| thread_terminal_pointers | → terminal_pointers |
| chat_sessions | → terminal_sessions |
| sandbox_volumes | → volumes |
| lease_resource_snapshots | → resource_snapshots |
| library_recipes | → container.sandbox_templates（新概念，无直接前身） |

### TODO
- [x] devices 表结构（见 container-schema.md）
- [x] sandboxes 表结构（与旧 sandbox_instances 字段映射，见 container-schema.md）
- [x] workspaces 表结构（与旧 sandbox_leases 字段映射，见 container-schema.md）

---

## Schema 5: hub (3 表) ✅ 已确认

### 表清单

```
hub.marketplace_items        — 市场项目（skill/agent/member），含全文搜索 tsvector
hub.marketplace_versions     — 版本历史，每次发布一条 snapshot
hub.marketplace_publishers   — 发布者 profile（冗余存储，不 FK 到 identity.users）
```

### 关键决策

1. **已独立服务** — hub 是唯一已经独立部署的模块，schema 最干净
2. **无跨 schema 依赖** — publisher_user_id 只存 ID，不需要 FK 到 identity
3. **fjj 文档已有定义** — 字段直接沿用

---

## FAQ：讨论过程中的关键转向和决策点

### Q1: 为什么 schema 叫 identity 而不是 users？
最初选了 `users`，但发现 `users.users`（schema.table）读起来很别扭。`identity.users` 更自然——schema 是"身份层"，table 是"用户"。

### Q2: model_providers 为什么独立建表而不是放 user_settings JSONB？
最初想放 JSONB。但讨论后确认：模型提供商有生命周期（状态、健康检查、错误、启停），是动态资产不是偏好。资产需要独立 CRUD，JSONB 不合适。

### Q3: model_mappings 为什么也独立建表？
最初 mapping 放 user_settings JSONB。但用户需要对单个模型做启用/禁用操作，JSONB 数组的单元素更新很丑。独立表后一行一个模型，`UPDATE SET enabled = false` 就行。

### Q4: sandbox_configs 为什么从 user_settings 移出？
沙箱配置不是偏好，是随 device 注册走的。用户注册一台 MacBook 时带上它的沙箱能力，不是在"设置"页面配一个全局偏好。

### Q5: 为什么 chat 和 messaging 合成一个 schema 而不是两个？
messaging 的唯一消费者就是 chat。拆两个会制造人为边界：chat 前端 → chat API → 跨服务调用 → messaging API，多一跳没有收益。

### Q6: message_reads 表为什么删了？
PR #259 决策：用 chat_members.last_read_seq 水位线模型替代。不再追踪"谁读了哪条"，只追踪"谁读到了第几条"。写入量从 O(消息×读者) 降到 O(1)。

### Q7: thread_config 为什么删了？
代码分析证实：创建时不读、恢复时不读、运行时不读。6 个字段中 4 个和 threads 重复，queue_mode 从未被读取，agent 字段注释写着 "never pass it"。历史遗留中间层。

### Q8: panel_tasks 和 tool_tasks 为什么不同命运？
panel_tasks 是用户面向的任务面板——不需要，删。tool_tasks 是 agent 执行中的断点恢复——任务可能跑一小时，崩溃后需要从 DB 恢复，必须持久化。

### Q9: eval 表为什么不进生产？
纯开发工具，写入量大但只有开发者本机用。迁移到 Supabase 增加网络延迟和存储成本，无多租户收益。本地 SQLite 即可。

### Q10: agent_registry 为什么删了？
threads 表已有 agent_user_id 追踪"哪个 agent 在跑"。子 agent 关系是运行时临时的（跑完就没了），不需要持久化到独立表。

### Q11: 为什么叫 identity 不叫 auth？
`auth` 被 Supabase GoTrue 自带 schema 占了，不能用。

### Q12: assets 为什么放 identity 不独立？
assets 的 FK 指向 users.id，存的是用户主动管理的静态资源（头像、文件）。agent 产出物不进这里，走 container 的 volumes。assets 和 identity 绑定紧密。

### Q13: library_recipes 为什么从 agent 移到 container？
它描述的是"怎么启动一个工作区"（镜像、依赖、环境变量），是 container 域的事。agent 只是消费者。

### Q14: terminal 为什么指向 container 而不是 workspace？
终端是容器里的 shell 会话，你可以在同一个终端里 cd 切换不同工作区。terminal 和 workspace 是容器下的兄弟，不是父子。

### Q15: device/container/workspace 三层会不会过度设计？
确认保留三层。device 是持久的基础设施，container 是可启停的隔离环境，workspace 是项目上下文。向日葵式多设备管理需要这个粒度。

### Q16: thread 和 workspace 什么关系？
连接关系，非绑定。多个 thread 可同时连同一个 workspace。agent 切换工作区时框架提示文件上下文不可用，需切回访问。

### Q17: 数据库命名 vs 用户命名？
DB: container.devices / container.sandboxes / container.workspaces。用户界面: 设备 / 工作区。sandbox 层对用户不可见。

### Q18: leon → mycel 改名？
虚拟模型命名全量改：leon:large → mycel:large, leon:mini → mycel:mini 等。

### Q19: 为什么 message_deliveries 是 P0 不是未来再加？
Mycel 天然多设备：web 端 + 用户注册的本机设备（Electron daemon）+ 未来移动端。不是"可能有多设备"，是"一开始就是多设备"。单个 delivered_at 无法追踪每个设备的投递状态，推送通知也依赖此表（未投递的设备需要 push）。

### Q20: 为什么 chats 要冗余 last_message_at 和 last_message_preview？
会话列表按最新消息排序是 IM 的基本体验。不冗余的话，每次打开会话列表都要对每个 chat JOIN messages 取最新一条。50 个会话 = 50 次查询。冗余后一个 `ORDER BY last_message_at DESC` 搞定。代价是发消息时多一次 UPDATE chats，可以接受。

### Q21: count_unread_per_chat RPC 为什么必须？
用户掉线一小时后重连，需要一次拉取所有会话的未读数。逐 chat 查 `SELECT MAX(seq) - last_read_seq` 是 O(N) 次查询。RPC 在数据库侧一次算完返回，效率差几个数量级。

### Q22: 为什么 messages 要加 client_message_id？
弱网环境下客户端发消息，响应丢了，重试 → 同一条消息入库两次。client_message_id 是客户端生成的 UUID，服务端 `INSERT ... ON CONFLICT DO NOTHING` 实现幂等。这是 IM 基础设施级问题，不加必出重复消息。

### Q23: message_reactions 为什么现在就加？
Mycel 是 agent 平台。用户给 agent 输出点 👍/👎 是最自然的反馈方式，比打字说"不错"效率高得多。也可以作为 RLHF 训练信号收集。

### Q24: push_token 为什么归 container.devices 而不是 chat 单独建表？
push token 是设备属性——这台设备的 APNs/FCM token。设备注册时一起带上，chat 通过 message_deliveries.device_id 间接关联。放 chat 里是错误的归属。

### Q25: RLS 为什么是 P0？
Supabase Realtime 用 anon key 连接。无 RLS = 任何人可以 `supabase.channel('messages').on('INSERT', ...)` 收到所有 chat 的消息。这不是"安全加固"，是"不加就裸奔"。

### Q26: 为什么消息要支持转发（forwarded_from_message_id）？
agent 在一个 chat 里产出了结论/代码，用户想转发给另一个人或另一个 agent。不记录来源的话转发变成复制粘贴，丢失上下文溯源。一个字段的事，现在加比后面 ALTER TABLE 便宜。

### Q27: 为什么要 thread_root_id（线程化）？
reply_to_message_id 是扁平回复（"回复某条消息"），thread_root_id 是线程根（Slack 式"这条属于哪个讨论"）。agent 场景下一个复杂问题可能需要多轮展开讨论，线程化避免污染主聊天流。reply_to 和 thread_root 不冲突——一条消息可以同时属于一个线程且回复线程内的某条。

### Q28: 为什么 messages 加 search_vector？
agent 产出大量内容（代码、分析报告、日志）。"我让 agent 分析过一个 bug，在哪个对话里来着？"——没有搜索就只能手动翻。PostgreSQL tsvector + GIN 索引是零外部依赖方案。中文分词需 pg_jieba 扩展，但字段先占位，分词配置后加不影响 schema。

### Q29: E2E 加密为什么现在就预留字段？
不是现在实现加密，是预留 `encrypted_content BYTEA` + `encryption_key_id TEXT`。现在加两个空字段零成本；等要加密时只需填值。如果不预留，到时要 ALTER TABLE 加列 + 迁移存量数据 + 双写过渡，代价大得多。

### Q30: message_pins 为什么是独立表而不是 messages 加 pinned_at 字段？
一个 chat 里可以 pin 多条消息，需要查"这个 chat 所有 pinned 消息"。如果是字段，要 `WHERE chat_id = ? AND pinned_at IS NOT NULL`——可以工作但语义不清晰。独立表还能记录"谁 pin 的"（pinned_by），支持 unpin 就是删行。

---

## 全局视图

> 2026-04-13，所有 schema DDL 完成后汇总（34–38 文件已 commit）

---

### 一、跨 schema 依赖总矩阵

行 = 依赖方，列 = 被依赖方，格内列出具体的 应用层 FK 关系（无 DB 约束）。

| 依赖方 ↓ / 被依赖 → | **identity** | **chat** | **agent** | **container** | **hub** |
|-------------------|-------------|---------|----------|--------------|--------|
| **identity** | — | ❌ | ❌ | ❌ | ❌ |
| **chat** | `messages.sender_user_id` → users<br>`chat_members.user_id` → users<br>`contacts / relationships` → users<br>`message_attachments.asset_id` → assets | — | ❌ | `message_deliveries.device_id` → devices<br>（push 路由） | ❌ |
| **agent** | `agent_configs.agent_user_id` → users<br>`agent_configs.owner_user_id` → users<br>`threads.agent_user_id / owner_user_id` → users<br>`schedules.owner_user_id` → users<br>运行时读 model_providers + model_mappings（模型解析） | `run_events.message_id` → messages<br>（可选，关联 chat 消息） | — | `threads.current_workspace_id` → workspaces | ❌ |
| **container** | `devices.owner_user_id` → users<br>`sandboxes.owner_user_id` → users<br>`workspaces.owner_user_id` → users | ❌ | ❌ | — | ❌ |
| **hub** | `publishers.user_id` → users<br>`versions.published_by` → users | ❌ | ❌ | ❌ | — |

**结论**：
- identity 是纯根层，零外向依赖
- hub 是最孤立的叶节点，只依赖 identity.users（ID 存储，无 DB FK）
- agent 是依赖最广的模块（identity + container + 可选 chat）
- container 只依赖 identity（owner 冗余字段）
- **chat 缺失完整 DDL 和 RLS**（见下方一致性检查 #1）

---

### 二、全局 Realtime 发布清单

#### 汇总表

| schema | 表 | 事件 | 前端用途 | 优先级 |
|--------|---|------|---------|-------|
| identity | `identity.users` | UPDATE | chat 消息列表中头像 / 名字实时刷新 | 必须 |
| identity | `identity.model_providers` | UPDATE | 设置页：provider 探测失败即时提示 | 必须 |
| chat | `chat.messages` | INSERT | 新消息实时推送（IM 核心） | 必须 |
| chat | `chat.relationships` | INSERT, UPDATE | 好友请求通知 | 必须 |
| chat | `chat.chat_members` | UPDATE | 成员变更（入群/踢出/禁言） | 可选 |
| agent | `agent.threads` | UPDATE | `run_status` 变化驱动前端加载动画 / 状态栏 | 必须 |
| agent | `agent.tool_tasks` | INSERT, UPDATE | 长任务进度条（任务创建 + 进度更新） | 必须 |
| agent | `agent.schedules` | UPDATE | 设置页：next_run_at / last_run_at 实时更新 | 可选 |
| container | `container.devices` | UPDATE | 设备面板：在线 / 离线状态实时切换 | 必须 |
| container | `container.sandboxes` | INSERT, UPDATE | 沙盒启动进度（detached → creating → running） | 必须 |
| container | `container.workspaces` | INSERT, UPDATE | 工作区状态 + needs_refresh 完成通知 | 必须 |
| hub | `hub.marketplace_items` | UPDATE | item install_count / 状态变更 | 可选 |
| hub | `hub.marketplace_versions` | INSERT | 新版本发布通知（订阅了该 item 的用户） | 可选 |

#### 合并 ALTER PUBLICATION 语句

```sql
-- 必须开启的表（核心实时体验）
ALTER PUBLICATION supabase_realtime ADD TABLE
    identity.users,
    identity.model_providers,
    chat.messages,
    chat.relationships,
    agent.threads,
    agent.tool_tasks,
    container.devices,
    container.sandboxes,
    container.workspaces;

-- 可选（按需开启，低频或可轮询替代）
-- ALTER PUBLICATION supabase_realtime ADD TABLE
--     chat.chat_members,
--     agent.schedules,
--     hub.marketplace_items,
--     hub.marketplace_versions;
```

**注意**：Supabase Realtime 的 `postgres_changes` 在 broadcast 前自动应用 RLS，
但 `identity.users` 的 RLS 是 `USING(true)`——所有认证用户可读所有行，意味着任何用户的
profile 更新都会推给所有订阅者。**前端必须加 filter**（如订阅特定 user_id 集合）避免无关更新洪水。

---

### 三、全局 RLS 一致性检查

#### 问题清单

**#1 [阻塞] chat schema 完整 RLS 未定义**

32-schema-decisions.md 明确写了"RLS 策略必须——Supabase Realtime 用 anon key，无 RLS = 任何人能订阅任何 chat"，但至今没有 chat schema 的 DDL 文件。chat 有 9 张表，含 messages、relationships 等高敏感表，是当前设计中最大的空洞。

→ **待做**：参照 35（container）/ 36（identity）深度，补完 chat schema 完整 DDL + RLS + Realtime + RPC（至少覆盖 `chats / messages / chat_members / contacts / relationships`）。

---

**#2 [风险] identity.users USING(true) Realtime 广播放大**

policy 限定 `TO authenticated`（anon 无访问），但所有已登录用户的 UPDATE 事件都会推给全体订阅者。
大型部署下（1000+ 在线用户），一个用户改名会触发 1000+ 条 Realtime 推送。

→ 前端订阅时必须加 filter（e.g. `filter: 'id=in.(uid1,uid2,...)'`，只订阅当前 chat 里的成员）。
→ 未来如有保密 agent user 需求，考虑改为 `USING(id = auth.uid() OR type = 'human')`。

---

**#3 [需确认] hub 部分策略无 TO 角色限定 = anon 可访问**

hub 策略中 `publishers_select_all` / `items_select_published_or_own` / `versions_select_active_or_own` 未写 `TO authenticated`，依 PostgreSQL 规则会对 anon 角色生效（前提是 anon 有 table 的 GRANT）。

若 hub 是公开市场（未登录用户可浏览），这是期望行为；若需要登录才能访问，需加 `TO authenticated`。
→ **待确认**：hub 是否允许匿名浏览。当前假设允许，策略设计正确。

---

**#4 [性能] EXISTS 子查询 RLS 的热路径开销**

以下表的 RLS 策略使用 EXISTS 子查询验证父表归属：

| 表 | 子查询 | 覆盖索引 |
|---|-------|---------|
| `agent.agent_rules/skills/sub_agents` | `EXISTS (agent_configs WHERE owner_user_id = uid)` | `idx_agent_configs_owner`（已有） |
| `agent.run_events / summaries / file_operations / tool_tasks` | `EXISTS (threads WHERE owner_user_id = uid)` | `idx_threads_owner_active`（已有） |
| `agent.schedule_runs` | `EXISTS (schedules WHERE owner_user_id = uid)` | `idx_schedules_owner`（已有） |
| `container.workspaces` INSERT | `EXISTS (sandboxes WHERE owner_user_id = uid AND active)` | `idx_sandboxes_owner_active`（已有） |
| `identity.model_mappings` INSERT | `EXISTS (model_providers WHERE user_id = uid)` | UNIQUE(user_id,name) 前缀（已有） |

当前覆盖索引均已存在，无遗漏。但 `run_events` 高频 SELECT 时每行都触发一次 EXISTS，应在 API 层强制携带 `thread_id` 参数避免全表扫。

---

**#5 [安全] LangGraph 4 张表的 GRANT 红线**

`agent.checkpoints / checkpoint_blobs / checkpoint_writes / checkpoint_migrations` 无 RLS、无 GRANT。
前端通过 PostgREST 访问需要 GRANT，没有 GRANT 就无访问路径。

⚠️ **严禁** 执行 `GRANT SELECT ON agent.checkpoints TO authenticated / anon`（Supabase dashboard 误操作风险）。
checkpoints 中存有完整对话状态，含工具调用输入输出（可能含 API key、文件内容）。

---

**#6 [一致性] service_role RPC 内部校验不可省略**

所有 `SECURITY DEFINER` RPC 绕过 RLS，业务校验必须在 RPC 内显式执行：

| RPC | 关键内部校验 |
|-----|------------|
| `container.device_heartbeat` | `status != 'disabled'`（被封禁设备不能通过心跳复活） |
| `hub.publish_version` | `publisher.status != 'suspended'`（被封禁发布者不能发版本） |
| `identity.set_active_model` | `auth.uid()` 内部取，不信任参数（防 user_id 伪造） |
| `agent.set_thread_run_status` | 状态机合法转换矩阵（防非法状态跳转） |
| `hub.yank_version` | 调用者 `user_id = auth.uid()` 校验（防越权 yank） |

---

### 四、全局索引策略摘要

#### 各 schema 最关键查询路径

**identity（热路径：agent 每次调用都走）**
```
model_providers(user_id) WHERE status='active'     → idx_model_providers_user_active  ✅
model_mappings(user_id)  WHERE is_active=true      → idx_model_mappings_active         ✅
users(owner_user_id)     WHERE type='agent'        → idx_users_owner_agents            ✅
```

**chat（热路径：IM 核心）**
```
messages(chat_id, seq DESC)                        → idx_messages_chat_seq             ✅
chat_members(user_id)                              → idx_chat_members_user             ✅
chats(last_message_at DESC) WHERE active           → idx_chats_last_message            ✅
messages USING GIN(search_vector)                  → idx_messages_search               ✅
```

**agent（热路径：agent 重启 + 前端轮询）**
```
summaries(thread_id)     WHERE is_active=true      → idx_summaries_thread_active       ✅  ← 每次 agent 恢复都查
threads(owner_user_id, last_active_at DESC) WHERE active → idx_threads_owner_active   ✅
schedules(next_run_at ASC) WHERE enabled=true      → idx_schedules_next_run            ✅
message_queue(thread_id, id ASC)                   → idx_message_queue_thread         ✅
```

**container（热路径：心跳 + 状态协调）**
```
devices(owner_user_id, last_heartbeat_at DESC) WHERE online → idx_devices_online      ✅
sandboxes WHERE desired != observed AND active     → idx_sandboxes_state_mismatch     ✅
workspaces(refresh_hint_at ASC) WHERE needs_refresh → idx_workspaces_needs_refresh   ✅
```

**hub（热路径：市场浏览 + 搜索）**
```
marketplace_items USING GIN(search_vector)         → idx_items_search_vector          ✅
marketplace_items(install_count DESC) WHERE public → idx_items_install_count          ✅
marketplace_items USING GIN(tags)                  → idx_items_tags                   ✅
```

#### 已识别的缺失 / 待确认索引

| 表 | 缺失索引 | 影响 |
|---|---------|------|
| `chat.message_deliveries` | `(device_id)` | 设备下线时批量查未投递消息需要此索引；chat schema DDL 未完成，待补 |
| `chat.message_deliveries` | `(message_id, status)` | 查某条消息哪些设备未投递（推送补发逻辑）；待补 |
| `agent.run_events` | 月分区替代行级索引 | 无界增长表，单纯加索引无法根治；建议 `PARTITION BY RANGE(created_at)`，但 Supabase PostgREST 对分区表行为需验证 |
| `identity.assets` | `(owner_user_id, sha256)` WHERE active | 当前 sha256 索引不含 owner 前缀，同一哈希跨用户去重时需全表扫（低频，可接受） |
