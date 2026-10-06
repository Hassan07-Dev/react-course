# 20 — Advanced Component Patterns

> **What you'll learn:** the reusable composition patterns that separate a beginner who can build *a* component from an engineer who can design a *component API* — error boundaries, portals, refs into components, HOCs, render props, compound components, controlled/uncontrolled design, providers, polymorphism, and slots.

By the end of this lesson you will be able to:

- Catch render-time crashes with an **Error Boundary** and recover gracefully with the `react-error-boundary` package.
- Render UI **outside the DOM hierarchy** of its parent with `createPortal` (modals, tooltips, overlays) and understand how event bubbling still follows the React tree.
- Expose and consume refs to components using **React 19's ref-as-a-prop** (and the legacy `forwardRef`), plus `useImperativeHandle` for a deliberate imperative API.
- Recognize the **Higher-Order Component** and **render-prop** patterns, when they're still useful, and what replaced them.
- Build **compound components** (`<Tabs><Tab/></Tabs>`) that share implicit state via Context.
- Design **controlled vs uncontrolled** components and the provider pattern intentionally.
- Write **polymorphic** components with an `as` prop and compose **slots** through `children` and named props.
- Tell which patterns are *current* (2025/React 19) and which are *legacy but still in the wild*.

---

## Prerequisites

This is a synthesis lesson — it leans on most of what came before. You should be solid on:

- Function components, props, and `children` (see `04-components-and-props.md`).
- State with `useState` and the rules of updates (see `07-state-usestate.md`).
- `useEffect` and cleanup (see `08-useeffect-side-effects.md`).
- `useRef`, `useContext`, `useId` and the broader hook surface (see `09-hooks-deep-dive.md`).
- Custom hooks and reference stability (see `10-custom-hooks.md`).
- Context for sharing state down a tree (a dedicated Context lesson follows; the compound-component section here is a practical preview).

If Context feels shaky, focus extra attention on the **Compound components** and **Provider pattern** sections — they're the payoff of understanding Context well.

---

## Concept

Up to now you've mostly written *leaf* components: a component takes props, renders markup, manages a little local state. Advanced patterns are about something different — **designing the seam between components** so they compose cleanly. They answer questions like:

- "How do I let a parent reach *into* a child to call `.focus()`?"
- "How do I render a modal that visually sits at the top of the page even though it's declared deep inside a sidebar?"
- "How do I build a `<Tabs>` API that feels like writing HTML, where `<Tab>` just *knows* which panel it controls?"
- "What happens to my whole app when one component throws during render?"

A useful mental model: **leaf components are nouns; patterns are grammar.** They define how components attach to each other. Most patterns exist to solve one of three problems:

1. **Sharing logic** without copy-paste (custom hooks mostly won this — HOCs and render props are the older answers).
2. **Sharing state implicitly** between sibling components that should feel like one unit (compound components + Context).
3. **Breaking out of the tree** — escaping the React render tree (portals) or the React data flow (refs, error boundaries).

The rest of this lesson is a tour of each pattern with one solid example apiece, labeled **current** or **legacy** so you know what to reach for in 2025.

---

## Syntax & API

An annotated reference of every API this lesson touches. Skim it now; the Examples section uses each one in context.

```jsx
// --- Error Boundaries (class-only; no hook equivalent exists in React 19) ---
class ErrorBoundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) {  // render fallback (no side effects)
    return { error };
  }
  componentDidCatch(error, info) {          // log side effect; info.componentStack
    logToService(error, info.componentStack);
  }
  render() {
    return this.state.error ? this.props.fallback : this.props.children;
  }
}
```

```jsx
// --- Portals ---
import { createPortal } from 'react-dom';
createPortal(children, domNode, key?);
// Renders `children` into `domNode` (any DOM element), keeping React-tree context.
```

```jsx
// --- Refs into components ---
// React 19 (CURRENT): ref is just a normal prop on function components
function Input({ ref, ...props }) { return <input ref={ref} {...props} />; }

// Legacy (still supported): forwardRef wraps the component
const Input = React.forwardRef(function Input(props, ref) {
  return <input ref={ref} {...props} />;
});

// useImperativeHandle: expose a curated imperative API instead of the raw node
useImperativeHandle(ref, () => ({ focus() {/*...*/} }), [/* deps */]);
```

```jsx
// --- Context (powers compound components & providers) ---
const Ctx = React.createContext(defaultValue);
<Ctx value={...}>...</Ctx>   // React 19: <Ctx> is a valid provider; .Provider still works
const value = React.useContext(Ctx);
```

| API | Module | Status | Use it for |
| --- | --- | --- | --- |
| `getDerivedStateFromError` / `componentDidCatch` | class component | **current** (only option) | Catching render errors |
| `react-error-boundary` (`ErrorBoundary`, `useErrorBoundary`) | npm package | **current, recommended** | Hook-friendly boundaries + reset |
| `createPortal` | `react-dom` | **current** | Modals, tooltips, overlays |
| ref-as-a-prop | core | **current (React 19)** | Refs to function components |
| `forwardRef` | core | **legacy but supported** | Refs to components pre-19 |
| `useImperativeHandle` | core | **current** | Curated imperative API |
| HOC (`withX`) | pattern | **legacy** | Wrapping for cross-cutting concerns |
| Render props / function-as-children | pattern | **legacy-ish** | Sharing logic via JSX (mostly replaced by hooks) |
| Compound components | pattern + Context | **current** | Cohesive multi-part widgets |
| Polymorphic `as` prop | pattern | **current** | Design-system primitives |

---

## Examples

### 1. Error Boundary — the class version (current, because it's the only built-in)

There is still **no hook** that catches render errors. Error Boundaries are the one place you must write a class in modern React. A boundary catches errors thrown during **rendering, in lifecycle methods, and in constructors of the components below it**. It does **not** catch errors in event handlers, async code (`setTimeout`, promises), or the boundary's own render — those need a `try/catch`.

```jsx
// ErrorBoundary.jsx
import { Component } from 'react';

export class ErrorBoundary extends Component {
  state = { error: null };

  // Called during the "render" phase. Return the next state. NO side effects here.
  static getDerivedStateFromError(error) {
    return { error };
  }

  // Called during the "commit" phase. Side effects (logging) are allowed here.
  componentDidCatch(error, info) {
    console.error('Caught by boundary:', error, info.componentStack);
  }

  reset = () => this.setState({ error: null });

  render() {
    if (this.state.error) {
      // `fallback` can be a render function so it can show the error + a retry.
      return this.props.fallback(this.state.error, this.reset);
    }
    return this.props.children;
  }
}
```

```jsx
// App.jsx
import { ErrorBoundary } from './ErrorBoundary.jsx';

function Boom() {
  throw new Error('Render exploded');
}

export default function App() {
  return (
    <ErrorBoundary
      fallback={(error, reset) => (
        <div role="alert">
          <p>Something broke: {error.message}</p>
          <button onClick={reset}>Try again</button>
        </div>
      )}
    >
      <Boom />
    </ErrorBoundary>
  );
}
```

Place boundaries **strategically**, not around every component. A good rule: one near the root (so a crash shows a friendly page instead of a blank screen), plus boundaries around independent regions (a dashboard widget, a comment thread) so one failing region doesn't take down the others.

### 2. Error Boundary — the `react-error-boundary` package (current, recommended)

Writing the class by hand gets tedious and the reset logic is fiddly. The community-standard `react-error-boundary` package gives you a polished boundary plus a hook to *trigger* a boundary from event handlers/async code (which boundaries normally can't catch).

```bash
npm install react-error-boundary
```

```jsx
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

function Fallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>Oops: {error.message}</p>
      <button onClick={resetErrorBoundary}>Retry</button>
    </div>
  );
}

function Profile({ userId }) {
  const { showBoundary } = useErrorBoundary(); // forward async errors into a boundary
  // ...
  async function load() {
    try {
      await fetch(`/api/user/${userId}`);
    } catch (err) {
      showBoundary(err); // <- now the boundary catches an error from async code
    }
  }
  return <button onClick={load}>Load</button>;
}

export default function App({ userId }) {
  return (
    <ErrorBoundary
      FallbackComponent={Fallback}
      onReset={() => {/* clear cache, refetch, etc. */}}
      resetKeys={[userId]}  // auto-reset when userId changes
    >
      <Profile userId={userId} />
    </ErrorBoundary>
  );
}
```

`resetKeys` is the killer feature: when any value in that array changes, the boundary clears its error automatically — so navigating to a new user retries instead of staying stuck on the error screen.

### 3. Portal — a modal that escapes `overflow: hidden`

A portal renders children into a *different DOM node* while keeping them in the same **React tree**. This solves the classic CSS trap: a modal declared inside a `position: relative; overflow: hidden` card gets clipped. Render it through a portal into `document.body` and it floats free — while still receiving props and Context from its React parent.

```jsx
// Modal.jsx
import { createPortal } from 'react-dom';
import { useEffect } from 'react';

export function Modal({ open, onClose, children }) {
  // Close on Escape — an effect, with cleanup (see 08-useeffect-side-effects.md).
  useEffect(() => {
    if (!open) return;
    function onKey(e) {
      if (e.key === 'Escape') onClose();
    }
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [open, onClose]);

  if (!open) return null;

  return createPortal(
    <div className="overlay" onClick={onClose}>
      {/* stopPropagation so clicking the dialog doesn't close it */}
      <div className="dialog" role="dialog" aria-modal="true"
           onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body, // <- the modal lives here in the real DOM
  );
}
```

```jsx
// Usage — note the modal is declared deep inside, but renders to body
function Card() {
  const [open, setOpen] = useState(false);
  return (
    <div style={{ overflow: 'hidden', borderRadius: 8 }}>
      <button onClick={() => setOpen(true)}>Open</button>
      <Modal open={open} onClose={() => setOpen(false)}>
        <h2>I am not clipped by the card.</h2>
      </Modal>
    </div>
  );
}
```

**The crucial subtlety: events bubble through the React tree, not the DOM tree.** Even though the modal's markup lands in `document.body`, an `onClick` *on the `<Card>`* would fire if you clicked inside the modal, because React routes synthetic events along the component hierarchy. That's usually what you want, but it can surprise you — clicking inside a portaled dropdown can trigger a "click outside to close" handler on an ancestor if you wired it through React rather than native DOM listeners.

### 4. Refs to components — React 19 ref-as-a-prop vs legacy `forwardRef`

You can't put a `ref` on a custom function component and expect it to reach the DOM unless the component forwards it. **In React 19, `ref` is just a regular prop** — destructure it and pass it down. No wrapper needed.

```jsx
// React 19 — CURRENT. ref is a normal prop.
function TextField({ label, ref, ...props }) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
}

function Form() {
  const inputRef = useRef(null);
  return (
    <>
      <TextField label="Email" ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>Focus email</button>
    </>
  );
}
```

```jsx
// Legacy (React 16.3–18, still works in 19). You'll see this everywhere in older code.
import { forwardRef } from 'react';

const TextField = forwardRef(function TextField({ label, ...props }, ref) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
});
```

### 5. `useImperativeHandle` — expose a *curated* API, not the raw node

Sometimes you don't want callers touching the raw DOM node; you want to expose a small, intentional imperative surface. `useImperativeHandle` lets the parent's `ref.current` be an object *you* define.

```jsx
import { useRef, useImperativeHandle } from 'react';

function FancyInput({ ref }) {                 // React 19 ref-as-a-prop
  const innerRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => innerRef.current.focus(),
    clear: () => { innerRef.current.value = ''; },
  }), []); // empty deps: the handle object is stable

  return <input ref={innerRef} />;
}

function Parent() {
  const apiRef = useRef(null);
  return (
    <>
      <FancyInput ref={apiRef} />
      <button onClick={() => apiRef.current.focus()}>Focus</button>
      <button onClick={() => apiRef.current.clear()}>Clear</button>
    </>
  );
}
```

Use this sparingly. Imperative handles are for genuinely imperative actions — `focus()`, `scrollIntoView()`, `play()`, `open()` on a `<dialog>` — not for things that should be props/state.

### 6. Higher-Order Component (legacy) and Render Props (legacy-ish)

A **HOC** is a function that takes a component and returns a new, enhanced component. Before hooks, this was *the* way to share cross-cutting logic.

```jsx
// withLogger.jsx — a HOC. Convention: name it withX.
import { useEffect } from 'react';

export function withLogger(Wrapped) {
  return function WithLogger(props) {
    useEffect(() => {
      console.log(`${Wrapped.name} mounted`);
    }, []);
    return <Wrapped {...props} />; // forward all props through
  };
}
const LoggedButton = withLogger(Button);
```

A **render prop** (and its sibling, *function-as-children*) passes a function that the component calls to render its output, handing back internal state:

```jsx
// MouseTracker.jsx — function-as-children render prop
import { useState } from 'react';

function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>
      {children(pos)}            {/* children is a function */}
    </div>
  );
}

// Usage
<MouseTracker>
  {({ x, y }) => <p>{x}, {y}</p>}
</MouseTracker>
```

**Both are legacy for *logic sharing*** — a custom hook (`useMouse()`, `useLogger()`) does the same job with no wrapper-component nesting and no "wrapper hell" in DevTools (see `10-custom-hooks.md`). HOCs still earn their keep for a few things hooks can't do: wrapping a component to inject props you don't control the call site for, library boundaries (`React.memo`, `connect`), and conditionally swapping the rendered component. Render-as-children is still great for *layout* APIs (e.g. a `<List>` that calls `renderItem(item)`), where the function isn't sharing logic but customizing output.

### 7. Compound components — `<Tabs>` via Context (current)

Compound components let related parts share state implicitly so the consumer writes declarative, HTML-like markup. The shared state lives in a Context that the parent provides and the children consume.

```jsx
// Tabs.jsx
import { createContext, useContext, useState } from 'react';

const TabsContext = createContext(null);

export function Tabs({ defaultValue, children }) {
  const [active, setActive] = useState(defaultValue);
  return (
    <TabsContext value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsContext>
  );
}

export function TabList({ children }) {
  return <div role="tablist">{children}</div>;
}

export function Tab({ value, children }) {
  const { active, setActive } = useContext(TabsContext);
  const selected = active === value;
  return (
    <button role="tab" aria-selected={selected} onClick={() => setActive(value)}>
      {children}
    </button>
  );
}

export function TabPanel({ value, children }) {
  const { active } = useContext(TabsContext);
  return active === value ? <div role="tabpanel">{children}</div> : null;
}
```

```jsx
// Usage — clean, declarative, no prop-drilling of `active`
<Tabs defaultValue="profile">
  <TabList>
    <Tab value="profile">Profile</Tab>
    <Tab value="settings">Settings</Tab>
  </TabList>
  <TabPanel value="profile">Your profile…</TabPanel>
  <TabPanel value="settings">Your settings…</TabPanel>
</Tabs>
```

The magic is that `<Tab>` and `<TabPanel>` never receive `active` as a prop — they *reach up* through Context. The consumer just composes pieces. This is how Radix UI, Headless UI, and Reach UI structure their primitives.

### 8. Controlled vs uncontrolled + polymorphic `as` (current)

A well-designed reusable component supports **both** modes. *Controlled*: the parent owns the value via `value` + `onChange`. *Uncontrolled*: the component owns its own state, optionally seeded by `defaultValue`.

```jsx
function Rating({ value, defaultValue = 0, onChange }) {
  const isControlled = value !== undefined;
  const [internal, setInternal] = useState(defaultValue);
  const current = isControlled ? value : internal;

  function set(v) {
    if (!isControlled) setInternal(v); // own the state only when uncontrolled
    onChange?.(v);                      // always notify
  }

  return [1, 2, 3, 4, 5].map((n) => (
    <button key={n} aria-pressed={n <= current} onClick={() => set(n)}>★</button>
  ));
}
```

A **polymorphic** component renders a different element/component via an `as` prop — essential for design-system primitives (`<Box as="section">`, `<Text as="h1">`):

```jsx
function Box({ as: Component = 'div', ...props }) {
  return <Component {...props} />;
}
// <Box as="button" onClick={...}>Click</Box>  -> renders a real <button>
// <Box as={Link} to="/home">Home</Box>        -> renders a router Link
```

---

## Under the hood

**Error boundaries are class-only on purpose.** The error-catching mechanism hooks into React's commit-phase machinery in a way that doesn't map onto the hook model — React needs a stable lifecycle method (`getDerivedStateFromError`) it can call *during reconciliation* before the component's own hooks would even run. The React team has signaled a hook/component API may come eventually, but as of React 19 the class is the only built-in. When a render throws, React unwinds the tree upward looking for the nearest boundary, discards the subtree that failed, and re-renders the boundary with its fallback. If no boundary exists, **the entire root unmounts** — a blank page — which is why a root-level boundary is essential.

**Portals keep React-tree context, break DOM-tree position.** Internally a portal is just a special fiber whose children commit to a different container node. React's event system is built on a *synthetic* event layer that propagates along the **fiber tree**, not the DOM tree. So `event.stopPropagation()` and bubbling behave as if the portal content were a normal child — Context, theme providers, and event delegation all flow through. The only thing that changes is the physical DOM insertion point. This is why a portaled tooltip still reads your `ThemeContext`.

**Ref-as-a-prop is a real simplification, not sugar.** Pre-19, `ref` (and `key`) were *reserved* — stripped from `props` and handled specially, which is why `forwardRef` had to exist to thread `ref` through. React 19 stopped reserving `ref` for function components, so it arrives in `props` like anything else. `forwardRef` still works (and is needed for libraries supporting older React), but new code shouldn't reach for it. React publishes a `react/19/` codemod family — run the whole suite with `npx codemod@latest react/19/migration-recipe` — though note these codemods cover things like `ReactDOM.render` and string refs; there is no dedicated `forwardRef`-to-`ref`-prop codemod, so that conversion is manual (lint rules help flag it).

**`useImperativeHandle` runs during commit** and recomputes the handle object when its dependency array changes — exactly like `useMemo` for an imperative API. Passing `[]` gives you a stable handle; omitting deps recreates it every render (usually wrong).

**Compound components depend on Context identity.** Every `useContext(TabsContext)` consumer re-renders when the provider's `value` changes *by reference*. If you inline `value={{ active, setActive }}` you create a new object every render — fine here because the provider re-renders only when `active` changes anyway, but in hot paths memoize the value with `useMemo` to avoid waking up unrelated consumers (see the memoization discussion in `09-hooks-deep-dive.md`).

---

## Common mistakes & pitfalls

**Mistake 1 — expecting a boundary to catch event-handler or async errors.**

```jsx
// WRONG: this throw is NOT caught by any Error Boundary
function Save() {
  return <button onClick={() => { throw new Error('nope'); }}>Save</button>;
}
```

```jsx
// RIGHT: handle it locally, or forward to a boundary via useErrorBoundary
function Save() {
  const { showBoundary } = useErrorBoundary();
  return (
    <button onClick={() => {
      try { riskyThing(); }
      catch (e) { showBoundary(e); }
    }}>Save</button>
  );
}
```
*Why:* boundaries only catch errors thrown **during rendering/lifecycle**. Event handlers run outside that phase.

**Mistake 2 — doing side effects in `getDerivedStateFromError`.**

```jsx
// WRONG: getDerivedStateFromError must be pure; logging here is unreliable
static getDerivedStateFromError(error) {
  fetch('/log', { method: 'POST', body: error.message }); // ❌ side effect in render phase
  return { error };
}
```

```jsx
// RIGHT: derive state in getDerivedStateFromError, log in componentDidCatch
static getDerivedStateFromError(error) { return { error }; }
componentDidCatch(error, info) { fetch('/log', { method: 'POST', /* ... */ }); }
```

**Mistake 3 — a "click outside to close" handler that fires inside a portal.**

```jsx
// WRONG: clicking inside a portaled dropdown still closes it, because the
// React onClick on the wrapper sees the event bubble through the React tree.
<div onClick={close}>
  {createPortal(<Menu />, document.body)}
</div>
```

```jsx
// RIGHT: detect outside-clicks with a native listener that checks the real DOM node
useEffect(() => {
  function onDocClick(e) {
    if (menuRef.current && !menuRef.current.contains(e.target)) close();
  }
  document.addEventListener('mousedown', onDocClick);
  return () => document.removeEventListener('mousedown', onDocClick);
}, []);
```
*Why:* React events bubble through the component tree; native DOM `contains()` checks the actual DOM, where the portal content lives elsewhere.

**Mistake 4 — switching a component between controlled and uncontrolled.**

```jsx
// WRONG: value starts undefined (uncontrolled) then becomes a string (controlled)
<input value={state.name} onChange={...} />  // state.name is undefined at first
// React warns: "A component is changing an uncontrolled input to be controlled."
```

```jsx
// RIGHT: pick one. For controlled, seed with a defined value.
const [name, setName] = useState('');         // not undefined
<input value={name} onChange={(e) => setName(e.target.value)} />
```

**Mistake 5 — reaching for a HOC/render prop when a hook is cleaner.**

```jsx
// WRONG (in 2025): wrapper hell to share window-size logic
withWindowSize(withTheme(withAuth(Dashboard)));
```

```jsx
// RIGHT: compose hooks — flat, debuggable, typed
function Dashboard() {
  const size = useWindowSize();
  const theme = useTheme();
  const user = useAuth();
  // ...
}
```

---

## Best practices

- **Do** put an Error Boundary at the root *and* around independent regions; **don't** wrap every component.
- **Do** use `react-error-boundary` with `resetKeys` for production; **don't** hand-roll the class unless you have a reason.
- **Do** render modals/tooltips/menus through `createPortal`; **don't** fight `overflow`/`z-index` with CSS hacks.
- **Do** remember portal events bubble through the **React** tree; use native listeners for true "outside click" detection.
- **Do** use **ref-as-a-prop** in new React 19 code; **don't** reach for `forwardRef` unless supporting older React.
- **Do** expose a **curated** imperative API with `useImperativeHandle` (focus/scroll/play); **don't** leak raw nodes or push state through it.
- **Do** prefer **custom hooks** over HOCs/render-props for logic sharing; **do** keep HOCs for library boundaries and render-as-children for *layout* customization.
- **Do** build **compound components** for multi-part widgets and memoize the Context value in hot paths.
- **Do** support **both controlled and uncontrolled** modes and never silently switch a component between them.
- **Do** reach for a polymorphic `as` prop for design-system primitives; **don't** overuse it where a plain prop is clearer.

---

## Exercises

1. **Resilient widget.** Wrap a `<ChartWidget>` that sometimes throws during render in `react-error-boundary`. Show a fallback with a "Retry" button and a `resetKeys` tied to a `dataVersion` prop so the chart auto-recovers when new data arrives.
2. **Accessible modal.** Build a `Modal` with `createPortal` that closes on Escape and on overlay click (but not on dialog click), and that does **not** trigger a "click outside" handler placed on an ancestor.
3. **Imperative video.** Create a `<Video ref={...} />` (React 19 ref-as-a-prop) that exposes `play()`, `pause()`, and `seek(seconds)` via `useImperativeHandle`, hiding the raw `<video>` element.
4. **Compound accordion.** Build `<Accordion>` / `<AccordionItem>` / `<AccordionHeader>` / `<AccordionPanel>` sharing open-state through Context, allowing only one item open at a time.
5. **Polymorphic + controlled.** Build a `<Toggle as="button" pressed={...} onPressedChange={...} />` that is polymorphic *and* supports both controlled and uncontrolled use.

### Solutions

```jsx
// Exercise 1 — Resilient widget
import { ErrorBoundary } from 'react-error-boundary';

function Fallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>Chart failed: {error.message}</p>
      <button onClick={resetErrorBoundary}>Retry</button>
    </div>
  );
}

function Dashboard({ dataVersion }) {
  return (
    <ErrorBoundary FallbackComponent={Fallback} resetKeys={[dataVersion]}>
      <ChartWidget version={dataVersion} />
    </ErrorBoundary>
  );
}
```

```jsx
// Exercise 2 — Accessible modal (see Example 3; the key is stopPropagation +
// using onClick on the overlay only). Native outside-click detection is unnecessary
// here because the overlay click handler is the close trigger.
function Modal({ open, onClose, children }) {
  useEffect(() => {
    if (!open) return;
    const onKey = (e) => e.key === 'Escape' && onClose();
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [open, onClose]);
  if (!open) return null;
  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div role="dialog" aria-modal="true" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body,
  );
}
```

```jsx
// Exercise 3 — Imperative video
import { useRef, useImperativeHandle } from 'react';

function Video({ ref, ...props }) {
  const el = useRef(null);
  useImperativeHandle(ref, () => ({
    play: () => el.current.play(),
    pause: () => el.current.pause(),
    seek: (s) => { el.current.currentTime = s; },
  }), []);
  return <video ref={el} {...props} />;
}

function Player() {
  const v = useRef(null);
  return (
    <>
      <Video ref={v} src="/clip.mp4" />
      <button onClick={() => v.current.play()}>Play</button>
      <button onClick={() => v.current.seek(30)}>Skip to 0:30</button>
    </>
  );
}
```

```jsx
// Exercise 4 — Compound accordion (single-open)
import { createContext, useContext, useState } from 'react';

const AccordionCtx = createContext(null);
const ItemCtx = createContext(null);

function Accordion({ children }) {
  const [openId, setOpenId] = useState(null);
  return (
    <AccordionCtx value={{ openId, setOpenId }}>{children}</AccordionCtx>
  );
}
function AccordionItem({ id, children }) {
  return <ItemCtx value={id}>{children}</ItemCtx>;
}
function AccordionHeader({ children }) {
  const { openId, setOpenId } = useContext(AccordionCtx);
  const id = useContext(ItemCtx);
  return (
    <button aria-expanded={openId === id}
            onClick={() => setOpenId(openId === id ? null : id)}>
      {children}
    </button>
  );
}
function AccordionPanel({ children }) {
  const { openId } = useContext(AccordionCtx);
  const id = useContext(ItemCtx);
  return openId === id ? <div role="region">{children}</div> : null;
}
```

```jsx
// Exercise 5 — Polymorphic + controlled toggle
import { useState } from 'react';

function Toggle({ as: Component = 'button', pressed, defaultPressed = false,
                  onPressedChange, ...props }) {
  const isControlled = pressed !== undefined;
  const [internal, setInternal] = useState(defaultPressed);
  const current = isControlled ? pressed : internal;
  function toggle() {
    const next = !current;
    if (!isControlled) setInternal(next);
    onPressedChange?.(next);
  }
  return <Component aria-pressed={current} onClick={toggle} {...props} />;
}
```

---

## Key takeaways

- **Error Boundaries are class-only** (`getDerivedStateFromError` + `componentDidCatch`); use `react-error-boundary` for a hook-friendly API, `resetKeys`, and `useErrorBoundary` to forward async errors. Boundaries never catch event-handler or async errors.
- **Portals** (`createPortal`) move children to a different DOM node but keep them in the React tree — so Context and event bubbling follow the **React** hierarchy, not the DOM.
- **React 19 makes `ref` a normal prop** — `forwardRef` is legacy. Use `useImperativeHandle` to expose a deliberate imperative API.
- **HOCs and render props are legacy for logic sharing** — custom hooks replaced them. HOCs survive at library boundaries; function-as-children survives for layout/render customization.
- **Compound components** share implicit state through Context, giving consumers clean declarative markup.
- **Design components to support both controlled and uncontrolled modes**, and never switch between them at runtime.
- **Polymorphic `as` props** and **slot/children composition** are the backbone of flexible design-system primitives.

---

## Further reading

- React docs — [Error Boundaries / `Component` reference](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- React docs — [`createPortal`](https://react.dev/reference/react-dom/createPortal)
- React docs — [Passing refs / `ref` as a prop (React 19)](https://react.dev/reference/react-dom/components/common#ref-callback) and [`forwardRef`](https://react.dev/reference/react/forwardRef)
- React docs — [`useImperativeHandle`](https://react.dev/reference/react/useImperativeHandle)
- React docs — [`useContext`](https://react.dev/reference/react/useContext) and [Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context)
- React 19 release notes — [react.dev blog: React 19](https://react.dev/blog/2024/12/05/react-19) (ref-as-a-prop, deprecation of legacy patterns)
- [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary) (the de-facto package)
- Kent C. Dodds — [Compound Components with Context](https://kentcdodds.com/blog/compound-components-with-react-hooks) and [How to use React Context effectively](https://kentcdodds.com/blog/how-to-use-react-context-effectively)
