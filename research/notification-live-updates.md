# How to show a Notification without a page reload

Research for #5 (part of map #2, In-app Notifications). Checked on 2026-11-17 against React Router 8.3.0 (`package.json`), `ssr: true` (`react-router.config.ts`), and ADR 0006 (one SQLite file, `better-sqlite3`, Drizzle).

## Answer

Put the unread count in the `layout.app.tsx` loader, so every navigation refreshes it for free. On top of that, poll a small resource route with `useFetcher().load()` every 30 seconds while the tab is visible, and once when the window regains focus. Do not poll with `useRevalidator`, and do not build Server-Sent Events (SSE) now.

Why: the first part costs nothing extra, because React Router already re-runs active loaders on every navigation. The second part costs one indexed `COUNT` query per open tab per 30 seconds, which a synchronous SQLite read handles easily, and it works the same on one Node process or on Vercel. SSE gives updates within a second, but it needs in-memory pub/sub (only correct on one process) or Redis (on Vercel), and every open tab holds a function open until the duration limit. A course platform does not need sub-second Notifications.

## What the stack does today

- `app/routes/layout.app.tsx` has a loader that wraps every signed-in page. It already runs several synchronous queries (users, current user, recent courses with progress, team admin check).
- `app/db/index.ts` opens `data.db` with `better-sqlite3` in WAL mode. Queries are synchronous: no network round trip.
- `sonner` (a toast library) is already a dependency, so showing a toast when the count goes up needs no new package.

## Option 1: revalidate on navigation only

**How it works.** In framework mode with SSR, "route loaders are automatically revalidated after all navigations and form submissions" ([Route Module, shouldRevalidate](https://reactrouter.com/start/framework/route-module)). The [revalidation guide](https://reactrouter.com/how-to/optimize-revalidation) states the default as "active loaders are revalidated on navigations and successful submissions (`Link`, `Form`, `fetcher.submit`)". So if the layout loader returns `unreadNotificationCount`, the badge updates on every link click and every form post, with no new code.

**Load on SQLite.** One extra query per navigation, inside a loader that already runs. With an index on `(recipient_user_id, read_at)` the count is an index lookup.

**Vercel.** Ordinary request and response. No limits apply beyond normal function calls.

**Complexity.** Lowest. One field in an existing loader.

**Weakness.** A user who stays on one page (watching a lesson video, which is the main thing users do here) never sees a new Notification until they click something.

## Option 2: polling every N seconds

### 2a. `useRevalidator` on an interval

**How it works.** `useRevalidator()` returns `revalidate()`, meant to "revalidate the data on the page for reasons outside of normal data mutations like Window focus or polling on an interval" ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)).

**Load on SQLite.** Re-runs every active loader on the page: the whole layout loader plus the current page's loader. On a lesson page that is the user list, recent courses with progress counts per course, and the lesson data, every N seconds per tab, to learn one number. Wasteful.

**Verdict.** Works, but reloads far more than it needs to. Not recommended.

### 2b. `useFetcher().load()` against a resource route (recommended)

**How it works.** A resource route is a route module that "exports a loader or action but does not export a default component" ([Resource Routes](https://reactrouter.com/how-to/resource-routes)). For routes called from fetchers, the docs say to "return `data()`". Fetchers load data "without causing a navigation" ([useFetcher](https://reactrouter.com/api/hooks/useFetcher)). A small component in the layout calls `fetcher.load("/api/notifications/unread")` on a `setInterval`, skips the call when `document.visibilityState === "hidden"`, and calls it once on the `focus` / `visibilitychange` event. When the count rises, it shows a `sonner` toast.

A fetcher's loaded data is also revalidated "after action submissions and explicit `useRevalidator` calls" ([revalidation guide](https://reactrouter.com/how-to/optimize-revalidation)), so marking a Notification as read through an action refreshes the badge too.

**Load on SQLite.** One indexed `COUNT` per visible tab per interval. At 30 seconds, 200 open tabs is about 7 requests a second. `better-sqlite3` reads are synchronous and local, and WAL mode lets reads run while a write is in progress, so this is small. Hidden tabs cost nothing.

**Vercel.** Each poll is a short normal request, well inside the default 300 s maximum duration ([Functions limits](https://vercel.com/docs/functions/limitations)). Vercel's own realtime guide lists polling as the option when "a few seconds of staleness is acceptable" ([Publish and subscribe to realtime data](https://vercel.com/kb/guide/publish-and-subscribe-to-realtime-data-on-vercel)). Cost is many short invocations, billed on active CPU and memory time.

**Complexity.** Low. One resource route, one client component with an interval and a visibility check.

**Weakness.** Up to N seconds of delay.

## Option 3: Server-Sent Events through a resource route

**How it works.** A resource route loader returns `new Response(stream, { headers: { "Content-Type": "text/event-stream" } })` and keeps the stream open. The browser uses `EventSource`, which reconnects on its own ("Clients will reconnect if the connection is closed") and sends `Last-Event-ID` on reconnect so the server can resume ([HTML spec, server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)). The React Router docs do not cover event streams; the resource route guide only shows returning a `Response`, which is enough to build one by hand.

**How the stream learns about a new Notification.** Either:
- an in-memory event emitter that the code creating a Notification calls. Correct only while the app runs as one process, which ADR 0006 assumes; or
- each open stream queries SQLite every few seconds. That is polling again, moved to the server, while also holding a connection open.

**Load on SQLite.** Lowest with the in-memory emitter (no reads until something happens). Same as polling with the server-side loop.

**Vercel.** Streaming is on by default for Node.js functions. The maximum duration "includes time spent processing the request and sending the response, including streamed responses": 300 s on Hobby, 800 s maximum on Pro ([Functions limits](https://vercel.com/docs/functions/limitations)). So every connection is cut at the limit and the browser reconnects. Vercel's guide says "no two instances share memory", so an in-memory emitter does not work across instances; it points to Redis as the pub/sub layer ([Publish and subscribe to realtime data](https://vercel.com/kb/guide/publish-and-subscribe-to-realtime-data-on-vercel)). Memory is billed for the whole time a connection is open, even when idle (same guide). There is also a cap of 1,024 file descriptors "shared across all concurrent executions" ([Functions limits](https://vercel.com/docs/functions/limitations)), and every open connection uses one.

**Complexity.** Highest. Stream setup and cleanup on disconnect, heartbeats, reconnect handling, an emitter or Redis, and testing all of that.

**Strength.** Sub-second delivery.

## Comparison

| | Delay | SQLite load | Vercel fit | Work |
| --- | --- | --- | --- | --- |
| 1. Navigation only | Until next click | +1 query per navigation | Fine | Very low |
| 2a. `useRevalidator` poll | N seconds | Every active loader, every N s | Fine | Low |
| 2b. Fetcher poll of resource route | N seconds | 1 indexed count per visible tab per N s | Fine | Low |
| 3. SSE | Under 1 s | Lowest with emitter | Needs Redis; cut at 300/800 s; memory billed while open | High |

## A Vercel fact that matters more than this choice

Vercel functions have "a read-only filesystem with writable `/tmp` scratch space up to 500 MB", files in `/tmp` "belong to a single function instance" and "don't persist after the instance stops", and "a file-based database such as SQLite can't persist your application's data in a function" ([Is SQLite supported in Vercel?](https://vercel.com/kb/guide/is-sqlite-supported-in-vercel)). So `data.db` cannot live on Vercel as ADR 0006 stands. That is a hosting question outside this ticket. The recommended option (1 + 2b) does not depend on it: it works unchanged on one Node server today and on Vercel with a hosted database later. SSE with an in-memory emitter is the only option that would break on that move.

## Facts later tickets depend on

- Framework mode with SSR re-runs active loaders on every navigation and successful submission. A count returned from `layout.app.tsx` stays current on clicks with no extra code.
- A resource route has a loader and no default export; call it with `useFetcher().load()` and return `data()`.
- Fetcher-loaded data revalidates after actions, so a "mark as read" action refreshes the badge.
- Add an index covering the unread lookup, for example `(recipient_user_id, read_at)` on the Notifications table, since both the layout loader and the poll run it often.
- Suggested interval: 30 s, paused while the tab is hidden, plus one call on focus. Tune later if needed.
- Use `sonner` (already installed) for the toast when the count goes up.
- If sub-second delivery is ever required, SSE needs a shared pub/sub (Redis) once the app runs on more than one instance.

## Sources

- React Router, Route Module, `shouldRevalidate`: https://reactrouter.com/start/framework/route-module
- React Router, Revalidation Optimization: https://reactrouter.com/how-to/optimize-revalidation
- React Router, `useRevalidator`: https://reactrouter.com/api/hooks/useRevalidator
- React Router, `useFetcher`: https://reactrouter.com/api/hooks/useFetcher
- React Router, Resource Routes: https://reactrouter.com/how-to/resource-routes
- Vercel, Functions Limits: https://vercel.com/docs/functions/limitations
- Vercel, Streaming: https://vercel.com/docs/functions/streaming-functions
- Vercel KB, Publish and subscribe to realtime data on Vercel: https://vercel.com/kb/guide/publish-and-subscribe-to-realtime-data-on-vercel
- Vercel KB, Is SQLite supported in Vercel?: https://vercel.com/kb/guide/is-sqlite-supported-in-vercel
- WHATWG HTML, Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- This repo: `app/routes/layout.app.tsx`, `app/db/index.ts`, `react-router.config.ts`, `package.json`, `docs/adr/0006-one-sqlite-file-through-drizzle.md`
