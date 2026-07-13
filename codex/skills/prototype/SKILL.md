---
name: prototype
description: Prototype an idea in Magic Patterns (magicpatterns.com) starting from your local UI. Use when the user gives a prompt for something they want to prototype or explore and wants an iterable Magic Patterns design grounded in their existing code. Seeds a design by recreating the relevant local UI (recreate-as-react) and uploading it, then prompts Magic Patterns to design the new idea, returns the editor link, and opens it in the browser. Requires the Magic Patterns MCP server.
---

# Prototype an idea in Magic Patterns

Use this skill when the user has an idea they want to **prototype** and wants it grounded in the UI they already have. It seeds a Magic Patterns design from the relevant local UI, then prompts Magic Patterns to design the idea, hands back the editor link, and opens it in the browser.

It composes two mechanisms, used **in order, for different jobs** — not interchangeably:

- **Seed (deterministic):** recreate the relevant *existing* local UI with `recreate-as-react` and write it into a blank design (the `upload-to-magic-patterns` flow). This recreates what already exists — it does NOT build the new idea.
- **Prototype the idea (creative):** once seeded and published, use `send_prompt` to let Magic Patterns design the net-new idea. This is where the prototyping happens — let Magic Patterns do the creative heavy lifting; don't hand-build it.

Direct file writes *after* seeding are reserved for precise, deterministic tweaks (rename a label, swap a color, port one specific local component) — never for generating the new concept.

Magic Patterns renders **visual prototypes**, not production apps — local code must be ported to its constraints (handled by `recreate-as-react`; see `upload-to-magic-patterns`).

## Prerequisites

- The Magic Patterns MCP server must be connected (tools prefixed `magic-patterns`). If the tools are unavailable, tell the user to install/enable the Magic Patterns plugin or MCP server, and stop.
- Magic Patterns MCP usage requires a paid plan. A `Payment Required` / credits error means the user must upgrade at `magicpatterns.com/settings`.

## When to use

- **This skill (`prototype`)**: the user wants to prototype or evolve an idea grounded in their existing UI, and iterate on it.
- `upload-to-magic-patterns`: just mirror local UI into a hosted design, no iteration.

- `integrate-magic-patterns-design`: bring a Magic Patterns design back into the codebase as production code.

## Links: always hand back the editor link

Designs created via the MCP server use normal Magic Patterns access controls. Return the editor link and tell the user to open it in their browser; they may need to log in.

`create_design` returns two URLs:

- **`editorUrl`** (`…/c/<editorId>`) — the editor. **This is what you give the user to view or edit the prototype.**
- **`previewUrl`** (`project-<slug>.magicpatterns.app`) — an optional rendered preview URL. Do not rely on it as the primary no-login link.

Always return the `editorUrl`.

## Workflow

The two mechanisms are sequential and do different jobs: **seed** recreates existing UI deterministically, then **prompt** lets Magic Patterns design the new idea. Do them in this order.

### Step 1 — Clarify the goal and pick the seed (existing UI only)

From the user's prompt, work out (a) the idea they want to prototype and (b) the smallest existing screen/component it builds on. Keep the seed scope minimal — just enough existing UI to ground the prototype, not the whole app, and not the new feature.

### Step 2 — Seed: recreate the existing local UI in Magic Patterns

**Run the `upload-to-magic-patterns` skill** on the seed scope to mirror the relevant local UI into a fresh design. It does the whole seed for you — ports the UI with `recreate-as-react` (into a temp dir), then `create_design` → `get_design_status` → `write_artifact_files` → `publish_artifact`, and cleans up the temp dir. **Keep the `editorId` and `editorUrl` it returns** for the prompt step below. Don't re-implement that sequence here.

**Mirror only what already exists. Do NOT invent the new feature here.** If there is no relevant local UI to seed from, start from a minimal base instead.

**Fidelity matters most here.** The generated feature inherits the seed's look and layout, so an approximate seed produces an approximate (often ugly) result. `recreate-as-react` reproduces the existing UI faithfully — preferring the source's real component library, which MP installs from `package.json` (e.g. Cubed → `@radix-ui/themes` + its theme CSS + a `<Theme>` wrapper) — rather than rebuilding it in approximate Tailwind. A faithful, well-sized seed is what keeps the later `send_prompt` result from clashing or overflowing.

### Step 3 — Hand back the editor link (checkpoint)

Give the user the `editorUrl` so they can open the starting point before spending generation credits, and open it in the browser (see Step 6). Tell them they may need to log in.

### Step 4 — Prototype the idea with a prompt (the heavy lifting)

This is where the prototyping happens — **let Magic Patterns do the creative design work.** Write a clear spec of the desired feature (goals, behaviors, the user's desires, constraints) and send it:

1. `get_design_status(editorId)` first to get the **current** active `artifactId` — the editor is collaborative and the active artifact can change between steps.
2. `send_prompt(editorId, "<spec of the idea to prototype>")`.
3. Generation runs in the background. Poll `get_design_status` at most once every 60 seconds only if you need to programmatically inspect the result afterward (e.g. `read_artifact_files`).

**Default to this for any net-new or creative change. Do NOT hand-build the new feature with `write_artifact_files`** — that defeats the purpose of prototyping in Magic Patterns.

*Exception — direct file writes:* only for precise, deterministic tweaks on top of a generated result (rename a label, swap a color, port one specific local component). Use `write_artifact_files(...)` then `publish_artifact(...)`. Never use this path to generate the concept itself.

### Step 5 — Hand back the link and iterate

After each meaningful change, return the `editorUrl` so the user can view and keep iterating in Magic Patterns. Offer to keep iterating, or to pull the result back into the codebase with the `integrate-magic-patterns-design` skill.

### Step 6 — Open the editor in the browser

**Always open the `editorUrl` for the user — don't just print it.** Prefer the **embedded Cursor browser**: call the `cursor-ide-browser` MCP `browser_navigate` tool with the `editorUrl` (omit `position` so it opens in the background without stealing focus). If that MCP tool is unavailable, open it in the user's **default browser** with the OS opener: run `open <editorUrl>` on macOS, `xdg-open <editorUrl>` on Linux, or `start <editorUrl>` on Windows. Only skip opening if the user explicitly asked you not to. Open it at the Step 3 checkpoint (the seeded starting point) and again after a `send_prompt` change lands.

## Notes and limitations

- **`send_prompt` is asynchronous:** hand off the editor link for the user to view in their browser. Poll `get_design_status` only when you need to act on the result programmatically.
- **Keep scope tight:** a focused prototype compiles reliably and iterates fast. Resist seeding or generating the whole app.
- **Confirm before regenerating:** each `send_prompt` consumes credits.

## Related

- `recreate-as-react` — the porting engine used to build the seed.
- `upload-to-magic-patterns` — the seed step: recreate local UI and write it into a design directly.
- `inspiration` — explore several parallel design directions instead of building one idea.
- `integrate-magic-patterns-design` — bring the prototyped design back into the codebase as production code.
