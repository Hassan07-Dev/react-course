# 12 — Context API

> **What you'll learn:** how to share data across a deeply nested component tree without threading props through every layer, using React's Context API the modern (React 19) way — and how to do it without accidentally re-rendering your whole app.

By the end of this lesson you will be able to:

- Recognize the **prop-drilling** problem and explain why it hurts maintainability.
- Create a context with `createContext`, supply a value with a Provider, and read it with `useContext`.
- Use the **React 19** shorthand of rendering `<MyContext>` directly as the provider.
- Put **mutable state** inside a provider component so any consumer can both read and update shared data (auth user, theme, locale, cart).
- Compose **multiple contexts** cleanly without a "provider pyramid" mess.
- Understand and mitigate the **re-render caveat** with value memoization and the split state/dispatch pattern.
- Build a small global store with the **`useReducer` + Context** pattern.
- Decide when Context is the right tool and when to reach for a dedicated state library.

---

## Prerequisites

You should be comfortable with the material from earlier lessons:

- **04 — Components and props** (`04-components-and-props.md`): what props are and how data flows down.
- **05 — Rendering lists & conditionals** (`05-rendering-lists-conditionals.md`): conditional rendering.
- **06 — Handling events** (`06-handling-events.md`): event handlers and callbacks.
- **07 — State with `useState`** (`07-state-usestate.md`): local state and the rules of hooks.
- **08 — `useEffect` & side effects** (`08-useeffect-side-effects.md`): effects, dependency arrays, and persistence to `localStorage`.

You'll also see `useReducer` and `useMemo` used here. We touch on them enough to be self-contained, but they get fuller treatment in later lessons (`useReducer` in particular pairs with this one).

---

## Concept

### The problem: prop drilling

React's data flow is **one-directional and top-down**: a parent passes data to its children through props. That's a feature — it makes data flow easy to trace. But it falls apart when a value lives high in the tree and is needed by a component far below it.

Imagine an app where the logged-in user is known at the top (`<App>`), but the component that needs it — say an `<Avatar>` — is buried six layers deep:

```text
App (has `user`)
└── Layout
    └── Header
        └── Toolbar
            └── UserMenu
                └── Avatar  ← needs `user`
```

To get `user` to `<Avatar>`, you'd pass it as a prop through `Layout`, `Header`, `Toolbar`, and `UserMenu` — none of which actually *use* it. They're just couriers. This is **prop drilling**, and it has real costs:

- Every intermediate component's signature is polluted with props it doesn't care about.
- Renaming or adding a field means editing five files.
- The intermediate components become harder to reuse, because they now "know about" `user`.

### The solution: Context as a "broadcast channel"

**Context** lets a component **broadcast** a value to everything in its subtree, and lets any descendant **tune in** to that value directly — skipping all the couriers in between.

A useful analogy: props are like handing a note person-to-person down a line. Context is like a **radio station**. A **Provider** is the transmitter; it broadcasts a value on a frequency. Any component inside that Provider's subtree can put on a **receiver** (`useContext`) tuned to that frequency and hear the value instantly, no matter how deep it is. Components that don't tune in are unaffected.

Three pieces make this work:

1. **`createContext`** — defines the "frequency" (the context object) and a default value.
2. **The Provider** — wraps part of your tree and sets the value being broadcast.
3. **`useContext`** — the receiver a descendant uses to read the current value.

Context is **not** a state management library on its own. It's a *transport mechanism*. The state itself still lives in some component (usually via `useState` or `useReducer`); Context just makes that state reachable without drilling. Keep that distinction in your head — it explains most of Context's behavior and its limits.

---

## Syntax & API

### `createContext`

```js
import { createContext } from 'react';

const ThemeContext = createContext(defaultValue);
```

- **`defaultValue`** is used *only* when a consuming component has **no matching Provider above it** in the tree. It is **not** the value used when you render a Provider without a `value` prop, and it's not a "fallback for `undefined`." A common pattern is to pass `null` and assert in a custom hook (shown later).
- The returned object (`ThemeContext`) is what you import everywhere you need to provide or consume. Convention: capitalize it and name it `...Context`.
- Calling `createContext` at module scope (not inside a component) is the norm.

### The Provider

**Classic form (works in all versions):**

```jsx
<ThemeContext.Provider value={someValue}>
  {children}
</ThemeContext.Provider>
```

**React 19 shorthand — render the context object directly:**

```jsx
<ThemeContext value={someValue}>
  {children}
</ThemeContext>
```

In React 19 the context object itself is a valid provider component, so you can drop `.Provider`. Both forms are identical at runtime. `.Provider` is now the legacy form — it still works in React 19 and is not yet deprecated, but the React team has stated it will be deprecated in a future version (and ships a codemod to migrate). This lesson uses the React 19 form by default and notes the classic form where useful.

- **`value`** is the data broadcast to the subtree. It can be anything: a string, number, object, array, or a tuple of `[state, setState]`.
- A subtree can be wrapped by the same context's provider more than once; the **nearest** provider wins for any given consumer.

### `useContext`

```js
import { useContext } from 'react';

const value = useContext(ThemeContext);
```

- Returns the `value` from the **nearest** matching provider above the calling component. If there is none, returns the `defaultValue` from `createContext`.
- It's a hook, so it obeys the rules of hooks: call it at the top level of a component or custom hook, never conditionally (see `09-hooks-deep-dive.md`).
- A component that calls `useContext` will **re-render whenever the provider's `value` changes** (compared by `Object.is`). This is the central performance fact to internalize.

### `use` (React 19, brief note)

React 19 also adds a `use` API that can read a context: `const value = use(ThemeContext);`. Unlike `useContext`, `use` may be called conditionally (e.g. after an early `return`). For ordinary context reads, `useContext` is still the clear, idiomatic choice; reach for `use` only when you specifically need conditional reads.

---

## Examples

### Example 1 — The smallest possible context

A theme string broadcast from the top and read deep down.

```jsx
// theme-context.js
import { createContext } from 'react';

export const ThemeContext = createContext('light'); // default if no provider
```

```jsx
// App.jsx
import { useContext } from 'react';
import { ThemeContext } from './theme-context';

function Avatar() {
  // No props threaded in — we read directly from context.
  const theme = useContext(ThemeContext);
  return <div className={`avatar avatar--${theme}`}>🙂</div>;
}

function Toolbar() {
  // Toolbar doesn't know or care about theme — no prop drilling.
  return <Avatar />;
}

export default function App() {
  return (
    <ThemeContext value="dark">
      <Toolbar />
    </ThemeContext>
  );
}
```

`Avatar` receives `"dark"` even though `Toolbar` never passed it down. Remove the provider and `Avatar` falls back to `"light"`, the default. This is read-only; next we make it changeable.

### Example 2 — Putting STATE in the provider (a theme switcher)

The real power: store `useState` *inside* a provider component and broadcast both the value **and** the updater. Now any consumer can read **and** change shared state.

```jsx
// ThemeProvider.jsx
import { createContext, useContext, useState, useMemo } from 'react';

const ThemeContext = createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  // Memoize so the value object is stable across renders where theme didn't change.
  const value = useMemo(
    () => ({
      theme,
      toggleTheme: () => setTheme((t) => (t === 'light' ? 'dark' : 'light')),
    }),
    [theme],
  );

  return <ThemeContext value={value}>{children}</ThemeContext>;
}

// Custom hook: the recommended way to consume.
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (ctx === null) {
    throw new Error('useTheme must be used inside a <ThemeProvider>');
  }
  return ctx;
}
```

```jsx
// App.jsx
import { ThemeProvider, useTheme } from './ThemeProvider';

function ThemeToggleButton() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button onClick={toggleTheme}>
      Switch to {theme === 'light' ? 'dark' : 'light'} mode
    </button>
  );
}

function Page() {
  const { theme } = useTheme();
  return (
    <main className={theme} style={{ padding: 24 }}>
      <h1>Current theme: {theme}</h1>
      <ThemeToggleButton />
    </main>
  );
}

export default function App() {
  return (
    <ThemeProvider>
      <Page />
    </ThemeProvider>
  );
}
```

Two important moves here:

1. **The custom `useTheme` hook** hides `useContext(ThemeContext)` and throws a helpful error if used outside the provider. Consumers import one tidy hook; they never touch the raw context object. This is the single most valuable convention in this lesson.
2. **`useMemo` on the `value`** keeps the broadcast object referentially stable when `theme` hasn't changed, which matters for performance (explained in *Under the hood*).

### Example 3 — Persisting and an auth context

A realistic **auth context** that exposes `user`, `login`, and `logout`, and persists the session to `localStorage` (see `08-useeffect-side-effects.md` for the effect pattern).

```jsx
// auth-context.jsx
import { createContext, useContext, useState, useEffect, useMemo, useCallback } from 'react';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(() => {
    // Lazy initializer — runs once, reads any persisted session.
    const raw = localStorage.getItem('auth.user');
    return raw ? JSON.parse(raw) : null;
  });

  useEffect(() => {
    if (user) localStorage.setItem('auth.user', JSON.stringify(user));
    else localStorage.removeItem('auth.user');
  }, [user]);

  const login = useCallback(async (email, password) => {
    // Pretend network call.
    const fakeUser = { id: 1, email, name: email.split('@')[0] };
    setUser(fakeUser);
    return fakeUser;
  }, []);

  const logout = useCallback(() => setUser(null), []);

  const value = useMemo(
    () => ({ user, isAuthenticated: user !== null, login, logout }),
    [user, login, logout],
  );

  return <AuthContext value={value}>{children}</AuthContext>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (ctx === null) throw new Error('useAuth must be used inside <AuthProvider>');
  return ctx;
}
```

```jsx
// LoginGate.jsx
import { useState } from 'react';
import { useAuth } from './auth-context';

export default function LoginGate({ children }) {
  const { isAuthenticated, user, login, logout } = useAuth();
  const [email, setEmail] = useState('');

  if (!isAuthenticated) {
    return (
      <form
        onSubmit={(e) => {
          e.preventDefault();
          login(email, 'secret');
        }}
      >
        <input
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          placeholder="you@example.com"
        />
        <button type="submit">Log in</button>
      </form>
    );
  }

  return (
    <div>
      <p>Welcome, {user.name}</p>
      <button onClick={logout}>Log out</button>
      {children}
    </div>
  );
}
```

`login`, `logout`, and `isAuthenticated` are all available to *any* descendant via `useAuth()`. Wrapping `login`/`logout` in `useCallback` keeps them stable so the memoized `value` only changes when `user` actually changes.

### Example 4 — Composing multiple contexts

Real apps have several independent contexts (theme, auth, locale, cart). They don't need to know about each other; you nest their providers. To avoid a deeply indented "pyramid of doom," compose them into one component.

```jsx
// AppProviders.jsx
import { ThemeProvider } from './ThemeProvider';
import { AuthProvider } from './auth-context';
import { LocaleProvider } from './locale-context';

export function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider>{children}</LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}
```

```jsx
// main.jsx
import { createRoot } from 'react-dom/client';
import { AppProviders } from './AppProviders';
import App from './App';

createRoot(document.getElementById('root')).render(
  <AppProviders>
    <App />
  </AppProviders>,
);
```

A component can consume any combination it needs:

```jsx
function Header() {
  const { user } = useAuth();
  const { theme } = useTheme();
  const { t } = useLocale();
  return <header className={theme}>{user ? t('greeting', user.name) : t('signedOut')}</header>;
}
```

Order of nesting matters only if one provider depends on another (e.g. a provider that reads `useAuth` must sit *inside* `AuthProvider`).

### Example 5 — `useReducer` + Context = a mini global store

For state with several actions (a shopping cart), `useReducer` keeps update logic in one place, and Context distributes it. We **split into two contexts**: one for the data, one for the dispatch function. (Why we split is the punchline of *Under the hood*.)

```jsx
// cart-store.jsx
import { createContext, useContext, useReducer } from 'react';

const CartStateContext = createContext(null);
const CartDispatchContext = createContext(null);

function cartReducer(state, action) {
  switch (action.type) {
    case 'add': {
      const existing = state.items.find((i) => i.id === action.item.id);
      const items = existing
        ? state.items.map((i) =>
            i.id === action.item.id ? { ...i, qty: i.qty + 1 } : i,
          )
        : [...state.items, { ...action.item, qty: 1 }];
      return { ...state, items };
    }
    case 'remove':
      return { ...state, items: state.items.filter((i) => i.id !== action.id) };
    case 'clear':
      return { ...state, items: [] };
    default:
      throw new Error(`Unknown cart action: ${action.type}`);
  }
}

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });
  return (
    <CartStateContext value={state}>
      <CartDispatchContext value={dispatch}>{children}</CartDispatchContext>
    </CartStateContext>
  );
}

export function useCartState() {
  const ctx = useContext(CartStateContext);
  if (ctx === null) throw new Error('useCartState must be used inside <CartProvider>');
  return ctx;
}

export function useCartDispatch() {
  const ctx = useContext(CartDispatchContext);
  if (ctx === null) throw new Error('useCartDispatch must be used inside <CartProvider>');
  return ctx;
}
```

```jsx
// usage
function AddButton({ product }) {
  const dispatch = useCartDispatch(); // only subscribes to dispatch (stable!)
  return <button onClick={() => dispatch({ type: 'add', item: product })}>Add</button>;
}

function CartCount() {
  const { items } = useCartState(); // re-renders only when items change
  const count = items.reduce((n, i) => n + i.qty, 0);
  return <span>🛒 {count}</span>;
}
```

`AddButton` reads **only** dispatch. Because React guarantees `dispatch` from `useReducer` is referentially stable for the component's lifetime, `AddButton` **never re-renders** when the cart changes. Only components reading `CartStateContext` (like `CartCount`) re-render. That's the whole point of splitting.

---

## Under the hood

### How a context read finds its value

When a component calls `useContext(SomeContext)`, React walks **up the rendered tree** (the fiber tree) from that component looking for the nearest provider of `SomeContext`, and returns that provider's current `value`. If it finds none, it returns the `defaultValue` from `createContext`. This lookup is by **context object identity**, which is why you import and reuse the *same* `SomeContext` object everywhere.

### The re-render caveat (the thing experts watch)

When a provider's `value` changes, **every component that reads that context re-renders** — and it does so *unconditionally*, bypassing the normal `React.memo` shortcut. `React.memo` only short-circuits re-renders caused by *prop* changes; a context change is a separate subscription, so a `memo`'d component still re-renders if it (or something it renders) consumes the changed context.

How does React decide a value "changed"? It compares the new `value` to the previous one with **`Object.is`** (reference equality for objects). This produces a classic trap:

```jsx
// ⚠️ A brand-new object every render → Object.is is always false → all consumers re-render every time.
<AuthContext value={{ user, login, logout }}>
```

Even if `user`, `login`, and `logout` are unchanged, the object literal `{ ... }` is a *new reference* on every render of the provider component. Consumers re-render needlessly. The fix is `useMemo` (and `useCallback` for the functions inside it), as in Examples 2 and 3:

```jsx
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
```

Now the object reference is reused across renders unless a dependency actually changes.

### Why split state and dispatch

Even a perfectly memoized single value re-renders *all* consumers whenever *any* part of it changes. If a component only needs the `dispatch` function but reads the combined `{ state, dispatch }` value, it re-renders every time `state` changes — wasted work. Splitting into two contexts means a component subscribes only to the slice it needs. Dispatch consumers stay quiet because `dispatch` is stable; state consumers re-render only on state changes. This is the cheapest, most reliable optimization for Context-based stores and needs no third-party library.

A related, finer-grained technique is **context selectors** (subscribing to a *computed slice* of the value so you only re-render when that slice changes). React doesn't ship a built-in selector for context; libraries like `use-context-selector` provide one. If you find yourself wanting selectors, that's often the signal to consider a state library (next section, and `16-state-management.md`).

### Context is not "global state" — it's tree-scoped

The value a consumer sees depends on **where it sits in the tree**, not on a global singleton. Two subtrees can have two providers of the same context with different values, and each subtree sees its own. This makes contexts composable and testable: in tests you wrap the component under test in a provider with whatever value you want. It also means a context value is *only* available below its provider — render order and placement matter.

### Cost model

`useContext` itself is cheap — it's a subscription, not a render of the provider. The cost is the **downstream re-renders** of consumers. So the optimization story is always about (a) keeping `value` referentially stable and (b) narrowing what each component subscribes to. Context is excellent for **low-frequency, broadly-read** data (theme, auth, locale). It's a poor fit for **high-frequency** updates read by many components (e.g. mouse position, an animation clock, a fast-typing input mirrored everywhere) — those will thrash. Reach for local state, a ref, or a library instead.

### Context vs. a state library — when to switch (see `16-state-management.md`)

Context plus `useReducer` is a genuine, dependency-free global store and is enough for a great many apps. Consider a dedicated library (Zustand, Redux Toolkit, Jotai, etc.) when you hit one or more of:

- **Frequent updates with many subscribers** and you need fine-grained selectors to avoid re-renders.
- **Complex async / server-cache** needs — for *server* data specifically, a data-fetching library (TanStack Query, RTK Query) is usually the better answer than either Context or a client store.
- **Cross-cutting middleware/devtools** (time-travel, action logging) you'd otherwise hand-roll.

If your shared data is read widely but changes rarely, **Context is the right tool** — don't add a library just because. The full comparison is covered in `16` (state management libraries).

---

## Common mistakes & pitfalls

**1. New object literal as `value` every render**

```jsx
// ❌ WRONG — new reference each render → every consumer re-renders constantly.
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext value={{ theme, setTheme }}>{children}</ThemeContext>
  );
}
```

```jsx
// ✅ RIGHT — memoize the value.
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  return <ThemeContext value={value}>{children}</ThemeContext>;
}
```

`setTheme` from `useState` is already stable, so `[theme]` is the only real dependency. Why it matters: without `useMemo`, `Object.is(prev, next)` is always `false`, forcing all consumers to re-render even when nothing they care about changed.

**2. Consuming a context outside its provider**

```jsx
// ❌ WRONG — useTheme returns the default (or null), often crashing later with a confusing error.
function Standalone() {
  const { theme } = useTheme(); // no <ThemeProvider> above this → ctx is null
  return <div className={theme} />;
}
```

```jsx
// ✅ RIGHT — the custom hook throws a clear error, and you wrap the component.
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (ctx === null) throw new Error('useTheme must be used inside <ThemeProvider>');
  return ctx;
}
// ...render <ThemeProvider><Standalone /></ThemeProvider>
```

Why it matters: the `createContext(null)` + throwing custom hook turns a silent, far-away bug into an immediate, descriptive one.

**3. Putting everything in one giant context**

```jsx
// ❌ WRONG — one mega-context. A theme toggle re-renders cart components and vice versa.
const AppContext = createContext(null);
// value = { user, theme, locale, cart, dispatch, ... }
```

```jsx
// ✅ RIGHT — separate concerns into separate contexts.
const AuthContext = createContext(null);
const ThemeContext = createContext(null);
const CartContext = createContext(null);
```

Why it matters: a context's consumers all re-render together. Unrelated data in one context couples unrelated re-renders.

**4. Expecting `defaultValue` to apply when `value` is omitted**

```jsx
// ❌ WRONG mental model: thinks consumers get 'light' here.
const ThemeContext = createContext('light');
// <ThemeContext>{children}</ThemeContext>  // value is undefined, NOT 'light'
```

Rendering a provider *without* a `value` prop broadcasts `undefined`, not the default. The default applies **only when there is no provider at all**. Always pass `value` explicitly.

**5. Trying to read context conditionally with `useContext`**

```jsx
// ❌ WRONG — breaks the rules of hooks.
if (loggedIn) {
  const { user } = useContext(AuthContext);
}
```

```jsx
// ✅ RIGHT — read at the top level; branch on the value.
const { user } = useContext(AuthContext);
if (loggedIn) {
  /* use user */
}
```

(If you genuinely need a conditional read, React 19's `use(AuthContext)` is allowed after an early return — but prefer the simple top-level read.)

---

## Best practices

- **Do** wrap each context in a custom `useX()` hook that throws if the provider is missing. Export the hook, keep the raw context object private to the module.
- **Do** memoize object/array `value`s with `useMemo`, and wrap functions you put in the value with `useCallback`.
- **Do** split a store into separate state and dispatch contexts (or separate contexts per concern) so consumers subscribe only to what they need.
- **Do** keep provider state-ownership co-located: the `useState`/`useReducer` lives in the provider component, not scattered.
- **Do** use `useReducer` + Context for multi-action shared state (cart, wizard, editor) — it's a real store with zero dependencies.
- **Don't** put high-frequency, fast-changing values (mouse/scroll position, animation frames) in a widely-read context.
- **Don't** create a new context value object inline in JSX without memoizing.
- **Don't** reach for Context to avoid passing a prop *one* level — that's overkill; just pass the prop.
- **Don't** treat Context as a cache for server data; use a data-fetching library for that.
- **Don't** read a context outside its provider and rely on the default for real logic — make missing providers loud.

---

## Exercises

1. **Locale context (warm-up).** Create a `LocaleProvider` holding a `locale` string (`'en'` or `'es'`) and a `setLocale` updater. Expose a `useLocale()` custom hook. Add a `<select>` that switches locale and a `<Greeting>` component deep in the tree that shows `"Hello"` for `'en'` and `"Hola"` for `'es'`. Memoize the value.

2. **Notification context (intermediate).** Build a `NotificationProvider` that holds an array of toast messages. Expose `notify(message)` (adds a toast) and `dismiss(id)` (removes one). Any component should be able to call `useNotifications().notify('Saved!')`. Render the toasts from a `<Toaster />` near the root. Use `useCallback` so the functions are stable.

3. **Split cart store (intermediate→advanced).** Take the cart from Example 5 and add a `total` derived from item prices. Add a `<MiniCart>` showing count + total and a `<ClearButton>`. Prove that the `<ClearButton>` does **not** re-render when items are added (hint: it should only consume dispatch; add a `console.log('ClearButton render')`).

4. **Combine contexts (advanced).** Compose `AuthProvider`, `ThemeProvider`, and your `LocaleProvider` into a single `<AppProviders>`. Make a `ThemedGreeting` component that reads all three: it greets the logged-in user by name (or "guest"), in the chosen locale, styled by the chosen theme. Then write a tiny render check confirming only the affected consumers re-render when each value changes.

### Solutions

**Exercise 1 — Locale context**

```jsx
// locale-context.jsx
import { createContext, useContext, useState, useMemo } from 'react';

const LocaleContext = createContext(null);
const DICT = { en: { hello: 'Hello' }, es: { hello: 'Hola' } };

export function LocaleProvider({ children }) {
  const [locale, setLocale] = useState('en');
  const value = useMemo(
    () => ({ locale, setLocale, t: (key) => DICT[locale][key] }),
    [locale],
  );
  return <LocaleContext value={value}>{children}</LocaleContext>;
}

export function useLocale() {
  const ctx = useContext(LocaleContext);
  if (ctx === null) throw new Error('useLocale must be used inside <LocaleProvider>');
  return ctx;
}

// usage
function LocaleSwitcher() {
  const { locale, setLocale } = useLocale();
  return (
    <select value={locale} onChange={(e) => setLocale(e.target.value)}>
      <option value="en">English</option>
      <option value="es">Español</option>
    </select>
  );
}
function Greeting() {
  const { t } = useLocale();
  return <p>{t('hello')}!</p>;
}
```

**Exercise 2 — Notification context**

```jsx
// notification-context.jsx
import { createContext, useContext, useState, useCallback, useMemo } from 'react';

const NotificationContext = createContext(null);
let nextId = 1;

export function NotificationProvider({ children }) {
  const [toasts, setToasts] = useState([]);

  const dismiss = useCallback((id) => {
    setToasts((list) => list.filter((t) => t.id !== id));
  }, []);

  const notify = useCallback((message) => {
    const id = nextId++;
    setToasts((list) => [...list, { id, message }]);
    setTimeout(() => dismiss(id), 3000);
  }, [dismiss]);

  const value = useMemo(() => ({ toasts, notify, dismiss }), [toasts, notify, dismiss]);
  return <NotificationContext value={value}>{children}</NotificationContext>;
}

export function useNotifications() {
  const ctx = useContext(NotificationContext);
  if (ctx === null) throw new Error('useNotifications must be used inside <NotificationProvider>');
  return ctx;
}

export function Toaster() {
  const { toasts, dismiss } = useNotifications();
  return (
    <div className="toaster">
      {toasts.map((t) => (
        <div key={t.id} className="toast" onClick={() => dismiss(t.id)}>
          {t.message}
        </div>
      ))}
    </div>
  );
}
```

**Exercise 3 — Split cart store with derived total**

```jsx
// Building on Example 5's cart-store.jsx. Items now carry `price`.
function MiniCart() {
  const { items } = useCartState();
  const count = items.reduce((n, i) => n + i.qty, 0);
  const total = items.reduce((sum, i) => sum + i.qty * i.price, 0);
  return <span>🛒 {count} item(s) — ${total.toFixed(2)}</span>;
}

function ClearButton() {
  const dispatch = useCartDispatch(); // dispatch only → stable → no re-render on add
  console.log('ClearButton render'); // logs once on mount, then NOT on add
  return <button onClick={() => dispatch({ type: 'clear' })}>Clear cart</button>;
}
```

Because `ClearButton` subscribes only to `CartDispatchContext` and `dispatch` is referentially stable, adding items re-renders `MiniCart` but never `ClearButton`.

**Exercise 4 — Composed providers + cross-context consumer**

```jsx
// AppProviders.jsx
import { AuthProvider } from './auth-context';
import { ThemeProvider } from './ThemeProvider';
import { LocaleProvider } from './locale-context';

export function AppProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider>{children}</LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

// ThemedGreeting.jsx
import { useAuth } from './auth-context';
import { useTheme } from './ThemeProvider';
import { useLocale } from './locale-context';

export function ThemedGreeting() {
  const { user } = useAuth();
  const { theme } = useTheme();
  const { t } = useLocale();
  const name = user ? user.name : 'guest';
  return (
    <p className={theme}>
      {t('hello')}, {name}!
    </p>
  );
}
```

`ThemedGreeting` re-renders when *any* of the three contexts it reads changes — which is correct, since it displays all three. A component reading only one of them re-renders only on that one's change. That selectivity is the payoff of keeping the three concerns in separate contexts rather than one merged value.

---

## Key takeaways

- **Context solves prop drilling** by letting a Provider broadcast a value to its entire subtree, read directly via `useContext` — no courier props.
- The three pieces are **`createContext(default)`**, a **Provider** supplying `value`, and **`useContext`** reading the nearest provider's value. In **React 19**, render `<MyContext value={...}>` directly (no `.Provider`).
- The **`defaultValue`** applies only when there is **no provider** above the consumer — not when `value` is omitted.
- Put **state inside the provider** (`useState`/`useReducer`) and broadcast `[value, updater]` so consumers can both read and update shared data.
- **Every consumer re-renders when `value` changes** (compared by `Object.is`), even through `React.memo`. **Memoize** object values with `useMemo` and functions with `useCallback`.
- **Split state and dispatch** into separate contexts (or one context per concern) so components subscribe only to the slice they need; `useReducer` + Context is a real, dependency-free mini store.
- Context fits **low-frequency, broadly-read** data (auth, theme, locale, cart). For high-frequency updates, fine-grained selectors, or server caching, reach for a state/data library (`16`).
- Wrap each context in a **custom hook that throws** when its provider is missing — the highest-value convention in this lesson.

---

## Further reading

- React docs — **Passing Data Deeply with Context**: <https://react.dev/learn/passing-data-deeply-with-context>
- React docs — **Scaling Up with Reducer and Context**: <https://react.dev/learn/scaling-up-with-reducer-and-context>
- React reference — **`createContext`**: <https://react.dev/reference/react/createContext>
- React reference — **`useContext`**: <https://react.dev/reference/react/useContext>
- React reference — **`use`** (conditional context reads, React 19): <https://react.dev/reference/react/use>
- React reference — **`useReducer`**: <https://react.dev/reference/react/useReducer>
- React docs — **`React.memo`** (how memoization interacts with context): <https://react.dev/reference/react/memo>
- React 19 — **`<Context>` as a provider** (legacy `Context.Provider`; deprecation planned for a future version): <https://react.dev/blog/2024/12/05/react-19>
- Community — **`use-context-selector`** (fine-grained context subscriptions): <https://github.com/dai-shi/use-context-selector>
