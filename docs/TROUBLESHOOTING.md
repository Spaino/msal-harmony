# Troubleshooting

## `handleResponse()` returns false

Check, in order:

1. **Scheme mismatch** — `config.redirectUri` scheme must equal the
   `module.json5` skill scheme. Host must match too (`auth` in the default).
2. **Wrong app instance** — the object calling `handleResponse()` must be the
   same object that created the pending request. Keep it process-wide.
3. **Ability restart** — `launchType` must be `singleton`; otherwise the
   callback starts a new ability with no pending state.
4. **State mismatch** — do not start a second interactive request before the
   first one finishes.
5. **URI missing** — `want.uri` may be empty; log only its presence, never the
   full URL because it contains the authorization code.
6. **Different redirect URI** — iOS `msauth.<bundleId>://auth` is not the
   HarmonyOS default. If you reuse it, set it explicitly in config and
   `module.json5`.

## `AADSTS50011: redirect URI mismatch`

The redirect URI sent in the authorization request is not registered in the
Entra app. Register the exact string, including scheme, host and path. Custom
schemes are case-sensitive in practice; copy the value from `config.redirectUri`.

## `INTERACTION_REQUIRED`

Silent auth cannot continue (no refresh token, revoked consent, Conditional
Access, etc.). Run interactive login again. Do not loop on silent calls.

## `INVALID_GRANT`

The refresh token is invalid or expired. Remove the account and run interactive
login. Check that the app registration and tenant still allow the user.

## `INVALID_STATE`

The redirect does not belong to the pending request. Common causes:

- a stale browser redirect from an older login;
- two concurrent interactive requests;
- the app was restarted and the pending state was lost;
- the redirect URI/scheme is shared with another app.

Cancel the pending session, clear the old browser tab, and retry.

## `CACHE_ERROR` / Asset Store failure

The default cache uses Asset Store. Some devices/managed environments reject
it. Options:

- retry after device unlock;
- verify the app has not been restricted by device management;
- provide a custom `MsalCacheStorage` backed by an enterprise key store;
- use `MsalMemoryCacheStorage` for a temporary diagnostic build (tokens are
  lost on process exit).

Do not write tokens to plaintext files as a production fallback. See
[TOKEN_CACHE.md](TOKEN_CACHE.md).

## Embedded WebView is blank or blocked

- Use `createAuthorizationRequest()`, not `acquireTokenInteractive()`, for
  embedded.
- Ensure `onLoadIntercept` calls `handleResponse()` on every navigation and
  returns `true` for the matching redirect.
- Conditional Access may require a managed/system browser. Fall back to
  `acquireTokenInteractive()` and tell the user.
- Do not disable TLS or mix content.

## `INTERACTIVE_SESSION_ALREADY_RUNNING`

A previous interactive session is still pending. Call
`cancelCurrentWebAuthSession()`, close the old WebView/page, then retry.
Starting a second login without canceling the first is unsupported.

## App killed while browser is open

0.1.0 does not persist pending authorization state. If the process dies while
the browser is in the foreground, the callback cannot complete. Relaunch the app
and sign in again. The token cache (if already written) may still be valid.

## H5 `msLogin` returns null or hangs

- The native bridge action name must match what H5 calls (`msLogin`).
- The bridge must run on a valid UI ability context.
- The app version may predate the bridge; gate native login by app version.
- A previous pending login may be open; cancel it first.
- Check that the server-side token exchange succeeded; return a clear failure
  code to H5 instead of hanging.
- Never return the Microsoft access token to H5.

## Build error: cannot find `@msal/harmony`

- Check the `file:` path in `oh-package.json5` is relative to that module.
- Run `ohpm install`.
- Confirm the HAR is the release artifact and its SHA-256 matches
  `artifacts/SHA256SUMS`.

## Build error: API version

The HAR declares `compatibleSdkVersion: 17`. A project compiling with an older
API level cannot consume it. Use API 17+ (HarmonyOS 5.0.5+).

## How to collect useful logs

```ts
MsalLogger.setLogLevel(MsalLogLevel.INFO);
MsalLogger.setLogCallback((level, message, containsPii) => {
  // remove user identifiers if your log pipeline is shared
  console.info(`[MSAL] ${message}`);
});
```

Include:

- SDK/HAR version and SHA-256;
- HarmonyOS/OpenHarmony version;
- DevEco Studio and compile/target/compatible SDK;
- the `MsalError` code, `oauthError`, `correlationId`;
- **redacted** steps.

Never paste tokens, authorization codes, raw claims or client secrets into a
public issue.
