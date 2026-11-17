# Changelog

All notable changes to this distribution are documented here.

## [0.1.0] - 2026-11-17

### Added

- First community preview of the HarmonyOS MSAL HAR (`@msal/harmony@0.1.0`).
- Authorization Code + PKCE (S256) public-client login.
- System-browser flow through Implicit Want and `handleResponse(url)`.
- ArkWeb embedded flow through `createAuthorizationRequest()` and `onLoadIntercept`.
- Silent token acquisition, refresh-token renewal and multi-account cache.
- Account enumeration, removal and OIDC sign-out.
- AAD, B2C, CIAM and ADFS authorities; national cloud support.
- Asset Store token cache, memory cache and pluggable `MsalCacheStorage`.
- Error codes, correlation ids and configurable logging.
- 30 host-side protocol tests.

### Known limitations

- Community preview; not independently security-audited.
- No broker / Authenticator device SSO, POP, Intune/MAM/WPJ or Native Authentication.
- No login-session resume after process death.
- Authority metadata discovery is not implemented; use `MsalAuthorityEndpoints` when needed.
- Embedded WebView may be blocked by Conditional Access; system browser is recommended.

[0.1.0]: https://github.com/Spaino/msal-harmony/releases/tag/v0.1.0
