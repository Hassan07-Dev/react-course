# 01 — JavaScript Foundations for React (ES6+)

> **What you'll learn:** the exact slice of modern JavaScript that React (v19) leans on every day, and *why* each feature exists in idiomatic React code — so the rest of the course teaches you React, not JS surprises.

By the end of this lesson you will be able to:

- Use `let`/`const`, arrow functions, template literals, destructuring, spread/rest, and default params fluently.
- Read and write ES module `import`/`export` the way a Vite + React project expects.
- Transform arrays with `map`/`filter`/`reduce`/`find`/`some`/`every`/`includes` to produce UI lists.
- Render conditionally with ternaries and `&&`/`||` short-circuiting, and guard data with `?.` and `??`.
- Explain why React needs *new references* for state, and copy objects/arrays immutably (shallow vs deep).
- Reason about reference equality (`===`, `Object.is`), truthiness pitfalls (`0`, `""`, `NaN`), and closures.
- Use Promises, `async`/`await`, and `fetch` + JSON, and describe the event loop at a high level.

---

## Prerequisites

This is the first lesson, so the only prerequisite is **basic programming literacy**: you know what a variable, function, object, array, and loop are in *some* language. You do **not** need prior React. You should have **Node.js 18+** and **npm** installed; we use a Vite project structure throughout the course (covered in `02-getting-started.md` and the project-setup lesson). Everything here is plain JavaScript you can paste into a browser console or a `.js`/`.jsx` file.

If a concept here feels shaky, slow down — every later lesson assumes this material. Components are just functions; props are just objects; rendering a list is just `array.map`. React is "JavaScript you already know, arranged a certain way."

---

## Concept

React has remarkably little proprietary syntax. A component is a **plain JavaScript function that returns markup**. State updates, list rendering, conditional UI, data fetching — all of it is expressed with ordinary ES6+ JavaScript. So the single biggest predictor of whether React "clicks" for you is your comfort with modern JS.

Three ideas dominate everything React does:

1. **Expressions over statements.** JSX (lesson `03-jsx-in-depth.md`) only lets you embed *expressions* in `{ }` — things that produce a value. That's why React code is full of ternaries, `&&`, and `.map()` instead of `if` statements and `for` loops. You think in "what value does this produce?"
2. **Immutability and references.** React decides what to re-render by comparing the *identity* (reference) of values, not their contents. Mutating an object in place is invisible to React; creating a new object is the signal. Half of this lesson is really about teaching your hands to create new values instead of editing old ones.
3. **Closures.** Hooks (`useState`, `useEffect`, etc., from lesson `07-state-usestate.md` onward) rely entirely on closures — functions remembering the variables that were in scope when they were created. If closures are fuzzy, hooks will feel like magic; once they click, hooks feel inevitable.

Keep those three threads in mind. Every feature below serves one of them.

---

## Syntax & API

A fast reference. Each item is expanded with examples in the next section.

```js
// Block-scoped bindings
let count = 0;        // reassignable
const user = {};      // not reassignable (the binding; the object can still mutate)

// Arrow functions
const add = (a, b) => a + b;          // implicit return
const sq  = n => n * n;               // single param, no parens needed
const obj = () => ({ ok: true });     // return an object literal: wrap in ()

// Template literals
const msg = `Hello, ${user.name}! 1 + 1 = ${1 + 1}`;

// Destructuring
const { name, age = 18 } = user;            // object, with default
const [first, , third] = [10, 20, 30];      // array, skipping holes
const { profile: { city } } = user;         // nested

// Spread / rest
const copy   = { ...user, active: true };   // spread into new object
const merged = [...a, ...b];                 // spread into new array
const fn = (first, ...rest) => rest;         // rest gathers extra args

// Default parameters
function greet(name = "friend") { return `Hi ${name}`; }

// ES modules
import React from "react";                   // default import
import { useState, useEffect } from "react"; // named imports
export default function App() {}             // default export
export const VERSION = "1.0";                // named export

// Array methods that build UI
list.map(fn); list.filter(fn); list.reduce(fn, init);
list.find(fn); list.some(fn); list.every(fn); list.includes(value);

// Conditionals as expressions
cond ? a : b;            // ternary
isLoggedIn && <Panel />; // render Panel only when truthy
name || "Anonymous";     // fallback when falsy

// Safe access
user?.profile?.city;     // optional chaining: undefined instead of throwing
value ?? "default";      // nullish coalescing: fallback only for null/undefined

// Equality
a === b;                 // strict equality (no coercion)
Object.is(a, b);         // like === but treats NaN===NaN and -0 !== +0

// Async
const p = fetch(url);                 // returns a Promise
const res = await fetch(url);         // inside an async function
const data = await res.json();        // parse JSON body (also a Promise)
```

---

## Examples

### 1. `let`/`const` and block scope

```js
const MAX = 100;          // never reassign → const (the default choice)
let total = 0;            // will be reassigned → let
for (let i = 0; i < 3; i++) { /* i is scoped to this loop */ }
// console.log(i);        // ReferenceError — i does not leak out
```

`const` and `let` are **block-scoped** (live only inside the nearest `{ }`), unlike the old `var` which was function-scoped and hoisted in confusing ways. Note that `const` freezes the *binding*, not the value: `const arr = []; arr.push(1)` is legal, but `arr = [1]` is not.

> **Why this matters in React:** Default to `const` for props, state values, and derived data — it signals "this won't be reassigned in this render," which is almost always true. The `let i` block-scoping fix is also what makes loops behave correctly with closures (see Example 9), the classic interview bug that used to plague event handlers.

### 2. Arrow functions, implicit return, lexical `this`

```js
const double = n => n * 2;                 // implicit return
const point  = () => ({ x: 0, y: 0 });     // return object literal → wrap in ()
const nums   = [1, 2, 3].map(n => n * 2);  // [2, 4, 6]
```

Arrow functions have two superpowers React loves: a terse **implicit return** (no `return` keyword when the body is a single expression), and **lexical `this`** — they do not create their own `this`, they inherit it from the enclosing scope.

> **Why this matters in React:** Inline callbacks like `onClick={() => setOpen(true)}` and list mappers `items.map(item => <Row {...item} />)` are arrow functions. Because they're expressions returning a value, they slot straight into JSX.

### 3. Template literals

```js
const name = "Ada";
const greeting = `Hello, ${name}`;                 // interpolation
const cls = `btn ${active ? "btn--on" : ""}`;      // dynamic className
const multi = `line one
line two`;                                          // real newlines
```

Backtick strings interpolate any expression inside `${ }` and span multiple lines.

> **Why this matters in React:** Building dynamic `className` strings, image `src` URLs, and `aria-label` text is constant. `` `card ${isSelected ? "card--active" : ""}` `` is everywhere.

### 4. Destructuring (object, array, nested, params)

```js
const user = { name: "Ada", role: "admin", profile: { city: "London" } };

const { name, role } = user;                     // pull fields out
const { profile: { city } } = user;              // nested
const { theme = "light" } = user;                // default if missing

const [first, second] = ["a", "b"];              // array by position
const [head, ...tail] = [1, 2, 3];               // head=1, tail=[2,3]

// Destructuring in parameters (the React component signature!)
function Avatar({ src, alt = "avatar", size = 40 }) {
  return { src, alt, size };
}
```

> **Why this matters in React:** Every function component destructures its props in the parameter list: `function Profile({ user, onSave }) { ... }`. Hooks return arrays you destructure by position: `const [count, setCount] = useState(0)` (lesson `07-state-usestate.md`). The *order* matters for arrays, the *name* matters for objects.

### 5. Spread, rest, and default parameters

```js
const base   = { color: "red", size: "M" };
const variant = { ...base, size: "L" };          // {color:"red", size:"L"}  (later wins)

const a = [1, 2], b = [3, 4];
const all = [...a, ...b];                         // [1,2,3,4]

function sum(...nums) {                           // rest: gather args into array
  return nums.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3);                                     // 6

function paginate(page = 1, perPage = 20) {       // defaults
  return { page, perPage };
}
```

Spread (`...`) *expands* a collection; rest (`...` in a parameter list or destructuring) *gathers* values. Same syntax, opposite directions.

> **Why this matters in React:** Spread props forward attributes: `<input {...inputProps} />`. Immutable state updates rely on spread: `setUser({ ...user, name: "New" })` and `setItems([...items, newItem])` create **new** references so React notices the change (see Under the hood).

### 6. ES modules

```js
// math.js
export const PI = 3.14159;          // named export
export function area(r) { return PI * r * r; }
export default function Circle() {} // one default export per file

// app.js
import Circle, { PI, area } from "./math.js";   // default + named
import { useState as useStateAlias } from "react"; // rename on import
import * as Math2 from "./math.js";              // namespace import
```

Each file is a module with its own scope. `export` exposes bindings; `import` pulls them in. A file may have **many named exports** but **at most one default export**.

> **Why this matters in React:** Components live in their own files. The conventional pattern is `export default function Button() {}` and `import Button from "./Button.jsx"`. `import { useState } from "react"` is a named import of a hook. Vite resolves these at dev time and bundles them for production.

### 7. Array methods that build UI

```js
const users = [
  { id: 1, name: "Ada",  active: true,  age: 36 },
  { id: 2, name: "Linus", active: false, age: 54 },
  { id: 3, name: "Grace", active: true,  age: 41 },
];

users.map(u => u.name);                       // ["Ada","Linus","Grace"] → list rows
users.filter(u => u.active);                  // only active users
users.find(u => u.id === 2);                  // first match or undefined
users.some(u => u.active);                    // true (at least one)
users.every(u => u.active);                   // false (not all)
users.map(u => u.name).includes("Ada");       // true
users.reduce((total, u) => total + u.age, 0); // 131 (sum of ages)
```

Each of these is **pure**: it returns a new value and does not mutate the source array. `map` transforms 1→1, `filter` keeps a subset, `reduce` folds many into one, `find` returns one element (or `undefined`), `some`/`every`/`includes` return booleans.

> **Why this matters in React:** Rendering a list is literally `{users.map(u => <li key={u.id}>{u.name}</li>)}` (lesson `05-rendering-lists-conditionals.md`). `filter` powers search/visibility; `reduce` computes totals shown in the UI; `some`/`every` drive "select all" checkboxes. You almost never write a `for` loop in JSX because `for` is a statement and JSX needs expressions.

### 8. Conditionals, optional chaining, nullish coalescing

```js
const user = null;

// Ternary — choose between two expressions
const label = user ? "Logout" : "Login";

// && short-circuit — render only when left side is truthy
const banner = user && `Welcome ${user.name}`;   // null short-circuits

// || fallback — first truthy value (careful: 0 and "" are falsy!)
const display = user?.name || "Guest";

// ?. optional chaining — stop at null/undefined instead of throwing
const city = user?.profile?.city;                 // undefined, no crash

// ?? nullish coalescing — fallback ONLY for null/undefined
const count = user?.cartCount ?? 0;               // keeps a real 0
```

> **Why this matters in React:** `{isLoading && <Spinner />}` and `{error ? <Error/> : <Data/>}` are the two workhorse conditional-render patterns (lesson `05-rendering-lists-conditionals.md`). `?.` safely reads possibly-missing API data; `??` gives defaults without clobbering legitimate `0`/`""` values — a trap `||` falls into.

### 9. Closures (the heart of hooks)

```js
function makeCounter() {
  let count = 0;                  // captured variable
  return function increment() {   // closure: remembers `count`
    count += 1;
    return count;
  };
}
const next = makeCounter();
next(); // 1
next(); // 2  — the inner function "remembers" count between calls
```

A **closure** is a function bundled with the variables that were in scope when it was created. The inner `increment` keeps `count` alive even after `makeCounter` returned.

> **Why this matters in React:** Every render of a component creates fresh closures over that render's props and state. An event handler "sees" the values from the render it was created in — which is exactly why stale-closure bugs happen in `useEffect`/`useCallback` and why dependency arrays exist (lessons `08-useeffect-side-effects.md`, `09-hooks-deep-dive.md`). Understanding closures now will save you hours later.

### 10. Promises, async/await, fetch + JSON

```js
// Promise-based
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then(res => {
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();          // .json() also returns a Promise
    });
}

// async/await — same thing, flatter
async function getUserAsync(id) {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return await res.json();
}
```

A **Promise** represents a value that will exist later (or fail later). `async`/`await` is syntactic sugar over Promises: `await` pauses the function until the Promise settles, letting you write asynchronous code that *reads* synchronously. `fetch` returns a Promise of a `Response`; calling `.json()` returns another Promise for the parsed body.

> **Why this matters in React:** Data fetching inside `useEffect` (lesson `08-useeffect-side-effects.md`) uses exactly this. A common pattern is an `async` helper *inside* the effect, because the effect callback itself must not be `async` (it would return a Promise where React expects a cleanup function). Modern React also offers `use()` and Suspense for data (later lessons), but `fetch` + `await` is the foundation.

---

## Under the hood

**References vs values.** Primitives (`number`, `string`, `boolean`, `null`, `undefined`, `symbol`, `bigint`) are compared by value. Objects and arrays are compared by **reference** — two different objects with identical contents are not `===`:

```js
const a1 = { a: 1 };
const a2 = { a: 1 };
a1 === a2               // false — different objects in memory
const o = { a: 1 };
o === o                 // true — same reference
```

React's re-render decisions ride on this. When you call a state setter, React compares the *new* value to the *previous* one with `Object.is`. If you mutate state in place, the reference is unchanged, `Object.is(prev, next)` is `true`, and **React skips the update — your UI goes stale**:

```js
// ❌ Mutation: same reference → React sees "no change"
user.name = "New";
setUser(user);                 // ignored

// ✅ New reference → React re-renders
setUser({ ...user, name: "New" });
```

This is *the* reason immutability is non-negotiable in React. The same logic governs `React.memo`, `useMemo`, and `useEffect` dependency comparisons (lesson `13-performance.md`): they all compare references with `Object.is`.

**`Object.is` vs `===`.** They agree except in two cases: `Object.is(NaN, NaN)` is `true` (but `NaN === NaN` is `false`), and `Object.is(-0, +0)` is `false` (but `-0 === +0` is `true`). React uses `Object.is` for its comparisons.

**Shallow vs deep copy.** Spread (`{...obj}`, `[...arr]`) and `Object.assign` make a **shallow** copy — top level is new, but nested objects are *shared by reference*:

```js
const state = { user: { name: "Ada" }, tags: ["x"] };
const copy = { ...state };
copy.user === state.user;       // true! nested object is shared
copy.user.name = "Mut";         // ⚠️ mutates the original too
```

To update something nested immutably, copy at *every level you touch*:

```js
const next = { ...state, user: { ...state.user, name: "New" } };
```

For genuinely deep clones of plain data, modern environments offer `structuredClone(state)` — but you rarely need it in React; targeted shallow copies are the idiom (and tools like Immer exist for deep update ergonomics).

**The event loop (high level).** JavaScript runs on a single thread with a **call stack**. Long-running async work (`fetch`, timers) is handed off to the host (browser/Node). When it completes, its callback is queued. Promise callbacks go on the **microtask queue** (drained fully after each task, before rendering); `setTimeout` callbacks go on the **macrotask queue**. The loop: run a task to completion → drain all microtasks → (browser may paint) → next task.

```js
console.log("A");
setTimeout(() => console.log("B"), 0);   // macrotask
Promise.resolve().then(() => console.log("C")); // microtask
console.log("D");
// Logs: A, D, C, B  — sync first, then microtasks, then the timer
```

> Knowing microtasks drain before paint explains why `await`-ed state updates can batch, and why a `setTimeout(…, 0)` "after paint" trick differs from a Promise `.then`.

**`this` overview.** In a regular function, `this` depends on *how the function is called* (the object before the dot, or `undefined`/global if called bare). Arrow functions ignore all that and capture `this` from their defining scope. **In modern function-component React you almost never write `this`** — it mattered for old class components (lesson `21-class-components-legacy.md`). The practical rule: prefer arrow functions for callbacks and you sidestep `this` entirely.

---

## Common mistakes & pitfalls

**1. Mutating state instead of replacing it**

```js
// ❌ WRONG — mutates, React won't re-render
const handleAdd = () => { items.push(newItem); setItems(items); };

// ✅ RIGHT — new array reference
const handleAdd = () => setItems([...items, newItem]);
```

*Why:* React compares references with `Object.is`. The mutated array is the same reference, so React bails out.

**2. `||` eating a legitimate `0` or `""`**

```js
// ❌ WRONG — count of 0 becomes 5
const limit = props.count || 5;

// ✅ RIGHT — only null/undefined fall through
const limit = props.count ?? 5;
```

*Why:* `0`, `""`, `NaN`, `false`, `null`, `undefined` are all falsy, so `||` rejects valid zeros and empty strings. `??` only rejects `null`/`undefined`.

**3. Rendering a falsy number with `&&`**

```js
// ❌ WRONG — when items.length is 0, React renders the literal "0"
return <div>{items.length && <List items={items} />}</div>;

// ✅ RIGHT — force a boolean
return <div>{items.length > 0 && <List items={items} />}</div>;
```

*Why:* `0 && X` evaluates to `0`, and React renders `0` as visible text (it skips `false`/`null`/`undefined` but *not* `0`). Covered again in `05-rendering-lists-conditionals.md`.

**4. Forgetting `()` when an arrow returns an object**

```js
// ❌ WRONG — {} is read as a function body, returns undefined
const make = () => { id: 1 };

// ✅ RIGHT — wrap the object literal in parentheses
const make = () => ({ id: 1 });
```

**5. Shallow copy + nested mutation**

```js
// ❌ WRONG — nested object still shared, original mutated
const next = { ...state };
next.user.name = "X";

// ✅ RIGHT — copy each touched level
const next = { ...state, user: { ...state.user, name: "X" } };
```

**6. Making the `useEffect` callback `async`**

```js
// ❌ WRONG — async returns a Promise; React expects a cleanup fn
useEffect(async () => { const d = await fetch(url); }, []);

// ✅ RIGHT — declare an inner async function and call it
useEffect(() => {
  async function load() { const d = await fetch(url); /* setState(d) */ }
  load();
}, []);
```

*Why:* See lesson `08-useeffect-side-effects.md` — the effect's return value is reserved for cleanup.

---

## Best practices

- **Default to `const`**; reach for `let` only when you truly reassign. Never use `var`.
- **Treat state as read-only.** Always produce a *new* object/array (`{...x}`, `[...x]`, `.map`, `.filter`) — never `push`, `splice`, `sort`, or direct assignment on state.
- **Use `??` for defaults, `||` only when you genuinely want any falsy value replaced.**
- **Use `?.` to read uncertain data**, but don't sprinkle it everywhere — if a value should always exist, let it throw so bugs surface.
- **Prefer array methods (`map`/`filter`/`reduce`) over loops** in render logic; they're expressions and they don't mutate.
- **Force booleans before `&&`** (`list.length > 0 &&`) to avoid rendering stray `0`s.
- **Destructure props in the parameter list** and provide defaults there.
- **Keep effect callbacks synchronous**; put `await` inside an inner async function.
- **Copy nested data at every level you change**; remember spread is shallow.
- **Prefer arrow functions for callbacks** to avoid `this` entirely.

---

## Exercises

**Exercise 1 — Immutable updates.** Given `const state = { count: 0, tags: ["a"] };`, write expressions that (a) increment `count`, and (b) append `"b"` to `tags`, each returning a brand-new object without mutating `state`.

**Exercise 2 — List pipeline.** Given an array of products `[{ id, name, price, inStock }]`, produce an array of the *names* of in-stock products priced under 50, and separately compute the total price of all in-stock products. Use `filter`, `map`, and `reduce`.

**Exercise 3 — Safe access with defaults.** Given a possibly-incomplete `order` object, read `order.customer.address.city` without throwing, defaulting to `"Unknown"`, and read `order.discount` defaulting to `0` *even when the discount is legitimately `0`*.

**Exercise 4 — Closure counter.** Write a `makeAdder(step)` factory that returns a function which, each time it's called, adds `step` to a private running total and returns the new total. Demonstrate two independent counters.

**Exercise 5 — Async fetch.** Write an `async` function `loadTitles(url)` that fetches a JSON array of `{ title }` objects, throws on a non-OK response, and returns an array of just the titles. Handle the error at the call site with `try/catch`.

### Solutions

```js
// Exercise 1
const state = { count: 0, tags: ["a"] };
const incremented = { ...state, count: state.count + 1 };      // {count:1, tags:["a"]}
const tagged      = { ...state, tags: [...state.tags, "b"] };  // tags: ["a","b"]
// state itself is untouched.

// Exercise 2
const products = [
  { id: 1, name: "Pen",   price: 3,  inStock: true },
  { id: 2, name: "Desk",  price: 90, inStock: true },
  { id: 3, name: "Pad",   price: 12, inStock: false },
];
const cheapInStockNames = products
  .filter(p => p.inStock && p.price < 50)
  .map(p => p.name);                                  // ["Pen"]
const totalInStock = products
  .filter(p => p.inStock)
  .reduce((sum, p) => sum + p.price, 0);              // 93

// Exercise 3
const order = { discount: 0 };                        // no customer/address!
const city = order?.customer?.address?.city ?? "Unknown"; // "Unknown"
const discount = order?.discount ?? 0;                // 0 (kept, not overridden)

// Exercise 4
function makeAdder(step) {
  let total = 0;                       // private, captured by closure
  return () => (total += step);
}
const byTwo = makeAdder(2);
const byTen = makeAdder(10);
byTwo(); // 2
byTwo(); // 4
byTen(); // 10 — independent state

// Exercise 5
async function loadTitles(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const items = await res.json();      // expects [{ title }, ...]
  return items.map(item => item.title);
}

// Call site
try {
  const titles = await loadTitles("/api/posts");
  console.log(titles);
} catch (err) {
  console.error("Failed to load:", err.message);
}
```

---

## Key takeaways

- React adds little syntax of its own; mastering ES6+ JavaScript *is* most of the battle.
- Default to `const`; arrow functions give implicit returns and lexical `this`; template literals build dynamic strings.
- Destructuring (especially in parameters) and spread/rest are the backbone of props handling and immutable updates.
- `map`/`filter`/`reduce`/`find`/`some`/`every`/`includes` turn data into UI — they're pure and return new values.
- Use `?.` to read uncertain data and `??` (not `||`) for defaults so real `0`/`""` survive; force booleans before `&&`.
- React re-renders by comparing **references** with `Object.is`, so always create new objects/arrays for state; spread is shallow — copy every nested level you touch.
- Closures (functions remembering their creation-time scope) underpin every hook and explain stale-value bugs.
- Promises + `async`/`await` + `fetch`/`.json()` power data fetching; the event loop runs sync code, then microtasks, then macrotasks.

---

## Further reading

- React docs — [JavaScript in JSX with Curly Braces](https://react.dev/learn/javascript-in-jsx-with-curly-braces)
- React docs — [Updating Objects in State](https://react.dev/learn/updating-objects-in-state) and [Updating Arrays in State](https://react.dev/learn/updating-arrays-in-state)
- React docs — [Rendering Lists](https://react.dev/learn/rendering-lists) and [Conditional Rendering](https://react.dev/learn/conditional-rendering)
- MDN — [Destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring), [Spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax), [Optional chaining `?.`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining), [Nullish coalescing `??`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)
- MDN — [Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures), [`Object.is()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is), [`structuredClone()`](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)
- MDN — [Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises), [`fetch()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch)
- [JavaScript.info](https://javascript.info/) — thorough, beginner-friendly deep dives on every topic above.

Next: **`03-jsx-in-depth.md`** — how this JavaScript turns into UI with JSX.
