# 18 — TypeScript with React

> **What you'll learn:** how to write React components, hooks, and data layers with TypeScript so the compiler catches bugs before the browser does — and how to read the errors when it complains.

By the end of this lesson you will be able to:

- Explain **why TypeScript pays off** in React projects (safety, autocomplete, fearless refactoring) and scaffold a typed project with the Vite `react-ts` template.
- Type **props** with `interface`/`type`, including optional props, defaults, and discriminated **union props**.
- Type **`children`** and the special React types (`React.ReactNode`, `React.ReactElement`, `JSX.Element`) and know which to use when.
- Type **DOM events** precisely (`React.ChangeEvent`, `React.FormEvent`, `React.MouseEvent`, `React.KeyboardEvent`).
- Type the core hooks: **`useState<T>`**, **`useRef`** (DOM vs mutable), **`useReducer`** with discriminated-union actions, and **`useContext`** with a guarded custom-hook pattern.
- Write **generic components** and **fully-typed custom hooks**.
- Type **API responses** and async data, and wield the everyday **utility types** (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`).
- Decide for yourself in the **`React.FC` debate** and decode the compiler's most common error messages.

---

## Prerequisites

You should be comfortable with the material from these earlier lessons before starting:

- **`01-js-foundations.md`** — modern JS, destructuring, modules. TypeScript is a superset of this.
- **`04-components-and-props.md`** — function components and props, which we are now going to type.
- **`06-handling-events.md`** — event handlers; here we give their arguments static types.
- **`07-state-usestate.md`** and **`08-useeffect-side-effects.md`** — `useState`/`useEffect`, since we type them.
- **`09-hooks-deep-dive.md`** and **`10-custom-hooks.md`** — `useRef`, `useReducer`, `useContext`, and authoring custom hooks. This lesson types each of them.

You do **not** need to be a TypeScript expert. We introduce each TS feature in context. If you have never seen TypeScript at all, just know it is JavaScript plus a static type layer that is erased at build time — your browser never runs a single byte of it.

---

## Concept

### What TypeScript actually is

TypeScript is JavaScript with **type annotations** that a compiler (`tsc`, or in Vite's case the bundler's transpiler) checks and then **erases**. The output is plain JavaScript. Types exist only at *author time* and *build time*; there is **no runtime type checking** — `number` is not validated when a real value arrives from the network. This single fact explains most TS-in-React surprises: the compiler trusts what you tell it, so when you `fetch` JSON and call it a `User`, TypeScript believes you even if the server lied.

Think of types as a **contract you write down** and the compiler as a tireless reviewer who reads every call site and flags any place the contract is violated. In a UI codebase, props are contracts between a parent and a child, state shape is a contract between renders, and an API response is a contract between client and server. React is, structurally, a giant tree of contracts — which is exactly why it benefits so much from static typing.

### Why bother — three concrete wins

1. **Safety.** "Cannot read property `name` of undefined" is the single most common React crash. With TypeScript, a value that *might* be `undefined` has the type `User | undefined`, and the compiler refuses to let you read `.name` until you've narrowed it. Whole classes of `undefined`/`null` bugs vanish at author time.

2. **Developer experience (DX).** Because the editor knows the exact shape of every prop, state value, and hook return, you get accurate autocomplete, inline docs, and go-to-definition. You stop guessing whether the prop is `onChange` or `onChanged`.

3. **Fearless refactoring.** Rename a prop, change a field from `string` to `number`, split a component — and every stale call site lights up red instantly. In a plain-JS React app the same change is a grep-and-pray exercise; in TypeScript it is a checklist the compiler hands you.

The cost is real but small: a little ceremony up front, and occasional fights with the type checker. The trade is overwhelmingly worth it for any app you intend to keep.

---

## Syntax & API

### Project setup

Scaffold a typed React project with Vite:

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

The `react-ts` template gives you `.tsx` files (`.tsx` = TypeScript + JSX; plain logic files are `.ts`), a `tsconfig.json`, and the type packages already wired up. The key dev dependencies are:

```json
{
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "typescript": "^5.6.0"
  }
}
```

`@types/react` is the package that supplies all the React type definitions (`React.ReactNode`, event types, hook signatures). React ships *no* types of its own; they live in this community-maintained `@types/react` package. **Keep `@types/react` major-version-aligned with `react`** — React 19 needs `@types/react@^19`.

### `tsconfig.json` basics

The template's config is sensible. The settings you should understand:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true
  }
}
```

- **`"jsx": "react-jsx"`** — the modern automatic runtime. You do **not** need `import React from 'react'` at the top of every file anymore (React 17+). You still import the hooks you use.
- **`"strict": true`** — turns on the full strict family (`strictNullChecks`, `noImplicitAny`, etc.). **Always keep this on.** Most of TypeScript's value comes from strict null checking. Turning it off is the single most common way teams sabotage themselves.
- **`"lib": ["DOM", ...]`** — makes `window`, `document`, and the `HTMLElement` types available.

### Typing props

```tsx
// As an interface (open to declaration merging, conventional for object shapes)
interface ButtonProps {
  label: string;
  disabled?: boolean;            // optional
  onClick: () => void;
}

// As a type alias (can express unions, intersections, mapped types)
type ButtonProps2 = {
  label: string;
  variant: 'primary' | 'secondary';   // union of string literals
};
```

`interface` and `type` are nearly interchangeable for component props. Use whichever your team standardizes on; reach for `type` when you need a union or a computed type, and `interface` when you might want consumers to extend it.

### The special React types

| Type | Meaning | Use for |
|------|---------|---------|
| `React.ReactNode` | Anything renderable: elements, strings, numbers, arrays, `null`, `undefined`, booleans | Typing `children` — the **default choice** |
| `React.ReactElement` | A single JSX element object (`{ type, props, key }`) | When you require exactly one element, e.g. a render prop result |
| `React.JSX.Element` | The return type of a JSX expression; effectively `ReactElement` | Rarely needed explicitly; let it be inferred. In `@types/react@19` the global `JSX` namespace was removed, so reference it as `React.JSX.Element` (the bare `JSX.Element` still resolves inside `.tsx` files under `"jsx": "react-jsx"`, but `React.JSX` is the future-proof form) |
| `React.ComponentType<P>` | A component (function or class) taking props `P` | Higher-order components, `as` props |
| `React.CSSProperties` | A typed style object | `style={{ color: 'red' }}` props |

### Event types

`@types/react` provides synthetic-event types parameterized by the element:

```tsx
React.ChangeEvent<HTMLInputElement>     // input/textarea/select change
React.FormEvent<HTMLFormElement>        // form submit
React.MouseEvent<HTMLButtonElement>     // click
React.KeyboardEvent<HTMLInputElement>   // keydown/keyup
React.FocusEvent<HTMLInputElement>      // focus/blur
```

These are **React synthetic events**, not raw DOM events — see `06-handling-events.md` for the runtime difference.

---

## Examples

### Example 1 — A typed presentational component

```tsx
// Greeting.tsx
interface GreetingProps {
  name: string;
  enthusiasm?: number; // optional
}

function Greeting({ name, enthusiasm = 1 }: GreetingProps) {
  return <h1>Hello, {name}{'!'.repeat(enthusiasm)}</h1>;
}

export default Greeting;
```

Default values are expressed with **ordinary JS default destructuring** (`enthusiasm = 1`). You do *not* need React's legacy `defaultProps` for function components — it is deprecated for them in React 19. Because `enthusiasm` is `enthusiasm?: number`, its type inside the body is `number | undefined`, but the `= 1` default narrows it to `number`. Calling `<Greeting />` without a `name` is now a **compile error**, which is exactly what we want.

### Example 2 — A typed controlled input with events

```tsx
// SearchBox.tsx
import { useState } from 'react';

interface SearchBoxProps {
  onSearch: (query: string) => void;
}

function SearchBox({ onSearch }: SearchBoxProps) {
  const [query, setQuery] = useState(''); // inferred as string

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setQuery(e.target.value); // e.target is typed HTMLInputElement → .value is string
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    onSearch(query);
  }

  function handleKeyDown(e: React.KeyboardEvent<HTMLInputElement>) {
    if (e.key === 'Escape') setQuery('');
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={query} onChange={handleChange} onKeyDown={handleKeyDown} />
      <button type="submit">Search</button>
    </form>
  );
}

export default SearchBox;
```

Because we annotated `e` as `React.ChangeEvent<HTMLInputElement>`, `e.target.value` is known to be a `string` and `e.key` is autocompleted. Note an alternative: if you write the handler **inline** in the JSX (`onChange={e => ...}`), TypeScript *infers* the event type for you from the `onChange` prop — no annotation needed. You only annotate when the handler is a standalone named function.

### Example 3 — Discriminated union props (variants that change required fields)

A button is either a real `<button>` *or* a link `<a>`. Each needs different props. A discriminated union models this precisely:

```tsx
// SmartButton.tsx
type SmartButtonProps =
  | { as: 'button'; onClick: () => void; href?: never }
  | { as: 'link'; href: string; onClick?: never };

function SmartButton(props: SmartButtonProps) {
  if (props.as === 'link') {
    // TS narrows: props.href is string here
    return <a href={props.href}>Go</a>;
  }
  // here props.onClick is () => void
  return <button onClick={props.onClick}>Click</button>;
}

// <SmartButton as="link" href="/x" />           ✅
// <SmartButton as="button" onClick={() => {}} /> ✅
// <SmartButton as="link" onClick={() => {}} />   ❌ compile error
```

The `as` field is the **discriminant**. The `href?: never` / `onClick?: never` markers stop callers from mixing the two shapes. This is one of TypeScript's most powerful patterns for component APIs.

### Example 4 — Typed `useReducer` with a discriminated-union action

```tsx
// counterReducer.ts
interface State {
  count: number;
  step: number;
}

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setStep'; step: number }   // carries a payload
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.step }; // action.step is known to exist here
    case 'reset':
      return { count: 0, step: 1 };
    default: {
      // exhaustiveness check: if we add a new Action and forget a case,
      // this line becomes a compile error.
      const _never: never = action;
      return state;
    }
  }
}
```

```tsx
// Counter.tsx
import { useReducer } from 'react';
import { reducer } from './counterReducer';

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'setStep', step: 5 })}>step 5</button>
      {/* dispatch({ type: 'setStep' }) ❌ — TS requires the `step` field */}
    </>
  );
}
```

The discriminated union does triple duty: `dispatch` only accepts valid actions, the reducer's `switch` **narrows** `action` so payload fields are available without casts, and the `never` default gives **exhaustiveness checking** — add a new action type and forget to handle it, and the compiler points at the exact spot.

### Example 5 — Typed context with the custom-hook guard pattern

```tsx
// auth-context.tsx
import { createContext, useContext, useState, type ReactNode } from 'react';

interface User {
  id: string;
  name: string;
}

interface AuthContextValue {
  user: User | null;
  login: (name: string) => void;
  logout: () => void;
}

// Default is undefined so we can DETECT "used outside provider".
const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const login = (name: string) => setUser({ id: crypto.randomUUID(), name });
  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// The guard: throws if there is no provider, and NARROWS away `undefined`.
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (ctx === undefined) {
    throw new Error('useAuth must be used within an <AuthProvider>');
  }
  return ctx; // type is AuthContextValue, never undefined
}
```

Consumers call `const { user, login } = useAuth();` and get a fully typed value with **no `undefined` to guard against** — the custom hook already narrowed it. The alternative is the **non-null assertion** `useContext(AuthContext)!`, but that silences the check without the helpful runtime error; the guard pattern is strictly better. (Provider mechanics are covered in the context lesson, e.g. `12-context-api.md`.)

### Example 6 — A generic component

A `List` that works for any item type, inferring the type from its `items` prop:

```tsx
// List.tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keySelector: (item: T) => string | number;
}

function List<T>({ items, renderItem, keySelector }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={keySelector(item)}>{renderItem(item, i)}</li>
      ))}
    </ul>
  );
}

// Usage — T is inferred as { id: number; title: string }:
// <List
//   items={[{ id: 1, title: 'a' }]}
//   keySelector={(b) => b.id}
//   renderItem={(b) => b.title}  // b is fully typed
// />
```

The `<T>` after the function name declares a type parameter. At each call site TypeScript **infers** `T` from `items`, then enforces that `renderItem` and `keySelector` receive that same `T`. (In `.tsx` files an arrow-function generic can be ambiguous with JSX; write `<T,>` with a trailing comma, or just use a `function` declaration as above.)

### Example 7 — A typed custom hook for async API data

```tsx
// types.ts
export interface User {
  id: number;
  name: string;
  email: string;
}
```

```tsx
// useFetch.ts
import { useEffect, useState } from 'react';

interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

export function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });

  useEffect(() => {
    let cancelled = false;
    setState({ data: null, loading: true, error: null });

    fetch(url)
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<T>; // we ASSERT the JSON matches T
      })
      .then((data) => {
        if (!cancelled) setState({ data, loading: false, error: null });
      })
      .catch((error: unknown) => {
        if (!cancelled) {
          setState({
            data: null,
            loading: false,
            error: error instanceof Error ? error : new Error(String(error)),
          });
        }
      });

    return () => { cancelled = true; };
  }, [url]);

  return state;
}
```

```tsx
// UserCard.tsx
import { useFetch } from './useFetch';
import type { User } from './types';

function UserCard({ id }: { id: number }) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${id}`);

  if (loading) return <p>Loading…</p>;
  if (error) return <p>Error: {error.message}</p>;
  if (!user) return null; // narrows `user` to User below
  return <h2>{user.name} — {user.email}</h2>;
}
```

The hook is generic over the response type `T`. Calling `useFetch<User>(...)` types `data` as `User | null`, which forces the component to handle the loading/null cases before reading `.name`. Note the honest comment: `res.json()` returns `any`, so the `as Promise<T>` is an **unchecked assertion**. For real safety against a lying server, validate with a runtime schema library (e.g. **Zod**) and infer the type from the schema — that closes the gap between compile-time types and runtime data.

### Example 8 — Utility types in practice

```tsx
import type { User } from './types';

// Partial<T> — all fields optional. Great for "update" patches.
type UserPatch = Partial<User>;             // { id?, name?, email? }

// Pick<T, K> — keep only some fields.
type UserPreview = Pick<User, 'id' | 'name'>; // { id, name }

// Omit<T, K> — drop some fields (e.g. server-assigned id on create).
type NewUser = Omit<User, 'id'>;            // { name, email }

// Record<K, V> — a map/dictionary type.
type UsersById = Record<number, User>;      // { [id: number]: User }

// ReturnType<typeof fn> — derive a type from a function's return.
function makeUser() { return { id: 1, name: 'Ana', email: 'a@x.io' }; }
type MadeUser = ReturnType<typeof makeUser>; // { id, name, email }

function updateUser(id: number, patch: UserPatch) { /* ... */ }
```

`ReturnType<typeof useFetch>` is especially handy for typing a hook's return without restating it. These utilities let you derive related types from one source of truth, so a change to `User` propagates everywhere automatically.

---

## Under the hood

### Types are erased — there is no runtime safety net

The compiler checks your code and then **strips every annotation**. The shipped bundle is plain JS. Consequences:

- A value typed `User` that actually arrives as `{ name: 123 }` from the API will **not** throw at the boundary. TypeScript's `User` is a *promise you made*, not a runtime guard. This is why API boundaries need runtime validation (Zod, Valibot) if you don't control the server.
- `instanceof`, `typeof`, and explicit checks still work at runtime and are how you should narrow — and TypeScript *understands* those checks (control-flow narrowing).

### How `@types/react` makes JSX type-safe

When you write `<input value={query} onChange={handleChange} />`, the compiler looks up `React.JSX.IntrinsicElements['input']` from `@types/react` (in `@types/react@19` this lives under the `React.JSX` namespace rather than a global `JSX` one). That declaration lists every valid HTML attribute and event handler for `<input>` with their exact types. That's why `value={42}` on a text input errors and `onChange` expects `(e: ChangeEvent<HTMLInputElement>) => void`. For your own components, `<Greeting name="x" />` is type-checked against the props type you wrote. JSX is, to the type system, just a function-call-shaped expression with a known parameter type.

### How hook inference works

`useState`'s signature is roughly:

```tsx
function useState<S>(initial: S | (() => S)): [S, Dispatch<SetStateAction<S>>];
```

When you call `useState(0)`, TypeScript infers `S = number` from the argument. When the initial value can't pin down the full type — e.g. `useState(null)` infers `S = null`, or `useState([])` infers `S = never[]` — you **must** supply the type argument: `useState<User | null>(null)`. The setter type `Dispatch<SetStateAction<S>>` is why `setCount(c => c + 1)` (the updater form) is accepted: `SetStateAction<S>` is `S | ((prev: S) => S)`.

### `useRef`: two uses, one mutable type (React 19)

`useRef` covers two everyday jobs — a handle to a DOM node, and a plain mutable "instance variable" that survives renders without triggering one:

```tsx
// DOM ref — pass it to a JSX `ref` prop; `.current` is HTMLDivElement | null.
const divRef = useRef<HTMLDivElement>(null);   // RefObject<HTMLDivElement | null>

// Mutable "instance variable" — you own and write `.current`.
const timer = useRef<number>(0);               // RefObject<number>
```

In `@types/react@19` the old split between a *read-only* `RefObject` and a *mutable* `MutableRefObject` is **gone** — every ref `useRef` returns now has a writable `.current`, so both lines above let you assign `.current`. (Before React 19, passing `null` selected a read-only overload and you needed `useRef<HTMLDivElement | null>(null)` to get a writable one; that workaround is no longer necessary.) Two other React 19 type changes worth knowing: `useRef` now **requires an argument** (`useRef()` with no argument is a type error — pass `useRef(undefined)` if you have no initial value), and the convenience overload makes `useRef<HTMLDivElement>(null)` resolve to `RefObject<HTMLDivElement | null>`, so the `| null` is already baked in.

Reading `divRef.current` after mount still requires a null check, because React only fills a DOM ref in after the render commits — see `09-hooks-deep-dive.md` for ref timing.

### `strictNullChecks` and narrowing

Under `strict`, `User | null` and `User | undefined` are *different* from `User`, and you cannot access members until you eliminate the empty cases. The compiler tracks this through `if`, early `return`, `&&`, optional chaining (`user?.name`), and `instanceof`. This control-flow analysis is the engine behind most of TS's bug-catching power in React — it forces you to handle the "still loading / not found" branches that cause real crashes.

---

## Common mistakes & pitfalls

### Pitfall 1 — `useState` that needs a type argument

```tsx
// ❌ WRONG: inferred as `never[]`, so you can never push a real user.
const [users, setUsers] = useState([]);
setUsers([{ id: 1, name: 'Ana' }]); // Error: not assignable to never[]
```

```tsx
// ✅ RIGHT: tell useState the element type.
const [users, setUsers] = useState<User[]>([]);
```

When the initial value doesn't reveal the full intended type (empty array, `null`), supply the generic argument explicitly.

### Pitfall 2 — Annotating event handlers as the wrong type

```tsx
// ❌ WRONG: DOM `Event` ≠ React synthetic event; `e.target.value` is `any`/error.
function handleChange(e: Event) {
  setQuery(e.target.value);
}
```

```tsx
// ✅ RIGHT: use the React synthetic event parameterized by the element.
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  setQuery(e.target.value); // string
}
```

### Pitfall 3 — Using `any` to silence the compiler

```tsx
// ❌ WRONG: `any` turns off type checking for this value entirely.
const data: any = await res.json();
return <h2>{data.naem}</h2>; // typo sails through — runtime undefined
```

```tsx
// ✅ RIGHT: type the shape (and ideally validate at the boundary).
const data = (await res.json()) as User;
return <h2>{data.name}</h2>; // `data.naem` is now a compile error
```

`any` is contagious and disables the very safety you adopted TS for. When you truly don't know a type, use `unknown` and narrow it — `unknown` forces a check before use.

### Pitfall 4 — `JSX.Element` vs `ReactNode` for children

```tsx
// ❌ WRONG: rejects strings, numbers, arrays, null — only one element allowed.
function Card({ children }: { children: React.JSX.Element }) {
  return <div className="card">{children}</div>;
}
// <Card>hello</Card>  ❌  <Card>{items.map(...)}</Card> ❌
```

```tsx
// ✅ RIGHT: ReactNode accepts everything renderable.
function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}
```

Use `ReactNode` for `children` unless you specifically require exactly one element.

### Pitfall 5 — Mismatched `@types/react` version

If `react` is `19.x` but `@types/react` is `18.x`, you get baffling errors about `ReactNode`, `JSX`, or `ref` not matching. Keep the major versions aligned. After upgrading React, upgrade the types in the same step.

### Pitfall 6 — Reading a DOM ref without a null check

```tsx
// ❌ WRONG: ref.current is HTMLInputElement | null; .focus() may be on null.
const ref = useRef<HTMLInputElement>(null);
useEffect(() => { ref.current.focus(); }, []); // Object is possibly 'null'
```

```tsx
// ✅ RIGHT: narrow first.
useEffect(() => { ref.current?.focus(); }, []);
```

---

## Best practices

- **Keep `strict: true`.** Disabling it throws away most of TypeScript's value. Treat strict-null errors as bugs you're being warned about, not noise.
- **Prefer inference; annotate boundaries.** Let `useState(0)` infer; annotate props, function parameters, custom-hook returns, and API shapes — the places types act as contracts.
- **Use `interface` or `type` for props consistently;** reach for `type` when you need unions/intersections, `interface` when extension matters. Don't agonize over it.
- **Use `React.ReactNode` for `children`** by default; only use `ReactElement`/`JSX.Element` when you genuinely require a single element.
- **Annotate standalone event handlers; let inline handlers infer.** Use the `React.*Event<HTMLxxxElement>` family, never the raw DOM `Event`.
- **Model variants with discriminated unions** (a literal `type`/`as`/`kind` field) and add a `never` exhaustiveness check in reducers and switches.
- **Wrap context in a guard custom hook** that throws when used outside its provider, instead of scattering non-null assertions (`!`).
- **Avoid `any`. Prefer `unknown` + narrowing.** Validate untrusted data (API responses) at runtime with a schema library and infer the type from it.
- **Use `import type { … }` for type-only imports** — it documents intent and is reliably erased from the bundle.
- **Keep `@types/react` aligned with `react`'s major version.**

### The `React.FC` debate

`React.FC<Props>` (and `React.FunctionComponent`) is a built-in type for "a function component taking `Props`":

```tsx
const Greeting: React.FC<GreetingProps> = ({ name }) => <h1>Hi {name}</h1>;
```

Many teams **avoid `React.FC`** today. Why:

- Historically it forced an implicit `children` prop onto *every* component, even ones that take no children. (This was removed in `@types/react@18`, but the habit of avoiding it stuck.)
- It doesn't play as cleanly with **generic components** — you can't easily write `React.FC` with a type parameter.
- The plain-function form is simpler and infers just as well.

The recommended modern style is the plain annotation on the parameter:

```tsx
// ✅ Preferred: annotate props directly, return type inferred.
function Greeting({ name }: GreetingProps) {
  return <h1>Hi {name}</h1>;
}
```

`React.FC` is not *wrong* — it's a defensible choice and some codebases standardize on it. But the plain function is the lower-friction default, and it's what most React 19 + TS guides now recommend.

---

## Exercises

Work each exercise before reading its solution.

### Exercise 1 — Type a `Badge` component
Create a `Badge` component with a required `label: string` and an optional `tone` that is one of `'info' | 'success' | 'error'`, defaulting to `'info'`. Render `<span className={`badge badge-${tone}`}>{label}</span>`.

### Exercise 2 — Typed toggle hook
Write a custom hook `useToggle(initial = false)` that returns a tuple `[value: boolean, toggle: () => void]`. Type it so the consumer destructures with correct types and `toggle` flips the value using the functional updater.

### Exercise 3 — Discriminated-union reducer
Build a `useReducer`-based shopping cart. State is `{ items: { id: number; qty: number }[] }`. Support actions `add` (payload `id`), `remove` (payload `id`), and `clear`. Include a `never` exhaustiveness check.

### Exercise 4 — Generic select
Write a generic `Select<T>` component with props `options: T[]`, `getLabel: (o: T) => string`, `getValue: (o: T) => string`, and `onChange: (o: T) => void`. Render a `<select>` and fire `onChange` with the chosen option object.

### Exercise 5 — Typed async data with utility types
Given `interface Post { id: number; title: string; body: string; authorId: number }`, write a `useFetch<T>`-based component that lists post **previews** showing only `id` and `title` (use `Pick`), and type a `createPost` function argument as a `Post` without its server-assigned `id` (use `Omit`).

---

## Solutions

### Solution 1

```tsx
// Badge.tsx
interface BadgeProps {
  label: string;
  tone?: 'info' | 'success' | 'error';
}

function Badge({ label, tone = 'info' }: BadgeProps) {
  return <span className={`badge badge-${tone}`}>{label}</span>;
}

export default Badge;
```

### Solution 2

```tsx
// useToggle.ts
import { useCallback, useState } from 'react';

export function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue((v) => !v), []);
  return [value, toggle];
}

// Usage:
// const [open, toggleOpen] = useToggle();  // open: boolean, toggleOpen: () => void
```

The explicit `[boolean, () => void]` return annotation makes the tuple positions stable; without it TS would widen to `(boolean | (() => void))[]`.

### Solution 3

```tsx
// cart.ts
interface CartItem { id: number; qty: number; }
interface CartState { items: CartItem[]; }

type CartAction =
  | { type: 'add'; id: number }
  | { type: 'remove'; id: number }
  | { type: 'clear' };

export function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add': {
      const existing = state.items.find((i) => i.id === action.id);
      if (existing) {
        return {
          items: state.items.map((i) =>
            i.id === action.id ? { ...i, qty: i.qty + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { id: action.id, qty: 1 }] };
    }
    case 'remove':
      return { items: state.items.filter((i) => i.id !== action.id) };
    case 'clear':
      return { items: [] };
    default: {
      const _never: never = action; // exhaustiveness check
      return state;
    }
  }
}
```

### Solution 4

```tsx
// Select.tsx
interface SelectProps<T> {
  options: T[];
  getLabel: (o: T) => string;
  getValue: (o: T) => string;
  onChange: (o: T) => void;
}

function Select<T>({ options, getLabel, getValue, onChange }: SelectProps<T>) {
  function handleChange(e: React.ChangeEvent<HTMLSelectElement>) {
    const chosen = options.find((o) => getValue(o) === e.target.value);
    if (chosen) onChange(chosen);
  }

  return (
    <select onChange={handleChange}>
      {options.map((o) => (
        <option key={getValue(o)} value={getValue(o)}>
          {getLabel(o)}
        </option>
      ))}
    </select>
  );
}

export default Select;

// Usage — T inferred as { code: string; name: string }:
// <Select
//   options={[{ code: 'us', name: 'USA' }]}
//   getValue={(c) => c.code}
//   getLabel={(c) => c.name}
//   onChange={(c) => console.log(c.name)}
// />
```

### Solution 5

```tsx
// posts.ts
export interface Post {
  id: number;
  title: string;
  body: string;
  authorId: number;
}

export type PostPreview = Pick<Post, 'id' | 'title'>;
export type NewPost = Omit<Post, 'id'>;

export async function createPost(input: NewPost): Promise<Post> {
  const res = await fetch('/api/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(input),
  });
  return (await res.json()) as Post;
}
```

```tsx
// PostList.tsx
import { useFetch } from './useFetch'; // from Example 7
import type { Post, PostPreview } from './posts';

function PostList() {
  const { data, loading, error } = useFetch<Post[]>('/api/posts');

  if (loading) return <p>Loading…</p>;
  if (error) return <p>Error: {error.message}</p>;
  if (!data) return null;

  const previews: PostPreview[] = data.map(({ id, title }) => ({ id, title }));

  return (
    <ul>
      {previews.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}

export default PostList;
```

---

## Reading TypeScript errors

A quick decoder for the messages you'll hit most:

- **`Type 'X' is not assignable to type 'Y'.`** — You gave a value of shape `X` where `Y` was expected. Read the **last** line of the error; it usually pinpoints the offending field.
- **`Object is possibly 'null' / 'undefined'.`** — You read a member of something that might be empty. Narrow with `?.`, an `if`, or an early return.
- **`Property 'foo' does not exist on type '…'.`** — A typo, or you're in a union branch where `foo` isn't present (narrow on the discriminant first).
- **`Argument of type '…' is not assignable to parameter of type 'never'.`** — Classic `useState([])`-inferred-as-`never[]` symptom; add the generic argument.
- **`No overload matches this call.`** — Often a `useRef`/event-handler shape mismatch. Check the element type parameter.

Read errors **bottom-up**: TypeScript prints the high-level mismatch first and the precise reason last.

---

## Key takeaways

- TypeScript is **erased at build time** — it catches bugs at author time but gives **no runtime guarantees**; validate untrusted data (API responses) at the boundary.
- Scaffold with `npm create vite@latest app -- --template react-ts`, keep **`strict: true`**, and keep **`@types/react` aligned with `react`'s major version**.
- Type **props** with `interface`/`type`, defaults via destructuring, and **variants via discriminated unions**; use **`React.ReactNode`** for `children`.
- Annotate standalone event handlers with the **`React.*Event<HTMLxxxElement>`** family; inline handlers infer automatically.
- Let `useState` **infer**, but supply `useState<T>()` when the initial value can't reveal the type; understand `useRef`'s **DOM (read-only) vs mutable** overloads.
- Model reducer actions as **discriminated unions** with a `never` exhaustiveness check; wrap context in a **guard custom hook** instead of using `!`.
- Write **generic components and hooks** to stay DRY, and derive related types with **`Partial`/`Pick`/`Omit`/`Record`/`ReturnType`**.
- Prefer the plain typed-function component over **`React.FC`**, and never reach for **`any`** — use `unknown` and narrow.

---

## Further reading

- React docs — [TypeScript](https://react.dev/learn/typescript)
- React docs — [Type-checking with hooks reference](https://react.dev/reference/react) (each hook page notes its TS usage)
- TypeScript Handbook — [Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) and [Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- TypeScript Handbook — [Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)
- [React TypeScript Cheatsheets](https://react-typescript-cheatsheet.netlify.app/) — community reference for component, hook, and HOC patterns
- Vite docs — [Getting Started / templates](https://vite.dev/guide/) (the `react-ts` template)
- [Zod](https://zod.dev/) — runtime schema validation to close the compile-time/runtime gap at API boundaries
