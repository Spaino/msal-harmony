# MSAL for HarmonyOS v0.1.0 — Community Preview

First public, **artifact-only** release of the community HarmonyOS MSAL SDK.

## Download

- `msal-harmony-0.1.0.har`
- `msal-harmony-0.1.0.zip` (HAR + docs + checksums)

## Highlights

- Authorization Code + PKCE (S256) for public clients
- System browser and ArkWeb embedded login
- Silent token acquisition and refresh-token renewal
- Multi-account token cache with Asset Store default
- AAD, B2C, CIAM and ADFS authorities
- OIDC sign-out, error codes, correlation IDs and configurable logging
- 30 host-side protocol tests

## Compatibility

- Runtime: HarmonyOS / OpenHarmony API 17+
- HAR compile target: OpenHarmony SDK 20
- Validated with DevEco Studio 26.0.0 and a HarmonyOS 6 device

## Known limitations

- Community preview; not independently security-audited.
- No broker/Authenticator SSO, POP, Intune/MAM/WPJ or Native Authentication.
- No login-session resume after process death.
- Embedded WebView may be blocked by Conditional Access; prefer system browser.

## Verify

```bash
shasum -a 256 -c SHA256SUMS
```

## License / disclaimer

MIT. This is an unofficial community project, not affiliated with or endorsed
by Microsoft Corporation. Microsoft, Microsoft Entra and MSAL are trademarks of
Microsoft Corporation. See `LICENSE` and `NOTICE`.

## Checksum

```text
HAR SHA-256: 3ba43f0a96c5dccc6458f3b82ae9f8933b1481f162e60b49ddb8df1b50cb1eac
```

The release also publishes a `SHA256SUMS` file covering the HAR and ZIP.
