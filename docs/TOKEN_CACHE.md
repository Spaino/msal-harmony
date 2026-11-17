# Token cache

The SDK caches access tokens, refresh tokens, id tokens, account metadata and
claims. Treat the cache as a secret.

## Default: Asset Store

`MsalAssetCacheStorage` is the default when `config.cacheConfig.storage` is
omitted:

```ts
const config = new MsalPublicClientApplicationConfig(clientId);
const app = new MsalPublicClientApplication(config, context);
```

Properties:

- stored in HarmonyOS Asset Store;
- protected by the device's credential-encrypted storage;
- accessible only to the app;
- readable after first device unlock (`DEVICE_FIRST_UNLOCKED`);
- not written to ordinary files or Preferences.

## Memory cache

For tests and short-lived processes:

```ts
import { MsalMemoryCacheStorage } from '@msal/harmony';

config.cacheConfig.storage = new MsalMemoryCacheStorage();
```

The cache is lost when the process exits.

## Custom cache

```ts
import { MsalCacheStorage } from '@msal/harmony';

export class EnterpriseCacheStorage implements MsalCacheStorage {
  async read(): Promise<string | undefined> {
    // return the serialized cache document, or undefined
  }

  async write(data: string): Promise<void> {
    // encrypt and persist data
  }

  async clear(): Promise<void> {
    // remove the persisted document
  }
}
```

Set it before creating the application:

```ts
config.cacheConfig.storage = new EnterpriseCacheStorage();
```

The value passed to `write()` is the serialized cache document. Do not log it.

## Asset Store failures

Some devices or managed environments may reject Asset Store writes. The default
storage throws `MsalErrorCode.CACHE_ERROR`. Applications can implement a custom
`MsalCacheStorage` that:

- tries Asset Store;
- falls back to an app-private encrypted file/key store;
- logs only the error code, never the cache content.

Do not silently fall back to plaintext storage for production. If you use a
file fallback, encrypt it with a key protected by the platform keystore and
document the threat model.

## Multiple accounts

The cache supports multiple accounts. Use:

```ts
const accounts = await app.allAccounts();
const account = await app.accountForIdentifier(identifier);
```

`account.identifier` is the stable home account id. Persist it if you need to
restore a specific account after an app restart.

## Removing data

```ts
await app.removeAccount(account);
```

This removes the account and its tokens from the local cache. It does not
delete browser cookies. Use `signoutWithAccount()` with
`signoutFromBrowser = true` for an OIDC end-session that clears the browser
session.

## Process death

0.1.0 has no login-session resume. If HarmonyOS kills the process while the
browser is in the foreground, the pending `state` / PKCE verifier is lost and
the callback cannot complete. The user must retry. The token cache itself
survives if Asset Store/custom storage succeeded.
