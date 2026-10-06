# 05 — Conditional Rendering & Lists

> **What you'll learn:** how to make React render different UI based on conditions, and how to turn arrays of data into arrays of elements — correctly, with keys that survive reordering.

By the end of this lesson you will be able to:

- Choose the right conditional-rendering technique (`if`/early return, ternary, `&&`, lookup map) for each situation.
- Recognize and fix the classic `&&` "0 / NaN leak" bug.
- Render a list from an array with `.map()` returning JSX elements.
- Explain what a `key` is, why React's reconciler needs it, and the rules for choosing one.
- Demonstrate the index-key reordering bug and fix it with a stable id.
- Render empty states, nested lists, and lists that are filtered and sorted before rendering.
- Avoid the common "returned nothing" and "missing `return` in `.map`" mistakes.

---

## Prerequisites

Before this lesson you should be comfortable with:

- **JSX basics** — embedding JavaScript expressions with `{ }`, the fact that JSX is an expression that compiles to function calls (see `03-jsx-in-depth.md`).
- **Components & props** — passing data into a component (see `04-components-and-props.md`).
- **State with `useState`** — because most conditions and lists are driven by state that changes over time (see `07-state-usestate.md`).

You should also know plain-JavaScript array methods: `Array.prototype.map`, `filter`, `sort`, and `slice`. Conditional rendering in React is *just JavaScript* — there is no special template language, so the better your JS, the easier this lesson is.

---

## Concept

React UIs are a **function of data**. You hand React a tree of elements describing what the screen should look like *right now*, and React makes the DOM match. The natural question follows: how do you describe a tree whose *shape* depends on data — show a spinner while loading, a list when loaded, an error banner when something broke?

The answer is the key insight of this whole lesson: **JSX is just JavaScript expressions, so you use ordinary JavaScript to decide what to render.** There is no `*ngIf`, no `v-if`, no `{{#each}}`. You use `if`, the ternary operator `? :`, the logical `&&` operator, and `.map()`. React imposes almost nothing new — it just gives you a few rules about *one* thing: keys.

Two mental models to carry through:

1. **Rendering returns a value.** A component returns one of: a React element (`<div>…</div>`), an array of elements, a string/number, or `null` / `undefined` / `false` (which all render *nothing*). So "conditional rendering" is just "pick which value to return."

2. **A list is an array of elements.** When you write `{items.map(...)}`, you're embedding an *array* of elements into JSX. React happily renders an array of children. The only catch: React needs to track each child across re-renders so it can update them efficiently — that's what `key` is for. Think of keys like name tags at a conference: if everyone wears a stable name tag, you can rearrange the chairs and still know who's who. Without name tags, React falls back to "the person in seat 3," which breaks the moment people swap seats.

---

## Syntax & API

There is no dedicated "rendering API" here — these are JavaScript constructs used inside (or just before) a `return`. The one genuinely React-specific piece is the `key` prop.

```jsx
// 1. if / else before the return — full statements, any logic
function Component({ status }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <Error />;
  return <Data />;
}

// 2. Early return null — render nothing
if (!user) return null; // valid: null renders nothing

// 3. Ternary inside JSX — choose between two expressions
<div>{isLoggedIn ? <Dashboard /> : <LoginForm />}</div>

// 4. Logical && — render the right side only when left is truthy
<div>{hasMessages && <Badge />}</div>

// 5. Lookup map — render one of many by key
const VIEWS = { list: <ListView />, grid: <GridView /> };
<div>{VIEWS[currentView]}</div>

// 6. Lists with map — array in, array of elements out
<ul>
  {items.map((item) => (
    <li key={item.id}>{item.name}</li> // key is REQUIRED on list items
  ))}
</ul>
```

**Values and how they render:**

| You return / embed | React renders |
| --- | --- |
| A React element | that element |
| A string or number | the text |
| `null`, `undefined`, `false`, `true` | **nothing** (no DOM node) |
| `0` (a number) | the text **`0`** ⚠️ |
| `NaN` | the text **`NaN`** ⚠️ |
| An array | each item, in order (needs keys) |

**`key`** — a special prop React reads (you can't access it as a normal prop inside the child). Rules:

- Put it on the **outermost element returned for each list item** (including a `<Fragment>` if that's the root).
- Must be **unique among siblings** (not globally unique).
- Must be **stable** across renders for the same logical item — derive it from the data's identity (`item.id`), not from `Math.random()` or the array index (usually).

---

## Examples

### Example 1 — `if`/early return vs. ternary vs. `&&`

A small, fully-runnable component that shows the three core techniques side by side.

```jsx
import { useState } from 'react';

export default function Greeting() {
  const [user, setUser] = useState(null);

  // Technique A: if / early return — best when the WHOLE component differs.
  if (user === null) {
    return (
      <button onClick={() => setUser({ name: 'Ada', unread: 3 })}>
        Log in
      </button>
    );
  }

  return (
    <div>
      {/* Technique B: ternary — choose between two branches inline */}
      <h1>{user.name ? `Welcome, ${user.name}` : 'Welcome'}</h1>

      {/* Technique C: && — show the badge ONLY when there are unread items */}
      {user.unread > 0 && <span className="badge">{user.unread} new</span>}

      <button onClick={() => setUser(null)}>Log out</button>
    </div>
  );
}
```

Notes: the early `return` keeps the logged-out UI completely separate and avoids nesting. Inside the returned JSX we use a ternary for "this or that" and `&&` for "this or nothing."

### Example 2 — The `&&` 0-leak bug, and three fixes

`&&` returns its *left* operand when that operand is falsy. The values `false`, `null`, and `undefined` render nothing — but `0` and `NaN` are falsy values that **do** render as visible text. This is the single most common conditional-rendering bug.

```jsx
function Cart({ count }) {
  // 🐛 BUG: when count === 0, `count && ...` evaluates to 0,
  // and React renders the literal text "0" in the DOM.
  return <div>{count && <span>{count} items in cart</span>}</div>;
}
```

Three correct fixes — pick based on intent:

```jsx
// Fix 1: compare to make the left side a real boolean
<div>{count > 0 && <span>{count} items in cart</span>}</div>

// Fix 2: coerce with Boolean() (or the !! idiom)
<div>{Boolean(count) && <span>{count} items in cart</span>}</div>
<div>{!!count && <span>{count} items in cart</span>}</div>

// Fix 3: use a ternary with an explicit "nothing" branch
<div>{count ? <span>{count} items in cart</span> : null}</div>
```

The same trap applies to `array.length &&` — `[].length` is `0`, which leaks. Always write `array.length > 0 && …`.

### Example 3 — Render one of many via a lookup map

When you have more than two branches, a chain of ternaries becomes unreadable. A lookup object (or `switch` inside a helper) scales far better.

```jsx
import { useState } from 'react';

const TABS = {
  profile: ProfilePanel,
  billing: BillingPanel,
  security: SecurityPanel,
};

function ProfilePanel() { return <p>Your profile</p>; }
function BillingPanel() { return <p>Your invoices</p>; }
function SecurityPanel() { return <p>2FA & passwords</p>; }

export default function Settings() {
  const [tab, setTab] = useState('profile');
  const Panel = TABS[tab] ?? ProfilePanel; // fall back if key is unknown

  return (
    <div>
      <nav>
        {Object.keys(TABS).map((name) => (
          <button key={name} onClick={() => setTab(name)} aria-pressed={tab === name}>
            {name}
          </button>
        ))}
      </nav>
      <Panel />
    </div>
  );
}
```

We store *component references* (capitalized, not rendered) in the map, then render the chosen one as `<Panel />`. The `?? ProfilePanel` guards against an unknown key. The `switch` form is equivalent and handy when each branch needs different props:

```jsx
function renderPanel(tab) {
  switch (tab) {
    case 'billing': return <BillingPanel currency="USD" />;
    case 'security': return <SecurityPanel mfaRequired />;
    default: return <ProfilePanel />;
  }
}
```

### Example 4 — A real list with stable keys, empty state, filter + sort

This is the workhorse pattern: take data, transform it (filter/sort), map to elements, and handle the empty case.

```jsx
import { useState } from 'react';

const INITIAL = [
  { id: 'a1', name: 'Refactor auth', done: false, priority: 2 },
  { id: 'b2', name: 'Write tests', done: true, priority: 1 },
  { id: 'c3', name: 'Fix login bug', done: false, priority: 3 },
];

export default function TaskList() {
  const [tasks] = useState(INITIAL);
  const [showDone, setShowDone] = useState(true);

  // Transform BEFORE mapping. Note: sort() mutates, so copy with [...] first.
  const visible = [...tasks]
    .filter((t) => (showDone ? true : !t.done))
    .sort((a, b) => b.priority - a.priority); // highest priority first

  return (
    <div>
      <label>
        <input
          type="checkbox"
          checked={showDone}
          onChange={(e) => setShowDone(e.target.checked)}
        />
        Show completed
      </label>

      {/* Empty state: length > 0 avoids the 0-leak */}
      {visible.length > 0 ? (
        <ul>
          {visible.map((task) => (
            <li key={task.id} style={{ opacity: task.done ? 0.5 : 1 }}>
              {task.name} — P{task.priority}
            </li>
          ))}
        </ul>
      ) : (
        <p>No tasks to show.</p>
      )}
    </div>
  );
}
```

Key points: we build `visible` *before* `return`, copy the array before `.sort()` (which mutates in place — sorting state directly is a subtle bug), and use `task.id` — a stable data identity — as the key. The empty state uses a ternary so we always render *something* meaningful.

### Example 5 — Nested lists

Each level of nesting needs its own keys, unique among *its* siblings.

```jsx
const GROUPS = [
  { id: 'g1', title: 'Frontend', members: [{ id: 'u1', name: 'Ada' }, { id: 'u2', name: 'Lin' }] },
  { id: 'g2', title: 'Backend', members: [{ id: 'u3', name: 'Sam' }] },
];

function Teams() {
  return (
    <div>
      {GROUPS.map((group) => (
        <section key={group.id}>
          <h2>{group.title}</h2>
          <ul>
            {group.members.map((member) => (
              <li key={member.id}>{member.name}</li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

`u1` and `u3` don't need to be globally unique — only unique inside their own `<ul>`. The outer `key={group.id}` and inner `key={member.id}` are independent.

---

## Under the hood

### Why keys exist: reconciliation

When state changes, your component re-renders and returns a *new* tree of elements. React doesn't throw away the DOM and rebuild it; it **diffs** the new element tree against the previous one and applies the minimal set of DOM mutations. This diffing process is called **reconciliation**.

For a list of children, React needs to answer: "is the third element in the new array *the same logical thing* as the third element in the old array, or a different one?" Without help, React matches children **by position**. If you insert an item at the top, every position shifts, so React thinks every item changed — it mutates DOM nodes that didn't need to change, and any component state attached to those nodes (input text, focus, animation progress) gets associated with the *wrong* item.

`key` overrides position-based matching with **identity-based matching**. React builds a map from key → previous element. When it sees the same key in the new render, it knows "this is the same item, just possibly moved or updated," so it preserves and moves the existing DOM node and its component state instead of recreating it.

### What actually breaks with index keys

Index keys (`key={i}`) are *positional* keys wearing an `id` costume — they tell React "the item at position 2 is keyed 2," which is exactly the position-based matching that keys are supposed to fix. They're harmless for a **static, append-only** list, but they corrupt state when items **reorder, insert at the front, or get filtered out**. Concretely: each list item has internal DOM/component state (an uncontrolled `<input>`'s typed value, focus, a CSS transition). With index keys, after a reorder React keeps "input keyed 0" in DOM position 0 but feeds it the *new* item-0's data — so the text you typed for item A now sits next to item B's label. We demonstrate this exact bug in Pitfalls below.

### `key` is not a prop, and changing it remounts

`key` is consumed by React itself — it never reaches the child as `props.key` (reading it logs a warning and returns `undefined`). If you need the value inside the child too, pass it twice: `<Row key={item.id} id={item.id} />`.

A powerful corollary: **changing a component's `key` forces React to unmount the old instance and mount a fresh one**, resetting all its state. This is the intended escape hatch for "reset this subtree when X changes" — e.g. `<EditForm key={selectedUserId} user={user} />` gives each user a clean form.

### Arrays as children & Fragments

React accepts an array as a child node and renders each element in order. That's literally what `.map()` produces. When a list item needs multiple sibling elements but no wrapper DOM node, use a keyed Fragment — and note the `key` goes on the Fragment:

```jsx
import { Fragment } from 'react';

items.map((item) => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </Fragment>
));
```

The shorthand `<>…</>` **cannot take a key** — you must use the explicit `<Fragment>` form when a key is required.

### Booleans render nothing — by design

`true && <X/>` renders `<X/>`; `false && <X/>` evaluates to `false`, and React renders `false` (and `true`, `null`, `undefined`) as *nothing*. This is what makes `&&` work as a conditional at all. The runtime explicitly skips these values when building the host tree — they produce no text node, unlike `0` and `NaN`, which are numbers and therefore become text.

---

## Common mistakes & pitfalls

### Pitfall 1 — The `&&` number leak

```jsx
// ❌ WRONG — renders "0" when messages is empty
{messages.length && <MessageList items={messages} />}
```

```jsx
// ✅ RIGHT — force a boolean
{messages.length > 0 && <MessageList items={messages} />}
```

`[].length` is `0`, a falsy number that React renders as visible text. Always compare or coerce.

### Pitfall 2 — Index keys on a reorderable list

```jsx
// ❌ WRONG — index keys + per-row state = state attaches to the wrong row
function BuggyList({ rows }) {
  return rows.map((row, i) => (
    <li key={i}>
      <input defaultValue={row.label} /> {/* type here, then reorder rows */}
    </li>
  ));
}
```

After you type into row A and the list reorders (say a new item is prepended), your typed text stays in DOM position 0 but that position now belongs to a *different* row — the input value and the label desynchronize.

```jsx
// ✅ RIGHT — stable identity key tracks each row through reorders
function FixedList({ rows }) {
  return rows.map((row) => (
    <li key={row.id}>
      <input defaultValue={row.label} />
    </li>
  ));
}
```

Use the data's own id. If your data genuinely has no id, generate one *when you create the item* (e.g. `crypto.randomUUID()`), store it on the item, and reuse it — never generate a key inside render.

### Pitfall 3 — Forgetting `return` inside a `.map` arrow with braces

```jsx
// ❌ WRONG — braces create a block body; nothing is returned → renders nothing
{items.map((item) => {
  <li key={item.id}>{item.name}</li>; // computed then discarded
})}
```

```jsx
// ✅ RIGHT — either add an explicit return…
{items.map((item) => {
  return <li key={item.id}>{item.name}</li>;
})}

// …or use a parenthesized implicit return (no braces)
{items.map((item) => (
  <li key={item.id}>{item.name}</li>
))}
```

`{ }` after `=>` is a *function block*, not an object — without `return` you get `undefined`. The parenthesized form `(` … `)` is an implicit return and is the idiomatic choice for JSX.

### Pitfall 4 — Sorting/reversing state in place

```jsx
// ❌ WRONG — sort() mutates the array held in state
const sorted = tasks.sort((a, b) => a.priority - b.priority);
```

```jsx
// ✅ RIGHT — copy first
const sorted = [...tasks].sort((a, b) => a.priority - b.priority);
```

Mutating state objects breaks React's change detection and can cause stale or skipped renders. Always derive a fresh array.

### Pitfall 5 — A branch that returns nothing by accident

```jsx
// ❌ WRONG — if `data` is undefined, the function falls off the end → returns undefined,
// which is fine to render but if you MEANT to show a fallback you've hidden a bug.
function Profile({ data }) {
  if (data) {
    return <h1>{data.name}</h1>;
  }
  // (no return here)
}
```

```jsx
// ✅ RIGHT — be explicit about the empty case
function Profile({ data }) {
  if (!data) return <p>Loading…</p>;
  return <h1>{data.name}</h1>;
}
```

Returning `undefined`/`null` is *legal* (renders nothing) but should be a deliberate choice, not an accident.

---

## Best practices

- **Do** prefer early `return` / `if` for whole-component branching; reserve ternaries for small inline either/or and `&&` for "render or nothing."
- **Do** always write `array.length > 0 && …` or `Boolean(x) && …` — never bare `x &&` when `x` could be `0` or `NaN`.
- **Do** key every `.map()` output on a **stable, unique** identity from your data (`item.id`).
- **Don't** use the array index as a key for lists that can reorder, filter, or insert in the middle. Index keys are acceptable only for static, never-reordered lists.
- **Don't** use `Math.random()` or `Date.now()` as keys — a new key every render forces a full remount and destroys state and performance.
- **Do** put the `key` on the **outermost** element returned per item; use `<Fragment key={…}>` (not `<>`) when you need a keyed wrapper-less group.
- **Do** filter and sort into a derived variable *above* the `return`, copying the array (`[...arr]`) before `.sort()`/`.reverse()`.
- **Do** render an explicit empty state (`<p>No results</p>`) rather than letting an empty list render to a blank screen.
- **Don't** put heavy logic inside JSX — compute it before `return` and keep the JSX declarative.
- **Do** reach for a lookup map or `switch` once you have three or more branches instead of nesting ternaries.

---

## Exercises

1. **Status switch.** Build a `<Notice status={...} />` component that renders a green "Saved", a yellow "Saving…", or a red "Failed to save" message depending on `status` (`'idle' | 'saving' | 'error' | 'success'`). For `'idle'`, render nothing. Use whichever conditional technique is cleanest.

2. **Fix the leak.** You're given `function Inbox({ unread }) { return <header>{unread && <Dot />}</header>; }`. It renders a stray `0` when the inbox is empty. Fix it without changing what's shown when `unread` is positive.

3. **Filtered, sorted list with empty state.** Given an array of products `{ id, name, price, inStock }`, render only in-stock products, sorted by price ascending, each as `<li>`. Show "Nothing in stock" when the filtered list is empty. Give each `<li>` a correct key.

4. **Prove the index-key bug.** Build a list of three rows, each with an uncontrolled `<input>`, plus a button that prepends a new row to the front. Key the rows by index, type into the first input, click the button, and observe the desync. Then fix it with stable ids.

5. **Tabbed view with a lookup map.** Build a component with three tabs (`'overview' | 'specs' | 'reviews'`) where the active panel comes from a lookup object, and the tab buttons are themselves rendered with `.map()` over the object keys.

### Solutions

```jsx
// Solution 1 — Status switch via lookup map (clean for >2 branches)
const NOTICES = {
  saving: { color: 'goldenrod', text: 'Saving…' },
  error: { color: 'crimson', text: 'Failed to save' },
  success: { color: 'green', text: 'Saved' },
};

function Notice({ status }) {
  const notice = NOTICES[status];
  if (!notice) return null; // covers 'idle' and any unknown status
  return <p style={{ color: notice.color }}>{notice.text}</p>;
}
```

```jsx
// Solution 2 — Fix the leak with a boolean comparison
function Inbox({ unread }) {
  return <header>{unread > 0 && <Dot />}</header>;
}
// (Boolean(unread) && ... or unread ? <Dot/> : null are equally valid.)
```

```jsx
// Solution 3 — Filter + sort + empty state
function ProductList({ products }) {
  const visible = [...products]
    .filter((p) => p.inStock)
    .sort((a, b) => a.price - b.price);

  if (visible.length === 0) return <p>Nothing in stock</p>;

  return (
    <ul>
      {visible.map((p) => (
        <li key={p.id}>
          {p.name} — ${p.price.toFixed(2)}
        </li>
      ))}
    </ul>
  );
}
```

```jsx
// Solution 4 — The bug, then the fix
import { useState } from 'react';

// 🐛 Buggy version: key={i}. Type in the top input, click "Add", watch it desync.
function Buggy() {
  const [rows, setRows] = useState([{ id: 1, label: 'one' }]);
  const add = () =>
    setRows((r) => [{ id: Date.now(), label: 'new' }, ...r]); // prepend
  return (
    <div>
      <button onClick={add}>Add to top</button>
      {rows.map((row, i) => (
        <input key={i} defaultValue={row.label} /> // ❌ index key
      ))}
    </div>
  );
}

// ✅ Fixed: key={row.id} — typed text stays with its row through the prepend.
function Fixed() {
  const [rows, setRows] = useState([{ id: 1, label: 'one' }]);
  const add = () => setRows((r) => [{ id: Date.now(), label: 'new' }, ...r]);
  return (
    <div>
      <button onClick={add}>Add to top</button>
      {rows.map((row) => (
        <input key={row.id} defaultValue={row.label} />
      ))}
    </div>
  );
}
```

```jsx
// Solution 5 — Tabbed view with a lookup map + mapped tab buttons
import { useState } from 'react';

const PANELS = {
  overview: () => <p>Overview content</p>,
  specs: () => <p>Technical specs</p>,
  reviews: () => <p>Customer reviews</p>,
};

export default function Tabs() {
  const [active, setActive] = useState('overview');
  const Panel = PANELS[active];
  return (
    <div>
      <nav>
        {Object.keys(PANELS).map((name) => (
          <button
            key={name}
            aria-pressed={active === name}
            onClick={() => setActive(name)}
          >
            {name}
          </button>
        ))}
      </nav>
      <Panel />
    </div>
  );
}
```

---

## Key takeaways

- Conditional rendering is plain JavaScript: `if`/early `return` for whole-component branches, ternary for inline either/or, `&&` for "render or nothing," lookup map/`switch` for many branches.
- `null`, `undefined`, `false`, and `true` render nothing — but `0` and `NaN` render as visible text, which is the source of the `&&` leak bug. Guard with `> 0`, `Boolean()`, or a ternary.
- A list is just an array of elements produced by `.map()`; React renders arrays in order.
- Every mapped element needs a `key` that is **stable** and **unique among siblings**, placed on the outermost returned element (or `<Fragment>`).
- Keys drive **reconciliation** — identity-based matching instead of position-based — so item state and DOM survive reorders.
- **Index keys break reorderable/filterable lists**; use the data's own id. Never use random keys.
- Compute filtering and sorting into a derived variable before `return`, and copy the array before `.sort()` to avoid mutating state.
- In a `.map` arrow, use a parenthesized implicit return or an explicit `return` — bare braces return `undefined` and render nothing.

---

## Further reading

- React docs — [Conditional Rendering](https://react.dev/learn/conditional-rendering)
- React docs — [Rendering Lists](https://react.dev/learn/rendering-lists)
- React docs — [Keeping list items in order with `key`](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key)
- React docs — [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state) (how keys reset a subtree)
- React reference — [`<Fragment>` (`<>…</>`)](https://react.dev/reference/react/Fragment)
- React reference — [`createElement` / how arrays-as-children work](https://react.dev/reference/react/createElement)
- MDN — [`Array.prototype.map()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map), [`filter()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter), and [`sort()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort)
- See also `07-state-usestate.md` (state drives conditions/lists) and `08-useeffect-side-effects.md` (loading/error states from async data).
