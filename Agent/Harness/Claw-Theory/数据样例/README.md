# 数据样例索引

> 配合 Phase 7 各节笔记使用——读到"加载 SOUL.md"或"写入 sessions/<id>.json"时，来这里看**文件长什么样**。

## 配置类（启动时读）

| 文件 | 来源 | 用途 | 涉及章节 |
|---|---|---|---|
| [[01 - workspace 配置文件]] | `workspace/*.md` | 8 个 markdown 启动文件（SOUL / IDENTITY / TOOLS / USER / HEARTBEAT / BOOTSTRAP / AGENTS / MEMORY） | [[06 - Intelligence]] |
| [[02 - CRON 调度配置]] | `workspace/CRON.json` | cron job 定义（4 种调度类型） | [[07 - Heartbeat & Cron]] |

## 运行时数据（运行时写）

| 文件 | 来源 | 用途 | 涉及章节 |
|---|---|---|---|
| [[03 - 会话持久化]] | `workspace/.sessions/<id>.json` | 单个会话的消息历史 | [[03 - Sessions]] |
| [[04 - 投递队列]] | `workspace/outbox/*.json` | QueuedDelivery 单条投递任务 | [[08 - Delivery]] |
| [[05 - AuthProfile 状态]] | 内存（教学版）/ `workspace/profiles.json`（生产） | API key 轮换状态 | [[09 - Resilience]] |

## 路由与多 Agent（s05 抽象）

| 文件 | 来源 | 用途 | 涉及章节 |
|---|---|---|---|
| [[06 - 路由绑定]] | 内存（教学版）/ `workspace/bindings.json`（生产） | 5 级路由绑定 | [[05 - Gateway & Routing]] |

---

## 怎么用

1. **读笔记时遇到数据文件名** → 来这里找对应样例
2. **想看真实数据** → `claw0/workspace/` 目录下有完整范例（submodule 里）
3. **想改自己的 PuinClaw** → 复制样例，按需修改

## 真实出处

所有样例**都来自** claw0 教学版的 `workspace/` 目录（路径：[`claw0/workspace/`](../claw0/workspace/)）。教学版故意用简短示例（每个文件 10-30 行）让读者一眼看清结构。
