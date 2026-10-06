# 26 — Hands-On Projects & a Study Plan

> **What you'll learn:** how to convert everything you've studied in this course into durable skill by building a graded sequence of seven projects, and how to schedule the whole journey into a realistic 12-week plan with checkpoints.

By the end of this lesson you will be able to:

- Pick the right project for your current level and know exactly when you've "finished" it.
- Map each project's requirements back to the specific lessons that teach the concepts it exercises.
- Self-assess against explicit acceptance criteria instead of vague "it works" feelings.
- Use hints to get unstuck without being handed a full solution.
- Push each build further with stretch goals that mirror real production concerns.
- Follow a week-by-week study plan that interleaves reading and building.
- Run your own checkpoints to know whether you're truly ready to move on.

> **Read this lesson differently.** Every other lesson in this course gives you full, runnable solutions. This one deliberately does **not**. The single biggest predictor of whether React "clicks" is hours spent building things that don't yet exist. The code below is scaffolding, requirements, and hints — the implementation is yours. Treat the absence of a solution as the point, not an omission.

---

## Prerequisites

You don't need *every* lesson done before you start Project 1 — the study plan at the end interleaves reading and building. But here's the dependency map so you know what each project leans on:

- **Projects 1–2** assume `01-js-foundations.md` through `07-state-usestate.md`, plus `02-getting-started.md` for Vite, and `06-handling-events.md`. Project 2 also pulls in `05-rendering-lists-conditionals.md` and `10-custom-hooks.md`.
- **Project 3** adds `08-useeffect-side-effects.md` and `15-data-fetching.md`.
- **Project 4** adds the reducer material from `09-hooks-deep-dive.md`.
- **Project 5** adds `14-routing.md` and `11-forms.md`.
- **Project 6** adds `12-context-api.md` and `16-state-management.md`.
- **The capstone** assumes the whole course, including `13-performance.md`, `17-styling.md`, `19-testing.md`, and `20-advanced-patterns.md`.

If a project references a concept you haven't read yet, that's a signal to go read that lesson first — building is the *application* of the reading, not a replacement for it.

---

## Concept

There are two ways to "know" React, and only one of them survives contact with a real codebase.

The first is **recognition knowledge**: you read a lesson, the example makes sense, you nod. This feels like learning but evaporates within days because you never had to *retrieve* anything — the answer was always on the page in front of you.

The second is **generative knowledge**: you face a blank `App.jsx` and have to produce the structure yourself — decide where state lives, what the component tree looks like, which effect runs when. This is hard, occasionally frustrating, and it is the only kind that turns into expertise. Cognitive science calls the underlying mechanisms *retrieval practice* and *desirable difficulty*: the struggle to generate an answer from memory is precisely what cements it.

> **Analogy.** Reading lessons is like watching cooking videos. You can watch a hundred and still burn the first omelette you actually attempt. The burnt omelette teaches you more about heat and timing than the hundred videos did, because now the feedback is *yours*. These projects are your kitchen.

A few principles shape how this lesson is structured:

1. **Graded difficulty.** Each project introduces one or two genuinely new capabilities while reusing the previous ones. You are never asked to learn five new things at once. Project 1 is pure state-and-events; by Project 7 you're orchestrating auth, server cache, routing, validation, types, tests, and a deploy — but each was added one layer at a time.
2. **Acceptance criteria, not vibes.** "Build a todo app" is a weak instruction; you'll declare victory the moment something renders. Each project here ships a checklist of observable behaviours. You're done when every box is checked — not before, not after.
3. **Hints over solutions.** A hint nudges you toward the right hook or the right data shape without writing it for you. If you copy a full solution, you get recognition knowledge again. The hints are calibrated so that a stuck learner gets unstuck, but a coasting learner still has to think.
4. **Stretch goals mirror production.** The base requirements make the app *work*; the stretch goals make it *good* — accessibility, error boundaries, optimistic updates, persistence, tests. Real engineering is mostly stretch goals.

---

## Syntax & API

This lesson is about *applying* APIs, not introducing new ones, so the "API" here is the workflow you'll repeat for every project. Internalize this loop:

```bash
# 1. Scaffold a fresh Vite + React project (one per project — keep them separate)
npm create vite@latest project-name -- --template react
cd project-name
npm install
npm run dev          # dev server with HMR, usually http://localhost:5173

# 2. Build against the acceptance criteria, committing as you complete each one
git init
git add -A && git commit -m "feat: counter increments and decrements"

# 3. Lint and build before you call it done
npm run lint         # if you added ESLint (Vite's react-ts/react templates can include it)
npm run build        # production build must succeed with no errors
npm run preview      # serve the production build locally to sanity-check
```

A minimal Vite entry point you'll recreate in every project (this is the boilerplate Vite generates — know it cold):

```jsx
// src/main.jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';
import './index.css';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

> **StrictMode note.** In development, `<StrictMode>` intentionally double-invokes component bodies, effects (mount → unmount → mount), and reducers to surface impurities. If an effect "fires twice" while you build, that's StrictMode doing its job — see `08-useeffect-side-effects.md`. Don't "fix" it by removing StrictMode; fix the effect's cleanup.

The per-project template you'll see below has this shape:

| Field | Meaning |
| --- | --- |
| **Goal** | One sentence: what the finished app does. |
| **Concepts practiced** | Which lessons this build exercises. |
| **Acceptance criteria** | Observable, checkable behaviours. You're done when all pass. |
| **Hints** | Nudges — hook names, data shapes, gotchas. Not solutions. |
| **Stretch goals** | Optional production-grade extensions. |

---

## Examples

> These are the seven projects. Each is a complete spec, not a code dump. Build them in order — later projects assume the muscle memory of earlier ones.

### Project 1 — Counter & Tip Calculator

**Goal.** A single-page app with two independent widgets: a counter (increment / decrement / reset) and a restaurant tip calculator (bill amount + tip % → tip and total).

**Concepts practiced.** `useState` and the updater function (`07-state-usestate.md`), event handlers and `onChange`/`onClick` (`06-handling-events.md`), controlled inputs (`11-forms.md`), rendering derived values in JSX (`03-jsx-in-depth.md`).

**Acceptance criteria.**

- [ ] Counter shows a number, starts at `0`, has `+`, `−`, and `Reset` buttons that work.
- [ ] Counter never goes below `0` (decrement at `0` is a no-op).
- [ ] Tip calculator has a bill `<input type="number">` and a tip-percent control (buttons for 10/15/20% **or** a slider).
- [ ] Tip amount and grand total update live as either input changes, formatted to 2 decimals.
- [ ] An optional "split between N people" field divides the total by N and shows per-person cost.
- [ ] No console warnings (no uncontrolled-to-controlled input warning).

**Hints.**

- The counter and the calculator hold *separate* state. Resist the urge to put everything in one object until you have a reason to.
- Bill and tip are **input** (state). Tip amount and total are **derived** — compute them during render from the inputs; do **not** store them in their own `useState`. Storing derived data is the single most common beginner mistake (covered in `07-state-usestate.md`).
- Use the functional updater for the counter: `setCount(c => c + 1)`. It's the safe default whenever the next value depends on the previous one.
- Guard against an empty bill field: `Number('')` is `0`, but `Number(undefined)` is `NaN`. Format with `value.toFixed(2)` only after coercing safely.

**Stretch goals.**

- Add currency formatting with `Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })`.
- Add a custom tip-percent input alongside the presets, and highlight the active preset.
- Make the counter support a configurable step size.

---

### Project 2 — Todo App with Filters + localStorage

**Goal.** A classic todo list: add, toggle, edit, and delete tasks; filter by All / Active / Completed; persist across page reloads.

**Concepts practiced.** Lists and stable `key`s (`05-rendering-lists-conditionals.md`), immutable array updates (`07-state-usestate.md`), conditional rendering for the filter views, a **custom hook** for persistence (`10-custom-hooks.md`), and the `useEffect` sync to `localStorage` (`08-useeffect-side-effects.md`).

**Acceptance criteria.**

- [ ] A form adds a new todo; submitting clears the input and the input refocuses.
- [ ] Empty / whitespace-only todos are rejected.
- [ ] Each todo can be toggled complete (with a visible completed style) and deleted.
- [ ] Double-clicking a todo lets you edit its text inline; `Enter` saves, `Escape` cancels.
- [ ] Filter buttons (All / Active / Completed) change which todos show; the active filter is visually marked.
- [ ] A footer shows the count of remaining active items and a "Clear completed" button.
- [ ] Todos and the current filter survive a full page reload (`localStorage`).
- [ ] No `key` warnings in the console; keys are stable IDs, **not** array indices.

**Hints.**

- Generate IDs with `crypto.randomUUID()` — stable and built into the browser. Never use the array index as a key when items can be reordered, inserted, or deleted (`05-rendering-lists-conditionals.md` explains exactly why).
- Every update is immutable: add with `[...todos, newTodo]`, toggle with `todos.map(t => t.id === id ? { ...t, done: !t.done } : t)`, delete with `todos.filter(t => t.id !== id)`.
- The filtered list is **derived** — compute it in render from `todos` + `filter`. Don't keep a separate `filteredTodos` state.
- Write a `useLocalStorage(key, initialValue)` custom hook that returns `[value, setValue]` like `useState`, but reads the initial value from storage (lazy initializer: `useState(() => JSON.parse(localStorage.getItem(key)) ?? initial)`) and writes on change. This is the reusable piece — see `10-custom-hooks.md`.
- Watch out for the lazy initializer: pass a *function* to `useState`, not the result of calling one, or you'll hit `localStorage` on every render.

**Stretch goals.**

- Add drag-and-drop reordering (this is where stable keys pay off).
- Sync across browser tabs by listening for the `storage` window event inside your hook.
- Add a `useReducer`-based version of the todo logic and compare it with the `useState` version — useful warm-up for Project 4.

---

### Project 3 — Weather / GitHub-User Lookup

**Goal.** Type a query (a city, or a GitHub username) and fetch live data, handling the full loading → success → error lifecycle.

**Concepts practiced.** `fetch` and `async`/`await`, `useEffect` for data fetching with cleanup and `AbortController` (`08-useeffect-side-effects.md`, `15-data-fetching.md`), loading/error UI states, and request cancellation to avoid race conditions.

> Use the GitHub users API — `https://api.github.com/users/{username}` — it needs no API key. (For weather, Open-Meteo's free API needs no key either.)

**Acceptance criteria.**

- [ ] A search input + submit triggers a fetch for the entered username.
- [ ] While the request is in flight, a loading indicator shows.
- [ ] On success, render the result (avatar, name, bio, repo count, followers).
- [ ] On a 404 or network error, show a friendly error message — never a blank screen or an unhandled rejection in the console.
- [ ] Rapidly changing the query does **not** show stale results from an earlier, slower request (no race condition).
- [ ] Re-submitting the same query is handled gracefully (no duplicate flicker).

**Hints.**

- Model state as a small state machine, not three loose booleans. A single `status` field of `'idle' | 'loading' | 'success' | 'error'` plus `data` and `error` is far easier to reason about than `isLoading`, `isError`, and `data` that can contradict each other.
- `fetch` does **not** reject on HTTP 404/500 — it only rejects on network failure. Check `response.ok` and `throw` yourself otherwise.
- Cancel in-flight requests with `AbortController`: create one per request, pass `signal` to `fetch`, and call `controller.abort()` in the effect's cleanup. An aborted fetch throws an `AbortError` — swallow that specific error rather than showing it to the user.
- Put the fetch in a `useEffect` keyed on the submitted query, *or* extract a `useFetch(url)` custom hook (`10-custom-hooks.md`) — the latter is reusable across the weather and GitHub variants.
- `15-data-fetching.md` shows why raw `useEffect` fetching is fragile (no caching, manual cancellation, waterfalls). Feel the pain here on purpose — it's the motivation for React Query in the capstone.

**Stretch goals.**

- Add debouncing so it fetches as you type (300ms after the last keystroke) instead of on submit.
- Cache results in a `Map` so re-searching a previous query is instant.
- Combine both data sources (weather + GitHub) behind one search box with a tabbed result view.

---

### Project 4 — Quiz App

**Goal.** A multi-question quiz that walks the user through questions one at a time, tracks the score, supports a timer, and shows a results screen — all driven by a reducer.

**Concepts practiced.** `useReducer` for non-trivial state transitions (`09-hooks-deep-dive.md`), conditional flow/screens, derived UI from a single state object, and (optionally) `useEffect` for the per-question timer.

**Acceptance criteria.**

- [ ] App moves through three screens: *start* → *question* → *results*.
- [ ] Questions come from a static array of `{ question, options, correctIndex }`.
- [ ] Selecting an answer locks it, marks correct/incorrect, and enables a "Next" button.
- [ ] Score increments only for correct answers and is shown on the results screen.
- [ ] A "Restart" button on the results screen returns to the start screen with a fresh state.
- [ ] All state transitions go through the reducer — no scattered `setState` calls mutating overlapping fields.

**Hints.**

- This is the project where `useReducer` earns its keep. Your state is one object: `{ status, index, answer, score }`. Your actions are `'start'`, `'answer'`, `'next'`, `'finish'`, `'restart'`. The reducer is a pure `(state, action) => newState` function — see `09-hooks-deep-dive.md`.
- Reducers must be **pure**: no fetching, no `Date.now()` inside them, no mutation of the incoming `state`. Return a new object every time.
- Derive everything you can in render: the current question is `questions[state.index]`; "is this the last question?" is `state.index === questions.length - 1`. Don't store these.
- For a timer, dispatch a `'tick'` action from a `setInterval` inside `useEffect`, and clean up the interval on unmount/dependency change. StrictMode will mount the effect twice in dev — your cleanup must handle that.

**Stretch goals.**

- Add a global countdown for the whole quiz; when it hits zero, jump straight to results.
- Fetch questions from the Open Trivia DB (`https://opentdb.com/api.php`) instead of a static array — now you're combining Project 3's fetching with Project 4's reducer.
- Persist a high score with the `useLocalStorage` hook from Project 2.

---

### Project 5 — Markdown Notes App with Routing

**Goal.** A notes app with full CRUD (create, read, update, delete), client-side routing, and live Markdown preview. Each note has its own URL.

**Concepts practiced.** `react-router-dom` v6+ (routes, nested routes, `useParams`, `useNavigate`, `<Link>`/`<NavLink>`, `<Outlet>`) from `14-routing.md`, controlled forms (`11-forms.md`), immutable CRUD over a list (`07-state-usestate.md`), and persistence via your `useLocalStorage` hook (`10-custom-hooks.md`).

```bash
npm install react-router-dom
```

**Acceptance criteria.**

- [ ] Routes exist for: list view (`/`), single note (`/notes/:id`), and edit/new (`/notes/:id/edit`, `/notes/new`).
- [ ] A sidebar lists all notes with their titles; the active note is highlighted via `<NavLink>`.
- [ ] Creating a note navigates to its edit view; saving navigates to its read view.
- [ ] The note body is written in Markdown and rendered to HTML in the read view.
- [ ] Deleting a note removes it and navigates back to the list; deleting the currently-open note doesn't leave a broken URL.
- [ ] Visiting a non-existent note ID shows a "Not found" UI, not a crash.
- [ ] Notes persist across reloads.
- [ ] A catch-all `*` route renders a 404 page.

**Hints.**

- Use a Markdown renderer like `react-markdown` (`npm install react-markdown`). If you instead use `dangerouslySetInnerHTML` with a raw Markdown-to-HTML library, you **must** sanitize the output (e.g. with `dompurify`) — never inject unsanitized user HTML.
- Lay out shared chrome (sidebar + header) in a layout route that renders `<Outlet />`; child routes fill the outlet. This is the v6 nested-routing pattern from `14-routing.md`.
- `useParams()` gives you `:id` as a **string** — match it against your note IDs accordingly.
- After a save or delete, navigate imperatively with `useNavigate()`. After delete, navigate *before* the now-deleted note's route tries to render stale data.
- Keep notes state in one place (lifted to the layout/App, or in context) so both the sidebar and the detail view read the same source of truth.

**Stretch goals.**

- Add search/filter over note titles and bodies.
- Use React Router **loaders and actions** (data router API) instead of effects for reads and form submissions.
- Add unsaved-changes protection with a `useBlocker` / navigation prompt.

---

### Project 6 — E-commerce Cart

**Goal.** A product catalog with a shopping cart that lives in global state, computes derived totals, and persists — the kind of cross-cutting state that local `useState` can no longer carry cleanly.

**Concepts practiced.** Global state via the Context API (`12-context-api.md`) **or** Zustand (`16-state-management.md`), derived/selector computations, immutable updates to a keyed collection, and `useReducer` or a store for cart actions.

**Acceptance criteria.**

- [ ] A product grid renders from a static catalog (`{ id, name, price, image }`).
- [ ] "Add to cart" adds an item; adding an already-present item increases its quantity rather than duplicating it.
- [ ] A cart drawer/page lists items with quantity steppers and remove buttons.
- [ ] Cart subtotal, item count, tax, and grand total are **derived** from cart state, not stored.
- [ ] The header cart badge shows the live total item count from anywhere in the tree (no prop drilling).
- [ ] Removing the last unit of an item removes the line entirely.
- [ ] Cart contents persist across reloads.
- [ ] Empty-cart state is handled with a friendly message.

**Hints.**

- This is the project that justifies global state. With Context, expose a `CartProvider` and a `useCart()` hook; store the cart as a map/array and the *mutators* (`addItem`, `removeItem`, `setQty`) in the context value. See `12-context-api.md` for the provider pattern and the "split state vs. dispatch into two contexts" performance trick.
- Totals are derived — compute them with a selector or a `useMemo` over cart items (`13-performance.md`). Storing `total` in state means it can drift out of sync; never do it.
- If you reach for Zustand (`16-state-management.md`), the win is selectors: components subscribe to *slices* (`useStore(s => s.itemCount)`) and only re-render when that slice changes. Compare the re-render behaviour against the naive Context version — wrap the catalog in `React.memo` and watch.
- Quantity steppers must clamp at a minimum of 1 (or trigger removal at 0). Decide the rule and enforce it in one place.

**Stretch goals.**

- Add quantity-based discounts or a promo-code field that adjusts the total.
- Persist with a Zustand `persist` middleware instead of a hand-rolled effect.
- Add an order-summary checkout step (no real payment) with a form (`11-forms.md`).

---

### Project 7 — CAPSTONE: A Full App

**Goal.** Build one cohesive, production-shaped application that integrates **every** pillar of the course. The exact domain is your choice — a task tracker, a recipe manager, a bookmarking app, a habit tracker — pick something you'd actually use. What matters is the *architecture*, not the domain.

**Concepts practiced.** Everything: components and composition (`04-components-and-props.md`, `20-advanced-patterns.md`), routing with protected routes (`14-routing.md`), server data and caching with React Query (`15-data-fetching.md`, `16-state-management.md`), forms with validation (`11-forms.md`), performance hygiene (`13-performance.md`), styling system (`17-styling.md`), TypeScript throughout, testing (`19-testing.md`), and deployment.

```bash
# Start from the TypeScript template this time
npm create vite@latest capstone -- --template react-ts
cd capstone
npm install
npm install @tanstack/react-query react-router-dom react-hook-form zod @hookform/resolvers
npm install -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

**Acceptance criteria.**

- [ ] **TypeScript everywhere.** No `any` in your own code; props, state, and API responses are typed. `npm run build` (which type-checks) passes clean.
- [ ] **Authentication.** A login flow stores a session (token in memory + refresh strategy, or a mock auth provider). Logging out clears it.
- [ ] **Protected routes.** Unauthenticated users hitting a protected URL are redirected to login and returned to their target after signing in.
- [ ] **Server data via React Query.** All remote reads use `useQuery`; mutations use `useMutation` with cache invalidation. No raw `useEffect` fetching in feature code. (You may use a mock API like JSON Server, MSW, or a free backend.)
- [ ] **Forms with validation.** At least one non-trivial form uses `react-hook-form` + `zod` (or equivalent) with inline error messages, disabled-while-submitting, and server-error surfacing — see `11-forms.md`.
- [ ] **Loading & error states** are handled everywhere data is fetched, including empty states.
- [ ] **Performance.** The app is profiled with React DevTools; obvious wasted renders are eliminated with `memo`/`useMemo`/`useCallback` *where measured to matter* (`13-performance.md`) — not sprinkled blindly.
- [ ] **Tests.** Critical flows (login, a CRUD operation, form validation) have tests with React Testing Library + Vitest; the suite is green (`19-testing.md`).
- [ ] **Accessibility.** Forms have labels, interactive elements are keyboard-reachable, and there are no console a11y warnings.
- [ ] **Deployed.** The production build is live on a public URL (Netlify, Vercel, or GitHub Pages) with SPA routing configured so deep links don't 404.

**Hints.**

- **Layer the architecture.** Roughly: `api/` (typed fetchers + query hooks), `features/` (domain UI), `components/` (shared dumb UI), `routes/` (route config + guards), `lib/` (auth, query client). Keep server state (React Query) and client state (local/Context/Zustand) clearly separated — this is the central lesson of `16-state-management.md`.
- **Protected routes**: a `<RequireAuth>` wrapper that checks auth state and `<Navigate to="/login" state={{ from: location }} replace />` when missing. After login, read `location.state.from` to bounce back.
- **React Query setup**: one `QueryClient`, wrap the app in `<QueryClientProvider>`. Co-locate query keys, and after a mutation call `queryClient.invalidateQueries({ queryKey: [...] })`. Try an optimistic update on one mutation to feel the full pattern.
- **Validation**: define a `zod` schema, infer the TS type from it (`z.infer<typeof schema>`), and wire it into `react-hook-form` with `zodResolver` (imported from `@hookform/resolvers/zod`): `useForm({ resolver: zodResolver(schema) })`. One schema becomes both your runtime validation and your compile-time type.
- **SPA deploy gotcha**: client-side routing needs a rewrite rule so every path serves `index.html` (Netlify `_redirects` / Vercel rewrites / a `404.html` trick for GitHub Pages). Without it, refreshing on `/notes/42` 404s.
- **Don't gold-plate before it works.** Get the happy path end-to-end first (auth → list → create → deploy), then layer in validation polish, tests, and performance. Vertical slices beat horizontal perfection.

**Stretch goals.**

- Add an error boundary (`20-advanced-patterns.md`) around route segments with a recovery action.
- Add infinite scroll or pagination with React Query's `useInfiniteQuery`.
- Add a CI pipeline (GitHub Actions) that runs lint + type-check + tests on every push.
- Add end-to-end tests with Playwright for the login + critical flow.
- Code-split routes with `React.lazy` + `<Suspense>` and confirm the bundle shrinks.

---

## Under the hood

Why does building work when re-reading doesn't? A few mental models worth carrying into every project:

**The "where does this state live?" decision is the whole game.** Most React bugs trace back to state in the wrong place — duplicated, stored-instead-of-derived, lifted too high, or trapped too low. Each project above is secretly a drill in that one decision. Project 1 teaches "derive, don't store." Project 2 teaches "one source of truth, immutable updates." Project 6 teaches "lift to global only when truly cross-cutting." When you get stuck, the first question is almost always *where should this value live, and is it input or derived?* (`07-state-usestate.md` and `16-state-management.md` are the canonical references.)

**Server state is not client state.** Project 3 makes you hand-roll fetching with `useEffect` so that the capstone's switch to React Query lands as relief, not magic. The deep truth (`15-data-fetching.md`): data living on a server is *cached, asynchronous, and shared* — it needs deduping, background refetching, staleness, and invalidation. `useState` + `useEffect` can fake this, but you'll reimplement a worse version of React Query by hand. Knowing *why* is what separates copying a tutorial from making an architecture decision.

**StrictMode is a truth serum.** Across these projects you'll see effects fire twice and reducers run twice in dev. That's `<StrictMode>` deliberately stress-testing your code for impurity and missing cleanup. The mental model: React reserves the right to call your render and your reducers *any number of times*, and to mount/unmount components at will (this is also what makes future concurrent features safe). Code that breaks under double-invocation is buggy code that happened to work — fix the cleanup, don't disable the warning. (`08-useeffect-side-effects.md` covers this in depth.)

**Reconciliation rewards stable keys.** Project 2's drag-and-drop and Project 6's cart steppers both expose how React diffs lists: a `key` is React's identity for a list item across renders. Index keys lie about identity the moment the list reorders, causing state to attach to the wrong row. This isn't a style nit — it's the reconciler's contract (`05-rendering-lists-conditionals.md`).

**Performance work must be measured, not guessed.** The capstone explicitly asks you to *profile first*. The runtime mental model (`13-performance.md`): re-rendering is cheap; it's *expensive* re-renders and large subtrees that hurt. `memo`/`useMemo`/`useCallback` add their own cost and complexity, so applying them without a measured problem is a net loss. Build the habit of opening the Profiler before reaching for a memo.

---

## Common mistakes & pitfalls

These are the failure modes you'll actually hit while building. Each shows the wrong instinct and the fix.

**1. Storing derived data in state.**

```jsx
// WRONG — total is redundant state that can drift out of sync
const [bill, setBill] = useState(0);
const [tipPct, setTipPct] = useState(15);
const [total, setTotal] = useState(0);

function handleBill(e) {
  setBill(Number(e.target.value));
  setTotal(Number(e.target.value) * (1 + tipPct / 100)); // stale tipPct!
}
```

```jsx
// RIGHT — derive during render; one source of truth
const [bill, setBill] = useState(0);
const [tipPct, setTipPct] = useState(15);
const total = bill * (1 + tipPct / 100); // always correct, no sync code
```

*Why:* derived values recomputed each render are never stale. Storing them forces you to write (and forget) sync code. (`07-state-usestate.md`.)

**2. Using the array index as a `key`.**

```jsx
// WRONG — index keys break when items reorder/insert/delete
{todos.map((todo, i) => <TodoItem key={i} todo={todo} />)}
```

```jsx
// RIGHT — a stable, unique ID
{todos.map((todo) => <TodoItem key={todo.id} todo={todo} />)}
```

*Why:* React uses `key` as the item's identity. Index keys make React reuse the wrong DOM node and component state when the list changes (`05-rendering-lists-conditionals.md`).

**3. Race conditions in fetch effects.**

```jsx
// WRONG — a slow earlier request can overwrite a fast later one
useEffect(() => {
  fetch(`https://api.github.com/users/${query}`)
    .then((r) => r.json())
    .then(setData); // stale response may land last
}, [query]);
```

```jsx
// RIGHT — abort the in-flight request on cleanup
useEffect(() => {
  const controller = new AbortController();
  fetch(`https://api.github.com/users/${query}`, { signal: controller.signal })
    .then((r) => {
      if (!r.ok) throw new Error('Not found');
      return r.json();
    })
    .then(setData)
    .catch((err) => {
      if (err.name !== 'AbortError') setError(err);
    });
  return () => controller.abort();
}, [query]);
```

*Why:* without cancellation, responses can resolve out of order and the UI shows stale data (`15-data-fetching.md`). In the capstone, React Query handles this for you — which is the point.

**4. Mutating state directly.**

```jsx
// WRONG — mutating the existing array; React may not re-render
todos.push(newTodo);
setTodos(todos);
```

```jsx
// RIGHT — new array reference
setTodos([...todos, newTodo]);
```

*Why:* React decides whether to re-render by comparing references. Mutating in place keeps the same reference and can skip the update (and corrupts time-travel/StrictMode behaviour) (`07-state-usestate.md`).

**5. Side effects inside a reducer.**

```jsx
// WRONG — reducers must be pure
function reducer(state, action) {
  if (action.type === 'answer') {
    fetch('/api/log', { method: 'POST' }); // impure!
    return { ...state, answer: action.payload };
  }
}
```

```jsx
// RIGHT — keep the reducer pure; do effects in useEffect/event handlers
function reducer(state, action) {
  if (action.type === 'answer') return { ...state, answer: action.payload };
  return state;
}
// log the answer in the click handler or an effect keyed on state.answer
```

*Why:* reducers run during render and may be called multiple times (StrictMode); effects belong outside (`09-hooks-deep-dive.md`).

---

## Best practices

- **Do** scaffold each project as its own Vite app and commit per acceptance criterion — your git log becomes a record of progress.
- **Do** check off acceptance criteria literally; "looks done" is not "done."
- **Do** build the happy path end-to-end before polishing — vertical slices over horizontal perfection.
- **Do** prefer deriving values over storing them, and immutable updates over mutation, every single time.
- **Do** profile before optimizing; add `memo`/`useMemo`/`useCallback` only where measured (`13-performance.md`).
- **Do** type the capstone fully and let `npm run build` be your type-check gate.
- **Don't** copy full solutions from elsewhere — the struggle is the learning. Use hints, then your own head.
- **Don't** disable `<StrictMode>` to silence double-invocation; fix the cleanup instead.
- **Don't** reach for global state (Context/Zustand) until prop drilling actually hurts (`16-state-management.md`).
- **Don't** skip the deploy step on the capstone — shipping surfaces a whole class of bugs (routing rewrites, env vars, build-only failures) that `npm run dev` hides.

---

## Exercises

These are meta-exercises that sit *around* the seven projects — they sharpen self-assessment and reflection, which is what turns finished projects into transferable skill.

1. **Acceptance-test yourself (easy).** Take Project 1 and, *before* writing code, expand its acceptance criteria into 8–10 even more specific, observable checks (e.g. "typing letters into the bill field is rejected or ignored"). Then build to your own spec.
2. **Refactor across paradigms (medium).** Build Project 2's todo logic with `useState`, then rebuild the same logic with `useReducer`. Write 3 sentences on which you'd choose and why.
3. **Diagnose a race (medium).** In Project 3, deliberately remove the `AbortController`, then type quickly in the search box against a throttled connection (DevTools → Network → Slow 3G). Observe stale results, then fix it and confirm.
4. **State-location audit (hard).** For Project 6, list every piece of state and classify each as *input*, *derived*, *server*, or *global-client*. Justify each. Then verify your code matches the classification (no derived state in `useState`, no server state hand-fetched).
5. **Ship it (hard).** Deploy the capstone and then break it on purpose: refresh on a deep protected route while logged out. Document what happens and why, and confirm your route guard + SPA rewrite handle it.

### Solutions

These are *worked approaches*, not full implementations — the builds themselves are yours.

```text
Exercise 1 — Sample expanded criteria for the tip calculator:
  - Bill field accepts only non-negative numbers; negatives clamp to 0.
  - Empty bill renders tip/total as $0.00, not NaN.
  - Switching tip preset updates total within the same frame (no stale value).
  - Split-by-N with N<1 is rejected or clamped to 1.
  - Per-person value rounds to 2 decimals and sums back to the total (±1 cent).
Build to this list; you're done only when all pass.
```

```text
Exercise 2 — useState vs useReducer for todos:
  useState wins when updates are few and independent (toggle one field).
  useReducer wins when many actions touch one overlapping state object,
  when next state depends on previous state in non-trivial ways, or when
  you want all transitions in one testable pure function. Todos sit on the
  boundary; once you add filters + editing + bulk "clear completed",
  useReducer's single dispatch surface tends to read cleaner. (See 09.)
```

```jsx
// Exercise 3 — the fix is the AbortController pattern from the pitfalls section.
// Confirm by logging the resolved query alongside the current query:
useEffect(() => {
  const controller = new AbortController();
  const current = query;
  fetchUser(query, controller.signal)
    .then((data) => {
      console.log('resolved for', current, 'current is', query);
      setData(data);
    })
    .catch((e) => { if (e.name !== 'AbortError') setError(e); });
  return () => controller.abort();
}, [query]);
// With abort in place, you never see "resolved for X, current is Y".
```

```text
Exercise 4 — State-location audit for the cart:
  catalog items ........... server state (fetched) OR static input data
  cart contents ........... global-client state (Context/Zustand) — cross-cutting
  cart subtotal/tax/total . DERIVED — compute via selector/useMemo, never store
  item count badge ........ DERIVED from cart contents
  drawer open/closed ...... local UI state (useState in the layout)
  Pass criterion: no totals in useState; cart read from one source everywhere.
```

```text
Exercise 5 — Deep-link-while-logged-out expectation:
  1. Browser requests /items/42 directly.
  2. SPA rewrite rule serves index.html (else: 404 from host — fix the rewrite).
  3. App boots, RequireAuth sees no session, <Navigate to="/login"
     state={{ from: '/items/42' }} replace /> fires.
  4. After login, read location.state.from and navigate back to /items/42.
  If any step fails, that's the bug to document and fix.
```

---

## Key takeaways

- Generative practice (building from a blank file) beats recognition (re-reading) — that's why this lesson withholds solutions on purpose.
- Each project adds one or two new capabilities while reusing the rest; build them **in order**.
- "Done" means every acceptance-criterion box is checked — define done observably, not by vibes.
- The recurring skill across all seven projects is deciding **where state lives** and whether a value is input, derived, server, or global.
- Project 3 deliberately makes raw fetching painful so the capstone's React Query feels earned, not magical.
- StrictMode's double-invocation is a feature: it exposes impure code and missing cleanup — fix the code, never the warning.
- Optimize only what you've profiled; type the capstone fully; and actually deploy it — shipping reveals a class of bugs `dev` hides.

---

## A realistic 12-week study plan

This maps the full course (lessons `01`–`20`, plus this projects lesson and the closing lesson) onto twelve weeks at roughly **8–10 focused hours/week**. Read in the morning, build in the afternoon. Adjust pace to your life — the *order* matters more than the calendar.

| Week | Read (lessons) | Build / Practice | Checkpoint — you should be able to… |
| --- | --- | --- | --- |
| **1** | `01-js-foundations`, `02-getting-started`, `03-jsx-in-depth` | Scaffold a Vite app; render static JSX; small JSX katas (map, ternary). | Explain JSX → `createElement`, run `npm run dev`, use destructuring/spread/arrow fns fluently. |
| **2** | `04-components-and-props`, `05-rendering-lists-conditionals` | Build a static UI from components + props; render a list with proper keys. | Compose components, pass/typecheck props, render lists without key warnings. |
| **3** | `06-handling-events`, `07-state-usestate` | **Project 1 — Counter & Tip Calculator.** | Manage local state, handle events, derive (not store) values. |
| **4** | `08-useeffect-side-effects`, `10-custom-hooks` | **Project 2 — Todo App + localStorage** (write `useLocalStorage`). | Use effects with cleanup, extract a reusable custom hook, do immutable updates. |
| **5** | `15-data-fetching`, revisit `08` | **Project 3 — Weather / GitHub lookup** (loading/error/abort). | Fetch async, model a status state machine, cancel stale requests. |
| **6** | `09-hooks-deep-dive` | **Project 4 — Quiz App** (reducer-driven). | Choose `useReducer` over `useState` deliberately; keep reducers pure. |
| **7** | `11-forms`, `14-routing` | **Project 5 — Markdown Notes** (routing + CRUD) — part 1: routing + list/detail. | Configure v6 routes, nested layouts, `useParams`/`useNavigate`, controlled forms. |
| **8** | `12-context-api`, `16-state-management` | Finish Project 5; **start Project 6 — E-commerce Cart** (global state). | Decide local vs. global state; build a provider + `useCart`; derive totals. |
| **9** | `13-performance`, `17-styling` | Finish Project 6; profile it; eliminate measured wasted renders; apply a styling approach. | Use the Profiler; apply `memo`/`useMemo`/`useCallback` only where measured; style consistently. |
| **10** | `19-testing`, `20-advanced-patterns` | Add tests to Projects 2/4/6; practice error boundaries + a compound component. | Write RTL + Vitest tests for user flows; reach for advanced patterns when warranted. |
| **11** | (TypeScript + React Query refresh: `15`, `16`) | **Capstone — part 1:** TS scaffold, auth, protected routes, React Query reads. | Type a full app; guard routes; fetch all server data through React Query. |
| **12** | Closing lesson + review weak spots | **Capstone — part 2:** mutations + invalidation, form validation, tests, **deploy**. | Ship a live, typed, tested, deployed app — and explain every architectural choice. |

**How to use the checkpoints.** At the end of each week, do the checkpoint *without notes*. If you can't, that week isn't done — re-build the relevant exercise or re-read the lesson before advancing. Skipping a failed checkpoint is how people reach Week 12 and discover their foundation is hollow.

**Two failure modes to avoid.** First, *tutorial purgatory* — endlessly reading/watching without building. The plan forces a build almost every week; honor it. Second, *gold-plating* — polishing Project 2 for three weeks. Hit the acceptance criteria, ship it, move on. There's a whole career ahead to perfect things.

---

## Further reading

- **React — Learn React (official tutorial & guides):** <https://react.dev/learn>
- **React — Thinking in React** (the canonical "where does state live" essay): <https://react.dev/learn/thinking-in-react>
- **React — Choosing the State Structure / You Might Not Need an Effect:** <https://react.dev/learn/choosing-the-state-structure> and <https://react.dev/learn/you-might-not-need-an-effect>
- **Vite — Getting Started:** <https://vite.dev/guide/>
- **React Router (v6/v7) docs:** <https://reactrouter.com/>
- **TanStack Query (React Query):** <https://tanstack.com/query/latest>
- **React Hook Form:** <https://react-hook-form.com/> · **Zod:** <https://zod.dev/>
- **React Testing Library:** <https://testing-library.com/docs/react-testing-library/intro/> · **Vitest:** <https://vitest.dev/>
- **Free, no-key APIs for practice:** GitHub Users <https://docs.github.com/en/rest/users>, Open-Meteo <https://open-meteo.com/>, Open Trivia DB <https://opentdb.com/api_config.php>
- **Deployment:** Netlify <https://docs.netlify.com/>, Vercel <https://vercel.com/docs>, GitHub Pages <https://docs.github.com/en/pages>

> **Where to next.** With these seven projects in your portfolio and the checkpoints passed, you've crossed from "learning React" to "building with React." The closing lesson of the course ties together where to go from here — patterns, the ecosystem, and keeping current.
