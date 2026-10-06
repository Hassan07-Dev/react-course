# 08 — useEffect & Side Effects

> **What you'll learn:** how to safely connect your React components to the world outside React — the network, the DOM, timers, and subscriptions — using `useEffect`, and just as importantly, when *not* to reach for it.

By the end of this lesson you will be able to:

- Define what a *side effect* is and recognize it in your own code.
- Use `useEffect(setup, deps)` correctly with all three dependency-array forms (omitted, `[]`, `[a, b]`).
- Write **cleanup functions** that prevent leaks, stale subscriptions, and race conditions.
- Reason about **effect timing** relative to browser paint, and survive **StrictMode**'s deliberate double-invocation in development.
- Fetch data with proper `loading` / `error` state and an `AbortController` to cancel stale requests.
- Obey the **exhaustive-deps** ESLint rule and understand the bugs that appear when you lie about dependencies.
- Decide when you **don't need an effect at all**, and what to do instead.
- Map `useEffect` onto the old class lifecycle methods so you can read legacy code.

---

## Prerequisites

Before this lesson you should be comfortable with:

- **JSX and rendering** (see `03-jsx-in-depth.md`) — what "a render" actually is.
- **Components, props, and composition** (see `04-components-and-props.md`).
- **`useState` and state updates** (see `07-state-usestate.md`) — effects almost always read or set state.
- **Rules of Hooks** (see `09-hooks-deep-dive.md`) — `useEffect` is a hook and obeys the same call-order rules.

You should also have a Vite + React 19 project running so you can paste the examples in:

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

`useLayoutEffect` and the rendering-vs-commit timeline are deep-dived in `09-hooks-deep-dive.md`; we touch them here only briefly.

---

## Concept

React's core job is **pure rendering**: given props and state, produce JSX. A pure render must not touch anything outside React — no network calls, no `document.title = ...`, no `setInterval`, no `console.log`-to-an-analytics-server. React may call your component function many times, skip calls, throw them away, or run them twice in development, so anything with an observable effect on the outside world must happen *outside* render.

A **side effect** is exactly that: any operation that reaches beyond computing the return value. Concretely:

- **Data fetching** (`fetch`, `axios`, a WebSocket handshake).
- **Subscriptions** (`addEventListener`, a store subscription, an `IntersectionObserver`).
- **Timers** (`setTimeout`, `setInterval`).
- **Imperative DOM mutation** that React doesn't own (focusing an input, measuring a node, integrating a non-React chart library).
- **Logging / analytics** that hit an external system.

The mental model: a component renders to describe *what the UI should look like*. An **effect describes how to synchronize your component with an external system** after React has put that UI on the screen. The word *synchronize* is deliberate — `useEffect` is best understood not as "run this code after render" but as "keep this external thing in step with my component's current state, and tear it down when it no longer applies."

Here's the analogy. Imagine you move into a new apartment (mount). You set up utilities — electricity, internet (your effect's *setup*). When you move out (unmount), you cancel them (the *cleanup*). And if you move to a different apartment (a dependency changed), you cancel the old utilities *and* set up new ones at the new place. You never want two electricity accounts running for an apartment you left — that's a leak, and `useEffect` cleanup is how you avoid it.

---

## Syntax & API

```jsx
import { useEffect } from 'react';

useEffect(setup, dependencies?);
```

- **`setup`** — a function React runs after committing the render to the DOM. It may optionally **return a cleanup function**. React runs cleanup before the next `setup` and once more when the component unmounts.
- **`dependencies`** (optional) — an array of *reactive values* (props, state, and values derived from them) the effect reads. React compares each item to its previous value with `Object.is`. If any changed, it re-runs the effect (cleanup → setup).

The three forms — memorize these, they are the whole game:

```jsx
// 1. No dependency array → setup runs after EVERY render (commit).
useEffect(() => {
  console.log('runs after every render');
});

// 2. Empty array → setup runs ONCE after the initial mount (+ cleanup on unmount).
useEffect(() => {
  console.log('runs once after mount');
  return () => console.log('runs once on unmount');
}, []);

// 3. With deps → runs after mount, then again whenever a, b, or c changes by Object.is.
useEffect(() => {
  console.log('runs when a, b, or c changes');
}, [a, b, c]);
```

The cleanup signature:

```jsx
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id); // ← cleanup
}, []);
```

`useEffect` returns nothing. The `setup` function must be synchronous and must **not** be `async` directly (an `async` function returns a Promise, which React would mistake for a cleanup function). Define an `async` helper *inside* and call it:

```jsx
useEffect(() => {
  async function load() { /* await ... */ }
  load();
}, []);
```

---

## Examples

### Example 1 — Sync the document title (the "hello world" of effects)

```jsx
import { useState, useEffect } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Clicked ${count} times`;
  }, [count]); // re-sync the title whenever count changes

  return <button onClick={() => setCount(c => c + 1)}>Clicked {count}</button>;
}
```

`document.title` is a side effect — it mutates something React doesn't own. We list `count` as a dependency because the effect *reads* it; whenever `count` changes the title is re-synchronized.

### Example 2 — A timer with cleanup

```jsx
import { useState, useEffect } from 'react';

export default function Stopwatch() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setSeconds(s => s + 1); // functional update — no need to depend on `seconds`
    }, 1000);
    return () => clearInterval(id); // cleanup: stop the old interval
  }, []); // set up once

  return <p>{seconds}s elapsed</p>;
}
```

Note the **functional updater** `s => s + 1`. If we wrote `setSeconds(seconds + 1)` we'd need `seconds` in the deps, which would tear down and recreate the interval every second. Using the updater form keeps the effect's dependency list empty *honestly*.

### Example 3 — Window event subscription

```jsx
import { useState, useEffect } from 'react';

export default function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() { setWidth(window.innerWidth); }
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize); // unsubscribe
  }, []);

  return <p>Window width: {width}px</p>;
}
```

The cleanup removes *the exact same function reference* we added. Forgetting this leaks a listener on every re-subscribe and keeps the component's closure alive after unmount.

### Example 4 — Data fetching with loading, error, and an AbortController

This is the example that matters most in real apps. We fetch a user by `userId`; when `userId` changes mid-flight, we must cancel the in-flight request so a slow earlier response can't overwrite a faster later one (a **race condition**).

```jsx
import { useState, useEffect } from 'react';

export default function UserCard({ userId }) {
  const [user, setUser] = useState(null);
  const [status, setStatus] = useState('idle'); // 'idle' | 'loading' | 'success' | 'error'
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setStatus('loading');
    setError(null);

    async function load() {
      try {
        const res = await fetch(`https://api.example.com/users/${userId}`, {
          signal: controller.signal,
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        setUser(data);
        setStatus('success');
      } catch (err) {
        if (err.name === 'AbortError') return; // expected on cancel — ignore
        setError(err);
        setStatus('error');
      }
    }

    load();
    return () => controller.abort(); // cancel the in-flight request
  }, [userId]); // re-fetch whenever userId changes

  if (status === 'loading' || status === 'idle') return <p>Loading…</p>;
  if (status === 'error') return <p role="alert">Error: {error.message}</p>;
  return <h2>{user.name}</h2>;
}
```

Why each piece exists:

- **`[userId]`** — the effect synchronizes the displayed user with the current `userId`. Change the id, re-run.
- **`AbortController` + `controller.abort()` in cleanup** — when `userId` changes, React runs cleanup *before* the next setup, aborting the previous fetch. The aborted fetch rejects with an `AbortError`, which we deliberately swallow.
- **The race-condition guard** — even without `AbortController`, an alternative is a boolean flag: `let ignore = false; ... if (!ignore) setUser(data); ... return () => { ignore = true; };`. `AbortController` is stronger because it also stops the network work, but the `ignore` flag is the minimum viable fix and worth knowing.

> In production you'd usually reach for a data library — TanStack Query, SWR, or a framework loader — which wraps all of this (caching, dedup, retries) for you. But you must understand the manual version first; it's covered further in `15-data-fetching.md`.

### Example 5 — Subscribing to an external store, the modern way

For subscriptions to *external* stores, React 19 prefers `useSyncExternalStore` over a raw effect (it avoids tearing during concurrent rendering). Mentioned here for completeness; deep-dived in `16-state-management.md`. A raw-effect subscription is still fine for DOM events as in Example 3.

---

## Under the hood

**Where effects sit in the render lifecycle.** A React update has three conceptual phases:

1. **Render phase** — React calls your component function to compute the new tree. Pure, no DOM writes, can be interrupted/retried.
2. **Commit phase** — React applies the diff to the real DOM.
3. **After commit** — the browser paints, then React flushes your `useEffect` callbacks **asynchronously, after paint**.

That last point is the key timing fact: **`useEffect` runs after the browser has painted.** The user sees the new frame *before* your effect fires. This is what makes effects feel "non-blocking" — a heavy fetch in an effect won't delay the visible update. The trade-off: if your effect *synchronously* mutates layout (e.g. measuring a node and repositioning it), the user may see a flicker of the pre-effect state. That is precisely the job of **`useLayoutEffect`**, which fires *before* paint and blocks it — covered in `09-hooks-deep-dive.md`. Rule of thumb: reach for `useEffect` by default; use `useLayoutEffect` only when you must read/write layout before the user sees the frame.

**When cleanup runs, precisely.** For an effect with dependencies, on each re-run React runs the *previous* render's cleanup **before** running the new setup. On unmount it runs the last cleanup. The ordering for a dependency change is always: `cleanup(old) → setup(new)`. This is why the apartment analogy holds — you cancel the old utilities before signing up for the new.

**StrictMode double-invocation.** In development, when you wrap your app in `<StrictMode>`, React intentionally **mounts, unmounts, and remounts** every component once on initial mount. The visible consequence: your effect's setup runs, then cleanup runs, then setup runs again — so you see effects fire **twice** in dev. This is not a bug to suppress; it's a smoke test. If your effect breaks (double timers, doubled subscriptions, doubled fetches that don't cancel) when run twice, it has a latent cleanup bug that would also bite in production during fast navigation or React's own remounting. **The fix is never to disable StrictMode — it's to write a correct cleanup.** All the examples above already pass this test: Example 2 clears its interval, Example 3 removes its listener, Example 4 aborts its fetch. In production, StrictMode does *not* double-invoke; the behavior is dev-only.

```jsx
// main.jsx — StrictMode is on by default in a fresh Vite template
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**Dependency comparison.** React stores the previous deps array and compares element-by-element with `Object.is`. `Object.is(NaN, NaN)` is `true` and `Object.is(0, -0)` is `false`, but the practical gotcha is **reference identity for objects, arrays, and functions**: a new object literal `{}` is `!== ` the previous one even if shallowly equal, so putting an inline object/array/function in deps makes the effect run every render. Stabilize such values with `useMemo` / `useCallback` (see `09-hooks-deep-dive.md`) or, better, depend on primitives.

**Closures capture the render they were created in.** Each render produces a fresh `setup` function that closes over *that render's* props and state. An effect with `[]` captures the *first* render's values forever. This is the single biggest source of "stale closure" bugs (see the pitfalls below) and the entire reason the exhaustive-deps rule exists.

---

## Common mistakes & pitfalls

### Pitfall 1 — Lying about dependencies (the stale closure)

```jsx
// ❌ WRONG: empty deps, but the effect reads `count`
useEffect(() => {
  const id = setInterval(() => {
    console.log(count);     // always logs 0 — captured from the first render
    setCount(count + 1);    // 0 + 1 every time → stuck at 1
  }, 1000);
  return () => clearInterval(id);
}, []); // ← lie: count is used but not listed
```

```jsx
// ✅ RIGHT: use a functional updater so you don't need `count` as a dep
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);   // reads the latest state via the updater
  }, 1000);
  return () => clearInterval(id);
}, []); // honest: the effect no longer reads count directly
```

The `[]` froze the closure on render #1, so `count` was permanently `0`. The fix removes the *need* to read `count`, keeping deps honestly empty.

### Pitfall 2 — Async function passed directly to useEffect

```jsx
// ❌ WRONG: setup is async → returns a Promise, which React treats as cleanup
useEffect(async () => {
  const data = await fetchThing();
  setThing(data);
}, []);
```

```jsx
// ✅ RIGHT: define an async function inside and call it
useEffect(() => {
  let ignore = false;
  (async () => {
    const data = await fetchThing();
    if (!ignore) setThing(data);
  })();
  return () => { ignore = true; };
}, []);
```

### Pitfall 3 — Missing cleanup leaks listeners/timers

```jsx
// ❌ WRONG: subscribes again on every relevant change, never unsubscribes
useEffect(() => {
  window.addEventListener('scroll', onScroll);
}); // no deps, no cleanup → a new listener every single render
```

```jsx
// ✅ RIGHT
useEffect(() => {
  window.addEventListener('scroll', onScroll);
  return () => window.removeEventListener('scroll', onScroll);
}, [onScroll]); // and stabilize onScroll with useCallback if it's defined in render
```

### Pitfall 4 — Race condition on rapid prop changes

```jsx
// ❌ WRONG: a slow earlier response can overwrite a fast later one
useEffect(() => {
  fetch(`/api/items/${id}`).then(r => r.json()).then(setItem);
}, [id]); // no cancellation → out-of-order results
```

```jsx
// ✅ RIGHT: ignore stale responses (or use AbortController as in Example 4)
useEffect(() => {
  let ignore = false;
  fetch(`/api/items/${id}`).then(r => r.json()).then(data => {
    if (!ignore) setItem(data);
  });
  return () => { ignore = true; };
}, [id]);
```

### Pitfall 5 — Using an effect to transform data for render (you don't need an effect)

```jsx
// ❌ WRONG: derived state synced via an effect — extra render, can go stale
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);
```

```jsx
// ✅ RIGHT: just compute it during render
const fullName = firstName + ' ' + lastName;
// (wrap in useMemo only if the computation is genuinely expensive)
```

### Pitfall 6 — Using an effect to reset state when a prop changes

```jsx
// ❌ WRONG: reset child state in an effect — renders the stale UI for one frame first
useEffect(() => { setComment(''); }, [userId]);
```

```jsx
// ✅ RIGHT: give the component a key so React remounts it fresh
<Profile userId={userId} key={userId} />
// inside Profile, useState('') naturally re-initializes on remount
```

### Pitfall 7 — Putting event-handler logic in an effect

```jsx
// ❌ WRONG: "POST on submit" expressed as an effect watching a flag
useEffect(() => {
  if (submitted) { fetch('/api/buy', { method: 'POST' }); }
}, [submitted]);
```

```jsx
// ✅ RIGHT: do it in the event handler — it's caused by a user action, not by rendering
function handleSubmit() {
  fetch('/api/buy', { method: 'POST' });
}
```

> The litmus test from the React docs' **"You Might Not Need an Effect"**: effects are for *synchronizing with external systems*. If the code is (a) computing a render value, (b) resetting/adjusting state from props, or (c) responding to a specific user interaction, it belongs in render, a `key`, or an event handler respectively — not an effect.

### Pitfall 8 — Mapping from class lifecycles, mechanically

If you're reading legacy class code, here's the translation. It's a *guide*, not a 1:1 mapping — the effect mindset is "synchronize," not "run on this lifecycle moment":

| Class lifecycle | Hook equivalent |
| --- | --- |
| `componentDidMount` | `useEffect(() => { … }, [])` |
| `componentDidUpdate` | `useEffect(() => { … }, [dep])` (runs on dep change; runs on mount too) |
| `componentWillUnmount` | the cleanup function returned from `useEffect` |
| `componentDidMount` + `componentDidUpdate` together | `useEffect(() => { … })` (no array) |

The mismatch to watch for: a single `componentDidUpdate` often handled many concerns with `if (prevProps.x !== this.props.x)` checks. The hooks idiom is the opposite — **split unrelated concerns into separate effects**, each with its own focused dependency list.

---

## Best practices

- **Do** treat every effect as *synchronization*: "keep X in sync with these reactive values." If you can't phrase it that way, you probably don't need an effect.
- **Do** always return a cleanup function when you start something that outlives a single tick (timer, listener, subscription, fetch).
- **Do** keep StrictMode on in development and make your effects survive double-invocation. Double-firing in dev = your cleanup works.
- **Do** obey the `eslint-plugin-react-hooks` **exhaustive-deps** rule. Never silence it with `// eslint-disable-next-line` to hide a missing dep — instead remove the *need* for the dep (functional updaters, moving constants out of the component, `useCallback`/`useMemo`, or an Effect Event via `useEffectEvent`, stable as of React 19.2).
- **Do** prefer functional state updates (`setX(x => …)`) inside effects to avoid depending on the current state.
- **Do** split unrelated logic into multiple effects, each with one job and one dependency list.
- **Don't** put an `async` function directly as the `setup` argument.
- **Don't** use effects to derive render data, reset state on prop change, or run event-handler logic.
- **Don't** fetch data with bare `fetch` in large apps when a cache-aware library (TanStack Query/SWR) or a framework loader is available — but understand the manual pattern (Example 4) so you know what they do.
- **Don't** disable StrictMode to "fix" double effects.

---

## Exercises

1. **Title sync.** Build a `<DocumentTitle text={...} />` component that sets `document.title` to its `text` prop, re-syncing whenever `text` changes, and restores the previous title on unmount. (Hint: capture the old title in setup, restore it in cleanup.)

2. **Debounced search.** Build a `<Search />` with a controlled `<input>`. After the user stops typing for 400 ms, log the query (pretend it's an API call). Cancel the pending log if the user types again before 400 ms. Make it pass StrictMode without firing twice for one keystroke.

3. **Online/offline badge.** Render "Online" or "Offline" by subscribing to the window's `online` and `offline` events. Initialize from `navigator.onLine`. Clean up both listeners on unmount.

4. **Race-safe fetch.** Extend `UserCard` from Example 4: add a numeric input that changes `userId` rapidly. Prove (via a `console.log` in the abort branch) that stale requests are aborted and never overwrite the latest user. No flicker of a wrong user allowed.

5. **Refactor away an effect.** Given a component holding `items` state and a separate `visibleItems` state synced via `useEffect(() => setVisibleItems(items.filter(...)), [items, query])`, remove the effect entirely and the extra state. Compute `visibleItems` during render instead.

### Solutions

```jsx
// Exercise 1 — DocumentTitle
import { useEffect } from 'react';

function DocumentTitle({ text }) {
  useEffect(() => {
    const previous = document.title;
    document.title = text;
    return () => { document.title = previous; }; // restore on unmount or before re-sync
  }, [text]);
  return null;
}
```

```jsx
// Exercise 2 — Debounced search
import { useState, useEffect } from 'react';

function Search() {
  const [query, setQuery] = useState('');

  useEffect(() => {
    if (query === '') return;                 // nothing to search
    const id = setTimeout(() => {
      console.log('searching for:', query);   // pretend API call
    }, 400);
    return () => clearTimeout(id);            // cancel if query changes within 400ms
  }, [query]);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search…"
    />
  );
}
```

```jsx
// Exercise 3 — Online/offline badge
import { useState, useEffect } from 'react';

function ConnectionStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const goOnline = () => setOnline(true);
    const goOffline = () => setOnline(false);
    window.addEventListener('online', goOnline);
    window.addEventListener('offline', goOffline);
    return () => {
      window.removeEventListener('online', goOnline);
      window.removeEventListener('offline', goOffline);
    };
  }, []);

  return <span>{online ? '🟢 Online' : '🔴 Offline'}</span>;
}
```

```jsx
// Exercise 4 — Race-safe fetch (key additions to Example 4)
import { useState, useEffect } from 'react';

function UserCardRaceSafe() {
  const [userId, setUserId] = useState(1);
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    (async () => {
      try {
        const res = await fetch(`https://api.example.com/users/${userId}`, {
          signal: controller.signal,
        });
        setUser(await res.json());
      } catch (err) {
        if (err.name === 'AbortError') {
          console.log('aborted stale request for id', userId); // proves cancellation
          return;
        }
        throw err;
      }
    })();
    return () => controller.abort();
  }, [userId]);

  return (
    <>
      <input
        type="number"
        value={userId}
        onChange={(e) => setUserId(Number(e.target.value))}
      />
      <h2>{user?.name ?? 'Loading…'}</h2>
    </>
  );
}
```

```jsx
// Exercise 5 — No effect needed; derive during render
import { useState } from 'react';

function ItemList({ items }) {
  const [query, setQuery] = useState('');
  // Derived value — recomputed each render from source data. No effect, no extra state.
  const visibleItems = items.filter((item) =>
    item.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{visibleItems.map((i) => <li key={i.id}>{i.name}</li>)}</ul>
    </>
  );
}
```

---

## Key takeaways

- A **side effect** is anything that reaches outside pure render: fetching, subscribing, timers, manual DOM, external logging. It belongs in `useEffect`, an event handler, or render — never inside the render body as a bare statement.
- `useEffect(setup, deps)` has three forms: **no array** (every render), **`[]`** (once after mount), **`[a, b]`** (when a listed dep changes by `Object.is`).
- The **cleanup function** runs **before each re-run** and **once on unmount** — it is how you avoid leaks, doubled subscriptions, and race conditions.
- Effects run **after the browser paints**; `useLayoutEffect` runs before paint (see `09-hooks-deep-dive.md`).
- **StrictMode double-invokes effects in dev** on purpose — passing it means your cleanup is correct; never disable it to hide the symptom.
- Use **`AbortController`** (or an `ignore` flag) to make data-fetching effects race-safe, and track `loading`/`error` state explicitly.
- **Obey exhaustive-deps.** Lying about deps causes stale-closure bugs; fix the cause (functional updates, stabilized refs) rather than silencing the lint.
- **You often don't need an effect** — derive data in render, reset state with a `key`, and run user-triggered logic in event handlers.

---

## Further reading

- React docs — [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- React docs — [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- React docs — [Lifecycle of Reactive Effects](https://react.dev/learn/lifecycle-of-reactive-effects)
- React docs — [Removing Effect Dependencies](https://react.dev/learn/removing-effect-dependencies)
- React reference — [`useEffect`](https://react.dev/reference/react/useEffect)
- React reference — [`useLayoutEffect`](https://react.dev/reference/react/useLayoutEffect) and [`useSyncExternalStore`](https://react.dev/reference/react/useSyncExternalStore)
- ESLint — [`eslint-plugin-react-hooks` (exhaustive-deps)](https://www.npmjs.com/package/eslint-plugin-react-hooks)
- MDN — [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
