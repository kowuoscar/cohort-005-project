# Hosted in-app notification feed services for Cadence

**Question:** Which hosted third-party services can deliver an in-app notification feed for Cadence, and how do they compare?

**Constraints:** React Router 8 (framework mode, SSR), Drizzle/SQLite, Tailwind/shadcn. No real auth: a dev user switcher, and recipients are identified by our DB user id. The service must be hosted (no self-hosting), have a free tier big enough for dev, ship a React SDK, and push in real time to an open page. v1 is the in-app feed only. We want to avoid signed user tokens or HMAC unless the service requires them even in dev.

**Checked:** 2026-11-21. Package versions come from `npm view` on that date. Vendor docs came from official pages and the vendors' own `llms-full.txt` exports (docs.knock.app, docs.novu.co, courier.com/docs, docs.suprsend.com). Items marked **(unverified)** were not confirmed from a primary source.

## Comparison

| Criterion | Knock | Novu Cloud | Courier | MagicBell | SuprSend |
| --- | --- | --- | --- | --- | --- |
| Free tier | 10k messages/mo, unlimited envs | 10k workflow runs/mo, hard stop (402) | 10k sends/mo (Developer) | 1,000 deliveries/mo, 1 project | 10k notifications/mo, "multiple environments" |
| Signed token required? | No. Enhanced security is off by default | No. HMAC is an opt-in toggle | **Yes. A backend-minted JWT is always required** | **Yes. User JWT signed with the project secret** | No. Enhanced security is an opt-in toggle |
| Prebuilt feed | Popover, full feed, bell | `<Inbox />` popover/bell, tabs | `CourierInbox`, popup menu | `Inbox`, `FloatingInbox` | `Inbox` popover, `NotificationFeed` |
| Headless hooks | Yes (`useNotifications`, `useNotificationStore`) | Yes (`useNotifications`, `useCounts`, `useNovu`) | Yes (`useCourier`) | None in current pkg (old headless pkg deprecated) | Yes (`@suprsend/react-core`: `useFeedData`, `useFeedClient`) |
| Theming | `--rnf-*` CSS vars, `rnf-` classes, `renderItem`/`renderHeader`, skip CSS import | `appearance` (variables, `elements` accept CSS class strings), render props | Theme objects, render slots; web components with shadow DOM | `--magicbell-*` CSS vars, class names | `theme` style object (Emotion), component overrides |
| Real-time | Socket, built in | WebSocket, built in | WebSocket (`listenForUpdates()`) | "Persistent connection" | Socket, built in |
| Read/all-read/archive/URL | All four | All four | All four | Read/archive/click (mark-all-read unverified) | All four |
| Fan-out | Objects + subscriptions; 1,000 recipients/trigger | Topics; bulk 100 events/req; broadcast | Lists, audiences, bulk jobs | Broadcast up to 1,000 recipients, topics | Lists + broadcast; 100 recipients/trigger |
| Dev/test envs | Development + Production, keys per env | Development + Production (custom envs on Team+) | Production + Test, keys per env | (unverified) | Sandbox, Staging, Production; Test Mode |
| React pkg (latest) | `@knocklabs/react` 0.15.1 | `@novu/react` 3.20.0 | `@trycourier/courier-react` 9.3.5 | `@magicbell/react` 1.14.0 | `@suprsend/react` 1.6.0 |
| React 19 peer | Yes (`^17 \|\| ^18 \|\| ^19`) | Yes (`^18 \|\| ^19`) | Yes (`>=18`) | No peer deps declared (unverified) | `>=16.8` |
| Node pkg (latest) | `@knocklabs/node` 1.41.1 | `@novu/api` 3.20.0 | `@trycourier/courier` 10.1.0 | `magicbell-js` 1.8.0 | `@suprsend/node-sdk` 1.15.1 |

## Knock

- **Free tier:** The Developer plan includes "up to 10,000" messages a month, unlimited environments, unlimited feeds and feed retention, and pre-built components. There is no hard cap; Knock contacts you about overages. Knock branding stays on. [Pricing](https://knock.app/pricing)
- **Identity:** "Enhanced security mode is off by default." Without it, Knock "will not reject requests that do not include a signed user token". The tradeoff is that "a request with your public key can act as any user." Knock says to enable it before release. `KnockProvider` takes `apiKey` and `user: { id }`, so our DB id works unchanged. [Security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication), [headless feed](https://docs.knock.app/in-app-ui/react/headless/feed)
- **Prebuilt UI:**
  - Components: `NotificationFeedPopover`, `NotificationFeed`, and `NotificationIconButton`. [React overview](https://docs.knock.app/in-app-ui/react/overview)
  - Theming: override `--rnf-` CSS variables, or skip the `@knocklabs/react/dist/index.css` import and style the unique `rnf-` classes yourself. `renderItem` and `renderHeader` replace parts of the feed, and `colorMode="dark"` switches the palette. [Feed component](https://docs.knock.app/in-app-ui/react/feed), [theme.css](https://github.com/knocklabs/javascript/blob/main/packages/react/src/theme.css)
  - Fit: there is no Tailwind-native mode, but a CSS variable bridge to shadcn tokens is feasible.
  - Dependencies: the package pulls in `@telegraph/*`, Radix, and `lucide-react` (`npm view @knocklabs/react dependencies`).
- **Headless:** `useNotifications(knockClient, channelId)` and `useNotificationStore(feedClient)`. The headless hooks do not mark items seen automatically. [Headless feed](https://docs.knock.app/in-app-ui/react/headless/feed)
- **Read state and actions:** The feed client marks items seen, read, or archived. `feedClient.markAllAsRead()` appears in the `useNotifications` docs. The prebuilt feed archives an item with its "x" button. `KnockFeedProvider` "will attach a redirect handler if `action_url` is not empty", and `onNotificationClick` overrides it. [Feed component](https://docs.knock.app/in-app-ui/react/feed), [llms-full.txt, "In-app feeds" and `useNotifications`](https://docs.knock.app/llms-full.txt)
- **Real-time:** The feed uses a socket connection for live updates. `auto_manage_socket_connection` disconnects idle tabs, and cross-tab sync is experimental. [Feed component](https://docs.knock.app/in-app-ui/react/feed)
- **SSR:** The Next.js quickstart puts the components in a `"use client"` file because they "use client-side React APIs" ([llms-full.txt, "Get started"](https://docs.knock.app/llms-full.txt)). React Router has no `"use client"`, so plan to render the feed client-only, behind a `ClientOnly`/`useEffect` mount or lazy import. The SSR behavior itself is **(unverified)**. `next` and `@tanstack/react-router` are optional peers, which is harmless here.
- **Server SDK and fan-out:**
  - Trigger with `@knocklabs/node`. A trigger takes "up to 1000 recipients at a time". For larger groups Knock recommends subscriptions. [Triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api), [llms-full.txt](https://docs.knock.app/llms-full.txt)
  - Objects plus subscriptions give fan-out. There is "no limit to the number of recipients you can subscribe to an object". Each subscription call adds or removes up to 100 recipients, and nesting depth is 2. A course could map to an object. [Subscriptions](https://docs.knock.app/concepts/subscriptions)
- **Dev/test:** Development and Production environments have isolated data and their own API keys, and content is promoted upward. No separate "test mode" is documented. [Environments](https://docs.knock.app/concepts/environments)

## Novu Cloud

- **Free tier:**
  - 10,000 workflow runs a month, counted across all environments.
  - Free "stops new workflow runs at its 10,000 included runs". Triggers then fail with HTTP 402.
  - A topic trigger counts one run per matching subscriber.
  - Custom environments need the Team plan or above.
  - Source: [llms-full.txt, "Billing"](https://docs.novu.co/llms-full.txt); [pricing](https://novu.co/pricing) was not fetched directly.
- **Identity:** `<Inbox applicationIdentifier subscriber="..."/>` takes our id as `subscriberId`. HMAC is enabled by a dashboard toggle on the Novu In-App integration ("enable `Security HMAC encryption` toggle"), and Novu frames it as production prep. [Prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)
  - Conflict: Novu's agent-skill reference says "HMAC is mandatory in production" ([skill: inbox security](https://docs.novu.co/.mintlify/skills/inbox-integration/references/security)). The official page describes HMAC as opt-in. Treat it as optional in Development and expect to enable it for production.
  - Keyless mode allows instant UI trials. [Setup inbox](https://docs.novu.co/platform/inbox/setup-inbox)
- **Prebuilt UI:** `<Inbox />` is themed through `appearance` (`baseTheme`, `variables`, `elements`, `icons`). Each `elements` entry "can be either a style object or a set of CSS classes", which makes Tailwind classes workable. Styles are injected into `<head>`, or into a shadow root if you mount it there. The Inbox shows a "Development mode" striped footer in Dev, and branding removal is paid. [Styling](https://docs.novu.co/platform/inbox/configuration/styling), [customize items](https://docs.novu.co/platform/inbox/advanced-customization/customize-notification-items)
- **Headless:** `useNotifications` (pagination plus real time), `useCounts` (unread/unseen badges), and `useNovu` for actions. Hooks "automatically listen for real-time events". [Headless mode](https://docs.novu.co/platform/inbox/headless-mode)
- **Read state and actions:** The client exposes `novu.notifications.readAll()` and `archiveAll()`. Read, unread, and archive work per item. An in-app step's `redirect: { url, target }` is followed on click. [llms-full.txt](https://docs.novu.co/llms-full.txt), [click behavior](https://docs.novu.co/platform/inbox/advanced-customization/notification-click-behavior)
- **Real-time:** "`@novu/js` automatically maintains a WebSocket connection". EU needs `socketUrl`. [Prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)
- **SSR:** Novu says "The Inbox component is client-side only" (Next.js examples use `"use client"`). [Skill: Next.js examples](https://docs.novu.co/.mintlify/skills/inbox-integration/references/nextjs-inbox-examples). Use client-only rendering in React Router.
- **Server SDK and fan-out:**
  - Use `@novu/api`. `@novu/node` is legacy and was last published 2025-12. [llms-full.txt](https://docs.novu.co/llms-full.txt)
  - Topics handle fan-out. Bulk trigger "is limited to 100 events per request". `POST /v1/events/trigger/broadcast` reaches all subscribers. [Bulk trigger](https://docs.novu.co/api-reference/events/bulk-trigger-event), [broadcast](https://docs.novu.co/api-reference/events/broadcast-event-to-all)
  - The maximum array size for a single trigger's `to` is **(unverified)**.
- **Dev/test:** Development and Production environments with environment-scoped keys. [API authentication](https://docs.novu.co/api-reference/authentication)
- **Rate limits (Free):** 20 rps for configuration endpoints, 60 rps for event triggers. [llms-full.txt](https://docs.novu.co/llms-full.txt)

## Courier

- **Free tier:** "The Developer plan is free and includes 10,000 sends a month". Sends over the cap get `THROTTLED`. [llms-full.txt, FAQ/billing](https://www.courier.com/docs/llms-full.txt)
- **Identity (blocker):** "Courier's client SDKs authenticate with a JWT." Your backend mints it with `POST /auth/issue-token` (scope such as `user_id:<id> inbox:read:messages inbox:write:events`), and the client calls `signIn({ userId, jwt })`. Since v8, client keys "no longer authenticate anything, so every app needs a backend endpoint that issues a token." There is no unsigned mode, even in Test. [Authenticate users](https://www.courier.com/docs/in-app/authenticate-users), [v8 migration](https://www.courier.com/docs/sdk-libraries/courier-react-v8-migration-guide)
  - Minting is a single server call, which would fit a React Router loader keyed off the dev-switcher user.
- **Prebuilt UI:** `CourierInbox` and a popup menu are themed with `lightTheme`/`darkTheme` objects and `mode`. Render slots (`header`, `listItem`, `emptyState`, and others) replace parts of the UI. The components wrap web components that use shadow DOM, so global Tailwind classes will not reach the default internals **(inference; unverified for slot content)**. [Customize](https://www.courier.com/docs/in-app/customize-the-inbox), [custom inbox](https://www.courier.com/docs/in-app/build-a-custom-inbox), [React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web)
- **Headless:** `useCourier` returns messages, unread counts, and actions. [React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web)
- **Read state and actions:** Read, archive, `archiveAll`, and `archiveRead` header actions are available. `onMessageClick` and `onMessageActionClick` handle clicks. [Customize](https://www.courier.com/docs/in-app/customize-the-inbox)
- **Real-time:** You must call `inbox.listenForUpdates()` after sign-in to open the WebSocket. [React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web)
- **SSR:** Courier says the components "render client-side only", with `"use client"` in Next.js. Importing on the server is a no-op because the elements register only when `window` exists. [React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web)
- **Server SDK and fan-out:** `@trycourier/courier` handles sends to users, lists, and audiences. Bulk jobs and broadcasts cover large groups. [llms-full.txt, Lists/Audiences/Bulk](https://www.courier.com/docs/llms-full.txt)
- **Dev/test:** Every workspace has Production and Test environments, with keys per environment. "Courier has no separate test mode: a Test key is what puts you in the Test environment." [Environments](https://www.courier.com/docs/workspaces/environments)

## MagicBell

- **Free tier:** The "Builder" plan is $0 with "1,000 Deliveries/mo", one project, and unlimited users. Extra deliveries are charged per use. [Pricing](https://www.magicbell.com/pricing)
- **Identity (blocker):** `MagicBellProvider` takes `token`, "The MagicBell User JWT … You can generate these on your own server." The JWT is HS256, signed with the project Secret Key, and carries `user_external_id` and/or `user_email` plus `api_key`. No unsigned mode or toggle is documented. [React docs](https://www.magicbell.com/docs/libraries/magicbell-react), [User JWT](https://www.magicbell.com/docs/api/authentication/user)
- **Prebuilt UI:** `Inbox`, `FloatingInbox`, `InlineNotification`, and `ToastContainer`. Theming uses `--magicbell-*` CSS vars and predefined `.magicbell--*` class names. `className`/`unstyled` exist only on some buttons, not on Inbox. [React docs](https://www.magicbell.com/docs/libraries/magicbell-react)
- **Headless:** `@magicbell/react` 1.14.0 exports components only (checked `dist/esm/index.d.ts` on unpkg). `@magicbell/react-headless` and `@magicbell/magicbell-react` still resolve on npm, but their README says "This package is deprecated. Please use `@magicbell/react` or `magicbell-js`" ([README](https://unpkg.com/@magicbell/react-headless@5.5.1/README.md)). Custom UI means using `magicbell-js` directly.
- **Read state and actions:** "Users can view, mark as read, archive, and delete notifications". The broadcast schema has `actionUrl`. Mark-all-read is **(unverified)**. [In-app channel](https://www.magicbell.com/docs/channels/in_app), [Node client](https://www.magicbell.com/docs/libraries/magicbell-js)
- **Real-time:** "MagicBell maintains a persistent connection". The transport is unnamed in the docs; the package depends on `reconnecting-websocket`. [In-app channel](https://www.magicbell.com/docs/channels/in_app)
- **Server SDK and fan-out:** `magicbell-js/project-client` has `broadcasts.createBroadcast` (project JWT). A broadcast's `recipients` array allows up to 1,000 entries (users or topics). [Node client](https://www.magicbell.com/docs/libraries/magicbell-js), [Broadcast](https://www.magicbell.com/docs/primitive/broadcast)
- **Gaps:** No SSR guidance. React 19 compatibility is **(unverified)**: no `peerDependencies` are declared and dev types are `@types/react ^18`. The dev/test environment story is **(unverified)**.

## SuprSend

- **Free tier:** "10k Notifications/mo", "Multiple Environments", and an in-app feed "with SuprSend branding". [Pricing](https://www.suprsend.com/pricing)
- **Identity:** Client SDKs use a public API key plus the user's `distinct_id`. Enhanced security mode is a switch you enable ("recommended for Production workspaces"). When on, requests without an ES256 user JWT are rejected. Token verification "only runs with Enhanced Security Mode on". [Client authentication](https://docs.suprsend.com/docs/client-authentication), [llms-full.txt](https://docs.suprsend.com/llms-full.txt)
- **Prebuilt UI:**
  - Components: `@suprsend/react` provides `Inbox` (popover) and `NotificationFeed` (full-page/side sheet).
  - Theming: a `theme` prop takes a style object, built on Emotion per the package deps. `themeType` switches dark/light.
  - Overrides: `bellComponent`, `badgeComponent`, `notificationComponent`, `headerRightComponent` (receives `markAllRead`), and `noNotificationsComponent`.
  - Fit: this is not Tailwind-native; for full control, use headless.
  - Sources: [React in-app feed](https://docs.suprsend.com/docs/react-in-app-feed), [customising feed](https://docs.suprsend.com/docs/react-customising-feed)
- **Headless:** `@suprsend/react-core` provides `useFeedData`, `useFeedClient`, and `useFeed`. [Headless feed](https://docs.suprsend.com/docs/react-sdk-headless-feed)
- **Read state and actions:** Mark-all-read is in the header. Archived items can show in a tab through `stores` with `{archived: true}`. Clicking a card opens the template's `Action URL`, and handlers can override it. [Customising feed](https://docs.suprsend.com/docs/react-customising-feed)
- **Real-time:** A feed socket is built in, with a connection-status dot that is on by default since 1.3.0. [React in-app feed](https://docs.suprsend.com/docs/react-in-app-feed)
- **Server SDK and fan-out:** `@suprsend/node-sdk` triggers workflows with "up to 100 recipients in a single workflow trigger". Lists plus broadcasts cover large groups. [Broadcast](https://docs.suprsend.com/docs/broadcast), [llms-full.txt](https://docs.suprsend.com/llms-full.txt)
- **Dev/test:** Accounts start with Sandbox, Staging, and Production workspaces, each with its own keys. Staging adds Test Mode. [Workspaces](https://docs.suprsend.com/docs/workspaces), [Test mode](https://docs.suprsend.com/docs/developer/test-mode)
- **Caveats:**
  - `@suprsend/react-inbox` (3.9.0) is marked deprecated on npm. Use `@suprsend/react` 1.6.0 instead.
  - SSR guidance is **(unverified)**.

## Recommendation (for a later human decision)

**Top pick: Knock.**
- It meets every constraint without signed tokens: enhanced security is off by default, and `user.id` is our DB id.
- The free tier is enough (10k messages/mo), and the separate Development environment has its own keys.
- It has a real headless API (`useNotifications`/`useNotificationStore`), so a shadcn-native feed is possible. The prebuilt popover can also be restyled through `--rnf-` vars and `rnf-` classes as a fallback.
- Objects and subscriptions map naturally to "everyone enrolled in a course" without looping.
- The React 19 peer range is explicit.

**Runner-up: Novu Cloud.**
- HMAC is also opt-in, and the free tier is the same size.
- Its `appearance.elements` accepts CSS class strings, the most Tailwind-friendly of the prebuilt UIs.
- Its hooks are good.
- It ranks below Knock for three reasons:
  - The Free plan hard-stops at 10k runs, with Dev and Prod sharing the quota.
  - Its own docs conflict on whether HMAC is mandatory in production.
  - The Dev-mode footer appears in development.

**Not recommended for v1 under these constraints:** Courier and MagicBell both require a backend-signed JWT for every client session, including in dev. MagicBell's free tier (1,000 deliveries) is also thin, and its headless React package is deprecated. SuprSend fits the identity constraint, but its prebuilt UI is Emotion-styled and its React package lineage recently changed.

**Caveats:**
- **Unsigned mode is insecure.** In Knock, Novu, and SuprSend, unsigned client access lets anyone holding the public key read any user's feed by id. That is fine with a dev switcher, but whichever service is chosen, production will need token or HMAC signing in a loader once real auth exists.
- **Render the feed client-only.** All of these UIs are client-side, so in React Router SSR they should mount only on the client. None of the vendors document React Router specifically.
- **Volatile details.** Free-tier numbers and package versions were read on 2026-11-21 and change often. Knock and Novu pricing pages could change, and Novu's pricing page itself was not fetched (figures are from its docs).
