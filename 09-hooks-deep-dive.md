# 09 — Hooks Deep Dive (Rules + the Core Hooks)

> **What you'll learn:** the two Rules of Hooks and the deep "why" behind them, plus a complete, expert-level tour of every remaining built-in React hook — when to reach for each one and how it behaves at runtime.

By the end of this lesson you will be able to:

- State and apply the **two Rules of Hooks**, and explain *why* call order (not names) is what React uses to match hooks to their state.
- Configure and trust **`eslint-plugin-react-hooks`** to catch violations automatically.
- Use **`useRef`** for both DOM access and as a mutable, render-stable "box" that does not trigger re-renders.
- Consume context with **`useContext`** (provider creation is covered in `12-context-api.md`).
- Choose and write a **`useReducer`** with lazy initialization, and combine it with context.
- Apply **`useMemo`** and **`useCallback`** for expensive computations and referential stability, and articulate the `useCallback`/`useMemo` equivalence.
- Use the lower-level hooks **`useLayoutEffect`**, **`useImperativeHandle`**, **`useId`**, and **`useDebugValue`** correctly.
- Follow a **decision guide** that maps a problem to the right hook.

---

## Prerequisites

You should be comfortable with the material from these earlier lessons before starting:

- **`03-jsx-in-depth.md`** — how JSX compiles and how React renders a component tree.
- **`04-components-and-props.md`** — function components and props.
- **`07-state-usestate.md`** — `useState`, the render cycle, and immutable state updates. This lesson assumes you already know `useState`.
- **`08-useeffect-side-effects.md`** — `useEffect`, dependency arrays, and cleanup. `useLayoutEffect` here is explained relative to `useEffect`.

If you have not read `07` and `08`, read them first — this lesson treats `useState` and `useEffect` as known building blocks and focuses on *everything else*.

---

## Concept

A **hook** is a special function whose name starts with `use` that lets a function component "hook into" React features: state, lifecycle, context, refs, and more. Hooks were introduced in React 16.8 and are the standard way to write React today. Class components and lifecycle methods are legacy.

### The mental model: hooks live on the component instance

When React renders a function component, it does not just call your function and forget about it. React maintains, *behind the scenes*, a list of "memory cells" attached to that particular component instance — one cell per hook call, stored in the order the hooks were called. Think of it as a notebook with numbered lines:

```
Line 1: useState(0)      → remembers the count
Line 2: useState('')     → remembers the name
Line 3: useEffect(...)    → remembers the effect + deps
```

On the **first render**, React walks down your function top to bottom. Each hook call appends a new cell. On **every subsequent render**, React walks down again — but this time it *reads back* the cells in the same order, matching "the first `useState` I see" to "line 1," "the second hook I see" to "line 2," and so on.

This is the single most important idea in the whole lesson: **React identifies a hook purely by its position in the call sequence, not by any name or key.** There is no label tying `count` to its cell — only the fact that `useState(0)` was the *first* hook called.

That design is what makes hooks ergonomic (no boilerplate, no `this`), but it also dictates a strict contract: **the order and number of hook calls must be identical on every render.** Break that, and React reads the wrong cell — your `count` state suddenly returns the `name` value. This is why the Rules of Hooks exist; they are not style preferences, they are correctness requirements.

---

## Syntax & API

### The two Rules of Hooks

1. **Only call hooks at the top level.** Do not call hooks inside loops, conditions, nested functions, or after an early `return`. Hooks must run in the same order on every render.
2. **Only call hooks from React functions.** Call them from function components or from *custom hooks* (functions whose names start with `use`). Never from regular JS functions, event handlers, or class components.

```jsx
// ✅ Top level of a component
function Profile() {
  const [name, setName] = useState('');
  useEffect(() => {/* ... */});
  // ...
}

// ✅ Top level of a custom hook (covered fully in 10-custom-hooks.md)
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  // ...
  return width;
}
```

### `eslint-plugin-react-hooks`

```bash
npm install --save-dev eslint-plugin-react-hooks
```

This official plugin ships two rules:

- **`react-hooks/rules-of-hooks`** — flags violations of the two rules above.
- **`react-hooks/exhaustive-deps`** — warns when a dependency array is missing a value used inside the effect/callback/memo.

A Vite + ESLint flat-config snippet. In v5+ the plugin ships ready-made flat-config presets under `configs.flat`, so the simplest setup is to add the recommended preset directly to your config array:

```js
// eslint.config.js
import reactHooks from 'eslint-plugin-react-hooks';

export default [
  reactHooks.configs.flat.recommended,
];
```

If you prefer to wire the rules by hand (e.g. to scope them to specific files), register the plugin and turn on each rule explicitly:

```js
// eslint.config.js
import reactHooks from 'eslint-plugin-react-hooks';

export default [
  {
    files: ['**/*.{js,jsx}'],
    plugins: { 'react-hooks': reactHooks },
    rules: {
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
    },
  },
];
```

### The hooks covered in this lesson

```js
const ref          = useRef(initialValue);
const value        = useContext(SomeContext);
const [state, dispatch] = useReducer(reducer, initialArg, init?);
const memoized     = useMemo(() => computeExpensive(a, b), [a, b]);
const memoizedFn   = useCallback((e) => doThing(a), [a]);
useLayoutEffect(() => { /* runs synchronously before paint */ }, [deps]);
useImperativeHandle(ref, () => ({ focus() {/*...*/} }), [deps]);
const id           = useId();
useDebugValue(value, format?);
```

Detailed notes per hook follow in **Examples** and **Under the hood**.

---

## Examples

### Example 1 — `useRef`: a DOM handle

`useRef(initial)` returns a stable object `{ current: initial }`. The *same object* is returned on every render. Mutating `ref.current` does **not** trigger a re-render. Its first job is grabbing a DOM node:

```jsx
import { useRef } from 'react';

function SearchBox() {
  const inputRef = useRef(null);

  function focusInput() {
    inputRef.current.focus(); // imperative DOM call
  }

  return (
    <div>
      <input ref={inputRef} placeholder="Search…" />
      <button onClick={focusInput}>Focus the input</button>
    </div>
  );
}
```

When you pass a ref to a built-in element's `ref` attribute, React sets `inputRef.current` to the DOM node after commit, and back to `null` on unmount.

### Example 2 — `useRef`: a mutable box that survives renders

Because a ref persists across renders *without* causing them, it is the right tool for values that are not part of the visual output: timer IDs, the previous value of a prop, a "has this already run" flag.

```jsx
import { useState, useRef, useEffect } from 'react';

function Stopwatch() {
  const [seconds, setSeconds] = useState(0);
  const intervalRef = useRef(null); // holds the timer id between renders

  function start() {
    if (intervalRef.current !== null) return; // already running
    intervalRef.current = setInterval(() => {
      setSeconds((s) => s + 1);
    }, 1000);
  }

  function stop() {
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  }

  // Cleanup if the component unmounts mid-run
  useEffect(() => () => clearInterval(intervalRef.current), []);

  return (
    <div>
      <p>{seconds}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

Storing the interval id in state would be wrong: it is not rendered, and updating it would needlessly re-render. A ref is exactly the "instance variable" a class component used to give you via `this.intervalId`.

A classic "previous value" pattern uses a ref written inside an effect:

```jsx
function usePrevious(value) {
  const ref = useRef(undefined);
  useEffect(() => {
    ref.current = value; // runs *after* render commits
  }, [value]);
  return ref.current; // returns the value from the previous render
}
```

### Example 3 — `useContext`: consuming shared data

`useContext(SomeContext)` reads the value of the nearest matching `<SomeContext.Provider>` above in the tree. Creating and providing context is the subject of `12-context-api.md`; here we focus on *consuming*.

```jsx
import { createContext, useContext } from 'react';

const ThemeContext = createContext('light'); // default used if no provider

function ThemedButton() {
  const theme = useContext(ThemeContext); // 'light' | 'dark'
  return <button className={`btn btn-${theme}`}>Click</button>;
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
}
```

A component calling `useContext` re-renders whenever the provider's `value` changes. That has performance implications — see **Under the hood**.

### Example 4 — `useReducer`: predictable state transitions

`useReducer(reducer, initialArg, init?)` is `useState`'s heavier sibling. Instead of setting state directly, you `dispatch` an *action* and a pure `reducer(state, action)` computes the next state.

```jsx
import { useReducer } from 'react';

const initialState = { count: 0, step: 1 };

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.step };
    case 'reset':
      return initialState;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count} (step {state.step})</p>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <input
        type="number"
        value={state.step}
        onChange={(e) => dispatch({ type: 'setStep', step: Number(e.target.value) })}
      />
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

**When to prefer `useReducer` over `useState`:**

- State has multiple sub-values that change together, or the next value depends on the previous in non-trivial ways.
- The update logic is complex enough that you want it described in one tested, pure function rather than scattered across handlers.
- You want to pass a single stable `dispatch` down deep instead of many `setX` callbacks. `dispatch` has a **stable identity** across renders, so it never needs to be in dependency arrays and never breaks `React.memo`.

**Lazy initialization** (the third argument) defers expensive setup to the first render only:

```jsx
function init(initialCount) {
  return { count: initialCount, step: 1 }; // could be an expensive computation
}

function Counter({ initialCount }) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  // init(initialCount) runs once on mount, not on every render
}
```

### Example 5 — `useReducer` + Context (the scalable state pattern)

For app-wide state without a library, combine a reducer with context so any descendant can read state and dispatch actions:

```jsx
import { createContext, useContext, useReducer } from 'react';

const TodosContext = createContext(null);
const TodosDispatchContext = createContext(null);

function todosReducer(todos, action) {
  switch (action.type) {
    case 'added':
      return [...todos, { id: action.id, text: action.text, done: false }];
    case 'toggled':
      return todos.map((t) => (t.id === action.id ? { ...t, done: !t.done } : t));
    default:
      throw new Error('Unknown action: ' + action.type);
  }
}

export function TodosProvider({ children }) {
  const [todos, dispatch] = useReducer(todosReducer, []);
  return (
    <TodosContext.Provider value={todos}>
      <TodosDispatchContext.Provider value={dispatch}>
        {children}
      </TodosDispatchContext.Provider>
    </TodosContext.Provider>
  );
}

// Convenience custom hooks (see 10-custom-hooks.md)
export const useTodos = () => useContext(TodosContext);
export const useTodosDispatch = () => useContext(TodosDispatchContext);
```

Splitting state and dispatch into two contexts means components that only dispatch (a button) don't re-render when the todos list changes.

### Example 6 — `useMemo`: skip an expensive recomputation

`useMemo(factory, deps)` runs `factory` and caches its return value, recomputing only when a dependency changes. Use it to avoid expensive work on every render:

```jsx
import { useMemo, useState } from 'react';

function ProductList({ products }) {
  const [query, setQuery] = useState('');

  // Without useMemo, this filter+sort runs on every keystroke-driven re-render.
  const visible = useMemo(() => {
    return products
      .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
      .sort((a, b) => a.price - b.price);
  }, [products, query]);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{visible.map((p) => <li key={p.id}>{p.name}</li>)}</ul>
    </>
  );
}
```

`useMemo` also provides **referential stability**: the returned object/array keeps the same identity between renders while deps are unchanged. That matters when the value is passed to a `React.memo` child or used as another hook's dependency.

### Example 7 — `useCallback` + `React.memo`

`useCallback(fn, deps)` memoizes a *function identity*. Every render normally creates a brand-new function object. If you pass a fresh function as a prop to a `React.memo` child, the child re-renders anyway because its prop changed by identity. `useCallback` fixes that:

```jsx
import { useCallback, useState, memo } from 'react';

const Child = memo(function Child({ onAdd }) {
  console.log('Child rendered');
  return <button onClick={onAdd}>Add</button>;
});

function Parent() {
  const [items, setItems] = useState([]);
  const [text, setText] = useState('');

  // Stable identity: only changes if its deps change (here: never).
  const handleAdd = useCallback(() => {
    setItems((prev) => [...prev, prev.length]); // functional update → no `items` dep
  }, []);

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <Child onAdd={handleAdd} />
      <p>{items.length} items</p>
    </>
  );
}
```

Typing in the input re-renders `Parent`, but `Child` does **not** re-render because `handleAdd` is referentially stable and `Child` is wrapped in `memo`. Remove the `useCallback` and `Child` re-renders on every keystroke.

The key identity to remember:

```js
useCallback(fn, deps) === useMemo(() => fn, deps);
```

`useCallback` is literally `useMemo` specialized to memoize a function — it returns the function itself rather than the result of *calling* it.

### Example 8 — `useLayoutEffect`: measure before paint

`useLayoutEffect` has the same signature as `useEffect` but fires **synchronously after DOM mutations and before the browser paints**. Use it only when you must read layout (size/position) and synchronously re-render to avoid a visible flicker.

```jsx
import { useState, useRef, useLayoutEffect } from 'react';

function Tooltip({ children }) {
  const ref = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    // Measured before paint, so the user never sees the un-positioned frame.
    setHeight(ref.current.getBoundingClientRect().height);
  }, []);

  return (
    <div ref={ref} style={{ top: -height }}>
      {children}
    </div>
  );
}
```

With a plain `useEffect`, the browser would paint the tooltip at the wrong position for one frame, then correct it — a visible flicker. Because `useLayoutEffect` blocks paint, **overuse hurts performance**; reach for `useEffect` by default (see `08-useeffect-side-effects.md`).

### Example 9 — `useImperativeHandle` + ref forwarding

Sometimes a parent needs to call an *imperative method* on a child (focus an input, play a video). `useImperativeHandle` lets a child customize what its `ref` exposes, hiding the raw DOM node behind a small API. In React 19, `ref` is a regular prop on function components — no `forwardRef` wrapper needed.

```jsx
import { useRef, useImperativeHandle } from 'react';

function FancyInput({ ref, ...props }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ''; },
  }), []);

  return <input ref={inputRef} {...props} />;
}

function Form() {
  const fancyRef = useRef(null);
  return (
    <>
      <FancyInput ref={fancyRef} placeholder="Name" />
      <button onClick={() => fancyRef.current.focus()}>Focus</button>
      <button onClick={() => fancyRef.current.clear()}>Clear</button>
    </>
  );
}
```

The parent can only call `focus()` and `clear()` — it cannot touch the underlying DOM node directly. Treat imperative handles as an escape hatch; prefer props/state when possible.

### Example 10 — `useId` and `useDebugValue`

`useId()` returns a **stable, unique string** that is identical on server and client (so it is SSR/hydration-safe). Use it to link form controls and labels for accessibility — never as a list `key`.

```jsx
import { useId } from 'react';

function PasswordField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Password</label>
      <input id={id} type="password" aria-describedby={`${id}-hint`} />
      <p id={`${id}-hint`}>Must be 8+ characters.</p>
    </>
  );
}
```

`useDebugValue(value, format?)` adds a label to a *custom hook* in React DevTools. It does nothing in production-style usage and should only appear inside custom hooks:

```jsx
import { useSyncExternalStore, useDebugValue } from 'react';

function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  // useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)
  const isOnline = useSyncExternalStore(subscribe, () => navigator.onLine, () => true);
  useDebugValue(isOnline ? 'Online' : 'Offline'); // shown next to the hook in DevTools
  return isOnline;
}
```

The optional second argument is a formatter run lazily only when DevTools inspects the hook — useful when formatting is expensive.

---

## Under the hood

**Hooks are a linked list keyed by call order.** Internally React stores hooks for a component as an ordered list on the "fiber" (React's per-component work unit). On mount it pushes a node per hook. On update it walks a cursor down that same list in lockstep with your function's execution. If render N calls 3 hooks and render N+1 calls 2 (because a condition flipped), the cursor and the list desynchronize — the third hook's state gets read by the second call. React 19 detects this and throws *"Rendered fewer hooks than expected"* / *"Rendered more hooks than expected."* This is the runtime proof of why Rule #1 exists.

**`useState` is implemented on top of `useReducer`.** Conceptually, `useState(init)` is `useReducer((state, action) => typeof action === 'function' ? action(state) : action, init)`. That is why functional updates (`setX(prev => …)`) work and why both share batching behavior.

**`dispatch` and `setState` identities are stable forever.** React guarantees the dispatch function from `useReducer` and the setter from `useState` never change across renders. So you may safely omit them from dependency arrays — the `exhaustive-deps` lint rule knows this.

**`useMemo`/`useCallback` are caches, not guarantees.** React may *throw away* a memoized value to free memory and recompute it later (documented behavior). So your factory must be pure and your code must work correctly even if the value is recomputed. Never rely on `useMemo` for correctness — only as a performance optimization. If you need a value computed *exactly once* and never discarded, that is a ref-initialization pattern, not `useMemo`.

**`useEffect` vs `useLayoutEffect` timing.** After React commits DOM changes:
1. `useLayoutEffect` cleanups + setups run **synchronously**, blocking the browser from painting.
2. The browser paints.
3. `useEffect` cleanups + setups run **asynchronously**, after paint.

So `useLayoutEffect` can read final layout and mutate state before the user sees anything, at the cost of delaying paint. On the server, `useLayoutEffect` does not run and React warns; guard SSR layout reads accordingly.

**Context propagation.** `useContext` subscribes the component to its nearest provider. When the provider's `value` changes (by `Object.is` comparison), *every* consumer re-renders — React does not diff which fields you read. That is why an unstable inline object `value={{ user, setUser }}` forces all consumers to re-render every time the provider re-renders. Memoize the value or split contexts (see Example 5 and `12-context-api.md`).

**Refs update during commit, not render.** `inputRef.current` is set *after* React commits to the DOM, so reading a DOM ref during render gives `null` on the first render. Read DOM refs in effects or event handlers, never in the render body.

---

## Common mistakes & pitfalls

**1. Calling a hook conditionally.**

```jsx
// ❌ WRONG — hook call count changes between renders
function Profile({ userId }) {
  if (userId) {
    const [data, setData] = useState(null); // sometimes called, sometimes not
  }
  // ...
}
```

```jsx
// ✅ RIGHT — hook at top level; branch on the value, not the call
function Profile({ userId }) {
  const [data, setData] = useState(null);
  if (!userId) return <p>No user selected.</p>; // early return is fine AFTER all hooks
  // ...
}
```

The hook must run unconditionally; you may branch on its result or `return` *after* every hook has been called.

**2. Reading a DOM ref during render.**

```jsx
// ❌ WRONG — current is null on first render
function Box() {
  const ref = useRef(null);
  const w = ref.current.offsetWidth; // TypeError: ref.current is null
  return <div ref={ref}>{w}</div>;
}
```

```jsx
// ✅ RIGHT — read it after commit
function Box() {
  const ref = useRef(null);
  useLayoutEffect(() => {
    console.log(ref.current.offsetWidth);
  }, []);
  return <div ref={ref} />;
}
```

**3. Mutating a ref to drive the UI.**

```jsx
// ❌ WRONG — UI never updates; refs don't re-render
function Likes() {
  const count = useRef(0);
  return <button onClick={() => count.current++}>{count.current}</button>;
}
```

```jsx
// ✅ RIGHT — if it's displayed, it's state
function Likes() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```

Rule of thumb: if a value appears in the rendered output, it belongs in state, not a ref.

**4. Memoizing everything ("just in case").**

```jsx
// ❌ WRONG — useMemo costs more than it saves here
const total = useMemo(() => a + b, [a, b]); // trivial math; the hook overhead > the work
```

```jsx
// ✅ RIGHT — just compute it
const total = a + b;
```

`useMemo`/`useCallback` have their own cost (storing deps, comparing them). Use them when the computation is genuinely expensive or when referential stability is *needed* by `React.memo` / a dependency array — not reflexively.

**5. Lying to the dependency array.**

```jsx
// ❌ WRONG — handleAdd captures a stale `items` and never updates
const handleAdd = useCallback(() => {
  setItems([...items, items.length]);
}, []); // items is used but omitted
```

```jsx
// ✅ RIGHT — functional update removes the dependency cleanly
const handleAdd = useCallback(() => {
  setItems((prev) => [...prev, prev.length]);
}, []);
```

Trust `exhaustive-deps`. If a dependency is awkward, fix the code (functional updates, moving the value, splitting the effect) rather than silencing the lint.

---

## Best practices

- **Do** keep all hook calls at the top level, before any early `return`, loop, or condition.
- **Do** install and obey `eslint-plugin-react-hooks` — treat both its rules as build-blocking.
- **Do** use a ref for non-rendered, render-stable values (timers, previous values, instance flags) and for DOM access.
- **Do** prefer `useReducer` when state transitions are complex, interrelated, or worth unit-testing as a pure function.
- **Do** use lazy init (`useState(() => …)` / `useReducer(reducer, arg, init)`) for expensive initial values.
- **Do** memoize a context provider's `value` (or split state/dispatch contexts) to avoid waking every consumer.
- **Do** reach for `useEffect` by default and only escalate to `useLayoutEffect` when you must measure/mutate the DOM before paint.
- **Don't** call hooks from event handlers, plain functions, conditionals, or `try/catch`.
- **Don't** store displayed data in a ref, or expect ref mutations to re-render.
- **Don't** wrap trivial computations in `useMemo`/`useCallback`; profile first, optimize the hot paths.
- **Don't** suppress `exhaustive-deps` to "fix" a bug — that hides stale closures.
- **Don't** use `useId` for list keys; use it only for accessibility id wiring.

---

## Exercises

1. **Render-tracker ref.** Build a `<RenderCounter>` that displays how many times the *parent* it's mounted in has rendered, without itself causing extra renders. Use a ref to count renders and display the count.
2. **`usePrevious` hook.** Write the `usePrevious(value)` custom hook and use it in a counter that shows both the current and previous count.
3. **Reducer-driven form.** Convert a sign-up form with `name`, `email`, and `agreed` fields from three `useState` calls to a single `useReducer`, with a `'reset'` action.
4. **Memoized child.** Given a `Parent` with an unrelated input and a `List` child wrapped in `React.memo`, make the list's `onSelect` handler stable so typing in the input does not re-render the list. Verify with a `console.log`.
5. **Imperative video player (stretch).** Build a `<Video>` component that exposes `play()` and `pause()` via `useImperativeHandle`, and a parent with two buttons that drive it.

### Solutions

**1 — Render-tracker ref**

```jsx
import { useRef } from 'react';

function RenderCounter() {
  const renders = useRef(0);
  renders.current += 1; // mutating a ref during render is OK for a debug counter
  return <small>Parent rendered {renders.current} times</small>;
}
```

**2 — `usePrevious`**

```jsx
import { useState, useRef, useEffect } from 'react';

function usePrevious(value) {
  const ref = useRef(undefined);
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}

function Counter() {
  const [count, setCount] = useState(0);
  const prev = usePrevious(count);
  return (
    <button onClick={() => setCount((c) => c + 1)}>
      now {count}, was {prev ?? '—'}
    </button>
  );
}
```

**3 — Reducer-driven form**

```jsx
import { useReducer } from 'react';

const initial = { name: '', email: '', agreed: false };

function reducer(state, action) {
  switch (action.type) {
    case 'field':
      return { ...state, [action.field]: action.value };
    case 'reset':
      return initial;
    default:
      throw new Error('Unknown action: ' + action.type);
  }
}

function SignUp() {
  const [state, dispatch] = useReducer(reducer, initial);
  const set = (field) => (e) =>
    dispatch({ type: 'field', field, value: e.target.type === 'checkbox' ? e.target.checked : e.target.value });

  return (
    <form onSubmit={(e) => { e.preventDefault(); dispatch({ type: 'reset' }); }}>
      <input value={state.name} onChange={set('name')} placeholder="Name" />
      <input value={state.email} onChange={set('email')} placeholder="Email" />
      <label>
        <input type="checkbox" checked={state.agreed} onChange={set('agreed')} /> I agree
      </label>
      <button type="submit" disabled={!state.agreed}>Sign up</button>
    </form>
  );
}
```

**4 — Memoized child**

```jsx
import { useState, useCallback, memo } from 'react';

const List = memo(function List({ items, onSelect }) {
  console.log('List rendered');
  return (
    <ul>
      {items.map((i) => <li key={i} onClick={() => onSelect(i)}>{i}</li>)}
    </ul>
  );
});

function Parent() {
  const [text, setText] = useState('');
  const items = ['a', 'b', 'c'];
  const onSelect = useCallback((i) => console.log('selected', i), []);
  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <List items={items} onSelect={onSelect} />
    </>
  );
}
```

> Note: for `List` to truly skip re-renders, `items` must also be stable. Move it out of the component or wrap it in `useMemo`, since the array literal above is recreated each render.

**5 — Imperative video player**

```jsx
import { useRef, useImperativeHandle } from 'react';

function Video({ ref, src }) {
  const videoRef = useRef(null);
  useImperativeHandle(ref, () => ({
    play: () => videoRef.current.play(),
    pause: () => videoRef.current.pause(),
  }), []);
  return <video ref={videoRef} src={src} width={320} />;
}

function Player() {
  const playerRef = useRef(null);
  return (
    <>
      <Video ref={playerRef} src="/clip.mp4" />
      <button onClick={() => playerRef.current.play()}>Play</button>
      <button onClick={() => playerRef.current.pause()}>Pause</button>
    </>
  );
}
```

---

## Key takeaways

- React matches hooks to their stored state **by call order**, so hooks must run unconditionally and in the same order every render — that *is* the reason for the Rules of Hooks.
- `eslint-plugin-react-hooks` enforces both rules and `exhaustive-deps`; treat its warnings as errors.
- `useRef` gives you a render-stable, mutation-silent `.current` box — perfect for DOM nodes, timers, previous values, and instance flags; never for displayed data.
- `useReducer` centralizes complex/interrelated state transitions in a pure, testable function and hands you a stable `dispatch`; pair it with context for app-wide state.
- `useMemo` caches a computed value and `useCallback` caches a function identity — `useCallback(fn, deps) === useMemo(() => fn, deps)` — and both are optimizations React may discard, never correctness guarantees.
- `useLayoutEffect` runs synchronously before paint (measure/mutate without flicker) but blocks rendering; default to `useEffect`.
- `useImperativeHandle` exposes a curated imperative API through a ref; `useId` produces SSR-safe accessibility ids; `useDebugValue` labels custom hooks in DevTools.

### Decision guide — which hook for which problem

| You need to… | Reach for |
| --- | --- |
| Track a value shown in the UI | `useState` (see `07`) |
| Run a side effect after render | `useEffect` (see `08`) |
| Hold a value across renders without re-rendering | `useRef` |
| Access or imperatively control a DOM node | `useRef` (+ `useImperativeHandle` to curate the API) |
| Manage complex/interrelated state transitions | `useReducer` |
| Read shared data from above in the tree | `useContext` |
| Avoid recomputing an expensive value | `useMemo` |
| Keep a function/object identity stable for `memo`/deps | `useCallback` / `useMemo` |
| Measure layout & re-render before the browser paints | `useLayoutEffect` |
| Generate a stable, SSR-safe id for a11y | `useId` |
| Label a custom hook in DevTools | `useDebugValue` |

---

## Further reading

- React docs — Rules of Hooks: https://react.dev/reference/rules/rules-of-hooks
- React docs — Built-in hooks reference: https://react.dev/reference/react/hooks
- `useRef`: https://react.dev/reference/react/useRef
- `useContext`: https://react.dev/reference/react/useContext
- `useReducer`: https://react.dev/reference/react/useReducer
- `useMemo`: https://react.dev/reference/react/useMemo
- `useCallback`: https://react.dev/reference/react/useCallback
- `useLayoutEffect`: https://react.dev/reference/react/useLayoutEffect
- `useImperativeHandle`: https://react.dev/reference/react/useImperativeHandle
- `useId`: https://react.dev/reference/react/useId
- `useDebugValue`: https://react.dev/reference/react/useDebugValue
- `eslint-plugin-react-hooks`: https://www.npmjs.com/package/eslint-plugin-react-hooks
- "React hooks: not magic, just arrays" (the call-order mental model): https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e
