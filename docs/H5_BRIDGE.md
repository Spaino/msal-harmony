# H5 JSBridge: msLogin / msLogout

The HAR implements native MSAL/OAuth. It does **not** contain a specific
WebView bridge. The host app decides how H5 invokes native login and what token
it returns.

This page describes the contract used by the reference integration.

## Recommended architecture

```text
H5 WebView
  └─ msLogin
       ├─ native MSAL interactive/silent
       ├─ Microsoft access token
       ├─ POST your backend (exchange Microsoft token)
       └─ return your backend token to H5
```

Never return the Microsoft access token or refresh token to H5. The H5 page
only needs the token for **your** API.

## Native side

### msLogin

```ts
async function msLoginForH5(): Promise<H5LoginResult> {
  const msal = getMsalApp();

  let microsoftAccessToken = '';
  const accounts = await msal.allAccounts();

  if (accounts.length > 0) {
    try {
      const silent = new MsalSilentTokenParameters(['User.Read'], accounts[0]);
      const result = await msal.acquireTokenSilent(silent);
      microsoftAccessToken = result.accessToken;
    } catch (error) {
      // INTERACTION_REQUIRED / INVALID_GRANT: fall through to interactive
    }
  }

  if (microsoftAccessToken.length === 0) {
    const interactive = new MsalInteractiveTokenParameters(['User.Read']);
    interactive.promptType = MsalPromptType.SELECT_ACCOUNT;
    const result = await msal.acquireTokenInteractive(interactive);
    microsoftAccessToken = result.accessToken;
  }

  // Server-side exchange; do not put this endpoint or logic in H5.
  const backendToken = await exchangeMicrosoftTokenWithYourBackend(microsoftAccessToken);

  return {
    code: '0',
    payload: { accessToken: backendToken }
  };
}
```

`exchangeMicrosoftTokenWithYourBackend()` is your own HTTPS API. It takes the
Microsoft access token, validates it server-side, maps the Microsoft identity
to your user, and returns your app's token. Do not trust the client for
authorization decisions.

### msLogout

```ts
async function msLogoutForH5(): Promise<H5LogoutResult> {
  const msal = getMsalApp();
  const accounts = await msal.allAccounts();

  for (const account of accounts) {
    const params = new MsalSignoutParameters();
    params.signoutFromBrowser = false; // keep the app in control
    await msal.signoutWithAccount(account, params);
  }

  await clearYourBackendToken();

  return { code: '0' };
}
```

Set `signoutFromBrowser = true` only if your product explicitly wants to open
the OIDC logout page in the system browser.

## H5 side

Using `@ef-online/ef-js-bridge` as an example:

```ts
import { efJsBridge } from '@ef-online/ef-js-bridge';

const token = await efJsBridge.msLogin();
if (token) {
  setApiToken(token);
}

await efJsBridge.msLogout();
clearApiToken();
```

The actual bridge wrapper can differ. The native response shape should be:

```json
{
  "code": "0",
  "message": "ok",
  "payload": {
    "accessToken": "<your backend token>"
  }
}
```

Failure:

```json
{
  "code": "500",
  "message": "Microsoft login failed"
}
```

The H5 wrapper should read `payload.accessToken` and also tolerate a root
`accessToken` for backward compatibility:

```ts
const token = data?.accessToken || data?.payload?.accessToken;
```

## Bridge registration

The HarmonyOS app should expose the action names that the H5 bridge library
expects, for example `msLogin` and `msLogout`. Keep the native classes
registered in a module that is included in the HAP. The bridge method should:

- run on the UI ability context;
- reuse a process-wide `MsalPublicClientApplication`;
- serialize concurrent calls (the SDK allows only one pending interactive
  session);
- map `USER_CANCELED` to a user-friendly cancel code;
- never log tokens/codes.

## App-version gating

If H5 is served remotely, old app versions may not have the native bridge yet.
Gate the native path by app version, for example:

```ts
const nativeLoginEnabled =
  isInApp() &&
  isAppVersionAtLeast('1.1.22') &&
  isHarmonyOS();
```

Older versions should continue using the existing web login flow. Do not call
`msLogin` on an app build that predates the bridge; the call will fail or hang.

## Cache and logout semantics

- `msLogin` should first reuse a cached backend token if one is still valid.
- If the backend token is missing/expired but MSAL has a refresh token, try
  silent login before opening the Microsoft UI.
- `msLogout` clears MSAL accounts and your backend token. If the app also has a
  C-side/student session, decide explicitly whether teacher logout should clear
  it; do not mix the two token types.
- When the user switches roles (teacher ↔ student), preserve or clear the
  teacher backend token according to your product's session model. The SDK
  itself does not know about roles.

## Security checklist

- H5 receives only your backend token.
- The backend validates the Microsoft access token (signature, issuer,
  audience, scope, expiry) before issuing its own token.
- Do not put client secrets in the H5 or the app.
- Do not log authorization codes, tokens or raw claims.
- Use HTTPS for the exchange endpoint.
