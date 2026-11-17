# Build provenance

## Artifact

| Field | Value |
|---|---|
| Package | `@msal/harmony` |
| Version | `0.1.0` |
| File | `artifacts/msal-harmony-0.1.0.har` |
| Build mode | `release` (`debug: false`) |
| HAR format | gzip-compressed tar, byte-code HAR |
| Compile SDK | OpenHarmony SDK 20 |
| `compatibleSdkVersion` | 17 |
| Runtime | HMOS/OpenHarmony API 17+ |
| Build date | 2026-11-17 |
| Source maps in HAR | no |

The HAR contains:

- `package/ets/modules.abc` — compiled ArkTS bytecode;
- `package/src/main/ets/**/*.d.ets` — public declarations;
- `package/Index.d.ets`;
- `package/oh-package.json5`;
- no SDK implementation `.ets` source, tests or demo app.

## Verification

- 30 host-side protocol tests passed (`30 passed, 0 failed`).
- The HAR was consumed by a HarmonyOS 6 app; Microsoft login, browser callback,
  token exchange and H5 bridge were exercised on a real device.
- The release HAR is built with `buildMode=release`.
- SHA-256 is in `artifacts/SHA256SUMS` and in the GitHub release notes.

## Reproducibility

This is a community artifact build, not a verified reproducible build. The
private source tree and build toolchain versions are not published. If you need
strong supply-chain guarantees, build from source under your own control or
request a reproducible-build recipe.

## Integrity

```bash
shasum -a 256 -c artifacts/SHA256SUMS
```

Do not trust a HAR from an unofficial mirror without checking its checksum and
the release signature/notes.
