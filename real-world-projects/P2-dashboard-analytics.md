# P2 · Real-World Project: Analytics Dashboard

> **An analytics dashboard is one of the most architecturally demanding frontend projects because it requires every rendering strategy in one application: some charts are static (weekly summaries), some are real-time (live visitor count), some are user-personalized (your team's metrics), and some are computationally expensive (large dataset aggregations). This project covers the engineering decisions that make a dashboard performant and maintainable at scale: data visualization architecture, real-time update strategies, progressive loading patterns, and the specific challenges of rendering complex interactive charts in a Next.js App Router context.**

---

## Project Overview

**What you'll build:**

- Overview dashboard with key metrics (DAU, revenue, conversion rate)
- Real-time active user counter (SSE)
- Interactive charts with drill-down (Recharts)
- Date range picker affecting all charts
- CSV export for large datasets (streaming)
- Role-based dashboard views (admin vs team member)
- Saved report configurations

**Technology choices:**

- Next.js 15 (App Router)
- Recharts (charts — SSR-compatible)
- TanStack Query (client-side data refreshing)
- Zustand (dashboard filter state)
- Prisma + PostgreSQL with aggregation queries
- `date-fns` (date manipulation — tree-shakeable)

---

## Architecture Decision Record

### ADR-1: Which Parts of the Dashboard Are Server vs Client Components

```
THE TENSION:
  Charts require JavaScript in the browser (they're interactive, use canvas/SVG).
  But the DATA for charts can be fetched server-side.
  The question: how much should run server-side vs client-side?

HYBRID STRATEGY:

LAYER 1: Data fetching → Server Component (RSC)
  The async data fetch happens in a Server Component.
  No client JS overhead for the fetch itself.
  Passes data as props to chart components.

LAYER 2: Chart rendering → Client Component (required)
  Recharts needs the browser (it measures container dimensions, uses ResizeObserver).
  'use client' is required for all chart components.
  BUT: the data passed in is already fetched — no client-side data fetching
  for the initial render.

LAYER 3: Interactivity → Client Component
  Date range picker, drill-down, hover tooltips: all 'use client'.
  Filter changes trigger a full re-fetch via TanStack Query.

RESULT: Server renders the page shell and initial data; charts render in the
browser with pre-fetched data (no client-side fetch waterfall on initial load).
```

```tsx
// app/dashboard/page.tsx (Server Component)
export const dynamic = "force-dynamic"; // dashboard data is always fresh

export default async function DashboardPage() {
  const session = await requireAuth();
  const dateRange = getDefaultDateRange(); // last 30 days

  // Fetch summary metrics server-side (no client JS for this):
  const [kpiData, revenueData, userGrowthData] = await Promise.all([
    getKPIs(session.orgId, dateRange),
    getRevenueTimeSeries(session.orgId, dateRange),
    getUserGrowth(session.orgId, dateRange),
  ]);

  return (
    <DashboardLayout>
      {/* KPI cards: Server Component (no interactivity needed) */}
      <KPIGrid kpis={kpiData} />

      {/* Charts: Client Components with pre-fetched data */}
      <RevenueChart initialData={revenueData} orgId={session.orgId} />
      <UserGrowthChart initialData={userGrowthData} orgId={session.orgId} />

      {/* Real-time section: Client Component */}
      <LiveMetrics orgId={session.orgId} />
    </DashboardLayout>
  );
}
```

_Reference: Part X (Server Components), Part XI (Rendering Systems)_

---

### ADR-2: Chart Data Refresh Strategy

```
PROBLEM: Charts show stale data if the user stays on the dashboard for hours.
Options for keeping charts fresh:

OPTION A: Full page refresh (bad UX)
OPTION B: Manual "Refresh" button (acceptable but requires user action)
OPTION C: Automatic background refresh via TanStack Query
OPTION D: WebSocket for real-time push (overkill for charts that change every minute)
OPTION E: SSE for lightweight push (good for real-time counters)

OUR STRATEGY (context-dependent):

SUMMARY CHARTS (revenue, user growth): TanStack Query with refetchInterval=5min
  Rationale: these metrics change slowly; 5-minute background refresh is sufficient
  The initial data is pre-fetched by the Server Component (fast first paint)
  Subsequent refreshes happen in the background via TanStack Query

REAL-TIME COUNTER (live visitors): SSE
  Rationale: needs second-by-second updates; SSE is perfect for this use case
  (server → client, no need for bidirectional communication)
  Implementation: /api/live-visitors SSE endpoint, EventSource in the component

DRILL-DOWN DATA (clicked into a chart element): on-demand fetch via TanStack Query
  Rationale: only fetched when user explicitly requests more detail;
  no benefit from background refresh
```

```tsx
// features/dashboard/components/RevenueChart.tsx
"use client";
import { useQuery } from "@tanstack/react-query";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  Tooltip,
  ResponsiveContainer,
} from "recharts";

interface RevenueChartProps {
  initialData: RevenuePoint[];
  orgId: string;
  dateRange?: DateRange;
}

export function RevenueChart({
  initialData,
  orgId,
  dateRange,
}: RevenueChartProps) {
  const { data = initialData } = useQuery({
    queryKey: ["revenue", orgId, dateRange],
    queryFn: () => fetchRevenue(orgId, dateRange),
    initialData, // use server-fetched data as initial state (no loading flash)
    staleTime: 5 * 60 * 1000, // consider fresh for 5 minutes
    refetchInterval: 5 * 60 * 1000, // background refresh every 5 minutes
  });

  return (
    <div className="chart-container">
      <h2>Revenue</h2>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <XAxis dataKey="date" />
          <YAxis tickFormatter={(v) => `$${(v / 1000).toFixed(0)}k`} />
          <Tooltip
            formatter={(value: number) => [
              `$${value.toLocaleString()}`,
              "Revenue",
            ]}
          />
          <Line
            type="monotone"
            dataKey="revenue"
            stroke="#2563EB"
            dot={false}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

_Reference: Part XVI (State Management - TanStack Query), Part XVIII (WebSocket & SSE)_

---

### ADR-3: Global Date Range Filter

```
PROBLEM: The date range affects ALL charts on the dashboard.
When the user changes it, every chart should update.

STATE LOCATION: Zustand store (shared global state, changes frequently,
URL sync for shareability)

APPROACH: dual representation
  1. Zustand: the active filter (fast, reactive, drives chart refreshes)
  2. URL params: mirrors the Zustand state (shareable, bookmarkable,
     restored on page load)

WHY NOT JUST URL PARAMS:
  URL params work great for SSR (initial page load uses them).
  But for client-side filter changes, URL navigation causes a full server round-trip
  in Next.js App Router — the charts re-render on the server and stream down.
  For rapid filter interactions (changing date range repeatedly), this is too slow.

  Instead: Zustand drives the chart data fetching client-side (fast),
  URL is updated simultaneously (for shareability) but doesn't drive the renders.
```

```ts
// store/dashboard-filter-store.ts
import { create } from "zustand";
import { subscribeWithSelector } from "zustand/middleware";

interface DashboardFilterStore {
  dateRange: DateRange;
  granularity: "hour" | "day" | "week" | "month";
  setDateRange: (range: DateRange) => void;
  setGranularity: (g: "hour" | "day" | "week" | "month") => void;
}

export const useDashboardFilters = create<DashboardFilterStore>()(
  subscribeWithSelector((set) => ({
    dateRange: getDefaultDateRange(),
    granularity: "day",

    setDateRange: (range) => {
      set({ dateRange: range });
      // Sync to URL without navigation (replaceState):
      const url = new URL(window.location.href);
      url.searchParams.set("from", range.from.toISOString());
      url.searchParams.set("to", range.to.toISOString());
      window.history.replaceState({}, "", url);
    },

    setGranularity: (granularity) => set({ granularity }),
  })),
);
```

_Reference: Part XVI (State Management), Part XXIV (Anti-Patterns: State Management)_

---

## Real-Time Live Visitor Counter

```tsx
// app/api/live-visitors/route.ts
export const runtime = "nodejs"; // SSE requires Node.js (not Edge)

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const orgId = searchParams.get("orgId");

  const session = await getSession();
  if (!session || session.orgId !== orgId) {
    return new Response("Unauthorized", { status: 401 });
  }

  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      const send = (count: number) => {
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify({ count })}\n\n`),
        );
      };

      // Send initial count:
      const initial = await getActiveSessions(orgId!);
      send(initial);

      // Poll for updates (or use Redis pub/sub for real implementation):
      const interval = setInterval(async () => {
        const count = await getActiveSessions(orgId!);
        send(count);
      }, 5000); // every 5 seconds

      // Cleanup when client disconnects:
      request.signal.addEventListener("abort", () => {
        clearInterval(interval);
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
    },
  });
}

// features/dashboard/components/LiveMetrics.tsx
("use client");
import { useEffect, useState } from "react";

export function LiveMetrics({ orgId }: { orgId: string }) {
  const [visitorCount, setVisitorCount] = useState<number | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const eventSource = new EventSource(`/api/live-visitors?orgId=${orgId}`);

    eventSource.onopen = () => setIsConnected(true);

    eventSource.onmessage = (event) => {
      const { count } = JSON.parse(event.data);
      setVisitorCount(count);
    };

    eventSource.onerror = () => {
      setIsConnected(false);
      // EventSource auto-reconnects
    };

    return () => eventSource.close();
  }, [orgId]);

  return (
    <div className="live-metric">
      <span
        className={`live-indicator ${isConnected ? "live-indicator--active" : ""}`}
        aria-hidden="true"
      />
      <span className="sr-only">
        {isConnected ? "Live data" : "Reconnecting"}
      </span>
      <strong>{visitorCount ?? "—"}</strong>
      <span>active visitors right now</span>
    </div>
  );
}
```

_Reference: Part XVIII (WebSocket & SSE)_

---

## Large Dataset Export (Streaming CSV)

```ts
// app/api/export/route.ts
export async function GET(request: Request) {
  const session = await getSession();
  if (!session) return new Response("Unauthorized", { status: 401 });

  const { searchParams } = new URL(request.url);
  const dateRange = parseDateRange(searchParams);

  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      // CSV header:
      controller.enqueue(
        encoder.encode("date,visitors,pageviews,revenue,conversion_rate\n"),
      );

      // Stream data in batches (cursor-based):
      let cursor: string | null = null;
      let hasMore = true;

      while (hasMore) {
        const batch = await db.dailyMetrics.findMany({
          where: {
            orgId: session.orgId,
            date: { gte: dateRange.from, lte: dateRange.to },
            ...(cursor ? { id: { gt: cursor } } : {}),
          },
          orderBy: { date: "asc" },
          take: 1000,
        });

        for (const row of batch) {
          const line =
            [
              row.date.toISOString().split("T")[0],
              row.visitors,
              row.pageviews,
              row.revenue.toFixed(2),
              (row.conversionRate * 100).toFixed(2) + "%",
            ].join(",") + "\n";
          controller.enqueue(encoder.encode(line));
        }

        cursor = batch[batch.length - 1]?.id ?? null;
        hasMore = batch.length === 1000;
      }

      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/csv",
      "Content-Disposition": `attachment; filename="analytics-export-${Date.now()}.csv"`,
    },
  });
}
```

_Reference: Part XVIII (Networking - Streaming)_

---

## Performance Considerations for Charts

```
RECHARTS IN NEXT.JS APP ROUTER:
  Recharts uses ResizeObserver and requires a browser environment.
  It CANNOT render in a Server Component.
  Always wrap Recharts components in 'use client'.

  BUT: you can still benefit from SSR for the chart's DATA:
  → Server Component fetches data
  → Passes to 'use client' chart component as initialData
  → TanStack Query takes over for subsequent refreshes

PERFORMANCE OPTIMIZATION FOR CHARTS:

1. LARGE DATASETS: don't render more data points than pixels
   A chart that's 800px wide doesn't benefit from 10,000 data points.
   Downsample to the chart's pixel width:
   function downsample<T>(data: T[], maxPoints: number): T[] {
     if (data.length <= maxPoints) return data;
     const step = Math.ceil(data.length / maxPoints);
     return data.filter((_, i) => i % step === 0);
   }

2. CHART RE-RENDERS: charts re-render when their data prop changes.
   Use useMemo to stabilize the data reference:
   const chartData = useMemo(() => processForChart(rawData), [rawData]);

3. CANVAS vs SVG: Recharts uses SVG by default.
   For >1000 data points, consider canvas-based alternatives (Chart.js, Victory Native)
   because SVG DOM nodes are expensive to create and update.

4. ANIMATION: disable animations for large datasets (they're slow):
   <LineChart isAnimationActive={data.length < 100}>

5. CODE SPLITTING: chart libraries are large.
   dynamic import chart components so they're not in the initial bundle:
   const RevenueChart = dynamic(() => import('./RevenueChart'), { ssr: false });
```

_Reference: Part XV (Performance), Part XXIV (Performance Anti-Patterns)_

---

## Accessibility for Data Visualizations

```tsx
// Making charts accessible to screen readers:
function AccessibleLineChart({
  data,
  title,
}: {
  data: DataPoint[];
  title: string;
}) {
  return (
    <figure>
      <figcaption>{title}</figcaption>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data} role="img" aria-label={`${title} line chart`}>
          {/* ... chart config ... */}
        </LineChart>
      </ResponsiveContainer>
      {/* Data table for screen readers (visually hidden): */}
      <table className="sr-only">
        <caption>{title} data</caption>
        <thead>
          <tr>
            <th scope="col">Date</th>
            <th scope="col">Value</th>
          </tr>
        </thead>
        <tbody>
          {data.map((point) => (
            <tr key={point.date}>
              <th scope="row">{formatDate(point.date)}</th>
              <td>{point.value.toLocaleString()}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </figure>
  );
}
```

_Reference: Part XX (Accessibility Engineering)_

---

## Testing Strategy

```
UNIT TESTS:
  - Date range utility functions (getDefaultDateRange, formatDateRange)
  - Data processing functions (downsample, aggregateByPeriod)
  - Filter store state transitions
  - CSV formatting logic

INTEGRATION TESTS (MSW):
  - Dashboard page with mocked API responses (loading → data → refresh)
  - Date range filter change → verify TanStack Query refetch with new params
  - Export endpoint (test streaming response via testApiHandler)
  - SSE endpoint (mock getActiveSessions, verify events are sent)

E2E TESTS (Playwright):
  - Login → View dashboard → Verify KPIs are visible
  - Change date range → Verify charts update
  - Click "Export" → Verify CSV download begins
  - Verify live visitor counter changes (mock SSE in Playwright)
```

---

## Key Learning Outcomes

After building this project, you should be able to articulate:

1. **The Server Component + Client Component chart pattern:** Why chart data can be fetched server-side even though charts must render client-side, and how to pass initial data to avoid a client-side loading flash

2. **The dual state representation pattern:** Why the date range filter needs both Zustand (for fast client updates) and URL params (for shareability), and how to keep them in sync

3. **SSE for real-time metrics:** Why SSE is the right choice for live visitor counts (not WebSocket, not polling), and the specific implementation pattern including cleanup

4. **Streaming CSV exports:** How to export large datasets without loading everything into memory, and why Response streaming is the correct approach

5. **Chart accessibility:** The visually-hidden table pattern for making visualizations accessible without degrading the visual design

---

_Part of the [React + Next.js Engineering Handbook](../../README.md)_
