# 06 — Handling Events

> **What you'll learn:** how to wire user interactions (clicks, typing, submits, key presses, focus) into your React components using JSX event handlers, and how React's SyntheticEvent system really works underneath.

By the end of this lesson you will be able to:

- Attach event handlers in JSX using the correct **camelCase** prop names (`onClick`, `onChange`, `onSubmit`, `onKeyDown`, …).
- Explain the critical difference between **passing a function reference** (`onClick={fn}`) and **calling a function** (`onClick={fn()}`) — and why the latter can cause an infinite render loop.
- Use the **SyntheticEvent** object: `e.preventDefault()`, `e.stopPropagation()`, `e.target`, `e.currentTarget`, and `e.target.value`.
- Pass **arguments** to handlers using arrow wrappers, `.bind`, or `data-*` attributes — and know the tradeoffs.
- Handle **form submission** correctly with `preventDefault`, plus keyboard and focus events.
- Understand React's **event delegation** model (listeners attached at the root container since React 17) and the **capture phase** (`onClickCapture`).

---

## Prerequisites

Before this lesson you should be comfortable with:

- **JSX syntax and expressions** — embedding `{...}` in markup (see `03-jsx-in-depth.md`).
- **Function components and props** (see `04-components-and-props.md`).
- **State with `useState`** — most handlers update state, so you should know how `setState` triggers a re-render (see `07-state-usestate.md`).
- Basic JavaScript: functions, closures, arrow functions, and `Array` methods.

Handlers very often run *side effects* or read DOM nodes; the deeper treatment of effects and refs comes later (see `08-useeffect-side-effects.md` and `09-hooks-deep-dive.md`). This lesson stays focused on the events themselves.

---

## Concept

A user interface is mostly a long conversation: the user does something (clicks, types, submits, hovers), and your app responds. In the DOM, those "somethings" are **events**, and the functions that respond to them are **event handlers** (or *event listeners*).

In plain JavaScript you wire events imperatively:

```js
const button = document.getElementById('buy');
button.addEventListener('click', () => {
  console.log('bought!');
});
```

You first find the node, then attach a listener, and later you must remember to `removeEventListener` to avoid leaks. React flips this around. Instead of reaching into the DOM, you **declare** which handler belongs to which element directly in your JSX:

```jsx
<button onClick={() => console.log('bought!')}>Buy</button>
```

React reads that declaration and manages the actual `addEventListener`/`removeEventListener` bookkeeping for you. When the component unmounts, the wiring is cleaned up automatically. This is the same philosophy as the rest of React: **you describe what the UI should be, React makes the DOM match.** Events are just the part of that description that says "when X happens, run this."

A useful mental model: an event handler prop like `onClick` is *not* a string-valued HTML attribute — it is a real JavaScript value (a function) that you hand to React. React holds onto that function and invokes it when the corresponding browser event fires on (or bubbles up to) that element. Because it's a normal function, it closes over the props and state of the render it was created in — a fact that becomes very important once you learn about stale closures in effects.

---

## Syntax & API

### Attaching a handler

The handler prop name is the DOM event name, **camelCased** and prefixed with `on`:

```jsx
<button onClick={handleClick}>Click</button>
<input onChange={handleChange} />
<form onSubmit={handleSubmit}>…</form>
<div onMouseEnter={handleEnter} onMouseLeave={handleLeave}>…</div>
<input onKeyDown={handleKeyDown} onFocus={handleFocus} onBlur={handleBlur} />
```

| DOM (HTML/JS) | React prop | Fires when |
| --- | --- | --- |
| `click` | `onClick` | element is clicked / activated |
| `input` / `change` | `onChange` | a form control's value changes (React fires on every keystroke) |
| `submit` | `onSubmit` | a `<form>` is submitted |
| `mouseenter` / `mouseleave` | `onMouseEnter` / `onMouseLeave` | pointer enters/leaves (no bubbling) |
| `mouseover` / `mouseout` | `onMouseOver` / `onMouseOut` | pointer enters/leaves (bubbles) |
| `keydown` / `keyup` / `keypress` | `onKeyDown` / `onKeyUp` / `onKeyPress` | keyboard activity |
| `focus` / `blur` | `onFocus` / `onBlur` | element gains/loses focus (React's versions bubble) |
| `input` | `onInput` | raw input event (rarely needed; prefer `onChange`) |

> **Naming gotcha:** It is `onClick`, not `onclick` (lowercase is the DOM property name and React will warn). It is `onChange`, not `onchange`.

### The handler signature

Every handler receives a single argument — the **SyntheticEvent**:

```jsx
function handleClick(event) {
  // event is a React SyntheticEvent
}
```

Common members:

```jsx
event.preventDefault();   // cancel the default browser behavior
event.stopPropagation();  // stop the event from bubbling further up
event.target;             // the DOM node that originally triggered the event
event.currentTarget;      // the DOM node whose handler is currently running
event.target.value;       // for inputs: the current value
event.nativeEvent;        // the underlying browser Event object
event.type;               // 'click', 'change', …
event.key;                // for keyboard events: 'Enter', 'Escape', 'a', …
event.shiftKey;           // boolean modifier flags: shiftKey/ctrlKey/altKey/metaKey
```

### Reference vs. call — the one rule you must internalize

```jsx
// ✅ Pass the function REFERENCE. React calls it later, on click.
<button onClick={handleClick}>OK</button>

// ✅ Pass an inline arrow function (also a reference) so you can add args.
<button onClick={() => handleClick(id)}>OK</button>

// ❌ This CALLS handleClick immediately during render and passes its
//    RETURN VALUE as the handler. Almost never what you want.
<button onClick={handleClick()}>OK</button>
```

### Capture-phase handlers

Append `Capture` to register during the capture phase instead of bubbling:

```jsx
<div onClickCapture={onCapture} onClick={onBubble}>…</div>
```

---

## Examples

### Example 1 — A counter (reference vs. call)

```jsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount((c) => c + 1);
  }

  return (
    <div>
      <p>Count: {count}</p>
      {/* Pass the reference — React calls increment on each click */}
      <button onClick={increment}>+1</button>

      {/* Inline arrow when you need an argument */}
      <button onClick={() => setCount((c) => c + 5)}>+5</button>

      {/* Reset to zero */}
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

We pass `increment` *by reference*. If we had written `onClick={increment()}`, `increment` would run during render, call `setCount`, schedule another render, run again… an infinite loop (see *Common mistakes*).

### Example 2 — Controlled input with `onChange` and `e.target.value`

```jsx
import { useState } from 'react';

export default function NameField() {
  const [name, setName] = useState('');

  function handleChange(e) {
    // e.target is the <input>; .value is its current text
    setName(e.target.value);
  }

  return (
    <div>
      <label>
        Name:{' '}
        <input type="text" value={name} onChange={handleChange} />
      </label>
      <p>Hello, {name || 'stranger'}!</p>
    </div>
  );
}
```

React fires `onChange` on **every keystroke** (unlike the native DOM `change`, which only fires on blur). Reading `e.target.value` and storing it in state is the *controlled component* pattern — the input's displayed value is driven by state. Forms get a full lesson of their own (see `11-forms.md`).

### Example 3 — Form submit with `preventDefault`

```jsx
import { useState } from 'react';

export default function SearchForm() {
  const [query, setQuery] = useState('');
  const [submitted, setSubmitted] = useState(null);

  function handleSubmit(e) {
    e.preventDefault(); // stop the browser's full-page reload
    setSubmitted(query.trim());
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search…"
      />
      <button type="submit">Go</button>
      {submitted && <p>You searched for: {submitted}</p>}
    </form>
  );
}
```

Without `e.preventDefault()`, pressing Enter or clicking the submit button would reload the page (the default HTML form behavior), wiping your React state. Putting `onSubmit` on the `<form>` (not `onClick` on the button) is the correct, accessible choice: it also fires when the user presses Enter inside a field.

### Example 4 — Passing arguments three ways

```jsx
const products = [
  { id: 'a1', name: 'Coffee' },
  { id: 'b2', name: 'Tea' },
];

export default function ProductList({ onBuy }) {
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>
          {p.name}

          {/* 1) Arrow wrapper — most common, most readable */}
          <button onClick={() => onBuy(p.id)}>Buy A</button>

          {/* 2) Function.prototype.bind — pre-applies args, event comes last */}
          <button onClick={onBuy.bind(null, p.id)}>Buy B</button>

          {/* 3) data-* attribute — no per-row closure created */}
          <button data-id={p.id} onClick={handleBuy}>Buy C</button>
        </li>
      ))}
    </ul>
  );

  function handleBuy(e) {
    // currentTarget is the button the handler is attached to
    onBuy(e.currentTarget.dataset.id);
  }
}
```

The arrow wrapper (1) is the idiomatic choice. `bind` (2) reads awkwardly because the event ends up as a *trailing* argument: `onBuy(id, event)`. The `data-*` approach (3) keeps a single shared handler — useful in tight lists where you want to avoid allocating one closure per item (though in practice this micro-optimization rarely matters; see *Under the hood*).

### Example 5 — Keyboard and focus events

```jsx
import { useState } from 'react';

export default function CommandBox() {
  const [value, setValue] = useState('');
  const [log, setLog] = useState([]);

  function handleKeyDown(e) {
    if (e.key === 'Enter') {
      setLog((prev) => [...prev, value]);
      setValue('');
    } else if (e.key === 'Escape') {
      setValue('');
    }
  }

  return (
    <div>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onKeyDown={handleKeyDown}
        onFocus={() => console.log('focused')}
        onBlur={() => console.log('blurred')}
        placeholder="Type and press Enter"
      />
      <ul>{log.map((line, i) => <li key={i}>{line}</li>)}</ul>
    </div>
  );
}
```

`e.key` is the modern, readable property (`'Enter'`, `'Escape'`, `'ArrowUp'`, `'a'`). Prefer it over the deprecated `e.keyCode`. Use `onKeyDown` rather than `onKeyPress` (the latter is deprecated).

### Example 6 — Bubbling, capture, and stopping propagation

```jsx
export default function PropagationDemo() {
  return (
    <div
      onClickCapture={() => console.log('1. parent CAPTURE')}
      onClick={() => console.log('4. parent BUBBLE')}
    >
      <button
        onClickCapture={() => console.log('2. button CAPTURE')}
        onClick={(e) => {
          console.log('3. button BUBBLE');
          // e.stopPropagation(); // uncomment to silence step 4
        }}
      >
        Click me
      </button>
    </div>
  );
}
```

Click the button and the console prints `1 → 2 → 3 → 4`. The event first travels **down** through capture-phase handlers (parent then child), then **up** through bubble-phase handlers (child then parent). Calling `e.stopPropagation()` inside the button's bubble handler suppresses step 4.

---

## Under the hood

### React doesn't attach a listener per element

When you write a hundred `onClick` props, React does **not** call `addEventListener` a hundred times. Instead React uses **event delegation**: it attaches a *single* listener per event type and lets native DOM bubbling carry events up to it, then dispatches to your handlers.

**Where that root listener lives changed in React 17.** In React 16 and earlier, React attached its delegated listeners to `document`. Since React 17 (and still true in React 19), React attaches them to the **root DOM container** — the node you pass to `createRoot(domNode)`. This matters in real apps:

- You can run **multiple React roots** (or React alongside jQuery / another framework / a different React version) on one page without their event systems fighting over `document`.
- A `e.stopPropagation()` inside a React tree now stops propagation *within that root's container*, behaving more like you'd intuit.

```jsx
// main.jsx — listeners are delegated to #root, not document
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(<App />);
```

### What a SyntheticEvent is

The object passed to your handler is a **`SyntheticEvent`** — React's cross-browser wrapper around the native browser event. Its purpose is to normalize differences between browsers so that `e.target`, `e.key`, `e.preventDefault()`, etc. behave identically everywhere. The real browser event is always available at `e.nativeEvent` if you need it.

`SyntheticEvent` implements the same interface as the DOM Event spec, so `preventDefault()` and `stopPropagation()` work as you expect, and `e.bubbles`, `e.type`, `e.currentTarget` are all present.

### The pooling history (important for understanding old code)

In React 16 and earlier, SyntheticEvents were **pooled**: after the handler finished running synchronously, React would *reset all the event's properties to `null`* and reuse the object for the next event, as a performance optimization. This meant you could not access the event asynchronously:

```jsx
// React 16 ONLY — this logged null because the event was recycled
function handleClick(e) {
  setTimeout(() => console.log(e.type), 0); // -> null in React 16
}
```

To use an event later you had to call `e.persist()`. **Event pooling was removed in React 17.** In React 17, 18, and 19, the event object is no longer recycled, `e.persist()` is a harmless no-op, and you can freely read the event inside a `setTimeout`, a promise, or an async function. If you see `e.persist()` in a codebase, it's legacy and safe to delete on modern React.

### `target` vs `currentTarget`

- `e.target` — the actual DOM node where the event originated (e.g. the `<span>` inside a button you clicked).
- `e.currentTarget` — the node whose handler is *currently executing* (the element you put `onClick` on).

During delegation, `currentTarget` is set fresh for each handler as React walks the tree, which is why it's reliable inside the handler but **not** afterward — by the time an async callback runs, `currentTarget` may read `null`. Capture a value synchronously (`const id = e.currentTarget.dataset.id;`) if you need it later.

### Closures, not strings

Because each handler is a closure created during a render, it "remembers" the props and state of *that* render. Update state and the component re-renders, producing **new** handler closures bound to the **new** state. This is why `onClick={increment}` always sees fresh state through the functional updater — and why, in effects, you can get *stale* closures (covered in `08-useeffect-side-effects.md`).

---

## Common mistakes & pitfalls

### 1) Calling the handler instead of passing it (the infinite loop)

```jsx
// ❌ WRONG: setCount runs during render → re-render → runs again → loop
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={setCount(count + 1)}>{count}</button>;
}
```

```jsx
// ✅ RIGHT: pass a function React can call later
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

`onClick={setCount(count + 1)}` *invokes* `setCount` while rendering. That schedules a render, which runs the call again — React throws **"Too many re-renders."** Wrap it in an arrow function so it only runs on click.

### 2) Forgetting `preventDefault` on forms

```jsx
// ❌ WRONG: the page reloads on submit, blowing away React state
<form onSubmit={() => save(data)}>…</form>
```

```jsx
// ✅ RIGHT
<form onSubmit={(e) => { e.preventDefault(); save(data); }}>…</form>
```

### 3) Lowercase or string handlers (HTML habits)

```jsx
// ❌ WRONG: onclick is ignored by React; string is not a function
<button onclick="doThing()">Go</button>
```

```jsx
// ✅ RIGHT: camelCase prop, function value
<button onClick={doThing}>Go</button>
```

### 4) Reading the event asynchronously and assuming `currentTarget` survives

```jsx
// ❌ RISKY: currentTarget is null by the time the timer fires
function handleClick(e) {
  setTimeout(() => console.log(e.currentTarget.dataset.id), 100); // throws
}
```

```jsx
// ✅ RIGHT: read what you need synchronously, then close over it
function handleClick(e) {
  const id = e.currentTarget.dataset.id;
  setTimeout(() => console.log(id), 100);
}
```

(`e.target` is generally still valid asynchronously on React 17+, but `currentTarget` is reset after dispatch — capturing the value is the safe habit.)

### 5) Using `onChange` like the native DOM `change`

```jsx
// ❌ Misconception: "onChange only fires when the field loses focus"
// React's onChange fires on EVERY keystroke. If you truly want
// fire-on-blur semantics, use onBlur instead.
<input onBlur={(e) => validate(e.target.value)} />
```

### 6) Putting submit logic on the button's `onClick`

```jsx
// ❌ WRONG: won't fire when the user presses Enter in a field, and bypasses
//    native form semantics/validation
<form>
  <input />
  <button onClick={save}>Save</button>
</form>
```

```jsx
// ✅ RIGHT: handle submit on the form
<form onSubmit={(e) => { e.preventDefault(); save(); }}>
  <input />
  <button type="submit">Save</button>
</form>
```

---

## Best practices

- **Do** pass a function *reference* (`onClick={fn}`) or an inline arrow (`onClick={() => fn(x)}`). **Don't** call it (`onClick={fn()}`).
- **Do** use camelCase event prop names (`onClick`, `onKeyDown`).
- **Do** handle form submission via `onSubmit` on the `<form>` with `e.preventDefault()`, and give the submit button `type="submit"`.
- **Do** extract handlers into named functions when they grow beyond a line or two — easier to read, test, and debug than long inline arrows. Keep tiny one-liners inline.
- **Do** use the functional updater (`setCount(c => c + 1)`) inside handlers when the new state depends on the old (see `07-state-usestate.md`).
- **Do** prefer `e.key === 'Enter'` over deprecated `e.keyCode`/`onKeyPress`.
- **Don't** prematurely worry about "a new arrow function on every render." It's almost always fine; reach for `useCallback` only when profiling shows a real problem with memoized children (see `13-performance.md`).
- **Don't** call `e.persist()` in new code — pooling is gone since React 17.
- **Don't** reach into the DOM with `addEventListener` for things React already covers; use the JSX props. Use refs + native listeners only for events React doesn't expose declaratively (covered in `09-hooks-deep-dive.md`).
- **Don't** rely on `e.currentTarget` after an `await` or inside a timer — capture the value first.

---

## Exercises

1. **Toggle button.** Build a `Toggle` component with a single button that flips between "ON" and "OFF" each click. Use a boolean in `useState`.

2. **Live character counter.** Build a `<textarea>` whose value is stored in state, with a label showing "N / 280 characters". Turn the count red (via inline style) when it exceeds 280. Use `onChange` and `e.target.value`.

3. **Keyboard-driven todo input.** An input that adds its trimmed text to a list when the user presses **Enter**, and clears the input when the user presses **Escape**. Ignore empty submissions. Use `onKeyDown` and `e.key`.

4. **Stop the bubble.** Build a clickable "card" `<div>` (logs `"card clicked"`) containing a "Delete" button (logs `"deleted"`). Clicking Delete must **not** also trigger the card's click. Use `e.stopPropagation()`.

5. **Argument passing, three ways.** Render a list of three colors as buttons. Clicking a button sets the page background to that color. Implement one button with an arrow wrapper, one with `.bind`, and one with a `data-color` attribute read via `e.currentTarget.dataset`.

### Solutions

```jsx
// 1) Toggle button
import { useState } from 'react';

export default function Toggle() {
  const [on, setOn] = useState(false);
  return (
    <button onClick={() => setOn((prev) => !prev)}>
      {on ? 'ON' : 'OFF'}
    </button>
  );
}
```

```jsx
// 2) Live character counter
import { useState } from 'react';

export default function Tweet() {
  const [text, setText] = useState('');
  const over = text.length > 280;
  return (
    <div>
      <textarea value={text} onChange={(e) => setText(e.target.value)} />
      <p style={{ color: over ? 'red' : 'inherit' }}>
        {text.length} / 280 characters
      </p>
    </div>
  );
}
```

```jsx
// 3) Keyboard-driven todo input
import { useState } from 'react';

export default function TodoInput() {
  const [text, setText] = useState('');
  const [todos, setTodos] = useState([]);

  function handleKeyDown(e) {
    if (e.key === 'Enter') {
      const trimmed = text.trim();
      if (!trimmed) return;
      setTodos((prev) => [...prev, trimmed]);
      setText('');
    } else if (e.key === 'Escape') {
      setText('');
    }
  }

  return (
    <div>
      <input
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="Add a todo, press Enter"
      />
      <ul>{todos.map((t, i) => <li key={i}>{t}</li>)}</ul>
    </div>
  );
}
```

```jsx
// 4) Stop the bubble
export default function Card() {
  function handleCard() {
    console.log('card clicked');
  }
  function handleDelete(e) {
    e.stopPropagation(); // prevents handleCard from also running
    console.log('deleted');
  }
  return (
    <div onClick={handleCard} style={{ border: '1px solid #ccc', padding: 16 }}>
      <p>A clickable card</p>
      <button onClick={handleDelete}>Delete</button>
    </div>
  );
}
```

```jsx
// 5) Argument passing, three ways
const COLORS = ['tomato', 'gold', 'mediumseagreen'];

export default function ColorPicker() {
  function setBg(color) {
    document.body.style.backgroundColor = color;
  }
  function handleData(e) {
    setBg(e.currentTarget.dataset.color);
  }

  return (
    <div>
      {/* arrow wrapper */}
      <button onClick={() => setBg(COLORS[0])}>{COLORS[0]}</button>
      {/* bind */}
      <button onClick={setBg.bind(null, COLORS[1])}>{COLORS[1]}</button>
      {/* data-* attribute */}
      <button data-color={COLORS[2]} onClick={handleData}>{COLORS[2]}</button>
    </div>
  );
}
```

---

## Key takeaways

- Wire events declaratively in JSX with **camelCase** props (`onClick`, `onChange`, `onSubmit`, `onKeyDown`, `onFocus`, …); React manages the actual DOM listeners.
- **Pass a function, never call it:** `onClick={fn}` or `onClick={() => fn(x)}`, never `onClick={fn()}` — calling during render can cause an infinite re-render loop.
- Handlers receive a **SyntheticEvent**, a cross-browser wrapper with `preventDefault()`, `stopPropagation()`, `target`, `currentTarget`, and `nativeEvent`.
- **Event pooling was removed in React 17** — you can read events asynchronously now, and `e.persist()` is obsolete.
- Pass arguments via **arrow wrapper** (idiomatic), `.bind`, or **`data-*` attributes** read from `e.currentTarget.dataset`.
- Handle forms with **`onSubmit` + `e.preventDefault()`** on the `<form>`, and remember React's `onChange` fires on every keystroke.
- Events follow **capture then bubble**; use `onClickCapture` for the capture phase and `e.stopPropagation()` to halt bubbling.
- Since **React 17**, delegated listeners live on the **root container**, not `document` — which makes multiple roots and stopPropagation behave sanely.

---

## Further reading

- React docs — [Responding to Events](https://react.dev/learn/responding-to-events)
- React docs — [`<input>` / common components: event handler props & SyntheticEvent reference](https://react.dev/reference/react-dom/components/common#react-event-object)
- React docs — [Reacting to Input with State](https://react.dev/learn/reacting-to-input-with-state) (events that drive state)
- React 17 blog — [Changes to Event Delegation](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#changes-to-event-delegation) (root container) and [No Event Pooling](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#no-event-pooling)
- MDN — [Introduction to events](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events) and [Event bubbling and capture](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Event_bubbling)
- MDN — [`KeyboardEvent.key` values](https://developer.mozilla.org/en-US/docs/Web/API/UI_Events/Keyboard_event_key_values)
