# 120 · Next.js Interview Questions

> **Next.js interview questions at senior/staff level probe the intersection of framework mechanics and engineering judgment: when to use SSR vs SSG vs ISR and WHY, how Next.js's caching layers interact, what happens during streaming SSR, how the App Router's server/client boundary works, and how to diagnose production issues specific to Next.js's architecture. This document covers the questions that distinguish engineers who have USED Next.js from those who UNDERSTAND it.**

---

## Questions by Category

- [Rendering Strategies](#rendering-strategies)
- [App Router Architecture](#app-router-architecture)
- [Caching](#caching)
- [Server Components and Server Actions](#server-components-and-server-actions)
- [Performance and Optimization](#performance-and-optimization)
- [Deployment and Infrastructure](#deployment-and-infrastructure)
- [Practical Debugging Scenarios](#practical-debugging-scenarios)

---

## Rendering Strategies

### Q1: Explain the difference between SSR, SSG, ISR, and PPR. When would you choose each?

**Complete Answer:**

**SSR (Server-Side Rendering):** The page is rendered on-demand for each request. The server fetches data, renders HTML, and sends it. The HTML reflects the state at the MOMENT OF THE REQUEST.

- Use when: data changes frequently and must be fresh per-request (user-specific pages, real-time dashboards, auth-dependent content)
- Next.js config: `export const dynamic = 'force-dynamic'` OR using `cookies()`, `headers()`, or `searchParams` (which Next.js detects and auto-makes dynamic)
- Cost: latency for every request (can't be CDN-cached at the page level)

**SSG (Static Site Generation):** The page is rendered ONCE at build time. The resulting HTML is served from a CDN.

- Use when: content changes infrequently (docs, blog posts, marketing pages), or can be fully determined at build time via `generateStaticParams`
- Cost: build time grows linearly with pages; rebuilding all pages to update any page

**ISR (Incremental Static Regeneration):** Pages are statically generated but can be revalidated in the background after a time interval or on-demand.

- Use when: content changes occasionally (product catalog, news articles) and some staleness is acceptable; or when you want CDN caching with controlled freshness
- Next.js config: `export const revalidate = 3600` (revalidate every hour) or via `revalidatePath()` / `revalidateTag()` for on-demand
- Mechanism: serves stale content while regenerating in the background (stale-while-revalidate)

**PPR (Partial Prerendering, Next.js 15 experimental):** A single page can contain BOTH static and dynamic content. Static sections are pre-rendered at build time; dynamic sections are streamed in at request time. The static shell is served from CDN instantly; dynamic parts stream in.

- Use when: most of a page is static but a few sections are personalized/dynamic
- Mechanism: Suspense boundaries mark the dynamic/static boundary; the static part is the Suspense fallback; the dynamic part streams in

**Decision framework:**

```
Is data user-specific (auth, session)? → SSR (or PPR if most is static)
Does data change per-request (search, real-time)? → SSR
Does data change occasionally (hourly/daily)? → ISR
Is data truly static (build-time determined)? → SSG
Can the page be split into static + dynamic sections? → PPR (if available)
```

_Handbook reference: Part XI (Next.js Rendering Systems)_

---

### Q2: What triggers a page to become dynamic in Next.js App Router?

**Complete Answer:**

Next.js uses STATIC ANALYSIS to determine if a route can be statically generated. A route becomes dynamic (cannot be pre-rendered at build time) when Next.js detects the use of any API that requires request-time information:

**Dynamic APIs (automatically trigger dynamic rendering):**

- `cookies()` — reads from the HTTP request's Cookie header
- `headers()` — reads from the HTTP request headers
- `searchParams` — the page's URL query parameters (only for `page.tsx`, not for layouts)
- `request` (in Route Handlers) — the raw Request object

**Why:** These APIs require access to the SPECIFIC REQUEST being processed. They return different values for different requests, so the output cannot be pre-computed at build time.

**What does NOT trigger dynamic rendering:**

- `params` — URL path parameters ARE available at build time via `generateStaticParams`
- `fetch()` — by default, Next.js CACHES fetch responses; the cached response is used at build time
- Database queries via the cached `fetch` — same as above

**Forcing behavior explicitly:**

```ts
export const dynamic = "force-dynamic"; // always dynamic, ignore caching
export const dynamic = "force-static"; // always static, error if dynamic APIs used
export const revalidate = 60; // ISR with 60-second revalidation
export const fetchCache = "force-cache"; // cache all fetch calls in this segment
```

**The diagnostic question in interviews:** "If a Next.js page uses `const user = await db.query(...)` directly (not via fetch), is it static or dynamic?" Answer: it depends on whether the query uses request-specific data. A query for all products → static (same result every time). A query for `WHERE userId = ${userId}` where userId comes from `cookies()` → dynamic.

_Handbook reference: Part XI (Rendering Systems), Part XII (Caching)_

---

## App Router Architecture

### Q3: Explain the server/client boundary in Next.js App Router. How does the 'use client' directive propagate through a component tree?

**Complete Answer:**

In Next.js App Router, all components are Server Components by default — they run ONLY on the server and produce zero client-side JavaScript.

**The 'use client' boundary:**

- Adding `'use client'` at the top of a file declares that file AND ALL COMPONENTS IT IMPORTS as Client Components
- 'use client' propagates DOWNWARD — once a component is marked as a Client Component, all its child imports are also Client Components (unless they have their own server-only protection via `import 'server-only'`)
- 'use client' does NOT propagate UPWARD — a parent Server Component can import a child Client Component

**What this means in practice:**

```
ServerComponent (no 'use client')
├── ServerLayout (no 'use client') → runs on server ✅
├── ClientSidebar ('use client') → runs on server AND client
│   └── ServerComputedList (no 'use client') → BUT this is imported by a Client Component,
│                                               so it becomes a Client Component too ⚠️
└── ClientButton ('use client') → runs on server AND client
```

**The "Client Component boundary" vs "Client Module boundary":**

- `'use client'` is a MODULE-level directive, not a component-level directive
- Everything imported by a 'use client' module becomes client-side, including non-React modules
- A Server Component CANNOT be imported INSIDE a Client Component's module (would be bundled to the client with all its server dependencies)
- BUT: a Server Component CAN be PASSED AS CHILDREN to a Client Component (because children is a prop, which is serializable as RSC payload)

**The correct pattern for Server Components inside Client Components:**

```tsx
// ✅ CORRECT: pass Server Component as children (prop) to Client Component
<ClientWrapper>
  <ServerComponent /> {/* This is a prop, not an import — stays server-side */}
</ClientWrapper>;

// ❌ WRONG: import and render Server Component inside Client Component module
("use client");
import ServerComponent from "./ServerComponent"; // ← bundled to client side!
```

_Handbook reference: Part X (Server Components)_

---

### Q4: How does Next.js App Router handle navigation differently from a traditional SPA?

**Complete Answer:**

Traditional SPAs (client-side routing with React Router or similar) navigate entirely in the browser: the JavaScript is already loaded, routing happens by re-rendering components, and data fetching is triggered by hooks in the newly rendered components.

**Next.js App Router navigation:**

**Initial load:**

- Server renders the full page as HTML + RSC payload
- Browser receives HTML (displays immediately)
- JavaScript bundle loads and hydrates only the Client Component parts
- Server Components are NOT hydrated (no JS for them)

**Subsequent navigation (client-side):**

- User clicks a `<Link>` component
- Next.js Router fetches the RSC payload for the target page (not full HTML)
- The payload is a serialized component tree describing the new page
- React merges the new RSC payload into the existing component tree (keeping Client Component state where layouts are shared)
- Only changed portions of the page update (layouts shared between routes are NOT re-mounted)

**The Router Cache (client-side):**

- Next.js caches RSC payloads in memory for routes visited in the current session
- Back/forward navigation serves from this cache (instant)
- `router.refresh()` invalidates the cache for the current route

**What makes this different:**

1. Shared layouts persist without re-mounting (unlike full page navigations)
2. No "loading" flash for routes that have been pre-fetched
3. `<Link>` prefetches the destination route's RSC payload on hover/viewport entry
4. Server Components re-render on EVERY navigation to their route (no client-side state for Server Components — they run on the server each time)

_Handbook reference: Part IX (Next.js Core - App Router), Part XII (Caching - Router Cache)_

---

## Caching

### Q5: Describe Next.js's four caching layers. How do they interact?

**Complete Answer:**

Next.js App Router has four distinct caching layers, each with different scope and invalidation mechanisms:

**1. Request Memoization (React `cache()`):**

- Scope: ONE server render request
- What it caches: the return value of a function, deduplicated by arguments
- Lifetime: a single render pass (reset per request)
- Purpose: prevent duplicate database/fetch calls within one page's render tree
- Example: `const getUser = cache(async (id) => db.users.findById(id))` — multiple Server Components calling `getUser('123')` within one render only hit the DB once

**2. Data Cache (Next.js's extended `fetch()` cache):**

- Scope: persistent across multiple requests and deployments
- What it caches: fetch() responses, keyed by URL + request options
- Lifetime: until explicitly revalidated (`revalidatePath`, `revalidateTag`) or the `revalidate` time elapses
- Purpose: avoid re-fetching the same external data on every request; ISR behavior
- Stored: filesystem (`.next/cache/fetch-cache`) or external (Vercel's data cache)
- Note: ONLY applies to `fetch()` — Prisma queries, native http, etc. are NOT cached

**3. Full Route Cache:**

- Scope: persistent across requests for STATICALLY rendered routes
- What it caches: the rendered HTML and RSC payload for static pages
- Lifetime: until the data cache is revalidated (invalidating data → route cache is invalidated too)
- Purpose: serve pre-rendered HTML without any server computation

**4. Router Cache (client-side):**

- Scope: the current browser session
- What it caches: RSC payloads for visited/prefetched routes
- Lifetime: during the session (configurable: `staleTime` in `next.config.js`)
- Purpose: instant navigation for previously visited routes; no server round-trip
- Invalidation: `router.refresh()`, or automatically on Server Actions that call `revalidatePath/Tag`

**Interaction:**
A request for a static page:
→ Router Cache HIT? → serve from browser memory (instant)
→ Full Route Cache HIT? → serve pre-rendered HTML from CDN
→ Data Cache HIT? → use cached fetch responses to render
→ Data Cache MISS → actually call fetch(), cache the result
→ Request Memoization → deduplicate within the render

_Handbook reference: Part XII (Next.js Caching Systems)_

---

### Q6: A team member says "my page isn't showing updated data even after deploying." Walk me through how you'd diagnose this.

**Complete Answer:**

This is a caching diagnostic question with multiple layers to check systematically:

**Step 1: Identify which cache layer is serving stale data**

Start by bypassing each layer:

a) **Client-side (Router Cache):** Force a hard refresh (Ctrl+Shift+R in Chrome). Does updated data appear? If yes → Router Cache is holding the stale page. Fix: call `router.refresh()` in a Server Action, or configure shorter `staleTimes` in next.config.js.

b) **CDN/Full Route Cache:** Open the URL in an incognito tab or with `curl -H "Cache-Control: no-cache"`. Does updated data appear? If yes → the CDN is serving a cached version. Fix: `revalidatePath()` or check the `revalidate` setting on the route.

c) **Data Cache:** Add `console.log` in the Server Component to see what data is being fetched. If the log shows stale data even in a new request → the `fetch()` call is hitting the data cache. Fix: check `next: { revalidate }` options on the fetch, call `revalidateTag()`, or use `cache: 'no-store'` to disable caching for that fetch.

d) **Database/External API:** Check if the data is actually updated at the source. Use Prisma Studio or direct database query. If the DB shows new data but the app shows old → it's a Next.js cache issue. If the DB also shows old data → it's a mutation issue, not a cache issue.

**Step 2: Implement the fix**

If the route should be dynamic (always fresh): `export const dynamic = 'force-dynamic'`

If the route should be ISR (fresh after N seconds): `export const revalidate = 60`

If specific data should be invalidated after a mutation:

```ts
"use server";
export async function updateProduct(id: string, data: ProductData) {
  await db.products.update({ where: { id }, data });
  revalidateTag(`product-${id}`); // invalidate all fetches with this tag
  revalidatePath("/products"); // invalidate the products listing page
}
```

**Step 3: Add observability**

In development: Next.js logs cache behavior in the terminal. In production: add logging to Server Components to confirm which path of the cache is being hit.

_Handbook reference: Part XII (Next.js Caching Systems)_

---

## Server Components and Server Actions

### Q7: What can Server Actions do that API Route Handlers cannot, and vice versa?

**Complete Answer:**

**Server Actions (advantages over Route Handlers):**

- **Progressive enhancement:** A form with a Server Action works without JavaScript (native form submission) — Route Handlers require JS to call via `fetch()`
- **Direct integration with React:** Server Actions can call `revalidatePath()`, `revalidateTag()`, `redirect()`, and `cookies()` — React and Next.js handle the orchestration
- **Automatic CSRF protection:** Next.js validates the `Origin` header for Server Actions; no manual CSRF token needed
- **Colocated with the page:** Server Actions can be defined in the same file as the Server Component using them (with 'use server' at the function level)
- **Type safety with no API layer:** TypeScript types flow directly from the Server Action function to the Client Component calling it

**Route Handlers (advantages over Server Actions):**

- **HTTP semantics:** Route Handlers support all HTTP methods (GET, PUT, DELETE) with full control over status codes, headers, and response format
- **External API access:** External clients (mobile apps, third-party services, webhooks) can call Route Handlers as a standard REST/JSON API
- **Streaming responses:** Route Handlers can return streaming responses for large data exports, SSE, or AI streaming
- **Webhook handling:** Incoming webhooks from Stripe, GitHub, etc. are received by Route Handlers (Server Actions are invoked by YOUR frontend, not external services)
- **Binary responses:** Route Handlers can return any content type (images, PDFs, binary); Server Actions return JavaScript-serializable data

**The practical split:**

- User-triggered mutations (form submit, button click) → Server Actions
- Webhooks, external API consumers, file downloads, SSE → Route Handlers
- Data fetching for initial page render → Server Components directly (no Action or Route Handler needed)

_Handbook reference: Part IX (Next.js Core - Server Actions), API Routes_

---

## Performance and Optimization

### Q8: How does Next.js implement streaming SSR? What happens to the HTML structure?

**Complete Answer:**

Next.js's streaming SSR uses HTTP chunked transfer encoding to send HTML to the browser progressively as it becomes available, rather than waiting for the complete render.

**The mechanism:**

1. React's `renderToReadableStream` produces a `ReadableStream` of HTML chunks
2. Next.js returns this stream as the HTTP response with `Transfer-Encoding: chunked`
3. The browser receives and renders chunks as they arrive (showing content earlier)

**What the HTML looks like:**

For a page with a Suspense boundary:

```html
<!-- CHUNK 1 (sent immediately): -->
<html>
  <head>
    ...
  </head>
  <body>
    <div id="__next">
      <div class="shell">
        <header>...</header>
        <!--$?-->
        <!-- Suspense fallback placeholder -->
        <template id="B:0"></template>
        <div class="skeleton">Loading...</div>
        <!--/$-->
        <footer>...</footer>
      </div>
    </div>
  </body>

  <!-- CHUNK 2 (sent when the async component resolves, e.g., 800ms later): -->
  <div hidden id="S:0">
    <!-- The real content that replaces the skeleton: -->
    <div class="product-list">...</div>
  </div>
  <script>
    // Client-side script that swaps the fallback for the real content:
    $RC("B:0", "S:0"); // React's built-in "replace content" function
  </script>
</html>
```

**Key observations from this structure:**

- The static shell (header, footer, skeleton) is in CHUNK 1 — users see it immediately
- The dynamic content arrives later in CHUNK 2 — the `$RC` script swaps it in
- Suspense boundaries in RSC work out-of-order: if a footer's data resolves before a header's data, footer content arrives first in the stream
- The `hidden` attribute on the `S:0` div means it's in the DOM but invisible until the swap script runs

**Why this matters for LCP:** The SKELETON itself can be designed to be approximately the size of the real content, minimizing layout shift when content arrives. And the shell HTML is browser-renderable immediately, giving a good Time to First Byte + Fast First Contentful Paint.

_Handbook reference: Part XI (Next.js Rendering - Streaming), Part X (Server Components - Streaming)_

---

## Deployment and Infrastructure

### Q9: What's the difference between deploying Next.js on Vercel vs self-hosting? What architectural decisions change?

**Complete Answer:**

**Vercel deployment (native Next.js platform):**

- **Edge Network:** Static assets, full route cache, and some middleware automatically served from Vercel's global CDN
- **Serverless Functions:** Each Next.js route handler and page render runs as a separate serverless function (cold starts are a consideration)
- **Edge Runtime:** Middleware and edge-configured routes run on Vercel's V8-based edge network (faster, lower latency, but no Node.js APIs)
- **Automatic ISR:** Vercel's infrastructure handles ISR's "stale-while-revalidate" logic and revalidation automatically
- **Remote Cache:** Turborepo remote caching integrated; build caching across PRs
- **Streaming:** Vercel's infrastructure supports response streaming natively

**Self-hosting (Node.js server):**

- Run `next start` on a Node.js process — a traditional HTTP server, not serverless
- Persistent connection handling: WebSockets work natively (serverless can't maintain them)
- **Caching changes:** The Full Route Cache writes to the local filesystem — if you have multiple server instances (horizontal scaling), they don't share cache → use Redis or an external cache adapter
- **ISR:** Works via the Data Cache persisted to filesystem — multiple instances again need shared storage
- **No automatic Edge:** You must configure your own CDN (Cloudflare, AWS CloudFront) in front of Next.js
- **Custom server:** You can use `server.js` to add WebSocket support, custom middleware, or a different HTTP library

**Key questions when self-hosting:**

1. How do you handle horizontal scaling? (shared cache layer needed)
2. How do you configure CDN caching for static assets and ISR pages?
3. How do you handle Edge Middleware? (run separately or in Node.js server process)
4. What is your deployment pipeline for deploying without downtime? (blue/green or rolling)

**The honest tradeoff:** Vercel is the path of least resistance for Next.js-specific features, but creates vendor lock-in. Self-hosting gives full control but requires infrastructure investment to replicate what Vercel provides out of the box.

_Handbook reference: Part IX (Next.js Core)_

---

## Practical Debugging Scenarios

### Q10: A Next.js page shows content from 3 hours ago even though the database was updated. The `revalidate` is set to 60 seconds. What's wrong?

**Complete Answer (structured as an interview discussion):**

"There are several possible explanations, and I'd investigate them in order:

**1. Is the revalidation actually triggering?**

`export const revalidate = 60` means Next.js will RE-FETCH the data after 60 seconds have elapsed. But the key word is 'after a request arrives for that page AFTER 60 seconds has elapsed.' If nobody visits the page for 60 minutes, the data isn't automatically re-fetched — ISR is stale-while-revalidate, not a background polling mechanism.

Check: has anyone actually visited the page after the 60-second window? If yes, continue.

**2. Is the Data Cache invalidating or bypassing?**

The `revalidate` export on the page sets the `fetch` cache's revalidation time. But this only applies to `fetch()` calls. If the data is fetched via Prisma directly (not via `fetch()`), it's NOT covered by the data cache and the `revalidate` setting has no effect on that data source.

Check: how is the data being fetched? If via Prisma/direct DB: add a `fetch()` layer or use `noStore()` to opt into dynamic rendering.

**3. Is the CDN caching the response?**

If there's a CDN in front of Next.js (Cloudflare, AWS CloudFront), the CDN might be caching the page HTML independently, overriding Next.js's revalidation. The CDN's cache-control headers need to allow Next.js's ISR behavior.

Check: the `Cache-Control` header on the response. Vercel sets `s-maxage=1, stale-while-revalidate=60` for ISR pages. A custom CDN that ignores this and caches for 24 hours would cause exactly this symptom.

**4. Router Cache on the client:**

If the reporter is testing by clicking back/forward without hard-refreshing, the Router Cache (client-side) might be serving the stale page from memory. A hard refresh or incognito window would bypass this.

**5. The mutation path is wrong:**

If the data was updated directly in the database (bypassing the Next.js application), `revalidatePath()` was never called, and the Data Cache has no reason to know the data changed. The ISR 60-second window might not have expired yet (or the Cache Entry's timestamp reset when the page was last visited).

In an interview setting, I'd walk through these in order, check each with a concrete diagnostic step, and fix the root cause."

_Handbook reference: Part XII (Caching), Part XIII (Next.js Advanced Internals)_

---

### Q11: How would you architect a Next.js application that needs to support both anonymous users (fully cached, CDN-served) and authenticated users (personalized content)?

**Complete Answer:**

This is a common real-world architectural challenge. The solution involves separating the static and dynamic content at multiple levels:

**Approach 1: Edge middleware for route-level separation**

```ts
// middleware.ts
export function middleware(request: NextRequest) {
  const session = request.cookies.get("session");

  if (!session) {
    // Anonymous: rewrite to a static/CDN-cached path
    return NextResponse.rewrite(
      new URL("/static" + request.nextUrl.pathname, request.url),
    );
  }
  // Authenticated: pass through to dynamic rendering
  return NextResponse.next();
}
```

**Approach 2: Partial Prerendering (PPR — Next.js 15+)**

- The static shell (navigation, product listing, footer) is pre-rendered and CDN-cached
- Dynamic sections (user greeting, cart count, personalized recommendations) are Suspense-wrapped
- The static shell is served instantly from CDN; dynamic sections stream in per-request

**Approach 3: Static page + client-side personalization**

- The page is statically generated (fully CDN-cacheable)
- Personalized data (user name, cart, recommendations) is fetched client-side after hydration
- This is the classic "shell + client fetch" pattern — good for pages where the delay in personalized content is acceptable

**Approach 4: Vary by cookie (CDN configuration)**

- Configure the CDN to cache separate versions based on the session cookie presence
- `Vary: Cookie` header tells CDNs to cache different responses for different cookie values
- Works but can cause CDN cache pollution (many variations → less effective caching)

**My recommendation for most applications:**

- Static, anonymous pages: fully static (SSG/ISR), no personalization
- Authenticated pages: Server Components with `force-dynamic` (or PPR once stable)
- The separation between "the public site" and "the authenticated app" is the clearest architectural boundary — deploy them to different routes or subdomains with different caching strategies

_Handbook reference: Part XI (Rendering Strategies), Part IX (Middleware)_

---

## Further Reading

- [Next.js documentation: App Router](https://nextjs.org/docs/app) — the authoritative reference
- [Next.js documentation: Caching](https://nextjs.org/docs/app/building-your-application/caching) — the four-layer cache documentation
- [Vercel: How Next.js works](https://vercel.com/blog/how-react-18-improves-application-performance) — architectural deep dives
- [Next.js RFC: Partial Prerendering](https://nextjs.org/blog/next-14#partial-prerendering-preview) — PPR architectural overview
- Related handbook sections: Parts IX–XIII (Next.js internals)
- Next in this handbook: [121 · System Design for Frontend Engineers](./03-system-design.md)

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
