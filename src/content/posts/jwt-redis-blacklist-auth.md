---
title: Spring Security + JWT + Redis 黑名单的无状态鉴权方案
date: 2025-09-12
excerpt: 这篇记录我在做认证鉴权时对 JWT 主动失效的理解，重点是 Redis 黑名单、权限版本、过滤器链和异常处理。
tags: [Spring Security, JWT, Redis, 认证鉴权, Java]
---

JWT 这块刚学的时候，我觉得它的优点很清楚：无状态、适合分布式、服务端不用保存 Session。用户登录后拿到 Token，后面请求带上 Token，后端校验签名和过期时间就行。

但真正设计登录退出逻辑时，会发现 JWT 有个很经典的问题：它签发出去以后，在过期前默认都是有效的。用户退出登录、修改密码、账号被封禁时，怎么让已经发出去的 Token 失效？

这篇就是我对 Spring Security + JWT + Redis 黑名单方案的理解。

## JWT 无状态的好处和问题

传统 Session 方案里，服务端保存登录态。用户退出时，服务端删 Session 就行。JWT 不一样，它把用户信息、过期时间等内容放在 Token 里，服务端只校验 Token 是否可信。

好处是：

- 服务端不用集中存 Session。
- 多个服务实例校验逻辑一致。
- 微服务或网关场景更方便。
- 横向扩容简单。

问题是：

- Token 泄露后，在过期前可能一直能用。
- 用户退出登录不能天然让 Token 失效。
- 修改密码后旧 Token 仍然可能有效。
- 账号封禁后需要额外机制阻止访问。

所以 JWT 的“无状态”并不是没有代价。

## 基础登录流程

一个比较常见的流程是：

1. 用户提交账号密码。
2. Spring Security 认证用户名和密码。
3. 认证成功后生成 access token。
4. Token 里写入用户 ID、角色、过期时间、jti。
5. 客户端后续请求在 Header 里带上 Token。

Token 里最好有一个唯一 ID，也就是 `jti`：

```json
{
  "sub": "10001",
  "jti": "token-uuid",
  "roles": ["USER"],
  "exp": 1770000000
}
```

后面做黑名单时，就可以把 `jti` 存到 Redis。

## Redis 黑名单怎么做

用户退出登录时，后端解析当前 Token，拿到 `jti` 和剩余过期时间，然后写入 Redis：

```text
auth:blacklist:token-uuid -> 1
TTL = token 剩余有效时间
```

之后每次请求校验 Token 时，除了检查签名和过期时间，还要查 Redis：

```text
如果 jti 在黑名单里，就拒绝访问。
```

TTL 很重要。如果不设置过期时间，黑名单会越积越多。Token 本来过期后就没用了，黑名单也应该自动删除。

## 过滤器链里的位置

JWT 校验一般写在 Spring Security 的自定义过滤器里，放在用户名密码认证过滤器之前。

大概流程是：

```java
String token = resolveToken(request);
if (token == null) {
    filterChain.doFilter(request, response);
    return;
}

Claims claims = jwtService.parse(token);
if (blacklistService.contains(claims.getId())) {
    throw new AuthenticationException("token revoked");
}

UserDetails user = userDetailsService.loadUserById(claims.getSubject());
Authentication authentication = buildAuthentication(user);
SecurityContextHolder.getContext().setAuthentication(authentication);
filterChain.doFilter(request, response);
```

这里有几个细节：

- Header 解析失败要返回 401。
- Token 过期也返回 401。
- 权限不足返回 403。
- 不要把具体签名错误、密钥信息暴露给前端。
- 每次请求结束要注意清理 SecurityContext。

这些都是比较基础但容易漏的地方。

## 修改密码后怎么让所有 Token 失效

Redis 黑名单适合让单个 Token 失效。比如用户退出登录，就把当前 Token 加黑名单。

但如果用户修改密码，可能希望所有历史 Token 都失效。把所有 Token 都找出来加入黑名单不现实。

可以引入 `tokenVersion`：

```text
用户表：token_version = 3
JWT：token_version = 3
```

每次校验 Token 时，比对 Token 里的版本和 Redis/MySQL 中的用户版本。如果用户修改密码，就把版本号加 1。旧 Token 里的版本还是 3，新版本是 4，就拒绝访问。

这个思路比批量拉黑所有 Token 更简单。

## Redis 挂了怎么办

这是一个真实系统里必须想的问题。如果认证链路依赖 Redis，Redis 不可用时怎么办？

有两种策略：

- fail closed：Redis 不可用就拒绝请求，更安全。
- fail open：Redis 不可用时暂时放行，更保证可用性。

选哪种要看业务。如果是后台管理、支付、权限敏感系统，我倾向 fail closed。如果是普通内容浏览，可以短时间 fail open，但要告警。

更稳妥的是：

- Redis 做高可用。
- 黑名单查询设置短超时。
- JWT 有效期不要太长。
- 关键操作再次校验权限版本。

## access token 和 refresh token

如果 access token 有效期太长，泄露风险高；太短，用户体验差。所以一般会配 refresh token。

简单做法：

- access token 有效期短，比如 30 分钟。
- refresh token 有效期长，比如 7 天。
- access token 过期后，用 refresh token 换新 token。
- refresh token 存 Redis 或数据库，支持主动撤销。

这样可以兼顾安全和体验。学生项目里不一定要全部实现，但设计时可以说明扩展方案。

## 小结

JWT 的无状态很适合集群和微服务，但它天然不擅长主动失效。Redis 黑名单可以解决退出登录和单 Token 撤销，权限版本号可以解决修改密码、封禁账号后的批量失效。

我觉得认证鉴权这块不能只写“使用 JWT 实现无状态登录”。更完整的说法应该是：基于 Spring Security + JWT 实现认证，结合 Redis 黑名单实现主动失效，结合权限版本处理批量失效，并区分 401、403 等异常返回。这样才更像一个完整后端方案。
