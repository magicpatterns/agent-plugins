---
name: recreate-as-react
description: Recreate a component, screen, or pasted code from any codebase as a self-contained React + TypeScript + Tailwind prototype (App.tsx / index.tsx / index.css, named exports, one component per file) that renders in the Magic Patterns prototype environment and faithfully reproduces the original's styling, colors, spacing, and fonts. Use when the user asks for a "React recreation", a "React version" of a component, "recreate X as React", or as the porting engine for the upload-to-magic-patterns and prototype skills.
---

# Recreate as React

Turn a component, screen, or pasted snippet into a **self-contained React + TypeScript + Tailwind prototype** — a small set of files that compiles and renders standalone (in the **Magic Patterns prototype environment** or any React sandbox) and reproduces the original's styling as faithfully as possible.

**Method: static.** Read the source and resolve its design tokens by inspection — do NOT run the app, Storybook, or a browser. Fidelity comes from carefully tracing whatever styling system the codebase uses down to concrete values, then re-expressing them in Tailwind (or the source's own props/tokens).

This works for **any codebase**. The styling system varies (Tailwind, CSS/SCSS, CSS Modules, styled-components / Emotion, vanilla-extract, a design-system library like Radix/MUI/Chakra/Mantine/shadcn, or plain inline styles). Step 2 is about identifying which one is in play and following the chain to real pixel/color/font values.

## Where this runs — the Magic Patterns environment

The file set this skill produces is exactly the scaffold Magic Patterns renders (`App.tsx`, `index.tsx`, `index.css`, `tailwind.config.js`), so it can be written straight into a Magic Patterns design. Conform to that environment so the upload compiles and renders:

- **React + TypeScript. No `any`** — define prop types/interfaces.
- **Named exports only** — never `export default` a React component.
- **One React component per file**; split large screens into sub-components rather than a monolith.
- **Tailwind CSS v3** — use it for net-new UI; for reproduced UI, prefer the source's own styling (see Step 2). Keep the `index.css` imports in the exact order below.
- **Icons:** `lucide-react` is available (import full names ending in `Icon`, e.g. `UserIcon`). For fidelity, inlining the **exact `<svg>` markup copied 1:1 from source** is preferred and also compiles in MP — never guess or redraw an icon.
- **Routing** (only if needed): `BrowserRouter` from `react-router-dom`.
- **Fonts:** `@import url(...)` in `index.css` — never `next/font`.
- **Images:** **absolute URLs only** (the original remote URL or the project's served URL) — app-relative paths won't resolve.
- **Package preferences when applicable:** `sonner` (toasts), `recharts` (charts), `@xyflow/react` (node/canvas), Leaflet (maps). Avoid `@react-three/fiber` / `@react-three/drei`.
- **The code must compile** — no unresolved imports, missing symbols, or type errors. If a dependency is unavailable, stub it locally so it still compiles.

## Output contract

Produce a **folder of React files** (default `./<ComponentName>/`). Keep it self-contained so it compiles and renders with no backend.

### Files to emit

```
<ComponentName>/
├── App.tsx              # entry component — renders the recreated UI
├── index.tsx            # standard mount (ReactDOM root → <App/>)
├── index.css            # Tailwind imports (exact order below) + fonts/tokens
├── tailwind.config.js   # any resolved custom colors/spacing/fonts
├── <ComponentName>.tsx  # the recreated component (named export)
├── components/          # sub-components, one per file (as needed)
└── mockData.ts          # hardcoded mock data (as needed)
```

`index.css` must keep these in **this exact order** — never reorder or remove them:

1. `@import url(...)` font imports (if any)
2. `@import 'tailwindcss/base';`
3. `@import 'tailwindcss/components';`
4. `@import 'tailwindcss/utilities';`
5. other `@import`s
6. custom CSS (keyframes, token vars, base `font-family`)

`App.tsx` is the entry point and renders `<ComponentName />`. Hand back the folder path when done.

### Output location when composed for upload

When another skill runs this as its porting step (`upload-to-magic-patterns`, `prototype`), do **not** write into the user's project. Emit the file set into a temporary working directory (`mktemp -d`; refer to it as `$TMP`, e.g. `$TMP/<ComponentName>/`) so the composing skill can read the files, write them into the Magic Patterns artifact, and then delete `$TMP`. Only write to `./<ComponentName>/` in the user's project when this skill is used standalone.

## Workflow

### Step 1 — Locate the source (use subagents to explore)

You MUST delegate the codebase exploration to `explore` subagents via the Task tool — do not read/grep the files yourself. Launch them with the `composer-2.5-fast` model (fast, read-only), fanned out concurrently in a single batch, then do the React assembly yourself with what they return. Split the work, e.g.:

- One subagent finds the target component file(s) and lists its subcomponents / styled wrappers.
- One subagent **follows the component's imports into design-system/workspace packages and `node_modules`** (monorepo sibling packages, `@scope/*` packages) and, for every imported UI symbol (icons, logos, illustrations, bespoke SVG components), opens its real source and returns the **exact `<svg>` markup / asset path** — not a description. For any third-party design-system primitive (segmented control, tabs, switch, select, etc.), also read the **library's component CSS/default rendering** and return the actual default/hover/selected part styles (track color, indicator background + shadow, label colors, per-`size` padding/height/radius).
- One subagent identifies the styling system and locates the theme/token definitions (`tailwind.config`, `:root`/`globals.css` CSS vars, JS theme object, or the library's palette), resolving colors from the **theme source-of-truth file** (the palette/token definition) as concrete hex.
- One subagent finds the font setup, prioritizing **local font files** (`public/fonts/`, `assets/fonts/`, `@font-face` `src: url(...)`, `next/font/local`, `.woff2`/`.woff`/`.ttf`/`.otf`) and then any `@import`/`<link>`/`font-family` declarations, returning the real file paths.
- One subagent finds the **real image assets** (public/static image files, imported image sources) and returns their real paths / URLs, intrinsic width/height, and any hover/state variants.

Instruct each subagent to return the concrete resolved values (hex colors, px/rem, font families, weights, radii, shadows, verbatim SVG markup, asset paths) and exact file/line citations — not summaries — so you can assemble the React without re-reading.

- Named target (e.g. a component name, or "the pricing card", "the settings sidebar") → have a subagent find the component file(s) and everything that affects appearance (subcomponents, styled wrappers, imported CSS/theme files).
- Pasted code → use it directly, but still resolve any tokens, classes, or theme imports it references (delegate that lookup to a subagent).

### Step 2 — Identify the styling system and resolve to concrete values (the fidelity core)

First determine how this component is styled, then trace every style down to a real value (hex color, px/rem size, font family, weight, radius, shadow). **Prefer the source's own styling approach — don't re-derive it.** Rebuilding from scratch in approximate Tailwind is the #1 cause of an ugly recreation. Common systems:

- **Tailwind classes** → keep them verbatim where possible. Resolve custom classes from the project's `tailwind.config` (custom colors, spacing, fonts) into the concrete value and emit as an arbitrary value (`bg-[#...]`) or add the token to the output `tailwind.config.js`.
- **Semantic / aliased utility classes** (`bg-primary`, `text-foreground`, `border-border`, `bg-muted`, common in shadcn/Tailwind theme setups) are **not literal colors** — they resolve to CSS variables (`--primary`, `--foreground`, ...) defined in `globals.css`/`:root` or `tailwind.config`. Follow them to the real value; e.g. `border-border` is usually a light gray, `text-foreground` near-black — not the accent.
- **Plain CSS / SCSS / CSS Modules** → read the stylesheet; translate each rule to Tailwind utilities, or drop it into `index.css` if it's complex (keyframes, pseudo-elements, complex selectors).
- **styled-components / Emotion / vanilla-extract** → read the styled definitions and template literals; translate the resulting CSS to Tailwind (or inline `style={{…}}` when a utility can't express it).
- **Design-system library** (Radix/Radix Themes, MUI, Chakra, Mantine, shadcn, Ant, or an in-house wrapper) → **prefer importing the real package — Magic Patterns installs whatever is in the prototype's `package.json` (a real npm install).** Wrap in its theme provider with the same props the app uses so you get its exact token CSS for free. **Best case — the app's own design system is on npm:** `@magicpatterns/cubed` is published, so import it directly (`import '@magicpatterns/cubed/styles.css'`, components from `@magicpatterns/cubed`, wrap in its `ThemeProvider` with the same props the app uses — e.g. `<ThemeProvider accentColor="indigo" panelBackground="translucent">`). Cubed is a thin wrapper over **Radix Themes (`@radix-ui/themes`)** and re-exports its components nearly 1:1 (`Flex`, `Box`, `Text`, `Heading`, `Button`, `IconButton`, `Select`, `SegmentedControl`, `Tooltip`, …), so if you can't use Cubed, map `@magicpatterns/cubed` → `@radix-ui/themes` (`import '@radix-ui/themes/styles.css'`, `<Theme accentColor="indigo" grayColor="slate">`). The same applies to shadcn (Radix primitives), MUI, etc.: prefer the real package. Otherwise, map its layout/spacing props to Tailwind and resolve its **design tokens** (below) to concrete values.
- **Inline `style={{...}}`** → carry values over directly as inline `style` or the equivalent Tailwind class.

**Never infer a color from convention or memory — read the element.** The most damaging errors come from assuming what a color "should" be instead of reading the actual `className`/`style` on that element. Chat UIs are a classic trap: many apps use a white/bordered user-message bubble, not an accent-colored one — so do not paint the user bubble (or a button, badge, highlight) the accent color just because that's the common pattern. Open the source for each element and use the color it actually declares.

**Follow imports into design-system / workspace packages — they are the source of truth.** A component's appearance is often defined in code it imports, not in the file you're looking at. When a component renders an imported icon, logo, or bespoke SVG, open that component in its package (a monorepo sibling package or `node_modules`) and reproduce its **actual markup**. Never approximate a design-system component from its name — trace it to its definition.

**Icons, SVGs, and the logo MUST match the source 1:1 — never guess.**

- Copy the exact `<svg>` markup — same `viewBox`, `<path>` data, `fill`/`stroke`, `stroke-width`, and dimensions — verbatim from source (app file, icon package, or asset file). Do NOT redraw, simplify, approximate, or invent path data. Do NOT substitute a "closest glyph".
- If an icon/logo is rendered via a component or asset import, follow it into its package/asset source and copy the real SVG it emits, including the exact colors passed in for the relevant state.
- The brand logo/wordmark is a common miss: copy it verbatim from its source rather than eyeballing or reusing a stale/guessed path.
- The ONLY acceptable fallback when a source SVG truly cannot be located: stop and tell the user which icon could not be found rather than shipping a guessed one.

**Design-system primitives: reproduce the library's real rendering, not a guess.** For a third-party design-system component, the appearance (backgrounds, borders, shadows, the active/selected indicator, sizing per `size`/`variant`) is defined by **the library's own CSS and variant defaults** — not by the app usage or the prop names. Resolve the real look by importing the real package, reading the library's component CSS/source in `node_modules`, or using accurate knowledge of that library's documented default rendering, then reproduce it. **Do NOT accent-fill a selected/active segment, tab, toggle, or menu item by default** — many design systems render the active state as a neutral raised surface (white/panel background + subtle shadow, dark label text) on a muted track, not the brand accent.

**Resolving design tokens (CSS variables / theme scales).** When you see `var(--...)`, a theme object, or a scale reference, follow it to the source-of-truth value — do NOT guess a Tailwind named color that "looks close":

- Find where the token is defined (a `:root`/theme CSS file, a JS theme object, or the library's published palette) and use that exact value.
- Apply it as a Tailwind arbitrary value (`bg-[#...]`, `text-[#...]`, `border-[#...]`), add it to `tailwind.config.js`, or define it as a CSS var in `index.css` — but never leave a bare `var(--…)` undefined.
- For libraries with a numbered color scale (e.g. Radix's 1–12): **1–2** app/subtle backgrounds, **3–5** component backgrounds (normal/hover/active), **6–8** borders/separators/focus ring, **9–10** solid fills (9 = the accent solid, e.g. a primary button), **11** secondary/low-contrast text, **12** high-contrast text. Resolve the actual hex from the scale in use.

**Map layout/spacing/size props to Tailwind** regardless of library:

- `direction="column"` → `flex-col`; `align="center"` → `items-center`; `justify="between"` → `justify-between`
- `gap`, `px`, `py`, `p`, `m*` spacing steps → the matching Tailwind step (verify the scale — some libraries' step N ≠ Tailwind's step N)
- Preserve `size`/`variant`/`weight` semantics by translating to the resolved font size, weight, and fill.

**Border radius — resolve the exact value, don't approximate.** `rounded-md` (6px) rarely matches the source. Trace the real radius to a px/rem value and emit it as an arbitrary utility (`rounded-[10px]`). Named library radii (`radius="large"`, `--radius-3`) are tokens, not pixels — look up the resolved px (many libraries multiply by a scaling factor). A pill uses `rounded-full`; match per-corner radii (`rounded-t-[10px]`); keep `overflow-hidden` where a rounded parent clips children; inner radius usually = outer radius minus padding/border.

**Preserve intent.** Keep responsive prefixes (`md:`, `lg:`), visibility/sizing (`hidden md:inline-flex`, `w-fit`, `whitespace-nowrap`), and hover/active/focus/disabled visual states.

**Prioritize mobile responsiveness — build mobile-first.** The recreation must render and read well on small screens, not just desktop. Carry over every responsive prefix and breakpoint the source declares (`sm:`/`md:`/`lg:`/`xl:`), and where the source has no explicit mobile handling, add sensible responsive behavior yourself so nothing overflows at narrow widths: stack multi-column layouts vertically, let flex rows wrap, collapse or make navigation scrollable, use fluid widths (`w-full`, `max-w-*`, percentage/`min()`/`clamp()`) instead of fixed pixel widths, keep touch targets comfortably tappable, and allow horizontal scroll only where genuinely unavoidable (e.g. wide tables). Default to a phone-width viewport as the baseline and layer desktop enhancements on top with breakpoints.

### Step 3 — Strip runtime, keep the UI

Make the prototype self-contained so it renders with no backend. This applies to **runtime logic only** — never to styling:

- [ ] Real API calls, `fetch`/`axios`, server actions, tRPC, DB queries → replace with **hardcoded mock data** (inline or `mockData.ts`).
- [ ] Auth guards, protected routes, session logic → render unconditionally.
- [ ] Environment variables and secrets → inline safe constants.
- [ ] Project-specific aliases/imports that won't resolve → local stubs or equivalents.
- [ ] Stub interactivity: controlled inputs with local `useState`, handlers that update local state; no network.

### Step 4 — Fonts

Detect the font source and reproduce it via `@import url(...)` at the **top** of `index.css`. Prefer the codebase's real typeface:

1. **Local font files** → if the project ships fonts (`public/fonts/`, `@font-face src: url(...)`, `next/font/local`, `.woff2`/`.woff`/`.ttf`/`.otf`), add a `@font-face` block in `index.css` pointing at an absolute path or the project's served URL, and note the source in a comment.
2. **Google Fonts** → add the matching `@import url('https://fonts.googleapis.com/css2?family=...&display=swap');`.
3. **Third-party (Typekit/Adobe)** → mirror the `@import url(...)`.
4. **Closest web substitute** → last resort; note the substitution.

Apply the correct `font-family`, weights, sizes, and line-heights on the root/body and headings (via `index.css` base layer or `tailwind.config.js` `fontFamily`).

### Step 5 — Plan the files, then parallelize the writes with subagents

You (the main model) do the **thinking**: from the concrete values resolved in Step 2, produce a high-level spec of the output — the file list, which component lives in which file, the props/mock-data shapes, and the resolved tokens (colors, spacing, radii, fonts, verbatim SVG markup) each file needs. Then **delegate the actual file writing to `composer-2.5-fast` subagents (via the Task tool), fanned out concurrently in a single batch** so independent files are written in parallel.

- Give each subagent a self-contained brief: the exact file path to write, the full resolved values it needs (concrete hex, px/rem, font families, verbatim `<svg>` markup, mock-data shape), and the Output-contract rules (named exports, one component per file, Tailwind arbitrary values, no `any`, `index.css` import order). Subagents must not re-derive styling or re-read the source — hand them the resolved values so they only assemble.
- Group by independence: e.g. one subagent per sub-component file, one for `index.css` + `tailwind.config.js`, one for `mockData.ts`. Keep files that must agree on a shared interface (e.g. a component and its props type) in the same brief.
- After the subagents return, you assemble/verify: confirm imports line up across files, the `index.css` import order is intact, and everything compiles. Fix any seams yourself.

Skeletons for the entry files (include these in the relevant subagent brief):

```tsx
// index.tsx
import React from 'react';
import { createRoot } from 'react-dom/client';
import { App } from './App';
import './index.css';

const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

```tsx
// App.tsx
import React from 'react';
import { ComponentName } from './ComponentName';

export function App() {
  return <ComponentName />;
}
```

```css
/* index.css */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
@import 'tailwindcss/base';
@import 'tailwindcss/components';
@import 'tailwindcss/utilities';

:root {
  --accent: #3e63dd;
}
body {
  font-family: 'Inter', ui-sans-serif, system-ui, sans-serif;
}
```

- Raster images (PNG/JPG/WebP): reference the real asset in `src` at its intrinsic width/height, preserving state variants. Use an **absolute URL** (the original remote URL, or the project's served URL) so it resolves anywhere — do not use app-relative paths that won't resolve. Emoji/placeholder only as a last resort when no asset exists.
- Icons/logos/bespoke SVGs: inline the exact `<svg>` copied 1:1 from source.
- If the source uses many custom colors, define them once (CSS vars in `index.css` or `tailwind.config.js` `theme.extend.colors`) and reference them.

## Fidelity checklist

- [ ] Output is the React file set (`App.tsx`, `index.tsx`, `index.css`, `tailwind.config.js`, component files) and compiles — no unresolved imports or type errors, no `any`
- [ ] Named exports only; one component per file; `index.css` Tailwind imports in the exact order
- [ ] Colors match the resolved token/theme values (not guessed named colors), resolved from the theme file
- [ ] Every icon/SVG/logo matches source 1:1 (same viewBox/path/fill/stroke, copied verbatim, nothing guessed)
- [ ] Real image assets used at correct dimensions via absolute URLs — no emoji/placeholder stand-ins, no app-relative paths
- [ ] Design-system primitives reproduce the library's real rendering (track/indicator/shadow/label); selected state matches the library (not an invented accent fill)
- [ ] Spacing, font sizes, weights, and line-heights match
- [ ] Borders, radii, and shadows match (radius resolved to exact px/rem, per-corner and `rounded-full` preserved, `overflow-hidden` kept where the parent clips children)
- [ ] Layout and responsive behavior preserved (flex/grid, breakpoints, intrinsic sizing)
- [ ] Mobile-first: renders and reads well at phone widths — no overflow, columns stack, nav collapses/scrolls, fluid widths instead of fixed pixels, touch-friendly targets
- [ ] Hover/active/focus/disabled visual states preserved
- [ ] Fonts load via `@import url()` in `index.css` and apply (no serif fallback)
- [ ] Runtime stripped: mock data instead of fetch/auth/env; nothing needs a backend

## Common mistakes

- Rebuilding the UI from scratch in approximate Tailwind instead of reproducing the source's real values/library — the most common cause of an ugly recreation.
- Stubbing or approximating a library Magic Patterns could just install — MP installs from `package.json`; if the source uses Cubed / Radix Themes / shadcn / MUI, add the real package and import its theme CSS instead of hand-rolling look-alikes.
- Inferring an element's color from UI convention/memory instead of reading its source (e.g. accent-coloring a user chat bubble the code declares as `bg-white border-border text-foreground`).
- Treating semantic classes (`bg-primary`, `text-foreground`, `border-border`) as literal colors instead of resolving their theme CSS variables; leaving bare `var(--…)` undefined.
- Guessing a Tailwind named color (`bg-indigo-600`) instead of resolving the actual token/theme value.
- Guessing/redrawing an SVG or brand logo instead of copying it 1:1 (a distinctive icon comes out the wrong shape).
- Default-exporting the root component.
- Reordering or deleting the Tailwind imports in `index.css` — breaks all styling.
- Emoji or a generic placeholder instead of the real image asset; app-relative image paths that won't resolve (use absolute URLs).
- Stopping at the app file instead of following imports into the design-system package/`node_modules`.
- Accent-filling a selected segment/tab/toggle when the library renders the active state as a neutral raised indicator (white + shadow, dark text).
- Approximating border radius with `rounded-md`/`rounded-lg` instead of resolving the exact px value; dropping per-corner radii, `rounded-full`, or the `overflow-hidden` that clips a rounded parent.
- Pasting production runtime code (fetch/auth/env) verbatim so it fails to compile or renders blank — strip and stub it.
- Building desktop-only — dropping the source's responsive behavior or hardcoding fixed pixel widths so the recreation overflows or breaks on mobile. Build mobile-first and keep it usable at small viewport widths.

## Example

User: "Take the `<PricingCard>` from my codebase and give me a React version."

1. Delegate exploration: subagents find `PricingCard` and subcomponents, follow imports for icons/logo/tokens, resolve the theme palette and fonts, and return concrete values + verbatim SVG.
2. Identify the styling system; resolve every color/spacing/font token to a concrete value (arbitrary Tailwind values, `tailwind.config.js` tokens, or inline styles). Inline the icons/logo SVG 1:1.
3. Strip any runtime (fetch/auth) → mock data; add the app's font `@import` to `index.css`.
4. Plan the file list + resolved values, then fan out `composer-2.5-fast` subagents to write `App.tsx`, `index.tsx`, `index.css`, `tailwind.config.js`, `PricingCard.tsx` (named export) in parallel from that spec. Verify the seams compile and hand back `./PricingCard/`.

## Related

- `upload-to-magic-patterns` — runs this skill as its porting step, then writes the file set into a hosted Magic Patterns design.
- `prototype` — seeds a Magic Patterns design with this skill, then prompts Magic Patterns to design a new idea on top.
