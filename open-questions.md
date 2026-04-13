# Schema 设计 — 待解决问题清单

> 2026-04-08 夜，从 review 中提取的未完成项

---

## 一、container 3 个核心表 DDL（最高优先）

container 是三层模型的核心，没有字段定义等于没有设计。

- [x] **container.devices** — 用户的计算端点。需要：设备类型、在线/离线状态、心跳时间、push_token、platform、能力声明
- [x] **container.environments** — 设备上的隔离执行环境（原 sandbox_instances）。需要：与旧 sandbox_instances 字段映射、provider 信息、状态机
- [x] **container.workspaces** — 环境内的项目工作区（原 sandbox_leases）。需要：与旧 sandbox_leases 字段映射、项目路径、状态

参考：fjj 文档中 staging.sandbox_leases / staging.sandbox_instances 的字段定义

---

## 二、全局视图（4 项）

所有表确认后才能完整输出，但可以先起草框架。

- [x] **跨 schema 依赖总矩阵** — 哪个 schema 读/写哪个 schema 的什么表，汇总成一张表
- [x] **全局 RLS 策略** — 每张开了 Realtime 的表需要什么 RLS 规则，防止数据泄露
- [x] **全局 Realtime 发布清单** — 哪些表需要加入 supabase_realtime publication
- [x] **全局索引策略** — 每个 schema 的关键查询和对应索引（chat 已做，其他 4 个没做）

---

## 三、深度不一致（4 个 schema 缺系统审查）

chat 做了 IM 审查 + 性能审查 + 安全审查。以下 schema 至少需要一轮同等深度的审查：

- [x] **identity** — DDL 新表有，但缺索引、RLS、Realtime 定义
- [x] **agent** — 零 DDL（只有表名和注释），缺索引、RLS、安全审查、Realtime
- [x] **container** — 核心表 DDL 待设计（见第一项），缺索引、RLS、安全审查、Realtime
- [x] **hub** — 零 DDL（只有 3 行描述），缺索引、RLS

### 每个 schema 审查应覆盖的维度

| 维度 | 内容 |
|------|------|
| DDL | 每张表完整 CREATE TABLE |
| 索引 | 关键查询对应的索引设计 |
| RLS | Supabase Row Level Security 策略 |
| Realtime | 哪些表需要开 Realtime，订阅什么事件 |
| 安全 | 权限检查、数据泄露防护、输入校验 |
| RPCs | 需要的存储过程 |

---

## 四、其他零散问题

- [x] **agent schema 没有 Realtime 定义** — threads 表可能需要 Realtime（agent 运行状态实时推送）
- [x] **container schema 没有 Realtime 定义** — devices 在线/离线状态可能需要 Realtime
- [x] **hub schema 非常薄** — 只有 3 行描述，需要补完整 DDL 和 marketplace_items 的索引（全文搜索 tsvector 已有）
- [x] **跨 schema RPC** — count_unread_per_chat 等 RPC 的 schema 归属和 search_path 配置

---

## 文件索引

| 文件 | 内容 |
|------|------|
| `/tmp/mycel-scan/32-schema-decisions.md` | 主决策文档（5 schema 表清单 + 34 条 FAQ） |
| `/tmp/mycel-scan/31-schema-v2.md` | 初版完整 DDL（已部分过时，被 32 覆盖） |
| `/tmp/mycel-scan/TODO-table-design.md` | 命名约定记录 |
| `/tmp/mycel-scan/00-scan-index.md` ~ `12-*.md` | Haiku 扫描原始数据 |
| `/tmp/mycel-scan/20-22-*.md` | Sonnet 分析报告 |
| `/tmp/mycel-scan/30-final-architecture.md` | Opus 初版架构（已部分被讨论覆盖） |
| `teams/member/yyh/current-runtime-database-reference-2026-04-08.md` | fjj 参考文档（表结构 ground truth） |
| GitHub OpenDCAI/Mycel#267 | PoC: Remote Sandbox 通信验证 |
| GitHub OpenDCAI/Mycel#268 | RFC: Mycel 模块拆分 |
