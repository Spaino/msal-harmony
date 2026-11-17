# Security policy

This SDK handles access tokens, refresh tokens, id tokens and account claims.
Treat it as security-sensitive code.

## Supported versions

| Version | Security updates |
|---|---|
| 0.1.x | Best effort, community preview |
| < 0.1 | Not supported |

## Reporting a vulnerability

**Do not open a public GitHub issue for a vulnerability.**

Use GitHub's private **Report a vulnerability** / Security Advisory flow for
this repository, or contact the maintainers through the security contact
listed in the repository profile. Include:

- affected version and HAR checksum;
- HarmonyOS / DevEco Studio / SDK version;
- a minimal reproduction;
- impact assessment;
- whether tokens, authorization codes or user data may be exposed.

Please allow a reasonable disclosure window before publishing details.

## What the implementation does

- Uses Authorization Code + PKCE (S256) for public clients.
- Does not store or ship a client secret.
- Generates `state`, `nonce` and the PKCE verifier with the platform
  cryptographic random source.
- Checks the redirect URI prefix and the `state` value before redeeming a code.
- Validates id_token `nonce`, `aud`, `azp` (when applicable) and expiry.
- Uses HTTPS for authority and token endpoints.
- Rejects extra query parameters that try to override protocol-critical fields.
- Does not log access tokens, refresh tokens, authorization codes or raw claims.
- Uses HarmonyOS Asset Store by default for the token cache.

## What it does not do

- It does not verify the id_token/JWT signature locally. Do not use the
  id_token as proof of identity on a resource server.
- It does not replace server-side authorization. A client can be tampered with.
- It does not provide device attestation, broker-backed SSO, Conditional
  Access device compliance or Intune integration.
- It has not been independently audited.

## Integrator checklist

1. Register the exact redirect URI and keep `validateAuthority` enabled.
2. Prefer the system browser. Embedded WebViews can weaken SSO and may conflict
   with Conditional Access.
3. Request the minimum scopes.
4. Keep `MsalPublicClientApplication` process-wide; never log the token cache.
5. Handle `INTERACTION_REQUIRED` by re-running interactive auth.
6. Validate every incoming deep link before forwarding it to `handleResponse`.
7. If Asset Store is unavailable on a device, provide a `MsalCacheStorage`
   backed by an enterprise key store; do not fall back to plaintext silently.
8. Treat `result.idToken` as display metadata; authorize API calls with
   `result.accessToken` and validate it on the resource server.

## H5 bridge note

If your app exposes `msLogin` to a WebView, return your **own backend token**,
not the Microsoft access token. The H5 page must not receive refresh tokens,
authorization codes or raw Microsoft tokens.
