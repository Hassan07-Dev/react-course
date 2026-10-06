# 13 — Performance Optimization

> **What you'll learn:** how React decides to re-render, how to *stop* unnecessary re-renders surgically (not everywhere), and how to keep a UI responsive under heavy load with memoization, code-splitting, virtualization, and profiling.

By the end of this lesson you will be able to:

- Explain precisely **when and why** a component re-renders (state, props, parent re-render, context).
- Use `React.memo`, `useMemo`, and `useCallback` to let memoized children **actually** skip re-renders — and know when each one is wasted effort.
- Restructure components by **colocating** and **lifting state down** to shrink the re-render blast radius.
- Use **stable keys** to preserve or intentionally reset component state.
- Split your bundle with `React.lazy` + `Suspense` and dynamic `import()`, including route-based splitting.
- Virtualize huge lists with `react-window`.
- Profile real apps with the **React DevTools Profiler** and "Highlight updates."
- Keep input responsive with a first taste of `useTransition` / `useDeferredValue`, and understand what the **React Compiler** automates.

---

## Prerequisites

You should be comfortable with the material from earlier lessons:

- **Components, props, and JSX** (see `04-components-and-props.md`) — props are how data flows down.
- **State with `useState`** and the render-as-a-function-of-state model (see `07-state-usestate.md`).
- **`useEffect`** and dependency arrays (see `08-useeffect-side-effects.md`) — the same referential-equality rules apply here.
- **`useContext`** (see `12-context-api.md`) — context propagation is a common cause of wide re-renders.
- **Custom hooks and lists/keys** (see `05-rendering-lists-conditionals.md`) — keys are central to the "reset vs preserve state" trick.

If those concepts are shaky, performance work will feel like guesswork. The single most important prerequisite mindset: **a re-render is not a DOM update.** React re-running your function is cheap-ish; the slow part is usually *what you do inside* render plus the DOM diff. Keep that distinction front and center.

---

## Concept

### What a "render" actually is

In React, a **render** is React calling your component function to get a fresh tree of elements (the JSX you return). React then **reconciles** that new tree against the previous one (the "diff") and, only where they differ, **commits** changes to the real DOM.

So there are three phases:

1. **Render (a.k.a. "render phase")** — your function runs, hooks run, you return elements. Pure, no DOM touched.
2. **Reconciliation** — React compares new elements to old, decides minimal DOM ops.
3. **Commit** — React mutates the DOM and runs layout effects / effects.

A component re-renders when **one** of these is true:

- **Its own state changed** (a `useState`/`useReducer` setter was called with a different value).
- **Its parent re-rendered.** This is the big one beginners miss: when a parent renders, React re-renders *all* its children by default, regardless of whether their props changed.
- **A context it consumes changed value** (the nearest matching `Provider` got a new `value`).
- Less commonly, a `key` change or a forced update.

> **The mental model:** rendering cascades *downward*. A state update at the top of your tree can, by default, re-run every component below it. That is usually fine — React is fast — but it becomes the bottleneck when "below it" is a 10,000-row table or a component doing expensive math in render.

### Why props "changing" is subtle

When a parent re-renders, it creates **new** objects, arrays, and functions in its render body:

```jsx
function Parent() {
  const config = { color: 'blue' };        // new object every render
  const onClick = () => doThing();          // new function every render
  return <Child config={config} onClick={onClick} />;
}
```

Even if the *values* look identical, `config` and `onClick` are **brand-new references** each render. Any optimization that compares props by reference (`React.memo`, `useEffect` deps, etc.) will see "changed." This is the root cause of most failed memoization, and the reason `useMemo`/`useCallback` exist. We'll return to it constantly.

### Bailing out

React gives you escape hatches to **bail out** of re-rendering a subtree:

- `React.memo(Component)` — skip re-render if props are shallow-equal to last time.
- `useMemo(fn, deps)` — cache an expensive computed value across renders.
- `useCallback(fn, deps)` — cache a function reference across renders.

These don't make rendering faster. They make rendering **happen less often**. The art is applying them only where the savings beat the bookkeeping cost.

---

## Syntax & API

### `React.memo`

```jsx
const MemoizedComponent = React.memo(Component, arePropsEqual?);
```

- Wraps a component so React **skips re-rendering** it when its parent re-renders, *provided* the new props are shallow-equal (`Object.is` per prop) to the previous props.
- `arePropsEqual(prevProps, nextProps)` is an **optional custom comparator**. Return `true` to skip the render, `false` to render. (Note this is the *opposite* polarity of the old class `shouldComponentUpdate`, which returned `true` to render.)
- Memoization is **bypassed** if the component reads changed context, or has its own state change — `memo` only guards against *parent-driven* re-renders.

### `useMemo`

```jsx
const cachedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

- Runs the factory function during render **only when a dependency changed** (by `Object.is`). Otherwise returns the previous cached value.
- Use it for (a) expensive computations and (b) **referential stability** of objects/arrays you pass to memoized children or to effect deps.
- The deps array follows the exact same rules as `useEffect` (see `08-useeffect-side-effects.md`).

### `useCallback`

```jsx
const memoizedFn = useCallback(() => { doSomething(a); }, [a]);
```

- `useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)` — it memoizes a **function reference** so it stays stable across renders until a dep changes.
- Pointless unless that function is passed to a `memo`-wrapped child or used as an effect dependency.

### `React.lazy` + `Suspense`

```jsx
const Heavy = React.lazy(() => import('./Heavy.jsx'));

<Suspense fallback={<Spinner />}>
  <Heavy />
</Suspense>
```

- `lazy(load)` takes a function returning a `Promise` that resolves to a module with a **default export** that is a component.
- `Suspense` shows `fallback` while the chunk (or any suspending child) is loading.

### `useTransition` / `useDeferredValue` (intro only — full treatment in `22-modern-react-19.md`)

```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => setFilter(value));   // marks this update low-priority

const deferredQuery = useDeferredValue(query); // lags behind, lets urgent updates win
```

---

## Examples

### Example 1 — Watching the cascade (and why `memo` helps)

Open React DevTools, enable **"Highlight updates when components render"** (gear icon → General), and run this:

```jsx
import { useState } from 'react';

function ExpensiveTree() {
  console.log('ExpensiveTree rendered');
  // pretend this subtree is large
  return <p>I am expensive.</p>;
}

export default function App() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>Count: {count}</button>
      <ExpensiveTree />
    </div>
  );
}
```

Click the button: `ExpensiveTree rendered` logs **every click**, even though it has no props and never changes. That's the parent-cascade rule. Now wrap it:

```jsx
import { memo } from 'react';

const ExpensiveTree = memo(function ExpensiveTree() {
  console.log('ExpensiveTree rendered');
  return <p>I am expensive.</p>;
});
```

Now it logs **once**. Because `ExpensiveTree` has no props, the shallow comparison always passes and React bails out. This is the cleanest win for `memo`: a leaf with stable (or no) props under a frequently-rendering parent.

### Example 2 — `memo` defeated by a fresh function, then fixed with `useCallback`

```jsx
import { useState, memo } from 'react';

const Button = memo(function Button({ onClick, children }) {
  console.log('Button rendered:', children);
  return <button onClick={onClick}>{children}</button>;
});

export default function Toolbar() {
  const [count, setCount] = useState(0);

  // 🐌 New function reference every render → memo can't bail out
  const handleSave = () => console.log('saved');

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Re-render parent</button>
      <Button onClick={handleSave}>Save</Button>
    </>
  );
}
```

`Button` re-renders on every parent render because `handleSave` is a new reference each time, so the shallow prop check fails. Fix the reference with `useCallback`:

```jsx
import { useState, useCallback, memo } from 'react';

const Button = memo(function Button({ onClick, children }) {
  console.log('Button rendered:', children);
  return <button onClick={onClick}>{children}</button>;
});

export default function Toolbar() {
  const [count, setCount] = useState(0);

  const handleSave = useCallback(() => console.log('saved'), []); // stable

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Re-render parent</button>
      <Button onClick={handleSave}>Save</Button>
    </>
  );
}
```

Now `Button` logs once. **Key insight:** `useCallback` only pays off *because* `Button` is wrapped in `memo`. Without `memo`, the stable reference buys you nothing — the child re-renders anyway. The two tools are a pair.

### Example 3 — `useMemo` for an expensive computation and for a stable object prop

```jsx
import { useMemo, useState, memo } from 'react';

const Chart = memo(function Chart({ data, options }) {
  console.log('Chart rendered');
  return <div>{data.length} points, theme: {options.theme}</div>;
});

export default function Dashboard({ points }) {
  const [theme, setTheme] = useState('light');
  const [tick, setTick] = useState(0);

  // (a) Expensive derive — only recompute when `points` changes
  const data = useMemo(() => {
    console.log('recomputing data…');
    return points.map((p) => ({ ...p, scaled: p.value * 100 }));
  }, [points]);

  // (b) Stable object so Chart's shallow prop check can pass
  const options = useMemo(() => ({ theme }), [theme]);

  return (
    <>
      <button onClick={() => setTick((t) => t + 1)}>Tick {tick}</button>
      <button onClick={() => setTheme((t) => (t === 'light' ? 'dark' : 'light'))}>
        Toggle theme
      </button>
      <Chart data={data} options={options} />
    </>
  );
}
```

Clicking **Tick** re-renders `Dashboard` but neither `points` nor `theme` changed, so `data` and `options` keep their references and `Chart` bails out. Clicking **Toggle theme** recreates `options` (intentionally) and re-renders `Chart`. You've made the cost track the *real* inputs instead of every render.

### Example 4 — Stable keys to preserve vs reset state

Keys aren't only for lists — they control component **identity**. Same key = React reuses the instance and its state. Different key = React unmounts the old and mounts a fresh one.

```jsx
import { useState } from 'react';

function ProfileForm() {
  const [draft, setDraft] = useState('');
  return (
    <input
      value={draft}
      onChange={(e) => setDraft(e.target.value)}
      placeholder="Type a note…"
    />
  );
}

export default function App() {
  const [userId, setUserId] = useState('alice');
  return (
    <>
      <button onClick={() => setUserId('alice')}>Alice</button>
      <button onClick={() => setUserId('bob')}>Bob</button>
      {/* key by userId → switching users wipes the draft, as intended */}
      <ProfileForm key={userId} />
    </>
  );
}
```

Switching from Alice to Bob remounts `ProfileForm`, clearing `draft`. Remove the `key` and the typed text would *leak* across users — a classic state-preservation bug. Use keys deliberately: stable keys preserve state and avoid needless remounts; changing keys is the cleanest way to reset a subtree.

### Example 5 — Colocation / lifting state *down*

Often the cheapest "optimization" is structural. Suppose a search box lives high up and its keystrokes re-render a giant list:

```jsx
// 🐌 BEFORE: `query` lives in App, so every keystroke re-renders BigList
export default function App() {
  const [query, setQuery] = useState('');
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <BigList />   {/* re-renders on every keystroke even though it ignores query */}
    </>
  );
}
```

Push the state *down* into the only component that needs it:

```jsx
// ✅ AFTER: query is colocated with the input → BigList never re-renders on typing
function SearchBox() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

export default function App() {
  return (
    <>
      <SearchBox />
      <BigList />
    </>
  );
}
```

No `memo`, no `useCallback` — just smaller scope. **Reach for structure before reaching for memoization.**

### Example 6 — Code-splitting with `React.lazy` + `Suspense`

A heavy chart library shouldn't be in your initial bundle if most users never open the chart:

```jsx
import { lazy, Suspense, useState } from 'react';

const AnalyticsChart = lazy(() => import('./AnalyticsChart.jsx'));

export default function ReportPage() {
  const [open, setOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setOpen(true)}>Show analytics</button>
      {open && (
        <Suspense fallback={<p>Loading chart…</p>}>
          <AnalyticsChart />
        </Suspense>
      )}
    </div>
  );
}
```

The `AnalyticsChart` chunk downloads only when `open` flips to `true`. Vite automatically code-splits on dynamic `import()`, so you get a separate JS file with no extra config.

Route-based splitting is the highest-leverage version — split per page so each route ships only its own code:

```jsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home.jsx'));
const Settings = lazy(() => import('./pages/Settings.jsx'));

const withSuspense = (el) => <Suspense fallback={<p>Loading…</p>}>{el}</Suspense>;

const router = createBrowserRouter([
  { path: '/', element: withSuspense(<Home />) },
  { path: '/settings', element: withSuspense(<Settings />) },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

### Example 7 — Virtualizing a huge list with `react-window`

Rendering 50,000 `<li>` nodes will create 50,000 DOM elements and tank scroll performance. Virtualization renders only the rows currently in the viewport.

```bash
npm install react-window
```

react-window v2 (current) exposes a single `List` component. You pass a `rowComponent`, the `rowCount`, a `rowHeight`, and any per-row data via `rowProps`. The row component receives `index` and `style` automatically, plus whatever you put in `rowProps`:

```jsx
import { List } from 'react-window';

const items = Array.from({ length: 50000 }, (_, i) => `Item #${i}`);

// Receives `index` and `style` by default, plus props from `rowProps`.
function Row({ index, style, items }) {
  // `style` positions the row absolutely — you MUST spread it
  return <div style={style}>{items[index]}</div>;
}

export default function HugeList() {
  return (
    <List
      rowComponent={Row}
      rowCount={items.length}
      rowHeight={35}          // each row's height in px
      rowProps={{ items }}
      style={{ height: 400 }} // viewport height; v2 auto-sizes width
    />
  );
}
```

Only the rows in (and just around) the viewport exist in the DOM regardless of list length. (`@tanstack/react-virtual` is a popular, more flexible alternative for variable heights and grids.)

> **Version note:** react-window **v1** used `FixedSizeList`/`VariableSizeList` with a children-as-function API (`itemCount`, `itemSize`, `width`/`height` props). **v2** (a rewrite) replaced those with the unified `List`/`Grid` components shown above and auto-memoizes the row renderer. The examples here target v2; if you're on a v1 codebase, the prop names differ.

### Example 8 — Responsiveness with `useDeferredValue` (preview of `22-modern-react-19.md`)

```jsx
import { useState, useDeferredValue, useMemo } from 'react';

function Results({ query }) {
  const list = useMemo(
    () => bigArray.filter((x) => x.includes(query)),
    [query]
  );
  return <ul>{list.map((x) => <li key={x}>{x}</li>)}</ul>;
}

export default function Search() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query); // lags behind during fast typing

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={deferredQuery} />
    </>
  );
}
```

The input stays snappy (it tracks `query`), while the expensive `Results` re-render uses `deferredQuery` and is allowed to fall behind. We cover the full concurrent story in lesson 22.

---

## Under the hood

**Rendering is depth-first and synchronous (in the render phase).** When you call a setter, React schedules work. On the next render pass it calls the owner component, walks into its children, and for each child decides: *can I bail out?* The bail-out checks, in order, are roughly:

1. Did this component's own state/reducer produce a new value? If not, and props are referentially identical to last time, React may reuse the previous output (the "eager bailout" for `useState` when you set the same value).
2. Is the component wrapped in `memo`? Run the (shallow or custom) prop comparison; if equal, **skip the function call and reuse the prior subtree** entirely.
3. Otherwise, call the function and reconcile.

**`memo` only guards the parent → child edge.** It does nothing about state changes inside the component or context updates it subscribes to. A `memo` component that reads a changing context still re-renders on every context change — `memo` cannot stop that.

**Why shallow comparison?** Deep equality on arbitrary props would be slower and unpredictable than just re-rendering. React compares each prop with `Object.is`. That's why a new object/array/function literal — even with identical contents — always "differs." Internalize: *content equal ≠ reference equal.*

**`useMemo`/`useCallback` are caches keyed on the deps array.** React stores `[value, deps]` per hook call slot. On re-render it compares each dep with `Object.is`; if all match, it returns the cached value without running the factory. This is *not* a guarantee — React reserves the right to discard the cache (e.g. for memory) and recompute. **Never rely on `useMemo` for correctness**, only as a performance hint. If you need a value to be created exactly once for correctness (a ref, a subscription), use `useRef` or lazy `useState` init instead.

**Context propagation.** When a `Provider`'s `value` changes (by `Object.is`), *every* consumer of that context re-renders, no matter how deep, and `memo` boundaries do **not** stop it. That's why an unstable provider `value={{ user, setUser }}` (new object each render) is a notorious performance trap — wrap it in `useMemo`. See `12-context-api.md`.

**The cost you're actually paying.** A re-render that produces identical output is cheap-ish but not free: the function runs, hooks run, and React diffs the returned tree. The expensive parts are (a) heavy computation in the render body, and (b) large subtrees. Memoization targets *both* by skipping the function call. But the bookkeeping — extra closures, deps arrays, comparison work — has its own cost, which is why blanket memoization can be net-negative.

**React Compiler.** React's compiler (a build-time plugin — Babel, plus integrations for Vite, Next.js, and React Native — which reached **1.0 stable in October 2025**) statically analyzes your components and **auto-inserts memoization** equivalent to hand-written `useMemo`/`useCallback`/`memo` where it can prove it's safe. When you adopt it, the goal is to delete most manual memoization and let the compiler do it more thoroughly and correctly than humans typically do. It relies on your components following the **Rules of React** (pure render, no mutation of props/state). The compiler doesn't change *when* React re-renders conceptually — it changes how aggressively cheap bailouts are inserted. Until you've adopted it and verified, the manual techniques in this lesson remain the playbook.

---

## Common mistakes & pitfalls

### Pitfall 1 — Memoizing everything "just in case"

```jsx
// 🐌 WRONG: every value and handler wrapped, child isn't even memoized
function Row({ label }) {
  const style = useMemo(() => ({ padding: 4 }), []);
  const onClick = useCallback(() => console.log(label), [label]);
  return <div style={style} onClick={onClick}>{label}</div>;
}
```

```jsx
// ✅ RIGHT: plain code — the child does no expensive work and isn't memo-wrapped
function Row({ label }) {
  return (
    <div style={{ padding: 4 }} onClick={() => console.log(label)}>
      {label}
    </div>
  );
}
```

`useMemo`/`useCallback` here add allocations and comparison work for **zero** benefit, because nothing downstream is memoized to benefit from stable references. **Measure first.** Memoize when the Profiler shows a real, repeated cost.

### Pitfall 2 — `useCallback` without a `memo` child

```jsx
// 🐌 WRONG: stable handler, but ListItem re-renders anyway
function ListItem({ onSelect }) { /* not wrapped in memo */ }

const onSelect = useCallback((id) => select(id), []);
```

```jsx
// ✅ RIGHT: memoize the child so the stable reference actually pays off
const ListItem = memo(function ListItem({ onSelect }) { /* … */ });

const onSelect = useCallback((id) => select(id), []);
```

`useCallback` only matters when the consumer compares references. Pair it with `memo` (or an effect dep) or drop it.

### Pitfall 3 — Index as key in a reorderable list

```jsx
// 🐌 WRONG: index keys mismatch state to the wrong row after reordering
{rows.map((row, i) => <EditableRow key={i} row={row} />)}
```

```jsx
// ✅ RIGHT: stable identity preserves the right component state
{rows.map((row) => <EditableRow key={row.id} row={row} />)}
```

Index keys cause React to reuse the wrong instances on insert/reorder/delete, leaking input state into the wrong rows. Use a stable, data-derived id. See `05-rendering-lists-conditionals.md`.

### Pitfall 4 — Unstable context value

```jsx
// 🐌 WRONG: new object every render → all consumers re-render every time
<AuthContext.Provider value={{ user, login }}>{children}</AuthContext.Provider>
```

```jsx
// ✅ RIGHT: memoize the value so consumers only re-render when it truly changes
const value = useMemo(() => ({ user, login }), [user, login]);
<AuthContext.Provider value={value}>{children}</AuthContext.Provider>
```

### Pitfall 5 — Expensive work directly in the render body

```jsx
// 🐌 WRONG: sorts a huge array on EVERY render, even unrelated ones
function Table({ rows }) {
  const sorted = rows.slice().sort(expensiveComparator);
  return <Grid rows={sorted} />;
}
```

```jsx
// ✅ RIGHT: only sort when `rows` changes
function Table({ rows }) {
  const sorted = useMemo(() => rows.slice().sort(expensiveComparator), [rows]);
  return <Grid rows={sorted} />;
}
```

### Pitfall 6 — Optimizing without measuring

The biggest pitfall of all: scattering `memo`/`useMemo`/`useCallback` based on vibes. Profile, find the *one* component re-rendering 200×/sec or the *one* 80ms render, fix that, re-profile. Most components never need any of this.

---

## Best practices

- **Do** measure with the Profiler *before* optimizing, and re-measure after. Treat every memo as a hypothesis to verify.
- **Do** prefer structural fixes first: colocate state, lift state down, split components so updates have a smaller blast radius.
- **Do** pair `React.memo` with `useCallback`/`useMemo` — they only work as a set. A `memo` child needs referentially-stable object/function props.
- **Do** use stable, data-derived `key`s; change a `key` deliberately when you *want* to reset a subtree's state.
- **Do** code-split at route boundaries and around heavy, rarely-used features with `lazy` + `Suspense`.
- **Do** virtualize any list/grid that can exceed a few hundred rows.
- **Don't** wrap everything in `useMemo`/`useCallback` "for safety" — it adds cost and noise with no benefit unless a memoized consumer exists.
- **Don't** rely on `useMemo` for correctness; React may discard the cache. Use `useRef`/lazy state init for "create once" guarantees.
- **Don't** create new context `value` objects inline; memoize them.
- **Don't** do expensive computation in the render body unmemoized.
- **Consider** adopting the React Compiler; when you do, plan to remove most manual memoization and let it handle bailouts.

---

## Exercises

1. **Spot the cascade.** Build a `Counter` button and an unrelated `<Clock />` that logs on each render. Use DevTools "Highlight updates" to confirm `Clock` flashes on every count click. Then make it stop flashing — *without* `memo`. (Hint: structure.)

2. **Fix the broken memo.** Given a `memo`-wrapped `<Avatar user={…} onClick={…} />` that re-renders on every parent render, identify the two unstable props and stabilize them so `Avatar` renders only when its data changes.

3. **Route split.** Convert a two-page app (`Home`, `Reports`) so each page is loaded with `React.lazy` and wrapped in `Suspense`. Verify in the Network tab that the `Reports` chunk loads only on navigation.

4. **Virtualize.** Render a list of 20,000 items normally, profile the initial mount, then convert it to `react-window`'s `List` component and compare the DOM node count and mount time.

5. **(Stretch) Keep typing smooth.** Build a search input filtering a 10,000-item array in render. Make the input lag-free using `useDeferredValue`, and add a "results are catching up" style by comparing the live value to the deferred one (`query !== deferred`). Note: don't wrap the input's own setter in `useTransition` — controlled inputs must update synchronously.

### Solutions

**Exercise 1 — structural fix (no `memo`):**

```jsx
import { useState } from 'react';

function Clock() {
  console.log('Clock rendered');
  return <p>tick</p>;
}

function Counter() {                 // state colocated here
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}

export default function App() {       // App never re-renders, so Clock doesn't either
  return (
    <>
      <Counter />
      <Clock />
    </>
  );
}
```

Because `count` lives inside `Counter`, clicking re-renders only `Counter`. `App` and `Clock` are untouched — no `memo` required.

**Exercise 2 — stabilize the props:**

```jsx
import { memo, useMemo, useCallback, useState } from 'react';

const Avatar = memo(function Avatar({ user, onClick }) {
  console.log('Avatar rendered');
  return <img src={user.url} alt={user.name} onClick={onClick} />;
});

export default function Profile({ name, url }) {
  const [count, setCount] = useState(0);

  const user = useMemo(() => ({ name, url }), [name, url]);   // stable object
  const onClick = useCallback(() => console.log('avatar click'), []); // stable fn

  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>Re-render ({count})</button>
      <Avatar user={user} onClick={onClick} />
    </>
  );
}
```

`user` and `onClick` were the unstable props (a fresh object and fresh function each render). Memoizing both lets `Avatar`'s shallow comparison pass, so it renders only when `name`/`url` change.

**Exercise 3 — route splitting:**

```jsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home.jsx'));
const Reports = lazy(() => import('./pages/Reports.jsx'));

const wrap = (el) => <Suspense fallback={<p>Loading…</p>}>{el}</Suspense>;

const router = createBrowserRouter([
  { path: '/', element: wrap(<Home />) },
  { path: '/reports', element: wrap(<Reports />) },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

Vite emits a separate chunk per dynamic `import()`; the `Reports` chunk appears in the Network tab only when you navigate to `/reports`.

**Exercise 4 — virtualization:**

```jsx
import { List } from 'react-window';

const items = Array.from({ length: 20000 }, (_, i) => `Row ${i}`);

const Row = ({ index, style, items }) => <div style={style}>{items[index]}</div>;

export default function BigList() {
  return (
    <List
      rowComponent={Row}
      rowCount={items.length}
      rowHeight={28}
      rowProps={{ items }}
      style={{ height: 500 }}
    />
  );
}
```

In DevTools' Elements panel you'll see only a couple dozen `<div>`s instead of 20,000, and the Profiler mount drops from hundreds of ms to a few ms.

**Exercise 5 — responsive search (stretch):**

```jsx
import { useState, useMemo, useDeferredValue } from 'react';

const data = Array.from({ length: 10000 }, (_, i) => `entry-${i}`);

function Results({ query }) {
  const list = useMemo(() => data.filter((x) => x.includes(query)), [query]);
  return <ul>{list.slice(0, 100).map((x) => <li key={x}>{x}</li>)}</ul>;
}

export default function Search() {
  const [query, setQuery] = useState('');
  const deferred = useDeferredValue(query);

  // The view is "behind" whenever the deferred value hasn't caught up yet.
  const isPending = query !== deferred;

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)} // controlled input: update synchronously
        style={{ opacity: isPending ? 0.6 : 1 }}
      />
      <Results query={deferred} />
    </>
  );
}
```

The input remains responsive because its `value` is updated **synchronously** (`setQuery` in `onChange`), while the expensive filtering runs against `deferred`, which is allowed to lag behind during fast typing. Comparing `query !== deferred` tells you the view is still catching up, so you can dim the input while it does. Note: you must **not** wrap a controlled input's setter in `startTransition` — React requires controlled-input updates to be synchronous, so transitions can't drive the input's `value`. For this "snappy input, laggy view" pattern, `useDeferredValue` is the right tool. Full details in `22-modern-react-19.md`.

---

## Key takeaways

- A component re-renders when its **own state** changes, its **parent** re-renders, or a **context it consumes** changes. The parent cascade is the most-missed cause.
- A re-render is not a DOM mutation; the costs are **expensive work in render** and **large subtrees** — that's what you're optimizing.
- `React.memo` bails out of **parent-driven** re-renders via shallow prop comparison; it can't stop re-renders from local state or context.
- `useMemo`/`useCallback` exist to give **referentially stable** props so `memo` children can actually skip — they're a *pair*, not standalone wins, and `useMemo` is never a correctness guarantee.
- Prefer **structure first**: colocate state, lift state down, and use **stable keys** (deliberately changing a key resets a subtree).
- **Code-split** with `React.lazy` + `Suspense` (route-based is highest-leverage) and **virtualize** large lists with `react-window`.
- **Profile, then optimize.** Blanket memoization is a net negative; the React DevTools Profiler tells you the one place that matters.
- The **React Compiler** auto-inserts memoization; adopting it lets you delete most manual `memo`/`useMemo`/`useCallback`.

---

## Further reading

- React docs — [`memo`](https://react.dev/reference/react/memo)
- React docs — [`useMemo`](https://react.dev/reference/react/useMemo) and [`useCallback`](https://react.dev/reference/react/useCallback)
- React docs — ["You Might Not Need an Effect" / Keeping components pure](https://react.dev/learn/keeping-components-pure)
- React docs — [`lazy`](https://react.dev/reference/react/lazy) and [`<Suspense>`](https://react.dev/reference/react/Suspense)
- React docs — [`useTransition`](https://react.dev/reference/react/useTransition) and [`useDeferredValue`](https://react.dev/reference/react/useDeferredValue)
- React docs — [React Compiler](https://react.dev/learn/react-compiler)
- React docs — [Preserving and resetting state](https://react.dev/learn/preserving-and-resetting-state) (the `key` mental model)
- React DevTools — [Profiler guide](https://react.dev/learn/react-developer-tools)
- [`react-window`](https://github.com/bvaughn/react-window) and [`@tanstack/react-virtual`](https://tanstack.com/virtual/latest)
