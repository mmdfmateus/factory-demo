# How Knock's in-app feed fits Cadence

Research for [#3](https://github.com/mmdfmateus/factory-demo/issues/3), a child of the map [#1 In-app Notifications](https://github.com/mmdfmateus/factory-demo/issues/1). Knock is already the chosen vendor; this note covers how Cadence would use it.

Researched 2026-11-12 against Knock's docs (docs.knock.app, fetched as `.md` via `https://docs.knock.app/llms.txt`) and the published npm tarballs. Where a claim comes from reading SDK source rather than docs, it says so.

## SDK versions checked

| Package | Version | Notes |
| --- | --- | --- |
| `@knocklabs/node` | 1.41.1 | Server SDK, generated with Stainless. Default export `Knock`; also exports `signUserToken`. |
| `@knocklabs/react` | 0.15.1 | Pre-built feed components plus re-exported hooks. Peer deps: React 17–19. `next` and `@tanstack/react-router` are **optional** peers, used only by the `./next` and `./tanstack` subpath exports (a guides location sensor), not by the feed. |
| `@knocklabs/react-core` | 0.17.1 | Hooks and providers (`KnockProvider`, `KnockFeedProvider`, `useNotifications`, `useNotificationStore`, `usePreferences`). Depends on `@knocklabs/client`, `swr`, `@tanstack/react-store`. |
| `@knocklabs/client` | 0.23.1 | Browser JS client that the React packages wrap. It owns the Phoenix websocket. |

All three are pre-1.0 except the Node SDK, so pin exact versions.

## 1. Identifying users, and how DevUI user switching maps onto it

- A Knock **user** is keyed by your own ID: a string of up to 256 characters with no `/` or `#`. It **cannot be changed** once set, and Knock recommends a stable primary key. Only `id` is required; `name`, `email`, `avatar`, `timezone`, and `locale` are reserved properties, and anything else is a custom property that gets deep-merged on each upsert. ([Users](https://docs.knock.app/concepts/users), [Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients))
- There are three ways to identify a user, and all of them **upsert**: ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients))
  - **Direct**: `PUT /v1/users/{id}`, which is `users.update` in the Node SDK.
  - **Inline**: pass recipient *objects*, not ID strings, in `workflows.trigger`. Knock guarantees the user is identified before the run starts. Passing `["user-1"]` does **not** identify the user; you need `[{ id: "user-1", name: ... }]`.
  - **Bulk**: `POST /v1/users/bulk/identify`, for imports.
- `KnockProvider` also **identifies the user client-side** when it mounts. If the user doesn't exist, it creates one from the `user` prop. Docs recommend server-side identification for anything beyond in-app display. ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients))
- Users are **per environment**. Development and production each hold their own user set. ([Users FAQ](https://docs.knock.app/concepts/users))
- Knock suggests **prefixing IDs by type** when you have different kinds of users, for example `customer_123`. ([Users FAQ](https://docs.knock.app/concepts/users))

**DevUI switching.** Cadence's `users.id` is an autoincrement integer, and `api.switch-user.ts` just rewrites the cookie session's user ID, so Knock user ID = `String(users.id)`. The React SDK handles a user change cleanly. When `user.id` (or `userToken`) changes, `useAuthenticatedKnockClient` re-authenticates in place, `knock.authenticate()` tears down and rebuilds the websocket, and `KnockFeedProvider` is keyed by `feedProviderKey(userId, feedId, options)`, so the feed subtree remounts and refetches. (Source: `react-core/src/modules/core/hooks/useAuthenticatedKnockClient.ts`, `react-core/src/modules/feed/context/KnockFeedProvider.tsx`, `client/src/knock.ts`.) Because the DevUI switch is a form POST that redirects, the page reloads anyway, so the new user's feed mounts fresh.

Caveat: the seed DB is re-creatable, so the same integer ID can belong to different people across reseeds. Knock keeps the old user's messages under that ID, so a reseed leaves stale feed items in the development environment. See the open questions.

## 2. Triggering workflows from server actions (Node SDK)

```ts
import Knock from "@knocklabs/node";
const knock = new Knock(); // reads KNOCK_API_KEY (secret key); KNOCK_BASE_URL optional

await knock.workflows.trigger("refund-request-submitted", {
  recipients: [{ id: String(admin.id), name: admin.name, email: admin.email }],
  actor: { id: String(student.id), name: student.name },
  data: { refundRequestId: 42, courseTitle: "..." },
  // tenant?: "...", cancellation_key?: "..."
});
```

- Signature from the 1.x README and `src/resources/workflows.ts`: `workflows.trigger(key, { recipients, actor?, data?, tenant?, cancellation_key?, settings? })`. It returns `{ workflow_run_id }`. ([README](https://www.npmjs.com/package/@knocklabs/node))
- **Recipient shapes.** A `RecipientRequest` is one of: ([Recipients](https://docs.knock.app/concepts/recipients))
  - a user ID string (the user must already be identified)
  - an inline user object `{ id, ...props }`
  - an inline object `{ id, collection, ...props }`
- **One vs many recipients.** The same call handles both; `recipients` is an array **capped at 1,000 per trigger** (SDK docstring on `WorkflowTriggerParams.recipients`). Knock fans out one workflow run per recipient and dedupes recipients.
- **Larger audiences use Objects and Subscriptions.** You model a topic as an Object (e.g. `{ collection: "courses", id: "17" }`), subscribe recipients to it (up to 100 per call, unlimited in total), then trigger the workflow with the object as the recipient. Knock fans out to the object **and** all its subscribers, deduped. ([Subscriptions](https://docs.knock.app/concepts/subscriptions), [Objects](https://docs.knock.app/concepts/objects)) This is the natural fit for a future "all Students enrolled in a course" Instructor announcement.
- **`actor`** is the person who caused the event. The feed shows actors (avatar and name) and counts unique actors in batches. ([In-app API](https://docs.knock.app/in-app-ui/api-overview))
- **`data`** has a 10 MB limit. Everything in `data` is passed through onto the feed item, so Cadence IDs can drive custom rendering and links. ([In-app API, "Passing through data"](https://docs.knock.app/in-app-ui/api-overview))
- **Idempotency.** Only `POST /workflows/:key/trigger` honors an `Idempotency-Key` header, with a 24-hour window. ([Idempotent requests](https://docs.knock.app/api-reference/overview/idempotent-requests)) The Node SDK attaches an auto-generated key to every non-GET request and reuses it across its own retries (default `maxRetries: 2`), so SDK retries are safe. To make *application-level* retries safe, pass `{ idempotencyKey: "refund-submitted:42" }` as the request option. (Source: `src/client.ts`, `src/internal/request-options.ts`.)
- **Rate limits** are tiered per endpoint, scoped per environment (and per user when a signed token is used), and return `429` when exceeded. ([Rate limits](https://docs.knock.app/api-reference/overview/rate-limits))
- The trigger is a plain outbound HTTPS call. It fits a React Router `action` the same way `sendEmail()` does in `app/lib/mailer.server.ts`. Workflows must exist and be **committed** in the target environment before a trigger works; an inactive workflow returns `workflow_inactive`. ([Triggering overview](https://docs.knock.app/send-notifications/triggering-workflows/overview))

## 3. In-app feed channel: components vs headless, SSR, user tokens

**Pre-built components** (`@knocklabs/react`): `KnockProvider` takes `apiKey` (public), `user`, and an optional `userToken`. Inside it, `KnockFeedProvider` takes `feedId` (the in-app channel ID). The UI pieces are `NotificationIconButton`, `NotificationFeedPopover`, and `NotificationFeed`. They need `import "@knocklabs/react/dist/index.css"` unless you replace the styles. ([React feed](https://docs.knock.app/in-app-ui/react/feed)) Knock also documents Toast and Inbox recipes. ([React SDK overview](https://docs.knock.app/in-app-ui/react/sdk/overview))

**Headless**: call `useKnockClient()`, then `useNotifications(knock, channelId, { mode? })` to get a `Feed`, then `useNotificationStore(feed)` to get `{ items, metadata }`. Call `feed.fetch()` yourself in an effect. You are responsible for `markAllAsSeen()` and `markAsSeen()`; otherwise `unseen_count` never drops. The default `compact` mode drops `activities`, extra actors, and nested `data`; `rich` returns everything. ([Headless feed](https://docs.knock.app/in-app-ui/react/headless/feed))

**SSR compatibility.** No Knock doc covers React Router or generic SSR. The only framework guide is Next.js, where the feed lives in a `"use client"` component and triggers run in a `"use server"` action. ([Next.js quick start](https://docs.knock.app/getting-started/quick-start/nextjs)) Reading the published source:

- `dist/esm/index.mjs` carries a `"use client"` directive. That does nothing in React Router 8, which renders the component on the server and then hydrates it.
- The websocket is guarded by `typeof window !== "undefined" && typeof WebSocket !== "undefined"`, and `BroadcastChannel` and `visibilitychange` are guarded too (`client/src/api.ts`, `pageVisibility.ts`, `clients/feed/feed.ts`). So **no socket opens during SSR**, and the pre-built feed fetches in `useEffect`.
- But `KnockProvider` builds the client and calls `knock.authenticate()` inside `useMemo`, and `useNotifications` calls `knock.feeds.initialize()` plus `listenForUpdates()` in a `useState` initializer. Both run during the server render. With the default `identificationStrategy: "inline"` and a `user` object, `authenticate()` fires an **identify HTTP request from the Node server on every SSR render**, and then again in the browser. (`client/src/knock.ts` lines ~163–180.)
- The server-rendered feed is empty or loading, then fills in after hydration. You can't server-render feed contents through these components.
- Practical mitigations:
  - Set `identificationStrategy="skip"`, which `KnockProvider` exposes, and identify server-side instead.
  - Or gate it with `enabled={isHydrated}`. `enabled` is a 0.17 prop: when false, the client stays idle with no identify call, no requests, and no socket.
  - Or render the Knock subtree client-only, behind a mounted flag.

**Signed user tokens (enhanced security mode)**: ([Security & authentication](https://docs.knock.app/in-app-ui/security-and-authentication))

- Enhanced security is **off by default**. While it's off, the public key can act as *any user*, so you must turn it on before production.
- With it on, every client request needs a JWT signed with RS256 using the environment's signing key, conventionally `KNOCK_SIGNING_KEY` in base64 PEM. The token carries `sub` = user ID, plus `iat` and `exp`.
- Node: `import { signUserToken } from "@knocklabs/node"` and call `await signUserToken(userId, { expiresInSeconds?, signingKey?, grants? })`. Expiry defaults to 3600 s. (Source: `src/lib/tokenSigner.ts`.) The doc page still shows the older `@knocklabs/node/lib/tokenSigner` import path, so use the README's root export.
- Pass the token to `KnockProvider` as `userToken`. The `onUserTokenExpiring` callback lets the client fetch a fresh token before it expires.
- Docs suggest matching token expiry to session expiry.
- Tokens can carry UCAN-style `grants` with `explicit_grants: true` to limit endpoints, and grants can scope a token to a tenant or an object.
- **Fit for Cadence:** the root loader already reads the session user, so it can call `signUserToken(String(userId))` and return `{ knockUserId, knockToken }`. No extra request, and a DevUI switch yields a new token on the reload.

## 4. Real-time updates and read / seen / archived states

- **Real-time.** Knock runs a managed websocket. The only event is `new-message`, which carries badge-count metadata rather than message contents, and "might be throttled". The SDK refetches on that event. ([In-app API](https://docs.knock.app/in-app-ui/api-overview), [Socket behavior](https://docs.knock.app/in-app-ui/feeds/socket-behavior))
- **Socket options** on `defaultFeedOptions`:
  - `auto_manage_socket_connection` (with `auto_manage_socket_connection_delay`, default 2000 ms) disconnects inactive tabs.
  - `__experimentalCrossBrowserUpdates` syncs tabs via `BroadcastChannel`.
  - ([React feed](https://docs.knock.app/in-app-ui/react/feed))
- **Engagement statuses.** These are independent flags, each with a timestamp. ([Message statuses](https://docs.knock.app/send-notifications/message-statuses), [In-app feed channel](https://docs.knock.app/integrations/in-app/feeds))
  - `seen` (`seen_at`) means the message was rendered in the feed. The pre-built popover marks everything seen on open.
  - `read` (`read_at`) means the user clicked it, or used "mark all as read".
  - `interacted` (`interacted_at`) means the cell or a button was clicked.
  - `link_clicked` requires link tracking.
  - `archived` (`archived_at`) is set by the cell's "x". Archived items are **excluded from the feed by default**; fetch them with `archived: "only"`.
- **Badge counts.** Feed metadata carries `unseen_count` and `unread_count`. The pre-built badge defaults to `unseen`; set `badgeCountType="unread"` to change it.
- Each status has mark and unmark endpoints plus batch and bulk variants, all callable with the public key. ([Security & authentication](https://docs.knock.app/in-app-ui/security-and-authentication))
- In-app messages skip straight to the `delivered` status. ([Message statuses](https://docs.knock.app/send-notifications/message-statuses))
- **Messages are immutable.** Their content can't be edited after send. To correct one, archive it and trigger again. ([Feeds overview FAQ](https://docs.knock.app/in-app-ui/feeds/overview))
- Feed order is reverse-chronological only, with no pinning. ([Feeds overview FAQ](https://docs.knock.app/in-app-ui/feeds/overview))
- Feed filters are `status`, `tenant` (+ `has_tenant`), `source` (workflow key), and `data` properties. ([Filtering feeds](https://docs.knock.app/in-app-ui/feeds/filtering-in-app-feeds))

## 5. Per-user preferences

- **PreferenceSet.** A user's `PreferenceSet` holds booleans under `categories`, `channels`, `channel_types` (`in_app_feed` is one type), and `workflows`, which can nest channel types. ([Preferences overview](https://docs.knock.app/preferences/overview))
- **Defaults and precedence.**
  - Anything unset defaults to `true`.
  - A category set to `false` blocks the workflow even if a channel is set to `true`.
  - With multiple categories, any single `false` blocks.
- **Evaluation and merging.** Knock evaluates preferences at trigger time and **automatically skips** opted-out recipients; skipped messages aren't billed. The merge order is recipient `default` set, then tenant default, then the environment-level default set.
- **Overrides.** A workflow can be set to "Override recipient preferences", for must-send notifications.
- **UI options.**
  - The **hosted preference center** is a signed URL, configured in the dashboard, and covers the `default` set only.
  - The **custom preference center** uses the `usePreferences` hook from `react-core` or `@knocklabs/client`. The public key can read and write the user's own preferences.
  - ([Preferences overview](https://docs.knock.app/preferences/overview), [React preferences](https://docs.knock.app/in-app-ui/react/preferences))
- **Plan limits.** Per-tenant preferences are an **Enterprise** feature; plain per-user preferences are on every plan. ([Pricing](https://knock.app/pricing))

## 6. Development vs production, and the free tier

- **Environments.** New accounts start with **production only**. Turning on "Enable environments" adds a **development** environment below production, and you **can't turn it off afterwards**. ([Environments](https://docs.knock.app/version-control/environments))
- **What lives per environment.**
  - Each environment has its own public and secret API keys and signing key, plus its own users, objects, tenants, messages, and logs. Data is never copied between environments.
  - Workflows are version-controlled: you create them in development, commit, and promote upward. Channel provider credentials and variables must be set per environment.
- **Enhanced security per environment.** You can leave enhanced security off in development and turn it on in production, as the docs suggest. ([Security & authentication](https://docs.knock.app/in-app-ui/security-and-authentication))
- **No offline mode.** Knock has no local emulator or offline mode in the docs or SDK. You can point the Node SDK elsewhere with `baseURL` / `KNOCK_BASE_URL` (source: `src/client.ts`), but there's no first-party mock.
- **Free plan.** The pricing page now calls the free tier **"Free"**, not "Developer". It costs $0 with no card. ([Pricing](https://knock.app/pricing))
  - **Included:** 10,000 messages sent per month (one message = one recipient on one channel, so an in-app item counts as 1), plus 500 guide recipients and 500 agent credits.
  - **Unlimited:** stored profiles, workflows, in-app channels/feeds, environments, and team members. Feed retention is unlimited.
  - **Also included:** batching, preferences, pre-built and bring-your-own components, Google SSO, and a 99.9% uptime SLA.
  - **Not included:** Knock branding can't be removed (that starts at Starter, $250/mo). Log retention is 30 days. No translations (i18n is Enterprise; `KnockProvider`'s `i18n` prop only overrides UI strings). No per-tenant preferences or branding.
  - **No hard cap:** going over 10k messages doesn't stop sending; Knock contacts you about adding billing.
  - **Not billed:** messages suppressed by preferences, and failed messages.

## 7. Constraints on how the feed looks

- **Theming** works through `--rnf-*` CSS variables (full list in [theme.css](https://github.com/knocklabs/javascript/blob/main/packages/react/src/theme.css)). You can also replace the stylesheet entirely, since all classes are prefixed `rnf-`. `colorMode="light"|"dark"` is available too. ([Styling](https://docs.knock.app/in-app-ui/feeds/styling))
- **Custom rendering**: ([React feed](https://docs.knock.app/in-app-ui/react/feed))
  - `renderItem` replaces the cell entirely, or wraps `NotificationCell` with children or a custom `avatar`.
  - `renderHeader` replaces the header and status filter.
  - `onNotificationClick` and `onNotificationButtonClick` handle clicks.
  - `initialFilterStatus` sets the starting filter.
  - Each item's `action_url` (set in the template) gives automatic click-through.
- **Content comes from Knock templates.** The template editor produces `blocks` (Markdown, rendered as HTML, plus text and button sets), and the pre-built cell renders those blocks with `dangerouslySetInnerHTML`. Copy lives in the Knock dashboard, not in Cadence code, unless you render your own text from `data`. ([In-app API](https://docs.knock.app/in-app-ui/api-overview))
- **Fully custom UI with shadcn and Tailwind is supported** through the headless hooks. You then own seen/read marking, the badge, filtering, and empty states.
- **The Free plan shows Knock branding** on the pre-built feed. Removing it starts at Starter. A headless UI doesn't render the pre-built footer; whether that's allowed on Free isn't documented (see open questions).
- **Batching** happens in the workflow (a batch step with an optional `batch_key`, windows up to 90 days, and sliding windows). The result is *one* feed item with `activities` and `total_activities`; read those in `rich` mode. ([Batch function](https://docs.knock.app/designing-workflows/batch-function))

## Implications for Cadence

1. **Server module mirroring the mailer.** Add a `notifier.server.ts` with a `NotificationTransport` interface, where `knockTransport(secretKey)` calls `workflows.trigger`. When `KNOCK_API_KEY` is absent, fall back to an `outboxTransport(".outbox/notifications")` that writes JSON, just like `mailer.server.ts`. Tests inject the outbox transport. Knock has no emulator, so this seam is the only offline option.
2. **Identity.** Knock user ID = `String(users.id)`. Identify inline on every trigger, sending `name`, `email`, and `avatar` from `users`. Set `identificationStrategy="skip"` on `KnockProvider` so SSR renders don't fire identify calls. Consider a prefix such as `cadence-user-17` so reseeds can be detected and pruned.
3. **Root loader.** Return `knockUserId`, `knockToken` (from `signUserToken`, only when a signing key is configured), the public key, and the feed channel ID. `KnockProvider` wraps the app shell with `enabled` set only after hydration and when a user exists. The DevUI switch then needs no special handling: the redirect reloads with a new user and token.
4. **Look.** Pick between restyling the pre-built popover with `--rnf-*` variables plus `renderItem` (fast; shows Knock branding on Free) and a headless shadcn UI (matches the app; Cadence owns mark-as-seen). This is the prototype ticket's call.
5. **Fan-out.** Up to 1,000 recipients per trigger covers per-event notifications. Future **Instructor announcements** fit Objects plus Subscriptions (one object per course, Enrollments as subscriptions) or a Knock Broadcast. Neither is precluded.
6. **Preferences.** Knock enforces preferences at trigger time with no Cadence code, as long as each Notification kind is its own workflow or carries a category. That makes "one workflow per Notification kind" the natural model.
7. **Environments.** Use Knock's development environment for local and staging keys, with enhanced security off. Use production with enhanced security on. Enabling environments is irreversible, which is fine.

## Open questions this research couldn't settle

- **Free-plan branding with a headless UI.** Is a "Powered by Knock" mark required when Cadence renders its own feed UI on Free? The pricing page lists "Remove Knock branding" for the pre-built components only.
- **React Router 8 runtime check.** Hydration cleanliness is unverified. Reading the source suggests no socket opens on the server and the feed renders empty until mount, but nobody has run it under React Router 8 SSR yet. A spike should confirm this and the `enabled` gating.
- **Stale users after reseed.** How should development-environment users and messages be handled when the seed DB is recreated and integer IDs get reused? Options: delete users via the API on seed, use a per-seed ID prefix, or accept stale data.
- **Rate-limit numbers.** The page lists tiers from 1 to 1,000 requests per second, but the fetched markdown didn't show which tier `workflows.trigger` and feed reads fall into. Check the API reference for each endpoint before any bulk backfill.
- **Who owns Notification copy.** Knock templates in the dashboard, which can't be reviewed in Git unless you use the Knock CLI, or Cadence-rendered text from `data`? This is a process decision, not a Knock fact.
