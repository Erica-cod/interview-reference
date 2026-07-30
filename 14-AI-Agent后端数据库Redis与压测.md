# AI Agent 后端：MongoDB、Redis、限流与压测

## 30 秒总述

> 项目按职责划分存储：LocalStorage 保存最近 10 轮消息用于页面秒开；MongoDB 保存用户、会话、原始消息、计划、派生记忆和 Agent 检查点，是事实源；Redis 保存 BFF Session、OIDC 临时状态、CSRF、认证限流和语义/工具缓存等高频短状态。MongoDB 适合会话聚合和可变结构，Redis 利用 TTL、原子计数与 `SET NX` 承担临时协调。项目提供 k6 SSE 压测和 Redis 限流微基准，但没有归档可对外宣称的最终吞吐数字。

## 一、存储边界

| 层 | 保存内容 | 定位 |
| --- | --- | --- |
| LocalStorage | 最近 10 轮完整消息 | 页面秒开，不是事实源 |
| MongoDB | users、conversations、messages、plans、memory、Agent checkpoint | 长期事实与可恢复状态 |
| Redis | Session、OIDC/CSRF、限流、语义缓存、工具缓存 | 高频、共享、短生命周期 |

必须说清楚：

- `messages` 保存全部原始消息；摘要和切片只是可重建派生层。
- Agent Session 主检查点在 MongoDB TTL 集合，不是全部存 Redis。
- SSE 并发槽和 LLM Queue 当前仍是单进程 `Map/Array`。
- Redis 不替代原始消息数据库。

## 二、为什么选择 MongoDB

主要访问模式：

```text
按 userId 查会话
按 conversationId + timestamp 分页读消息
追加消息、读取计划
按会话查摘要与记忆
恢复 Agent sessionState
```

选择原因：

- 数据以用户和会话为聚合边界，复杂 Join 较少。
- `sources[]`、`tasks[]`、摘要字段和 `sessionState` 嵌套且变化快。
- 支持复合索引、唯一索引和 TTL 索引。
- Atlas 可额外提供全文与向量搜索，但不能和普通 B-Tree 索引混为一谈。

为什么消息不嵌入 conversation：

- 消息无界增长，可能逼近单文档 16 MB 限制。
- 每条消息都更新同一个大文档，会形成写热点。
- 大文档传输成本高，不便分页和时间排序。
- 有界且整体读写的 `plan.tasks[]` 则适合嵌入。

建模原则：

```text
有界、一起读写 → 嵌入
无界、需要分页 → 独立集合并引用
```

代价与边界：

- 缺少关系库外键，关联一致性由应用保证。
- Schema 灵活不等于没有治理，需要版本与迁移。
- 订单、账单、组织权限等强关系强事务数据更适合 MySQL。

面试回答：

> 选择 MongoDB 不是因为它必然比 MySQL 快，而是当前核心数据以会话为聚合边界，嵌套数组和可变 AI 字段多，查询主要按用户、会话和时间进行，复杂 Join 与跨表事务少。若以后增加订单、账单和组织权限，我会用 MySQL 承担强关系数据，形成混合存储。

## 三、核心集合与索引

| 集合 | 用途 | 关键索引 |
| --- | --- | --- |
| `users` | 用户信息 | `userId unique` |
| `conversations` | 会话元数据 | `userId + updatedAt` |
| `messages` | 全部原始消息 | `conversationId + timestamp` |
| `plans` | 计划与有界任务数组 | `planId unique`、`userId + updatedAt` |
| `memory_items` | 原文检索切片 | 用户、会话、状态、发生时间 |
| `memory_summaries` | 摘要与来源映射 | 用户、会话、状态、创建时间 |
| `conversation_token_states` | 压缩触发状态 | 会话用户唯一、状态重试时间 |
| `multi_agent_sessions` | 中断恢复检查点 | `sessionId unique`、`expiresAt TTL` |

重要设计：

- `messages.clientMessageId` 使用 partial unique，防止超时重试重复写入。
- `memory_summaries.sourceMessageIds` 支持摘要回到原文。
- `stream_progress` 和 `json_repair_failures` 使用 TTL 自动清理。

索引原则：

```text
等值过滤字段在前
范围或排序字段在后
业务唯一 ID 使用 unique
临时状态使用 TTL
幂等请求使用 partial unique
```

索引不是越多越好，要用 `explain("executionStats")` 检查命中、扫描量和写放大。
