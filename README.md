# React from Scratch to Expert — Complete Course

A self-contained, **27-lesson** course that takes you from "I've never written React" to **expert**. Every lesson is a deep, standalone document with: a from-first-principles explanation, full syntax/API reference, multiple runnable examples, an "under the hood" section, common pitfalls (wrong vs. right code), best practices, hands-on exercises **with solutions**, and a key-takeaways summary.

Targets **current React (v19)** with function components + hooks as the default, plus the modern ecosystem (Vite, React Router, TanStack Query, Redux Toolkit, Zustand, TypeScript, Tailwind, Testing Library, Next.js).

> **Total:** ~120,000 words across 27 lessons. Treat it as a textbook — read in order, type out every example, and do the exercises. Reading is not learning; **building** is.

---

## How to use this course

1. **Go in order.** Each lesson lists its prerequisites and builds on the last.
2. **Type every example by hand** into a live project (don't copy-paste). Muscle memory matters.
3. **Do the exercises before reading the solutions.** Struggle first — that's where the learning happens.
4. **Build the projects in Lesson 26** as you reach the required concepts. The capstone proves you're job-ready.
5. **Use Lesson 27** for self-assessment, interview prep, and quick reference once you're through the material.

### Set up your practice project (do this now)

You need [Node.js](https://nodejs.org) (LTS) installed. Then:

```bash
npm create vite@latest react-playground -- --template react
cd react-playground
npm install
npm run dev
```

Open the printed `localhost` URL. You now have a hot-reloading React 19 app to paste every example into. (Full walkthrough in **Lesson 02**.)

---

## Curriculum

### Part I — Foundations (the absolute basics)
| # | Lesson | What you'll learn |
|---|--------|-------------------|
| 01 | [JavaScript Foundations for React (ES6+)](01-js-foundations.md) | Exactly the JS React relies on: arrow funcs, destructuring, spread/rest, modules, array methods, optional chaining, immutability, closures, async/await — each tied to *why React needs it*. |
| 02 | [What is React & Setting Up Your Environment](02-getting-started.md) | What React is, declarative vs. imperative, the Virtual DOM, scaffolding with Vite, project structure, `createRoot`, `StrictMode`, your first component. |
| 03 | [JSX In Depth](03-jsx-in-depth.md) | What JSX really is, how it compiles, expressions in `{}`, fragments, attribute differences from HTML, the style object, and the XSS-prone escape hatches. |
| 04 | [Components & Props](04-components-and-props.md) | Function components, composition, the `children` prop, passing/destructuring props, one-way data flow, component vs. element. |
| 05 | [Conditional Rendering & Lists](05-rendering-lists-conditionals.md) | `if`/ternary/`&&`, the `0`/`&&` leak bug, rendering lists with `.map`, and **keys** + the index-key trap. |
| 06 | [Handling Events](06-handling-events.md) | `onClick`/`onChange`/`onSubmit`, the SyntheticEvent system, passing arguments, `preventDefault`/`stopPropagation`, capture vs. bubble. |

### Part II — Core React (state & hooks)
| # | Lesson | What you'll learn |
|---|--------|-------------------|
| 07 | [State & the useState Hook](07-state-usestate.md) | State vs. props, lazy init, **state-as-a-snapshot**, the functional updater, batching, immutable object/array updates, resetting state with a key. |
| 08 | [useEffect & Side Effects](08-useeffect-side-effects.md) | The dependency array, cleanup, StrictMode double-invoke, fetching with `AbortController`, the exhaustive-deps rule, and "You Might Not Need an Effect". |
| 09 | [Hooks Deep Dive](09-hooks-deep-dive.md) | The **Rules of Hooks** (and why), plus `useRef`, `useContext`, `useReducer`, `useMemo`, `useCallback`, `useLayoutEffect`, `useImperativeHandle`, `useId` — with a "which hook for which problem" guide. |
| 10 | [Custom Hooks](10-custom-hooks.md) | Reusing stateful logic. Build `useToggle`, `useLocalStorage`, `useFetch`, `useDebounce`, `useOnClickOutside`, `useInterval`, and more from scratch. |
| 11 | [Forms & Inputs](11-forms.md) | Controlled vs. uncontrolled, every input type, one handler for many fields, validation, and a peek at React 19 form Actions + React Hook Form. |
| 12 | [Context API](12-context-api.md) | Killing prop-drilling: `createContext`, Provider, `useContext`, the re-render caveat, and the `useReducer` + Context mini-store pattern. |

### Part III — Building real applications
| # | Lesson | What you'll learn |
|---|--------|-------------------|
| 13 | [Performance Optimization](13-performance.md) | The re-render model, `React.memo`, correct `useMemo`/`useCallback` (and when *not* to), code-splitting with `lazy`/`Suspense`, virtualization, the Profiler, and the React Compiler. |
| 14 | [Client-Side Routing with React Router](14-routing.md) | `BrowserRouter`, nested routes + `Outlet`, dynamic params, `useNavigate`, search params, data-router loaders/actions, lazy routes, and protected routes. |
| 15 | [Data Fetching & Async UI](15-data-fetching.md) | Fetching properly (loading/error/abort/races), then **TanStack Query** (caching, mutations, invalidation), SWR, and Suspense + the `use()` hook. |
| 16 | [Global State Management](16-state-management.md) | Categorizing state, lifting up, then **Redux Toolkit** and **Zustand** side-by-side, plus a decision matrix for choosing a tool. |
| 17 | [Styling React Apps](17-styling.md) | Plain CSS, CSS Modules, Sass, CSS-in-JS (styled-components/Emotion), **Tailwind**, `clsx`, theming, and component libraries (MUI, shadcn/ui). |
| 18 | [TypeScript with React](18-typescript-react.md) | Typing props, children, events, and every hook; generic components, typed context, utility types, and the `React.FC` debate. |
| 19 | [Testing React Applications](19-testing.md) | Testing-Library philosophy, Vitest/Jest setup, queries + `user-event`, async tests, mocking with MSW, testing hooks, and E2E with Playwright. |

### Part IV — Advanced & modern React
| # | Lesson | What you'll learn |
|---|--------|-------------------|
| 20 | [Advanced Component Patterns](20-advanced-patterns.md) | Error boundaries, portals, refs (`forwardRef` + React 19 ref-as-prop), HOCs, render props, compound components, polymorphic `as` props. |
| 21 | [Class Components & Lifecycle](21-class-components-legacy.md) | Legacy class syntax, `this.setState`, the full lifecycle, the binding problem, each method mapped to its hook equivalent, and a migration walkthrough. *(For reading old code + interviews.)* |
| 22 | [Modern React (Concurrent, Suspense, RSC, React 19 APIs)](22-modern-react-19.md) | Concurrent rendering, `useTransition`/`useDeferredValue`, Suspense, `useId`/`useSyncExternalStore`, the `use()` API, Server Components, Actions, `useOptimistic`, and the React Compiler. |
| 23 | [Going Full-Stack: Next.js](23-nextjs-fullstack.md) | The App Router, Server vs. Client Components, server data fetching, route handlers, Server Actions, SSR/SSG/ISR/streaming, metadata, and deployment. |
| 24 | [Tooling, Build & the React Ecosystem](24-tooling-ecosystem.md) | Package managers, Vite internals, env vars, ESLint + Prettier, path aliases, React DevTools, bundle analysis, deployment, and must-know libraries. |
| 25 | [Best Practices, Architecture & Accessibility](25-best-practices-architecture.md) | Folder structure, component design, when (not) to abstract, error strategy, **a11y**, **security** (XSS), and a pragmatic performance checklist. |

### Part V — Mastery
| # | Lesson | What you'll learn |
|---|--------|-------------------|
| 26 | [Hands-On Projects & a 12-Week Study Plan](26-projects-and-exercises.md) | Seven graded build-it-yourself projects (counter → full capstone with auth, data, TS, and tests) plus a week-by-week schedule mapping all 27 lessons. |
| 27 | [Expert Checklist, Interview Q&A, Glossary & Cheat Sheets](27-expert-checklist-glossary.md) | A self-assessment mastery checklist, a thorough interview Q&A bank, a full glossary, and quick-reference cheat sheets. |

---

## Suggested pace

- **Fast track (motivated, full-time): ~6 weeks.** ~4–5 lessons/week + projects in parallel.
- **Steady (alongside a job): 12 weeks.** Follow the schedule in [Lesson 26](26-projects-and-exercises.md).
- **Don't rush Parts I–II.** Lessons 07–09 (state + hooks) are where most beginners form bad habits. Master them before moving on.

## What "expert" means here

By the end you will be able to:

- Build, structure, and ship a non-trivial React app end-to-end.
- Explain *why* React re-renders, what the Virtual DOM does, and how reconciliation + keys work.
- Reach for the right hook, the right state tool, and the right data-fetching strategy — and justify each choice.
- Write performant, accessible, well-tested, type-safe components.
- Use React 19's concurrent features, Suspense, and Server Components, and know what a framework (Next.js) adds.
- Read and maintain legacy class-component code.
- Pass a senior React interview (see [Lesson 27](27-expert-checklist-glossary.md)).

---

## Official references

- **React docs:** https://react.dev — the canonical source; read it alongside this course.
- **React blog (release notes):** https://react.dev/blog
- **Vite:** https://vite.dev
- **TypeScript + React cheatsheet:** https://react-typescript-cheatsheet.netlify.app

Happy building. Start with **[Lesson 01 →](01-js-foundations.md)**
