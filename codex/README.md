# Magic Patterns — Codex plugin

Use [Magic Patterns](https://magicpatterns.com) from OpenAI Codex: prototype ideas,
generate UI inspiration, upload local UI, and integrate Magic Patterns designs into your
codebase. Bundles the Magic Patterns MCP server (`.mcp.json`).

## Skills

| Skill | What it does |
| --- | --- |
| `prototype` | Prototype an idea in Magic Patterns, seeded from your local UI, then iterate into a live design. |
| `inspiration` | Generate four parallel UI concepts for a screen/component and return a link to compare them. |
| `upload-to-magic-patterns` | Upload local UI into Magic Patterns and return the editor link. |
| `integrate-magic-patterns-design` | Adapt a Magic Patterns design into an existing codebase as production code. |

## Install

Install via the Codex plugin directory (`/plugins`), or add the MCP server directly by
copying the `magic-patterns` block from `.mcp.json` into `~/.codex/config.toml` as
`[mcp_servers.magic-patterns]` (set `url = "https://mcp.magicpatterns.com/mcp"`).

> Generated from `agent-plugins/` in the Magic Patterns monorepo. Do not edit here; edit
> the source of truth and re-run the export.
