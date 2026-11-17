# MSAL for HarmonyOS（社区版）

**HarmonyOS 原生微软身份登录 SDK，以编译好的 HAR 产物分发。**

[下载最新版本](https://github.com/Spaino/msal-harmony/releases/latest) · [English](README.en.md) · [发布说明](RELEASE_NOTES.md) · [安装集成](docs/INSTALL.md) · [API 参考](docs/API.md) · [H5 JSBridge](docs/H5_BRIDGE.md) · [问题排查](docs/TROUBLESHOOTING.md)

![HarmonyOS](https://img.shields.io/badge/HarmonyOS-API%2017%2B-000000?logo=harmonyos)
![Version](https://img.shields.io/badge/version-0.1.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-community%20preview-orange)

> **非 Microsoft 官方项目。** 本项目是社区维护的 HarmonyOS 原生实现，
> 与 Microsoft Corporation 没有隶属或背书关系。Microsoft、Microsoft Entra、
> MSAL 是 Microsoft Corporation 的商标。使用前请阅读 [免责声明](#免责声明)。

---

## 这个仓库是什么

这是一个**产物分发仓库**：只包含编译后的 `.har`、类型声明、文档和校验和，
不包含 SDK 的 ArkTS 实现源码。

```text
msal-harmony-0.1.0.har
  ├── package/ets/modules.abc       # 编译后的 ArkTS 字节码
  ├── package/src/**/*.d.ets        # 公开类型声明
  ├── package/Index.d.ets           # 入口声明
  └── package/oh-package.json5      # @msal/harmony@0.1.0
```

HAR 已按 `buildMode=release` 构建（`debug=false`），不包含实现源码或 source map。
类型声明用于 DevEco Studio 的代码补全和 ArkTS 类型检查。

## 特性

- **Authorization Code + PKCE（S256）** 公共客户端登录，不保存 client secret
- **系统浏览器登录**：通过 Implicit Want 打开默认浏览器，回调 URI 交回 SDK
- **ArkWeb 内嵌登录**：`createAuthorizationRequest()` + `Web.onLoadIntercept()`
- **静默令牌**：access token 缓存命中、refresh token 续期
- **多账户缓存**：按 `homeAccountId` / `username` 查询账户
- **OIDC 登出**：可清理本地缓存，也可打开 end-session 清理浏览器会话
- **Asset Store 持久化**：默认使用 HarmonyOS 硬件级加密的 Asset Store
- **可插拔缓存**：实现 `MsalCacheStorage` 即可接入企业密钥库
- **Authority 支持**：AAD、B2C、CIAM、ADFS，以及中国云、US Gov、德国云、法国云
- **安全校验**：state、nonce、PKCE、id_token `aud` / `azp` / `exp`
- **可控日志**：日志级别和回调；SDK 本身不输出 token、code 或 claims 原文
- **30 个主机侧协议单元测试**通过

## 适用范围

适合：

- HarmonyOS 原生 App 需要微软 / Microsoft Entra 账号登录
- 需要把 Microsoft access token 换成自有服务端 token 的业务
- 需要在 H5 容器里通过 JSBridge 复用原生微软登录
- 需要系统浏览器或 ArkWeb 两种登录体验

**不适合**：

- 需要 Microsoft Authenticator broker / 设备 SSO / Intune 的高保障场景
- 需要 POP（Proof of Possession）令牌
- 需要 Native Authentication（邮箱 OTP / 原生密码）
- 需要进程被杀后恢复浏览器登录会话
- 需要服务端校验 Microsoft token 的场景（服务端仍应自行校验签名、issuer、audience）

## 环境要求

| 项目 | 要求 |
|---|---|
| 运行时 | HarmonyOS / OpenHarmony API 17+（`compatibleSdkVersion: 17`） |
| HAR 构建目标 | OpenHarmony SDK 20 |
| 推荐 IDE | DevEco Studio 5.0.5+；0.1.0 使用 DevEco Studio 26.0.0 验证 |
| 语言 | ArkTS |
| 网络权限 | `ohos.permission.INTERNET` |
| 回调方式 | 自定义 scheme + `launchType: singleton` |

> HAR 的 `compatibleSdkVersion` 是 17。实际项目可以用更高的 compileSdk / targetSdk；
> 0.1.0 在 HarmonyOS 6（API 26）工程中完成过编译和真机回调验证。

## 安装

> 当前只通过 GitHub Releases 分发，未发布到 ohpm 公共仓库。

### 1. 下载 HAR

从 [Releases](https://github.com/Spaino/msal-harmony/releases/latest) 下载：

```text
msal-harmony-0.1.0.har
```

放到工程的公共目录，例如：

```text
libs/msal-harmony-0.1.0.har
```

### 2. 在模块中声明依赖

在需要使用 SDK 的模块的 `oh-package.json5` 中：

```json5
{
  "dependencies": {
    "@msal/harmony": "file:../libs/msal-harmony-0.1.0.har"
  }
}
```

`file:` 路径是相对于该模块 `oh-package.json5` 的路径，请按实际目录调整。

然后执行：

```bash
ohpm install
```

### 3. 声明网络权限和回调 scheme

在 `entry/src/main/module.json5` 中：

```json5
{
  "module": {
    "requestPermissions": [
      { "name": "ohos.permission.INTERNET" }
    ],
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "launchType": "singleton",
        "exported": true,
        "skills": [
          {
            "actions": ["ohos.want.action.viewData"],
            "entities": ["entity.system.browsable"],
            "uris": [
              {
                "scheme": "msal11111111-2222-3333-4444-555555555555",
                "host": "auth"
              }
            ]
          }
        ]
      }
    ]
  }
}
```

`scheme` 必须和 `config.redirectUri` 完全一致。默认重定向 URI 格式是：

```text
msal<clientId>://auth
```

例如 clientId 是 `11111111-2222-3333-4444-555555555555`：

```text
msal11111111-2222-3333-4444-555555555555://auth
```

> 这是 HarmonyOS 社区版的默认格式。它与 MSAL iOS 的
> `msauth.<bundleId>://auth` 不同。如果要复用已有 iOS / Android 注册，
> 请显式设置 `config.redirectUri`，并在 `module.json5` 注册对应 scheme。

### 4. 注册 Microsoft Entra 应用

在 [Microsoft Entra 管理中心](https://entra.microsoft.com/)：

1. 注册应用，选择 **Mobile and desktop applications**。
2. 添加自定义 Redirect URI：`msal<clientId>://auth`（或你自己的显式 URI）。
3. 在 **API permissions** 添加所需 scope，例如 `User.Read`。
4. 不需要 client secret：这是公共客户端，使用 PKCE。

## 快速开始

### 创建进程级单例

SDK 的 `MsalPublicClientApplication` 必须保持为同一个实例，否则浏览器回调回来时
找不到 pending 的 `state` / `code_verifier`。

```ts
// MsalService.ets
import { common } from '@kit.AbilityKit';
import {
  MsalAuthority,
  MsalPublicClientApplication,
  MsalPublicClientApplicationConfig
} from '@msal/harmony';

const CLIENT_ID = 'YOUR_CLIENT_ID';
const AUTHORITY = 'https://login.microsoftonline.com/YOUR_TENANT_ID';
const REDIRECT_URI = `msal${CLIENT_ID}://auth`;

let app: MsalPublicClientApplication | undefined;

export function getMsalApp(context?: common.UIAbilityContext): MsalPublicClientApplication {
  if (app !== undefined) {
    return app;
  }
  if (context === undefined) {
    throw new Error('MsalPublicClientApplication requires a UIAbilityContext.');
  }

  const config = new MsalPublicClientApplicationConfig(
    CLIENT_ID,
    MsalAuthority.parse(AUTHORITY),
    REDIRECT_URI
  );
  app = new MsalPublicClientApplication(config, context);
  return app;
}
```

### 交互式登录

```ts
import { MsalInteractiveTokenParameters, MsalPromptType } from '@msal/harmony';

const msal = getMsalApp();
const parameters = new MsalInteractiveTokenParameters(['User.Read']);
parameters.promptType = MsalPromptType.SELECT_ACCOUNT;

const result = await msal.acquireTokenInteractive(parameters);
console.info(`signed in: ${result.account.username}`);
console.info(`expires: ${result.expiresOn.toISOString()}`);
```

`acquireTokenInteractive()` 会打开系统浏览器，并返回一个 Promise。Promise 在宿主 App
把回调 URI 传给 `handleResponse()` 后完成。

### 转发浏览器回调

在 `EntryAbility` 的 `onCreate` 和 `onNewWant` 中都要处理：

```ts
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
import { MsalPublicClientApplication } from '@msal/harmony';
import { getMsalApp } from '../service/MsalService';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    const msal = getMsalApp(this.context);
    this.forwardMsalResponse(msal, want);
  }

  onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    const msal = getMsalApp(this.context);
    this.forwardMsalResponse(msal, want);
  }

  private forwardMsalResponse(msal: MsalPublicClientApplication, want: Want): void {
    if (want.uri !== undefined && want.uri.length > 0) {
      const handled = msal.handleResponse(want.uri);
      if (handled) {
        return;
      }
    }
    // 继续处理 AppLink / 其他 deep link
  }
}
```

`handleResponse()` 返回 `true` 表示这是当前 MSAL 会话的回调，并已被消费；
返回 `false` 时宿主可以继续处理其他 URI。

### 静默获取令牌

```ts
import { MsalSilentTokenParameters } from '@msal/harmony';

const account = await msal.accountForIdentifier(result.account.identifier);
if (account === undefined) {
  // 缓存被清理，需要重新交互登录
} else {
  const silent = new MsalSilentTokenParameters(['User.Read'], account);
  const refreshed = await msal.acquireTokenSilent(silent);
}
```

`acquireTokenSilent()` 的行为：

1. 命中未过期的 access token，直接返回；
2. 否则用 refresh token 换新 token；
3. refresh token 失效（`invalid_grant`）时返回 `INTERACTION_REQUIRED`，应重新交互登录。

### 账户与登出

```ts
import { MsalSignoutParameters } from '@msal/harmony';

const accounts = await msal.allAccounts();
const account = await msal.accountForUsername('user@contoso.com');

if (account !== undefined) {
  const signout = new MsalSignoutParameters();
  signout.signoutFromBrowser = true; // 同时打开 OIDC end-session，清浏览器 cookie
  await msal.signoutWithAccount(account, signout);
}
```

## ArkWeb 内嵌登录

系统浏览器是推荐路径。如果产品必须把登录页嵌在 App 内：

```ts
import { webview } from '@kit.ArkWeb';
import { MsalInteractiveTokenParameters } from '@msal/harmony';

const controller = new webview.WebviewController();
const msal = getMsalApp();
const request = await msal.createAuthorizationRequest(
  new MsalInteractiveTokenParameters(['User.Read'])
);

Web({ src: request.url, controller: controller })
  .onLoadIntercept((event) => {
    const url = event.data.getRequestUrl();
    if (msal.handleResponse(url)) {
      return true; // 拦截回调，阻止 Web 继续导航
    }
    return false;
  });

const result = await request.result;
```

注意：

- 同一时间只允许一个 pending 交互会话；
- 回调被拦截后关闭 `Web` 组件即可实现“登录后自动 dismiss”；
- 内嵌 WebView 可能被 Conditional Access 策略拒绝；生产环境优先系统浏览器；
- `acquireTokenInteractive()` 不能驱动内嵌组件，内嵌必须使用
  上面的 `createAuthorizationRequest()`。

## H5 JSBridge

HAR 只负责原生 OAuth/MSAL，不包含某个 H5 容器的 bridge。社区版文档给出了
`msLogin` / `msLogout` 的推荐契约和原生侧实现方式：

- [H5 JSBridge 集成](docs/H5_BRIDGE.md)

核心原则：

- H5 不接触 Microsoft access token；
- 原生完成 MSAL 登录后，用 Microsoft token 调业务服务端换取 **B 端 token**；
- `msLogin` 只把业务 token 返回给 H5；
- `msLogout` 清理 MSAL 缓存和业务 token。

## 令牌缓存

默认使用 `MsalAssetCacheStorage`：

- 数据写入 HarmonyOS Asset Store；
- 由系统使用硬件级密钥保护；
- 仅当前 App 可访问；
- 设备首次解锁后可读（`DEVICE_FIRST_UNLOCKED`）。

测试或特殊场景可替换为内存缓存：

```ts
import { MsalMemoryCacheStorage } from '@msal/harmony';

config.cacheConfig.storage = new MsalMemoryCacheStorage();
```

企业密钥库 / 文件兜底等场景实现接口即可：

```ts
import { MsalCacheStorage } from '@msal/harmony';

export class MyCacheStorage implements MsalCacheStorage {
  async read(): Promise<string | undefined> { /* ... */ }
  async write(data: string): Promise<void> { /* ... */ }
  async clear(): Promise<void> { /* ... */ }
}
```

> 缓存里包含 access token、refresh token、id token、账户和 claims。
> 不要打印、上传或跨用户共享缓存内容。

## 日志与错误

```ts
import { MsalError, MsalErrorCode, MsalLogger, MsalLogLevel } from '@msal/harmony';

MsalLogger.setLogLevel(MsalLogLevel.INFO);
MsalLogger.setLogCallback((level, message, containsPii) => {
  // SDK 不输出 token / code / claims；这里也不要额外打印它们
  console.info(`[MSAL] ${message}`);
});

try {
  await msal.acquireTokenSilent(parameters);
} catch (error) {
  if (error instanceof MsalError) {
    if (error.code === MsalErrorCode.INTERACTION_REQUIRED) {
      // 重新走交互式登录
    }
    console.error(`${error.code} ${error.message} ${error.oauthError ?? ''}`);
  }
}
```

常见错误：

| 错误码 | 含义 | 处理 |
|---|---|---|
| `INTERACTION_REQUIRED` | 需要重新登录/同意 | 重新交互登录 |
| `USER_CANCELED` | 用户取消/超时 | 提示后重试 |
| `INVALID_GRANT` | refresh token 失效 | 清缓存后重新登录 |
| `INVALID_STATE` | 回调 state 不匹配 | 拒绝回调，重新发起 |
| `INVALID_RESPONSE` | token / id_token 格式异常 | 记录 correlationId 排查 |
| `NETWORK_ERROR` | 网络失败 | 重试或提示 |
| `FAILED_AUTHORITY_VALIDATION` | authority 未通过校验 | 加 `knownAuthorities` 或修正配置 |
| `CACHE_ERROR` | Asset Store 读写失败 | 检查设备能力，改用自定义缓存 |
| `INTERACTIVE_SESSION_ALREADY_RUNNING` | 已有 pending 登录 | 先 cancel 或等待完成 |

## API 速览

| 类型 | 作用 |
|---|---|
| `MsalPublicClientApplication` | 登录、静默令牌、账户、登出入口 |
| `MsalPublicClientApplicationConfig` | clientId、authority、redirectUri、缓存、端点 |
| `MsalAuthority` | AAD / B2C / CIAM / ADFS authority |
| `MsalInteractiveTokenParameters` | 交互式登录参数 |
| `MsalSilentTokenParameters` | 静默令牌参数 |
| `MsalSignoutParameters` | 登出参数 |
| `MsalResult` | access token、id token、账户、authority、correlationId |
| `MsalAccount` | 账户标识、username、tenant profiles |
| `MsalCacheStorage` | 自定义 token 缓存接口 |
| `MsalLogger` / `MsalError` | 日志和错误 |

完整说明见 [API 参考](docs/API.md)。

## 与 MSAL iOS 的关系

本 SDK 参考 MSAL iOS 的公开 API、OAuth 2.0 / OIDC 行为和令牌缓存模型，
用 ArkTS 重新实现 HarmonyOS 平台层：

| iOS MSAL | HarmonyOS 社区版 |
|---|---|
| `ASWebAuthenticationSession` / Safari | 系统浏览器 + Implicit Want |
| `WKWebView` | `createAuthorizationRequest()` + ArkWeb `Web` |
| Keychain | Asset Store |
| `NSURLSession` | `@ohos.net.http` |
| Objective-C completion block | `Promise` / `async` |
| `handleMSALResponse` | `handleResponse(url)` |

这不是 Objective-C 代码的逐行翻译，也不包含 Microsoft 的源代码。

## 已知限制（0.1.0）

- 未经过第三方安全审计；
- 不支持 Authenticator broker / 设备 SSO / SSO extension；
- 不支持 POP、Intune / MAM / WPJ；
- 不支持 Native Authentication；
- 不做 authority metadata discovery，端点按 AAD v2.0 规则拼接，可用
  `MsalAuthorityEndpoints` 覆盖；
- 不支持进程被杀后的登录会话恢复；
- 内嵌 WebView 可能受 Conditional Access 限制；
- 只支持 Bearer token，不支持 POP；
- 日志和错误信息不包含 token，但集成方仍需避免记录敏感数据。

## 安全

请阅读 [SECURITY.md](SECURITY.md)。关键点：

- 只使用授权码 + PKCE，不保存 client secret；
- `state`、`nonce`、`code_verifier` 使用系统安全随机数；
- 回调校验 state；id_token 校验 nonce、aud、exp；
- token 缓存默认在 Asset Store；
- SDK 不记录 token、code、refresh token 或 raw claims；
- 服务端仍需验证 access token 的签名、issuer、audience、scope；
- 本 SDK 尚未经过独立安全审计。

## 校验产物

```bash
shasum -a 256 -c SHA256SUMS
```

`artifacts/SHA256SUMS` 包含 HAR 的 SHA-256。发布页同时提供
`msal-harmony-0.1.0.zip` 和校验和。

## 版本与升级

- 包名：`@msal/harmony`
- 当前版本：`0.1.0`
- 版本遵循 SemVer；`0.x` 阶段不保证 API 完全稳定。
- 升级前请阅读 [CHANGELOG.md](CHANGELOG.md)，并重新执行 `ohpm install`。

## 反馈与贡献

- Bug / 集成问题：使用 GitHub Issues，并附上 DevEco Studio / SDK / 设备版本，
  以及**脱敏后**的 `MsalLogger` 日志。
- 安全漏洞：不要公开提交 Issue，请阅读 [SECURITY.md](SECURITY.md)。
- 本仓库只分发产物，不接受 SDK 实现源码 PR；功能建议和集成问题欢迎反馈。

## 免责声明

本项目是独立的社区项目，不是 Microsoft 官方产品，也未获得 Microsoft 的认可或支持。
Microsoft、Microsoft Entra、MSAL 及相关标识是 Microsoft Corporation 的商标。
使用本 SDK 即表示你理解并接受：你需要自行评估其安全性、合规性和适用性，
并自行承担使用风险。SDK 按 “AS IS” 提供，不附带任何明示或暗示担保。

## License

MIT。详见 [LICENSE](LICENSE)。

参与本项目即表示同意遵守 [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)。
