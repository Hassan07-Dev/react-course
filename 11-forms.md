# 11 — Forms & Inputs

> **What you'll learn:** how to capture user input in React the idiomatic way — making React state the single source of truth with *controlled components* — across every kind of input (text, textarea, select, checkbox, radio, number, file), how to scale to many fields with one handler, how to submit and validate forms, when *uncontrolled* components with refs are the better tool, and a first look at React 19's built-in form Actions and the popular form libraries you'll eventually reach for.

By the end of this lesson you will be able to:

- Build **controlled inputs** where `value`/`checked` is driven by state and `onChange` writes it back.
- Correctly wire up every native input type, including the special cases (`<select multiple>`, checkboxes vs. radios, `<input type="file">`).
- Handle **many fields with a single `onChange`** using the `name` attribute and computed object keys.
- Submit forms with `onSubmit` + `e.preventDefault()` and read all field values reliably.
- Implement **uncontrolled components** with `useRef` + `defaultValue`/`defaultChecked`, and know when they beat controlled ones.
- Add **client-side validation** — native (`required`, `pattern`, `type`) and custom — with error messages and a disabled submit button.
- Recognize what **React 19 form Actions / `useActionState` / `useFormStatus`** give you (full treatment in `22-modern-react-19.md`).
- Decide *when* to reach for a library like **React Hook Form** or **Formik**, and write a minimal RHF form.

---

## Prerequisites

You should already be comfortable with:

- **JSX** and how attributes map to DOM props — see `03-jsx-in-depth.md` (note `className`, `htmlFor`, and camelCased DOM props).
- **Components & props** — see `04-components-and-props.md`.
- **Event handling** and the SyntheticEvent (`onChange`, `onSubmit`, `e.target`, `e.preventDefault()`) — see `06-handling-events.md`.
- **`useState`** and the rule that state updates trigger re-renders, plus updating objects immutably — see `07-state-usestate.md`.
- **Refs** are introduced here for uncontrolled inputs; the deep dive is in the refs lesson, but everything you need is shown inline.

---

## Concept

In plain HTML, form elements own their own state. You type into an `<input>` and the browser's DOM node remembers what you typed — your JavaScript only finds out if it goes and *asks* the DOM (`inputEl.value`). The DOM is the source of truth.

React flips this. The central React philosophy is **UI is a function of state**: given the same state, you always render the same UI. An input that secretly holds its own value *outside* of React state breaks that contract — now there are two sources of truth (React's idea of the value, and the DOM's actual value) and they can drift apart.

The React-idiomatic fix is the **controlled component**:

1. You store the field's value in React state (`useState`).
2. You feed that state *into* the input via the `value` prop (or `checked` for checkboxes/radios).
3. On every keystroke, the input fires `onChange`; your handler reads the new value and writes it back to state.

That creates a tight loop:

```
state ──(value prop)──▶ <input> ──(onChange)──▶ setState ──▶ re-render ──▶ state ...
```

The input never moves on its own. React renders it with whatever the state says, the user's keystroke is just a *request* to change state, and the change only becomes visible after React re-renders. This is sometimes called a **one-way data binding completed by an event handler** — distinct from the two-way binding in frameworks like Angular or Vue's `v-model`, though the net effect feels similar.

**Analogy.** A controlled input is like a thermostat display that shows the temperature stored in a central computer. Turning the dial doesn't *directly* change the number on the screen — it sends a request to the computer, the computer updates its stored value, and *then* the screen redraws. The screen can never show a temperature the computer doesn't know about. An uncontrolled input, by contrast, is a sticky note on the fridge: it holds its own value, and you only know what's on it when you walk over and read it.

Why bother with the loop instead of letting the DOM hold the value? Because once the value lives in state you get, for free: instant validation as the user types, conditionally enabling/disabling other UI, formatting input (uppercase, masking, trimming), driving multiple inputs from one piece of data, and a value you can submit, reset, or persist with zero DOM reading. The cost is a re-render per keystroke — usually negligible, and addressable with libraries when it isn't.

---

## Syntax & API

### Controlled text-like inputs

```jsx
const [value, setValue] = useState('');

<input
  type="text"          // also: email, password, search, tel, url, number, ...
  value={value}        // state drives the DOM
  onChange={(e) => setValue(e.target.value)}  // DOM event writes state back
/>
```

| Element | Controlled by | Read the new value from |
| --- | --- | --- |
| `<input type="text\|email\|password\|number\|...">` | `value` (string) | `e.target.value` (always a **string**) |
| `<textarea>` | `value` (string, **not** children) | `e.target.value` |
| `<select>` | `value` (string) | `e.target.value` |
| `<select multiple>` | `value` (array of strings) | `Array.from(e.target.selectedOptions, o => o.value)` |
| `<input type="checkbox">` | `checked` (boolean) | `e.target.checked` |
| `<input type="radio">` | `checked` (boolean, per option) | `e.target.value` of the chosen one |
| `<input type="file">` | **cannot** be controlled | `e.target.files` (a `FileList`) |

Key annotations:

- **`value` is always a string** in the DOM. `type="number"` still gives you `e.target.value` as a string (`"42"`), not a number. Coerce with `Number(e.target.value)` if you need a number — but watch for the empty-string case (`Number("")` is `0`, which is usually wrong).
- **`<textarea>` uses `value`, not children.** In HTML you write `<textarea>hi</textarea>`; in React you write `<textarea value={text} onChange={...} />`.
- **`<select>` uses `value` on the `<select>` itself**, not `selected` on `<option>` (that's the HTML way).
- **`onChange` in React fires on every keystroke**, like the native `input` event — not only on blur like the native DOM `change` event. See `06-handling-events.md`.

### The single-handler pattern (`name` + computed key)

```jsx
function handleChange(e) {
  const { name, value, type, checked } = e.target;
  setForm((prev) => ({
    ...prev,
    [name]: type === 'checkbox' ? checked : value, // computed property key
  }));
}
```

Each input carries a `name` matching a key in your state object; one handler updates the right key via `[name]`.

### Submission

```jsx
<form onSubmit={handleSubmit}>...</form>

function handleSubmit(e) {
  e.preventDefault();   // stop the browser's full-page navigation/reload
  // ...use state here...
}
```

### Uncontrolled inputs (refs)

```jsx
const inputRef = useRef(null);

<input ref={inputRef} defaultValue="initial" />   // defaultValue, NOT value
// later: inputRef.current.value
```

- Use **`defaultValue`** / **`defaultChecked`** to set the *initial* value of an uncontrolled input. After that, the DOM owns it.
- Read the current value imperatively from `ref.current.value` (or `.checked`, `.files`).

### React 19 form Actions (preview)

```jsx
// Full treatment in 22-modern-react-19.md
<form action={async (formData) => { await save(formData.get('email')); }} />

const [state, formAction, isPending] = useActionState(actionFn, initialState);
const { pending } = useFormStatus(); // inside a child of a <form>
```

---

## Examples

### Example 1 — A single controlled text input

The "hello world" of forms. Note the live preview that re-renders on every keystroke — proof the state is the source of truth.

```jsx
import { useState } from 'react';

export default function NameField() {
  const [name, setName] = useState('');

  return (
    <div>
      <label htmlFor="name">Name</label>
      <input
        id="name"
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Type your name"
      />
      <p>Hello, {name || 'stranger'}!</p>
      <p>{name.length} characters</p>
    </div>
  );
}
```

Because `value={name}`, the input can *only* ever show what's in state. The `onChange` handler is the single doorway through which the value changes. Try adding `value={name.toUpperCase()}` and you'll see the input force-uppercase as you type — that's the power of controlling the value.

### Example 2 — Every input type, one state object

A realistic sign-up form covering text, number, textarea, single select, multiple select, checkbox, and a radio group — all driven by one state object and one `handleChange`.

```jsx
import { useState } from 'react';

const INITIAL = {
  username: '',
  age: '',
  bio: '',
  country: 'us',
  languages: [],     // multi-select → array
  newsletter: false, // checkbox → boolean
  plan: 'free',      // radio group → string
};

export default function SignUpForm() {
  const [form, setForm] = useState(INITIAL);

  function handleChange(e) {
    const { name, value, type, checked, selectedOptions } = e.target;

    setForm((prev) => {
      if (type === 'checkbox') return { ...prev, [name]: checked };
      if (type === 'select-multiple') {
        return { ...prev, [name]: Array.from(selectedOptions, (o) => o.value) };
      }
      return { ...prev, [name]: value };
    });
  }

  function handleSubmit(e) {
    e.preventDefault();
    // age is still a string here ("29"); coerce when you actually need a number:
    const payload = { ...form, age: form.age === '' ? null : Number(form.age) };
    console.log('Submitting', payload);
  }

  function handleReset() {
    setForm(INITIAL); // resetting is just setting state back
  }

  return (
    <form onSubmit={handleSubmit} onReset={handleReset}>
      <label>
        Username
        <input name="username" value={form.username} onChange={handleChange} />
      </label>

      <label>
        Age
        <input name="age" type="number" min="0" value={form.age} onChange={handleChange} />
      </label>

      <label>
        Bio
        <textarea name="bio" value={form.bio} onChange={handleChange} rows={3} />
      </label>

      <label>
        Country
        <select name="country" value={form.country} onChange={handleChange}>
          <option value="us">United States</option>
          <option value="pk">Pakistan</option>
          <option value="de">Germany</option>
        </select>
      </label>

      <label>
        Languages (Ctrl/Cmd-click for multiple)
        <select name="languages" multiple value={form.languages} onChange={handleChange}>
          <option value="js">JavaScript</option>
          <option value="py">Python</option>
          <option value="go">Go</option>
        </select>
      </label>

      <label>
        <input
          name="newsletter"
          type="checkbox"
          checked={form.newsletter}
          onChange={handleChange}
        />
        Subscribe to newsletter
      </label>

      <fieldset>
        <legend>Plan</legend>
        {['free', 'pro', 'team'].map((plan) => (
          <label key={plan}>
            <input
              name="plan"
              type="radio"
              value={plan}
              checked={form.plan === plan}
              onChange={handleChange}
            />
            {plan}
          </label>
        ))}
      </fieldset>

      <button type="submit">Create account</button>
      <button type="reset">Reset</button>

      <pre>{JSON.stringify(form, null, 2)}</pre>
    </form>
  );
}
```

Things to notice:

- **One handler, many fields.** The `name` attribute selects the state key via `[name]`; the `type` tells the handler whether to read `.checked`, `selectedOptions`, or `.value`.
- **Radio groups** share a `name` so the browser treats them as one group; each option's `checked` is `form.plan === plan`. The selected value comes from `e.target.value`.
- **Multiple select** stores an array and is read with `Array.from(e.target.selectedOptions, o => o.value)`. `e.target.type` for a `<select multiple>` is the string `"select-multiple"`.
- **The `age` number stays a string in state.** That's fine — coerce only at submit time (and handle empty string explicitly), which sidesteps a class of `NaN`/`0` bugs.
- **Reset is just `setForm(INITIAL)`** — no DOM clearing needed, because the DOM is a projection of state.

### Example 3 — The file input (the special case)

`<input type="file">` is **always uncontrolled**: its value is set by the user picking a file, and for security reasons JavaScript cannot programmatically set it. So you never give it a `value` prop — you read from `e.target.files` (or a ref) in `onChange`.

```jsx
import { useState } from 'react';

export default function AvatarUpload() {
  const [file, setFile] = useState(null);
  const [previewUrl, setPreviewUrl] = useState('');

  function handleFileChange(e) {
    const picked = e.target.files?.[0] ?? null;
    setFile(picked);
    setPreviewUrl(picked ? URL.createObjectURL(picked) : '');
  }

  return (
    <div>
      {/* No value prop — file inputs are read-only from JS */}
      <input type="file" accept="image/*" onChange={handleFileChange} />
      {file && (
        <p>
          {file.name} — {(file.size / 1024).toFixed(1)} KB
        </p>
      )}
      {previewUrl && <img src={previewUrl} alt="preview" width={120} />}
    </div>
  );
}
```

To submit a file you typically build a `FormData` and `fetch` it. In real apps, remember to `URL.revokeObjectURL(previewUrl)` in a cleanup effect to avoid leaking object URLs — see `08-useeffect-side-effects.md` for the cleanup pattern.

### Example 4 — Uncontrolled form with refs

When you don't need the value on every keystroke (no live validation, no dependent UI), an uncontrolled form is less code and re-renders zero times while typing.

```jsx
import { useRef } from 'react';

export default function NewsletterSignup() {
  const emailRef = useRef(null);
  const agreeRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    const email = emailRef.current.value.trim();
    const agreed = agreeRef.current.checked;
    if (!agreed) return alert('Please accept the terms.');
    console.log('Signing up', { email, agreed });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={emailRef} type="email" defaultValue="" placeholder="you@example.com" />
      <label>
        <input ref={agreeRef} type="checkbox" defaultChecked={false} />
        I accept the terms
      </label>
      <button type="submit">Subscribe</button>
    </form>
  );
}
```

Note `defaultValue`/`defaultChecked`, not `value`/`checked`. You read the values only at submit time. No `useState`, no re-renders.

### Example 5 — Validation with error messages and a disabled submit

A controlled form that validates as the user types, shows per-field errors, and disables the submit button until the form is valid.

```jsx
import { useState } from 'react';

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

function validate({ email, password }) {
  const errors = {};
  if (!email) errors.email = 'Email is required.';
  else if (!EMAIL_RE.test(email)) errors.email = 'Enter a valid email.';
  if (!password) errors.password = 'Password is required.';
  else if (password.length < 8) errors.password = 'At least 8 characters.';
  return errors;
}

export default function LoginForm() {
  const [form, setForm] = useState({ email: '', password: '' });
  const [touched, setTouched] = useState({});
  const errors = validate(form);
  const isValid = Object.keys(errors).length === 0;

  function handleChange(e) {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  }

  function handleBlur(e) {
    setTouched((prev) => ({ ...prev, [e.target.name]: true }));
  }

  function handleSubmit(e) {
    e.preventDefault();
    setTouched({ email: true, password: true }); // reveal all errors on submit
    if (!isValid) return;
    console.log('Logging in', form);
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label>
        Email
        <input
          name="email"
          type="email"
          value={form.email}
          onChange={handleChange}
          onBlur={handleBlur}
          aria-invalid={touched.email && !!errors.email}
        />
      </label>
      {touched.email && errors.email && <span role="alert">{errors.email}</span>}

      <label>
        Password
        <input
          name="password"
          type="password"
          value={form.password}
          onChange={handleChange}
          onBlur={handleBlur}
          aria-invalid={touched.password && !!errors.password}
        />
      </label>
      {touched.password && errors.password && <span role="alert">{errors.password}</span>}

      <button type="submit" disabled={!isValid}>Log in</button>
    </form>
  );
}
```

We compute `errors` fresh on every render from the current `form` (derived state — no need to store it; see `07-state-usestate.md` on derived values). The `touched` map prevents yelling at the user before they've interacted with a field. `noValidate` turns off the browser's native bubbles so our custom messages are the only ones shown. `aria-invalid` and `role="alert"` keep it accessible.

### Example 6 — React 19 form Actions (preview)

React 19 lets a `<form>` take an `action` function that receives the `FormData` automatically. Combined with `useActionState` (form result + pending state) and `useFormStatus` (read pending status from a child button), you get progressive-enhancement-friendly forms with much less wiring. This is previewed here and covered fully in `22-modern-react-19.md`.

```jsx
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus(); // reads the enclosing <form>'s state
  return <button type="submit" disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}

async function saveName(prevState, formData) {
  const name = formData.get('name');
  if (!name) return { error: 'Name is required' };
  await new Promise((r) => setTimeout(r, 600)); // pretend network
  return { ok: true, name };
}

export default function ActionForm() {
  const [state, formAction] = useActionState(saveName, { });

  return (
    <form action={formAction}>
      <input name="name" defaultValue="" />
      <SubmitButton />
      {state?.error && <p role="alert">{state.error}</p>}
      {state?.ok && <p>Saved {state.name}!</p>}
    </form>
  );
}
```

No `onChange` per field, no manual `preventDefault`, no `isPending` state — the action receives `FormData` and React tracks pending state for you. Note inputs here use `name` + `defaultValue` (uncontrolled), which is exactly what `FormData` reads.

---

## Under the hood

**What "controlled" actually means to React.** When you pass `value` (or `checked`) to an input, React, after committing, *sets the DOM node's property to match*. If the DOM value diverges from the prop — say, a keystroke changed the DOM but you didn't update state in `onChange` — React forces the DOM back to the prop's value on the next render. That's why an input with `value={something}` and **no** `onChange` becomes read-only: React keeps resetting the DOM to the unchanging prop. (React warns about exactly this in dev.)

**The keystroke timeline.** You type "a":

1. The browser updates the DOM input's value to "a" and dispatches an input event.
2. React's synthetic `onChange` fires; your handler calls `setName('a')`.
3. React schedules a re-render. The component runs again with `name === 'a'`.
4. React reconciles and writes `value="a"` to the DOM — which already says "a", so usually no visible change.

Because steps 2–4 happen synchronously within the same event tick (React batches and flushes before the browser paints), you don't see flicker. But if your `onChange` is *slow* or you transform the value, the cursor can jump — see pitfalls.

**Cursor / caret position.** When you transform the value in `onChange` (e.g. force uppercase, insert formatting characters), React replaces the whole DOM value, and the browser can reset the caret to the end. For phone-number or currency masking you sometimes need to manage `selectionStart`/`selectionEnd` yourself, or use a masking library. This is a real source of "why does my cursor jump to the end" bugs.

**Why `value` is always a string.** The DOM `HTMLInputElement.value` property is a string by spec, even for `type="number"`. React mirrors the DOM, so `e.target.value` is a string. For `type="number"`, `e.target.valueAsNumber` exists (returns `NaN` for empty), but reading `e.target.value` and coercing yourself is the common, predictable path.

**Controlled vs uncontrolled is decided on first render and is sticky.** If an input's `value` is `undefined` on the first render and a string later, React treats it as switching from uncontrolled to controlled and warns. The classic cause is `value={form.email}` where `form.email` starts `undefined`. Always initialize controlled values to `''` (or the right type) — never `undefined`/`null`.

**`<select multiple>` and `FormData`.** With Actions/`FormData`, a multi-select contributes multiple entries under the same key; use `formData.getAll('languages')` to get them all. With controlled state you collect them via `selectedOptions`.

**Performance.** Each keystroke in a controlled form re-renders the component owning that state. For a handful of fields this is free. For very large forms (dozens of fields, expensive children) the re-render-per-keystroke cost is why **React Hook Form** keeps inputs *uncontrolled* by default and only subscribes the components that truly need to react — minimizing renders. Controlled is the right default; reach for RHF when measured re-renders hurt.

---

## Common mistakes & pitfalls

### 1. Controlled input with no `onChange` (read-only by accident)

```jsx
// WRONG — value is fixed by state but nothing updates it; field appears frozen
const [name, setName] = useState('');
<input value={name} />
```

```jsx
// RIGHT — close the loop with onChange
<input value={name} onChange={(e) => setName(e.target.value)} />
// (Or, if you truly want it read-only, say so explicitly:)
<input value={name} readOnly />
```

React keeps resetting the DOM to `name`, so typing does nothing. Either add `onChange` or mark it `readOnly`.

### 2. Initializing a controlled value to `undefined`/`null`

```jsx
// WRONG — form.email is undefined on first render → "uncontrolled to controlled" warning
const [form, setForm] = useState({});
<input value={form.email} onChange={handleChange} />
```

```jsx
// RIGHT — initialize every controlled field to a defined value
const [form, setForm] = useState({ email: '' });
<input value={form.email ?? ''} onChange={handleChange} />
```

### 3. Mutating the state object instead of replacing it

```jsx
// WRONG — mutates the existing object; React may not see a change / bugs compound
function handleChange(e) {
  form[e.target.name] = e.target.value;
  setForm(form); // same reference → no re-render guaranteed
}
```

```jsx
// RIGHT — build a new object (immutable update, see 07-state-usestate.md)
function handleChange(e) {
  const { name, value } = e.target;
  setForm((prev) => ({ ...prev, [name]: value }));
}
```

### 4. Forgetting `e.preventDefault()` on submit

```jsx
// WRONG — browser does a full-page reload, your handler's work is lost
function handleSubmit(e) {
  fetch('/api', { method: 'POST', body: JSON.stringify(form) });
}
```

```jsx
// RIGHT
function handleSubmit(e) {
  e.preventDefault();
  fetch('/api', { method: 'POST', body: JSON.stringify(form) });
}
```

(React 19 Actions via `action={fn}` handle this for you — no `preventDefault` needed.)

### 5. Reading `checked` from `e.target.value` for a checkbox

```jsx
// WRONG — value is the literal "on" (or your value attr), not the boolean state
setForm((p) => ({ ...p, newsletter: e.target.value }));
```

```jsx
// RIGHT — checkboxes report their boolean via .checked
setForm((p) => ({ ...p, newsletter: e.target.checked }));
```

### 6. Putting `value` on a file input

```jsx
// WRONG — file inputs cannot be controlled; React warns and it can't work
<input type="file" value={file} onChange={...} />
```

```jsx
// RIGHT — read from e.target.files (or a ref); no value prop
<input type="file" onChange={(e) => setFile(e.target.files?.[0] ?? null)} />
```

### 7. Stale state when reading sibling fields inside `onChange`

```jsx
// WRONG — closes over a possibly-stale `form` if updates are batched
function handleChange(e) {
  setForm({ ...form, [e.target.name]: e.target.value });
}
```

```jsx
// RIGHT — functional updater always sees the latest state
function handleChange(e) {
  const { name, value } = e.target;
  setForm((prev) => ({ ...prev, [name]: value }));
}
```

---

## Best practices

- **Default to controlled components.** They make validation, formatting, and dependent UI trivial and keep React as the single source of truth.
- **Initialize every controlled field to a concrete value** (`''`, `false`, `[]`) — never `undefined`/`null` — to avoid the controlled/uncontrolled switch warning.
- **Use one handler + `name` + computed keys** for multi-field forms; branch on `e.target.type` for checkboxes and multi-selects.
- **Always use the functional updater** (`setForm(prev => ...)`) inside change handlers so batched updates never read stale state.
- **`e.preventDefault()` in every `onSubmit`** (unless using React 19 `action`).
- **Treat `errors` as derived state** — compute from the current values during render rather than storing a duplicate that can desync.
- **Gate error display on a `touched`/`submitted` flag** so you don't scold users mid-typing; keep messages accessible with `role="alert"` and `aria-invalid`.
- **Reach for uncontrolled + refs** when you only need values at submit time and want zero typing re-renders (and for file inputs, which *must* be uncontrolled).
- **Pair native validation with custom** (`required`, `type="email"`, `pattern`, `min`/`max`) but add `noValidate` on the form if you render your own messages, so the two don't conflict.
- **Don't write your own giant form framework.** Past ~10 fields, complex validation, async checks, or field arrays, evaluate **React Hook Form** before hand-rolling more.
- **Prefer React 19 Actions** for new submit-and-await flows where available — less boilerplate, built-in pending state.

---

## Exercises

1. **Live character counter (controlled).** Build a `<textarea>` limited to 280 characters. Show a live "N/280" counter, turn it red when over a soft warning of 260, and disable a "Post" button when over 280.

2. **Multi-field profile form.** Create a form with `firstName`, `lastName`, a `role` single-`<select>`, and a `skills` multi-`<select>`. Use **one** `handleChange`. On submit, `console.log` the assembled object (skills as an array).

3. **Uncontrolled search box.** Make a search form using a `ref` and `defaultValue`. The value should only be read on submit. Confirm (via a render counter) that typing causes **zero** re-renders.

4. **Validated signup with dependent fields.** Build `email`, `password`, and `confirmPassword`. Validate: email format, password ≥ 8 chars, and confirm must match. Show per-field errors only after blur, disable submit until valid, and reveal all errors on submit attempt.

5. **React Hook Form rewrite.** Re-implement Exercise 4 with React Hook Form using `register`, `handleSubmit`, and `formState.errors`. Compare the line count and the number of re-renders to your controlled version.

### Solutions

**Exercise 1**

```jsx
import { useState } from 'react';

export default function TweetBox() {
  const [text, setText] = useState('');
  const LIMIT = 280;
  const WARN = 260;
  const over = text.length > LIMIT;

  return (
    <div>
      <textarea value={text} onChange={(e) => setText(e.target.value)} rows={4} />
      <p style={{ color: text.length > WARN ? 'crimson' : 'inherit' }}>
        {text.length}/{LIMIT}
      </p>
      <button type="button" disabled={over}>Post</button>
    </div>
  );
}
```

**Exercise 2**

```jsx
import { useState } from 'react';

const INITIAL = { firstName: '', lastName: '', role: 'dev', skills: [] };

export default function ProfileForm() {
  const [form, setForm] = useState(INITIAL);

  function handleChange(e) {
    const { name, value, type, selectedOptions } = e.target;
    setForm((prev) => ({
      ...prev,
      [name]:
        type === 'select-multiple'
          ? Array.from(selectedOptions, (o) => o.value)
          : value,
    }));
  }

  function handleSubmit(e) {
    e.preventDefault();
    console.log(form);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="firstName" value={form.firstName} onChange={handleChange} placeholder="First" />
      <input name="lastName" value={form.lastName} onChange={handleChange} placeholder="Last" />
      <select name="role" value={form.role} onChange={handleChange}>
        <option value="dev">Developer</option>
        <option value="design">Designer</option>
        <option value="pm">Product</option>
      </select>
      <select name="skills" multiple value={form.skills} onChange={handleChange}>
        <option value="react">React</option>
        <option value="node">Node</option>
        <option value="sql">SQL</option>
      </select>
      <button type="submit">Save</button>
    </form>
  );
}
```

**Exercise 3**

```jsx
import { useRef } from 'react';

let renders = 0;

export default function SearchBox() {
  renders += 1; // increments only on actual re-renders
  const queryRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    console.log('Searching for:', queryRef.current.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={queryRef} defaultValue="" placeholder="Search…" />
      <button type="submit">Go</button>
      <small>Component renders: {renders}</small>
    </form>
  );
}
```

Typing never calls `setState`, so the render count stays put — proof the uncontrolled input owns its value.

**Exercise 4**

```jsx
import { useState } from 'react';

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

function validate({ email, password, confirmPassword }) {
  const e = {};
  if (!EMAIL_RE.test(email)) e.email = 'Enter a valid email.';
  if (password.length < 8) e.password = 'At least 8 characters.';
  if (confirmPassword !== password) e.confirmPassword = 'Passwords must match.';
  return e;
}

export default function Signup() {
  const [form, setForm] = useState({ email: '', password: '', confirmPassword: '' });
  const [touched, setTouched] = useState({});
  const errors = validate(form);
  const isValid = Object.keys(errors).length === 0;

  const onChange = (e) =>
    setForm((p) => ({ ...p, [e.target.name]: e.target.value }));
  const onBlur = (e) =>
    setTouched((p) => ({ ...p, [e.target.name]: true }));

  function onSubmit(e) {
    e.preventDefault();
    setTouched({ email: true, password: true, confirmPassword: true });
    if (isValid) console.log('OK', form);
  }

  const field = (name, type = 'text') => (
    <>
      <input name={name} type={type} value={form[name]} onChange={onChange} onBlur={onBlur} />
      {touched[name] && errors[name] && <span role="alert">{errors[name]}</span>}
    </>
  );

  return (
    <form onSubmit={onSubmit} noValidate>
      {field('email', 'email')}
      {field('password', 'password')}
      {field('confirmPassword', 'password')}
      <button type="submit" disabled={!isValid}>Sign up</button>
    </form>
  );
}
```

**Exercise 5** (requires `npm install react-hook-form`)

```jsx
import { useForm } from 'react-hook-form';

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export default function SignupRHF() {
  const { register, handleSubmit, watch, formState: { errors, isValid } } = useForm({
    mode: 'onBlur',
    defaultValues: { email: '', password: '', confirmPassword: '' },
  });
  const password = watch('password');

  const onSubmit = (data) => console.log('OK', data);

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input
        {...register('email', {
          required: 'Email is required.',
          pattern: { value: EMAIL_RE, message: 'Invalid email.' },
        })}
      />
      {errors.email && <span role="alert">{errors.email.message}</span>}

      <input
        type="password"
        {...register('password', {
          required: 'Password is required.',
          minLength: { value: 8, message: 'Min 8 chars.' },
        })}
      />
      {errors.password && <span role="alert">{errors.password.message}</span>}

      <input
        type="password"
        {...register('confirmPassword', {
          validate: (v) => v === password || 'Passwords must match.',
        })}
      />
      {errors.confirmPassword && <span role="alert">{errors.confirmPassword.message}</span>}

      <button type="submit" disabled={!isValid}>Sign up</button>
    </form>
  );
}
```

Notice there's no `useState` and no per-field `onChange`: RHF registers each input as uncontrolled and only re-renders where needed. For most forms this is less code *and* faster.

### A note on libraries — when to reach for one

You can build any form with `useState` + the patterns above, and for small forms you should. Reach for a library when you hit:

- Many fields and you measure janky typing from re-renders.
- Complex/async validation (e.g. "is this username taken?"), schema validation (Zod/Yup), or field arrays (dynamic add/remove rows).
- Lots of repetitive wiring you'd rather not hand-roll.

**React Hook Form** is the modern default: uncontrolled-first (registers inputs via refs, minimal re-renders), tiny, great TypeScript support, integrates with Zod/Yup via resolvers. **Formik** is the older, controlled-style incumbent — still widely used in existing codebases, more re-renders, larger API surface; fine to maintain but RHF is generally preferred for new projects. Either way, the controlled/uncontrolled mental model from this lesson is exactly what those libraries are built on.

---

## Key takeaways

- A **controlled component** stores its value in React state and is driven by `value`/`checked` + `onChange` — React is the single source of truth.
- `value` is **always a string**; `type="number"` doesn't change that — coerce explicitly and handle the empty string.
- Each input type has a quirk: `textarea` uses `value` (not children), `select` uses `value` on the select, `select multiple` uses an array via `selectedOptions`, checkboxes use `checked`, radios share a `name`, and **file inputs are always uncontrolled** (read `e.target.files`).
- Scale to many fields with **one `onChange`** using the `name` attribute and **computed keys** (`[name]`), always with the functional updater.
- Submit with `onSubmit` + `e.preventDefault()`; reset by setting state back to the initial object.
- **Uncontrolled + `useRef` + `defaultValue`/`defaultChecked`** is the right call when you only need values at submit time (zero typing re-renders) and is mandatory for file inputs.
- Validation is best as **derived state** computed during render, revealed via a `touched`/submitted flag, with the submit button disabled until valid.
- **React 19 Actions** (`action={fn}`, `useActionState`, `useFormStatus`) cut boilerplate for submit-and-await flows (see `22-modern-react-19.md`); **React Hook Form** is the go-to library once forms get big or validation gets complex.

---

## Further reading

- React docs — [`<input>`](https://react.dev/reference/react-dom/components/input) (controlled vs uncontrolled, all props)
- React docs — [`<select>`](https://react.dev/reference/react-dom/components/select) and [`<textarea>`](https://react.dev/reference/react-dom/components/textarea)
- React docs — [Reacting to Input with State](https://react.dev/learn/reacting-to-input-with-state)
- React docs — [Sharing State Between Components](https://react.dev/learn/sharing-state-between-components) (lifting form state up)
- React docs — [`useActionState`](https://react.dev/reference/react/useActionState) and [`useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus)
- React docs — [`<form>` Actions](https://react.dev/reference/react-dom/components/form)
- React docs — [Manipulating the DOM with Refs](https://react.dev/learn/manipulating-the-dom-with-refs) (for uncontrolled inputs)
- [React Hook Form](https://react-hook-form.com/) — official docs and the [Get Started](https://react-hook-form.com/get-started) guide
- [Formik](https://formik.org/docs/overview) — official docs
- MDN — [Client-side form validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation) and [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData)
