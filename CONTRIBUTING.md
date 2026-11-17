# Contributing

This repository is a **distribution channel for the compiled HAR**, not the SDK
source repository. Source pull requests are not accepted here.

## Issues are welcome

When opening an issue, include:

- SDK version (`0.1.0`) and HAR SHA-256;
- DevEco Studio, HarmonyOS/OpenHarmony SDK and device versions;
- host project `build-profile.json5` `compileSdkVersion` / `compatibleSdkVersion`;
- expected vs actual behavior;
- **redacted** `MsalLogger` output (never include tokens, codes or claims);
- a minimal reproduction if possible.

## Code of conduct

Participation is covered by [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

## Security issues

Do not open a public issue. Follow [SECURITY.md](SECURITY.md).

## Feature requests

Feature requests are welcome. Please describe the MSAL iOS/Android behavior you
need and the HarmonyOS platform constraint. The community edition intentionally
does not implement broker/SSO, POP, Intune or Native Authentication yet.

## Documentation

Documentation fixes and integration examples can be proposed as issues or pull
requests against this repository. Do not add SDK implementation source.
