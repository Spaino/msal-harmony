# MSAL for HarmonyOS v0.1.0 — Community Preview

HarmonyOS 原生微软登录 SDK，**仅分发编译产物**。

## 产物

- `msal-harmony-0.1.0.har` — `@msal/harmony@0.1.0`，release 构建
- `msal-harmony-0.1.0.zip` — HAR + 文档 + 校验和

## 特性

- Authorization Code + PKCE（S256），公共客户端
- 系统浏览器 + ArkWeb 内嵌登录
- 静默令牌与 refresh token 续期
- 多账户缓存，默认 Asset Store
- AAD / B2C / CIAM / ADFS authority
- OIDC 登出、错误码、correlation id、可配置日志
- 30 个主机侧协议测试

## 兼容性

- 运行时 API 17+
- HAR 使用 OpenHarmony SDK 20 构建
- DevEco Studio 26.0.0 / HarmonyOS 6 真机验证

## 限制

- 社区预览版，未经过独立安全审计
- 不支持 broker/SSO、POP、Intune、Native Authentication
- 不支持进程被杀后的登录会话恢复

## 校验

```bash
shasum -a 256 -c SHA256SUMS
```

## 声明

MIT。非 Microsoft 官方项目。Microsoft、Microsoft Entra、MSAL 是 Microsoft
Corporation 的商标。

## 校验和

```text
HAR SHA-256: 3ba43f0a96c5dccc6458f3b82ae9f8933b1481f162e60b49ddb8df1b50cb1eac
```

发布页同时提供 `SHA256SUMS`，覆盖 HAR 和 ZIP。
