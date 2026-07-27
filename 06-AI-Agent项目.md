# AI Agent 兴趣教练平台小抄

## 面试后最终口径：Host 调度闭环（2 分钟）

> 这个项目初版是固定执行 `Planner -> Critic -> Host`。Host 虽然会返回 `next_agents` 和约束，但编排器下一轮没有真正按它调度；另外初版把 Planner 和 Critic 的语义相似度当作主要收敛条件，相似只能说明“说得像”，不能证明方案正确。这是我复盘后重点补齐的两个闭环。
>
> 现在第一轮仍固定让 Planner 产出结构化方案、Critic 给出评审，先建立可比较的基线。Critic 不只输出自然语言意见，还输出 `feasible / realistic / complete` 三项检查以及带严重级别的风险。Host 本身是确定性的 TypeScript 策略，不让模型自由决定流程：它综合未解决高风险、结构覆盖率、风险数量变化、覆盖率变化和轮次预算，产生 `revise / challenge / verify / finalize / terminate` 五类动作。
>
> 例如还有高风险时，Host 返回 `revise`，下一轮只调度 Planner，并通过 `must_address` 把具体风险带过去；Planner 修订后不会直接结束，而是返回 `verify`，下一轮只让 Critic 验证。只有可行性、现实性、完整性全部通过、没有高风险、覆盖率达到阈值且满足最小轮次，才进入 Reporter。达到最大轮次也不会伪装成成功，而是 `terminate`，让 Reporter 明确披露未解决风险。
>
> `next_agents` 现在已经是编排器的真实执行指令，不再只是展示字段；Host 的共识、风险、覆盖率和停滞趋势也会存进 `multi_agent_sessions`，断点恢复后继续使用。Embedding 相似度只用于发现首轮过快一致或重复表达：首轮结构已经合格但相似度过高，会额外让 Critic 做压力测试；它不再直接代表方案质量。
>
> 这套设计的核心取舍是“模型负责生成和评审，代码负责流程控制”。它比一个大 Prompt 更容易测试和定位失败，但会增加调用次数和延迟，因此简单问答不进入多 Agent。当前是个人项目规模，最大轮次默认 5；如果做企业级长任务，我会再补持久化 run、队列、工具审计和更完整的离线回归集，这些不能说成当前已经全部上线。

### 第 7 题：React 状态管理与高频渲染控制

我当时这个设计主要解决的不是网络接收问题，而是 React 在流式输出场景下的高频渲染问题。大模型返回 chunk 的频率可能很高，如果每来一个 chunk 我就直接 setState，把文本追加到 message 里，React 会被迫做很多次 render、diff 和 commit，最后浏览器还要跟着做 DOM 更新、样式计算和绘制。用户看到的可能不是“更实时”，反而是页面开始卡、滚动不顺，甚至输入框打字都有延迟。

我在项目里把这个链路拆成了三层。第一层是 TransformStream，它主要负责把服务端 SSE 或流式响应里的 chunk 解析成前端可消费的事件，比如文本增量、工具调用状态、结束事件、错误事件。第二层是 buffer 队列，我不会每解析到一小段文本就立刻更新 React state，而是先把这些增量放进队列里。第三层是 requestAnimationFrame，我会在下一帧统一把 buffer 里的内容取出来合并，比如把多个 token 拼成一段文本，然后只 setState 一次。这样本质上是把“按 chunk 更新”变成“按浏览器帧更新”。

如果不用这个方案，最直接的问题就是 setState 次数太多。虽然 React 18 有一定的 batching，但流式数据很多时候来自异步 reader、stream 回调或者自定义事件链路，实际还是可能形成非常密集的更新。每次更新都会让消息组件重新渲染，如果内容还要做 Markdown 渲染，标题、列表、代码块这些结构也会被反复解析；如果下面还有长会话列表、自动滚动、代码高亮，那成本会继续放大。最后就容易出现 long task，主线程被占住，用户滚动消息列表时会掉帧，输入框响应变慢，页面看起来一卡一卡的。

我做 rAF 批量刷新还有一个考虑是，它和浏览器绘制节奏更一致。正常一秒 60 帧的话，大概 16ms 一帧，模型就算在这 16ms 内吐了很多小 chunk，我也只需要在这一帧更新一次 UI。这样用户感知上仍然是连续输出的，但 React 的渲染压力会小很多。这里我做的取舍是，不追求每个 token 毫秒级立刻上屏，而是保证整体交互稳定。对用户来说，稳定顺滑的流式输出比每个字符都马上渲染更重要。

当然也不是所有事件都等 rAF。像流结束、报错、工具调用完成这类状态，我会考虑立即 flush 一次，避免状态停留在 loading；普通文本增量则进入 buffer 合并。验证上我会用 Chrome Performance 看直接 setState 和批量刷新的差异，主要看 render 次数、long task、FPS，以及输入和滚动有没有明显延迟。如果直接 setState 时主线程上出现很多密集的小 render，而 rAF 后变成按帧合并的更新，就说明这个方案确实把高频 chunk 对 React 的压力降下来了。


## 90 秒项目介绍

> 这是一个基于 React 18、TypeScript、Modern.js BFF、Hono、MongoDB 和 Redis 的 AI Agent 兴趣教练平台。它支持学习规划、任务拆解和多轮对话。我主要想解决三类工程问题：复杂任务如何稳定编排、模型输出如何实时平滑展示、长会话如何控制前端渲染和上下文成本。
>
> 编排上把任务拆成 Planner、Critic、Host、Reporter 四个职责。Critic 输出结构化有效性和风险，确定性的 Host 策略再决定下一轮真正执行哪个 Agent，而不是始终跑固定链路。传输上由 BFF 调模型并通过 TransformStream 返回 SSE，前端用 TextDecoder 增量解码，把 chunk 进入 buffer，再由 requestAnimationFrame 按帧 flush，避免每个 token 都触发 React 和 Markdown 重算。
>
> 长会话前端用虚拟列表控制 DOM 数量，LocalStorage 只缓存最近 10 轮用于秒开；MongoDB `messages` 保存完整原文。服务端异步生成切片、Embedding 和带来源消息 ID 的长期摘要，构建上下文时再按 token 预算装入最近两轮、摘要和相关记忆。BFF 还负责密钥保护、上下文拼接、鉴权、限流和流式转发。项目重点不是接一个聊天接口，而是把模型的不确定输出放进可校验、可降级的工程链路。

## 一次“生成学习计划”的完整链路

1. 前端提交目标与约束；BFF 校验身份和会话归属，加载最近窗口与混合召回的历史记忆。
2. 第一轮 Planner 输出结构化方案，Critic 输出有效性检查、分级风险和修改建议。
3. Host 按确定性策略决策：有高风险走 `revise`，Planner 修订后走 `verify`，首轮过快一致走 `challenge`。
4. 编排器把 `next_agents` 当作下一轮真实指令：可能只执行 Planner，也可能只执行 Critic，然后 Host 再判断。
5. 满足结构化终止条件走 `finalize`；达到最大轮次走 `terminate`，都由 Reporter 生成最终结果，但后者必须披露剩余风险。
6. 每轮 history、`next_agents` 和 Host 趋势状态写入 `multi_agent_sessions`，支持断点恢复；生成过程通过 SSE 返回。
7. 最终消息同步写入 `messages`；后台异步派生 `memory_items` 和 `memory_summaries`。索引或摘要失败不阻断用户主链路，也不会删除原始消息。

分层职责一句话：

- 前端：会话 UI、流式展示、取消、错误态和卡片渲染。
- BFF：密钥保护、鉴权、限流、上下文组装、编排和流式转发。
- Agent：Planner 规划、Critic 校验、Reporter 表达；Host 用代码策略控制状态迁移。
- 存储：LocalStorage 是最近 10 轮的秒开缓存；`messages` 是原始消息事实源；`memory_items` 和 `memory_summaries` 是可重建派生层；`conversation_token_states` 是 token 双账本；`multi_agent_sessions` 是带 TTL 的短期执行检查点。
- 当前边界：没有把独立 run 表、完整工具审计和任务队列说成已实现。
- 企业化改造：补持久化 run、Outbox/队列、角色级工具权限、审计和基于 runId 的进度订阅。

## SSE 为什么不用 WebSocket

- 场景是服务端持续生成、客户端单向消费，不需要持续双向通信。
- SSE 基于 HTTP，接入网关、鉴权和 BFF 更直接；事件格式和断线恢复更简单。
- WebSocket 更适合高频双向、低延迟互动。
- 注意：原生 EventSource 只支持 GET 且难自定义请求头；需要 POST 或自定义协议时可用 `fetch + ReadableStream` 解析 SSE 格式。

## 流式链路

```text
模型字节流 -> BFF TransformStream -> fetch/ReadableStream
-> TextDecoder(stream: true) -> buffer -> RAF flush
-> Markdown 中间态容错 -> React 渲染
```

- TextDecoder 的流式模式避免 UTF-8 多字节字符被 chunk 拆开造成乱码。
- RAF 合并同一帧内多个 chunk，降低 setState、Markdown parse、layout/paint 频率。
- 页面切换或重新提问时用 AbortController 取消旧流，防止旧响应覆盖新会话。

### 为什么 buffer + RAF 有用

> 它不是让模型或网络返回更快，而是把高频小 chunk 合并成按帧的 UI 更新。若每个 chunk 都 `setState`，消息组件会频繁 render，Markdown 会反复解析，代码高亮、自动滚动和长列表布局也会重复执行，最终产生 Long Task、滚动掉帧和输入延迟。buffer 先积累文本，同一帧只 flush 一次；RAF 回调只做取 buffer 和轻量 state 更新，重计算仍要缓存、分片或 Worker。

验证指标：

- React Profiler：消息组件 commit 次数和单次/总耗时。
- Performance：Long Task、主线程占用和滚动时帧稳定性。
- 交互：流式输出时输入框响应、滚动和取消是否正常。
- 体验边界：低速 token 可立即刷或缩短等待，避免为了批量而增加明显首字延迟。

## Markdown 只返回一半怎么办

- `rawContent` 保存真实模型输出，不修改、不入临时补全。
- `renderContent` 根据状态机对未闭合代码块、链接等高置信结构做临时补全。
- 不对语义不确定的内容强修；解析失败时降级纯文本。
- 消息完成后用最终原文重新解析，临时补全自然消失。

## 虚拟列表为什么不是分页

- 对话是连续上下文，产品上需要顺滑回看和定位，不适合传统分页跳转。
- 虚拟列表解决 DOM 数量；懒加载解决资源体积；两者不是同一问题。
- 风险：动态行高、图片加载、代码块展开导致测量变化；需稳定 key、尺寸重测、滚动锚点和 overscan。
- 不说“自研虚拟列表”；简历使用现成库，重点讲选型和边界处理。

## 多 Agent 为什么不用一个大 Prompt

- 复杂任务把规划、校验、流程控制和最终表达混在一个上下文里，容易职责冲突和推理漂移。
- 拆角色后可单独验证输入输出、失败重试和终止条件。
- 代价是 token、延迟和调度复杂度；简单任务仍应单模型完成，不为 Agent 而 Agent。

## Host 为什么用代码策略，不让模型直接决定

- 调度、最大轮次和终止属于控制面，结果必须可复现、可测试、可设预算。
- 模型负责 Planner 的生成和 Critic 的语义判断；Host 读取结构化字段后执行明确状态迁移。
- `next_agents` 是执行指令，`must_address` 是下一 Agent 的输入约束，两者都必须进入下一轮，才叫闭环。
- 相似度只表示两段文本接近，不能证明风险已关闭；主要质量信号是有效性检查、未解决高风险和覆盖率。
- 当前默认最少 2 轮、最多 5 轮；阈值是可配置策略，后续应通过固定回归集调参，不宣称它们是通用最优值。

### 一句话状态机

```text
高风险 -> revise(Planner)
Planner刚修订 -> verify(Critic)
首轮过快一致 -> challenge(Critic)
结构检查全部通过 -> finalize(Reporter)
达到轮次预算 -> terminate(Reporter披露风险)
```

### Host 专项验证

- 高相似但有高风险，不能结束。
- `next_agents=['critic']` 时，下一轮不能再执行 Planner。
- Planner 单独修订后必须由 Critic 验证。
- 达到最大轮次时，Reporter 能拿到终止原因和未解决风险。
- Embedding 失败时，文本相似度降级仍可完成决策。

## 为什么做 BFF

- 保护 API Key、Prompt、模型路由和工具服务。
- 统一鉴权、限流、日志、错误映射、上下文拼接。
- 前端不感知模型供应商切换。
- 代价是多一跳和服务运维，因此要做超时、取消、流式透传和可观测性。

## OIDC + PKCE

- OIDC 在 OAuth 2.0 授权基础上提供标准身份信息。
- SPA 无法安全保存 client secret；PKCE 用 verifier/challenge 防止授权码被截获后直接兑换 token。
- BFF 模式可把 token 保存在服务端，浏览器只持 HttpOnly/Secure/SameSite Session Cookie；状态改变请求仍需 CSRF 防护。

## Redis 限流

- 多实例下单机内存计数无法共享，Redis 可统一计数。
- `INCR + EXPIRE` 要考虑原子性，可用 Lua；固定窗口有边界突刺，可按场景选滑动窗口或令牌桶。
- AI 接口限流既防刷，也控制模型成本。

## 核心技术选型对比

### SSE、WebSocket、轮询与 fetch 流

| 方案 | 优势 | 代价 | 适用判断 |
| --- | --- | --- | --- |
| 短轮询 | 实现最简单，普通 HTTP 基建即可 | 空请求多、实时性受间隔限制 | 低频状态查询 |
| 长轮询 | 比短轮询实时，兼容传统 HTTP | 连接管理复杂，仍需反复建请求 | 旧环境或过渡方案 |
| WebSocket | 全双工、低延迟、高频双向 | 网关/鉴权/重连/心跳更复杂 | 协作、游戏、实时控制 |
| 原生 EventSource | SSE 协议、自动重连简单 | 通常 GET，头部和请求体受限 | 简单服务端单向推送 |
| fetch + ReadableStream | 可 POST、自定义头、可 Abort | 需自己解析事件和处理重连 | LLM 对话流式输出 |

本项目选择 fetch 流式读取 SSE 格式：业务是单向 token/状态推送，同时需要 POST Prompt、鉴权头和主动取消。

### 单模型、固定 Workflow 与多 Agent

- **单模型**：延迟和成本最低，适合目标明确、一步可完成的任务。
- **固定 Workflow**：步骤已知，用代码状态机编排最稳定，例如固定的“检索 -> 生成 -> 校验”。
- **Agent**：步骤或工具选择需要模型动态决策，灵活但不可预测性更高。
- **多 Agent**：职责需要隔离、交叉校验时使用；代价是 token、轮次和调试复杂度。

面试时强调：能用固定 Workflow 解决就不盲目上多 Agent。本项目的 Planner/Critic 等设计用于复杂规划实验，简单问答走短路径。

### 全量历史、摘要与检索记忆

| 方案 | 优点 | 问题 |
| --- | --- | --- |
| 全量历史拼接 | 实现简单，不丢显式上下文 | token、延迟持续增长，噪声增加 |
| 滚动摘要 | 上下文稳定、成本低 | 摘要可能丢细节；如果覆盖原文，错误会累积且不可恢复 |
| 最近窗口 + 派生摘要 + 历史检索 | 近期连续、旧事实可召回、摘要可重建 | 多一套异步任务、状态机和一致性处理 |

本项目采用“原文事实源 + 两类派生记忆 + token 预算装箱”：

1. 浏览器 LocalStorage 只保留最近 10 轮完整消息，用于页面秒开；离线产生但尚未同步的消息是保护性例外。它不是长期事实源。
2. MongoDB `messages` 保存所有原始消息。记忆压缩只新增派生数据，不因“已经总结”而删除消息；用户主动删除会话是另一条数据生命周期。
3. 新消息落库后，后台按 1200 字符、120 字符重叠切片，写入 `memory_items` 并生成 Embedding。Embedding 失败仍保留全文检索数据，不阻断聊天。
4. 新增原文达到阈值或上下文压力升高时，后台摘要任务读取尚未总结的旧消息，提取目标、偏好和约束，写入 `memory_summaries`。每条摘要都带 `sourceMessageIds`、起止消息 ID、原文/摘要 token 数和模型版本。
5. 查询时做 BM25/关键词与向量召回，用 RRF 合并名次，再按相关性、时间衰减和重要性重排。
6. 构建上下文前先算输入预算；最近两轮原文是强制项，摘要和长期记忆按“相关性价值 / token 成本”装入，预算用完立即停止。

LocalStorage 常被说成“约 5MB”，但这只是不同浏览器、不同 origin 下常见的配额量级，不是可靠的业务协议。不能先把它塞满再考虑迁移；本项目直接按消息轮数限制为 10 轮，容量只作为异常保护。

### token 到底怎么算

不能把每轮 `total_tokens` 累加后直接和模型上下文窗口比较。历史消息会在多轮请求中反复发送，累计值会重复计数，它表示账单，不表示下一轮上下文大小。

本项目使用三种口径：

- **调用前预算**：没有真实 usage，只能对 system prompt、当前消息、工具 Schema 和候选历史做保守估算。
- **调用后校准**：OpenAI 兼容接口请求 `stream_options.include_usage=true`，读取流末尾的 `prompt_tokens / completion_tokens / total_tokens`；Ollama 映射 `prompt_eval_count / eval_count`。
- **新增原文量**：只估算本轮新写入的 user + assistant 原文，累加到 `unsummarizedTokens`，不把重复发送的历史算进去。

输入预算公式：

```text
inputBudget = contextWindow
              - outputReserve
              - ceil(contextWindow × safetyMarginRatio)
```

然后：

```text
historyBudget = inputBudget
                - systemPromptTokens
                - currentMessageTokens
                - toolSchemaTokens
```

状态结构：

```ts
{
  conversationId: "conv-123",

  // 上一轮实际输入；供应商不返回 usage 时才用估算值
  lastInputTokens: 7200,

  // 所有模型调用的累计账单，用于成本统计，不用于判断当前上下文
  lifetimeBillableTokens: 32500,

  // 上次长期摘要之后新增长的原始 user + assistant 内容
  unsummarizedTokens: 4600,

  summarizedThroughMessageId: "msg-88",
  compressionStatus: "idle" // idle | running | failed
}
```

当前默认在 `unsummarizedTokens >= 4000`，或上一轮输入达到可用输入预算的 65% 时尝试启动后台摘要；先用 MongoDB 原子更新抢占任务，避免同一会话重复压缩。阈值是工程初值，应该根据摘要质量、P95 延迟和 token 成本调参。

### 摘要任务没跑完，用户又提问

用户请求不等待后台摘要，按状态降级：

```text
已有可用摘要
→ 最近 2 轮原文 + 摘要 + 相关长期记忆

compressionStatus = running
→ 最近 2 轮原文 + 已有摘要/记忆
→ 按相关性/token 成本删除低价值片段
→ 压力仍高且没有可用摘要时，临时同步压缩；失败则直接截断低价值项

compressionStatus = failed
→ 从 MongoDB messages / memory_items 继续做原文全文或混合检索
```

后台任务超过 10 分钟仍为 `running` 时允许其他 worker 重新抢占；失败会记录错误和重试时间。摘要始终只是加速与压缩层，绝不能成为唯一事实源。

### 两个候选方案怎么“拉踩”

| 候选 | 方案 | 优点 | 被追问时的短板 |
| --- | --- | --- | --- |
| A：滚动摘要替换历史 | LocalStorage 留 5～10 轮；达到累计 token 阈值后，异步 Agent 分块总结；以后主要发送滚动摘要 | 代码和查询链路简单；上下文长度稳定；数据库体积和推理成本低 | 累计 `total_tokens` 重复计算历史，触发口径不准；摘要一旦漏掉否定、数字或约束会代代累积；若原文被覆盖则无法重建与审计；任务未完成时缺少可靠降级 |
| B：事实源 + 派生记忆 + 双账本（本项目） | LocalStorage 10 轮秒开；Mongo 保存全部原文；切片/Embedding/结构化摘要均为派生层；调用前预算、调用后 usage 校准；按相关性/token 装箱 | 可重建、可追溯；摘要失败仍能查原文；区分成本账本与上下文压力；能明确回答并发摘要和失败问题 | Mongo 与派生索引占用更高；要处理最终一致性、任务抢占、重试、索引版本和质量评测；实现复杂度明显高 |

面试结论：A 适合低风险、短生命周期、允许信息损失的聊天；B 适合目标、偏好、约束会影响后续行为的 Agent。不是说 A 错，而是 B 用额外存储和工程复杂度换取可恢复性与可审计性。

### 原文和摘要能不能共用一个计分公式

可以，项目现在默认这样实现；统一的是“特征归一化后的 utility 公式”，不能把 BM25 分数和摘要关键词重合率直接相加。

```text
utility = 相关性
          + 来源保真度
          + 信息密度
          + 来源覆盖率
          - 重复惩罚

packingScore = utility / tokenCost^0.7
```

- 原文和摘要共同参与 BM25 + Embedding + RRF，得到同量纲的相关性。
- 原文的保真度更高；摘要的信息密度和覆盖率更高。这些是同一公式下的特征值，不是两套公式。
- 日期、数字、错误码、版本、路径和“用户原话”查询提高原文保真权重。
- 如果摘要和原文的 `sourceMessageIds` 重合，后进入的候选扣重复分。
- 必须先做相关性门槛，不能让一个零相关短片段靠“原文可靠、token 少”入选。

实际链路：

```text
一次 query Embedding
→ memory_items 原文切片 + memory_summaries 摘要进入共同候选池
→ BM25 / Embedding 各自排序，RRF 融合
→ 同一 utility 公式（两种候选取不同的特征值）
→ 先扣最近 2 轮强制原文的 token
→ 按 utility / tokenCost^0.7 装箱
→ sourceMessageIds 重叠则扣分，预算用完停止
```

新摘要会保存自己的 Embedding、重要性和 `sourceMessageIds`；旧摘要没有 Embedding 时仍可走 BM25。默认使用统一链路，`MEMORY_UNIFIED_SCORING=false` 可退回原文/摘要分开计分的旧链路，所以它既是故障开关，也能作为线上 A/B 对照组。

项目中的 4 组固定离线样本结果：

| 指标 | 分开计分基线 | 统一计分实验 |
| --- | ---: | ---: |
| 证据召回 | 62.5% | 100% |
| 精确原文召回 | 0% | 100% |
| 重复 token 比例 | 17.9% | 0% |
| 平均预算利用率 | 57.1% | 77.1% |

注意口径：代码已经切到统一链路，但这组数字只来自精确日期、语义偏好、来源重复、精确错误码四类人工回归样本，只能证明公式和实现可行，不能说已经取得生产收益。正式扩大流量需要真实脱敏对话、人工 Gold Evidence 标签，并与环境变量控制的旧链路对比。

为什么用 RRF：BM25 分数和余弦相似度不在同一量纲，直接相加难校准；RRF 只利用各路排名，先得到稳定融合结果，再叠加时间和重要性。

生产与本地的差别：配置 Atlas Search/Vector Search 索引时在数据库侧召回；没有索引时会在最多 500 条候选内做应用层 BM25 和余弦计算，方便本地运行，但 QPS 上升后应切数据库原生索引。项目提供建索引和历史回填脚本，但不能说已经在生产数据执行。

### 为什么拆 `messages`、`memory_items` 和 `memory_summaries`

- `messages` 保证原文、顺序和会话展示；`memory_items` 面向切片、Embedding 版本和检索排序；`memory_summaries` 面向压缩和目标/偏好/约束提取。
- 一条长消息可能对应多个记忆块，直接把检索字段塞回消息会让模型和索引演进互相影响。
- 派生索引可以按 `messageId + chunkIndex` 幂等重建；摘要用来源 ID 的哈希保证幂等。Embedding 或摘要模型升级时只重建派生层，不改事实源。
- 代价是最终一致性。当前有后台执行、失败状态和延迟重抢；生产化还应把进程内后台任务升级为持久队列/Outbox，并补积压、重试和摘要质量监控。

### 前端直连、API 网关与 BFF

- 前端直连模型：链路短，但 API Key、Prompt、模型路由和工具权限暴露，不适合生产。
- 通用网关：擅长鉴权、限流和路由，但不承担页面定制的数据聚合与上下文编排。
- BFF：按前端场景裁剪数据、拼接上下文、流式透传；代价是服务维护和额外一跳。

最终用 BFF 管理 AI 特有的 Prompt/工具/模型编排，通用安全和流量能力仍可下沉网关。

### 简单 JWT、OIDC + PKCE 与 BFF Session

- 自签 JWT 适合单系统内部鉴权，但登录、授权、登出、密钥轮换需要自行定义。
- OIDC 适合第三方/统一身份体系，标准完整但流程和运维更复杂。
- PKCE 解决公开客户端无法安全保存 secret 时的授权码截获风险。
- BFF Session 把 token 留在服务端，浏览器仅持 HttpOnly Cookie；代价是 Session 存储和 CSRF 防护。

本项目是为了学习标准身份链路而实现轻量 IdP；面试时不要暗示它达到成熟身份平台的生产安全水平。

### 固定窗口、滑动窗口与令牌桶

- 固定窗口：简单，但窗口边界可能瞬间放过双倍流量。
- 滑动窗口：更平滑，Redis 操作和存储成本更高。
- 令牌桶：允许可控突发，适合有平均速率与峰值需求的接口。
- AI 接口还要按用户、模型和 token 成本分层限额，不能只按请求次数。

## 取舍题收尾模板

> 我们先按任务复杂度走最短路径：简单任务单模型，固定步骤用代码 Workflow，确实需要动态工具决策才进入 Agent。传输层因为是单向生成且需要 POST/取消，所以选择 fetch 流而不是 WebSocket；安全和编排集中在 BFF。每引入一层灵活性，都同时增加超时、轮次、成本预算和可观测日志，避免系统只“能跑”但不可控。

## 最大风险与复盘

> 最大风险不是模型答错一次，而是错误结果进入后续工具链。我的兜底是 Schema、工具参数校验、超时/重试、最大轮次、终止条件和可观测日志。简单任务不进入多 Agent，复杂任务才支付额外成本。
