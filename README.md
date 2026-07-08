# CheckFox accessibility audit skill

An agent skill that lets an AI agent run a real web accessibility audit against
**RGAA**, **RAWeb**, and **WCAG** — one criterion at a time — through the
[CheckFox](https://checkfox.eu) Audit Engine over the Model Context Protocol (MCP).

The agent reads each criterion's official methodology and tests, gathers its own
evidence (source analysis, a real browser, or CheckFox's automated scanner),
decides a status, and writes the findings and a suggested fix back into a CheckFox
audit. CheckFox is the read-and-write substrate; the agent is the auditor.

## What's in here

- [`SKILL.md`](./SKILL.md) — the skill itself: golden rules, the audit loop,
  how to choose a verification method per criterion, and the connection setup.
  Load it into a Claude Agent Skill, a `SKILL.md`-aware client, or paste it into
  an agent's system prompt.

## Requirements

- A [CheckFox](https://checkfox.eu) account with at least one audit.
- Access to the CheckFox MCP server, via **either**:
  - the **native OAuth connector** (Claude on the web / Desktop — no key to
    manage), or
  - a **scoped API key** (`mcp:read`, and `mcp:write` to record verdicts or run
    the scanner) generated in *User settings → API keys*.
- An MCP-capable client (Claude Code, Claude Desktop, or any HTTP MCP client).

`run_scanner` additionally needs the scanner add-on enabled on the audit's
workspace plan; every other tool works on any plan.

## Quick start

Endpoint: `https://api.checkfox.eu/mcp`

```bash
claude mcp add --transport http checkfox \
  https://api.checkfox.eu/mcp \
  --header "Authorization: Bearer cfx_live_…"
```

Then ask your agent to audit: it will `get_audit`, walk each sample with
`get_next_criterion`, and record verdicts with `update_criterion`. Full
step-by-step guide: <https://checkfox.eu/mcp>.

## Languages

`SKILL.md` is English. The same operating guide and the whole criteria
referential are served **in French** straight from the MCP server — append
`?lang=fr` to any resource URI (e.g. `checkfox://skill/audit?lang=fr`,
`checkfox://criteria/rgaa?lang=fr`).

## Built on Geoffrey Crofte's accessibility expertise

The methodology this skill drives reflects the accessibility expertise of
[Geoffrey Crofte](https://github.com/geoffreycrofte). For a vendor-neutral,
standards-first companion skill, see the
[Luxembourg accessibility skillset](https://github.com/geoffreycrofte/luxembourg-accessibility-skillset).

## Staying in sync

The operating-guide portion of `SKILL.md` mirrors the canonical source that the
CheckFox MCP server serves live as the `checkfox://skill/audit` resource, so a
connected agent always gets a version that matches the server's exact tool
surface. When the server's tools change, this file is updated to match.

## Learn more

- CheckFox MCP guide: <https://checkfox.eu/mcp>
- CheckFox: <https://checkfox.eu>
