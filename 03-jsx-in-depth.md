# 03 — JSX In Depth

> **What you'll learn:** JSX is not HTML — it's a thin syntactic layer that compiles to plain JavaScript function calls, and understanding that compilation is the key to reading, writing, and debugging every React component you'll ever touch.

By the end of this lesson you will be able to:

- Explain what JSX is, why it exists, and why it is **not** HTML.
- Describe how Babel transpiles JSX into `React.createElement` (classic runtime) or `jsx()` (automatic runtime).
- Embed JavaScript expressions inside JSX with `{}` and predict exactly what renders.
- Apply the single-root rule and use Fragments (`<>...</>` and `<React.Fragment key>`).
- Use the correct JSX attribute names (`className`, `htmlFor`, camelCase events, the object `style` prop, `value` vs `defaultValue`, boolean attributes).
- Predict how React renders strings, numbers, arrays, booleans, `null`, `undefined`, and why objects throw.
- Understand `dangerouslySetInnerHTML` and the XSS risk it carries.
- Treat JSX as a first-class JavaScript **expression** you can assign, return, and compose.

## Prerequisites

You should be comfortable with the material from the earlier lessons:

- **`02-getting-started.md`** — what React is, the component model, and the idea of a virtual DOM.
- **`02-getting-started.md`** — a working Vite + React 19 project so you can run every snippet here. Create one with `npm create vite@latest my-app -- --template react`, then `cd my-app && npm install && npm run dev`.

You also need solid **JavaScript fundamentals**: expressions vs statements, arrow functions, the ternary operator (`cond ? a : b`), logical operators (`&&`, `||`, `??`), array `.map()`, template literals, and object literals. JSX is "just JavaScript with one extra syntax," so the more fluent your JS, the easier JSX becomes.

## Concept

When you write this in a `.jsx` file:

```jsx
const greeting = <h1 className="title">Hello, world</h1>;
```

it **looks** like you embedded HTML into JavaScript. You did not. There is no HTML here at all. `<h1 className="title">Hello, world</h1>` is a piece of **JSX syntax** that a compiler (Babel, or the SWC compiler Vite uses) rewrites into a normal JavaScript function call *before* the browser ever sees it. The browser never receives JSX — it receives the compiled JavaScript.

### JSX is a function-call literal in disguise

The single most useful mental model is this: **a JSX tag is sugar for a function call that builds a plain JavaScript object.** That object — a *React element* — is a lightweight description of what you want on screen. It is *not* a DOM node. Writing `<h1>` does not create an `<h1>` element in the document; it creates an object that says "I would like an h1 here, with these props and these children." React later reads that description and decides what to actually do to the real DOM (this reconciliation process is covered in later lessons).

So JSX is to `React.createElement` what an array literal `[1, 2, 3]` is to `new Array(1, 2, 3)`, or what an object literal `{ a: 1 }` is to a constructor call. It's a more readable, more writable surface over an ordinary value-producing expression.

### Why have JSX at all?

You *could* write `React.createElement(...)` by hand. People did, briefly, and it was miserable for anything beyond a few nodes — deeply nested UI becomes an unreadable pyramid of nested calls. JSX lets you express tree-shaped UI in a tree-shaped syntax, with editor support (auto-closing tags, highlighting), while preserving the full power of JavaScript: any value, any expression, any function can flow into the tree. That blend — markup-like structure, full programmatic control — is JSX's whole reason to exist.

### The HTML resemblance is intentional but skin-deep

JSX deliberately *resembles* HTML so it's approachable, but it follows **JavaScript** rules, not HTML rules. That's the source of nearly every beginner surprise: `class` becomes `className`, `for` becomes `htmlFor`, `onclick` becomes `onClick`, `style` takes an object instead of a string, every tag must be closed, and you can only return one root node. Once you internalize "JSX is JavaScript, the HTML look is a costume," each of these stops being a special rule to memorize and becomes an obvious consequence.

## Syntax & API

### `React.createElement` — the classic compile target

```js
React.createElement(type, props, ...children)
// type     : string ('div', 'h1') for DOM tags, or a component function/class
// props    : an object of attributes/props, or null if none
// children : zero or more child elements/values, spread as trailing args
//          → returns a plain object: { type, props: { ...props, children }, key, ... }
//            (in React 19 `ref` is a regular prop and lives inside props, not at top level)
```

### The automatic runtime — `jsx` / `jsxs` (React 17+, default today)

Modern toolchains (the `automatic` JSX runtime, which Vite uses by default for React 17+) compile to functions imported from `react/jsx-runtime` instead:

```js
// from 'react/jsx-runtime'
jsx(type, props, key)    // used when there is 0 or 1 static child
jsxs(type, props, key)   // used when children is a static array (multiple children)
// children live INSIDE props.children here, and `key` is passed as a separate 3rd argument
```

You almost never call these yourself — the compiler emits them. The key practical upshot: **with the automatic runtime you no longer need `import React from 'react'` just to use JSX.** The compiler injects the right import for you.

### Expression container

```jsx
{ /* any JavaScript expression */ }
```

Anywhere inside JSX — as a child or as an attribute value — `{}` drops you back into JavaScript-expression mode. **Expressions only**, not statements: `{user.name}`, `{2 + 2}`, `{isOpen ? 'open' : 'closed'}`, `{items.map(...)}` are fine; `{if (x) {...}}`, `{for (...)}`, `{const y = 1}` are not.

### Fragments

```jsx
<>...</>                         // shorthand fragment — groups children, renders no wrapper DOM
<React.Fragment>...</React.Fragment>
<React.Fragment key={id}>...</React.Fragment>   // long form — required when you need a key
```

### Attribute quick-reference

| HTML | JSX | Notes |
| --- | --- | --- |
| `class` | `className` | `class` is a reserved JS word |
| `for` | `htmlFor` | `for` is a reserved JS word |
| `onclick`, `onchange` | `onClick`, `onChange` | camelCase, value is a function |
| `style="color:red"` | `style={{ color: 'red' }}` | object, camelCased CSS keys |
| `tabindex` | `tabIndex` | camelCase |
| `<input value="x">` | `value` / `defaultValue` | controlled vs uncontrolled |
| `<br>` | `<br />` | every tag must close |
| `disabled` | `disabled={true}` / `disabled` | boolean attribute |
| `data-*`, `aria-*` | `data-*`, `aria-*` | kept as-is (the exception to camelCase) |

## Examples

### Example 1 — A component returning JSX

```jsx
// src/Welcome.jsx
function Welcome() {
  return (
    <section className="welcome">
      <h1>Hello, React 19</h1>
      <p>JSX compiles to JavaScript.</p>
    </section>
  );
}

export default Welcome;
```

The parentheses after `return` are not JSX syntax — they're ordinary JS grouping parentheses that let the JSX start on the next line without JavaScript's automatic-semicolon-insertion silently turning `return` into `return;`. **Always wrap multiline JSX in `(` ... `)`.**

### Example 2 — Embedding expressions with `{}`

```jsx
function Profile() {
  const user = { firstName: 'Ada', lastName: 'Lovelace', age: 36 };
  const fullName = `${user.firstName} ${user.lastName}`;

  return (
    <article>
      <h2>{fullName}</h2>
      <p>Next year you will be {user.age + 1}.</p>
      <p>Status: {user.age >= 18 ? 'adult' : 'minor'}</p>
      {/* a comment inside JSX must live in an expression container */}
      <a href={'https://example.com/' + user.firstName.toLowerCase()}>profile</a>
    </article>
  );
}
```

Note three uses of `{}`: as text content (`{fullName}`), inside arithmetic (`{user.age + 1}`), and as an **attribute value** (`href={...}`). Attribute values use `{}` for JS expressions and plain quotes only for literal strings (`href="..."`).

### Example 3 — Conditional and list rendering

```jsx
function TodoList({ todos }) {
  if (todos.length === 0) {
    return <p>No todos yet. </p>;
  }

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id} className={todo.done ? 'done' : 'pending'}>
          {todo.text}
          {todo.starred && <span aria-label="starred"> ⭐</span>}
        </li>
      ))}
    </ul>
  );
}

// usage: <TodoList todos={[{ id: 1, text: 'Learn JSX', done: false, starred: true }]} />
```

`{todos.map(...)}` renders an **array of elements** — React accepts arrays as children. Each element needs a stable `key` so React can track items across re-renders (keys are covered in depth in the lists lesson). `{todo.starred && <span/>}` is the classic short-circuit pattern: if `todo.starred` is truthy, the `<span>` renders; if falsy, the expression evaluates to `false`, which React renders as nothing.

### Example 4 — Fragments and keyed fragments

```jsx
function Definition() {
  // Return two siblings with no wrapping DOM element:
  return (
    <>
      <dt>JSX</dt>
      <dd>A syntax extension that compiles to React.createElement calls.</dd>
    </>
  );
}

function Glossary({ entries }) {
  return (
    <dl>
      {entries.map((entry) => (
        // Need a key on a fragment? You MUST use the long form:
        <React.Fragment key={entry.id}>
          <dt>{entry.term}</dt>
          <dd>{entry.definition}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}
```

The shorthand `<>...</>` **cannot take attributes** — including `key`. When a fragment is the item produced inside a `.map()`, switch to `<React.Fragment key={...}>`.

### Example 5 — The `style` object and `dangerouslySetInnerHTML`

```jsx
function Banner({ html, accent }) {
  const boxStyle = {
    backgroundColor: accent,   // camelCase: background-color → backgroundColor
    padding: 16,               // bare number → '16px' for length properties
    border: '2px solid black',
  };

  return (
    <div style={boxStyle}>
      {/* ONLY use this with content you fully trust — see Under the hood */}
      <div dangerouslySetInnerHTML={{ __html: html }} />
    </div>
  );
}
```

`style` is an **object**, its keys are camelCased CSS properties, and numeric values for length-type properties get `px` appended automatically (`padding: 16` → `padding: 16px`; but unitless properties like `lineHeight`, `zIndex`, `flexGrow` stay raw).

### Example 6 — Controlled vs uncontrolled inputs

```jsx
import { useState } from 'react';

function NameForm() {
  const [name, setName] = useState('');
  return (
    <form>
      {/* CONTROLLED: React owns the value; pair `value` with `onChange` */}
      <input value={name} onChange={(e) => setName(e.target.value)} />

      {/* UNCONTROLLED: DOM owns the value; set the INITIAL value only */}
      <input defaultValue="anonymous" />
    </form>
  );
}
```

`value` makes React the source of truth (a *controlled* input — see the forms lesson). `defaultValue` sets only the initial DOM value and then steps back. Passing `value` without `onChange` produces a read-only field and a console warning.

## Under the hood

### What the compiler actually emits

Take this tree:

```jsx
const tree = (
  <div className="card" id="c1">
    <h1>Title</h1>
    <p>Body text</p>
  </div>
);
```

**Classic runtime** (`React.createElement`) compiles it to:

```js
const tree = React.createElement(
  'div',
  { className: 'card', id: 'c1' },
  React.createElement('h1', null, 'Title'),
  React.createElement('p', null, 'Body text')
);
```

**Automatic runtime** (today's default, Vite/React 19) compiles it to roughly:

```js
import { jsx as _jsx, jsxs as _jsxs } from 'react/jsx-runtime';

const tree = _jsxs('div', {
  className: 'card',
  id: 'c1',
  children: [
    _jsx('h1', { children: 'Title' }),
    _jsx('p', { children: 'Body text' }),
  ],
});
// `jsxs` (the "s" = static array of children) is used because there are 2 children.
```

Either way, the result is the same kind of value: a plain object. Logging `tree` shows approximately:

```js
{
  // In React 19 the brand changed from Symbol.for('react.element') to:
  $$typeof: Symbol(react.transitional.element),   // marks it as a genuine React element (anti-injection guard)
  type: 'div',
  key: null,
  props: {
    className: 'card',
    id: 'c1',
    children: [ /* the h1 and p element objects */ ],
  },
  // (React ≤18 also hoisted a top-level `ref` field here; in React 19 `ref`
  //  is a regular prop and lives inside `props` like any other.)
}
```

This is the payoff of the whole lesson: **JSX produces inert description objects.** Nothing renders, no DOM is touched, no component function runs, until React's renderer walks this tree.

### Capitalization decides `type`

The compiler uses the **first letter** to decide whether `type` is a string or a reference:

- `<div>` → `type: 'div'` — a lowercase string, treated as a DOM tag.
- `<Welcome>` → `type: Welcome` — a reference to the `Welcome` function in scope, treated as a component.

That's why **components must be capitalized**: `<welcome />` would compile to `createElement('welcome', ...)`, asking the DOM for an unknown `<welcome>` tag instead of calling your function.

### How values become rendered output

When React renders the `children` of an element, each value is interpreted by type:

- **string / number** → rendered as text. `{42}` shows `42`, `{'hi'}` shows `hi`.
- **array** → each item is rendered in order (this is why `.map()` works).
- **`true`, `false`, `null`, `undefined`** → render **nothing** (no node, no text). This is the deliberate design behind `{cond && <X/>}` and `{maybe ?? null}`.
- **a React element** → rendered as its subtree.
- **a plain object** like `{ a: 1 }` → **throws**: *"Objects are not valid as a React child."* React has no idea how to display an arbitrary object, so rather than guess (or silently print `[object Object]`) it errors loudly. To show object data, pick fields (`{user.name}`) or stringify it (`{JSON.stringify(user)}`).

A subtle trap: `{0}` and `{NaN}` are numbers, so they **do** render — `0 && <X/>` renders the literal `0` on screen, not nothing. Guard with a boolean: `{count > 0 && <X/>}`.

### Whitespace handling

JSX trims leading/trailing whitespace on lines and collapses blank lines, then collapses runs of whitespace *within* a line to a single space (much like HTML). Newlines between tags adjacent to text become a single space; newlines between two tags are removed. When you genuinely need a space between two expressions, insert an explicit one: `{first}{' '}{last}` or `{`${first} ${last}`}`. A literal non-breaking space is `{' '}`.

### `dangerouslySetInnerHTML` and XSS

React escapes all string children by default. If a string contains `<script>alert(1)</script>`, React renders it as visible **text**, not as executable markup — that auto-escaping is React's built-in defense against cross-site scripting (XSS). `dangerouslySetInnerHTML={{ __html: someString }}` is the deliberate escape hatch that bypasses that protection and injects raw HTML. The clumsy, alarming name is intentional: it forces you to acknowledge the danger. If `someString` ever contains attacker-controlled content, you've handed an attacker the ability to run arbitrary script in your users' browsers. Only use it with content you fully control, or that you've sanitized with a vetted library (e.g. DOMPurify) first.

### Props are immutable, elements are read-only

The element object React hands you (and the `props` object inside a component) is meant to be treated as **frozen**. Never mutate `props` or push into `element.props.children`. React relies on elements being stable descriptions; mutation breaks reconciliation in ways that are extremely hard to debug.

## Common mistakes & pitfalls

**1. Returning multiple sibling elements without a wrapper.**

```jsx
// ❌ WRONG — JSX expressions must have one parent; this is a syntax error
function Bad() {
  return (
    <h1>Title</h1>
    <p>Body</p>
  );
}
```

```jsx
// ✅ RIGHT — wrap in a Fragment (no extra DOM) or a single element
function Good() {
  return (
    <>
      <h1>Title</h1>
      <p>Body</p>
    </>
  );
}
```

A JSX expression compiles to a single `createElement` call, which returns a single value — so you must have one root.

**2. Using `class` and `for`.**

```jsx
// ❌ WRONG — `class`/`for` are reserved JS words; React ignores/warns
<label class="lbl" for="email">Email</label>
```

```jsx
// ✅ RIGHT
<label className="lbl" htmlFor="email">Email</label>
```

**3. Passing a CSS string to `style`.**

```jsx
// ❌ WRONG — style is an object in JSX, not a string
<div style="color: red; font-size: 20px">hi</div>
```

```jsx
// ✅ RIGHT — object with camelCased keys (note the DOUBLE braces:
//            outer = JSX expression, inner = the object literal)
<div style={{ color: 'red', fontSize: 20 }}>hi</div>
```

**4. The `0` short-circuit leak.**

```jsx
// ❌ WRONG — when items.length is 0, this renders the number "0" on screen
{items.length && <List items={items} />}
```

```jsx
// ✅ RIGHT — coerce to a real boolean first
{items.length > 0 && <List items={items} />}
```

**5. Rendering a raw object.**

```jsx
// ❌ WRONG — throws "Objects are not valid as a React child"
function Bad({ user }) {
  return <p>{user}</p>;
}
```

```jsx
// ✅ RIGHT — render a field, or stringify deliberately
function Good({ user }) {
  return <p>{user.name}</p>;            // or: {JSON.stringify(user)}
}
```

**6. Forgetting to close tags.**

```jsx
// ❌ WRONG — valid in HTML, a syntax error in JSX
<img src="logo.png">
<br>
```

```jsx
// ✅ RIGHT — self-close every void element
<img src="logo.png" />
<br />
```

**7. Putting statements inside `{}`.**

```jsx
// ❌ WRONG — `if` is a statement, not an expression
<p>{if (loggedIn) { 'Hi' } else { 'Bye' }}</p>
```

```jsx
// ✅ RIGHT — use a ternary (an expression), or compute before the return
<p>{loggedIn ? 'Hi' : 'Bye'}</p>
```

## Best practices

- **Do** wrap multiline JSX in parentheses after `return` to dodge automatic semicolon insertion.
- **Do** prefer the shorthand `<>...</>` for grouping; reach for `<React.Fragment key>` only when you need a key.
- **Do** keep the logic *above* the `return` — compute variables, then drop simple expressions into the JSX. Deeply nested ternaries in markup are a smell.
- **Do** coerce to real booleans (`x > 0`, `Boolean(x)`, `!!x`) before `&&` to avoid rendering `0`/`NaN`.
- **Do** give list items a stable, unique `key` — never the array index when the list can reorder (see the lists & keys lesson).
- **Don't** use `dangerouslySetInnerHTML` with any content you don't fully control; sanitize first.
- **Don't** mutate `props` or element objects — treat them as read-only.
- **Don't** add `import React from 'react'` just to use JSX in a modern setup; the automatic runtime handles it. Still import the named hooks/APIs you actually use (`import { useState } from 'react'`).
- **Don't** confuse `value` (controlled) with `defaultValue` (uncontrolled initial value); pick one per input.

## Exercises

1. **Compile by hand.** Without running a tool, write the `React.createElement` (classic runtime) equivalent of:
   ```jsx
   <ul className="menu">
     <li>Home</li>
     <li>About</li>
   </ul>
   ```

2. **Expression embedding.** Build a `<PriceTag />` component that takes props `{ label, cents }` and renders `Coffee: $3.50` from `{ label: 'Coffee', cents: 350 }`. Format the cents into dollars inside JSX.

3. **Conditional + list.** Build a `<Notifications />` component taking a prop `items` (array of `{ id, text, unread }`). Render "All caught up!" when the array is empty; otherwise render a `<ul>` where unread items get `className="unread"`. Avoid the `0` short-circuit bug.

4. **Fragment with keys.** Build a `<KeyValueGrid />` taking `pairs` (array of `{ id, k, v }`) and rendering, for each pair, a `<dt>` and `<dd>` as **siblings** with no extra wrapper element, correctly keyed.

5. **Safe vs unsafe HTML.** Build a `<Comment />` taking `{ author, body }`. Render `body` as escaped text (the safe default). Then add a second variant `<RichComment />` using `dangerouslySetInnerHTML` and write a one-sentence comment explaining the precondition that must hold before it's acceptable.

### Solutions

```jsx
// 1 — compile by hand (classic runtime)
React.createElement(
  'ul',
  { className: 'menu' },
  React.createElement('li', null, 'Home'),
  React.createElement('li', null, 'About')
);
// Automatic-runtime equivalent: jsxs('ul', { className: 'menu',
//   children: [ jsx('li', { children: 'Home' }), jsx('li', { children: 'About' }) ] })

// 2 — expression embedding
function PriceTag({ label, cents }) {
  return <span>{label}: ${(cents / 100).toFixed(2)}</span>;
}

// 3 — conditional + list
function Notifications({ items }) {
  if (items.length === 0) {
    return <p>All caught up!</p>;
  }
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id} className={item.unread ? 'unread' : undefined}>
          {item.text}
        </li>
      ))}
    </ul>
  );
}

// 4 — fragment with keys
function KeyValueGrid({ pairs }) {
  return (
    <dl>
      {pairs.map((pair) => (
        <React.Fragment key={pair.id}>
          <dt>{pair.k}</dt>
          <dd>{pair.v}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}

// 5 — safe vs unsafe HTML
function Comment({ author, body }) {
  // body is auto-escaped: <script> shows as text, never executes.
  return (
    <p>
      <strong>{author}:</strong> {body}
    </p>
  );
}

function RichComment({ author, body }) {
  // PRECONDITION: `body` must be sanitized/trusted HTML (e.g. run through
  // DOMPurify). Otherwise this is an XSS hole — attacker HTML would execute.
  return (
    <p>
      <strong>{author}:</strong>{' '}
      <span dangerouslySetInnerHTML={{ __html: body }} />
    </p>
  );
}
```

## Key takeaways

- JSX is **not HTML** — it's syntactic sugar compiled to `React.createElement` (classic) or `jsx()`/`jsxs()` (automatic runtime) calls that return plain JavaScript element objects.
- Those element objects are **inert descriptions**; nothing touches the DOM until React renders them.
- `{}` embeds any JavaScript **expression** (not statements) as a child or attribute value.
- A JSX expression must have **one root**; group siblings with a Fragment (`<>...</>`, or `<React.Fragment key>` when a key is needed).
- Attribute names follow **JavaScript** rules: `className`, `htmlFor`, camelCase events, object `style`, `value` vs `defaultValue`, self-closing tags, boolean attributes.
- React renders strings/numbers as text, arrays per-item, and renders **nothing** for `true`/`false`/`null`/`undefined`; plain **objects throw**.
- React **escapes strings** by default for XSS safety; `dangerouslySetInnerHTML` opts out and must only be used with trusted/sanitized content.
- JSX is a first-class **expression**: assign it to variables, return it from functions, and pass it around like any value.

## Further reading

- React docs — *Writing Markup with JSX*: https://react.dev/learn/writing-markup-with-jsx
- React docs — *JavaScript in JSX with Curly Braces*: https://react.dev/learn/javascript-in-jsx-with-curly-braces
- React docs — *Rendering Lists* (keys & fragments in lists): https://react.dev/learn/rendering-lists
- React reference — *Fragment (`<>...</>`)*: https://react.dev/reference/react/Fragment
- React reference — *`createElement`*: https://react.dev/reference/react/createElement
- React reference — *common components & `dangerouslySetInnerHTML`*: https://react.dev/reference/react-dom/components/common
- *Introducing the New JSX Transform* (automatic runtime): https://legacy.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html
- Babel REPL — paste JSX and watch it compile in real time: https://babeljs.io/repl

---

*Next up: see `04-components-and-props.md` for how these JSX elements become reusable components, and `08-useeffect-side-effects.md` once you're rendering dynamic data.*
