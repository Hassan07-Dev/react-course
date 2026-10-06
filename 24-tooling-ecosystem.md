# 24 — Tooling, Build & the React Ecosystem

> **What you'll learn:** How a real React project is wired together — the package manager, the build tool (Vite), linting/formatting, debugging, production builds, deployment, and the handful of ecosystem libraries every serious React developer reaches for.

By the end of this lesson you will be able to:

- Choose and use a package manager (npm, yarn, pnpm) and read/write a `package.json` fluently, including semver ranges, scripts, and the `dependencies` vs `devDependencies` split.
- Run and configure **Vite**: dev server, HMR, the production build, `vite.config.js`, plugins, and environment variables via `import.meta.env`.
- Set up **ESLint** (flat config) with `eslint-plugin-react-hooks` and `react-refresh`, plus **Prettier**, and make them coexist.
- Configure **path aliases** for clean absolute imports, and understand the TypeScript tooling story.
- Debug React apps with browser DevTools, source maps, and the **React Developer Tools** extension (Components & Profiler tabs).
- Build for production, analyze your bundle size, and deploy a SPA to Vercel, Netlify, or GitHub Pages — including the all-important SPA fallback rewrite.
- Recognize and reach for the must-know ecosystem libraries (router, data fetching, state, forms, animation, i18n).

---

## Prerequisites

You should be comfortable with the material from earlier lessons:

- Function components, JSX, and props (lessons 01–05).
- Hooks — `useState`, `useEffect`, and the rules of hooks (see `08-useeffect-side-effects.md` and `09-hooks-deep-dive.md`).
- A basic mental model of how React renders and re-renders (see `05-rendering-lists-conditionals.md`).

You do **not** need to know how to deploy or configure builds yet — that's exactly what this lesson teaches. You'll also want Node.js (v18 or newer; v20+ recommended) installed, which gives you `npm` for free.

---

## Concept

Writing a React component is one thing. Shipping a React *application* is another. Between your `.jsx` source and a user's browser sits a whole **toolchain**:

1. The browser cannot run JSX, and historically could not run bare ES module imports efficiently for hundreds of files. Something has to **transform** JSX into `React.createElement`-style calls (or the modern JSX runtime) and **bundle** your modules.
2. During development you want **instant feedback** — change a file, see it update without losing component state. That's **Hot Module Replacement (HMR)**.
3. For production you want the opposite of "instant": you want **small, optimized, cacheable** files. That's the **build**.
4. You want guardrails — a **linter** to catch the bug-prone patterns React is famous for (stale closures, missing effect dependencies) and a **formatter** so nobody argues about semicolons.
5. Finally you need to put the result somewhere users can reach it — a **deploy** to a static host or CDN.

Think of the toolchain as a **kitchen**. Your components are recipes. Vite is the stove and the prep station — fast when you're cooking (dev), and it also plates everything beautifully for service (build). npm is the pantry that stocks ingredients. ESLint and Prettier are the health inspector and the plating standard. The browser DevTools are the diner's-eye view when something tastes off. The ecosystem libraries are the pre-made sauces you don't want to reduce from scratch every time.

The key insight for an expert: **the toolchain is mostly invisible when it works, and you should keep it that way.** Use the conventions, understand them deeply enough to debug them, and don't hand-roll a webpack config in 2026 unless you have a very specific reason.

---

## Syntax & API

### `package.json` essentials

```json
{
  "name": "my-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier --write ."
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "eslint": "^9.0.0",
    "vite": "^6.0.0"
  }
}
```

- **`dependencies`** ship to the browser / are needed at runtime (React itself, a router, a state library).
- **`devDependencies`** are only needed to build, lint, or test (Vite, ESLint, Prettier, `@vitejs/plugin-react`). They are *not* installed when someone runs `npm install --production`, and they don't bloat your bundle — Vite only bundles what you actually `import`.
- **`"type": "module"`** tells Node your `.js` files are ES modules (so `vite.config.js` can use `import`/`export`).
- **`scripts`** are run with `npm run <name>`. A few names have built-in shortcuts that work without `run` — `npm start`, `npm test` (alias `npm t`), `npm stop`, `npm restart`. Custom names like `dev` and `build` always need `npm run` (`npm dev` errors).

### Semver ranges

Versions are `MAJOR.MINOR.PATCH`. The range prefix controls what `npm install` will accept on updates:

| Range | Meaning | Example matches |
| --- | --- | --- |
| `^19.0.0` | Compatible with major 19 — allows minor & patch | `19.4.2`, not `20.0.0` |
| `~19.0.0` | Allows patch only | `19.0.7`, not `19.1.0` |
| `19.0.0` | Exact pin | only `19.0.0` |
| `*` / `latest` | Anything (avoid) | any version |

The **lockfile** (`package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml`) records the *exact* resolved versions so installs are reproducible. **Commit the lockfile.**

### Package manager commands (cheat sheet)

```bash
# npm
npm install                 # install everything from package.json
npm install react-router    # add a runtime dependency
npm install -D eslint       # add a dev dependency
npm uninstall lodash
npm run dev                 # run a script

# yarn (classic / berry)
yarn                        # install
yarn add react-router
yarn add -D eslint
yarn dev

# pnpm (fast, disk-efficient via a content-addressable store)
pnpm install
pnpm add react-router
pnpm add -D eslint
pnpm dev
```

`pnpm` symlinks from a global store so the same package version isn't copied per project — fast and space-saving. All three read the same `package.json`; pick one per project and commit *its* lockfile only.

### Vite config

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: { port: 5173, open: true },
  build: { sourcemap: true, outDir: 'dist' },
});
```

### Env vars

```js
import.meta.env.MODE         // 'development' | 'production'
import.meta.env.DEV          // boolean
import.meta.env.PROD         // boolean
import.meta.env.VITE_API_URL // only VITE_-prefixed vars are exposed to client code
```

---

## Examples

### Example 1 — Scaffold and run a project

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

`npm create vite` downloads the scaffolder, asks (or accepts via flags) for a framework + variant, and lays down a minimal project. `--template react` gives you JavaScript + JSX; use `react-ts` for TypeScript. After `npm install`, `npm run dev` starts the Vite dev server (default `http://localhost:5173`) with HMR enabled. The generated structure:

```
my-app/
├─ index.html          # the real entry point — note the <script type="module" src="/src/main.jsx">
├─ package.json
├─ vite.config.js
├─ public/             # static assets copied as-is to the build root
└─ src/
   ├─ main.jsx         # createRoot + render
   ├─ App.jsx
   └─ assets/
```

```jsx
// src/main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';
import './index.css';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Note that **`index.html` is the entry**, not a JS file — Vite treats HTML as a first-class module graph root. The `<div id="root">` inside it is where React mounts.

### Example 2 — Environment variables done right

Create env files at the project root. Vite loads them by mode.

```bash
# .env                 -> loaded always
VITE_API_URL=https://api.example.com

# .env.development     -> only in `npm run dev`
VITE_API_URL=http://localhost:4000

# .env.production      -> only in `npm run build`
VITE_API_URL=https://api.example.com
```

```jsx
// src/api.js
const BASE = import.meta.env.VITE_API_URL;

export async function getUsers() {
  const res = await fetch(`${BASE}/users`);
  if (!res.ok) throw new Error('Failed to load users');
  return res.json();
}
```

**Critical security point:** anything prefixed `VITE_` is **inlined into the client bundle as plaintext** at build time. Any user can open DevTools and read it. **Never** put secrets (API private keys, DB credentials, signing secrets) in a `VITE_` variable. Those belong on a server / serverless function. A variable named `API_SECRET` (no `VITE_` prefix) is deliberately *not* exposed to client code — Vite ignores it for `import.meta.env`. Add `.env*` files to `.gitignore` (commit a `.env.example` with empty values instead).

### Example 3 — A useful `vite.config.js` with aliases, proxy, and chunking

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { fileURLToPath, URL } from 'node:url';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  server: {
    port: 3000,
    proxy: {
      // Forward /api during dev to your backend, avoiding CORS pain
      '/api': { target: 'http://localhost:4000', changeOrigin: true },
    },
  },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom'],
        },
      },
    },
  },
});
```

Now imports can be absolute: `import Button from '@/components/Button'` instead of `'../../../components/Button'`. The `proxy` lets the browser hit `/api/users` in dev and Vite forwards it, sidestepping CORS. `manualChunks` splits React into its own long-term-cacheable chunk so app changes don't bust the vendor cache.

For editors and (non-TS) jsconfig-aware tooling to resolve the alias, add:

```json
// jsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

### Example 4 — Conditional code by environment

```jsx
// src/components/DebugBanner.jsx
export default function DebugBanner() {
  // import.meta.env.DEV is statically replaced; in production the whole
  // block is tree-shaken away by the bundler.
  if (!import.meta.env.DEV) return null;
  return <div style={{ background: '#fee', padding: 8 }}>DEV MODE</div>;
}
```

Because `import.meta.env.DEV` is a compile-time constant, the production build replaces it with `false`, and dead-code elimination removes the branch entirely. This is the idiomatic way to ship dev-only diagnostics with zero production cost.

---

## Under the hood

### Why Vite is fast in dev

Older bundlers (webpack-style) built the *entire* dependency graph into a bundle before serving the first byte — slow as projects grew. Vite splits the world in two:

- **Dependencies** (rarely change): pre-bundled once with **esbuild** (written in Go, 10–100× faster than JS-based bundlers) into a few optimized files cached in `node_modules/.vite`.
- **Your source code** (changes constantly): served **on demand as native ES modules**. When the browser requests `/src/App.jsx`, Vite transforms *just that file* (JSX → JS) and returns it. No full bundle. Modern browsers' native `import` handles the graph. This is why dev startup is near-instant regardless of project size.

### How HMR preserves state

When you save a component, Vite sends an HMR update over a WebSocket. The `@vitejs/plugin-react` plugin uses **React Refresh** (a.k.a. Fast Refresh): it swaps the new version of the component's function into the running tree **without unmounting it**, so `useState` values survive. The catch: Fast Refresh only works cleanly when a module exports **only** React components. Mixing a component and a non-component export in one file can force a full reload — that's exactly what the `react-refresh` ESLint rule warns about (see below).

### What the production build actually does

`vite build` runs **Rollup** to:

1. Tree-shake (drop unused exports).
2. Minify JS (esbuild) and CSS.
3. Code-split: each dynamic `import()` becomes its own chunk for lazy loading (pairs with `React.lazy` / `Suspense`).
4. Hash filenames (`index-a1b2c3.js`) for cache-busting, and rewrite references in `index.html`.
5. Emit everything to `dist/`.

The output is **static files** — HTML, JS, CSS, assets. That's why React SPAs deploy to any static host or CDN.

### Source maps

A source map (`.map` file or inline) maps minified production code back to your original `.jsx` lines, so a stack trace points at `App.jsx:42` instead of `index-a1b2c3.js:1:88421`. `build.sourcemap: true` emits them. They make debugging production issues possible — but they also expose your source, so some teams ship them to an error-tracking service (Sentry) rather than to the public CDN.

### The mental model to keep

> **Dev = native ESM + esbuild, optimized for speed of iteration. Build = Rollup bundle, optimized for the bytes users download.** They behave slightly differently; always smoke-test a `vite preview` of the real build before deploying.

---

## Common mistakes & pitfalls

**1. Putting a secret in a `VITE_` variable.**

```jsx
// ❌ WRONG — this key is now in every user's downloaded JS bundle
const stripe = import.meta.env.VITE_STRIPE_SECRET_KEY;
```

```jsx
// ✅ RIGHT — only the publishable key is client-safe; the secret stays server-side
const stripe = import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY;
// secret key lives in a server route / serverless function, never the client
```

**2. Refreshing a deep route on a static host → 404.**

```
# ❌ Visiting https://site.com/dashboard directly 404s, because there is no
#    /dashboard/index.html file — the router only exists in JS that never loaded.
```

```
# ✅ Configure the host to rewrite ALL paths to /index.html (the SPA fallback),
#    so React Router can take over client-side. See the deployment section below.
```

**3. Committing `node_modules` or skipping the lockfile.**

```bash
# ❌ node_modules is huge, OS-specific, and rebuildable — never commit it.
#    Forgetting the lockfile means teammates get different versions.
```

```gitignore
# ✅ .gitignore
node_modules
dist
.env
.env.*
!.env.example
```
Do **commit** `package-lock.json` / `pnpm-lock.yaml`.

**4. Importing assets by raw path instead of through the module graph.**

```jsx
// ❌ Path is relative to nothing predictable; breaks after the hashing build.
<img src="/src/assets/logo.png" />
```

```jsx
// ✅ Import it — Vite fingerprints the file and gives you the final URL.
import logo from '@/assets/logo.png';
<img src={logo} />
```

**5. Disabling the hooks lint rule to "make the warning go away."**

```jsx
// ❌ Silencing the dependency warning hides a real stale-closure bug.
useEffect(() => { doThing(count); }, []); // eslint-disable-line
```

```jsx
// ✅ List the real dependencies (or restructure). See 08-useeffect-side-effects.md.
useEffect(() => { doThing(count); }, [count]);
```

**6. Forgetting to set `base` for GitHub Pages subpaths.**

A project served from `https://user.github.io/my-repo/` needs `base: '/my-repo/'` in `vite.config.js`, otherwise every asset 404s because it's requested from the domain root.

---

## Best practices

- **Do** pick one package manager per project and commit its lockfile; **don't** mix npm and yarn in the same repo.
- **Do** use `^` ranges for libraries you trust to follow semver, but rely on the lockfile for reproducibility.
- **Do** keep build-only tools in `devDependencies`; **don't** dump everything into `dependencies`.
- **Do** prefix client-readable config with `VITE_`; **never** expose secrets to the client — they belong on a server.
- **Do** run ESLint and Prettier in CI and a pre-commit hook; **don't** rely on people remembering to run them.
- **Do** set up a `@` path alias early — refactors get painless.
- **Do** smoke-test `npm run build && npm run preview` before deploying; the production build can surface issues dev never showed.
- **Do** configure the SPA fallback rewrite on your host **before** you first deploy a router-based app.
- **Don't** reach for Redux on day one; start with `useState`/`useContext` (see `12-context-api.md`) and add a library only when you feel the pain.
- **Do** enable source maps (at least for your error tracker) so production stack traces are legible.
- **Don't** disable `eslint-plugin-react-hooks` rules to mute warnings — fix the underlying issue.

### A solid ESLint flat config + Prettier setup

ESLint 9 uses **flat config** (`eslint.config.js`), not the old `.eslintrc`. A modern React setup:

```bash
npm install -D eslint @eslint/js eslint-plugin-react-hooks \
  eslint-plugin-react-refresh globals prettier
```

```js
// eslint.config.js
import js from '@eslint/js';
import globals from 'globals';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';

export default [
  { ignores: ['dist'] },
  {
    files: ['**/*.{js,jsx}'],
    languageOptions: {
      ecmaVersion: 'latest',
      globals: globals.browser,
      parserOptions: { ecmaFeatures: { jsx: true }, sourceType: 'module' },
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...js.configs.recommended.rules,
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': ['warn', { allowConstantExport: true }],
    },
  },
];
```

- **`eslint-plugin-react-hooks`** enforces the Rules of Hooks (`rules-of-hooks`) and exhaustive effect dependencies (`exhaustive-deps`) — the two rules that prevent the most common React bugs.
- **`eslint-plugin-react-refresh`** warns when a file exports non-components alongside components, which would break Fast Refresh.

Keep the two tools in their lanes: **ESLint for code-quality rules, Prettier for formatting.** Don't run formatting rules through ESLint (the old `eslint-plugin-prettier` approach is now discouraged). Just run them separately:

```json
// .prettierrc.json
{ "singleQuote": true, "semi": true, "printWidth": 100 }
```

```jsonc
// package.json scripts
"lint": "eslint .",
"format": "prettier --write \"src/**/*.{js,jsx,css}\""
```

### TypeScript tooling

For a typed project, scaffold with `--template react-ts`. You get `tsconfig.json`, `.tsx` files, and `vite-env.d.ts`. Vite uses **esbuild to strip types** during dev/build (fast, but it does **not** type-check). So you run the compiler separately for checking only:

```json
// package.json
"scripts": {
  "build": "tsc -b && vite build",
  "typecheck": "tsc --noEmit"
}
```

Type your env vars for autocomplete and safety:

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## Debugging & analyzing

### React Developer Tools

Install the **React Developer Tools** browser extension (Chrome/Firefox/Edge). It adds two tabs to DevTools:

- **Components** — inspect the live component tree, see each component's props, state, and hooks; edit them live; jump to the source. The "Highlight updates when components render" setting visually flashes components that re-render — gold for spotting wasted renders (pairs with `13-performance.md`).
- **Profiler** — record an interaction and see a flame graph of which components rendered, how long each took, and *why* they rendered. This is how you find performance problems empirically instead of guessing.

### Breakpoints & source maps

In the browser **Sources** panel, open your `.jsx` file (visible thanks to source maps) and click a line number to set a breakpoint. Or drop `debugger;` in code. You can also use **conditional breakpoints** (right-click a line number) to break only when, say, `id === 42`. `console.log` is fine, but `console.table(arrayOfObjects)` and `console.group` are underused and much nicer for structured data.

### Analyzing bundle size

```bash
npm run build
```

Vite prints a per-file size table with gzip estimates. For a visual breakdown:

```bash
npm install -D rollup-plugin-visualizer
```

```js
// vite.config.js (excerpt)
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true, gzipSize: true }), // opens a treemap after build
  ],
});
```

The treemap shows which dependency eats the most bytes. Common wins: import only what you use (`import debounce from 'lodash/debounce'` instead of all of `lodash`), lazy-load heavy routes with `React.lazy`, and watch out for accidentally bundling a date or charting library twice.

---

## Building & deploying

### The build

```bash
npm run build      # -> dist/  (static files)
npm run preview     # serve dist/ locally to verify the real build
```

### SPA fallback — the one rewrite you must not forget

A client-rendered React app has exactly one HTML file (`index.html`). React Router builds the *appearance* of many URLs in JavaScript. So `https://site.com/settings` only works if the host returns `index.html` for **any** path and lets the JS router resolve it. Each host configures this differently.

**Vercel** — zero config for Vite SPAs in most cases; to be explicit:

```json
// vercel.json
{ "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }
```

**Netlify** — a `_redirects` file in `public/` (so it lands in `dist/`):

```
/*    /index.html   200
```

Netlify build settings: build command `npm run build`, publish directory `dist`.

**GitHub Pages** — served from a subpath, and it has no rewrite engine, so people use a `404.html` trick. First set the base path:

```js
// vite.config.js
export default defineConfig({ base: '/my-repo/' });
```

Then deploy `dist/` to the `gh-pages` branch (the `gh-pages` npm package automates this):

```bash
npm install -D gh-pages
```
```json
// package.json
"scripts": { "deploy": "vite build && gh-pages -d dist" },
"homepage": "https://yourname.github.io/my-repo"
```
Because Pages has no SPA rewrite, also copy `index.html` to `404.html` in the published output (some setups use `HashRouter` to dodge the problem entirely — URLs look like `/#/settings`).

For Vercel and Netlify, the typical flow is to connect the Git repo; every push to `main` triggers a build (`npm run build`) and publishes `dist/` to their global CDN automatically. Set your `VITE_*` env vars in the host's dashboard — they're injected at build time.

---

## A short tour of must-know ecosystem libraries

React is intentionally minimal — it's a view library, not a framework. These are the libraries you'll meet on nearly every real project. Install with `npm install <name>`.

- **react-router (`react-router-dom`)** — client-side routing: map URLs to components, nested layouts, data loaders, navigation. The default choice for SPAs. Covered in depth in a later lesson.
- **@tanstack/react-query** — server-state management: caching, background refetching, deduping, retries, and stale-while-revalidate for data fetched over the network. Replaces a mountain of `useEffect` + `useState` fetch boilerplate. Use it for *server* state; use React state/`useState` for *UI* state.
- **Redux Toolkit (`@reduxjs/toolkit` + `react-redux`)** — the modern, official way to use Redux for complex *global client* state. The Toolkit removes the historical boilerplate (action types, immutable reducers via Immer, `createSlice`). Reach for it on large apps with lots of cross-cutting shared state and a need for time-travel debugging.
- **zustand** — a tiny, hook-first global state store. Far less ceremony than Redux: `const useStore = create((set) => ({ count: 0, inc: () => set(s => ({ count: s.count + 1 })) }))`. Great default when Context + reducer starts to creak but Redux feels heavy.
- **react-hook-form** — performant forms with minimal re-renders (uncontrolled inputs + refs under the hood), plus validation and easy integration with schema validators like Zod. The go-to once forms get non-trivial (see `11-forms.md` for the controlled-input fundamentals it builds on).
- **framer-motion (now `motion`)** — declarative animations and gestures: `<motion.div animate={{ opacity: 1 }} />`, layout animations, enter/exit transitions via `AnimatePresence`. The most ergonomic animation library in the ecosystem.
- **react-i18next** — internationalization: translation catalogs, pluralization, interpolation, language switching, lazy-loaded locale bundles. The standard for multi-language apps.

A pragmatic adoption order for a new app: start with **react-router** + **react-query** (those two cover routing and data for most apps), add **react-hook-form** when forms appear, reach for **zustand** or **Redux Toolkit** only when shared client state genuinely demands it, and layer in **framer-motion** / **react-i18next** as polish and reach require.

---

## Exercises

1. **Scaffold & inspect.** Create a fresh Vite React app, run it, then open `package.json`. Move any one dependency you believe is build-only into `devDependencies` if it's misplaced, delete `node_modules` and the lockfile, reinstall, and confirm the app still runs.
2. **Env vars & security.** Add a `VITE_APP_NAME` variable and render it in the header. Then add a *non*-prefixed `SECRET_TOKEN` and prove (via `console.log(import.meta.env)`) that it does **not** appear client-side. Explain in a comment why.
3. **Aliases.** Add a `@` alias for `src/`, create `src/components/Card.jsx`, and import it from `App.jsx` as `@/components/Card`. Add the matching `jsconfig.json` so your editor resolves it.
4. **Lint a real bug.** Set up the ESLint flat config above. Write a `useEffect` that reads a state variable but has an empty dependency array, and watch `exhaustive-deps` flag it. Fix it correctly.
5. **Build, analyze, deploy.** Run a production build, add `rollup-plugin-visualizer`, and identify your largest dependency. Then deploy the `dist/` output to Netlify (or Vercel) and verify that refreshing a non-root route works (i.e., your SPA fallback is configured).

### Solutions

**Exercise 2 — env vars & security:**

```bash
# .env
VITE_APP_NAME=Dashboard
SECRET_TOKEN=super-secret-do-not-ship
```

```jsx
// src/App.jsx
export default function App() {
  // VITE_APP_NAME is exposed; SECRET_TOKEN is undefined here on purpose.
  console.log(import.meta.env); // SECRET_TOKEN will NOT be present
  return <h1>{import.meta.env.VITE_APP_NAME}</h1>;
}
// Only VITE_-prefixed vars are inlined into the client bundle. Non-prefixed
// vars are withheld so secrets can't leak into code every user can download.
```

**Exercise 3 — aliases:**

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { fileURLToPath, URL } from 'node:url';

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
});
```
```json
// jsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }
```
```jsx
// src/App.jsx
import Card from '@/components/Card';
export default function App() {
  return <Card />;
}
```

**Exercise 4 — lint a real bug:**

```jsx
// ❌ Triggers react-hooks/exhaustive-deps: `count` is used in the effect but missing from the dependency array.
import { useEffect, useState } from 'react';
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    console.log('count is', count); // stale on every change
  }, []); // <- ESLint underlines this
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```
```jsx
// ✅ Declare the dependency so the effect re-runs with fresh values.
useEffect(() => {
  console.log('count is', count);
}, [count]);
```

**Exercise 5 — Netlify SPA fallback:**

```
# public/_redirects   (ends up in dist/_redirects after build)
/*    /index.html   200
```
```bash
npm run build      # produces dist/
# Netlify: build command "npm run build", publish directory "dist"
# Refreshing https://your-site.netlify.app/anything now serves index.html (200)
```

---

## Key takeaways

- `package.json` is the project's contract: `dependencies` ship at runtime, `devDependencies` only build/lint/test; semver ranges plus a committed lockfile give you flexibility *and* reproducibility.
- Vite serves source as native ES modules with esbuild in dev (instant, HMR via React Fast Refresh) and bundles with Rollup for production (tree-shaken, minified, hashed static files).
- `import.meta.env` exposes only `VITE_`-prefixed variables to the client — **never** put secrets there; they're plaintext in the bundle.
- ESLint flat config + `eslint-plugin-react-hooks` + `react-refresh` catches the most common React bugs; Prettier handles formatting — keep them in separate lanes.
- React Developer Tools (Components + Profiler) plus source maps and breakpoints are how experts debug both correctness and performance.
- SPAs are static files; every static host needs a **fallback rewrite to `/index.html`** or deep-link refreshes 404 — and GitHub Pages subpaths also need `base`.
- React is a view library: lean on the ecosystem (router, react-query, zustand/Redux Toolkit, react-hook-form, framer-motion, react-i18next) and adopt each only when you feel the need.

---

## Further reading

- React — Build a React app from scratch / Start a New Project: <https://react.dev/learn/build-a-react-app-from-scratch> and <https://react.dev/learn/creating-a-react-app>
- React DevTools: <https://react.dev/learn/react-developer-tools>
- Vite Guide & Config Reference: <https://vite.dev/guide/> and <https://vite.dev/config/>
- Vite Env Variables and Modes: <https://vite.dev/guide/env-and-mode>
- ESLint flat config: <https://eslint.org/docs/latest/use/configure/configuration-files>
- `eslint-plugin-react-hooks`: <https://www.npmjs.com/package/eslint-plugin-react-hooks>
- Prettier: <https://prettier.io/docs/>
- npm `package.json` & semver: <https://docs.npmjs.com/cli/configuring-npm/package-json> and <https://docs.npmjs.com/about-semantic-versioning>
- pnpm: <https://pnpm.io/motivation>
- TanStack Query: <https://tanstack.com/query/latest>
- React Router: <https://reactrouter.com/>
- Redux Toolkit: <https://redux-toolkit.js.org/>  ·  Zustand: <https://zustand.docs.pmnd.rs/>
- React Hook Form: <https://react-hook-form.com/>  ·  Motion: <https://motion.dev/>  ·  react-i18next: <https://react.i18next.com/>
- Deploying Vite SPAs: <https://vite.dev/guide/static-deploy>
