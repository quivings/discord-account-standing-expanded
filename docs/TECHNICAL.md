# Technical notes

How Safety Hub Expanded works, which requests it sends, and how to run its tests. For what it is and how to use it, see the [main README](../README.md).

## Session modes

The script picks a mode when it loads:

- **Active account.** No suspended token is present, so the page reads `GET /safety-hub/@me` through Discord's own REST client with your normal session.
- **Suspended session.** Discord keeps a suspended-user token in memory while the suspension screen is open. The page reads it from `AuthenticationStore` and uses the `/safety-hub/suspended/...` endpoints, which take the token in the request body.

Both kinds of session can do nearly the same things through different endpoints. The exceptions:

- Only a suspended session can tie age verification to one specific violation.
- Only an active account can reset its age-verification state.
- Google Wallet verification needs Discord's Android app.

Resetting age verification is listed under Developer but has no button, because it resets your verification state.

## Endpoints and request bodies

Request bodies match what Discord's mobile app sends, checked against [Wumpus-Central/discord-mobile-datamining](https://github.com/Wumpus-Central/discord-mobile-datamining) on September 13, 2026.

| Action | Active account | Suspended session |
| --- | --- | --- |
| Load record | `GET /safety-hub/@me` | `POST /safety-hub/suspended/@me` `{token}` |
| Appeal | `PUT /safety-hub/request-review/{id}` `{signal, user_input}` | `PUT /safety-hub/suspended/request-review/{id}` `{signal, user_input, token}` |
| List verification methods | `GET /age-verification/methods` (+ `/v2`) | `POST /age-verification/suspended/methods` (+ `/v2`) `{token}` |
| Start verification | `POST /age-verification/verify` (+ `/v2`) `{method, vendor}` | `POST /safety-hub/suspended/request-verification` `{token, from_classification_id, method}`; `/v2` `{token, method, vendor}` |
| Check status | `GET /users/@me/age-verification/check` | `POST /safety-hub/suspended/check-verification` `{token}`; `/v2` `{token, requested_at}` |
| Manual age review | `POST /age-verification/manual-review` | `POST /age-verification/suspended/manual-review` `{token}` |
| Reset age verification | `POST /users/@me/age-verification/reset` | — |

The decompiled app doesn't show where `requested_at` comes from. The page sends the ISO time you started v2 verification on the page, or the current time if you didn't. You can change it in the request runner.

The Developer page lists every known endpoint, grouped by area, and can load any of them into the request runner without sending it.

## Classification labels

The 569 classification names come from the enum Discord's clients ship. The page infers where a violation came from by the code's range (for example, 3000s are automatic instant actions and 5000s are decisions by Discord's team) plus name suffixes such as `_SAFETY_DISPATCH` and `_GUILD_ADMIN`. The ranges aren't perfectly clean: most automated-detection codes sit under 1000, and a few takedown codes sit in the 7000s internal-tools range. Treat the label as a hint.

## CAPTCHA handling

When Discord asks for hCaptcha, a **Verify to continue** prompt appears. Select the checkbox and complete the challenge; the pending request resumes automatically. **Cancel request** stops the retry, and <kbd>Esc</kbd> cancels the prompt first and closes the page on a second press.

- CAPTCHA responses on this script's requests are handled before Discord's default modal, which would otherwise open underneath the page.
- It uses the site key and `captcha_rqdata` Discord supplies, shows a visible checkbox, and raises the puzzle above the page.
- The retry resends the same method, path, and body with `X-Captcha-Key`, plus `X-Captcha-Rqtoken` and `X-Captcha-Session-Id` when Discord returns them.
- CAPTCHA tokens stay with their own request. Separate requests get separate solves, and each action allows at most three solves.
- Cancellation, late callbacks, expiry, loading errors, closing, and repasting are all handled. The prompt offers a reload when hCaptcha can't load or expires.

The request interceptor and CAPTCHA headers were checked against [Discord's public web client](https://discord.com/assets/web.f34f869e4c648af0.js) on September 13, 2026. See also hCaptcha's [configuration and callbacks](https://docs.hcaptcha.com/configuration) and [client integration and rqdata](https://github.com/hCaptcha/react-hcaptcha/blob/master/README.md).

## Tokens

The suspended token stays in memory in `window.__sh.state.token` while the page is open and is never saved or logged.

If Discord's REST client can't be found, the fallback uses plain `fetch`: suspended requests carry the token in the body, and an active session sends its own token from `AuthenticationStore` as the `Authorization` header. That token is read for each request and never stored. Requests only go to Discord's API host.

## Theme

The page reads Discord's theme variables (`--background-base-lower`, `--text-default`, `--control-primary-background-default`, and others), so it follows the user's theme. Outside Discord it falls back to the dark theme's values. Variable names were checked against Discord's web CSS on September 13, 2026.

## Console helpers

Repasting the file closes the previous copy. Once it's running:

```js
window.__sh.show("capabilities") // switch pages: overview, violations, capabilities, age, developer
window.__sh.reload()             // fetch the record again
window.__sh.close()              // close the page
window.__sh.state                // the loaded record and session mode
window.__sh.api                  // the request helpers the page uses
```

## Development

The script is standalone; there is no build step. With Node.js installed:

```text
npm install
npx playwright install chromium
npm run check
npm test
```

`npm test` runs 30 browser tests against a simulated Discord page. None of them submit real account actions.

- **15 in `tests/captcha.test.cjs`** cover CAPTCHA handling.
- **15 in `tests/hub.test.cjs`** cover the page:
  - the endpoints and bodies each session sends
  - Discord's theme colors
  - the capability comparison
  - appeals
  - the fetch fallback
  - escaping of server text
  - keyboard navigation
  - a 400px layout

Discord's internal modules and API can change, and these tests don't cover whether Discord's servers accept a request from a live account.

`npm run test:live` is an optional smoke check. It loads the real hCaptcha SDK with [hCaptcha's official test key](https://docs.hcaptcha.com/#integration-testing-test-keys) on a local page, confirms the checkbox is visible and its test response reaches a mock request, and saves a screenshot to `test-results/hcaptcha-visible.png`. It sends no Discord requests.
