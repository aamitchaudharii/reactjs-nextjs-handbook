# ⚛️ React + Next.js Engineering Handbook

> **The deepest open-source React & Next.js engineering resource on GitHub.**
> Built for engineers who want to understand _how_ and _why_, not just _what_.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Stars](https://img.shields.io/github/stars/your-org/react-nextjs-engineering-handbook?style=social)](https://github.com/your-org/react-nextjs-engineering-handbook)

---

## 🧭 What This Handbook Is

This is **not** a tutorial.  
This is **not** a "getting started" guide.  
This is **not** surface-level documentation.

This is a **deep engineering handbook** — a university-level course built around:

- 🔬 React internals and runtime behavior
- ⚙️ Next.js architecture and rendering systems
- 🧠 How browsers actually render your UI
- 🚀 Production performance engineering
- 🏗️ Scalable frontend architecture
- 🔍 Real-world debugging workflows
- 📐 Advanced patterns and anti-patterns

If you have ever asked:

- _"Why does my component re-render so often?"_
- _"What actually happens after I call `setState`?"_
- _"How does Next.js generate HTML on the server?"_
- _"What is React Fiber and why does it matter?"_
- _"How does hydration work and why does it break?"_
- _"What is a rendering waterfall?"_
- _"How do I scale a React app to 10 million users?"_

...then this handbook was written for you.

---

## 🎯 Who This Is For

| Audience                             | What You Will Get                                              |
| ------------------------------------ | -------------------------------------------------------------- |
| **Intermediate frontend developers** | Deep understanding of React internals beyond docs              |
| **Senior React engineers**           | Architecture patterns, scaling strategies, performance systems |
| **Next.js engineers**                | Full App Router internals, RSC, streaming, caching             |
| **Self-taught developers**           | Structured university-level progression                        |
| **Senior interview candidates**      | Every deep-dive topic interviewers actually ask                |
| **Production engineers**             | Real-world debugging, profiling, architecture decisions        |

---

## 📚 Table of Contents

### Part I — React Core Internals

- [01 · What React Actually Is](docs/react-core/01-what-react-is.md)
- [02 · JSX Transformation & Runtime](docs/react-core/02-jsx-transformation.md)
- [03 · Virtual DOM Deep Dive](docs/react-core/03-virtual-dom.md)
- [04 · Component Architecture](docs/react-core/04-component-architecture.md)
- [05 · Unidirectional Data Flow](docs/react-core/05-unidirectional-data-flow.md)

### Part II — React Rendering Internals

- [06 · The Render Phase](docs/react-rendering/01-render-phase.md)
- [07 · The Commit Phase](docs/react-rendering/02-commit-phase.md)
- [08 · The React Scheduler](docs/react-rendering/03-scheduler.md)
- [09 · Batching & Update Queues](docs/react-rendering/04-batching.md)
- [10 · Effects: Layout vs Passive](docs/react-rendering/05-effects.md)

### Part III — React Fiber Architecture

- [11 · Why Fiber Exists](docs/react-fiber/01-why-fiber.md)
- [12 · Fiber Node Structure](docs/react-fiber/02-fiber-node.md)
- [13 · Fiber Tree Traversal](docs/react-fiber/03-fiber-tree-traversal.md)
- [14 · Interruptible Rendering](docs/react-fiber/04-interruptible-rendering.md)
- [15 · Time Slicing](docs/react-fiber/05-time-slicing.md)

### Part IV — Reconciliation

- [16 · The Diffing Algorithm](docs/reconciliation/01-diffing-algorithm.md)
- [17 · Element Identity & Keys](docs/reconciliation/02-keys.md)
- [18 · Child Reconciliation](docs/reconciliation/03-child-reconciliation.md)
- [19 · DOM Reuse Strategies](docs/reconciliation/04-dom-reuse.md)

### Part V — Hooks Internals

- [20 · useState Internals](docs/hooks-internals/01-usestate.md)
- [21 · useEffect Internals](docs/hooks-internals/02-useeffect.md)
- [22 · useMemo & useCallback](docs/hooks-internals/03-usememo-usecallback.md)
- [23 · useRef Behavior](docs/hooks-internals/04-useref.md)
- [24 · useReducer & useContext](docs/hooks-internals/05-usereducer-usecontext.md)
- [25 · Custom Hook Architecture](docs/hooks-internals/06-custom-hooks.md)

### Part VI — Re-rendering System

- [26 · What Causes Re-renders](docs/react-rendering/06-what-causes-rerenders.md)
- [27 · Reference Equality & Object Identity](docs/react-rendering/07-reference-equality.md)
- [28 · Context Re-render Propagation](docs/react-rendering/08-context-rerenders.md)
- [29 · Re-render Optimization Patterns](docs/react-rendering/09-optimization-patterns.md)

### Part VII — Concurrent React

- [30 · Concurrent Rendering Model](docs/concurrent-react/01-concurrent-model.md)
- [31 · startTransition & useTransition](docs/concurrent-react/02-transitions.md)
- [32 · useDeferredValue](docs/concurrent-react/03-deferred-value.md)
- [33 · Suspense Internals](docs/concurrent-react/04-suspense.md)
- [34 · Scheduling Priorities](docs/concurrent-react/05-scheduling.md)

### Part VIII — React Compiler

- [35 · React Compiler Architecture](docs/react-compiler/01-compiler.md)
- [36 · Automatic Memoization](docs/react-compiler/02-auto-memoization.md)
- [37 · Future of React](docs/react-compiler/03-future-react.md)

### Part IX — Next.js Core

- [38 · What Next.js Actually Is](docs/nextjs-core/01-what-nextjs-is.md)
- [39 · App Router vs Pages Router](docs/nextjs-core/02-app-vs-pages-router.md)
- [40 · File-System Routing Internals](docs/nextjs-core/03-routing-internals.md)
- [41 · Layout System Architecture](docs/nextjs-core/04-layout-system.md)
- [42 · Build Pipeline Internals](docs/nextjs-core/05-build-pipeline.md)

### Part X — Next.js Rendering Systems

- [43 · SSR — Server-Side Rendering](docs/nextjs-rendering/01-ssr.md)
- [44 · CSR — Client-Side Rendering](docs/nextjs-rendering/02-csr.md)
- [45 · SSG — Static Site Generation](docs/nextjs-rendering/03-ssg.md)
- [46 · ISR — Incremental Static Regeneration](docs/nextjs-rendering/04-isr.md)
- [47 · Streaming Rendering](docs/nextjs-rendering/05-streaming.md)
- [48 · Partial Rendering](docs/nextjs-rendering/06-partial-rendering.md)
- [49 · Edge Rendering](docs/nextjs-rendering/07-edge-rendering.md)

### Part XI — React Server Components

- [50 · What RSC Actually Is](docs/server-components/01-what-rsc-is.md)
- [51 · Server vs Client Execution](docs/server-components/02-server-vs-client.md)
- [52 · RSC Wire Protocol (Flight)](docs/server-components/03-flight-protocol.md)
- [53 · Streaming RSC Payloads](docs/server-components/04-streaming-payloads.md)
- [54 · RSC Patterns & Tradeoffs](docs/server-components/05-patterns.md)

### Part XII — Hydration

- [55 · How Hydration Works](docs/hydration/01-how-hydration-works.md)
- [56 · Hydration Mismatches](docs/hydration/02-hydration-mismatches.md)
- [57 · Progressive Hydration](docs/hydration/03-progressive-hydration.md)
- [58 · Selective Hydration](docs/hydration/04-selective-hydration.md)

### Part XIII — Next.js Caching Systems

- [59 · Browser Caching](docs/caching/01-browser-caching.md)
- [60 · CDN & Edge Caching](docs/caching/02-cdn-caching.md)
- [61 · Request Memoization](docs/caching/03-request-memoization.md)
- [62 · Data Cache](docs/caching/04-data-cache.md)
- [63 · Full Route Cache](docs/caching/05-full-route-cache.md)
- [64 · Router Cache](docs/caching/06-router-cache.md)
- [65 · Cache Invalidation Strategies](docs/caching/07-cache-invalidation.md)

### Part XIV — Next.js Advanced Internals

- [66 · Turbopack Architecture](docs/nextjs-core/06-turbopack.md)
- [67 · Middleware Internals](docs/nextjs-core/07-middleware.md)
- [68 · Server Actions](docs/nextjs-core/08-server-actions.md)
- [69 · Parallel & Intercepting Routes](docs/nextjs-core/09-parallel-routes.md)

### Part XV — Browser Rendering Internals

- [70 · Browser Rendering Pipeline](docs/browser-internals/01-rendering-pipeline.md)
- [71 · Critical Rendering Path](docs/browser-internals/02-critical-rendering-path.md)
- [72 · Reflow, Repaint & Compositing](docs/browser-internals/03-reflow-repaint.md)
- [73 · GPU Acceleration & Layers](docs/browser-internals/04-gpu-layers.md)
- [74 · Long Tasks & Main Thread](docs/browser-internals/05-long-tasks.md)

### Part XVI — Performance Engineering

- [75 · React Profiler & Flamegraphs](docs/performance/01-react-profiler.md)
- [76 · Memoization Engineering](docs/performance/02-memoization.md)
- [77 · Virtualization & Windowing](docs/performance/03-virtualization.md)
- [78 · Code Splitting & Lazy Loading](docs/performance/04-code-splitting.md)
- [79 · Bundle Optimization](docs/performance/05-bundle-optimization.md)
- [80 · Large List Rendering](docs/performance/06-large-lists.md)
- [81 · requestAnimationFrame & requestIdleCallback](docs/performance/07-raf-ric.md)

### Part XVII — State Management Deep Dive

- [82 · Context API Internals](docs/state-management/01-context-internals.md)
- [83 · Redux Architecture](docs/state-management/02-redux.md)
- [84 · Zustand Internals](docs/state-management/03-zustand.md)
- [85 · TanStack Query Internals](docs/state-management/04-tanstack-query.md)
- [86 · Server State vs Client State](docs/state-management/05-server-vs-client-state.md)
- [87 · Optimistic Updates](docs/state-management/06-optimistic-updates.md)

### Part XVIII — Build Systems

- [88 · AST Transformations & Babel](docs/build-systems/01-ast-babel.md)
- [89 · SWC Architecture](docs/build-systems/02-swc.md)
- [90 · Webpack Internals](docs/build-systems/03-webpack.md)
- [91 · Vite Internals](docs/build-systems/04-vite.md)
- [92 · Tree Shaking & Module Graphs](docs/build-systems/05-tree-shaking.md)
- [93 · ESM vs CommonJS](docs/build-systems/06-esm-cjs.md)

### Part XIX — Networking Engineering

- [94 · HTTP/1.1 vs HTTP/2 vs HTTP/3](docs/networking/01-http-versions.md)
- [95 · Streaming & Chunked Transfer](docs/networking/02-streaming.md)
- [96 · WebSocket & SSE](docs/networking/03-websocket-sse.md)
- [97 · Rendering Waterfalls](docs/networking/04-rendering-waterfalls.md)
- [98 · BFF Architecture](docs/networking/05-bff.md)

### Part XX — Architecture Patterns

- [99 · Feature-Based Architecture](docs/architecture/01-feature-based.md)
- [100 · Atomic Design](docs/architecture/02-atomic-design.md)
- [101 · Monorepo Architecture](docs/architecture/03-monorepo.md)
- [102 · Micro Frontend Architecture](docs/architecture/04-micro-frontends.md)
- [103 · Config-Driven UI](docs/architecture/05-config-driven-ui.md)

### Part XXI — Design Systems

- [104 · Design Token Systems](docs/design-systems/01-design-tokens.md)
- [105 · Theming Architecture](docs/design-systems/02-theming.md)
- [106 · Component API Design](docs/design-systems/03-component-api.md)

### Part XXII — Security

- [107 · XSS in React Apps](docs/security/01-xss.md)
- [108 · CSRF & Token Security](docs/security/02-csrf.md)
- [109 · CSP Headers](docs/security/03-csp.md)
- [110 · SSR Security Risks](docs/security/04-ssr-security.md)

### Part XXIII — Testing Engineering

- [111 · Testing Philosophy](docs/testing/01-philosophy.md)
- [112 · Component Testing Deep Dive](docs/testing/02-component-testing.md)
- [113 · Integration Testing](docs/testing/03-integration-testing.md)
- [114 · E2E with Playwright](docs/testing/04-playwright.md)

### Part XXIV — Debugging Engineering

- [115 · React Profiler Workflow](docs/debugging/01-react-profiler.md)
- [116 · Memory Leak Detection](docs/debugging/02-memory-leaks.md)
- [117 · Hydration Debugging](docs/debugging/03-hydration-debugging.md)
- [118 · Bundle Analysis Workflows](docs/debugging/04-bundle-analysis.md)

### Part XXV — Anti-Patterns

- [119 · The 20 Most Dangerous Anti-Patterns](docs/anti-patterns/01-top-antipatterns.md)
- [120 · Context Abuse](docs/anti-patterns/02-context-abuse.md)
- [121 · Render Trap Patterns](docs/anti-patterns/03-render-traps.md)

### Part XXVI — Real-World Projects

- [P1 · Real-Time Analytics Dashboard](docs/real-world-projects/01-analytics-dashboard.md)
- [P2 · Collaborative Kanban System](docs/real-world-projects/02-kanban.md)
- [P3 · Rich Text Editor](docs/real-world-projects/03-rich-text-editor.md)
- [P4 · Infinite Scrolling Data Table](docs/real-world-projects/04-infinite-table.md)
- [P5 · Config-Driven Admin Dashboard](docs/real-world-projects/05-admin-dashboard.md)
- [P6 · Canvas Rendering Engine](docs/real-world-projects/06-canvas-engine.md)

### Part XXVII — Interview Preparation

- [122 · Senior Frontend Interview: React Internals](docs/interview/01-react-internals.md)
- [123 · Senior Frontend Interview: Performance](docs/interview/02-performance.md)
- [124 · Senior Frontend Interview: Architecture](docs/interview/03-architecture.md)
- [125 · System Design: Frontend](docs/interview/04-system-design.md)

### Part XXVIII — Learning Roadmap

- [Learning Roadmap & Daily Path](docs/roadmap.md)
- [Exercises & Challenges](docs/exercises/README.md)

---

## 🗺️ Architecture Overview

```mermaid
graph TD
    A[Your React Code] --> B[React Runtime]
    B --> C[Fiber Reconciler]
    C --> D{Render Phase}
    D --> E[Build Fiber Tree]
    E --> F[Diff Trees]
    F --> G{Commit Phase}
    G --> H[DOM Mutations]
    G --> I[Layout Effects]
    G --> J[Passive Effects]
    H --> K[Browser]
    K --> L[Style/Layout]
    L --> M[Paint]
    M --> N[Composite]
    N --> O[Screen]

    style A fill:#61dafb,color:#000
    style B fill:#764abc,color:#fff
    style C fill:#764abc,color:#fff
    style K fill:#e8491d,color:#fff
    style O fill:#27ae60,color:#fff
```

---

## ⚡ Quick Navigation by Problem

**"My component re-renders too often"**
→ [Re-rendering System](docs/react-rendering/06-what-causes-rerenders.md) → [Memoization](docs/performance/02-memoization.md) → [Reference Equality](docs/react-rendering/07-reference-equality.md)

**"My Next.js page is slow"**
→ [Rendering Systems](docs/nextjs-rendering/01-ssr.md) → [Caching](docs/caching/01-browser-caching.md) → [Bundle Optimization](docs/performance/05-bundle-optimization.md)

**"I don't understand hydration errors"**
→ [Hydration Internals](docs/hydration/01-how-hydration-works.md) → [Hydration Mismatches](docs/hydration/02-hydration-mismatches.md) → [Hydration Debugging](docs/debugging/03-hydration-debugging.md)

**"I need to scale my frontend"**
→ [Architecture Patterns](docs/architecture/01-feature-based.md) → [State Management](docs/state-management/01-context-internals.md) → [Monorepo](docs/architecture/03-monorepo.md)

**"Preparing for a senior React interview"**
→ [Interview: React Internals](docs/interview/01-react-internals.md) → [Interview: Performance](docs/interview/02-performance.md) → [Interview: Architecture](docs/interview/03-architecture.md)

---

## 📐 How to Use This Handbook

### Path 1: The Systematic Engineer

Read every section in order. Build deep, interconnected mental models. Takes 3–6 months at a comfortable pace.

### Path 2: The Problem Solver

Jump to sections that solve your current engineering challenge. Use Quick Navigation above.

### Path 3: The Interview Candidate

Focus on Parts I–VII (React internals), Part X (Next.js rendering), Part XVI (performance). Study the Interview section.

### Path 4: The Architect

Focus on Parts IX–XIV (Next.js), Part XX (architecture), Part XVII (state management), Part XXVI (projects).

---

## 🚦 Reading Conventions

Throughout this handbook, you will see these markers:

> 🔬 **Internals** — Explains how something works inside React/Next.js source code

> ⚠️ **Anti-Pattern** — A dangerous or inefficient pattern to avoid

> ✅ **Good Practice** — A recommended, production-safe pattern

> 🧪 **Exercise** — A hands-on challenge to solidify understanding

> 📊 **Profiling** — A performance measurement example

> 🏭 **Production Note** — An insight from real production systems

> 💡 **Mental Model** — A conceptual framework to simplify understanding

---

## 🧱 Repository Structure

```
react-nextjs-engineering-handbook/
├── README.md                        ← You are here
├── CONTRIBUTING.md
├── docs/
│   ├── react-core/                  ← What React is
│   ├── react-rendering/             ← Render/commit phases, batching
│   ├── react-fiber/                 ← Fiber architecture internals
│   ├── reconciliation/              ← Diffing, keys, DOM reuse
│   ├── hooks-internals/             ← Every hook, deeply explained
│   ├── concurrent-react/            ← Transitions, Suspense, scheduling
│   ├── react-compiler/              ← Compiler & future React
│   ├── nextjs-core/                 ← Next.js internals & routing
│   ├── nextjs-rendering/            ← SSR/CSR/ISR/Streaming
│   ├── server-components/           ← RSC & Flight protocol
│   ├── hydration/                   ← Hydration internals
│   ├── caching/                     ← All 4 Next.js cache layers
│   ├── browser-internals/           ← Browser pipeline & layout
│   ├── performance/                 ← Profiling & optimization
│   ├── state-management/            ← Redux, Zustand, TanStack, etc.
│   ├── build-systems/               ← Webpack, Vite, SWC, Turbopack
│   ├── networking/                  ← HTTP, streaming, waterfalls
│   ├── architecture/                ← Feature arch, monorepo, microfrontends
│   ├── design-systems/              ← Tokens, theming, APIs
│   ├── security/                    ← XSS, CSRF, CSP
│   ├── testing/                     ← Testing philosophy & tools
│   ├── debugging/                   ← Profiler, memory, bundle
│   ├── anti-patterns/               ← What NOT to do and why
│   ├── real-world-projects/         ← Full production project breakdowns
│   ├── interview/                   ← Senior frontend interview prep
│   ├── exercises/                   ← Hands-on challenges
│   └── roadmap.md                   ← Learning progression map
├── diagrams/                        ← Mermaid & architecture diagrams
└── assets/                          ← Images, screenshots, profiling data
```

---

## 🤝 Contributing

This handbook grows through community engineering expertise.  
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

All contributions must meet the quality bar:

- Deep technical accuracy
- Step-by-step explanation style
- Internal architecture focus
- Good/bad practice examples
- Mermaid diagrams where helpful

---

## 📜 License

MIT — Free to use, share, and build upon.  
If this helped you, please ⭐ the repository.

---

## ✍️ Acknowledgments

Built for engineers by engineers.  
Inspired by the complexity of real production systems.  
Committed to the belief that **understanding internals makes better engineers**.

---

<div align="center">

**Start here →** [What React Actually Is](docs/react-core/01-what-react-is.md)

_or_

**Jump to →** [Learning Roadmap](docs/roadmap.md)

</div>
