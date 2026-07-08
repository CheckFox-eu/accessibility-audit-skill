---
name: checkfox-accessibility-audit
description: >-
  Audit web accessibility against RGAA / RAWeb / WCAG one criterion at a time using
  the CheckFox MCP server: read each criterion's official methodology and tests,
  gather your own evidence (source analysis, a real browser, or CheckFox's scanner),
  decide a status, and write the findings back into a CheckFox audit. Use when the
  user wants an AI agent to run or continue a CheckFox accessibility audit.
---

# Auditing accessibility with the CheckFox MCP

You are connected to the CheckFox Audit Engine over MCP. CheckFox is the
read-and-write substrate for an accessibility audit; **you are the auditor**. For
each criterion you read the official methodology and tests, gather your own
evidence, decide a status, and write your findings back into the audit.

> Not connected yet? See **Setup** at the bottom of this file, then come back here.

## Golden rules

- **Decide how to verify each criterion yourself.** Most criteria are judgement
  calls no scanner can settle. Pick the cheapest sufficient evidence.
- **Never invent evidence.** If you cannot verify a criterion with the tools you
  have, leave it `Not tested` and say so, rather than guessing a status.
- **You augment humans, you never overwrite them.** Writes only land on a
  criterion still marked `Not tested`; an existing verdict is protected.

## The loop

Work one sample at a time, one criterion at a time:

1. `get_audit` — see the audit's samples (pages/components) and progress. Pick a
   sample.
2. `get_next_criterion` for that sample.
   - `done: true` → the sample is complete; move to the next sample.
   - Otherwise read `criterion.tests` and `criterion.methodology` **in full**.
3. Choose how to verify **this** criterion:
   - **Structural / attribute checks** (alt text, labels, headings, landmarks,
     lang, doctype, id uniqueness): analyse the page source. You may call
     `run_scanner` once per sample and reuse its mappings as corroboration.
   - **Behavioural checks** (keyboard operability, focus visibility and order,
     reflow at 320px, motion control, status messages, no keyboard trap): drive a
     real browser and exercise the page. Do not rely on the scanner for these.
   - **Editorial / judgement checks** (link relevance, meaningful alternatives,
     content order, consistent navigation): reason over the rendered content.
4. Decide a status: `Success` | `Failure` | `N/A` | `Derogation`. For
   `Failure`/`Derogation`, set `user_impact` (`low` | `disturbing` | `blocking`).
5. `update_criterion` with:
   - `problem` — what fails and where (selectors, snippets, which test of the
     criterion), concrete enough for a developer to locate it.
   - `solution` — how to fix it, with a short corrected code sample when useful.
6. Repeat. If a write returns `WRITE_GUARD`, that criterion was already decided
   by a human — skip it and continue.

## Using the scanner

`run_scanner` (optional, `mcp:write`) returns
`{ proposal: { violation_count, mappings[] } }`. Each mapping carries
`criterion_id`, `source` (`axe` | `checkfox`), and `needs_review`
(CheckFox judgement findings you must confirm, never auto-apply). Treat it as one
input to your decision — then still call `update_criterion` yourself with your
own findings and solution.

## Running several agents in parallel

Writes are per-criterion and guarded, so you can shard an audit safely. Give each
agent a disjoint range (by sample, or by criterion-number range). To avoid two
agents doing the same work, call `get_next_criterion` with `claim: true` and a
stable `agent_id`; the server skips criteria other agents hold and claims the one
it hands you. A claim is released automatically when you `update_criterion`, or on
its TTL. Call `release_criterion` if you claim one but decide not to audit it.

## When you get an error

- `WRITE_GUARD` — already decided by a human; skip it.
- `Missing required scope` — the key lacks `mcp:read`/`mcp:write`.
- `404` on an audit/sample — your key's user has no access to it.
- `429` — rate limited; wait and retry, honouring `Retry-After`.

## Tools

| Tool | Scope | Purpose |
| --- | --- | --- |
| `list_audits` | `mcp:read` | Audits you can access. |
| `get_audit` | `mcp:read` | One audit: samples + completion progress. |
| `list_criteria` | `mcp:read` | A sample's criteria with current status. |
| `get_next_criterion` | `mcp:read` | Next `Not tested` criterion + full methodology/tests. |
| `get_criterion_methodology` | `mcp:read` | A criterion's methodology by guideline + number. |
| `run_scanner` | `mcp:write` | Optional automated scan for extra signal. |
| `update_criterion` | `mcp:write` | Record a verdict (guarded write). |
| `release_criterion` | `mcp:write` | Drop an advisory claim you hold. |

The full accessibility referential is also exposed as MCP **resources**
(`checkfox://criteria/{guideline}` and `checkfox://criteria/{guideline}/{num}`);
append `?lang=fr` for French. This guide itself is served live as
`checkfox://skill/audit` (also `?lang=fr`).

## Setup

Connect the CheckFox MCP once, then follow the loop above.

**Endpoint:** `https://api.checkfox.eu/mcp`

**Option A — native connector (no key).** In Claude, open Settings → Connectors →
Add custom connector, name it `CheckFox`, paste the endpoint as the URL, and sign
in. Grant `mcp:read` (and `mcp:write` to record verdicts / run the scanner).

**Option B — API key.** In CheckFox, open User settings → API keys → Generate new
key, tick `mcp:read` (and `mcp:write`), and configure your client:

```bash
claude mcp add --transport http checkfox \
  https://api.checkfox.eu/mcp \
  --header "Authorization: Bearer cfx_live_…"
```

For Claude Desktop or any stdio-only client, bridge via `mcp-remote`; for a raw
check, `POST` a `tools/list` JSON-RPC call to the endpoint with your Bearer key.
Full guide: <https://checkfox.eu/mcp>.
