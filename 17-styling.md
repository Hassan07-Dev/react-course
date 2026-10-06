# 17 — Styling React Apps

> **What you'll learn:** every practical way to put styles on the screen in a modern React 19 + Vite app — from plain CSS to CSS Modules, Sass, inline styles, CSS-in-JS, Tailwind, and full component libraries — plus the judgment to pick the right one.

By the end of this lesson you will be able to:

- Import plain CSS and reason about the global, cascading nature of stylesheets in a component world.
- Use **CSS Modules** for locally-scoped class names and `composes` for reuse.
- Apply **inline style objects** correctly and explain exactly where they fall short.
- Set up **Sass/SCSS** in Vite and use nesting, variables, and partials.
- Use **CSS-in-JS** (styled-components, Emotion): tagged templates, props-driven styles, theming — and articulate the runtime cost and the industry shift to zero-runtime.
- Set up **Tailwind CSS** with Vite, use utility classes, responsive/state variants, `@apply`, and theme configuration.
- Conditionally combine class names with **clsx**.
- Build a **light/dark theme** with CSS custom properties.
- Survey **MUI, Chakra UI, Ant Design, and shadcn/ui**, and choose an approach for a real project.

---

## Prerequisites

You should already be comfortable with:

- JSX, including the `className` attribute and embedding JavaScript expressions with `{}` (see `03-jsx-in-depth.md`).
- Components and props (see `04-components-and-props.md`).
- Conditional rendering and lists (see `05-rendering-lists-conditionals.md`).
- `useState` for the interactive examples (see `07-state-usestate.md`).
- A working **Vite** project created with `npm create vite@latest` (see `02-getting-started.md`).

A little plain CSS knowledge — selectors, the cascade, specificity, the box model — will make this lesson land. You don't need to be a CSS expert; we'll build the React-specific intuition here.

---

## Concept

In a browser, **styles are global by default**. A `.card` rule in any stylesheet styles *every* element with `class="card"` on the whole page. The cascade resolves conflicts by specificity and source order. This model is fine for a hand-authored page but becomes hostile at scale: two engineers independently write `.title { color: red }` and `.title { color: blue }`, and whoever loads last wins. Nobody can safely delete a CSS rule because they can't prove nothing else depends on it.

React pushes us toward **components** — self-contained, reusable units of UI. The natural wish is for styling to be component-scoped too: a `Button`'s styles should *belong* to `Button` and not leak out or be clobbered. Every styling approach in this lesson is, at heart, an answer to one question:

> **How do we keep the power of CSS while taming its global, leaky nature so styles compose like components do?**

The approaches sit on a spectrum:

- **Plain CSS** — global, you manage scoping by hand (naming conventions like BEM).
- **CSS Modules** — the *build tool* renames your classes to be unique, so they're scoped automatically. Still real CSS files.
- **Sass/SCSS** — a CSS superset that adds nesting/variables/mixins; orthogonal to scoping (you can use Sass *with* Modules).
- **Inline styles** — styles live on the element via the `style` prop; trivially scoped, but a limited subset of CSS.
- **CSS-in-JS** — you write styles *in JavaScript*, colocated with the component; the library generates scoped class names at runtime (or, increasingly, at build time).
- **Utility-first (Tailwind)** — you compose pre-made atomic classes (`flex`, `p-4`, `text-red-500`) in `className`; the build tool emits only the CSS you used.
- **Component libraries** — someone already made the buttons, modals, and tables, styled and accessible; you assemble.

There is no single winner. The right tool depends on team size, design-system maturity, performance budget, and SSR strategy. We'll end with concrete guidance.

A useful analogy: plain CSS is a **shared whiteboard** anyone can write on (and erase). CSS Modules give each component its **own labeled drawer**. Inline styles are **sticky notes** on individual elements — fast but you can't write a media query on a sticky note. Tailwind is a **box of Lego bricks**: small, composable, no naming. CSS-in-JS is a **3D printer** that fabricates the exact brick you describe in code.

---

## Syntax & API

### Plain CSS import

```jsx
import './App.css';            // global side-effect import; injected into <head>
<div className="card">…</div>  // matches .card anywhere on the page
```

Vite treats `import './x.css'` as a side effect: the bundler collects the CSS and injects it. There is no scoping — the file is global.

### CSS Modules

```jsx
import styles from './Button.module.css'; // file MUST end in .module.css
<button className={styles.primary}>…</button>
```

`styles` is an object mapping your authored class names to generated unique names (e.g. `Button_primary_a1b2c`). Compose with `composes` *inside* the CSS file:

```css
.primary { composes: base; background: rebeccapurple; }
```

### Inline style object

```jsx
<div style={{ backgroundColor: 'tomato', paddingTop: 8 }} />
```

`style` takes a **JS object**, not a string. Properties are **camelCase** (`backgroundColor`, not `background-color`). Numeric values for length properties get `px` appended automatically (`paddingTop: 8` → `8px`); unitless properties (`lineHeight`, `zIndex`, `opacity`) stay raw. Custom properties keep their dashes: `{ '--accent': 'red' }`.

### Sass/SCSS (Vite)

```bash
npm install -D sass-embedded   # or: sass
```

```jsx
import './styles.scss';                 // global
import styles from './Card.module.scss'; // scoped via Modules + Sass together
```

### styled-components / Emotion (tagged template literals)

```jsx
import styled from 'styled-components';   // or '@emotion/styled'

const Button = styled.button`
  background: ${(props) => (props.$primary ? 'rebeccapurple' : 'white')};
  padding: 8px 16px;
`;
<Button $primary>Click</Button>
```

`styled.button` returns a real React component that renders a `<button>` with a generated class. `$`-prefixed props are *transient* in styled-components: they're used for styling but **not** forwarded to the DOM.

### Tailwind utility classes

```jsx
<button className="rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700 md:px-6">
  Save
</button>
```

`hover:` and `md:` are **variants**: state and responsive prefixes that conditionally apply a utility.

### clsx — conditional class composition

```jsx
import clsx from 'clsx';
clsx('btn', isActive && 'btn--active', { 'btn--disabled': disabled });
// → "btn btn--active"  (falsy entries dropped, object keys included when truthy)
```

---

## Examples

We'll build the same `Button` several ways so you can compare directly. Assume a fresh Vite + React project.

### Example 1 — Plain CSS + global stylesheet

```css
/* src/Button.css */
.btn {
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  font: inherit;
  cursor: pointer;
}
.btn--primary { background: rebeccapurple; color: white; }
.btn--ghost   { background: transparent; color: rebeccapurple; }
```

```jsx
// src/Button.jsx
import './Button.css';

export default function Button({ variant = 'primary', children, ...rest }) {
  return (
    <button className={`btn btn--${variant}`} {...rest}>
      {children}
    </button>
  );
}
```

Simple and zero-tooling. The risk: `.btn` and `.btn--primary` are global. Another file's `.btn` will collide. The BEM naming (`block__element--modifier`) is a *convention* to reduce collisions, not an enforcement.

### Example 2 — CSS Modules (automatic scoping)

```css
/* src/Button.module.css */
.base {
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  font: inherit;
  cursor: pointer;
}
.primary { composes: base; background: rebeccapurple; color: white; }
.ghost   { composes: base; background: transparent; color: rebeccapurple; }
```

```jsx
// src/Button.jsx
import styles from './Button.module.css';

export default function Button({ variant = 'primary', children, ...rest }) {
  return (
    <button className={styles[variant]} {...rest}>
      {children}
    </button>
  );
}
```

Now `styles.primary` becomes something like `Button_primary_x7f2a` at build time — **guaranteed unique**. You can name a class `.base` here and `.base` in another module with zero risk. `composes: base` pulls in the shared rules without duplicating them; the generated `className` will contain *both* hashed names. This is the workhorse of medium-sized apps: real CSS files, full tooling support, no global leakage.

### Example 3 — Inline styles for truly dynamic values

Inline styles shine when a value is computed at runtime and can't be a fixed class — e.g. a progress bar width driven by state.

```jsx
// src/ProgressBar.jsx
import { useState } from 'react';

export default function ProgressBar() {
  const [pct, setPct] = useState(40);
  return (
    <div>
      <div style={{ height: 12, background: '#eee', borderRadius: 6 }}>
        <div
          style={{
            width: `${pct}%`,           // dynamic — perfect for inline
            height: '100%',
            background: 'rebeccapurple',
            borderRadius: 6,
            transition: 'width 200ms',
          }}
        />
      </div>
      <input type="range" value={pct} onChange={(e) => setPct(+e.target.value)} />
    </div>
  );
}
```

A `width: 40%` would need a unique class per value — wasteful. Inline is the right call here. But note the limits (covered in *Under the hood*): no `:hover`, no media queries, no pseudo-elements.

### Example 4 — Sass/SCSS with Modules

```bash
npm install -D sass-embedded
```

```scss
/* src/Card.module.scss */
$radius: 10px;
$pad: 16px;

.card {
  border-radius: $radius;
  padding: $pad;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);

  .title {            // real nesting — compiles to .card .title
    margin: 0 0 8px;
    font-size: 1.2rem;
  }

  &:hover { box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15); }
}
```

```jsx
// src/Card.jsx
import styles from './Card.module.scss';

export default function Card({ title, children }) {
  return (
    <section className={styles.card}>
      <h2 className={styles.title}>{title}</h2>
      {children}
    </section>
  );
}
```

Sass adds variables, nesting, and `@mixin`/`@include`. Combined with Modules (`*.module.scss`) you keep scoping *and* get Sass ergonomics. Note: modern plain CSS now has native nesting and custom properties, so Sass is less essential than it once was — but mixins and loops still have no CSS equivalent.

### Example 5 — styled-components with props and a theme

```bash
npm install styled-components
```

```jsx
// src/styled/Button.jsx
import styled from 'styled-components';

const Button = styled.button`
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  cursor: pointer;
  color: ${(p) => p.theme.onPrimary};
  background: ${(p) => (p.$variant === 'ghost' ? 'transparent' : p.theme.primary)};
  &:hover { filter: brightness(1.1); }
`;
export default Button;
```

```jsx
// src/App.jsx
import { ThemeProvider } from 'styled-components';
import Button from './styled/Button.jsx';

const theme = { primary: 'rebeccapurple', onPrimary: 'white' };

export default function App() {
  return (
    <ThemeProvider theme={theme}>
      <Button>Primary</Button>
      <Button $variant="ghost" style={{ color: 'rebeccapurple' }}>Ghost</Button>
    </ThemeProvider>
  );
}
```

Styles are colocated with the component and react to props and a theme via React Context. Emotion's `@emotion/styled` API is nearly identical; Emotion also offers a `css` prop. The tradeoff — runtime cost — is in *Under the hood*.

### Example 6 — Tailwind + clsx (the modern default for many teams)

Set up Tailwind v4 with the official Vite plugin:

```bash
npm install tailwindcss @tailwindcss/vite
npm install clsx
```

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

```css
/* src/index.css */
@import "tailwindcss";

/* Customize the theme with CSS variables (Tailwind v4 style) */
@theme {
  --color-brand: oklch(0.55 0.2 295);
}
```

```jsx
// src/Button.jsx
import clsx from 'clsx';

export default function Button({ variant = 'primary', className, ...rest }) {
  return (
    <button
      className={clsx(
        'rounded px-4 py-2 font-medium transition hover:brightness-110',
        variant === 'primary' && 'bg-brand text-white',
        variant === 'ghost' && 'bg-transparent text-brand',
        className, // let callers extend/override
      )}
      {...rest}
    />
  );
}
```

```jsx
// src/index.css already imported in main.jsx; usage:
<Button>Save</Button>
<Button variant="ghost" className="md:px-6">Cancel</Button>
```

Tailwind v4 configures the theme **in CSS** via `@theme` (the old `tailwind.config.js` is optional now). `bg-brand` exists because we declared `--color-brand`. The `@tailwindcss/vite` plugin scans your files and emits only the utilities you actually used. `clsx` keeps the conditional class logic readable.

If you find yourself repeating the same utility cluster, extract it with `@apply`:

```css
/* src/index.css */
@layer components {
  .btn-primary {
    @apply rounded bg-brand px-4 py-2 font-medium text-white transition hover:brightness-110;
  }
}
```

Use `@apply` sparingly — overusing it recreates the global-class problem Tailwind was meant to avoid.

### Example 7 — Light/dark theme with CSS custom properties

This approach needs **no library** and works with any of the above. Define tokens on `:root` and override under a `data-theme` attribute or a media query.

```css
/* src/theme.css */
:root {
  --bg: #ffffff;
  --fg: #1a1a1a;
  --accent: rebeccapurple;
}
:root[data-theme='dark'] {
  --bg: #121212;
  --fg: #f0f0f0;
  --accent: #b794f6;
}
body { background: var(--bg); color: var(--fg); transition: background 200ms; }
```

```jsx
// src/ThemeToggle.jsx
import { useEffect, useState } from 'react';

export default function ThemeToggle() {
  const [theme, setTheme] = useState(
    () => localStorage.getItem('theme') ?? 'light',
  );

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
  }, [theme]); // see 08-useeffect-side-effects.md for the effect rules

  return (
    <button onClick={() => setTheme((t) => (t === 'light' ? 'dark' : 'light'))}>
      Switch to {theme === 'light' ? 'dark' : 'light'} mode
    </button>
  );
}
```

Flipping one attribute on `<html>` re-themes the entire app instantly because every color reads from a `var(--…)`. This is the foundation that MUI, Chakra, and shadcn/ui all build on.

---

## Under the hood

**How CSS Modules scope.** During the build, Vite (via PostCSS / Lightning CSS) parses each `*.module.css` file and rewrites every class selector to a hashed name derived from the filename, the original name, and the content. It produces a JS map (`{ primary: 'Button_primary_x7f2a' }`) that you import. There's *zero* runtime cost — it's pure compile-time renaming. The collision-proofing is the hash, not magic.

**Why inline styles can't do everything.** The `style` prop becomes the element's inline `style` attribute. CSS pseudo-classes (`:hover`, `:focus`), pseudo-elements (`::before`), media queries, `@keyframes`, and descendant selectors **cannot** exist in an inline style — they require a stylesheet rule with a selector. Inline styles also have very high specificity (they beat almost any class rule), which makes them hard to override. And React sets them via the DOM `style` object property-by-property, which for large dynamic objects can be marginally slower than toggling a class. Rule of thumb: inline only for *values* that are genuinely computed at runtime; everything static belongs in a class.

**How CSS-in-JS generates styles at runtime.** When you render a `styled.button`, the library: (1) evaluates the template literal with the current props/theme, (2) hashes the resulting CSS string into a class name, (3) checks an in-memory cache; if new, it **inserts a rule into a `<style>` tag in the document head**, then (4) attaches that class to the element. This is *runtime* work on every render where styles depend on changing props. For most apps it's imperceptible, but on large lists or under SSR it shows up: serializing and re-hydrating the inserted styles adds payload and CPU. This cost — plus React 19's compiler and Server Components disliking render-time side effects — is why the industry is moving to **zero-runtime CSS-in-JS** (e.g. **vanilla-extract**, **Linaria**, **Panda CSS**, and Atlassian's **`@compiled`**, whose API mirrors styled-components/Emotion): you author styles the same way, but a build step extracts them to a static `.css` file. styled-components itself is now in maintenance mode, which the community reads as further confirmation of the shift toward compile-time approaches and Tailwind. For new projects, prefer Tailwind or a zero-runtime CSS-in-JS over a runtime one.

**How Tailwind stays small.** Tailwind does not ship its giant utility catalog. The Vite plugin scans your source for class-name *strings* and generates CSS only for the ones it finds. Critically, it matches literal substrings — `clsx('bg-' + color)` won't be seen (the string `bg-red-500` never appears in source). Always write **complete, static** class names; use a lookup map if you must vary them: `const tones = { red: 'bg-red-500', blue: 'bg-blue-500' }`.

**CSS custom properties vs. Sass variables.** `$radius` in Sass is resolved and discarded at build time — it cannot change in the browser. `var(--radius)` is a *live* DOM value: change it (even from JS, even per-element) and everything reading it updates without a rebuild. That's why runtime theming uses CSS variables, not Sass variables.

**`className` vs. `class`.** JSX uses `className` because `class` is a reserved word in JavaScript and JSX compiles to JS (see `03-jsx-in-depth.md`). React 19 *does* now pass through unknown attributes and supports `class` in some host contexts, but `className` remains the idiomatic, lint-clean choice — keep using it.

---

## Common mistakes & pitfalls

**1. Passing a string to `style`.**

```jsx
// WRONG — style is not a string in React
<div style="color: red; padding: 8px" />
```

```jsx
// RIGHT — an object with camelCased keys
<div style={{ color: 'red', padding: 8 }} />
```

React expects an object; a string throws in dev. `padding: 8` auto-appends `px`.

**2. Forgetting `.module.` in the filename.**

```jsx
// WRONG — Button.css is global; `styles` is an empty object, classes are undefined
import styles from './Button.css';
<button className={styles.primary}>…</button> // className={undefined}
```

```jsx
// RIGHT — Modules require the .module.css extension
import styles from './Button.module.css';
<button className={styles.primary}>…</button>
```

Only files named `*.module.css` are treated as CSS Modules by Vite. A normal `.css` import gives you `undefined` lookups.

**3. Building Tailwind class names with string concatenation.**

```jsx
// WRONG — Tailwind's scanner never sees "bg-red-500", so the CSS isn't generated
const color = 'red';
<div className={`bg-${color}-500`} />
```

```jsx
// RIGHT — full static class names, chosen via a map
const BG = { red: 'bg-red-500', green: 'bg-green-500' };
<div className={BG[color]} />
```

**4. Manually joining classes and getting `false`/`undefined` in the string.**

```jsx
// WRONG — produces "btn false" when not active
<button className={'btn ' + (active && 'btn--active')} />
```

```jsx
// RIGHT — clsx drops falsy values
import clsx from 'clsx';
<button className={clsx('btn', active && 'btn--active')} />
```

**5. Reaching for inline styles to do `:hover`.**

```jsx
// WRONG — pseudo-classes don't exist in inline styles
<a style={{ color: 'blue', ':hover': { color: 'red' } }}>link</a> // ignored
```

```css
/* RIGHT — use a class with a real selector */
.link { color: blue; }
.link:hover { color: red; }
```

**6. Leaking a styling prop to the DOM (styled-components).**

```jsx
// WRONG — `primary` becomes an invalid DOM attribute: <button primary="true">
const Btn = styled.button`background: ${(p) => (p.primary ? 'purple' : 'gray')};`;
<Btn primary />
```

```jsx
// RIGHT — transient `$` props are consumed for styling, not forwarded
const Btn = styled.button`background: ${(p) => (p.$primary ? 'purple' : 'gray')};`;
<Btn $primary />
```

**7. Import order accidents with global CSS.** Two global stylesheets defining `.btn` resolve by *load order*, which is bundler-dependent and brittle. Don't rely on it — scope with Modules/Tailwind instead of fighting the cascade.

---

## Best practices

- **Pick one primary approach per project and commit.** Mixing Tailwind, CSS Modules, and styled-components in one codebase fragments the mental model. A small amount of CSS variables for theming is the acceptable exception — it layers under everything.
- **Default to scoped styles** (CSS Modules or Tailwind). Reserve plain global CSS for true resets, fonts, and `:root` tokens.
- **Use inline styles only for runtime-computed values** (positions, dynamic widths, animation frames). Everything static goes in a class.
- **Theme with CSS custom properties**, not build-time variables, so light/dark and runtime theming work without a rebuild.
- **Prefer compile-time over runtime CSS-in-JS** for new projects (Tailwind, vanilla-extract, Panda) — better SSR, smaller bundles, friendlier to React Server Components and the React Compiler.
- **Always write complete, static Tailwind class strings**; vary them through lookup maps so the scanner can see them.
- **Use `clsx`** (tiny, ~0.5 KB) for any conditional class logic instead of manual string building.
- **Let components accept a `className` prop** and merge it last, so callers can extend styling without forking the component.
- **Don't over-`@apply`** in Tailwind — it pulls you back toward unscoped global classes; extract a real React component instead.
- **For accessibility-heavy primitives** (modals, comboboxes, menus), don't hand-roll — lean on Radix/MUI/Chakra/Ant Design, which solve focus trapping and ARIA for you.

---

## Exercises

Work these in a fresh `npm create vite@latest -- --template react` project. Solutions follow.

1. **CSS Modules card.** Create `Alert.module.css` with a `.base` class and three composing variants (`.info`, `.success`, `.danger`) using `composes`. Build `Alert.jsx` taking a `tone` prop and rendering the right class. Render one of each.

2. **clsx toggle.** Build a `Tab` component that takes `label`, `active`, and `disabled` props. Use `clsx` so the rendered class is `tab` plus `tab--active` when active and `tab--disabled` when disabled. Render three tabs with state controlling which is active.

3. **Tailwind responsive button.** Set up the Tailwind Vite plugin. Build a `Button` that is full-width on mobile (`w-full`) but auto-width from the `md` breakpoint (`md:w-auto`), with a `hover:` darken effect and a `disabled:` opacity change.

4. **CSS-variable theming.** Implement a light/dark toggle using a `data-theme` attribute on `<html>` and `var(--bg)`/`var(--fg)` tokens. Persist the choice in `localStorage` and restore it on load.

5. **(Stretch) styled-components themed button.** Wrap an app in `ThemeProvider`, define `{ primary, onPrimary }`, and make a `styled.button` whose background comes from the theme and whose variant comes from a transient `$variant` prop.

### Solutions

**Solution 1 — CSS Modules card**

```css
/* src/Alert.module.css */
.base { padding: 12px 16px; border-radius: 8px; border: 1px solid; }
.info    { composes: base; background: #e8f0fe; border-color: #4285f4; }
.success { composes: base; background: #e6f4ea; border-color: #34a853; }
.danger  { composes: base; background: #fce8e6; border-color: #ea4335; }
```

```jsx
// src/Alert.jsx
import styles from './Alert.module.css';

export default function Alert({ tone = 'info', children }) {
  return <div className={styles[tone]}>{children}</div>;
}

// usage
// <Alert tone="info">Heads up</Alert>
// <Alert tone="success">Saved</Alert>
// <Alert tone="danger">Failed</Alert>
```

**Solution 2 — clsx toggle**

```jsx
// src/Tabs.jsx
import { useState } from 'react';
import clsx from 'clsx';
import './tabs.css';

function Tab({ label, active, disabled, onSelect }) {
  return (
    <button
      disabled={disabled}
      onClick={onSelect}
      className={clsx('tab', { 'tab--active': active, 'tab--disabled': disabled })}
    >
      {label}
    </button>
  );
}

export default function Tabs() {
  const [current, setCurrent] = useState('home');
  const tabs = [
    { id: 'home', label: 'Home' },
    { id: 'profile', label: 'Profile' },
    { id: 'settings', label: 'Settings', disabled: true },
  ];
  return (
    <div>
      {tabs.map((t) => (
        <Tab
          key={t.id}
          label={t.label}
          disabled={t.disabled}
          active={current === t.id}
          onSelect={() => setCurrent(t.id)}
        />
      ))}
    </div>
  );
}
```

```css
/* src/tabs.css */
.tab { padding: 8px 16px; border: none; background: #eee; cursor: pointer; }
.tab--active { background: rebeccapurple; color: white; }
.tab--disabled { opacity: 0.5; cursor: not-allowed; }
```

**Solution 3 — Tailwind responsive button**

```jsx
// src/Button.jsx
export default function Button({ children, ...rest }) {
  return (
    <button
      className="w-full rounded bg-blue-600 px-4 py-2 text-white transition
                 hover:bg-blue-700 disabled:opacity-50 md:w-auto"
      {...rest}
    >
      {children}
    </button>
  );
}
// Requires @tailwindcss/vite in vite.config.js and `@import "tailwindcss";` in index.css.
```

**Solution 4 — CSS-variable theming**

```css
/* src/theme.css */
:root { --bg: #fff; --fg: #1a1a1a; }
:root[data-theme='dark'] { --bg: #121212; --fg: #f0f0f0; }
body { background: var(--bg); color: var(--fg); transition: background 200ms; }
```

```jsx
// src/ThemeToggle.jsx
import { useEffect, useState } from 'react';
import './theme.css';

export default function ThemeToggle() {
  const [theme, setTheme] = useState(() => localStorage.getItem('theme') ?? 'light');

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
  }, [theme]);

  return (
    <button onClick={() => setTheme((t) => (t === 'light' ? 'dark' : 'light'))}>
      {theme === 'light' ? 'Dark' : 'Light'} mode
    </button>
  );
}
```

**Solution 5 — styled-components themed button**

```jsx
// src/App.jsx
import styled, { ThemeProvider } from 'styled-components';

const Button = styled.button`
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  cursor: pointer;
  color: ${(p) => p.theme.onPrimary};
  background: ${(p) => (p.$variant === 'ghost' ? 'transparent' : p.theme.primary)};
`;

const theme = { primary: 'rebeccapurple', onPrimary: 'white' };

export default function App() {
  return (
    <ThemeProvider theme={theme}>
      <Button>Primary</Button>{' '}
      <Button $variant="ghost" style={{ color: theme.primary }}>Ghost</Button>
    </ThemeProvider>
  );
}
```

---

## Component libraries (overview)

When you don't want to build buttons, modals, and date pickers from scratch, reach for a library:

- **MUI (Material UI)** — huge, mature, implements Google's Material Design. Comprehensive components and a powerful `sx` prop + theme system (built on Emotion by default). Great when you want batteries-included and Material aesthetics; can feel heavy and opinionated to restyle.
- **Chakra UI** — accessible, composable components with a clean style-props API (`<Box p={4} bg="gray.100">`) and first-class theming. Excellent developer experience; lighter conceptual footprint than MUI.
- **Ant Design (antd)** — dense, enterprise/dashboard-oriented component set (tables, forms, date pickers) with a distinct visual language. Popular for admin tools; large bundle and a strong default look that takes effort to fully rebrand.
- **shadcn/ui** — *not* an installed dependency. You run a CLI that **copies component source into your repo**, built on **Radix UI primitives** (unstyled, accessible behavior) + **Tailwind** for styling. You own and edit the code. Maximum control and no version-lock to a design system, at the cost of maintaining the copied components yourself. Increasingly the default for Tailwind-first teams.

The split: MUI/Chakra/antd are **dependencies you theme**; shadcn/ui is **source you own**. Radix (under shadcn) is the behavior-only layer you can also use directly with any styling approach.

### Choosing an approach

| Situation | Recommended approach |
|---|---|
| Tiny app / prototype / learning | Plain CSS or CSS Modules |
| Medium app, custom design, a team | **CSS Modules** (+ Sass if you want mixins) or **Tailwind** |
| Design-system-driven product, want speed | **Tailwind + shadcn/ui** (own the components) |
| Enterprise dashboard, lots of tables/forms | **Ant Design** or **MUI** |
| Want accessible primitives, custom look | **Radix** (+ Tailwind or CSS Modules) |
| Legacy CSS-in-JS codebase | styled-components/Emotion — but plan migration to zero-runtime |
| New SSR / React Server Components app | **Tailwind** or **zero-runtime CSS-in-JS** (avoid runtime CSS-in-JS) |

When unsure on a new greenfield project in 2025+: **Tailwind for styling, shadcn/ui + Radix for complex accessible components, CSS variables for theming.** It's performant, SSR-friendly, framework-aligned, and gives you full ownership.

---

## Key takeaways

- CSS is global by default; every React styling approach exists to **scope** styles to components.
- **CSS Modules** rename classes to unique names at build time — real CSS, zero runtime cost, no collisions. `composes` shares rules.
- **Inline `style`** takes a camelCased object and auto-adds `px` to numbers; use it only for runtime-computed values — it can't do `:hover`, media queries, or pseudo-elements.
- **Sass** adds nesting/variables/mixins at build time; combine with Modules. Native CSS nesting + custom properties now cover much of it.
- **Runtime CSS-in-JS** (styled-components/Emotion) is ergonomic but injects styles at render time; the industry is moving to **zero-runtime** (vanilla-extract, Panda, Linaria) and Tailwind for SSR/perf reasons.
- **Tailwind** composes atomic utilities, supports `hover:`/`md:` variants and `@apply`, and emits only the classes it can *see as literal strings* in your source.
- Use **`clsx`** for conditional classes and **CSS custom properties** for live, runtime theming (light/dark via a `data-theme` attribute).
- Component libraries split into **theme-a-dependency** (MUI, Chakra, antd) and **own-the-source** (shadcn/ui on Radix + Tailwind); pick by team size, design needs, and SSR strategy.

---

## Further reading

- React docs — adding styles / `className`: <https://react.dev/learn/adding-interactivity> and the JSX reference at <https://react.dev/reference/react-dom/components/common#applying-css-styles>
- Vite — CSS, CSS Modules, and Sass support: <https://vite.dev/guide/features#css>
- MDN — CSS custom properties (variables): <https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties>
- MDN — `prefers-color-scheme`: <https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme>
- Tailwind CSS — install with Vite: <https://tailwindcss.com/docs/installation/using-vite>
- styled-components docs: <https://styled-components.com/docs>
- Emotion docs: <https://emotion.sh/docs/introduction>
- vanilla-extract (zero-runtime CSS-in-JS): <https://vanilla-extract.style/>
- clsx: <https://github.com/lukeed/clsx>
- Radix UI primitives: <https://www.radix-ui.com/primitives>
- shadcn/ui: <https://ui.shadcn.com/>
- MUI: <https://mui.com/> · Chakra UI: <https://chakra-ui.com/> · Ant Design: <https://ant.design/>

> Next up, styling meets data fetching and routing in later lessons — keep your styling layer decoupled from those concerns so it stays portable. See `08-useeffect-side-effects.md` for the effect rules used in the theme toggle.
