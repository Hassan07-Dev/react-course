# 25 — Best Practices, Architecture & Accessibility

> **What you'll learn:** how to structure, design, and harden a React 19 codebase so it stays maintainable, accessible, and secure as it grows from a toy app into a production system.

By the end of this lesson you will be able to:

- Choose between **feature-based** and **type-based** project structure, and apply **colocation** deliberately.
- Use **barrel files** where they help and recognize where they hurt (bundle bloat, circular imports).
- Design **small, single-responsibility components** using **composition over configuration** and clean prop contracts.
- Decide **where logic lives** (component, custom hook, or service) and resist **premature abstraction**.
- Implement a coherent **error-handling strategy** combining error boundaries with async error handling.
- Build **accessible** UIs: semantic HTML, labels, focus management, keyboard navigation, contrast, and automated a11y testing.
- Defend against **XSS**, handle **tokens/secrets** correctly, and keep **dependencies** healthy.
- Run a pragmatic **performance and code-review checklist** specific to React.

---

## Prerequisites

This is a capstone-style lesson that ties earlier threads together. You should be comfortable with:

- Components and props — see `04-components-and-props.md`.
- State and effects — see `07-state-usestate.md` and `08-useeffect-side-effects.md`.
- Custom hooks — see `10-custom-hooks.md` (the primary tool for extracting logic).
- Context — see `12-context-api.md` (for understanding when *not* to reach for it).
- Performance — see `13-performance.md` (memoization, the compiler, render cost).
- Data fetching — see `15-data-fetching.md` and state management — see `16-state-management.md`.
- Testing — see `19-testing.md` (we extend it here with a11y testing).

You do not need TypeScript, but a few notes call out where types prevent whole classes of bugs.

---

## Concept

Best practices are not a style guide you memorize — they are **the accumulated answers to one question: how do we keep the cost of change low?** Every decision in this lesson trades a little effort now to avoid a lot of pain later. The skill is knowing *when* the trade is worth it.

Think of a codebase as a **city**. Early on, a few buildings on one street are fine. As it grows, you need zoning (where things go), addressing (naming), roads (data flow), and building codes (accessibility, security). Without those, every new feature requires archaeology before construction. Good architecture is mostly about making **"where does this go?"** and **"what touches this?"** answerable in seconds.

Three principles run underneath everything:

1. **Colocation** — things that change together should live together. The closer a file is to where it's used, the cheaper it is to find, change, and delete.
2. **Single responsibility** — a unit (component, hook, module) should have one reason to change. When a file has two jobs, every edit risks breaking the unrelated one.
3. **Least power / least abstraction** — reach for the simplest mechanism that solves *today's* problem. Abstraction is a loan against future flexibility; you pay interest in indirection until you actually need it.

Accessibility and security are not separate "extra" concerns bolted on at the end — they are part of *correctness*. An inaccessible button is a broken button for a real subset of users; an XSS hole is a correctness bug that ships your users' sessions to an attacker. Treat them like you treat a failing test.

---

## Syntax & API

There's little new *API* here; this lesson is about applying APIs you know. The few primitives that matter:

```jsx
// Error boundary — MUST be a class component (no hook equivalent in React 19).
class ErrorBoundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) {
    return { error }; // update state so the next render shows fallback UI
  }
  componentDidCatch(error, info) {
    // info.componentStack — log to your error service here
    reportError(error, info);
  }
  render() {
    if (this.state.error) return this.props.fallback;
    return this.props.children;
  }
}
```

```jsx
// Composition primitive: children + named "slots" passed as props.
function Card({ header, children, footer }) {
  return (
    <section className="card">
      {header && <header>{header}</header>}
      <div className="card__body">{children}</div>
      {footer && <footer>{footer}</footer>}
    </section>
  );
}
```

```jsx
// The escape hatch you must understand to use safely.
<div dangerouslySetInnerHTML={{ __html: sanitizedHtml }} />
// Key facts:
//  - takes an OBJECT with a single __html string key
//  - bypasses React's automatic escaping → only ever pass SANITIZED html
```

```jsx
// Accessibility-relevant DOM you'll lean on:
<label htmlFor="email">Email</label>      // associates label with control
<input id="email" aria-describedby="email-help" />
<button aria-label="Close" onClick={close}>×</button>  // accessible name for icon button
<div role="alert">…</div>                  // announced by screen readers immediately
```

```jsx
// Focus management primitive: a ref + an effect.
const ref = useRef(null);
useEffect(() => { ref.current?.focus(); }, []);
```

These are the building blocks. The rest of the lesson is judgment about how to assemble them.

---

## Examples

### Example 1 — From type-based to feature-based structure

A common starter layout groups files by **type**:

```
src/
  components/   Button.jsx  Modal.jsx  UserCard.jsx  InvoiceRow.jsx ...
  hooks/        useUser.js  useInvoices.js ...
  utils/        format.js   api.js ...
  pages/        Dashboard.jsx  Invoices.jsx ...
```

This is fine at ~10 files. By 100 files, a single feature (say "invoices") is smeared across four directories, and `components/` becomes a junk drawer. Refactor toward **feature-based** structure, where each feature owns its slice:

```
src/
  app/                      # app shell, routing, providers
    router.jsx
    providers.jsx
  features/
    invoices/
      components/           # invoice-only components
        InvoiceRow.jsx
        InvoiceTable.jsx
      hooks/
        useInvoices.js
      api/
        invoicesApi.js
      InvoicesPage.jsx
      index.js              # public surface of the feature
    auth/
      components/
      hooks/
      authApi.js
      AuthProvider.jsx
      index.js
  shared/                   # genuinely cross-feature primitives
    ui/                     # Button, Modal, Input (design system)
    hooks/                  # useDebounce, useMediaQuery
    lib/                    # api client, formatting, constants
  main.jsx
```

**The rule of thumb:** start type-based for tiny apps; flip to feature-based the moment a second team member or a third feature appears. The win is that a feature becomes a *unit you can reason about, move, or delete*. The `shared/` folder is the pressure valve — but be strict: something belongs in `shared/` only when **two or more features actually use it**. Promoting too eagerly recreates the junk drawer.

### Example 2 — Splitting a component by responsibility (and the modern take on container/presentational)

A bloated "smart" component mixing fetching, formatting, and markup:

```jsx
// ❌ Does too much: data, derived state, and presentation entangled.
function UserDashboard({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    let active = true;
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((u) => active && setUser(u))
      .finally(() => active && setLoading(false));
    return () => { active = false; };
  }, [userId]);

  if (loading) return <p>Loading…</p>;
  const fullName = `${user.first} ${user.last}`;
  const joined = new Date(user.createdAt).toLocaleDateString();
  return (
    <div>
      <h1>{fullName}</h1>
      <p>Joined {joined}</p>
    </div>
  );
}
```

The classic fix was "container vs presentational" components. The **modern** version of that split uses a **custom hook** for the data/logic and a pure component for the view:

```jsx
// hooks/useUser.js — logic lives in a hook (reusable, testable in isolation).
export function useUser(userId) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    let active = true;
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((u) => active && setUser(u))
      .finally(() => active && setLoading(false));
    return () => { active = false; };
  }, [userId]);
  return { user, loading };
}
```

```jsx
// UserProfile.jsx — presentational: props in, markup out. Trivial to test/storybook.
function UserProfile({ name, joinedLabel }) {
  return (
    <div>
      <h1>{name}</h1>
      <p>Joined {joinedLabel}</p>
    </div>
  );
}

// UserDashboard.jsx — thin orchestrator that wires the hook to the view.
function UserDashboard({ userId }) {
  const { user, loading } = useUser(userId);
  if (loading) return <p>Loading…</p>;
  return (
    <UserProfile
      name={`${user.first} ${user.last}`}
      joinedLabel={new Date(user.createdAt).toLocaleDateString()}
    />
  );
}
```

Now `UserProfile` is a pure function of its props, `useUser` is unit-testable without rendering, and `UserDashboard` reads top-to-bottom. (In production you'd usually fetch via React Query / `use` + Suspense — see `15-data-fetching.md` — but the *separation* principle is identical.)

### Example 3 — Composition over configuration

When a component sprouts a flag for every variation, you get **prop explosion**:

```jsx
// ❌ Configuration soup. Every new need adds another boolean.
<Modal
  title="Delete account"
  showHeader
  showFooter
  showCloseButton
  primaryLabel="Delete"
  primaryVariant="danger"
  secondaryLabel="Cancel"
  bodyText="This cannot be undone."
  bodyAlign="center"
/>
```

Compose with `children` and slots instead — the caller assembles exactly what it needs:

```jsx
// ✅ Composition. Modal owns behavior (focus trap, escape, overlay); caller owns content.
function Modal({ children, onClose, labelledBy }) {
  // ...focus trap + Escape handling lives here (see Example 4)...
  return (
    <div role="presentation" className="overlay" onClick={onClose}>
      <div role="dialog" aria-modal="true" aria-labelledby={labelledBy}
           onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}
Modal.Header = ({ children }) => <header className="modal__header">{children}</header>;
Modal.Body   = ({ children }) => <div className="modal__body">{children}</div>;
Modal.Footer = ({ children }) => <footer className="modal__footer">{children}</footer>;
```

```jsx
<Modal onClose={close} labelledBy="del-title">
  <Modal.Header><h2 id="del-title">Delete account</h2></Modal.Header>
  <Modal.Body><p>This cannot be undone.</p></Modal.Body>
  <Modal.Footer>
    <Button onClick={close}>Cancel</Button>
    <Button variant="danger" onClick={confirm}>Delete</Button>
  </Modal.Footer>
</Modal>
```

The component now has a *closed* set of behavioral responsibilities and an *open* set of content possibilities. New layouts require zero changes to `Modal`.

### Example 4 — An accessible, focus-managed modal

This combines several a11y practices: dialog role, accessible name, **focus moves into the dialog on open and returns on close**, Escape closes, and focus is trapped.

```jsx
import { useEffect, useRef } from "react";

function AccessibleModal({ onClose, labelledBy, children }) {
  const dialogRef = useRef(null);
  const previouslyFocused = useRef(null);

  useEffect(() => {
    previouslyFocused.current = document.activeElement;          // remember opener
    const node = dialogRef.current;
    const focusables = node.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    (focusables[0] ?? node).focus();                              // move focus in

    function onKeyDown(e) {
      if (e.key === "Escape") onClose();
      if (e.key === "Tab" && focusables.length) {                // trap Tab
        const first = focusables[0];
        const last = focusables[focusables.length - 1];
        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault(); last.focus();
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault(); first.focus();
        }
      }
    }
    node.addEventListener("keydown", onKeyDown);
    return () => {
      node.removeEventListener("keydown", onKeyDown);
      previouslyFocused.current?.focus();                        // restore focus on close
    };
  }, [onClose]);

  return (
    <div className="overlay" role="presentation" onClick={onClose}>
      <div ref={dialogRef} role="dialog" aria-modal="true"
           aria-labelledby={labelledBy} tabIndex={-1}
           onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}
```

A sighted mouse user barely notices any of this. A keyboard or screen-reader user *cannot use the modal at all* without it. That asymmetry is exactly why a11y is correctness, not polish.

### Example 5 — Error boundary + async error handling together

Error boundaries catch errors thrown **during render, in lifecycle, and in constructors of the components below them**. They do **not** catch errors in event handlers, `setTimeout`, or promise rejections. You need both mechanisms.

```jsx
function App() {
  return (
    <ErrorBoundary fallback={<p role="alert">Something went wrong. Please refresh.</p>}>
      <Dashboard />
    </ErrorBoundary>
  );
}
```

```jsx
// Async errors: handle in the handler, and surface to the UI as state.
function SaveButton({ id }) {
  const [error, setError] = useState(null);
  async function handleSave() {
    try {
      const res = await fetch(`/api/items/${id}`, { method: "PUT" });
      if (!res.ok) throw new Error(`Save failed (${res.status})`);
    } catch (e) {
      setError(e.message);            // event-handler errors are NOT caught by boundaries
    }
  }
  return (
    <>
      <button onClick={handleSave}>Save</button>
      {error && <p role="alert">{error}</p>}
    </>
  );
}
```

Strategy: put a **top-level boundary** to prevent a white screen, **section-level boundaries** around independent regions (a failing sidebar shouldn't kill the main content), and handle **expected** async failures (network, validation) as local state with user-facing messaging. Reserve boundaries for the *unexpected*.

---

## Under the hood

**Why colocation reduces bugs, mechanically.** When a component, its styles, its tests, and its hook live in one folder, deleting the feature is a single `rm -rf` and your bundler tree-shakes nothing dangling. When they're scattered, dead code accumulates because no one is confident it's safe to remove. Architecture isn't aesthetic — it changes the *probability* that the next change is correct.

**Barrel files (`index.js` re-exports) and what they actually cost.** A barrel like:

```js
// shared/ui/index.js
export { Button } from "./Button.jsx";
export { Modal } from "./Modal.jsx";
export { Tooltip } from "./Tooltip.jsx";
```

…lets callers write `import { Button } from "@/shared/ui"`. The cost: importing *anything* from the barrel makes the module graph reference *everything* the barrel touches. Modern bundlers (Vite/Rollup, esbuild) tree-shake well for pure ES modules, but barrels routinely defeat it because (a) a re-exported module may have side effects, or (b) the analysis gives up under deep barrel chains. The result is **slower dev cold-starts** and occasionally **fatter production bundles**. Worse, barrels are the #1 cause of **circular imports**: `A → index → B → index → A`. The pragmatic stance: a thin barrel at a **feature's public boundary** is good (it documents the API); a barrel that re-exports your *entire* `shared/ui` is a liability — prefer deep imports (`@/shared/ui/Button`) for leaf components.

**How error boundaries work in the runtime.** React renders a component tree. If a render throws, React unwinds to the nearest ancestor that implements `getDerivedStateFromError`/`componentDidCatch`, discards the subtree, and re-renders that boundary's fallback. There is *no hook* form because the catching has to happen at the class instance level during React's commit phase — `try/catch` around your own code can't see React's internal render of children. This is why every robust app keeps at least one class component, even in an otherwise hooks-only codebase. In dev, React also surfaces caught errors in the console and (with Strict Mode) double-invokes renders to flush out impurities.

**The accessibility tree.** The browser builds a parallel **accessibility tree** from your DOM, computing a *role*, *name*, *state*, and *value* for each node. `<button>Save</button>` automatically has role `button` and accessible name "Save". `<div onClick>` has role `generic` and no name — invisible to assistive tech. ARIA attributes patch this tree (`role`, `aria-label`, `aria-expanded`), but **native semantics are always cheaper and more correct than ARIA** — hence the first rule of ARIA: *don't use ARIA if a native element does the job.* When you render an icon-only button, the visible text is empty, so the accessible name must come from `aria-label`.

**XSS and why React is mostly-but-not-always safe.** React escapes all string children by default: `<div>{userInput}</div>` renders `&lt;script&gt;` text, not a tag. The dangerous surfaces are exactly the ones that bypass this: `dangerouslySetInnerHTML`, injecting URLs into `href`/`src` (`javascript:` schemes), and spreading unsanitized props onto DOM nodes. The runtime can't know whether a string is safe HTML — that's a domain decision — so it hands you the loaded gun with a scary name and trusts you to sanitize first.

---

## Common mistakes & pitfalls

**1. Trusting the client / putting secrets in the bundle.**

```jsx
// ❌ Anything in client code ships to the browser. This key is public.
const stripe = new Stripe("sk_live_51H...");      // secret key leaked in JS bundle
if (user.role === "admin") renderDeleteButton();   // "auth" the server never checks
```

```jsx
// ✅ Secrets live server-side; the client only holds PUBLISHABLE values.
// .env exposes only VITE_-prefixed vars to the client — and even those are PUBLIC.
const stripe = new Stripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY);
// The server independently re-checks authorization on every mutating request.
```

The client UI is a *convenience*, never a *security boundary*. Hiding a button does not protect the endpoint behind it. Every privileged action must be authorized again on the server.

**2. Unsanitized `dangerouslySetInnerHTML`.**

```jsx
// ❌ Stored XSS: a malicious comment runs script in every viewer's session.
<div dangerouslySetInnerHTML={{ __html: comment.body }} />
```

```jsx
// ✅ Sanitize first (e.g. DOMPurify), and prefer rendering as text when you can.
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(comment.body) }} />
// Better still, if you don't need HTML: <div>{comment.body}</div>  // auto-escaped
```

**3. Div-as-button (kills keyboard + screen-reader users).**

```jsx
// ❌ Not focusable, no Enter/Space activation, no role, no name.
<div className="btn" onClick={submit}>Submit</div>
```

```jsx
// ✅ Use the element the platform already made accessible.
<button type="button" onClick={submit}>Submit</button>
```

**4. Mutating state directly.**

```jsx
// ❌ Mutation: React may not re-render; bugs that "only happen sometimes".
function addTodo(text) {
  todos.push({ text });   // mutates existing array
  setTodos(todos);        // same reference → React can bail out of re-render
}
```

```jsx
// ✅ Produce a new reference immutably.
function addTodo(text) {
  setTodos((prev) => [...prev, { text }]);
}
```

Immutability is what lets React (and memoization via `React.memo`/`useMemo`) cheaply decide "did this change?" by reference equality. Break it and you break change detection. See `13-performance.md`.

**5. Premature abstraction (DRY worship).**

```jsx
// ❌ A "flexible" mega-component invented after seeing TWO similar screens.
<DataView
  source={endpoint} columns={cfg} renderRow={fn} emptyState={node}
  pagination variant="compact" onSelect={...} selectionMode="multi" /* …14 more props */
/>
```

When two pieces of code look similar but change for *different reasons*, merging them couples unrelated futures. The cure is the **rule of three**: duplicate freely until the third occurrence reveals the *real* shared shape, then extract. Wrong abstractions are more expensive than duplication because every consumer must bend around them.

**6. `useEffect` for things that aren't effects.**

```jsx
// ❌ Deriving state in an effect — extra render, stale-risk.
useEffect(() => { setFullName(`${first} ${last}`); }, [first, last]);
```

```jsx
// ✅ Derive during render. No effect, no extra state.
const fullName = `${first} ${last}`;
```

Effects are for *synchronizing with external systems* (see `08-useeffect-side-effects.md`), not for computing values you already have.

---

## Best practices

- **Do** organize by feature once you pass a handful of files; keep `shared/` strict (promote on second real use, not on a hunch).
- **Do** keep components small and single-purpose; if a file does fetching *and* layout *and* formatting, split it.
- **Do** push reusable logic into custom hooks (`10-custom-hooks.md`) and pure I/O into `services/`-style modules; keep components declarative.
- **Do** prefer composition (`children`, slots, compound components) over a growing pile of boolean/config props.
- **Do** name predictably: `PascalCase` components, `useX` hooks, `handleX` event handlers, `isX/hasX` booleans, `onX` callback props. File name matches the default export.
- **Do** treat state as immutable; always return new references from updaters.
- **Don't** abstract before the third occurrence; duplication is cheaper than the wrong abstraction.
- **Don't** use `<div onClick>` where a `<button>`/`<a>` belongs; reach for ARIA only after native semantics fall short.
- **Don't** put secrets, business authorization, or trust in client code — re-validate everything server-side.
- **Don't** render untrusted HTML without sanitizing; avoid `dangerouslySetInnerHTML` unless you must.
- **Do** wrap the app and independent regions in error boundaries; handle expected async failures as local UI state.
- **Do** run automated a11y checks in CI (`jest-axe`/`vitest-axe`) and lint with `eslint-plugin-jsx-a11y`.

---

## Exercises

**Exercise 1 — Restructure.** Take a type-based `src/` (a `components/`, `hooks/`, `utils/` layout) with two features ("cart" and "catalog") and reorganize it into feature folders with a strict `shared/`. Write a one-paragraph rationale for each file you place in `shared/`.

**Exercise 2 — Decompose.** Given a 150-line `OrdersPage` that fetches orders, filters them, formats currency, and renders a table, split it into: a `useOrders` hook, a pure `OrdersTable` presentational component, a `formatCurrency` util, and a thin `OrdersPage` orchestrator.

**Exercise 3 — Accessible menu.** Build a dropdown menu button (`aria-haspopup`, `aria-expanded`) that opens a list, supports Arrow keys to move between items, Escape to close, and returns focus to the trigger on close.

**Exercise 4 — Harden.** Take a comment list that renders `dangerouslySetInnerHTML={{ __html: c.body }}` and a "delete" button shown only to admins. Make it safe: sanitize the HTML and write a sentence describing what the *server* must additionally enforce.

**Exercise 5 — Test a11y.** Add a `vitest`/`jest` test using `axe` that asserts your component from Exercise 3 has no detectable accessibility violations.

### Solutions

**Solution 2 (decompose):**

```jsx
// lib/formatCurrency.js
export const formatCurrency = (cents, currency = "USD") =>
  new Intl.NumberFormat(undefined, { style: "currency", currency }).format(cents / 100);

// features/orders/hooks/useOrders.js
import { useEffect, useState } from "react";
export function useOrders(filter) {
  const [orders, setOrders] = useState([]);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    let active = true;
    setLoading(true);
    fetch(`/api/orders?status=${encodeURIComponent(filter)}`)
      .then((r) => r.json())
      .then((d) => active && setOrders(d))
      .finally(() => active && setLoading(false));
    return () => { active = false; };
  }, [filter]);
  return { orders, loading };
}

// features/orders/components/OrdersTable.jsx — pure presentational
import { formatCurrency } from "@/lib/formatCurrency";
export function OrdersTable({ orders }) {
  return (
    <table>
      <caption>Orders</caption>
      <thead><tr><th scope="col">ID</th><th scope="col">Total</th></tr></thead>
      <tbody>
        {orders.map((o) => (
          <tr key={o.id}><td>{o.id}</td><td>{formatCurrency(o.totalCents)}</td></tr>
        ))}
      </tbody>
    </table>
  );
}

// features/orders/OrdersPage.jsx — thin orchestrator
import { useState } from "react";
import { useOrders } from "./hooks/useOrders";
import { OrdersTable } from "./components/OrdersTable";
export function OrdersPage() {
  const [filter, setFilter] = useState("open");
  const { orders, loading } = useOrders(filter);
  return (
    <section>
      <label htmlFor="status">Status</label>
      <select id="status" value={filter} onChange={(e) => setFilter(e.target.value)}>
        <option value="open">Open</option>
        <option value="closed">Closed</option>
      </select>
      {loading ? <p>Loading…</p> : <OrdersTable orders={orders} />}
    </section>
  );
}
```

**Solution 3 (accessible menu, abbreviated core):**

```jsx
import { useRef, useState, useEffect } from "react";

function Menu({ label, items }) {
  const [open, setOpen] = useState(false);
  const [active, setActive] = useState(0);
  const triggerRef = useRef(null);
  const listRef = useRef(null);

  useEffect(() => {
    if (open) listRef.current?.querySelectorAll('[role="menuitem"]')[active]?.focus();
  }, [open, active]);

  function onKeyDown(e) {
    if (e.key === "Escape") { setOpen(false); triggerRef.current?.focus(); }
    if (e.key === "ArrowDown") { e.preventDefault(); setActive((i) => (i + 1) % items.length); }
    if (e.key === "ArrowUp")   { e.preventDefault(); setActive((i) => (i - 1 + items.length) % items.length); }
  }

  return (
    <div onKeyDown={onKeyDown}>
      <button ref={triggerRef} aria-haspopup="menu" aria-expanded={open}
              onClick={() => setOpen((o) => !o)}>
        {label}
      </button>
      {open && (
        <ul role="menu" ref={listRef}>
          {items.map((it, i) => (
            <li key={it.id} role="menuitem" tabIndex={-1}
                onClick={() => { it.onSelect(); setOpen(false); triggerRef.current?.focus(); }}>
              {it.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Solution 5 (a11y test):**

```jsx
import { render } from "@testing-library/react";
import { axe } from "vitest-axe";          // or "jest-axe"
import * as matchers from "vitest-axe/matchers";
import { expect, test } from "vitest";
import Menu from "./Menu.jsx";

// Register the matcher (or do this once in a setup file:
// `import "vitest-axe/extend-expect"`). With jest-axe:
// `import { toHaveNoViolations } from "jest-axe"; expect.extend({ toHaveNoViolations });`
expect.extend(matchers);

test("Menu has no a11y violations", async () => {
  const { container } = render(
    <Menu label="Actions" items={[{ id: "1", label: "Edit", onSelect() {} }]} />
  );
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

---

## Key takeaways

- **Architecture optimizes for the cost of change**: colocation, single responsibility, and least abstraction make "where does this go?" and "what touches this?" cheap to answer.
- **Go feature-based as you grow**; keep `shared/` strict and use barrels only at feature boundaries — never as a giant re-export hub, to avoid bundle bloat and circular imports.
- **Logic belongs in custom hooks and service modules**, not stuffed into components; keep presentational components pure and testable.
- **Prefer composition over configuration** to avoid prop explosion; let callers assemble content while the component owns behavior.
- **Resist premature abstraction** — follow the rule of three; the wrong abstraction costs more than duplication.
- **Combine error boundaries (render-time) with try/catch + state (async)** — boundaries don't catch event-handler or promise errors.
- **Accessibility is correctness**: semantic HTML first, ARIA only when needed, manage focus, support the keyboard, meet contrast, and test with `axe`.
- **Never trust the client**: secrets and authorization live on the server; sanitize any HTML you inject; keep dependencies patched.

---

## Further reading

- React docs — Thinking in React: <https://react.dev/learn/thinking-in-react>
- React docs — Reusing logic with custom hooks: <https://react.dev/learn/reusing-logic-with-custom-hooks>
- React docs — `Component` (error boundaries / `componentDidCatch`): <https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary>
- React docs — `dangerouslySetInnerHTML` and DOM components: <https://react.dev/reference/react-dom/components/common#dangerously-setting-the-inner-html>
- MDN — ARIA Authoring Practices (patterns for menus, dialogs, etc.): <https://www.w3.org/WAI/ARIA/apg/patterns/>
- WCAG 2.2 Quick Reference (contrast, focus, keyboard): <https://www.w3.org/WAI/WCAG22/quickref/>
- `eslint-plugin-jsx-a11y`: <https://github.com/jsx-eslint/eslint-plugin-jsx-a11y>
- `jest-axe`: <https://github.com/nickcolley/jest-axe> · `vitest-axe`: <https://github.com/chaance/vitest-axe>
- OWASP XSS Prevention Cheat Sheet: <https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html>
- DOMPurify (HTML sanitization): <https://github.com/cure53/DOMPurify>
- Vite — env variables and modes (`import.meta.env`, `VITE_` prefix): <https://vite.dev/guide/env-and-mode>
