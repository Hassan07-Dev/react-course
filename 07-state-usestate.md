# 07 — State & the useState Hook

> **What you'll learn:** how to give a component *memory* with `useState`, why state is a snapshot rather than a live variable, and how to update state correctly — including objects and arrays — without introducing the classic stale-value bugs.

By the end of this lesson you will be able to:

- Explain what **state** is and how it differs from **props**.
- Declare state with `const [value, setValue] = useState(initial)` and read/update it.
- Use **lazy initialization** to avoid recomputing an expensive initial value on every render.
- Reason about "**state as a snapshot**" and why reading state right after setting it returns the *old* value.
- Choose between the **direct** updater (`setX(next)`) and the **functional** updater (`setX(prev => …)`), and know when the functional form is mandatory.
- Understand **automatic batching** and how React 19 groups multiple `setState` calls into one render.
- Update **objects and arrays immutably** with spread, `map`, `filter`, and `concat`.
- Decide between **multiple state variables vs one object**, avoid storing **derived values**, and **reset state with a `key`**.

---

## Prerequisites

You should already be comfortable with:

- **JSX and rendering** — how a component returns JSX that React turns into UI (see `03-jsx-in-depth.md`).
- **Components & props** — passing read-only data down from parent to child (see `04-components-and-props.md`).
- **Event handling** — wiring `onClick`, `onChange`, etc. to functions (see `06-handling-events.md`).
- **The Rules of Hooks** at a basic level: hooks are called at the top level of a component, never inside loops, conditions, or nested functions. We'll reinforce this here, and `09-hooks-deep-dive.md` covers it in depth.

A working Vite + React 19 project (`npm create vite@latest my-app -- --template react`) so you can paste and run every example.

---

## Concept

A pure function of props alone can render UI, but it can't *remember* anything. Click a button, type in a field, toggle a panel open — all of these require the component to hold a value that survives across renders and that, when changed, makes the UI update. That value is **state**.

Think of a component as a function that React calls over and over. Each call produces a fresh batch of JSX. Plain local variables can't help here:

```jsx
function Counter() {
  let count = 0; // reset to 0 on EVERY render — useless as memory
  return <button onClick={() => count++}>{count}</button>;
}
```

Clicking increments a local variable, but React never re-runs the function, and even if it did, `count` would be re-initialized to `0`. The number on screen never changes. State solves both halves of the problem: React **persists** the value between renders, and **re-renders** the component when you change it.

**State vs props.** Props are arguments passed *in* from a parent — the component does not own them and must never mutate them (they are read-only). State is owned *internally* by the component; the component is the only one that can change it, via the setter `useState` hands you. A useful analogy: props are the ingredients a chef receives; state is the chef's own scratchpad of what's currently on the stove. A parent can pass new props on the next render (props *can* change over time — they just change from the outside), but only the component itself can call `setState`.

The mental model that trips up almost every beginner: **state is a snapshot, not a live binding.** Within a single render, the state variable is a fixed value — a photograph of state at the moment that render began. Calling the setter does *not* mutate that variable; it tells React "next time you render me, use this value instead," and schedules a re-render. We'll lean hard on this idea throughout the lesson.

---

## Syntax & API

`useState` is a Hook imported from React:

```jsx
import { useState } from 'react';
```

Its signature:

```js
const [value, setValue] = useState(initialState);
```

- **`initialState`** — the value used on the *first* render only. On every subsequent render React ignores this argument and returns the current state. It can be any value: number, string, boolean, object, array, `null`, or a function (see lazy init below).
- **Return value** — an array of exactly two elements, which we destructure:
  - **`value`** — the current state for *this* render (a snapshot).
  - **`setValue`** — the **setter**. Calling it schedules a re-render with a new value. By convention it's named `set` + the capitalized state name (`count` → `setCount`).

The setter accepts two forms:

```js
// 1. Direct value — "the next state is exactly this"
setValue(nextValue);

// 2. Functional updater — "compute the next state from the previous state"
setValue(prevValue => computeNext(prevValue));
```

**Lazy initialization.** If computing the initial value is expensive, pass a *function* instead of a value. React calls it once, on the first render only:

```js
// Runs expensiveInit() on EVERY render, throwing away the result after the first — wasteful
const [data, setData] = useState(expensiveInit());

// Runs expensiveInit() exactly once, on first render — correct
const [data, setData] = useState(() => expensiveInit());
```

Note the difference: `useState(expensiveInit())` *calls* the function during every render and passes its result; `useState(() => expensiveInit())` passes the function itself, and React invokes it only when it needs an initial value.

Key behavioral notes:

- The setter is **stable** across renders — React guarantees its identity never changes, so it's safe to omit from `useEffect` dependency arrays (more in `08-useeffect-side-effects.md`).
- Updating state to a value that is **`Object.is`-equal** to the current value lets React **bail out** of re-rendering that component (though it may still re-render once before bailing).
- You may declare **as many `useState` calls as you like** in one component; React tracks them by call order, which is why hooks must run unconditionally.

---

## Examples

### Example 1 — Counter (the "hello world" of state)

```jsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

`useState(0)` gives `count = 0` on the first render. Clicking `+1` calls `setCount(count + 1)`, which schedules a re-render. On the next render, `useState` returns the updated value, the JSX re-evaluates, and the new number appears. The `Reset` button shows the direct form setting an explicit value.

### Example 2 — Toggle (boolean state)

```jsx
import { useState } from 'react';

export default function Disclosure() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <section>
      <button onClick={() => setIsOpen(open => !open)}>
        {isOpen ? 'Hide' : 'Show'} details
      </button>
      {isOpen && <p>Here are the secret details.</p>}
    </section>
  );
}
```

Note the functional updater `setIsOpen(open => !open)`: the next value depends on the previous one, so deriving it from `prev` is the robust choice. Also note `{isOpen && <p>…</p>}` — conditional rendering driven by state.

### Example 3 — Why functional updates matter (multiple updates in one event)

```jsx
import { useState } from 'react';

export default function TripleCounter() {
  const [count, setCount] = useState(0);

  function addThreeWrong() {
    setCount(count + 1); // count is 0 (this render's snapshot)
    setCount(count + 1); // still 0 + 1
    setCount(count + 1); // still 0 + 1 → ends at 1, not 3
  }

  function addThreeRight() {
    setCount(c => c + 1); // 0 → 1
    setCount(c => c + 1); // 1 → 2
    setCount(c => c + 1); // 2 → 3 ✅
  }

  return (
    <div>
      <p>{count}</p>
      <button onClick={addThreeWrong}>+3 (broken)</button>
      <button onClick={addThreeRight}>+3 (correct)</button>
    </div>
  );
}
```

In `addThreeWrong`, `count` is a snapshot frozen at `0` for the whole event handler, so all three calls compute `0 + 1`. In `addThreeRight`, each updater receives the *latest queued* state, so they chain `0 → 1 → 2 → 3`. **Rule of thumb: if the next state depends on the previous state, use the functional form.**

### Example 4 — Controlled form field

```jsx
import { useState } from 'react';

export default function NameForm() {
  const [name, setName] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    alert(`Hello, ${name || 'stranger'}!`);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}
        onChange={e => setName(e.target.value)}
        placeholder="Your name"
      />
      <button type="submit">Greet</button>
    </form>
  );
}
```

This is a **controlled input**: state is the single source of truth for the field's value, and `onChange` pushes every keystroke back into state. Forms are covered fully in `11-forms.md`; here it's just an illustration of string state.

### Example 5 — A list: add, remove, edit (immutable updates)

```jsx
import { useState } from 'react';

let nextId = 0;

export default function TodoList() {
  const [todos, setTodos] = useState([]);
  const [draft, setDraft] = useState('');

  function addTodo() {
    const text = draft.trim();
    if (!text) return;
    // Create a NEW array — never todos.push(...)
    setTodos(prev => [...prev, { id: nextId++, text, done: false }]);
    setDraft('');
  }

  function removeTodo(id) {
    setTodos(prev => prev.filter(t => t.id !== id));
  }

  function toggleTodo(id) {
    setTodos(prev =>
      prev.map(t => (t.id === id ? { ...t, done: !t.done } : t))
    );
  }

  function editTodo(id, text) {
    setTodos(prev =>
      prev.map(t => (t.id === id ? { ...t, text } : t))
    );
  }

  return (
    <div>
      <input value={draft} onChange={e => setDraft(e.target.value)} />
      <button onClick={addTodo}>Add</button>
      <ul>
        {todos.map(t => (
          <li key={t.id}>
            <input
              type="checkbox"
              checked={t.done}
              onChange={() => toggleTodo(t.id)}
            />
            <input
              value={t.text}
              onChange={e => editTodo(t.id, e.target.value)}
              style={{ textDecoration: t.done ? 'line-through' : 'none' }}
            />
            <button onClick={() => removeTodo(t.id)}>✕</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Every mutation produces a **brand-new array**: `[...prev, item]` to add, `filter` to remove, `map` returning a new object (`{ ...t, … }`) for the changed row to edit/toggle. We never call `prev.push`, `prev.splice`, or assign `t.done = …`. That immutability is what lets React detect the change and re-render — and it's required for the reconciliation rules in `05-rendering-lists-conditionals.md` to work correctly.

### Example 6 — Lazy initialization from localStorage

```jsx
import { useState } from 'react';

export default function ThemeToggle() {
  // The function runs once; reading localStorage on every render would be wasteful.
  const [theme, setTheme] = useState(() => {
    return localStorage.getItem('theme') ?? 'light';
  });

  function toggle() {
    setTheme(prev => {
      const next = prev === 'light' ? 'dark' : 'light';
      localStorage.setItem('theme', next);
      return next;
    });
  }

  return <button onClick={toggle}>Theme: {theme}</button>;
}
```

The initializer `() => localStorage.getItem('theme') ?? 'light'` runs only on the first render. (Side note: persisting to `localStorage` is more idiomatically done in an effect — see `08-useeffect-side-effects.md` — but inlining it here keeps the lazy-init point front and center.)

---

## Under the hood

**Where does the value live?** Not in your component function — it lives inside React's internal data structure for that component instance (its *fiber*). Each `useState` call appends a "hook" object to a linked list attached to the fiber. On every render React walks that list **in call order**, which is precisely why hooks must be called unconditionally and in the same order each time: React matches the *N*th `useState` call this render to the *N*th slot from last render purely by position. Put a hook behind an `if` and the positions shift, corrupting which slot owns which value. (Full treatment in `09-hooks-deep-dive.md`.)

**What `setState` actually does.** The setter does not change the `value` variable in the closure you're currently running — that variable is a `const` captured at render time and is immutable for the life of that render. Instead the setter pushes an *update* onto that hook's queue and flags the fiber as needing work. React then schedules a re-render. When it re-renders, it replays the queued updates to compute the new value and returns *that* from `useState`. This is the literal meaning of "**state is a snapshot**": the `value` you read is whatever was computed at the start of *this* render and stays fixed until the next render produces a new snapshot.

A consequence that surprises everyone:

```jsx
function handleClick() {
  setCount(count + 1);
  console.log(count); // logs the OLD value, not the incremented one
}
```

The `console.log` runs in the *current* render's scope, where `count` is still the old snapshot. The new value only exists in the *next* render. If you need to act on the value you just computed, compute it into a local variable first.

**Automatic batching (React 18+, still the model in React 19).** When you call `setState` multiple times — whether in an event handler, an effect, a timeout, a promise `.then`, or a native event — React batches all the updates and performs **one** re-render at the end, rather than one render per call. This is why `setCount(count+1)` three times only advances by one (each used the same snapshot), and why functional updaters are the fix: React applies the queued updater functions in sequence against the running result, so `c => c + 1` three times yields `+3` even though it's all one render. Batching is a performance optimization that makes the snapshot model consistent everywhere. (Pre-18, batching only happened inside React event handlers; React 18 made it universal. There is a rarely-needed escape hatch, `flushSync` from `react-dom`, to force a synchronous render — you almost never need it.)

**Bail-out on equal state.** If you call the setter with a value that is `Object.is`-equal to the current state, React may skip re-rendering. This is why mutating an object and setting the *same reference* back (`obj.x = 1; setObj(obj)`) silently does nothing — the reference is unchanged, so React sees "same state" and bails. Producing a new reference (`setObj({ ...obj, x: 1 })`) is what signals a real change.

**Derived state should not be stored.** Anything you can compute from existing state or props during render should be computed during render, not stored in its own state variable. Storing it means you now have two sources of truth that can drift out of sync, plus extra renders to keep them aligned:

```jsx
// ❌ fullName is derived — storing it invites bugs
const [firstName, setFirstName] = useState('Ada');
const [lastName, setLastName] = useState('Lovelace');
const [fullName, setFullName] = useState('Ada Lovelace');

// ✅ just compute it
const fullName = `${firstName} ${lastName}`;
```

If the computation is genuinely expensive, memoize it with `useMemo` (see `09-hooks-deep-dive.md`) — but it still isn't *state*.

**Resetting state with a `key`.** State is tied to a component's position in the tree. If you want to wipe all of a component's internal state and start fresh, change its `key` prop — React treats a different key as a *different* component instance, unmounts the old one (discarding its state) and mounts a new one:

```jsx
<ProfileForm key={selectedUserId} userId={selectedUserId} />
```

When `selectedUserId` changes, the form's internal `useState` values reset automatically — no manual "clear all fields" logic. This is the idiomatic React way to reset a subtree's state.

---

## Common mistakes & pitfalls

**1. Mutating state directly.**

```jsx
// ❌ WRONG — mutates the existing array, same reference, React bails out
function addItem(item) {
  items.push(item);
  setItems(items);
}
```

```jsx
// ✅ RIGHT — new array reference
function addItem(item) {
  setItems(prev => [...prev, item]);
}
```

Mutation breaks change detection and can corrupt features that rely on referential equality (memoization, `key`-based reconciliation). Always produce a new object/array.

**2. Expecting state to update synchronously.**

```jsx
// ❌ WRONG — count is still the old snapshot here
setCount(count + 1);
sendAnalytics(count); // sends the stale value
```

```jsx
// ✅ RIGHT — compute the value, use it, then set
const next = count + 1;
setCount(next);
sendAnalytics(next);
```

**3. Using the direct form for sequential updates.**

```jsx
// ❌ WRONG — all three read the same snapshot, net effect +1
setScore(score + 1);
setScore(score + 1);
setScore(score + 1);
```

```jsx
// ✅ RIGHT — functional updater chains the queued values
setScore(s => s + 1);
setScore(s => s + 1);
setScore(s => s + 1);
```

**4. Calling the initializer instead of passing it (lazy init).**

```jsx
// ❌ WRONG — buildHugeList() runs on EVERY render
const [list, setList] = useState(buildHugeList());
```

```jsx
// ✅ RIGHT — runs once
const [list, setList] = useState(() => buildHugeList());
```

**5. The stale-closure bug in callbacks and intervals.** A function created in one render closes over *that render's* snapshot of state. If it lives on (in a `setInterval`, an event listener, a debounced callback), it keeps seeing the old value.

```jsx
// ❌ WRONG — the interval closes over count = 0 forever, so it sets 1 each tick
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []); // empty deps → effect runs once → count is permanently 0 in this closure
```

```jsx
// ✅ RIGHT — functional updater always sees the latest state, no stale closure
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

The functional updater sidesteps the closure entirely because it asks React for the latest value rather than relying on the captured one. Effects and their dependencies are covered in `08-useeffect-side-effects.md`.

**6. Conditionally calling a hook.**

```jsx
// ❌ WRONG — breaks hook ordering, React throws
if (loggedIn) {
  const [name, setName] = useState('');
}
```

```jsx
// ✅ RIGHT — always call hooks at the top level
const [name, setName] = useState('');
if (loggedIn) {
  // use name here
}
```

---

## Best practices

- **Do** name setters consistently: `count` / `setCount`, `user` / `setUser`.
- **Do** use the functional updater (`setX(prev => …)`) whenever the next value depends on the current value, or inside async/delayed callbacks.
- **Do** treat all state as immutable — copy with spread, `map`, `filter`, `slice`, `concat`; never `push`/`splice`/`sort`-in-place or assign to properties.
- **Do** use lazy initialization (`useState(() => …)`) for any non-trivial initial computation.
- **Do** split unrelated values into separate state variables; group values that genuinely change together into one object.
- **Do** compute derived values during render instead of storing them in state.
- **Do** reset a subtree's state by changing its `key` rather than writing manual reset code.
- **Don't** read state immediately after setting it and expect the new value — it's the next render's value.
- **Don't** store props in state just to "copy" them; render from props directly unless you specifically need an editable local copy (and even then, reconsider).
- **Don't** put more in state than necessary — minimal, non-redundant state is easier to keep correct.

---

## Exercises

Try each before reading the solution.

1. **Step counter with bounds.** Build a counter starting at `0` with `+1`, `-1`, and `Reset` buttons. It must never go below `0`. Use functional updates.

2. **Tag input.** Render a text field and an "Add" button. Adding pushes the trimmed, non-empty, non-duplicate value into a list of tags rendered as chips, each with a remove (✕) button. Clear the field after adding.

3. **Settings object.** Hold a single state object `{ darkMode, fontSize, notifications }`. Provide a checkbox for `darkMode`, a number input for `fontSize`, and a checkbox for `notifications` — each updating only its own field immutably.

4. **Derived total (no extra state).** Given a list of `{ name, price, qty }` items in state, render each row plus a grand total. The total must be computed during render, never stored in state. Add a button to bump any item's `qty` by 1.

5. **Reset-by-key.** Render a `<select>` of user IDs and a child `<ProfileEditor userId={…} />` with its own `name`/`email` state. Switching the selected user must reset the editor's fields automatically using the `key` technique — no manual clearing.

### Solutions

**Solution 1 — Step counter with bounds**

```jsx
import { useState } from 'react';

export default function BoundedCounter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
      <button onClick={() => setCount(c => Math.max(0, c - 1))}>-1</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

**Solution 2 — Tag input**

```jsx
import { useState } from 'react';

export default function TagInput() {
  const [draft, setDraft] = useState('');
  const [tags, setTags] = useState([]);

  function addTag() {
    const value = draft.trim();
    if (!value || tags.includes(value)) return;
    setTags(prev => [...prev, value]);
    setDraft('');
  }

  function removeTag(target) {
    setTags(prev => prev.filter(t => t !== target));
  }

  return (
    <div>
      <input
        value={draft}
        onChange={e => setDraft(e.target.value)}
        onKeyDown={e => e.key === 'Enter' && addTag()}
      />
      <button onClick={addTag}>Add</button>
      <ul>
        {tags.map(tag => (
          <li key={tag}>
            {tag} <button onClick={() => removeTag(tag)}>✕</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**Solution 3 — Settings object**

```jsx
import { useState } from 'react';

export default function Settings() {
  const [settings, setSettings] = useState({
    darkMode: false,
    fontSize: 16,
    notifications: true,
  });

  function update(field, value) {
    setSettings(prev => ({ ...prev, [field]: value }));
  }

  return (
    <form>
      <label>
        <input
          type="checkbox"
          checked={settings.darkMode}
          onChange={e => update('darkMode', e.target.checked)}
        />
        Dark mode
      </label>
      <label>
        Font size
        <input
          type="number"
          value={settings.fontSize}
          onChange={e => update('fontSize', Number(e.target.value))}
        />
      </label>
      <label>
        <input
          type="checkbox"
          checked={settings.notifications}
          onChange={e => update('notifications', e.target.checked)}
        />
        Notifications
      </label>
      <pre>{JSON.stringify(settings, null, 2)}</pre>
    </form>
  );
}
```

The single `update(field, value)` helper uses a computed property key `[field]` and spreads the previous object so only one field changes per call.

**Solution 4 — Derived total (no extra state)**

```jsx
import { useState } from 'react';

export default function Cart() {
  const [items, setItems] = useState([
    { id: 1, name: 'Pen', price: 2, qty: 3 },
    { id: 2, name: 'Notebook', price: 5, qty: 1 },
  ]);

  // Derived — computed during render, NOT stored in state
  const total = items.reduce((sum, it) => sum + it.price * it.qty, 0);

  function bump(id) {
    setItems(prev =>
      prev.map(it => (it.id === id ? { ...it, qty: it.qty + 1 } : it))
    );
  }

  return (
    <div>
      <ul>
        {items.map(it => (
          <li key={it.id}>
            {it.name} — ${it.price} × {it.qty} = ${it.price * it.qty}
            <button onClick={() => bump(it.id)}>+1</button>
          </li>
        ))}
      </ul>
      <strong>Total: ${total}</strong>
    </div>
  );
}
```

**Solution 5 — Reset-by-key**

```jsx
import { useState } from 'react';

function ProfileEditor({ userId }) {
  // This local state resets whenever the component is remounted (new key).
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  return (
    <fieldset>
      <legend>Editing user {userId}</legend>
      <input
        placeholder="Name"
        value={name}
        onChange={e => setName(e.target.value)}
      />
      <input
        placeholder="Email"
        value={email}
        onChange={e => setEmail(e.target.value)}
      />
    </fieldset>
  );
}

export default function App() {
  const [userId, setUserId] = useState('u1');

  return (
    <div>
      <select value={userId} onChange={e => setUserId(e.target.value)}>
        <option value="u1">User 1</option>
        <option value="u2">User 2</option>
        <option value="u3">User 3</option>
      </select>

      {/* key change → React remounts ProfileEditor → its state resets */}
      <ProfileEditor key={userId} userId={userId} />
    </div>
  );
}
```

Switching users changes the `key`, so React discards the old `ProfileEditor` instance (and its `name`/`email` state) and mounts a fresh one with empty fields — zero manual reset code.

---

## Key takeaways

- **State is a component's memory**: `useState` persists a value across renders and triggers a re-render when you change it. Props come from outside and are read-only; state is owned and updated internally.
- The signature is `const [value, setValue] = useState(initial)`; the initial value is used only on the first render.
- **State is a snapshot** — within one render the value is fixed, so reading it right after calling the setter returns the *old* value; the new value appears on the next render.
- Use the **functional updater** `setX(prev => …)` whenever the next state depends on the previous one, in batched sequential updates, and in async/delayed callbacks (this also kills **stale-closure** bugs).
- React **batches** multiple `setState` calls into a single re-render everywhere (event handlers, effects, timeouts, promises).
- Always update **immutably** — spread/`map`/`filter`/`concat` to create new objects and arrays; never mutate then set, or React's change detection bails out.
- **Don't store derived values** in state — compute them during render; and **reset a subtree's state by changing its `key`**.
- Use **lazy initialization** (`useState(() => expensiveInit())`) so costly initial computations run once, not every render.

---

## Further reading

- React docs — [`useState` reference](https://react.dev/reference/react/useState)
- React docs — [State: A Component's Memory](https://react.dev/learn/state-a-components-memory)
- React docs — [Render and Commit](https://react.dev/learn/render-and-commit)
- React docs — [State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)
- React docs — [Queueing a Series of State Updates](https://react.dev/learn/queueing-a-series-of-state-updates)
- React docs — [Updating Objects in State](https://react.dev/learn/updating-objects-in-state)
- React docs — [Updating Arrays in State](https://react.dev/learn/updating-arrays-in-state)
- React docs — [Choosing the State Structure](https://react.dev/learn/choosing-the-state-structure)
- React docs — [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- React docs — [`flushSync` reference](https://react.dev/reference/react-dom/flushSync) (the rarely-needed batching escape hatch)
