# Architecture

This is a high-level description of the distributed 0.1.0 artifact. The SDK
implementation source is not part of this repository.

## Layers

```text
┌─────────────────────────────────────────────────────────────┐
│                    Host HarmonyOS app                       │
│  UIAbility · ArkUI page · H5 JSBridge · backend exchange   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│              MsalPublicClientApplication                    │
│  pending auth · state validation · account orchestration    │
└───────┬───────────────────────┬───────────────────────┬─────┘
        │                       │                       │
┌───────▼────────┐     ┌────────▼────────┐     ┌────────▼──────┐
│ OAuth client   │     │ Token cache     │     │ Crypto / PKCE │
│ URL / exchange │     │ accounts/tokens │     │ random/SHA256 │
└───────┬────────┘     └────────┬────────┘     └───────────────┘
        │                       │
┌───────▼────────┐     ┌────────▼────────┐
│ HTTP client    │     │ Cache storage   │
│ @ohos.net.http │     │ Asset Store     │
└────────────────┘     └─────────────────┘
```

## System-browser flow

1. `acquireTokenInteractive()` builds an authorization URL with:
   - `client_id`, `redirect_uri`, `response_type=code`;
   - `state`, `nonce`;
   - PKCE `code_challenge` (S256);
   - normalized scopes.
2. It opens the URL with an Implicit Want
   (`ohos.want.action.viewData`).
3. Microsoft redirects to the custom scheme.
4. The host ability forwards `want.uri` to `handleResponse()`.
5. The SDK validates the redirect/state, redeems the code with the PKCE
   verifier, validates the id_token nonce/aud/exp, and writes the token cache.
6. The pending Promise resolves with `MsalResult`.

The host must keep the application object alive between steps 2 and 5.

## Embedded flow

`createAuthorizationRequest()` performs the same URL generation and pending
state setup but does not open a browser. The host renders the URL in ArkWeb,
forwards navigations to `handleResponse()`, and awaits `request.result`.
Closing the `Web` component after the redirect is the dismissal.

## Cache model

The cache document contains accounts and credentials. The default storage is
Asset Store. Any `MsalCacheStorage` implementation can replace it.

## Trust boundaries

- The app is a **public client**; it cannot keep secrets.
- Microsoft access tokens are for calling resource APIs; they are not a
  server-side authorization decision by themselves.
- The backend must validate Microsoft tokens before issuing its own token.
- The host should validate deep links before forwarding them to the SDK.
- Embedded WebViews weaken the browser boundary and may conflict with
  Conditional Access.

## Deliberate non-goals (0.1.0)

Broker/Authenticator SSO, device SSO extensions, Intune/MAM/WPJ, POP tokens,
Native Authentication, authority metadata discovery and process-death session
resume are not implemented.
