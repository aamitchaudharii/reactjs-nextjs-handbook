# 119 · React Internals Interview Questions

> **This document covers the React internals questions that appear at senior and staff engineer interviews — not "what is useState?" but "explain how React's reconciliation algorithm determines which DOM nodes to update," "what is the priority model in React's concurrent scheduler," and "how does React Fiber enable interruptible rendering." These are questions that probe whether a candidate understands React at the level of being able to predict its behavior in non-obvious situations, debug issues that can't be solved by reading documentation, and make sound architectural decisions about rendering strategy.**

Each question below is formatted with: the interview question, a complete answer at the depth expected for senior/staff level, and the key concepts that separate a strong answer from a surface-level one. The answers deliberately mirror the depth of coverage in the earlier sections of this handbook — if a concept feels unfamiliar, the relevant handbook document is cross-referenced.

---

## Questions by Category

- [Rendering and Reconciliation](#rendering-and-reconciliation)
- [Fiber Architecture](#fiber-architecture)
- [Hooks Internals](#hooks-internals)
- [Concurrent React](#concurrent-react)
- [Server Components](#server-components)
- [Performance](#performance)
- [Common Trap Questions](#common-trap-questions)

---

## Rendering and Reconciliation

### Q1: Explain React's reconciliation algorithm. How does React decide what to update in the DOM?

**Complete Answer:**

React's reconciliation algorithm (the "diffing" algorithm) compares the previous virtual DOM tree with the new one after a render, and determines the minimal set of DOM operations needed to update the actual DOM.

The algorithm has two key heuristics that make it O(n) instead of the theoretically O(n³) optimal tree diff:

**Heuristic 1 — Elements of different types produce entirely different trees.** If a `<div>` becomes a `<span>`, React tears down the entire subtree and builds a new one from scratch. No partial reuse is attempted across type boundaries.

**Heuristic 2 — The `key` prop provides stable identity across renders.** Within a list, React uses `key` to match elements across renders. An element with `key="product-123"` in the new tree will be matched to the element with `key="product-123"` in the previous tree, even if it moved position. Without `key`, React uses positional matching (first child maps to first child, etc.).

The reconciliation process:

1. React renders (calls) the component functions, producing a new VDOM tree
2. React walks both trees simultaneously, comparing corresponding nodes
3. For each node pair: if the type matches, update only the changed props; if the type differs, replace the subtree
4. List children are matched by `key` (with key) or by position (without key)
5. The resulting set of DOM mutations is applied in a single commit phase

**Key differentiators from a surface answer:** Understanding that reconciliation happens at the VDOM level (not the DOM), that the algorithm is intentionally heuristic (not perfect), and that `key` is about IDENTITY not uniqueness — a key that changes causes unmount+remount, a stable key causes update in place.

_Handbook reference: Parts I-IV (React Core, Reconciliation)_

---

### Q2: What is the difference between "rendering" and "committing" in React?

**Complete Answer:**

React's update cycle has two distinct phases with very different characteristics:

**The Render Phase (pure computation):**

- React calls component functions to produce a new VDOM tree
- React runs reconciliation (diffing the new tree against the previous one)
- React builds a "work-in-progress" tree of changes to apply
- This phase is PURE — no side effects, no DOM mutations
- In Concurrent React, this phase is INTERRUPTIBLE — React can pause it to handle higher-priority work (user input) and resume later
- Can run MULTIPLE TIMES for a single commit (if interrupted and retried)

**The Commit Phase (side effects and DOM mutation):**

- React applies the DOM mutations computed during the render phase
- React runs `useLayoutEffect` cleanups and setups (synchronously)
- React runs `useEffect` cleanups and setups (asynchronously, after paint)
- This phase is SYNCHRONOUS and UNINTERRUPTIBLE — once it starts, it completes
- DOM updates happen here, so effects that read the DOM are valid in this phase

**Why this matters:** The separation explains why:

- You must never mutate the DOM during rendering (render phase is pure)
- `useLayoutEffect` fires before the browser paints (it's in the commit phase)
- `useEffect` fires after the browser paints (asynchronously after commit)
- React strict mode double-invokes render functions (to catch impure renders) but NOT the commit phase

_Handbook reference: Part II (Rendering), Part V (Hooks Internals)_

---

### Q3: How does React use keys? What happens when a key changes vs when a key is reused?

**Complete Answer:**

`key` is React's mechanism for providing EXPLICIT IDENTITY to elements in a list or across conditional renders. React uses keys to match elements between renders rather than relying on positional matching.

**When keys are STABLE (same key, same type):** React updates the existing component instance in place — state is preserved, the component is NOT unmounted. Only changed props are updated.

**When a key CHANGES (different key, same type):** React treats this as a completely different element — it UNMOUNTS the previous component (running cleanup effects, destroying state) and MOUNTS a new one (running initialization effects). This is the `key` reset pattern used deliberately.

**When keys are ABSENT in lists:** React uses positional matching. If item 0 is removed from a list of 3, React sees:

- Position 0: "type matches, update props" (wrong: B rendered for A's DOM node)
- Position 1: "type matches, update props" (wrong: C rendered for B's DOM node)
- Position 2: "removed" (destroys C's DOM node)

This causes incorrect DOM reuse, state contamination, and animation/focus bugs.

**Practical misconceptions to address:**

- `key` must be STABLE and PREDICTABLE — not Math.random() (new key each render = unmount/remount on every render)
- `key` doesn't have to be globally unique — unique among siblings is sufficient
- `key` is not a prop — you cannot read it inside the component
- `key` can be used outside lists — on any element to force remounting on change

_Handbook reference: Part I (React Core), Part XVI (Anti-Patterns: Index as Key)_

---

## Fiber Architecture

### Q4: What is React Fiber and why was it introduced?

**Complete Answer:**

React Fiber is a complete rewrite of React's internal reconciliation engine, introduced in React 16 (2017). The name "Fiber" refers to the data structure at its core: a fiber is a JavaScript object representing a unit of work in React's component tree.

**Why Fiber was needed:** The pre-Fiber ("Stack Reconciler") architecture used the native JavaScript call stack for reconciliation. The JS call stack is non-interruptible — once a reconciliation run started, it ran to completion, no matter how long it took. For large trees, this could block the main thread for hundreds of milliseconds, causing dropped frames and janky UI.

**What Fiber enables:**

1. **Interruptibility:** React can pause reconciliation mid-tree, yield to the browser's event loop (processing user input, compositing frames), and resume later
2. **Priority:** Different updates have different priorities — user input is urgent, background data fetching is less urgent; Fiber's scheduler can prioritize accordingly
3. **Concurrency:** Multiple in-progress renders can exist simultaneously (concurrent features like `startTransition`, `Suspense`, concurrent rendering)
4. **Error boundaries:** The Fiber tree structure enables catching render errors at specific boundaries without crashing the entire tree

**The Fiber data structure:** Each React component instance corresponds to a fiber node containing: the component type, current props, current state, a link to the alternate (work-in-progress) fiber, parent/child/sibling fibers, effects to apply, and the priority of pending work.

**Double buffering:** React maintains TWO Fiber trees — the "current" tree (what's rendered) and the "work-in-progress" tree (what's being prepared). Reconciliation happens on the work-in-progress tree; commit swaps them.

_Handbook reference: Part III (React Fiber)_

---

### Q5: What is the "work loop" in React's Fiber architecture?

**Complete Answer:**

The work loop is React's internal scheduling mechanism that processes fiber nodes (units of work) one at a time, yielding to the browser's event loop between units when necessary.

The work loop operates on two levels:

**The outer loop (across fibers):** React picks the highest-priority pending fiber, processes it, moves to the next fiber in the tree (child, then sibling, then parent's sibling), and continues. After each fiber, the loop checks: "has a higher-priority update arrived? Has the current time slice expired?" If yes → yield to the browser (via `MessageChannel` or `setTimeout`), schedule a callback to resume.

**The inner work (per fiber):** For each fiber, React:

1. Calls the component function (render phase)
2. Compares the new output with the previous (reconciliation)
3. Creates or updates child fibers
4. Schedules effects to run in the commit phase

**Yielding mechanism:** Fiber uses `MessageChannel` (preferred) or `setTimeout(fn, 0)` to yield to the browser's event loop. This allows the browser to process paint operations and user input between fiber units. The scheduler targets 5ms time slices (a heuristic to stay within one frame at 200Hz displays).

**Priority lanes:** React 18 introduced "lanes" — a bitmask system where each bit represents a priority level. The work loop always processes the highest-priority lane first, allowing urgent updates (user input) to interrupt and preempt lower-priority work (background data updates).

_Handbook reference: Part III (React Fiber), Part VI (Concurrent React)_

---

## Hooks Internals

### Q6: How does React know which hook call corresponds to which hook? Why must hooks be called in the same order every render?

**Complete Answer:**

React stores hooks as a LINKED LIST on the fiber node. Each hook call during a component's render appends a new node to this list. React identifies each hook by its POSITION in the list — the first `useState` call is list node 0, the second is list node 1, etc.

**The mechanics:**

- First render: React creates the linked list, initializing each node with the hook's initial value/callback
- Subsequent renders: React traverses the EXISTING linked list in order, updating each node with the new values

```
Fiber.memoizedState → hook1 → hook2 → hook3 → null
                       (useState)(useRef)(useEffect)
```

**Why order must be stable:** React has no mechanism to identify hooks by name or intent — only by position. If `useState` is conditionally called on the first render but skipped on the second:

- First render: list = [useState(A), useRef(B), useEffect(C)]
- Second render: React traverses list but calls: [useRef(B_call), useEffect(C_call)]
- React's traversal hands node 0 (useState data) to the useRef call → wrong type, wrong data → undefined behavior

**The rule "don't call hooks conditionally" is a RUNTIME INVARIANT, not just a style guide.** React enforces this with a development-mode checker that counts hook calls and warns if the count changes between renders.

**useState specifically:** The linked list node for `useState` stores:

- `memoizedState`: the current state value
- `queue`: a circular linked list of pending state updates
- `dispatch`: the setter function (stable identity — same object every render)

_Handbook reference: Part V (Hooks Internals)_

---

### Q7: Explain how `useEffect` knows when to re-run. What exactly does React compare in the dependency array?

**Complete Answer:**

React uses **Object.is()** to compare dependency array values between renders. `Object.is()` is essentially `===` with two exceptions:

- `Object.is(NaN, NaN) === true` (NaN equals itself)
- `Object.is(+0, -0) === false` (+0 and -0 are different)

For each render, React:

1. Runs the component function
2. Compares the new dependency array to the previous one, element by element using `Object.is()`
3. If ANY dependency differs → schedules the effect to run (cleanup previous, run new)
4. If ALL dependencies are the same → skips the effect

**The key insight about reference types:** `Object.is()` on objects, arrays, and functions compares REFERENCES, not values. `{} !== {}` always (different object references). This is why:

- An object prop created inline `{ a: 1 }` causes the effect to re-run every render (new reference each time)
- A function created during render causes the effect to re-run (new function reference each time)
- `useCallback` and `useMemo` solve this by providing stable references

**What "empty dependency array" means:** The dependency array is compared to the PREVIOUS dependency array. An empty array `[]` always equals a previous empty array `[]` (vacuously — all zero dependencies are equal). So the effect runs ONCE (after mount) and never re-runs (cleanup runs on unmount).

**A common subtle bug:** If the effect uses values from the component's scope WITHOUT listing them in the dependency array, those values are "captured" at the time the effect function was created — the stale closure problem.

_Handbook reference: Part V (Hooks Internals - useEffect)_

---

## Concurrent React

### Q8: What is `startTransition` and how does it change React's rendering behavior?

**Complete Answer:**

`startTransition` marks a state update as "non-urgent" — it tells React that the associated re-render can be INTERRUPTED and DELAYED if more urgent updates arrive.

**Without startTransition (urgent update):**

```
User types → setState → React must synchronously complete the full re-render
             before returning control to the browser → main thread blocked → lag
```

**With startTransition (non-urgent):**

```
User types → immediate setState (urgent: input value updates instantly)
           → startTransition(() => setSearchResults(query)) (non-urgent)
           → React starts rendering search results
           → User types again → React INTERRUPTS the results render
           → React processes the new input character (urgent) first
           → React resumes/restarts the search results render with new query
```

**The mechanism:** `startTransition` marks updates with a "Transition" priority lane — lower than Default priority. React's scheduler processes higher-priority lanes first. When a Default-priority update arrives while a Transition is rendering, React yields the Transition render, processes the Default update, then either resumes or restarts the Transition render.

**`useTransition` vs `startTransition`:** `useTransition()` returns `[isPending, startTransition]` — the `isPending` flag is true while the transition render is in progress, allowing you to show loading indicators. `startTransition` (the function) is the same capability without the pending state.

**Important constraint:** Updates inside `startTransition` must NOT include urgent user-visible state (like input value). If the input value is inside startTransition, the UI appears frozen while the user types — that's worse than the original lag.

_Handbook reference: Part VI (Concurrent React - startTransition)_

---

### Q9: How does React Suspense work at the implementation level?

**Complete Answer:**

React Suspense works by catching a specific type of exception during the render phase — a Promise being thrown (not an Error).

**The protocol:**

1. A component in the Suspense boundary's subtree THROWS a Promise (instead of returning JSX)
2. React CATCHES this thrown Promise as it walks up the component tree
3. React finds the nearest Suspense boundary
4. React renders the Suspense boundary's `fallback` content
5. React SUBSCRIBES to the thrown Promise's resolution
6. When the Promise resolves, React RETRIES rendering the subtree that threw
7. If the component no longer throws (data is now available) → render the real content; replace the fallback

**React.cache() and the Suspense contract:** A data-fetching function wrapped with React.cache() (or a library like TanStack Query) implements this protocol: on first call, it throws a Promise (while data is loading); on subsequent calls with the same arguments (once resolved), it returns the data synchronously without throwing.

**Why this requires a concurrent renderer:** In React 16 (sync rendering), a thrown Promise during render would be an unhandled exception. Concurrent React can CATCH the thrown Promise, render an alternative (the fallback), and schedule retry — all without crashing the render.

**Waterfall vs parallel with Suspense:** Multiple Suspense boundaries in a tree each independently manage their loading states. React starts rendering ALL of them concurrently — if data for one resolves first, that section renders while others still show their fallbacks. There's no mandatory top-down serialization.

_Handbook reference: Part VI (Concurrent React - Suspense), Part X (Server Components - Streaming)_

---

## Server Components

### Q10: What is the React Server Component payload and how does it differ from SSR HTML?

**Complete Answer:**

React Server Components produce a serialized component tree representation (the RSC payload), NOT HTML. This is a fundamental difference from SSR.

**SSR (before RSC):**

- Server renders React components → HTML string → sent to browser
- Browser receives HTML (can display immediately)
- Browser downloads JavaScript → React hydrates (attaches event listeners)
- Hydration requires React to re-render the ENTIRE tree in the browser to match the server HTML

**RSC Payload:**

- Server renders Server Components → RSC payload (a custom streaming format)
- The RSC payload describes the component tree structure, data, and WHERE Client Component boundaries are
- Client Components' JavaScript is NOT included in the payload — they're referenced by identifier
- Browser receives the payload, uses it to construct the React tree
- Client Components are rendered IN THE BROWSER using their downloaded JavaScript
- The RSC payload enables PARTIAL HYDRATION: only Client Components need to be hydrated

**What the RSC payload looks like (simplified):**

```
// Server Component output:
J0:["$","div",null,{"children":["Hello",["$","$ClientButton",null,{"id":"btn"}]]}]
// $ClientButton = a reference to the Button Client Component
// The actual Button JS is loaded separately and rendered in the browser
```

**Key architectural properties:**

- Server Component code NEVER ships to the browser (db queries, secrets stay server-side)
- The payload is STREAMABLE — can be sent in chunks as data resolves
- On navigation (client-side routing), Next.js fetches only the RSC payload for the NEW page (not a full HTML page reload)
- RSC payload is more efficient for client-side navigation than full HTML pages

_Handbook reference: Part X (Server Components)_

---

## Performance

### Q11: A team member suggests wrapping every component in React.memo "for performance." What's your response?

**Complete Answer (suitable for whiteboard/discussion format):**

"That's actually a common misconception. React.memo is a targeted optimization that has COSTS — it's not universally beneficial.

**What React.memo does:** It wraps a component so that React SKIPS re-rendering it when the parent re-renders, IF the component's props haven't changed (by shallow equality comparison).

**The cost of React.memo:**

1. A shallow equality comparison of ALL props on EVERY parent render
2. Memory to store the previous props for comparison
3. For simple/cheap components, this comparison can cost MORE than just re-rendering

**When React.memo actually helps (when the cost is worth it):**

1. The component's render is genuinely expensive (profiler shows >5ms)
2. The component re-renders FREQUENTLY from parent renders (not from its own state)
3. The props are STABLE references (not inline objects/arrays which create new references every render)

**When React.memo HURTS or is neutral:**

- Simple components (a span with text): comparison is slower than re-render
- Props include inline objects/arrays: `<Memo style={{ color: 'red' }} />` — the style prop is a new object every render, comparison always fails, memo never skips
- Components that re-render from their OWN state: memo only blocks parent-triggered re-renders

**The right approach:** Profile first (React DevTools Profiler), identify SPECIFIC components that are expensively and unnecessarily re-rendering, then apply React.memo to those specific components with stable prop references (using useCallback/useMemo in the parent as needed)."

_Handbook reference: Part XV (Performance), Part XXIV (Anti-Patterns: Over-Memoization)_

---

## Common Trap Questions

### Q12: Can you call a hook inside a conditional? What if you need conditional hook behavior?

**Complete Answer:**

You CANNOT call hooks inside conditionals (or loops, or nested functions). This is enforced by React's runtime rules and the ESLint plugin `eslint-plugin-react-hooks`.

**Why:** Hooks are identified by their position in the linked list on the fiber node (see Q6). Calling hooks conditionally changes how many hooks execute, corrupting the position-based identification.

**How to achieve conditional behavior WITHOUT conditional hooks:**

```tsx
// ❌ WRONG: conditional hook call
function Component({ showAnalytics }: { showAnalytics: boolean }) {
  if (showAnalytics) {
    useAnalyticsTracking(); // violates the rules of hooks
  }
}

// ✅ CORRECT: hook always called, condition INSIDE the hook
function Component({ showAnalytics }: { showAnalytics: boolean }) {
  useAnalyticsTracking(showAnalytics); // hook decides internally whether to act
}

function useAnalyticsTracking(enabled: boolean) {
  useEffect(() => {
    if (!enabled) return; // condition is inside the hook, not outside
    trackPageView();
  }, [enabled]);
}

// ✅ CORRECT: different component for each conditional branch
function Component({ showAnalytics }: { showAnalytics: boolean }) {
  return showAnalytics ? <ComponentWithAnalytics /> : <ComponentWithout />;
}
// Each component has its own stable hook list
```

**The "enabled" pattern with useQuery:** TanStack Query's `enabled` option is exactly this — a hook that's always called but conditionally acts:

```ts
useQuery({ queryKey: ["user"], queryFn: fetchUser, enabled: !!userId });
// The hook is always called; it internally decides whether to fetch based on `enabled`
```

_Handbook reference: Part V (Hooks Internals)_

---

### Q13: What is the difference between `useLayoutEffect` and `useEffect`?

**Complete Answer:**

Both `useLayoutEffect` and `useEffect` run after a component renders, but they fire at DIFFERENT MOMENTS in the lifecycle:

**`useLayoutEffect`:** Fires synchronously AFTER all DOM mutations in the commit phase, but BEFORE the browser has painted. The component's DOM is already updated, but the user hasn't seen the update yet. This is analogous to the old `componentDidMount`/`componentDidUpdate` lifecycle methods.

**`useEffect`:** Fires asynchronously AFTER the browser has painted. The user has already seen the updated UI when this effect runs.

**When to use `useLayoutEffect`:**

- Reading layout-affecting DOM properties (element dimensions, scroll position, computed styles) BEFORE the user sees the component
- Synchronously updating DOM properties after a render to PREVENT FLICKER (e.g., setting a CSS variable based on measured dimensions — if done in `useEffect`, there'd be one frame where dimensions are wrong)
- Animations that must start synchronously to avoid a visible "jump"

**When to use `useEffect` (most cases):**

- Data fetching (non-blocking)
- Event subscription (non-layout-affecting)
- Analytics/logging
- Anything that can happen AFTER the user sees the update

**SSR gotcha:** `useLayoutEffect` produces a React SSR warning because it can't run on the server (there's no DOM). Use `useEffect` or `useSafeLayoutEffect` (which falls back to useEffect during SSR) for components that render server-side.

**The mental model:** `useLayoutEffect` is for reading and immediately writing DOM state within the same frame. `useEffect` is for everything else.

_Handbook reference: Part V (Hooks Internals - useLayoutEffect)_

---

### Q14: How does React batch state updates? What is "automatic batching" in React 18?

**Complete Answer:**

Batching is React's mechanism for grouping multiple state updates into a SINGLE re-render, rather than re-rendering after each individual update.

**Pre-React 18 batching (limited):**
React automatically batched updates that originated from React event handlers:

```tsx
const handleClick = () => {
  setCount((c) => c + 1); // ← batched
  setName("Alice"); // ← batched with the above → ONE re-render
};
```

But did NOT batch updates from async callbacks, setTimeout, or native event listeners:

```tsx
setTimeout(() => {
  setCount((c) => c + 1); // ← NOT batched in React 17
  setName("Alice"); // ← NOT batched → TWO re-renders
}, 0);
```

**React 18 "Automatic Batching":** ALL state updates are now batched, regardless of their origin — setTimeout, promises, native event listeners, React event handlers. All produce ONE re-render per batch.

```tsx
// React 18: automatically batched
setTimeout(() => {
  setCount((c) => c + 1);
  setName("Alice");
  // → ONE re-render (React 18)
  // → TWO re-renders (React 17)
}, 0);
```

**Opting OUT of batching (rare):** `flushSync()` forces a synchronous re-render immediately:

```tsx
import { flushSync } from "react-dom";
flushSync(() => setCount((c) => c + 1)); // re-render happens here, synchronously
setName("Alice"); // this triggers another re-render
```

Use `flushSync` only when you need DOM state to be updated BEFORE the next line of code runs (e.g., reading a DOM measurement that depends on the state update).

_Handbook reference: Part VI (Concurrent React - Batching)_

---

## Further Reading

- [React Architecture Overview](https://react.dev/learn/thinking-in-react) — the mental model React expects engineers to have
- [Sebastian Markbåge: React's Architecture](https://github.com/reactjs/react-basic) — the formal model behind React's design
- [Andrew Clark: React Fiber architecture](https://github.com/acdlite/react-fiber-architecture) — the original Fiber design document
- Related handbook sections: Parts I–VI (React internals), Part X (Server Components)
- Next in this handbook: [120 · Next.js Interview Questions](./02-nextjs-questions.md)

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
