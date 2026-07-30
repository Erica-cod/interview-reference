# ZBotService 生产慢 SQL 接口排查

## 一、30 秒案例

> ZBotService 的练习排行榜在本地几百条数据时只有几十毫秒，但生产 `exercise_record` 达到十万级后超过 10 秒并触发接口超时。我先用接口分段计时把问题定位到排行榜 SQL，再还原真实 SQL 和生产数据分布。查询一次请求重复扫描练习表、逐行解析 JSON、全量聚合与排序，最后才分页，所以 `PageSize=10` 并没有减少前面的工作。短期通过合并扫描、提前过滤、减少 JSON 解析、缩小投影和匹配查询方式的联合索引降本；长期把指标结构化并迁移到事件表、日聚合表或排行榜快照，避免每次实时扫描全部历史记录。

## 二、现象与为什么本地没暴露

```text
本地：exercise_record 几百条，接口几十毫秒
生产：exercise_record 十万级，排行榜/统计 10s+，触发超时
```

小数据掩盖问题：

- 全表扫描两次仍很快，数据容易全部驻留 Buffer Pool。
- 本地缺少并发、锁竞争和资源争抢。
- 几百次 `JSON_VALUE` 解析成本不明显。
- 小规模 Hash Aggregate 和 Sort 不会 Spill 到 tempdb。

生产成本近似为：`扫描次数 × 数据量 + 每行 JSON 解析 + 全量聚合/排序 + 并发重复执行`。

## 三、原排行榜为什么慢

一次请求大致执行：

```text
扫描 exercise_record：统计次数和不同练习日期
再次扫描 exercise_record：过滤 /ie，JSON_VALUE 统计时长
扫描 exercise_event：按用户聚合 dwell_seconds
关联 user_info → 全量用户排序 → 最后 OFFSET/FETCH
```

核心 SQL 一次按 `state='Active'` 和 `user_id` 分组，统计次数及 `COUNT(DISTINCT CAST(create_time AS DATE))`；同一请求再次按 `state='Active' AND route='/ie'` 扫描，使用 `JSON_VALUE(record_data,'$.SpeechTime')` 汇总时长；最后才执行 `ORDER BY TotalPracticeDays DESC OFFSET/FETCH`。

所以 Top 10 实际是：

```text
读取十万条 → 聚合全部用户 → 排序全部用户 → 返回十条
```

`TOP/OFFSET` 只限制最终输出，不一定减少前面的 Scan、Aggregate 和 Sort。

## 四、第一步：先确定慢在哪一层

接口超时不一定是 SQL 慢，还可能是网关、连接池、锁、第三方服务或请求取消。

使用 `Stopwatch` 分别记录接口总耗时、排行榜查询、总数查询和其他处理，并在同一 `traceId` 下输出。

如果总耗时 12.3s、排行榜 SQL 12.1s，才能把“接口慢”缩小为“SQL 慢”。`SqlException.Number=-2` 通常指向 SQL Command Timeout；若网关先 504，还要检查取消信号是否传到数据库命令。

## 五、第二步：比较真实数据分布

分别查询总行数、Active 行数、用户数，以及按记录数排序的 Top 用户。

确认：

- 总量和 Active 占比。
- 是否少数用户数据严重倾斜。
- JSON 字段平均大小。
- 本地与生产索引是否一致。
- SQL Server 统计信息是否过期。

## 六、第三步：拿到真实 SQL

- 原生 SQL 记录模板和参数类型。
- EF Core 使用 `query.ToQueryString()`。
- Command Interceptor 记录 `traceId、SQL 模板、参数类型、耗时、返回行数、异常`。
- 生产日志必须脱敏，不长期输出 Token、邮箱和完整用户输入。

## 七、第四步：Query Store、阻塞和执行计划

Query Store 重点看 Duration、CPU、Logical Reads、执行次数和计划变化：

| 现象 | 可能原因 |
| --- | --- |
| Duration 高、CPU 低、`LCK_M_*` | 锁阻塞 |
| Duration 高、Logical Reads 高 | 扫描过多或索引不匹配 |
| CPU 与 Reads 都高 | JSON、函数、聚合成本高 |
| 平均正常、P99 高 | 锁、Spill、参数嗅探或争抢 |

实时阻塞通过 `sys.dm_exec_requests` 联合 `sys.dm_exec_sql_text` 查看 `wait_type、blocking_session_id、elapsed、CPU、logical_reads`。

在等量环境或只读副本开启：

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

Actual Plan 重点看：

- Scan 还是 Seek，实际读取多少行。
- Actual Rows 与 Estimated Rows 是否严重偏差。
- Key Lookup 执行次数。
- JSON 对应的 Compute Scalar。
- Hash Aggregate、Sort 是否 Spill 到 tempdb。
- 条件进入 Seek Predicate 还是普通 Predicate。

优化前后优先比较 `Logical Reads、Scan Count、Actual Rows Read、CPU、Elapsed、Spill`，不能只看一次毫秒数。

## 八、按假设逐步优化

1. 合并两个统计子查询：目标是 `exercise_record` Scan Count 从 2 降到 1。
2. 提前过滤 `user_id/state/parent_id`，减少进入 JSON、聚合和排序的行数。
3. 优先使用结构化 `dwell_seconds`，必要时才 `JSON_VALUE`。
4. 将 `speech_seconds、score、practice_date` 从 JSON 提取成结构化列。
5. 列表先分页，只投影需要字段；分享状态只查询当前页 ID。
6. 全局排行榜迁移到事件表、日聚合表或定时快照。

分享表数据量炸弹：

```csharp
var pageIds = exerciseRecords.Select(x => x.Id).ToList();
var sharedIds = db.ExerciseRecordShares
    .Where(x => pageIds.Contains(x.ExerciseRecordId))
    .Select(x => x.ExerciseRecordId).ToHashSet();
```

避免为 10 条当前页加载整张分享表，并给 `exercise_record_share(exercise_record_id)` 建索引。

## 九、联合索引怎么设计

原则：

```text
高频等值过滤/Join → Key 前部
范围与 ORDER BY → 等值字段之后
只用于返回 → INCLUDE
大 JSON/长文本 → 通常不放覆盖索引
```

单用户历史：

```sql
CREATE INDEX IX_exercise_record_user_history
ON exercise_record(user_id,state,parent_id,create_time DESC)
INCLUDE(id,route,title,speaker,dwell_seconds);
```

全局排行榜查询是：

```sql
WHERE state='Active' GROUP BY user_id
```

索引 `(user_id,state,...)` 无法利用 `state` 直接 Seek；更匹配的是 `(state,user_id,create_time)` 或 Active 过滤索引。但即使索引命中，全局排行榜仍要处理大量 Active 数据，长期方案是预聚合，不是无限堆索引。

## 十、执行计划一句话

> SQL 是需求描述，执行计划是数据库完成需求的流水线；算子是流水线工序。排查就是看每道工序读多少行、输出多少行、执行多少次、占多少内存，以及是否出现扫描、回表、错误估算或落盘。

典型问题链：

```text
多次 Scan + JSON Compute Scalar
+ 多次 Hash Aggregate + 全量 Sort
```

对应优化：

```text
合并子查询 → 减少 Scan
提前过滤 → 减少后续输入行
结构化 JSON → 减少逐行计算
联合索引 → 改善 Seek/排序
预聚合快照 → 避免实时全量统计
```

## 十一、完整面试口述

> 当时生产排行榜接口报警超时，我没有直接加索引，而是先对接口和各 SQL 分段计时，确认绝大部分时间消耗在排行榜查询。再对比本地与生产数据，本地几百条、生产十万级，所以本地扫描没有暴露问题。
>
> 还原 SQL 后发现，一次请求重复扫描 `exercise_record`，逐行用 `JSON_VALUE` 解析时长，之后全量聚合用户并排序，最后才分页。数据库侧结合 Query Store、`STATISTICS IO/TIME` 和 Actual Plan，按 Duration、CPU、Logical Reads 和等待类型区分锁等待与扫描计算，再检查 Scan、Compute Scalar、Hash Aggregate、Sort、行数估算和 Spill。
>
> 短期合并重复扫描、提前过滤、减少 JSON 解析、缩小列表投影，并按查询模式设计联合索引；长期把统计字段结构化，迁移到事件表、日聚合或排行榜快照。优化不是只看接口毫秒数，还要验证 Logical Reads、Scan Count、Actual Rows Read 和 P95/P99。

## 十二、证据边界

- Git 能确认 SQL、索引、耗时日志和统计数据源做过调整。
- 仓库未保存生产 Query Store 截图、Actual Plan 或完整压测结果。
- 内存 LINQ 的 30 万条模拟不等于 SQL Server 执行计划测试。
- 只有本人确实使用过 Query Store，才说“通过 Query Store 定位”；否则说“会用 Query Store 补齐数据库侧证据”。
- 不把提交信息中的预估耗时包装成可复核的生产结果。
