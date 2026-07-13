---
name: recreate-as-raw-html
description: Recreate a component, screen, or pasted code from any codebase as a single self-contained raw HTML file (Tailwind Play CDN + web fonts) that faithfully reproduces its styling, colors, spacing, and fonts. Use when the user asks for a "raw HTML recreation", an "HTML version" of a component, "make an HTML copy of X", or references a component/screen by name (e.g. "Take the <ComponentName> and give me raw HTML").
---

# Recreate as raw HTML

Turn a component, screen, or pasted snippet into **one self-contained `.html` file** that opens standalone in a browser and reproduces the original's styling as faithfully as possible.

**Method: static.** Read the source and resolve its design tokens by inspection — do NOT run the app, Storybook, or a browser. Fidelity comes from carefully tracing whatever styling system the codebase uses down to concrete values, then re-expressing them in Tailwind.

This works for **any codebase**. The styling system varies (Tailwind, CSS/SCSS, CSS Modules, styled-components / Emotion, vanilla-extract, a design-system library like Radix/MUI/Chakra/Mantine/shadcn, or plain inline styles). Step 2 is about identifying which one is in play and following the chain to real pixel/color/font values.

## Output contract

Produce a single HTML document:

- Tailwind via the Play CDN: `<script src="https://cdn.tailwindcss.com"></script>`
- Font `<link>`s / `@import`s in `<head>` for the fonts the source uses
- Static markup only — no React/Vue/framework, no build step. Add JS only for trivial visual behavior if the user asks; otherwise omit it.
- Save as `./<ComponentName>.html` (or next to where the user is working) and hand back the path.

## Workflow

### Step 1 — Locate the source (use subagents to explore)

You MUST delegate the codebase exploration to `explore` subagents via the Task tool — do not read/grep the files yourself. Launch them with the `composer-2.5-fast` model (fast, read-only), fanned out concurrently in a single batch, then do the HTML assembly yourself with what they return. Split the work, e.g.:

- One subagent finds the target component file(s) and lists its subcomponents / styled wrappers.
- One subagent **follows the component's imports into design-system/workspace packages and `node_modules`** (monorepo sibling packages, `@scope/*` packages) and, for every imported UI symbol (icons, logos, illustrations, bespoke SVG components), opens its real source and returns the **exact `<svg>` markup / asset path** — not a description. For any third-party design-system primitive (segmented control, tabs, switch, select, etc.), also read the **library's component CSS/default rendering** and return the actual default/hover/selected part styles (track color, indicator background + shadow, label colors, per-`size` padding/height/radius).
- One subagent identifies the styling system and locates the theme/token definitions (`tailwind.config`, `:root`/`globals.css` CSS vars, JS theme object, or the library's palette), resolving colors from the **theme source-of-truth file** (the palette/token definition) as concrete hex.
- One subagent finds the font setup, prioritizing **local font files** (`public/fonts/`, `assets/fonts/`, `@font-face` `src: url(...)`, `next/font/local`, `.woff2`/`.woff`/`.ttf`/`.otf`) and then any `@import`/`<link>`/`font-family` declarations, returning the real file paths.
- One subagent finds the **real image assets** (public/static image files, imported image sources) and returns their real paths, intrinsic width/height, and any hover/state variants.

Instruct each subagent to return the concrete resolved values (hex colors, px/rem, font families, weights, radii, shadows, verbatim SVG markup, asset paths) and exact file/line citations — not summaries — so you can assemble the HTML without re-reading.

- Named target (e.g. a component name, or "the pricing card", "the settings sidebar") → have a subagent find the component file(s) and everything that affects appearance (subcomponents, styled wrappers, imported CSS/theme files).
- Pasted code → use it directly, but still resolve any tokens, classes, or theme imports it references (delegate that lookup to a subagent).

### Step 2 — Identify the styling system and resolve to concrete values (the fidelity core)

First determine how this component is styled, then trace every style down to a real value (hex color, px/rem size, font family, weight, radius, shadow). Common systems:

- **Tailwind classes** → keep them verbatim where the CDN supports them. Resolve custom classes from the project's `tailwind.config` (custom colors, spacing, fonts) into the concrete value and emit as an arbitrary value (`bg-[#...]`).
- **Semantic / aliased utility classes** (`bg-primary`, `text-foreground`, `border-border`, `bg-muted`, `text-accent`, common in shadcn/Tailwind theme setups) are **not literal colors** — they resolve to CSS variables (`--primary`, `--foreground`, ...) defined in `globals.css`/`:root` or `tailwind.config`. Follow them to the real value; e.g. `border-border` is usually a light gray, `text-foreground` near-black — not the accent.
- **Plain CSS / SCSS / CSS Modules** → read the stylesheet; translate each rule to Tailwind utilities, or drop it into an inline `<style>` block if it's complex (keyframes, pseudo-elements, complex selectors).
- **styled-components / Emotion / vanilla-extract** → read the styled definitions and template literals; translate the resulting CSS to Tailwind/inline styles.
- **Design-system library** (Radix/Radix Themes, MUI, Chakra, Mantine, shadcn, Ant, or an in-house wrapper) → the library's props and CSS-variable tokens hide real values. Map its layout/spacing props to Tailwind, and resolve its **design tokens** (see below) to concrete values.
- **Inline `style={{...}}`** → carry values over directly as inline `style` or the equivalent Tailwind class.

**Never infer a color from convention or memory — read the element.** The most damaging errors come from assuming what a color "should" be instead of reading the actual `className`/`style` on that element. Chat UIs are a classic trap: many apps use a white/bordered user-message bubble, not an accent-colored one — so do not paint the user bubble the accent color (or a primary button, badge, or highlight) just because that's the common pattern. Open the source for each element and use the color it actually declares.

**Follow imports into design-system / workspace packages — they are the source of truth.** A component's appearance is often defined in code it imports, not in the file you're looking at. When a component renders an imported icon, logo, or bespoke SVG component, open that component in its package (a monorepo sibling package or `node_modules`) and reproduce its **actual markup**. Never approximate a design-system component from its name — trace it to its definition and copy what it renders.

**Icons, SVGs, and the logo MUST match the source 1:1 — never guess.**

- Copy the exact `<svg>` markup — same `viewBox`, `<path>` data, `fill`/`stroke`, `stroke-width`, and dimensions — verbatim from the source (app file, icon package, or asset file). Do NOT redraw, simplify, approximate, or invent path data. Do NOT substitute a "closest glyph".
- If an icon/logo is rendered via a component or an asset import, follow it into its package/asset source and copy the real SVG it emits, including the exact colors passed in for the relevant state.
- The brand logo/wordmark is a common miss: copy it verbatim from its source rather than eyeballing or reusing a stale/guessed path.
- The ONLY acceptable fallback when a source SVG truly cannot be located: stop and tell the user which icon could not be found rather than shipping a guessed one. (A distinctive icon guessed instead of copied comes out the wrong shape entirely.)

**Design-system primitives: reproduce the library's real rendering, not a guess.** For a third-party design-system component, the appearance (backgrounds, borders, shadows, the active/selected indicator, sizing per `size`/`variant`) is defined by **the library's own CSS and variant defaults** — not by the app usage or the prop names. You cannot infer it from the app file, which may only be a bare `<SomeControl size="1">` usage.

- Resolve the real look by reading the library's component CSS/source in `node_modules` (or using accurate knowledge of that library's documented default rendering), then translate that to HTML/Tailwind.
- Library part-name selectors in `className` overrides (generic form `[&_.<library-part-name>]`) reveal the component's internal anatomy — use them to know which parts exist and which one is being styled, and as a pointer to the exact component to read.
- **Do NOT accent-fill a selected/active segment, tab, toggle, or menu item by default.** Many design systems render the active state as a neutral raised surface (e.g. white/panel background + a subtle shadow, with normal dark label text) on a muted track — not the brand accent. Reproduce the library's actual selected-state styling; if unsure, read the library CSS rather than guessing. (For example, a segmented control's selected item is often a white raised indicator with a shadow, not a solid accent fill.)

**Resolving design tokens (CSS variables / theme scales).** When you see `var(--...)`, a theme object, or a scale reference, follow it to the source-of-truth value — do NOT guess a Tailwind named color that "looks close":

- Find where the token is defined (a `:root`/theme CSS file, a JS theme object, or the library's published palette) and use that exact value.
- Apply it as a Tailwind arbitrary value (`bg-[#...]`, `text-[#...]`, `border-[#...]`) or, only when no utility fits, an inline `style`.
- For libraries that use a numbered color scale (e.g. Radix's 1–12), the step conveys role: **1–2** app/subtle backgrounds, **3–5** component backgrounds (normal/hover/active), **6–8** borders/separators/focus ring, **9–10** solid fills (9 = the accent solid, e.g. a primary button), **11** secondary/low-contrast text, **12** high-contrast text. Resolve the actual hex from the scale in use (the theme's accent + gray).

**Map layout/spacing/size props to Tailwind** regardless of library:

- `direction="column"` → `flex-col`; `align="center"` → `items-center`; `justify="between"` → `justify-between`
- `gap`, `px`, `py`, `p`, `m*` spacing steps → the matching Tailwind step (verify the scale — some libraries' step N ≠ Tailwind's step N)
- Preserve `size`/`variant`/`weight` semantics by translating to the resolved font size, weight, and fill.

**Border radius — resolve the exact value, don't approximate.** Radius is a common failure point: `rounded-md` (6px) rarely matches the source. Trace the real radius to a px/rem value and emit it as an arbitrary utility (`rounded-[10px]`) rather than the nearest named step.

- Named library radii (`radius="large"`, `size="2"`, `--radius-3`, `borderRadius: 'md'`) are **tokens, not pixels** — look up the resolved px in the theme/scale; many libraries also multiply the base radius by a scaling factor, so verify the computed value.
- A "pill"/fully-rounded element uses a huge radius (e.g. `border-radius: 9999px`) → `rounded-full`, not `rounded-lg`.
- Match **per-corner** radii when the source only rounds some corners (`rounded-t-[10px]`, `rounded-l-lg`) and radius that changes responsively.
- A parent with radius plus `overflow-hidden` clips children — keep the `overflow-hidden` or inner corners will bleed past the rounded parent.
- When elements are nested, inner radius usually = outer radius minus the padding/border; preserve that difference instead of reusing the same value.

**Preserve intent.** Keep responsive prefixes (`md:`, `lg:`), visibility/sizing (`hidden md:inline-flex`, `w-fit`, `whitespace-nowrap`), and hover/active/focus/disabled visual states.

### Step 3 — Fonts

Detect the font source and reproduce it in `<head>`. **Always prefer the codebase's own local font files over a Google Fonts (or other web) substitute** — a substitute only approximates the real typeface. Order of preference:

1. **Local font files (preferred whenever they exist)** → check the project for bundled fonts (`public/fonts/`, `assets/fonts/`, `src/fonts/`, `@font-face` `src: url(...)` rules, `next/font/local` declarations, `.woff2`/`.woff`/`.ttf`/`.otf` files). If found, reference them with a `@font-face` block pointing at the file. Since the output must open standalone, resolve the path so the file loads: use an absolute path to the local file (e.g. `file:///.../public/fonts/Foo.woff2`) or the project's served URL, and only copy the files next to the HTML if the user asks. Note in a comment where the fonts came from.
2. **Google Fonts** → only if the font is a Google font and no local file exists: add the matching `<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...">`.
3. **Third-party (Typekit/Adobe, etc.)** `@import url(...)` → mirror the same `@import` or `<link>` when there's no local copy.
4. **Closest web substitute** → last resort when the real font isn't available locally or via a known service; pick the nearest match and note the substitution.
5. **System/default UI font** → for a system stack, set an explicit `font-family` on the root so it doesn't fall back to Times.

Put font `@import url()` first (above other CSS). Apply the correct `font-family`, weights, sizes, and line-heights on the root and headings.

### Step 4 — Assemble the single file

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...&display=swap" />
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
      /* Optional: resolved token vars, keyframes, base font-family */
      :root { --accent: #3e63dd; }
      body { font-family: "Inter", ui-sans-serif, system-ui, sans-serif; }
    </style>
  </head>
  <body>
    <!-- recreated markup -->
  </body>
</html>
```

- Icons/SVGs/logo: paste the **exact `<svg>` markup copied 1:1 from source** (see Step 2) — never a guessed or closest-glyph substitute. Inline SVG is already self-contained.
- Raster images (PNG/JPG/WebP/GIF): **reference the real asset in `src`** rather than inlining it, at its intrinsic width/height, preserving state variants (e.g. hover swaps). Point `src` at the real image so it resolves when the HTML is opened from its saved location — prefer a **path relative to the HTML file** (e.g. `src="public/img/foo.png"` when the HTML sits at the project root), or the **original remote URL** if the source used one. Emoji/placeholder only as an absolute last resort when no asset exists.
- If the source uses many custom colors, define them once as CSS vars in the inline `<style>` and reference them, to keep markup readable.

## Fidelity checklist

- [ ] Colors match the resolved token/theme values (not guessed named colors), resolved from the theme file
- [ ] Every icon/SVG matches source 1:1 (same viewBox/path/fill/stroke, copied verbatim, nothing guessed)
- [ ] Brand logo/wordmark copied verbatim from its source
- [ ] Real image assets used at correct dimensions (SVG inline; raster referenced by a path that resolves from the HTML's location, or a remote URL) — no emoji/placeholder stand-ins
- [ ] Design-system primitives reproduce the library's real rendering (track/indicator/shadow/label); selected state matches the library (not an invented accent fill)
- [ ] Spacing, font sizes, weights, and line-heights match
- [ ] Borders, radii, and shadows match (radius resolved to the exact px/rem, per-corner and `rounded-full` preserved, `overflow-hidden` kept where the parent clips children)
- [ ] Layout and responsive behavior preserved (flex/grid, breakpoints, intrinsic sizing)
- [ ] Hover/active/focus/disabled visual states preserved
- [ ] Fonts load and apply (no Times fallback)
- [ ] No framework/import artifacts, no editor-only props; file opens standalone in a browser

## Common mistakes

- Inferring an element's color from UI convention/memory instead of reading its source — e.g. painting a user chat bubble the accent color when the code says `bg-white border-border text-foreground`.
- Treating semantic classes (`bg-primary`, `text-foreground`, `border-border`) as literal colors instead of resolving their theme CSS variables.
- Guessing a Tailwind named color (`bg-indigo-600`) instead of resolving the actual token/theme value.
- Guessing/redrawing an SVG instead of copying it 1:1 (a distinctive icon comes out the wrong shape).
- Substituting a closest-glyph icon when the real SVG exists in source.
- Emoji or a generic placeholder instead of the real image asset.
- Referencing an image by a path that won't resolve from where the HTML is saved (e.g. an app-served `/img/...` path or a stale relative path) — make the `src` relative to the HTML file's location, or use the remote URL.
- Stopping at the app file instead of following imports into the design-system package/`node_modules`.
- Accent-filling a selected segment/tab/toggle when the library renders the active state as a neutral raised indicator (white + shadow, dark text).
- Inferring a design-system component's look from the app usage/prop names instead of the library's CSS/variant defaults.
- Eyeballing colors instead of reading the theme palette file.
- Substituting a Google/web font when the codebase ships the real font locally — use the local file via `@font-face` instead.
- Dropping fonts, so the page falls back to a serif default.
- Leaving library layout props (`gap`, `px`, `direction`) unconverted — they do nothing in raw HTML.
- Assuming a library's spacing step equals Tailwind's step without checking the scale.
- Approximating border radius with `rounded-md`/`rounded-lg` instead of resolving the exact px value; dropping per-corner radii, `rounded-full`, or the `overflow-hidden` that clips a rounded parent.
- Omitting hover/active/disabled states that the original clearly has.
- Splitting into multiple files — the output must be a single self-contained `.html`.

## Example

User: "Take the `<ComponentName>` from my codebase and give me a raw HTML recreation."

1. Search for the named component, read it and its subcomponents.
2. Identify the styling system; convert framework markup to plain HTML, map layout props to Tailwind, and resolve every color/spacing/font token to a concrete value (arbitrary Tailwind values or inline styles).
3. Add the app's font `<link>` and set the root `font-family`.
4. Emit `<ComponentName>.html` with the Tailwind CDN and hand back the path.
