# 10 — Custom Hooks

> **What you'll learn:** how to extract reusable *stateful logic* into your own `use*` functions, why they isolate state per call, and how to build a small library of production-grade custom hooks from scratch.

By the end of this lesson you will be able to:

- Explain *why* custom hooks exist and the precise thing they let you reuse (logic, not state).
- Apply the `use*` naming convention and understand why the linter and React itself depend on it.
- Compose hooks — call built-in hooks (and other custom hooks) from inside your own.
- Decide when to return an **array** vs an **object** from a hook, and why.
- Build and reason about real hooks: `useToggle`, `usePrevious`, `useLocalStorage`, `useDebounce`, `useWindowSize`, `useOnClickOutside`, `useFetch`, and `useInterval`.
- Avoid the three classic custom-hook pitfalls: breaking the Rules of Hooks, returning unstable references, and overloading one hook with too many responsibilities.

---

## Prerequisites

This lesson assumes you are comfortable with the material from earlier lessons. You should already know:

- Function components and JSX (see `04-components-and-props.md`).
- `useState` and the rules of state updates (see `07-state-usestate.md`).
- `useRef` for mutable values that don't trigger re-renders (see `09-hooks-deep-dive.md`).
- `useEffect`, dependency arrays, and **cleanup functions** (see `08-useeffect-side-effects.md`).
- `useCallback` / `useMemo` for reference stability (see `09-hooks-deep-dive.md`).
- The **Rules of Hooks** — only call hooks at the top level, only from React functions (introduced in `09-hooks-deep-dive.md`).

If any of those feel shaky, custom hooks will feel like magic in a bad way. They are *not* magic — they are ordinary functions that happen to call the hooks you already know.

---

## Concept

### The problem custom hooks solve

Imagine two components that both need to know the current browser window width. Component A is a responsive navbar; component B is a chart that re-lays-out on resize. Both need:

1. A piece of state holding the width.
2. A `resize` event listener on `window`.
3. Cleanup that removes the listener when the component unmounts.

Before hooks, the only ways to share this were higher-order components or render props — both of which create "wrapper hell" and obscure data flow. With **custom hooks**, you write that logic *once* as a function named `useWindowSize`, and both components call it. The plumbing lives in one place; the components just read a value.

### The single most important idea

> **A custom hook lets you reuse stateful *logic*, not state itself.**

When component A calls `useWindowSize()` and component B calls `useWindowSize()`, they each get their **own independent** `useState` and their **own** effect. There is no shared variable behind the scenes. The hook is a *recipe* that is re-run for each component that uses it. This is the exact opposite of, say, a module-level variable, which would be shared globally.

Analogy: a custom hook is like a **cookie cutter**, not a cookie. Calling the hook stamps out a fresh set of state/effects for the calling component. If you want components to share the *same* state, that's what Context or a store is for (see `12-context-api.md`) — a custom hook alone does not do that.

### A custom hook is *just a function*

There is no special React machinery for declaring a custom hook. The "secret" is two-fold:

1. It is a JavaScript function.
2. Its name starts with `use`, and it calls at least one other hook (built-in or custom).

That's it. React doesn't register your hook anywhere. The reason it works is that hooks are tracked by **call order** within the currently-rendering component, and a custom hook's body executes inline inside that component's render. So when `MyComponent` calls `useToggle()`, the `useState` *inside* `useToggle` is, as far as React is concerned, just another hook call belonging to `MyComponent`.

### Why the `use` prefix matters

The `use` prefix is not decorative. It is the signal that lets:

- **The React ESLint plugin (`eslint-plugin-react-hooks`)** know that this function obeys the Rules of Hooks — so it should be checked for conditional calls, and so the `react-hooks/exhaustive-deps` rule can analyze it.
- **The React Compiler** (the opt-in build-time optimizer recommended with React 19) know the function is a hook and treat its call boundaries correctly when it auto-memoizes.
- **Other developers** know at a glance "this may call hooks, so I must call it like a hook" (top level, not in a loop/condition).

If you name a hook-calling function `getToggle` instead of `useToggle`, the linter will *not* apply the Rules of Hooks to it, and you'll get silent, subtle bugs. Conversely, if you prefix a function with `use` but it calls no hooks, that's harmless but misleading. Rule of thumb: **`use` ⇔ calls hooks.**

---

## Syntax & API

A custom hook has no dedicated API. You compose existing hooks. Here are the building blocks you'll lean on, with signatures:

```js
// State that triggers re-render when changed.
const [state, setState] = useState(initialValue);
const [state, setState] = useState(() => expensiveInit()); // lazy initializer, runs once

// Mutable container that persists across renders, does NOT trigger re-render on .current change.
const ref = useRef(initialValue); // { current: initialValue }

// Run side effects after render; return a cleanup function.
useEffect(() => {
  // effect
  return () => {/* cleanup */};
}, [dep1, dep2]); // re-runs when a dependency changes

// useLayoutEffect: same shape, but fires synchronously before the browser paints.
useLayoutEffect(() => {/* ... */}, [deps]);

// Stable function/value identity across renders.
const fn = useCallback(() => {/* ... */}, [deps]);
const value = useMemo(() => compute(), [deps]);
```

**Custom hook signature pattern** (this is the *only* convention to internalize):

```jsx
function useSomething(arg1, arg2) {
  // 1. Call built-in hooks at the top level (never conditionally).
  const [x, setX] = useState(/* ... */);
  useEffect(() => {/* ... */}, [/* ... */]);

  // 2. Return whatever the consumer needs: a value, a tuple, or an object.
  return x;             // single value
  // return [x, setX];  // tuple (positional, renameable)
  // return { x, setX };// object (named, self-documenting)
}
```

Notes:

- A hook can take any arguments and return anything (a value, an array, an object, even JSX or a function).
- A hook can call other custom hooks. Composition is unlimited as long as every call respects the Rules of Hooks.
- A hook must be called **unconditionally and at the top level** of a component or another hook — never inside `if`, loops, nested functions, or after an early `return`.

---

## Examples

We'll build progressively harder hooks. Every one is fully runnable in a Vite + React 19 project (`.jsx` files, ES modules).

### Example 1 — `useToggle` (the "hello world" of custom hooks)

```jsx
// hooks/useToggle.js
import { useCallback, useState } from 'react';

export function useToggle(initial = false) {
  const [on, setOn] = useState(initial);

  // useCallback keeps `toggle` referentially stable across renders.
  const toggle = useCallback(() => setOn((prev) => !prev), []);
  const setTrue = useCallback(() => setOn(true), []);
  const setFalse = useCallback(() => setOn(false), []);

  return { on, toggle, setTrue, setFalse, setOn };
}
```

```jsx
// components/Sidebar.jsx
import { useToggle } from '../hooks/useToggle';

export default function Sidebar() {
  const { on: isOpen, toggle, setFalse } = useToggle(false);

  return (
    <>
      <button onClick={toggle}>{isOpen ? 'Close' : 'Open'} sidebar</button>
      {isOpen && (
        <aside>
          <button onClick={setFalse}>×</button>
          <nav>…</nav>
        </aside>
      )}
    </>
  );
}
```

Why it matters: the boolean-flip logic and the stable callbacks are extracted once. Note the functional updater `setOn((prev) => !prev)` — it lets `toggle` have an empty dependency array, so its identity never changes (important if you pass it to a memoized child).

### Example 2 — `usePrevious` (a hook that returns a value, using only a ref)

Sometimes you need the value a prop/state had on the **previous** render — for animations, "changed from X to Y" logic, or diffing.

```jsx
// hooks/usePrevious.js
import { useEffect, useRef } from 'react';

export function usePrevious(value) {
  const ref = useRef(undefined);

  // The effect runs AFTER render commits, so during render `ref.current`
  // still holds the value from the previous render.
  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}
```

```jsx
// components/Counter.jsx
import { useState } from 'react';
import { usePrevious } from '../hooks/usePrevious';

export default function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <p onClick={() => setCount((c) => c + 1)}>
      Now: {count}, before: {prevCount ?? '—'}
    </p>
  );
}
```

The trick: because effects run *after* the render that reads `ref.current`, the value lags by exactly one render. This is the canonical "store last value in a ref" pattern.

### Example 3 — `useLocalStorage` (sync state to `localStorage`, lazy init, JSON, error handling)

This is where custom hooks start to earn their keep. We want `useState`-like ergonomics, but persisted across page reloads.

```jsx
// hooks/useLocalStorage.js
import { useCallback, useEffect, useState } from 'react';

export function useLocalStorage(key, initialValue) {
  // Lazy initializer: read from storage ONCE on first render, not every render.
  const [value, setValue] = useState(() => {
    try {
      const raw = window.localStorage.getItem(key);
      return raw !== null ? JSON.parse(raw) : initialValue;
    } catch (err) {
      // Corrupt JSON, or storage disabled (private mode / quota): fall back.
      console.warn(`useLocalStorage: failed to read "${key}"`, err);
      return initialValue;
    }
  });

  // Accept either a value or an updater function, mirroring useState's API.
  const setStoredValue = useCallback(
    (next) => {
      setValue((prev) => (typeof next === 'function' ? next(prev) : next));
    },
    []
  );

  // Persist to localStorage whenever value or key changes.
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (err) {
      console.warn(`useLocalStorage: failed to write "${key}"`, err);
    }
  }, [key, value]);

  // Bonus: sync across browser tabs via the `storage` event.
  useEffect(() => {
    function handleStorage(event) {
      if (event.key === key && event.newValue !== null) {
        try {
          setValue(JSON.parse(event.newValue));
        } catch {
          /* ignore malformed cross-tab payloads */
        }
      }
    }
    window.addEventListener('storage', handleStorage);
    return () => window.removeEventListener('storage', handleStorage);
  }, [key]);

  return [value, setStoredValue];
}
```

```jsx
// components/ThemeToggle.jsx
import { useLocalStorage } from '../hooks/useLocalStorage';

export default function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  return (
    <button onClick={() => setTheme((t) => (t === 'light' ? 'dark' : 'light'))}>
      Theme: {theme}
    </button>
  );
}
```

Key design choices:

- **Lazy initializer** (`useState(() => …)`): reading `localStorage` is synchronous I/O; we only want to do it once, so we pass a function rather than calling `getItem` on every render.
- **`try/catch` everywhere**: `JSON.parse` throws on corrupt data; `localStorage` throws when disabled or over quota. A persistence hook that crashes the app is worse than one that silently degrades.
- **Updater-function support**: by checking `typeof next === 'function'`, we keep the same API as `useState`, so consumers can write `setTheme((t) => …)`.
- We return an **array** so callers can name the pair whatever they want (`theme`, `setTheme`).

### Example 4 — `useDebounce` (debounce a fast-changing value)

Debouncing delays reacting to a value until it has stopped changing for `delay` ms — perfect for search-as-you-type.

```jsx
// hooks/useDebounce.js
import { useEffect, useState } from 'react';

export function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    // Cleanup cancels the pending timeout if `value` changes before it fires.
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}
```

```jsx
// components/Search.jsx
import { useEffect, useState } from 'react';
import { useDebounce } from '../hooks/useDebounce';

export default function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 400);

  useEffect(() => {
    if (!debouncedQuery) return;
    // Fire the network request only after typing pauses for 400ms.
    console.log('searching for', debouncedQuery);
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search…"
    />
  );
}
```

The whole mechanism is one effect with a `clearTimeout` cleanup. Each keystroke replaces the value, which re-runs the effect, which cancels the previous timer and starts a new one. Only the *last* timer survives long enough to fire.

### Example 5 — `useWindowSize` (event listener + cleanup)

```jsx
// hooks/useWindowSize.js
import { useEffect, useState } from 'react';

export function useWindowSize() {
  const [size, setSize] = useState(() => ({
    width: window.innerWidth,
    height: window.innerHeight,
  }));

  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    window.addEventListener('resize', handleResize);
    handleResize(); // sync once in case size changed before listener attached
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}
```

```jsx
// components/Responsive.jsx
import { useWindowSize } from '../hooks/useWindowSize';

export default function Responsive() {
  const { width } = useWindowSize();
  return <p>{width < 768 ? 'Mobile layout' : 'Desktop layout'}</p>;
}
```

Note the **cleanup** removing the listener on unmount — without it you'd leak listeners every time the component remounts. The empty dependency array `[]` is correct here because `handleResize` reads live `window` values each time it fires, so it never needs to be recreated.

### Example 6 — `useOnClickOutside` (ref + document listener)

Closes a dropdown/modal when the user clicks anywhere outside it.

```jsx
// hooks/useOnClickOutside.js
import { useEffect } from 'react';

export function useOnClickOutside(ref, handler) {
  useEffect(() => {
    function listener(event) {
      // Do nothing if clicking the ref's element or its descendants.
      if (!ref.current || ref.current.contains(event.target)) return;
      handler(event);
    }
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}
```

```jsx
// components/Dropdown.jsx
import { useCallback, useRef, useState } from 'react';
import { useOnClickOutside } from '../hooks/useOnClickOutside';

export default function Dropdown() {
  const [open, setOpen] = useState(false);
  const ref = useRef(null);

  // Stabilize the handler so the effect doesn't re-subscribe on every render.
  const close = useCallback(() => setOpen(false), []);
  useOnClickOutside(ref, close);

  return (
    <div ref={ref} style={{ position: 'relative' }}>
      <button onClick={() => setOpen((o) => !o)}>Menu</button>
      {open && <ul className="menu">…</ul>}
    </div>
  );
}
```

Here the hook *receives* a ref from the caller rather than creating it, because the caller needs that same ref to attach to the DOM node. We wrap `close` in `useCallback` so the `handler` dependency is stable and the effect doesn't re-subscribe each render.

### Example 7 — `useFetch` (loading / data / error + abort)

A realistic data-fetching hook with cancellation to avoid race conditions and "set state on unmounted component" warnings.

```jsx
// hooks/useFetch.js
import { useEffect, useState } from 'react';

export function useFetch(url, options) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { ...options, signal: controller.signal })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((json) => {
        setData(json);
        setLoading(false);
      })
      .catch((err) => {
        if (err.name === 'AbortError') return; // expected on cleanup; ignore
        setError(err);
        setLoading(false);
      });

    // Abort the in-flight request if `url` changes or the component unmounts.
    return () => controller.abort();
  }, [url]); // intentionally not depending on `options` (see note below)

  return { data, error, loading };
}
```

```jsx
// components/User.jsx
import { useFetch } from '../hooks/useFetch';

export default function User({ id }) {
  const { data, error, loading } = useFetch(`/api/users/${id}`);

  if (loading) return <p>Loading…</p>;
  if (error) return <p>Error: {error.message}</p>;
  return <h2>{data.name}</h2>;
}
```

The `AbortController` is the linchpin: if `id` changes rapidly, each new request aborts the previous one, so a slow earlier response can't overwrite a faster later one (the classic race condition). We deliberately omit `options` from the dependency array because callers typically pass a fresh object literal each render, which would refetch infinitely — a real-world note we'll revisit in Pitfalls. For serious apps, prefer a library like TanStack Query (see `15-data-fetching.md`); this hook is for understanding the mechanics.

### Example 8 — `useInterval` (the Dan Abramov ref pattern)

`setInterval` plus React state has a famous trap: the callback you pass to `setInterval` captures the state value from the render it was created in (a stale closure). Dan Abramov's pattern stores the latest callback in a ref so the interval always calls the freshest version, while keeping the interval itself stable.

```jsx
// hooks/useInterval.js
import { useEffect, useRef } from 'react';

export function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  // Always keep the ref pointing at the latest callback, every render.
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  // Set up the interval once; it reads `savedCallback.current` each tick.
  useEffect(() => {
    if (delay === null) return; // pass null to pause
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

```jsx
// components/Clock.jsx
import { useState } from 'react';
import { useInterval } from '../hooks/useInterval';

export default function Clock() {
  const [count, setCount] = useState(0);
  const [running, setRunning] = useState(true);

  // No stale closure: the ref always holds the latest callback.
  useInterval(() => setCount((c) => c + 1), running ? 1000 : null);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setRunning((r) => !r)}>
        {running ? 'Pause' : 'Resume'}
      </button>
    </>
  );
}
```

Why two effects? The first keeps `savedCallback.current` fresh on every render *without* tearing down the interval. The second creates the interval only when `delay` changes (including `null` → pause). The interval's tick reads `savedCallback.current`, so it always runs the latest logic. Passing `delay = null` cleanly pauses by skipping interval creation.

---

## Under the hood

### How React tracks hooks inside a custom hook

React maintains, for each component instance (each fiber), an ordered linked list of "hook slots." Every `useState`, `useEffect`, `useRef`, etc. call — *wherever it physically appears in the code* — consumes the next slot in order on each render. A custom hook's body runs **inline** during the calling component's render, so its hook calls simply append to that same list.

This is why **call order must be identical on every render**. If `useToggle` is called conditionally, the slots shift and React reads the wrong state into the wrong hook. That's the deep reason behind Rule of Hooks #1 (see `09-hooks-deep-dive.md`). A custom hook doesn't get its own "namespace" of slots — there's one flat list per component.

### Why each call is isolated

Because the slots belong to the *component instance*, calling `useWindowSize()` in component A and component B allocates slots in A's fiber and B's fiber respectively — totally separate memory. There is no module-level state. This is the mechanical reason "custom hooks reuse logic, not state."

### Identity / stability and the React Compiler

Functions and objects you create inside a hook are **new on every render** unless wrapped in `useCallback`/`useMemo` or stored in a `useRef`. If a hook returns an object literal `return { data, error }`, that object is a new reference each render — which matters when consumers put it in a dependency array or pass it to `React.memo` children.

The **React Compiler** (a separate, opt-in build-time tool — the `babel-plugin-react-compiler` package, recommended alongside React 19) can auto-memoize much of this, reducing the need for manual `useCallback`/`useMemo`. But you have to add it to your build yourself, and it still relies on the `use*` naming and the Rules of Hooks to identify hooks correctly. Until you've confirmed the compiler is enabled, write hooks as if it isn't: stabilize the references you intend consumers to depend on.

### Render-time vs commit-time

A subtle mental model: the *body* of your hook runs during **render** (must be pure — no DOM reads/writes, no subscriptions). The `useEffect`/`useLayoutEffect` callbacks run during **commit** (after the DOM is updated). `usePrevious` exploits this gap: it reads `ref.current` during render, then updates it in an effect during commit, producing a one-render lag.

### `useEffectEvent` (the modern successor to the ref pattern)

The `useInterval` "save callback in a ref" pattern exists to read the latest value without re-subscribing. React provides a hook, `useEffectEvent` (imported from `'react'`), designed exactly for this — extracting non-reactive logic out of effects into an "Effect Event" so you don't list it as a dependency. It shipped as **stable in React 19.2** (October 2025); the `eslint-plugin-react-hooks` linter knows to keep Effect Events out of dependency arrays. The ref pattern shown above remains a perfectly valid, fully portable approach (and is what you'll see in most existing codebases), but on React 19.2+ you can reach for `useEffectEvent` directly. Note that an Effect Event must be declared inside the component or hook that uses it, may only be called from inside an effect, and must not be passed to other components or hooks as a value.

---

## Common mistakes & pitfalls

### Pitfall 1 — Breaking the Rules of Hooks (conditional calls)

**WRONG:**

```jsx
function Profile({ userId }) {
  if (!userId) return <p>No user</p>;
  // ❌ Hook called after an early return — call order changes between renders.
  const { data } = useFetch(`/api/users/${userId}`);
  return <h2>{data?.name}</h2>;
}
```

**RIGHT:**

```jsx
function Profile({ userId }) {
  // ✅ Hook is always called, unconditionally, at the top.
  const { data, loading } = useFetch(userId ? `/api/users/${userId}` : null);
  if (!userId) return <p>No user</p>;
  if (loading) return <p>Loading…</p>;
  return <h2>{data?.name}</h2>;
}
```

Hooks (including custom ones) must be called in the same order every render. Move the call above any conditional return, and handle the "off" state via the hook's arguments (e.g., a `null` URL that the hook short-circuits).

### Pitfall 2 — Returning unstable references

**WRONG:**

```jsx
function useUser(id) {
  const [user, setUser] = useState(null);
  // ❌ New object every render → consumers' effects/memo re-run constantly.
  const actions = { reload: () => fetchUser(id).then(setUser) };
  return { user, actions };
}
```

**RIGHT:**

```jsx
import { useCallback, useMemo, useState } from 'react';

function useUser(id) {
  const [user, setUser] = useState(null);
  const reload = useCallback(() => fetchUser(id).then(setUser), [id]);
  // ✅ `actions` keeps a stable identity unless `reload` changes.
  const actions = useMemo(() => ({ reload }), [reload]);
  return { user, actions };
}
```

If consumers depend on your returned functions/objects (in `useEffect` deps, `React.memo`, etc.), wrap them so their identity is stable. Setter functions from `useState` are already stable — you never need to memoize those.

### Pitfall 3 — Doing too much in one hook

**WRONG:**

```jsx
// ❌ Fetches data, manages a modal, syncs to localStorage, AND tracks scroll.
function useDashboard() { /* 200 lines of unrelated concerns */ }
```

**RIGHT:**

```jsx
// ✅ Small, focused, composable hooks. Compose them in the component.
function Dashboard() {
  const { data } = useFetch('/api/dashboard');
  const modal = useToggle(false);
  const [layout, setLayout] = useLocalStorage('layout', 'grid');
  // …
}
```

Each hook should do one thing. Small hooks compose; god-hooks don't. If a hook's name needs "and" in it, split it.

### Pitfall 4 — Missing effect cleanup (listener / timer leaks)

**WRONG:**

```jsx
useEffect(() => {
  window.addEventListener('resize', handleResize); // ❌ never removed
}, []);
```

**RIGHT:**

```jsx
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize); // ✅
}, []);
```

Any subscription, timer, or controller created in an effect must be torn down in the returned cleanup — otherwise you leak across remounts and trip React 19 Strict Mode's double-invoke check (see `08-useeffect-side-effects.md`).

### Pitfall 5 — Creating a ref the caller actually needs

If a hook needs to attach to a DOM node the *consumer* renders (like `useOnClickOutside`), accept the ref as an argument rather than creating it internally — otherwise the consumer can't put it on the element.

---

## Best practices

- **Do** prefix every hook with `use` and call hooks unconditionally at the top level.
- **Do** keep each hook focused on a single concern; compose small hooks instead of building god-hooks.
- **Do** return an **array** when the values are positional and order is obvious (`[value, setValue]`) so callers can rename freely; return an **object** when there are 3+ values or they're not naturally ordered (`{ data, error, loading }`) for self-documenting, optional destructuring.
- **Do** clean up every subscription, timer, listener, and `AbortController` in the effect's cleanup.
- **Do** stabilize returned functions/objects with `useCallback`/`useMemo` (or a ref) when consumers may depend on their identity.
- **Do** use a **lazy initializer** for any expensive or I/O-bound initial state (`useState(() => …)`).
- **Don't** call hooks inside loops, conditions, nested functions, or after early returns.
- **Don't** assume two components calling the same hook share state — they don't.
- **Don't** name a function `use*` unless it calls hooks, and don't call a hook from a plain non-React function.
- **Don't** prematurely extract; wait until logic is genuinely reused or the component is getting cluttered.

---

## Exercises

1. **`useCounter`** — Build `useCounter(initial = 0, { min, max, step = 1 } = {})` returning `{ count, increment, decrement, reset, set }`. `increment`/`decrement` must clamp to `min`/`max` when provided, and all callbacks must be referentially stable.

2. **`useHover`** — Build a hook returning `[ref, isHovered]` such that attaching `ref` to any element tracks whether the pointer is over it via `mouseenter`/`mouseleave`. Ensure listeners are cleaned up and re-attached if the node changes.

3. **`useMediaQuery`** — Build `useMediaQuery(query)` returning a boolean that reflects whether a CSS media query (e.g. `'(min-width: 768px)'`) currently matches, using `window.matchMedia` and its `change` event, with proper cleanup and a sensible SSR-safe initial value.

4. **`useDebouncedCallback`** *(harder)* — Instead of debouncing a value (like `useDebounce`), debounce a *function*. Return a stable debounced wrapper such that calling it repeatedly only invokes the underlying callback once, `delay` ms after the last call. Cancel any pending call on unmount.

### Solutions

```jsx
// 1. useCounter
import { useCallback, useState } from 'react';

export function useCounter(initial = 0, { min, max, step = 1 } = {}) {
  const clamp = useCallback(
    (n) => {
      let v = n;
      if (typeof min === 'number') v = Math.max(min, v);
      if (typeof max === 'number') v = Math.min(max, v);
      return v;
    },
    [min, max]
  );

  const [count, setCount] = useState(() => clamp(initial));

  const increment = useCallback(() => setCount((c) => clamp(c + step)), [clamp, step]);
  const decrement = useCallback(() => setCount((c) => clamp(c - step)), [clamp, step]);
  const reset = useCallback(() => setCount(clamp(initial)), [clamp, initial]);
  const set = useCallback((n) => setCount((c) => clamp(typeof n === 'function' ? n(c) : n)), [clamp]);

  return { count, increment, decrement, reset, set };
}
```

```jsx
// 2. useHover
import { useCallback, useEffect, useRef, useState } from 'react';

export function useHover() {
  const [hovered, setHovered] = useState(false);
  const ref = useRef(null);

  const callbackRef = useCallback((node) => {
    // We use a callback ref so we re-bind when the node changes.
    ref.current = node;
  }, []);

  useEffect(() => {
    const node = ref.current;
    if (!node) return;
    const enter = () => setHovered(true);
    const leave = () => setHovered(false);
    node.addEventListener('mouseenter', enter);
    node.addEventListener('mouseleave', leave);
    return () => {
      node.removeEventListener('mouseenter', enter);
      node.removeEventListener('mouseleave', leave);
    };
  }); // no deps: re-bind each render so a swapped node stays tracked

  return [callbackRef, hovered];
}
```

```jsx
// 3. useMediaQuery
import { useEffect, useState } from 'react';

export function useMediaQuery(query) {
  const [matches, setMatches] = useState(() =>
    typeof window !== 'undefined' && 'matchMedia' in window
      ? window.matchMedia(query).matches
      : false
  );

  useEffect(() => {
    const mql = window.matchMedia(query);
    const onChange = (e) => setMatches(e.matches);
    setMatches(mql.matches); // sync in case it changed between render and effect
    mql.addEventListener('change', onChange);
    return () => mql.removeEventListener('change', onChange);
  }, [query]);

  return matches;
}
```

```jsx
// 4. useDebouncedCallback
import { useCallback, useEffect, useRef } from 'react';

export function useDebouncedCallback(callback, delay = 300) {
  const callbackRef = useRef(callback);
  const timeoutRef = useRef(null);

  useEffect(() => {
    callbackRef.current = callback; // keep latest callback (avoid stale closure)
  }, [callback]);

  // Cancel any pending timer on unmount.
  useEffect(() => () => clearTimeout(timeoutRef.current), []);

  return useCallback(
    (...args) => {
      clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => callbackRef.current(...args), delay);
    },
    [delay]
  );
}
```

---

## Key takeaways

- A custom hook is **just a function** named `use*` that calls other hooks — there's no special React machinery.
- It reuses stateful **logic, not state**: every call gets its own isolated `useState`/`useEffect` slots in the calling component's fiber.
- The `use` prefix is load-bearing — it's how the linter, the React Compiler, and your teammates know the Rules of Hooks apply.
- Hooks must be called **unconditionally at the top level**; this follows directly from how React tracks hooks by call order.
- **Return an array** for positional pairs (`[value, setter]`), **an object** for 3+ or named values (`{ data, error, loading }`).
- Always **clean up** listeners, timers, and `AbortController`s in the effect's cleanup return.
- **Stabilize** returned functions/objects with `useCallback`/`useMemo`/`useRef` when consumers depend on their identity.
- Keep hooks **small and single-purpose**; the "store latest callback in a ref" pattern (`useInterval`) avoids stale closures for subscription-style effects.

---

## Further reading

- React docs — **Reusing Logic with Custom Hooks**: <https://react.dev/learn/reusing-logic-with-custom-hooks>
- React docs — **Rules of Hooks**: <https://react.dev/reference/rules/rules-of-hooks>
- React docs — **`useState`**, **`useEffect`**, **`useRef`**, **`useCallback`**, **`useMemo`** reference: <https://react.dev/reference/react/hooks>
- React docs — **Removing Effect Dependencies** (Effect Events, the ref pattern): <https://react.dev/learn/removing-effect-dependencies>
- React docs — **`useEffectEvent`** reference (stable since React 19.2): <https://react.dev/reference/react/useEffectEvent>
- React docs — **React Compiler**: <https://react.dev/learn/react-compiler>
- Dan Abramov — **Making setInterval Declarative with React Hooks**: <https://overreacted.io/making-setinterval-declarative-with-react-hooks/>
- **usehooks-ts** (well-typed reference implementations of these hooks): <https://usehooks-ts.com/>
- `eslint-plugin-react-hooks` (the `rules-of-hooks` and `exhaustive-deps` lint rules): <https://www.npmjs.com/package/eslint-plugin-react-hooks>
