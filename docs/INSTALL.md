# Installation and integration

This guide takes a new HarmonyOS app from zero to a working Microsoft sign-in
with the community HAR.

## 0. Before you start

You need:

- a Microsoft Entra app registration;
- a HarmonyOS app with a `UIAbility`;
- `ohos.permission.INTERNET`;
- DevEco Studio and a HarmonyOS/OpenHarmony SDK.

This SDK is a **public client**. Do not create or embed a client secret.

### Register the redirect URI

In [Microsoft Entra admin center](https://entra.microsoft.com/) → App registrations
→ your app → **Authentication** → **Add a platform** → **Mobile and desktop applications**:

```text
msal<clientId>://auth
```

Example:

```text
msal11111111-2222-3333-4444-555555555555://auth
```

This is the HarmonyOS default. It is **not** the iOS default
(`msauth.<bundleId>://auth`). If you already have an iOS/Android registration,
you can set `config.redirectUri` to that URI and register the same scheme here.

## 1. Add the HAR

Download the HAR from [Releases](https://github.com/Spaino/msal-harmony/releases/latest), then place it at:

```text
libs/msal-harmony-0.1.0.har
```

Add the dependency to the module that uses the SDK, for example
`entry/oh-package.json5`:

```json5
{
  "dependencies": {
    "@msal/harmony": "file:../libs/msal-harmony-0.1.0.har"
  }
}
```

The `file:` path is relative to that module's `oh-package.json5`. If the module
is at the project root, use `file:./libs/msal-harmony-0.1.0.har`.

Run:

```bash
ohpm install
```

## 2. Configure the ability

`entry/src/main/module.json5`:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
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

Why `singleton`?

The browser launches a new `Want` for the redirect. With `singleton`, the
existing ability receives it in `onNewWant`, so the same
`MsalPublicClientApplication` instance (with its pending state and PKCE
verifier) is still alive.

If your app already handles App Links, keep those skills as well. The MSAL
skill can coexist with them.

## 3. Create the MSAL singleton

```ts
// src/main/ets/service/MsalService.ets
import { common } from '@kit.AbilityKit';
import {
  MsalAuthority,
  MsalPublicClientApplication,
  MsalPublicClientApplicationConfig
} from '@msal/harmony';

const CLIENT_ID = '11111111-2222-3333-4444-555555555555';
const AUTHORITY = 'https://login.microsoftonline.com/YOUR_TENANT_ID';
const REDIRECT_URI = `msal${CLIENT_ID}://auth`;

let instance: MsalPublicClientApplication | undefined;

export function getMsalApp(context?: common.UIAbilityContext): MsalPublicClientApplication {
  if (instance !== undefined) {
    return instance;
  }
  if (context === undefined) {
    throw new Error('MsalPublicClientApplication requires a UIAbilityContext.');
  }

  const config = new MsalPublicClientApplicationConfig(
    CLIENT_ID,
    MsalAuthority.parse(AUTHORITY),
    REDIRECT_URI
  );

  // Optional: only needed for non-AAD authorities or custom endpoints.
  // config.knownAuthorities = ['https://sts.contoso.com'];

  instance = new MsalPublicClientApplication(config, context);
  return instance;
}
```

`config` is copied by the application constructor. Mutating it afterwards does
not change the initialized app.

If you use the default redirect URI, `REDIRECT_URI` can also be generated with:

```ts
MsalPublicClientApplicationConfig.defaultRedirectUri(CLIENT_ID)
```

Keep the scheme in `module.json5` in sync.

## 4. Forward the browser callback

Both `onCreate` (cold start) and `onNewWant` (already running) must forward the
URI:

```ts
// entry/src/main/ets/entryability/EntryAbility.ets
import { AbilityConstant, UIAbility, Want } from '@kit.AbilityKit';
import { MsalPublicClientApplication } from '@msal/harmony';
import { getMsalApp } from '../service/MsalService';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    const msal = getMsalApp(this.context);
    this.forwardMsal(msal, want);
  }

  onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    const msal = getMsalApp(this.context);
    this.forwardMsal(msal, want);
  }

  private forwardMsal(msal: MsalPublicClientApplication, want: Want): void {
    if (want.uri !== undefined && want.uri.length > 0) {
      if (msal.handleResponse(want.uri)) {
        return;
      }
      // Not an MSAL response: continue with AppLink / other deep links.
    }
  }
}
```

`handleResponse()` returns `true` only when the URI matches this app's pending
MSAL session and was consumed. Do not call `handleResponse()` on arbitrary
user-provided URLs without checking the prefix; the SDK validates state, but the
host should still avoid forwarding unrelated links.

## 5. Sign in

```ts
import { MsalInteractiveTokenParameters, MsalPromptType } from '@msal/harmony';
import { getMsalApp } from '../service/MsalService';

const msal = getMsalApp();
const parameters = new MsalInteractiveTokenParameters(['User.Read']);
parameters.promptType = MsalPromptType.SELECT_ACCOUNT;

const result = await msal.acquireTokenInteractive(parameters);
console.info(`signed in as ${result.account.username}`);
```

`acquireTokenInteractive()` opens the system browser. The Promise resolves only
after the ability forwards the redirect URI to `handleResponse()`.

## 6. Silent token and accounts

```ts
import { MsalSilentTokenParameters } from '@msal/harmony';

const accounts = await msal.allAccounts();
const account = accounts.length > 0 ? accounts[0] : undefined;

if (account !== undefined) {
  try {
    const silent = new MsalSilentTokenParameters(['User.Read'], account);
    const result = await msal.acquireTokenSilent(silent);
  } catch (error) {
    // INTERACTION_REQUIRED / INVALID_GRANT -> interactive login
  }
}
```

`allAccounts()` only returns the local cache. There is no device-wide broker or
SSO extension on HarmonyOS.

## 7. Sign out

```ts
import { MsalSignoutParameters } from '@msal/harmony';

const account = await msal.accountForUsername('user@contoso.com');
if (account !== undefined) {
  const params = new MsalSignoutParameters();
  params.signoutFromBrowser = true;
  await msal.signoutWithAccount(account, params);
}
```

`signoutFromBrowser = true` opens the OIDC end-session endpoint and clears the
browser session cookie. Set it to `false` if your product uses a separate
browser session policy.

## 8. Verify

1. Tap sign-in.
2. Browser opens the Microsoft login page.
3. Complete sign-in.
4. App receives the custom-scheme URI.
5. `handleResponse()` returns `true`.
6. The original `acquireTokenInteractive()` Promise resolves.

If step 4 fails, check:

- the scheme/host in `module.json5`;
- `config.redirectUri` matches exactly;
- the ability is `launchType: singleton`;
- the same app instance is used in `onNewWant`;
- the redirect URI is registered in Entra.

## 9. Next steps

- [Embedded ArkWeb](EMBEDDED_WEBVIEW.md)
- [Token cache](TOKEN_CACHE.md)
- [H5 JSBridge](H5_BRIDGE.md)
- [Troubleshooting](TROUBLESHOOTING.md)
