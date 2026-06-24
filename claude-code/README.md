# Magic Patterns — Claude Code plugin

Use [Magic Patterns](https://magicpatterns.com) from Claude Code: prototype ideas,
generate UI inspiration, upload local UI, and integrate Magic Patterns designs into your
codebase. Bundles the Magic Patterns MCP server so installing also wires up the
connection.

## Skills

| Skill | What it does |
| --- | --- |
| `prototype` | Prototype an idea in Magic Patterns, seeded from your local UI, then iterate into a live design. |
| `inspiration` | Generate four parallel UI concepts for a screen/component and return a link to compare them. |
| `upload-to-magic-patterns` | Upload local UI into Magic Patterns and return the editor link. |
| `integrate-magic-patterns-design` | Adapt a Magic Patterns design into an existing codebase as production code. |

## Install

```
/plugin marketplace add magicpatterns/magic-patterns-plugins
/plugin install magic-patterns@magic-patterns
```

The marketplace catalog lives at the repository root (`.claude-plugin/marketplace.json`)
and points at this plugin under `claude-code/plugins/magic-patterns`.

> Generated from `agent-plugins/` in the Magic Patterns monorepo. Do not edit here; edit
> the source of truth and re-run the export.
