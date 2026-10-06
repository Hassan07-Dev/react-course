# 04 — Components & Props

> **What you'll learn:** How to define React function components, compose them into a tree, and pass data down to them with props — the two ideas that the entire rest of React is built on.

By the end of this lesson you will be able to:

- Define a **function component** and explain the PascalCase naming rule and why React enforces it.
- Compose components out of other components and reason about the resulting **component tree**.
- Pass and read **props**, including strings, expressions, objects, functions, and JSX.
- Use **destructuring**, **default values**, and the special **`children`** prop.
- Apply (and avoid abusing) the **spread operator** `{...obj}` for forwarding props.
- Explain **one-way data flow** and why props are **read-only** (component purity).
- Distinguish a **component**, an **element**, and an **instance** — a distinction most beginners never learn cleanly.
- Add runtime validation with **PropTypes**, and know when to reach for TypeScript instead.

---

## Prerequisites

You should already be comfortable with:

- **JSX** — what it is, embedding `{expressions}`, attributes, fragments, and the fact that JSX compiles to function calls (see `03-jsx-in-depth.md`).
- **Project setup** — running a Vite + React 19 app with `npm run dev` (see `02-getting-started.md`).
- **Modern JavaScript** — `import`/`export`, arrow functions, object/array destructuring, the spread operator, and template literals. If any of those feel shaky, skim `01-js-foundations.md` first.

This lesson deliberately uses **no state and no effects**. We render static UI from data passed in. State arrives in `07-state-usestate.md`; side effects in `08-useeffect-side-effects.md`.

---

## Concept

### A component is just a function that returns UI

In React, the unit of UI is the **component**. A function component is a plain JavaScript function that returns a description of what should appear on screen (JSX). That's the whole idea:

```jsx
function Welcome() {
  return <h1>Hello, world</h1>;
}
```

Think of a component as a **stamp**. The function is the rubber stamp; every time React "presses" it, you get one piece of UI. Press it three times and you get three identical stamps on the page. The stamp itself is reusable; each impression is independent.

But a stamp that always prints the same thing is boring. We want a stamp that can print *"Hello, **Ada**"* and *"Hello, **Lin**"* from the same carving. The carving is the component; the word we slot in each time is a **prop**.

### Props are the function's arguments

If a component is a function, **props are its parameters**. The parent decides what values to pass; the child reads them and renders accordingly. Crucially, the data flows **one way**: from parent down to child. A child can never reach up and change what its parent gave it.

```jsx
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}

// Parent passes data down:
<Welcome name="Ada" />
<Welcome name="Lin" />
```

This is the analogy that makes React click: **the UI is a tree of function calls, and props are the arguments threaded down that tree.** Your whole app is one big expression: `App()` returns elements that mention `Header`, `Sidebar`, `Content`, each of which returns more elements mentioning smaller components, all the way down to `<h1>` and `<button>`.

### Why one-way and read-only?

React deliberately makes props flow one direction and forbids a component from mutating them. The payoff is **predictability**: to understand what any component renders, you only need to know the props it received. You never have to hunt for some distant child that secretly rewrote them. This property is called **purity** — same inputs, same output, no surprises — and it's what lets React safely re-render, batch, and (in React 19's compiler era) optimize your tree. We'll lean on purity heavily in later lessons.

---

## Syntax & API

### Defining a component

```jsx
// PascalCase name, returns JSX (or null).
function Greeting() {
  return <p>Hi there</p>;
}

// Arrow-function form is equally valid:
const Greeting = () => <p>Hi there</p>;
```

Rules and notes:

- **Name must start with a capital letter** (`Greeting`, not `greeting`). JSX treats lowercase names as built-in HTML tags (`<div>`, `<span>`) and capitalized names as your components. `<greeting />` would render a literal — and invalid — DOM element; `<Greeting />` calls your function. This is a hard requirement, not a style choice.
- A component must return a **single root** — one element, a [Fragment](https://react.dev/reference/react/Fragment) (`<>...</>`), an array, a string, a number, or `null` (render nothing). It cannot return two sibling elements without a wrapper.
- A function is "a component" only by virtue of how it's used (rendered as `<Thing />`). The same function called as `Thing()` is just a function call returning an element.

### Exporting components

```jsx
// Greeting.jsx — default export (one main component per file is common)
export default function Greeting() {
  return <p>Hi</p>;
}

// Named exports (multiple components per file)
export function Avatar() { /* ... */ }
export function Badge() { /* ... */ }
```

```jsx
// Importing
import Greeting from './Greeting.jsx';      // default
import { Avatar, Badge } from './ui.jsx';   // named
```

### Reading props

```jsx
// 1. Single props object
function Card(props) {
  return <h2>{props.title}</h2>;
}

// 2. Destructure in the signature (idiomatic)
function Card({ title }) {
  return <h2>{title}</h2>;
}

// 3. Destructure with default values
function Card({ title = 'Untitled', elevated = false }) {
  return <h2 className={elevated ? 'shadow' : ''}>{title}</h2>;
}

// 4. Capture the rest with a rest element
function Card({ title, ...rest }) {
  // `rest` is an object of every other prop passed in
}
```

Notes:

- `props` is always a **plain object**. React builds it from the JSX attributes.
- Defaults via destructuring (`{ title = 'Untitled' }`) are the modern way. The legacy `Component.defaultProps` static has been **removed for function components in React 19** (it was deprecated with a warning in 18.3) — React now ignores it on function components, so you **must** use destructuring defaults. (`defaultProps` still works on class components, which have no ES6-default equivalent.)
- The special prop **`key`** is *not* delivered in `props` — React consumes it itself; see `05-rendering-lists-conditionals.md`. (Historically `ref` was also withheld and required `forwardRef`. **As of React 19, `ref` is a regular prop**: a function component can destructure `{ ref }` directly and `forwardRef` is no longer needed — it's deprecated. We cover refs in `09-hooks-deep-dive.md`.)

### Passing different value types

```jsx
<Card title="Static string" />               {/* string literal: quotes */}
<Card title={user.name} />                    {/* expression: braces */}
<Card count={3} active={true} />              {/* number, boolean */}
<Card active />                               {/* shorthand for active={true} */}
<Card style={{ padding: 8 }} />              {/* object: double braces */}
<Card onSave={handleSave} />                  {/* function reference */}
<Card icon={<StarIcon />} />                  {/* JSX as a prop */}
<Card tags={['a', 'b']} />                    {/* array */}
```

### The `children` prop

```jsx
function Panel({ children }) {
  return <section className="panel">{children}</section>;
}

// Whatever you nest between the tags becomes `children`:
<Panel>
  <h2>Title</h2>
  <p>Body text</p>
</Panel>
```

### Spreading props

```jsx
const props = { type: 'text', placeholder: 'Email', maxLength: 80 };
<input {...props} />                       // forward an object as individual props
<Child {...rest} extra="x" />              // later keys override earlier ones
```

---

## Examples

### Example 1 — A reusable component with one prop

```jsx
// src/Greeting.jsx
export default function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}
```

```jsx
// src/App.jsx
import Greeting from './Greeting.jsx';

export default function App() {
  return (
    <main>
      <Greeting name="Ada" />
      <Greeting name="Lin" />
      <Greeting name="Grace" />
    </main>
  );
}
```

One component definition, three independent renders. Each `<Greeting />` gets its own `props` object. The parent (`App`) is fully in control of the `name` values; `Greeting` just renders what it's handed. This is the entire reuse story of React in eight lines.

### Example 2 — Multiple props, defaults, and an expression prop

```jsx
// src/UserBadge.jsx
export default function UserBadge({ name, role = 'Member', online = false }) {
  return (
    <div className="badge">
      <span className={online ? 'dot dot--green' : 'dot dot--gray'} />
      <strong>{name}</strong>
      <em>{role}</em>
    </div>
  );
}
```

```jsx
// src/App.jsx
import UserBadge from './UserBadge.jsx';

const currentUser = { name: 'Ada Lovelace', role: 'Admin' };

export default function App() {
  return (
    <>
      <UserBadge name="Lin" />                          {/* role defaults to "Member" */}
      <UserBadge name={currentUser.name} role={currentUser.role} online />
    </>
  );
}
```

Things to notice:

- `role = 'Member'` supplies a **default** when the parent omits the prop.
- `online` (with no value) is shorthand for `online={true}` — a boolean prop.
- `name={currentUser.name}` is an **expression** prop: braces, not quotes. Writing `name="currentUser.name"` would pass the literal string `"currentUser.name"`.

### Example 3 — Composition with `children`: a `Card` with header and body

This is the canonical pattern: a generic container that doesn't know or care what it wraps.

```jsx
// src/Card.jsx
export default function Card({ title, footer, children }) {
  return (
    <div className="card">
      {title && <header className="card__header">{title}</header>}
      <div className="card__body">{children}</div>
      {footer && <footer className="card__footer">{footer}</footer>}
    </div>
  );
}
```

```jsx
// src/App.jsx
import Card from './Card.jsx';

export default function App() {
  return (
    <div className="grid">
      <Card title="Welcome">
        <p>This paragraph and the button below are the card's children.</p>
        <button type="button">Get started</button>
      </Card>

      <Card
        title="Account"
        footer={<small>Last login: today</small>}
      >
        <ul>
          <li>Plan: Pro</li>
          <li>Seats: 5</li>
        </ul>
      </Card>
    </div>
  );
}
```

```css
/* src/index.css (just enough to see the structure) */
.grid { display: flex; gap: 16px; }
.card { border: 1px solid #ddd; border-radius: 8px; overflow: hidden; min-width: 220px; }
.card__header { padding: 8px 12px; font-weight: 700; background: #f5f5f5; }
.card__body { padding: 12px; }
.card__footer { padding: 8px 12px; border-top: 1px solid #eee; color: #666; }
```

`Card` is reusable precisely because it is ignorant. The `title` and `footer` props accept **JSX as props** (`footer={<small>...</small>}`), and `children` captures the nested content. The `title && ...` guards mean an omitted `title` renders nothing rather than an empty header.

### Example 4 — Composition + a function prop + spreading

Here `Toolbar` composes `Button`, passes a function down, and forwards arbitrary attributes via spread.

```jsx
// src/Button.jsx
export default function Button({ children, variant = 'primary', ...rest }) {
  // `rest` captures onClick, type, disabled, aria-*, etc. and forwards them.
  return (
    <button className={`btn btn--${variant}`} {...rest}>
      {children}
    </button>
  );
}
```

```jsx
// src/Toolbar.jsx
import Button from './Button.jsx';

export default function Toolbar({ onSave }) {
  return (
    <div className="toolbar">
      <Button onClick={onSave}>Save</Button>
      <Button variant="ghost" type="button" disabled aria-label="Refresh list">
        Refresh
      </Button>
    </div>
  );
}
```

```jsx
// src/App.jsx
import Toolbar from './Toolbar.jsx';

export default function App() {
  // We'll properly handle events in lesson 06; this is just a function prop.
  function handleSave() {
    console.log('saving…');
  }
  return <Toolbar onSave={handleSave} />;
}
```

`Button` declares the props it cares about (`children`, `variant`) and **spreads the rest** onto the underlying `<button>`. That single `{...rest}` is why `disabled`, `type`, `aria-label`, and `onClick` all reach the real DOM element without `Button` enumerating them. We'll see in *Pitfalls* why blind spreading is a double-edged sword.

---

## Under the hood

### Component vs. Element vs. Instance

This trio trips up almost everyone. Pin it down now and React stops feeling magical.

- **Component** — the function (or class) itself. The *blueprint*. `function Card(props) {...}`. There is exactly one `Card` definition in your bundle.
- **Element** — a plain, immutable JavaScript **object** describing *one* thing you want on screen: which component (or HTML tag), and which props. Writing `<Card title="Hi" />` does **not** call `Card`. It creates an object roughly like:

  ```js
  // What <Card title="Hi" /> compiles to (React 19 uses the jsx runtime).
  // Note: React 19 tags elements with Symbol.for('react.transitional.element');
  // older versions used Symbol.for('react.element'). The exact tag is an internal detail.
  { $$typeof: Symbol(react.transitional.element), type: Card, props: { title: 'Hi' }, key: null }
  ```

  Elements are **descriptions, not the thing itself** — like a recipe vs. the cooked dish, or a `<note>` saying "render a Card with title Hi." They're cheap to create and throw away. React creates a fresh tree of elements on every render.

- **Instance** — the running, in-memory presence of a component that React manages: its place in the tree, its state, its hooks. With function components you never touch the instance directly (no `this`); React owns it. Each `<Card />` you render is a separate instance with its own props and (later) its own state.

The mental model: **you describe (elements), React reconciles, the DOM updates.** Your JSX produces an element tree; React compares it against the previous tree and applies the minimal DOM changes. Because elements are just objects, "rendering" is mostly object creation plus a diff — fast and predictable. Reconciliation is covered in depth in `05-rendering-lists-conditionals.md`.

### What `<Card title="Hi" />` actually does

JSX is compiled (by Vite's React plugin) to calls into React's runtime. In React 17+ with the automatic runtime, `<Card title="Hi" />` becomes `jsx(Card, { title: 'Hi' })`. That returns the element object above. Nothing about `Card`'s body runs until React decides to render that element — at which point React calls `Card({ title: 'Hi' })` and recurses into whatever *it* returns. So your component tree is realized by React walking the element tree top-down, calling each component function as it goes.

### Props is one frozen object per render

For a given render, a component's `props` is a single object React assembles from the JSX attributes. Conceptually it's a snapshot: the values you read are the values for *this* render. React doesn't hand you a live, mutable view into the parent. In development React even **freezes** the props object, so a stray write to a prop throws a `TypeError`. Separately, `StrictMode` calls your component function **twice** in development; if your render impurely mutates data, the doubled run makes the bug visible (e.g. duplicated items). Both are dev-only and exist to enforce purity, which we explore in `05-rendering-lists-conditionals.md`.

### `children` is just a normal prop

There's nothing structurally special about `children` — it's a prop named `children` that React populates from the JSX you nest between a component's tags. `<Panel>hi</Panel>` is equivalent to `<Panel children="hi" />`. Because it's an ordinary prop, `children` can be a string, an element, an array of elements, a number, `null`, or even a function (the "render prop" pattern, lesson 16). You can also pass *named* slots as separate props (like `footer={<small/>}` above) when "between the tags" isn't expressive enough.

---

## Common mistakes & pitfalls

### 1. Lowercase component name

```jsx
// ❌ WRONG — React treats `card` as an HTML tag, renders an unknown <card> element,
// and never calls your function. No error, just broken/empty output.
function card({ title }) { return <h2>{title}</h2>; }
<card title="Hi" />
```

```jsx
// ✅ RIGHT — PascalCase so JSX resolves it to your component.
function Card({ title }) { return <h2>{title}</h2>; }
<Card title="Hi" />
```

### 2. Mutating props

```jsx
// ❌ WRONG — props are read-only. Mutating the props OBJECT or its contents is a
// purity violation. In development React freezes props, so writing to a property
// throws a TypeError; mutating a nested object/array won't throw but corrupts data
// the parent owns, and StrictMode's double-render surfaces the bug as duplicated work.
function TagList({ tags }) {
  tags.push('extra');         // mutates the parent's array — never do this
  return <span>{tags.join(', ')}</span>;
}
```

```jsx
// ✅ RIGHT — derive a new value; never write back to props.
function TagList({ tags }) {
  const allTags = [...tags, 'extra'];   // local copy, fine
  return <span>{allTags.join(', ')}</span>;
}
```

If a child needs to *change* data, it doesn't mutate props — the parent owns the data and passes a **callback** down (`onChange`) that the child calls. That's "lifting state up," covered in `16-state-management.md`.

### 3. Quoting an expression prop

```jsx
// ❌ WRONG — passes the literal string "user.name", not the value.
<Greeting name="user.name" />
// ❌ WRONG — passes the literal string "{user.name}".
<Greeting name="{user.name}" />
```

```jsx
// ✅ RIGHT — braces evaluate the expression.
<Greeting name={user.name} />
```

### 4. Calling a component instead of rendering it

```jsx
// ❌ WRONG — calls the function directly. It works visually but breaks hooks,
// state identity, and reconciliation. React sees raw output, not an element/instance.
function App() {
  return <div>{Greeting({ name: 'Ada' })}</div>;
}
```

```jsx
// ✅ RIGHT — render it as an element so React owns the instance.
function App() {
  return <div><Greeting name="Ada" /></div>;
}
```

### 5. Defining a component inside another component

```jsx
// ❌ WRONG — `Row` is redefined every render → it's a *new* component type each time,
// so React unmounts/remounts it, destroying any state and DOM. Subtle, painful bugs.
function Table({ rows }) {
  function Row({ cell }) { return <td>{cell}</td>; }   // recreated each render
  return <tbody>{rows.map((r) => <tr><Row cell={r} /></tr>)}</tbody>;
}
```

```jsx
// ✅ RIGHT — define components at module top level so their identity is stable.
function Row({ cell }) { return <td>{cell}</td>; }

function Table({ rows }) {
  return <tbody>{rows.map((r, i) => <tr key={i}><Row cell={r} /></tr>)}</tbody>;
}
```

### 6. Spreading props blindly onto a DOM element

```jsx
// ❌ RISKY — forwards EVERY incoming prop, including internal ones React will warn
// about as unknown DOM attributes (e.g. `variant`, `isActive`), polluting the DOM.
function Button(props) {
  return <button {...props}>{props.children}</button>;
}
<Button variant="ghost" isActive onClick={fn}>Go</Button>
// → React warns: "Unknown DOM attribute `variant`/`isActive`."
```

```jsx
// ✅ RIGHT — pull out the props you consume, spread only the remainder.
function Button({ variant, isActive, children, ...rest }) {
  return (
    <button className={`btn btn--${variant} ${isActive ? 'is-active' : ''}`} {...rest}>
      {children}
    </button>
  );
}
```

Spreading is great for **forwarding** standard DOM attributes to a leaf element; it's dangerous when you spread an object that mixes component-internal props with DOM-bound ones.

---

## Best practices

- **Do** name components in PascalCase and keep them at module top level (never nested inside another component's body).
- **Do** destructure props in the signature with defaults: `function Card({ title = 'Untitled' }) {…}`. It documents the component's API at a glance.
- **Do** keep components **pure**: given the same props, render the same output; never mutate props or perform side effects during render (see `05-rendering-lists-conditionals.md`).
- **Do** prefer **composition with `children`** over passing dozens of configuration props. A `<Card>` that wraps arbitrary content beats a `<Card>` with `showHeader`, `headerText`, `bodyText`, `bodyImage`… flags.
- **Do** pass functions down as props (`onSave`, `onChange`) to let children communicate upward without breaking one-way data flow.
- **Don't** spread an unfiltered props object onto a DOM element — extract what you own, spread the rest.
- **Don't** quote expression props (`name={value}`, not `name="{value}"`).
- **Don't** call components as functions (`Greeting(props)`); render them (`<Greeting {...props} />`).
- **Prefer** one main component per file with a default export; use named exports for small co-located helpers.
- **Avoid** `defaultProps` on function components — it was **removed for function components in React 19** (React ignores it there). Use destructuring defaults.

---

## Exercises

1. **Greeting, parameterized.** Create `Greeting.jsx` that takes a `name` prop and a `punctuation` prop defaulting to `'!'`. Render three greetings from `App` — one of which overrides the punctuation to `'.'`.

2. **Avatar with fallback.** Build `Avatar.jsx` taking `src`, `alt`, and `size` (default `40`). If `src` is missing, render a `<div>` showing the first letter of `alt` instead of an `<img>`. Apply `size` as both width and height via inline style.

3. **Generic `Card` with slots.** Build a `Card` that accepts `title` (string or JSX), optional `footer` (JSX), and `children`. Render a header only when `title` is provided. Use it twice in `App`, once with a footer and once without.

4. **`Stack` layout via children.** Create a `Stack.jsx` that renders its `children` in a flex column with a configurable `gap` prop (default `8`). Pass any nested elements as children. (No `.map` needed — just render `children` inside a styled `div`.)

5. **Validated `Rating`.** Build a `Rating` component that takes a numeric `value` (0–5) and renders that many filled stars (`★`) followed by empty ones (`☆`) up to 5. Add **PropTypes** so `value` is a required number, and `max` is an optional number defaulting to `5`.

### Solutions

```jsx
// 1. src/Greeting.jsx
export default function Greeting({ name, punctuation = '!' }) {
  return <h2>Hello, {name}{punctuation}</h2>;
}

// src/App.jsx
import Greeting from './Greeting.jsx';
export default function App() {
  return (
    <>
      <Greeting name="Ada" />
      <Greeting name="Lin" />
      <Greeting name="Grace" punctuation="." />
    </>
  );
}
```

```jsx
// 2. src/Avatar.jsx
export default function Avatar({ src, alt, size = 40 }) {
  const style = { width: size, height: size, borderRadius: '50%' };
  if (!src) {
    return (
      <div
        style={{ ...style, display: 'grid', placeItems: 'center', background: '#ddd' }}
        aria-label={alt}
      >
        {alt ? alt[0].toUpperCase() : '?'}
      </div>
    );
  }
  return <img src={src} alt={alt} style={style} />;
}
```

```jsx
// 3. src/Card.jsx
export default function Card({ title, footer, children }) {
  return (
    <div className="card">
      {title && <header className="card__header">{title}</header>}
      <div className="card__body">{children}</div>
      {footer && <footer className="card__footer">{footer}</footer>}
    </div>
  );
}

// src/App.jsx
import Card from './Card.jsx';
export default function App() {
  return (
    <>
      <Card title="With footer" footer={<small>Updated today</small>}>
        <p>Body content here.</p>
      </Card>
      <Card title={<span>JSX title 🎉</span>}>
        <p>No footer on this one.</p>
      </Card>
    </>
  );
}
```

```jsx
// 4. src/Stack.jsx
export default function Stack({ gap = 8, children }) {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap }}>
      {children}
    </div>
  );
}

// Usage:
// <Stack gap={16}>
//   <h1>Title</h1>
//   <p>Paragraph</p>
//   <button type="button">Action</button>
// </Stack>
```

```jsx
// 5. src/Rating.jsx
import PropTypes from 'prop-types';

export default function Rating({ value, max = 5 }) {
  const stars = Array.from({ length: max }, (_, i) =>
    i < value ? '★' : '☆'
  );
  return <span role="img" aria-label={`${value} of ${max} stars`}>{stars.join('')}</span>;
}

Rating.propTypes = {
  value: PropTypes.number.isRequired,
  max: PropTypes.number,
};
```

> To use PropTypes, install the standalone package: `npm install prop-types`. It is no longer bundled with React. PropTypes checks run **only in development** and log a console warning on mismatch — they add no production overhead and throw nothing.

---

## A note on PropTypes vs. TypeScript

`PropTypes` gives **runtime** validation: in development, React logs a warning if a prop's type or presence is wrong. It's a lightweight safety net for plain-JS projects:

```jsx
import PropTypes from 'prop-types';

function Card({ title, count, onSave }) { /* ... */ }

Card.propTypes = {
  title: PropTypes.string.isRequired,
  count: PropTypes.number,
  onSave: PropTypes.func,
};
```

Common validators: `PropTypes.string`, `.number`, `.bool`, `.func`, `.array`, `.object`, `.node` (anything renderable), `.element` (a React element), `.oneOf([...])`, `.shape({...})`, `.arrayOf(PropTypes.number)`, each chainable with `.isRequired`.

In modern codebases, **TypeScript supersedes PropTypes**. TypeScript validates props at *compile time* (you see errors in your editor before the app ever runs), describes far richer shapes, and never ships to production. The two overlap, so teams pick one — and that one is increasingly TypeScript. We cover typing props with `interface`/`type` and generics in `18-typescript-react.md`. Reach for PropTypes only in JS-only projects or when consuming untyped data at a boundary.

---

## Key takeaways

- A **function component** is a PascalCase function that returns JSX (or `null`); render it as `<Component />`, never call it as `Component()`.
- **Props** are the component's arguments — a single read-only object the parent passes down. Destructure them in the signature with defaults.
- Pass strings with quotes, everything else (numbers, booleans, objects, arrays, functions, JSX) inside `{braces}`.
- **`children`** is just a prop holding the JSX nested between a component's tags — the foundation of composition. Named slots (`footer={<…/>}`) handle multi-slot layouts.
- **Spread** (`{...rest}`) forwards leftover props to a leaf element; extract the props you consume first so you don't leak internal props onto the DOM.
- Data flows **one way** (parent → child) and props are **read-only**, which keeps components pure and predictable. Children influence parents by calling **callback props**, not by mutating.
- **Component** = the function (blueprint); **element** = a plain object describing one render; **instance** = React's running, stateful presence in the tree.
- Validate props with **PropTypes** at runtime in JS projects, but prefer **TypeScript** (lesson 18) for compile-time safety in serious codebases.

---

## Further reading

- React docs — [Your First Component](https://react.dev/learn/your-first-component)
- React docs — [Importing and Exporting Components](https://react.dev/learn/importing-and-exporting-components)
- React docs — [Passing Props to a Component](https://react.dev/learn/passing-props-to-a-component)
- React docs — [Passing JSX as children](https://react.dev/learn/passing-props-to-a-component#passing-jsx-as-children)
- React docs — [Keeping Components Pure](https://react.dev/learn/keeping-components-pure)
- React docs — [`createElement`](https://react.dev/reference/react/createElement) (what JSX compiles to; understand the element object)
- React docs — [Render and Commit](https://react.dev/learn/render-and-commit) (how elements become DOM)
- npm — [`prop-types`](https://www.npmjs.com/package/prop-types) and the legacy [Typechecking with PropTypes](https://legacy.reactjs.org/docs/typechecking-with-proptypes.html) reference
- Next: `05-rendering-lists-conditionals.md` (why purity matters), then `07-state-usestate.md` (making components interactive).
