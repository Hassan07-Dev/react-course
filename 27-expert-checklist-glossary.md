# 27 — Expert Mastery Checklist, Interview Q&A, Glossary & Cheat Sheets

> **What you'll learn:** how to honestly self-assess your React mastery, answer the hard interview questions with precision, and revise the entire course from a single set of cheat sheets and a glossary.

By the end of this capstone lesson you will be able to:

- Run a structured **self-assessment** across every major topic in the course and find your gaps.
- Give **concise, technically correct answers** to the most common (and trickiest) React interview questions.
- Define **every term** the course uses, from "reconciliation" to "transition," without hand-waving.
- Read and use **cheat sheets** for hooks, JSX rules, lifecycle↔hooks mapping, and the ESLint hook rules.
- Distinguish **CSR / SSR / SSG / RSC** and explain when each applies.
- Reason about **what actually triggers a re-render** and why `React.memo` / `useMemo` / `useCallback` matter (and when they don't).
- Lay out a **concrete "what to learn next"** roadmap to go beyond this course.

---

## Prerequisites

This lesson assumes you have worked through the whole course. It is a synthesis, not new material. In particular you should already understand:

- JSX and the render model — see `03-jsx-in-depth.md` and `04-components-and-props.md`.
- `useState`, events, and lists/keys — see `05-rendering-lists-conditionals.md`, `06-handling-events.md`, `07-state-usestate.md`.
- Effects, the dependency array, and cleanup — see `08-useeffect-side-effects.md` and `09-hooks-deep-dive.md`.
- Custom hooks and the Rules of Hooks — see `10-custom-hooks.md`.
- Context, performance, and state management — see `12-context-api.md`, `13-performance.md`, `16-state-management.md`.
- TypeScript with React and testing — see `18-typescript-react.md` and `19-testing.md`.

If any checklist item below makes you pause, jump back to the referenced lesson before continuing.

---

## Concept

An expert is not someone who has memorized every API. An expert is someone who has an accurate **mental model** of how the system behaves and can predict its output. This lesson exists to pressure-test that mental model.

There are three ways to verify mastery, and we use all three:

1. **The checklist** — "Can I *explain* X, and can I *implement* X?" Explaining proves understanding; implementing proves it isn't an illusion. The two are different skills, and interviews probe both.
2. **The Q&A bank** — interview questions are deliberately chosen to expose shallow understanding. "What is the Virtual DOM?" sounds trivial but the follow-ups ("so is it always faster than direct DOM manipulation?") separate the memorizer from the engineer.
3. **The cheat sheets** — a compressed, lookup-friendly reference. Experts don't keep every signature in their head; they keep the *shape* of the API in their head and look up the details fast.

Think of this lesson like the rubber-duck and the answer key bundled together. Read each question, answer it out loud *before* reading the answer, and mark where you were vague. Vagueness is the gap.

---

## Syntax & API

This lesson references APIs covered across the course; the only "new" deliverable is the reference material itself. The canonical signatures you should be able to recall cold:

```js
const [state, setState] = useState(initialValue);            // 07
useEffect(setup, deps?);                                     // 08
const memoized = useMemo(() => compute(a, b), [a, b]);       // 13
const cb = useCallback(() => doThing(a), [a]);               // 13
const ref = useRef(initialValue);                            // 09
const value = useContext(MyContext);                         // 12
const [state, dispatch] = useReducer(reducer, init);         // 16
const [isPending, startTransition] = useTransition();        // R19
const deferred = useDeferredValue(value);                    // R19
const id = useId();                                          // 09
const value = use(promiseOrContext);                         // R19
const [state, formAction, isPending] = useActionState(fn, s);// R19 (Actions)
const { pending } = useFormStatus();                         // R19 (Actions)
```

Memoization wrappers:

```jsx
const MemoComponent = React.memo(Component, areEqual?); // skip re-render if props are shallow-equal
const Lazy = React.lazy(() => import('./Heavy.jsx'));   // code-split component
```

These are all real React 19 APIs. The full cheat-sheet table is in **Cheat Sheets** below.

---

## Examples

These examples each crystallize one interview-grade concept into runnable code.

### Example 1 — Proving the stale-closure bug (and three fixes)

```jsx
import { useState, useEffect, useRef } from 'react';

// BUG: this counter freezes at 1 because the interval closes over count=0 forever.
function BrokenCounter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount(count + 1), 1000); // stale `count`
    return () => clearInterval(id);
  }, []); // empty deps → effect runs once → closure captures count=0
  return <p>{count}</p>;
}

// FIX A: functional update — never reads the captured value.
function FixedFunctional() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount((c) => c + 1), 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{count}</p>;
}

// FIX B: a ref that always holds the latest value.
function FixedRef() {
  const [count, setCount] = useState(0);
  const latest = useRef(count);
  latest.current = count; // updated every render
  useEffect(() => {
    const id = setInterval(() => setCount(latest.current + 1), 1000);
    return () => clearInterval(id);
  }, []);
  return <p>{count}</p>;
}
```

The stale closure is the single most common React bug. The interviewer wants to hear: *"the effect runs once, so its closure captured `count` at the value it had on that render; calling `setCount(count + 1)` always computes `0 + 1`. The functional updater sidesteps the closure entirely."* See `08-useeffect-side-effects.md` and `09-hooks-deep-dive.md`.

### Example 2 — When `React.memo` actually helps (and when it's wasted)

```jsx
import { memo, useState, useCallback } from 'react';

const ExpensiveChild = memo(function ExpensiveChild({ onClick, label }) {
  console.log('render ExpensiveChild');
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [, forceTick] = useState(0);

  // WITHOUT useCallback, `handleClick` is a new function each render,
  // so memo() sees a changed prop and re-renders the child anyway.
  const handleClick = useCallback(() => setCount((c) => c + 1), []);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => forceTick((t) => t + 1)}>Re-render parent</button>
      <ExpensiveChild onClick={handleClick} label="Increment" />
    </>
  );
}
```

The lesson: `memo` compares props by shallow equality. A new inline function or object prop **defeats** it. `useCallback`/`useMemo` exist precisely to keep referential identity stable so `memo` can do its job. Without the child being wrapped in `memo`, `useCallback` here would buy you *nothing*. See `13-performance.md`.

### Example 3 — Controlled vs uncontrolled, side by side

```jsx
import { useState, useRef } from 'react';

function Controlled() {
  const [value, setValue] = useState(''); // React owns the value
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value.toUpperCase())}
    />
  );
}

function Uncontrolled() {
  const inputRef = useRef(null); // the DOM owns the value
  const submit = () => alert(inputRef.current.value);
  return (
    <>
      <input ref={inputRef} defaultValue="hi" />
      <button onClick={submit}>Read</button>
    </>
  );
}
```

Controlled inputs make every keystroke flow through state — required for validation, transformation, or syncing. Uncontrolled inputs let the DOM keep the value and you read it on demand — simpler, fewer re-renders, great for plain forms or file inputs. See `11-forms.md`.

### Example 4 — React 19 Actions: a form with pending state, no manual `useState`

```jsx
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus(); // reads the enclosing <form>'s pending state
  return <button disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}

async function saveName(previousState, formData) {
  const name = formData.get('name');
  if (!name) return { error: 'Name required' };
  await fetch('/api/name', { method: 'POST', body: formData });
  return { ok: true };
}

export default function NameForm() {
  const [state, formAction, isPending] = useActionState(saveName, {});
  return (
    <form action={formAction}>
      <input name="name" />
      <SubmitButton />
      {state.error && <p role="alert">{state.error}</p>}
      {state.ok && <p>Saved!</p>}
    </form>
  );
}
```

React 19 **Actions** let an async function be passed to `<form action={...}>`. `useActionState` tracks the returned state and a pending flag; `useFormStatus` lets a *child* (like the button) read the parent form's pending state without prop drilling. This is the modern replacement for the manual `isLoading` + `try/catch` boilerplate.

---

## Under the hood

To answer expert questions, hold these mental models:

- **A render is just calling your function.** When React renders `<Foo />`, it literally calls `Foo(props)`. The return value is a tree of React elements — plain objects like `{ type, props, key }`, not DOM nodes. Rendering is pure and side-effect-free; nothing has touched the DOM yet.

- **The Virtual DOM is a description, reconciliation is the diff.** React keeps the element tree from the previous render and compares it to the new one (**reconciliation**), then computes the minimal set of real DOM mutations (the **commit**). The Virtual DOM isn't magic speed — it's a *batching and diffing* layer that trades a little CPU for far fewer, well-coordinated DOM writes and a declarative programming model.

- **The diffing heuristic is O(n), not O(n³).** React assumes: (1) two elements of different `type` produce different trees (so it unmounts and rebuilds), and (2) `key`s tell React which children are stable across renders. Keys are how React matches list items between renders; a wrong/index key causes state to attach to the wrong item. See `05-rendering-lists-conditionals.md`.

- **Hooks are positional.** React stores hook state in a linked list keyed by **call order**, per component instance. That is *why* the Rules of Hooks exist: call them at the top level, never in conditions/loops, so the order is identical every render. Break the order and React reads the wrong slot.

- **Fiber + two phases.** React's reconciler ("Fiber") works in a **render phase** (interruptible, can be paused/restarted — must be pure) and a **commit phase** (synchronous, applies DOM mutations and runs `useLayoutEffect`, then later `useEffect`). Concurrent features (transitions, Suspense) rely on the render phase being interruptible.

- **State updates are batched.** Multiple `setState` calls inside the same event (and, in React 18+, inside promises/timeouts too — *automatic batching*) are coalesced into one re-render. This is why reading state right after setting it gives the old value.

- **Events are synthetic and delegated.** React attaches **one** listener per event type at the React root (React 17+; pre-17 it was `document`) and dispatches a pooled-free `SyntheticEvent` to your handler. You write `onClick` on thousands of elements but there aren't thousands of DOM listeners.

- **Effects run after paint.** `useEffect` fires after the browser paints; `useLayoutEffect` fires synchronously after DOM mutation but before paint. Cleanup runs before the next effect and on unmount. In Strict Mode (dev only), React intentionally mounts→unmounts→remounts once to surface missing cleanup.

---

## Common mistakes & pitfalls

**1. Mutating state instead of replacing it.**

```jsx
// WRONG — mutates the existing array; React sees the same reference, skips re-render
const add = (item) => { items.push(item); setItems(items); };

// RIGHT — new reference
const add = (item) => setItems((prev) => [...prev, item]);
```

**2. Using array index as a `key` for a reorderable list.**

```jsx
// WRONG — index keys break when items reorder/insert/delete
{rows.map((row, i) => <Row key={i} {...row} />)}

// RIGHT — stable identity
{rows.map((row) => <Row key={row.id} {...row} />)}
```

**3. Lying about the dependency array.**

```jsx
// WRONG — uses `query` but omits it; effect won't re-run when query changes
useEffect(() => { fetchData(query); }, []);

// RIGHT — include every value the effect reads
useEffect(() => { fetchData(query); }, [query]);
```

**4. Forgetting effect cleanup → race conditions / leaks.**

```jsx
// WRONG — a slow earlier response can overwrite a newer one
useEffect(() => {
  fetch(`/api?q=${q}`).then((r) => r.json()).then(setData);
}, [q]);

// RIGHT — ignore stale responses
useEffect(() => {
  let cancelled = false;
  fetch(`/api?q=${q}`).then((r) => r.json()).then((d) => { if (!cancelled) setData(d); });
  return () => { cancelled = true; };
}, [q]);
```

**5. Reaching for Context as a state manager for hot, frequently-changing state.**

```jsx
// WRONG — every consumer re-renders on every keystroke when value is one big object
<AppContext.Provider value={{ user, theme, cart, setCart }}>

// RIGHT — split contexts by change frequency, or use a store (Zustand/Redux) for hot state
<UserContext.Provider value={user}>
  <CartContext.Provider value={cartValue}>...
```

See `12-context-api.md` and `16-state-management.md`.

**6. Calling a hook conditionally.**

```jsx
// WRONG — violates Rules of Hooks; hook order changes between renders
if (isLoggedIn) { const [name, setName] = useState(''); }

// RIGHT — hook at top level, branch inside
const [name, setName] = useState('');
if (isLoggedIn) { /* use name */ }
```

---

## Best practices

- **Do** model state as the minimal source of truth; derive everything else during render instead of storing duplicate state.
- **Do** prefer functional updates (`setX(prev => …)`) whenever the next value depends on the previous one.
- **Do** give list items stable, data-derived keys.
- **Do** include every reactive value an effect reads in its dependency array, and let `eslint-plugin-react-hooks` enforce it.
- **Do** measure before memoizing — profile with the React DevTools Profiler; `memo`/`useMemo`/`useCallback` have a cost.
- **Don't** put rendering logic side effects (fetching, subscriptions, DOM writes) in the render body — use effects/event handlers.
- **Don't** mutate props, state, or context values.
- **Don't** disable the exhaustive-deps lint rule to silence a warning; fix the underlying dependency instead.
- **Don't** reach for Redux/global state when lifting state up or Context solves it; don't reach for Context when prop drilling is one level deep.
- **Don't** use `useLayoutEffect` unless you must measure or mutate the DOM before paint — prefer `useEffect`.

---

## Exercises

**Exercise 1 (warm-up).** Without looking, fill in the hooks cheat-sheet table from memory: name every hook and its one-line purpose. Then check against the table below.

**Exercise 2.** Take `BrokenCounter` from Example 1. Explain *in writing* why it freezes, then implement the fix using `useReducer` instead of the functional updater.

**Exercise 3.** Write a 2-sentence answer for each of: Virtual DOM, reconciliation, why keys matter, controlled vs uncontrolled, and `useMemo` vs `useCallback`. Compare against the Q&A bank.

**Exercise 4 (interview sim).** A teammate wrapped every component in `React.memo` "for performance." Explain when this is harmless, when it helps, and when it actively hurts — with a concrete example of each.

**Exercise 5 (capstone).** Build a search box that: debounces input, fetches results, cancels stale requests, shows a pending indicator via a transition, and renders results with stable keys. Use React 19 features where they fit.

### Solutions

**Solution 2.**

```jsx
import { useReducer, useEffect } from 'react';

function reducer(count, action) {
  return action === 'tick' ? count + 1 : count;
}

function ReducerCounter() {
  const [count, dispatch] = useReducer(reducer, 0);
  useEffect(() => {
    const id = setInterval(() => dispatch('tick'), 1000);
    return () => clearInterval(id);
  }, []); // dispatch identity is stable, so empty deps is correct here
  return <p>{count}</p>;
}
```

The interval closes over `dispatch`, whose identity React guarantees is stable. `dispatch('tick')` recomputes from the *current* state inside the reducer, so there is no stale value — the same reason the functional updater works.

**Solution 5.**

```jsx
import { useState, useEffect, useTransition, useDeferredValue } from 'react';

function Search() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query); // results lag behind input, stay responsive
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  useEffect(() => {
    if (!deferredQuery) { setResults([]); return; }
    let cancelled = false;
    const id = setTimeout(() => {
      fetch(`/api/search?q=${encodeURIComponent(deferredQuery)}`)
        .then((r) => r.json())
        .then((data) => {
          if (!cancelled) startTransition(() => setResults(data));
        });
    }, 300); // debounce
    return () => { cancelled = true; clearTimeout(id); }; // cancel stale request + timer
  }, [deferredQuery]);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {isPending && <span>Searching…</span>}
      <ul>
        {results.map((r) => <li key={r.id}>{r.title}</li>)}
      </ul>
    </div>
  );
}
```

This combines a debounce (timer + cleanup), stale-request cancellation (the `cancelled` flag), responsiveness (`useDeferredValue` + `useTransition`), and stable keys (`r.id`).

---

## Interview Q&A bank

Answer each aloud before reading the answer.

**Q: What is the Virtual DOM and what is reconciliation?**
A: The Virtual DOM is an in-memory tree of plain JS objects (React elements) describing what the UI should look like. Reconciliation is the process of diffing the new tree against the previous one to compute the minimal real-DOM changes, which are then applied in the commit phase. The VDOM's value is the declarative model and batched updates — not guaranteed speed.

**Q: Explain the diffing algorithm and the role of keys.**
A: React's heuristic diff is O(n). Different element `type` ⇒ tear down and rebuild. Same type ⇒ update props in place. For lists, `key` identifies which child is which across renders so React can move/keep/remove rather than rebuild. Index keys break identity when items reorder or are inserted/removed, causing wrong state or DOM reuse.

**Q: Why do hooks have rules?**
A: React tracks hook state by call order in a per-instance list. Calling hooks at the top level (never in conditions/loops/nested functions) keeps the order stable across renders so React reads the correct state slot. They must also be called only from React functions (components or custom hooks).

**Q: How do `useState` and closures interact — what is a stale closure?**
A: Each render is a snapshot; the state value and any handlers close over the values from *that* render. If an effect/callback that was created on an old render later runs, it sees the old value. Functional updates (`setX(prev => …)`) or refs avoid relying on the captured value.

**Q: `useMemo` vs `useCallback` vs `React.memo`?**
A: `useMemo` caches a *computed value*. `useCallback` caches a *function reference* (`useCallback(fn, d)` ≈ `useMemo(() => fn, d)`). `React.memo` wraps a *component* to skip re-render when its props are shallow-equal. The two hooks are how you keep props referentially stable so `memo` can skip work.

**Q: Controlled vs uncontrolled components?**
A: Controlled = React state is the single source of truth (`value` + `onChange`). Uncontrolled = the DOM holds the value, read via a ref (`defaultValue`). Controlled gives you validation/transformation/live sync; uncontrolled is simpler and re-renders less.

**Q: Context vs Redux?**
A: Context is a *dependency-injection* mechanism for passing a value down the tree without prop drilling; it has no built-in update optimization — every consumer re-renders when the value changes. Redux (or Zustand/Jotai) is a *state-management* library with a store, selectors, and fine-grained subscription so only components reading changed slices re-render. Use Context for low-frequency global values (theme, user, locale); use a store for hot, frequently-updated, or complex shared state.

**Q: `useEffect` cleanup and the dependency array?**
A: The deps array lists every reactive value the effect reads; the effect re-runs when any changes (empty array = once after mount; omitted = every render). The cleanup function runs before the next effect run and on unmount — use it to cancel subscriptions, timers, and in-flight requests to avoid leaks and races.

**Q: CSR vs SSR vs SSG vs RSC?**
A: **CSR** (client-side rendering): the server sends an empty shell; JS renders everything in the browser (e.g., a Vite SPA). **SSR**: HTML is rendered per-request on the server then hydrated. **SSG**: HTML is rendered at *build* time and served statically. **RSC** (React Server Components): components run on the server, send a serialized render output (not HTML, not extra JS) to the client, and never ship their code to the browser — combined with client components for interactivity.

**Q: What causes a component to re-render?**
A: (1) its own state changed (`setState`), (2) its parent re-rendered (default behavior, regardless of prop changes), (3) a Context value it consumes changed, or (4) a `useSyncExternalStore` subscription fired. Note: changing props does *not* by itself trigger a re-render — the parent re-rendering does.

**Q: What are Suspense and transitions?**
A: `<Suspense fallback={…}>` lets you declaratively show a fallback while a child is "suspending" (e.g., lazy code or data via the `use` hook / supported data libs). Transitions (`useTransition`, `startTransition`) mark a state update as non-urgent so React can keep the UI responsive, interrupt the work, and show pending state instead of blocking on a janky update.

**Q: How does event handling/delegation work in React?**
A: React attaches one listener per event type at the root container and dispatches `SyntheticEvent` objects (a cross-browser wrapper) to your handlers. You don't attach real listeners per element. `e.stopPropagation()`/`e.preventDefault()` work on the synthetic event.

**Q: What does "lifting state up" mean?**
A: When two siblings need the same state, move it to their nearest common parent and pass it down via props (plus setters/callbacks). The parent becomes the single source of truth.

**Q: What are solutions to prop drilling?**
A: Lift state to the right level; pass JSX as `children`/render props (composition) so intermediate components don't need the data; use Context for cross-cutting values; or use a state-management store with selectors.

**Q: What are error boundaries?**
A: Class components implementing `getDerivedStateFromError` and/or `componentDidCatch` that catch render-phase errors in their subtree and show a fallback UI instead of crashing the whole app. Hooks can't be error boundaries yet; libraries like `react-error-boundary` wrap the class for ergonomic use. They do *not* catch errors in event handlers or async code.

**Q: What are React 19 Actions?**
A: Async functions wired into `<form action={fn}>` (and `formAction` on buttons) that handle submission. Paired hooks: `useActionState` (track returned state + pending), `useFormStatus` (read the enclosing form's pending state from a child), and `useOptimistic` (show an optimistic value while the action runs). They standardize form submission, pending UI, and error handling.

---

## Mastery checklist

Mark each: **E** = I can explain it, **I** = I can implement it from scratch.

**Foundations**
- [ ] JSX is sugar for `React.createElement`; I can read the compiled output. (`03`)
- [ ] Components are functions returning elements; props are read-only inputs. (`04`)
- [ ] Rendering lists with stable keys; conditional rendering patterns. (`05`)
- [ ] Synthetic events, `preventDefault`, passing args to handlers. (`06`)

**State & effects**
- [ ] `useState`, batching, functional updates, lazy initial state. (`07`)
- [ ] `useEffect` lifecycle, deps array, cleanup, Strict Mode double-invoke. (`08`)
- [ ] `useRef`, `useReducer`, `useContext`, `useMemo`, `useCallback`, `useId`. (`09`)
- [ ] Authoring custom hooks and sharing logic. (`10`)

**Forms & data**
- [ ] Controlled vs uncontrolled inputs; form validation. (`11`)
- [ ] Data fetching, loading/error states, race-condition handling. (`15`)

**Architecture**
- [ ] Context API and its re-render characteristics. (`12`)
- [ ] State management choices (lift / Context / reducer / external store). (`16`)
- [ ] Performance: `memo`, memoization hooks, profiling, code-splitting. (`13`)
- [ ] Routing fundamentals. (`14`)
- [ ] Styling approaches. (`17`)
- [ ] Advanced patterns (compound components, render props, HOCs, portals). (`20`)

**Quality & types**
- [ ] Typing components, props, hooks, and events with TypeScript. (`18`)
- [ ] Testing components and hooks; what to test and how. (`19`)

**React 19 / runtime depth**
- [ ] Reconciliation, Fiber, render vs commit phases.
- [ ] Suspense, `useTransition`, `useDeferredValue`, `use`.
- [ ] Actions: `useActionState`, `useFormStatus`, `useOptimistic`.
- [ ] CSR / SSR / SSG / RSC trade-offs.
- [ ] Error boundaries.

If you can check **both E and I** on every line, you have completed the course's stated goal.

---

## Cheat Sheets

### Hooks quick-reference

| Hook | Signature | Use it for |
| --- | --- | --- |
| `useState` | `const [s, setS] = useState(init)` | Local reactive state |
| `useReducer` | `const [s, dispatch] = useReducer(r, init)` | Complex/multi-action state transitions |
| `useEffect` | `useEffect(setup, deps?)` | Side effects after paint (subscriptions, fetch, sync) |
| `useLayoutEffect` | `useLayoutEffect(setup, deps?)` | DOM measurement/mutation before paint |
| `useInsertionEffect` | `useInsertionEffect(setup, deps?)` | Injecting styles (CSS-in-JS libs) before layout |
| `useRef` | `const r = useRef(init)` | Mutable value that doesn't trigger re-render; DOM refs |
| `useImperativeHandle` | `useImperativeHandle(ref, () => api, deps?)` | Expose an imperative API from a child |
| `useMemo` | `const v = useMemo(fn, deps)` | Cache an expensive computed value |
| `useCallback` | `const fn = useCallback(fn, deps)` | Cache a function reference |
| `useContext` | `const v = useContext(Ctx)` | Read a Context value |
| `useId` | `const id = useId()` | SSR-safe unique IDs for a11y attributes |
| `useSyncExternalStore` | `useSyncExternalStore(sub, getSnap, getServerSnap?)` | Subscribe to external stores safely |
| `useDebugValue` | `useDebugValue(value)` | Label custom hooks in DevTools |
| `useTransition` | `const [isPending, start] = useTransition()` | Mark updates as non-urgent |
| `useDeferredValue` | `const d = useDeferredValue(v)` | Defer a value to keep UI responsive |
| `use` | `const v = use(promiseOrContext)` | Read a promise/Context (suspends; can be conditional) |
| `useActionState` | `const [s, action, pending] = useActionState(fn, init)` | Form action state + pending (R19) |
| `useFormStatus` | `const { pending } = useFormStatus()` | Read enclosing form's pending state (R19, from `react-dom`) |
| `useOptimistic` | `const [opt, addOpt] = useOptimistic(state, fn)` | Optimistic UI during an action (R19) |

### JSX rules

- Return a **single root** (use a Fragment `<>…</>` to group siblings).
- Attributes are camelCase: `className`, `htmlFor`, `onClick`, `tabIndex`.
- Close every tag: `<img />`, `<br />`.
- `{}` embeds JS expressions (not statements). Use ternaries/`&&` for conditionals.
- `{0 && <X/>}` renders `0`; coerce: `{count > 0 && <X/>}`.
- `style` takes an object with camelCased props: `style={{ marginTop: 8 }}`.
- Components must be **PascalCase**; lowercase ⇒ treated as a DOM tag.
- `key` belongs on the element you map over, must be unique among siblings, stable across renders.

### Lifecycle ↔ hooks mapping

| Class lifecycle | Hook equivalent |
| --- | --- |
| `constructor` initial state | `useState`/`useReducer` initializer |
| `componentDidMount` | `useEffect(setup, [])` |
| `componentDidUpdate` | `useEffect(setup, [deps])` |
| `componentWillUnmount` | cleanup `return () => {}` in `useEffect` |
| `shouldComponentUpdate` | `React.memo` (+ `useMemo`/`useCallback`) |
| `getDerivedStateFromError` / `componentDidCatch` | Error boundary (class only) |
| `getSnapshotBeforeUpdate` / sync DOM reads | `useLayoutEffect` |

### Common ESLint / Rules of Hooks

- **Only call hooks at the top level** — never in loops, conditions, or nested functions.
- **Only call hooks from React function components or custom hooks** (custom hooks start with `use`).
- **`react-hooks/exhaustive-deps`** — include all reactive values used inside effects/`useMemo`/`useCallback`. Fix the dependency, don't suppress the rule.
- **Don't ignore `react-hooks/rules-of-hooks`** — it catches conditional hook calls statically.
- React 19's **React Compiler** can auto-memoize; it still requires Rules-of-Hooks-clean code to do so safely.

### What to learn next — roadmap

- **Advanced TypeScript with React:** generic components, discriminated-union props, `as const`, template-literal types, typing context/reducers. Build on `18-typescript-react.md`.
- **Next.js in depth:** App Router, RSC + server actions, streaming SSR, caching layers, route handlers, middleware.
- **React Native:** the same mental model, different primitives (`View`/`Text`), Expo, native modules.
- **Animation:** Framer Motion / `motion`, `react-spring`, the View Transitions API, layout animations.
- **Testing depth:** React Testing Library patterns, MSW for network mocking, Playwright/Cypress E2E, accessibility testing. Build on `19-testing.md`.
- **Performance:** the React Compiler, virtualization (`@tanstack/react-virtual`), bundle analysis, Suspense data patterns, profiling flame charts.
- **Design systems:** headless component libraries (Radix), tokens, theming, Storybook, a11y as a first-class concern.

---

## Key takeaways

- A render is **calling your function**; reconciliation diffs the resulting element tree and the commit applies minimal DOM changes — the Virtual DOM is a *model*, not a speed guarantee.
- **Hooks are positional**, which is the entire reason for the Rules of Hooks.
- **Stale closures** come from each render being a snapshot; functional updates and refs are the escape hatches.
- A component re-renders from **its own state, its parent re-rendering, or a consumed Context changing** — props alone don't trigger it.
- `useMemo` caches values, `useCallback` caches functions, `React.memo` skips component re-renders — and they only help when referential stability lets `memo` do its job.
- Pick the **smallest tool**: lift state → Context for low-frequency globals → external store for hot/complex state.
- React 19 standardizes async UI with **Actions, transitions, Suspense, and `use`** — prefer them over hand-rolled loading boilerplate.
- Mastery = **explain + implement**; use the checklist to find the lines where you can only do one.

---

## Further reading

- React docs — Describing the UI, Adding Interactivity, Managing State, Escape Hatches: <https://react.dev/learn>
- Rules of Hooks: <https://react.dev/reference/rules/rules-of-hooks>
- Reconciliation / "Preserving and Resetting State": <https://react.dev/learn/preserving-and-resetting-state>
- `useTransition`, `useDeferredValue`, `use`: <https://react.dev/reference/react>
- React 19 release notes (Actions, `useActionState`, `useOptimistic`, `useFormStatus`): <https://react.dev/blog/2024/12/05/react-19>
- Server Components: <https://react.dev/reference/rsc/server-components>
- React DevTools Profiler: <https://react.dev/learn/react-developer-tools>
- `eslint-plugin-react-hooks`: <https://www.npmjs.com/package/eslint-plugin-react-hooks>
