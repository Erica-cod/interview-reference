# AI Agent 后端：MongoDB、Redis、限流与压测

## 30 秒总述

> 这个项目的数据按职责分成三层：浏览器 LocalStorage 只保存最近 10 轮完整消息，用于页面秒开；MongoDB 保存用户、会话、所有原始消息、计划、派生记忆和 Agent 检查点，是持久化事实源；Redis 保存语义响应缓存、BFF Session、OIDC 临时状态、CSRF Token 和全局限流计数等高频、短生命周期数据。
>
> MongoDB 不是因为一定比 MySQL 快，而是当前数据以会话为聚合边界，嵌套数组和 AI 派生字段多，Schema 变化快，查询又主要是按 `userId / conversationId / timestamp` 读写，复杂 Join 和跨表事务较少。Redis 则利用 TTL、原子计数、`SET NX`、Set 和 ZSet 解决 MongoDB 不适合承担的临时状态与高频访问。
>
> 项目有 k6 的 `/api/chat` SSE 压测脚本，也有 Node.js 直接压 Redis 的限流微基准，比较 Pipeline、Lua 和 WATCH/MULTI CAS 在 10～300 并发下的 RPS、P50、P95、P99。但仓库没有归档 Redis benchmark 的实际数值，所以面试时只能说“有可复现脚本”，不能声称已经验证了某个吞吐上限。

## 一、整体存储边界

```text
浏览器 LocalStorage
└─ 最近 10 轮完整消息
   └─ 目的：页面秒开，不是长期事实源

MongoDB
├─ users / conversations / messages / plans
├─ memory_items / memory_summaries
├─ conversation_token_states
├─ multi_agent_sessions
└─ stream_progress / json_repair_failures
   └─ 目的：持久化事实、派生记忆、可恢复检查点

Redis
├─ 语义响应缓存
├─ BFF Session / OIDC 临时状态 / CSRF
├─ 认证限流计数与消费锁
└─ 工具调用缓存
   └─ 目的：高频、共享、短生命周期和原子操作
```

必须明确：

- `messages` 保存所有原始消息，不能因为生成了摘要就删除。
- `memory_items` 和 `memory_summaries` 是可重建的派生层，不是唯一事实源。
- Redis 不是原始消息数据库。
- 当前主链路的 Agent Session 使用 MongoDB TTL 集合，不要说成全部存 Redis。
- 当前 SSE 连接限制和 LLM 请求队列是进程内 `Map / Array`，也不是 Redis。

---

## 二、为什么选择 MongoDB，而不是 MySQL

### 2.1 当前业务更接近文档模型

项目中的主要访问方式是：

```text
根据 userId 查会话列表
根据 conversationId 按时间读取消息
向一个 conversation 追加新消息
读取一条计划及其 tasks
根据 conversationId 检索摘要和记忆片段
恢复一个 Agent Session 的完整 sessionState
```

这类请求大多围绕单个用户或会话聚合，不依赖复杂多表 Join。

数据中还有大量嵌套或可变字段：

- `message.sources[]`
- `message.metadata`
- `plan.tasks[]`
- `memorySummary.goals[]`
- `memorySummary.preferences[]`
- `memorySummary.constraints[]`
- `sourceMessageIds[]`
- `embedding: number[]`
- `sessionState`

MongoDB 可以直接保存这些对象和数组。Agent 状态或模型返回结构升级时，也不必每增加一个可选字段就修改 SQL 表。

### 2.2 MongoDB 对当前项目的具体优势

- 文档结构适合会话、计划、摘要和 Agent 状态。
- Schema 演进成本较低，适合仍在快速迭代的 AI 应用。
- 支持普通复合索引和 TTL 索引。
- MongoDB Atlas 可以额外建立全文搜索和向量搜索索引。
- 原始消息、记忆切片和摘要可以分集合存储并独立重建。
- 按 `conversationId + timestamp` 追加和分页读取比较直接。

### 2.3 为什么不把所有消息嵌入 conversation

`conversation` 只保存标题、用户、更新时间、消息数量等元信息，消息单独放在 `messages` 集合。

原因：

- 会话可能持续增长，嵌入所有消息可能逼近 MongoDB 单文档 16 MB 限制。
- 每来一条消息都修改同一个大文档，容易形成写热点。
- 大文档更新和网络传输成本会持续上升。
- 历史消息不好单独分页和建立时间索引。

相反，`plan.tasks[]` 是有界、通常整体读取和修改的，因此可以嵌入 `plan` 文档。

这体现的是建模原则：

```text
有界、一起读写的数据 → 可以嵌入
无界增长、需要分页的数据 → 独立集合并引用
```

### 2.4 MongoDB 的代价

- 没有 MySQL 外键，ID 关联的一致性主要由应用代码保证。
- 灵活 Schema 可能产生历史数据格式不一致，需要版本字段和迁移程序。
- 复杂报表、关系查询和跨实体事务不如关系型数据库自然。
- Atlas Search/Vector Search 是额外能力，不能把它和普通 MongoDB B-Tree 索引混为一谈。

### 2.5 什么情况下应该使用 MySQL

如果项目增加以下核心业务，会更适合采用 MySQL 或混合存储：

- 订单、支付、退款和账单。
- 租户、角色、权限等强关系模型。
- 强外键约束。
- 跨多实体的强事务。
- 大量结构化统计和复杂 Join。

可以演进为：

```text
MySQL  → 账号、组织、权限、套餐、订单、账单
MongoDB → conversation、messages、AI memory
Redis   → Session、缓存、限流、锁、临时状态
```

### 面试回答

> 我选 MongoDB 不是因为它天然比 MySQL 快，而是因为当前核心数据以会话文档为聚合边界，消息来源、任务、摘要和 Agent 状态里有较多数组及可变字段，查询也主要按用户、会话和时间范围进行，几乎没有复杂 Join 和跨表事务。MongoDB 可以降低模型演进和嵌套数据映射成本，同时提供复合索引、TTL 和 Atlas 搜索能力。它的代价是缺少外键和复杂关系查询能力，所以如果以后加入订单、账单和组织权限，我会让 MySQL 承担强关系数据，而不是强行全部继续放 MongoDB。

---

## 三、MongoDB 集合结构

模型定义：`api/db/models.ts`

索引初始化：`api/db/connection.ts`

### 3.1 `users`

```ts
{
  userId: string,
  username?: string,
  createdAt: Date,
  lastActiveAt: Date,
  metadata?: {
    userAgent?: string,
    firstIp?: string
  }
}
```

主要索引：

```js
{ userId: 1 } unique
{ createdAt: 1 }
```

### 3.2 `conversations`

```ts
{
  conversationId: string,
  userId: string,
  title: string,
  createdAt: Date,
  updatedAt: Date,
  lastAccessedAt: Date,
  messageCount: number,
  isActive: boolean,
  isArchived?: boolean,
  archivedAt?: Date
}
```

主要索引：

```js
{ conversationId: 1 } unique
{ userId: 1, updatedAt: -1 }
{ userId: 1, isActive: 1 }
{ userId: 1, isArchived: 1 }
{ userId: 1, lastAccessedAt: -1 }
```

### 3.3 `messages`

```ts
{
  messageId: string,
  clientMessageId?: string,
  conversationId: string,
  userId: string,
  role: "user" | "assistant" | "system",
  content: string,
  contentPreview?: string,
  contentLength?: number,
  thinking?: string,
  sources?: Array<object>,
  modelType?: string,
  timestamp: Date,
  metadata?: {
    tokens?: number,
    duration?: number
  }
}
```

主要索引：

```js
{ messageId: 1 } unique

// 按会话正序读取历史消息
{ conversationId: 1, timestamp: 1 }

// 查用户某个会话的近期消息
{ userId: 1, conversationId: 1, timestamp: -1 }

// 防止客户端超时重试导致重复写消息
{ conversationId: 1, userId: 1, clientMessageId: 1 }
// partial unique：只有 clientMessageId 存在时参与唯一约束
```

`clientMessageId` 是幂等键。前端因为网络超时重发同一条消息时，数据库不会插入两份。

### 3.4 `plans`

```ts
{
  planId: string,
  userId: string,
  title: string,
  goal: string,
  tasks: Array<{
    title: string,
    estimated_hours?: number,
    deadline?: string,
    tags?: string[],
    status?: string
  }>,
  createdAt: Date,
  updatedAt: Date,
  isActive: boolean
}
```

主要索引：

```js
{ planId: 1 } unique
{ userId: 1, updatedAt: -1 }
{ userId: 1, isActive: 1 }
```

### 3.5 `memory_items`

从原始消息切分得到的检索单元：

```ts
{
  memoryId: string,
  messageId: string,
  conversationId: string,
  userId: string,
  role: string,
  chunkIndex: number,
  text: string,
  textHash: string,
  embedding?: number[],
  embeddingModel?: string,
  embeddingVersion?: string,
  embeddingStatus?: string,
  importance?: number,
  status: string,
  occurredAt: Date,
  createdAt: Date,
  updatedAt: Date
}
```

主要索引：

```js
{ memoryId: 1 } unique
{ userId: 1, conversationId: 1, status: 1, occurredAt: -1 }
{ messageId: 1, embeddingVersion: 1 }
```

### 3.6 `memory_summaries`

```ts
{
  summaryId: string,
  conversationId: string,
  userId: string,
  summary: string,
  goals: string[],
  preferences: string[],
  constraints: string[],
  sourceMessageIds: string[],
  fromMessageId?: string,
  toMessageId?: string,
  sourceTokenCount: number,
  summaryTokenCount: number,
  embedding?: number[],
  embeddingModel?: string,
  embeddingVersion?: string,
  embeddingStatus?: string,
  importance?: number,
  status: string,
  createdAt: Date,
  updatedAt: Date
}
```

主要索引：

```js
{ summaryId: 1 } unique
{ userId: 1, conversationId: 1, status: 1, createdAt: -1 }
```

`sourceMessageIds` 用来追溯摘要来源。如果摘要出现信息损失，可以回到原文验证或重新生成。

### 3.7 `conversation_token_states`

```ts
{
  conversationId: string,
  userId: string,
  lastInputTokens: number,
  lifetimeBillableTokens: number,
  unsummarizedTokens: number,
  summarizedThroughMessageId?: string,
  compressionStatus: "idle" | "running" | "failed",
  compressionRetryAt?: Date,
  compressionFailureReason?: string,
  usageSource?: "provider" | "estimated",
  createdAt: Date,
  updatedAt: Date
}
```

主要索引：

```js
{ conversationId: 1, userId: 1 } unique
{ compressionStatus: 1, compressionRetryAt: 1 }
```

三类 token 不要混淆：

- `lastInputTokens`：上一轮实际输入压力。
- `lifetimeBillableTokens`：整个会话累计模型消费，用于成本统计。
- `unsummarizedTokens`：自上次生成长期摘要后新增的原始内容，用于触发压缩。

### 3.8 `multi_agent_sessions`

```ts
{
  sessionId: string, // conversationId:assistantMessageId
  conversationId: string,
  userId: string,
  assistantMessageId: string,
  completedRounds: number,
  sessionState: object,
  userQuery: string,
  createdAt: Date,
  updatedAt: Date,
  expiresAt: Date
}
```

主要索引：

```js
{ sessionId: 1 } unique
{ sessionId: 1, userId: 1 }
{ expiresAt: 1 } TTL
```

这是当前主链路的 Agent 中断恢复检查点。

### 3.9 临时诊断集合

`stream_progress` 保存 SSE 续传所需的累计文本、思考内容、来源和发送位置，通过 `lastUpdateAt` 的 30 分钟 TTL 清理。

`json_repair_failures` 保存模型 JSON 修复失败的预览、策略和错误信息，通过 `expiresAt` TTL 清理。

### 3.10 索引设计原则

```text
等值过滤字段在前
排序或范围字段在后
唯一业务 ID 使用 unique
临时状态使用 TTL
幂等请求使用 partial unique
```

索引不是越多越好。每个索引都会占内存并增加写入维护成本，应继续使用：

```js
explain("executionStats")
```

检查是否真正命中索引、扫描了多少文档，以及是否存在冗余索引。

---

## 四、为什么使用 Redis

MongoDB 适合持久化事实，但以下数据更适合 Redis：

- 每次认证请求都会更新的限流计数。
- 需要自动过期的登录 state、CSRF 和 Session。
- 多实例之间共享的短期状态。
- 需要 `SET NX` 原子抢占的消费锁。
- 需要按时间或分数排序的缓存索引。
- 命中后希望毫秒级返回的语义缓存和工具缓存。

Redis 失败时要分场景处理：

- 语义缓存、工具缓存失败：降级为重新计算或直接访问后端，不阻断主业务。
- 认证限流 Redis 失败：降级为进程内计数，至少继续防洪。
- BFF Session、OIDC 和 CSRF 依赖 Redis：Redis 故障会影响登录态，不能笼统说所有 Redis 数据都可无损降级。

---

## 五、Redis Key 和数据结构

### 5.1 语义响应缓存

```text
embedding_cache:detail:{cacheId}
类型：String
值：JSON
内容：requestText、embedding、response、model、mode、hitCount、timestamp
```

```text
embedding_cache:user:{userId}:list
类型：ZSet
member：cacheId
score：createdAt
```

ZSet 用于按时间排序并删除最旧条目；详情单独用 String JSON 保存，便于整体设置 TTL。

当前策略大致是：

- 每个用户最多保留约 30 条。
- TTL 约 30 天。
- 极高相似度可直接复用响应。
- 中等相似度交给本地模型改写。
- 未命中才走正常模型链路。

### 5.2 BFF Session 和 OIDC

```text
bff:session:{sid}
类型：String JSON
TTL：约 7 天
```

浏览器 Cookie 只保存 HttpOnly Session ID，Access Token 等敏感内容放在服务端 Redis。

```text
bff:oidc:login:{state}
类型：String JSON
TTL：约 10 分钟
```

```text
bff:oidc:login_lock:{state}
类型：String
写入：SET key value NX EX 60
```

`NX` 表示不存在时才能写入，用于防止同一个授权 state 被并发重复消费。

OIDC Provider 还使用：

```text
oidc:{model}:{id}                 String JSON + TTL
oidc:grant:{grantId}              Set
oidc:uid:{model}:{uid}            String 索引
oidc:usercode:{model}:{userCode}  String 索引
```

Set 适合保存一个 Grant 关联的多个 Token/Session Key，撤销 Grant 时可以一次找到全部关联实体。

### 5.3 CSRF

```text
bff:csrf:{csrfSid}
类型：String
值：csrfToken
TTL：约 12 小时
```

浏览器通过 HttpOnly Cookie 保存 `csrfSid`，状态改变请求还要把 CSRF Token 放入请求头，服务端进行双重校验。

### 5.4 认证限流

```text
bff:ratelimit:{rule}:{subject}:{fixedWindowBucket}
类型：String Integer
操作：INCR、EXPIRE
```

例子：

```text
bff:ratelimit:auth_login_ip:10.0.0.8:29351234 = 18
```

`fixedWindowBucket` 把时间切成固定窗口。每来一个请求执行 `INCR`，超过阈值返回 429。

项目不是只按 IP 限流，而是可以组合：

- global：防止整个认证入口被打穿。
- IP：拦截单来源异常流量。
- device：降低 NAT 环境下多人共享 IP 的误伤。

### 5.5 工具调用缓存

```text
tool:cache:{toolName}:{md5(keyData)}
类型：String JSON + TTL
```

还有一个更长 TTL 的 stale 副本，在上游工具失败时允许返回旧结果。Redis 不可用时，工具缓存可以退化为进程内 Map。

当前按模式清缓存使用了 Redis `KEYS`。小规模可工作，但生产大 Key 空间中可能阻塞 Redis，应改成 `SCAN`、版本化 Key 或标签索引。

### 5.6 Redis 数据结构为什么这样选择

| 数据结构 | 项目用途 | 原因 |
| --- | --- | --- |
| String | Session、JSON 详情、Token、缓存 | 整体读取和整体 TTL 简单 |
| Integer String | 限流计数 | `INCR` 是原子操作 |
| ZSet | 用户缓存时间索引 | 能按 score 排序和删除最旧数据 |
| Set | OIDC Grant 关联实体 | 成员去重，便于批量撤销 |
| `SET NX EX` | 登录 state 消费锁 | 原子完成“仅首次写入 + 自动过期” |
| Pipeline | 批量发送 Redis 命令 | 减少网络往返 |

当前没有使用 Redis Stream 承载 Agent 或 SSE 队列。

---

## 六、Pipeline、Lua 和 WATCH/MULTI CAS 是什么

这三个不是三种限流算法，而是并发修改 Redis 计数器的三种实现方式。

假设需求是：

```text
一个用户在 60 秒内最多请求 100 次
```

Redis 保存：

```text
bff:ratelimit:login:user123:当前窗口 = 37
```

每来一个请求都要把 37 加一，并保证这个 Key 最终会过期。

### 6.1 Pipeline：减少网络往返

普通写法：

```text
客户端 -> Redis：INCR key
Redis -> 客户端：38
客户端 -> Redis：EXPIRE key 60
Redis -> 客户端：成功
```

Pipeline：

```js
redis.pipeline()
  .incr(key)
  .expire(key, 60)
  .exec();
```

它把多条命令批量发给 Redis，主要解决网络 RTT：

```text
一次网络往返
→ Redis 执行 INCR
→ Redis 执行 EXPIRE
```

优点：

- 网络往返少。
- 吞吐量高。
- 实现简单。

缺点：

- Pipeline 不是事务，不保证整组命令的业务原子性。
- 当前 benchmark 每次都执行 `EXPIRE`，会刷新 TTL。

如果 Key 本身包含固定窗口编号，刷新 TTL 不会改变当前窗口的计数边界，但会让已经结束的旧窗口 Key 晚一些被清理。

记忆：

> Pipeline 主要解决批量命令的网络性能问题。

### 6.2 Lua：在 Redis 服务端原子执行

```lua
local count = redis.call('INCR', KEYS[1])

if count == 1 then
  redis.call('EXPIRE', KEYS[1], 60)
end

return count
```

含义：

```text
计数器加一
→ 如果是第一次创建
→ 设置过期时间
→ 返回最新计数
```

优点：

- 一次网络往返。
- 整段脚本原子执行。
- TTL 只在第一次创建时设置。
- 适合限流、库存扣减和安全释放锁等短小复合操作。

缺点：

- 调试和维护比普通命令复杂。
- Lua 执行期间会阻塞 Redis 处理其他命令，脚本必须短小。
- Redis Cluster 下涉及多个 Key 时，要保证相关 Key 位于同一 slot。

记忆：

> Lua 把判断、修改和设置过期合成服务端的一次原子操作。

对于这个固定窗口限流，Lua 通常是三者里更合适的正式实现。

### 6.3 WATCH/MULTI CAS：乐观锁

CAS 是 Compare And Set，可以理解为：

```text
我先读取数据
→ 提交前确认这期间没人修改
→ 没人修改才提交
→ 被别人修改过就重新读取并重试
```

流程：

```text
WATCH key
→ GET count
→ PTTL key
→ 客户端计算 count + 1
→ MULTI
→ SET 新 count，并保留原 TTL
→ EXEC
```

冲突示例：

```text
请求 A：WATCH，读到 37
请求 B：把 37 改成 38
请求 A：EXEC
Redis：返回 null，提交失败
请求 A：重新读取并重试
```

优点：

- 可以实现较复杂的“读取—判断—修改”。
- 不需要编写 Lua。
- 适合并发冲突较少的乐观更新。

缺点：

- 网络往返多。
- 热点 Key 并发越高，冲突和重试越多。
- P95/P99 尾延迟容易升高。
- 限流计数器正是高竞争热点，因此 CAS 通常不适合作为最终方案。

记忆：

> CAS 是先读再改，被别人抢先修改就重试；低竞争可用，高竞争容易重试放大。

### 6.4 三者对比

| 方案 | 网络往返 | 整体原子性 | 高竞争表现 | 适用场景 |
| --- | ---: | --- | --- | --- |
| Pipeline | 约 1 次 | 不保证 | 较好 | 简单批量命令 |
| Lua | 1 次 | 脚本整体原子 | 通常最好 | 限流、库存、锁等短逻辑 |
| WATCH/MULTI CAS | 多次 | 提交时校验 | 冲突越多越慢 | 低冲突的条件更新 |

### 6.5 当前正式代码是什么

当前认证限流正式代码是：

```ts
const count = await redis.incr(key);

if (count === 1) {
  await redis.expire(key, windowSec + 2);
}
```

这不是 benchmark 中的 Pipeline，也还没有使用 Lua。

正常情况下可以工作，但存在一个很小的故障窗口：

```text
INCR 成功
→ 进程或网络恰好在 EXPIRE 前失败
→ Key 可能没有 TTL
```

更严谨的升级方式是使用短 Lua 脚本，把 `INCR` 和首次 `EXPIRE` 合并成原子操作。

### 面试回答

> Pipeline 主要减少网络 RTT，但不能把多条命令变成一个原子业务操作；Lua 把 INCR 和首次设置 TTL 放在 Redis 服务端原子执行，一次往返，比较适合高竞争限流计数器；WATCH/MULTI 是乐观锁，提交前如果 Key 被其他请求改过就重试，高并发热点 Key 下容易产生大量冲突。因此 CAS 更适合作为对照方案，正式限流更适合 Lua。

---

## 七、项目做过什么压测

### 7.1 k6：测试 `/api/chat` SSE

目录：`test/k6/`

#### `chat_sse_smoke.js`

- 对象：`POST /api/chat`
- 默认：1 VU、10 秒
- 检查：200 或 429，以及 429 时的队列响应头。

#### `chat_sse_queue_429.js`

- 对象：`POST /api/chat`
- 默认：10 VU、20 秒。
- 配合降低 `MAX_SSE_CONNECTIONS`，主动触发 429。
- 检查 `Retry-After`、`X-Queue-Token` 和可选队列位置。

#### `chat_sse_large_markdown.js`

- 先请求 `/api/auth/csrf`，再请求 `/api/chat`。
- 默认 2 VU、6 次迭代。
- 请求文本大小：5K、20K、100K 字符。
- 指标：TTFB、总耗时、响应体大小、403 数量。
- 脚本阈值：P95 TTFB 小于 5 秒，总耗时小于 120 秒。

### 7.2 Node benchmark：直接测试 Redis 限流计数器

脚本：

```text
scripts/bench/bench-auth-rate-limit.js
```

它不是请求登录接口，而是绕过 HTTP、BFF 和业务逻辑，直接对 Redis 做微基准。

默认参数：

```text
并发档位：10、50、100、200、300
每种模式持续：8 秒
Redis 客户端池：最多 32 个连接
模式：pipeline、lua、cas
```

采集指标：

```text
RPS
P50
P95
P99
errors
CAS retries
```

三种测试操作：

```text
Pipeline → INCR + EXPIRE 批量发送
Lua      → 服务端原子 INCR，首次设置 EXPIRE
CAS      → WATCH + GET/PTTL + MULTI/EXEC，冲突后重试
```

### 7.3 当前压测证据的真实边界

仓库目前具备：

- Redis benchmark 脚本。
- k6 SSE 场景脚本。
- Jest 的少量并发和队列语义测试。
- 前端 Lighthouse、Playwright 性能结果。

仓库目前缺少：

- Redis benchmark 的实际 JSON/Markdown 结果。
- 完整的后端容量测试原始输出。
- 固定机器、Redis、MongoDB、应用实例和网络环境说明。
- CPU、内存、连接池、Redis 命中率和 MongoDB 慢查询曲线。

Git 历史中添加 benchmark 的提交说明也是“添加可复现脚本”，而不是“归档 benchmark 结果”。脚本最后只把 JSON 输出到控制台，没有自动写入文件。

因此不能把“脚本支持 300 并发”说成“系统已验证稳定支持 300 并发”。

### 7.4 当前 Redis benchmark 还需修正的地方

三种实现的语义还没有完全对齐：

- Pipeline 每次刷新 TTL。
- Lua 只在第一次创建 Key 时设置 TTL。
- CAS 保留已有 TTL，但可能大量重试。
- 当前 RPS 的 `opCount` 包含发生错误的逻辑操作，应同时报告成功 RPS。
- 用配置的持续秒数计算 RPS，任务尾部可能略微超时，应使用真实开始和结束时间。

正式对比前应当：

1. 统一三种方案的固定窗口和 TTL 语义。
2. 分开报告 attempted RPS 与 successful RPS。
3. 记录错误率和 CAS 平均/最大重试次数。
4. 至少预热一次并重复运行多轮。
5. 固定 Redis 版本、机器配置、本地或网络部署方式。
6. 将原始 JSON 和汇总 Markdown 一起提交。

### 7.5 面试中的准确说法

> 项目有两层性能测试。第一层用 k6 对 `/api/chat` 的 SSE 建连、限流排队和大文本场景进行测试，关注 TTFB、总耗时、错误率和 429 语义；第二层用 Node.js 直接压 Redis 热点计数器，在 10 到 300 并发下比较 Pipeline、Lua 和 WATCH/MULTI CAS，采集 RPS、P50、P95、P99、错误和重试。不过仓库没有归档 Redis 微基准的实际执行结果，所以我不会引用一个没有原始证据的具体吞吐数字。现阶段能确认的是测试方法和脚本已经具备，容量结论需要在固定环境下重跑并保存结果。

---

## 八、如果重新做一次可信压测

不要把所有依赖混在一次测试里，应分层定位瓶颈。

### 第一层：Redis 微基准

```text
目标：只比较限流计数实现
排除：HTTP、BFF、MongoDB、LLM
指标：成功 RPS、P50/P95/P99、错误率、重试率、Redis CPU
```

### 第二层：MongoDB 数据接口

```text
目标：会话列表、历史消息分页、消息写入、记忆检索
排除：真实 LLM
指标：RPS、P95/P99、连接池等待、扫描文档数、慢查询
```

### 第三层：BFF + SSE，Mock LLM

```text
目标：验证 BFF、鉴权、队列、SSE 建连和流式转发容量
方法：用可控延迟的 Mock LLM 返回固定 token 流
指标：建连成功率、TTFB、活跃连接数、429、断连率、CPU、内存
```

真实 LLM 的限流和响应抖动会污染应用服务器容量结果，因此容量压测要先使用 Mock LLM。

### 第四层：真实模型小并发验证

```text
目标：验证真实端到端体验与成本
指标：TTFT、TTLB、超时率、模型错误率、token 成本
```

真实模型测试主要看用户体验和外部供应商限制，不应该用来证明 BFF 自身的最大吞吐。

---

## 九、高频追问

### 1. 为什么不全用 MongoDB

> MongoDB 可以持久化 Session 和计数，但限流、锁、短期 Token 和高频缓存需要低延迟、TTL 以及原子计数。Redis 在这些访问模式下更合适，也可以由多个 BFF 实例共享。

### 2. 为什么不全用 Redis

> Redis 主要是内存系统，成本更高，也不适合把所有原始长文本消息和可审计事实长期放在里面。原始消息需要可靠持久化和分页检索，因此 MongoDB 是事实源，Redis 只是缓存和临时协调层。

### 3. Redis 挂了怎么办

> 语义缓存和工具缓存直接绕过；认证限流进入进程内保险模式；但是 BFF Session、OIDC 和 CSRF 会受到影响，所以 Redis 需要持久化、监控、主从或托管高可用，不能把所有 Redis 故障都描述成无感降级。

### 4. MongoDB 有事务吗

> MongoDB 支持事务，但当前主流程主要按单文档或单集合操作设计，尽量避免依赖跨集合事务。消息写入与记忆派生之间采用“原始消息先持久化、派生任务允许失败重建”的最终一致性思路。

### 5. MongoDB 索引是不是越多越好

> 不是。索引提高查询速度，但会增加写放大、内存和磁盘占用。索引必须从真实查询和排序方式出发，并用 `explain("executionStats")` 验证。

### 6. 为什么限流更推荐 Lua

> 因为这里既要原子增加计数，又要在第一次创建时设置 TTL。Lua 可以一次网络往返在 Redis 内部原子完成；Pipeline 只减少 RTT，CAS 在热点 Key 下会大量重试。

### 7. Redis 的 Pipeline 是事务吗

> 不是。Pipeline 只是把多条命令批量发送，减少网络往返；需要事务语义可以使用 MULTI/EXEC，需要带判断的原子复合操作通常用 Lua。

### 8. CAS 为什么在高并发下慢

> 多个请求同时 WATCH 同一个热点 Key，任何一个先提交都会让其他请求的 EXEC 失败。失败者必须重新读取和提交，并发越高冲突越多，尾延迟和重试量就越大。

### 9. 能不能说做过 300 并发压测

> 可以说脚本覆盖到 300 并发，但不能说系统已经稳定支撑 300 并发。前者是测试配置，后者必须有固定环境、原始结果、错误率、P95/P99 和资源曲线证明。

### 10. 项目当前最大的后端扩展性问题是什么

> SSE 限制和 LLM 队列当前是单进程内存结构。单实例可以保护本机资源，但横向扩容后不同实例无法共享排队状态。企业化部署需要根据语义拆分：本机连接保护仍可本地计数，全局业务配额和分布式队列则应迁移到 Redis、消息队列或专门的任务系统。

---

## 十、ZBotService 生产慢 SQL 案例

### 10.1 现象

```text
本地测试：
exercise_record 只有几百条
接口几十毫秒，感觉没有问题

生产环境：
exercise_record 达到十万级
排行榜或统计接口超过 10 秒
最终触发接口超时报警
```

本地没有暴露问题的原因：

- 几百条数据全表扫描两次仍然很快。
- 数据容易全部进入 Buffer Pool，物理 IO 少。
- 本地几乎没有并发、锁竞争和资源争抢。
- 几百次 `JSON_VALUE` 解析不明显。
- 小数据量 Hash 和 Sort 不会写入 tempdb。

生产数据量放大后，查询成本变成：

```text
扫描次数 × 数据量
+ 每行 JSON 解析
+ 全量分组聚合
+ 全量排序
+ 并发请求重复执行
```

### 10.2 本地 Git 记录能确认什么

ZBotService 的本地 Git 记录中有多轮相关优化：

| Commit | 内容 |
| --- | --- |
| `f0bc7f6` | 活跃值排行榜超时；合并重复扫描、增加耗时日志、`AsNoTracking` 和索引脚本 |
| `906f707` | 排行榜超过 10 秒；增加两个 `exercise_record` 联合索引 |
| `e1ff623` | 把排行榜统计从原始 `exercise_record` 迁移到更轻量的 `exercise_event` |
| `53466a0` | 历史列表去掉大 JSON 模糊搜索、提前过滤分页、缩小返回字段 |
| `6232a41` | 练习统计增加有效记录过滤，优先使用 `dwell_seconds`，必要时才解析 JSON |

提交记录中写过：

```text
本地数据：约 756 条
本地耗时：约 76ms
生产预计：10 万条以上
未优化时：10s+
```

证据边界：

- Git 能确认增加过接口开始/结束耗时日志。
- Git 能确认 SQL、索引和统计数据源改过。
- `6232a41` 的 30 万条测试是内存 LINQ 模拟，不是 SQL Server 实际执行计划测试。
- 仓库没有保存 Query Store 截图、生产 Actual Plan 或完整压测输出。
- “生产低于 100ms”在提交信息中是预估，不是可复核的生产测量结果。

面试时只有本人确实使用过 Query Store，才能说“当时通过 Query Store 定位”。否则应说：

> 当时通过接口和 SQL 分段耗时定位到数据库查询；如果重新完整复盘，我会用 Query Store、STATISTICS IO/TIME 和实际执行计划补齐数据库侧证据。

### 10.3 原排行榜 SQL 为什么慢

一次排行榜请求大致执行：

```text
第一次扫描 exercise_record
→ 统计每个用户的练习次数
→ COUNT DISTINCT 练习日期

第二次扫描 exercise_record
→ 过滤 /ie
→ 对每行执行 JSON_VALUE(record_data, '$.SpeechTime')
→ 统计练习时长

扫描 exercise_event
→ 按 user_id 汇总 dwell_seconds

三个聚合结果关联 user_info
→ 按练习天数排序所有用户
→ 最后 OFFSET/FETCH 返回一页
```

核心 SQL 形态：

```sql
SELECT
    user_id,
    COUNT(*) AS TotalPracticeCount,
    COUNT(DISTINCT CAST(create_time AS DATE)) AS TotalPracticeDays
FROM exercise_record
WHERE state = 'Active'
GROUP BY user_id;
```

同一次请求又执行：

```sql
SELECT
    user_id,
    SUM(
        CAST(
            ISNULL(JSON_VALUE(record_data, '$.SpeechTime'), 0)
            AS INT
        )
    ) AS TotalPracticeSeconds
FROM exercise_record
WHERE state = 'Active'
  AND route = '/ie'
GROUP BY user_id;
```

最后才：

```sql
ORDER BY TotalPracticeDays DESC
OFFSET @Offset ROWS
FETCH NEXT @PageSize ROWS ONLY;
```

所以 `PageSize = 10` 只限制最终返回值，不限制前面的扫描、聚合和排序：

```text
读取十万条
→ 聚合成两万个用户
→ 排序两万个用户
→ 最后返回十条
```

### 10.4 当前代码中另一个数据量炸弹

旧历史列表为了设置 `IsShared`，每次请求都会加载整个分享表：

```csharp
var sharedRecordIds = dbContext.ExerciseRecordShares
    .Select(s => s.ExerciseRecordId)
    .Distinct()
    .ToHashSet();
```

即使当前页只有 10 条，也会读取全部分享记录。

应改成：

```csharp
var pageIds = exerciseRecords
    .Select(record => record.Id)
    .ToList();

var sharedRecordIds = dbContext.ExerciseRecordShares
    .Where(share => pageIds.Contains(share.ExerciseRecordId))
    .Select(share => share.ExerciseRecordId)
    .ToHashSet();
```

并建立：

```sql
CREATE INDEX IX_share_exercise_record_id
ON dbo.exercise_record_share(exercise_record_id);
```

---

## 十一、生产慢 SQL 怎么一步步排查

### 11.1 第一步：确认到底是哪一层超时

接口超时不一定是数据库慢，需要先区分：

```text
网关超时
ASP.NET 请求取消
SQL Command Timeout
获取数据库连接等待
数据库锁等待
第三方服务超时
```

给请求分段计时：

```csharp
var totalWatch = Stopwatch.StartNew();

var rankingWatch = Stopwatch.StartNew();
var ranking = ExecuteRankingQuery();
rankingWatch.Stop();

var countWatch = Stopwatch.StartNew();
var totalCount = QueryTotalCount();
countWatch.Stop();

logger.LogInformation(
    "Ranking completed total={Total}ms query={Query}ms count={Count}ms",
    totalWatch.ElapsedMilliseconds,
    rankingWatch.ElapsedMilliseconds,
    countWatch.ElapsedMilliseconds
);
```

如果日志是：

```text
接口总耗时：12.3s
排行榜 SQL：12.1s
总数 SQL：20ms
其他处理：不到 100ms
```

就可以把“接口慢”缩小成“排行榜 SQL 慢”。

如果异常包含：

```text
SqlException.Number = -2
Execution Timeout Expired
```

说明是 SQL Command Timeout。若网关先返回 504，但后台 SQL 仍继续执行，则要同时检查网关和请求取消是否真正传递到数据库命令。

### 11.2 第二步：比较本地与生产数据分布

不仅看总行数，还要看有效记录比例、用户倾斜和热点：

```sql
SELECT COUNT(*) AS TotalRecords
FROM dbo.exercise_record;

SELECT COUNT(*) AS ActiveRecords
FROM dbo.exercise_record
WHERE state = 'Active';

SELECT COUNT(DISTINCT user_id) AS UserCount
FROM dbo.exercise_record;

SELECT TOP 20
    user_id,
    COUNT(*) AS RecordCount
FROM dbo.exercise_record
GROUP BY user_id
ORDER BY RecordCount DESC;
```

要确认：

- 总数据量差多少。
- Active 占比多少。
- 是否少数用户拥有大量记录。
- `record_data` 平均大小是否很大。
- 生产索引和本地索引是否一致。
- 生产统计信息是否过期。

同一个 SQL 在均匀小数据和倾斜大数据下，优化器可能选择完全不同的计划。

### 11.3 第三步：拿到真实 SQL

原生 SQL 直接记录最终 SQL 模板和参数；EF Core LINQ 可以使用：

```csharp
var sql = query.ToQueryString();
```

也可以使用 EF Core Command Interceptor，记录：

```text
traceId
SQL 模板
参数类型和必要参数
执行时间
返回行数
异常类型
```

生产日志不要长期启用敏感参数输出，邮箱、Token 和用户输入需要脱敏。

### 11.4 第四步：用 Query Store 找历史慢查询

SQL Server Query Store 可以保留：

- SQL 文本。
- 执行计划。
- 执行次数。
- 平均与最大 Duration。
- CPU。
- Logical Reads。
- 执行计划是否发生变化。

SSMS 中可以进入：

```text
Database
→ Query Store
→ Top Resource Consuming Queries
```

按 Duration、CPU、Logical Reads 排序。

判断方向：

| 现象 | 更可能的原因 |
| --- | --- |
| Duration 高、CPU 低、`LCK_M_*` 高 | 锁阻塞 |
| Duration 高、Logical Reads 很高 | 扫描数据过多、索引不匹配 |
| CPU 和 Logical Reads 都高 | JSON、函数计算、聚合开销 |
| Physical Reads 高 | 读取数据量大或缓存未命中 |
| 平均正常但 P99 很高 | 锁、Spill、参数嗅探或资源争抢 |

### 11.5 第五步：检查实时阻塞

```sql
SELECT
    r.session_id,
    r.status,
    r.wait_type,
    r.blocking_session_id,
    r.total_elapsed_time,
    r.cpu_time,
    r.logical_reads,
    t.text
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.session_id <> @@SPID
ORDER BY r.total_elapsed_time DESC;
```

如果：

```text
wait_type = LCK_M_S
blocking_session_id != 0
```

重点查长事务和锁。

如果：

```text
blocking_session_id = 0
logical_reads 很高
CPU 也很高
```

更可能是扫描、JSON 计算、聚合和排序本身太重。

### 11.6 第六步：读取执行计划和 IO

在生产只读副本或生产等量环境中：

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

使用真实参数执行，并打开 Actual Execution Plan。

重点检查：

- Table Scan / Clustered Index Scan。
- Index Seek 实际读取了多少行。
- Actual Rows 与 Estimated Rows 差距。
- Key Lookup 的执行次数。
- Hash Aggregate 和 Sort 是否 Spill 到 tempdb。
- 是否出现隐式类型转换。
- 条件进入 `Seek Predicate` 还是只成为普通 `Predicate`。

优化前后不能只看毫秒，还应比较：

```text
Logical Reads
Scan Count
Actual Rows Read
Actual Rows
CPU Time
Elapsed Time
Spill
```

毫秒受缓存和机器负载影响，Logical Reads 更能说明数据库实际做了多少工作。

### 11.7 第七步：一次只验证一个假设

第一轮，合并两个统计子查询：

```text
目标：exercise_record Scan Count 从 2 降到 1
```

第二轮，先过滤：

```sql
WHERE user_id = @UserId
  AND state = 'Active'
  AND parent_id IS NULL
```

目标是减少进入 JSON、聚合和排序算子的行数。

第三轮，减少 JSON 解析：

```sql
CASE
    WHEN dwell_seconds IS NOT NULL
         AND dwell_seconds > 0
    THEN dwell_seconds
    ELSE COALESCE(
        TRY_CAST(JSON_VALUE(record_data, '$.SpeechTime') AS INT),
        0
    )
END
```

第四轮，提取结构化列：

```text
speech_seconds
score
practice_date
```

第五轮，迁移到日聚合表或排行榜快照。

这样才能知道每一步到底减少了什么成本。

---

## 十二、执行计划中的“算子”是什么

SQL 表达的是“我要什么”，执行计划表达的是“数据库准备怎样得到它”。

例如：

```sql
SELECT TOP 10
    user_id,
    COUNT(*) AS total
FROM dbo.exercise_record
WHERE state = 'Active'
GROUP BY user_id
ORDER BY total DESC;
```

可能得到：

```text
Index Scan
→ Filter
→ Hash Match (Aggregate)
→ Sort
→ Top
→ SELECT
```

其中“扫描、过滤、聚合、排序、取前 10 条”都是算子。执行计划是一棵树，SQL Server 图形计划通常从右向左阅读。

### 12.1 Index Seek

使用索引直接定位目标值或连续范围：

```text
B+ Tree 根节点
→ 中间节点
→ 目标叶子范围
```

例如索引：

```sql
(user_id, state, parent_id, create_time DESC)
```

查询：

```sql
WHERE user_id = @UserId
  AND state = 'Active'
  AND parent_id IS NULL
ORDER BY create_time DESC
```

可以直接定位目标用户的有效根记录，且结果已经按时间倒序。

Seek 不代表只读一行。如果某用户有 10 万条记录，Seek 也可能读取 10 万行。

### 12.2 Index Scan / Table Scan / Clustered Index Scan

`Index Scan` 表示扫描索引的大部分或全部内容。

`Table Scan` 表示扫描没有聚集索引的堆表。

`Clustered Index Scan` 基本相当于扫描整张聚集表，因为聚集索引叶子节点就是完整数据。

Scan 不一定错误：

- 表只有几十行时，扫描可能更便宜。
- 查询需要读取 90% 数据时，扫描可能合理。
- 扫描窄覆盖索引可能比扫描宽表便宜。

真正的问题是：

```text
读取 100 万行
→ 最终只返回 10 行
```

### 12.3 Filter 与 Seek Predicate

理想情况：

```text
索引利用条件直接定位
→ 条件出现在 Seek Predicate
```

不理想情况：

```text
先扫描大量行
→ Filter 或普通 Predicate 再过滤
```

最左匹配不完整时，后面的字段经常只能作为 Residual Predicate，不能缩小真正的 Seek 范围。

### 12.4 Key Lookup

索引中有查询条件，但没有返回列：

```text
Index Seek 找到主键
→ Key Lookup 回到聚集索引取 title、route 等字段
```

查 10 条时回表 10 次问题不大；查 10 万条时回表 10 万次可能非常慢。

执行计划要看：

```text
Actual Number of Executions
```

可以用 INCLUDE 减少回表：

```sql
CREATE INDEX IX_history
ON dbo.exercise_record(
    user_id,
    state,
    parent_id,
    create_time DESC
)
INCLUDE (
    id,
    route,
    title,
    speaker,
    dwell_seconds
);
```

但不要为了消灭 Lookup，把 `record_data` 之类的大 JSON 全塞进索引。

### 12.5 Compute Scalar

表示逐行计算表达式，例如：

```sql
CAST(create_time AS DATE)
JSON_VALUE(record_data, '$.SpeechTime')
DATEDIFF(...)
CASE WHEN ... END
```

单次计算可能很快，但总成本是：

```text
单次成本
× 实际处理行数
× 并发请求数
```

十万行 JSON 解析和十行 JSON 解析不是同一个量级。

### 12.6 Hash Match (Aggregate)

用于：

```sql
COUNT
SUM
GROUP BY
COUNT DISTINCT
```

数据库在内存中构造哈希表：

```text
userA → count=10, seconds=500
userB → count=18, seconds=920
```

适合大量、无序输入。

如果实际行数远高于预估，内存申请不足，中间数据会写入 tempdb：

```text
Hash Spill
```

这不是独立算子，而是 Hash 算子的执行警告，会显著放大尾延迟。

### 12.7 Stream Aggregate

如果输入已经按 `user_id` 排序：

```text
userA
userA
userA
userB
userB
```

数据库可以边读边聚合：

```text
读完 userA → 输出 userA
读完 userB → 输出 userB
```

它通常比维护大 Hash 表更省内存。

例如：

```sql
INDEX (state, user_id, create_time)
```

配合：

```sql
WHERE state = 'Active'
GROUP BY user_id
```

过滤 Active 后的数据天然更接近按 `user_id` 有序，可能有利于 Stream Aggregate。

### 12.8 Sort

普通历史列表可以让联合索引直接提供顺序：

```sql
INDEX (
    user_id,
    state,
    parent_id,
    create_time DESC
)
```

排行榜排序的是实时聚合结果：

```sql
ORDER BY COUNT(DISTINCT ...) DESC
```

原始表索引没有这个最终值，因此通常必须：

```text
聚合全部用户
→ Sort
→ Top 10
```

内存不足时还会出现 Sort Spill。

### 12.9 Nested Loops

工作方式类似：

```csharp
foreach (var outerRow in outerRows)
{
    FindInnerRows(outerRow.Key);
}
```

适合：

- 外层数据少。
- 内层关联字段有索引。

风险：

```text
外层十万行
× 内层一次查找
= 十万次查找
```

### 12.10 Hash Match Join

先把一侧放入哈希表，再用另一侧匹配。

适合：

- 两侧数据量较大。
- 没有适合 Merge 的顺序。
- 大批量等值关联。

风险是内存不足后 Hash Spill。

### 12.11 Merge Join

两侧都按关联字段有序时，像拉拉链一样合并：

```text
左 userA ↔ 右 userA
左 userB ↔ 右 userB
```

适合有索引顺序的大批量关联。如果为了 Merge Join 还要先对两边执行大 Sort，不一定划算。

### 12.12 Top 为什么不一定减少前面的工作

```text
Index Scan
→ Hash Aggregate
→ Sort
→ Top 10
```

虽然只返回 10 条，Top 位于流水线最后：

```text
扫描十万行
→ 聚合两万个用户
→ 排序两万个结果
→ 取十条
```

如果维护了预计算排行榜：

```sql
user_ranking(total_practice_days DESC)
```

才可能变成：

```text
按排行榜索引读取
→ Top 10
```

### 12.13 Actual Rows 和 Estimated Rows

每个算子都要比较：

```text
Estimated Number of Rows
Actual Number of Rows
```

例如：

```text
Estimated Rows = 1,000
Actual Rows = 120,000
```

优化器严重低估数据量后可能：

- 错误选择 Nested Loops。
- 产生大量 Key Lookup。
- 给 Hash/Sort 申请太少内存。
- Spill 到 tempdb。

常见原因：

- 统计信息过期。
- 数据分布倾斜。
- 参数嗅探。
- 隐式类型转换。
- 多个查询条件之间存在相关性。

不要只看图形计划上的“Cost 百分比”。它主要是基于估算的相对成本，不等于真实执行耗时。

### 12.14 把排行榜执行计划串起来

原始排行榜可能近似为：

```text
Clustered Index Scan exercise_record
→ Filter state='Active'
→ Compute Scalar CAST(create_time AS DATE)
→ Hash Aggregate 按 user_id 统计次数和天数
                         \
                          Hash/Merge Join
                         /
Clustered Index Scan exercise_record
→ Filter state='Active' AND route='/ie'
→ Compute Scalar JSON_VALUE(record_data)
→ Hash Aggregate 按 user_id 统计时长

exercise_event Scan
→ Hash Aggregate 按 user_id 汇总 dwell_seconds

三个结果
→ Join user_info
→ Sort TotalPracticeDays DESC
→ Offset/FETCH
```

问题不是某一个孤立算子，而是：

```text
多次 Scan
+ 大量 Compute Scalar
+ 多次 Hash Aggregate
+ 最后全量 Sort
```

优化一一对应：

```text
合并子查询
→ 两次 Scan 降为一次

先做有效条件过滤
→ 减少进入后续算子的行数

结构化 JSON 指标
→ 减少 Compute Scalar

联合索引
→ 支持 Seek、排序和有序聚合

日聚合表
→ 减少 Hash Aggregate 输入规模

排行榜快照
→ Top 10 不再依赖全量实时 Sort
```

记忆：

> SQL 是需求描述，执行计划是数据库完成需求的流水线；算子就是流水线上的每道工序。排查慢 SQL，就是看每道工序读了多少行、输出多少行、执行多少次、用了多少内存，以及有没有扫描、回表、错误估算或落盘。

---

## 十三、SQL Server 联合索引与最左匹配

假设索引：

```sql
(user_id, state, route, create_time)
```

物理排序顺序是：

```text
先 user_id
→ 同一 user_id 内按 state
→ 同一 state 内按 route
→ 同一路由内按 create_time
```

可以高效 Seek：

```sql
WHERE user_id = @UserId;

WHERE user_id = @UserId
  AND state = 'Active';

WHERE user_id = @UserId
  AND state = 'Active'
  AND route = '/ie';
```

不能完整使用最左前缀：

```sql
WHERE state = 'Active';

WHERE route = '/ie';

WHERE state = 'Active'
  AND route = '/ie';
```

下面的查询通常只能充分利用 `user_id`：

```sql
WHERE user_id = @UserId
  AND route = '/ie';
```

因为中间跳过了 `state`。

还要记住：

> 联合索引遇到第一个范围条件后，后面的字段通常不能继续缩小 Seek 范围，只能作为剩余过滤或覆盖字段。

例如索引：

```sql
(user_id, create_time, route)
```

查询：

```sql
WHERE user_id = @UserId
  AND create_time >= @StartTime
  AND route = '/ie';
```

`user_id` 是等值，`create_time` 是范围；`route` 通常只能对已经读取的时间范围继续过滤。

如果路由也是高频等值条件，更合适的是：

```sql
(user_id, route, create_time)
```

索引字段选择原则：

```text
高频等值过滤、Join 字段 → 索引 Key 前部
范围和 ORDER BY 字段     → 等值字段之后
只用于 SELECT 返回       → INCLUDE
大 JSON、大文本           → 通常不要放进覆盖索引
```

### 单用户历史索引

```sql
CREATE INDEX IX_exercise_record_user_history
ON dbo.exercise_record(
    user_id,
    state,
    parent_id,
    create_time DESC
)
INCLUDE (
    id,
    route,
    title,
    speaker,
    dwell_seconds
);
```

如果永远只查 Active 根记录，可以考虑更小的过滤索引：

```sql
CREATE INDEX IX_exercise_record_active_root_history
ON dbo.exercise_record(
    user_id,
    create_time DESC
)
INCLUDE (
    id,
    route,
    title,
    speaker,
    dwell_seconds
)
WHERE state = 'Active'
  AND parent_id IS NULL;
```

### 全局排行榜索引

历史提交曾增加：

```sql
(user_id, state, create_time)
(user_id, state, route)
```

它们适合单用户查询，但排行榜是：

```sql
WHERE state = 'Active'
GROUP BY user_id
```

没有指定最左侧 `user_id`，因此无法利用 `state` 直接 Seek。

更匹配全局查询的是：

```sql
CREATE INDEX IX_exercise_record_state_user_create
ON dbo.exercise_record(
    state,
    user_id,
    create_time
)
INCLUDE (
    route,
    dwell_seconds
);
```

也可以考虑过滤索引：

```sql
CREATE INDEX IX_exercise_record_active_user_create
ON dbo.exercise_record(
    user_id,
    create_time
)
INCLUDE (
    route,
    dwell_seconds
)
WHERE state = 'Active';
```

但如果大部分记录都是 Active，仍然要扫描大量数据。全局排行榜的长期方案仍是预聚合，不是无限堆索引。

---

## 十四、慢 SQL 面试口述

> 当时最初只知道生产接口报警超时，我没有直接假设是网络问题或盲目加索引，而是先给排行榜接口和数据库查询分别计时。日志显示绝大部分时间消耗在排行榜 SQL，异常也指向 SQL Command Timeout。接着对比本地和生产数据规模，本地只有几百条，生产已经是十万级，所以本地全表扫描没有暴露问题。
>
> 我把查询还原成实际 SQL 后发现，一次排行榜请求会扫描 `exercise_record` 两次：一次统计练习次数和天数，另一次逐行使用 `JSON_VALUE` 统计练习时长；之后还要扫描事件表、聚合全部用户、排序，最后才分页。因此即使只返回 10 条，数据库也要完成全量计算。
>
> 数据库侧我会结合 Query Store、`STATISTICS IO/TIME` 和 Actual Execution Plan。先根据 Duration、CPU、Logical Reads 和等待类型区分锁阻塞与扫描计算；再看两个 `exercise_record Scan`、JSON 的 Compute Scalar、Hash Aggregate、Sort、Key Lookup，以及实际行数和预估行数是否严重偏差。优化前后不只比较毫秒，还比较 Logical Reads、Scan Count、Actual Rows Read 和是否发生 tempdb Spill。
>
> 短期先合并两个统计子查询，把两次扫描降成一次；增加有效状态和根记录过滤；参数化 SQL；历史列表只投影当前页需要的列；再按照查询方式设计联合索引。比如单用户历史使用 `user_id、state、parent_id、create_time`，前三个完成等值过滤，最后一个直接提供倒序分页。
>
> 但全局排行榜即使加索引，仍然需要处理全部 Active 记录，所以最终不能只靠索引。我会把 JSON 中的统计指标结构化，并迁移到 `exercise_event`、日聚合表或排行榜快照，避免每次从全部历史原始记录重新计算。
