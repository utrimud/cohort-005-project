# Hosted notification vendors for Cadence

Research for [issue #3](https://github.com/utrimud/cohort-005-project/issues/3). All sources were read on **2026-11-25**. Sources are vendors' own docs, pricing pages, npm registry records and status pages. Nothing was tested and no account was created.

**Unverified** means no current primary source was found. It does not mean the vendor lacks the feature.

This is a shortlist, not a choice.

## Answer

Shortlist: **Knock**, **Novu Cloud**, **SuprSend**. All three offer a feed API, headless React hooks, live updates, per-user signing on the server, Postmark, preferences, batching, idempotency keys and user deletion.

| | Strongest reason | Main cost |
| - | - | - |
| **Knock** | Most completely documented: token refresh, delivery retries, per-run sandbox, indefinite feed, full user deletion | $250/month after the free tier; no first-party web push (needs Firebase); EU data residency only "expected by the end of 2026" |
| **Novu Cloud** | Cheapest at 10,000 recipients, by a wide margin; US or EU on every plan | Feed items expire after 30 or 90 days; per-user hash has no expiry; no first-party web push; Free plan hard-stops |
| **SuprSend** | First-party VAPID web push, with no Firebase project; expiring signed tokens | Batching and preferences need the $250/month plan; custom data region needs Enterprise; several limits undocumented |

Courier and MagicBell could work but did not make the shortlist. OneSignal and Engagespot are dropped. See [Not shortlisted](#not-shortlisted).

## Findings that shape the spec

1. **Web push divides the field.** SuprSend and MagicBell send browser push themselves using VAPID keys. Knock, Novu and Courier document pushing through a provider such as Firebase Cloud Messaging (FCM). Knock says it plainly: it "does not have a first-party web push channel" ([Knock FCM tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm)). Every option still needs Cadence to serve a service worker and ask permission.
2. **Billing units differ, so list prices do not compare directly.** Knock, SuprSend and MagicBell bill each channel separately. Courier bills one unit per recipient per send, whatever the channels. Novu bills one unit per workflow run per recipient, and bills events that are later merged into a digest ([Novu trigger](https://docs.novu.co/platform/concepts/trigger)).
3. **Feed retention decides who owns the feed.** Knock's is indefinite, Courier's is one year, Novu's is 30 or 90 days. If Notifications must outlive that, Cadence needs its own copy.
4. **Signed tokens are opt-in.** Knock's and SuprSend's enforcement is off until enabled; Novu's is a dashboard toggle. The spec must require turning it on in production.
5. **Idempotency keys last about a day** for every candidate (Courier's can be extended to a year). They cover retries, not late replays or a backfill, so Cadence needs its own durable record of what was sent.
6. **No vendor was found to offer an offline emulator.** Keyless local development and tests need a Cadence-side transport, like the `.outbox/` one in the current mailer.
7. **Nobody's React UI is documented for React Router 8.** Peer ranges allow React 19 for Knock, Novu, Courier and SuprSend. Treat prebuilt components as client-only, and prove SSR in a prototype.

## Comparison

### Feed, state and live delivery

| | Feed API and React | Read and seen | Live delivery | React 19 and SSR |
| - | - | - | - | - |
| **Knock** | Feed API, prebuilt components and headless hooks ([feeds](https://docs.knock.app/in-app-ui/feeds/overview), [React](https://docs.knock.app/in-app-ui/react/overview)) | Separate read, seen, archived; unread and unseen counts ([API](https://docs.knock.app/in-app-ui/api-overview)) | Managed WebSocket with reconnection ([sockets](https://docs.knock.app/in-app-ui/feeds/socket-behavior)) | Peer range includes React 19 ([npm](https://www.npmjs.com/package/@knocklabs/react)); `next` is an optional peer. React Router SSR unverified |
| **Novu** | `<Inbox />` plus headless `useNotifications` and `useCounts` ([headless](https://docs.novu.co/platform/inbox/headless-mode)) | Read, archived; unread and unseen counts (same page) | Hooks update in real time; WebSocket host per region ([security](https://docs.novu.co/platform/additional-resources/security)) | Supports 18 and 19 ([setup](https://docs.novu.co/platform/inbox/setup-inbox)); has a [Remix quickstart](https://docs.novu.co/platform/quickstart/remix). React Router 8 SSR unverified |
| **SuprSend** | Drop-in and headless React ([overview](https://docs.suprsend.com/docs/react-inbox-integration)) | Read, seen, archive ([headless](https://docs.suprsend.com/docs/react-sdk-headless-feed)) | Socket, with an optional connection-status hook (same page) | Peer `>=16.8` ([npm](https://www.npmjs.com/package/@suprsend/react)); SSR unverified |
| **Courier** | Inbox components and `useCourier` hook ([custom inbox](https://www.courier.com/docs/in-app/build-a-custom-inbox)) | Read, opened, archived ([behaviour](https://www.courier.com/docs/in-app/how-the-inbox-behaves)); a separate seen state unverified | Realtime connection, synced across devices (same page) | Peer `>=18` ([npm](https://www.npmjs.com/package/@trycourier/courier-react)); "render client-side only" ([React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web)) |
| **MagicBell** | Inbox and floating inbox ([React](https://www.magicbell.com/docs/libraries/react)); headless package on [npm](https://www.npmjs.com/package/@magicbell/react-headless), its relation to API v2 unverified | Read, seen, archived ([notification](https://www.magicbell.com/docs/primitive/notification)) | Persistent connection ([in-app](https://www.magicbell.com/docs/channels/in_app)) | Headless peer is `>=18.3.1`; main package lists no peers. SSR unverified |

### Web push and email

| | Web push | Still needed | Email through Postmark |
| - | - | - | - |
| **Knock** | Through FCM ([tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm)) or [OneSignal](https://docs.knock.app/integrations/push/one-signal) | Firebase project, Firebase SDK, VAPID key, service worker, token registration | Yes ([Postmark](https://docs.knock.app/integrations/email/postmark)); sandbox mode and payload overrides |
| **Novu** | Through FCM ([FCM](https://docs.novu.co/platform/integrations/push/fcm)), [Pusher Beams](https://docs.novu.co/platform/integrations/push/pusher-beams) or [OneSignal](https://docs.novu.co/platform/integrations/push/onesignal); or [Push Webhook](https://docs.novu.co/platform/integrations/push/push-webhook) to Cadence's own endpoint | A push provider account, or Cadence sending VAPID push itself; service worker and token registration | Yes ([Postmark](https://docs.novu.co/platform/integrations/email/postmark)); choosing a message stream unverified |
| **SuprSend** | First-party, VAPID ([React web push](https://docs.suprsend.com/docs/react-webpush)); no intermediary provider ([pricing](https://www.suprsend.com/pricing)) | Service worker at site root importing SuprSend's script from jsDelivr; `registerPush()` on a user action | Yes, server token ([Postmark](https://docs.suprsend.com/docs/postmark)) |
| **Courier** | [FCM](https://www.courier.com/docs/integrations/push/firebase-fcm) and others. A search of the full docs found no browser push guide; browser token handling unverified | FCM or similar, service worker, and token upload through the [token API](https://www.courier.com/docs/send/push/overview) | Yes, with message-stream override ([Postmark](https://www.courier.com/docs/integrations/email/postmark)) |
| **MagicBell** | First-party, VAPID ([web push](https://www.magicbell.com/docs/channels/web_push)), with a React button | VAPID key pair; service worker importing MagicBell's script ([React](https://www.magicbell.com/docs/libraries/react)) | No named integration; only through generic [SMTP](https://www.magicbell.com/docs/channels/email/smtp). [Postmark offers SMTP](https://postmarkapp.com/smtp-service); the pair was not tested |

Firebase lists Cloud Messaging as a no-cost product ([pricing](https://firebase.google.com/pricing)). Safari behaviour of each vendor's web push SDK is unverified. On iPhone and iPad, web push works only for a site added to the Home Screen ([OneSignal's iOS guide](https://documentation.onesignal.com/docs/en/web-push-for-ios)).

### Preferences and signed identity

| | Preferences | Per-user signing |
| - | - | - |
| **Knock** | By channel, category and workflow; workflow override for mandatory messages ([preferences](https://docs.knock.app/preferences/overview)) | RS256 JWT with `exp`, limited grants, and `onUserTokenExpiring` refresh ([security](https://docs.knock.app/in-app-ui/security-and-authentication)). Off by default |
| **Novu** | Global, per workflow and channel; critical workflows cannot be turned off ([preferences](https://docs.novu.co/platform/concepts/preferences)) | HMAC-SHA256 of the subscriber ID ([production](https://docs.novu.co/platform/inbox/prepare-for-production)). The documented hash input carries no expiry; per-user revocation short of changing the secret key is unverified |
| **SuprSend** | Channel and category; **Business plan and up** ([pricing](https://www.suprsend.com/pricing)) | ES256 JWT with `exp` and a refresh callback ([client auth](https://docs.suprsend.com/docs/client-authentication)). Enhanced security mode must be enabled |
| **Courier** | Per topic, with `REQUIRED` topics; hosted or embedded centre ([preferences](https://www.courier.com/docs/recipients/preferences/overview)) | JWT issued by Courier's API, scoped to user. Never expires unless `expires_in` is set; SDK does not refresh it ([auth](https://www.courier.com/docs/in-app/authenticate-users)) |
| **MagicBell** | Channel preferences and topic subscriptions ([user](https://www.magicbell.com/docs/primitive/user)) | HS256 JWT; docs example uses one year ([user JWT](https://www.magicbell.com/docs/api/authentication/user)) |

Courier's issuing step is a network call to Courier; the other four sign locally.

### Reliability and fan-out

| | Idempotency | Delivery retries | Limits | Fan-out | Batching |
| - | - | - | - | - | - |
| **Knock** | `Idempotency-Key` on trigger, 24 h ([docs](https://docs.knock.app/api-reference/overview/idempotent-requests)) | Up to 8 over 30 min ([delivery](https://docs.knock.app/send-notifications/delivering-notifications)) | Trigger is [tier 5](https://docs.knock.app/api-reference/workflows/trigger): the [1,000/s row](https://docs.knock.app/api-reference/overview/rate-limits) if the rows run from tier 1 to 5 (labels did not render). Feed reads are [per user](https://docs.knock.app/api-reference/users/feeds/list_items) | [1,000 recipients per trigger](https://docs.knock.app/send-notifications/triggering-workflows/api); unlimited object subscribers, deduplicated ([subscriptions](https://docs.knock.app/concepts/subscriptions)) | Fixed or sliding, up to 90 days; all plans ([batch](https://docs.knock.app/designing-workflows/batch-function), [pricing](https://knock.app/pricing)) |
| **Novu** | Header, 24 h, **errors cached too** ([docs](https://docs.novu.co/api-reference/idempotency)) | "Depends on provider config and Novu's retry policy" ([full docs](https://docs.novu.co/llms-full.txt)); figures unverified | Trigger 60 / 240 / 600 per second on Free / Pro / Team; 512 KB payload ([limits](https://docs.novu.co/api-reference/rate-limiting), [payload](https://docs.novu.co/api-reference/payload-limits)) | Topics up to 100,000 ([topics](https://docs.novu.co/platform/concepts/topics)) | Digest up to 24 h / 7 d / 90 d on Free / Pro / Team ([limits](https://docs.novu.co/platform/developer/limits)) |
| **SuprSend** | Key in body, 24 h ([docs](https://docs.suprsend.com/reference/idempotent-requests)) | [Vendor fallback](https://docs.suprsend.com/docs/vendor-fallback) (Essentials and up); general retry schedule unverified | 100 records and 800 KB per request; each record counts as a call ([trigger](https://docs.suprsend.com/reference/trigger-workflow-api)). Trigger rate limit unverified | 100 recipients per trigger ([trigger](https://docs.suprsend.com/docs/trigger-workflow)); unlimited object subscribers, added 100 at a time ([objects](https://docs.suprsend.com/docs/object-subscriptions)) | [Batch](https://docs.suprsend.com/docs/batch) and digest; **Business and up** |
| **Courier** | POST header, 25 h, up to 1 year; stores errors too ([API](https://www.courier.com/docs/reference/api-overview)) | Provider 429s retried up to 24 h, about 104 attempts ([integrations](https://www.courier.com/docs/integrations/overview)) | Send has no request-count limit; 6 MB body ([API](https://www.courier.com/docs/reference/api-overview)) | Lists; bulk jobs take 1,000 recipients per call ([bulk](https://www.courier.com/docs/guides/send-in-bulk)) | Batch up to 7 days, 1,000 events ([batch](https://www.courier.com/docs/journeys/nodes/batch)); pricing table ticks all plans |
| **MagicBell** | Header on some endpoints, 24 h ([API](https://www.magicbell.com/docs/api)) | Unverified | **500/min per IP**, then a 60 s block (same page) | 1,000 recipients per broadcast, or topics ([broadcast](https://www.magicbell.com/docs/primitive/broadcast)) | No digest or batch command listed ([workflows](https://www.magicbell.com/workflows)) |

### Retention, deletion and location

| | Retention | Deletion | Data location |
| - | - | - | - |
| **Knock** | Feed indefinite; logs 30 d (90 d Enterprise) ([retention](https://docs.knock.app/manage-your-account/data-retention)) | Hard-deletes user, preferences, messages and runs ([deletion](https://docs.knock.app/managing-recipients/deleting-users)) | EU residency "by the end of 2026" ([security](https://docs.knock.app/developer-tools/security)); current exact region unverified |
| **Novu** | Feed 30 d Free, 90 d Pro and Team; activity 1 d / 7 d / 90 d ([limits](https://docs.novu.co/platform/developer/limits)) | Deletes subscriber, messages, preferences and topic subscriptions ([API](https://docs.novu.co/api-reference/subscribers/delete-a-subscriber)) | US (Virginia) or EU (Frankfurt) on all plans; fixed at signup ([security](https://docs.novu.co/platform/additional-resources/security)) |
| **SuprSend** | Logs 30 d Free, 90 d paid ([pricing](https://www.suprsend.com/pricing)); feed default unverified, per-message expiry available ([inbox](https://docs.suprsend.com/docs/in-app-inbox-template)) | User deletion ([API](https://docs.suprsend.com/reference/delete-user)); whether messages go too is unverified | EU option ([security](https://docs.suprsend.com/docs/security)); "Custom Data Center Region" is Enterprise only ([pricing](https://www.suprsend.com/pricing)); default region unverified |
| **Courier** | Inbox 1 year ([behaviour](https://www.courier.com/docs/in-app/how-the-inbox-behaves)). Business logs: 90 d in [logs docs](https://www.courier.com/docs/monitor/logs), 30 d on [pricing](https://www.courier.com/pricing) | Several calls; documented [request table](https://www.courier.com/docs/security/compliance) does not list messages. Their erasure is unverified | US, or EU (Ireland) on Business and up ([residency](https://www.courier.com/docs/security/data-residency)) |
| **MagicBell** | Unverified | "Removes a user and all associated data" ([API](https://www.magicbell.com/docs/api/reference)) | Product data location unverified |

Correcting something already sent also differs. Knock cannot edit a feed item, only archive and resend ([feeds](https://docs.knock.app/in-app-ui/feeds/overview)). Novu can delete messages by transaction ID ([API](https://docs.novu.co/api-reference/messages/delete-messages-by-transactionid)).

### Local and test environments

- **Knock:** unlimited environments ([pricing](https://knock.app/pricing)), but a new account has only production until more are enabled, which is irreversible ([environments](https://docs.knock.app/version-control/environments)). Sandbox per channel, except in-app ([sandbox](https://docs.knock.app/integrations/overview)), and per trigger ([trigger](https://docs.knock.app/send-notifications/triggering-workflows/api)).
- **Novu:** development and production on Free and Pro, up to 10 on Team ([limits](https://docs.novu.co/platform/developer/limits)). Development runs are billed ([billing](https://docs.novu.co/platform/account/billing)).
- **SuprSend:** sandbox, staging and production by default ([workspaces](https://docs.suprsend.com/docs/workspaces)). Test Mode blocks or redirects email, SMS and WhatsApp in staging; feed and push need support to include ([test mode](https://docs.suprsend.com/docs/developer/test-mode)).
- **Courier:** test and production, up to 20 in total ([environments](https://www.courier.com/docs/workspaces/environments)). Test sends are billed ([billing](https://www.courier.com/docs/workspaces/billing)).
- **MagicBell:** separate projects, one on Builder and five on Startup ([pricing](https://www.magicbell.com/pricing)). A test mode is unverified.

All of these are cloud environments needing keys and a network.

## Price

Prices are US dollars per month, before tax, and exclude Postmark and any push provider.

| | Start free | First paid plan | Over the free allowance |
| - | - | - | - |
| [Knock](https://knock.app/pricing) | 10,000 messages | Starter $250, 50,000 messages, then $0.005 | No hard stop; they contact you |
| [Novu](https://novu.co/pricing) | 10,000 runs, 5 workflows | Pro from $30, 30,000 runs, then $1.20 per 1,000, 20 workflows | Rejected with HTTP 402 until reset ([billing](https://docs.novu.co/platform/account/billing)) |
| [SuprSend](https://www.suprsend.com/pricing) | 10,000 notifications; no batching or preferences | Essentials $100, 50,000, then $0.002. Business $250 (adds batching and preferences), 50,000, then $0.005 | Keeps processing; they ask you to upgrade |
| [Courier](https://www.courier.com/pricing.md) | 10,000 sends | Business $0.005 for **every** send, including the first 10,000 | Needs a card and Business |
| [MagicBell](https://www.magicbell.com/pricing) | 1,000 deliveries | Startup $249, 50,000 deliveries, then $0.0025 | Builder behaviour unverified |

Vendor branding: Knock and SuprSend remove it from the first paid plan up; Novu removes it on Pro.

### At 10,000 monthly recipients

These are **calculations from the list prices above, not quotes**. Each assumes every recipient gets the stated number of Notifications and every Notification is delivered in full. Real push volume will be lower, because not everyone grants permission. Preferences and digesting lower it further, except for Novu and Courier: Novu bills digested events, and Courier bills preference-blocked sends.

| Scenario | Knock | Novu Pro | SuprSend Essentials | SuprSend Business | Courier Business | MagicBell |
| - | -: | -: | -: | -: | -: | -: |
| 4 each, feed only | $250 | $42 | $100 | $250 | $200 | $249 |
| 4 each, feed + push | $400 | $42 | $160 | $400 | $200 | $324 |
| 4 each, feed + push + email | $600 | $42 | $240 | $600 | $200 | $424 |
| 10 each, feed only | $500 | $114 | $200 | $500 | $500 | $374 |
| 10 each, feed + push | $1,000 | $114 | $400 | $1,000 | $500 | $624 |
| 10 each, feed + push + email | $1,500 | $114 | $600 | $1,500 | $500 | $874 |

- 4 each = 40,000 events; 10 each = 100,000 events. Multiply by channels for per-channel billers.
- Novu assumes one workflow per event covering all channels, and the listed $30 plan plus listed overage. The "from $30" tiers may differ; the in-app calculator was not checked.
- Knock and SuprSend also offer per-user pricing through sales (both pricing pages). At this size it may be much cheaper, or not available; get a quote before ruling either out.
- At one Notification each, 10,000 recipients already reach the free ceiling on per-recipient billing, and pass it with a second channel on per-channel billing.

## Signs of longevity

| | Operating now | Maintenance | Company |
| - | - | - | - |
| Knock | [Status](https://status.knock.app): fully operational | Changelog entry 2026-11-24; Node SDK 2026-11-10 ([npm](https://www.npmjs.com/package/@knocklabs/node)) | $12M round, 2024, secondary source ([TechCrunch](https://techcrunch.com/2024/02/15/knock-takes-the-pain-out-of-building-notification-workflows)) |
| Novu | Site and docs live; status page not reachable, so unverified | React SDK 2026-11-23 ([npm](https://www.npmjs.com/package/@novu/react)); [changelog](https://novu.co/changelog) 2026-11-16; [public repo](https://github.com/novuhq/novu) pushed 2026-11-25 | Open core, which gives an exit route, though Cadence rules out self-hosting. Funding unverified |
| SuprSend | [Status](https://status.suprsend.com): operational | React SDK 2026-11-17 ([npm](https://www.npmjs.com/package/@suprsend/react)); [changelog](https://docs.suprsend.com/changelog/overview) 2026-10-30; Node SDK older, 2026-05-19 | Funding unverified |
| Courier | [Status](https://status.courier.com): operational | React SDK 2026-11-25 ([npm](https://www.npmjs.com/package/@trycourier/courier-react)); [changelog](https://www.courier.com/changelog) 2026-11-19 | $35M round, 2022, secondary source ([TechCrunch](https://techcrunch.com/2022/06/22/courier-lands-35m-to-build-a-service-for-app-notifications)) |
| MagicBell | [Status](https://status.magicbell.com): online | React SDK 2026-07-29 ([npm](https://www.npmjs.com/package/@magicbell/react)); last [blog](https://www.magicbell.com/blog) date seen 2026-05-13 | Funding unverified; slowest visible pace of the five |

A web search found no 2026 shutdown or acquisition news for these five. That is weak evidence, not confirmation. Customer counts, profitability and runway are unverified.

## Not shortlisted

- **Courier: runner-up.** Excellent idempotency and cheap multi-channel sends. Left out because no browser push guide was found, a separate seen state is unverified, inbox tokens need a network call and manual refresh, and message erasure is unverified. It is the natural substitute if push turns out simple and batch limits suit.
- **MagicBell: possible.** First-party web push, but no documented digest, no named Postmark integration, a 500-per-minute-per-IP limit that one Node process hits quickly, and retention unverified. Broadcasts of 1,000 ease the limit.
- **OneSignal: dropped.** Feed is a required capability and it has none: it "does not currently store the history of messages sent to each individual user" and tells you to store it yourself ([activity feed](https://documentation.onesignal.com/docs/en/create-an-activity-feed)). It remains a possible push provider behind Knock or Novu; list price would be $19 plus $0.004 per web push subscriber, so about $59 at 10,000 ([pricing](https://onesignal.com/pricing)).
- **Engagespot: dropped.** The site and pricing page return "Site not found" 404s ([home](https://engagespot.co), [pricing](https://engagespot.co/pricing)), and the React package was last published 2025-05-09 ([npm](https://www.npmjs.com/package/@engagespot/react-component)). Its [status page](https://status.engagespot.co) still reports the API as operational, so the service may run, but current terms and price cannot be confirmed.

## Trade-offs between the three

- **Push without Firebase:** SuprSend only. With Knock or Novu, choose Firebase, OneSignal, or a webhook back to Cadence that sends VAPID push itself.
- **Cost:** Novu is several times cheaper in every scenario. SuprSend Essentials is second, but loses batching and preferences. Knock and SuprSend Business have the same list rates.
- **Feed history:** Knock needs no extra storage. Novu loses items after 90 days on paid self-serve plans. SuprSend's default needs asking.
- **Token safety:** Knock and SuprSend expire and refresh tokens. Novu's documented hash is fixed for a user; decide whether that satisfies the map's "server-signed per-user vendor tokens".
- **EU data:** Novu has it on every plan today. SuprSend's self-serve position needs a written answer. Knock does not have it yet.
- **Outages and fan-out:** All accept a key for safe retries. Knock documents the most about delivery after acceptance; Novu and SuprSend need questions answered.
- **Operating risk:** If free-tier volume is exceeded, Novu rejects triggers; Knock and SuprSend keep going.

## Ask vendors, or prototype, before choosing

1. Does the React UI render cleanly in a React Router 8 SSR app, and does push work in current Safari?
2. Novu: can one user's hash be revoked, and what is the delivery retry schedule?
3. SuprSend: default feed retention, trigger rate limit, what user deletion erases, and which region Essentials and Business use.
4. Knock: is EU residency live, and what would per-user pricing cost for 10,000?
5. Is an EU data location actually a Cadence requirement? The map does not say.
