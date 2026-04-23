# peec-ai-mcp

A companion skill for the [Peec AI MCP server](https://docs.peec.ai/mcp/introduction). Drop it into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.) to get correct, actionable guidance on how to use the Peec MCP — including the gotchas the official docs omit or get wrong.

Released under [CC BY 4.0](./LICENSE). Reuse, adapt, redistribute — keep the attribution.

> **Status:** v1.4.1 — patch release. Pre-flight checklist gains a new item #12: "Write-time metric-type re-check before any number is typeset." Items 1–2 fire at read-time when a value is first pulled from Peec; this new item fires at write-time when the value is placed in a slide, email, or report — the point at which metric-type context most often evaporates. Names the specific failure mode: a `position` value of 2.0 typeset next to a competitor's 2.8 reads as if the bigger number wins, inverting the Rank-where-lower-is-better scale. Fix is to either flip the presentation (ordinal labels `#1`/`#2`/`#3`, "lowest = best" caption) or replace raw numbers with ordinals. §7.37 gains a companion "Read-time vs write-time checks — run this rule twice, not once" block reinforcing the same discipline from the metric-type-rules side. PRs welcome.
>
> **Previous: v1.4.0 — minor release.** §7.5 now enumerates the broader conditions under which the retrieval/parametric split shifts engine-wide (well-known brands, recognised standards, commercial categories, regulated verticals) with a strategic-implications block and a cross-reference to the tracking-strategy-builder §11.13. §7.12 gains a "data vs interpretation" warning block on `get_actions` output — homepage-keyword-stuffing, Reddit-tactic imports, sister-brand inspiration, and PR-to-affiliate routing are all flagged as systematic interpretation errors — plus an EDITORIAL/COMPARISON affiliate-default callout with verification indicators (no affiliate params, no commission disclosure, editorial byline) and a vertical-prior flip rule for commercial verticals. §8.1b adds a "Combined / group visibility" recipe: the right way to compute a multi-brand visibility figure is `get_brand_report` with `filters=[{field: "brand_id", operator: "in", values: [...]}]` (never sum percentages), with an alternative manual chat-ID union approach and explicit reporting-provenance rule.
>
> **Value scales with task complexity.** This is a comprehensive reference (~16k words / ~26k tokens when loaded) designed to replace trial-and-error on multi-step Peec work — full visibility reports, per-engine comparisons, competitive gap analysis, source-authority audits, project tune-ups. For trivial single-tool lookups like `list_projects` or `list_brands`, Peec's own tool descriptions are usually enough; the frontmatter deliberately avoids triggering on those. The payoff lands on tasks where data-interpretation gotchas (sentiment scale, position semantics, retrieval-vs-citation, `get_actions` two-step workflow, `list_prompts.volume` type coercion, `get_url_content` refresh cadence) or schema asymmetries (§7.39, §7.31) would otherwise produce confidently wrong output. Internal A/B testing across complexity tiers (simple lookups → deep multi-step audits) showed skill-assisted runs on the complex tier produced materially more correct output at a modest token cost, while the simple tier saw effectively no benefit — which is why the description is scoped to analysis and multi-step work, not every Peec mention.

## What this skill teaches the agent

- **Data model clarity.** Brands, prompts, topics, and tags are orthogonal in Peec — no `brand_id` foreign key on prompts, no write path to "assign" prompts to brands. The skill makes this explicit so agents don't attempt impossible workflows.
- **All 27 MCP tools** (15 read-only, 8 write, 4 destructive) with correct usage.
- **7 slash-command prompts** (`/peec_weekly_pulse` etc.) and when to use each.
- **42 data-literacy gotchas.** Scales are mixed within a single row, `position` means rank-among-tracked-brands not overall, `list_models` returns 16 engines while the `model_id` filter enum lists 19, the sentiment formula isn't what the docs imply, `get_actions` is reliably callable from pass-through clients (schema-strict clients strip params — fallback recipe included), `retrieval_rate` and `citation_rate` can exceed 1.0, domain and URL reports return different types for the same-named columns, engine-returned empty response bodies silently inflate non-mention counts, `get_brand_report` dimension labels can return null within the first 24h after a prompt-write wave, `list_search_queries` (fanout) returns zero rows for AI Overview, AI Mode, and Copilot but is confirmed to return data for ChatGPT and Grok, `list_prompts.volume` returns string ordinals (not the 1–5 integers the schema claims), and more.
- **Hidden features** — the `gap` filter on domain/URL reports, `mentioned_brand_count` filter, `regex` on `create_brand`, wave-based execution for bulk changes, scraped content via `get_url_content` (with the 5-day refresh cadence and dual classification fields documented).
- **11 composite recipes** covering the analyses people actually want to run — full visibility reports, per-engine head-to-head comparisons, competitive gap analysis, source authority audit, safe test-entity lifecycle, and more.

## What's in this repo

| File | Purpose |
|---|---|
| [`SKILL.md`](./SKILL.md) | The skill itself. Load this into your MCP agent. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to propose changes. |
| [`LICENSE`](./LICENSE) | CC BY 4.0. |

For **connecting the Peec MCP server to your client**, follow Peec's official setup docs at [docs.peec.ai/mcp/introduction](https://docs.peec.ai/mcp/introduction). This skill deliberately doesn't duplicate that content — Peec maintains the up-to-date client list and connection flows.

## Install

### Claude Desktop / Cowork

Skills load from your Cowork mount's `.claude/skills/` directory. Clone into there, restart the app, and the skill's description will trigger on Peec-related queries.

```bash
# macOS path — adjust for your system
cd ~/Library/Application\ Support/Claude/skills
git clone https://github.com/rebelytics/peec-ai-mcp peec-ai-mcp
```

### Claude Code

```bash
mkdir -p .claude/skills
git clone https://github.com/rebelytics/peec-ai-mcp .claude/skills/peec-ai-mcp
```

Claude Code will pick the skill up automatically from the project's skills directory.

### Cursor / VS Code / Windsurf

These clients don't have a unified skills model yet. For now, copy `SKILL.md` into your project and reference it explicitly in your AI context (e.g. via Cursor's `@Files` or VS Code's context pinning).

### OpenAI Codex

Place `SKILL.md` under `~/.codex/skills/peec-ai-mcp/SKILL.md` if you're running a Codex version that supports skill files (December 2025+), or reference the content explicitly in your prompt preamble.

### n8n

n8n doesn't have a skill concept. Include the relevant sections of `SKILL.md` in the system prompt of any workflow that calls Peec MCP — especially §7 (data-literacy gotchas).

### Connecting the MCP server itself

Installing the skill doesn't connect your agent to Peec — you still need to add the Peec MCP server to your client. Follow Peec's official per-client setup instructions at [docs.peec.ai/mcp/introduction](https://docs.peec.ai/mcp/introduction).

## Why this exists

The Peec MCP server exposes a rich 27-tool surface, but the official documentation (a) doesn't explain several critical data-interpretation subtleties that lead agents to produce confidently wrong analysis, and (b) doesn't cover all client apps. This skill fills those gaps.

The goal is that any agent — Claude, Cursor, Codex, n8n, whatever comes next — loads this skill and gets behaviour grounded in how Peec actually responds, not how the docs say it should.

## Contributing

Found a discrepancy? Hit a bug? Discovered a new tool, parameter, or client app that works? Open an [issue](https://github.com/rebelytics/peec-ai-mcp/issues) or a PR.

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for what's in scope and how to submit changes.

## Credits

- Author: [Eoghan Henn](https://rebelytics.com)
- Not affiliated with [Peec AI](https://peec.ai). They make the product; this skill is an independent guide.

## License

CC BY 4.0 — see [`LICENSE`](./LICENSE). Reuse, adapt, redistribute. Keep the attribution.
