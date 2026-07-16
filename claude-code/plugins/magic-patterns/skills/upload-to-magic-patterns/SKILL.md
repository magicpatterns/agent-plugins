---
name: upload-to-magic-patterns
description: Upload UI you are building locally into Magic Patterns (magicpatterns.com) and return the editor link. Use when the user wants to push a component/screen they are working on in their codebase into Magic Patterns as a design — for review, stakeholder sign-off, or further iteration. Requires the Magic Patterns MCP server.
---

# Upload local UI to Magic Patterns

Use this skill to push UI from the current codebase into a **hosted Magic Patterns design** and hand back the editor link. You port the local UI into a Magic Patterns-ready React file set with the `recreate-as-react` skill, then **write those files into a blank design directly** — you do not prompt Magic Patterns to generate anything. The destination is a real Magic Patterns design, created through the Magic Patterns MCP server.

Magic Patterns renders **visual prototypes**, not production apps. Local production code will not run there as-is — it must be ported to Magic Patterns' constraints first. `recreate-as-react` does that port; this skill wires the result into a design.

**Keep the scope minimal.** Upload only what the user wants to prototype — the specific component or screen and the smallest set of files needed to render it. This is not a tool for mirroring the whole app; a tighter upload compiles more reliably and is faster to iterate on.

## Prerequisites

- The Magic Patterns MCP server must be connected (tools prefixed `magic-patterns`). If the tools are unavailable, tell the user to install/enable the Magic Patterns plugin or MCP server, and stop.
- Magic Patterns MCP usage requires a paid plan. A `Payment Required` / credits error means the user must upgrade at `magicpatterns.com/settings`.

## Workflow

### Step 1 — Pick the minimal scope to prototype

Identify the specific component(s) or screen the user wants to prototype, and the smallest set of files needed to render it. Do not pull in the whole app — keep the upload tight to what the user actually wants to see.

### Step 2 — Port to a Magic Patterns-ready file set (run `recreate-as-react`)

Run the **`recreate-as-react`** skill to turn the target into a self-contained React + TypeScript + Tailwind file set (`App.tsx`, `index.tsx`, `index.css`, `tailwind.config.js`, component files). That skill already targets the Magic Patterns environment and carries the fidelity rules that make the upload look right — follow it in full. In particular it:

**Emit into a temp dir, not the project.** Tell `recreate-as-react` to write the file set into a temporary working directory (`mktemp -d`, referred to as `$TMP`) so nothing is left behind in the user's repo — this skill reads those files and writes them into the design, then deletes `$TMP` (Step 4).

### Step 3 — Create a blank design and upload the files directly

Create a blank design and write the ported files into it. **Do not use `send_prompt` or any AI generation** — write the files yourself. Use the Magic Patterns MCP tools in this order:

1. `create_design` with no prompt → "start from scratch": returns `editorId`, the active `artifactId` (a blank scaffold: `App.tsx`, `index.tsx`, `index.css`, `tailwind.config.js`), and the `editorUrl`. This returns immediately.
2. `get_design_status(editorId)` → confirm the active `artifactId` and that `isGenerating` is false.
3. `write_artifact_files(artifactId, files)` → write the ported files from `$TMP` directly. Include `App.tsx` as the entry component and update `index.css` if you added fonts/tokens.
4. `publish_artifact(artifactId, editorId)` → compiles and activates the artifact so the preview renders.

If you need to preserve the original scaffold as a separate version, call `create_new_artifact(artifactId, name)` before writing and use the returned artifact ID for the writes.

### Step 4 — Hand back the link, open it, and clean up

Return the `editorUrl` to the user so they can keep iterating in Magic Patterns, and tell them they may need to log in. If `publish_artifact` reported compile errors, fix the offending files and publish again before returning the link.

**Always open the `editorUrl` for the user — don't just print it.** Open it in the user's **default browser** with the OS opener: run `open <url>` on macOS, `xdg-open <url>` on Linux, or `start <url>` on Windows. Only skip opening if the user explicitly asked you not to.

Then delete the temp working directory (`rm -rf "$TMP"`) so nothing is left in the user's project.

## Common mistakes to avoid

- **Uploading too much** — pulling in the whole app instead of the minimal scope the user wants to prototype. Keep it tight.
- **Prompting instead of writing files** — this skill writes the files directly with `write_artifact_files`; do not use `send_prompt` to generate the UI.
- **Skipping the `recreate-as-react` port** — pasting production code unchanged (live `fetch`/auth/env) fails to compile or renders blank. Always port first.
- **Rebuilding the UI from scratch in approximate Tailwind** — the most common cause of an ugly upload. `recreate-as-react` prefers importing the source's real component library (e.g. Cubed → `@radix-ui/themes`); let it.
- **Reordering or deleting the Tailwind imports** in `index.css` — breaks all styling.
- **Default-exporting the root component** or leaving non-`lucide-react` icons in place.
- **Relative image paths** — use absolute URLs.
- **Returning the link before `publish_artifact` succeeds** — confirm it compiled first.
- **Leaving the temp dir behind** — delete `$TMP` after uploading.

## Related

- `recreate-as-react` — the porting engine: turns the local UI into the Magic Patterns-ready React file set this skill uploads.
- `prototype` — seed with this flow, then prompt Magic Patterns to design a new idea on top.
- `integrate-magic-patterns-design` — go the other direction: bring a Magic Patterns design into this codebase as production code.
