# AI Agent 认证架构与 Redis 限流压测

> 独立专题：OAuth2/OIDC、双 Token、PKCE、BFF Session、Redis 分布式限流，以及 Pipeline、Lua、WATCH/MULTI CAS 压测。

---

## 一、30 秒项目概述

> 项目使用 OAuth2/OIDC Authorization Code Flow + PKCE 完成统一登录，使用 access token 和 refresh token 管理资源访问与续期。敏感 Token 不直接交给浏览器，而是由 BFF 托管在 Redis Session 中，浏览器只保存 HttpOnly Session Cookie。登录、OIDC callback 和 CSRF 等安全敏感接口则使用 Redis 做多实例共享的固定窗口限流。我还针对限流计数器实现了 Pipeline、Lua 和 WATCH/MULTI CAS 三种方案的专项压测，最终认为 Lua 在原子性、网络 RTT 和热点 Key 稳定性方面更适合该场景。

---

## 二、OIDC、OAuth2、JWT 和双 Token 的关系

这四个概念不是互斥关系：

| 概念 | 主要职责 |
| --- | --- |
| OAuth2 | 解决授权问题，让客户端获得访问资源的权限 |
| OIDC | 在 OAuth2 上增加身份认证层，统一表达“用户是谁” |
| JWT | Token 的一种自包含数据格式 |
| 双 Token | 使用 access token 和 refresh token 管理访问与续期 |

项目不是没有使用 JWT 和双 Token：

- `id_token` 本身就是 JWT。
- access token 用于访问受保护资源。
- refresh token 用于无感刷新 access token。
- 浏览器不直接保存双 Token，Token 由 BFF 托管。

最准确的一句话：

> 项目不是在 OIDC/OAuth 和 JWT 之间二选一，而是用 OIDC/OAuth2 定义标准认证授权流程，用 JWT `id_token` 表达身份，用双 Token 管理访问和续期，再通过 BFF Session 降低浏览器侧的 Token 暴露风险。

---

## 三、为什么不只使用简单 JWT

简单 JWT 的优点：

- 自包含。
- 资源服务器可以本地验签。
- 不需要每次查询 Session。
- 适合服务间调用和水平扩展。

但项目存在第三方登录、SSO 和多端接入需求。单纯自己签发 JWT 只能解决“如何表达和验证身份”，不能完整解决：

- 第三方 IdP 接入。
- 统一登录和单点登录。
- 授权码交换。
- 标准化用户身份 Claim。
- Discovery 和 JWKS 公钥发现。
- access token、refresh token 的签发、刷新和撤销。
- 单点退出、账号风控和审计。

JWT 的无状态也有代价：

> 在不增加服务端状态的情况下，尚未过期的 JWT 很难被立即撤销。

例如用户退出、账号被禁用、权限改变或者设备被踢下线时，旧 JWT 可能在过期前继续使用。要实现立即失效，通常还要增加：

- Token 黑名单。
- Token Version。
- Session Store。
- Token Introspection。
- 更短的 Token 有效期。

这些机制会重新引入服务端状态或增加刷新成本。因此项目选择混合架构：

```text
OIDC/OAuth2：标准认证授权
JWT id_token：表达身份
双 Token：访问与续期
BFF Session：浏览器安全会话与集中控制
Redis：共享会话、临时状态和限流计数
```

---

## 四、项目真实登录流程

```text
浏览器
  │ 点击登录
  ▼
BFF 生成 state、nonce、code_verifier
  │
  ├─ 将登录上下文写入 Redis，并设置短 TTL
  ▼
重定向到 OIDC IdP
  │ 用户完成身份认证
  ▼
IdP 返回 authorization code
  │
  ▼
BFF 使用 code + code_verifier 换取
access_token + refresh_token + id_token
  │
  ├─ 使用 JWKS 校验 id_token 签名
  ├─ 校验 iss、aud、exp、nonce
  ├─ 创建随机 Session ID
  ├─ Token 和用户信息写入 Redis Session
  ▼
浏览器只得到 HttpOnly Session Cookie
```

浏览器收到的 Cookie 类似：

```http
Set-Cookie: __Host-bff_sid=随机值;
HttpOnly;
Secure;
SameSite=Lax;
Path=/
```

代码依据：

- `api/lambda/_utils/bffOidcAuth.ts:230`：使用 JWKS 验证 JWT `id_token` 的签名、issuer 和 audience。
- `api/lambda/_utils/bffOidcAuth.ts:243`：创建包含 access token、refresh token、id token 的 BFF Session。
- `api/lambda/_utils/bffOidcAuth.ts:257`：将 Session 写入 Redis，并设置 TTL。
- `api/lambda/_utils/bffOidcAuth.ts:73`：下发 HttpOnly、SameSite Cookie，HTTPS 环境增加 Secure。

---

## 五、state、nonce 和 PKCE 分别解决什么

| 参数 | 解决的问题 | 校验方式 |
| --- | --- | --- |
| `state` | 登录回调 CSRF、错误流程关联 | 回调时与 Redis 中的登录上下文匹配 |
| `nonce` | `id_token` 重放 | JWT 中的 nonce 必须与发起登录时一致 |
| `code_verifier` / `code_challenge` | 授权码被截获后被攻击者换 Token | IdP 校验 verifier 计算出的 challenge |

PKCE 流程：

```text
1. BFF 生成高随机性的 code_verifier
2. code_challenge = BASE64URL(SHA256(code_verifier))
3. 授权请求只携带 code_challenge
4. BFF 使用 authorization code 换 Token 时提交 code_verifier
5. IdP 重新计算 challenge，并与授权阶段保存的值比较
```

即使 authorization code 在跳转链路中泄漏，攻击者没有 `code_verifier`，也不能拿授权码换取 Token。

PKCE 对没有 client secret 的 SPA、移动端等公共客户端尤其重要。本项目的 BFF 可以保存 client secret，但仍保留 PKCE，作为授权码泄漏防护和纵深防御。

---

## 六、为什么使用双 Token

access token：

- 生命周期较短。
- 用于访问受保护资源。
- 泄漏后的风险窗口相对有限。

refresh token：

- 生命周期较长。
- 用于无感刷新 access token。
- 可以结合 Token 轮换、旧 Token 失效和重放检测。

双 Token 解决的是安全性与用户体验之间的平衡：

> access token 太长会扩大泄漏风险窗口；access token 很短但没有 refresh token，又会导致用户频繁重新登录。

### 为什么不让浏览器直接保存双 Token

项目没有将 access token 和 refresh token 存入 localStorage，而是：

```text
浏览器 Cookie：
只保存不可读的随机 sid

Redis Session：
保存 user、access token、refresh token、id token、过期时间
```

这样做的优点：

- HttpOnly Cookie 无法被前端 JavaScript 直接读取。
- 降低 XSS 直接窃取 refresh token 的风险。
- Token 刷新、轮换、撤销、权限校验和设备风控集中在服务端。
- 删除 Redis Session 即可让本应用会话快速失效。
- 前端只需请求 `/api/auth/me`，不需要自己解析和刷新 Token。

需要承认的代价：

- 每次认证需要访问 Redis。
- Redis 成为认证链路的重要依赖。
- Cookie 会自动携带，写操作仍需要 SameSite、CSRF Token 和 Origin/Referer 校验。

---

## 七、认证选型完整面试说辞

> 因为项目存在第三方登录、SSO 和多端接入需求，简单 JWT 更适合解决单个系统里的身份声明问题，但缺少完整的标准化认证授权流程。需要说明的是，OIDC/OAuth 和 JWT 并不冲突：OAuth2 解决授权，OIDC 在 OAuth2 上增加身份层，JWT 只是 Token 格式，我们的 `id_token` 本身就是 JWT。
>
> 登录采用 Authorization Code Flow + PKCE。BFF 生成 `state`、`nonce` 和 `code_verifier`，把 `code_challenge` 发给 IdP。回调后，BFF 使用 authorization code 和 `code_verifier` 换取 access token、refresh token 和 id token，并通过 JWKS 校验 id token 的签名以及 `iss`、`aud`、`exp`、`nonce` 等声明。
>
> 双 Token 中，短期 access token 用于访问资源，长期 refresh token 用于无感续期。出于安全考虑，我们没有把双 Token 直接暴露给浏览器，而是由 BFF 托管在 Redis Session 中，浏览器只持有 HttpOnly、Secure、SameSite 的 Session Cookie。这样可以降低 XSS 窃取 refresh token 的风险，同时把刷新、撤销、风控和审计集中在服务端。

---
