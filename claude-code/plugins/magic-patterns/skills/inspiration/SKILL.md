---
name: inspiration
description: Generate four distinct UI design concepts for a screen or component in Magic Patterns (magicpatterns.com) and return a link where the user can compare them and copy the prompt to build one. Use when the user wants design inspiration, alternative directions, layout options, variations, wireframes, or a "show me a few approaches" exploration before committing to an implementation. Requires the Magic Patterns MCP server.
---

# Generate design inspiration concepts in Magic Patterns

Use this skill when the user wants to **explore directions** rather than build one specific thing. It asks Magic Patterns to generate **four parallel UI concepts** for the same target, then returns the editor link where the user can compare the concepts in Magic Patterns.

Under the hood this reuses the existing Magic Patterns flow: it works inside a design (a "chat pattern"), runs the built-in `/Inspiration` command via `send_prompt`, and surfaces the result inside the Magic Patterns editor. There are no dedicated inspiration MCP tools — compose the existing ones.

## Prerequisites

- The Magic Patterns MCP server must be connected (tools prefixed `magic-patterns`). If the tools are unavailable, tell the user to install/enable the Magic Patterns plugin or MCP server, and stop.
- Magic Patterns MCP usage requires a paid plan; one inspiration run generates four concepts and consumes credits. A `Payment Required` / credits error means the user must upgrade at `magicpatterns.com/settings`.

## When to use

- **This skill (`inspiration`)**: the user wants several directions to compare before committing — layout/IA exploration, alternative visual treatments, "give me options".
- `prototype`: the user already knows what they want and wants to build/iterate one concrete idea.
- `upload-to-magic-patterns`: just mirror local UI into a hosted design, no concepts.
- `integrate-magic-patterns-design`: bring a chosen concept back into the codebase as production code.

## Best results: give Magic Patterns context first

Concepts are far stronger when Magic Patterns can see the real UI and theme they are riffing on. Decide between **reuse** and **seed**:

- **Reuse an existing design** — if the user points at a `magicpatterns.com` design (or you just created/seeded one in this session), run inspiration on that. Resolve a URL with `get_editor_id_from_url(url)`.
- **Seed from local UI** — if the user is exploring directions for a component in *this* codebase, first mirror the relevant local UI into a fresh design with the `upload-to-magic-patterns` flow, then run inspiration on it so the concepts inherit the real layout and theme.
- **Blank fallback** — only if there's nothing to ground it; inspiration on an empty design produces generic concepts.

## Workflow

### Step 1 — Get an editor to run inspiration in

- Existing design: `editorId = get_editor_id_from_url(url)`.
- Otherwise seed: run the `upload-to-magic-patterns` flow (`create_design` → `write_artifact_files` → `publish_artifact`) to mirror the relevant local UI, and keep the returned `editorId`. Keep the seed scope minimal — just enough context to ground the concepts.

### Step 2 — Run the four concepts

1. `get_design_status(editorId)` to confirm the design is ready and not already generating.
2. `send_prompt(editorId, "/Inspiration <clear description of the target and what to explore>")`.

Write the request to name the target (screen/component) and the kind of variation wanted (layout, hierarchy, visual treatment, density, etc.). The `/Inspiration` command is verbatim and capitalized. Magic Patterns generates one shared scene plus four focal concept variations in parallel.

### Step 3 — Hand back the editor link

Generation is asynchronous (typically a few minutes). Return the editor link and tell the user to open it in their browser; they may need to log in:

```
https://www.magicpatterns.com/c/<editorId>
```

Tell the user the four concepts render inside the editor. Poll `read_recent_message_history(editorId)` or `get_design_status(editorId)` only if you need to programmatically confirm progress before another tool call.

### Step 4 — Continue after they pick (optional)

Once the user has a concept they like, they can paste the copied prompt back here (or into Magic Patterns) to build it, or you can pull a chosen direction into this codebase with the `integrate-magic-patterns-design` skill.

## Notes and limitations

- **Async + slow:** concept generation typically takes a few minutes. Don't poll aggressively — hand back the editor link and let the user watch it in Magic Patterns, or poll `read_recent_message_history` at a relaxed interval.
- **Normal access controls:** designs created via MCP may require login. Don't promise a public no-login link.
- **Selection happens in the browser:** comparing and choosing a concept happens in the editor. There is no MCP tool to read individual concept statuses or pick one programmatically.
- **Don't run repeatedly:** each run is four generations and consumes credits. Confirm the request rather than firing multiple inspiration runs.

## Related

- `prototype` — build and iterate one concrete idea grounded in local UI.
- `upload-to-magic-patterns` — seed a design with local UI before inspiring, for context-aware concepts.
- `integrate-magic-patterns-design` — bring a chosen concept into this codebase as production code.
