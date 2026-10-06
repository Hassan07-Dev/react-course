# 23 — Going Full-Stack: Next.js & React Frameworks

> **What you'll learn:** why react.dev now tells you to start with a *framework* instead of bare React, and how the most popular one — Next.js with its App Router — turns your component knowledge into a complete full-stack application with routing, server rendering, data fetching, and mutations, all in one codebase.

By the end of this lesson you will be able to:

- Explain **why react.dev recommends a framework** and what concrete problems frameworks solve (routing, bundling, data fetching, SSR/SSG, server actions).
- Build **file-based routes** with the Next.js App Router: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, nested layouts, route groups, dynamic `[id]` and catch-all `[...slug]` segments.
- Distinguish **Server Components (the default) from Client Components** (`"use client"`), know when to reach for each, and pass data correctly across the boundary.
- Fetch data in async Server Components and control **caching and revalidation** (`revalidate`, `cache: "no-store"`).
- Write **Route Handlers** (`app/api/.../route.ts`) for JSON APIs and **Server Actions** for mutations and forms.
- Choose between **SSR, SSG (`generateStaticParams`), ISR, streaming with Suspense, and CSR**, and add **SEO metadata**.
- Navigate with `next/link` and `useRouter`, optimize images and fonts at a high level, and **deploy to Vercel**.
- Name the leading alternatives — **React Router framework mode**, **TanStack Start**, **Remix** — and know roughly where each fits.

---

## Prerequisites

This is a capstone-style lesson. You should already be comfortable with:

- **Components, props, and JSX** (lessons 03–05) — everything here is still components.
- **`useState` / `useEffect`** (see `07-state-usestate.md`, `08-useeffect-side-effects.md`) — Client Components use these exactly as before.
- **Client-side routing concepts** (see `14-routing.md`) — the App Router is a *server-first* re-imagining of the same ideas; we contrast them directly.
- **Data fetching** (see `15-data-fetching.md`) — you know the three states (pending/success/error) and `Suspense`. Server Components change *where* fetching happens, not the underlying truth.
- **Forms and actions** (see `11-forms.md` and `22-modern-react-19.md`) — Server Actions are the server-side extension of the action/`useActionState` model.

You'll also want Node 18.18+ (Node 20+ recommended) installed. We use **TypeScript** throughout, which is the default Next.js experience.

---

## Concept

### Why a framework at all?

For years the React docs taught you to start with `create-react-app` or a bare Vite SPA: one `index.html`, one JS bundle, React mounts into a `<div id="root">`, everything renders in the browser. That works, but the moment a real product grows you discover React-the-library deliberately leaves a *huge* amount unsolved:

- **Routing** — React has no built-in router. You bolt one on (lesson 14).
- **Data fetching** — no opinion; you hand-roll effects or add TanStack Query (lesson 15).
- **Code-splitting / bundling** — you configure it yourself.
- **Server rendering (SSR/SSG)** — sending HTML on the first request for speed and SEO is genuinely hard to wire up by hand: you need a server that runs React, serializes state, and "hydrates" on the client.
- **Backend glue** — talking to a database, hiding secrets, mutating data. A pure SPA has nowhere to put server code.

Around 2023–2024 the React team concluded that beginners and teams alike kept reinventing these wheels, badly. So **react.dev's "Start a New React Project" guidance now points you at a framework** (Next.js, React Router, Expo, etc.) rather than wiring React up from scratch. The pitch is simple: a framework is *React plus all the production concerns already solved and integrated*. You still write components and hooks — you just get routing, bundling, SSR, and a server runtime for free, designed to work together.

The analogy: bare React is an **engine**. It's powerful and you *can* build a car around it, but you'll spend months fabricating the chassis, wiring, and dashboard. A framework is the **whole car** — same engine, but with the steering, brakes, and seats already bolted on and tested together. You drive instead of weld.

> Bare React still has its place — embedding a widget in an existing page, a design-system playground, a tiny SPA. But for a *new product*, the framework is the recommended default. The rest of this lesson uses **Next.js**, the most widely deployed React framework, because it best showcases every piece: routing, SSR/SSG, server data fetching, and mutations.

### The big mental shift: the server runs React too

The single idea that unlocks everything in this lesson: **with a framework, your React components can run on the server**, at request time or build time, *before* any JavaScript reaches the browser. The server renders your component tree to HTML, streams it to the user (fast first paint, great SEO), and only the *interactive* pieces get shipped as JavaScript and "hydrated."

Next.js's **App Router** pushes this to its logical conclusion with **React Server Components (RSC)**: components are server-side **by default**, and you opt *into* the browser with `"use client"`. This inverts the old SPA model where everything was client-side and you opted into SSR.

### File-based routing

Instead of declaring routes in JavaScript (`<Route path="/blog/:id">`), the App Router derives routes from the **folder structure** inside an `app/` directory. A folder is a URL segment; a special `page.tsx` file inside it makes that segment a navigable page. This is the same "URLs as pages" goal from lesson 14, expressed as files instead of config.

---

## Syntax & API

### Creating a project

```bash
npx create-next-app@latest my-app
# answer the prompts: TypeScript = Yes, App Router = Yes, ESLint = Yes
cd my-app
npm run dev          # http://localhost:3000
```

### The special files of the App Router

Everything lives under `app/`. These filenames are **reserved** and have specific meaning in each route folder:

| File | Role |
| --- | --- |
| `page.tsx` | The unique UI for a route. Makes the segment publicly routable. |
| `layout.tsx` | Shared UI wrapping a segment **and all its children**; persists across navigation, does not re-render. Receives `children`. |
| `loading.tsx` | Instant loading UI shown via `Suspense` while the segment's data/RSC payload streams in. |
| `error.tsx` | Error boundary for the segment (**must** be a Client Component). |
| `not-found.tsx` | UI rendered when `notFound()` is called or a route doesn't match. |
| `template.tsx` | Like `layout` but re-mounts on every navigation (fresh state). |
| `route.ts` | A **Route Handler** — an API endpoint, not a page. Cannot coexist with `page.tsx` in the same folder. |

### Dynamic & special folder names

| Folder | URL effect |
| --- | --- |
| `app/blog/page.tsx` | `/blog` |
| `app/blog/[slug]/page.tsx` | `/blog/:slug` — dynamic segment, value via `params` |
| `app/shop/[...slug]/page.tsx` | catch-all: `/shop/a/b/c` → `params.slug = ["a","b","c"]` |
| `app/shop/[[...slug]]/page.tsx` | optional catch-all: also matches `/shop` |
| `app/(marketing)/page.tsx` | **route group**: `(marketing)` organizes files but is **not** in the URL |
| `app/@modal/...` | parallel route slot |

### Server vs Client Component directives

```tsx
// Server Component (DEFAULT — no directive needed).
// Can be async, can touch the DB/secrets, ships ZERO JS to the browser.
export default async function Page() { /* ... */ }
```

```tsx
"use client"; // MUST be the first line of the file.
// Now this component (and the modules it imports) is a Client Component:
// it ships JS, can use useState/useEffect/onClick, browser-only APIs.
import { useState } from "react";
export default function Counter() { /* ... */ }
```

### Page & layout props (Next.js 15)

In Next.js 15, `params` and `searchParams` are **Promises** and must be awaited:

```tsx
// app/blog/[slug]/page.tsx
export default async function Page({
  params,
  searchParams,
}: {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}) {
  const { slug } = await params;
  const { sort } = await searchParams;
  // ...
}
```

### Data fetching & caching

```ts
// Static by default in many setups; opt into freshness explicitly:
await fetch(url, { cache: "no-store" });               // never cache (dynamic / SSR)
await fetch(url, { next: { revalidate: 60 } });        // ISR: re-fetch at most every 60s
await fetch(url, { next: { tags: ["posts"] } });       // tag for on-demand revalidation

// Route-segment config (top of a page/layout):
export const dynamic = "force-dynamic";  // always SSR
export const revalidate = 3600;          // ISR for the whole segment
```

### Navigation, metadata, mutations

```tsx
import Link from "next/link";                 // client navigation, prefetched
import { useRouter } from "next/navigation";  // imperative (Client Components only)
import { redirect, notFound } from "next/navigation";

import type { Metadata } from "next";
export const metadata: Metadata = { title: "Home" };          // static SEO
export async function generateMetadata(): Promise<Metadata> { /* dynamic SEO */ }

export async function generateStaticParams() { /* pre-render dynamic routes at build (SSG) */ }
```

---

## Examples

### Example 1 — A minimal route with a nested layout

```tsx
// app/layout.tsx — the ROOT layout (required). Wraps every page.
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: { default: "My Store", template: "%s · My Store" },
  description: "The best demo store on the internet.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <header><nav>{/* site nav */}</nav></header>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

```tsx
// app/page.tsx — the "/" route. A Server Component (no directive).
export default function HomePage() {
  return <h1>Welcome to the store</h1>;
}
```

The root `layout.tsx` is the only place you render `<html>` and `<body>`. Every `page.tsx` is slotted in as `children`. Layouts **persist** across navigation: moving from `/` to `/about` keeps the header mounted and only swaps the page.

### Example 2 — Fetching data in an async Server Component

Server Components can be `async` and `await` directly in the body — no `useEffect`, no loading state in your code. The `fetch` runs on the server, so the API key never touches the browser.

```tsx
// app/posts/page.tsx
type Post = { id: number; title: string };

export default async function PostsPage() {
  // Re-fetch at most once per minute (ISR). Runs on the server.
  const res = await fetch("https://api.example.com/posts", {
    next: { revalidate: 60 },
    headers: { Authorization: `Bearer ${process.env.API_KEY}` }, // secret stays server-side
  });
  if (!res.ok) throw new Error("Failed to load posts"); // caught by error.tsx
  const posts: Post[] = await res.json();

  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

Compare this to the manual effect-based fetch from `15-data-fetching.md`: there's no `useState`, no three-state juggling, no race condition to cancel. The framework handles streaming and the loading state is declared separately (next example).

### Example 3 — `loading.tsx`, `error.tsx`, and `not-found.tsx`

Drop these files next to `page.tsx` and Next.js wires them into Suspense/error boundaries automatically.

```tsx
// app/posts/loading.tsx — shown instantly while PostsPage streams.
export default function Loading() {
  return <p>Loading posts…</p>;
}
```

```tsx
// app/posts/error.tsx — MUST be a Client Component (it uses interactivity).
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <p>Something went wrong: {error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

```tsx
// app/not-found.tsx — rendered when notFound() is called.
export default function NotFound() {
  return <h2>404 — That page doesn't exist.</h2>;
}
```

### Example 4 — Dynamic segment + SSG with `generateStaticParams`

```tsx
// app/blog/[slug]/page.tsx
import { notFound } from "next/navigation";
import type { Metadata } from "next";

type Props = { params: Promise<{ slug: string }> };

// Pre-render these pages at BUILD time (SSG).
export async function generateStaticParams() {
  const posts: { slug: string }[] = await fetch("https://api.example.com/posts").then((r) =>
    r.json(),
  );
  return posts.map((p) => ({ slug: p.slug }));
}

// Per-page SEO, computed from the data.
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const post = await fetch(`https://api.example.com/posts/${slug}`).then((r) => r.json());
  return { title: post.title, description: post.excerpt };
}

export default async function BlogPostPage({ params }: Props) {
  const { slug } = await params;
  const res = await fetch(`https://api.example.com/posts/${slug}`);
  if (res.status === 404) notFound(); // renders the nearest not-found.tsx
  const post = await res.json();

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}
```

`generateStaticParams` makes this an **SSG** route: at `next build`, Next.js renders one static HTML file per slug. Add `export const revalidate = 600` to upgrade it to **ISR** (regenerate stale pages in the background). Omit `generateStaticParams` and read a cookie/header, and it becomes **SSR** (rendered per request).

### Example 5 — A Route Handler (REST/JSON API)

`route.ts` files turn a folder into an API endpoint. Export functions named after HTTP verbs.

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(req: NextRequest) {
  const limit = Number(req.nextUrl.searchParams.get("limit") ?? "10");
  const posts = await db.post.findMany({ take: limit });
  return NextResponse.json(posts);
}

export async function POST(req: NextRequest) {
  const body = await req.json();
  const created = await db.post.create({ data: body });
  return NextResponse.json(created, { status: 201 });
}
```

This is your escape hatch for webhooks, third-party integrations, or APIs consumed by mobile apps. For *your own UI's* mutations, prefer Server Actions (next example) — they're typed end-to-end and need no manual `fetch`.

### Example 6 — Server Actions: a mutation with a form

A **Server Action** is an async function marked `"use server"` that runs **only on the server** but can be called directly from client code or wired into a `<form action={...}>`. No API route, no client `fetch`.

```tsx
// app/posts/new/actions.ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";

export async function createPost(formData: FormData) {
  const title = String(formData.get("title") ?? "").trim();
  if (!title) return; // validate on the server — never trust the client

  await db.post.create({ data: { title } });
  revalidatePath("/posts"); // invalidate the cached list so it shows the new post
  redirect("/posts");
}
```

```tsx
// app/posts/new/page.tsx — a Server Component renders the form.
import { createPost } from "./actions";

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Post title" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

This form **works without JavaScript** (progressive enhancement): the browser POSTs to the action. With JS enabled, Next.js intercepts it for a smooth client experience. To add pending state and validation feedback, combine the action with `useActionState` and `useFormStatus` from a Client Component — see `22-modern-react-19.md` for the full pattern.

### Example 7 — Crossing the server/client boundary

Fetch on the server, then hand the data to an interactive Client Component as props. Server-rendered data flows *down*; interactivity stays isolated in the client island.

```tsx
// app/dashboard/page.tsx — Server Component: fetches, then delegates.
import LikeButton from "./LikeButton";

export default async function Dashboard() {
  const stats = await fetch("https://api.example.com/stats", { cache: "no-store" }).then((r) =>
    r.json(),
  );
  return (
    <section>
      <h1>Likes: {stats.likes}</h1>
      <LikeButton initialLikes={stats.likes} />
    </section>
  );
}
```

```tsx
// app/dashboard/LikeButton.tsx — Client Component: holds the interactivity.
"use client";
import { useState } from "react";

export default function LikeButton({ initialLikes }: { initialLikes: number }) {
  const [likes, setLikes] = useState(initialLikes);
  return <button onClick={() => setLikes((n) => n + 1)}>👍 {likes}</button>;
}
```

The key rule: **props passed from a Server Component to a Client Component must be serializable** (strings, numbers, plain objects, arrays, even Server Action functions — but *not* arbitrary class instances, Dates round-trip awkwardly, and you can't pass a closure that captures server-only state).

---

## Under the hood

### What a Server Component actually produces

A Server Component never runs in the browser. The server executes it and produces the **RSC payload** — a compact, streamable description of the rendered tree (not HTML, not JSON, a special serialized format). On the client, React reads that payload and reconciles it into the DOM. Crucially, the *code* for Server Components is never sent to the browser, which is why importing a 2 MB markdown parser in a Server Component adds **zero** to your bundle.

Client Components are different. The `"use client"` directive marks a **boundary**: that module and everything it imports gets bundled and shipped to the browser, then **hydrated** (React attaches event listeners to the already-rendered HTML). The directive is *transitive at the boundary*: you only write `"use client"` at the top of the entry file, not in every child — but every component reachable from it becomes client code.

A useful mental model: **the tree is server by default, and `"use client"` plants a flag.** Everything from that flag down (within that import graph) is a client island. You can render Server Components *inside* Client Components, but only by passing them as `children`/props — a Client Component can't `import` and render a Server Component directly, because by then it's too late to run on the server.

### Streaming and Suspense

Next.js renders the page top-down and **streams** HTML to the browser as it's ready. A `loading.tsx` is sugar for wrapping the segment in `<Suspense fallback={<Loading/>}>`. Slow data lower in the tree doesn't block the shell — the header and layout flush immediately, the slow part streams in when ready. You can place your own `<Suspense>` boundaries inside a page for finer-grained streaming. This is the same Suspense you met in `15-data-fetching.md`, now driven by the server.

### The caching layers (and why they confuse people)

Next.js historically had several caches: the **Request Memoization** (dedupes identical `fetch`es in one render), the **Data Cache** (persists `fetch` results across requests, controlled by `revalidate`/`tags`), the **Full Route Cache** (cached rendered routes), and the client-side **Router Cache**. The big gotcha: defaults have shifted between versions. In older App Router versions `fetch` was cached aggressively by default, surprising people with stale data; **Next.js 15 made `fetch` and GET Route Handlers uncached by default.** The expert habit is to **be explicit**: state `cache: "no-store"` or `next: { revalidate }` on every fetch rather than relying on the default, and use `revalidatePath`/`revalidateTag` after mutations.

### `params`/`searchParams` are async now

In Next.js 15 these became Promises (to enable more streaming). If you see code doing `params.slug` directly without `await`, it's pre-15 syntax and will warn or break. Always `await params`.

---

## Common mistakes & pitfalls

**1. Using a hook or `onClick` in a Server Component.**

```tsx
// ❌ WRONG — Server Component (no "use client"). Throws at build/runtime.
export default function Counter() {
  const [n, setN] = useState(0);          // hooks aren't allowed on the server
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

```tsx
// ✅ RIGHT — opt into the client.
"use client";
import { useState } from "react";
export default function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

*Why:* `useState`, `useEffect`, event handlers, and browser APIs only exist in the client runtime. The directive moves the component there.

**2. Importing a Server Component into a Client Component.**

```tsx
// ❌ WRONG
"use client";
import ServerOnlyTable from "./ServerOnlyTable"; // becomes client code, loses server powers
export default function Panel() {
  return <ServerOnlyTable />;
}
```

```tsx
// ✅ RIGHT — pass the Server Component through as children from a server parent.
// app/page.tsx (Server Component):
import Panel from "./Panel";
import ServerOnlyTable from "./ServerOnlyTable";
export default function Page() {
  return <Panel><ServerOnlyTable /></Panel>; // Panel is "use client", but children render on server
}
```

*Why:* a Client Component's imports are bundled for the browser. Passing server content as `children` lets the server render it and the client slot it in.

**3. Leaking secrets to the client.**

```tsx
// ❌ WRONG — this runs in the browser; the key is now public.
"use client";
const data = await fetch(url, { headers: { Authorization: process.env.API_KEY } });
```

```tsx
// ✅ RIGHT — fetch with the secret in a Server Component or Server Action.
export default async function Page() {
  const data = await fetch(url, { headers: { Authorization: `Bearer ${process.env.API_KEY}` } });
  // env vars without NEXT_PUBLIC_ are server-only; client code can't read them.
}
```

*Why:* only `NEXT_PUBLIC_`-prefixed env vars are exposed to the browser. Keep data access on the server.

**4. Forgetting to revalidate after a mutation.**

```tsx
// ❌ WRONG — DB is updated, but the cached list still shows old data.
"use server";
export async function createPost(fd: FormData) {
  await db.post.create({ data: { title: String(fd.get("title")) } });
}
```

```tsx
// ✅ RIGHT
"use server";
import { revalidatePath } from "next/cache";
export async function createPost(fd: FormData) {
  await db.post.create({ data: { title: String(fd.get("title")) } });
  revalidatePath("/posts"); // tell Next the cached /posts is stale
}
```

**5. Using `<a href>` for internal navigation.**

```tsx
<a href="/about">About</a>            {/* ❌ full page reload, loses SPA speed */}
<Link href="/about">About</Link>       {/* ✅ client navigation + automatic prefetch */}
```

**6. Reaching for `useEffect` to fetch initial data.** In the App Router, server-fetching in an async component is almost always better: no waterfall, no spinner flash, SEO-friendly. Save `useEffect` fetching for client-only, post-interaction data.

---

## Best practices

- **Default to Server Components.** Only add `"use client"` when you need state, effects, event handlers, or browser APIs. Push the boundary as far down the tree as possible to keep client bundles small.
- **Keep client islands leaf-shaped.** A tiny `LikeButton` is better than marking a whole page `"use client"`.
- **Be explicit about caching.** Always state `cache`/`revalidate` on `fetch`; don't rely on version-dependent defaults. After writes, `revalidatePath`/`revalidateTag`.
- **Fetch on the server; mutate with Server Actions.** Reserve Route Handlers for external consumers (webhooks, mobile, third parties).
- **Validate inside Server Actions.** The client is untrusted; re-check inputs and auth on the server even if the form looks validated.
- **Always `await params` and `searchParams`** (Next 15+).
- **Use `next/link` for internal links, `next/image` for images, `next/font` for fonts** — they give prefetching, lazy-loading, automatic sizing, and zero layout shift for free.
- **Co-locate special files** (`loading.tsx`, `error.tsx`) next to the `page.tsx` they protect; remember `error.tsx` must be a Client Component.
- **Read the data where you render it.** Request Memoization dedupes identical fetches, so fetching the same resource in a layout *and* a page is fine — don't contort your tree to "fetch once and prop-drill."
- **Don't** put secrets in `NEXT_PUBLIC_` vars or in Client Components.

---

## Exercises

Spin up a project first: `npx create-next-app@latest course-app` (TypeScript + App Router), then `npm run dev`.

**Exercise 1 — Routes & layout.** Create `/`, `/about`, and `/blog`. Add a root layout with a nav bar (using `next/link`) that appears on all three. Verify the nav persists (doesn't flash) when navigating.

**Exercise 2 — Server data fetching + loading.** Make `/blog` an async Server Component that fetches posts from `https://jsonplaceholder.typicode.com/posts` and lists their titles. Add a `loading.tsx`. Add `next: { revalidate: 30 }` to the fetch.

**Exercise 3 — Dynamic route + SSG + 404.** Add `/blog/[id]` that fetches a single post. Add `generateStaticParams` to pre-render the first 10 posts, `generateMetadata` for the title, and call `notFound()` when the post doesn't exist. Add a `not-found.tsx`.

**Exercise 4 — Server Action mutation.** Add `/blog/new` with a form wired to a Server Action that "creates" a post (just log it server-side or push to an in-memory array), then `revalidatePath("/blog")` and `redirect("/blog")`.

**Exercise 5 — Boundary discipline.** Add an interactive "Reading time estimator" to the post page: a Client Component with a slider (`useState`) that recomputes minutes from the post body. Keep the *page* a Server Component and pass the body text down as a prop.

### Solutions

```tsx
// Exercise 1 — app/layout.tsx
import Link from "next/link";
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <nav style={{ display: "flex", gap: 12 }}>
          <Link href="/">Home</Link>
          <Link href="/about">About</Link>
          <Link href="/blog">Blog</Link>
        </nav>
        <main>{children}</main>
      </body>
    </html>
  );
}
// app/page.tsx
export default function Home() { return <h1>Home</h1>; }
// app/about/page.tsx
export default function About() { return <h1>About</h1>; }
```

```tsx
// Exercise 2 — app/blog/page.tsx
type Post = { id: number; title: string };
export default async function Blog() {
  const posts: Post[] = await fetch("https://jsonplaceholder.typicode.com/posts", {
    next: { revalidate: 30 },
  }).then((r) => r.json());
  return (
    <ul>
      {posts.slice(0, 20).map((p) => (
        <li key={p.id}><a href={`/blog/${p.id}`}>{p.title}</a></li>
      ))}
    </ul>
  );
}
// app/blog/loading.tsx
export default function Loading() { return <p>Loading posts…</p>; }
```

```tsx
// Exercise 3 — app/blog/[id]/page.tsx
import { notFound } from "next/navigation";
import type { Metadata } from "next";
import ReadingTime from "./ReadingTime";
type Props = { params: Promise<{ id: string }> };

export async function generateStaticParams() {
  return Array.from({ length: 10 }, (_, i) => ({ id: String(i + 1) }));
}
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const post = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`).then((r) => r.json());
  return { title: post.title ?? "Post" };
}
export default async function PostPage({ params }: Props) {
  const { id } = await params;
  const res = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
  if (res.status === 404) notFound();
  const post = await res.json();
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <ReadingTime body={post.body} />
    </article>
  );
}
// app/blog/not-found.tsx
// export default function NotFound() { return <h2>Post not found</h2>; }
```

```tsx
// Exercise 4 — app/blog/new/actions.ts
"use server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
const drafts: string[] = [];
export async function createPost(formData: FormData) {
  const title = String(formData.get("title") ?? "").trim();
  if (!title) return;
  drafts.push(title);
  console.log("created:", title);
  revalidatePath("/blog");
  redirect("/blog");
}
// app/blog/new/page.tsx
import { createPost } from "./actions";
export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" required placeholder="Title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

```tsx
// Exercise 5 — app/blog/[id]/ReadingTime.tsx
"use client";
import { useState } from "react";
export default function ReadingTime({ body }: { body: string }) {
  const [wpm, setWpm] = useState(200);
  const words = body.trim().split(/\s+/).length;
  const minutes = Math.max(1, Math.round(words / wpm));
  return (
    <p>
      ~{minutes} min read
      <input type="range" min={100} max={400} value={wpm}
        onChange={(e) => setWpm(Number(e.target.value))} />
      ({wpm} wpm)
    </p>
  );
}
```

---

## Images, fonts & deployment (high level)

**Images** — `next/image` renders an optimized `<img>`: it lazy-loads, serves modern formats (WebP/AVIF), generates responsive `srcset`s, and reserves space to prevent layout shift. You pass explicit `width`/`height` (or `fill`):

```tsx
import Image from "next/image";
<Image src="/hero.png" alt="Hero" width={800} height={400} priority />;
```

**Fonts** — `next/font` self-hosts Google/local fonts at build time (no external request, no layout shift):

```tsx
import { Inter } from "next/font/google";
const inter = Inter({ subsets: ["latin"] });
// then: <body className={inter.className}>
```

**Deployment to Vercel** — Vercel is built by the Next.js team and is the path of least resistance: push your repo to GitHub/GitLab, import it at vercel.com, and every push deploys. Server Components, Route Handlers, ISR, and Server Actions all "just work" on Vercel's serverless/edge runtime. You can also self-host (`next build && next start` behind Node, or in Docker) or deploy static exports to any static host when your app is fully SSG (`output: "export"`).

---

## The alternatives (brief)

Next.js is the most popular, but not the only framework — and react.dev lists several:

- **React Router (framework mode)** — the router from lesson 14 now ships a full framework mode (the merge of the Remix runtime into React Router v7). You get file/config routes, loaders, actions, and SSR while staying close to the web-standard request/response model. Great if you already think in React Router terms.
- **Remix** — the project that pioneered nested routes, loaders, and form actions on web standards; its runtime now powers React Router framework mode. Philosophy: lean on the platform (Fetch, FormData, Web Streams).
- **TanStack Start** — a newer full-stack framework from the TanStack team (TanStack Router + Query lineage), emphasizing end-to-end type safety, type-safe routing, and server functions. Worth watching if you love TanStack Query's ergonomics.
- **Expo / React Native** — react.dev also recommends Expo for *native* apps; different target, same component model.

All of them embrace the same core shift this lesson is about: **React runs on the server, routing is first-class, and data fetching/mutations live next to your components.** Learn the model once and it transfers.

---

## Key takeaways

- **react.dev now recommends starting with a framework** because routing, bundling, SSR/SSG, data fetching, and a server runtime are production necessities React-the-library leaves unsolved.
- **The App Router is server-first:** components are **Server Components by default**, and `"use client"` opts a subtree into the browser. Keep client islands small.
- **Routing is file-based:** folders are URL segments, `page.tsx` makes a route, and `layout/loading/error/not-found.tsx` are special files. `[id]` is dynamic, `[...slug]` is catch-all, `(group)` organizes without affecting the URL.
- **Fetch data in async Server Components** — no `useEffect`, secrets stay server-side. Control freshness explicitly with `cache: "no-store"` and `next: { revalidate }`; revalidate after writes.
- **Route Handlers** (`route.ts`) build APIs; **Server Actions** (`"use server"`) handle mutations and forms with progressive enhancement and no manual `fetch`.
- **Rendering strategies are per-route:** SSG (`generateStaticParams`), ISR (`revalidate`), SSR (`force-dynamic`/dynamic data), streaming (Suspense/`loading.tsx`), and CSR (Client Components).
- **`next/link`, `next/image`, `next/font`, and the Metadata API** give navigation, image/font optimization, and SEO with almost no effort; **Vercel** is the zero-config deploy target.
- The same model powers **React Router framework mode, Remix, and TanStack Start** — the concepts transfer.

---

## Further reading

- React docs — **Start a New React Project**: <https://react.dev/learn/start-a-new-react-project>
- React docs — **Server Components**: <https://react.dev/reference/rsc/server-components>
- React docs — **`"use client"`** and **`"use server"`**: <https://react.dev/reference/rsc/use-client> · <https://react.dev/reference/rsc/use-server>
- Next.js — **App Router docs**: <https://nextjs.org/docs/app>
- Next.js — **Routing fundamentals & file conventions**: <https://nextjs.org/docs/app/building-your-application/routing>
- Next.js — **Data Fetching, Caching & Revalidating**: <https://nextjs.org/docs/app/building-your-application/data-fetching>
- Next.js — **Server Actions and Mutations**: <https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations>
- Next.js — **Route Handlers**: <https://nextjs.org/docs/app/building-your-application/routing/route-handlers>
- Next.js — **Metadata API**: <https://nextjs.org/docs/app/building-your-application/optimizing/metadata>
- Next.js — **Deploying**: <https://nextjs.org/docs/app/building-your-application/deploying>
- React Router (framework mode): <https://reactrouter.com/> · Remix: <https://remix.run/> · TanStack Start: <https://tanstack.com/start>

> **Next up:** with full-stack rendering under your belt, the remaining lessons return to deeper React internals and patterns — see the later lessons in this folder. For the client-side counterpart to this server-first model, revisit `14-routing.md` and `15-data-fetching.md` and notice how the *same* problems are solved on each side of the network.
