# What web push demands of a browser-only Cadence

Research for [issue #4](https://github.com/utrimud/cohort-005-project/issues/4), checked 2026-11-25. This states constraints and consequences. It does not choose the opt-in UX, the vendor, or the logout policy.

Claims link to their source. "Source code" means the behaviour was read from a browser's or framework's code, not documented as a promise. Browser code is the `main` branch, which may be ahead of the shipped release. Items I could not confirm are under [Unverified](#unverified).

## The short version

1. **On iPhone and iPad, push needs a Home Screen install.** A normal tab cannot offer it. Cadence has no web app manifest and no PNG icons today.
2. **Permission is scarce.** It needs a click in Firefox and Safari, can expire unanswered on Chrome Android, is quiet by default in Edge, and Chrome can take it away later.
3. **Every push must show an alert.** Safari unsubscribes an origin that repeatedly fails to. So on Safari, "no push while a tab is open" has to be decided before sending, not in the service worker.
4. **Subscriptions belong to a browser profile, not to a Cadence user.** Today's logout only clears a cookie, so pushes for the previous person would keep arriving.
5. **Push is best effort, and taps are not dependable on iOS.** An open WebKit bug has reports, up to iOS 27.2.1, of taps not reaching the site. The Notification Feed must be the record; push can only point to it.
6. **The client side is modest in this stack:** one hand-written, push-only worker in `public/`, a manifest, icons and a registration call. A vendor can remove most server work but none of the browser rules.

## 1. Support by browser and OS

Current stable releases on the research date: Chrome 159 ([release feed](https://chromiumdash.appspot.com/fetch_releases?channel=Stable&platform=Windows&num=2)), Firefox 161 ([product details](https://product-details.mozilla.org/1.0/firefox_versions.json)), iOS Safari 27.2 ([MDN compat data](https://github.com/mdn/browser-compat-data/blob/main/browsers/safari_ios.json)).

| Where | Push works? | Condition |
| --- | --- | --- |
| Chrome, Edge desktop | Yes | A server key is mandatory. ([compat data](https://github.com/mdn/browser-compat-data/blob/main/api/PushManager.json)) The browser process must be running. ([web.dev FAQ](https://web.dev/articles/push-notifications-faq), an older page about Chrome) |
| Chrome Android | Yes | Notifications only through a service worker; `new Notification()` throws. ([compat data](https://github.com/mdn/browser-compat-data/blob/main/api/Notification.json)) |
| Firefox desktop, Android | Yes | Click required. ([compat data](https://github.com/mdn/browser-compat-data/blob/main/api/PushManager.json)) A Windows-only helper for delivery while closed exists, but both its preferences default to false. ([Firefox source docs](https://firefox-source-docs.mozilla.org/toolkit/mozapps/pushnotificationhelper/index.html)) |
| Safari on Mac | Yes | Safari 16+ on macOS 13+, no install needed, works with Safari closed. ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers), [WebKit](https://webkit.org/blog/12945/meet-web-push/)) |
| iPhone, iPad, **in a tab** | **No** | Push is limited to Home Screen web apps, and `Notification` is undefined elsewhere. ([PushManager](https://github.com/mdn/browser-compat-data/blob/main/api/PushManager.json), [Notification](https://github.com/mdn/browser-compat-data/blob/main/api/Notification.json)) |
| iPhone, iPad, **Home Screen** | Yes, 16.4+ | See below. |
| Android and iOS WebViews | No | [compat data](https://github.com/mdn/browser-compat-data/blob/main/api/PushManager.json) |
| Chrome Incognito | No | Push is not initialised for off-the-record profiles. ([source code](https://github.com/chromium/chromium/blob/main/chrome/browser/push_messaging/push_messaging_service_impl.cc)) |

No Apple Developer Program membership is needed. ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))

### iPhone and iPad

- The person must add the site to the Home Screen and then, in that web app, tap something that asks for permission. ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
- iOS 16.4 to 18: the manifest must set `display` to `standalone` or `fullscreen` (or the page carry the web-app-capable meta tag), otherwise the icon is just a bookmark. ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
- iOS 26 and later: every site added opens as a web app by default, manifest or not, but the person can switch "Open as Web App" off. ([WebKit, Safari 26](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)) Older versions still need the manifest.
- The Safari 27.0 notes mention no change to web push. ([WebKit, Safari 27](https://webkit.org/blog/18325/webkit-features-for-safari-27-0/))
- Third-party iOS browsers can offer Add to Home Screen too; the result is the same web app. ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
- WebKit describes installation as a Share-menu action by the user; I found no API for a page to trigger it. **Consequence:** plan for Cadence to explain it, not perform it.
- Home Screen website data is isolated from Safari. ([WebKit](https://webkit.org/tracking-prevention/)) **Consequence:** expect people to sign in again inside the installed app.
- One site can be installed several times on a device; name plus manifest `id` identify each copy. ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)) **Consequence:** one person can have several subscriptions on one phone. Choose a manifest `id` once and keep it.
- Subscriptions whose Home Screen icon no longer exists are removed. ([source code](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/webpushd/PushService.mm))

## 2. Permission prompt rules

- **Click required** in Firefox (72+ desktop, 79+ Android; [compat data](https://github.com/mdn/browser-compat-data/blob/main/api/Notification.json)) and Safari, where Apple says to call subscribe *immediately* from the event handler ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)). WebKit rejects without transient activation. ([source code](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/Modules/push-api/PushManager.cpp)) **Consequence:** register the worker and fetch any key *before* the click; a slow call between click and subscribe can lose the gesture.
- **HTTPS only**, with `localhost` exempt. ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)) Testing on a real phone needs an HTTPS URL.
- **Chrome Android from 155** (scheduled stable date [2026-10-06](https://chromiumdash.appspot.com/fetch_milestone_schedule?mstone=155), so it applies now): a non-blocking prompt that can time out, resolving to `default`. The person may allow it later from Site Controls, and Chrome tells sites to watch the permission's `change` event. ([Chrome, July 2026](https://developer.chrome.com/blog/notification-prompts-android)) **Consequence:** three outcomes, not two, and permission can become `granted` with no click in Cadence at that moment.
- **Edge:** quiet requests (an address-bar icon instead of a flyout) are currently enabled for all users. ([Microsoft](https://learn.microsoft.com/en-us/microsoft-edge/web-platform/site-impacting-changes))
- **Chrome quiet prompts** apply to sites with low acceptance, or people who often block. ([Chrome](https://developer.chrome.com/blog/web-push-rate-limits)) Chromium's default temporary block is 7 days, after 3 dismissals or 4 ignores (1 or 2 under the quiet UI). ([source code](https://chromium.googlesource.com/chromium/src/+/main/components/permissions/permission_decision_auto_blocker.cc))
- **Denied is out of Cadence's hands:** `subscribe()` rejects. ([Push API](https://www.w3.org/TR/push-api/#subscribe-method)) Recovery is in browser settings, or OS settings on Apple platforms. Mac users can also turn off all site requests. ([Safari](https://support.apple.com/guide/safari/customize-website-notifications-sfri40734/mac))
- **Permission is not per Cadence account or per Notification type** (it belongs to the site in that browser). Such preferences must be stored by Cadence or its vendor.
- **Chrome can revoke it** from sites with very low engagement *and* high volume, excluding installed web apps ([Google, October 2025](https://blog.google/chromium/automatic-notification-permission/)), alongside abuse-based revocation ([Google, August 2026](https://blog.google/security/the-multi-layered-defenses-that-harden-chrome-against-abusive-notifications/)).

## 3. Service worker requirements

- The script must be same-origin, served with a JavaScript MIME type, and covers only paths at or below its own location unless `Service-Worker-Allowed` widens that. ([Service Workers](https://www.w3.org/TR/service-workers/), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register))
- One scope has one registration and one push subscription. Subscribing again with different options, such as another key, rejects with `InvalidStateError`. ([Push API](https://www.w3.org/TR/push-api/#subscribe-method)) **Consequence:** a vendor worker and a Cadence worker cannot both live at `/`. They must be combined, or use separate scopes and so separate subscriptions.
- `userVisibleOnly: true` is required by Chrome and Edge ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)) and WebKit ([source code](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/Modules/push-api/PushManager.cpp)).
- If a push shows nothing:
  - **Safari:** Apple says it revokes permission ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)); WebKit says the subscription ([WebKit](https://webkit.org/blog/12945/meet-web-push/)). Current code limits it to 3 ([source code](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/Shared/WebPushDaemonConstants.h)). Do not design to the number.
  - **Chrome:** spends a per-origin budget, then shows its own generic notification. ([source code](https://chromium.googlesource.com/chromium/src/+/main/chrome/browser/push_messaging/push_messaging_notification_manager.cc)) Edge shows a generic one too. ([Microsoft](https://learn.microsoft.com/en-us/microsoft-edge/progressive-web-apps/how-to/push))
  - **Firefox:** a quota, refreshed by visits, from which visible pushes are exempt ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)); default 16 ([source code](https://github.com/mozilla-firefox/firefox/blob/main/modules/libpref/init/all.js)).
  - **Consequence:** the push handler must show something on every path, including malformed payloads, inside `waitUntil`.
- **Worker loss:** Safari deletes service worker registrations after 7 days without interaction with the site; Home Screen apps are exempt. ([WebKit](https://webkit.org/tracking-prevention/)) WebKit says this renders a classic push subscription useless. ([WebKit](https://webkit.org/blog/16535/meet-declarative-web-push/)) **Consequence:** on Safari for Mac, an infrequent visitor can silently lose push.

### Declarative Web Push

A standard JSON payload (`"web_push": 8030` and a `notification` with `title` and `navigate`) lets the browser show and open a notification without worker code. With `mutable: true` a worker may replace it, with the original as fallback. It is in the [Push API working draft of 21 November 2026](https://www.w3.org/TR/2026/WD-push-api-20261121/).

- **WebKit:** iOS/iPadOS 18.4 and macOS 15.5 (announced in its beta). Exempt from the silent-push penalty; the subscription survives worker removal. ([WebKit](https://webkit.org/blog/16535/meet-declarative-web-push/))
- **Firefox:** fixed for 162 ([Bugzilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1935325), [Nightly notes](https://www.firefox.com/en-US/firefox/162.0a1/releasenotes/)), which is in beta, so not yet in stable 161. Firefox 158 shipped only the `navigate` option. ([MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/158))
- **Chrome:** `window.pushManager` and `navigate` listed as unsupported. ([Window](https://github.com/mdn/browser-compat-data/blob/main/api/Window.json), [Notification](https://github.com/mdn/browser-compat-data/blob/main/api/Notification.json))
- **Consequence:** a worker is still required. Sending the declarative shape everywhere, with the worker parsing the same JSON, is WebKit's compatibility advice. It is available only if the vendor allows control of the raw payload.

## 4. VAPID and the sending side

- VAPID is a P-256 key pair and a signed token with `exp` at most 24 hours ahead. ([RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)) Apple rejects a `sub` that is not a URL or `mailto:`, and asks that tokens not be refreshed more than hourly. ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
- A subscription is bound to the key that created it; replacing the key needs a **new subscription made in the browser**. ([RFC 8292, 4.2](https://www.rfc-editor.org/rfc/rfc8292#section-4.2)) **Consequence:** this is the main vendor lock-in. If the vendor will not import or export the private key, leaving means re-subscribing every browser when it next visits.
- Payloads are encrypted with `aes128gcm`. ([RFC 8291](https://www.rfc-editor.org/rfc/rfc8291)) Push services still see timing, frequency and size. ([Push API](https://www.w3.org/TR/push-api/#security-and-privacy-considerations)) A vendor doing the encryption sees the content.
- `TTL` is mandatory; `Urgency` and `Topic` are optional; `Topic` replaces an undelivered message. ([RFC 8030](https://www.rfc-editor.org/rfc/rfc8030#section-5)) Apple stores for at most 30 days. ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers))
- The endpoint is a capability URL to keep secret, and subscribing must be protected against CSRF. ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))
- **Chrome rate limits** (since January 2026): sites with high volume and little engagement are limited to no less than 1,000 messages a minute, then HTTP 429. Prompts shown per time on site count too. Penalties last 1, 7, then 14 days. ([Chrome](https://developer.chrome.com/blog/web-push-rate-limits)) **Consequence:** a new lesson fanned out to thousands could meet this only if Cadence were flagged; retries must honour 429 regardless.

## 5. Payload and action limits

- Push services must accept bodies up to 4096 bytes ([RFC 8030, 7.2](https://www.rfc-editor.org/rfc/rfc8030#section-7.2)), leaving **at most 3993 bytes of plaintext** ([RFC 8291, 4](https://www.rfc-editor.org/rfc/rfc8291#section-4)). Apple rejects over 4 KB. ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)) Mozilla warns bridged connections may allow only **2744 bytes**. ([autopush](https://mozilla-services.github.io/autopush-rs/http.html)) **Consequence:** send title, short body, URL and ID; treat about 2.5 KB, including any vendor envelope, as the ceiling.
- **Action buttons:** Chrome 2 ([source code](https://github.com/chromium/chromium/blob/main/third_party/blink/public/common/notifications/notification_constants.h)); Firefox since 152 ([MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/152)), also 2 ([source code](https://github.com/mozilla-firefox/firefox/blob/main/dom/notification/NotificationUtils.h)); **Safari none** ([compat data](https://github.com/mdn/browser-compat-data/blob/main/api/Notification.json)). **Consequence:** every push must be complete with one destination and no buttons. A Refund Request decision, for example, cannot be offered only from a button.
- `image` and `renotify` are Chromium-only; `tag` has no effect in Safari; `requireInteraction` is partial. ([compat data](https://github.com/mdn/browser-compat-data/blob/main/api/Notification.json)) **Consequence:** client-side replacement by `tag` does not reach Apple users, so grouping belongs before sending.
- No documented limit on displayed characters was found.

## 6. Expiry and renewal

- A push service may expire a subscription at any time, answered with 404. ([RFC 8030, 7.3](https://www.rfc-editor.org/rfc/rfc8030#section-7.3)) Apple documents 410 for expired tokens ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)); Mozilla uses both ([autopush](https://mozilla-services.github.io/autopush-rs/http.html)). **Consequence:** remove on 404 or 410, not on 429, 5xx or 403 (an authentication fault).
- `expirationTime` is optional ([Push API](https://www.w3.org/TR/push-api/)), and Chromium's expiry feature is off by default ([source code](https://github.com/chromium/chromium/blob/main/components/push_messaging/push_messaging_features.cc)). It cannot schedule renewal.
- `pushsubscriptionchange` is not dependable. Chrome 138 added it for re-granted permission, **with both subscriptions empty** ([Chrome 138](https://developer.chrome.com/release-notes/138)); Firefox omits those properties and iOS is listed as unsupported ([compat data](https://github.com/mdn/browser-compat-data/blob/main/api/ServiceWorkerGlobalScope.json)). A handler must not assume `oldSubscription` exists.
- **Consequence:** reconcile on page load while signed in: read permission and `getSubscription()`, report the endpoint, and re-subscribe if granted but missing. Where WebKit needs a click, that waits for one. A person who never visits has no repair path. Server-side state means "last known", never "reachable".
- Unregistering the worker deactivates a subscription, except a root-scoped one where window-level push exists. ([Push API](https://www.w3.org/TR/push-api/#subscription-deactivation))
- A 201 does not prove a person saw it: an undecryptable message is acknowledged and shows nothing. ([Push API](https://www.w3.org/TR/push-api/))

## 7. Logout and shared devices

The spec ties a subscription to a worker registration scope in the browser; it has no notion of a site user. ([Push API](https://www.w3.org/TR/push-api/)) Consequences, read against `app/lib/session.ts`, `app/routes/api.logout.ts` and `app/routes/api.switch-user.ts`:

- `destroySession` only expires `cadence_session`. Worker, permission and subscription survive, so **the previous person's Notifications keep appearing on that device after logout**, including on the Lock Screen of an iPhone ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)).
- A second person signing in gets the **same** endpoint. Rows need to be unique by endpoint and reassign the owner; a second owner would send one person's Notifications to the other. DevUI's user switch is the same event.
- The session cookie sets no `maxAge`, and a closed browser never runs logout code. **Logout cleanup alone is insufficient;** the server-side link needs its own lifetime or revalidation rule.
- Two cleanups differ: `unsubscribe()` deactivates the subscription ([Push API](https://www.w3.org/TR/push-api/#unsubscribe-method)); server-side unlinking keeps it for the next sign-in, and must work even if the browser call fails. Which, and in what order, is a spec decision.
- `POST /api/logout` is a server action and `unsubscribe()` is browser code, so a plain form post cannot do both. A `Clear-Site-Data: "storage"` response unregisters workers ([spec](https://www.w3.org/TR/clear-site-data/), supported in [current browsers](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Clear-Site-Data.json)), but also clears local storage, including the theme set in `app/root.tsx`; and a window-accessible subscription may outlive its worker.
- A push already accepted can arrive after logout. The worker cannot safely hide it because Safari penalises silence; a generic body is permitted. A short TTL narrows the window.
- Permission can persist across subscriptions ([Push API](https://www.w3.org/TR/push-api/#subscribe-method)), so the next person may be subscribed **without seeing a browser prompt**. Their consent is something Cadence must record.
- A click opens whatever session exists *then*. Destination loaders must authorise as normal.

## 8. When a Cadence tab is already open

- **Nothing in the spec suppresses the alert.** The push event fires, or the declarative notification shows, with no open-tab condition. ([Push API](https://www.w3.org/TR/push-api/)) Expect an in-app update **and** an OS alert unless the design prevents it.
- **Chrome** exempts an origin with a *visible* tab from having to show one. ([source code](https://chromium.googlesource.com/chromium/src/+/main/chrome/browser/push_messaging/push_messaging_notification_manager.cc), [web.dev](https://web.dev/articles/push-notifications-common-notification-patterns))
- **Safari** documents no exemption: "always showing a notification". ([WebKit](https://webkit.org/blog/12945/meet-web-push/)) In current code every non-declarative push goes on the silent-push timer. ([source code](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/webpushd/WebPushDaemon.mm))
- On iOS, one tester (26.6.2) reports `clients.matchAll` returning no window from a push worker even with the app visible. ([WebKit bug 268797](https://bugs.webkit.org/show_bug.cgi?id=268797), comment 40)
- **Consequence:** the options, none chosen here:
  1. always show both;
  2. suppress in the worker for visible Chrome clients, accepting both on Safari;
  3. decide before sending, using presence from the live connection (racy, and presence on a laptop says nothing about a phone);
  4. delay push and cancel it if read in the Feed.
- Each subscribed browser gets a copy. Reading in the Feed cannot remotely close alerts elsewhere, because a push that only closes another shows nothing.

### Clicking

- `notificationclick` should focus an existing window or open the URL ([web.dev](https://web.dev/articles/push-notifications-common-notification-patterns)); opening needs transient activation ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Clients/openWindow)).
- **[WebKit bug 268797](https://bugs.webkit.org/show_bug.cgi?id=268797) is open**, last updated 2026-11-13. Reporters find that taps on Home Screen apps, up to iOS 27.2.1, may only bring the app forward. Reports disagree: one traced theirs to a stale worker, one calls declarative push reliable, another saw its `navigate` dropped after suspension. These are user reports, not Apple statements.
- **Consequence:** do not depend on a tap to mark a Notification read or as the only path to an item. The Feed must make it findable wherever the app opens. Test on real devices before launch.

## 9. Fit with React Router 8 and Vite

Cadence runs React Router 8.3.0 in SSR mode, Vite 7.3.7 and `react-router-serve`, with no manifest, worker or client entry override. No build was run for this research.

- **Location:** Vite serves `public/` at `/` and copies it unhashed ([Vite](https://vite.dev/guide/assets.html#the-public-directory)), so `public/sw.js` becomes `/sw.js` with root scope: the simplest fit. Nothing transforms it, so plain JavaScript, no `~/` imports, no `import.meta.env`. Vite's worker imports are for *web* workers and emit separate chunks ([Vite](https://vite.dev/guide/features.html#web-workers)); under `/assets/` they would get the wrong scope.
- **Plugin option:** `vite-plugin-pwa` 2.0.0 declares Vite 3 to 8 ([package](https://github.com/vite-pwa/vite-plugin-pwa/blob/main/package.json)). Push alone does not need its precaching.
- **Caching:** `react-router-serve` 8.3.0 gives `/assets` one year immutable, serves the rest of the client build with no explicit max age, then `public` with one hour. ([source code](https://github.com/remix-run/react-router/blob/react-router%408.3.0/packages/react-router-serve/cli.ts)) Express's default is 0. ([Express](https://expressjs.com/en/resources/middleware/serve-static.html)) The built copy is checked first. A CDN in front must not cache the worker for long. With the default `updateViaCache: "imports"`, update checks skip the HTTP cache for the main script but not for scripts it imports. ([Service Workers](https://www.w3.org/TR/service-workers/#dom-serviceworkerregistration-updateviacache))
- **No `fetch` handler.** A push-only worker should not intercept requests, so React Router's documents and data requests behave as today.
- **Old workers persist.** Payloads need to be backward compatible, and leaving a vendor means shipping a replacement or no-op worker at the **same URL** ([Chrome](https://developer.chrome.com/docs/workbox/remove-buggy-service-workers)).
- **Registration** is browser-only: an effect or the client entry, never a loader or SSR. Permission is unknown on the server, so opt-in UI must render after hydration.
- **Routing:** static files are served before React Router (same source), so `/sw.js` and the manifest need no routes. A resource route could generate the worker, setting its own headers.
- **Manifest:** a static file linked through `links` in `app/root.tsx`, with `display: standalone`, a stable `id`, `start_url` and PNG icons; Cadence has only `favicon.svg`.
- **Installed-app UI:** it opens "like any other app" instead of in a browser ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)), so expect no browser navigation controls (not device-tested), while Cadence has a fixed sidebar and no header. Small-screen navigation becomes part of the push scope.
- **Subscribe endpoint:** take the user from the session, never from the body. React Router 8.3.0's origin check runs for document and single-fetch actions, but `handleResourceRequest` does not call it ([server.ts](https://github.com/remix-run/react-router/blob/react-router%408.3.0/packages/react-router/lib/server-runtime/server.ts)). **Consequence:** a resource route for subscriptions needs its own Origin check. Given the hard-coded session secret and a switcher that trusts a posted ID, real login (already a prerequisite) must land first.
- **TypeScript:** `tsconfig.json` uses the DOM library and includes everything; a TypeScript worker would need its own config with the WebWorker library.
- **Key to the client:** the VAPID public key can come from a loader so it differs by environment; changing it forces re-subscription.

## 10. What a push vendor can and cannot remove

General terms only, from the rules above; vendors are compared separately.

| Burden | Can a vendor take it? |
| --- | --- |
| VAPID signing, encryption, queues, retries, 429 handling, 404/410 pruning | Yes, in principle |
| Many subscriptions per person, fan-out, preferences, digests, cross-channel delay | Depends on the vendor |
| Ready-made worker and subscribe SDK | Depends; a file must still be hosted on Cadence's origin |
| HTTPS, same-origin worker, scope rules | **No** |
| iOS Home Screen install, manifest, icons, tap | **No** |
| Browser prompts, denials, revocations, Chrome's engagement limits | **No**; it can only offer a soft prompt, which is still Cadence's UX |
| 4 KB limit, no Safari buttons, always-visible rule | **No** |
| WebKit click faults, Safari's 7-day worker deletion | **No** (declarative payloads can soften the second) |
| Knowing who is signed in, and about logout | **No**; Cadence must tell it |
| Deciding who receives what, and authorising the click destination | **No** |
| Guaranteed delivery, or proof a person saw it | **No** |

Logout behaviour varies even inside one vendor: OneSignal's `logout()` unlinks the user from the subscription, and with identity verification also disables that subscription ([OneSignal](https://documentation.onesignal.com/docs/en/web-sdk-reference)). It handles VAPID but still needs the site's manifest, Home Screen instructions and a tap ([OneSignal](https://documentation.onesignal.com/docs/en/web-push-for-ios)).

Questions for each vendor: who owns the VAPID private key, and can it be imported or exported? Can our worker host theirs? Can we control the raw (declarative) payload? Does logout remove the device link server-side? Is a signed user token enforced when registering a device? Can it suppress or delay push when the Notification is read or the person is live? Does it ever send pushes that show nothing?

## Unverified

- **iOS 26+ push with no manifest at all.** The icon opens as a web app, but compat data still says a manifest is needed. Ship one.
- **`notificationclick` on iOS.** Compat data says unsupported; bug reports show it firing sometimes. Needs a device test.
- **Safari with Cadence visible.** No exemption found, but not tested; WebKit's time limit for showing an alert not found.
- **Firefox with a visible tab:** whether suppressing there uses quota.
- **Firefox closed:** Mozilla's support article could not be fetched; rollout of the Windows helper is unknown.
- **Private windows** in Firefox and Safari.
- **Samsung Internet and Opera:** compat data mirrors Chrome; not checked independently.
- **Edge mobile** prompts, and Chrome closed on desktop beyond the older web.dev FAQ.
- **`pushsubscriptionchange` in Mac Safari:** listed as supported, triggers not found.
- **Inactivity expiry at Google's push service for web subscriptions:** Firebase's documented 270 days is for Android registrations.
- **`vite-plugin-pwa` with React Router 8.3.**
- **Chrome's revocation and rate-limit thresholds** are not published, so the risk to Cadence cannot be quantified.

## Left for the map

- What logout, session expiry and user switching do to a device's subscription, and how long a stale link may live.
- Which open-tab behaviour, and whether Safari may differ.
- Whether the iOS installed app is in scope, which pulls in manifest, icons, standalone navigation and signing in again.
- What text may appear on a Lock Screen or shared device.
- How people learn that push has silently stopped.
- Who holds the VAPID key, since it sets the cost of leaving the vendor.
