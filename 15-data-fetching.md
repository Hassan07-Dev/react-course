# 15 — Data Fetching & Async UI

> **What you'll learn:** how to fetch data into a React component *correctly* — first by hand with effects, then with the libraries that exist precisely because the hand-rolled version is so easy to get wrong — and how React 19's `use()` hook plus Suspense let you read promises during render.

By the end of this lesson you will be able to:

- Fetch data inside an effect with proper `loading` / `error` / `data` state, an async helper, and dependency tracking on the query parameter.
- Cancel stale requests with `AbortController` and reason about **race conditions** so a slow earlier response can't overwrite a fast later one.
- Articulate *why* hand-rolled fetching collapses under real-world needs: caching, deduplication, refetching, and stale-data invalidation.
- Use **TanStack Query** (`QueryClient`, `QueryClientProvider`, `useQuery`, `useMutation`, `invalidateQueries`, optimistic updates) and explain why it is the de-facto standard.
- Recognize **SWR** as the lighter alternative and know when to reach for it.
- Read a promise during render with the **`use()` hook** behind a `<Suspense>` boundary and an error boundary.
- Preview the optimistic-UI primitives (`useOptimistic`) and know where the deep dive lives.
- Implement the *same* endpoint two ways — manual effect vs TanStack Query — and see exactly how much code the library removes.

---

## Prerequisites

Before this lesson you should be comfortable with:

- **`useState` and immutable updates** (see `07-state-usestate.md`) — every fetching pattern juggles at least loading/error/data state.
- **`useEffect`, cleanup, and dependency arrays** (see `08-useeffect-side-effects.md`) — the manual pattern *is* an effect, and its cleanup function is where cancellation lives. This lesson assumes you internalized why effects re-run when a dependency changes.
- **Rendering lists & conditionals** (see `05-rendering-lists-conditionals.md`) — you'll branch on status and map over results.
- **JavaScript promises, `async/await`, and `fetch`** (see `01-js-foundations.md`).

You'll also want a Vite + React 19 project to paste examples into:

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

We touch **Error Boundaries** here but the full treatment is in `20-advanced-patterns.md`, and the deep dive on **`useOptimistic`** and form actions is in `22-modern-react-19.md`.

---

## Concept

Rendering is **synchronous and pure**: given props and state, a component returns JSX *right now*. But data lives on a server, and the network is **asynchronous** — the answer arrives milliseconds or seconds later, possibly out of order, possibly never. The entire challenge of data fetching is bridging that gap: how do you turn "a request is in flight" into something React can render *now*, then update the screen when reality arrives?

The honest mental model is that **every fetch is really three states layered on top of each other**, not one value:

- **pending** — the request is in flight; we have nothing yet.
- **success** — we have data.
- **error** — the request failed (network down, 500, JSON parse error, timeout).

A beginner writes `const [data, setData] = useState()` and stops there. An expert always models all three, because a UI that ignores loading flashes empty content and a UI that ignores errors hangs forever on a spinner the day the API is down.

There are two fundamentally different *times* at which you can fetch:

1. **In an effect, after render** (the classic pattern). The component renders once in its loading state, then the effect fires, the data arrives, you call `setState`, and the component re-renders with data. This is "fetch-on-render."
2. **During render, by reading a promise** (React 19's `use()` + Suspense). The component "suspends" — React pauses it, shows a fallback, and resumes when the promise resolves. This pushes loading state out of your component and into a boundary.

Both are valid. The effect pattern works everywhere and is what you'll maintain in existing code. The `use()` pattern is newer, cleaner, and where React is heading — but it needs a promise *source* that doesn't recreate on every render (which is exactly what a caching library gives you for free).

The analogy: fetching in an effect is like **ordering takeout and watching the door** — you place the order (effect runs), put up a "food is coming" sign (loading state), and swap it for the meal when it arrives (setState). Fetching with `use()` + Suspense is like **a restaurant with a host stand**: you ask for your table, the host (Suspense) seats you in the waiting area (fallback) and only lets you into the dining room (your component renders with data) once it's ready. You never personally manage the "is the table ready yet?" flag — the host does.

---

## Syntax & API

### The manual effect pattern (built-ins only)

```jsx
import { useState, useEffect } from 'react';

useEffect(() => {
  const controller = new AbortController();      // cancellation token
  async function run() { /* fetch, set state */ }
  run();
  return () => controller.abort();               // cleanup cancels in-flight request
}, [queryParam]);                                 // re-run when the param changes
```

- **`AbortController`** — a Web API. `controller.signal` is passed to `fetch(url, { signal })`. Calling `controller.abort()` rejects the in-flight fetch with an `AbortError`, which you catch and ignore.
- **`fetch`** — does **not** reject on HTTP 4xx/5xx. You must check `response.ok` yourself and throw, otherwise a 404 looks like success.

### TanStack Query (`@tanstack/react-query` v5)

```bash
npm install @tanstack/react-query
```

```jsx
import { QueryClient, QueryClientProvider, useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

const queryClient = new QueryClient();           // one per app, holds the cache
```

- **`useQuery({ queryKey, queryFn, staleTime, enabled, ... })`** — declarative read. Returns `{ data, error, status, isPending, isLoading, isError, isFetching, refetch, ... }`.
  - **`queryKey`** — a serializable array that *uniquely identifies* this data, e.g. `['user', userId]`. The cache key and the dependency array in one.
  - **`queryFn`** — an async function returning the data (or throwing). Receives `{ signal }` so cancellation is automatic.
  - **`staleTime`** — how long (ms) cached data is considered *fresh*; while fresh, no background refetch happens. Default `0`.
  - **`gcTime`** — how long unused data stays in cache before garbage collection (default 5 min).
- **`useMutation({ mutationFn, onSuccess, onMutate, onError, onSettled })`** — for writes (POST/PUT/DELETE). Returns `{ mutate, mutateAsync, isPending, ... }`.
- **`queryClient.invalidateQueries({ queryKey })`** — marks matching queries stale and refetches them. The primary way to keep reads in sync after a write.

### React 19 `use()` + Suspense

```jsx
import { use, Suspense } from 'react';

function Profile({ userPromise }) {
  const user = use(userPromise);   // suspends until resolved; reads value when ready
  return <h1>{user.name}</h1>;
}

<Suspense fallback={<Spinner />}>
  <Profile userPromise={userPromise} />
</Suspense>
```

- **`use(promise)`** — reads the resolved value of a promise during render. If pending, the component suspends to the nearest `<Suspense>`. If rejected, it throws to the nearest **error boundary**.
- **`use(context)`** — also reads context, and unlike `useContext` it may be called conditionally / inside loops. (Context is covered in `12-context-api.md`.)
- **`<Suspense fallback={…}>`** — renders `fallback` while any descendant is suspended.

---

## Examples

### Example 1 — Manual fetch with loading / error / data

The baseline every React developer must be able to write without thinking.

```jsx
import { useState, useEffect } from 'react';

function UserCard({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    async function load() {
      try {
        const res = await fetch(`https://api.example.com/users/${userId}`, {
          signal: controller.signal,
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        setUser(data);
      } catch (err) {
        if (err.name !== 'AbortError') setError(err); // ignore intentional cancels
      } finally {
        // Only the *latest* effect reaches finally for a non-aborted run.
        if (!controller.signal.aborted) setLoading(false);
      }
    }

    load();
    return () => controller.abort();
  }, [userId]); // re-fetch whenever the id changes

  if (loading) return <p>Loading…</p>;
  if (error) return <p role="alert">Failed: {error.message}</p>;
  return <h2>{user.name}</h2>;
}
```

Why each piece matters:

- **`signal` + cleanup `abort()`** — if `userId` changes (or the component unmounts) before the response lands, the old request is canceled. Without this you both leak the request and risk a "set state on unmounted component" class of bug.
- **`if (!res.ok) throw`** — `fetch` resolves on a 500. You must manufacture the error.
- **`err.name !== 'AbortError'`** — an aborted request rejects; that's expected, not a real error, so we swallow it.

### Example 2 — The race condition, made explicit

Even *with* cancellation, the subtlest fetching bug is the **race condition**. Suppose the user types in a search box and we fetch on every keystroke. Request for `"re"` is slow; request for `"react"` is fast and returns first. If we naively `setResults`, the slow `"re"` response lands *last* and overwrites the correct `"react"` results — the UI shows stale data with no error.

`AbortController` already fixes this here (the old fetch is aborted before the new one starts), but there's a second, library-free guard worth knowing — an **ignore flag** — for cases where you can't abort the underlying work:

```jsx
import { useState, useEffect } from 'react';

function Search({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    let active = true; // closes over THIS effect run only
    const controller = new AbortController();

    fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal: controller.signal })
      .then((r) => r.json())
      .then((data) => {
        if (active) setResults(data); // stale runs have active === false
      })
      .catch((err) => {
        if (err.name !== 'AbortError') console.error(err);
      });

    return () => {
      active = false;       // mark this run stale
      controller.abort();   // and cancel the network call
    };
  }, [query]);

  return <ul>{results.map((r) => <li key={r.id}>{r.title}</li>)}</ul>;
}
```

Each effect run gets its **own** `active` variable via closure. When `query` changes, React runs the previous run's cleanup (`active = false`) *before* the new run, so a late-arriving stale response sees `active === false` and is dropped. This is the single most important idea in manual fetching: **only the latest request is allowed to write state.**

### Example 3 — Why this gets painful

Now imagine three components on the same page each fetch `/api/users/42`. With the manual pattern you fire **three identical requests**, store the result in three places, and have no shared way to refetch when the user is updated elsewhere. You'd hand-build: a cache keyed by URL, a way to dedupe in-flight requests, a "this data is 30 seconds old, refetch in the background" timer, and invalidation after writes. That's a few hundred lines of fiddly, bug-prone infrastructure — *per app*. This is the gap libraries fill.

### Example 4 — The same endpoint with TanStack Query

First, wrap your app once:

```jsx
// main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App.jsx';

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 30_000, retry: 1 } },
});

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </StrictMode>
);
```

Now the *entire* component from Example 1 collapses to this:

```jsx
import { useQuery } from '@tanstack/react-query';

async function fetchUser({ queryKey, signal }) {
  const [, userId] = queryKey;                 // ['user', userId]
  const res = await fetch(`https://api.example.com/users/${userId}`, { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

function UserCard({ userId }) {
  const { data: user, isPending, isError, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: fetchUser,
  });

  if (isPending) return <p>Loading…</p>;
  if (isError) return <p role="alert">Failed: {error.message}</p>;
  return <h2>{user.name}</h2>;
}
```

What you got for free, without writing a line of it:

- **Caching & dedupe** — render three `UserCard`s with the same `userId` and only **one** network request fires; all three read the same cache entry.
- **Cancellation** — the `signal` is provided by the library and wired to query lifecycle automatically.
- **Background refetch** — after `staleTime` elapses, refetching on window focus / reconnect keeps data fresh without a spinner (because you already have stale data to show).
- **Race safety** — the library guarantees only the latest result for a `queryKey` is committed.

### Example 5 — Mutations + invalidation + optimistic update

Reads are half the story. Here's updating a user's name, then keeping the read in sync, with an **optimistic** UI that updates instantly and rolls back on failure.

```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function RenameUser({ userId }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: async (newName) => {
      const res = await fetch(`/api/users/${userId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: newName }),
      });
      if (!res.ok) throw new Error('Update failed');
      return res.json();
    },

    // 1. Run before the request — patch the cache optimistically.
    onMutate: async (newName) => {
      await queryClient.cancelQueries({ queryKey: ['user', userId] });
      const previous = queryClient.getQueryData(['user', userId]);
      queryClient.setQueryData(['user', userId], (old) => ({ ...old, name: newName }));
      return { previous }; // context handed to onError
    },

    // 2. If the server rejects, roll back to the snapshot.
    onError: (_err, _newName, context) => {
      queryClient.setQueryData(['user', userId], context.previous);
    },

    // 3. Whatever happens, re-sync with the server as source of truth.
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
  });

  return (
    <button
      disabled={mutation.isPending}
      onClick={() => mutation.mutate('Ada Lovelace')}
    >
      {mutation.isPending ? 'Saving…' : 'Rename'}
    </button>
  );
}
```

The `onMutate` → `onError` → `onSettled` triad is the canonical optimistic-update recipe: snapshot, apply, roll back on failure, reconcile on completion. React also ships a lighter local-only primitive, **`useOptimistic`**, for transient UI optimism (full coverage in `22-modern-react-19.md`).

### Example 6 — SWR, the lighter alternative

[SWR](https://swr.vercel.app/) (from Vercel) covers the read side with a smaller surface. The name is "stale-while-revalidate": serve cache, then refetch.

```jsx
import useSWR from 'swr';

const fetcher = (url) => fetch(url).then((r) => {
  if (!r.ok) throw new Error('HTTP ' + r.status);
  return r.json();
});

function UserCard({ userId }) {
  const { data, error, isLoading } = useSWR(`/api/users/${userId}`, fetcher);
  if (isLoading) return <p>Loading…</p>;
  if (error) return <p role="alert">Failed</p>;
  return <h2>{data.name}</h2>;
}
```

SWR is excellent for read-heavy apps and is tiny. TanStack Query has a richer mutation story, devtools, infinite queries, and finer cache control — pick it when writes and cache orchestration are central. Either beats hand-rolling.

### Example 7 — React 19 `use()` + Suspense + error boundary

`use()` reads a promise *during render*. The promise must come from a stable source (here, created outside the component or memoized) — recreating it every render would re-suspend forever.

```jsx
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary'; // see 20-advanced-patterns.md

// Created once, outside render, so its identity is stable.
const userPromise = fetch('/api/users/42').then((r) => {
  if (!r.ok) throw new Error('HTTP ' + r.status);
  return r.json();
});

function Profile() {
  const user = use(userPromise);   // suspends while pending, throws on reject
  return <h2>{user.name}</h2>;
}

export default function App() {
  return (
    <ErrorBoundary fallback={<p role="alert">Could not load user.</p>}>
      <Suspense fallback={<p>Loading…</p>}>
        <Profile />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Notice `Profile` has **no loading or error branches at all** — those concerns moved up to `<Suspense>` and `<ErrorBoundary>`. In real apps you don't create the promise in module scope like this; you let a framework (or TanStack Query's `useSuspenseQuery`) supply a cached, stable promise.

---

## Under the hood

**`fetch` and HTTP errors.** `fetch` only rejects on *network* failure (DNS, offline, CORS preflight). A `404` or `500` is a *successful* HTTP exchange from `fetch`'s point of view — `response.ok` is the boolean you must check. Forgetting this is the #1 silent fetching bug.

**Effects, StrictMode, and double-mount.** In development, React 19 StrictMode mounts each component twice to flush out missing cleanup (see `08-useeffect-side-effects.md`). Your fetch effect will run, abort, and run again. This is *why* the abort/ignore pattern is non-negotiable: without it you'd see duplicate requests and possible state churn in dev. It also faithfully simulates the real production scenario where a dependency changes mid-flight.

**Why the race condition exists at all.** State setters captured in a closure don't know about other effect runs. Every render creates a fresh effect closure with its own variables; React runs the *previous* closure's cleanup before the *next* closure's setup. The `active` flag and `AbortController` both exploit that ordering: cleanup invalidates the old run before the new one can finish. There is no global "current request" — you manufacture that guarantee per-run.

**How TanStack Query dedupes.** The cache is a `Map` keyed by a *hashed, stable serialization* of the `queryKey`. Mounting a second observer for an existing key subscribes to the same cache entry instead of firing a new `queryFn`. In-flight promises are themselves cached, so concurrent mounts share one promise. `staleTime` is a timestamp comparison: `lastUpdated + staleTime > now` ⇒ fresh ⇒ no refetch. `gcTime` starts a timer when the *last* observer unsubscribes; if nobody re-subscribes before it fires, the entry is evicted.

**How `use()` + Suspense actually pause.** When `use(promise)` sees a pending promise, it **throws the promise** (an internal mechanism, not a real error). React catches it at the nearest `<Suspense>`, renders the fallback, and attaches a `.then` to retry rendering the subtree when it resolves. A rejected promise is thrown as a real error to the nearest error boundary. This is why the promise identity must be stable: a new promise each render = a new throw each render = an infinite fallback. The library or framework is responsible for caching the promise; `use()` itself does not cache.

**Suspense is not a data-fetching library.** It's a *coordination* mechanism for "something isn't ready." It has no cache, no dedupe, no refetch. That's the whole reason TanStack Query / SWR / a framework loader still exist in a `use()` world — they own the data; Suspense only owns the *waiting*.

---

## Common mistakes & pitfalls

**1. Making the effect callback itself `async`.**

```jsx
// WRONG — an async function returns a Promise, but useEffect expects
// the return value to be a cleanup function (or undefined).
useEffect(async () => {
  const data = await fetch(url).then((r) => r.json());
  setData(data);
}, [url]);
```

```jsx
// RIGHT — define an async function inside and call it.
useEffect(() => {
  let active = true;
  (async () => {
    const data = await fetch(url).then((r) => r.json());
    if (active) setData(data);
  })();
  return () => { active = false; };
}, [url]);
```

React will try to call the returned Promise as a cleanup function and your cancellation is gone.

**2. Trusting `fetch` to reject on a 500.**

```jsx
// WRONG — a 500 lands in .then with an error body; you render garbage.
const data = await fetch(url).then((r) => r.json());
```

```jsx
// RIGHT — check res.ok and throw.
const res = await fetch(url);
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
```

**3. Forgetting the query param in the dependency array.**

```jsx
// WRONG — fetches the first userId once, never updates when the prop changes.
useEffect(() => { loadUser(userId); }, []); // eslint-disable... (don't!)
```

```jsx
// RIGHT — depend on what the effect reads.
useEffect(() => { loadUser(userId); }, [userId]);
```

The `exhaustive-deps` ESLint rule (see `08-useeffect-side-effects.md`) catches this; never silence it to hide the bug.

**4. An unstable `queryFn` identity vs an unstable `queryKey`.** The `queryKey` is what identifies the cache entry — interpolate every variable the request depends on:

```jsx
// WRONG — every user shares the cache entry ['user'] and shows stale data.
useQuery({ queryKey: ['user'], queryFn: () => fetchUser(userId) });
```

```jsx
// RIGHT — the id is part of the key.
useQuery({ queryKey: ['user', userId], queryFn: () => fetchUser(userId) });
```

**5. Creating the `use()` promise inside the component.**

```jsx
// WRONG — new promise every render → suspends forever.
function Profile({ userId }) {
  const user = use(fetch(`/api/users/${userId}`).then((r) => r.json()));
  return <h2>{user.name}</h2>;
}
```

```jsx
// RIGHT — get a stable, cached promise from a library / loader, or hoist it.
function Profile({ userPromise }) {
  const user = use(userPromise);
  return <h2>{user.name}</h2>;
}
```

**6. No error UI.** Shipping only `if (loading)` means the day the API 500s, users stare at a spinner forever. Always handle the error branch (or use an error boundary, see `20-advanced-patterns.md`).

---

## Best practices

- **Always model three states** — pending, success, error — never just `data`.
- **Cancel stale work** with `AbortController` *and* an `active` flag in manual effects; one guards the network, the other guards `setState`.
- **Check `response.ok`** and throw on non-2xx so errors flow through one path.
- **Don't hand-roll a cache.** The moment you need dedupe, refetch, or invalidation, adopt TanStack Query or SWR. It is not over-engineering — it's the standard.
- **Make `queryKey` a complete identity** — include every variable the request depends on.
- **Tune `staleTime`** to your data's volatility: high for reference data, low/zero for fast-changing data.
- **Invalidate after mutations** (`invalidateQueries`) so reads re-sync; reach for optimistic updates only where instant feedback matters.
- **Put loading/error concerns at boundaries** when using `use()` + Suspense; keep the component body focused on the happy path.
- **Never fetch in event handlers and then `setState` without guarding** for the component still being mounted — prefer mutations through the library.
- **Keep the `QueryClient` as a single app-level instance**; recreating it wipes the cache.

---

## Exercises

1. **Manual fetch with full state.** Build a `<Post postId={n} />` that fetches `https://jsonplaceholder.typicode.com/posts/{postId}`, handles loading/error/data, cancels on `postId` change, and renders the title and body.

2. **Beat the race.** Build a `<Search />` with a controlled `<input>`. Debounce is optional; the requirement is that rapidly typing then clearing the box must never leave stale results on screen. Prove it works by throttling your network in DevTools.

3. **Port to TanStack Query.** Reimplement Exercise 1 with `useQuery`. Render two `<Post postId={1} />` side by side and confirm in the Network tab that only **one** request fires.

4. **Mutation + invalidation.** Add a "toggle completed" button to a `<Todo id={n} />` (use `jsonplaceholder` `/todos/{id}`, `PATCH`). Use `useMutation` with optimistic update via `onMutate`/`onError`/`onSettled`. The checkbox must flip instantly and roll back if you simulate a failure.

5. **`use()` + Suspense.** Render a profile with `use(promise)` behind `<Suspense>` and an error boundary. Make the promise reject and confirm the error boundary catches it.

## Solutions

**Exercise 1**

```jsx
import { useState, useEffect } from 'react';

function Post({ postId }) {
  const [post, setPost] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);
    (async () => {
      try {
        const res = await fetch(
          `https://jsonplaceholder.typicode.com/posts/${postId}`,
          { signal: controller.signal }
        );
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        setPost(await res.json());
      } catch (err) {
        if (err.name !== 'AbortError') setError(err);
      } finally {
        if (!controller.signal.aborted) setLoading(false);
      }
    })();
    return () => controller.abort();
  }, [postId]);

  if (loading) return <p>Loading…</p>;
  if (error) return <p role="alert">{error.message}</p>;
  return <article><h2>{post.title}</h2><p>{post.body}</p></article>;
}
```

**Exercise 2**

```jsx
import { useState, useEffect } from 'react';

function Search() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!query) { setResults([]); return; }
    let active = true;
    const controller = new AbortController();
    fetch(`https://jsonplaceholder.typicode.com/posts?q=${encodeURIComponent(query)}`,
      { signal: controller.signal })
      .then((r) => r.json())
      .then((data) => { if (active) setResults(data.slice(0, 5)); })
      .catch((err) => { if (err.name !== 'AbortError') console.error(err); });
    return () => { active = false; controller.abort(); };
  }, [query]);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{results.map((r) => <li key={r.id}>{r.title}</li>)}</ul>
    </>
  );
}
```

**Exercise 3**

```jsx
import { useQuery } from '@tanstack/react-query';

function Post({ postId }) {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['post', postId],
    queryFn: async ({ signal }) => {
      const res = await fetch(
        `https://jsonplaceholder.typicode.com/posts/${postId}`, { signal });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    },
  });
  if (isPending) return <p>Loading…</p>;
  if (isError) return <p role="alert">{error.message}</p>;
  return <article><h2>{data.title}</h2><p>{data.body}</p></article>;
}
// <><Post postId={1} /><Post postId={1} /></>  → one network request total.
```

**Exercise 4**

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function Todo({ id }) {
  const qc = useQueryClient();
  const { data: todo } = useQuery({
    queryKey: ['todo', id],
    queryFn: async ({ signal }) => {
      const r = await fetch(`https://jsonplaceholder.typicode.com/todos/${id}`, { signal });
      if (!r.ok) throw new Error('HTTP ' + r.status);
      return r.json();
    },
  });

  const toggle = useMutation({
    mutationFn: async (completed) => {
      const r = await fetch(`https://jsonplaceholder.typicode.com/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed }),
      });
      if (!r.ok) throw new Error('Update failed');
      return r.json();
    },
    onMutate: async (completed) => {
      await qc.cancelQueries({ queryKey: ['todo', id] });
      const previous = qc.getQueryData(['todo', id]);
      qc.setQueryData(['todo', id], (old) => ({ ...old, completed }));
      return { previous };
    },
    onError: (_e, _v, ctx) => qc.setQueryData(['todo', id], ctx.previous),
    onSettled: () => qc.invalidateQueries({ queryKey: ['todo', id] }),
  });

  if (!todo) return <p>Loading…</p>;
  return (
    <label>
      <input
        type="checkbox"
        checked={todo.completed}
        disabled={toggle.isPending}
        onChange={(e) => toggle.mutate(e.target.checked)}
      />
      {todo.title}
    </label>
  );
}
```

**Exercise 5**

```jsx
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

const userPromise = fetch('https://jsonplaceholder.typicode.com/users/0')
  .then((r) => {
    if (!r.ok) throw new Error('User not found'); // /users/0 → 404, rejects
    return r.json();
  });

function Profile() {
  const user = use(userPromise);
  return <h2>{user.name}</h2>;
}

export default function App() {
  return (
    <ErrorBoundary fallback={<p role="alert">Could not load user.</p>}>
      <Suspense fallback={<p>Loading…</p>}>
        <Profile />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## Key takeaways

- Every fetch is **three states** — pending, success, error. Always model all three.
- The manual pattern lives in an effect: async helper inside, dependency on the query param, cleanup that calls `AbortController.abort()` and flips an `active` flag.
- **Race conditions** are the defining hazard: ensure only the latest request can write state. Cancellation guards the network; the ignore flag guards `setState`.
- `fetch` does **not** reject on HTTP errors — check `response.ok` yourself.
- Hand-rolled fetching collapses once you need **caching, dedupe, background refetch, or invalidation** — that's exactly the gap libraries fill.
- **TanStack Query** is the standard: `useQuery` for reads (with `queryKey`/`queryFn`/`staleTime`), `useMutation` + `invalidateQueries` for writes, and the `onMutate`/`onError`/`onSettled` triad for optimistic updates. **SWR** is the lighter read-focused alternative.
- React 19's **`use()`** reads a promise during render and **suspends** to the nearest `<Suspense>`; rejections throw to an error boundary. The promise must come from a **stable, cached source**.
- Suspense coordinates *waiting*; it is not a cache — that's why you still need a data library underneath it.

---

## Further reading

- React docs — [Fetching data (`use`)](https://react.dev/reference/react/use) and [`<Suspense>`](https://react.dev/reference/react/Suspense)
- React docs — [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) and [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects) (race-condition section)
- [TanStack Query v5 docs](https://tanstack.com/query/latest/docs/framework/react/overview) — Queries, Mutations, [Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates), Query Invalidation
- [SWR docs](https://swr.vercel.app/) — stale-while-revalidate, mutation, and Suspense mode
- [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary) — the standard error-boundary component used above
- MDN — [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) and [`fetch()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch)
- Sibling lessons: `08-useeffect-side-effects.md` (effects & cleanup), `20-advanced-patterns.md` (catching thrown errors), `22-modern-react-19.md` (`useOptimistic` & form actions)
