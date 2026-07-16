---
name: inspiration
description: Explore several bold, meaningfully-different UI design directions for a screen or component in Magic Patterns (magicpatterns.com). First recreates the current UI as a faithful baseline, then creates a shareable magicpatterns.com/inspiration/<id> document and streams in four self-contained HTML concepts tuned to the kind of exploration wanted (entrypoint/discoverability, information architecture/navigation, or visual display). The hosted comparison link is the deliverable. Use when the user wants design inspiration, alternative directions, layout options, variations, or a "show me a few approaches" exploration before committing to an implementation.
---

# Inspiration

Explore **four parallel design directions** for a target screen or component. This skill recreates a faithful **baseline** of the current UI, **creates a placeholder `magicpatterns.com/inspiration/<id>` document up front**, then generates **four concepts** and **streams each one directly into the document** via subagents. The hosted comparison link is the deliverable, and publishing is required, not optional.

Concepts are pushed straight into the document through the `inspiration_add_variant` MCP tool as each subagent finishes — **no concept HTML files are written to disk, and the concept HTML never round-trips through the main agent's context.** This is what keeps the flow fast and avoids the timid "publish a placeholder concept and hope to fix it later" failure mode.

The goal is **bold, grounded divergence**. Pin the product down ONCE using its real content and its real design tokens (colors, fonts, spacing, SVGs), then make the four concepts **meaningfully different from each other**: different layout philosophies, arrangements, or product framings, not four shades of the same idea. Every concept inherits the baseline's tokens so it still looks like the same product; the design *direction* is the only axis that changes.

## Prerequisites

- The Magic Patterns MCP server must be connected (tools prefixed `magic-patterns`). Publishing uses two tools: `create_inspiration_document` (creates the placeholder document + returns the link and one id per concept) and `inspiration_add_variant` (fills a single concept in with its HTML). Iterating on an existing document (see "Iterating on an existing document" below) uses `get_inspiration_document` (loads the current concepts + baseline), `inspiration_clear_variants` (resets all concepts for a full replace), and `inspiration_update_variant` (revises one concept in place). If the tools are unavailable, tell the user to install/enable the Magic Patterns plugin or MCP server.
- Publishing requires a Magic Patterns account; an authentication error from either tool means the user needs to sign in / connect the MCP server.

## When to use

- **This skill (`inspiration`)**: the user wants several directions to compare before committing — layout/IA exploration, alternative visual treatments, "give me options".
- `prototype`: the user already knows what they want and wants to build/iterate one concrete idea.
- `upload-to-magic-patterns`: just mirror local UI into a hosted design, no concepts.
- `integrate-magic-patterns-design`: bring a chosen concept back into the codebase as production code.

## Output contract

The deliverable is a hosted **`magicpatterns.com/inspiration/<id>` link** that renders the four concepts side by side. You obtain the link **up front** by creating a placeholder document (Step 3), then each concept is filled in via `inspiration_add_variant` (Step 4) — the page shows a loading tile per concept until its HTML arrives, then flips to ready once all four are in.

The **only** local file this skill writes is the baseline, into a temporary working directory (create one with `mktemp -d`; refer to it below as `$TMP`):

```
$TMP/<Target>-inspiration/
└── baseline.html            # faithful recreation of the current UI (grounds the concepts; also uploaded as the document's baseline)
```

Concepts are **not** written to disk — each subagent generates its concept HTML in memory and pushes it straight to `inspiration_add_variant`. The baseline is a single self-contained document that opens standalone and follows the **`recreate-as-raw-html` output contract**: Tailwind Play CDN (`<script src="https://cdn.tailwindcss.com"></script>`), real fonts via `<head>` `<link>`/`@font-face`, static markup only (no framework/build step), verbatim 1:1 SVGs, and tokens resolved to concrete values. Every concept must follow the same contract.

## Workflow

### Step 1 — Build the baseline (compose `recreate-as-raw-html`)

First create the temporary working directory (`mktemp -d`, referred to as `$TMP`). Then follow the `recreate-as-raw-html` skill end-to-end to produce **one faithful baseline** of the current UI, saved as `$TMP/<Target>-inspiration/baseline.html` — reusing its fidelity rules and its parallel `explore` subagent fan-out to locate the source, follow imports, resolve tokens/fonts, and copy SVGs verbatim.

**Optimize for speed: parallelize the exploration.** Per `recreate-as-raw-html` Step 1, first check what's already in context and only fan out subagents for the concerns that are actually missing (skip any concern whose concrete resolved values are already present). For whatever you do explore, you MUST delegate the codebase exploration to `explore` subagents via the Task tool — do not read/grep the files yourself. Launch them with the `composer-2.5-fast` model (fast, read-only), **fanned out concurrently in a single batch (one message, multiple Task calls)**, then do the HTML assembly yourself with what they return. Splitting the work across many parallel subagents is the primary speed lever, so favor more concurrent subagents over one doing everything sequentially. Split the work, e.g.:

- One subagent finds the target component file(s) and lists its subcomponents / styled wrappers.
- One subagent **follows the component's imports into design-system/workspace packages and `node_modules`** (monorepo sibling packages, `@scope/*` packages) and, for every imported UI symbol (icons, logos, illustrations, bespoke SVG components), opens its real source and returns the **exact `<svg>` markup / asset path** — not a description. For any third-party design-system primitive (segmented control, tabs, switch, select, etc.), also read the **library's component CSS/default rendering** and return the actual default/hover/selected part styles (track color, indicator background + shadow, label colors, per-`size` padding/height/radius).
- One subagent identifies the styling system and locates the theme/token definitions (`tailwind.config`, `:root`/`globals.css` CSS vars, JS theme object, or the library's palette), resolving colors from the **theme source-of-truth file** (the palette/token definition) as concrete hex.
- One subagent finds the font setup, prioritizing **local font files** (`public/fonts/`, `assets/fonts/`, `@font-face` `src: url(...)`, `next/font/local`, `.woff2`/`.woff`/`.ttf`/`.otf`) and then any `@import`/`<link>`/`font-family` declarations, returning the real file paths.
- One subagent finds the **real image assets** (public/static image files, imported image sources) and returns their real paths, intrinsic width/height, and any hover/state variants.

Instruct each subagent to return the concrete resolved values (hex colors, px/rem, font families, weights, radii, shadows, verbatim SVG markup, asset paths) and exact file/line citations — not summaries — so you can assemble the baseline without re-reading.

As you build the baseline, **capture the resolved design tokens** — concrete hex colors, font families/weights, spacing steps, radii, shadows, and verbatim SVG markup. The four concepts MUST reuse these exact values so they read as the real product, not generic mockups. Do not re-derive or guess tokens for the concepts; carry the baseline's resolved values forward.

**Then pin the shared invariants once.** These three are constant across all four concepts — the design *direction* is the only thing that changes. Write them down before generating concepts:

- **focus** — one line naming the exact UI surface being explored (the smallest component, section, layout region, or interaction under ideation). Example: "The empty state of the projects list".
- **sharedCopy** — the real content the concepts render, reused **verbatim** (headings, labels, one representative item's real text). Never paraphrase it, invent extra content, or echo the user's prompt as a title.
- **baselineStyle** — the resolved shared aesthetic every concept honors: palette, typography, border radius, spacing density, light/dark, accent. This is what keeps the four feeling like the same product.

Static resolution only — do NOT run the app, Storybook, or a browser.

### Step 2 — Auto-detect the inspiration type

Infer which **kind** of exploration the request is about, state the detected type in one line, and proceed (no confirmation needed). If genuinely ambiguous, pick the closest type and say which one you chose.

- **Entrypoint / discoverability** — *where* a new feature or action should live and how users get to it. Cues: "where should X go", "should this be in the sidebar", "how do users find/discover X", "add an entry point for X".
- **Information architecture / navigation** — *how* content and flows are organized and navigated. Cues: "reorganize", "too many tabs", "restructure the flow", "group these", "this screen is cluttered".
- **Visual display / UI treatment** — the *visual look* of a specific component or screen. Cues: "make it look better", "restyle", "different layout for this card/list", "give this a fresh look".

### Step 3 — Create the placeholder document

You (the main model) do the **creative thinking** first: turn the detected type's four archetypes (matrices in Step 4) into four bold, distinct directions and **name each concept distinctively** (e.g. "Spotlight", "Command Bar", "Split Canvas") with a 1-2 sentence **direction** thesis describing the approach that makes it different. The name becomes the concept's label; the direction is the core of each subagent brief.

**Detect the source repo** so the document keeps that context. Check you're inside a git work tree and read the origin remote:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null && git config --get remote.origin.url
```

Normalize the remote to a plain `https://github.com/<owner>/<repo>` URL (strip a trailing `.git`, and rewrite an SSH remote like `git@github.com:owner/repo.git` to `https://github.com/owner/repo`). If there is no `.git` folder or no remote, just omit `repositoryUrl` — publishing must still work without it.

**Create the placeholder** with the `create_inspiration_document` MCP tool, declaring the four concepts **without html**:

- `title`: the `focus` (e.g. "Projects list empty state").
- `files`: one entry per concept, each `{ name, description }` — the concept's name and direction thesis, **and no `html`**. Send all 4 (they render as loading tiles).
- `repositoryUrl`: the normalized URL from above, when you resolved one.
- `baseline`: `{ html, focus, sharedCopy, baselineStyle }` — `html` is the full contents of `baseline.html`, and the other three are the invariants you pinned in Step 1.

The tool returns `{ id, url, variants: [{ id, name }] }`. **Record the `id` (the inspiration id), the `url`, and each concept's `variantId`** — you hand each subagent its own `variantId` in the next step. If the tool reports an authentication error, the MCP server isn't connected/signed in — tell the user how to connect it and stop.

**Immediately open the `url` before generating the concepts (Step 4).** The page shows a faded, shimmering baseline in each concept tile and polls for updates, so the user watches every concept stream in live as its subagent finishes, instead of staring at a blank wait. Prefer the **embedded Cursor browser**: call the `cursor-ide-browser` MCP `browser_navigate` tool with the link (omit `position` so it opens in the background without stealing focus). If that MCP tool is unavailable, open it in the user's **default browser** with the OS opener: run `open <url>` on macOS, `xdg-open <url>` on Linux, or `start <url>` on Windows. **This `url` is the ONLY thing you open in a browser** — never open the local `baseline.html`, which is scaffolding for publishing, not for previewing. Only skip opening if the user explicitly asked you not to.

### Step 4 — Generate and stream in the four concepts (parallel subagents)

**Delegate the concept generation to subagents (via the Task tool), fanned out concurrently in a single batch**, one subagent per concept. Each subagent assembles its concept HTML from the brief you hand it and **calls `inspiration_add_variant` directly to push the HTML into the document** — it does NOT write a file and does NOT return the HTML to you. This keeps every concept's HTML out of your context.

**Subagent type + model.** Launch each concept subagent as **`generalPurpose`** (it needs MCP access to call `inspiration_add_variant`; the read-only `explore` type cannot) with the **`composer-2.5-fast`** model (each subagent is mostly assembling HTML from a fully-resolved brief, so speed wins).

**Divergence mandate.** The four must be genuinely different design philosophies, arrangements, or product framings — **not four shades of the same idea**. Take each of the detected type's four archetypes (matrices below) and push it to its **boldest coherent expression**, not a timid tweak. If two directions would look similar, make them diverge harder. The only axis that changes is the design direction; `focus`, `sharedCopy`, and `baselineStyle` stay pinned.

**Craft rules:**

- **Make the explored dimension the large, obvious focal change** — the distinctive idea should read at a glance. Keep everything else consistent with the baseline; render secondary/repeated content as it is in the baseline (or as neutral skeleton) so it doesn't compete with the idea.
- **Honor the baseline tokens** — same palette, typography, radius, density, light/dark, accent. Never flip light to dark (or vice versa) or jump to an unrelated visual language; the divergence is structural/directional, not a rebrand.
- **Auto-play any signature motion via CSS on a loop.** If a direction's idea involves motion or an interaction (a reveal, a transition, a marquee), demonstrate it as a self-running CSS `@keyframes`/`animation` loop (play → briefly hold → reset → repeat) so it's visible when the file is glanced at, not gated behind `:hover`, a click, or JS. Small vanilla JS is fine *in addition*, for making controls genuinely functional when the file is opened for a closer look.
- **No meta text.** No titles, captions, or frame labels inside the file that narrate the concept, and never echo the user's prompt as a heading.

**Subagent brief** — self-contained; the subagent must not re-read the source or re-derive styling. Include:

- The `inspirationId` and this concept's `variantId` (both from Step 3), and the instruction: generate the concept HTML, then call the `inspiration_add_variant` MCP tool with `{ inspirationId, variantId, html }` — passing the full HTML directly. Do NOT write an HTML file; do NOT return the HTML in the response (just confirm the concept was added).
- The concept's **name** and **direction** thesis, plus the specific archetype it expresses and precisely what to change vs. hold constant.
- The pinned invariants — `focus`, `sharedCopy` (render verbatim), `baselineStyle` — plus the concrete resolved values the file needs: hex colors, spacing/radii, font families/weights (and their `<head>` `<link>`/`@font-face`), and verbatim `<svg>` markup. The simplest reliable brief is "start from this baseline HTML, keep the tokens/copy, and change only X" — include the baseline HTML (or its salient tokens/markup) inline in the brief.
- The output-contract + craft rules: single self-contained HTML document, Tailwind Play CDN, real fonts, 1:1 SVGs, no guessed colors, honor baseline tokens, signature motion auto-plays via CSS, no meta labels.

**Fallback.** If a subagent reports it cannot call `inspiration_add_variant` (no MCP access in that subagent), have it return the finished HTML instead, and you call `inspiration_add_variant` yourself for that concept. Prefer the direct-push path.

**Entrypoint / discoverability** (same baseline chrome; the entrypoint's placement moves):

1. Primary nav / sidebar item.
2. Top bar / header action (button or icon).
3. Contextual / inline (empty-state CTA, row action, or a context menu on the relevant object).
4. Global launcher (command palette, modal, or a "+" / create menu).

**Information architecture / navigation** (same content; the structure changes):

1. Tabbed layout.
2. Grouped sidebar sections (explicit hierarchy).
3. Stepped wizard / progressive flow.
4. Single page with progressive disclosure (accordions / collapsible sections).

**Visual display / UI treatment** (same data; the visual treatment changes):

1. Alternate layout container (e.g. cards ↔ list ↔ table) and/or density.
2. Hierarchy & emphasis shift (typography scale, spacing, color emphasis).
3. Alternate component pattern (e.g. grid ↔ feed, split view).
4. Bold, distinct direction (imagery/mood) while staying within the brand tokens.

After the subagents return, each concept should be live on the page (the document flips to ready once all four are in). If a subagent failed or its concept drifted (didn't reuse the baseline tokens/copy/SVGs, didn't express its distinct direction, or its signature motion doesn't auto-play), regenerate that concept and re-push it with `inspiration_add_variant` (targeting the same `variantId`).

### Step 5 — Hand back the shareable link

Return the `url` from Step 3 to the user and tell them it renders the four concepts side by side and can be shared. Note the detected type. If publishing hit an authentication error, tell the user how to connect the MCP server.

**If the tab isn't already open, open it now** — using the same opener described in Step 3. Normally it's already open from Step 3, where the concepts fill in live as each subagent finishes, so you only need to open it here if opening was skipped there (e.g. the browser tool was unavailable then). Never open it twice.

**End your response by asking the user to pick a direction** — something along the lines of: "Let me know which one you like the best and I can implement it into this codebase. Alternatively, let me know if you want me to brainstorm more variants or flesh one out further." The exact wording can vary, but keep the same intent: invite them to choose one to implement, or ask for more variants or a deeper pass on one.

### Step 6 — Clean up the temp folder

Once you've streamed in the concepts (Step 4) and handed back the link (Step 5), **delete the temporary working directory** so nothing is left behind: `rm -rf "$TMP"` (this removes `baseline.html`). The hosted link is the deliverable; the baseline file was only scaffolding for the create call.

## Iterating on an existing document (follow-up messages)

When the user follows up about an inspiration document you already created (e.g. "swap these out", "update C & D", "make them focus more on the messaging"), **don't rebuild from scratch** — iterate on the existing document so the link stays the same.

First **load the current state** with `get_inspiration_document` (pass the `inspirationId`). It returns the current `variants` (each with `id`, `name`, `description`, and its previous `html`) and the `baseline` (`focus`, `sharedCopy`, `baselineStyle`, and the baseline `html`). You need this because a follow-up turn usually doesn't have the concept ids or the pinned invariants in context anymore.

Then **detect the intent** and take one of two paths:

**Replace all** — the user wants a fresh set of directions ("give me different options", "these aren't working, try again"). Call `inspiration_clear_variants(inspirationId)` to reset every concept back to an empty placeholder (this also drops their pre-created "Iterate" rooms). Then invent four new directions **from the baseline** and stream them in with `inspiration_add_variant` exactly like Step 4 — parallel `generalPurpose` + `composer-2.5-fast` subagents, one per concept, each pushing its own HTML directly (and updating the concept's `name`/`description` on the fill). Reuse the same pinned `focus`/`sharedCopy`/`baselineStyle` from the loaded baseline so the new concepts still read as the same product.

**Update a subset (or all)** — the user wants specific concepts revised ("update Concept C", "make B and D denser", "lean harder into the imagery"). For **only the targeted concepts**, fan out one `generalPurpose` + `composer-2.5-fast` subagent per concept. Give each subagent that concept's **previous `html`** plus the requested change, and instruct it to **revise from the previous version** (keep everything the user didn't ask to change), then call `inspiration_update_variant(inspirationId, variantId, html)` to push the revised HTML directly. **Leave the untouched concepts alone** — do not regenerate concepts the user didn't mention. The same craft/fidelity rules apply: honor the baseline tokens, render `sharedCopy` verbatim, copy SVGs 1:1, auto-play signature motion via CSS.

After either path, **open the same `url`** for the user (using the opener described in Step 3) — the document updates in place, and any tile being regenerated shows the faded baseline + shimmer until its revised concept streams in.

## Fidelity & guardrails

Carry over from [`recreate-as-raw-html`](../recreate-as-raw-html/SKILL.md):

- Every concept (and the baseline) is a single self-contained HTML document (Tailwind CDN, real fonts, static markup) that opens standalone.
- Colors/spacing/radii/fonts are resolved to concrete values from the theme source of truth — never guessed named colors.
- Icons/SVGs/logo copied 1:1 from source — never redrawn, simplified, or substituted.
- Concepts reuse the baseline's resolved tokens/fonts/SVGs verbatim; the exploration changes layout, placement, or treatment — never the underlying brand values, and never the theme (no light↔dark flip, no unrelated visual language).
- The four must be **meaningfully different** directions, not four shades of one idea — push each archetype to its boldest coherent expression.
- Any signature motion auto-plays via CSS on a loop; it is never gated behind `:hover`, a click, or JS. Extra vanilla JS is fine for functional controls when the concept is opened, but the motion itself must run on its own.
- Keep it static HTML apart from that motion/those controls. Add JS only when a concept genuinely needs it; otherwise omit it.
- Do NOT run the app, Storybook, or a browser — static resolution only.

## Common mistakes

- **Publishing a placeholder concept "to test the flow"** — never fill a concept with stub/placeholder HTML. Declare the four concepts empty at create time (Step 3), then push each one's real HTML via `inspiration_add_variant`.
- **Pulling the concept HTML back through the main agent** — the subagents push HTML straight to `inspiration_add_variant`; don't have them return the HTML for you to forward (except the no-MCP fallback).
- **Four shades of the same idea** — concepts that don't diverge enough (the primary failure). Make each a distinct design philosophy/arrangement/framing.
- Flipping the theme or jumping to a new visual language — the divergence is structural/directional; honor the baseline tokens (palette, type, light/dark, radius, density).
- Signature motion that doesn't auto-play (gated behind hover/click/JS) so it's invisible at a glance — drive it with CSS `@keyframes`/`animation` on a loop.
- Skipping the baseline and jumping straight to concepts — the baseline is what grounds every concept in the real tokens and copy (and it's uploaded as the document's baseline).
- Making concepts generic (guessed colors/fonts) instead of inheriting the baseline's resolved values.
- Varying `focus`, `sharedCopy`, or `baselineStyle` between concepts — hold those pinned so only the design direction changes and the comparison stays meaningful.
- Redrawing or substituting SVGs/logos instead of copying them 1:1 from the baseline.
- Writing concept HTML files to disk — only the baseline is written locally; concepts go straight into the document.
- On a follow-up: rebuilding a brand-new document instead of iterating on the existing one via `inspiration_clear_variants` / `inspiration_update_variant` (the share link should stay the same).
- On an update: regenerating every concept when the user only asked to revise a subset — touch only the concepts they named, and leave the rest as-is.
- On an update: discarding the previous HTML and starting the concept over instead of revising from it — load the previous `html` via `get_inspiration_document` and change only what was asked.

## Related

- `recreate-as-raw-html` — the baseline engine: recreates the current UI as faithful raw HTML that grounds every concept.
- `prototype` — build and iterate one concrete idea grounded in local UI.
- `upload-to-magic-patterns` — mirror local UI into a hosted design.
- `integrate-magic-patterns-design` — bring a chosen concept into this codebase as production code.
