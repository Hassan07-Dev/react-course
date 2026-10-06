# 16 — Global State Management

> **What you'll learn:** how to categorize state, escalate from local state to lifting up to Context to a dedicated library, and how the three dominant approaches — Context + `useReducer`, Zustand, and Redux Toolkit — solve the same problem differently.

- Classify any piece of state into one of five buckets (local UI, lifted/shared, server cache, URL, global app) and pick the right tool for each.
- Apply the "lift state up" pattern correctly, and recognize when lifting becomes prop-drilling pain.
- Build a medium-app global store with **Context + `useReducer`** and understand its re-render cost.
- Use **Redux Toolkit** the modern way: `configureStore`, `createSlice`, `useSelector`, `useDispatch`, `createAsyncThunk`, and **RTK Query**.
- Use **Zustand** with `create`, selectors, and the `persist` middleware — and explain why it's loved.
- Recognize **Jotai** and **Recoil** atoms at a glance.
- Apply a decision matrix to choose a state solution for a real project.

---

## Prerequisites

You should be comfortable with the material from earlier lessons:

- `useState` and the render-as-a-function-of-state mental model (lesson 06).
- `useReducer` for structured state transitions, and the `useContext` hook (see **09-hooks-deep-dive.md**).
- `useEffect` and cleanup for side effects (see **08-useeffect-side-effects.md**).
- Custom hooks for extracting reusable stateful logic (see **10-custom-hooks.md**).
- A working Vite project (`npm create vite@latest`) with React 19.

If "Context + `useReducer`" is unfamiliar, read lesson 12 first — this lesson builds directly on it.

---

## Concept

"State management" sounds like one problem, but it's really five problems wearing a trench coat. The single most valuable skill in this lesson is **classification**: before reaching for any tool, decide which *kind* of state you have.

### The five categories of state

1. **Local UI state** — owned by exactly one component and irrelevant to anyone else. A dropdown's open/closed flag, the text in a search box before submit, a hover state. **Tool: `useState`/`useReducer` inside the component.** Don't globalize it.

2. **Lifted / shared state** — two or three nearby components need the same value. A parent form needs to know what its child inputs hold. **Tool: lift the state to the nearest common ancestor and pass it down as props.**

3. **Server cache state** — data that *lives on a server* and you're caching a copy in the browser: the list of products, the current user's profile, a paginated feed. This is the category people most often get wrong. It is **not** really "your" state — it's a cache of someone else's state, with staleness, refetching, deduplication, and background sync concerns. **Tool: a server-cache library like React Query (TanStack Query) or RTK Query** — *not* a generic global store. (Covered in depth in **15-data-fetching.md**; mentioned here so you don't shove it into a global store.)

4. **URL state** — state that belongs in the address bar: the current route, the active tab, search filters, pagination page. If a user should be able to bookmark or share it, it's URL state. **Tool: the router** (search params, route params — see **14-routing.md**).

5. **Global app state** — genuinely cross-cutting client state that many unrelated components read and write: the auth/session object, the theme, a feature-flag set, the shopping **cart**. This is the only category that truly needs "global state management." **Tool: Context, Zustand, Redux Toolkit, Jotai, etc.**

> The expert move: most of what beginners think is "global state" is actually server cache (category 3) or URL state (category 4). Pull those out first, and the genuinely-global pile shrinks dramatically — often to just auth + theme + maybe a cart.

### The escalation ladder

You should always start at the bottom rung and climb only when you feel real pain:

```
useState in one component
        │  (a sibling needs it too)
        ▼
lift state up to a common parent  →  pass via props
        │  (props threaded through 5+ layers — "prop drilling")
        ▼
Context (+ useReducer for non-trivial transitions)
        │  (too many re-renders / store is large & structured / need devtools / async-heavy)
        ▼
a dedicated library (Zustand, Redux Toolkit, Jotai…)
```

**Lifting state up** is the foundational pattern. If a `<PriceInput>` and a `<ResultLabel>` both depend on a quantity, you move the quantity into their shared parent and pass `value` down plus an `onChange` callback up. State has a single source of truth, and children stay "dumb." You lift to the *nearest* common ancestor — no higher, or you create unnecessary re-renders.

The pain that ends lifting is **prop drilling**: threading a prop through many intermediate components that don't use it, just to reach a deep leaf. That's the signal to reach for Context.

### Why Context isn't a state manager (but is a great delivery mechanism)

Context (lesson 12) solves *delivery* — it teleports a value down the tree without prop drilling. It does **not** solve *re-render efficiency*. When a context value changes, **every** component that calls `useContext` for that context re-renders, even if it only cares about an unchanged slice. For a low-frequency value like theme, that's fine. For a high-frequency value like a cart that updates on every keystroke of a quantity field, it can get slow. Libraries like Zustand and Redux exist largely to solve this **selector problem**: subscribe to a *slice* and re-render only when that slice changes.

---

## Syntax & API

### Lifting state up (no library — plain React)

```jsx
function Parent() {
  const [qty, setQty] = useState(1);          // single source of truth
  return (
    <>
      <PriceInput value={qty} onChange={setQty} />   // down: data, up: callback
      <ResultLabel qty={qty} />
    </>
  );
}
```

### Context + useReducer (built-in)

```jsx
import { createContext, useContext, useReducer } from 'react';

const StoreContext = createContext(null);           // value flows through here

function reducer(state, action) { /* return next state */ }

function StoreProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <StoreContext.Provider value={{ state, dispatch }}>
      {children}
    </StoreContext.Provider>
  );
}

const useStore = () => useContext(StoreContext);     // custom consumer hook
```

> React 19 lets you render `<StoreContext>` directly as a provider (without `.Provider`). Both forms work in 19; `.Provider` is shown here for clarity and broad compatibility.

### Redux Toolkit (RTK) — the modern Redux

```bash
npm install @reduxjs/toolkit react-redux
```

| API | Signature (abridged) | Notes |
|---|---|---|
| `configureStore` | `configureStore({ reducer, middleware?, ... })` | Creates the store. Bundles redux-thunk + DevTools automatically. |
| `createSlice` | `createSlice({ name, initialState, reducers, extraReducers? })` | Returns `{ reducer, actions }`. Reducers use **Immer** so you can "mutate." |
| `<Provider store>` | from `react-redux` | Wraps the app once. |
| `useSelector` | `useSelector(state => state.x)` | Subscribes to a slice; re-renders only when the selected value changes (`===`). |
| `useDispatch` | `const dispatch = useDispatch()` | Returns the dispatch function. |
| `createAsyncThunk` | `createAsyncThunk(typePrefix, payloadCreator)` | Generates `pending`/`fulfilled`/`rejected` action types for async flows. |
| `createApi` (RTK Query) | `createApi({ baseQuery, endpoints })` | Full data-fetching/caching layer; generates hooks like `useGetXQuery`. |

### Zustand

```bash
npm install zustand
```

| API | Signature | Notes |
|---|---|---|
| `create` | `create((set, get) => ({ ... }))` | Returns a hook. `set` patches state (shallow-merges); `get` reads current state. |
| store hook | `useStore(state => state.slice)` | Pass a selector to subscribe to a slice — no Provider needed. |
| `persist` | `persist(config, { name })` middleware | Persists to `localStorage` (or custom storage). |
| `useShallow` | `useStore(useShallow(s => ({...})))` | From `zustand/react/shallow`; shallow-compares an object selector. |

### Jotai / Recoil (atoms — brief)

```bash
npm install jotai      # Jotai
```

```jsx
import { atom, useAtom } from 'jotai';
const countAtom = atom(0);                    // an atom = one unit of state
function C() { const [c, setC] = useAtom(countAtom); /* ... */ }
```

Recoil's API is similar (`atom({ key, default })`, `useRecoilState`) but Recoil is effectively unmaintained (archived in 2025) — **prefer Jotai** for a fresh atomic-state project.

---

## Examples

We'll build the **same shopping cart** three ways — Context, Zustand, RTK — so you can compare them directly. Each cart supports: add an item, change its quantity, remove it, and read a derived total.

### Example 1 — Cart with Context + useReducer

```jsx
// CartContext.jsx
import { createContext, useContext, useReducer, useMemo } from 'react';

const CartContext = createContext(null);

const initialState = { items: [] }; // items: { id, name, price, qty }

function cartReducer(state, action) {
  switch (action.type) {
    case 'add': {
      const found = state.items.find((i) => i.id === action.item.id);
      if (found) {
        return {
          items: state.items.map((i) =>
            i.id === action.item.id ? { ...i, qty: i.qty + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { ...action.item, qty: 1 }] };
    }
    case 'setQty':
      return {
        items: state.items.map((i) =>
          i.id === action.id ? { ...i, qty: Math.max(1, action.qty) } : i
        ),
      };
    case 'remove':
      return { items: state.items.filter((i) => i.id !== action.id) };
    default:
      return state;
  }
}

export function CartProvider({ children }) {
  const [state, dispatch] = useReducer(cartReducer, initialState);
  // Memoize so the context value identity is stable unless state changes.
  const value = useMemo(() => ({ state, dispatch }), [state]);
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

export function useCart() {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error('useCart must be used inside <CartProvider>');
  return ctx;
}
```

```jsx
// App.jsx
import { CartProvider, useCart } from './CartContext';

function AddButton() {
  const { dispatch } = useCart();
  const item = { id: 'sku-1', name: 'Coffee Mug', price: 12 };
  return <button onClick={() => dispatch({ type: 'add', item })}>Add Mug</button>;
}

function CartList() {
  const { state, dispatch } = useCart();
  const total = state.items.reduce((sum, i) => sum + i.price * i.qty, 0);
  return (
    <ul>
      {state.items.map((i) => (
        <li key={i.id}>
          {i.name} — ${i.price} ×
          <input
            type="number"
            value={i.qty}
            onChange={(e) =>
              dispatch({ type: 'setQty', id: i.id, qty: Number(e.target.value) })
            }
          />
          <button onClick={() => dispatch({ type: 'remove', id: i.id })}>x</button>
        </li>
      ))}
      <li><strong>Total: ${total}</strong></li>
    </ul>
  );
}

export default function App() {
  return (
    <CartProvider>
      <AddButton />
      <CartList />
    </CartProvider>
  );
}
```

**What to notice:** The reducer keeps all transition logic in one pure function. `useMemo` stabilizes the context object. The cost: every consumer of `useCart` re-renders on *any* cart change — `AddButton` re-renders even though it reads nothing from `state`. For a small cart this is invisible; at scale it's the reason people leave Context.

### Example 2 — The same cart in Zustand

```jsx
// cartStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useCartStore = create(
  persist(
    (set, get) => ({
      items: [],
      add: (item) =>
        set((state) => {
          const found = state.items.find((i) => i.id === item.id);
          return found
            ? {
                items: state.items.map((i) =>
                  i.id === item.id ? { ...i, qty: i.qty + 1 } : i
                ),
              }
            : { items: [...state.items, { ...item, qty: 1 }] };
        }),
      setQty: (id, qty) =>
        set((state) => ({
          items: state.items.map((i) =>
            i.id === id ? { ...i, qty: Math.max(1, qty) } : i
          ),
        })),
      remove: (id) =>
        set((state) => ({ items: state.items.filter((i) => i.id !== id) })),
      total: () => get().items.reduce((s, i) => s + i.price * i.qty, 0),
    }),
    { name: 'cart-storage' } // localStorage key — survives refresh, for free
  )
);
```

```jsx
// App.jsx (Zustand — no Provider!)
import { useCartStore } from './cartStore';

function AddButton() {
  // Subscribe ONLY to the `add` action → this component never re-renders on item changes.
  const add = useCartStore((s) => s.add);
  return (
    <button onClick={() => add({ id: 'sku-1', name: 'Coffee Mug', price: 12 })}>
      Add Mug
    </button>
  );
}

function CartList() {
  const items = useCartStore((s) => s.items);   // re-renders only when items change
  const setQty = useCartStore((s) => s.setQty);
  const remove = useCartStore((s) => s.remove);
  const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
  return (
    <ul>
      {items.map((i) => (
        <li key={i.id}>
          {i.name} — ${i.price} ×
          <input
            type="number"
            value={i.qty}
            onChange={(e) => setQty(i.id, Number(e.target.value))}
          />
          <button onClick={() => remove(i.id)}>x</button>
        </li>
      ))}
      <li><strong>Total: ${total}</strong></li>
    </ul>
  );
}

export default function App() {
  return (
    <>
      <AddButton />
      <CartList />
    </>
  );
}
```

**What to notice:** No `<Provider>`, no context boilerplate. `AddButton` selects only `s.add`, so it is *never* re-rendered by quantity changes — the selector model fixes Context's over-rendering. `persist` gives free `localStorage` durability with one config line. The store is just a hook you import anywhere, including outside React.

### Example 3 — The same cart in Redux Toolkit

```jsx
// cartSlice.js
import { createSlice } from '@reduxjs/toolkit';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    // Immer lets you "mutate" — RTK produces an immutable update under the hood.
    add(state, action) {
      const found = state.items.find((i) => i.id === action.payload.id);
      if (found) found.qty += 1;
      else state.items.push({ ...action.payload, qty: 1 });
    },
    setQty(state, action) {
      const i = state.items.find((x) => x.id === action.payload.id);
      if (i) i.qty = Math.max(1, action.payload.qty);
    },
    remove(state, action) {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
  },
});

export const { add, setQty, remove } = cartSlice.actions; // auto-generated action creators
export default cartSlice.reducer;

// A reusable selector — co-locating derived state next to the slice is a good habit.
export const selectTotal = (state) =>
  state.cart.items.reduce((s, i) => s + i.price * i.qty, 0);
```

```jsx
// store.js
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';

export const store = configureStore({
  reducer: { cart: cartReducer }, // -> state.cart
});
```

```jsx
// App.jsx (RTK)
import { Provider, useSelector, useDispatch } from 'react-redux';
import { store } from './store';
import { add, setQty, remove, selectTotal } from './cartSlice';

function AddButton() {
  const dispatch = useDispatch();
  return (
    <button onClick={() => dispatch(add({ id: 'sku-1', name: 'Coffee Mug', price: 12 }))}>
      Add Mug
    </button>
  );
}

function CartList() {
  const items = useSelector((s) => s.cart.items);
  const total = useSelector(selectTotal);
  const dispatch = useDispatch();
  return (
    <ul>
      {items.map((i) => (
        <li key={i.id}>
          {i.name} — ${i.price} ×
          <input
            type="number"
            value={i.qty}
            onChange={(e) => dispatch(setQty({ id: i.id, qty: Number(e.target.value) }))}
          />
          <button onClick={() => dispatch(remove(i.id))}>x</button>
        </li>
      ))}
      <li><strong>Total: ${total}</strong></li>
    </ul>
  );
}

export default function App() {
  return (
    <Provider store={store}>
      <AddButton />
      <CartList />
    </Provider>
  );
}
```

**What to notice:** `createSlice` generates the action creators (`add`, `setQty`, `remove`) for you — no hand-written action-type constants. The "mutating" syntax (`found.qty += 1`, `state.items.push`) is **Immer** drafting; you never actually mutate the real state. `useSelector` gives the same slice-level subscription Zustand has. The trade-off is more files and ceremony — which pays off when the app is large and structured.

### Example 4 — Async with `createAsyncThunk`

```jsx
// productsSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchProducts = createAsyncThunk('products/fetch', async () => {
  const res = await fetch('/api/products');
  if (!res.ok) throw new Error('Failed to load products');
  return res.json();                       // becomes action.payload of `fulfilled`
});

const productsSlice = createSlice({
  name: 'products',
  initialState: { list: [], status: 'idle', error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (s) => { s.status = 'loading'; })
      .addCase(fetchProducts.fulfilled, (s, a) => { s.status = 'ready'; s.list = a.payload; })
      .addCase(fetchProducts.rejected, (s, a) => { s.status = 'error'; s.error = a.error.message; });
  },
});

export default productsSlice.reducer;
```

```jsx
import { useEffect } from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { fetchProducts } from './productsSlice';

function Products() {
  const dispatch = useDispatch();
  const { list, status } = useSelector((s) => s.products);
  useEffect(() => { dispatch(fetchProducts()); }, [dispatch]);
  if (status === 'loading') return <p>Loading…</p>;
  return <ul>{list.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

`createAsyncThunk` turns one async function into three lifecycle actions. **But notice:** this is *server cache* (category 3). RTK Query does this better.

### Example 5 — RTK Query (the right tool for server cache)

```jsx
// productsApi.js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const productsApi = createApi({
  reducerPath: 'productsApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api/' }),
  endpoints: (build) => ({
    getProducts: build.query({ query: () => 'products' }),
  }),
});

export const { useGetProductsQuery } = productsApi; // auto-generated hook
```

```jsx
// store.js — wire the API reducer + middleware
import { configureStore } from '@reduxjs/toolkit';
import { productsApi } from './productsApi';

export const store = configureStore({
  reducer: { [productsApi.reducerPath]: productsApi.reducer },
  middleware: (getDefault) => getDefault().concat(productsApi.middleware),
});
```

```jsx
import { useGetProductsQuery } from './productsApi';

function Products() {
  const { data = [], isLoading, isError } = useGetProductsQuery();
  if (isLoading) return <p>Loading…</p>;
  if (isError) return <p>Failed.</p>;
  return <ul>{data.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

One `build.query` declaration generates a fully-cached, auto-deduplicated, auto-refetching hook — no thunks, no loading-flag bookkeeping. This is the RTK equivalent of React Query (lesson 17). **If your app is already on Redux, use RTK Query for server data instead of `createAsyncThunk`.**

---

## Under the hood

**Context's re-render rule.** A `Context.Provider` notifies every consumer whenever its `value` changes by **reference identity** (`Object.is`). If you write `value={{ state, dispatch }}` inline, you create a *new object every render*, so all consumers re-render every time the provider's parent renders — even if `state` didn't change. That's why Example 1 wraps `value` in `useMemo`. Critically, `useContext` has **no selector**: you cannot subscribe to "just `state.items`." Any change to the provided value re-renders all consumers. This is fundamental, not a bug.

**The selector model (Zustand & Redux).** Both libraries keep state in an external store and use React 18+'s `useSyncExternalStore` under the hood. When you pass a selector `s => s.items`, the library runs it after each store update and compares the result to the previous result (`===` by default). Your component re-renders **only when the selected value's reference changes**. This is the key efficiency win over raw Context. It also means: if your selector returns a *new object/array literal every call* (e.g. `s => ({ a: s.a, b: s.b })`), the `===` check always fails and you re-render constantly — fix it with `useShallow` (Zustand) or `useSelector` with a shallow-equality function / memoized selector (Redux).

**Immer in RTK.** Inside a `createSlice` reducer, `state` is an Immer **draft** — a Proxy that records the "mutations" you perform and produces a brand-new immutable state from them. You may **either** mutate the draft **or** return a new value, but never both in one reducer. This is why `state.items.push(...)` is safe: nothing real is mutated; Immer replays your writes onto a structural copy.

**Zustand's `set` shallow-merges at the top level only.** `set({ items: [...] })` replaces `items` and leaves sibling keys intact, but nested objects are *not* deep-merged — you must spread them yourself. `set` also accepts an updater function `set(state => ...)` for transitions that depend on current state, which is the safe form.

**No Provider in Zustand** because the store is a module-level singleton living outside React. The downside: a singleton store is shared across all consumers, which complicates SSR and isolated tests (state leaks between tests unless you reset it). For those cases Zustand offers a Context-based store factory, but for client apps the singleton is the loved default.

**Server state is special.** Category 3 (server cache) deserves its own machinery because it has properties global stores don't model: staleness, refetch-on-focus, request deduplication, pagination caches, optimistic updates with rollback. Forcing it into a generic store means you reimplement all of that by hand. This is the single biggest architectural lesson here — see **15-data-fetching.md**.

---

## Common mistakes & pitfalls

**1. Putting server data in a global store.**

```jsx
// WRONG — manual cache, manual loading flags, stale forever
const useStore = create((set) => ({
  users: [], loading: false,
  fetchUsers: async () => {
    set({ loading: true });
    const r = await fetch('/api/users');
    set({ users: await r.json(), loading: false });
  },
}));
```

```jsx
// RIGHT — server cache belongs in React Query / RTK Query
const { data: users, isLoading } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then((r) => r.json()),
});
// caching, dedup, refetch, staleness — all handled. (See lesson 17.)
```

**2. Inline (unmemoized) Context value.**

```jsx
// WRONG — new object every render → all consumers re-render
<CartContext.Provider value={{ state, dispatch }}>{children}</CartContext.Provider>
```

```jsx
// RIGHT
const value = useMemo(() => ({ state, dispatch }), [state]);
<CartContext.Provider value={value}>{children}</CartContext.Provider>
```

**3. Selector that returns a fresh object every render (Zustand).**

```jsx
// WRONG — new object identity each call → re-renders on EVERY store change
const { add, remove } = useCartStore((s) => ({ add: s.add, remove: s.remove }));
```

```jsx
// RIGHT — separate atomic selectors, OR useShallow for an object selector
import { useShallow } from 'zustand/react/shallow';
const { add, remove } = useCartStore(useShallow((s) => ({ add: s.add, remove: s.remove })));
// or: const add = useCartStore((s) => s.add); const remove = useCartStore((s) => s.remove);
```

**4. Mutating state outside Immer (plain React / Zustand `set`).**

```jsx
// WRONG — mutates existing array; React/selectors don't detect the change
const add = (item) => set((state) => { state.items.push(item); return state; });
```

```jsx
// RIGHT — return a new array reference
const add = (item) => set((state) => ({ items: [...state.items, item] }));
```

**5. Reaching for Redux on day one.** Spinning up `configureStore`, slices, and `<Provider>` for a three-component toy is over-engineering. Climb the ladder; adopt a library only when Context's re-renders, the store's size, or async complexity actually hurt.

**6. Forgetting RTK Query middleware.** `createApi` only works if you `.concat(api.middleware)` in `configureStore` and add `[api.reducerPath]: api.reducer`. Skip either and queries silently never cache or refetch.

---

## Best practices

- **Classify first.** Decide local vs. lifted vs. server vs. URL vs. global *before* choosing a tool. Most "global state" is actually server cache or URL state.
- **Start small, escalate on real pain.** `useState` → lift up → Context → library. Don't skip rungs preemptively.
- **Keep server state out of your global store.** Use React Query or RTK Query for anything that came from a server.
- **Always subscribe with selectors** in Zustand/Redux. Select the narrowest slice you need; avoid object-literal selectors without `useShallow`.
- **Co-locate logic.** Keep reducers/actions/selectors next to the slice or store, not scattered.
- **Memoize Context values** with `useMemo`, and consider splitting state and dispatch into two contexts so dispatch-only consumers don't re-render.
- **Prefer RTK over hand-rolled Redux.** Never write raw `switch` reducers, action-type string constants, or `connect` for new code.
- **Don't over-globalize.** If only two sibling components need a value, lift it — don't put it in the global store.
- **Persist deliberately.** `persist` (Zustand) or `redux-persist` is great for cart/theme, but never persist sensitive tokens to `localStorage`.

---

## Exercises

1. **Classify.** For each, name the category (local / lifted / server / URL / global) and the tool you'd pick: (a) a modal's open flag, (b) the logged-in user object, (c) the list of blog posts from `/api/posts`, (d) the active filter `?status=active` in the URL, (e) the value of a controlled `<input>` two siblings both read.

2. **Lift state up.** Build a `<TemperatureConverter>` with a Celsius input and a Fahrenheit input that stay in sync. The shared temperature must live in the parent; both inputs are controlled and edit the same source of truth.

3. **Context counter store.** Build a Context + `useReducer` store with `increment`, `decrement`, and `reset`. Memoize the context value. Add a `<Display>` and a `<Controls>` component; confirm both read from the same store.

4. **Zustand to-do store.** Create a Zustand store with `todos`, `addTodo(text)`, `toggle(id)`, and `remove(id)`. Use `persist` so todos survive a refresh. In the list component, subscribe to `todos` only; in the add form, subscribe to `addTodo` only.

5. **RTK slice + thunk.** Build a `counterSlice` with `increment`/`decrement` reducers and an `incrementAsync` thunk (via `createAsyncThunk`) that resolves after a 1-second delay, then adds 1. Wire it with `configureStore` and `<Provider>`.

### Solutions

**Solution 2 — Lifted temperature converter**

```jsx
function TemperatureConverter() {
  const [celsius, setCelsius] = useState(20); // single source of truth
  const fahrenheit = celsius * 9 / 5 + 32;
  return (
    <div>
      <label>°C
        <input type="number" value={celsius}
          onChange={(e) => setCelsius(Number(e.target.value))} />
      </label>
      <label>°F
        <input type="number" value={Math.round(fahrenheit)}
          onChange={(e) => setCelsius((Number(e.target.value) - 32) * 5 / 9)} />
      </label>
    </div>
  );
}
```

**Solution 3 — Context counter store**

```jsx
import { createContext, useContext, useReducer, useMemo } from 'react';

const CounterContext = createContext(null);
const reducer = (n, action) => {
  switch (action.type) {
    case 'inc': return n + 1;
    case 'dec': return n - 1;
    case 'reset': return 0;
    default: return n;
  }
};

function CounterProvider({ children }) {
  const [count, dispatch] = useReducer(reducer, 0);
  const value = useMemo(() => ({ count, dispatch }), [count]);
  return <CounterContext.Provider value={value}>{children}</CounterContext.Provider>;
}
const useCounter = () => useContext(CounterContext);

const Display = () => <h1>{useCounter().count}</h1>;
function Controls() {
  const { dispatch } = useCounter();
  return (
    <>
      <button onClick={() => dispatch({ type: 'inc' })}>+</button>
      <button onClick={() => dispatch({ type: 'dec' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>reset</button>
    </>
  );
}
export default function App() {
  return <CounterProvider><Display /><Controls /></CounterProvider>;
}
```

**Solution 4 — Zustand to-do store**

```jsx
// todoStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useTodos = create(
  persist(
    (set) => ({
      todos: [],
      addTodo: (text) =>
        set((s) => ({ todos: [...s.todos, { id: crypto.randomUUID(), text, done: false }] })),
      toggle: (id) =>
        set((s) => ({ todos: s.todos.map((t) => (t.id === id ? { ...t, done: !t.done } : t)) })),
      remove: (id) => set((s) => ({ todos: s.todos.filter((t) => t.id !== id) })),
    }),
    { name: 'todos' }
  )
);
```

```jsx
import { useTodos } from './todoStore';

function AddForm() {
  const addTodo = useTodos((s) => s.addTodo); // subscribe to action only
  const [text, setText] = useState('');
  return (
    <form onSubmit={(e) => { e.preventDefault(); if (text) { addTodo(text); setText(''); } }}>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button>Add</button>
    </form>
  );
}
function List() {
  const todos = useTodos((s) => s.todos);     // subscribe to data only
  const toggle = useTodos((s) => s.toggle);
  const remove = useTodos((s) => s.remove);
  return (
    <ul>
      {todos.map((t) => (
        <li key={t.id}>
          <span onClick={() => toggle(t.id)} style={{ textDecoration: t.done ? 'line-through' : '' }}>
            {t.text}
          </span>
          <button onClick={() => remove(t.id)}>x</button>
        </li>
      ))}
    </ul>
  );
}
export default function App() { return (<><AddForm /><List /></>); }
```

**Solution 5 — RTK slice + async thunk**

```jsx
// counterSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const incrementAsync = createAsyncThunk('counter/incAsync', async () => {
  await new Promise((r) => setTimeout(r, 1000));
  return 1;
});

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (s) => { s.value += 1; },
    decrement: (s) => { s.value -= 1; },
  },
  extraReducers: (b) =>
    b.addCase(incrementAsync.fulfilled, (s, a) => { s.value += a.payload; }),
});

export const { increment, decrement } = counterSlice.actions;
export default counterSlice.reducer;
```

```jsx
// store.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';
export const store = configureStore({ reducer: { counter: counterReducer } });
```

```jsx
// App.jsx
import { Provider, useSelector, useDispatch } from 'react-redux';
import { store } from './store';
import { increment, decrement, incrementAsync } from './counterSlice';

function Counter() {
  const value = useSelector((s) => s.counter.value);
  const dispatch = useDispatch();
  return (
    <div>
      <h1>{value}</h1>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
      <button onClick={() => dispatch(incrementAsync())}>+1 (async)</button>
    </div>
  );
}
export default function App() { return (<Provider store={store}><Counter /></Provider>); }
```

---

## Key takeaways

- **Classify before you tool.** Local UI, lifted/shared, server cache, URL, and global app state each have a different right answer.
- **Lift state up** to the nearest common ancestor first; reach for Context only when prop-drilling actually hurts.
- **Context delivers a value but has no selectors** — every consumer re-renders on any change. Memoize the value; split state/dispatch contexts to reduce churn.
- **Zustand** = minimal global state: `create((set, get) => …)`, selector subscriptions, no Provider, `persist` for free durability. Loved for its simplicity.
- **Redux Toolkit** = structured global state at scale: `configureStore` + `createSlice` (Immer-powered, auto-generated actions) + `useSelector`/`useDispatch`, with `createAsyncThunk` and **RTK Query** for async/server data.
- **Server cache is not global state** — use React Query or RTK Query for it (lesson 17).
- **Decision matrix:** server data → React Query / RTK Query; simple global client state → Zustand (or Context for low-frequency values like theme); large, structured, multi-team apps → Redux Toolkit; fine-grained atomic state → Jotai.

---

## Further reading

- React docs — [Managing State](https://react.dev/learn/managing-state) (sharing, lifting, scaling with reducers + context).
- React docs — [Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context) and [Scaling Up with Reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context).
- [Redux Toolkit — official docs](https://redux-toolkit.js.org/) and the [RTK Query overview](https://redux-toolkit.js.org/rtk-query/overview).
- [Zustand — GitHub & docs](https://github.com/pmndrs/zustand).
- [Jotai — official docs](https://jotai.org/).
- [TanStack Query (React Query) docs](https://tanstack.com/query/latest) — the canonical server-cache library (see lesson 17).
- Kent C. Dodds — [Application State Management with React](https://kentcdodds.com/blog/application-state-management-with-react) (the "right tool per category" philosophy).
