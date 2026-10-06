# 21 — Class Components & Lifecycle (Legacy, for Reading Old Code)

> **What you'll learn:** how React's older class-component model works — syntax, `this.state`/`this.setState`, the full lifecycle, error boundaries, and exactly how to read and migrate this code to modern hooks.

By the end of this lesson you will be able to:

- Recognize and read a class component written by extending `React.Component`.
- Use `this.props`, `this.state`, and `this.setState` correctly, including its async batching, merge semantics, and the functional updater form.
- Explain and fix the infamous `this`-binding problem with the three idiomatic solutions.
- Trace the complete commit lifecycle: `constructor` → `render` → `componentDidMount` → `componentDidUpdate` → `componentWillUnmount`, plus `shouldComponentUpdate`, `getDerivedStateFromProps`, and `getSnapshotBeforeUpdate`.
- Build an **error boundary** with `getDerivedStateFromError` / `componentDidCatch` — the one thing classes can still do that hooks cannot.
- Use `React.PureComponent` and understand its shallow-compare bailout.
- Map every lifecycle method to its function-component / hooks equivalent and migrate a real class to hooks, line by line.

> **Why this lesson exists:** Hooks (lessons 07–10) are the modern default and what you should write today. But class components are still alive in millions of lines of production code, in interview questions, and in the React docs' historical sections. You need to *read* them fluently and *migrate* them confidently. You almost never need to *write* a new one — the single exception is an error boundary.

---

## Prerequisites

Make sure you're comfortable with these earlier lessons first:

- **01 — JS Foundations** (`01-js-foundations.md`): `class`, `extends`, `super`, `this`, arrow functions, and how `this` is determined at *call time*. This lesson leans hard on that.
- **04 — Components and Props** (`04-components-and-props.md`): the component-as-function mental model and props.
- **07 — State with useState** (`07-state-usestate.md`): the hooks state model you'll map class state onto.
- **08 — useEffect & Side Effects** (`08-useeffect-side-effects.md`): the effect model that replaces lifecycle methods.
- **09 — Hooks Deep Dive** (`09-hooks-deep-dive.md`) and **13 — Performance** (`13-performance.md`): for `React.memo`, `useMemo`, and the bailout discussion.

---

## Concept

Before hooks landed in React 16.8 (February 2019), there were two kinds of components:

1. **Function components** — back then they were "stateless functional components": a function that took props and returned JSX. They had no state and no lifecycle.
2. **Class components** — JavaScript classes that `extends React.Component`. These were the *only* way to hold state or run code at specific moments ("when I appear on screen", "when I'm about to disappear").

A class component is, at heart, an object with a `render()` method that React calls. React **instantiates** the class (`new YourComponent(props)`), keeps that instance alive across renders, stores your mutable data on `this.state`, and re-invokes `render()` whenever props or state change. Contrast this with a function component, which React simply **calls** as a plain function on every render — there is no long-lived `this` instance; per-render data lives in hooks that React tracks behind the scenes.

The mental shift is worth stating plainly:

- **Class** = one long-lived instance, mutated over time. "Lifecycle methods" are callbacks React fires at moments in that instance's life.
- **Function + hooks** = the function runs fresh every render; you *synchronize* with the outside world via effects, rather than reacting to named lifecycle moments.

A useful analogy: a class component is like a **building with a doorbell, a maintenance log, and a demolition crew** — distinct events ("someone moved in", "the tenants changed", "we're tearing it down"). A function component is more like a **photograph re-taken every render** — and `useEffect` is the instruction "after you develop this photo, go make the world match it."

You will spend most of your career writing the photograph version. This lesson teaches you to read the building.

---

## Syntax & API

### Minimal class component

```jsx
import React from 'react';

class Greeting extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}!</h1>;
  }
}

export default Greeting;
```

- It **must** `extend React.Component` (or `React.PureComponent`).
- It **must** define `render()`, which returns JSX, a string, a number, `null`, a fragment, or an array of elements.
- Props arrive as `this.props` — read-only, exactly like function-component props.
- `render()` must be **pure**: no side effects, no `setState`, no DOM mutation. It should compute output purely from `this.props` and `this.state`.

### State & the constructor

```jsx
class Counter extends React.Component {
  constructor(props) {
    super(props);            // required before using `this`
    this.state = { count: 0, name: props.initialName };
  }

  render() {
    return <p>{this.state.count}</p>;
  }
}
```

You can also use the **class-fields** syntax (standard JS, supported by Vite/Babel out of the box) to skip the constructor:

```jsx
class Counter extends React.Component {
  state = { count: 0 };        // class field — no constructor needed
  render() { return <p>{this.state.count}</p>; }
}
```

### `this.setState`

```jsx
// Object form — shallow-MERGED into current state, then re-render:
this.setState({ count: 5 });

// Functional updater — receives the up-to-date (prevState, props):
this.setState((prevState, props) => ({ count: prevState.count + 1 }));

// Optional callback — runs AFTER the re-render commits:
this.setState({ count: 5 }, () => console.log('committed', this.state.count));
```

Three properties to burn into memory:

1. **Asynchronous / batched.** `setState` does not update `this.state` immediately. React batches updates and re-renders once. Reading `this.state.count` on the next line gives you the *old* value.
2. **Shallow merge.** `this.setState({ count: 5 })` leaves other keys (e.g. `name`) untouched. This is unlike the `useState` setter, which *replaces* the whole value. (See "Common mistakes".)
3. **Functional updater** `setState(prev => ...)` is the safe way to derive new state from old, because React guarantees `prev` reflects all queued updates — essential when you call `setState` multiple times in a row.

### The lifecycle, in firing order

| Phase | Method | Signature | When it fires |
| --- | --- | --- | --- |
| Mount | `constructor` | `(props)` | Once, before first render. Init state, bind methods. |
| Mount + Update | `static getDerivedStateFromProps` | `(props, state) → partialState \| null` | Before every render. Rare, advanced. |
| Update only | `shouldComponentUpdate` | `(nextProps, nextState) → bool` | Before `render` on updates. Return `false` to skip render. |
| Mount + Update | `render` | `()` | Every render. Pure. |
| Update only | `getSnapshotBeforeUpdate` | `(prevProps, prevState) → snapshot` | After `render`, just before DOM is mutated. |
| Mount | `componentDidMount` | `()` | Once, after first commit to the DOM. |
| Update | `componentDidUpdate` | `(prevProps, prevState, snapshot)` | After every re-render commit. |
| Unmount | `componentWillUnmount` | `()` | Just before the component is removed. |
| Error | `static getDerivedStateFromError` | `(error) → partialState` | After a child throws during render. |
| Error | `componentDidCatch` | `(error, errorInfo)` | After a child throws; for logging. |

> The `componentWillMount`, `componentWillReceiveProps`, and `componentWillUpdate` methods are **deprecated/unsafe** (renamed with an `UNSAFE_` prefix). You may still see them in old code, but never write them. They were removed conceptually because they don't play well with React's concurrent/async rendering.

### `React.PureComponent`

```jsx
class Row extends React.PureComponent {
  render() { return <li>{this.props.label}</li>; }
}
```

`PureComponent` implements `shouldComponentUpdate` for you with a **shallow comparison** of props and state. It re-renders only if a top-level prop or state key changed by reference. It is the class equivalent of wrapping a function component in `React.memo` (see `13-performance.md`).

---

## Examples

### Example 1 — A counter with state and an event handler

```jsx
import React from 'react';

class Counter extends React.Component {
  state = { count: 0 };

  // Arrow class field => `this` is permanently bound to the instance.
  handleClick = () => {
    // Functional updater: correct even if clicked rapidly / batched.
    this.setState((prev) => ({ count: prev.count + 1 }));
  };

  render() {
    return (
      <button onClick={this.handleClick}>
        Clicked {this.state.count} times
      </button>
    );
  }
}

export default Counter;
```

Why the arrow class field? It sidesteps the binding problem (next section). The functional updater means three rapid clicks reliably produce `+3`, not `+1`.

### Example 2 — The full mount/update/unmount lifecycle with a subscription

This component subscribes to `window` resize events on mount and cleans up on unmount — the canonical "side effect with teardown" pattern.

```jsx
import React from 'react';

class WindowWidth extends React.Component {
  state = { width: window.innerWidth };

  handleResize = () => {
    this.setState({ width: window.innerWidth });
  };

  componentDidMount() {
    // Runs once, after first paint. Safe place for subscriptions, fetches, DOM reads.
    window.addEventListener('resize', this.handleResize);
    console.log('mounted');
  }

  componentDidUpdate(prevProps, prevState) {
    // Runs after every re-render. GUARD with a comparison or you loop forever.
    if (prevState.width !== this.state.width) {
      console.log(`width changed: ${prevState.width} → ${this.state.width}`);
    }
  }

  componentWillUnmount() {
    // Runs just before removal. Mirror every subscription set up in didMount.
    window.removeEventListener('resize', this.handleResize);
    console.log('unmounted');
  }

  render() {
    return <p>Window is {this.state.width}px wide</p>;
  }
}

export default WindowWidth;
```

Read this as: *set up in `componentDidMount`, react in `componentDidUpdate`, tear down in `componentWillUnmount`.* That trio is precisely what a single `useEffect` with a cleanup function collapses into.

### Example 3 — `shouldComponentUpdate` / `PureComponent` for a perf bailout

```jsx
import React from 'react';

// Manual control:
class ExpensiveRow extends React.Component {
  shouldComponentUpdate(nextProps) {
    // Only re-render if the value we actually display changed.
    return nextProps.value !== this.props.value;
  }
  render() {
    console.log('rendering row', this.props.id);
    return <li>{this.props.value}</li>;
  }
}

// Equivalent shallow-compare via PureComponent (no hand-written method):
class ExpensiveRowPure extends React.PureComponent {
  render() {
    return <li>{this.props.value}</li>;
  }
}
```

### Example 4 — `getSnapshotBeforeUpdate` to preserve scroll position

This is the *only* lifecycle method without a clean one-liner hooks equivalent (you reconstruct it with refs + `useLayoutEffect`). It captures DOM info *after* render is computed but *before* React mutates the DOM, then hands that value to `componentDidUpdate`.

```jsx
import React from 'react';

class ChatLog extends React.Component {
  listRef = React.createRef();

  getSnapshotBeforeUpdate(prevProps) {
    // New messages arriving? Capture distance from the bottom BEFORE DOM changes.
    if (prevProps.messages.length < this.props.messages.length) {
      const el = this.listRef.current;
      return el.scrollHeight - el.scrollTop; // <- the "snapshot"
    }
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    // `snapshot` is whatever getSnapshotBeforeUpdate returned.
    if (snapshot !== null) {
      const el = this.listRef.current;
      el.scrollTop = el.scrollHeight - snapshot; // keep view pinned
    }
  }

  render() {
    return (
      <ul ref={this.listRef} style={{ overflow: 'auto', height: 300 }}>
        {this.props.messages.map((m) => <li key={m.id}>{m.text}</li>)}
      </ul>
    );
  }
}

export default ChatLog;
```

### Example 5 — An Error Boundary (write this one for real)

Error boundaries **cannot** be expressed with hooks in React 19 — there is no `useErrorBoundary`. So this is the one class you may legitimately author in a modern codebase (or use a library like `react-error-boundary` that wraps a class for you).

```jsx
import React from 'react';

class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  // Render-phase: derive fallback UI state from the thrown error.
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // Commit-phase: side effect — log to your monitoring service.
  componentDidCatch(error, errorInfo) {
    console.error('Caught by boundary:', error, errorInfo.componentStack);
    // logToSentry(error, errorInfo);
  }

  handleReset = () => this.setState({ hasError: false, error: null });

  render() {
    if (this.state.hasError) {
      return (
        <div role="alert">
          <p>Something went wrong: {this.state.error?.message}</p>
          <button onClick={this.handleReset}>Try again</button>
        </div>
      );
    }
    return this.props.children;
  }
}

export default ErrorBoundary;
```

Usage:

```jsx
<ErrorBoundary>
  <Dashboard />
</ErrorBoundary>
```

Boundaries catch errors thrown during **rendering, in lifecycle methods, and in constructors of the tree below them**. They do **not** catch errors in event handlers, in `setTimeout`/async code, during server-side rendering, or thrown by the boundary itself — those you handle with ordinary `try/catch`.

---

## Under the hood

**A class component is a real, persistent instance.** When React first encounters `<Counter />`, it calls `new Counter(props)`. That object lives in React's internal fiber for as long as the component stays mounted at that position in the tree. `this.state` is mutable storage on that object, but **you must never assign to it directly** (`this.state.count = 5`) — React won't know to re-render and won't merge correctly. Only `setState` schedules work.

**`setState` is a request, not a command.** Calling it pushes an "updater" (either your partial object or your function) into a queue on the fiber and schedules a re-render. During the next render pass React drains the queue: starting from the previous state, it applies each updater in order (merging objects, invoking functions with the running result), producing the next `this.state`. This is why the functional form is reliable under batching and the object form is not:

```jsx
// state.count === 0
this.setState({ count: this.state.count + 1 }); // queues "set count to 1"
this.setState({ count: this.state.count + 1 }); // also reads 0 → queues "set count to 1"
// Result: 1, not 2.

this.setState((p) => ({ count: p.count + 1 })); // queues fn → runs on 0 → 1
this.setState((p) => ({ count: p.count + 1 })); // queues fn → runs on 1 → 2
// Result: 2.
```

**Batching.** In React 18+ (which React 19 builds on), `setState` calls are batched **everywhere** — inside event handlers, promises, `setTimeout`, native callbacks — into a single re-render. (Before React 18, only React event handlers batched; async code didn't.) The `this.state` you read synchronously after `setState` is always the pre-update value.

**The two phases.** React's work splits into a **render phase** (compute the next tree — must be pure and may be paused/restarted/discarded by concurrent React) and a **commit phase** (apply changes to the DOM — runs once, synchronously). Mapping the lifecycle onto these phases is the key expert insight:

- *Render phase* (pure, no side effects): `constructor`, `getDerivedStateFromProps`, `shouldComponentUpdate`, `render`, `getDerivedStateFromError`. Because concurrent React may call these more than once or throw them away, putting side effects here is a bug.
- *Commit phase* (DOM is live, side effects allowed): `getSnapshotBeforeUpdate` (just before mutation), then `componentDidMount` / `componentDidUpdate` / `componentWillUnmount` / `componentDidCatch` (after mutation).

This is exactly why the old `componentWillMount`/`componentWillReceiveProps`/`componentWillUpdate` were deprecated: they ran in the render phase but people did side effects in them, which breaks under concurrent rendering.

**Lifecycle ↔ hooks mapping.** Here is the cheat sheet to keep forever:

| Class lifecycle | Hooks equivalent |
| --- | --- |
| `constructor` (init state) | `useState(initial)` / lazy `useState(() => compute())` |
| `render` | the function body's `return` |
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate(prevProps, prevState)` | `useEffect(() => { ... }, [deps])` (runs after deps change) |
| `componentWillUnmount` | the cleanup `return () => { ... }` inside `useEffect` |
| didMount + didUpdate + willUnmount (one concern) | a single `useEffect` with setup + cleanup |
| `shouldComponentUpdate` | wrap component in `React.memo` (+ `useMemo`/`useCallback` for props) |
| `PureComponent` | `React.memo(Component)` |
| `getDerivedStateFromProps` | usually *don't*; compute during render, or use a `key`, or rare `useState`-during-render pattern |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` + a `ref` to read DOM before paint |
| `getDerivedStateFromError` / `componentDidCatch` | **no hook** — keep a class boundary (or `react-error-boundary`) |

> Caveat on the `useEffect` mapping: an empty-array `useEffect` fires after the **first commit**, like `componentDidMount`, but if a dep changes it runs the cleanup *then* the effect again — there is no separate "did update" vs "did mount" distinction. Model effects as *synchronization*, not as discrete lifecycle events (see `08-useeffect-side-effects.md`).

---

## Common mistakes & pitfalls

### Pitfall 1 — The `this` binding problem

A method passed as a callback **loses its `this`** because JS binds `this` at call time, not definition time. React calls your handler as a bare function, so `this` is `undefined` (in strict mode, which modules are).

```jsx
// WRONG — `this` is undefined inside handleClick when fired
class Toggle extends React.Component {
  state = { on: false };
  handleClick() {
    this.setState({ on: !this.state.on }); // TypeError: Cannot read 'setState' of undefined
  }
  render() {
    return <button onClick={this.handleClick}>Toggle</button>;
  }
}
```

Three correct fixes:

```jsx
// RIGHT (A) — arrow class field (preferred, most common in modern class code)
class Toggle extends React.Component {
  state = { on: false };
  handleClick = () => {
    this.setState((p) => ({ on: !p.on })); // arrow captures lexical `this`
  };
  render() { return <button onClick={this.handleClick}>Toggle</button>; }
}

// RIGHT (B) — bind in the constructor
class Toggle extends React.Component {
  constructor(props) {
    super(props);
    this.state = { on: false };
    this.handleClick = this.handleClick.bind(this); // permanent binding
  }
  handleClick() { this.setState((p) => ({ on: !p.on })); }
  render() { return <button onClick={this.handleClick}>Toggle</button>; }
}

// RIGHT (C) — inline arrow in JSX (fine, but allocates a new fn each render)
render() {
  return <button onClick={() => this.handleClick()}>Toggle</button>;
}
```

Option (C) defeats `PureComponent` bailouts on children (new function reference each render), so prefer (A) for handlers passed to memoized children.

### Pitfall 2 — Mutating state directly

```jsx
// WRONG — direct mutation; React never re-renders, merge breaks
this.state.count = this.state.count + 1;
this.state.items.push(newItem);
```

```jsx
// RIGHT — always go through setState with a new value
this.setState((p) => ({ count: p.count + 1 }));
this.setState((p) => ({ items: [...p.items, newItem] })); // new array
```

### Pitfall 3 — Forgetting that `setState` merges (not replaces)

This trips up developers coming *from* hooks. `this.setState` shallow-merges; `useState`'s setter replaces.

```jsx
// state = { user: 'ada', count: 0 }
this.setState({ count: 1 });
// state is now { user: 'ada', count: 1 } — `user` PRESERVED automatically.

// In hooks, the equivalent would DROP user unless you spread:
setState({ ...state, count: 1 });
```

### Pitfall 4 — Reading `this.state` right after `setState`

```jsx
// WRONG — logs the OLD count
this.setState({ count: 5 });
console.log(this.state.count); // not 5 yet (still previous value)
```

```jsx
// RIGHT — use the callback if you must read the committed value
this.setState({ count: 5 }, () => console.log(this.state.count)); // 5
```

### Pitfall 5 — Unguarded `setState` in `componentDidUpdate` → infinite loop

```jsx
// WRONG — every render triggers another render forever
componentDidUpdate() {
  this.setState({ renderedAt: Date.now() });
}
```

```jsx
// RIGHT — guard with a comparison of prev vs current
componentDidUpdate(prevProps) {
  if (prevProps.userId !== this.props.userId) {
    this.fetchUser(this.props.userId);
  }
}
```

### Pitfall 6 — Side effects in the render phase

```jsx
// WRONG — render must be pure; this fires on every (possibly discarded) render
render() {
  fetch('/api/log');           // side effect in render
  document.title = this.props.title; // DOM mutation in render
  return <div />;
}
```

```jsx
// RIGHT — do effects in commit-phase lifecycle methods
componentDidMount() { document.title = this.props.title; }
componentDidUpdate(prev) {
  if (prev.title !== this.props.title) document.title = this.props.title;
}
```

---

## Best practices

- **Do** write new components as functions with hooks. Reach for a class only to build an **error boundary** (or use `react-error-boundary`).
- **Do** use arrow class fields for event handlers to avoid the binding dance and keep references stable.
- **Do** use the **functional updater** `setState(prev => ...)` whenever new state depends on old state.
- **Do** guard `setState` calls inside `componentDidUpdate` with a `prevProps`/`prevState` comparison.
- **Do** mirror every subscription/listener set up in `componentDidMount` with a teardown in `componentWillUnmount`.
- **Don't** mutate `this.state` directly — always go through `setState` with fresh values.
- **Don't** put side effects, `setState`, or DOM reads/writes in `render`, `shouldComponentUpdate`, or `getDerivedStateFromProps`.
- **Don't** write the deprecated `UNSAFE_componentWillMount` / `WillReceiveProps` / `WillUpdate`; refactor them out when you find them.
- **Don't** overuse `getDerivedStateFromProps` — it's almost always replaceable by computing during render or remounting with a `key` prop.
- **Don't** rely on `shouldComponentUpdate` for correctness (e.g. skipping necessary updates) — it's a performance optimization only.

---

## Exercises

1. **Read & bind.** Given the broken `Toggle` below, identify why clicking throws, then fix it three different ways (arrow field, constructor bind, inline arrow).

   ```jsx
   class Toggle extends React.Component {
     state = { on: false };
     handleClick() { this.setState({ on: !this.state.on }); }
     render() { return <button onClick={this.handleClick}>{String(this.state.on)}</button>; }
   }
   ```

2. **Lifecycle tracing.** Write a `Clock` class that displays the current time, updating every second via `setInterval` started in `componentDidMount` and cleared in `componentWillUnmount`. Add a `console.log` in each lifecycle method and predict the order they fire when the clock mounts, ticks 3 times, then unmounts.

3. **Error boundary.** Wrap a `<BuggyButton />` (which throws during render when `this.state.exploded` is true) in an `ErrorBoundary` that shows a fallback and a "reset" button. Verify that an error thrown *inside the click handler* is **not** caught by the boundary, and explain why.

4. **Migration (capstone).** Migrate the `Clock` class from Exercise 2 to a function component with hooks, preserving behavior exactly. Then migrate a component that fetches a user by `this.props.userId` in `componentDidMount` and re-fetches in `componentDidUpdate` when `userId` changes.

5. **PureComponent reasoning.** Explain why passing `onClick={() => this.handleClick()}` to a `PureComponent` child causes it to re-render every time the parent renders, and rewrite it so the child can bail out.

### Solutions

**Solution 1** — `handleClick` is passed as a bare reference, so `this` is `undefined` at call time. Fix (arrow field shown; the other two are in Pitfall 1):

```jsx
class Toggle extends React.Component {
  state = { on: false };
  handleClick = () => this.setState((p) => ({ on: !p.on }));
  render() { return <button onClick={this.handleClick}>{String(this.state.on)}</button>; }
}
```

**Solution 2** — Clock class. Firing order on mount: `constructor` → `render` → `componentDidMount`. Each tick: `render` → `componentDidUpdate` (×3). On unmount: `componentWillUnmount`.

```jsx
import React from 'react';

class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = { time: new Date().toLocaleTimeString() };
    console.log('constructor');
  }
  componentDidMount() {
    console.log('didMount');
    this.timer = setInterval(() => {
      this.setState({ time: new Date().toLocaleTimeString() });
    }, 1000);
  }
  componentDidUpdate() { console.log('didUpdate'); }
  componentWillUnmount() {
    console.log('willUnmount');
    clearInterval(this.timer);
  }
  render() {
    console.log('render');
    return <p>{this.state.time}</p>;
  }
}

export default Clock;
```

**Solution 3** — Error boundary catches render-phase errors but not event-handler errors, because handlers run *outside* React's render/commit work — React isn't on the call stack to catch them. Handle those with `try/catch`.

```jsx
import React from 'react';

class BuggyButton extends React.Component {
  state = { exploded: false };
  render() {
    if (this.state.exploded) throw new Error('Boom during render');
    return <button onClick={() => this.setState({ exploded: true })}>Explode (render)</button>;
  }
}

// Usage — boundary from Example 5 catches the render throw:
// <ErrorBoundary><BuggyButton /></ErrorBoundary>

// This throw is NOT caught — it's in an event handler:
class HandlerThrow extends React.Component {
  handleClick = () => { throw new Error('Boom in handler'); };
  render() { return <button onClick={this.handleClick}>Explode (handler)</button>; }
}
```

**Solution 4** — Clock migrated to hooks. The `constructor` becomes `useState`; `componentDidMount` + `componentWillUnmount` collapse into one `useEffect([])` with cleanup:

```jsx
import { useState, useEffect } from 'react';

function Clock() {
  const [time, setTime] = useState(() => new Date().toLocaleTimeString());

  useEffect(() => {
    const timer = setInterval(() => {
      setTime(new Date().toLocaleTimeString());
    }, 1000);
    return () => clearInterval(timer); // == componentWillUnmount
  }, []); // [] == "run once after mount"

  return <p>{time}</p>;
}

export default Clock;
```

The fetch-on-mount-and-on-userId-change class → hook migration. The class needs `componentDidMount` *and* a guarded `componentDidUpdate`; the hook needs just one effect keyed on `userId`:

```jsx
// CLASS (before)
class UserCard extends React.Component {
  state = { user: null };
  componentDidMount() { this.load(this.props.userId); }
  componentDidUpdate(prev) {
    if (prev.userId !== this.props.userId) this.load(this.props.userId);
  }
  async load(id) {
    const res = await fetch(`/api/users/${id}`);
    this.setState({ user: await res.json() });
  }
  render() { return <p>{this.state.user?.name ?? 'Loading…'}</p>; }
}
```

```jsx
// FUNCTION + HOOKS (after) — one effect replaces didMount + didUpdate
import { useState, useEffect } from 'react';

function UserCard({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let cancelled = false;            // guard against out-of-order responses
    setUser(null);                    // reset while loading the new id
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => { if (!cancelled) setUser(data); });
    return () => { cancelled = true; }; // cleanup on userId change / unmount
  }, [userId]);                          // re-run whenever userId changes

  return <p>{user?.name ?? 'Loading…'}</p>;
}

export default UserCard;
```

Note the hooks version adds a `cancelled` guard the class lacked — the effect cleanup runs before each re-fetch, fixing a race the class quietly had. See `08-useeffect-side-effects.md` for the full data-fetching effect pattern and `15-data-fetching.md` for production-grade approaches.

**Solution 5** — `() => this.handleClick()` creates a **new function object on every render**, so the child's shallow prop comparison always sees a "changed" `onClick` and re-renders. Pass a stable reference instead:

```jsx
class Parent extends React.Component {
  handleClick = () => { /* ... */ };       // one stable reference for the instance's life
  render() { return <PureChild onClick={this.handleClick} />; } // child can now bail out
}
```

---

## Key takeaways

- Class components extend `React.Component`, define `render()`, read props via `this.props`, and hold state in `this.state`. They are **legacy**; hooks (lessons 07–10) are the modern default — you read classes far more than you write them.
- `this.setState` is **asynchronous/batched**, **shallow-merges** the object you pass, and offers a **functional updater** `prev => next` that you should use whenever new state depends on old.
- The `this`-binding problem is solved with **arrow class fields** (preferred), **constructor `.bind`**, or **inline arrows** (with a perf caveat).
- The lifecycle splits into a pure **render phase** (`constructor`, `getDerivedStateFromProps`, `shouldComponentUpdate`, `render`) and a side-effect-safe **commit phase** (`getSnapshotBeforeUpdate`, `componentDidMount`/`DidUpdate`/`WillUnmount`).
- Set up subscriptions in `componentDidMount`, react (guarded) in `componentDidUpdate`, and always tear down in `componentWillUnmount`.
- `PureComponent` ≈ `React.memo`: a shallow-prop/state comparison bailout. `shouldComponentUpdate` is the manual version — a performance tool, never a correctness one.
- **Error boundaries** (`getDerivedStateFromError` + `componentDidCatch`) are the one thing that still *requires* a class in React 19; they catch render/lifecycle errors but **not** event-handler or async errors.
- Migration shorthand: `constructor` → `useState`; `didMount`+`willUnmount` → one `useEffect([])` with cleanup; guarded `didUpdate` → `useEffect([deps])`; `PureComponent` → `React.memo`.

---

## Further reading

- React docs — [`Component` API reference](https://react.dev/reference/react/Component) (the full legacy class API, every lifecycle method documented).
- React docs — [`PureComponent`](https://react.dev/reference/react/PureComponent).
- React docs — [How to migrate from classes to function components / "You Might Not Need an Effect"](https://react.dev/learn/you-might-not-need-an-effect) and the [`Component` page's migration notes](https://react.dev/reference/react/Component#migrating-a-component-with-lifecycle-methods-from-a-class-to-a-function).
- React docs — [`Component.componentDidCatch`](https://react.dev/reference/react/Component#componentdidcatch) and the broader [error-boundary guidance](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary).
- Library — [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary) by Brian Vaughn (the idiomatic way to use boundaries without hand-writing a class).
- Legacy React docs — [State and Lifecycle](https://legacy.reactjs.org/docs/state-and-lifecycle.html) and the classic [interactive lifecycle diagram](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/) (invaluable for visualizing firing order).
- Internals — [React 18 automatic batching](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching) (why `setState` batches everywhere now).
