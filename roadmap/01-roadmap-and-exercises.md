# Part XXVII · Roadmap & Exercises

> **This is the final document in the React + Next.js Engineering Handbook. It answers three questions: which order to study the handbook's parts for maximum conceptual coherence, what exercises synthesize concepts across multiple parts, and what to build next to continue growing beyond the handbook's scope. If you've read every document in order, you now have the mental models of a senior React/Next.js engineer — this document is about turning that knowledge into practiced, embodied skill.**

---

## Learning Roadmaps by Starting Point

Not everyone begins at the same level. Use the roadmap that matches your current baseline.

### Roadmap A: Junior → Mid-Level (0–2 years React experience)

```
FOUNDATION TRACK (4–6 weeks):
  Week 1: Part I (React Core) + Part II (React Rendering)
    → Build: a custom useState clone from scratch (no React) to understand the model
  Week 2: Part V (Hooks Internals) + Part IX (Next.js Core)
    → Build: a data-fetching hook that handles loading/error/success states
  Week 3: Part XVI (State Management) — focus on Context and useState patterns
    → Build: a cart system using Context + useReducer, then refactor to Zustand
  Week 4: Part XXII (Testing) — unit testing first
    → Write tests for everything you've built in weeks 1–3
  Week 5: Part XV (Performance) — React Profiler and memoization
    → Profile your cart application, fix the unnecessary re-renders
  Week 6: Real-World Project P1 (E-Commerce)
    → Build the full checkout flow from ADR-1 through the testing checklist

AFTER FOUNDATION:
  Part XI (Next.js Rendering) → Part XII (Caching) → Part XIX (Architecture)
  → Project P3 (SaaS) to apply architecture patterns
```

### Roadmap B: Mid-Level → Senior (2–5 years React experience)

```
DEPTH TRACK (6–8 weeks):
  Week 1: Part III (Fiber Architecture) + Part VI (Concurrent React)
    → The mental model shift: understand WHY React works the way it does
  Week 2: Part X (Server Components) + Part XIII (Next.js Advanced Internals)
    → Build: a page that uses RSC streaming + Suspense + error.tsx boundaries
  Week 3: Part XII (Caching) — the four cache layers deeply
    → Experiment: build a page, observe each cache layer in DevTools
  Week 4: Part XXIII (Debugging) + Part XXIV (Anti-Patterns)
    → Audit: take an existing codebase and identify every anti-pattern
  Week 5: Part XXI (Security) — all four documents
    → Audit: apply the security checklist from P1 to your own applications
  Week 6: Part XXII (Testing) — integration + E2E
    → Write a Playwright test for the most critical user flow in your work
  Weeks 7–8: Real-World Projects P2 + P4 (Dashboard + Chat)
    → These cover the hardest architectural challenges at mid-level

AFTER DEPTH:
  Part XXV (Interview Prep) → practice the internals questions with a peer
  Part XXVII (this document) → work through the synthesis exercises
```

### Roadmap C: Senior → Staff (5+ years, aiming for Staff/Principal)

```
BREADTH + JUDGMENT TRACK (ongoing):
  Priority 1: Part XIX (Architecture Patterns) — deeply
    → For each pattern, think of a real system you've built and map it
  Priority 2: Part XXV (System Design) — do the worked examples
    → Practice explaining design decisions out loud, not just in writing
  Priority 3: Part XXVI (Real-World Projects) — all 6
    → Focus on the ADRs: would you have made the same decisions?
  Priority 4: Part XXVII Synthesis Exercises — the multi-part ones
    → These are the closest to real engineering judgment problems

STAFF-LEVEL FOCUS:
  "What would I change if this system needed to support 100x more users?"
  "What are the security holes in this architecture?"
  "How would I explain this caching decision to a new team member?"
  These meta-questions are the staff-level lens.
```

---

## Synthesis Exercises

These exercises deliberately cross multiple handbook sections — they can't be solved by remembering one concept.

---

### Synthesis Exercise 1: The Performance Audit

**Skills tested:** Parts I, III, XV, XXIII, XXIV

**Setup:** Take any moderately complex React application you have access to (your own project, an open-source app you've cloned, or a new Next.js app with synthetic complexity).

**Tasks:**

1. **Baseline measurement.** Run Lighthouse in Chrome DevTools (4x CPU throttling, Slow 4G). Record: LCP, INP, CLS, Total Blocking Time, Total JavaScript size. Screenshot the results.

2. **Bundle analysis.** Run `ANALYZE=true next build`. Identify the three largest chunks and the three largest individual modules. For each, ask: Is this necessary for the initial page load? Could it be lazy-loaded?

3. **Profiler audit.** Open React DevTools Profiler. Record the 10 most common user interactions. For each: how many components re-rendered? Which ones are unnecessary (re-rendered despite unchanged props/state)? Document your findings.

4. **Anti-pattern scan.** Apply the detection checklists from Parts XXIV (all three documents). How many instances of each anti-pattern do you find? Tally them.

5. **Fix one thing.** Choose the highest-impact issue from steps 2–4. Implement the fix. Re-measure from step 1. Did the metric improve? By how much?

6. **Write an ADR.** Document your fix as an Architecture Decision Record: what you found, what alternatives you considered, what you implemented, and what the measured result was.

---

### Synthesis Exercise 2: The Security Audit

**Skills tested:** Parts XXI, IX, X, XXII

**Setup:** Use the e-commerce application from P1 or any Next.js app with authentication and data mutations.

**Tasks:**

1. **CSP audit.** Does the application have a Content Security Policy? If yes, evaluate it against the checklist in doc 104. If no, implement one (using the middleware nonce approach from doc 104). Verify with SecurityHeaders.com.

2. **Server Action audit.** Find every Server Action in the codebase. For each:
   - Does it check authentication before any DB operation?
   - Does it validate its input with Zod or similar?
   - Does it check authorization (not just authentication — is this user allowed to do THIS operation)?
   - Does it rely on any client-supplied values for pricing, permissions, or IDs? (If so, fix it.)

3. **Dependency audit.** Run `npm audit`. Run the Socket.dev scanner on the project. Document every high/critical vulnerability found and whether it's exploitable in your context.

4. **Header verification.** Use the security header configuration from doc 107. Apply it to your application. Verify each header is present with correct values in the browser Network tab.

5. **Write the security checklist.** Using the checklists from P1 as a template, write a custom security checklist for your specific application's threat model.

---

### Synthesis Exercise 3: The Architecture Review

**Skills tested:** Parts XIX, XVI, XI, XII, XXV (System Design)

**Scenario:** Your team has been asked to build a "collaborative design tool" — similar to Figma but for a specific domain. The product manager has given you the following requirements:

- Users can create and edit "designs" (complex JSON documents)
- Multiple users can edit the same design simultaneously
- Changes are reflected in real-time for all collaborators
- Designs can be shared via a public link (view-only)
- Free tier: 3 designs; Pro tier: unlimited designs
- The team is 4 engineers, deploying on Vercel

**Tasks:**

1. **Apply the System Design Framework.** Work through all 10 design dimensions (from doc 121: Component Architecture, State Management, Rendering Strategy, Data Fetching, Performance, Real-Time, Accessibility, Error Handling, Security, Monitoring). Write 3–5 sentences for each.

2. **Identify the Hard Problems.** What are the three genuinely difficult technical problems in this system? (Hint: real-time conflict resolution is one — see P4.) For each hard problem, propose a solution and explain why.

3. **Draw the ADR for the rendering strategy.** Should the editor be CSR? SSR? A hybrid? What content can be statically generated? Write this as an ADR with rationale and alternatives considered.

4. **Design the state structure.** The design document (JSON) is the central data structure. Where does it live? (Client state? Server state? Both?) How do real-time changes flow through the state? What happens on network disconnect?

5. **Identify the multi-tenancy requirements.** The free/pro tier split is a multi-tenancy concern (from P3). How would you implement feature gating for the "3 designs" limit? Describe all three layers of enforcement.

6. **Present it.** Explain your design to a colleague in 15 minutes. Time yourself. Adjust where you run long or get stuck.

---

### Synthesis Exercise 4: The Debugging Sprint

**Skills tested:** Parts XXIII, XXII, XV

**Setup:** Intentionally introduce bugs into a test application. (Alternatively, find bugs in an existing codebase.)

**Bugs to introduce (or find):**

```tsx
// Bug 1: Stale closure in useEffect (from doc 112)
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const interval = setInterval(() => {
      console.log(count); // always logs 0
      setCount(count + 1); // always sets to 1
    }, 1000);
    return () => clearInterval(interval);
  }, []);
  return <div>{count}</div>;
}

// Bug 2: Missing error boundary (from doc 114)
// A component that sometimes throws during render with no boundary

// Bug 3: Memory leak (from doc 115)
// An event listener added in useEffect without cleanup

// Bug 4: Hydration mismatch (from doc 113)
// A component that renders differently on server vs client

// Bug 5: N+1 query (from any data-fetching code)
// A list that fetches each item individually instead of batch-fetching
```

**Tasks:**

1. For each bug: form a hypothesis BEFORE looking at the code. Write it down.
2. Use the appropriate debugging tool (React DevTools, Node.js Inspector, heap snapshot, Profiler) to confirm or refute the hypothesis.
3. Fix the bug.
4. Write a test that would have caught this bug BEFORE it was introduced.

---

### Synthesis Exercise 5: The Interview Simulation

**Skills tested:** Part XXV (all four documents)

**Format:** Find a colleague or friend who has read the handbook. Take turns interviewing each other.

**Interviewer instructions:** Draw one question from each category below. Do not tell the interviewee which question you're asking until they've answered it.

**Interviewee instructions:** Answer each question as if in a real interview. Use the STAR + Reflection + Influence framework for behavioral questions. Use the system design framework for design questions.

```
INTERNALS (choose one):
  - Explain how React's key prop works. What happens when a key changes?
  - How does useEffect know when to re-run? What is Object.is()?
  - What is React Fiber and why was it introduced?
  - Explain the difference between the render phase and commit phase.

NEXT.JS (choose one):
  - What triggers a page to become dynamic in Next.js App Router?
  - Describe Next.js's four caching layers and how they interact.
  - How does streaming SSR work? What does the HTML look like?
  - What can Server Actions do that Route Handlers cannot?

SYSTEM DESIGN (choose one):
  - Design a product search feature for an e-commerce platform with 10M products.
  - Design a real-time collaborative whiteboard.
  - Design a notification system that supports push, email, and in-app.

BEHAVIORAL (choose one):
  - Tell me about a time you pushed back on a product requirement.
  - Describe a technical decision that turned out to be wrong.
  - How do you approach mentoring less experienced engineers?
```

**Debrief:** After each answer, the interviewer gives feedback: What strong signals did they hear? What was missing? What would have made the answer stronger?

---

## What to Build Next

The handbook ends here, but engineering growth doesn't. The highest-leverage activities after completing this handbook:

### 1. Read the Source Code

Nothing develops internals understanding like reading the actual React and Next.js source:

- **React reconciler:** `packages/react-reconciler/src/ReactFiberWorkLoop.js` — the work loop itself
- **React hooks:** `packages/react-reconciler/src/ReactFiberHooks.js` — how useState stores state in the fiber
- **Next.js App Router:** `packages/next/src/server/app-render/` — how RSC rendering and streaming work

Set a goal: read 50 lines of React source per week. Don't try to understand everything. Focus on the parts that connect to concepts from this handbook.

### 2. Contribute to Open Source

The fastest way to develop judgment is to have your code reviewed by engineers who work on frameworks you depend on. Options:

- React: good first issues labeled in the GitHub repo
- Next.js: bug fixes, documentation improvements
- Testing Library: adding test cases, improving query suggestions
- TanStack Query: edge case handling, TypeScript improvements

### 3. Build Something Unreasonably Ambitious

Pick one idea that seems too hard for your current level. Build it anyway. The P1–P6 projects in this handbook are designed to be challenging — but real engineering growth comes from the projects you design yourself, encounter the problems you didn't anticipate, and solve them with the tools and mental models from this handbook.

Ideas to consider:

- A browser-based collaborative code editor (Monaco + CRDTs)
- A full design system with Storybook + automated visual regression
- A Next.js application that renders a network topology of 100K+ nodes (using the canvas approach from your own work)
- A real-time multiplayer game built entirely in Next.js App Router

### 4. Teach What You Know

The best way to discover gaps in your understanding is to explain it to someone else. Options:

- Write blog posts on concepts from this handbook that you found hardest
- Give a talk at a local meetup on React Fiber or Next.js caching
- Mentor a junior engineer through one of the real-world projects
- Start a study group that works through the synthesis exercises together

### 5. Stay Current

The React/Next.js ecosystem moves fast. The concepts in this handbook age well (Fiber, reconciliation, the rendering model), but specific APIs change. Stay current by:

- Following the React team's blog (`react.dev/blog`)
- Following the Next.js changelog (`nextjs.org/blog`)
- Following Vercel's Engineering blog
- Subscribing to: Josh Comeau's newsletter, TkDodo's React Query blog, Lee Robinson's writing
- Watching the React and Next.js GitHub repositories' Discussion section

---

## The Complete Handbook at a Glance

```
PART I      React Core                        (01–09)
PART II     React Rendering                   (10–16)
PART III    React Fiber Architecture          (17–22)
PART IV     Reconciliation Deep Dive          (23–27)
PART V      Hooks Internals                   (28–36)
PART VI     Concurrent React                  (37–44)
PART VII    React Compiler                    (45–50)
PART VIII   Advanced React Patterns           (51–59)

PART IX     Next.js Core                      (60–70)
PART X      React Server Components           (71–75)
PART XI     Next.js Rendering Systems         (76–82)
PART XII    Next.js Caching Systems           (83–89)
PART XIII   Next.js Advanced Internals        (90–92)
PART XIV    Browser Rendering Internals       (93)
PART XV     Performance Engineering           (94–100)
PART XVI    State Management Deep Dive        (101–106)
PART XVII   Build Systems                     (107–112)
PART XVIII  Networking Engineering            (113–117)
PART XIX    Architecture Patterns             (118–122)
PART XX     Design Systems                    (123–125)
PART XXI    Security                          (104–107)
PART XXII   Testing                           (108–111)
PART XXIII  Debugging                         (112–115)
PART XXIV   Anti-Patterns                     (116–118)
PART XXV    Interview Prep                    (119–122)
PART XXVI   Real-World Projects               (P1–P6)
PART XXVII  Roadmap & Exercises               (this document)

TOTAL: 127 documents across 27 parts
```

---

## Final Note

Engineering excellence is not a destination — it's a practice. The engineers who stand out at senior and staff levels aren't those who know the most facts about React internals; they're those who have developed the **judgment** to apply the right tool to the right problem, the **communication skills** to explain their reasoning to others, and the **humility** to keep learning as the ecosystem evolves.

Every concept in this handbook was written to develop judgment, not to catalog facts. The mental models — rendering as a pure function of state, the server/client boundary as a module boundary, caching as four independent layers, performance debugging as hypothesis-driven investigation — are the durable assets. The specific APIs will change. The mental models will not.

Build something. Break it. Debug it. Understand it. Repeat.

---

_This concludes the React + Next.js Engineering Handbook._

_Repository: [github.com/your-org/react-nextjs-engineering-handbook](https://github.com)_
_License: MIT_
_Contributions welcome — see CONTRIBUTING.md_
