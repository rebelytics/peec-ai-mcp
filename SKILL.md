---
name: peec-ai-mcp
description: Companion skill for the Peec AI MCP server (https://api.peec.ai/mcp). Load when the user does Peec reporting, analysis, or multi-step work — visibility reports, per-engine comparisons, competitive gap analysis, source-authority audits, project tune-ups (brands, prompts, topics, tags), or Peec slash commands (`peec_weekly_pulse`, `peec_competitor_radar`, `peec_engine_scorecard`, `peec_topic_heatmap`, `peec_prompt_grader`, `peec_source_authority`, `peec_campaign_tracker`). Also load for Peec data interpretation (sentiment, position, visibility, share of voice, retrieval vs citation, `get_actions` two-step workflow, `list_prompts.volume` ordinals, `get_url_content` 5-day refresh cadence) or when combining two or more Peec tools. Skip for trivial single-tool lookups like `list_projects`, `list_brands`, `list_topics` where Peec's own tool descriptions suffice. Teaches agents the real behaviour of the Peec MCP, including gotchas the official docs omit or get wrong.
version: 2.0.0
license: CC-BY-4.0
origin: https://github.com/rebelytics/peec-ai-mcp
maintainer: Eoghan Henn / rebelytics (eoghan@rebelytics.com)
---

# Peec AI MCP Companion Skill

Open-source guidance for AI agents working with the Peec AI MCP server. Agent-agnostic, project-agnostic, CC BY 4.0. Reshare and adapt freely; keep the attribution line.

> Living document. If you discover behaviour that contradicts this file — or a new Peec feature that isn't covered — open an issue or PR at [github.com/rebelytics/peec-ai-mcp](https://github.com/rebelytics/peec-ai-mcp). See `CONTRIBUTING.md` for the workflow.
>
> Behaviour observations here were verified against the live server, with the fastest-drifting areas (fanout engine coverage, pagination caps, model catalogue) re-verified since. Peec iterates; things drift.

---

## 1. What Peec AI MCP is (for the agent)

Peec AI (peec.ai) monitors how brands appear across AI search engines (ChatGPT, AI Overviews, Perplexity, Gemini, Grok, Copilot, Claude, etc.) by running tracked prompts daily and analysing the responses. The MCP server exposes that data — and the ability to mutate the underlying configuration — to any MCP-capable agent.

Server URL: `https://api.peec.ai/mcp`
Transport: Streamable HTTP
Auth: OAuth 2.0 (browser consent, token persists)

The surface is **27 tools** (15 read-only, 8 write, 4 destructive), plus **7 slash-command "prompts"** that bundle pre-canned analyses. (Drift note: the server has since started announcing a 16th read-only tool, `list_model_channels` — see §3 — bringing the live surface to 28; trust `tools/list` on connection over any count in prose.) Peec's own `/mcp/tools` reference page now enumerates all 27 tools correctly (15 read + 12 write, where Peec groups `create_*`/`update_*`/`delete_*` under a single "write" heading); earlier versions of this skill flagged the docs as incomplete, but that's since been corrected upstream. This skill's value is in the data-interpretation subtleties and behavioural gotchas §7 catalogues, not in filling a missing tool list. Write-operation consent and verification patterns live in §7.11.

### Glossary of core terms

Terms used throughout this skill. Each carries a specific, non-obvious meaning in Peec's data model.

- **Visibility** — how often a brand appears in AI responses. `mentions ÷ total tracked chats`, returned as a 0–1 ratio (UI shows 0–100). See §7.3 / §7.37.
- **Share of Voice (SoV)** — a brand's share of all mentions across the tracked brand roster. 0–1 ratio (UI shows 0–100). "Tracked" is load-bearing — see **position**.
- **Sentiment** — 0–100 score, neutral at 50. Formula: `50 + (sentiment_sum / sentiment_count) × 50`. See §7.3.
- **Position** — average rank **among tracked brands**, not overall. Lower = better. See §7.4.
- **Own brand** — the `is_own=true` row in `list_brands`. Every other row is a **tracked competitor**.
- **Mention** — the brand name appears in the AI response text, with or without a web fetch.
- **Retrieval** — the AI engine fetched one or more URLs while answering. Counted in `get_domain_report` and `get_url_report`.
- **Citation** — a retrieved URL the model *explicitly references* in its response. All citations are retrievals; not all retrievals become citations.
- **Parametric memory** — the model answered from training data without fetching web sources. `sources: []` on a chat payload is the tell.
- **Fanout** — when an AI engine rewrites the user's prompt into sub-queries before searching. Exposed via `list_search_queries(chat_id=...)`.
- **Scraper variant** — a model ID suffixed `-scraper` (e.g. `chatgpt-scraper`). Replays prompts through the real consumer app; captures what users see, not raw API output. Prefer for visibility monitoring. See §7.2.
- **UGC** — user-generated content. One of 8 values of `domain_report.classification`, covering Reddit, YouTube, forums, etc. See §7.30.
- **Idempotent** — safe to run twice. Peec's soft-deletes are **not** idempotent — the second call returns "not found". See §7.34.
- **Soft-delete** — removes the entity from `list_*` and filters but preserves it server-side. The only form Peec exposes. See §7.34.
- **ID prefixes** — `or_` (project), `kw_` (brand — legacy name from Peec's keyword-tracking origins), `pr_` (prompt), `tp_` (topic), `tg_` (tag), `ch_` (chat). Opaque namespace markers, not semantic types (`kw_` is a brand, not a keyword).

### 1.1 Data model — brands, prompts, topics, tags are orthogonal

The single most misread part of Peec's surface. Prompts do not "belong to" brands — no `brand_id` foreign key exists, and `update_prompt` cannot move a prompt between brands.

**Entities:**

- **Project** (`or_…`) — root container; everything else hangs off a project.
- **Brand** (`kw_…`) — a *detection pattern* applied against AI response text. Fields: `name`, `aliases`, `regex`, `domains`, `is_own`. No link to prompts, topics, or tags.
- **Topic** (`tp_…`) — folder-like grouping for prompts. A prompt has zero or one topic.
- **Tag** (`tg_…`) — label attached to prompts. A prompt carries zero or more tags.
- **Prompt** (`pr_…`) — a question tracked daily. Fields: `text`, `country_code`, `topic_id`, `tag_ids`. **No `brand_id` field.**
- **Chat** (`ch_…`) — one AI-engine response to one prompt on one day. Chat text is scanned against every brand's detection pattern, producing mention counts and brand reports.

**The brand ↔ prompt relationship is read-only and indirect.** Prompts generate chats; chats contain response text; brand detection runs against that text at ingestion. No write path links a prompt to a brand. "Assigning" or "moving" prompts between brands is a category error — brands aren't containers, they're detection patterns.

**Practical implications:**

- `update_prompt` mutates only `topic_id` and `tag_ids` — not `text`, not any notional `brand_id`. See §7.13.
- `update_brand` mutates detection patterns (`name`, `aliases`, `regex`, `domains`). None reference prompts.
- For "which prompts mentioned Brand X", run `list_chats(brand_id=…)` and inspect the `prompt_id` column.
- `create_brand` defaults `is_own=false` — it always creates a competitor. Own-brand election is not exposed via MCP.
- The **own brand's TLD list matters** for classification. If `domains=["example.de"]` but the company also operates `example.com`, the domain report classifies `example.com` as `CORPORATE`, not `OWN`. See §7.10.

---

## Quick Start — First Report in 3 Minutes

Assumes a Peec AI account (peec.ai) with at least one project, and an MCP-capable client (Claude Desktop used in the example below). Peec publishes official per-client setup instructions at [docs.peec.ai/mcp/introduction](https://docs.peec.ai/mcp/introduction); follow those and come back here once the connector is live.

1. **Connect.** Settings → Connectors → Add custom connector → URL `https://api.peec.ai/mcp` → Save → **click "Connect"** on the connector card → authorise in browser. The "Connect" click after saving is easy to miss.
2. **Verify.** Ask the agent: *"List my Peec AI projects."* Expect at least one `or_…` project ID in a columnar JSON table (§4).
3. **Pull a first brand report.** Ask: *"For project `<id>`, show my own brand's visibility, mentions, SoV, sentiment, and position over the last 30 days, broken down by AI model."* The agent will chain `list_brands` (find `is_own=true`) → `get_brand_report` (filter on that `brand_id`, dimension `model_id`, default 30-day window).
4. **Three interpretation rules before you report any numbers:**
   - **Scales are mixed in the same row.** `visibility` and `share_of_voice` come back as **0–1 ratios** (multiply by 100 for percent). `sentiment` is already **0–100** (neutral = 50). `position` starts at **1** and lower is better. Don't treat all four as one scale. Details in §7.37.
   - **Position is rank among *tracked* brands, not overall.** A position of 1.6 means "1.6th in the competitor roster you configured", not "2nd overall in ChatGPT's full response". §7.4.
   - **Mentions include parametric answers.** Visibility counts every response where the brand name appears in text, including answers where the engine didn't fetch a single URL. Retrievals and citations count only real web fetches. §7.5.
5. **Empty data ≠ broken tool.** Engines returning zeros are usually inactive on your plan (§7.1). Seven distinct causes of empty results are catalogued in §7.8 — check that list before concluding "the tool is broken".

That's the minimum useful loop. For the full feature map, keep reading — §8 collects the seven recipes that cover Peec's published primary use cases.

---

## Pre-flight checklist — before you report any numbers

Run through this twelve-item check before putting Peec figures in front of a human.

1. **Metric type identified for every column.** Classify each numeric column as Ratio (0–1, multiply by 100), Score (0–100, neutral at 50 for sentiment), Rank (1+, lower is better), Rate (can exceed 1.0), or Count (integer). See §7.37. If you can't name the type, don't display the number.
2. **Scale normalised to what the human expects.** Ratios rendered as percentages (`0.33` → `33%`, not `0.33%`). Sentiment left as 0–100 with `50 = neutral` explicit. Position left as a rank with "lower is better" called out. Rate kept as-is, with anomalies (`retrieval_rate=1.8`) explained rather than rewritten. See §7.3 / §7.37.
3. **Position read as rank-among-tracked-brands, not overall.** Any position figure comes with the "among tracked competitors" qualifier. See §7.4.
4. **Dimensions and filters validated against the tool schema, not guessed.** `get_brand_report` valid dimensions: `prompt_id, model_id, model_channel_id, tag_id, topic_id, date, country_code, chat_id`. `brand_id` is filter-only, never a dimension. See §8.1.
5. **Column names confirmed against the actual response payload.** Don't assume a column exists because a recipe says to sort on it. Default undimensioned `get_domain_report` returns `retrieved_percentage`, `retrieval_rate`, `citation_rate` — not `retrieval_count`. See §7.39 / §8.10.
6. **Inactive engines flagged, not reported as zero visibility.** Check `list_models(is_active=true)` before claiming a brand is invisible on Perplexity/Claude/Gemini. See §7.1 / §7.8.
7. **Empty results diagnosed against the seven-cause list in §7.8** before concluding anything is broken. Soft-deleted brands, inactive engines, parametric answers, plan limits, filter-mismatch, engine-returned empty-response bodies, and regulated-vertical content-policy refusals all look the same on the surface.
8. **Unresolved `kw_…` IDs flagged as soft-deleted**, not as bugs. See §7.38.
9. **Engine-returned empty-response chats flagged, not silently absorbed into non-mention aggregates.** Where the frequency is non-trivial, filter them out of visibility/SoV denominators or report them as a separate "engine no-answer" rate. See §7.8 cause 6 and §7.37 caveat.
10. **Dimension labels confirmed present AND row count sanity-checked before reporting per-dimension breakdowns.** `get_brand_report` with a dimension set has two early-window failure modes: null-label rows (~24h post-write-wave, correct row count with `null` in the dimension column) and zero-rows entirely (~48h post-write-wave, observed on dimensioned queries combined with a `brand_id` filter). Cross-reference the row count against `list_models(is_active=true)` / `list_topics` / `list_tags`, verify the dimension column is populated, and if either check fails use the per-filter workaround (separate call per engine/topic/tag) or the tag-filter-without-dimension pattern (§7.40 workaround 5) before attributing metrics.
11. **Fanout data scope confirmed per engine before drawing cross-engine conclusions.** `list_search_queries` returns zero rows for AI Overview (`google-0`), AI Mode (`google-1`), and Copilot (`microsoft-0`); ChatGPT (`openai-0`) and Grok (`xai-0`) are confirmed to return fanout. Other engines haven't been tested — verify empirically before relying on fanout for any engine not in that list. Don't report "what the AI searches for" as an engine-agnostic signal. See §7.41.
12. **Write-time metric-type re-check before any number is typeset.** Items 1–2 fire when the number is first read from Peec. This item fires again when the number is typeset into a slide, email, or report — the point at which metric-type context most often evaporates. Before any Peec `position`, `visibility`, `share_of_voice`, `sentiment`, `retrieval_rate`, or `citation_rate` value is placed in a deliverable, re-read §7.37 and confirm the displayed framing matches the metric's scale. For `position` specifically: if the rendered presentation invites a higher-is-better reading — e.g., values ordered ascending without directional context, a "2.0 vs 2.8" comparison that reads like a score, or bold raw numerals without a "lowest = best" footnote — either flip the presentation (use ordinal labels `#1` / `#2` / `#3`, add a "lowest = best" caption) or replace the raw number with an ordinal. The same discipline applies to sentiment (`50 = neutral` must be explicit; a brand at 62 is not "62% positive"), and to `retrieval_rate` / `citation_rate` values that exceed 1.0 (which break a naïve percentage reading — see §7.37). This check has teeth because it names the specific failure pattern: the data is correct, the interpretation is inverted, and both look identical on the slide. See §7.37, §7.4.

If any item fails, stop and fix before reporting. Partial passes produce partial trust.

---

## 2. Setup

**Connection parameters (same for every client):**

- URL: `https://api.peec.ai/mcp`
- Transport: Streamable HTTP
- Auth: OAuth 2.0 (browser consent, token persists)
- Scope: read + write (12 write tools — see §7.11 for consent and verification patterns)

**For per-client install instructions, follow Peec's official setup docs at [docs.peec.ai/mcp/introduction](https://docs.peec.ai/mcp/introduction).** Peec maintains the up-to-date list of supported clients and their step-by-step connection flows; this skill deliberately doesn't duplicate that content because it's environment-dependent and gets stale quickly.

**Network sandboxing:** If you're running inside a sandboxed agent environment (Cowork, certain cloud IDEs, corporate proxies), make sure `api.peec.ai` is on the outbound allowlist. A blocked request surfaces as a generic connection / 403 error with no in-skill signpost that it's a network-layer issue rather than a Peec bug.

**Verify your setup** with: *"List my Peec AI projects, then show me the tracked brands and topics in the first one."* Expected: two sequential tool calls (`list_projects` → `list_brands` + `list_topics`), both returning columnar JSON with at least one row each. If the agent says it has no Peec integration, the connector isn't connected — revisit Peec's official docs and confirm the OAuth consent completed.

Before connecting, make sure the user has a Peec AI account (peec.ai) with at least one project and is logged in in the same browser they'll use for OAuth. OAuth "silently completes" if the browser session is already authenticated — so the absence of a consent screen does not mean the connector is broken.

---


## Section map — where to read what

This skill uses progressive disclosure: this file holds the data model,
the quick start, the pre-flight checklist, and setup; the detailed
surface documentation lives in `references/` and is loaded when its
territory comes up. Section numbers are global — a cross-reference like
"§7.25" resolves via this map. The load triggers are mandatory: calling
a tool without having read the tool-surface file, or reporting numbers
without checking §7's catalogue, is how the confidently-wrong outputs
this skill exists to prevent get produced.

| File | Sections | Load trigger |
|---|---|---|
| `references/tools-and-responses.md` | §3–§5 | Before your first Peec tool call of the session — tools, response format, pagination, slash commands |
| `references/power-features.md` | §6 | When a task needs gap filters, brand regex, `get_actions`, wave-based bulk writes, or scraped content |
| `references/gotchas.md` | §7 | Contents list before reporting any numbers; full entries for every tool/field in your workflow |
| `references/recipes.md` | §8 | When running a composite analysis (visibility report, engine comparison, gap analysis, source audit, …) |

## 9. Contributing

Spotted a discrepancy between this skill and Peec's actual behaviour? Discovered a new tool, parameter, or slash-command prompt? Hit a setup quirk in a client not yet covered? Please contribute back:

- **Issues and PRs:** [github.com/rebelytics/peec-ai-mcp](https://github.com/rebelytics/peec-ai-mcp)
- **Workflow:** See `CONTRIBUTING.md` in this repo for what's in scope and how to submit changes.

The goal is that anyone pulling this skill six months from now gets behaviour grounded in current reality, not a stale snapshot. Contributions are the mechanism that keeps that promise.

---

## 10. License & attribution

**License:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Reuse, adapt, redistribute — just keep attribution.

**Attribution:**
> Peec AI MCP Companion Skill, maintained by Eoghan Henn (rebelytics.com), github.com/rebelytics/peec-ai-mcp.

**Not affiliated with Peec AI.** Peec's team has not reviewed or endorsed this skill.
