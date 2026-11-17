# MSAL for HarmonyOS (Community Edition)

**An unofficial, native Microsoft identity SDK for HarmonyOS, distributed as a
compiled HAR artifact.**

[Download](https://github.com/Spaino/msal-harmony/releases/latest) · [中文](README.md) · [Install](docs/INSTALL.md) · [API](docs/API.md) · [H5 bridge](docs/H5_BRIDGE.md) · [Troubleshooting](docs/TROUBLESHOOTING.md)

![HarmonyOS](https://img.shields.io/badge/HarmonyOS-API%2017%2B-000000?logo=harmonyos)
![Version](https://img.shields.io/badge/version-0.1.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)

> **Not an official Microsoft project.** This is a community implementation and
> is not affiliated with, endorsed by, or supported by Microsoft Corporation.
> Microsoft, Microsoft Entra and MSAL are trademarks of Microsoft Corporation.

This repository is an **artifact distribution channel**. It ships the compiled
`msal-harmony-0.1.0.har`, public `.d.ets` declarations, documentation and
checksums. It does not contain the SDK implementation source.

## Features

- Authorization Code + PKCE (S256), public client, no client secret
- System browser via Implicit Want, plus ArkWeb embedded flow
- Silent token acquisition and refresh-token renewal
- Multi-account cache; Asset Store by default, pluggable `MsalCacheStorage`
- AAD, B2C, CIAM and ADFS authorities, including national clouds
- OIDC sign-out, error codes, correlation ids and configurable logging
- 30 host-side protocol tests

## Requirements

| Item | Requirement |
|---|---|
| Runtime | HarmonyOS / OpenHarmony API 17+ |
| HAR target | OpenHarmony SDK 20 |
| IDE | DevEco Studio 5.0.5+; 0.1.0 validated with DevEco Studio 26.0.0 |
| Language | ArkTS |
| Permission | `ohos.permission.INTERNET` |

## Install

> The HAR is distributed through GitHub Releases only; it is not published to the ohpm public registry.

1. Download `msal-harmony-0.1.0.har` from [Releases](https://github.com/Spaino/msal-harmony/releases/latest).
2. Put it in `libs/msal-harmony-0.1.0.har`.
3. Add the dependency to the consuming module's `oh-package.json5`:

```json5
{
  "dependencies": {
    "@msal/harmony": "file:../libs/msal-harmony-0.1.0.har"
  }
}
```

4. Run `ohpm install`.
5. Declare `ohos.permission.INTERNET`, `launchType: singleton`, and the redirect
   scheme/host in `module.json5`.

The default redirect URI is:

```text
msal<clientId>://auth
```

MSAL iOS uses `msauth.<bundleId>://auth`. To reuse an existing iOS registration,
set `config.redirectUri` explicitly and register the same scheme.

## Minimal usage

```ts
import { common } from '@kit.AbilityKit';
import {
  MsalAuthority,
  MsalInteractiveTokenParameters,
  MsalPromptType,
  MsalPublicClientApplication,
  MsalPublicClientApplicationConfig
} from '@msal/harmony';

let app: MsalPublicClientApplication | undefined;

export function getMsalApp(context?: common.UIAbilityContext): MsalPublicClientApplication {
  if (app !== undefined) {
    return app;
  }
  if (context === undefined) {
    throw new Error('MsalPublicClientApplication requires a UIAbilityContext.');
  }

  const config = new MsalPublicClientApplicationConfig(
    'YOUR_CLIENT_ID',
    MsalAuthority.parse('https://login.microsoftonline.com/YOUR_TENANT_ID'),
    'msalYOUR_CLIENT_ID://auth'
  );
  app = new MsalPublicClientApplication(config, context);
  return app;
}

const parameters = new MsalInteractiveTokenParameters(['User.Read']);
parameters.promptType = MsalPromptType.SELECT_ACCOUNT;

const result = await getMsalApp().acquireTokenInteractive(parameters);
```

The host ability must forward the browser redirect in **both** `onCreate` and
`onNewWant`:

```ts
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
import { MsalPublicClientApplication } from '@msal/harmony';
import { getMsalApp } from '../service/MsalService';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    this.forward(getMsalApp(this.context), want);
  }

  onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    this.forward(getMsalApp(this.context), want);
  }

  private forward(msal: MsalPublicClientApplication, want: Want): void {
    if (want.uri !== undefined && msal.handleResponse(want.uri)) {
      return;
    }
    // handle other deep links
  }
}
```

## Embedded ArkWeb

```ts
const request = await msal.createAuthorizationRequest(
  new MsalInteractiveTokenParameters(['User.Read'])
);

Web({ src: request.url, controller })
  .onLoadIntercept((event) => {
    if (msal.handleResponse(event.data.getRequestUrl())) {
      return true; // dismiss the Web component after the redirect
    }
    return false;
  });

const result = await request.result;
```

Prefer the system browser. Embedded WebViews can conflict with Conditional
Access policies.

## H5 JSBridge

The HAR does not include a container-specific WebView bridge. See
[H5 JSBridge](docs/H5_BRIDGE.md) for the recommended `msLogin` / `msLogout`
contract. Return your own backend token to H5, never the Microsoft access token.

## Security

Read [SECURITY.md](SECURITY.md). This SDK is a community preview and has not
been independently audited. Always validate access tokens on your server.

## Verify the artifact

```bash
shasum -a 256 -c SHA256SUMS
```

## License

MIT. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
