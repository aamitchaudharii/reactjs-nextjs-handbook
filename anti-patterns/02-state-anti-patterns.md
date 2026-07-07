# 117 · State Management Anti-Patterns

> **State management anti-patterns are a category unto themselves because state is the most consequential part of a React application's architecture — bugs in state management produce symptoms that are hard to trace (the UI shows wrong data), hard to reproduce (it only happens after a specific sequence of actions), and expensive to fix (often requiring architectural changes, not just bug fixes). This document covers the failure modes of each major state management approach: useState misuse, Redux/Zustand architectural mistakes, TanStack Query misuse, and the fundamental confusion between server state and client state that underlies many of these problems.**

The most dangerous state management anti-patterns aren't obvious mistakes — they're patterns that seem principled and work at small scale. Putting everything in Redux seemed like good architecture in 2018 (one central store, predictable data flow). Putting everything in global Zustand stores seemed simpler. Using useState for server data seemed pragmatic. Each of these patterns creates specific, predictable failure modes at scale that this document catalogs.

---

## Table of Contents

- [Anti-Pattern 1: Conflating Server State with Client State](#anti-pattern-1-conflating-server-state-with-client-state)
- [Anti-Pattern 2: Putting Everything in Redux/Zustand](#anti-pattern-2-putting-everything-in-reduxzustand)
- [Anti-Pattern 3: Storing Non-Serializable Data in Redux](#anti-pattern-3-storing-non-serializable-data-in-redux)
- [Anti-Pattern 4: Selector-less Store Access](#anti-pattern-4-selector-less-store-access)
- [Anti-Pattern 5: Manual Data Synchronization](#anti-pattern-5-manual-data-synchronization)
- [Anti-Pattern 6: useQuery Without Proper Key Structure](#anti-pattern-6-usequery-without-proper-key-structure)
- [Anti-Pattern 7: Ignoring Optimistic Update Rollback](#anti-pattern-7-ignoring-optimistic-update-rollback)
- [Anti-Pattern 8: State for UI Side Effects](#anti-pattern-8-state-for-ui-side-effects)
- [Anti-Pattern 9: Inconsistent Normalization](#anti-pattern-9-inconsistent-normalization)
- [Anti-Pattern 10: Premature Global State](#anti-pattern-10-premature-global-state)
- [Architecture Diagrams](#architecture-diagrams)
- [State Placement Decision Framework](#state-placement-decision-framework)
- [Mental Model](#mental-model)
- [Further Reading](#further-reading)

---

## Anti-Pattern 1: Conflating Server State with Client State

```tsx
// ❌ THE ANTI-PATTERN: managing server data with useState
function ProductPage({ productId }: { productId: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    setIsLoading(true);
    fetch(`/api/products/${productId}`)
      .then((r) => r.json())
      .then((data) => {
        setProduct(data);
        setIsLoading(false);
      })
      .catch((err) => {
        setError(err.message);
        setIsLoading(false);
      });
  }, [productId]);

  // This home-rolled approach lacks:
  // - Caching (re-fetches every mount, even if data was just loaded)
  // - Deduplication (multiple components fetching the same product → multiple requests)
  // - Background refresh (stale data never refreshes automatically)
  // - Retry logic (network errors permanently show an error)
  // - Race condition handling (productId changes mid-fetch → can show wrong data)
  // - Optimistic updates (mutations require manual local state management)
}

// WHY IT FAILS AT SCALE:
// You can never achieve full feature parity with a proper server state library
// through manual useState management — each new requirement adds more state
// and more complex useEffect interactions, eventually creating unmaintainable code.

// ✅ THE FIX: use server state for server state
import { useQuery } from "@tanstack/react-query";

function ProductPage({ productId }: { productId: string }) {
  const {
    data: product,
    isLoading,
    error,
  } = useQuery({
    queryKey: ["product", productId],
    queryFn: () => fetch(`/api/products/${productId}`).then((r) => r.json()),
    staleTime: 5 * 60 * 1000, // consider fresh for 5 minutes
  });
  // ✅ Caching, deduplication, background refresh, retry, race condition
  // handling all included — zero manual state management needed
}

// THE FUNDAMENTAL DISTINCTION:
// CLIENT STATE: values that exist only in the browser session
//   - Is this modal open?
//   - What has the user typed in this input (before submitting)?
//   - Which tab is active?
//   → Use: useState, Zustand, Redux
//
// SERVER STATE: data that lives on the server; the UI reflects it
//   - List of products, user profile, order history
//   - This data can change outside the user's browser (other users update it)
//   → Use: TanStack Query, SWR, or React Server Components (for Next.js)
```

---

## Anti-Pattern 2: Putting Everything in Redux/Zustand

```tsx
// ❌ THE ANTI-PATTERN: storing ALL state globally
// (a common overcorrection from prop drilling problems)

const useStore = create<AppState>()((set) => ({
  // Server state (WRONG: belongs in TanStack Query):
  products: [] as Product[],
  isLoadingProducts: false,
  currentUser: null as User | null,
  orders: [] as Order[],

  // Ephemeral UI state (WRONG: belongs in local useState):
  isModalOpen: false,
  searchInputValue: "",
  activeTab: "overview",
  tooltipTarget: null as string | null,

  // Local form state (WRONG: belongs in the form component):
  checkoutFormData: {} as CheckoutFormData,
  formErrors: {} as Record<string, string>,

  // Actual global client state (CORRECT for global store):
  authToken: null as string | null,
  theme: "light" as "light" | "dark",
  cartItems: [] as CartItem[],
}));

// WHY IT FAILS:
// 1. Every state change notifies EVERY subscriber (even unrelated components)
// 2. Ephemeral state (tooltip visibility) shouldn't survive navigation/remounting
// 3. Form state in a global store is impossible to reset when the form unmounts
// 4. Server state in a global store requires manual invalidation logic
// 5. The store becomes a dumping ground — no clear ownership of each piece of state

// ✅ THE FIX: right tool for each state category
// Global store (Zustand/Redux): authToken, theme, cartItems
// TanStack Query: products, currentUser, orders
// Local useState: isModalOpen, activeTab, tooltipTarget
// Form library (react-hook-form): checkoutFormData, formErrors
```

---

## Anti-Pattern 3: Storing Non-Serializable Data in Redux

```ts
// ❌ THE ANTI-PATTERN: storing non-serializable values in Redux state
const slice = createSlice({
  name: "ui",
  initialState: {
    componentRef: null as React.RefObject<HTMLElement> | null, // ← ref object
    lastUpdated: new Date(), // ← Date object
    socketConnection: null as WebSocket | null, // ← WebSocket instance
    pendingPromise: null as Promise<unknown> | null, // ← Promise
    eventEmitter: null as EventTarget | null, // ← DOM object
  },
  reducers: {
    /* ... */
  },
});

// WHY IT FAILS:
// 1. Redux DevTools: non-serializable values can't be inspected meaningfully
// 2. Time-travel debugging breaks (can't serialize/deserialize DOM objects)
// 3. Redux Toolkit's serializability middleware throws warnings/errors
// 4. Redux Persist (if used): can't serialize Date, Promise, DOM objects
//    → they come back as plain objects after hydration, breaking code that
//       expects instanceof Date or instanceof WebSocket

// ✅ THE FIX: Redux state should only hold serializable data
const slice = createSlice({
  name: "ui",
  initialState: {
    lastUpdated: null as string | null, // ISO string instead of Date
    // Refs: use useRef in components, not Redux
    // WebSocket: manage in a custom hook or service layer, not Redux
    // Promises: use createAsyncThunk and store only the result
    // EventEmitter: subscribe/unsubscribe in useEffect, not in store
  },
  reducers: {
    setLastUpdated: (state) => {
      state.lastUpdated = new Date().toISOString(); // string, serializable
    },
  },
});
```

---

## Anti-Pattern 4: Selector-less Store Access

```ts
// ❌ THE ANTI-PATTERN: accessing the entire store in every component
'use client';
import { useStore } from '@/store';

function CartIcon() {
  // ❌ This subscribes to the ENTIRE store:
  const store = useStore(); // all of Zustand state
  const cartCount = store.cart.items.length;

  return <span>{cartCount}</span>;
}

// WHY IT FAILS:
// CartIcon re-renders on EVERY Zustand state change —
// including changes to user profile, theme, search query, product list,
// etc. — even though CartIcon only cares about cart.items.length.

// WITH REDUX:
const state = useSelector(state => state); // ❌ re-renders on EVERY action
const cartCount = state.cart.items.length; // only uses 1% of the store

// ✅ THE FIX: selective subscription via selectors
// Zustand:
function CartIcon() {
  // ✅ Only subscribes to cart.items.length — re-renders ONLY when count changes:
  const cartCount = useStore(state => state.cart.items.length);
  return <span>{cartCount}</span>;
}

// Redux Toolkit with createSelector:
import { createSelector } from '@reduxjs/toolkit';

const selectCartCount = createSelector(
  [(state: RootState) => state.cart.items],
  (items) => items.length
);

function CartIcon() {
  const cartCount = useSelector(selectCartCount); // ✅ memoized selector
  return <span>{cartCount}</span>;
}

// PERFORMANCE IMPACT:
// A store with 20 slices, each changing regularly:
// Without selectors: each of 50 components re-renders on EVERY state change
// With selectors: each component re-renders only when ITS specific data changes
// 50x fewer re-renders in the worst case.
```

---

## Anti-Pattern 5: Manual Data Synchronization

```ts
// ❌ THE ANTI-PATTERN: manually keeping multiple state copies in sync
const useProductStore = create<ProductState>()((set) => ({
  products: [] as Product[],
  selectedProduct: null as Product | null,

  updateProduct: (updated: Product) =>
    set((state) => ({
      // Update in the list:
      products: state.products.map((p) => (p.id === updated.id ? updated : p)),
      // Update the selected product if it's the same one:
      selectedProduct:
        state.selectedProduct?.id === updated.id
          ? updated // ← duplicated state — must remember to sync this manually
          : state.selectedProduct,
    })),
}));

// WHY IT FAILS:
// Every mutation must update EVERY copy of the data.
// When a new place stores a product reference (an "edit history" array,
// a "recently viewed" list), the synchronization code must be updated.
// Miss one → subtle data consistency bug.

// ✅ THE FIX: normalize state (store each entity ONCE, reference by ID)
const useProductStore = create<ProductState>()((set) => ({
  productsById: {} as Record<string, Product>, // SINGLE SOURCE OF TRUTH
  productIds: [] as string[], // ordered list of IDs
  selectedProductId: null as string | null, // reference, not a copy

  updateProduct: (updated: Product) =>
    set((state) => ({
      // Only ONE update needed — selectedProductId still points to the right thing:
      productsById: { ...state.productsById, [updated.id]: updated },
      // No need to update selectedProductId — it still references the same ID,
      // and that ID now has the updated data in productsById
    })),
}));

// Derived selectors:
const selectSelectedProduct = (state: ProductState) =>
  state.selectedProductId ? state.productsById[state.selectedProductId] : null;
// selectedProduct is DERIVED from the single source of truth — always in sync.
```

---

## Anti-Pattern 6: useQuery Without Proper Key Structure

```ts
// ❌ THE ANTI-PATTERN: flat, un-hierarchical, or unstable query keys

// Static string keys — don't include parameters:
useQuery({ queryKey: ["products"], queryFn: fetchAllProducts });
// If productId changes, this query doesn't re-fetch because the key doesn't change

// Unstable key (object created inline):
useQuery({
  queryKey: [{ type: "product", id: productId }], // ← new object reference each render
  queryFn: fetchProduct,
  // TanStack Query compares keys by deep equality, so this works,
  // but it's brittle and harder to invalidate/refetch programmatically
});

// ❌ Non-hierarchical keys make invalidation imprecise:
useQuery({ queryKey: ["productList"] });
useQuery({ queryKey: ["productDetail"] });
useQuery({ queryKey: ["productReviews"] });
// After updating a product, you must invalidate each of these individually

// ✅ THE FIX: hierarchical key structure as arrays
// Query key factory pattern:
const productKeys = {
  all: ["products"] as const,
  lists: () => [...productKeys.all, "list"] as const,
  list: (filters: FilterParams) =>
    [...productKeys.lists(), { filters }] as const,
  details: () => [...productKeys.all, "detail"] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
  reviews: (productId: string) =>
    [...productKeys.detail(productId), "reviews"] as const,
};

// Usage:
useQuery({
  queryKey: productKeys.detail(productId),
  queryFn: () => fetchProduct(productId),
});
useQuery({
  queryKey: productKeys.reviews(productId),
  queryFn: () => fetchReviews(productId),
});

// Invalidation: precise or broad:
queryClient.invalidateQueries({ queryKey: productKeys.all }); // invalidate ALL product queries
queryClient.invalidateQueries({ queryKey: productKeys.detail(productId) }); // just one product
```

---

## Anti-Pattern 7: Ignoring Optimistic Update Rollback

```ts
// ❌ THE ANTI-PATTERN: optimistic updates without error rollback
const addToCartMutation = useMutation({
  mutationFn: (item: CartItem) => api.cart.addItem(item),

  onMutate: (newItem) => {
    // Optimistically update cache:
    queryClient.setQueryData(["cart"], (old: Cart) => ({
      ...old,
      items: [...old.items, newItem],
    }));
    // ❌ No rollback handling — if the mutation fails, the UI shows
    // the wrong state permanently (until next refetch or page reload)
  },
});

// WHY IT FAILS:
// If the API call fails (rate limited, network error, server error),
// the cart shows the item as added when it wasn't.
// User tries to check out → checkout fails because the item isn't in the DB.
// Confusing, potentially financial impact.

// ✅ THE FIX: complete optimistic update with rollback
const addToCartMutation = useMutation({
  mutationFn: (item: CartItem) => api.cart.addItem(item),

  onMutate: async (newItem) => {
    // Cancel outgoing refetches so they don't overwrite our optimistic update:
    await queryClient.cancelQueries({ queryKey: ["cart"] });

    // Snapshot the current state (for rollback):
    const previousCart = queryClient.getQueryData<Cart>(["cart"]);

    // Optimistically update:
    queryClient.setQueryData(["cart"], (old: Cart) => ({
      ...old,
      items: [...old.items, newItem],
    }));

    // Return context for rollback in onError:
    return { previousCart };
  },

  onError: (err, newItem, context) => {
    // ✅ Rollback to previous state on failure:
    if (context?.previousCart) {
      queryClient.setQueryData(["cart"], context.previousCart);
    }
    toast.error("Failed to add item to cart. Please try again.");
  },

  onSettled: () => {
    // Refetch to ensure server and client are in sync:
    queryClient.invalidateQueries({ queryKey: ["cart"] });
  },
});
```

---

## Anti-Pattern 8: State for UI Side Effects

```tsx
// ❌ THE ANTI-PATTERN: using state to trigger side effects imperatively

function ProductCard({ product }: { product: Product }) {
  const [shouldFetchRecommendations, setShouldFetchRecommendations] = useState(false);
  const [recommendations, setRecommendations] = useState([]);

  // State used as a "trigger" for an effect — an anti-pattern
  useEffect(() => {
    if (shouldFetchRecommendations) {
      fetchRecommendations(product.id).then(setRecommendations);
    }
  }, [shouldFetchRecommendations, product.id]);

  const handleExpand = () => {
    setShouldFetchRecommendations(true); // ← "telling" the effect to run
  };

  // Problems:
  // - shouldFetchRecommendations stays true even after fetching — confusing state
  // - Re-triggering the fetch requires setting it false then true again
  // - The intent is an EVENT ("user expanded the card") not persistent state

// ✅ THE FIX: handle events directly
function ProductCard({ product }: { product: Product }) {
  const [isExpanded, setIsExpanded] = useState(false);
  const { data: recommendations } = useQuery({
    queryKey: ['recommendations', product.id],
    queryFn: () => fetchRecommendations(product.id),
    enabled: isExpanded, // ← fetch only when expanded, using a STABLE state value
  });

  // OR: fetch on demand using an event handler directly:
  const handleExpand = async () => {
    setIsExpanded(true);
    // Imperative fetch if needed:
    const data = await fetchRecommendations(product.id);
    // TanStack Query's `enabled` pattern above is cleaner
  };
}
```

---

## Anti-Pattern 9: Inconsistent Normalization

```ts
// ❌ THE ANTI-PATTERN: mixing normalized and denormalized data
const useStore = create<State>()((set) => ({
  // Normalized:
  usersById: {} as Record<string, User>,
  userIds: [] as string[],

  // Denormalized (full user object embedded in each order):
  orders: [] as Array<{ id: string; total: number; user: User }>,

  // Now updating a user requires updating BOTH usersById AND every order:
  updateUser: (updated: User) =>
    set((state) => ({
      usersById: { ...state.usersById, [updated.id]: updated },
      orders: state.orders.map((order) =>
        order.user.id === updated.id
          ? { ...order, user: updated } // ← manual sync required
          : order,
      ),
    })),
}));

// WHY IT FAILS:
// As the app grows, the user object gets embedded in more places
// (comments, reviews, activity logs, messages...).
// Every user mutation must find and update every embedded copy.
// Miss one → stale user data shown in some contexts.

// ✅ THE FIX: normalize consistently — reference by ID everywhere
const useStore = create<State>()((set) => ({
  usersById: {} as Record<string, User>,
  userIds: [] as string[],
  // Store orders with userId reference, not embedded user:
  orders: [] as Array<{ id: string; total: number; userId: string }>,

  updateUser: (updated: User) =>
    set((state) => ({
      usersById: { ...state.usersById, [updated.id]: updated },
      // orders doesn't need to change — userId reference is still correct
      // The updated user data is auto-reflected via the usersById lookup
    })),
}));

// Selector to get order with full user:
const selectOrderWithUser = (orderId: string) => (state: State) => {
  const order = state.orders.find((o) => o.id === orderId);
  if (!order) return null;
  return { ...order, user: state.usersById[order.userId] };
};
```

---

## Anti-Pattern 10: Premature Global State

```ts
// ❌ THE ANTI-PATTERN: globalizing state before it needs to be global
// "I might need this in other components" → add it to the global store today

const useStore = create<State>()((set) => ({
  // These are used by exactly ONE component — why are they global?
  profileEditMode: false, // only ProfilePage uses this
  profileFormDraft: {} as Profile, // only ProfilePage uses this
  profileSaveStatus: "idle" as "idle" | "saving" | "saved" | "error",
}));

// WHY IT FAILS:
// 1. The profile edit state persists when the user navigates away
//    (they return to profile and see a half-edited form from 10 minutes ago)
// 2. Every component subscribing to the store gets slightly slower
//    as the store grows with more ephemeral state
// 3. The global store becomes harder to understand (what is profileSaveStatus
//    for? Who sets it? Who reads it?)
// 4. Testing the ProfilePage requires initializing global store state

// ✅ THE FIX: start with the smallest possible scope
function ProfilePage() {
  // This state belongs HERE — it's created when ProfilePage mounts
  // and destroyed when it unmounts (correct lifecycle semantics):
  const [isEditing, setIsEditing] = useState(false);
  const [draft, setDraft] = useState<Profile>({});
  const [saveStatus, setSaveStatus] = useState<
    "idle" | "saving" | "saved" | "error"
  >("idle");

  // If other components genuinely need this state LATER:
  // 1. Lift state to the common ancestor
  // 2. Extract to a custom hook
  // 3. THEN, if truly cross-component, move to global store
  // Don't preemptively globalize "just in case"
}
```

---

## Architecture Diagrams

### State type × storage location matrix

```mermaid
graph TD
    A["What kind of state?"] --> B{Server data\n(lives on the server)?}
    B -->|"Yes"| C["TanStack Query\nor React Server Components"]
    B -->|"No (client-only)"| D{Needed by multiple\nunrelated components?}
    D -->|"No"| E["Local useState\nor useReducer"]
    D -->|"Yes"| F{Changes frequently\nor rarely?}
    F -->|"Rarely (theme, auth)"| G["Context API"]
    F -->|"Frequently (cart, filters)"| H["Zustand / Redux Toolkit"]

    style C fill:#27ae60,color:#fff
    style E fill:#61dafb,color:#000
    style G fill:#764abc,color:#fff
    style H fill:#f39c12,color:#000
```

---

## State Placement Decision Framework

```
STEP 1: IS IT SERVER STATE?
  Does this data exist on the server and is the UI displaying it?
  → YES: use TanStack Query or RSC (not useState, not Redux/Zustand)
  → NO: continue to step 2

STEP 2: WHICH COMPONENTS NEED THIS STATE?
  Only the component itself (and maybe its direct children)?
  → Local useState

  Multiple components that could pass it as props without too many hops?
  → Lift state to the nearest common ancestor + props

  Many unrelated components throughout the tree?
  → Continue to step 3

STEP 3: HOW OFTEN DOES IT CHANGE?
  Rarely (user logged in, theme selected, feature flags)?
  → Context API (provider re-renders are less costly for infrequent changes)

  Frequently (shopping cart, real-time data, UI interaction state)?
  → Zustand or Redux Toolkit (subscription model, no cascade re-renders)

STEP 4: IS THE PATTERN CONSISTENT?
  Server data: ALWAYS TanStack Query
  Global UI state: ALWAYS the global store
  Local UI state: ALWAYS local useState
  NEVER mix these without a clear reason

THE COST OF INCONSISTENCY:
  A codebase where some server data is in TanStack Query and some is in Zustand
  and some is in local useState creates a cognitive burden on every developer:
  "Where is this data coming from? How is it fetched? How do I invalidate it?"
  Consistency in state placement makes answering these questions instant.
```

---

## Mental Model

> 💡 **The state management anti-pattern mental model:**
>
> State management anti-patterns all trace back to one mistake: **using the wrong container for the kind of data**. Server state in useState is like putting library books in a personal filing cabinet — the books live in the library (the server), but you're making your own copy in your home (useState), so your copy goes out of date the moment the library updates its record. Server state belongs in a library catalog system (TanStack Query) that knows where the authoritative source is and can fetch an updated copy when needed. Global state being dumped into a single context is like having one giant filing cabinet for everyone in the company — every time anyone files or retrieves anything, everyone has to stop what they're doing and wait (re-renders for all context consumers on every change). Granular stores with selectors are like individual departments' filing systems — the accounting department's changes don't interrupt engineering's work. The right container for each type of data is not a matter of preference — it determines whether the system remains comprehensible and performant as it grows.

---

## Further Reading

- [TanStack Query docs: Server State vs Client State](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults) — the foundational distinction
- [Zustand docs: Selecting state](https://docs.pmnd.rs/zustand/guides/prevent-rerenders-with-use-shallow) — selector-based subscriptions
- [Redux Toolkit docs: Normalized State](https://redux-toolkit.js.org/usage/usage-guide#normalizing-state-shape) — official normalization guide
- [Lenz Weber-Tronic: Selector patterns in Redux](https://blog.isquaredsoftware.com/2017/12/idiomatic-redux-using-reselect-selectors/) — selectors deep dive
- [TanStack Query: Query Key factories](https://tkdodo.eu/blog/effective-react-query-keys) — the hierarchical key pattern
- Related in this handbook: [79 · Context Internals](../state-management/01-context-internals.md), [80 · Redux](../state-management/02-redux.md), [82 · TanStack Query](../state-management/04-tanstack-query.md)
- Next in this handbook: [118 · Performance Anti-Patterns](./03-performance-anti-patterns.md)

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
