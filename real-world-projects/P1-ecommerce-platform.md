# P1 · Real-World Project: E-Commerce Platform

> **This project walks through building a production-grade e-commerce platform with Next.js App Router, applying the architectural principles, performance patterns, and engineering decisions covered across the entire handbook. Unlike tutorials that show only the happy path, this guide focuses on the decisions that separate a toy project from a production system: caching strategy for product catalogs, optimistic cart updates, Stripe integration with proper server-side validation, image optimization at scale, and the specific security considerations for handling payment flows.**

This is a capstone project that synthesizes concepts from every major section of the handbook. Cross-references point to the deeper treatment of each concept.

---

## Project Overview

**What you'll build:**

- Product catalog with search and filtering (SSG + ISR)
- Shopping cart with optimistic updates (Client State + Server Actions)
- Checkout flow with Stripe integration (Server Actions + secure payment handling)
- User authentication (NextAuth.js)
- Order history (protected RSC pages)
- Admin panel (role-based access)

**Technology choices:**

- Next.js 15 (App Router)
- Prisma + PostgreSQL
- TanStack Query (server state on client-rendered pages)
- Zustand (cart state — client-only persistence)
- NextAuth.js v5
- Stripe (payments)
- Vercel (deployment) or self-hosted Node.js

---

## Architecture Decision Record

### ADR-1: Rendering Strategy by Page Type

```
PRODUCT LISTING (/products, /products?category=electronics)
  Strategy: ISR with revalidate=60
  Rationale: Product catalog changes infrequently (inventory, pricing updates
  happen on admin actions, not per-request). A 60-second staleness window is
  acceptable and allows CDN caching.
  Implementation: export const revalidate = 60 at the route level.
  Exception: the /products?instock=true filter is dynamic (inventory changes
  frequently). Override with force-dynamic for filtered stock views.

PRODUCT DETAIL (/products/[slug])
  Strategy: SSG with generateStaticParams for popular products + ISR for rest
  Rationale: Most traffic goes to ~1000 popular products (80/20 rule).
  Pre-render those at build time; ISR handles the long tail.
  Implementation: generateStaticParams returns top 1000 product slugs;
  all others fall back to on-demand ISR.

CART (/cart)
  Strategy: CSR (Client Component)
  Rationale: Cart state lives in Zustand (persisted to localStorage). The cart
  is inherently user-specific and changes frequently — no benefit from SSR.
  The cart page renders as a client-side shell that reads from Zustand.

CHECKOUT (/checkout)
  Strategy: SSR (force-dynamic)
  Rationale: Checkout requires: session validation (authentication check),
  current cart contents (from the database, not localStorage — to prevent
  manipulation), and real-time inventory validation. All require request-time data.
  Security note: never trust cart contents from the client; always re-read
  from the DB on checkout initiation.

ORDER HISTORY (/account/orders)
  Strategy: SSR (force-dynamic)
  Rationale: Requires authentication; order data is user-specific.
  RSC fetches orders server-side, passes to Client Components for interaction.

ADMIN PANEL (/admin)
  Strategy: SSR + Middleware protection
  Rationale: Role-protected; data changes frequently; admins need real-time data.
```

_Reference: Part XI (Next.js Rendering Systems)_

---

### ADR-2: Cart State Architecture

```
THE DECISION: Cart state in Zustand (client) + synced to DB on checkout

CONSIDERED ALTERNATIVES:
  Option A: Cart in server DB only (no client state)
    PRO: single source of truth, works across devices
    CON: every cart interaction requires an API call + round trip → laggy UX

  Option B: Cart in localStorage/Zustand only (no server sync until checkout)
    PRO: instant UX, no server calls until needed
    CON: cart lost if localStorage is cleared, no cross-device sync

  Option C (chosen): Zustand + localStorage for UX, DB sync at checkout
    PRO: instant add-to-cart (optimistic, no round trip)
    PRO: persists across sessions on same device
    CON: cart is not shared across devices for anonymous users
    MITIGATION: on login, merge the Zustand cart with the DB cart (client-side)
```

```ts
// store/cart-store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  imageUrl: string;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: Omit<CartItem, "quantity">) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clearCart: () => void;
  total: number;
  itemCount: number;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (newItem) =>
        set((state) => {
          const existing = state.items.find(
            (i) => i.productId === newItem.productId,
          );
          if (existing) {
            return {
              items: state.items.map((i) =>
                i.productId === newItem.productId
                  ? { ...i, quantity: i.quantity + 1 }
                  : i,
              ),
            };
          }
          return { items: [...state.items, { ...newItem, quantity: 1 }] };
        }),

      removeItem: (productId) =>
        set((state) => ({
          items: state.items.filter((i) => i.productId !== productId),
        })),

      updateQuantity: (productId, quantity) =>
        set((state) => ({
          items:
            quantity === 0
              ? state.items.filter((i) => i.productId !== productId)
              : state.items.map((i) =>
                  i.productId === productId ? { ...i, quantity } : i,
                ),
        })),

      clearCart: () => set({ items: [] }),

      get total() {
        return get().items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      },

      get itemCount() {
        return get().items.reduce((sum, i) => sum + i.quantity, 0);
      },
    }),
    {
      name: "shopping-cart",
      partialize: (state) => ({ items: state.items }), // only persist items, not computed
    },
  ),
);
```

_Reference: Part XVI (State Management), Part XXIV (Anti-Patterns: State Management)_

---

### ADR-3: Checkout Security Model

```
CRITICAL: NEVER trust the client for financial transactions.

INSECURE PATTERN (don't do this):
  Client sends: { items: [{id: 'p1', price: 9.99, qty: 1}], total: 9.99 }
  Server creates Stripe PaymentIntent for $9.99
  → An attacker can send { price: 0.01 } and pay $0.01 for any item

SECURE PATTERN (what we implement):
  1. Client sends: { cartItems: [{productId: 'p1', quantity: 1}] }
     (only IDs and quantities — no prices from client)
  2. Server Action:
     a. Verifies user is authenticated
     b. Reads product prices from DATABASE for each productId
     c. Validates inventory (is each item still in stock at this quantity?)
     d. Calculates the total SERVER-SIDE using DB prices
     e. Creates Stripe PaymentIntent with the server-calculated amount
     f. Returns the client_secret to the client
  3. Client uses client_secret to render Stripe's payment UI
  4. After payment: Stripe calls our webhook to confirm
  5. Webhook handler (server): creates the order in DB, decrements inventory,
     sends confirmation email

THE WEBHOOK IS AUTHORITATIVE:
  We don't mark an order as complete based on the client's "payment succeeded" callback.
  We wait for the Stripe webhook (which calls our /api/webhooks/stripe endpoint).
  Stripe signs each webhook with a secret — we verify the signature.
  Only then do we create the order.
```

```ts
// app/actions/checkout.ts
"use server";
import Stripe from "stripe";
import { getSession } from "@/lib/auth/session";
import { db } from "@/lib/db";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

interface CartItemInput {
  productId: string;
  quantity: number;
}

export async function createPaymentIntent(cartItems: CartItemInput[]) {
  const session = await getSession();
  if (!session) throw new Error("Authentication required");

  // Fetch prices from DB (NEVER trust client-supplied prices):
  const products = await db.product.findMany({
    where: { id: { in: cartItems.map((i) => i.productId) } },
    select: { id: true, price: true, stock: true, name: true },
  });

  // Validate stock:
  for (const cartItem of cartItems) {
    const product = products.find((p) => p.id === cartItem.productId);
    if (!product) throw new Error(`Product ${cartItem.productId} not found`);
    if (product.stock < cartItem.quantity) {
      throw new Error(`${product.name} only has ${product.stock} in stock`);
    }
  }

  // Calculate total from DB prices:
  const totalCents = cartItems.reduce((sum, cartItem) => {
    const product = products.find((p) => p.id === cartItem.productId)!;
    return sum + Math.round(product.price * 100) * cartItem.quantity;
  }, 0);

  // Create PaymentIntent with server-calculated amount:
  const paymentIntent = await stripe.paymentIntents.create({
    amount: totalCents,
    currency: "usd",
    metadata: {
      userId: session.userId,
      // Store cart as metadata for webhook reconstruction:
      cartItems: JSON.stringify(cartItems),
    },
  });

  return { clientSecret: paymentIntent.client_secret! };
}
```

_Reference: Part XXI (Security - CSRF & Auth)_

---

## Product Catalog: Caching and Search

```ts
// app/products/page.tsx — ISR product listing
import { Suspense } from 'react';
import { ProductGrid } from '@/features/product-catalog';
import { SearchAndFilter } from '@/features/product-catalog';

export const revalidate = 60; // ISR: revalidate every minute

interface SearchParams {
  q?: string;
  category?: string;
  sort?: 'price-asc' | 'price-desc' | 'newest' | 'rating';
  page?: string;
}

export default async function ProductsPage({
  searchParams,
}: {
  searchParams: SearchParams;
}) {
  return (
    <div className="products-layout">
      <SearchAndFilter initialParams={searchParams} />
      <Suspense fallback={<ProductGridSkeleton />} key={JSON.stringify(searchParams)}>
        {/* key forces re-suspense when searchParams change */}
        <ProductGrid searchParams={searchParams} />
      </Suspense>
    </div>
  );
}
```

```ts
// features/product-catalog/components/ProductGrid.tsx
async function ProductGrid({ searchParams }: { searchParams: SearchParams }) {
  const products = await db.product.findMany({
    where: {
      ...(searchParams.q ? {
        name: { contains: searchParams.q, mode: 'insensitive' },
      } : {}),
      ...(searchParams.category ? { category: searchParams.category } : {}),
    },
    orderBy: {
      'price-asc': { price: 'asc' },
      'price-desc': { price: 'desc' },
      'newest': { createdAt: 'desc' },
      'rating': { averageRating: 'desc' },
    }[searchParams.sort ?? 'rating'] ?? { averageRating: 'desc' },
    take: 20,
    skip: (parseInt(searchParams.page ?? '1') - 1) * 20,
    select: {
      id: true, slug: true, name: true, price: true,
      imageUrl: true, averageRating: true, reviewCount: true,
    },
  });

  return (
    <ul className="product-grid">
      {products.map(product => (
        <li key={product.id}>
          <ProductCard product={product} />
        </li>
      ))}
    </ul>
  );
}
```

_Reference: Part XII (Next.js Caching), Part XVIII (Networking - BFF)_

---

## Image Optimization Strategy

```tsx
// features/product-catalog/components/ProductCard.tsx
import Image from "next/image";

function ProductCard({
  product,
  priority = false,
}: {
  product: Product;
  priority?: boolean; // true for first 4 above-the-fold cards
}) {
  return (
    <article className="product-card">
      <div className="product-card__image">
        <Image
          src={product.imageUrl}
          alt={product.name}
          width={400}
          height={400}
          priority={priority}
          loading={priority ? "eager" : "lazy"}
          sizes="(max-width: 640px) calc(50vw - 24px), (max-width: 1024px) calc(33vw - 24px), 280px"
          className="product-card__img"
        />
      </div>
      <div className="product-card__body">
        <h3 className="product-card__name">{product.name}</h3>
        <PriceDisplay price={product.price} />
        <Rating value={product.averageRating} count={product.reviewCount} />
      </div>
      <AddToCartButton product={product} />
    </article>
  );
}
```

_Reference: Part XV (Performance - Bundle Optimization), Part XXIV (Anti-Patterns: Image Performance)_

---

## Testing Strategy

```
UNIT TESTS (Vitest + RTL):
  - Cart store logic (pure state transitions)
  - Price formatting, discount calculation utilities
  - Individual component states (ProductCard loading/error/success)
  - Server Actions (mocked DB, mocked Stripe)

INTEGRATION TESTS (Vitest + MSW):
  - Add to cart flow (component + Zustand + MSW intercepting product fetch)
  - Search form (SearchInput + URL update + MSW for autocomplete)
  - Checkout Server Action (validation, DB interaction, Stripe call)

E2E TESTS (Playwright):
  CRITICAL PATH 1: Browse → Add to Cart → Checkout → Order Confirmation
  CRITICAL PATH 2: Search → Filter → Product Detail → Add to Cart
  CRITICAL PATH 3: Login → Order History → View Order Detail

  Use Stripe's test cards (4242 4242 4242 4242) for payment flow E2E tests
  Use a seeded test database (separate from production)
  Seed at least: 10 products, 1 regular user, 1 admin user, 2 completed orders
```

_Reference: Part XXII (Testing - all four docs)_

---

## Security Checklist

```
✅ Authentication: NextAuth.js with HTTP-only session cookies
✅ Authorization: Middleware protects /account/* and /admin/*
✅ Server Actions check auth before any DB write
✅ Prices never accepted from client (fetched from DB server-side)
✅ Stripe webhook signature verification (prevent spoofed webhooks)
✅ CSP header with nonces (in middleware.ts)
✅ All user-generated content (product reviews) sanitized with DOMPurify
✅ Rate limiting on checkout and auth endpoints
✅ Input validation with Zod on all Server Actions
✅ Prisma with parameterized queries (SQL injection prevention)
✅ Secrets in environment variables (never in source code)
✅ NEXT_PUBLIC_ prefix only for values safe to expose to clients
```

_Reference: Part XXI (Security - all four docs)_

---

## Performance Checklist

```
CORE WEB VITALS TARGETS:
  LCP: < 2.5s (product images are the LCP element on most pages)
  INP: < 200ms (Add to Cart button click)
  CLS: < 0.1 (images have explicit dimensions; no dynamic insertions above content)

OPTIMIZATIONS IMPLEMENTED:
✅ Product images: next/image with explicit width/height + lazy loading
✅ Hero image on homepage: priority={true} for LCP optimization
✅ Product listing: ISR (60s) for CDN caching
✅ Cart add: optimistic update in Zustand (instant UI response)
✅ Search: 300ms debounce on autocomplete
✅ Route prefetching: next/link prefetches on hover
✅ Code splitting: Checkout flow is dynamic import (only loaded on /checkout)
✅ Virtualization: Order history list (react-window for >50 orders)
✅ Bundle: no unused libraries, lodash-es instead of lodash
```

_Reference: Part XV (Performance Engineering)_

---

## Deployment Configuration

```yaml
# vercel.json (or equivalent configuration)
# For Vercel deployment with proper edge middleware

# Environment variables needed:
# DATABASE_URL (Neon, Supabase, or Railway PostgreSQL)
# NEXTAUTH_SECRET (random string, at least 32 chars)
# NEXTAUTH_URL (your production domain)
# STRIPE_SECRET_KEY (sk_live_... for production)
# STRIPE_PUBLISHABLE_KEY (pk_live_... - safe for NEXT_PUBLIC_)
# STRIPE_WEBHOOK_SECRET (from Stripe webhook dashboard)
# SENTRY_DSN (for error monitoring)
# NEXT_PUBLIC_SENTRY_DSN (browser error monitoring)
```

---

## Key Learning Outcomes

After building this project, you should be able to articulate:

1. **Why checkout must be server-side:** Security requirements (price validation, inventory check) that prevent client-side cart state from being authoritative for financial transactions

2. **The ISR + dynamic hybrid:** How different sections of an e-commerce site have different freshness requirements, leading to a mixed rendering strategy within one Next.js application

3. **Optimistic UI with rollback:** How Zustand's instant cart updates provide good UX while the server remains the authority on what's actually been purchased

4. **The webhook pattern:** Why Stripe's webhook is the authoritative signal for order completion, not the client's payment callback

5. **Cache invalidation strategy:** How `revalidateTag('product-{id}')` in admin product update actions propagates to invalidate product pages and listing pages

---

## Further Reading

- [Stripe Next.js integration guide](https://stripe.com/docs/stripe-js/react)
- [NextAuth.js v5 documentation](https://authjs.dev/)
- [Prisma with Next.js](https://www.prisma.io/nextjs)
- Related handbook sections: Parts IX–XIII (Next.js), Part XVI (State Management), Part XXI (Security)

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
