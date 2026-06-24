---
name: upload-to-magic-patterns
description: Upload UI you are building locally into Magic Patterns (magicpatterns.com) and return the editor link. Use when the user wants to push a component/screen they are working on in their codebase into Magic Patterns as a design — for review, stakeholder sign-off, or further iteration. The harness writes the files directly (no AI prompting). Requires the Magic Patterns MCP server.
---

# Upload local UI to Magic Patterns

Use this skill to push UI from the current codebase into a **hosted Magic Patterns design** and hand back the editor link. You create a blank design and **write the files into it directly** — you do not prompt Magic Patterns to generate anything. The destination is a real Magic Patterns design, created through the Magic Patterns MCP server.

Magic Patterns renders **visual prototypes**, not production apps. Local production code will not run there as-is — it must be ported to Magic Patterns' constraints first. The bulk of this skill is doing that port correctly so the upload actually compiles and renders.

**Keep the scope minimal.** Upload only what the user wants to prototype — the specific component or screen and the smallest set of files needed to render it. This is not a tool for mirroring the whole app; a tighter upload compiles more reliably and is faster to iterate on.

## Prerequisites

- The Magic Patterns MCP server must be connected (tools prefixed `magic-patterns`). If the tools are unavailable, tell the user to install/enable the Magic Patterns plugin or MCP server, and stop.
- Magic Patterns MCP usage requires a paid plan. A `Payment Required` / credits error means the user must upgrade at `magicpatterns.com/settings`.

## Workflow

### Step 1 — Pick the minimal scope to prototype

Identify the specific component(s) or screen the user wants to prototype, and the smallest set of files needed to render it. Read the relevant source so you understand the layout, hierarchy, states, and data shape. Do not pull in the whole app — keep the upload tight to what the user actually wants to see.

### Step 2 — Port to Magic Patterns constraints (the important part)

Adapt the code into a self-contained prototype that renders with no backend. Strip everything that needs a real runtime and replace it with static stand-ins:

**Keep:** UI components, layout, styling, visual states, and display logic that affects what the user sees.

**Remove / replace:**

- [ ] Real API calls, `fetch`/`axios`, server actions, tRPC, DB queries → replace with **hardcoded mock data** (inline or a `mockData.ts`).
- [ ] Auth guards, protected routes, session logic → render unconditionally.
- [ ] Environment variables and secrets → inline safe constants.
- [ ] Relative image paths → **absolute URLs** only.
- [ ] Project-specific aliases/imports that won't resolve → local stubs or equivalents.

**Reproduce the source faithfully — don't re-derive it (fidelity first):**

The #1 cause of an ugly upload is rebuilding the UI from scratch in approximate Tailwind. That drifts from the original colors/spacing and bakes in layout bugs the real component doesn't have. Reproduce, don't re-derive — in this order of preference:

- [ ] **Use the real component library — MP installs npm packages.** MP installs whatever is listed in the prototype's `package.json` (real npm install), so if the source is built on a published design system, import the *actual* package instead of approximating it. **Best case — the app's own design system is on npm:** `@magicpatterns/cubed` is published, so import it directly (`import '@magicpatterns/cubed/styles.css'`, components from `@magicpatterns/cubed`, wrap in its `ThemeProvider` with the same props the app uses — e.g. `<ThemeProvider accentColor="indigo" panelBackground="translucent">`). That brings the app's exact brand CSS layer, not just the underlying primitives — the most faithful option. Cubed is a thin wrapper over **Radix Themes (`@radix-ui/themes`)** and re-exports its components nearly 1:1 (`Flex`, `Box`, `Text`, `Heading`, `Button`, `IconButton`, `Select`, `SegmentedControl`, `Tooltip`, …), so if you can't use Cubed, map `@magicpatterns/cubed` → `@radix-ui/themes` (`import '@radix-ui/themes/styles.css'`, `<Theme accentColor="indigo" grayColor="slate">`) — same component names/props, real token system for free. The same applies to shadcn (Radix primitives), MUI, etc.: prefer the real package.
- [ ] **Keep the original styling approach verbatim.** Whatever styling the source uses — Cubed/Radix props (`px`, `gap`, `size`, `variant`, `radius`), inline styles, CSS variables, CSS-in-JS — KEEP it. Copy the real props / `style={{…}}` rather than translating to approximate Tailwind by eye. Do NOT convert a faithful source's styling system into a different one.
- [ ] **Only if the real library can't be installed: bring the tokens + stub the primitives.** Paste the library's CSS-variable definitions into `index.css` so original `var(--…)` references resolve (`indigo-100` ≠ `var(--indigo-3)`), and provide thin local equivalents for the components (Radix primitives are mostly `div`/`button` with inline styles, so this is near-verbatim). This is the fallback, not the default.
- [ ] **Preserve responsive + intrinsic sizing.** Carry over `hidden md:inline-flex`, content-sized controls, `white-space: nowrap`, and wrap/overflow behavior. This stops the layout from clipping or collapsing, and keeps it robust if a later `send_prompt` adds controls to the same row.

Reserve from-scratch Tailwind for genuinely net-new UI that has no source to reproduce.

**How faithful is faithful enough?** A seed only needs to be close enough to ground the design — importing the real library (above) gets you ~90% with little effort and is the right stopping point. Porting the app's actual component source files (e.g. `PromptInputV2`, `DesignSystemDropdown`) near-verbatim can get the last 10%, but those files pull in app-only hooks/contexts/queries/analytics you'd have to stub, which is slow and brittle. Don't chase pixel-perfection for a seed unless the user explicitly asks.

**Conform to the Magic Patterns environment:**

- React + TypeScript. **No `any`** — define prop types/interfaces.
- **Tailwind CSS v3** is available (use it for net-new UI; for reproduced UI, prefer the source's own styling per the fidelity rules above). `index.css` must keep the Tailwind imports in this exact order and never reorder/remove them:
  1. `@import url(...)` font imports (if any)
  2. `@import 'tailwindcss/base'`
  3. `@import 'tailwindcss/components'`
  4. `@import 'tailwindcss/utilities'`
  5. other `@import`s
  6. custom CSS
- Icons: **`lucide-react` only**, imported with full names ending in `Icon` (e.g. `UserIcon`). Swap any other icon library to the closest lucide equivalent.
- Exports: **named exports only** — never `export default` a React component.
- One React component per file; split large files instead of monoliths.
- Routing (if needed): `BrowserRouter` from `react-router-dom`.
- Fonts: `@import url(...)` in `index.css`, never `next/font`.
- Package preferences when applicable: `sonner` (toasts), `recharts` (charts), `@xyflow/react` (node/canvas), Leaflet (maps). Avoid `@react-three/fiber`/`@react-three/drei`.
- The code **must compile** — no unresolved imports, missing symbols, or type errors. If a dependency is unavailable, stub it locally so it still compiles.

The goal is visual fidelity to the local UI. "Don't paste production code verbatim" applies to **runtime** logic — fetch/auth/env/server code — which you strip and stub. It does **not** apply to **styling**: reproduce the source's styling (tokens, inline styles, library structure) as faithfully as the MP environment allows.

### Step 3 — Create a blank design and upload the files directly

Create a blank design and write your ported files into it. **Do not use `send_prompt` or any AI generation** — the harness writes the files itself. Use the Magic Patterns MCP tools in this order:

1. `create_design` with no prompt → "start from scratch": returns `editorId` and the active `artifactId` (a blank scaffold: `App.tsx`, `index.tsx`, `index.css`, `tailwind.config.js`). This returns immediately.
2. `get_design_status(editorId)` → confirm the active `artifactId` and that `isGenerating` is false.
3. `write_artifact_files(artifactId, files)` → write your ported files directly. Include `App.tsx` as the entry component and update `index.css` if you added fonts/tokens.
4. `publish_artifact(artifactId, editorId)` → compiles and activates the artifact so the preview renders.

If you need to preserve the original scaffold as a separate version, call `create_new_artifact(artifactId, name)` before writing and use the returned artifact ID for the writes.

### Step 4 — Hand back the link

Return the `editorUrl` to the user so they can open the design in their browser and keep iterating in Magic Patterns. Tell them they may need to log in. If `publish_artifact` reported compile errors, fix the offending files and publish again before returning the link.

## Common mistakes to avoid

- **Uploading too much** — pulling in the whole app instead of the minimal scope the user wants to prototype. Keep it tight.
- **Prompting instead of writing files** — this skill writes the files directly with `write_artifact_files`; do not use `send_prompt` to generate the UI.
- **Pasting production code unchanged** — live `fetch`/auth/env will fail to compile or render blank. Always port first.
- **Rebuilding the UI from scratch in approximate Tailwind** — the most common cause of an ugly upload. Prefer importing the source's real component library (e.g. Cubed → `@radix-ui/themes`), keep its styling approach, and only stub/approximate as a fallback.
- **Stubbing or approximating a library MP could just install** — MP installs from `package.json`. If the source uses Radix Themes / shadcn / MUI, add the real package and import its theme CSS instead of hand-rolling look-alikes.
- **Dropping the design tokens** (fallback path) — if you can't install the real library, leaving `var(--…)` references undefined (or guessing Tailwind colors for them) makes everything drift. Paste the token definitions into `index.css` first.
- **Reordering or deleting the Tailwind imports** in `index.css` — breaks all styling.
- **Default-exporting the root component** or leaving non-`lucide-react` icons in place.
- **Relative image paths** — use absolute URLs.
- **Returning the link before `publish_artifact` succeeds** — confirm it compiled first.

## Related

- `integrate-magic-patterns-design` — go the other direction: bring a Magic Patterns design into this codebase as production code.
