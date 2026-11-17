# API reference (0.1.0)

All imports come from the HAR package:

```ts
import {
  MsalPublicClientApplication,
  MsalPublicClientApplicationConfig,
  MsalAuthority,
  MsalAccount,
  MsalResult,
  MsalInteractiveTokenParameters,
  MsalSilentTokenParameters,
  MsalSignoutParameters,
  MsalCacheStorage,
  MsalAssetCacheStorage,
  MsalMemoryCacheStorage,
  MsalLogger,
  MsalError,
  MsalErrorCode
} from '@msal/harmony';
```

## MsalPublicClientApplication

```ts
constructor(config: MsalPublicClientApplicationConfig, context?: common.UIAbilityContext)

allAccounts(): Promise<MsalAccount[]>
accountForIdentifier(identifier: string): Promise<MsalAccount | undefined>
accountForUsername(username: string): Promise<MsalAccount | undefined>
accountsForParameters(params: MsalAccountEnumerationParameters): Promise<MsalAccount[]>

acquireTokenInteractive(params: MsalInteractiveTokenParameters): Promise<MsalResult>
createAuthorizationRequest(params: MsalInteractiveTokenParameters): Promise<MsalAuthorizationRequest>
handleResponse(uri: string): boolean
cancelCurrentWebAuthSession(): boolean

acquireTokenSilent(params: MsalSilentTokenParameters): Promise<MsalResult>

removeAccount(account: MsalAccount): Promise<void>
signoutWithAccount(account: MsalAccount, params: MsalSignoutParameters): Promise<void>
```

`createAuthorizationRequest()` returns:

```ts
class MsalAuthorizationRequest {
  readonly url: string;
  readonly state: string;
  readonly result: Promise<MsalResult>;
}
```

Only one interactive request may be pending at a time. A second request fails
with `INTERACTIVE_SESSION_ALREADY_RUNNING`.

## MsalPublicClientApplicationConfig

```ts
constructor(
  clientId: string,
  authority?: MsalAuthority,
  redirectUri?: string
)

clientId: string
redirectUri: string
authority: MsalAuthority
knownAuthorities: string[]
validateAuthority: boolean
extendedLifetimeEnabled: boolean
tokenExpirationBuffer: number
multipleCloudsSupported: boolean
bypassRedirectURIValidation: boolean
cacheConfig: MsalCacheConfig
endpoints?: MsalAuthorityEndpoints
extraQueryParameters: Map<string, string>
sliceParameters: Map<string, string>
```

Default authority: `https://login.microsoftonline.com/common`.
Default redirect URI: `msal<clientId>://auth`.

`cacheConfig.storage` accepts a `MsalCacheStorage`.

## MsalAuthority

```ts
static parse(url: string): MsalAuthority
static aad(cloud: MsalCloudInstance, audience: MsalAudienceType, tenant?: string): MsalAuthority

getAuthorizationEndpoint(): string
getTokenEndpoint(): string
getEndSessionEndpoint(): string
getRealm(): string
getAuthorityForTenant(tenant: string): MsalAuthority
```

`MsalAuthorityEndpoints` can override endpoints for ADFS or custom deployments.

## Parameters

### MsalInteractiveTokenParameters

```ts
constructor(scopes: string[], webviewParameters?: MsalWebviewParameters)

scopes: string[]
claimsRequest?: MsalClaimsRequest
correlationId?: string
extraQueryParameters: Map<string, string>
promptType: MsalPromptType
loginHint?: string
domainHint?: string
extraScopesToConsent: string[]
```

`MsalPromptType`: `SELECT_ACCOUNT`, `LOGIN`, `CONSENT`, `CREATE`,
`PROMPT_IF_NECESSARY`.

### MsalSilentTokenParameters

```ts
constructor(scopes: string[], account: MsalAccount)

forceRefresh: boolean
claimsRequest?: MsalClaimsRequest
correlationId?: string
extraQueryParameters: Map<string, string>
```

### MsalSignoutParameters

```ts
constructor(webviewParameters?: MsalWebviewParameters)

signoutFromBrowser: boolean // default true
```

## MsalResult

```ts
accessToken: string
expiresOn: Date
extendedLifeTimeToken: boolean
idToken: string
scopes: string[]
tenantProfile: MsalTenantProfile
account: MsalAccount
authority: MsalAuthority
correlationId: string
authenticationScheme: string
authorizationHeader: string
```

Use `accessToken` for API calls. `idToken` is display/identity metadata; do not
treat it as a verified identity on the server.

## MsalAccount

```ts
username: string
environment: string
homeAccountId: MsalAccountId
tenantProfiles: MsalTenantProfile[]
accountClaims: Map<string, Object>
identifier: string
```

`identifier` is the stable `homeAccountId` string to persist for silent calls.

## Cache storage

```ts
interface MsalCacheStorage {
  read(): Promise<string | undefined>
  write(data: string): Promise<void>
  clear(): Promise<void>
}

class MsalAssetCacheStorage implements MsalCacheStorage {
  constructor(clientId: string)
}

class MsalMemoryCacheStorage implements MsalCacheStorage {}
```

## Logging

```ts
enum MsalLogLevel { NOTHING, ERROR, WARNING, INFO, VERBOSE }

MsalLogger.setLogLevel(level: MsalLogLevel): void
MsalLogger.setLogCallback(callback: MsalLogCallback | undefined): void
```

The SDK never logs tokens, codes or raw claims. Do not add them in a callback.

## Errors

`MsalError` extends `Error` and exposes:

```ts
code: number
oauthError?: string
oauthSubError?: string
errorDescription?: string
httpStatusCode?: number
correlationId?: string
internalErrorCode?: number
responseBody?: string
cause?: string
```

Important codes:

| Code | Meaning |
|---|---|
| `INTERACTION_REQUIRED` | Silent auth cannot continue; run interactive |
| `USER_CANCELED` | User canceled/timed out |
| `INVALID_GRANT` | Refresh token invalid |
| `INVALID_STATE` | Redirect state mismatch |
| `INVALID_RESPONSE` | Token/id_token malformed |
| `NETWORK_ERROR` | HTTP/network failure |
| `FAILED_AUTHORITY_VALIDATION` | Authority not trusted |
| `CACHE_ERROR` | Cache read/write failure |
| `INTERACTIVE_SESSION_ALREADY_RUNNING` | Another pending session |
| `NO_BROWSER` | No ability/browser available |
| `WEBVIEW_NOT_SUPPORTED` | System-browser API used for embedded mode |

See `MsalErrorCode` in the declarations for the full list.
