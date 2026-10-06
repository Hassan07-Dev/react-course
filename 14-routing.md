# 14 — Client-Side Routing with React Router

> **What you'll learn:** how to turn a single React app into a multi-page experience — with URLs, the back button, bookmarks, and deep links all working — using React Router, the de-facto standard router for React.

By the end of this lesson you will be able to:

- Explain *why* a Single Page Application (SPA) needs a client-side router and how it differs from traditional server navigation.
- Set up React Router in a Vite project and render routes with `BrowserRouter`, `Routes`, and `Route`.
- Navigate declaratively with `Link`/`NavLink` and imperatively with `useNavigate`.
- Build **nested routes**, **layout routes**, **index routes**, and an `Outlet`-based shell.
- Read **dynamic URL segments** with `useParams`, **query strings** with `useSearchParams`, and inspect the URL with `useLocation`.
- Use the **data router API** (`createBrowserRouter` + `RouterProvider`) with route **loaders**, **actions**, `useLoaderData`, and `errorElement`.
- Lazy-load route code, build a **protected/auth-guarded route**, and write a working multi-page app.

---

## Prerequisites

You should be comfortable with:

- **Components, props, and JSX** (lessons 03–05) — routes render components.
- **`useState`** (see `07-state-usestate.md`) and **`useEffect`** (see `08-useeffect-side-effects.md`) — used in auth and data examples.
- **Custom hooks / `useContext`** (see `12-context-api.md` and `10-custom-hooks.md`) — the auth-guard pattern uses context.
- **The Vite project setup** from `02-getting-started.md` — we install and run inside a Vite app.
- A loose familiarity with **`fetch` / promises**, since loaders return data.

This lesson covers **library mode** routing for client-rendered SPAs. File-based, server-first routing (Next.js App Router) is a different model — we cross-reference it but cover it fully in `23-nextjs-fullstack.md`.

---

## Concept

### The problem: one HTML page, many "pages"

A traditional website is many HTML documents. When you click a link, the browser throws away the current page, makes a full HTTP request, and the server returns a brand-new document. The screen flashes white, all JavaScript state is lost, and you start fresh.

A **React SPA** is the opposite: the server sends *one* HTML file and one JavaScript bundle. React mounts once and then **re-renders pieces of the DOM** in response to state changes. This is fast and lets you keep state (a shopping cart, a half-typed form) alive as the user moves around. But it creates three problems:

1. **The URL never changes.** If everything is `/`, you can't bookmark "the product page" or share a deep link.
2. **The back button breaks.** The browser's history stack doesn't know your app moved from a list to a detail view.
3. **You need a mapping** from "what's in the URL" to "what to render."

A **client-side router** solves all three. It is essentially a big `switch` statement on the URL: *"when the path is `/users/42`, render the `UserProfile` component."* It does this **without a network round-trip** by intercepting link clicks and using the browser's **History API** (`pushState`, `popstate`) to change the URL and react to back/forward — all in JavaScript.

### The mental model: URL as state

The core insight: **the URL is just another piece of application state**, but one that lives *outside* React, owned by the browser. The router's job is to keep React's rendered output **synchronized** with that external state in both directions:

- URL changes → router re-renders the matching component (forward sync).
- User clicks a `Link` or calls `navigate()` → router updates the URL via the History API (reverse sync).

Think of it like a TV: the URL is the channel number, components are the shows, and the router is the tuner that maps channels to shows and updates the on-screen channel indicator when you press a button. Because the URL is shareable and bookmarkable, anything you encode into it (the current tab, a search query, a page number) becomes shareable too.

### React Router versions

React Router is at **v7** (released late 2024), which merged the Remix framework into the router. It supports two modes:

- **Library / "declarative" mode** — what most existing apps use and what this lesson focuses on. The v6 API (`BrowserRouter`, `Routes`, `createBrowserRouter`, loaders) is **fully preserved** in v7; if you know v6 you know v7's library mode. Code in this lesson is written against `react-router-dom@6` syntax, which works identically in v7's library mode.
- **Framework mode** — a full-stack, file-based, SSR-capable framework (the old Remix). That overlaps conceptually with Next.js; we touch it only briefly here.

Install whichever is current; the APIs below are stable across v6 and v7 library mode.

---

## Syntax & API

```bash
npm install react-router-dom
```

### Routers (pick one per app)

| API | Notes |
| --- | --- |
| `<BrowserRouter>` | Wraps your app, uses the HTML5 History API (clean URLs like `/about`). The classic, JSX-route approach. |
| `<HashRouter>` | URLs look like `/#/about`. Use only when you can't configure the server to serve `index.html` for all routes (e.g. static GitHub Pages without rewrites). |
| `createBrowserRouter(routes)` + `<RouterProvider router={...} />` | The **data router**. Enables loaders, actions, `errorElement`, deferred data. Preferred for new apps. |

### Route definition

```jsx
<Routes>
  <Route path="/" element={<Layout />}>
    <Route index element={<Dashboard />} />   {/* default child for "/" */}
    <Route path="users/:id" element={<User />} />
  </Route>
  <Route path="*" element={<NotFound />} />     {/* splat 404 */}
</Routes>
```

- `path` — URL pattern. `:id` is a **dynamic segment**; `*` is the **splat** (catch-all).
- `element` — the React element to render when the path matches.
- `index` — marks the **index route**: the child rendered when the parent's path matches exactly.
- Child `<Route>`s nest; the parent must render an `<Outlet />` for children to appear.

### Navigation components

| API | Signature / props | Notes |
| --- | --- | --- |
| `<Link to="/path">` | `to` (string or partial path object), `replace`, `state`, `relative` | Renders an `<a>` that navigates without a reload. |
| `<NavLink to="/path">` | adds `className`/`style` as functions receiving `{ isActive, isPending }`, plus `end` | For nav bars; auto-applies active styling. |
| `<Navigate to="/path" replace />` | declarative redirect rendered as an element | Redirects on render — used in guards. |
| `<Outlet />` | renders the matched child route | Placed in layout components. |

### Hooks

| Hook | Returns | Use |
| --- | --- | --- |
| `useNavigate()` | `navigate(to, opts?)` function | Programmatic navigation: `navigate('/login')`, `navigate(-1)` (back), `navigate(to, { replace: true, state })`. |
| `useParams()` | object of dynamic segments | `const { id } = useParams()` for `path="users/:id"`. |
| `useLocation()` | `{ pathname, search, hash, state, key }` | Inspect current URL; read navigation `state`. |
| `useSearchParams()` | `[searchParams, setSearchParams]` | Read/write the query string; `searchParams` is a `URLSearchParams`. |
| `useLoaderData()` | data returned by the route's `loader` | Data router only. |
| `useRouteError()` | the thrown error | Inside an `errorElement`. |
| `useActionData()` | data returned by the route's `action` | Data router, after a form submit. |
| `useNavigation()` | `{ state, location, formData }` | Global pending UI (`'idle' | 'loading' | 'submitting'`). |

### Data-router route objects

```jsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <ErrorPage />,
    loader: rootLoader,          // runs before render, returns data
    action: rootAction,          // handles <Form> submissions
    children: [ /* nested route objects */ ],
    lazy: () => import("./routes/dashboard"), // code-split this route
  },
]);
```

---

## Examples

### Example 1 — Minimal multi-page app

A Vite app with three pages and a nav bar. This is the smallest complete router setup.

```jsx
// src/main.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

```jsx
// src/App.jsx
import { Routes, Route, Link } from "react-router-dom";

function Home() {
  return <h1>Home</h1>;
}
function About() {
  return <h1>About us</h1>;
}
function NotFound() {
  return <h1>404 — page not found</h1>;
}

export default function App() {
  return (
    <div>
      <nav>
        <Link to="/">Home</Link> | <Link to="/about">About</Link>
      </nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </div>
  );
}
```

Clicking the links swaps the heading with **no page reload**, the URL updates, and the back button works. Any unknown path (`/xyz`) hits the `path="*"` splat and renders the 404. `BrowserRouter` must wrap everything that uses router features.

> **Vite dev note:** Vite's dev server already serves `index.html` for unknown paths, so reloading `/about` works locally. In production you must configure your static host to do the same fallback, or deep links return a real 404.

### Example 2 — Layout routes, nested routes, and `NavLink`

Real apps share a chrome (header/sidebar) across pages. A **layout route** renders that chrome plus an `<Outlet />` where children appear.

```jsx
// src/App.jsx
import { Routes, Route, NavLink, Outlet } from "react-router-dom";

function Layout() {
  const linkClass = ({ isActive }) =>
    isActive ? "nav-link active" : "nav-link";
  return (
    <div className="shell">
      <header>
        <NavLink to="/" end className={linkClass}>Home</NavLink>
        <NavLink to="/products" className={linkClass}>Products</NavLink>
        <NavLink to="/about" className={linkClass}>About</NavLink>
      </header>
      <main>
        <Outlet /> {/* child route renders here */}
      </main>
    </div>
  );
}

function Home() { return <h2>Welcome</h2>; }
function About() { return <h2>About</h2>; }
function ProductList() {
  return (
    <ul>
      <li><NavLink to="/products/1">Widget</NavLink></li>
      <li><NavLink to="/products/2">Gadget</NavLink></li>
    </ul>
  );
}

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Layout />}>
        <Route index element={<Home />} />
        <Route path="products" element={<ProductList />} />
        <Route path="about" element={<About />} />
        <Route path="*" element={<h2>Not found</h2>} />
      </Route>
    </Routes>
  );
}
```

Key points:

- The `<Route path="/" element={<Layout />}>` has **no element of its own to show as content** — it shows the shell and delegates the body to its children via `<Outlet />`.
- `index` means "render `<Home />` when the URL is exactly `/`."
- `<NavLink>` applies the `active` class automatically. **`end`** on the Home link prevents it from staying active for every nested path (without `end`, `/` matches `/products` too because `/products` starts with `/`).
- Child paths are **relative** — `path="products"` resolves to `/products` because the parent is `/`.

### Example 3 — Dynamic segments, programmatic navigation, query strings

A product detail page reached by `/products/:id`, a "go back" button, and a search box that lives in the URL.

```jsx
// ProductDetail.jsx
import { useParams, useNavigate } from "react-router-dom";

export function ProductDetail() {
  const { id } = useParams();        // string from the URL
  const navigate = useNavigate();
  return (
    <article>
      <h2>Product #{id}</h2>
      <button onClick={() => navigate(-1)}>← Back</button>
      <button onClick={() => navigate("/cart")}>Buy</button>
      <button onClick={() => navigate("/checkout", { state: { from: id } })}>
        Buy &amp; checkout
      </button>
    </article>
  );
}
```

```jsx
// ProductSearch.jsx
import { useSearchParams } from "react-router-dom";

export function ProductSearch({ allProducts }) {
  const [searchParams, setSearchParams] = useSearchParams();
  const q = searchParams.get("q") ?? "";        // ?q=widget → "widget"
  const filtered = allProducts.filter((p) =>
    p.name.toLowerCase().includes(q.toLowerCase())
  );

  return (
    <div>
      <input
        value={q}
        placeholder="Search…"
        onChange={(e) => {
          const next = e.target.value;
          // Updating search params updates the URL → shareable & back-button friendly.
          setSearchParams(next ? { q: next } : {}, { replace: true });
        }}
      />
      <ul>
        {filtered.map((p) => <li key={p.id}>{p.name}</li>)}
      </ul>
    </div>
  );
}
```

- `useParams()` returns URL segments as **strings** (`id` is `"42"`, not `42`) — convert if you need a number.
- `navigate(-1)` mimics the browser back button; `navigate(n)` moves `n` steps in history. `navigate(to, { replace: true })` swaps the current entry instead of pushing (so back doesn't return to it).
- `navigate(to, { state })` attaches data readable via `useLocation().state` on the destination — handy for "where did the user come from."
- `useSearchParams` makes the query string reactive: the search box state *is* the URL. Use `{ replace: true }` while typing so you don't create a history entry per keystroke.

### Example 4 — Data router: loaders, `useLoaderData`, and `errorElement`

The `BrowserRouter` examples fetch data inside components with `useEffect`, which causes the "render → empty → effect → fetch → re-render" waterfall. The **data router** lets each route declare a **loader** that runs *before* the component renders, so data is ready on first paint.

```jsx
// src/main.jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { createBrowserRouter, RouterProvider } from "react-router-dom";
import Root from "./Root.jsx";
import UserPage, { userLoader } from "./UserPage.jsx";
import ErrorPage from "./ErrorPage.jsx";

const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <ErrorPage />,         // catches errors from this subtree
    children: [
      {
        path: "users/:id",
        element: <UserPage />,
        loader: userLoader,              // runs before <UserPage /> renders
      },
    ],
  },
]);

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>
);
```

```jsx
// src/UserPage.jsx
import { useLoaderData } from "react-router-dom";

// A loader receives { params, request }. Throw a Response for errorElement to catch.
export async function userLoader({ params }) {
  const res = await fetch(`https://api.example.com/users/${params.id}`);
  if (!res.ok) {
    throw new Response("User not found", { status: res.status });
  }
  return res.json();                     // becomes useLoaderData()
}

export default function UserPage() {
  const user = useLoaderData();          // data is guaranteed present here
  return (
    <section>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </section>
  );
}
```

```jsx
// src/ErrorPage.jsx
import { useRouteError, isRouteErrorResponse, Link } from "react-router-dom";

export default function ErrorPage() {
  const error = useRouteError();
  if (isRouteErrorResponse(error)) {
    return <p>{error.status} — {error.data}</p>;
  }
  return <p>Something went wrong: {error?.message ?? "unknown error"}</p>;
}
```

Because the loader runs *before* render, `UserPage` never has to handle a loading-or-empty state for its own data — the router shows the previous page until the loader resolves (use `useNavigation().state === "loading"` to render a global spinner). Thrown `Response`s and errors bubble to the nearest `errorElement` instead of crashing the app.

### Example 5 — Actions and `<Form>` (mutations the router way)

The data router also handles **writes**. A route `action` receives the submitted form data; `<Form>` posts to it without a reload, and the router automatically **revalidates loaders** afterward so the UI reflects the change.

```jsx
// NewContact.jsx
import { Form, redirect, useNavigation } from "react-router-dom";

export async function newContactAction({ request }) {
  const formData = await request.formData();
  const name = formData.get("name");
  const res = await fetch("/api/contacts", {
    method: "POST",
    body: JSON.stringify({ name }),
    headers: { "Content-Type": "application/json" },
  });
  const created = await res.json();
  return redirect(`/contacts/${created.id}`);   // navigate after success
}

export default function NewContact() {
  const navigation = useNavigation();
  const busy = navigation.state === "submitting";
  return (
    <Form method="post">
      <input name="name" required />
      <button disabled={busy}>{busy ? "Saving…" : "Save"}</button>
    </Form>
  );
}
```

Register it with `action: newContactAction` on the route object. `redirect()` is a helper that returns a `Response` the router understands as "navigate here." No `useState`, no manual `fetch` in an event handler, no manual re-fetch of the list — the router revalidates.

---

## Under the hood

**The History API is the engine.** `BrowserRouter` creates a `history` object wrapping `window.history`. When you click a `<Link>`, React Router calls `e.preventDefault()` (so the browser does *not* do a full navigation) and then `history.pushState(...)`, which changes `window.location` and the address bar **without a network request**. The router stores the current location in React state/context; changing it triggers a re-render of `<Routes>`, which re-runs its matching algorithm and swaps elements.

**Back/forward** fire the browser's `popstate` event. The router listens for it, reads the new URL, updates its location state, and re-renders. This is why the back button "just works" — the router is a subscriber to the browser's own history stack, not a replacement for it.

**Route matching is ranked, not first-match.** Unlike a naive top-to-bottom `switch`, React Router scores every route by **specificity** and picks the best match. Static segments beat dynamic (`:id`) segments, which beat splats (`*`). So `/users/new` correctly matches a literal `users/new` route even when `users/:id` is also defined — order in the config doesn't matter. This is a frequent source of "why did the wrong route render" confusion for people coming from order-dependent routers.

**Nested routes form a tree, rendered as nested `<Outlet>`s.** A URL like `/dashboard/settings/profile` matches a *chain* of routes, and each parent's `<Outlet />` renders the next child. The matched chain is what enables shared layouts, nested loaders running **in parallel**, and error boundaries scoped to a subtree.

**Data router lifecycle.** With `createBrowserRouter`, a navigation is a transaction: (1) match the new URL, (2) call all matched **loaders in parallel** (and actions on submit), (3) once data resolves, commit the new route and render with data already in place. During step 2 the *old* UI stays on screen and `useNavigation().state` is `"loading"`. This eliminates the cascading fetch-on-mount waterfalls you get with `useEffect`-based loading and is closer to how server frameworks work. If a loader throws, the router renders the nearest `errorElement` instead of the route.

**Why two APIs coexist.** `BrowserRouter`/`<Routes>` is the older, render-driven model where routes are part of your component tree. `createBrowserRouter` lifts routes into a **static config object** known *before* render, which is exactly what's required to run loaders/actions ahead of rendering. New code should prefer the data router; the JSX `<Routes>` form remains for simpler apps and existing codebases.

**Relationship to Next.js.** React Router does routing *in the browser at runtime*. **Next.js** (lesson `23-nextjs-fullstack.md`) does **file-based routing**: a file at `app/users/[id]/page.jsx` *is* the route `/users/:id`, resolved by the framework on the server with built-in SSR, streaming, and server components. React Router v7's framework mode blurs this line, but the conceptual split is: React Router = a routing *library* you wire up; Next.js = a *framework* where the file system is the route table.

---

## Common mistakes & pitfalls

**1. Using `<a href>` for internal links — causes a full reload.**

```jsx
// WRONG — browser does a full server navigation, app state is destroyed
<a href="/about">About</a>
```

```jsx
// RIGHT — client-side navigation, no reload
import { Link } from "react-router-dom";
<Link to="/about">About</Link>
```

Reserve `<a href>` for **external** links only.

**2. Forgetting `<Outlet />` in a layout route.**

```jsx
// WRONG — children never render; only the header shows
function Layout() {
  return <header>My App</header>;
}
```

```jsx
// RIGHT — the matched child renders where the Outlet is
import { Outlet } from "react-router-dom";
function Layout() {
  return (<><header>My App</header><main><Outlet /></main></>);
}
```

**3. Home `NavLink` stuck active everywhere — missing `end`.**

```jsx
// WRONG — "/" is a prefix of every path, so Home is always "active"
<NavLink to="/" className={({isActive}) => isActive ? "active" : ""}>Home</NavLink>
```

```jsx
// RIGHT — `end` requires an exact match
<NavLink to="/" end className={({isActive}) => isActive ? "active" : ""}>Home</NavLink>
```

**4. Treating params as numbers.**

```jsx
// WRONG — id is the string "42"; this comparison fails
const { id } = useParams();
if (id === 42) { /* never true */ }
```

```jsx
// RIGHT — convert explicitly
const { id } = useParams();
const numericId = Number(id);
if (numericId === 42) { /* ok */ }
```

**5. Calling router hooks outside a router.** `useNavigate`, `useParams`, etc. throw if the component isn't rendered inside a `<BrowserRouter>` / `<RouterProvider>`. In tests, wrap with `<MemoryRouter>`.

**6. Calling `navigate()` during render instead of in an effect/handler.**

```jsx
// WRONG — side effect during render
function Page() {
  const navigate = useNavigate();
  navigate("/login"); // 💥 navigating while rendering
  return null;
}
```

```jsx
// RIGHT — declarative redirect (preferred for guards)
import { Navigate } from "react-router-dom";
function Page() {
  return <Navigate to="/login" replace />;
}
```

**7. Deep links 404 in production.** Reloading `/users/42` hits your server, which has no such file. Configure the host to **fall back to `index.html`** for all unknown routes (Netlify `_redirects`, `vercel.json`, nginx `try_files`). Vite dev handles this automatically; production hosting does not by default.

---

## Best practices

- **Do** prefer the **data router** (`createBrowserRouter`) for new apps — loaders/actions remove fetch waterfalls and give you `errorElement` for free.
- **Do** use `<Link>`/`<NavLink>` for internal navigation; never `<a href>`.
- **Do** add a **splat 404 route** (`path="*"`) at the top level so unknown URLs render a friendly page, not a blank screen.
- **Do** put shared UI in **layout routes** with `<Outlet />` instead of repeating headers per page.
- **Do** encode shareable UI state (search query, active tab, page number) in `useSearchParams` so URLs are bookmarkable.
- **Do** lazy-load heavy routes (`route.lazy` or `React.lazy` + `Suspense`) to shrink the initial bundle.
- **Don't** store data in `navigate(..., { state })` that must survive a refresh — `location.state` is lost on hard reload. Use the URL or a store for durable state.
- **Don't** redirect by calling `navigate()` during render; use `<Navigate>` or do it inside an action/loader.
- **Don't** rely on route order for correctness — matching is by specificity, so make paths unambiguous.
- **Don't** forget the production `index.html` fallback for deep links.

---

## Exercises

### Exercise 1 — Basic three-page app (warm-up)
Create a Vite app with `/`, `/blog`, and `/contact` routes, a nav bar using `NavLink` with an `.active` style, and a `path="*"` 404 page. Make sure the Home link is only active on `/`.

### Exercise 2 — Nested routes with a layout and dynamic detail
Add a `/posts` section: a layout route renders a sidebar and `<Outlet />`. `/posts` shows a list of 3 hard-coded posts (`Link` to each). `/posts/:postId` shows the post's title from `useParams`. Add a "Back to posts" button using `useNavigate`.

### Exercise 3 — Search via the query string
On `/posts`, add a search input bound to `useSearchParams` (`?q=`). Filter the list as the user types, keep the value in the URL (use `replace: true`), and ensure refreshing the page preserves the search.

### Exercise 4 — Data router with a loader and error page
Convert the posts feature to `createBrowserRouter`. Give `/posts/:postId` a `loader` that fetches from `https://jsonplaceholder.typicode.com/posts/:postId`, render the title/body with `useLoaderData`, throw a `Response` for missing posts, and add an `errorElement`.

### Exercise 5 — Protected route (challenge)
Add a fake auth context (`{ user, login, logout }`). Build a `RequireAuth` wrapper that redirects unauthenticated users to `/login` with `<Navigate>`, preserving the attempted URL in `location.state` so login can send them back.

---

## Solutions

**Exercise 1**

```jsx
// App.jsx
import { Routes, Route, NavLink } from "react-router-dom";

const cls = ({ isActive }) => (isActive ? "active" : "");

export default function App() {
  return (
    <>
      <nav>
        <NavLink to="/" end className={cls}>Home</NavLink>
        <NavLink to="/blog" className={cls}>Blog</NavLink>
        <NavLink to="/contact" className={cls}>Contact</NavLink>
      </nav>
      <Routes>
        <Route path="/" element={<h1>Home</h1>} />
        <Route path="/blog" element={<h1>Blog</h1>} />
        <Route path="/contact" element={<h1>Contact</h1>} />
        <Route path="*" element={<h1>404</h1>} />
      </Routes>
    </>
  );
}
```

**Exercise 2**

```jsx
import { Routes, Route, Link, Outlet, useParams, useNavigate } from "react-router-dom";

const POSTS = [
  { id: "1", title: "Hello" },
  { id: "2", title: "Routing" },
  { id: "3", title: "Hooks" },
];

function PostsLayout() {
  return (
    <div style={{ display: "flex", gap: 16 }}>
      <aside>
        <h3>Posts</h3>
        {POSTS.map((p) => <div key={p.id}><Link to={p.id}>{p.title}</Link></div>)}
      </aside>
      <main><Outlet /></main>
    </div>
  );
}

function PostDetail() {
  const { postId } = useParams();
  const navigate = useNavigate();
  const post = POSTS.find((p) => p.id === postId);
  return (
    <>
      <h2>{post ? post.title : "Unknown post"}</h2>
      <button onClick={() => navigate("/posts")}>Back to posts</button>
    </>
  );
}

export default function App() {
  return (
    <Routes>
      <Route path="/posts" element={<PostsLayout />}>
        <Route index element={<p>Pick a post.</p>} />
        <Route path=":postId" element={<PostDetail />} />
      </Route>
    </Routes>
  );
}
```

**Exercise 3**

```jsx
import { useSearchParams } from "react-router-dom";

function PostList({ posts }) {
  const [params, setParams] = useSearchParams();
  const q = params.get("q") ?? "";
  const shown = posts.filter((p) =>
    p.title.toLowerCase().includes(q.toLowerCase())
  );
  return (
    <>
      <input
        value={q}
        onChange={(e) =>
          setParams(e.target.value ? { q: e.target.value } : {}, { replace: true })
        }
      />
      <ul>{shown.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
    </>
  );
}
```

**Exercise 4**

```jsx
// router.jsx
import { createBrowserRouter, useLoaderData, useRouteError, isRouteErrorResponse } from "react-router-dom";

async function postLoader({ params }) {
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${params.postId}`);
  if (!res.ok) throw new Response("Not found", { status: 404 });
  return res.json();
}

function Post() {
  const post = useLoaderData();
  return <article><h2>{post.title}</h2><p>{post.body}</p></article>;
}

function ErrorView() {
  const err = useRouteError();
  return <p>{isRouteErrorResponse(err) ? `${err.status} ${err.data}` : "Error"}</p>;
}

export const router = createBrowserRouter([
  { path: "/posts/:postId", element: <Post />, loader: postLoader, errorElement: <ErrorView /> },
]);
// In main.jsx: <RouterProvider router={router} />
```

**Exercise 5**

```jsx
import { createContext, useContext, useState } from "react";
import { Navigate, useLocation, useNavigate } from "react-router-dom";

const AuthContext = createContext(null);
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = { user, login: (u) => setUser(u), logout: () => setUser(null) };
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
export const useAuth = () => useContext(AuthContext);

export function RequireAuth({ children }) {
  const { user } = useAuth();
  const location = useLocation();
  if (!user) {
    // Remember where they wanted to go.
    return <Navigate to="/login" replace state={{ from: location }} />;
  }
  return children;
}

export function Login() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  const from = location.state?.from?.pathname ?? "/";
  return (
    <button onClick={() => { login({ name: "Ada" }); navigate(from, { replace: true }); }}>
      Log in
    </button>
  );
}

// Usage in routes:
// <Route path="/dashboard" element={<RequireAuth><Dashboard /></RequireAuth>} />
```

### Bonus: lazy-loaded routes

```jsx
// React.lazy + Suspense (works with <Routes>)
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";
const Dashboard = lazy(() => import("./Dashboard.jsx"));

<Routes>
  <Route path="/dashboard" element={
    <Suspense fallback={<p>Loading…</p>}><Dashboard /></Suspense>
  } />
</Routes>
```

```jsx
// route.lazy (data router) — code-splits the whole route module incl. its loader
createBrowserRouter([
  { path: "/dashboard", lazy: () => import("./routes/dashboard.jsx") },
]);
// dashboard.jsx must export named `Component` and optionally `loader`, `action`, `errorElement`.
```

---

## Key takeaways

- A **client-side router** maps the URL to components and keeps React in sync with the browser's history — giving an SPA real URLs, working back/forward, bookmarks, and deep links.
- **React Router** is the standard; v6 and v7 library mode share the same API. `BrowserRouter` + `<Routes>`/`<Route>` is the classic form; `createBrowserRouter` + `RouterProvider` is the modern **data router**.
- Navigate declaratively with **`Link`/`NavLink`** (auto active styling, use `end` for `/`) and imperatively with **`useNavigate`**; redirect with **`<Navigate>`**.
- **Nested + layout routes** share chrome via `<Outlet />`; **index routes** are the default child; **`path="*"`** is the catch-all 404.
- Read the URL with **`useParams`** (segments, always strings), **`useSearchParams`** (reactive query string), and **`useLocation`**.
- The **data router** runs **loaders/actions before render** (no fetch waterfall), exposes data via **`useLoaderData`**, handles writes with `<Form>`/`action`, and catches errors with **`errorElement`**.
- **Lazy-load** routes (`route.lazy` or `React.lazy`) and guard private pages with a **`RequireAuth`** + `<Navigate>` pattern that preserves the intended destination.
- Routing here is *runtime, in the browser*; **Next.js** does file-based, server-first routing — covered in `23-nextjs-fullstack.md`.

---

## Further reading

- React Router official docs — <https://reactrouter.com>
- Tutorial (data router, loaders, actions) — <https://reactrouter.com/start/data/installation>
- `NavLink` / `Link` API — <https://reactrouter.com/api/components/NavLink>
- Loaders & data loading — <https://reactrouter.com/start/data/data-loading>
- Lazy loading routes — <https://reactrouter.com/how-to/lazy-loading-route-modules>
- MDN — History API — <https://developer.mozilla.org/en-US/docs/Web/API/History_API>
- MDN — `URLSearchParams` — <https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams>
- React docs — `lazy` & `Suspense` — <https://react.dev/reference/react/lazy>
