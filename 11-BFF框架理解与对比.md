# BFF 框架理解与对比

## 一句话定义

> BFF（Backend for Frontend）是运行在服务端、专门面向某一种前端体验的后端中间层。它属于后端代码，但通常不负责核心领域事实，重点负责数据聚合、接口适配、认证会话、协议转换和流程编排。

一句话区分：

> 领域后端回答“业务允许做什么”，BFF 回答“这个前端怎样组合和使用这些能力”。

## 30 秒回答

> 我理解 BFF 不是简单的接口代理，而是面向特定前端的服务端适配层。传统后端通常围绕用户、订单、库存等领域设计通用接口；BFF 则围绕页面和交互场景，把多个下游接口聚合、裁剪成前端直接需要的 View Model，同时集中处理 Session、权限、缓存和协议转换。
>
> 我的 AI Agent 项目使用 Modern.js Integrated BFF。一次 `/api/chat` 请求进入后，BFF 会完成身份和 CSRF 校验、按 token 预算组装最近两轮/摘要/相关记忆、单 Agent/多 Agent 路由、Host 调度和 SSE 转换；模型结束后记录真实 usage，并异步触发记忆压缩。核心优势不只是“少发几个请求”，而是把模型、权限和编排复杂度留在服务端，让 React 前端只消费稳定协议。

## 90 秒回答

> 传统前后端架构里，后端接口通常按照领域设计，要同时服务 Web、App 和其他调用方。一个复杂页面可能要分别请求用户、计划、任务和推荐服务，再由前端处理接口依赖、异常和数据拼装。BFF 在前端与领域服务之间增加一层面向当前客户端的服务端，它可以并行调用多个下游服务，把结果裁剪成页面需要的 View Model，因此能够减少浏览器网络往返和前端编排复杂度。
>
> 但 BFF 不应该替代领域后端。库存扣减、订单状态迁移、财务结算等事务和核心业务规则仍由领域服务负责；BFF 更适合页面数据聚合、Session、协议转换和客户端专属流程。即使 BFF 判断按钮可用，领域后端收到写请求后仍要重新校验。
>
> 我的项目使用字节 Web Infra 团队维护的 Modern.js Integrated BFF。`api/lambda/chat.ts` 按文件约定映射为 `/api/chat`，导出的 `post` 对应 POST 请求；运行时是 Node.js 上的 Hono。Modern.js 的 Rust 优势主要来自 Rsbuild/Rspack 构建链路，提升编译和 HMR，而不是说线上 BFF 请求由 Rust 执行。相比 Express/Koa，它减少了同仓开发、路由和构建胶水；相比 NestJS，它更轻、更适合页面与 AI 编排，但复杂领域服务能力较弱。

## 传统架构与 BFF

### 没有 BFF

```text
React 前端
├─ 请求用户服务
├─ 请求计划服务
├─ 请求任务服务
└─ 请求推荐服务

前端负责：
请求依赖、并发、聚合、错误兼容和字段裁剪
```

主要问题：

- 浏览器与服务端之间存在多次网络往返。
- React 组件容易混入接口编排和兼容逻辑。
- Web、App 对相同领域接口的字段需求不同。
- 下游接口变化可能直接扩散到多个客户端。
- API Key、模型 Prompt 和工具权限不能安全下发浏览器。

### 加入 BFF

```text
React 前端
     │ 一次页面请求
     ▼
Web BFF
├─ 并行调用用户服务
├─ 调用计划与任务服务
├─ 调用推荐/模型服务
└─ 聚合、裁剪、统一错误
     │
     ▼
页面直接需要的 View Model
```

例如：

```json
{
  "user": {},
  "currentPlan": {},
  "recentTasks": [],
  "recommendations": []
}
```

## BFF 的五类能力

### 1. 数据聚合

将多个领域接口组合成一个页面接口，减少前端请求和编排。

### 2. 数据裁剪与适配

同一份业务数据可以按客户端需要转换：

- Web 管理后台返回完整表格字段。
- 移动端只返回核心状态和操作。
- 数据大屏返回聚合指标而不是原始明细。

### 3. 协议转换

```text
模型供应商流
→ BFF 解析与归一化
→ text/status/error/done 等统一 SSE 事件
→ React 前端消费
```

更换模型供应商时，前端协议可以保持稳定。

### 4. 安全与会话

- API Key 和 Prompt 保存在服务端。
- OIDC Token 可留在 BFF，浏览器只持 HttpOnly Session Cookie。
- 集中处理 CSRF、限流、权限和工具白名单。

### 5. 流程编排

数据聚合型 BFF 组合多个查询；编排型 BFF 还会控制执行顺序、超时、降级和状态机。

我的 AI Agent 项目更偏编排型 BFF。

## BFF 与领域后端的边界

| 适合放 BFF | 应由领域后端负责 |
| --- | --- |
| 页面数据聚合与裁剪 | 核心业务规则 |
| View Model 转换 | 事务和强一致性 |
| Web 专属 Session | 库存、财务等事实数据 |
| SSE/供应商协议转换 | 跨客户端复用能力 |
| AI Prompt 与 Agent 编排 | 最终权限和状态校验 |
| 前端专属短期缓存 | 长期领域模型 |

以 MES 为例：

```text
订单 + 工位 + 物料 + 异常记录
→ 聚合成生产详情页
```

适合放 BFF。

```text
扣减物料库存
变更生产状态
执行报废和财务结算
```

必须由领域后端负责。BFF 可以做前置校验，但领域后端仍要最终校验。

## Modern.js Integrated BFF 原理

项目配置：

```ts
export default defineConfig({
  plugins: [appTools(), bffPlugin()],
  bff: {
    prefix: '/api',
  },
});
```

约定式路由：

```text
api/lambda/chat.ts
→ POST /api/chat

api/lambda/conversations.ts
→ GET/POST /api/conversations

api/lambda/conversations/[id].ts
→ GET/PUT/DELETE /api/conversations/:id
```

核心规则：

- 文件路径决定 URL。
- 导出的 `get/post/put/delete` 决定 HTTP Method。
- `RequestOption` 提供 `query/data/headers`。
- 下划线开头的工具文件不会被注册成路由。
- 仍然使用标准 HTTP/REST，不是只能在 Modern.js 内部调用的私有协议。

可以将框架原理理解为：

```text
构建阶段
api/lambda 文件
→ BFF 插件扫描
→ 生成服务端路由和调用适配
→ 分别构建客户端与服务端产物

运行阶段
/api/chat 请求
→ Modern.js Node Server
→ Hono 路由匹配
→ 解析 data/query/headers
→ 执行 chat.ts 的 post()
→ 返回 JSON 或 ReadableStream
```

## Rust、Rspack 与 Hono 不要混淆

```text
构建阶段：
Modern.js → Rsbuild → Rspack（Rust）

运行阶段：
HTTP → Node.js → Modern.js Server → Hono → TypeScript 业务逻辑
```

Rspack 的收益主要是：

- 更快的依赖图分析与编译。
- 更快的冷启动和增量构建。
- 更快的 HMR。
- 兼容 Webpack 生态的代码分包和资源处理。

不能说：

> Modern.js BFF 是 Rust 写的，所以线上接口 QPS 很高。

应该说：

> Modern.js 的构建性能主要来自 Rust 编写的 Rspack；BFF 线上运行在 Node.js，并使用轻量的 Hono。对 AI 接口而言，主要耗时仍然是模型、Embedding 和数据库 RPC，框架路由开销不是主要瓶颈。

## 我的 `/api/chat` 为什么是 BFF

```text
React 发起一次 /api/chat
        ↓
Modern.js BFF
├─ 校验 Session 和 CSRF
├─ 控制 SSE 并发和排队
├─ 创建/查询会话
├─ 读取最近窗口和历史记忆
├─ 缓存短路
├─ 单 Agent/多 Agent/分块策略路由
├─ 执行 Host 状态机
├─ 调用模型与工具
├─ 保存消息和执行检查点
└─ 转换成统一 SSE
        ↓
React 只处理稳定的流式事件
```

如果前端直连模型：

- 模型 Key、Prompt 和工具描述会暴露。
- 用户可以绕过服务端权限和成本控制。
- Agent 状态散落在浏览器，刷新后难恢复。
- 模型供应商协议直接污染 React 组件。
- Web、App 都要重复实现流解析、超时和重试。

## Modern.js 与其他 BFF 方案

### 对比 Express/Koa

| Modern.js Integrated BFF | Express/Koa |
| --- | --- |
| 前端和 BFF 同仓开发 | 通常单独初始化服务 |
| 文件约定自动生成路由 | 手动注册 Router |
| 统一开发、构建和部署 | 自己配置构建与开发代理 |
| TypeScript/调用集成更强 | 自由度和生态更高 |
| 约定较强 | 可以完全自定义结构 |

选择口径：

> 项目只有一个 React 客户端，BFF 主要服务聊天与 Agent 页面，Modern.js 能减少路由、类型、代理和构建胶水。如果需要建设一个独立平台服务，或者公司已有成熟 Node 服务体系，Express/Koa 的自由度更高。

### 对比 NestJS

| Modern.js BFF | NestJS |
| --- | --- |
| 面向前端集成和页面编排 | 面向完整后端应用 |
| 函数式、文件路由 | Controller、Module、Provider |
| 轻量、开发快 | DI、Guard、Pipe、Interceptor 完整 |
| 适合聚合、认证和 AI 控制面 | 适合复杂领域服务与多人协作 |

选择口径：

> 如果是复杂企业后端，我会更倾向 NestJS；当前项目核心是 React 和 AI 链路协同，因此选择轻量的 Modern.js BFF。随着业务增长，我在 BFF 内部增加 Clean Architecture 和依赖注入，避免所有业务堆在路由文件中。

### 对比 Next.js Route Handlers

| Modern.js | Next.js |
| --- | --- |
| `api/lambda/chat.ts` | `app/api/chat/route.ts` |
| `post()` | `POST()` |
| Modern.js/Rspack 体系 | Next.js/App Router 体系 |
| 适合渐进式 CSR/SSR 与 Integrated BFF | 与 RSC、SSR 和 Next 部署结合更深 |

选择口径：

> 两者都能实现同仓 BFF，不应说 Modern.js 绝对更快。当时项目已经使用 Modern.js，插件和 Rspack 构建链路集成最好；如果项目强依赖 RSC 和 Next.js 生态，Route Handlers 会更自然。

## BFF 的代价

- 多了一次服务端转发和部署单元。
- BFF 可能成为新的单点或性能瓶颈。
- 前端和 BFF 发布节奏容易强绑定。
- 如果把领域逻辑全部塞进 BFF，会演变成难维护的“小后端”。
- 多个客户端各有 BFF 时，公共逻辑可能重复。

对应治理：

- BFF 保持薄控制层，复杂逻辑下沉 use case 或领域服务。
- 设置超时、取消、限流、熔断和可观测 trace。
- 下游请求尽量并行，避免串行瀑布。
- 公共能力下沉共享服务，客户端差异留在 BFF。
- 静态资源走 CDN，长任务拆队列或工作流。

## QPS 上升十倍怎么分析

不要直接回答“给 BFF 加缓存”，先拆链路：

```text
BFF 路由
→ Session/Redis
→ MongoDB
→ Embedding
→ Agent/模型 RPC
→ SSE 长连接
```

优先检查：

1. 模型调用的并发、P95 和 token 成本。
2. Embedding 是否在同步主链路，能否异步化和批处理。
3. Mongo 查询是否命中索引，应用层 fallback 是否扫描过多候选。
4. SSE 长连接数、网关超时、断开取消和背压。
5. 多 Agent 是否对简单请求产生不必要调用。

优化方向：

- 简单问题绕过多 Agent。
- 记忆建索引异步进入队列。
- 使用数据库原生全文/向量索引。
- 对确定性且版本一致的请求做缓存和请求合并。
- 设置用户、模型和 token 维度的并发预算。

## 高频追问

### BFF 是后端吗

> 是服务端后端代码，但定位是面向特定前端的适配与编排层，不等同于承载全部领域逻辑的通用后端。

### BFF 最大优势是不是一次请求拿完数据

> 这是重要优势之一，但不完整。它还负责客户端专属的数据裁剪、Session、安全、协议转换和流程编排。

### BFF 会不会增加一次网络请求

> 会增加 BFF 到下游的服务跳转，但浏览器只需请求 BFF。BFF 可以在机房内并行调用多个服务，通常能减少公网往返和前端瀑布请求；是否更快仍需用 trace 验证。

### 为什么不用传统后端直接做

> 技术上可以。选择 BFF 的原因是把页面专属的聚合、Prompt、流协议和发布节奏从通用领域服务中隔离，不让客户端需求污染稳定领域接口。

### 为什么不用 Express

> Express 能完成相同功能。当前选择 Modern.js 是因为 React 与 BFF 同仓，文件路由、TypeScript、开发服务器和 Rspack 构建已经集成，项目规模下胶水成本更低。

### Modern.js BFF 为什么快

> 构建和开发速度主要来自 Rust Rspack；请求运行时使用 Node.js 和 Hono。线上 AI 请求快不快主要取决于模型、数据库、缓存和并发设计，不能归功于 Rust 构建工具。

## 容易失分的表达

- “BFF 不是后端。”
- “BFF 就是转发接口。”
- “用了 BFF 就一定只需要一个请求。”
- “BFF 可以代替所有领域后端。”
- “Modern.js BFF 是 Rust 写的。”
- “Hono 很快，所以模型接口整体就很快。”
- “Modern.js 一定比 Next.js、NestJS 更好。”

## 收尾模板

> BFF 没有脱离场景的绝对优势。我的场景是单一 React 客户端，但服务端需要组合认证、会话、历史检索、模型和 Agent 状态，还要转换流式协议，因此 Integrated BFF 能明显降低前端复杂度。它的代价是多一个服务层和部署边界，所以我把核心业务事实保留在数据库或领域服务，把页面专属的聚合与 AI 编排留在 BFF，并通过超时、限流、取消和 trace 控制风险。

## 官方资料

- [Modern.js BFF](https://modernjs.dev/guides/advanced-features/bff)
- [Modern.js BFF Function Routes](https://modernjs.dev/guides/advanced-features/bff/function)
- [Modern.js BFF Runtime Framework](https://modernjs.dev/guides/advanced-features/bff/frameworks.html)
- [Modern.js Web Server](https://modernjs.dev/guides/concept/server)
- [Rspack](https://www.rspack.dev/)
- [Hono](https://hono.dev/)
- [NestJS Controllers](https://docs.nestjs.com/controllers)
- [Next.js Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers)
