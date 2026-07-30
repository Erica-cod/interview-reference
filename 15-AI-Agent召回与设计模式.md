# AI Agent 召回与设计模式小抄

> 只把当前已实现、已测试的能力说成现状；Query 改写、Cross-Encoder、LLM-Wiki 和 Redis Streams 等仍属于后续研究。

## 一、30 秒项目定位

> 项目没有把向量库 Top K 直接拼给模型，而是围绕召回准确、证据完整、原文可追溯和 Token 可控设计记忆检索。写入时按语义结构切片，同时保存检索文本和原文；查询时使用 BM25 与向量多路召回并通过 RRF 融合；原文和摘要进入统一候选池，摘要命中后回源，原文命中后补邻居；K 与 K+1 接近时动态扩窗，最后在 Token Budget 内去重装箱。原始 `messages` 始终是事实源。

```text
原始消息 → 结构化切片/摘要 → BM25 + Embedding
→ RRF/统一重排 → 动态 Top K → 回源/邻居扩展
→ Token Budget 去重装箱 → LLM
```

## 二、固定分段问题怎么解决

> 固定分段破坏语义连续性是传统 RAG 的典型问题。项目从四层处理：第一，优先按标题、自然段和代码块边界切分，并保留 overlap；第二，用 BM25 与向量混合召回，兼顾精确 token 和语义改写；第三，摘要命中后根据来源 ID 回原文，原文命中后补前后邻居；第四，K 与 K+1 分数接近时动态扩窗，再在 Token Budget 内去重装箱。切片负责定位，层级关系负责导航，原文负责最终取证。

## 三、索引阶段

每个 Chunk 保存：

```text
rawText：未经改写的原文，最终提供给 LLM
retrievalText：补充章节标题和路径，用于 BM25/Embedding
parentId / sectionPath：层级信息
previousMemoryId / nextMemoryId：邻居关系
sourceMessageIds：摘要到原文的来源映射
```

切片原则：

- 优先语义边界，而不是只按固定字符数。
- overlap 一般可配置为 10%～20%，但要避免重复 Token 失控。
- 标题附加到 `retrievalText`，不篡改最终引用的 `rawText`。
- 原始消息不因生成摘要而删除。

## 四、为什么 BM25 和向量都要用

| 方式 | 擅长 | 弱点 |
| --- | --- | --- |
| BM25 | 错误码、ID、路径、版本号、代码符号 | 同义改写 |
| Embedding | 语义相似、自然语言改写 | 稀有 token、数字和精确字符串 |

> 向量解决“说法不同但意思相同”，BM25 解决“字符必须准确命中”，两者互补。

BM25 实现：

- 生产使用 MongoDB Atlas Search `$search`，底层 Lucene，中文使用 `lucene.cjk`。
- 本地/降级路径由 TypeScript 实现 BM25 公式，不依赖 BM25 NPM 包。
- 参数默认 `k1=1.2、b=0.75`，包含 IDF、词频饱和与文档长度归一化。

## 五、为什么用 RRF

BM25 与余弦相似度的分布不同，原始分数不适合直接相加：

```text
RRF(d) = Σ 1 / (k + rank_i(d))
```

RRF 只使用各路排名。某候选在词法和语义两路都靠前，融合后自然占优，也降低了跨检索器调分的敏感度。

## 六、原文和摘要如何统一竞争

统一候选：

```ts
{ id, kind: "raw" | "summary", content, sourceMessageIds,
  estimatedTokens, occurredAt, importance }
```

统一效用示意：

```text
utility = 0.6×相关性 + 0.2×保真度 + 0.1×信息密度 + 0.1×覆盖率
packingScore = utility / tokens^0.7
```

取舍：

- 原文保真度高但 Token 成本高。
- 摘要信息密度高但可能丢失精确细节。
- 概括问题可优先摘要；原话、日期、金额、错误码等问题优先保证原文。
- 原文和摘要进入同一候选池，不再分别取固定 Top K 后拼接。

## 七、摘要回源与邻居扩展

```text
摘要命中 → sourceMessageIds → 找来源原文
→ 用当前 Query 二次排序
→ 普通问题最多补 3 条，精确问题最多补 6 条

原文命中 → previousMemoryId / nextMemoryId
→ 补前后邻居，恢复跨 Chunk 上下文
```

摘要是导航节点，不是唯一证据。当前摘要回源后不会继续递归扩邻居，避免图扩展爆炸；直接命中的原文会补邻居。

## 八、K+1 怎么处理

初始 `K=4`：

```text
0.90, 0.86, 0.82, 0.80 | 0.79, 0.60
gap = |score[K] - score[K+1]| / max(|score[K]|, ε)
```

- `0.80` 与 `0.79` 差约 1.25%，低于 8%，纳入 K+1。
- `0.79` 与 `0.60` 出现明显断层，停止扩窗。
- 同时受最大候选数和 Token Budget 限制。

与 Otsu 的区别：

| 动态 Top K | Otsu |
| --- | --- |
| 看 K 附近的局部分数间隔 | 看全体分数直方图 |
| 决定召回多少条 | 将样本分为两类 |
| 局部 elbow 检测 | 最大化类间方差 |

## 九、Token Budget 装箱

```text
模型上下文
- 输出预留 - 安全边界 - System Prompt
- 当前问题 - Tools Schema - 最近对话
= 长期记忆预算
```

装箱步骤：

1. 过滤效用为 0 的候选。
2. 按 `packingScore` 排序。
3. 选择能够放入剩余预算的候选。
4. 相同 ID 硬去重，摘要与来源原文按 `sourceMessageIds` 软去重。
5. 最终按时间顺序交给模型，保证可读性。

## 十、NotebookLM 与 LLM-Wiki 的取舍

NotebookLM：

> 它的产品价值是来源约束、跨源综合和原文引用，但内部切片、索引和重排机制没有完整公开，不能声称它已经抛弃 RAG，也不能说项目复刻了它。我们只借鉴“回答可回到来源”的原则。

LLM-Wiki：

```text
来源 → 编译成目录、摘要、元数据和双向链接
→ Agent search/read/follow links → 证据不足则继续检索
```

> 它适合稳定知识库和多跳问题，但对话记忆高频变化，Wiki 编译会增加写放大、版本、断链和事实校验成本。当前只吸收层级导航思想，仍以 MongoDB 原文为事实源；后续稳定知识可增加 Wiki 导航层，但不能替代原文。

## 十一、评测口径

当前小规模回归 Fixture：

| 指标 | 原文/摘要分路 Top K | 统一召回 |
| --- | ---: | ---: |
| Evidence Recall | 62.5% | 100% |
| Exact Evidence Recall | 0% | 100% |
| Redundant Token Ratio | 17.9% | 0% |
| 平均预算利用率 | 约 57.1% | 约 77.1% |

> 这些数据只证明已知退化用例得到修复，不代表大规模线上效果。下一步需要人工标注集、Recall@K、MRR/NDCG、答案正确率、引用支持率、P95 延迟、Token 成本和消融实验。

## 十二、三个设计模式

### 发布—订阅

```text
Tab A 更新会话 → BroadcastChannel 发布事件
→ Tab B/C 订阅并刷新；不支持时降级 storage event
```

> 发布者不依赖具体消费者，订阅返回 `unsubscribe` 防止监听器泄漏。它比 DOM 观察者更像有事件通道的发布订阅。

### 单例

> DI Container、Provider/Tool Registry、Redis Client、LLM Queue、缓存和熔断器使用延迟初始化单例，复用昂贵资源并统一生命周期。它只是单 Node 进程内单例；Serverless 扩容后每个实例各有一份，跨实例状态仍需 Redis/数据库。单例中不能保存请求级用户数据。

### 适配器

```text
Ollama JSON Lines ─┐
                    ├→ StreamParser → ParsedChunk
OpenAI SSE ─────────┘
旧 ToolCall → Legacy Adapter → 标准 ToolExecutor
```

> 适配器把不同模型流和旧工具格式转成内部统一协议，上层无需到处判断供应商。适配器解决接口不兼容；策略解决同一目标的多种实现。

## 十三、实现边界

- Query 改写尚未进入主链路，避免增加延迟或改坏专有名词。
- 当前没有 Cross-Encoder，使用 BM25、Embedding、RRF 和显式效用重排。
- NotebookLM 是闭源产品；LLM-Wiki 是调研方向，不说成已上线。
- 小规模 Fixture 指标不能包装成线上 A/B 结果。

## 十四、最后 20 秒

> 召回链路使用结构化索引、BM25 与向量混合召回、RRF、原文摘要统一竞争、动态 Top K、摘要回源、邻居扩展和 Token Budget 装箱；原始消息始终是事实源。设计上用发布订阅解耦跨 Tab 通信，用单例管理共享资源，用适配器统一模型和工具协议。
