# 02 — What is React & Setting Up Your Environment

> **What you'll learn:** what React actually is and the problems it solves, the mental shift from imperative to declarative UI, and how to scaffold, run, build, and understand a real React 19 project with Vite — line by line.

By the end of this lesson you will be able to:

- Explain in your own words what React is, what it is *not*, and which problems it was built to solve.
- Distinguish **imperative** DOM manipulation from React's **declarative** model, and describe component-based architecture.
- Describe the **Virtual DOM** and **reconciliation** at an intuitive level, and what a **Single-Page Application (SPA)** is.
- Install a modern Node.js + npm toolchain and verify the versions.
- Scaffold a project with `npm create vite@latest`, install dependencies, and run the dev server, production build, and preview.
- Read and understand every generated file: `index.html`, `src/main.jsx`, `src/App.jsx`, `package.json`, and `vite.config.js`.
- Explain `createRoot` + `root.render`, the `#root` div, and what `<StrictMode>` does (including why it double-invokes in development).
- Write and render your very first component, and read error messages in the browser console.

---

## Prerequisites

This is an early lesson, so the bar is low — but you should be comfortable with the following before continuing:

- **Basic HTML & CSS** — what a `<div>` is, how elements nest, what an `id` and `class` are.
- **JavaScript fundamentals** — variables (`const`/`let`), functions, arrays, objects, template literals, and `import`/`export` (ES modules). If `export default function App() {}` looks alien, skim an ES-modules primer first.
- **A terminal** — you can open a shell and run a command. We'll use macOS/Linux-style commands; Windows users can use PowerShell or WSL.
- **Lesson 01** — assumed to have introduced the course roadmap and what "building UI on the web" means. The JSX syntax we touch lightly here is covered in depth in **see 03-jsx-in-depth.md**, and the components/props mechanics in **see 04-components-and-props.md**. State and interactivity arrive in **see 07-state-usestate.md**.

You do **not** need any prior React experience. That's the whole point.

---

## Concept

### What is React?

**React is a JavaScript library for building user interfaces out of components.** That sentence is doing a lot of work, so let's unpack each part.

- **JavaScript library** — not a full framework. React focuses on one job: rendering UI and keeping it in sync with your data. It deliberately leaves routing, data fetching, and global state to other tools (or to frameworks built *on top of* React, like Next.js).
- **Building user interfaces** — React draws what the user sees and handles updating it when things change. It's the *view* layer.
- **Out of components** — you compose your UI from small, reusable, self-contained pieces called **components**, the same way you build a Lego model out of bricks.

React was created at Facebook (now Meta) and open-sourced in 2013 to solve a very specific pain: **keeping a complex, interactive UI in sync with constantly changing data without your code turning into spaghetti.**

### The problem React solves: keeping the UI in sync with data

Imagine a "like" button that shows a count. Without React, in plain ("vanilla") JavaScript, you do something like this:

```js
// Vanilla JS — imperative
let likes = 0;
const button = document.querySelector("#like-btn");
const label = document.querySelector("#like-count");

button.addEventListener("click", () => {
  likes = likes + 1;                 // 1. update the data
  label.textContent = `${likes} likes`; // 2. manually update the DOM
});
```

Notice you must do **two** things every time: update the data *and* hand-write the DOM update. With one button that's fine. But a real app has hundreds of pieces of data feeding hundreds of places on screen. Every time data changes, you have to remember *exactly which DOM nodes* depend on it and surgically update each one. Miss one, and the screen shows stale, inconsistent information. This is the single biggest source of bugs in hand-written UI code.

### Imperative vs declarative UI

This is the central mental shift in this entire course, so slow down here.

- **Imperative** means *you give step-by-step instructions on how to change the screen.* "Find the label node. Set its text. Now find the badge. Toggle its class." You are the one micromanaging the DOM. The vanilla example above is imperative.
- **Declarative** means *you describe what the screen should look like for a given state, and let the system figure out how to get there.* You don't touch the DOM. You say "for `likes = 5`, the UI is a button reading `5 likes`," and React makes the actual DOM match that description.

A helpful analogy: ordering at a restaurant. **Imperative** is walking into the kitchen and personally cooking the dish step by step. **Declarative** is telling the waiter "I'll have the pasta" — you describe the desired result; the kitchen figures out the steps. React is the kitchen.

Here's the same like-button, declaratively, in React (don't worry about understanding every token yet — `useState` is fully covered in **see 07-state-usestate.md**):

```jsx
import { useState } from "react";

function LikeButton() {
  const [likes, setLikes] = useState(0);

  return (
    <button onClick={() => setLikes(likes + 1)}>
      {likes} likes
    </button>
  );
}
```

You never wrote `label.textContent = ...`. You described what the button *is* for a given `likes` value, and React keeps the real DOM matching. Change the data, the UI follows — automatically. That is the entire promise of React.

### Component-based architecture

A **component** is a JavaScript function that returns a description of some UI. Components can use other components, just like functions call other functions. You might have a `<Header>`, a `<Sidebar>`, and a `<Feed>`, and the `<Feed>` is made of many `<Post>` components, each containing a `<LikeButton>`.

This gives you three superpowers:

1. **Reuse** — write `<LikeButton>` once, use it on every post.
2. **Isolation** — each component manages its own little world; a bug in one rarely cascades into others.
3. **Composition** — you build big UIs by nesting small ones, the way functions compose into programs.

### The Virtual DOM & reconciliation (intuitively)

If React re-describes the *entire* UI every time data changes, isn't it slow to rebuild the whole page? It would be — touching the real browser DOM is expensive. React's trick is the **Virtual DOM**.

The Virtual DOM is a lightweight **JavaScript object** that mirrors what the real DOM should look like. It's cheap to create because it's just plain objects in memory, not real DOM nodes.

When your data changes, React:

1. Builds a **new** Virtual DOM tree describing the desired UI.
2. **Diffs** it against the previous Virtual DOM tree (this comparison step is called **reconciliation**).
3. Computes the **minimal set of real DOM operations** needed to go from old to new.
4. Applies *only those* changes to the real browser DOM.

Analogy: instead of repainting an entire wall every time you want to fix a scuff, you photograph the wall before and after, compare the two photos, and touch up only the spots that differed. React does the "compare two snapshots, touch up the diff" dance for you, on every update.

> Note: modern React (the Fiber architecture) is more sophisticated than "two trees and a diff," and there's a deeper treatment of reconciliation and keys later in the course (**see 05-rendering-lists-conditionals.md** and the rendering-internals lesson). For now, the snapshot-and-diff mental model is exactly right.

### Single-Page Applications (SPA)

A traditional website is **multi-page**: every link click asks the server for a brand-new HTML document, and the browser throws away the current page and renders the new one (the white flash you sometimes see).

A **Single-Page Application** loads one HTML page once, then JavaScript takes over: as you navigate, React swaps the visible components in place without a full page reload. It *feels* like a native app — instant, no flash. React is the canonical tool for building SPAs (though it can also render on the server; more on that much later in the course).

### React vs vanilla JS — when and why

Vanilla JS is perfectly fine for a static page with a sprinkle of interactivity. React earns its keep when your UI has **lots of state that changes over time** and **many places on screen that must stay consistent with that state**. The more dynamic your app, the more the declarative model saves you from bug-prone manual DOM juggling. React trades a small upfront learning curve and a JS bundle for a massive reduction in "the screen is out of sync with reality" bugs.

---

## Syntax & API

The setup-and-render surface area is tiny. Here are the pieces you'll use today.

### Project scaffolding (Vite)

```bash
# Create a new React project named "my-app" using Vite's react template
npm create vite@latest my-app -- --template react
```

- `npm create vite@latest` runs the latest Vite scaffolder.
- The lone `--` separates npm's own arguments from the arguments passed *to* the create script.
- `--template react` selects React with plain JavaScript/JSX. (Use `react-ts` for TypeScript; TS is introduced later in the course.)

### React entry-point APIs

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
```

- **`createRoot(domNode)`** — from `react-dom/client`. Creates a React **root** attached to a real DOM element. Signature: `createRoot(container, options?) → Root`. This is the React 18+ / 19 API. (The old `ReactDOM.render(...)` was the standard API through React 17, **deprecated** in React 18, and **removed** in React 19 — don't use it.)
- **`root.render(reactNode)`** — tells that root to render a React element tree into its container. Calling it again on the same root *updates* rather than re-creates.
- **`<StrictMode>`** — a built-in component that wraps your tree to enable extra development-only checks. It renders nothing visible and has **zero** effect in production builds.

### A component

```jsx
// A component is just a function that returns JSX.
// Its name MUST start with a capital letter.
function App() {
  return <h1>Hello, React</h1>;
}

export default App;
```

- A **function component** takes one argument (`props`, covered in **see 04-components-and-props.md**) and returns a React element (usually written as JSX).
- The capital-letter rule is real and enforced: `<App />` is treated as your component, while `<app />` is treated as a built-in HTML tag.

### npm scripts you'll run

```bash
npm install        # install dependencies listed in package.json
npm run dev        # start the Vite dev server (hot reload) on http://localhost:5173
npm run build      # produce an optimized production bundle in dist/
npm run preview    # serve the built dist/ locally to test the production build
```

---

## Examples

### Example 1 — Verify your toolchain

Before anything React-specific, confirm Node.js and npm are installed and recent enough.

```bash
node -v   # should print something like v20.x or v22.x
npm -v    # should print something like 10.x
```

React 19 + the current Vite require an **active or maintenance LTS Node** — current Vite requires **Node 20.19+ or 22.12+**, with **Node 20 LTS or Node 22 LTS recommended**. (Vite 6, pinned in the `package.json` later in this lesson, still ran on Node 18.18+, but Node 18 has since reached end-of-life and newer Vite has dropped it.) If `node -v` errors or shows an old version, install a current LTS:

- **Recommended (any OS):** use [nvm](https://github.com/nvm-sh/nvm) (Node Version Manager) so you can switch versions per project:

  ```bash
  # install nvm first (see its README), then:
  nvm install 22      # download & install Node 22 LTS
  nvm use 22          # use it in this shell
  node -v             # confirm
  ```

- **Or:** download the LTS installer from <https://nodejs.org>. npm ships *with* Node, so installing Node gives you both.

### Example 2 — Scaffold, install, and run

Now create the project end to end:

```bash
# 1. Scaffold
npm create vite@latest my-app -- --template react

# 2. Move into it (separate command — Vite tells you to do this)
cd my-app

# 3. Install dependencies into node_modules/
npm install

# 4. Start the dev server
npm run dev
```

After `npm run dev`, Vite prints something like:

```text
  VITE v6.0.0  ready in 312 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  press h + enter to show help
```

Open <http://localhost:5173/> in your browser. You'll see the Vite + React starter page with a spinning logo and a counter button. **Edit `src/App.jsx`, save, and watch the page update instantly** without a manual refresh — that's Vite's Hot Module Replacement (HMR).

To stop the dev server, press `Ctrl + C` in the terminal.

### Example 3 — Build and preview the production bundle

The dev server is for development only. To see what ships to users:

```bash
npm run build     # outputs an optimized, minified bundle to dist/
npm run preview   # serves dist/ at http://localhost:4173 to test it
```

`npm run build` runs Vite's production build: it bundles, minifies, tree-shakes, and hashes filenames for caching. `npm run preview` then statically serves that `dist/` folder so you can sanity-check the real artifact before deploying. (You deploy the *contents of `dist/`*, never the dev server.)

### Example 4 — Your very first component, from scratch

Let's replace the starter content with the simplest possible app. Open `src/App.jsx` and replace its entire contents with:

```jsx
// src/App.jsx
function App() {
  const name = "world";
  return (
    <main>
      <h1>Hello, {name}!</h1>
      <p>This is my first React component.</p>
    </main>
  );
}

export default App;
```

Save. The browser updates immediately. You just wrote a declarative component: it *describes* what the UI is (an `<h1>` and a `<p>`), and the `{name}` curly braces drop a JavaScript value into the markup (this is JSX interpolation — **see 03-jsx-in-depth.md**).

### Example 5 — Composing two components

Components compose. Create a second component and use it from `App`:

```jsx
// src/App.jsx
function Greeting({ who }) {
  return <h1>Hello, {who}!</h1>;
}

function App() {
  return (
    <main>
      <Greeting who="Ada" />
      <Greeting who="Linus" />
      <Greeting who="Grace" />
    </main>
  );
}

export default App;
```

One `Greeting` definition, three reuses, each customized via a **prop** (`who`). Three headings render. This is component-based architecture in miniature — and `props` get their own full lesson next (**see 04-components-and-props.md**).

---

## Under the hood

### The render path: from `index.html` to pixels

Here's the chain of events when your app boots, and it's worth internalizing because it demystifies "where does React even start?"

1. The browser loads **`index.html`**, which contains an empty `<div id="root"></div>` and a `<script type="module" src="/src/main.jsx"></script>`.
2. That script — **`main.jsx`** — runs. It calls `createRoot(document.getElementById("root"))` to claim that empty div as React's territory, then `root.render(<App />)` to render your tree into it.
3. React builds the Virtual DOM for `<App />`, reconciles it against the (empty) current state, and produces real DOM nodes inside `#root`.
4. From now on, React owns everything inside `#root`. You don't touch it manually.

Let's read the actual generated files.

#### `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + React</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

Surprise for newcomers from older tooling: **`index.html` lives at the project root, not in `public/`**, and it's the literal entry point Vite serves. The `<div id="root">` is the single mount point your whole app renders into. The `type="module"` script tells the browser to load `main.jsx` as an ES module — Vite handles transforming JSX on the fly.

#### `src/main.jsx`

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

- `createRoot(document.getElementById("root"))` grabs the `#root` div and creates a React root for it.
- `.render(...)` renders the element tree. Here that tree is `<App />` wrapped in `<StrictMode>`.
- Importing `./index.css` for its side effects pulls global styles into the bundle (Vite supports importing CSS directly from JS).

#### `src/App.jsx`

The starter version imports logos and uses `useState` for a counter. You already replaced it in Example 4. The key facts: it's a function component, it returns JSX, and it's the **default export** so `main.jsx` can `import App from "./App.jsx"`.

#### `package.json`

```json
{
  "name": "my-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "vite": "^6.0.0"
  }
}
```

- `"type": "module"` makes the whole project use ES modules (`import`/`export`) rather than CommonJS (`require`).
- `react` is the core library; `react-dom` is the renderer that talks to the browser DOM. They're separate because React can render to other targets (e.g., React Native renders to mobile). Both are runtime **dependencies**.
- `vite` and `@vitejs/plugin-react` are **devDependencies** — needed to build, not shipped to users.
- The three `scripts` are exactly the commands from the Examples section; `npm run <name>` executes them.

#### `vite.config.js`

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// https://vite.dev/config/
export default defineConfig({
  plugins: [react()],
});
```

`defineConfig` is just a helper that gives you editor autocomplete for the config object. The crucial line is `plugins: [react()]` — the React plugin teaches Vite how to transform JSX/`.jsx` files and wires up Fast Refresh (HMR for React components). You'll edit this file later when you add path aliases, proxies, etc.

### Why `<StrictMode>` double-invokes in development

This trips up *every* beginner the first time they add a `console.log` to a component and see it print **twice**.

`<StrictMode>` is a development-only wrapper that helps you catch unsafe patterns early. One of its checks: in **development only**, React **intentionally double-invokes** certain functions — component render functions, state initializers, and Effects (mount → unmount → remount) — to surface bugs caused by impure code or missing Effect cleanup.

Why double-invoke? React components are supposed to be **pure**: rendering the same inputs should produce the same output with no side effects during render. If your component mutates something it shouldn't (say, pushing into a shared array during render), running it twice makes the bug obvious *immediately* instead of in production months later. Similarly, mounting-unmounting-remounting an Effect verifies your cleanup logic actually works (Effects are covered in **see 08-useeffect-side-effects.md**).

Key facts to remember:

- The double-invoke happens **only in development**, and **only** when wrapped in `<StrictMode>`. Production builds run everything once.
- It is **not a bug** in your code or in React. Seeing two logs is the intended signal.
- The fix is never "remove StrictMode" — it's "make your render pure and your Effects properly cleaned up." Removing StrictMode just hides the warning while the underlying bug remains.

### Create React App is deprecated — use Vite

You will find countless tutorials using `npx create-react-app` (CRA). **CRA is deprecated.** The React team officially recommends, for a new SPA, either a build tool like **Vite** (or Parcel/Rsbuild) or a full framework like **Next.js** or **React Router**. CRA is slow to start, slow to rebuild, and no longer maintained. This course uses **Vite** throughout: it starts in milliseconds, has near-instant HMR, and is the de-facto modern standard. If a tutorial tells you to run `create-react-app`, mentally substitute the Vite command from this lesson.

### Reading errors in the console

Your two best friends are the browser **DevTools Console** (open with `F12` or `Cmd+Option+J` / `Ctrl+Shift+J`) and the **terminal** running `npm run dev`.

- **Red overlay in the browser** — Vite shows a full-screen error overlay for compile/syntax errors with the file, line, and a code frame. Read the top line; it usually names the exact problem (e.g., "Unexpected token").
- **Console errors at runtime** — React logs descriptive errors and warnings here. A click on the message expands a **component stack** showing which component threw — read it bottom-up to find your component.
- **"Each child in a list should have a unique key prop"** — a classic warning; covered in **see 05-rendering-lists-conditionals.md**.
- **Hooks rule errors** — e.g., "Rendered fewer hooks than expected"; covered when we reach hooks rules.
- **Terminal errors** — `npm run dev`/`npm run build` print module-resolution and build failures here (e.g., a missing import). Always glance at *both* the browser and the terminal when something breaks.

A quick habit: when you see a React error you don't recognize, copy the *first sentence* and search react.dev — React's error messages are unusually good and often link directly to docs.

---

## Common mistakes & pitfalls

**1. Lowercase component names**

```jsx
// WRONG — React treats <app /> as an unknown HTML tag, renders nothing useful
function app() {
  return <h1>Hi</h1>;
}
// ...later
root.render(<app />);
```

```jsx
// RIGHT — capitalized so React knows it's a component
function App() {
  return <h1>Hi</h1>;
}
root.render(<App />);
```

JSX uses capitalization to decide "your component" vs "HTML tag." Always capitalize component names.

**2. Forgetting `export default` (or mismatching import/export)**

```jsx
// WRONG — App.jsx defines App but never exports it
function App() {
  return <h1>Hi</h1>;
}
// main.jsx: import App from "./App.jsx"  →  App is undefined
```

```jsx
// RIGHT
function App() {
  return <h1>Hi</h1>;
}
export default App;
```

A `default` import (`import App from ...`) must pair with a `default` export. Named exports require named imports (`import { App } from ...`). Mismatches give "App is not defined" or a blank screen.

**3. Returning multiple sibling elements without a wrapper**

```jsx
// WRONG — a function can only return one value; this is a syntax error
function App() {
  return (
    <h1>Title</h1>
    <p>Body</p>
  );
}
```

```jsx
// RIGHT — wrap in one parent, or a Fragment (<>...</>) to add no extra DOM node
function App() {
  return (
    <>
      <h1>Title</h1>
      <p>Body</p>
    </>
  );
}
```

Fragments are detailed in **see 03-jsx-in-depth.md**.

**4. Using the old legacy render API**

```jsx
// WRONG — ReactDOM.render was deprecated in React 18 and removed in React 19
import ReactDOM from "react-dom";
ReactDOM.render(<App />, document.getElementById("root"));
```

```jsx
// RIGHT — React 18/19 uses createRoot from "react-dom/client"
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")).render(<App />);
```

**5. Editing the wrong file or forgetting to save**

If the page won't change, you may be editing a different copy of the project, or your editor hasn't saved. Confirm the dev server is running, that you edited `src/App.jsx` in *this* project, and that the file is saved. The terminal shows an `hmr update` line on each successful save.

**6. Running `cd` and the create command in one breath and getting confused**

```bash
# Vite scaffolds into ./my-app but does NOT cd for you.
npm create vite@latest my-app -- --template react
cd my-app          # you must do this yourself
npm install
npm run dev
```

Running `npm install` *before* `cd my-app` installs nothing useful and is a common first-day stumble.

**7. "Blank white page, no error"**

Usually a silent JS exception or a failed mount. Open the Console — there's almost always a red message. Confirm `index.html` has `<div id="root"></div>` and that `main.jsx` targets `"root"` (not `"app"` or a typo).

---

## Best practices

- **Do** use **Vite** (or a framework like Next.js) for new projects. **Don't** start anything new with Create React App — it's deprecated.
- **Do** keep `<StrictMode>` enabled in development. **Don't** delete it to silence double-logs; fix the underlying impurity instead.
- **Do** capitalize every component name and `export default` your top-level component.
- **Do** run on an LTS Node (20 or 22). **Don't** fight mystery errors on an ancient Node — upgrade first.
- **Do** keep both the **browser console** and the **terminal** visible while developing; errors appear in one or the other.
- **Do** ship the contents of `dist/` (from `npm run build`) to production. **Don't** deploy the dev server.
- **Do** commit a `.gitignore` that excludes `node_modules/` and `dist/` (Vite's template includes one). **Don't** commit `node_modules/`.
- **Do** treat components as pure functions of their inputs — no side effects during render. This pays off across the entire rest of the course.
- **Don't** reach for `document.querySelector`/manual DOM edits inside React. Describe UI declaratively and let React do the DOM work.

---

## Exercises

Try each before reading the solution.

**Exercise 1 — Scaffold and brand it.**
Create a new Vite React project called `my-first-app`, run the dev server, and change the browser tab title to `My First React App`. (Hint: the title lives in `index.html`, not in a component.)

**Exercise 2 — Your own component.**
In `src/App.jsx`, delete all starter content and render a `<main>` containing an `<h1>` that greets you by name and a `<p>` with today's date computed in JavaScript.

**Exercise 3 — Compose and reuse.**
Create a `Card` component that takes a `title` and `body` prop and renders them inside a `<section>`. Render three different `Card`s from `App`.

**Exercise 4 — Observe StrictMode.**
Add `console.log("App rendered")` at the top of your `App` function body. Note how many times it logs in dev. Then temporarily remove `<StrictMode>` in `main.jsx`, refresh, and count again. Explain the difference. (Put `<StrictMode>` back when done.)

**Exercise 5 (challenge) — Read an error on purpose.**
Deliberately break your app: return two sibling elements without a wrapper. Read the error in the terminal and browser overlay, then fix it with a Fragment. Write one sentence describing what the error said.

### Solutions

**Solution 1**

```bash
npm create vite@latest my-first-app -- --template react
cd my-first-app
npm install
npm run dev
```

Then edit `index.html`:

```html
<title>My First React App</title>
```

**Solution 2**

```jsx
// src/App.jsx
function App() {
  const today = new Date().toLocaleDateString();
  return (
    <main>
      <h1>Hello, Sam!</h1>
      <p>Today is {today}.</p>
    </main>
  );
}

export default App;
```

**Solution 3**

```jsx
// src/App.jsx
function Card({ title, body }) {
  return (
    <section>
      <h2>{title}</h2>
      <p>{body}</p>
    </section>
  );
}

function App() {
  return (
    <main>
      <Card title="React" body="A library for building UIs." />
      <Card title="Vite" body="A fast build tool and dev server." />
      <Card title="Components" body="Reusable pieces of UI." />
    </main>
  );
}

export default App;
```

**Solution 4**

```jsx
// src/App.jsx
function App() {
  console.log("App rendered");
  return <h1>Check the console</h1>;
}

export default App;
```

With `<StrictMode>` wrapping `<App />`, you'll see **two** `App rendered` logs in development — React intentionally double-invokes the render to surface impurity. Remove `<StrictMode>` and you'll see **one**. In a production build it would log once regardless. Lesson: the double log is expected dev behavior, not a bug. Keep `<StrictMode>` on.

**Solution 5**

```jsx
// The broken version throws a syntax error like:
//   "Adjacent JSX elements must be wrapped in an enclosing tag.
//    Did you want a JSX fragment <>...</>?"
// Vite shows this in the terminal and as a red browser overlay.

// Fixed:
function App() {
  return (
    <>
      <h1>Title</h1>
      <p>Body</p>
    </>
  );
}

export default App;
```

One-sentence answer: the error said adjacent JSX elements must be wrapped in an enclosing tag, and suggested a Fragment — because a function (and thus JSX) can only return a single root element.

---

## Key takeaways

- **React is a JavaScript library for building UIs from reusable components**, solving the problem of keeping the screen in sync with changing data.
- The core shift is **declarative over imperative**: you describe what the UI *is* for a given state; React figures out the DOM updates.
- React uses a **Virtual DOM** and **reconciliation** to diff snapshots and apply only the minimal real-DOM changes — efficiently, on your behalf.
- A **component** is a capitalized function returning JSX; you compose big UIs by nesting small components.
- Scaffold with `npm create vite@latest my-app -- --template react`; develop with `npm run dev`, ship with `npm run build`, verify with `npm run preview`.
- The render chain is `index.html` → `main.jsx` (`createRoot(...).render(<App/>)`) → your component tree mounted into `<div id="root">`.
- `<StrictMode>` is a **dev-only** helper that intentionally **double-invokes** renders/effects to expose impurity and missing cleanup — never strip it to hide the symptom.
- **Create React App is deprecated**; use Vite (or a framework). Read errors in both the **browser console** and the **terminal**.

---

## Further reading

- [react.dev — Quick Start](https://react.dev/learn) — the official tutorial that pairs perfectly with this lesson.
- [react.dev — Start a New React Project](https://react.dev/learn/start-a-new-react-project) — official guidance recommending Vite/frameworks over CRA.
- [react.dev — `createRoot`](https://react.dev/reference/react-dom/client/createRoot) — full reference for the root API.
- [react.dev — `<StrictMode>`](https://react.dev/reference/react/StrictMode) — exactly what it checks and why it double-invokes.
- [react.dev — Keeping Components Pure](https://react.dev/learn/keeping-components-pure) — the purity idea StrictMode enforces.
- [react.dev — Render and Commit](https://react.dev/learn/render-and-commit) — the render → reconcile → commit pipeline at a deeper level.
- [Vite — Getting Started](https://vite.dev/guide/) and [Vite — Why Vite](https://vite.dev/guide/why) — the build tool we use throughout.
- [Node.js downloads](https://nodejs.org/en/download) and [nvm](https://github.com/nvm-sh/nvm) — installing and managing Node versions.
