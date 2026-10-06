# 19 — Testing React Applications

> **What you'll learn:** how to write fast, confident, maintainable tests for React components and hooks using Vitest and React Testing Library — testing what the user experiences, not how the code is wired.

By the end of this lesson you will be able to:

- Explain the testing pyramid and the **React Testing Library (RTL)** philosophy of testing observable behavior.
- Set up **Vitest + @testing-library/react + jsdom + @testing-library/jest-dom** in a Vite project.
- Use `render`, the `screen` object, and the full query family (`getBy` / `queryBy` / `findBy` plus `*AllBy`) with the correct **query priority order**.
- Simulate realistic user interaction with **`@testing-library/user-event`**.
- Assert on the DOM with jest-dom matchers and write robust **async** tests with `findBy` / `waitFor`.
- Mock functions, modules, `fetch`, and whole APIs with **`vi.fn()`** and **MSW (Mock Service Worker)**.
- Test **custom hooks** with `renderHook` + `act`, understand **snapshot** tests and their caveats, measure **coverage**, and know when to reach for **E2E** tools (Playwright/Cypress).

---

## Prerequisites

Before this lesson you should be comfortable with:

- Function components, props, and JSX (lessons 02–05).
- `useState` and event handlers (see `07-state-usestate.md`).
- `useEffect` and data fetching side effects (see `08-useeffect-side-effects.md`).
- Writing custom hooks (see `10-custom-hooks.md`).
- Basic JavaScript promises / `async`–`await` and ES modules.

You do **not** need any prior testing experience. We build from zero.

---

## Concept

### Why test at all?

Tests are an executable specification. They let you change code fearlessly: if a refactor breaks user-visible behavior, a red test tells you *before* your users do. The goal is **confidence per unit of effort** — not 100% coverage for its own sake.

### The testing pyramid

A classic mental model for *where* to spend your testing budget:

```
        /\
       /  \      E2E (few)        — real browser, real backend-ish, slow, brittle
      /----\
     /      \    Integration (some) — several components together, real DOM, no network
    /--------\
   /          \  Unit (many)       — one function/hook/component in isolation, fast
  /____________\
```

- **Unit tests** are cheap and fast; run thousands per second.
- **Integration tests** render a few components together (a form, a list with its row) and assert behavior. In React, RTL nudges you toward this layer — most "component tests" are really small integration tests.
- **E2E tests** drive a real browser against a running app. They catch wiring bugs nothing else can, but they're slow and flaky, so keep them few and focused on critical user journeys.

Kent C. Dodds (RTL's author) popularized the **"testing trophy"** variation, which fattens the integration layer because that's where React apps get the best confidence-to-cost ratio. Either way: **lots of cheap tests, few expensive ones.**

### The RTL philosophy: test behavior, not implementation

The single most important idea in this lesson:

> **"The more your tests resemble the way your software is used, the more confidence they can give you."** — Kent C. Dodds

That means: **don't** test that a component "has state `count === 1`" or that a particular child component received a particular prop. **Do** test what a real user would observe: *"after I click the +1 button, the screen shows 'Count: 1'."*

Implementation-detail tests break when you refactor (rename state, swap a `useState` for a `useReducer`) even though nothing the user sees changed. Behavior tests survive refactors and fail only on real regressions. This is why RTL deliberately gives you **no** API to read component state, instances, or props — it only lets you query the rendered DOM, the same surface a user touches.

---

## Syntax & API

### `render(ui, options?)`

Mounts a React element into a container (a `div` appended to `document.body` in jsdom) and returns a result object.

```jsx
import { render, screen } from '@testing-library/react';

const { rerender, unmount, container, asFragment } = render(<App />);
```

- `rerender(ui)` — re-render with new props (good for prop-change tests).
- `unmount()` — unmount (good for cleanup-effect tests).
- `container` — the DOM node; an escape hatch, prefer `screen`.
- RTL auto-cleans up after each test when the test runner globals are present (Vitest/Jest), so you rarely call `cleanup()` yourself.

### `screen`

A singleton bound to `document.body`. **Prefer `screen.getByRole(...)` over destructuring queries from `render`** — it's less to maintain and always queries the whole document.

### Queries — the three axes

| | Throws if **0** found | Returns/awaits | Use for |
|---|---|---|---|
| `getBy*` | yes (0 or >1) | element, sync | element that **should be there now** |
| `queryBy*` | no → returns `null` | element-or-null, sync | asserting **absence** |
| `findBy*` | yes (0 or >1), retries | `Promise<element>` | element that **appears later** (async) |

Each has an **`*AllBy`** plural variant returning an array (and never throwing on >1):

- `getAllBy*` → array, throws if none.
- `queryAllBy*` → array, `[]` if none.
- `findAllBy*` → `Promise<array>`.

### Query *types* and their priority order

RTL ranks queries by how close they are to the user's experience. **Use the highest one that works:**

1. **`getByRole`** — by ARIA role (and accessible name): `getByRole('button', { name: /submit/i })`. The gold standard; reflects accessibility.
2. **`getByLabelText`** — form fields by their `<label>`. Best for inputs.
3. **`getByPlaceholderText`** — fallback when no label.
4. **`getByText`** — non-interactive content (paragraphs, divs).
5. **`getByDisplayValue`** — current value of a filled input.
6. **`getByAltText`** — images.
7. **`getByTitle`** — `title` attribute.
8. **`getByTestId`** — `data-testid`. **Last resort**, not user-visible.

### `@testing-library/user-event`

```jsx
import userEvent from '@testing-library/user-event';

const user = userEvent.setup();   // call once per test, before render
await user.click(button);          // every interaction is async — always await
await user.type(input, 'hello');
await user.keyboard('{Enter}');
await user.selectOptions(select, 'a');
await user.clear(input);
```

`user-event` simulates **full event sequences** (pointerdown → focus → mouseup → click, etc.), unlike the lower-level `fireEvent` which dispatches a single synthetic event. Prefer `user-event` for realism.

### jest-dom matchers (custom assertions)

```jsx
expect(el).toBeInTheDocument();
expect(el).toHaveTextContent('Count: 1');
expect(el).toBeVisible();
expect(el).toBeDisabled();        // / .toBeEnabled()
expect(input).toHaveValue('hi');
expect(input).toBeChecked();
expect(el).toHaveClass('active');
expect(el).toHaveAttribute('href', '/home');
expect(el).toHaveFocus();
```

### Async helpers

```jsx
import { waitFor, waitForElementToBeRemoved } from '@testing-library/react';

await waitFor(() => expect(mockFn).toHaveBeenCalled());  // retries the callback
await waitForElementToBeRemoved(() => screen.queryByText('Loading…'));
const row = await screen.findByText('Ada');               // = getBy wrapped in waitFor
```

### Hook testing

```jsx
import { renderHook, act } from '@testing-library/react';

const { result, rerender, unmount } = renderHook(() => useCounter(5));
act(() => { result.current.increment(); });
expect(result.current.count).toBe(6);
```

### Mocking with Vitest

```js
import { vi } from 'vitest';

const fn = vi.fn();                         // spy
vi.mock('./api');                           // auto-mock a module
vi.spyOn(console, 'error');                 // spy on a real method
vi.useFakeTimers();                         // control time
```

---

## Examples

### 0. Project setup

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm install -D vitest jsdom @testing-library/react @testing-library/user-event \
  @testing-library/jest-dom @testing-library/dom
```

`vite.config.js` — add the `test` block (Vitest reads the same config as Vite):

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,            // describe/it/expect without imports
    environment: 'jsdom',     // a DOM in Node
    setupFiles: './src/setupTests.js',
    css: false,
  },
});
```

`src/setupTests.js` — wire in the jest-dom matchers once:

```js
import '@testing-library/jest-dom/vitest';
```

`package.json` scripts:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "coverage": "vitest run --coverage"
  }
}
```

> **Jest instead of Vitest?** The RTL/jest-dom code below is identical. Replace `vi` with `jest`, use `jest.config.js` with `testEnvironment: 'jsdom'`, and put the jest-dom import in a file referenced by `setupFilesAfterEnv`. Vitest is the natural choice in Vite projects: same config, same transformer, near-zero setup, and an ESM-native runtime.

### 1. A trivial component — render and assert

`Greeting.jsx`:

```jsx
export default function Greeting({ name }) {
  return <h1>Hello, {name || 'stranger'}!</h1>;
}
```

`Greeting.test.jsx`:

```jsx
import { render, screen } from '@testing-library/react';
import Greeting from './Greeting';

describe('Greeting', () => {
  it('renders the provided name', () => {
    render(<Greeting name="Ada" />);
    expect(
      screen.getByRole('heading', { name: 'Hello, Ada!' })
    ).toBeInTheDocument();
  });

  it('falls back to "stranger" when no name is given', () => {
    render(<Greeting />);
    expect(screen.getByText(/hello, stranger!/i)).toBeInTheDocument();
  });
});
```

We query by **role** (`heading`) and by **text** — never by class name or test id. The second test exercises **conditional rendering** driven by a prop.

### 2. State changes and event handlers

`Counter.jsx`:

```jsx
import { useState } from 'react';

export default function Counter({ start = 0, onChange }) {
  const [count, setCount] = useState(start);

  function increment() {
    const next = count + 1;
    setCount(next);
    onChange?.(next);
  }

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```

`Counter.test.jsx`:

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Counter from './Counter';

describe('Counter', () => {
  it('increments the displayed count on click', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    expect(screen.getByText('Count: 0')).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: /increment/i }));

    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });

  it('respects the start prop', () => {
    render(<Counter start={10} />);
    expect(screen.getByText('Count: 10')).toBeInTheDocument();
  });

  it('calls onChange with the new value (testing a callback prop)', async () => {
    const user = userEvent.setup();
    const handleChange = vi.fn();
    render(<Counter onChange={handleChange} />);

    await user.click(screen.getByRole('button'));

    expect(handleChange).toHaveBeenCalledTimes(1);
    expect(handleChange).toHaveBeenCalledWith(1);
  });
});
```

Note we test the **outcome** ("Count: 1" is on screen) and the **callback contract** (`onChange` got `1`) — not that `count` is internally `1`.

### 3. A controlled form

`LoginForm.jsx`:

```jsx
import { useState } from 'react';

export default function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  const [error, setError] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    if (!email.includes('@')) {
      setError('Please enter a valid email');
      return;
    }
    setError('');
    onSubmit({ email });
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input id="email" value={email} onChange={(e) => setEmail(e.target.value)} />
      {error && <p role="alert">{error}</p>}
      <button type="submit">Log in</button>
    </form>
  );
}
```

`LoginForm.test.jsx`:

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import LoginForm from './LoginForm';

describe('LoginForm', () => {
  it('shows a validation error for a bad email and does not submit', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'not-an-email');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(screen.getByRole('alert')).toHaveTextContent('valid email');
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it('submits the email when valid', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'ada@example.com');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(onSubmit).toHaveBeenCalledWith({ email: 'ada@example.com' });
    expect(screen.queryByRole('alert')).not.toBeInTheDocument(); // asserting absence
  });
});
```

`getByLabelText` finds the input through its `<label htmlFor>` association — exactly how a screen reader and a sighted user find it. We use `queryByRole` (not `getBy`) to assert the alert is **absent**.

### 4. Async data fetching with `findBy`

`UserCard.jsx`:

```jsx
import { useEffect, useState } from 'react';

export default function UserCard({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(false);

  useEffect(() => {
    let active = true;
    fetch(`/api/users/${userId}`)
      .then((r) => {
        if (!r.ok) throw new Error('bad response');
        return r.json();
      })
      .then((data) => active && setUser(data))
      .catch(() => active && setError(true));
    return () => { active = false; };
  }, [userId]);

  if (error) return <p role="alert">Failed to load user</p>;
  if (!user) return <p>Loading…</p>;
  return <h2>{user.name}</h2>;
}
```

Test it by mocking `fetch` (we'll upgrade to MSW next):

```jsx
import { render, screen } from '@testing-library/react';
import UserCard from './UserCard';

describe('UserCard', () => {
  afterEach(() => vi.restoreAllMocks());

  it('shows the user name after fetch resolves', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue({
      ok: true,
      json: async () => ({ name: 'Grace Hopper' }),
    });

    render(<UserCard userId={1} />);

    // Loading state is shown synchronously...
    expect(screen.getByText('Loading…')).toBeInTheDocument();

    // ...then the name appears asynchronously. findBy retries until it does.
    expect(await screen.findByRole('heading', { name: 'Grace Hopper' }))
      .toBeInTheDocument();
  });

  it('shows an error message when the request fails', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue({ ok: false });
    render(<UserCard userId={1} />);
    expect(await screen.findByRole('alert')).toHaveTextContent(/failed to load/i);
  });
});
```

`findByRole` is `getByRole` wrapped in `waitFor`: it polls the DOM (default up to 1000 ms) until the heading exists, so we never need manual timers. **Whenever the UI updates after a promise, use `findBy` / `waitFor`** and `await` it — that also satisfies React's `act()` requirement so you won't see "not wrapped in act(...)" warnings.

### 5. Realistic API mocking with MSW

Hand-stubbing `fetch` couples tests to implementation (you stub the exact call). **MSW (Mock Service Worker)** intercepts requests at the network layer, so your component uses the *real* `fetch`/`axios` and MSW answers as if it were the server. This is the recommended approach for anything beyond trivial.

```bash
npm install -D msw
```

`src/mocks/server.js`:

```js
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Grace Hopper' })
  ),
];

export const server = setupServer(...handlers);
```

Wire it into `src/setupTests.js`:

```js
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers()); // undo per-test overrides
afterAll(() => server.close());
```

Now the `UserCard` test needs **no `fetch` mock** at all, and overriding one test is clean:

```jsx
import { http, HttpResponse } from 'msw';
import { render, screen } from '@testing-library/react';
import { server } from './mocks/server';
import UserCard from './UserCard';

it('uses the default MSW handler', async () => {
  render(<UserCard userId={1} />);
  expect(await screen.findByText('Grace Hopper')).toBeInTheDocument();
});

it('renders an error when the server 500s', async () => {
  server.use(
    http.get('/api/users/:id', () => new HttpResponse(null, { status: 500 }))
  );
  render(<UserCard userId={1} />);
  expect(await screen.findByRole('alert')).toHaveTextContent(/failed/i);
});
```

### 6. A full test file for a custom hook

`useCounter.js`:

```js
import { useCallback, useState } from 'react';

export default function useCounter(initial = 0, { step = 1 } = {}) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount((c) => c + step), [step]);
  const decrement = useCallback(() => setCount((c) => c - step), [step]);
  const reset = useCallback(() => setCount(initial), [initial]);
  return { count, increment, decrement, reset };
}
```

`useCounter.test.js`:

```js
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import useCounter from './useCounter';

describe('useCounter', () => {
  it('starts at the initial value', () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });

  it('increments and decrements by the step', () => {
    const { result } = renderHook(() => useCounter(0, { step: 3 }));

    act(() => result.current.increment());
    expect(result.current.count).toBe(3);

    act(() => result.current.decrement());
    expect(result.current.count).toBe(0);
  });

  it('resets to the initial value', () => {
    const { result } = renderHook(() => useCounter(10));
    act(() => result.current.increment());
    act(() => result.current.reset());
    expect(result.current.count).toBe(10);
  });

  it('reads new props after rerender', () => {
    const { result, rerender } = renderHook(
      ({ step }) => useCounter(0, { step }),
      { initialProps: { step: 1 } }
    );
    act(() => result.current.increment());
    expect(result.current.count).toBe(1);

    rerender({ step: 5 });
    act(() => result.current.increment());
    expect(result.current.count).toBe(6);
  });
});
```

`result.current` is a live reference to the hook's latest return value. **Any call that triggers a state update must be wrapped in `act()`** so React flushes the update before your assertions run. For hooks with their own async effects, you can `await` an async `act` or use `waitFor` against `result.current`. (For hooks that depend on context — e.g. a router or query client — pass a `wrapper` option to `renderHook`.)

### 7. Snapshot test

```jsx
import { render } from '@testing-library/react';
import Greeting from './Greeting';

it('matches the snapshot', () => {
  const { asFragment } = render(<Greeting name="Ada" />);
  expect(asFragment()).toMatchSnapshot();
});
```

First run writes `__snapshots__/...snap`; later runs diff against it. Update intentionally with `vitest -u`. Use snapshots sparingly (see caveats below).

---

## Under the hood

**jsdom is not a browser.** Vitest runs your tests in Node with `environment: 'jsdom'`, which is a pure-JS implementation of the DOM and HTML standards. It gives you `document`, `window`, elements, and events — but **no layout engine and no real rendering**. So `getBoundingClientRect()` returns zeros, `IntersectionObserver`/`ResizeObserver`/`matchMedia` are missing (polyfill or mock them), and `toBeVisible()` works off CSS `display`/`visibility`/`hidden` attributes, not actual pixels. When a test genuinely needs layout, that's a signal to use a real-browser tool (Playwright, or Vitest's experimental browser mode).

**What `render` really does.** It creates a container `div`, appends it to `document.body`, and calls React's `createRoot(container).render(ui)` inside `act()`. Because it mounts into the global document, `screen` (bound to `document.body`) can find anything rendered, including React portals (modals/tooltips) that escape the container.

**`act()` and the warning you'll eventually hit.** React batches state updates and runs effects in a unit called `act`. `render` and `user-event` already wrap their work in `act` for you. The famous *"An update to X inside a test was not wrapped in act(...)"* warning means a state update landed **after** your synchronous test code finished — almost always an un-awaited async update. The fix is essentially never to add a bare `act()`; it's to `await` a `findBy`/`waitFor` so the test stays alive until React settles.

**`findBy` = `getBy` + `waitFor`.** Internally `findByText(...)` calls `waitFor(() => getByText(...))`. `waitFor` re-runs your callback on a timer and on DOM mutations (via a `MutationObserver`) until it stops throwing or the timeout (default 1000 ms) elapses. That's why a `findBy` callback must *throw* on failure (a `getBy` does) — a `queryBy` returning `null` wouldn't trigger a retry.

**Why no access to state/props/instances.** RTL's queries operate only on the rendered DOM by design. There is no `wrapper.state()` or `wrapper.find(Child).props()` like the old Enzyme library offered, because those couple tests to internals. This is a feature: it forces black-box, refactor-proof tests.

**`user-event` vs `fireEvent`.** `fireEvent.click(el)` dispatches one `click` event. `user.click(el)` dispatches the entire sequence a browser would (`pointerover`, `pointerdown`, `mousedown`, `focus`, `pointerup`, `mouseup`, `click`) and respects things like `disabled` and pointer-events. That realism catches bugs (e.g. handlers attached to the wrong event) that `fireEvent` would miss. `user-event` is async because it advances microtasks between events.

---

## Common mistakes & pitfalls

**1. Forgetting to `await` user-event / async queries.**

```jsx
// ❌ WRONG — user.click returns a promise; assertion races the update
it('increments', () => {
  render(<Counter />);
  userEvent.click(screen.getByRole('button')); // not awaited, no setup()
  expect(screen.getByText('Count: 1')).toBeInTheDocument(); // flaky / fails
});
```

```jsx
// ✅ RIGHT
it('increments', async () => {
  const user = userEvent.setup();
  render(<Counter />);
  await user.click(screen.getByRole('button'));
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

**2. Using `getBy` to assert absence (it throws and crashes the test).**

```jsx
expect(screen.getByText('Error')).not.toBeInTheDocument(); // ❌ throws before the assertion
expect(screen.queryByText('Error')).not.toBeInTheDocument(); // ✅ queryBy returns null
```

**3. Reaching for `data-testid` first.**

```jsx
<button data-testid="submit-btn">Submit</button>;
screen.getByTestId('submit-btn'); // ❌ couples to a non-user attribute
```

```jsx
<button>Submit</button>;
screen.getByRole('button', { name: /submit/i }); // ✅ tests the accessible label
```

**4. Testing implementation details / internal state.**

```jsx
// ❌ "WRONG": asserting on internals (also impossible in RTL by design)
expect(component.state.isOpen).toBe(true);
```

```jsx
// ✅ RIGHT: assert what the user sees
await user.click(screen.getByRole('button', { name: /open/i }));
expect(screen.getByRole('dialog')).toBeVisible();
```

**5. Manual `act()` wrapping to silence warnings.**

```jsx
act(() => {}); // ❌ cargo-cult; hides the real issue
```

```jsx
expect(await screen.findByText('Loaded')).toBeInTheDocument(); // ✅ await the settle
```

**6. Wrapping reads of `result.current` instead of the update.**

```jsx
act(() => {
  expect(result.current.count).toBe(1); // ❌ assert outside act
});
```

```jsx
act(() => result.current.increment()); // ✅ act wraps the update only
expect(result.current.count).toBe(1);  // assert after
```

**7. `onUnhandledRequest` left silent** — a typo in a URL means MSW doesn't match and your component hangs. Set `server.listen({ onUnhandledRequest: 'error' })` so unmatched requests fail loudly.

---

## Best practices

- **Do** query by role/label/text; reserve `getByTestId` for the rare case with no accessible handle. Add roles/labels to your markup instead of test ids.
- **Do** use `user-event` (via `userEvent.setup()`), and `await` every interaction and every `findBy`/`waitFor`.
- **Do** assert on user-visible outcomes and callback contracts, never on internal state, instances, or child props.
- **Do** mock at the network boundary with **MSW**, not by stubbing `fetch` everywhere — your tests then exercise real request code.
- **Do** keep one clear behavior per `it`, with a descriptive name ("submits the email when valid").
- **Don't** test third-party libraries (React Router, your UI kit) — test *your* usage of them.
- **Don't** chase 100% coverage; cover branches and edge cases that matter. Coverage is a flashlight, not a goal.
- **Don't** snapshot large trees; prefer targeted assertions. Big snapshots get rubber-stamped on update and catch nothing.
- **Don't** put logic (loops, conditionals) in tests — they become things you'd want to test themselves.
- **Prefer** `*ByRole` with the `name` option; it doubles as a free accessibility check.

### Snapshot caveats

Snapshots are tempting but low-signal for components: a deliberate text change "breaks" them, developers update with `-u` without reading the diff, and they don't express intent. Use them for **small, stable** serializable output (a formatting util, an error object, a tiny presentational component), and keep them inline (`toMatchInlineSnapshot()`) so the expected value lives next to the test.

### Coverage

```bash
npm install -D @vitest/coverage-v8
npm run coverage
```

Add thresholds so coverage can't silently regress:

```js
// vite.config.js → test block
coverage: {
  provider: 'v8',
  reporter: ['text', 'html'],
  thresholds: { lines: 80, functions: 80, branches: 75, statements: 80 },
},
```

Treat the numbers as a conversation starter ("why is this branch never hit?"), not a target to game.

### E2E overview (Playwright / Cypress)

RTL+jsdom can't validate real navigation, real network, CSS layout, or cross-tab flows. For those critical journeys, add a **few** end-to-end tests in a real browser:

```js
// Playwright — e2e/login.spec.js
import { test, expect } from '@playwright/test';

test('user can log in', async ({ page }) => {
  await page.goto('http://localhost:5173');
  await page.getByLabel('Email').fill('ada@example.com');
  await page.getByRole('button', { name: 'Log in' }).click();
  await expect(page.getByText('Welcome, Ada')).toBeVisible();
});
```

Note the API mirrors RTL (`getByRole`, `getByLabel`) — your behavior-first habits transfer directly. Playwright runs cross-browser headless and is the current default recommendation; **Cypress** is the long-standing alternative with an excellent interactive runner. Keep E2E for happy-path + a couple of critical failure paths; let unit/integration tests cover the matrix.

---

## Exercises

1. **Toggle.** Write a `Toggle` component with a button labeled "Show"/"Hide" that conditionally renders a `<p>Secret</p>`. Test: it's hidden initially, appears after one click, disappears after a second click.
2. **Debounced search.** Build a `SearchBox` that calls an `onSearch` prop 300 ms after the user stops typing. Test it using Vitest fake timers so the test stays fast.
3. **MSW list.** Component `UserList` fetches `/api/users` and renders an `<li>` per user. Write tests for the success case (default handler) and an empty-list case (per-test `server.use` override) using MSW.
4. **Custom hook `useToggle`.** `const [on, toggle, set] = useToggle(false)`. Test initial value, `toggle`, and `set(true)` using `renderHook` + `act`.

### Solutions

```jsx
// 1. Toggle
import { useState } from 'react';
export function Toggle() {
  const [open, setOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setOpen((o) => !o)}>{open ? 'Hide' : 'Show'}</button>
      {open && <p>Secret</p>}
    </div>
  );
}

// Toggle.test.jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Toggle } from './Toggle';

it('toggles the secret', async () => {
  const user = userEvent.setup();
  render(<Toggle />);
  expect(screen.queryByText('Secret')).not.toBeInTheDocument();

  await user.click(screen.getByRole('button', { name: 'Show' }));
  expect(screen.getByText('Secret')).toBeInTheDocument();

  await user.click(screen.getByRole('button', { name: 'Hide' }));
  expect(screen.queryByText('Secret')).not.toBeInTheDocument();
});
```

```jsx
// 2. Debounced search
import { useEffect, useState } from 'react';
export function SearchBox({ onSearch }) {
  const [q, setQ] = useState('');
  useEffect(() => {
    const id = setTimeout(() => onSearch(q), 300);
    return () => clearTimeout(id);
  }, [q, onSearch]);
  return <input aria-label="Search" value={q} onChange={(e) => setQ(e.target.value)} />;
}

// SearchBox.test.jsx
import { render, screen, act } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SearchBox } from './SearchBox';

it('debounces calls to onSearch', async () => {
  vi.useFakeTimers();
  const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
  const onSearch = vi.fn();
  render(<SearchBox onSearch={onSearch} />);

  await user.type(screen.getByLabelText('Search'), 'react');
  expect(onSearch).not.toHaveBeenCalledWith('react'); // not yet

  act(() => vi.advanceTimersByTime(300));
  expect(onSearch).toHaveBeenLastCalledWith('react');
  vi.useRealTimers();
});
```

```jsx
// 3. UserList with MSW
import { useEffect, useState } from 'react';
export function UserList() {
  const [users, setUsers] = useState([]);
  useEffect(() => {
    fetch('/api/users').then((r) => r.json()).then(setUsers);
  }, []);
  if (users.length === 0) return <p>No users</p>;
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}

// UserList.test.jsx
import { http, HttpResponse } from 'msw';
import { render, screen } from '@testing-library/react';
import { server } from './mocks/server';
import { UserList } from './UserList';

it('renders the fetched users', async () => {
  server.use(http.get('/api/users', () =>
    HttpResponse.json([{ id: 1, name: 'Ada' }, { id: 2, name: 'Linus' }])));
  render(<UserList />);
  expect(await screen.findByText('Ada')).toBeInTheDocument();
  expect(screen.getByText('Linus')).toBeInTheDocument();
});

it('shows an empty state', async () => {
  server.use(http.get('/api/users', () => HttpResponse.json([])));
  render(<UserList />);
  expect(await screen.findByText('No users')).toBeInTheDocument();
});
```

```js
// 4. useToggle
import { useCallback, useState } from 'react';
export function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []);
  return [on, toggle, setOn];
}

// useToggle.test.js
import { renderHook, act } from '@testing-library/react';
import { useToggle } from './useToggle';

it('toggles and sets', () => {
  const { result } = renderHook(() => useToggle(false));
  expect(result.current[0]).toBe(false);

  act(() => result.current[1]());           // toggle
  expect(result.current[0]).toBe(true);

  act(() => result.current[2](false));      // set(false)
  expect(result.current[0]).toBe(false);
});
```

---

## Key takeaways

- **Test behavior, not implementation.** Query the DOM the way a user would; never assert on internal state, instances, or child props.
- **Vitest + RTL + jsdom + jest-dom** is the modern stack for Vite apps: `globals: true`, `environment: 'jsdom'`, a `setupTests` that imports the matchers.
- **Query priority:** `getByRole` → `getByLabelText` → `getByText` → … → `getByTestId` (last resort). Use `getBy` for present, `queryBy` for absent, `findBy` for async.
- **Always `await`** `user-event` interactions and `findBy`/`waitFor` — that also resolves React's `act()` warnings.
- **Mock at the network boundary with MSW** rather than stubbing `fetch`; your components run their real request code.
- **Test hooks** with `renderHook` + `act`, wrapping only the state-changing call in `act`.
- **Snapshots and coverage are tools, not targets** — keep snapshots tiny, treat coverage as a diagnostic.
- **A few E2E tests** (Playwright/Cypress) cover what jsdom can't: real browser, navigation, and layout.

---

## Further reading

- React docs — Testing overview: <https://react.dev/learn/testing> (see also the Hooks testing notes)
- React Testing Library docs: <https://testing-library.com/docs/react-testing-library/intro>
- Guiding Principles & query priority: <https://testing-library.com/docs/queries/about/#priority>
- `user-event`: <https://testing-library.com/docs/user-event/intro>
- `jest-dom` matchers: <https://github.com/testing-library/jest-dom>
- Vitest: <https://vitest.dev/> · Coverage: <https://vitest.dev/guide/coverage.html>
- MSW (Mock Service Worker): <https://mswjs.io/docs/>
- Playwright: <https://playwright.dev/> · Cypress: <https://www.cypress.io/>
- Kent C. Dodds — *Common mistakes with React Testing Library*: <https://kentcdodds.com/blog/common-mistakes-with-react-testing-library>
- Kent C. Dodds — *The Testing Trophy*: <https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications>

> **Cross-references:** state and events (`07-state-usestate.md`), effects and fetching (`08-useeffect-side-effects.md`), and custom hooks (`10-custom-hooks.md`) are the features most exercised here. A later lesson on CI can wire `vitest run --coverage` and Playwright into your pipeline.
