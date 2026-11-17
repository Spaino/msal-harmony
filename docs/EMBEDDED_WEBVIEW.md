# Embedded ArkWeb login

The system browser is the recommended and most compatible flow. Use the
embedded flow only when the product requires the Microsoft page inside the app.

## Why a separate API

`acquireTokenInteractive()` opens the system browser through an Implicit Want.
It cannot drive an ArkWeb `Web` component. For embedded login, use:

```ts
createAuthorizationRequest(params): Promise<MsalAuthorizationRequest>
```

This returns the authorization URL and a Promise for the eventual result. The
host renders the URL in `Web`, intercepts the redirect, and calls
`handleResponse()`.

## Complete pattern

```ts
import { webview } from '@kit.ArkWeb';
import { MsalInteractiveTokenParameters } from '@msal/harmony';

@Entry
@Component
struct LoginPage {
  @State authUrl: string = '';
  private controller: webview.WebviewController = new webview.WebviewController();
  private request?: MsalAuthorizationRequest;

  async start(): Promise<void> {
    const msal = getMsalApp();
    this.request = await msal.createAuthorizationRequest(
      new MsalInteractiveTokenParameters(['User.Read'])
    );
    this.authUrl = this.request.url;

    try {
      const result = await this.request.result;
      this.authUrl = ''; // dismiss the Web component
      // use result.accessToken
    } catch (error) {
      this.authUrl = '';
      // map MsalError to UI
    }
  }

  build() {
    Column() {
      if (this.authUrl.length > 0) {
        Web({ src: this.authUrl, controller: this.controller })
          .width('100%')
          .height('100%')
          .domStorageAccess(true)
          .onLoadIntercept((event) => {
            const url = event.data.getRequestUrl();
            if (getMsalApp().handleResponse(url)) {
              this.authUrl = ''; // auto-dismiss
              return true;      // block Web navigation to the custom scheme
            }
            return false;
          })
      } else {
        Button('Microsoft 登录').onClick(() => this.start())
      }
    }
  }
}
```

The exact ArkUI layout is up to the host. The critical pieces are:

1. `createAuthorizationRequest()` creates the pending state and URL.
2. `Web({ src: request.url })` renders Microsoft's page.
3. `onLoadIntercept` forwards every navigation URL to `handleResponse()`.
4. `handleResponse()` returns `true` for the matching redirect; return `true`
   from `onLoadIntercept` so the Web component does not try to navigate to the
   custom scheme.
5. Closing/hiding the `Web` component is the "dismiss".
6. Await `request.result` to get the token result or error.

## Cancel and retry

```ts
const cancelled = getMsalApp().cancelCurrentWebAuthSession();
```

`cancelCurrentWebAuthSession()` returns `true` if a pending session existed.
Call it from a close button and from page teardown so a half-open request does
not block the next attempt.

If the user closes the page without canceling, the SDK eventually times out.
`INTERACTIVE_SESSION_ALREADY_RUNNING` on the next attempt is a sign that the
previous session is still pending.

## Conditional Access

Embedded WebViews may be rejected by Conditional Access policies that require a
managed browser, device compliance or SSO. Handle the failure and offer a
system-browser fallback:

```ts
const fallback = new MsalInteractiveTokenParameters(['User.Read']);
const result = await getMsalApp().acquireTokenInteractive(fallback);
```

Do not silently switch to the system browser without telling the user; the
experience changes from an in-app page to the browser.

## Security notes

- Do not inject credentials or scripts into Microsoft's login page.
- Do not log the authorization URL (it contains `state` and PKCE challenge).
- Do not log the redirect URL (it contains the authorization code).
- Prefer system browser for production; embedded mode weakens the browser
  security boundary.
