# 121 · System Design for Frontend Engineers

> **Frontend system design interviews at senior/staff level test whether a candidate can reason about architectural trade-offs at scale: not just "how do you build a search box" but "how do you build a search box that handles 10 million users, works on 2G connections, and stays maintainable across 15 engineers." This document covers the framework for approaching frontend system design questions, the specific design dimensions that matter, worked examples at the depth expected in interviews, and the common mistakes candidates make that reveal a lack of architectural thinking.**

Frontend system design is different from backend system design: the bottlenecks are browser constraints (JavaScript execution time, network bandwidth, rendering budget), the failure modes are user-visible (janky scroll, blank page, stale data), and the "distribution" challenge is pushing code to millions of client devices rather than running services on controlled servers.

---

## Table of Contents

- [The Frontend System Design Framework](#the-frontend-system-design-framework)
- [Design Dimensions: The Standard Checklist](#design-dimensions-the-standard-checklist)
- [Worked Example 1: Design a Product Search Feature](#worked-example-1-design-a-product-search-feature)
- [Worked Example 2: Design an Infinite Scroll News Feed](#worked-example-2-design-an-infinite-scroll-news-feed)
- [Worked Example 3: Design a Real-Time Collaborative Document Editor](#worked-example-3-design-a-real-time-collaborative-document-editor)
- [Common Design Challenges and Solutions](#common-design-challenges-and-solutions)
- [Trade-off Articulation: How to Sound Senior](#trade-off-articulation-how-to-sound-senior)
- [What Interviewers Are Actually Evaluating](#what-interviewers-are-actually-evaluating)

---

## The Frontend System Design Framework

Use this structure for any frontend system design question. Spend roughly 3-5 minutes on each phase:

```
PHASE 1: CLARIFY REQUIREMENTS (5 minutes)
  Functional requirements: what must it DO?
  Non-functional requirements: scale, performance targets, constraints
  User characteristics: device types, network conditions, accessibility needs
  Team constraints: how many engineers, what's the existing stack

  SAMPLE CLARIFYING QUESTIONS:
  "How many concurrent users?" → affects real-time architecture
  "What are the target Core Web Vitals thresholds?" → affects rendering strategy
  "Mobile first or desktop first?" → affects component architecture
  "Are we building this as a new product or within an existing system?"
  "What's the expected data volume per page?" → affects pagination/virtualization

PHASE 2: HIGH-LEVEL ARCHITECTURE (10 minutes)
  Component hierarchy and data flow
  State management strategy (local vs global, client vs server)
  Rendering strategy (SSR/SSG/CSR/streaming)
  API integration approach (REST/GraphQL/RSC)
  Data flow diagram: where does data originate, where does it end up

PHASE 3: COMPONENT DESIGN (10 minutes)
  Key components and their APIs (props interface)
  State structure and ownership
  Interaction patterns and event handling
  Loading, error, and empty states for each section

PHASE 4: PERFORMANCE (5 minutes)
  Critical rendering path optimization
  Code splitting boundaries
  Caching strategy
  Network optimization (batching, prefetching, compression)

PHASE 5: ACCESSIBILITY AND EDGE CASES (5 minutes)
  Keyboard navigation
  Screen reader support
  Error handling
  Offline capability (if relevant)
  Internationalization considerations

PHASE 6: TRADE-OFFS AND ALTERNATIVES (5 minutes)
  What did you choose and WHY over alternatives?
  What would you change with more time or different constraints?
```

---

## Design Dimensions: The Standard Checklist

Cover EACH of these in any system design question — skipping one signals a gap:

```
1. COMPONENT ARCHITECTURE
   - What are the key components? (atomic → composite → page level)
   - What's the component hierarchy?
   - What are the component APIs (props interfaces)?
   - How is content composition achieved? (slots, render props, compound components)

2. STATE MANAGEMENT
   - What state exists? (server state vs client state)
   - Where does each piece of state live? (local → context → global store)
   - How does state update propagate? (unidirectional, how controlled)
   - What's the state update flow for the primary user action?

3. RENDERING STRATEGY
   - SSR vs SSG vs CSR vs streaming?
   - What's rendered server-side vs client-side?
   - What's the hydration strategy?
   - How does rendering change for authenticated vs anonymous users?

4. DATA FETCHING
   - Which API endpoints are needed?
   - What's the request/response shape?
   - How is caching handled? (stale-while-revalidate, max-age, tags)
   - How are loading and error states handled?
   - Is there pagination, infinite scroll, or cursor-based loading?

5. PERFORMANCE
   - What's the critical rendering path?
   - Where are code-split boundaries?
   - What's prefetched? What's lazy-loaded?
   - How are images and fonts handled?
   - What are the bundle size constraints?

6. REAL-TIME (if applicable)
   - WebSocket vs SSE vs polling?
   - How is the connection managed (reconnection, cleanup)?
   - How is optimistic UI handled?

7. ACCESSIBILITY
   - Keyboard navigation flow
   - Focus management for dynamic content
   - ARIA roles and live regions
   - Color contrast and text sizing

8. ERROR HANDLING
   - Network failures
   - Partial data scenarios
   - Rate limiting / quota exceeded
   - Session expiry during an operation

9. SECURITY
   - XSS prevention (user-generated content)
   - CSRF for mutations
   - Sensitive data handling

10. MONITORING AND OBSERVABILITY
    - Which metrics matter? (Core Web Vitals specific to this feature)
    - What errors should be tracked?
    - What user interactions indicate success/failure?
```

---

## Worked Example 1: Design a Product Search Feature

**Prompt:** "Design a product search feature for an e-commerce platform with 10 million products and 1 million monthly active users."

### Phase 1: Requirements Clarification

```
FUNCTIONAL:
  - User types a query → results appear as they type (autocomplete/instant search)
  - Results show: product name, price, image, rating
  - Filters: category, price range, rating, in-stock
  - Sorting: relevance, price asc/desc, rating, newest
  - Pagination or infinite scroll for results
  - "No results" state with suggestions
  - Search history (optional based on interview direction)

NON-FUNCTIONAL:
  - Latency: results appear within 200ms of last keystroke (or after 300ms debounce)
  - Availability: search must work even if secondary features (filters) fail
  - Scale: 1M MAU, assume 10% concurrent peak = 100K concurrent users
  - SEO: search result pages should be indexable (implies SSR for result pages)
  - Accessibility: fully keyboard-navigable, screen reader compatible
```

### Phase 2: High-Level Architecture

```
RENDERING STRATEGY:
  Search landing page (/search): SSR with query params → SEO-indexable result pages
  Autocomplete dropdown: CSR (triggered by user input, no SEO value)

  When user types:
    Debounced (300ms) → fetch autocomplete suggestions from search service
    → display in dropdown (CSR, no SSR needed — suggestions change per keystroke)

  When user submits search:
    → Navigate to /search?q=query&category=X&sort=relevance
    → Server renders results page (SSR for SEO) using URL params
    → Subsequent filter/sort changes: update URL params → Next.js re-fetches

DATA FLOW:
  URL (single source of truth for search state)
    ↓ (parsed by SearchPage server component)
  Search API (/api/search?q=...&filters=...&sort=...&page=1)
    ↓
  SearchResults component (RSC: renders products list)
  + SearchFilters component (RSC: renders available filters)
  + SearchPagination component (RSC: renders page controls)

  Autocomplete:
  User input → debounce(300ms) → /api/autocomplete?q=... → dropdown
```

### Phase 3: Component Design

```tsx
// Key component hierarchy:
<SearchPage>
  {" "}
  // Server Component (RSC) — receives searchParams
  <SearchHeader>
    <SearchInput /> // Client Component — handles user input, autocomplete
    <SearchFilters /> // Could be RSC (current filter state from URL)
  </SearchHeader>
  <SearchBody>
    <Suspense fallback={<ResultsSkeleton />}>
      <SearchResults /> // Server Component — fetches and renders products
    </Suspense>
    <SearchPagination /> // Server Component — page controls
  </SearchBody>
</SearchPage>;

// SearchInput API (Client Component — needs useState, useCallback):
interface SearchInputProps {
  initialQuery: string; // from URL (for controlled initial state)
  onSearch: (query: string) => void; // updates URL params
}

// The autocomplete state is LOCAL to SearchInput:
// [query, setQuery] — what's typed
// [suggestions, setSuggestions] — fetched suggestions
// [isOpen, setIsOpen] — dropdown visibility

// SearchResults API (Server Component):
interface SearchResultsProps {
  query: string;
  filters: FilterParams;
  sort: SortOption;
  page: number;
}
// Fetches from search API server-side, renders product cards
```

### Phase 4: Performance

```
DEBOUNCING: 300ms debounce on autocomplete fetch
  → prevents API flood on every keystroke
  → 300ms feels instant to users; 500ms+ starts feeling laggy

CACHING:
  Autocomplete responses: cache with SWR/TanStack Query, stale time 30s
  (same query → no refetch; responses are re-shown from cache)

  Search result pages (/search?q=...): ISR with revalidate=60
  (product catalog changes infrequently; 1-minute staleness acceptable)
  EXCEPT: in-stock filter should be dynamic (inventory changes frequently)

CODE SPLITTING:
  The autocomplete dropdown component: dynamic import (not needed on initial load)
  The filter panel: lazy on mobile (hidden behind a button)

IMAGE OPTIMIZATION:
  Product thumbnails in search results: next/image with:
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 25vw"
    loading="lazy" (below-the-fold results)
    priority={true} (first 4 results only — likely above fold)

URL AS STATE:
  The search state (query, filters, sort, page) lives in the URL.
  Benefits:
    - Shareable search results
    - Browser back/forward works correctly
    - Server can render the exact state (no client-side state bootstrap problem)
    - SSR can render the results page with SEO-appropriate metadata
```

### Phase 5: Accessibility

```
AUTOCOMPLETE KEYBOARD SUPPORT:
  ArrowDown/Up: navigate suggestions
  Enter: select highlighted suggestion or submit current query
  Escape: close suggestions, return focus to input
  Tab: close suggestions, move to next element

  ARIA attributes:
    <input role="combobox" aria-expanded={isOpen} aria-haspopup="listbox"
           aria-controls="search-suggestions" aria-autocomplete="list">
    <ul role="listbox" id="search-suggestions">
      <li role="option" aria-selected={isHighlighted}>...suggestion...</li>

RESULTS ANNOUNCEMENT:
  <p aria-live="polite" aria-atomic="true" className="sr-only">
    {results.total} results for "{query}"
  </p>
  → Announced to screen readers when results load without focus movement
```

---

## Worked Example 2: Design an Infinite Scroll News Feed

**Prompt:** "Design a social media news feed with infinite scroll for a platform with 500K DAU."

### Key Design Decisions (interview-format discussion)

```
PAGINATION STRATEGY: Cursor-based (not page-number based)
  WHY: Feed is continuously updated. If user reads 50 posts on "page 1" and new posts
  appear, "page 2" would contain duplicates. Cursor-based pagination gives them
  "posts after cursor X" which is stable as new content arrives.

  API design:
  GET /api/feed?cursor={lastPostId}&limit=20
  Response: { posts: Post[], nextCursor: string | null }

RENDERING:
  Initial load (first 20 posts): SSR for fast FCP and SEO
  Subsequent loads (as user scrolls): Client-side (append to existing DOM)

  Server Component renders initial posts; Intersection Observer triggers
  client-side fetches for additional pages as user approaches the bottom.

STATE MANAGEMENT:
  TanStack Query's useInfiniteQuery:
    - Handles cursor-based pagination natively
    - Caches each "page" of results
    - Background refresh for the first page (new posts appear automatically)
    - Handles loading, error, and refetch states

VIRTUALIZATION:
  At 500K DAU with users potentially loading 100+ posts in one session:
  Without virtualization: 100+ complex post DOM nodes → memory + scroll performance issues
  WITH react-window / TanStack Virtual: only visible posts in the DOM

  Implementation: useVirtualizer from @tanstack/virtual
  Caveat: virtualization with variable-height items (different post types) is complex
  → Pre-measure each item type, use estimated heights with dynamic updates

REAL-TIME NEW POSTS:
  Option A: "X new posts, click to load" banner (Twitter/X pattern)
    → Poll every 30s for new post count; show badge when count > 0
    → User clicks → jump to top + load new posts
    → Simple, avoids scroll position disruption

  Option B: Auto-insert new posts at top (problematic)
    → Constantly shifts scroll position as new content arrives
    → Creates disorienting UX; generally avoided

  Recommendation: Option A — less disruptive, user controls when they see new content
```

---

## Worked Example 3: Design a Real-Time Collaborative Document Editor

**Prompt:** "Design a real-time collaborative text editor like Google Docs."

### Architecture Discussion

```
THE CORE TECHNICAL CHALLENGE: Operational Transformation (OT) or CRDTs
  When two users edit simultaneously, their changes can conflict.
  Simple "last write wins" → data loss.

  OT: operations are transformed relative to concurrent operations
  CRDT: data structures designed for automatic conflict-free merging

  For an interview: acknowledge this is complex, propose using a library
  (Yjs or Automerge are the standard choices for CRDTs in the JS ecosystem)
  rather than implementing from scratch.

TRANSPORT: WebSocket (not SSE)
  WHY WebSocket: document editing requires BIDIRECTIONAL, low-latency communication.
  User A types → must send to server → server broadcasts to User B → B's editor updates
  SSE is unidirectional (server → client only); polling has too much latency.

ARCHITECTURE WITH YJS + NEXT.JS:
  - Y.Doc (Yjs): the CRDT data structure representing the document
  - y-websocket: WebSocket provider that syncs Y.Doc across clients
  - Client: a Next.js page with a rich text editor (Tiptap, Lexical, ProseMirror)
            connected to Yjs
  - Server: a separate Node.js process (NOT a serverless function!) running
            y-websocket-server — must be persistent for WebSocket connections

  WHY NOT SERVERLESS: WebSocket connections must persist for the entire editing session.
  Serverless functions have execution time limits and can't maintain long-lived connections.

CONFLICT RESOLUTION:
  Yjs handles this automatically for text operations:
  User A inserts 'X' at position 5
  User B (simultaneously) inserts 'Y' at position 5
  Yjs's CRDT: both insertions are preserved; a deterministic algorithm
  decides the final order ('XY' or 'YX')

PRESENCE (showing other users' cursors):
  Each user broadcasts their cursor position through the WebSocket
  Other clients render colored cursor indicators in real-time
  This is awareness state — separate from document state, not persisted

PERSISTENCE:
  Periodically save Y.Doc state to database (every N seconds or on specific events)
  On reconnect: load saved Y.Doc from database, then apply any offline changes
```

---

## Common Design Challenges and Solutions

### Challenge: Authentication + Caching Conflict

```
PROBLEM: Anonymous users get CDN-cached pages. Authenticated users need
personalized content. How do you serve both efficiently?

SOLUTION:
  Option A: Static shell + client-side personalization
    → CDN-cache the page without personalization
    → After hydration, fetch personalized data client-side
    → LCP is fast (cached shell); personalization loads slightly later

  Option B: Edge middleware separation
    → Middleware checks for session cookie
    → Anonymous → serve static/CDN version
    → Authenticated → route to dynamic server rendering

  Option C: Partial Prerendering (PPR)
    → Static shell is CDN-cached
    → Dynamic sections (greeting, cart count) stream in from server

  RECOMMENDATION: depends on how critical the personalization is to the
  initial impression. For a dashboard: Option B. For a marketing page
  with a "Hi Alice" greeting: Option A or C.
```

### Challenge: Optimistic Updates with Failure Recovery

```
PROBLEM: User likes a post. Should the UI update immediately?
What if the API call fails?

SOLUTION PATTERN:
  1. User clicks "Like"
  2. Immediately update UI (like count +1, heart turns red) [optimistic]
  3. Dispatch POST /api/posts/{id}/like
  4a. SUCCESS: confirm — UI state was already correct
  4b. FAILURE: rollback → revert UI to previous state + show error toast

  With TanStack Query:
    onMutate: → save snapshot, update optimistically
    onError: → rollback from snapshot
    onSettled: → refetch to ensure consistency

  UNDO PATTERN:
    Show "Undo" button for N seconds after the action
    If user undoes: cancel the pending request (or immediately dispatch unlike)
    This is more user-friendly than showing an error
```

### Challenge: Accessibility for Dynamic Content

```
PROBLEM: Search results load asynchronously. Screen readers don't
automatically announce dynamic content updates.

SOLUTION:
  1. aria-live="polite" region that announces result count:
     "24 results for 'laptop'" — announced after results load

  2. Focus management: when results replace the previous results,
     move focus to the first result or the results count heading

  3. Loading state: aria-busy="true" on the results container
     while loading; "Loading search results" for screen readers

  4. No-results state: use role="status" to announce "No results found"
```

---

## Trade-off Articulation: How to Sound Senior

Senior engineers are identified by their ability to articulate WHY they made each decision and WHAT THEY GAVE UP. Practice these patterns:

```
INSTEAD OF: "I'd use TanStack Query for data fetching."
SAY: "I'd use TanStack Query for the product list because it handles
caching, deduplication, and background refresh — which reduces API
calls by ~80% compared to useEffect + useState. The tradeoff is
bundle size (~13KB gzipped) and a learning curve for the team.
For a simple form with one mutation, I might just use a Server Action
directly — the complexity overhead wouldn't be worth it."

INSTEAD OF: "I'd use WebSocket for real-time updates."
SAY: "I'd use SSE rather than WebSocket for the notification feed
because it's unidirectional — we only push from server to client.
SSE requires less infrastructure, works over standard HTTP (no
WebSocket upgrade concerns with corporate proxies), and has built-in
reconnection. WebSocket would only make sense if users could also
push real-time data to other users — like in collaborative editing."

INSTEAD OF: "I'd make the page statically generated."
SAY: "The product listing page is a good SSG candidate — content
changes daily, not per-request. This lets us serve pre-rendered HTML
from a CDN with ~10ms TTFB instead of ~200ms from a server. The
tradeoff: data can be up to the ISR revalidation period (I'd set 60
seconds) stale. For the 'in-stock' indicator, I'd override this for
that specific component to be dynamic, since inventory changes
frequently and showing wrong stock is a business problem."
```

---

## What Interviewers Are Actually Evaluating

```
STRONG SIGNALS (what separates senior from mid-level):

✅ Asks clarifying questions before diving into solution
   (mid-level jumps straight to "I'd build it like this")

✅ Explains WHY at each decision point
   (not just "I'd use TanStack Query" but "because it solves caching,
   deduplication, and race conditions that I'd otherwise implement manually")

✅ Identifies the HARD PROBLEMS in the system
   (real-time conflict resolution, cache invalidation timing,
   the WebSocket-serverless incompatibility)

✅ Acknowledges trade-offs explicitly
   ("This gives us fast initial load at the cost of a slightly stale
   cache for up to 60 seconds")

✅ Considers non-functional requirements unprompted
   (accessibility, error states, network failures, mobile users)

✅ Scales the solution to the stated requirements
   (different architecture for 100 users vs 10M users)

WEAK SIGNALS (what mid-level candidates often do):

❌ Describes the tech stack without explaining the architecture
   ("I'd use Next.js with Redux and TanStack Query and TypeScript...")

❌ Only designs the happy path
   (no error states, no loading states, no edge cases)

❌ Ignores accessibility entirely

❌ Cannot explain why they chose one approach over an alternative

❌ Designs at the wrong level of abstraction
   (either too high-level "I'd have a search API"
   or too low-level "the search button would have an onClick handler")
```

---

## Further Reading

- [GreatFrontEnd: Frontend System Design](https://www.greatfrontend.com/system-design) — the most comprehensive resource for frontend system design interview preparation
- [Educative: Frontend System Design Interview](https://www.educative.io/courses/grokking-the-frontend-system-design-interview) — structured curriculum
- [Patterns.dev](https://www.patterns.dev/) — architectural patterns with code examples
- [web.dev: Core Web Vitals](https://web.dev/explore/learn-core-web-vitals) — the performance dimensions that matter in production
- [Yjs documentation](https://docs.yjs.dev/) — the leading CRDT library for collaborative features
- Related handbook sections: Part XIX (Architecture), Part XV (Performance), Part XVIII (Networking)
- Next in this handbook: [122 · Behavioral & Architecture Questions](./04-behavioral-questions.md)

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
