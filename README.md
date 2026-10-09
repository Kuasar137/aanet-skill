# aanet-skill

An [Agent Skill](https://github.com/anthropics/skills) for [AANet](https://aanet.space) —
a metered coordination service for autonomous agent swarms (shared
workspaces, distributed locks, budgeted sub-keys, file storage, messaging,
billed per operation via x402).

This isn't a package or a dependency — it's a single file, [`SKILL.md`](SKILL.md),
that teaches an agent *how to use AANet well*, including operational
lessons (a lock isn't a completion record, check your budget header before
guessing, prefer MCP over raw HTTP for sub-agents you spawn) that came from
actually dogfooding the service, not just its API reference.

## Using it

**Claude Code, one command** — installs the skill and the `aanet` remote MCP
server together:

```
/plugin marketplace add Kuasar137/aanet-skill
/plugin install aanet@aanet-tools
```

**Codex CLI** auto-discovers skills placed at `.agents/skills/<name>/SKILL.md`
in a project — clone or vendor this repo's `.agents/skills/aanet/SKILL.md`
into yours, or add the MCP server directly: `codex mcp add aanet --transport
http https://aanet.space/mcp`.

**Manual install for any Claude Code / Claude apps with Skills support:**
drop this repo (or just `SKILL.md`) into your skills directory, e.g.
`~/.claude/skills/aanet/` or your project's `.claude/skills/aanet/`.

**Any other agent harness that reads Markdown instructions:** point your
agent at the raw file —
`https://raw.githubusercontent.com/Kuasar137/aanet-skill/main/SKILL.md` —
or paste its contents into your system prompt / tool-instructions when a
task involves multi-agent coordination.

`SKILL.md` is duplicated at the repo root (for the raw-URL/manual-copy
install above), `skills/aanet/SKILL.md` (Claude Code plugin convention) and
`.agents/skills/aanet/SKILL.md` (Codex convention) — all three must stay
identical; update all three together.

The skill's own `description` frontmatter field is written for
description-based auto-triggering (an orchestrator deciding "does this task
need a coordination skill?") — adjust it if your harness's triggering
mechanism works differently.

## Related

- [aanet-examples](https://github.com/Kuasar137/aanet-examples) — runnable
  scripts demonstrating the lessons this skill calls out.
- [aanet.space/AGENTS.md](https://aanet.space/AGENTS.md) — the full API
  reference this skill assumes as background.

## License

MIT.
