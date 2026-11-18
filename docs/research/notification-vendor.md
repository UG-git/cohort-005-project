# Research: third-party Notification vendor

Question: which vendor should own Cadence's in-app Notification feed (storage, read/unread state, ideally the inbox UI)? See issue #2, child of map #1.

Researched 2026-11-18 against vendor pricing pages, official docs and published npm packages. Prices and limits change, so re-check the linked pages before signing up.

## Recommendation

**Novu (Novu Cloud, EU region in Frankfurt).** It is the only candidate that offers EU data residency **on the free plan**. It also has a React 19 `<Inbox />` that ships a server-safe build, so it renders under React Router 8 SSR without a client-only wrapper. It has a typed Node SDK (`@novu/api`), real-time over WebSocket, and simple HMAC subscriber auth that maps directly onto Cadence user ids.

Main caveat: the free plan allows only **5 workflows** and keeps Inbox messages for **30 days**. Cadence's Notification catalogue should therefore use a few generic, payload-driven workflows (for example, one `in-app-notification` workflow that takes subject, body and URL in the payload) rather than one workflow per event. The alternative is to move to Pro (from $30/month).

Runner-up: **Knock**, which has the best developer experience and unlimited workflows on its free plan, but no EU data residency yet ("expected by the end of 2026"). If EU hosting were negotiable, Knock would be a close call.

Building in-house is not needed: Novu meets every hard constraint.

## Comparison

| | Novu | Knock | Courier | SuprSend | MagicBell |
|---|---|---|---|---|---|
| Free tier | 10k workflow runs/mo, **max 5 workflows**, 2 environments, 3 team members, Inbox messages kept 30 days [1][3] | 10k messages/mo, unlimited workflows and environments [6] | 10k sends/mo ("Developer") [10] | 10k notifications/mo, Inbox included with SuprSend branding [13] | 1k deliveries/mo, 1 project [16] |
| EU hosting | **Yes, on Free.** Frankfurt; `eu.api.novu.co`, `wss://eu.socket.novu.co` [1][3] | **No.** "Coming soon", "expected by the end of 2026" [7] | Yes, but **paid plans only** (Business, pay-as-you-go $0.005/send, and Enterprise) [10][11] | Not documented. Only "Custom Data Center Region" on Enterprise [13][14] | Not documented [17] |
| React inbox | `@novu/react` (React 18/19), `<Inbox />`, plus hooks for a headless build [2] | `@knocklabs/react`, `KnockProvider` + `KnockFeedProvider` + feed popover [8] | `@trycourier/courier-react` v9, `<CourierInbox />` [12] | `@suprsend/react`, `SuprSendProvider` + `<Inbox />` [15] | `@magicbell/react`, `Provider token=...` [18] |
| SSR (RR8) | Package `exports` route the server/`import` condition to a stub build that renders a hidden `<div>`; the browser build mounts the real UI (verified in `@novu/react@3.20.0`) | Not documented; client-side feed | Not documented | Not documented | Not documented |
| Node SDK | `@novu/api`: `novu.trigger({ workflowId, to: { subscriberId }, payload })`; `serverURL` for EU [4] | `@knocklabs/node` [9] | `@trycourier/courier`, `COURIER_BASE_URL` for EU [11] | Node SDK [15] | REST/JS client [18] |
| Real-time | Yes (WebSocket) [2] | Yes (socket) [8] | Yes [12] | Yes (WebSocket) [15] | Toasts; real-time not stated [18] |
| Client auth | HMAC-SHA256(secretKey, subscriberId), hex, toggled per environment [5] | RS256 JWT (`sub` = user id), `signUserToken` helper; off by default ("enhanced security mode") [9] | JWT issued by backend, `signIn({ userId, jwt })` [12] | ES256 JWT (`entity_id` = distinct id) [14] | User JWT generated on your server [18] |

Also considered: NotificationAPI, now Pingram. Its free plan includes the EU region, but its pricing page only covers SMS, email and calls and doesn't mention in-app or inbox [19], so it is not a strong in-app candidate.

## Details for Novu

### Recipient identity
- A Novu **subscriber** is identified by a string `subscriberId`. Triggering a workflow to a new `subscriberId` creates the subscriber automatically [2][4]. Map it 1:1 to `users.id` (Cadence's integer primary key, as a string).
- The Inbox takes `applicationIdentifier` (environment), `subscriberId` and, when HMAC is on, `subscriberHash`. EU accounts must also pass `backendUrl="https://eu.api.novu.co"` and `socketUrl="wss://eu.socket.novu.co"` [2].
- HMAC: "If HMAC encryption is active in In-App provider settings and `subscriberHash` is not provided, the Inbox will not load." The hash is `createHmac('sha256', NOVU_SECRET_KEY).update(subscriberId).digest('hex')`, computed on the server [5]. HMAC is enabled per environment, so it can be left off in the Development environment, but turning it on there too keeps dev and production the same.
- API keys are bound to their region. Sending an EU key to the US host returns 401, and an account can't move regions by changing the URL [3]. **Create the account on `eu.dashboard.novu.co` from day one.**

### Fit with the DevUI panel
`api.switch-user.ts` sets the session cookie and redirects, so the layout loader runs again for the new user. The loader can return `{ subscriberId, subscriberHash }` for the current session user, and `<Inbox key={subscriberId} ... />` makes the widget remount and reconnect as the new user. The secret key never reaches the browser, and switching users needs no extra vendor call.

### SSR
`@novu/react` exposes `browser` and server (`import`/`require`) export conditions. Under SSR, `Inbox` resolves to a stub that renders a 1px hidden `ShadowRootDetector` div, and `useNovu`/`useCounts` return empty values. In the browser, the real component mounts (checked in the published `@novu/react@3.20.0` tarball, `package.json` `exports` and `dist/esm/server/index.js`). So the bell shows up after hydration and is not part of the server HTML. The unread count is not in the first paint unless Cadence fetches it in a loader.

### Testing without calling the vendor
No vendor offers a local emulator for this. Novu's "keyless" Inbox mode is for quick demos, and its data expires after 24 hours [2]. The usual approach works with any vendor: put the `@novu/api` client behind a small Cadence-owned port (for example, `notify(recipientUserId, kind, payload)`). Inject a recording fake in Vitest and assert on the recorded calls. Test the HMAC helper as a pure function. Only the real adapter imports `@novu/api`. The UI side needs no tests against Novu, because the component is the vendor's.

## Surprises that affect later decisions
1. **Novu's 5-workflow free cap.** This shapes the Notification catalogue and integration design: use generic, payload-driven workflows, or budget for Pro.
2. **Inbox retention on free is 30 days** (90 on paid) [3]. This affects the "how long Notifications are kept" question in Notification content.
3. **The bell is client-only under SSR.** It pops in after hydration, so a UI placement in a persistent header or sidebar avoids layout shift. Give it a fixed-size slot.
4. **Region lock-in at signup.** The EU account must be created on the EU dashboard. Keys don't work across regions.
5. **Shared Development environment and seeded ids.** Every developer's seeded database reuses user ids 1, 2, 3..., so if developers share one Novu Development environment, their Notifications mix. Consider a per-developer prefix in `subscriberId` (such as `dev-<name>-<id>`), or one Novu environment per developer. The free plan only allows 2 environments [1].
6. **Knock's EU residency is still "coming soon"** as of this date. Courier puts EU behind paid plans, and SuprSend behind Enterprise. If Novu's free cap becomes a problem, Courier Business (pay-as-you-go, EU) is the cheapest EU-hosted fallback.

## Sources
1. Novu pricing: https://novu.co/pricing
2. Novu Inbox setup: https://docs.novu.co/platform/inbox/setup-inbox
3. Novu security and compliance (EU region, retention, GDPR/DPA, SOC 2): https://docs.novu.co/platform/additional-resources/security
4. Novu TypeScript server SDK: https://docs.novu.co/platform/sdks/server/typescript
5. Novu Inbox: prepare for production (HMAC): https://docs.novu.co/platform/inbox/prepare-for-production
6. Knock pricing: https://knock.app/pricing
7. Knock plans and usage (EU residency roadmap): https://docs.knock.app/manage-your-account/knock-plans
8. Knock React feed: https://docs.knock.app/in-app-ui/react/feed
9. Knock in-app security and authentication: https://docs.knock.app/in-app-ui/security-and-authentication
10. Courier pricing: https://www.courier.com/pricing
11. Courier EU datacenter: https://www.courier.com/docs/platform/workspaces/eu-datacenter
12. Courier Inbox overview: https://www.courier.com/docs/platform/inbox/inbox-overview
13. SuprSend pricing: https://www.suprsend.com/pricing
14. SuprSend client authentication: https://docs.suprsend.com/docs/client-authentication
15. SuprSend React SDK: https://docs.suprsend.com/docs/react-sdk
16. MagicBell pricing: https://www.magicbell.com/pricing
17. MagicBell security and compliance: https://www.magicbell.com/security
18. MagicBell React library: https://www.magicbell.com/docs/libraries/react
19. Pingram (formerly NotificationAPI) pricing: https://www.pingram.io/pricing
