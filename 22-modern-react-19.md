# 22 — Modern React: Concurrent Features, Suspense, Server Components & React 19 APIs

> **What you'll learn:** how React's concurrent renderer makes UIs stay responsive under load, and the full toolbox of React 18/19 primitives — Transitions, `useDeferredValue`, Suspense, the `use()` API, Server Components, Actions, and the React Compiler — including precisely which pieces are stable in plain React versus which need a framework.

By the end of this lesson you will be able to:

- Explain what "interruptible / concurrent rendering" is and why automatic batching matters.
- Mark non-urgent updates with `useTransition` and standalone `startTransition`, and defer expensive derived values with `useDeferredValue`.
- Build Suspense boundaries that handle lazy components and async data, and nest them for staged loading.
- Use `useId` and `useSyncExternalStore` correctly for hydration-safe IDs and external-store subscriptions.
- Read promises and context with the new `use()` API.
- Describe the Server Component / Client Component split, the `"use client"` boundary, and what serializes across it.
- Wire up Actions: `<form action>`, `useActionState`, `useFormStatus`, and `useOptimistic`.
- Use React 19 ergonomics — `ref` as a prop, `<Context>` as provider, document metadata hoisting, preloading APIs — and understand what the React Compiler automates.

---

## Prerequisites

Before this lesson you should be comfortable with:

- Function components, JSX, and props (lessons 02–04).
- `useState` and the re-render model (see `07-state-usestate.md`).
- `useEffect` and the difference between rendering and side effects (see `08-useeffect-side-effects.md`).
- `useMemo` / `useCallback` and referential identity (see `09-hooks-deep-dive.md`).
- `useRef` and DOM refs (see `09-hooks-deep-dive.md`).
- The Context API (see `12-context-api.md`).
- `React.lazy` and code splitting basics (see `13-performance.md`).

This lesson assumes **React 19** with the function-component + hooks model. We use Vite for the client-only examples and note explicitly where a framework (Next.js, React Router framework mode) is required.

---

## Concept

For most of React's history, rendering was **synchronous and blocking**. When state changed, React walked the component tree, computed the new UI, and committed it to the DOM in one uninterruptible burst. If that burst took 80 ms, the main thread was frozen for 80 ms — keystrokes, clicks, and animations all waited. The renderer had exactly one priority: "do it all, now."

**Concurrent rendering** (the foundation laid in React 18 and built upon in 19) breaks that single priority into many. The key idea is **interruptible rendering**: React can start rendering a tree, pause partway through to handle something more urgent (like a keystroke), throw away the half-finished work, and restart later — all without ever showing the user a torn, half-updated screen. Think of a chef preparing a slow braise: when a customer walks up to the counter with a quick order, the chef can set the braise aside, serve the quick order, and come back to the braise. The braise is *non-urgent*; the counter order is *urgent*.

This is not multithreading — JavaScript is still single-threaded. React achieves it by rendering in small time-sliced chunks and yielding to the browser between chunks. Crucially, **rendering became a thing you can label by priority**. The whole modern feature set flows from this:

- **Transitions** let you tell React "this state update is non-urgent — keep the UI interactive while you work on it."
- **`useDeferredValue`** lets you render an expensive view from a *lagging* copy of a value, so typing stays snappy.
- **Suspense** lets a component *suspend* (declaratively say "I'm not ready, show a fallback") while data or code loads, and lets React coordinate that with transitions so the screen doesn't flash.

Layered on top, React 19 adds **Server Components** (components that run only on the server and never ship JS to the browser) and **Actions** (a first-class way to handle async mutations and form submissions with built-in pending/error/optimistic state). And the **React Compiler** automates the memoization you used to write by hand.

A recurring theme: **what is "stable React" versus "framework-dependent."** Concurrent rendering, Transitions, `useDeferredValue`, Suspense for lazy components, `useId`, `useSyncExternalStore`, `use()` for promises/context, and all the React 19 ergonomics work in a plain Vite app. **Server Components and Server Actions are a protocol** that requires a framework or custom bundler integration (Next.js App Router, React Router/Remix framework mode, or a Parcel/Vite RSC setup) — you cannot just `import` your way into them in a vanilla Vite SPA.

---

## Syntax & API

Annotated reference of the APIs covered in this lesson.

```js
// Concurrent entry point (React 18+). createRoot enables concurrent features.
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")).render(<App />);
```

```js
// Transitions
const [isPending, startTransition] = useTransition();
// startTransition(scope): updates inside scope are non-urgent.
startTransition(() => { setTab("posts"); });

// Standalone (no pending flag) — usable outside components too.
import { startTransition } from "react";

// React 19: actions are allowed inside transitions (async functions).
startTransition(async () => { await save(); setSaved(true); });
```

```js
// Deferred value: returns a possibly-stale copy of `value` during urgent updates.
const deferred = useDeferredValue(value /*, initialValue? */);
```

```jsx
// Suspense
import { Suspense, lazy } from "react";
const Profile = lazy(() => import("./Profile.jsx"));
<Suspense fallback={<Spinner />}>
  <Profile />
</Suspense>
```

```js
// Stable IDs (hydration-safe, unique per component instance)
const id = useId(); // e.g. ":r3:" — use as a prefix, append your own suffixes

// External store subscription (hydration- and concurrency-safe)
const snapshot = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);
//   subscribe(callback) => unsubscribe
//   getSnapshot() => current value (must be referentially stable when unchanged)
//   getServerSnapshot() => value used during SSR/hydration
```

```js
// The use() API (React 19) — reads a resource during render.
import { use } from "react";
const value = use(promise);  // suspends until the promise resolves
const theme = use(ThemeContext); // like useContext, but callable conditionally
```

```jsx
// React 19 ergonomics
function Input({ ref, ...props }) { return <input ref={ref} {...props} />; } // ref is a prop now
<ThemeContext value={dark}>...</ThemeContext>  // <Context> is itself the provider
<title>Page title</title>                       // metadata hoists to <head>

// Actions / forms (React 19)
const [state, formAction, isPending] = useActionState(actionFn, initialState);
const { pending, data, method, action } = useFormStatus();   // must be in a child of the <form>
const [optimistic, addOptimistic] = useOptimistic(actual, (state, next) => merged);

// Preloading (react-dom)
import { preload, preinit, prefetchDNS, preconnect } from "react-dom";
```

```js
// RSC directives (framework-dependent — at the top of a module)
"use client"; // marks the boundary: this module + its imports run on the client
"use server"; // marks a Server Action (exported async functions become callable from the client)
```

---

## Examples

### Example 1 — Transitions keep a slow tab switch responsive

A classic problem: clicking a tab triggers a heavy re-render that blocks the click feedback. Wrapping the state update in a transition tells React the new tab is non-urgent, so the button stays interactive and we can show a pending indicator.

```jsx
import { useState, useTransition } from "react";

function SlowList({ count }) {
  // Deliberately expensive render to simulate a heavy view.
  const items = [];
  for (let i = 0; i < count; i++) {
    items.push(<li key={i}>Item #{i}</li>);
  }
  return <ul>{items}</ul>;
}

export default function Tabs() {
  const [tab, setTab] = useState("home");
  const [isPending, startTransition] = useTransition();

  function selectTab(next) {
    // The urgent part (highlighting the clicked tab) is NOT in the transition;
    // the expensive content swap IS, so it can be interrupted.
    startTransition(() => setTab(next));
  }

  return (
    <div style={{ opacity: isPending ? 0.6 : 1 }}>
      <nav>
        <button onClick={() => selectTab("home")}>Home</button>
        <button onClick={() => selectTab("slow")}>Slow</button>
      </nav>
      {isPending && <p>Loading…</p>}
      {tab === "home" ? <p>Welcome home.</p> : <SlowList count={30000} />}
    </div>
  );
}
```

`isPending` is `true` while React renders the new tab in the background; the old UI stays on screen and clickable. Without the transition, the whole app would freeze until `SlowList` finished.

### Example 2 — `useDeferredValue` for a responsive search box

Here the *input* must update on every keystroke, but the *expensive filtered list* can lag behind. `useDeferredValue` gives you a second copy of the query that updates at low priority.

```jsx
import { useState, useDeferredValue, useMemo } from "react";

function Results({ query }) {
  const list = useMemo(() => {
    // Pretend this filters 50k records — expensive.
    return Array.from({ length: 50000 }, (_, i) => `Row ${i}`)
      .filter((row) => row.includes(query));
  }, [query]);

  return <p>{list.length} matches</p>;
}

export default function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <>
      {/* Updates immediately on every keystroke. */}
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {/* Renders from the lagging value, so typing never stutters. */}
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        <Results query={deferredQuery} />
      </div>
    </>
  );
}
```

The input never feels laggy because its own state is urgent; `Results` re-renders from `deferredQuery`, which React updates only when it has spare time.

### Example 3 — Suspense for data with `use()`

`use(promise)` suspends the component until the promise resolves; the nearest `<Suspense>` shows its fallback meanwhile. **Cache or hoist the promise** so it isn't recreated every render (in real apps a framework or a caching layer provides this).

```jsx
import { Suspense, use } from "react";

// Created ONCE at module scope so the same promise instance is reused.
const userPromise = fetch("https://api.example.com/user/1").then((r) => r.json());

function UserCard({ promise }) {
  const user = use(promise); // suspends until resolved
  return <h2>{user.name}</h2>;
}

export default function App() {
  return (
    <Suspense fallback={<p>Loading user…</p>}>
      <UserCard promise={userPromise} />
    </Suspense>
  );
}
```

Unlike hooks, `use()` may be called conditionally or inside loops — but the resource it reads should still be stable across renders.

### Example 4 — Nested Suspense boundaries for staged loading

Boundaries can nest so independent regions reveal as their own data arrives, instead of waiting for everything.

```jsx
import { Suspense } from "react";

export default function Dashboard() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Header />
      <Suspense fallback={<p>Loading chart…</p>}>
        <RevenueChart />
      </Suspense>
      <Suspense fallback={<p>Loading feed…</p>}>
        <ActivityFeed />
      </Suspense>
    </Suspense>
  );
}
```

The header appears as soon as it's ready; the chart and feed each pop in independently. If you wanted strict top-to-bottom reveal order, that's what `<SuspenseList>` was designed for — **but `SuspenseList` is still experimental and not in a stable release, so don't rely on it in production.**

### Example 5 — A form Action with `useActionState`, `useFormStatus`, and `useOptimistic`

React 19's Actions handle the pending/error/optimistic plumbing for you. This example runs in plain React 19 (the action is a regular client async function — no server needed).

```jsx
import { useActionState, useOptimistic } from "react";
import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus(); // reads the enclosing <form>'s state
  return <button disabled={pending}>{pending ? "Adding…" : "Add"}</button>;
}

export default function TodoForm() {
  // useActionState returns [state, formAction, isPending].
  // The second element is the action to pass straight to <form action>.
  const [todos, formAction] = useActionState(
    async (prev, formData) => {
      const text = formData.get("text");
      addOptimistic(text);                                // show immediately
      await new Promise((res) => setTimeout(res, 600));   // simulate network
      return [...prev, { id: Date.now(), text }];
    },
    [] // initial state
  );

  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newText) => [...state, { id: "temp", text: newText, pending: true }]
  );

  return (
    <form action={formAction}>
      <input name="text" required />
      <SubmitButton />
      <ul>
        {optimisticTodos.map((t) => (
          <li key={t.id} style={{ opacity: t.pending ? 0.5 : 1 }}>{t.text}</li>
        ))}
      </ul>
    </form>
  );
}
```

> Note: `addOptimistic` is called *inside* the action passed to `useActionState`, so the optimistic entry appears within the action's transition and is reconciled away once the action returns the real state. The `pending`, the resulting state, and the optimistic preview all come from React hooks — no manual `isLoading` booleans.

### Example 6 — Server / Client Component split (framework-dependent)

This only runs under a framework with RSC support (e.g. Next.js App Router). The default is a **Server Component**; `"use client"` opts a module into the client.

```jsx
// app/page.jsx — a Server Component (no directive = server by default in RSC frameworks)
import { db } from "./db";          // server-only code, never shipped to the browser
import LikeButton from "./LikeButton.jsx";

export default async function Page() {
  const post = await db.posts.find(1); // runs on the server during render
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <LikeButton postId={post.id} initialLikes={post.likes} />
    </article>
  );
}
```

```jsx
// LikeButton.jsx — a Client Component
"use client";
import { useState } from "react";

export default function LikeButton({ postId, initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);
  return <button onClick={() => setLikes((n) => n + 1)}>♥ {likes}</button>;
}
```

The Server Component fetches data directly (no API route, no `useEffect`), renders to a special serialized format, and ships **zero JavaScript** for itself. Only `LikeButton` — across the `"use client"` boundary — hydrates in the browser. Props passed from server to client (`postId`, `initialLikes`) must be **serializable**. Plain objects, arrays, strings, numbers, `null`, and many built-ins (`Date`, `Map`, `Set`, `BigInt`, typed arrays) cross fine; ordinary functions, class instances, and symbols do not — the only functions that can cross are Server Actions (which serialize as a reference the client calls back).

### Example 7 — React 19 ergonomics in one component

```jsx
import { createContext, use } from "react";

const ThemeContext = createContext("light");

// ref is just a prop now — no forwardRef wrapper needed.
function TextInput({ ref, label }) {
  const theme = use(ThemeContext); // use() can read context conditionally
  return (
    <label>
      {label}
      {/* <title>/<meta> placed anywhere hoist into <head> automatically */}
      <title>{`Editing: ${label}`}</title>
      <input ref={ref} className={theme} />
    </label>
  );
}

export default function Form() {
  return (
    // <Context> is the provider directly — no <ThemeContext.Provider>.
    <ThemeContext value="dark">
      <TextInput label="Name" />
    </ThemeContext>
  );
}
```

---

## Under the hood

**Fiber and lanes.** React's reconciler ("Fiber") represents work as a linked tree of fiber nodes. Each update is tagged with a **lane** — a bitmask priority. Urgent updates (clicks, typing, `setState` from a discrete event) get high-priority lanes; transition updates get a low-priority "transition lane." When React schedules work, it processes higher-priority lanes first and can **suspend or discard** in-progress low-priority work to service a new urgent update. This is the machinery behind "interruptible rendering": the renderer checks a deadline (via the scheduler) and yields to the browser, then resumes.

**Automatic batching.** In React 17, multiple `setState` calls were only batched inside React event handlers; updates inside `setTimeout`, promises, or native event handlers each triggered a separate render. React 18's `createRoot` made batching **automatic everywhere** — all synchronous state updates in the same tick are coalesced into one render regardless of origin. (Need to opt out for a specific update? `flushSync` from `react-dom` forces a synchronous flush.)

**Why concurrent features need `createRoot`.** The legacy `ReactDOM.render` runs in "legacy mode" with synchronous rendering and no concurrent features. `createRoot` enables the concurrent renderer; Vite's default React template already uses it.

**How Suspense actually works.** When a component calls `use(promise)` (or a framework data hook) and the data isn't ready, React **throws the promise** (conceptually — it's a control-flow signal, not an error). React catches it at the nearest `<Suspense>`, renders the fallback, and attaches a `.then` to re-attempt the subtree when the promise settles. This is why the promise must be stable: a fresh promise every render means React re-suspends forever. During a **transition**, React keeps the old content visible instead of showing the fallback (avoiding a jarring spinner flash) — Suspense and Transitions are designed to cooperate.

**`useDeferredValue` vs `useTransition`.** They solve the same class of problem from opposite ends. `useTransition` wraps the *source* of the update (the setter); `useDeferredValue` wraps the *consumed value* downstream. Use a transition when you control the state update; use `useDeferredValue` when the value comes from props you don't control. Internally, `useDeferredValue` schedules a low-priority re-render with the new value while returning the previous value to the urgent render.

**`useSyncExternalStore` and tearing.** Concurrent rendering can render the same component multiple times for one commit. If an external store mutates mid-render, different parts of the tree could read different values — **tearing**. `useSyncExternalStore` subscribes through React so the renderer always reads a consistent snapshot and forces a synchronous re-render on store changes, preventing tearing. It also takes a `getServerSnapshot` so SSR and hydration agree. This is why state libraries (Redux, Zustand) use it internally.

**`useId`** generates IDs derived from the component's position in the tree, identical on server and client — so SSR markup and hydration match (random IDs would mismatch). Always use it as a *prefix* and append static suffixes for multiple elements.

**RSC serialization.** Server Components render to a streaming, JSON-like payload (the "RSC payload") describing the UI tree, with references to Client Components by module ID. The browser's React runtime stitches this together and hydrates only the client islands. Because the server result is serialized, anything crossing the boundary as a prop must be serializable; functions can only cross via Server Actions (which serialize as a reference the client can call back).

**The React Compiler.** Historically you hand-wrote `useMemo`/`useCallback`/`React.memo` to avoid wasteful re-renders. The React Compiler is a build-time tool that analyzes your components and **automatically inserts memoization**, so equivalent computations and JSX are reused across renders without manual hints. It only transforms code that follows the Rules of React (pure render, no mutation of props/state). As of React 19 it's **opt-in via a Babel/Vite plugin and the `react-compiler-runtime`**, increasingly recommended but still rolling toward "on by default." When enabled, you write straightforward components and largely stop reaching for manual memoization (see `09-hooks-deep-dive.md` for the manual approach it replaces).

---

## Common mistakes & pitfalls

**1. Creating the promise during render with `use()`.**

```jsx
// ❌ WRONG — new promise every render → Suspense never resolves
function UserCard({ id }) {
  const user = use(fetch(`/api/user/${id}`).then((r) => r.json()));
  return <h2>{user.name}</h2>;
}
```

```jsx
// ✅ RIGHT — promise comes from a stable source (cache / parent / framework loader)
const cache = new Map();
function getUser(id) {
  if (!cache.has(id)) cache.set(id, fetch(`/api/user/${id}`).then((r) => r.json()));
  return cache.get(id);
}
function UserCard({ id }) {
  const user = use(getUser(id)); // stable promise per id
  return <h2>{user.name}</h2>;
}
```

A new promise each render re-suspends infinitely. Cache it or receive it as a prop.

**2. Putting urgent updates inside a transition.**

```jsx
// ❌ WRONG — the input lags because its own value is deprioritized
startTransition(() => setText(e.target.value));
```

```jsx
// ✅ RIGHT — keep the input urgent; defer only the expensive consumer
setText(e.target.value);
const deferredText = useDeferredValue(text);
```

Text inputs must update urgently or they feel broken. Transition the *result*, not the keystroke.

**3. Reading `useFormStatus` in the same component that renders the `<form>`.**

```jsx
// ❌ WRONG — pending is always false; the hook reads the *parent* form context
function MyForm() {
  const { pending } = useFormStatus();
  return <form action={save}><button disabled={pending}>Save</button></form>;
}
```

```jsx
// ✅ RIGHT — call it inside a child rendered within the form
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Save</button>;
}
function MyForm() {
  return <form action={save}><SubmitButton /></form>;
}
```

`useFormStatus` reads the nearest *ancestor* `<form>`, so it must live in a descendant.

**4. Assuming Server Components work in a vanilla Vite SPA.**

```jsx
// ❌ WRONG — there is no RSC runtime in a plain Vite SPA; an async component
// that awaits at the top level will not work as a Server Component here.
export default async function Page() {
  const data = await db.query(); // no server boundary exists
  return <div>{data}</div>;
}
```

```jsx
// ✅ RIGHT in a plain SPA — fetch on the client with Suspense + use()
const dataPromise = fetchData(); // stable
function Page() {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

RSC is a framework/bundler protocol. Without Next.js (or equivalent), use client-side data fetching.

**5. Subscribing to an external store with `useEffect` + `useState`.**

```jsx
// ❌ WRONG — risks tearing under concurrent rendering, and SSR mismatches
function useWidth() {
  const [w, setW] = useState(window.innerWidth);
  useEffect(() => {
    const on = () => setW(window.innerWidth);
    window.addEventListener("resize", on);
    return () => window.removeEventListener("resize", on);
  }, []);
  return w;
}
```

```jsx
// ✅ RIGHT — useSyncExternalStore is tear-safe and hydration-safe
import { useSyncExternalStore } from "react";
function useWidth() {
  return useSyncExternalStore(
    (cb) => { window.addEventListener("resize", cb); return () => window.removeEventListener("resize", cb); },
    () => window.innerWidth,
    () => 1024 // server snapshot
  );
}
```

---

## Best practices

- **Do** use `createRoot` (Vite's default) so concurrent features and automatic batching are active.
- **Do** mark navigation/tab/filter updates as transitions; **don't** transition keystrokes or controlled-input values.
- **Do** reach for `useDeferredValue` when the expensive value arrives as a prop you don't control; use `useTransition` when you own the setter.
- **Do** show an `isPending` / stale indicator during transitions so the lag is visible and intentional.
- **Don't** create promises inside render for `use()` — cache them or get them from a framework loader.
- **Do** nest Suspense boundaries to reveal independent regions early; **don't** wrap the whole app in one boundary that waits for everything.
- **Do** use `useId` for label/input associations and any SSR-rendered IDs; **never** use `Math.random()` for them.
- **Do** use `useSyncExternalStore` for any subscription to a mutable external source (browser APIs, non-React stores).
- **Do** prefer Actions (`<form action>`, `useActionState`, `useOptimistic`) over hand-rolled `isLoading`/`error` state for mutations.
- **Do** keep server→client props serializable; **don't** pass functions, class instances, or Dates expecting them to survive.
- **Do** enable the React Compiler and stop hand-writing `useMemo`/`useCallback` once it's on; **do** keep your render functions pure so the compiler can optimize them.
- **Don't** rely on `SuspenseList` in production — it's still experimental.

---

## Exercises

1. **Transition tabs.** Build a two-tab UI where one tab renders a list of 20,000 items. Wrap the tab-switch state update in `useTransition` and dim the content with `isPending`. Verify the buttons stay clickable during the switch.

2. **Deferred filter.** Render a controlled `<input>` and a component that filters a 30,000-element array by the query. Use `useDeferredValue` so typing is smooth, and reduce opacity while the displayed value is stale.

3. **Suspense data card.** Using `use()` and a module-scoped (cached) promise, fetch a user from a public API (e.g. `https://jsonplaceholder.typicode.com/users/1`) and render it inside `<Suspense fallback={…}>`.

4. **Tear-safe hook.** Write a `useOnlineStatus()` hook with `useSyncExternalStore` that subscribes to the browser `online`/`offline` events and returns `navigator.onLine`. Include a server snapshot of `true`.

5. **Optimistic form.** Build a comment form using `useActionState` and `useOptimistic` so a new comment appears instantly (greyed out) before the simulated 800 ms "save" resolves, then settles to normal.

### Solutions

```jsx
// Exercise 1 — Transition tabs
import { useState, useTransition } from "react";
function Big() {
  return <ul>{Array.from({ length: 20000 }, (_, i) => <li key={i}>{i}</li>)}</ul>;
}
export default function Ex1() {
  const [tab, setTab] = useState("a");
  const [isPending, startTransition] = useTransition();
  const go = (t) => startTransition(() => setTab(t));
  return (
    <div style={{ opacity: isPending ? 0.6 : 1 }}>
      <button onClick={() => go("a")}>A</button>
      <button onClick={() => go("b")}>B</button>
      {tab === "a" ? <p>Tab A</p> : <Big />}
    </div>
  );
}
```

```jsx
// Exercise 2 — Deferred filter
import { useState, useDeferredValue, useMemo } from "react";
export default function Ex2() {
  const [q, setQ] = useState("");
  const dq = useDeferredValue(q);
  const stale = q !== dq;
  const count = useMemo(
    () => Array.from({ length: 30000 }, (_, i) => `n${i}`).filter((s) => s.includes(dq)).length,
    [dq]
  );
  return (
    <>
      <input value={q} onChange={(e) => setQ(e.target.value)} />
      <p style={{ opacity: stale ? 0.5 : 1 }}>{count} matches</p>
    </>
  );
}
```

```jsx
// Exercise 3 — Suspense data card
import { Suspense, use } from "react";
const p = fetch("https://jsonplaceholder.typicode.com/users/1").then((r) => r.json());
function Card({ promise }) {
  const u = use(promise);
  return <div><h2>{u.name}</h2><p>{u.email}</p></div>;
}
export default function Ex3() {
  return <Suspense fallback={<p>Loading…</p>}><Card promise={p} /></Suspense>;
}
```

```jsx
// Exercise 4 — Tear-safe online hook
import { useSyncExternalStore } from "react";
function subscribe(cb) {
  window.addEventListener("online", cb);
  window.addEventListener("offline", cb);
  return () => {
    window.removeEventListener("online", cb);
    window.removeEventListener("offline", cb);
  };
}
export function useOnlineStatus() {
  return useSyncExternalStore(subscribe, () => navigator.onLine, () => true);
}
```

```jsx
// Exercise 5 — Optimistic comment form
import { useActionState, useOptimistic } from "react";
export default function Ex5() {
  // Pass the second element (formAction) straight to <form action>; call
  // addOptimistic inside the action so the preview lives in its transition.
  const [comments, formAction] = useActionState(async (prev, formData) => {
    const text = formData.get("text");
    addOptimistic(text);
    await new Promise((r) => setTimeout(r, 800));
    return [...prev, { id: Date.now(), text }];
  }, []);
  const [optimistic, addOptimistic] = useOptimistic(
    comments,
    (state, text) => [...state, { id: "temp", text, pending: true }]
  );
  return (
    <form action={formAction}>
      <input name="text" required />
      <button>Post</button>
      <ul>
        {optimistic.map((c) => (
          <li key={c.id} style={{ opacity: c.pending ? 0.5 : 1 }}>{c.text}</li>
        ))}
      </ul>
    </form>
  );
}
```

---

## Key takeaways

- **Concurrent rendering** makes renders interruptible and priority-aware; it's enabled by `createRoot` and underpins every feature in this lesson.
- **Automatic batching** (React 18+) coalesces all synchronous state updates in a tick into one render, everywhere — opt out with `flushSync`.
- **Transitions** (`useTransition`, standalone `startTransition`) mark updates non-urgent; **`useDeferredValue`** lags an expensive derived value so input stays responsive.
- **Suspense** lets components suspend on code or data; nest boundaries to reveal regions independently; promises read by **`use()`** must be stable.
- **`useId`** gives hydration-safe IDs and **`useSyncExternalStore`** gives tear-safe external subscriptions.
- **Server Components** run only on the server and ship no JS; the **`"use client"`** boundary marks the interactive islands; props across it must be serializable — and this whole protocol **needs a framework**.
- **Actions** (`<form action>`, `useActionState`, `useFormStatus`, `useOptimistic`) replace manual loading/error/optimistic state for mutations.
- React 19 ergonomics — **`ref` as a prop, `<Context>` as provider, metadata hoisting, preload APIs** — plus the **React Compiler** (auto-memoization) let you write simpler, less hand-tuned components.

---

## Further reading

- React docs — Hooks reference: <https://react.dev/reference/react/hooks>
- `useTransition`: <https://react.dev/reference/react/useTransition>
- `useDeferredValue`: <https://react.dev/reference/react/useDeferredValue>
- `<Suspense>`: <https://react.dev/reference/react/Suspense>
- `use`: <https://react.dev/reference/react/use>
- `useId`: <https://react.dev/reference/react/useId>
- `useSyncExternalStore`: <https://react.dev/reference/react/useSyncExternalStore>
- `useActionState`: <https://react.dev/reference/react/useActionState>
- `useFormStatus`: <https://react.dev/reference/react-dom/hooks/useFormStatus>
- `useOptimistic`: <https://react.dev/reference/react/useOptimistic>
- React 19 release notes: <https://react.dev/blog/2024/12/05/react-19>
- Server Components: <https://react.dev/reference/rsc/server-components>
- Server Actions / `"use client"` / `"use server"`: <https://react.dev/reference/rsc/use-client>
- React Compiler: <https://react.dev/learn/react-compiler>
- Preloading APIs (`preload`, `preinit`): <https://react.dev/reference/react-dom/preload>
