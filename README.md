# Magic Patterns plugins

Official [Magic Patterns](https://www.magicpatterns.com) plugins for **Claude Code**,
**Cursor**, and **OpenAI Codex** — prototype ideas, generate UI inspiration, upload local
UI, and integrate Magic Patterns designs straight from your coding agent. Each plugin
bundles the Magic Patterns MCP server, so installing also wires up the connection.

> **This repository is generated.** The source of truth is `agent-plugins/` in the Magic
> Patterns monorepo — the skills are authored once there and the MCP endpoint lives in one
> `mcp.json`. Every tree here is assembled by `agent-plugins/scripts/export-plugins.mjs`.
> Don't edit files here; they're overwritten on the next export.

## Layout

Each top-level directory is a self-contained plugin source that an independent marketplace
can review in isolation:

```
.claude-plugin/marketplace.json        Claude Code catalog (must live at the repo root)
claude-code/plugins/magic-patterns/    Claude Code plugin — manifest, skills/, .mcp.json
cursor/                                Cursor plugin       — manifest, skills/, mcp.json
codex/                                 Codex plugin        — manifest, skills/, .mcp.json
```

All three ship the same four skills and the same MCP server
(`https://mcp.magicpatterns.com/mcp`); only the per-tool packaging differs.

## Skills

| Skill | What it does |
| --- | --- |
| `prototype` | Prototype an idea in Magic Patterns, seeded from your local UI, then iterate into a live design. |
| `inspiration` | Generate four parallel UI concepts for a screen/component and return a link to compare them. |
| `upload-to-magic-patterns` | Upload local UI into Magic Patterns and return the editor link. |
| `integrate-magic-patterns-design` | Adapt a Magic Patterns design into an existing codebase as production code. |

## Install

**Claude Code**

```
/plugin marketplace add magicpatterns/agent-plugins
/plugin install magic-patterns@magic-patterns
```

**Cursor** — install from the Cursor Marketplace, or for local testing copy `cursor/` to
`~/.cursor/plugins/local/magic-patterns` and restart.

**Codex** — install `codex/` via the Codex plugin directory (`/plugins`), or add the MCP
server to `~/.codex/config.toml` as `[mcp_servers.magic-patterns]` with
`url = "https://mcp.magicpatterns.com/mcp"`.

## License

MIT — see [LICENSE](./LICENSE).
