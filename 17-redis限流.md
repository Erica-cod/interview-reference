
## 八、Redis 在认证系统里的职责

Redis 不只负责限流，也承担认证会话和临时状态：

| 场景 | Redis 中保存的内容 | 生命周期与边界 |
| --- | --- | --- |
| BFF Session | 用户、access token、refresh token、id token | 中期 TTL，浏览器只持 sid |
| OIDC 登录状态 | state、nonce、code_verifier、returnTo | 短期 TTL，回调后删除 |
| 登录状态消费锁 | `login_lock:{state}` | 使用 `SET NX EX` 防并发处理和重放 |
| CSRF | 服务端 Token | TTL，Cookie 只保存 csrf session id |
| 认证限流 | IP、设备、用户、全局计数器 | 固定窗口，Redis 异常降级为本地计数 |

Redis 的总体定位：

> 保存高频、共享、短生命周期、适合 TTL 或原子协调的数据。原始消息和长期事实仍然保存在 MongoDB。

---

## 九、Redis 分布式限流背景

登录、OIDC callback 和 CSRF 都属于安全敏感接口：

- 登录接口可能被撞库、爆破或恶意频繁跳转。
- OIDC callback 会访问 Redis 登录状态，并调用 IdP `/token` 交换 Token。
- CSRF 接口会创建和读取 Redis Token，可能被恶意流量放大。

因此项目在 BFF 层使用 Redis 做分布式固定窗口限流：

```text
bff:ratelimit:auth_login:ip_xxx:时间桶
```

例如：

```text
同一个 IP 或设备一分钟内最多请求 N 次；
超过阈值直接返回 429 和 Retry-After。
```

使用 Redis 而不是单机内存，是为了让多个 BFF 实例共享同一套限流计数。

---

## 十、最初 INCR + EXPIRE 的一致性问题

最初实现：

```text
count = INCR key
if count == 1:
    EXPIRE key ttl
```

正常情况下可以工作，但两个独立命令之间存在一个很小的一致性窗口：

```text
INCR 成功
→ BFF 实例宕机、进程退出或 Redis 连接中断
→ EXPIRE 没有执行
→ Key 没有 TTL
```

如果 Key 没有时间桶兜底，可能造成用户持续被限流；即使 Key 包含时间桶，也会留下不能自动清理的垃圾数据。

优化目标：

- 计数器并发更新正确。
- 首次创建时一定设置 TTL。
- 尽量减少网络 RTT。
- 热点 Key 下尾延迟稳定。

---

## 十一、三种方案对比

### 11.1 Pipeline

```text
Pipeline:
  INCR key
  EXPIRE key ttl
```

优点：

- 将命令批量发送。
- 减少网络 RTT。
- 实现简单，吞吐通常不错。

问题：

- Pipeline 是网络批处理，不是事务，也不是锁。
- 不能天然表达“只有 count 等于 1 才设置 TTL”的条件逻辑。
- 当前基准脚本每次都会执行 `EXPIRE`，会持续刷新 TTL，与严格固定窗口语义不完全一致。

### 11.2 Lua

```lua
local count = redis.call("INCR", KEYS[1])
if count == 1 then
    redis.call("EXPIRE", KEYS[1], ARGV[1])
end
return count
```

优点：

- 在 Redis 服务端一次执行。
- 整个脚本具有原子性。
- 只需要一次 RTT。
- 不需要冲突重试。
- 只在首次创建计数器时设置 TTL。

### 11.3 WATCH/MULTI CAS

```text
WATCH key
GET key
PTTL key

MULTI
SET key 新值并保留 TTL
EXEC
```

如果读取之后、提交之前 Key 被其他客户端修改，`EXEC` 返回空，客户端重新读取和提交。

特点：

- 属于乐观并发控制。
- 不会真正阻塞其他请求。
- 低冲突时能够工作。
- 热点 Key 并发越高，冲突、重试和尾延迟越严重。

准确表述：

> Pipeline、Lua 和 WATCH/MULTI CAS 不是三种分布式锁，而是三种并发更新 Redis 限流计数器的方式。CAS 具有乐观锁语义，Lua 依靠 Redis 脚本原子执行，Pipeline 只是批量发送命令。

---

## 十二、压测是怎么做的

压测脚本：

```text
scripts/bench/bench-auth-rate-limit.js
```

配置：

| 参数 | 当前脚本配置 |
| --- | --- |
| 限流窗口 | 60 秒 |
| 测试 TTL | 62 秒 |
| 每种模式持续时间 | 默认 8 秒 |
| 并发梯度 | 10、50、100、200、300 |
| Redis 连接池 | `min(concurrency, 32)` |
| 测试模式 | Pipeline、Lua、CAS |

测试过程：

```text
for concurrency in [10, 50, 100, 200, 300]:
    创建最多 32 条 Redis 连接

    for mode in [pipeline, lua, cas]:
        删除该模式对应的测试 Key
        创建 concurrency 个异步 Worker

        每个 Worker 在 8 秒内循环：
            记录高精度开始时间
            操作同一个热点 Key
            记录耗时、错误数和重试次数

        汇总 RPS、P50、P95、P99、errors、retries
```

测试 Key 相互隔离：

```text
bench:auth_rl:pipeline:100
bench:auth_rl:lua:100
bench:auth_rl:cas:100
```

每轮开始前执行 `DEL`，避免上一轮计数干扰。

Lua 在测试前执行：

```text
SCRIPT LOAD
```

获得脚本 SHA，压测时使用：

```text
EVALSHA
```

这样可以避免每次传输完整 Lua 脚本文本，更接近生产使用方式。

---

## 十三、为什么看 P95、P99 和重试次数

| 指标 | 含义 |
| --- | --- |
| RPS | 每秒完成的限流计数操作数 |
| P50 | 中位数请求延迟 |
| P95 | 95% 请求在该耗时内完成 |
| P99 | 观察高并发下的尾延迟和抖动 |
| errors | Redis 操作失败数 |
| retries | CAS 因并发冲突产生的重试总数 |

认证接口不能只看平均值：

> 平均值会被大量快速请求稀释，但少量高延迟请求会直接影响登录体验。发生攻击或流量洪峰时，更需要关注 P95、P99 是否失控。

CAS 的重试指标可以解释性能变化：

```text
并发升高
→ WATCH 冲突增加
→ EXEC 返回 null
→ 重新 GET、PTTL、MULTI、EXEC
→ RTT 和尾延迟增加
```

---

## 十四、压测结论

结果趋势：

- Pipeline 吞吐不错，但只减少网络往返，不能提供完整业务原子性。
- CAS 在并发升高后冲突重试增加，P95、P99 更容易恶化。
- Lua 一次 RTT、服务端原子执行、不需要冲突重试，语义和稳定性更适合固定窗口热点计数器。

推荐的生产实现：

```text
Lua 原子完成：
INCR
+ 首次设置 TTL
+ 返回当前计数
```

该优化的价值不是单纯提高 Redis QPS：

> 它同时解决了 INCR 与 EXPIRE 之间的一致性窗口，并降低高并发场景下的网络和重试成本。

---

## 十五、Redis 限流完整面试故事

> 我们项目的登录、OIDC 回调和 CSRF 接口都属于安全敏感接口，所以在 BFF 层使用 Redis 做分布式限流。限流采用固定时间窗口，例如针对某个 IP、设备或用户，在一分钟内最多允许请求多少次。
>
> 最初的实现是先执行 `INCR`，如果返回值是 1，再执行 `EXPIRE` 设置过期时间。这个方案正常情况下可以工作，但我检查异常场景时发现，它存在一个很小的原子性窗口：如果 `INCR` 成功后服务实例宕机或 Redis 连接中断，`EXPIRE` 没有执行，这个 Key 就可能没有 TTL，带来持续误限流或者垃圾数据问题。
>
> 因此我没有直接凭经验修改，而是针对热点计数器设计了 Redis 微基准压测，对比 Pipeline、Lua 和 WATCH/MULTI CAS。Pipeline 可以减少 RTT，但只是批量传输，不能保证整个业务逻辑原子执行；Lua 在 Redis 服务端执行，只有计数为 1 时才设置 TTL，整个脚本具有原子性，而且只需要一次网络往返；CAS 使用 WATCH 监听计数器，再通过 MULTI/EXEC 提交，如果提交前 Key 被修改就重新执行，在热点 Key 下容易产生大量冲突重试。
>
> 压测设置了 10、50、100、200、300 五档并发，每种方案持续运行 8 秒。每档并发创建异步 Worker 持续操作同一个热点 Key，连接池最多 32 条连接。每轮开始前删除测试 Key；Lua 提前通过 `SCRIPT LOAD` 加载，测试阶段使用 `EVALSHA`，避免重复传输脚本文本。
>
> 我记录了 RPS、P50、P95、P99、错误数以及 CAS 重试次数，因为认证接口在高并发下不能只看平均值，更需要关注尾延迟。结果趋势是 Pipeline 吞吐不错但业务语义不够严谨；CAS 的冲突重试和尾延迟会随并发升高；Lua 依靠一次 RTT、服务端原子执行和零冲突重试，整体最稳定。因此对于固定窗口热点限流计数器，我们认为 Lua 是更合适的实现方向。
>
> 这个优化的价值不只是性能提升，更重要的是同时解决了 `INCR` 和 `EXPIRE` 之间的一致性问题。

---

## 十六、认证与 Redis 组合面试稿

> 项目的认证体系采用 OAuth2/OIDC Authorization Code Flow + PKCE。OAuth2 解决授权，OIDC 在其上增加身份层，双 Token 负责访问凭证和续期。BFF 在回调时使用 code 和 code_verifier 换取 access token、refresh token 和 id token，再通过 JWKS 校验 id token 的签名以及 `iss`、`aud`、`exp`、`nonce`。
>
> 出于安全考虑，我们没有让前端直接保存双 Token，而是由 BFF 把 Token 和用户信息保存在 Redis Session，浏览器只持有 HttpOnly Session Cookie。Redis 还负责 OIDC 临时状态、CSRF Token 和认证接口的分布式限流计数。
>
> 限流最初采用 `INCR` 加条件 `EXPIRE`，我发现两个命令之间存在失败窗口，因此设计了 Pipeline、Lua、WATCH/MULTI CAS 三组微基准压测。在 10 到 300 五档并发下，分别观察 RPS、P50、P95、P99、错误数和 CAS 重试次数。结果趋势表明，Lua 可以通过一次 RTT 原子完成自增和首次设置 TTL，不需要 CAS 的冲突重试，因此是固定窗口热点计数器更合适的实现方向。

---
