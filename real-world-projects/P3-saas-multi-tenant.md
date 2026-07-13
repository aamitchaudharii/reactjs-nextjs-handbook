# P3 · Real-World Project: Multi-Tenant SaaS Application

> **Building a multi-tenant SaaS application introduces architectural challenges absent from single-tenant apps: every database query must be scoped to the correct tenant, middleware must route subdomains or URL segments to the right workspace, billing must be enforced before protected features are accessible, and a bug in tenant isolation can expose one customer's data to another. This project covers the engineering decisions that make a multi-tenant Next.js application secure, scalable, and maintainable — from the database schema strategy to the middleware routing pattern to the subscription gating system.**

---

## Project Overview

**What you'll build:**

- Workspace-based multi-tenancy (each org gets a subdomain: `acme.app.com`)
- Team membership with role-based access (owner, admin, member)
- Subscription tiers with feature gating (free, pro, enterprise)
- Billing integration with Stripe Billing (subscriptions, not one-time payments)
- Workspace settings (invite members, manage billing, customize branding)
- Audit log of all workspace actions

**Technology choices:**

- Next.js 15 (App Router + Middleware for subdomain routing)
- Prisma + PostgreSQL (Row Level Security for tenant isolation)
- NextAuth.js v5 (auth with workspace context in session)
- Stripe Billing (subscriptions with webhooks)
- Resend (transactional email for invitations)

---

## Architecture Decision Record

### ADR-1: Multi-Tenancy Strategy

```
THREE APPROACHES TO MULTI-TENANCY:

APPROACH A: Separate database per tenant
  PRO: complete isolation — one tenant's data literally cannot leak to another
  PRO: per-tenant database customization (schema migrations, backups)
  CON: expensive (hundreds of databases for hundreds of tenants)
  CON: connection pooling complexity
  USE WHEN: enterprise compliance requirements (HIPAA, SOC2 with strict isolation)

APPROACH B: Shared database, separate schemas
  PRO: better isolation than shared tables; cheaper than separate databases
  CON: Prisma doesn't support dynamic schema switching cleanly
  CON: Schema-per-tenant still has high operational overhead at scale
  USE WHEN: regulated industries with moderate tenant count

APPROACH C: Shared database, shared tables with tenant_id column (chosen)
  PRO: simple to implement and scale
  PRO: single Prisma schema, single migration path
  PRO: efficient at scale (thousands of tenants on one DB)
  CON: requires STRICT enforcement of tenant_id filtering in every query
       (a missing WHERE clause can expose all tenants' data)
  MITIGATION: PostgreSQL Row Level Security + application-level checks
  USE WHEN: standard B2B SaaS with no extreme isolation requirements

OUR CHOICE: Approach C with Row Level Security
```

```prisma
// prisma/schema.prisma

model Workspace {
  id          String   @id @default(cuid())
  slug        String   @unique // used for subdomain: slug.app.com
  name        String
  plan        Plan     @default(FREE)
  stripeCustomerId     String?
  stripeSubscriptionId String?
  createdAt   DateTime @default(now())
  members     WorkspaceMember[]
  projects    Project[]
  auditLogs   AuditLog[]
}

model WorkspaceMember {
  id          String    @id @default(cuid())
  workspaceId String
  userId      String
  role        Role      @default(MEMBER)
  joinedAt    DateTime  @default(now())
  workspace   Workspace @relation(fields: [workspaceId], references: [id])
  user        User      @relation(fields: [userId], references: [id])
  @@unique([workspaceId, userId])
}

enum Role { OWNER ADMIN MEMBER }
enum Plan { FREE PRO ENTERPRISE }

model Project {
  id          String    @id @default(cuid())
  workspaceId String    // EVERY resource has workspaceId
  name        String
  createdAt   DateTime  @default(now())
  workspace   Workspace @relation(fields: [workspaceId], references: [id])
}
```

---

### ADR-2: Subdomain Routing with Middleware

```
URL STRUCTURE:
  app.com/login           → auth pages (no workspace context)
  app.com/signup          → onboarding (create workspace)
  {slug}.app.com/         → workspace dashboard
  {slug}.app.com/projects → workspace projects
  {slug}.app.com/settings → workspace settings
  app.com/api/...         → API (workspace from auth token, not subdomain)

WHY SUBDOMAINS:
  URL-based routing (app.com/workspace/acme) is simpler to implement
  but subdomains provide:
  - Better perceived isolation ("this is YOUR app, at YOUR URL")
  - Cookie isolation (optional: separate session cookies per subdomain)
  - Future: per-tenant SSL certificates or custom domains
  - Cleaner URLs: acme.app.com/projects vs app.com/w/acme/projects
```

```ts
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

const ROOT_DOMAIN = process.env.NEXT_PUBLIC_ROOT_DOMAIN!; // e.g., 'app.com'

export function middleware(request: NextRequest) {
  const hostname = request.headers.get("host") ?? "";
  const url = request.nextUrl;

  // Strip port for local development:
  const currentHost = hostname.replace(`:${process.env.PORT ?? 3000}`, "");

  // Check if we're on a workspace subdomain:
  const isSubdomain =
    currentHost !== ROOT_DOMAIN &&
    currentHost !== `www.${ROOT_DOMAIN}` &&
    currentHost.endsWith(`.${ROOT_DOMAIN}`);

  if (isSubdomain) {
    // Extract the workspace slug from the subdomain:
    const workspaceSlug = currentHost.replace(`.${ROOT_DOMAIN}`, "");

    // Rewrite the URL to /workspace/[slug]/...
    // The user sees: acme.app.com/projects
    // Next.js renders: /workspace/acme/projects
    return NextResponse.rewrite(
      new URL(`/workspace/${workspaceSlug}${url.pathname}`, request.url),
    );
  }

  // Root domain: serve auth/marketing pages as-is
  return NextResponse.next();
}

export const config = {
  matcher: [
    // Skip static files and API routes:
    "/((?!api|_next/static|_next/image|favicon.ico).*)",
  ],
};
```

```
LOCAL DEVELOPMENT:
  Subdomains don't work on localhost by default.
  Use: acme.localhost:3000 (Chrome supports *.localhost natively)
  Or: add entries to /etc/hosts:
    127.0.0.1 acme.localhost
    127.0.0.1 beta.localhost
  Or: use a tool like localcan or Caddy for local SSL + wildcard DNS
```

_Reference: Part IX (Next.js Core - Middleware)_

---

### ADR-3: Tenant Context in the Session

```
THE PROBLEM: After login, the user might belong to MULTIPLE workspaces.
  "Which workspace is the current request for?" needs to be answered
  on every authenticated request.

APPROACH: Include workspaceId in the session based on the subdomain.

FLOW:
  1. User logs in at app.com/login → session created with userId
  2. User navigates to acme.app.com/dashboard
  3. Middleware rewrites URL to /workspace/acme/dashboard
  4. The page's auth check resolves acme.app.com → workspaceSlug=acme
     → queries DB for workspace by slug
     → verifies the logged-in user is a member of that workspace
     → enriches the session context with { workspaceId, workspaceSlug, role }
  5. All child Server Components use this enriched context for their DB queries

WHY NOT PUT WORKSPACE IN THE JWT/SESSION DIRECTLY:
  A user might belong to 10 workspaces. Their session would need to be
  invalidated and re-issued every time they join or leave a workspace.
  Instead: the session has userId, and the workspace context is derived
  at request time from the subdomain + database membership check.
```

```ts
// lib/workspace-context.ts
import { cache } from "react";
import { getSession } from "@/lib/auth/session";
import { db } from "@/lib/db";

export type WorkspaceContext = {
  workspaceId: string;
  workspaceSlug: string;
  plan: "FREE" | "PRO" | "ENTERPRISE";
  role: "OWNER" | "ADMIN" | "MEMBER";
  userId: string;
};

// cache() memoizes per-request: multiple Server Components calling
// getWorkspaceContext() within one render hit the DB only once:
export const getWorkspaceContext = cache(
  async (slug: string): Promise<WorkspaceContext> => {
    const session = await getSession();
    if (!session) {
      throw new Error("UNAUTHENTICATED");
    }

    const membership = await db.workspaceMember.findFirst({
      where: {
        userId: session.userId,
        workspace: { slug },
      },
      include: {
        workspace: { select: { id: true, slug: true, plan: true } },
      },
    });

    if (!membership) {
      throw new Error("UNAUTHORIZED"); // user is not a member of this workspace
    }

    return {
      workspaceId: membership.workspace.id,
      workspaceSlug: membership.workspace.slug,
      plan: membership.workspace.plan,
      role: membership.role,
      userId: session.userId,
    };
  },
);
```

---

### ADR-4: Feature Gating by Subscription Tier

```
PLAN LIMITS:
  FREE: 3 projects, 2 team members, 1GB storage, no API access
  PRO: unlimited projects, 20 team members, 50GB storage, API access
  ENTERPRISE: unlimited everything, SSO, audit logs, SLA

ENFORCEMENT LAYERS (defense in depth):
  Layer 1: UI (hide/disable features above the plan)
  Layer 2: Server Action / Route Handler validation (check plan before mutation)
  Layer 3: Database constraint (optional: Postgres check constraints for counts)

WHY ALL THREE LAYERS:
  Layer 1 alone: UI can be bypassed by calling the API directly
  Layer 1 + 2: strong enforcement, but no DB-level guarantee
  All three: belt + suspenders + safe — no edge case allows a limit bypass
```

```ts
// lib/feature-gates.ts
type Feature =
  | "unlimited_projects"
  | "api_access"
  | "sso"
  | "audit_logs"
  | "custom_domain";

const PLAN_FEATURES: Record<string, Feature[]> = {
  FREE: [],
  PRO: ["unlimited_projects", "api_access"],
  ENTERPRISE: [
    "unlimited_projects",
    "api_access",
    "sso",
    "audit_logs",
    "custom_domain",
  ],
};

const PLAN_LIMITS = {
  FREE: { projects: 3, members: 2, storage: 1 },
  PRO: { projects: Infinity, members: 20, storage: 50 },
  ENTERPRISE: { projects: Infinity, members: Infinity, storage: Infinity },
};

export function hasFeature(plan: string, feature: Feature): boolean {
  return PLAN_FEATURES[plan]?.includes(feature) ?? false;
}

export function getLimit(
  plan: string,
  resource: keyof typeof PLAN_LIMITS.FREE,
): number {
  return PLAN_LIMITS[plan as keyof typeof PLAN_LIMITS]?.[resource] ?? 0;
}

// Server Action using feature gate:
("use server");
export async function createProject(workspaceSlug: string, name: string) {
  const ctx = await getWorkspaceContext(workspaceSlug);

  // Enforce plan limits:
  if (!hasFeature(ctx.plan, "unlimited_projects")) {
    const currentCount = await db.project.count({
      where: { workspaceId: ctx.workspaceId },
    });
    if (currentCount >= getLimit(ctx.plan, "projects")) {
      throw new Error("PROJECT_LIMIT_REACHED");
    }
  }

  // Enforce RBAC: only owners and admins can create projects
  if (ctx.role === "MEMBER") {
    throw new Error("INSUFFICIENT_PERMISSIONS");
  }

  const project = await db.project.create({
    data: { name, workspaceId: ctx.workspaceId },
  });

  await writeAuditLog(ctx, "project.created", { projectId: project.id, name });
  revalidatePath(`/workspace/${workspaceSlug}/projects`);
  return project;
}
```

---

## Team Invitation Flow

```ts
// Inviting a new team member — involves email, tokens, and membership creation:

// app/actions/invite.ts
"use server";
import { Resend } from "resend";
import { nanoid } from "nanoid";

const resend = new Resend(process.env.RESEND_API_KEY);

export async function inviteMember(workspaceSlug: string, email: string) {
  const ctx = await getWorkspaceContext(workspaceSlug);

  // Check permissions (only owner/admin can invite):
  if (ctx.role === "MEMBER") throw new Error("INSUFFICIENT_PERMISSIONS");

  // Check member limit:
  const memberCount = await db.workspaceMember.count({
    where: { workspaceId: ctx.workspaceId },
  });
  if (memberCount >= getLimit(ctx.plan, "members")) {
    throw new Error("MEMBER_LIMIT_REACHED");
  }

  // Check: is this person already a member?
  const existingUser = await db.user.findUnique({ where: { email } });
  if (existingUser) {
    const existingMember = await db.workspaceMember.findFirst({
      where: { workspaceId: ctx.workspaceId, userId: existingUser.id },
    });
    if (existingMember) throw new Error("ALREADY_A_MEMBER");
  }

  // Create invitation token (expires in 7 days):
  const token = nanoid(32);
  await db.invitation.create({
    data: {
      email,
      workspaceId: ctx.workspaceId,
      token,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  // Send invitation email:
  await resend.emails.send({
    from: "invites@app.com",
    to: email,
    subject: `You've been invited to join ${ctx.workspaceSlug} on App`,
    react: InvitationEmail({
      workspaceName: ctx.workspaceSlug,
      inviterName: "A team member", // fetch actual name from session
      acceptUrl: `https://${ctx.workspaceSlug}.app.com/invite/${token}`,
    }),
  });

  await writeAuditLog(ctx, "member.invited", { email });
  return { success: true };
}
```

---

## Stripe Billing Integration

```ts
// Subscriptions differ from one-time payments (P1 e-commerce):
// - Subscriptions are ongoing; they can cancel, pause, upgrade, downgrade
// - Stripe sends webhooks for every subscription lifecycle event
// - Your DB must stay in sync with Stripe's subscription state

// app/api/webhooks/stripe/route.ts
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!,
    );
  } catch {
    return new Response("Invalid signature", { status: 400 });
  }

  switch (event.type) {
    case "customer.subscription.created":
    case "customer.subscription.updated": {
      const subscription = event.data.object as Stripe.Subscription;
      const plan = getPlanFromPriceId(subscription.items.data[0].price.id);
      await db.workspace.updateMany({
        where: { stripeCustomerId: subscription.customer as string },
        data: {
          plan,
          stripeSubscriptionId: subscription.id,
        },
      });
      break;
    }

    case "customer.subscription.deleted": {
      const subscription = event.data.object as Stripe.Subscription;
      await db.workspace.updateMany({
        where: { stripeCustomerId: subscription.customer as string },
        data: { plan: "FREE", stripeSubscriptionId: null },
      });
      break;
    }

    case "invoice.payment_failed": {
      // Notify workspace owner of failed payment
      const invoice = event.data.object as Stripe.Invoice;
      // ... send email, potentially downgrade access after grace period
      break;
    }
  }

  return new Response(null, { status: 200 });
}
```

---

## Audit Logging

```ts
// Every significant action in the workspace should be logged:

// lib/audit.ts
export async function writeAuditLog(
  ctx: WorkspaceContext,
  action: string,
  metadata: Record<string, unknown> = {},
) {
  // Non-blocking: don't await (audit log failure shouldn't block the action):
  db.auditLog
    .create({
      data: {
        workspaceId: ctx.workspaceId,
        userId: ctx.userId,
        action, // e.g., 'project.created', 'member.invited', 'plan.upgraded'
        metadata, // e.g., { projectId: 'p1', name: 'Website Redesign' }
        ipAddress: headers().get("x-forwarded-for") ?? null,
        userAgent: headers().get("user-agent") ?? null,
      },
    })
    .catch((err) => {
      console.error("[AuditLog] Failed to write audit log:", err);
      // Never throw — audit log failures are secondary to the primary action
    });
}
```

---

## Testing Strategy for Multi-Tenant Systems

```
THE CRITICAL TENANT ISOLATION TEST:
  Every data access test should verify that a user from Workspace A
  CANNOT access data from Workspace B, even with a valid session.

// Example isolation test:
test('user cannot access another workspace's projects', async () => {
  const workspaceA = await createTestWorkspace('workspace-a');
  const workspaceB = await createTestWorkspace('workspace-b');
  const userInA = await createTestUser({ workspaceId: workspaceA.id });

  // Authenticate as userInA but try to access workspace-b:
  await signInAs(userInA);
  const response = await fetch('http://workspace-b.localhost:3000/api/projects');
  expect(response.status).toBe(403); // NOT the projects from workspace-b
});

OTHER CRITICAL TEST CASES:
  - Creating a resource doesn't allow specifying a different workspaceId
  - Subscription downgrade enforces new limits on existing resources
  - Invitation token expiry is correctly enforced
  - Role change removes access to role-specific features immediately
  - Leaving a workspace removes all access
  - Workspace deletion cascades correctly (no orphaned data)
```

---

## Key Learning Outcomes

After building this project, you should be able to articulate:

1. **The three multi-tenancy strategies** and when to choose each — shared tables is usually right; separate databases is right for enterprise/compliance

2. **The subdomain routing pattern** — how Next.js middleware rewrites subdomain requests to workspace-specific routes without the user seeing the internal URL structure

3. **The session enrichment pattern** — why workspaceId comes from the subdomain + DB lookup, not from the session token, and why React's `cache()` prevents duplicate membership queries

4. **Defense-in-depth feature gating** — UI layer + Server Action layer + (optionally) DB constraint layer, and why all three are necessary

5. **Stripe subscription webhooks vs payment intents** — subscriptions use the Stripe Customer Portal and subscription lifecycle events; one-time payments use PaymentIntent; billing state always comes from Stripe via webhooks

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
