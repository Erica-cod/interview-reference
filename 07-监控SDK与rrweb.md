# 前端监控 SDK 与 rrweb 小抄

## 本次 AI 模拟面试：详细示例回答（原文）

### 第 5 题：前端监控 SDK 的采集、上报与隐私设计

我按一个具体的 TypeError 来讲。假设 React 页面里某个组件渲染时访问了 undefined.xxx，最后抛出 TypeError。

首先在 SDK 初始化的时候，我会先生成一个 sessionId。这个 sessionId 是页面会话维度的，一般会存在 sessionStorage 里，比如一次 tab 会话内保持不变。同时这个时候 rrweb 就开始 record，我会记录一个 replayStartTime，也就是这段录制开始的时间戳。后面所有 rrweb event 会按 sessionId 分片上传，比如每 5 秒或达到一定大小上传一批。

当 TypeError 真正发生时，捕获入口可能是 React ErrorBoundary，也可能是 window.onerror 或 unhandledrejection。进入 SDK 的错误处理函数后，我会第一时间生成 eventId，这个 eventId 是这一次错误事件的唯一 ID，比如用 nanoid 或 uuid。这个时机要尽量早，因为后面的错误详情、breadcrumb、网络请求、rrweb 回放片段都要靠它关联。

然后我会记录错误发生时的 timestamp，并计算 replay offset。比如 rrweb 是 10:00:00 开始录制，错误是 10:00:23.500 发生，那 offset 就是 23500ms。这个 offset 我不会单独拿来做全局查询，它主要是配合 sessionId 使用：sessionId 告诉我是哪一段用户会话，offset 告诉我在这段会话里的哪个时间点出错。

fingerprint 我会在错误信息规范化之后生成。客户端可以先生成一个初版 fingerprint，比如用 error name、归一化后的 message、当前 route、release 版本、stack 里的前几帧函数名来 hash。这里不会直接把完整行列号都放进去，因为构建后行列号可能变化很频繁，容易把同一个问题打散。服务端收到后会再结合 SourceMap 解析，把压缩后的 stack 还原成源文件、函数名、行列号，然后生成或修正最终 fingerprint。这样同一个组件、同一类 TypeError 会聚合到一起，而不是每个用户都变成一条新错误。

所以上报的错误 payload 里大概会有这些字段：eventId、sessionId、fingerprint、timestamp、replayOffset、route、release、errorName、message、stack、最近的 breadcrumbs，还有一些环境信息，比如浏览器、系统、页面 URL 模板。这里 eventId 用来查单次错误，fingerprint 用来聚合一类错误，sessionId 加 offset 用来定位回放。

服务端聚合时，我会先按 fingerprint 做错误聚合，统计发生次数、影响用户数、版本分布和最近发生时间。点进某一次具体错误时，其实查的是 eventId 对应的 error record。这个 record 里有 sessionId 和 replayOffset。

最后看 rrweb 回放时，我的查询链路是这样的：先通过 eventId 找到错误详情，拿到 sessionId = xxx 和 replayOffset = 23500ms；然后去 replay 存储里查这个 sessionId 下的 rrweb 分片，取 offset 前后的一段，比如前 10 秒到后 5 秒，也就是 13500ms 到 28500ms 之间的 events；前端播放器加载这段 events 后，直接 seek 到 23500ms 附近，并在时间轴上标一个 error marker。这样我就能看到用户在错误发生前点击了什么、输入了什么、路由有没有跳转、接口有没有失败，以及错误发生后页面是不是白屏或交互失效。

这里我觉得关键点是这几个字段的职责不能混：eventId 解决“这一条错误是谁”，fingerprint 解决“它属于哪一类问题”，sessionId 解决“它发生在哪次用户会话”，replay offset 解决“在回放里的哪个时间点”。如果这几个字段在采集时就打通，后面定位时就不是靠猜，而是可以从错误列表一路跳到具体回放现场。


## 高频追问：rrweb 如何处理敏感信息

### 90 秒口述版

> 我处理 rrweb 脱敏的原则是：敏感原文尽量不要离开浏览器，不能等上传到服务端以后再打码。整体分为采集端默认脱敏、业务区域分级、上报前二次校验和服务端权限治理四层。
>
> 第一层是输入数据。密码框 rrweb 默认会脱敏；项目里可以开启 `maskAllInputs`，让普通 input、textarea 的内容也默认不记录明文。如果只需要保护特定输入类型，可以使用 `maskInputOptions`；有定制规则时再通过 `maskInputFn` 处理，例如只保留字符长度，不保留真实内容。
>
> 第二层是 DOM 区域分级。像手机号、邮箱、用户资料这类需要保留页面结构但不能保留文本的区域，用 `rr-mask` 或 `maskTextSelector`，回放时保留布局，文本替换为掩码。像支付信息、身份证、富文本编辑器这类整个区域都不应该采集的内容，用 `rr-block` 或 `blockSelector`，rrweb 只留下同尺寸占位，不序列化内部节点。`rr-ignore` 主要是不记录输入事件，它不等于完整脱敏，所以不能用它保护真正敏感的数据。
>
> 第三层是监控 SDK 自己采集的数据。URL 会去掉 query 和 hash，或者只保留路由模板；接口 Breadcrumb 默认只记录 method、状态码、耗时和接口模板，不采集 Cookie、Authorization、请求体和响应体。业务确实需要参数时采用字段白名单，而不是依靠黑名单排除。
>
> 上报前再做一次 schema 白名单和敏感模式检查。如果发现 token、手机号等疑似明文，宁可丢弃对应字段、事件或分片，也不把原始数据传出去。服务端再配合 HTTPS、存储期限、回放访问权限和审计日志。
>
> 验证时我会在测试页面主动输入密码、手机号和 token，同时检查序列化后的 rrweb event、网络 payload 和最终回放，确认三个位置都不存在明文。这样脱敏不是“配置写上了”，而是有测试闭环。

官方配置参考：[rrweb Guide - Privacy](https://github.com/rrweb-io/rrweb/blob/master/guide.md#privacy)。

### `block`、`mask`、`ignore` 的区别

| 方式 | 行为 | 适用场景 |
| --- | --- | --- |
| `rr-block` / `blockSelector` | 不记录整个 DOM 子树，回放中只保留同尺寸占位 | 身份证、支付信息、富文本编辑器等高敏区域 |
| `rr-mask` / `maskTextSelector` | 保留 DOM 结构，但将文本替换为掩码 | 手机号、邮箱、用户资料等需要保留页面上下文的区域 |
| `rr-ignore` / `ignoreSelector` | 主要忽略输入事件 | 低价值交互；不能当成完整的内容脱敏 |
| `maskAllInputs` / `maskInputOptions` | 对全部或指定类型的输入值脱敏 | input、textarea、select 等表单内容 |

### 为什么不能只在服务端脱敏

> 因为原始敏感数据一旦离开浏览器，就可能进入网关日志、消息队列、失败重试和临时存储。服务端脱敏只能作为第二道防线，第一道防线必须放在 rrweb 序列化和 SDK 采集阶段。

### 脱敏验证清单

1. 在测试页面输入密码、手机号、邮箱、身份证号和模拟 token。
2. 检查 `emit` 得到的原始 rrweb event，确认没有敏感明文。
3. 检查浏览器 Network 中的上报 payload，确认 URL、请求头和业务参数已经过滤。
4. 打开最终回放，确认被 mask 的区域仍能定位问题，被 block 的区域只显示占位。
5. 测试动态插入的 DOM、弹窗、下拉框和路由切换，避免只保护首个 FullSnapshot、漏掉后续 IncrementalSnapshot。
6. 为脱敏规则增加自动化用例；新增敏感字段时需要同步更新标记和测试。

### 真实性边界

- 如果项目已经实际配置 `maskAllInputs`、`rr-mask` 或 `rr-block`，可以用“我实现了”。
- 如果目前只实现了基础输入脱敏，服务端审计、敏感模式扫描和字段白名单应说成“生产化时会补充的第二道防线”。
- 不要把“回放页面看不到明文”当作唯一证明；必须确认原始 event 和上报 payload 里也没有明文。

## 先说真正亮点

> rrweb 本身不是项目亮点，亮点是把“错误事件”和“用户现场”做成可查询的关联链路：错误发生时生成 `eventId`，读取当前 `sessionId`，记录 `timestamp/replayOffset`，规范化错误后生成 `fingerprint`；服务端按 fingerprint 聚合 issue、按 SourceMap 还原源码，再用 `eventId -> sessionId + replayOffset` 找到错误前后的 rrweb 分片，播放器直接 seek 到错误点并标 marker。

一句话记忆：`eventId` 找单次错误，`fingerprint` 聚合同类问题，`sessionId` 找会话，`replayOffset` 找会话内时间点。

## 90 秒项目介绍

> 这是一个面向 React/Vue 应用的轻量监控 SDK，覆盖错误、性能、用户行为、数据上报和 rrweb 会话回放。架构上把采集器、上报器和高级能力拆成模块，用 EventBus 解耦，业务通过 init 和配置按需开启。
>
> 错误侧统一采集 JS 运行时错误、未处理 Promise、资源加载失败和框架错误；性能侧用 Performance API/Web Vitals 记录 LCP、CLS、INP、TTFB、Long Task 和资源耗时；行为侧记录有限数量的点击、路由和请求 Breadcrumb。一次错误通过 `eventId/sessionId/fingerprint/replayOffset` 与聚合 issue、SourceMap 和 rrweb 分片关联，排查人员可以从错误详情直接跳到错误前后的用户现场，而不是单独翻日志和整段录屏。
>
> 上报侧用内存队列做批量与定时 flush，致命错误优先发送，普通行为和性能数据可采样；重试有次数与退避预算，队列满时优先丢低价值行为。SourceMap 在服务端按 release 解析，rrweb 只保存 DOM 快照和增量事件，并对输入和敏感 DOM 脱敏。SDK 的首要原则是不影响业务，所以所有采集都要错误隔离、可关闭、有资源上限，并避免拦截自身上报请求形成循环。

## 必背：一个 TypeError 如何走到 rrweb 回放

> 假设 React 页面在 render 中访问 `undefined.xxx`，抛出 TypeError。SDK 初始化时已经创建页面会话级 `sessionId`，启动 rrweb record，并保存 `replayStartTime`；录屏事件按 session 分片，达到时间或字节阈值就上传。
>
> 对这个 render TypeError，入口通常是 Error Boundary 或 `window.onerror`；如果是 Promise 拒绝，则由 `unhandledrejection` 捕获。进入统一 normalize 流程的第一时间生成本次错误唯一的 `eventId`，记录错误 `timestamp`，并计算 `replayOffset = timestamp - replayStartTime`。随后归一化 name、message、stack、route、release 和 component stack，生成客户端初版 fingerprint；服务端用与 release 对应的 SourceMap 还原源码位置后，再生成或修正最终 fingerprint，避免构建后的行列波动把同类错误打散。
>
> 错误 payload 至少带 `eventId、sessionId、fingerprint、timestamp、replayOffset、route、release、errorName、message、stack、breadcrumbs` 和脱敏后的环境信息。服务端用 fingerprint 聚合发生次数、影响用户、版本分布；点进某一次实例时用 eventId 查 error record，拿到 sessionId 和 replayOffset。
>
> 假设 offset 是 23500ms，回放服务就查该 session 下覆盖 13500ms 到 28500ms 的分片，也就是错误前 10 秒、后 5 秒。播放器加载后 seek 到 23500ms，并在时间轴标出 error marker。这样能直接看到报错前的点击、路由、接口失败和 DOM 变化，以及报错后是否白屏或交互失效。

这里的“前 10 秒/后 5 秒、每 5 秒分片”只是易讲的设计示例；若项目真实配置不同，必须改成真实值。若当前只做了本地 Demo，也要说“这是我实现/设计的查询链路”，不要说成大规模线上系统。

### 四个标识的生成时机与职责

| 字段 | 何时/哪里生成 | 解决的问题 |
| --- | --- | --- |
| `sessionId` | SDK 初始化时，页面会话级；可保存在 sessionStorage | 这是哪一次用户会话 |
| `eventId` | 单次错误进入统一处理函数时立即生成 | 这条具体错误是谁，串联错误详情与附属数据 |
| `fingerprint` | 错误规范化后生成初版，服务端 SourceMap 后可修正 | 哪些实例属于同一类 issue |
| `replayOffset` | 错误时间减录制开始时间，和 sessionId 配合 | 错误发生在该会话的哪个时间点 |

### 从后台点击到现场的查询链

```text
issue(fingerprint)
  -> error instance(eventId)
  -> error record(sessionId, replayOffset)
  -> replay chunks(sessionId, time range)
  -> Replayer.seek(replayOffset) + error marker
```

不要说“eventId 关联所有 rrweb event”。更准确的是：rrweb 分片通常按 `sessionId + chunk/time range` 存储；`eventId` 找到错误记录，再由记录里的 `sessionId + replayOffset` 定位分片。

## 采样、重试、脱敏和性能：不要分散讲

| 约束 | 取舍 |
| --- | --- |
| 采样 | 致命错误、白屏等高价值事件尽量全量；性能与普通行为按 session 稳定采样，保证一次会话上下文完整 |
| 重试 | 只重试网络错误、超时、429/部分 5xx；指数退避 + jitter + 最大次数；4xx 参数错误不盲重试 |
| 队列溢出 | 优先丢鼠标移动、普通行为等低价值事件，保留错误和错误前后的 Breadcrumb/回放窗口 |
| 脱敏 | 输入框、textarea、敏感 DOM 默认 mask；URL 参数、请求头和正文走白名单/模板化，不默认上传响应体 |
| 主线程开销 | 高频鼠标/滚动节流合并，分片压缩和重计算必要时进 Worker，队列/录屏时长/字节数都有上限 |
| 存储与权限 | 高敏页面关闭或强 mask，设置保存时长、访问权限和审计；回放不是“录制整个屏幕视频” |

### 怎么验证这条链真的可用

1. 人工触发一个固定 TypeError，确认 error record 的四个标识齐全。
2. 构建压缩产物后检查 SourceMap 是否还原到正确源文件和函数。
3. 从 eventId 进入详情，确认能命中正确 session 分片并 seek 到错误点。
4. 输入手机号/密码等敏感内容，确认 DOM、Breadcrumb 和请求字段均已 mask/过滤。
5. 模拟断网、429、500 和队列满，检查重试次数、优先级丢弃和幂等分片。
6. 观察 SDK 包体、初始化耗时、Long Task、上报字节数、成功率和回放命中率；没有真实线上数字就只讲测试口径。

## 错误采集矩阵

| 类型 | 入口 | 注意 |
| --- | --- | --- |
| JS 运行时错误 | `window.addEventListener('error')` | 记录 message、stack、URL、行列、release |
| Promise | `unhandledrejection` | reason 可能不是 Error，要归一化 |
| 资源加载失败 | 捕获阶段 `error` 事件 | script/img/link 错误不靠冒泡；不要仅用 transferSize=0 判断 |
| React | Error Boundary | 捕获渲染/生命周期错误，不捕获事件回调和异步错误 |
| Vue | `app.config.errorHandler` | 保留组件与阶段信息 |
| 接口 | 包装 fetch/XHR | 记录耗时、状态和脱敏后的 URL，不默认上传响应体 |

## 防止 SDK 伤害业务

- 每个模块 try/catch 隔离；初始化失败不阻塞主应用。
- 原始 fetch/XHR 引用保留；内部请求用不可冲突标记或上报域名白名单跳过。
- 队列、Breadcrumb、录屏和重试均有上限。
- 特性检测，不支持就降级或关闭。
- 采样、插件开关和远程熔断；SDK 本身也要有错误监控但防递归。

## 上报策略

- 正常批量上报：fetch，可设置内容类型和鉴权。
- 页面卸载：`sendBeacon` 适合小体积、尽力发送，不能设置任意请求头；`fetch(..., { keepalive: true })` 是另一选择。
- Image GET 只适合极小兼容打点，受 URL 长度、隐私和语义限制，不承载录屏大数据。
- 跨域仍需要服务端正确接收并配置策略；不要说 Beacon“完全不受跨域限制”。
- 重试只针对可重试错误，加指数退避和 jitter，并设最大次数；4xx 通常不盲目重试。

## SourceMap 为什么服务端解析

浏览器只上报压缩 stack、release 和文件信息；CI 将 SourceMap 与 release/chunk hash 绑定并上传到受控存储。服务端反解可避免 SourceMap 公开泄漏源码，也便于统一去重和版本隔离。

## Web Vitals

- LCP：主要内容显示速度。
- CLS：非预期布局偏移。
- INP：页面全生命周期的交互响应；FID 已被替代，只在兼容旧口径时说明。
- TTFB：服务器与网络响应起点。
- Long Task：主线程任务超过 50ms，帮助定位交互阻塞。

## rrweb 原理

> rrweb 不是录视频，而是 `FullSnapshot + IncrementalSnapshot + Replayer`。

1. record 开始时序列化完整 DOM，给节点分配唯一 ID，mirror 维护节点与 ID 映射。
2. MutationObserver 记录 DOM 新增、删除、移动、文本和属性变化；rrweb 还记录鼠标、滚动、输入、视口变化等增量事件。
3. event 带时间戳，mutation 中用 `parentId`、`nextId` 等描述结构位置；客户端按 session 和时间/大小阈值分片上报。
4. Replayer 在隔离环境里重建初始 DOM，再按时间应用增量事件；它复现的是 DOM 与交互轨迹，不会重新执行原业务应用逻辑。

### 为什么体积增长

- 长会话 mutation、鼠标移动、滚动和输入事件不断累积。
- 优化：降低高频事件采样、会话分段、周期 checkout、压缩/批量、过滤高噪 DOM、只在命中错误前后保留窗口。
- 隐私：默认 mask 输入、屏蔽敏感区域、限制跨域 iframe/canvas，并设置保存时长和访问权限。

## 413 怎么办

- 客户端先减量：分片、压缩、降低录屏/行为采样、限制批次和响应体采集。
- 服务端限制要按风险合理调整，不要直接把 body 上限无限放大。
- 大事件携带 session/chunk 序号，服务端幂等合并；失败只重传缺失块。

## 监控系统技术取舍

### 自研还是 Sentry/商业方案

| 方案 | 优势 | 代价 |
| --- | --- | --- |
| Sentry/商业方案 | 成熟、接入快、错误聚合与生态完整 | 成本、数据合规、深度定制受限 |
| 完全自研 | 数据和功能完全可控 | 采集只是开始，后端聚合、告警、查询维护成本很高 |
| 混合方案 | 通用错误复用成熟平台，特殊业务自研插件 | 两套链路的数据一致性与维护复杂 |

个人项目选择自研是为了学习采集、传输和回放原理；真实团队选型要看规模、合规、预算和定制需求，不能只说“自研更轻”。

### EventBus 还是模块直接调用

- 直接调用链路清晰、类型容易追踪，适合模块少且依赖稳定的 SDK。
- EventBus 解耦插件与采集模块，方便按需扩展，但事件名、顺序和异常传播更难追踪。
- 本项目使用 EventBus 做横向通知；核心上报路径仍应保持清晰调用关系，避免所有逻辑都事件化。

### monkey patch 还是浏览器原生观察能力

- fetch/XHR、history 等功能常需包装原方法才能拿到业务上下文。
- PerformanceObserver、全局 error 等原生能力侵入更小，应优先使用。
- monkey patch 必须保存原引用、透传 this/参数/返回值、支持多 SDK 共存和卸载恢复。
- 不应默认采集请求/响应正文，避免性能和隐私风险。

### fetch、Beacon、keepalive 与 Image

| 方式 | 适合 | 限制 |
| --- | --- | --- |
| fetch | 正常批量上报、需要状态和自定义头 | 卸载时可能被取消 |
| sendBeacon | 页面卸载时的小体积尽力发送 | 无法设置任意请求头，不适合大包 |
| fetch keepalive | 卸载阶段且仍需 fetch 语义 | 浏览器有体积/生命周期限制 |
| Image GET | 极小兼容打点 | URL 长度、无响应语义、隐私限制 |

选择策略：正常链路 fetch；卸载小包 Beacon/keepalive；Image 只做极端兼容，不承载错误详情或录屏。

### 实时上报还是批量上报

- 致命错误可立即上报，减少页面崩溃后的丢失概率。
- 行为、性能和普通日志批量/定时上报，降低请求数。
- 批量过大增加丢失和 413 风险；批量过小造成请求风暴。
- 通过条数、字节数、时间和页面生命周期共同触发 flush，并设全局内存上限。

### 全量采集还是采样

- 低流量、高价值错误可接近全量；高频行为和性能数据按用户/session 稳定采样。
- 随机每条采样会让一次会话数据不完整，最好先决定该 session 是否入样。
- 新版本、异常用户或特定页面可动态提高采样率，但要控制偏差。
- 指标分析时记录采样率和版本，否则不同批次不可直接比较。

### rrweb 常驻、按需还是环形缓冲

- 常驻全量录制复现能力强，但性能、存储和隐私成本最高。
- 报错后才启动录制无法看到错误前的操作。
- 更合理的是有限环形缓冲：低采样保留最近一段事件，命中错误后冻结错误前窗口并继续录制短暂后窗口。
- 高敏页面默认关闭或强 mask；canvas、视频和跨域 iframe 要明确能力边界。

### SourceMap 浏览器解析还是服务端解析

- 浏览器解析会增加包体/计算并暴露源码映射，不适合生产。
- 服务端解析需要构建上传、release 管理和存储清理，但安全和版本治理更好。
- CI 上传失败时要告警；找不到精确 release 时宁可保留原 stack，也不错误映射。

## 如何证明监控 SDK 值得接入

不能只看“采到了多少事件”，要同时观察：

- SDK 包体、初始化时间、主线程耗时和自身错误率。
- 上报请求数、字节数、成功率、重试率和丢弃率。
- 错误去重率、SourceMap 解析成功率、问题平均定位时间。
- 录屏命中率、隐私规则覆盖和存储成本。

> 监控的技术取舍本质是在可观测性、业务侵入、数据成本和隐私之间找平衡。最重要的指标是它是否更快定位问题，同时没有成为新的性能和稳定性问题。
