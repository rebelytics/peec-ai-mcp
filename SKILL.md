---
name: peec-ai-mcp
description: Companion skill for the Peec AI MCP server (https://api.peec.ai/mcp). Load when the user does Peec reporting, analysis, or multi-step work — visibility reports, per-engine comparisons, competitive gap analysis, source-authority audits, project tune-ups (brands, prompts, topics, tags), or Peec slash commands (`peec_weekly_pulse`, `peec_competitor_radar`, `peec_engine_scorecard`, `peec_topic_heatmap`, `peec_prompt_grader`, `peec_source_authority`, `peec_campaign_tracker`). Also load for Peec data interpretation (sentiment, position, visibility, share of voice, retrieval vs citation, `get_actions` two-step workflow, `list_prompts.volume` ordinals, `get_url_content` 5-day refresh cadence) or when combining two or more Peec tools. Skip for trivial single-tool lookups like `list_projects`, `list_brands`, `list_topics` where Peec's own tool descriptions suffice. Teaches agents the real behaviour of the Peec MCP, including gotchas the official docs omit or get wrong.
version: 1.3.0
license: CC-BY-4.0
origin: https://github.com/rebelytics/peec-ai-mcp
maintainer: Eoghan Henn / rebelytics (eoghan@rebelytics.com)
---

# Peec AI MCP Companion Skill

Open-source guidance for AI agents working with the Peec AI MCP server. Agent-agnostic, project-agnostic, CC BY 4.0. Reshare and adapt freely; keep the attribution line.

> Living document. If you discover behaviour that contradicts this file — or a new Peec feature that isn't covered — open an issue or PR at [github.com/rebelytics/peec-ai-mcp](https://github.com/rebelytics/peec-ai-mcp). See `CONTRIBUTING.md` for the workflow.
>
> Behaviour observations here are current as of April 2026. Peec iterates; things drift.

---

## 1. What Peec AI MCP is (for the agent)

Peec AI (peec.ai) monitors how brands appear across AI search engines (ChatGPT, AI Overviews, Perplexity, Gemini, Grok, Copilot, Claude, etc.) by running tracked prompts daily and analysing the responses. The MCP server exposes that data — and the ability to mutate the underlying configuration — to any MCP-capable agent.

Server URL: `https://api.peec.ai/mcp`
Transport: Streamable HTTP
Auth: OAuth 2.0 (browser consent, token persists)

The surface is **27 tools** (15 read-only, 8 write, 4 destructive), plus **7 slash-command "prompts"** that bundle pre-canned analyses. Peec's own `/mcp/tools` reference page now enumerates all 27 tools correctly (15 read + 12 write, where Peec groups `create_*`/`update_*`/`delete_*` under a single "write" heading); earlier versions of this skill flagged the docs as incomplete, but that's been corrected upstream as of April 2026. This skill's value is in the data-interpretation subtleties and behavioural gotchas §7 catalogues, not in filling a missing tool list. Write-operation consent and verification patterns live in §7.11.

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

Run through this eight-item check before putting Peec figures in front of a human.

1. **Metric type identified for every column.** Classify each numeric column as Ratio (0–1, multiply by 100), Score (0–100, neutral at 50 for sentiment), Rank (1+, lower is better), Rate (can exceed 1.0), or Count (integer). See §7.37. If you can't name the type, don't display the number.
2. **Scale normalised to what the human expects.** Ratios rendered as percentages (`0.33` → `33%`, not `0.33%`). Sentiment left as 0–100 with `50 = neutral` explicit. Position left as a rank with "lower is better" called out. Rate kept as-is, with anomalies (`retrieval_rate=1.8`) explained rather than rewritten. See §7.3 / §7.37.
3. **Position read as rank-among-tracked-brands, not overall.** Any position figure comes with the "among tracked competitors" qualifier. See §7.4.
4. **Dimensions and filters validated against the tool schema, not guessed.** `get_brand_report` valid dimensions: `prompt_id, model_id, model_channel_id, tag_id, topic_id, date, country_code, chat_id`. `brand_id` is filter-only, never a dimension. See §7.15 / §8.1.
5. **Column names confirmed against the actual response payload.** Don't assume a column exists because a recipe says to sort on it. Default undimensioned `get_domain_report` returns `retrieved_percentage`, `retrieval_rate`, `citation_rate` — not `retrieval_count`. See §7.39 / §8.10.
6. **Inactive engines flagged, not reported as zero visibility.** Check `list_models(is_active=true)` before claiming a brand is invisible on Perplexity/Claude/Gemini. See §7.1 / §7.8.
7. **Empty results diagnosed against the seven-cause list in §7.8** before concluding anything is broken. Soft-deleted brands, inactive engines, parametric answers, plan limits, filter-mismatch, engine-returned empty-response bodies, and regulated-vertical content-policy refusals all look the same on the surface.
8. **Unresolved `kw_…` IDs flagged as soft-deleted**, not as bugs. See §7.38.
9. **Engine-returned empty-response chats flagged, not silently absorbed into non-mention aggregates.** Where the frequency is non-trivial, filter them out of visibility/SoV denominators or report them as a separate "engine no-answer" rate. See §7.8 cause 6 and §7.37 caveat.
10. **Dimension labels confirmed present AND row count sanity-checked before reporting per-dimension breakdowns.** `get_brand_report` with a dimension set has two early-window failure modes: null-label rows (~24h post-write-wave, correct row count with `null` in the dimension column) and zero-rows entirely (~48h post-write-wave, observed on dimensioned queries combined with a `brand_id` filter). Cross-reference the row count against `list_models(is_active=true)` / `list_topics` / `list_tags`, verify the dimension column is populated, and if either check fails use the per-filter workaround (separate call per engine/topic/tag) or the tag-filter-without-dimension pattern (§7.40 workaround 5) before attributing metrics.
11. **Fanout data scope confirmed per engine before drawing cross-engine conclusions.** `list_search_queries` returns zero rows for AI Overview (`google-0`), AI Mode (`google-1`), and Copilot (`microsoft-0`); ChatGPT (`openai-0`) and Grok (`xai-0`) are confirmed to return fanout. Other engines haven't been tested — verify empirically before relying on fanout for any engine not in that list. Don't report "what the AI searches for" as an engine-agnostic signal. See §7.41.

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

## 3. The 27 tools at a glance

### Read-only (15)

| Tool | Purpose |
|---|---|
| `list_projects` | All projects accessible to the authenticated user |
| `list_brands` | Tracked brands in a project (own + competitors) |
| `list_topics` | Topics (folder-like groupings of prompts) |
| `list_prompts` | Prompts, filterable by topic_id or tag_id |
| `list_tags` | Cross-cutting labels applied to prompts |
| `list_models` | AI engine catalog (see §7: `is_active` filter is critical) |
| `list_chats` | Individual AI responses, filterable by brand/prompt/model |
| `get_chat` | Full chat payload (messages, sources, products, brands_mentioned) |
| `list_search_queries` | Sub-queries the AI engine fanned out to |
| `list_shopping_queries` | Shopping-mode queries + product listings |
| `get_brand_report` | Visibility, SoV, sentiment, position aggregates |
| `get_domain_report` | Source-domain retrieval + citation rates |
| `get_url_report` | Source-URL retrieval + citation rates + page classification |
| `get_url_content` | Scraped markdown of any indexed source URL |
| `get_actions` | Opportunity-scored recommendations — two-step workflow (`scope=overview` then drill down). Callable from pass-through clients despite empty declared schema; see §6.4 / §7.12 |

### Write / mutate (8)

| Tool | Purpose |
|---|---|
| `create_brand` | Track a new competitor (accepts `domains`, `aliases`, `regex`) |
| `update_brand` | Rename / adjust a brand (triggers background recalculation — see §7.19) |
| `create_topic` | Add a topic (optional `country_code`; `name.maxLength=64`) |
| `update_topic` | Rename a topic |
| `create_prompt` | Add a tracked prompt (requires `country_code`; `text.maxLength=200`) |
| `update_prompt` | Change topic_id / tag_ids (**not text** — see §7.13) |
| `create_tag` | Add a tag with one of 22 colours (see below) |
| `update_tag` | Rename or recolour a tag |

**Tag colour enum (22 values):** `gray`, `red`, `orange`, `yellow`, `lime`, `green`, `cyan`, `blue`, `purple`, `fuchsia`, `pink`, `emerald`, `amber`, `violet`, `indigo`, `teal`, `sky`, `rose`, `slate`, `zinc`, `neutral`, `stone`. The tool's JSON schema is the authoritative source if Peec adds colours later; this list matches the schema as of April 2026.

### Destructive (4)

| Tool | Purpose |
|---|---|
| `delete_brand` | Soft-delete brand |
| `delete_topic` | Soft-delete topic (detaches prompts, doesn't cascade) |
| `delete_prompt` | Soft-delete prompt (cascades to chats) |
| `delete_tag` | Soft-delete tag (detaches from prompts) |

All destructive operations are **soft-delete** — no hard-delete endpoint exposed. Soft-delete is **not idempotent from the client's perspective**: the second call on an already-deleted ID returns a "not found" error rather than a no-op success (see §7.34).

### 3.1 Write-operation quick reference

One-table summary of what each entity's `update_*` tool can actually change, plus whether the write triggers a background recalculation. Useful when planning a tune-up without cross-referencing five different §7 entries.

| Entity | Mutable fields via MCP | Triggers recalc? | Cost | Relevant §§ |
|---|---|---|---|---|
| Brand (`kw_…`) | `name`, `domains`, `aliases`, `regex` | Yes — on `name`, `regex`, `aliases` changes (409 on concurrent writes) | No plan credits | §7.19, §7.21 |
| Topic (`tp_…`) | `name` | No | No plan credits | §7.20 |
| Tag (`tg_…`) | `name`, `color` | No | No plan credits | §7.35, §7.36 |
| Prompt (`pr_…`) | `topic_id`, `tag_ids` (full replace) | No (but affects tag-filtered reports immediately) | **`create_prompt` consumes plan credits** — `update_prompt` does not | §7.13, §7.14, §7.17 |

**Immutable fields** (can only be changed by delete + recreate): prompt `text` (§7.13), any `is_own` assignment (create_brand defaults to false; own-brand election not exposed via MCP), tag and topic `country_code` once set.

---

## 4. Response format

Every read tool returns columnar JSON:

```json
{
  "columns": ["id", "name", "visibility", ...],
  "rows": [
    ["kw_…", "Example Brand", 0.14, ...],
    ...
  ],
  "rowCount": 8,
  "total": 8   // optional, present on some report tools
}
```

Helpers:
- Row is an array of values in column order. Zip to get a dict.
- `rowCount` is the page size, not the grand total. Use `total` when present.
- **Write return shapes:**
  - `create_*` tools return `{id: "…"}` — the new entity's ID, nothing else.
  - `update_*` and `delete_*` tools return `{success: true}` — no echo of the updated record, no diff.
  - Either way, call `list_*` afterwards if you need to verify state.
- **Plan credits — only `create_prompt` charges.** Per the tool's own description, creating a prompt consumes plan credits (prompts are the billable unit in Peec's pricing; every tracked prompt runs daily across the enabled engines). `create_brand`, `create_topic`, `create_tag`, and all `update_*`/`delete_*` tools do **not** consume plan credits. Before bulk-creating prompts on a TRIAL or small paid plan, check the remaining credit balance in the Peec UI — there's no `get_credit_balance` MCP endpoint. Failed `create_prompt` calls from credit exhaustion return a billing-shaped error rather than a schema error.
- **Scales are heterogeneous within a single row.** `get_brand_report` returns four headline metrics on three different scales: `visibility` and `share_of_voice` as 0–1 ratios, `sentiment` as 0–100, `position` starting at 1 (lower = better). The columnar JSON envelope gives no hint about this. Read §7.37 before building any report layer.

---

## 5. Slash-command prompts (Claude Desktop `/`, Cursor `@`)

These are server-side "prompts" in MCP terminology — canned multi-tool analyses with sensible defaults.

| Slash command | What it does | Arguments |
|---|---|---|
| `/peec_weekly_pulse` | Week-over-week digest: brands, competitors, sources, sentiment | `project` |
| `/peec_competitor_radar` | Flags competitors moving by more than a threshold | `project`, `threshold_pp` (default 10) |
| `/peec_engine_scorecard` | Per-engine breakdown of visibility, SoV, sentiment, position | `project` |
| `/peec_topic_heatmap` | Visibility × topic × engine with severity bands | `project` |
| `/peec_prompt_grader` | Grades your prompt set on balance, tag hygiene, funnel, duplicates | `project` |
| `/peec_source_authority` | Domain-as-source audit: retrieval, citation, authority gaps | `project` |
| `/peec_campaign_tracker` | Before/after comparison around a campaign date | `project`, `campaign_date` (YYYY-MM-DD), optional `urls` |

When the user asks "what's happening this week" or "give me a visibility digest", reach for `/peec_weekly_pulse` before hand-rolling queries.

**Not available in:** OpenAI Codex, n8n, most non-Claude/non-Cursor clients. Fall back to composing the equivalent tool calls manually.

---

## 6. Hidden features & power moves

### 6.1 Gap filter on domain/URL reports

The `filters` param on `get_domain_report` and `get_url_report` accepts a special `gap` operator not shown in the visible schema hints:

```json
{"field": "gap", "operator": "gte", "value": 1}
```

Returns domains/URLs where **competitors appear but the user's own brand doesn't**. Essential for competitive content audits.

### 6.2 `mentioned_brand_count` filter

```json
{"field": "mentioned_brand_count", "operator": "gte", "value": 2}
```

Finds sources where multiple tracked brands co-appear — useful for competitive co-citation patterns ("who's being compared to us?").

### 6.3 `regex` on `create_brand` / `update_brand`

When you're tracking a brand with complex naming variants (e.g. three word variations of the same name), pass a regex alongside `aliases`:

```json
{"name": "Example", "aliases": ["Example Co", "Ex."], "regex": "\\bEx(?:ample(?:\\s+Co)?|\\.)\\b"}
```

On `update_brand`, pass `regex: null` to clear an existing regex. The base Peec docs at `/identifying-your-competitors` describe aliases and regex at the brand-UI level but not specifically in the MCP tool context — this skill documents the MCP-side behaviour.

### 6.4 Two-step `get_actions` workflow

Always call with `scope=overview` first — it returns *navigation metadata*, not recommendations. Then drill down per slice:

- `scope=owned` (no extras)
- `scope=editorial` + `url_classification` (e.g. "LISTICLE")
- `scope=reference` + `domain` (e.g. "wikipedia.org")
- `scope=ugc` + `domain` (e.g. "reddit.com", "youtube.com")

As of April 2026 this workflow is reliably callable from Cowork and other pass-through MCP clients despite the tool's declared JSON schema still being empty. Schema-strict clients strip the `scope` parameter before sending — on those, the server rejects with *"No matching discriminator: scope"*. See §7.12 for the full behavioural matrix and when to fall back to the local approximation recipe in §8.8.

### 6.5 Wave-based execution for bulk config changes

When a tune-up touches many entities (tags, brands, prompts), execute in three waves rather than one sprint:

1. **Wave 1 — additive:** `create_tag`, `create_brand`. Zero risk; reversible by deletion. Confirm new entities appear in Peec UI before moving on.
2. **Wave 2 — mutation:** `update_prompt`, `update_topic`, `delete_brand`. Changes existing state. Pause to re-check UI.
3. **Wave 3 — creation:** `create_prompt`. Final bulk add. Resolve any new tag IDs from Wave 1 and substitute into the `tag_ids` array before calling.

Between waves, run the appropriate `list_*` tool to verify state. Use this pattern for any tune-up of more than ~20 operations.

**Intra-wave parallelism is safe for independent writes to different entities.** Within a single wave, operations that don't reference each other's outputs *and* don't target the same entity (e.g. 10 independent `create_tag` calls, or a batch of `create_prompt` calls whose tag IDs were resolved upstream) can be issued in parallel. Batches of 5–10 parallel MCP calls stay well under the published rate limit of **200 requests/minute per project** (see §7.24 and the official `/ratelimits` docs page). For inter-wave ordering, keep the strict sequence; for intra-wave independent writes to different entities, batch freely. **Do not parallelise multiple writes to the same entity** — especially `update_brand` calls against the same brand ID, which triggers a background recalculation and rejects concurrent updates with 409 (see §7.19).

**Pre-fetch tag and topic IDs once per wave.** When a wave will issue many writes that reference the same set of tag or topic IDs (common in Wave 3 prompt creates), call `list_tags` and `list_topics` once at the start of the wave and hold the ID map in-memory for the whole batch. Inter-wave refetches are only required if an intervening wave created new tags/topics.

The companion `peec-ai-project-tuneup` skill codifies this into a full methodology — load it when the user wants to overhaul a Peec project, not just query it.

### 6.6 Scraped content via `get_url_content`

After `get_url_report`, feed interesting URLs into `get_url_content` to pull the actual markdown the AI engine was reading. Pass the URL verbatim — trailing slashes and scheme changes break the lookup.

`get_url_content` has **two distinct failure modes** that an agent has to dispatch on:

- **Failure mode A — URL not indexed by Peec at all:** the call returns an error response with the text `"URL not found"`. This is a hard miss — no retry helps. Skip and continue.
- **Failure mode B — URL indexed but not yet scraped:** the call returns a success envelope with `content: null`. Scraping runs up to 24h after first encounter. A retry later in the day often succeeds — queue for a deferred re-run rather than skipping.

An agent looping over `get_url_report` → `get_url_content` must branch on these two cases, not conflate them. The error-vs-null distinction is the reliable signal.

**Response payload fields (April 2026).** A successful `get_url_content` response carries, in addition to the markdown body:

- `content_updated_at` — ISO timestamp of the last scrape for this URL. Use this, not "now", as the as-of date when quoting the page back to a user.
- `classification` — **domain-level** classification of the source (same enum as §7.30: `CORPORATE`, `EDITORIAL`, `INSTITUTIONAL`, `UGC`, `REFERENCE`, `COMPETITOR`, `OWN`, `OTHER`).
- `url_classification` — **page-level** classification of this specific URL (same enum as §7.31: `HOMEPAGE`, `CATEGORY_PAGE`, `PRODUCT_PAGE`, `LISTICLE`, `COMPARISON`, `PROFILE`, `DISCUSSION`, `HOW_TO_GUIDE`, `ARTICLE`, `OTHER`, `ALTERNATIVE`).

The dual classification matters: a page can be `EDITORIAL` at the domain level and `COMPARISON` at the page level, and the two enums never overlap. Don't conflate them; they answer different questions (who publishes this vs. what kind of page is it).

**5-day refresh cadence.** Source-URL page content is re-scraped every ~5 days, **not daily** (confirmed by Peec staff in the MCP challenge Slack channel and verified empirically across 8 URL samples in April 2026 — `content_updated_at` values cluster into 5-day buckets). Implications:

- For time-sensitive analysis (e.g. "what is this page saying right now?"), the content may be up to 5 days stale. Surface `content_updated_at` alongside any quoted content so the user knows the as-of date.
- Re-running `get_url_content` more frequently than every 5 days returns the same payload — don't build re-fetch loops tighter than that.
- A URL that returned `content: null` yesterday may return populated content today (first-encounter scrape can happen on any day); but once scraped, subsequent scrapes only happen on the 5-day cadence.

---

## 7. Data-literacy gotchas (READ THIS BEFORE REPORTING)

These are things the official documentation either omits or gets wrong. Ignoring them produces confident-sounding but materially wrong analysis.

### 7.1 `list_models` returns 16 engines (schema `model_id` enum is 19); only `is_active: true` are tracked

The MCP's `model_id` filter/dimension enum on report tools lists 19 values as of April 2026 (adds `claude-haiku-4.5`, `claude-sonnet-4`, `grok-4`, `google-ai-mode-scraper`, `google-ai-overview-scraper`, `microsoft-copilot-scraper` on top of the legacy set). `list_models` returns 16 of these — the 3 omitted are engines that exist in the enum but aren't yet surfaced through the listing tool. Filter on `is_active: true` before building engine breakdowns. On lower-tier plans users select a subset of engines; the others return empty data. On a TRIAL-tier project, `is_active: true` typically holds for only 3 engines out of 16. Higher tiers unlock more. Empty results for an inactive model look identical to "no data exists" — there's no error.

**Practical implication.** If you build a report from the `model_id` filter enum (19 values) instead of from `list_models` (16 values), three of those engines will return clean empty envelopes regardless of plan tier — treat them the same way you'd treat any other inactive engine: skip, don't report as "zero visibility". If the user asks about one of the three non-listed engines (`claude-sonnet-4`, `claude-haiku-4.5`, `grok-4` on most projects), tell them it's not available via the MCP's listing surface even though the enum accepts it.

### 7.2 `*-scraper` models measure consumer behaviour; raw model IDs measure API responses

`chatgpt-scraper` ≠ `gpt-4o`. The scraper variants replay queries through the actual consumer app (ChatGPT web UI, Grok web, AI Overview), capturing what users actually see. Raw model IDs (`gpt-4o`, `claude-sonnet-4`, `grok-4`) hit the model API directly — different results, different retrievals, different utility. For visibility monitoring, prefer scraper variants.

### 7.3 Sentiment formula is not `sentiment_sum / sentiment_count`

The doc hints that the "raw aggregation fields" let you do custom math. They don't — at least, not naively. Observed relationship:

```
sentiment = 50 + (sentiment_sum / sentiment_count) × 50
         = ((sentiment_sum / sentiment_count) + 1) / 2 × 100   # equivalent form
```

Where `sentiment_sum / sentiment_count` is on a -1..+1 scale (neutral = 0 → mid-50). If you compute the ratio directly and report it as a 0–100 sentiment score, you'll be off by a factor of 50 and you'll miss the neutral-centred axis.

Sample verification (four brands, 18-day window):

| Brand | sentiment_sum | sentiment_count | Derived | Displayed |
|---|---|---|---|---|
| Own Brand | 137.609 | 731 | 59.4 | 59 |
| Competitor A | 88.881 | 350 | 62.7 | 63 |
| Competitor B | 54.381 | 221 | 62.3 | 62 |
| Competitor C | 28.246 | 137 | 60.3 | 60 |

All four match the displayed value after rounding. Peec's docs describe sentiment as a 0–100 scale with typical scores in the 65–85 band but do not publish the formula — the formula above is the observed relationship, not an officially sanctioned expression.

**Critical caveat: `sentiment_count < mention_count`.** Not every mention carries a sentiment score. In the sample above, the own brand had 819 mentions but only 731 sentiment-scored mentions (88 mentions without a score). If you divide `sentiment_sum` by `mention_count` instead of `sentiment_count`, the result is wrong. Always use `sentiment_count` as the denominator.

### 7.4 `position` in brand reports = position among *tracked* brands, not overall position

Example: ChatGPT responds with a list of 7 recommendations — Brand A (1), Brand B (2), Brand C (3), Brand D (4), Own Brand (5), Brand F (6), Brand G (7). If only Own Brand is in `list_brands`, `position = 1`. It does **not** mean Own Brand ranked first overall.

Clients who report "position 1 in ChatGPT" based on this metric will materially misrepresent the data. Always verify with `get_chat` → inspect the actual assistant response text.

**Docs ambiguity note:** Peec's public metric docs describe position as "average ranking of the brand in AI responses", which reads as an overall-position metric. The observed MCP behaviour — and the only behaviour consistent with the data the server has access to — is that the ranking is computed across *tracked brands only*, not all brands named in the response. Treat the docs framing as marketing shorthand and the skill framing as the accurate mechanical behaviour.

### 7.5 Domain report is a *retrieval* report, not a *mention* report

`sources: []` in `get_chat` means the model answered from parametric memory. Brands can be mentioned in the response text without any source retrieval. So:

- `get_brand_report` counts text mentions (parametric or retrieved).
- `get_domain_report` counts retrievals (only when the model actually fetched URLs).

High-mention-count brands may have near-zero domain-report presence if the AI engine answers from memory — especially for well-known brands.

**Sources vs citations distinction** (from Peec's own docs): "sources" are every URL an AI model *accesses* while answering a prompt; "citations" are the subset of sources that the model *explicitly references* in the final response text. Peec's reports separate retrieval count from citation count for every domain/URL — don't conflate the two.

**Empirical-sampling rule for per-engine characterisation.** Engine behaviour is distributional, not categorical. Any per-engine characterisation that lands in a deliverable (deck claim, recommendation, strategic finding) must be grounded in a sample of `get_chat` responses drawn from the project's actual data — not from general "ChatGPT is parametric" / "AI Overview retrieves" assumptions.

Minimum sampling protocol:

- **Sample size:** at least 8 chats per engine per characterisation claim.
- **Topic diversity:** at least three distinct topics represented in the sample (avoid sampling eight chats from a single topic — engine behaviour varies by topic).
- **Record per chat:** whether `sources: []` (parametric), `sources` populated (retrieval), or the response body reads as a content-policy refusal (see §7.8 cause 7).
- **Report as a ratio, not a binary:** "4 of 8 sampled ChatGPT responses answered without fetching sources" is defensible; "ChatGPT is parametric" is not. If the sample is too small for a meaningful ratio, state the limitation in the deliverable rather than rounding up to a flat claim.

This rule applies to every engine in Peec's catalogue, including ones this skill characterises elsewhere (e.g., "AI Overview retrieves selectively" needs the same grounding). Cross-reference in `peec-ai-tracking-strategy-builder` §14: engine-behaviour claims in Phase B decks require the sampling step and the observed ratio.

### 7.6 Observed chat counts exceed `prompts × active_models × days`

Peec's `/understanding-chats` docs describe the cadence as "daily" and `/setting-up-your-prompts` adds that accepted prompts "start running immediately, joining the regular 24-hour cycle" — i.e. one run per prompt × model per day. In practice, observed chat counts for a project exceed `prompts × active_models × days` by a material factor.

Likely explanations — not conclusively verified:

- **Model channels multiply the count.** Each model (e.g. `gpt-4o`) can have multiple channels (`openai-0`, `openai-1`, etc.) representing different regional/setting variants. `list_chats` filters by `model_id`, but each model may emit several chats per day from different channels. The full `model_channel_id` enum observed on report tools as of April 2026 is: `openai-0, openai-1, openai-2, perplexity-0, perplexity-1, google-0, google-1, google-2, google-3, anthropic-0, anthropic-1, deepseek-0, meta-0, xai-0, xai-1, microsoft-0` (16 channels across 8 providers).
- **Back-fill on acceptance.** The "start running immediately" language suggests newly accepted prompts may be run multiple times shortly after acceptance to populate initial data.
- **Error retries.** Failed prompt runs may re-run on the same day without being deduped in the chat count.

Practical guidance: don't reconstruct "how many chats are expected" from simple arithmetic — it won't match. Use `list_chats` directly to get the observed count.

**Empirical anchor (TRIAL tier, April 2026).** A TRIAL-tier project with 70 prompts and 3 active engines produced **452 chats in its first 24-hour window** — 2.15× the simple prediction of 210 (70 × 3). Consistent with the model-channel multiplication hypothesis above. Use this as a rough expectation anchor when setting user expectations for day-1 chat volume on TRIAL-tier projects, but treat the multiplier as project-specific — it will vary with which engines are active (channel count differs per engine) and any acceptance back-fill in play.

### 7.7 `visibility_total` varies per dimension cell

In a multi-dimension breakdown (e.g. model × topic), `visibility_total` differs per cell. Google AI Overview in particular only triggers for some queries, so its denominator is smaller than ChatGPT's in the same topic. If you compute share-of-voice by summing across cells, normalise carefully.

### 7.8 Empty results have seven possible causes

An empty `rows: []` response — or an apparent "brand not mentioned" chat —
means one of:
1. No data actually exists for the filter.
2. The filter contains a typo or stale ID (non-existent `prompt_id`, `brand_id`, etc.).
3. The filter targets an inactive model on the user's plan.
4. **The date range is before the platform had data or in the future.** Report tools with a `start_date` that's far future or far past (e.g. `2030-01-01`, `2015-01-01`) return clean empty envelopes, not errors.
5. **The filter targets a soft-deleted entity.** Soft-deleted brands, prompts, topics, and tags still exist in the system but are excluded from filter matches. `list_chats(brand_id=<deleted>)` returns empty — identical in shape to "no activity".
6. **The engine itself returned an empty or placeholder response body** (e.g. `"No response."` as the entire chat body). The engine didn't fail to find your brand — it failed to answer. Detectable only by reading the chat payload via `get_chat`; aggregate fields treat it identically to "brand not mentioned". Distinguish from parametric-memory chats (engine answered, just didn't fetch sources) and from engine-wide outages (sister prompts from the same project/engine/day answered normally).
7. **Regulated-vertical content-policy refusal.** Certain engines (most commonly `chatgpt-scraper`, but also occasionally Claude and Gemini) produce refusal responses for regulated-category prompts — cannabis, CBD, seed, nutra, gambling, pharma, regulated finance, weapons — where the engine's content policy blocks a substantive answer. The `get_chat` response looks populated (the message body has text) but reads as "I can't help with that" or similar. Aggregate-level, it looks like a chat that produced no brand mentions; qualitatively it's a structural absence, not an evidence-based one. Indicators: short message bodies, refusal-language patterns ("can't help", "not able to provide", "recommend consulting a professional"), absence of topical content. In regulated verticals, sample at least 8 chats per engine on regulated-category prompts to estimate the refusal rate before relying on that engine's data for strategic conclusions. Filter refusals out of visibility/SoV denominators — they're neither parametric nor retrieval, they're refusal, and they mean something different strategically (parametric = model knows your brand from training; refusal = model will never mention your brand regardless of what you do).

Peec returns a clean empty array (or counts the chat as a non-mention) in all seven cases — no error, no warning. Before reporting "no data", validate filter IDs by listing first (which excludes soft-deleted entities, so a missing ID in `list_*` is itself a signal), sanity-check the date range against the platform's real coverage window, and for low-visibility chats spot-check `get_chat` payloads for empty bodies and refusal patterns.

### 7.9 Auto-selected competitors on project creation are often wrong

When a user creates a project, Peec auto-suggests competitors based on algorithmic similarity (brands that co-appear in at least two tracked prompts, per Peec's `/identifying-your-competitors` docs). In practice these skew toward **information sites** (forums, publications, wiki-likes) rather than actual commercial competitors. In multiple observed projects, Peec auto-selected content sites (reference databases, community forums, industry publications) while ignoring the actual commercial competition (direct retail rivals, marketplaces, adjacent brands) that the domain report reveals once data accumulates.

**Skill recipe:** after project creation, run `list_brands` + `get_domain_report` side-by-side. Any high-retrieval `CORPORATE`-classified domain *not* in `list_brands` is a candidate competitor. Use `create_brand` with `domains` + optional `regex` to add it.

### 7.10 `classification=COMPETITOR` in domain report ≠ commercial competitor

In `get_domain_report`, the `classification` column can be `COMPETITOR` — but this just means "domain of a tracked competitor brand", not "direct commercial rival". Don't use it to identify competitive threats; use mention overlap + domain retrieval volume instead.

**Multi-TLD brands and the `OWN` classification.** `classification=OWN` is driven by the `domains` array on the `is_own=true` brand row in `list_brands`, not by name matching or corporate-ownership inference. A brand with multiple TLDs (e.g. example.de, example.com, example.nl) will show `OWN` **only for the specific TLDs listed in that array** — all other TLDs of the same commercial entity fall back to `CORPORATE` (or whatever other classification applies). This is a project configuration completeness issue, not a classification bug. Agents auditing a domain report should check `list_brands(is_own=true).domains` first; if the array doesn't cover every TLD of the own brand, advise the user to update it via `update_brand` before drawing conclusions about "own vs. competitor" domain share.

### 7.11 Write-operation consent, verification, and safe-experimentation patterns

The MCP server exposes **12 write/destructive tools** — 8 `create_*`/`update_*` and 4 `delete_*`. Every write tool is flagged `readOnlyHint: false` in the server's tool annotations, and every `delete_*` tool is flagged `destructiveHint: true`. Clients that honour MCP annotations (Claude, Cursor, and most others) prompt for confirmation before the call runs. Writes are consequential — some are destructive, at least one is billable (`create_prompt` consumes plan credits), and `update_brand` triggers background metric recalculation with a 409-class concurrent-edit window (see §7.19). Agents should therefore:

- **Confirm with the user before any mutation — but treat batch authorisation as one consent event.** If the user says *"go ahead and create these 20 prompts and then delete these 5 tags"* as a single instruction, don't re-prompt on each individual call. If the plan changes mid-run (new entities, new destructive steps that weren't in the original batch), fall back to per-step confirmation. Rule of thumb: each batch's confirmation covers only the writes explicitly enumerated at batch approval time.
- **Prefer `list_*` for verification after a write.** Writes return either `{id: "…"}` for creates or `{success: true}` for updates/deletes — never an echo of the mutated record. Follow up with the matching `list_*` call if you need to confirm state, especially after bulk waves.
- **Gate on credit balance before bulk `create_prompt`.** `create_prompt` is the only credit-consuming call. On TRIAL or small paid plans, check the remaining balance in the Peec UI before dispatching a wave of prompt creates — there's no `get_credit_balance` endpoint, and credit-exhaustion errors look like billing-shaped rejections rather than schema errors.
- **For experimentation, create a dedicated test project in Peec** so writes don't pollute production data. Peec's soft-delete model means test clutter accumulates server-side even after "deletion" (§7.34) — isolating experimentation to a throwaway project keeps the production roster clean.

**Historical note.** Earlier versions of this skill flagged Peec's docs as incorrectly describing the MCP as "read-only". Peec has since corrected their documentation to enumerate the full write surface. The skill content here has been retained because the consent-and-verification guidance is the useful part — the "docs are wrong" framing is no longer accurate.

### 7.12 `get_actions` MCP schema is empty — the tool is still callable, but reviewers will strip params

**Status update (April 2026, re-verified):** `get_actions` is reliably callable from Cowork and other pass-through clients. Earlier versions of this skill described the tool as "broken" — that was a client-behaviour issue, not a server-side outage. The server-side two-step workflow works; the agent just has to be aware that the MCP tool's JSON schema is empty and some clients will strip parameters before sending.

**Behaviour by scope, re-confirmed April 2026:**

| Call | Status | Returns |
|---|---|---|
| `get_actions(scope=overview)` | ✓ works | Navigation metadata — counts and breakdowns by `url_classification` and `domain`, used to plan drill-down calls |
| `get_actions(scope=owned)` | ⚠︎ intermittent — has returned HTTP 422 at `POST .../get-action-owned` in otherwise-healthy sessions (April 2026) | Own-domain URL actions (no extra params required per schema) |
| `get_actions(scope=editorial, url_classification=<value>)` | ✓ works | Third-party editorial pages at that URL classification |
| `get_actions(scope=reference, domain=<value>)` | ✓ works | Reference-site (e.g. wikipedia.org) actions for that domain |
| `get_actions(scope=ugc, domain=<value>)` | ✓ works | UGC-site (e.g. reddit.com, youtube.com) actions for that domain |

The two-step workflow is: call `scope=overview` first to see which slices have data, then drill down by scope + the corresponding extra parameter. Full recipe in §6.4.

**Known client-layer caveat.** The tool's declared JSON schema is still empty (`{properties: {}, type: "object"}`). Schema-strict MCP clients strip all parameters before sending, and the server then rejects with *"No matching discriminator: scope"*. If you see that error, you're on a client that's stripping params — switch clients or use the §8.8 workaround. Cowork and other pass-through clients send the params through and the call succeeds.

**`scope=owned` 422 caveat (April 2026).** Confirmed intermittent on at least one production project: `scope=overview`, `scope=editorial`, `scope=reference`, and `scope=ugc` all returned 200 OK in the same session where `scope=owned` returned HTTP 422 at `POST .../get-action-owned`. The error surface (path exposed, no schema-mismatch details) points to a server-side issue on the owned endpoint, not client-side parameter stripping. If you hit it, the overview response already includes the OWNED opportunity score, relative-strength tier, and gap percentage per `url_classification` — that's usually enough for the strategic read without the drill-down text. For the full owned-URL gap list, fall back to §8.8's local approximation (filter `get_url_report` with the `gap` filter against the own brand's domains).

**When to use §8.8 (approximate locally) instead:**
- Running on a schema-strict client that strips params.
- Want a single composite output without chaining 4–5 scoped calls.
- Need the OWNED view on a project where `scope=owned` is returning 422 — see caveat above.

For most pass-through client sessions, prefer calling `get_actions` directly now. Report persistent `scope=owned` 422s and the empty tool schema to `support@peec.ai`.

### 7.13 `update_prompt` cannot change prompt text

Only `topic_id` and `tag_ids` are mutable. To "edit" a prompt's text, the required workflow is **delete + create**:

1. `delete_prompt(prompt_id)` — soft-deletes the prompt and **cascades to its chats** (all tracked history for that prompt is removed).
2. `create_prompt(text=..., topic_id=..., tag_ids=..., country_code=...)` — creates a fresh prompt.

Cost of the reframe: 1–N days of historical visibility data (depending on how long the prompt had been running) are lost. For a low-performing prompt this is usually worth it. For a high-performing prompt, consider whether the reframe actually needs to happen at all.

### 7.14 `update_prompt.tag_ids` is full replacement, not append

Passing `tag_ids: ["tg_new"]` replaces the entire tag set. To add a tag without losing existing ones, fetch the prompt first and merge:

```
existing = list_prompts(project_id) → find by id → read tag_ids
new_set = list(set(existing + ["tg_new"]))
update_prompt(prompt_id, tag_ids=new_set)
```

This is especially important when scripting bulk retags — an unmerged call silently strips every tag the prompt previously carried.

### 7.15 `create_prompt` has no `language` field — only `country_code`

Unlike Trakkr and Sistrix custom prompt tracking, Peec's `create_prompt` takes **only** a `country_code` (two-letter ISO, e.g. `DE`, `GB`, `US`) and the prompt text. Language is inferred from the text itself. Practical implication: when building multi-market prompt sets, you manage language at the level of the prompt text (write the German prompt in German; write the French prompt in French). There's no separate field to set.

The allowed country codes are constrained to a fixed enum of **92 ISO 3166-1 alpha-2 codes** as of April 2026, covering most of Europe, the Americas, Middle East/North Africa, and Asia-Pacific. If your target country isn't on the list, the call will fail validation.

### 7.16 `create_prompt.text` max length is 200 characters

`create_prompt` enforces `minLength: 1, maxLength: 200` on the `text` field. Budget your prompt text accordingly — long-form analytical prompts will be rejected. For very long prompts, break them into focused shorter ones.

### 7.17 `update_prompt` accepts `topic_id: null` to detach

Setting `topic_id: null` (not empty string, not omitted) removes the prompt's topic assignment. Useful when restructuring topics. To move a prompt from one topic to another, pass the new `topic_id` directly — no detach step needed.

### 7.18 Pagination: most `list_*` tools default to 100, but `list_projects` and `list_models` don't paginate at all

`list_brands`, `list_prompts`, `list_tags`, `list_topics`, `list_chats`, `list_search_queries`, `list_shopping_queries` accept `limit` (default 100, max 10000) and `offset` (default 0). The 100-item default is a **silent truncation trap** — a project with more than 100 prompts, brands, tags, or topics will have its state misread if you don't pass an explicit `limit` or chain `offset` calls.

`list_projects` and `list_models` have **no pagination parameters at all**. `list_projects` accepts only `include_inactive` (boolean); `list_models` accepts only `project_id`. These tools return the full set in a single call, so the truncation trap doesn't apply there — but if you're coding a generic paginator, special-case them.

**Two safe patterns for full-state reads on paginated tools:**

1. **Explicit large limit** — `list_prompts(project_id, limit=10000)` for any inventory task. The server caps at 10000 but that's almost always enough for a single project.
2. **Paginate until empty** — call with `offset=0, limit=100`, then `offset=100, limit=100`, … until `rowCount=0`. This is the pattern used during post-tune-up verification to prove no prompts were missed.

Both work; prefer the explicit-limit form for single-call simplicity, and the paginated form when you want an audit trail that explicitly proves completeness.

**Boundary behaviour:** `limit=0` returns a clean empty envelope rather than rejecting; `limit=1` works as expected; `limit=999999` is **silently capped** to the server's 10000 maximum without error. `limit=-1` and non-integer `limit` values are rejected client-side by the schema. The silent cap is the one to remember — an agent asking for "everything" by passing a huge number will silently get a truncated result, not an error.

### 7.19 `update_brand` triggers background metric recalculation

Changes to a brand's `name`, `regex`, or `aliases` cause Peec to reprocess all historical chats against the new matcher. The `/api-reference/project/update-brand` docs confirm: *"Changes to name, regex, or aliases trigger a background recalculation of metrics. During recalculation, further updates to these fields are blocked with a 409 Conflict response."* Practical consequences:

- If you need to update several aspects of the same brand, combine them into one `update_brand` call rather than sequential calls.
- Retries after a 409 during recalculation must wait for the recalc to finish. Back off ~30 seconds and retry.
- Aggregate metrics (visibility, SoV, mention counts) shift retroactively after recalculation completes. Don't compare pre-recalc numbers to post-recalc numbers without flagging the change in your report.

### 7.20 `create_topic` has optional `country_code` and 64-char `name` limit

`create_topic` takes `project_id` and `name` (required, `maxLength: 64`) plus an optional `country_code` from the same 92-country enum used by `create_prompt`. The `country_code` on a topic is advisory metadata (it doesn't force child prompts to inherit the country — each prompt still carries its own `country_code`).

Budget topic names tightly. 64 chars sounds like a lot but multi-word topic names in German, Spanish, or languages with long compound words hit the ceiling fast.

### 7.21 `update_brand.regex` accepts `null` to clear an existing pattern

The schema declares `regex` as `{anyOf: [{type: "string"}, {type: "null"}]}`. Passing an empty string (`""`) is not the same as null — empty string is treated as "match nothing" by the regex engine and will break brand detection. To remove a regex entirely, pass JSON `null`. To change it, pass the new pattern as a string.

### 7.22 Deprecated fields to avoid

Peec's API has sunset several fields in 2026. Don't rely on them, and if you find older tooling that does, flag for upgrade:

- `citation_avg`, `usage_count`, `usage_rate` — deprecated 2026-03-19 (v0.12.0) per Peec's API changelog; still returning but scheduled for removal. The modern equivalents are retrieval/citation counts split per entity.
- `normalizedUrl` — **removed** (not just deprecated) 2026-03-06 (v0.9.0) from the Get URLs Report endpoint. Any tooling still referencing this field will silently get `undefined`. Treat the raw `url` as canonical.

If a Peec API field appears in a response but isn't documented in the current docs, check the changelog before using it — Peec's deprecation pattern is to leave the field returning for a grace period before removing it.

### 7.23 HTTP API has features the MCP doesn't expose (yet)

Peec's HTTP API (separate from the MCP server, same platform) exposes functionality that is **not** callable via the MCP surface as of April 2026. Specifically:

- **Prompt and topic suggestions** with accept/reject endpoints — Peec can suggest prompts or topics based on project context, but these live in the HTTP API, not the MCP.
- **Model channel listing** (`list-model-channels`) — exposes finer detail about which models/channels a plan has access to.

Why this matters for the agent: do **not** hallucinate MCP tools for these capabilities. If a user asks for "prompt suggestions", the MCP has no tool for that; either direct them to the Peec UI (where the feature lives) or, if they want programmatic access, to the HTTP API. The MCP surface of 27 tools is the authoritative set for MCP agents.

### 7.24 Rate limits: 200 requests/minute per project

Peec publishes a rate limit of **200 requests/minute per project** (per the `/ratelimits` docs page). When the limit is hit, the server returns HTTP 429 with three headers:

- `X-RateLimit-Limit` — the ceiling (200)
- `X-RateLimit-Remaining` — the remainder in the current window
- `X-RateLimit-Reset` — seconds until the rate limit resets (not a Unix timestamp — check the value is small, not epoch-scale)

Practical implications for agents:

- The limit is 200/min sustained (≈3.3 req/sec averaged over a minute), not a per-second ceiling. Short bursts above 3.3/sec are fine as long as the rolling-minute total stays under 200. The intra-wave parallelism of 5–10 concurrent calls (§6.5) is a burst of 2–3 sec; even large tune-ups spread across several waves with inter-wave verification pauses stay well below 200/min in any rolling window.
- If you do hit a 429, back off using the `X-RateLimit-Reset` header (exponential backoff starting at 2s is more than adequate). Don't retry more aggressively than the header allows.
- Per-project rate limits mean multi-project batch workloads can legitimately parallelise across projects — a 10-concurrent call batch across 5 projects is 50 concurrent calls in aggregate without violating the limit, because each project has its own counter.

**MCP clients cannot see these headers.** The `X-RateLimit-*` headers live on the HTTP response envelope; MCP tool calls only surface the JSON body to the agent. The header-based backoff guidance above therefore applies to direct HTTP-API consumers (using `x-api-key` against `api.peec.ai`), not to MCP agents. MCP-side strategy is "pace by construction":

- Keep intra-wave parallelism to 5–10 concurrent calls with 1–2 second inter-wave pauses. Most time is spent waiting on verification reads, so the rolling-minute total stays well below 200 req/min.
- If an MCP tool call returns an error whose text mentions rate limiting or exhaustion, treat it as a 429-class signal and back off ≥30 seconds before retrying. There's no header to read.
- When building loops, prefer batching with explicit pauses over raw concurrency. A `for wave in waves: dispatch; sleep(2)` loop is safer than an unbounded `Promise.all`.

### 7.25 Authoritative source of truth for the tool catalogue

Peec's own documentation at `https://docs.peec.ai/mcp/tools` now enumerates all 27 tools — 15 read and 12 write (grouped under a single "write tools" heading that covers `create_*`, `update_*`, and `delete_*`). Earlier versions of this skill flagged that page as omitting `list_search_queries`, `list_shopping_queries`, and the entire write/destructive surface; those omissions have been corrected upstream as of April 2026.

In general, when the docs and the MCP disagree on surface or behaviour, **the authoritative source is what the MCP server announces on connection via `tools/list`** — not any prose description. Prose drifts; the tool catalogue is generated from the server's real schema. This skill documents per-tool behaviour (including the bits the docs still omit — empty-schema on `get_actions`, column asymmetries between reports, the `list_prompts.volume` type mismatch, etc.), not the tool list itself.

### 7.26 Empty collections are encoded inconsistently (`null` vs `[]`)

Different endpoints encode "no items" differently — and the inconsistency also shows up **within a single row of the same endpoint**:

- `list_brands` returns `aliases: null` on brands with no aliases set, but `domains: []` (empty array) on brands with no domains set. Same row, two different encodings.
- `list_prompts` returns `tag_ids: []` on prompts with no tags attached.
- Newly created entities via `create_brand` (no `domains` or `aliases` specified) come back with `domains: []` and `aliases: null` — confirming the split isn't a legacy-vs-new data artefact.

An agent that does `brand.aliases.length` or `brand.aliases.map(...)` on the `null` form will crash; `.length` or iteration works on the `[]` form. Defensive pattern:

```javascript
const aliases = brand.aliases ?? [];
const tags = prompt.tag_ids ?? [];   // harmless even though tag_ids is already []
```

This is a server-side inconsistency, not a client bug. When writing code that touches multiple endpoints, always coerce collection fields with `?? []` before iterating.

### 7.27 `create_prompt` rejects duplicates with a distinct error

Attempting to create a prompt whose `(text, country_code)` pair already exists returns:

```
Error: "This prompt already exists for this location"
```

This is a **distinct 409-class scenario** from the `update_brand` recalculation conflict in §7.19 — different tool, different cause. Agents should:

- Dedupe their input against a `list_prompts(limit=10000)` snapshot before bulk-loading.
- When this error surfaces mid-wave, treat it as "skip and continue" rather than retry — retrying will always hit the same error.
- Note that the deduplication key is `(text, country_code)` — the same prompt text in a different country is a separate prompt and will be accepted.

### 7.28 `list_models` display names lag behind upstream model IDs

Peec's model catalog shows occasional mismatches between `id` and `name`. Examples observed:

| `id` | `name` |
|---|---|
| `gemini-2.5-flash` | `Gemini 1.5 Flash` |
| `gpt-4o-search` | `GPT 5 Search` |

This is almost certainly a label-sync lag in Peec's internal model catalog. The practical implication for agents: **never text-match against `name`** — always filter by `id`. If you're building a UI summary that displays model names, consider showing the `id` alongside so the user can spot the discrepancy.

### 7.29 Chat payload uses camelCase where other endpoints use snake_case

`get_chat` returns chat payloads whose `sources[]` entries use camelCase keys:

- `urlNormalized`
- `citationCount`
- `citationPosition`

Every other endpoint in the MCP surface (`list_prompts`, `get_brand_report`, `get_domain_report`, `get_url_report`, etc.) uses `snake_case`. Additionally, the chat payload carries a `model_channel` object (`{id: "xai-0"}` etc.) alongside `model` (`{id: "grok-scraper"}`). This field matches the `model_channel_id` filter/dimension but is not otherwise documented.

**`model_channel` is the authoritative handle for fanout joins.** For fanout inspection on a specific chat, read `get_chat.model_channel.id` from the chat payload and pass it to `list_search_queries` (filter on `model_channel_id`). **Do not rely on `get_chat.queries`** — that field is often empty even when the chat had fanout that's visible via `list_search_queries`. The chat payload's `queries` field and the separate `list_search_queries` endpoint are not guaranteed to agree; the endpoint is the authoritative source, and `model_channel.id` is the key you need to join them.

**Do not conflate `urlNormalized` with the removed `normalizedUrl` field** (§7.22). That one was on the Get URLs Report endpoint and is gone in v0.9.0. `urlNormalized` lives on the chat sources payload, is currently active, and is not scheduled for deprecation that this skill has seen.

Defensive parsing: when walking chat sources, read both snake_case and camelCase forms and log a warning if only one exists — Peec may normalise one direction or the other in a future release.

### 7.30 `get_domain_report.classification` has 8 values, not 5

The skill's earlier references (and most narrative examples) mention five domain classifications: `CORPORATE`, `EDITORIAL`, `OWN`, `UGC`, `COMPETITOR`. The live enum is larger. Observed values:

```
CORPORATE, EDITORIAL, INSTITUTIONAL, UGC, REFERENCE,
COMPETITOR, OWN, OTHER
```

Concrete examples:
- `INSTITUTIONAL` — university / public-sector domains
- `REFERENCE` — Wikipedia-style reference content
- `OTHER` — fallback for domains that don't fit the other categories

An agent building a segmentation filter who only knows the five "obvious" values will silently exclude a material slice of domains — often including the `REFERENCE` bucket that matters most for authority-gap work. Always enumerate the full eight-value set when filtering.

### 7.31 `get_url_report.classification` has 11 values (10 observed + 1 schema-declared) — and isn't filterable server-side

Parallel to §7.30, the URL classification enum is also larger than most narrative examples suggest. Observed values:

```
HOMEPAGE, CATEGORY_PAGE, PRODUCT_PAGE, LISTICLE, COMPARISON,
PROFILE, DISCUSSION, HOW_TO_GUIDE, ARTICLE, OTHER
```

Client-side schema validation reveals one additional value — `ALTERNATIVE` — that exists in the schema but rarely appears in live data. Full documented enum is therefore 11 values.

**`get_url_report` does not accept `url_classification` (or `classification`) as a filter field.** The supported filter enum on the URL report is: `model_id, model_channel_id, tag_id, topic_id, prompt_id, country_code, chat_id, domain, url, mentioned_brand_id, mentioned_brand_count, gap`. Classification is returned as a *column* on every row, so you filter client-side after the response lands. Any recipe that appears to "filter by classification" is really pulling a broad URL set (typically via `gap >= 2`) and then narrowing the returned rows client-side. If an agent tries to pass `{field: url_classification, …}` to `get_url_report`, the tool will reject the call with a schema-validation error.

The same applies to `get_domain_report` — it exposes a `classification` column (§7.30) but no classification filter. Scope by domain name (from `list_brands.domains` or step-1 results) rather than by classification.

Recipe §8.2b (URL-level gap analysis) depends on picking among these values after the fact; an agent that narrows to `[LISTICLE, COMPARISON]` only will miss gap rows with classifications like `ARTICLE` or `HOW_TO_GUIDE` that are equally relevant to AI-search visibility. Pick the slice deliberately from the full enum, don't default to the two obvious values.

### 7.32 MCP output size limit — large responses auto-save to file

**High severity — silent response-shape change.**

MCP tool results are subject to a token cap (empirically ~130K characters). When a response exceeds the cap, the Cowork runtime (and equivalent runtimes in other clients) auto-saves the payload to a file and returns a file pointer in place of the data. The call still "succeeds" — but the in-band shape the agent receives is different from a normal columnar-JSON envelope.

Observed triggers:
- `get_domain_report(limit=10000, 12-month range)` — exceeded cap, saved to file.
- `get_url_report(limit=1000, 12-month range)` — exceeded cap, saved to file.

Implications for an agent:
1. **Large responses will not arrive in-band.** If your agent parses the first tool result as columnar JSON unconditionally, a file-pointer envelope will look like a parse failure.
2. **The file-pointer "success" path looks different from an empty-result path.** Don't conflate the two.
3. **Chained recipes can silently fragment across file drops.** A `get_domain_report → filter → get_url_report` workflow that pulls a wide date range at high limit can truncate silently if either call overflows.

**Defensive pattern — three layers:**
- Cap `limit` around 200–500 for analytical pulls. Only reach for 10000 when you specifically need the full dataset and have verified your date range is narrow enough.
- **Narrow the date range before widening the limit.** A 30-day window at limit=10000 is safer than a 12-month window at limit=1000.
- **Recognise the file-pointer envelope shape.** If the tool result isn't columnar JSON, check for a file path in the response before treating it as an error or as "no data".

This is MCP runtime behaviour, not Peec-server behaviour. It applies to any tool whose response is large enough to exceed the cap.

### 7.33 Parameter-fuzz error-layer catalogue — know which layer rejected your call

When a call fails, the error shape tells you which layer rejected it. This matters because retry strategies differ: client-schema errors mean "change the input"; server errors often mean "change the filter"; rate-limit errors mean "back off". Observed patterns:

| Bad input | Rejected by | Error text shape | Retry strategy |
|---|---|---|---|
| Bad enum (e.g. `classification="FOO"`) | MCP client-side schema | Full valid-values list echoed | Pick a valid enum value |
| Malformed date (not ISO, or invalid calendar date like `2025-02-29`) | MCP client schema pattern | Schema-pattern error with the regex | Fix the date; use a calendar-aware date library |
| Non-integer or out-of-range `limit` | MCP client-side schema | Type / minimum error | Fix the type |
| Fake `project_id` | Server (after auth) | `"project not found"` | Re-check the ID; don't retry |
| Fake `brand_id` / `prompt_id` / `tag_id` / `topic_id` | Server (after auth) | `"<entity> with ID <id> not found"` | Re-check the ID; don't retry |
| Soft-deleted entity ID (second delete, or filter after delete) | Server | Same `"not found"` shape as fake ID | Treat as "already done"; see §7.34 |
| Date-range order reversed (`start_date > end_date`) | Server | `"start_date must be on or before end_date"` | Swap the dates |
| `limit` above server max | Server | Silently capped to 10000 (not an error) | Accept the cap or paginate |
| Rate limit exhaustion (HTTP 429) | Server | Error text mentions "rate limit" / "too many requests" | Back off ≥30s; see §7.24 |

Practical guidance: when an agent encounters an error, branch on whether the error text contains the entity name (server error, fix the ID), a schema keyword (client error, fix the input shape), or rate-limit language (wait and retry). Don't treat all errors as transient and retry indiscriminately — you'll burn rate-limit budget against calls that will never succeed.

### 7.34 Soft-delete is not idempotent — second delete returns "not found"

The skill documents deletes as soft-delete (§3), which is true. What it doesn't make explicit is that soft-delete is **not idempotent from the client's perspective**. The first `delete_tag(tag_id)` or `delete_prompt(prompt_id)` returns `{success: true}`; the second call on the same ID returns an error:

> `"Tag with ID tg_… not found"`
> `"Prompt with ID pr_… not found"`

The shape is identical to what you'd get from a genuinely fake ID (§7.33) — there's no way for the client to distinguish "was deleted" from "never existed".

**Why this matters for bulk deletes:** a retry loop that treats any error as a transient failure will bounce forever on an already-deleted entity. The correct pattern is to **treat "not found" errors on `delete_*` as "already done; continue"**, not as a real failure. Keep a client-side record of deletion attempts so you can distinguish between a first-attempt "fake ID" (real error, investigate) and a retry-attempt "not found" (expected, continue).

The same pattern is expected on `delete_brand` and `delete_topic`.

### 7.35 `update_*` on identical values silently succeeds

`update_tag(tag_id=..., name="Same", color="blue")` on a tag that already has exactly that name and colour returns `{success: true}`. No echo of the record, no diff field, no indication that nothing actually changed. The same no-op-returns-success pattern is expected for `update_topic`, `update_prompt`, and `update_brand`.

Why this matters: an agent running a bulk `update_prompt(tag_ids=...)` across 67 prompts will see `{success: true}` for every call — including prompts whose existing tag set already matched the target. The write "succeeded" in the sense that the server accepted it; it did not in the sense that there was no actual data change.

**Defensive pattern:** when doing diff-driven bulk updates, compute the diff on the client against a `list_*` snapshot **before** issuing `update_*` calls. If the diff is empty, skip the call. This also reduces load against the 200 req/min rate limit (§7.24) because no-op writes still count against the budget.

### 7.36 Input-constraint asymmetry across `create_*` tools

Not all `create_*` tools enforce the same classes of constraints, which is a landmine for agents that assume symmetry. Known asymmetries:

| Tool / field | maxLength | Other constraints |
|---|---|---|
| `create_prompt.text` | 200 (hard, client-side schema) | `country_code` required from 92-country enum (§7.15) |
| `create_topic.name` | 64 (hard, client-side schema) | `country_code` optional (§7.20) |
| `create_tag.name` | **None enforced** — 169-char names accepted | No country code; 22-colour enum |
| `create_brand.name` | Not systematically probed | Accepts `domains`, `aliases`, `regex` |

Additionally, the **date-field regex is calendar-aware** — `2024-02-29` is accepted (leap year) but `2025-02-29` is rejected client-side (non-leap year). An agent computing dates by arithmetic (e.g. "365 days ago") can construct invalid calendar dates in non-leap years and get a schema error that looks like a Peec bug when it's actually correct rejection of a bad input. Always use a date library for date math.

Practical implication: do not build UI/validation logic on the assumption that name fields share a common maxLength, or that any string that "looks like a date" is a valid date. The enforcement surface is heterogeneous and needs per-tool handling.

### 7.37 Peec report tools return FIVE different metric types — treat each column by its type, not by name

**High severity — load-bearing interpretation rule.** Previous skill versions described a "three-scale" problem. Fresh testing in April 2026 revealed it's actually **five** distinct metric types, and the difference between them matters a lot. Getting this wrong produces report numbers that are off by 100× or that make no sense at all.

**The five metric types in any Peec report row:**

| Type | Wire scale | Display convention | Examples | Rule |
|---|---|---|---|---|
| **Ratio** | 0–1 float | 0–100% (multiply by 100) | `visibility`, `share_of_voice` on `get_brand_report`; `retrieved_share` on `get_domain_report` | These are fractions of a bounded whole. Multiply by 100 for display. |
| **Score** | 0–100 | 0–100 (no conversion) | `sentiment` (neutral = 50) | Already on display scale. Do not multiply. |
| **Rank** | 1+, lower better | 1+ with "lower is better" caption | `position` on `get_brand_report` | Never a percentage. Render with an arrow or explicit direction note. |
| **Rate** | **can exceed 1.0** | Averages-per-source, not a capped ratio | `retrieval_rate`, `citation_rate` on `get_url_report` and `get_domain_report` | **Do not multiply by 100 and display as a percentage.** A `retrieval_rate` of 1.8 means "on average, this domain is retrieved 1.8 times per tracked chat it appears in". Displaying as "180%" is nonsense. These are ratios with no upper bound. |
| **Count** | integer | raw integer | `mention_count`, `retrieval_count`, `citation_count`, `chat_count`, `mentioned_brand_count` | Display as-is. |

**Example own-brand 30-day `get_brand_report` row:**
`visibility=0.32` (Ratio → 32%), `share_of_voice=0.24` (Ratio → 24%), `sentiment=59` (Score → 59/100, slightly positive), `position=1.6` (Rank → 1.6th among tracked brands), `mention_count=819` (Count).

**The contradiction with Peec's public docs.** The brand metric pages under `docs.peec.ai/metrics/brand-metrics/` (one page each for visibility, share-of-voice, sentiment, and position) describe visibility and share-of-voice as 0–100 values — that's the **UI display convention**. The MCP server returns the 0–1 **wire format**. An agent reading the Peec docs first and then hitting the MCP will systematically report visibility and SoV as ~100× smaller than they actually are (`0.32` reported as "0.32%" rather than "32%"). Symmetric mistake on the Rate side: treating `retrieval_rate=1.8` as a ratio and reporting "180% retrieval rate" is meaningless but superficially plausible.

**Defensive pattern for any report layer:**
- Look up each column's type against the table above **before** applying any display transformation.
- For Ratio columns, multiply by 100; for Rate columns, never multiply.
- Leave Score columns alone.
- Render Rank with direction; never as a percentage.
- When concatenating values into prose, scale explicitly — "visibility of 32% (0.32 on the wire)" is safer than "visibility of 0.32" or "visibility of 32".
- If you encounter a column not in the table above, sample its values across several rows: a column with values consistently between 0 and 1 is a Ratio; a column with values spread from 0 to 100 is likely a Score; a column with values ≥1 that correlate with a count column is likely a Rate or Count.

**Caveat on Ratio denominators — engine-returned empty responses inflate "non-mention" counts.** `visibility` and `share_of_voice` are Ratios whose denominator is "chats in scope". That denominator includes chats where the engine returned an empty or placeholder response body (see §7.8 cause 6) — i.e. the engine didn't fail to surface the brand, it failed to answer at all. Where empty-response frequency is non-trivial, either (a) filter those chats out of the denominator before reporting, or (b) surface an "engine no-answer rate" alongside visibility so the reader can see the distinction. The headline number alone will understate true brand visibility by roughly the empty-response rate.

### 7.38 Aggregated reports may reference brand IDs that `list_brands` no longer returns

`get_domain_report` and `get_url_report` include a `mentioned_brand_ids` array (and `mentioned_brand_count` column) that record every tracked brand that co-appeared in the responses indexed by that domain/URL. This array can contain brand IDs that **do not appear in the current `list_brands` output** — typically 1–3 extra IDs.

**Likely cause:** soft-deleted brands. Peec's soft-delete (§7.34) removes the brand from `list_brands` filter matches but preserves the historical aggregated data. A brand that was tracked three months ago and then deleted will still appear in any aggregated report whose date range overlaps its active period.

**Other possible causes** (not disproven in live sampling):
- Brand ID renumbering on `update_brand` with `regex` changes (§7.19 triggers recalculation; the brand ID itself is stable, so this is unlikely to be the cause, but worth noting).
- Project configuration changes (a brand moved between projects in an older platform state).

**Practical guidance:** when an agent encounters a `mentioned_brand_ids` entry that doesn't resolve via `list_brands`, don't treat it as an error. The two reasonable responses are:
- **Ignore silently** — fine for summary-level reports where the unresolved brand doesn't materially change the narrative.
- **Flag as "formerly tracked"** — useful when `mentioned_brand_count` is a headline metric and you need the reader to understand that some of those mentions reference brands the roster no longer tracks.

Never retry or treat this as a client-side bug. The data is intentional; the `list_brands` filter exclusion is by design (§7.8 cause 5).

### 7.39 `get_domain_report` and `get_url_report` use different column names and types for the same concept

Peec's domain-level and URL-level reports **look similar** but have quietly divergent schemas. The retrieval-volume column is **named differently on each report** and has a different type, and the domain report's `_count` columns don't always appear in the default payload.

| Concept | `get_domain_report` | `get_url_report` |
|---|---|---|
| Retrieval volume | `retrieval_count` — **float** (aggregated retrievals weighted across chats) | `retrievals` — **integer** (raw count of retrievals for this URL) |
| Citation volume | `citation_count` — **float** | `citation_count` — **integer** |
| Retrieval rate | `retrieval_rate` — float (Rate, can exceed 1.0 — see §7.37) | not returned |
| Citation rate | `citation_rate` — float | `citation_rate` — float |
| Mentions | `mention_count` — integer | not applicable |

The actual column set on the default `get_url_report` (no dimension, no filter) observed in practice is: `url, classification, title, channel_title, citation_count, retrievals, citation_rate, mentioned_brand_ids`. Note the **absence** of `retrieval_count` and `retrieval_rate` — those live on the domain report only. The URL report's equivalent is the `retrievals` column (plain integer count) plus the derived `citation_rate`.

**Practical implication.** Code that reads from both reports and assumes a single shared column name like `retrieval_count` will silently miss URL-report data (the column doesn't exist there) or crash on the domain report (the column is there but it's a float, not an int). Always check the actual column list returned before writing aggregation logic.

**Additional column-visibility caveat on the domain report:** `retrieval_count` and `citation_count` **may not appear in the default, undimensioned `get_domain_report` response** — observed runs returned only `retrieved_percentage`, `retrieval_rate`, and `citation_rate`. The count columns surface on dimensioned queries (e.g. `dimensions=[model_id]`). If you need "top domains by retrieval volume" without a dimension, sort on `retrieved_percentage` (Ratio) rather than `retrieval_count`. See §8.10 step 1 for the corrected recipe.

**Cause (inferred, not confirmed):** the domain report aggregates across URLs at the same domain and weights by chat participation, which produces floats naturally; the URL report reports per-URL discrete retrievals as integers. The naming divergence (`retrieval_count` vs `retrievals`) looks like the URL-level column was renamed at some point and the domain-level one wasn't, or vice versa. Either way, **treat the two column names as non-interchangeable**.

**Defensive pattern:** when writing code that touches both reports, inspect `columns[]` on each response and map to a normalised internal key (e.g. both `retrieval_count` and `retrievals` → `retrieval_volume`), coercing to `float` uniformly. Don't assume schema symmetry across the two endpoints.

### 7.40 `get_brand_report` dimension columns may return null labels (observed on `model_id`)

When calling `get_brand_report` with `dimensions=[model_id]`, the response
may contain the correct number of rows (one per active engine) but with
`model_id: null` on every row — the dimension is clearly functioning
server-side (distinct aggregations per engine) but the label isn't
populating. The underlying chats have correct `model.id` values when
inspected via `get_chat`, so the aggregation layer is lagging behind the
data layer rather than the data being absent.

**Observable proxy for "recently queued" state.** Earlier versions of
this skill referenced a `volume_status: QUEUED` field on prompts as the
triggering condition. That field isn't exposed on `list_prompts` in
the current MCP schema (§7.42), so don't rely on it as a gate. The
observable signal that prompts are still being processed is
simpler: **the first full 24-hour aggregation window after a
`create_prompt` or `update_prompt` wave hasn't elapsed yet**. During
that window, dimensioned reports may return null labels. Keep a
client-side timestamp of when the last write-wave completed and use
that as the gate.

**Two failure modes, not one.** There's actually a pair of related
behaviours under this header — distinct enough to warrant different
workarounds:

- **Null-label rows** (observed in the first ~24h after a write wave):
  the response contains the correct number of rows, but the dimension
  label column is `null` on every row. Data is present; attribution
  isn't.
- **Zero-rows entirely** (observed around the 48h mark on at least one
  production project, when combining `dimensions=[topic_id]` or
  `dimensions=[model_id]` with a `brand_id` filter): `rowCount=0`. Data
  genuinely unreported for that filter-plus-dimension combination.
  Worse than null-label because the row count itself is wrong — no
  "correct count, null labels" signal to catch it.

**Workarounds, in order of cleanness:**
1. **Wait out the processing window.** Re-run the dimensioned report
   after the first full 24–48h processing cycle completes; both failure
   modes typically clear once the rollup catches up.
2. **Cross-reference row count against `list_models(is_active=true)` or
   `list_topics` / `list_tags`.** If the row count equals the expected
   segment count and labels are null, you're in the null-label mode and
   workaround 3 or 4 below will resolve it. If the row count is zero,
   you're in the zero-rows mode and workaround 5 is the reliable path.
3. **Run separate filtered calls per `model_id` (or per `topic_id` /
   `tag_id`).** Call `get_brand_report` once per engine with
   `filters=[{field: "model_id", operator: "in", values: [<id>]}]` — each
   call produces a single-row, self-labelling result. Works in both
   null-label and zero-rows modes.
4. **Use `visibility_total` as a heuristic label.** Only reliable when
   engines have materially different chat counts (e.g. AI Overview
   typically has fewer chats than ChatGPT on the same project).
   Unreliable when two engines have the same chat count. Doesn't apply
   to the zero-rows mode.
5. **Tag-filter-without-dimension (zero-rows-mode workaround).** On
   projects in the zero-rows window, dimensioned + `brand_id`-filtered
   queries fail but **tag-filtered queries without a dimension** return
   the full brand roster correctly. Use this pattern for branded-vs-
   non-branded analysis: `get_brand_report(filters=[{tag_id: <non-branded>}])`
   returns the full roster with visibility/SoV for the non-branded
   subset of prompts, no dimension needed. Combine with workaround 3
   (per-engine calls) for per-engine non-branded breakdowns.

**Do not report per-engine breakdowns with null labels or zero rows.**
Both are confident misreports in different shapes. Pre-flight checklist
item 10 enforces this — but it's worth re-stating: if the dimension
column is null on every row, or the rowCount is zero on a combination
you expect to have data, switch to the per-filter workaround (step 3 or
5) before putting numbers in front of a human.

**Related observation, same report, different dimension:** treat every
dimension with caution. Before attributing metrics to a dimension value,
confirm the label column is populated in the returned payload.

### 7.41 `list_search_queries` (fanout) returns zero for AI Overview, AI Mode, and Copilot — other engines vary

`list_search_queries` is documented as "the sub-queries the AI engine
fanned out to during retrieval" — but in practice some engines return
zero rows no matter the date range or filter combination. Verified as
of April 2026 across five production projects:

| Engine (`model_id` / channel) | Fanout data? |
| --- | --- |
| AI Overview (`google-ai-overview-scraper` / `google-0`) | ✗ zero rows |
| AI Mode (`google-ai-mode-scraper` / `google-1`) | ✗ zero rows |
| Copilot (`microsoft-copilot-scraper` / `microsoft-0`) | ✗ zero rows |
| ChatGPT (`chatgpt-scraper` / `openai-0`) | ✓ returns fanout |
| Grok (`grok-scraper` / `xai-0`) | ✓ returns fanout |

The three zero-row engines are the confirmed negatives — if your
reporting scope includes any of them, `list_search_queries` will not
help and you need to fall back to the `sources` array inside `get_chat`
payloads for that engine's retrieval signal. The two confirmed positives
(ChatGPT and Grok) are the engines we can currently rely on for fanout
mining.

**Every other engine — Perplexity, Gemini, the Claude models, DeepSeek,
Llama, Sonar, the `grok-4` and `gpt-4o*` API variants, etc. — is
untested.** They weren't active in any of the five projects tested, so
nothing is known about whether they return fanout. Do not assume either
way. Before relying on fanout for an engine not in the confirmed-positive
list above, call `list_search_queries` filtered on that engine over a
recent date range and verify `rowCount > 0`.

The shape of the observed split is "full-trace chat engines expose
fanout; SERP-style AI-answer surfaces don't", which is a plausible
hypothesis but not a rule — the authoritative check is always a per-
engine call, not an extrapolation from the observed pattern.

**Implications:**
- Fanout mining currently tells you what **ChatGPT and Grok** search
  for. Whether it also covers other engines is unverified — state the
  engine scope of any finding explicitly.
- Topics where the own brand's visibility lives primarily on AI
  Overview, AI Mode, or Copilot will have zero observable fanout — the
  retrieval surface for those engines has to be inferred from the
  `sources` array inside `get_chat` payloads, not from
  `list_search_queries`.
- Cross-engine conclusions drawn solely from fanout data are wrong by
  construction. State the engine scope explicitly in any finding
  derived from fanout, and flag any untested engines in the scope.

**Recipe adjustment.** In §8.3 ("What's the AI engine actually searching
for?"), the reliable answer currently covers ChatGPT and Grok. For AI
Overview, AI Mode, and Copilot, fall back to the chat-level `sources`
array as the retrieval signal. For any other engine, run a per-engine
`list_search_queries` probe first.

**Product feedback to flag upward.** `list_search_queries` should either
surface fanout from all active engines or expose a `source_engine` /
`engine_scope` field so the scope is discoverable without empirical
probing. Until it does, the pre-flight checklist (item 11) enforces
scope confirmation at the agent level.

**Re-verification cadence.** This table captures engine coverage as
verified in April 2026. Peec may expand fanout coverage without
announcement — re-probe the three zero-row engines on each major
project-tune-up session, and treat any positive result as a bonus
capability to integrate into §8.3. The authoritative check is always
a live per-engine call, not a memory of the last table state.

### 7.42 `list_prompts.volume` is a string ordinal, not an integer; `volume_status` is not exposed

`list_prompts` returns a `volume` column per prompt as of April 2026 — the
search-volume signal that was previously only visible in the Peec UI is now
pullable via MCP. But two details about this field catch agents out:

**1. `volume` is a string ordinal, not a number.** The published schema
describes `volume` as a 1–5 integer. In practice the column is populated
with lowercase string ordinals: `"very low"`, `"low"`, `"medium"`,
`"high"`, `"very high"`. Code that assumes a numeric type will either
coerce to NaN or throw. Treat the field as an enum of 5 ordered strings
and map to integers client-side if you need numeric sorting.

Observed distribution across projects tested in April 2026: the modal
value on most projects is `"very low"`; `"very high"` is rare. Don't
assume a normal distribution across the five ordinals.

**2. `volume_status` is NOT a field on `list_prompts`.** Earlier skill
text (now corrected in §7.40) referred to a `volume_status: QUEUED`
field as a trigger for aggregation lag. That field is documented in
some Peec materials but is not exposed on `list_prompts` in the current
MCP surface. Do not filter or gate on `volume_status` — the column
doesn't exist in responses and a schema-strict client will reject any
filter referencing it. Use the observable proxy documented in §7.40
(time since last `create_prompt` / `update_prompt` wave) instead.

**Practical use for strategy work.** The `volume` ordinal is a load-bearing
input for prompt-tracking strategy: a project where 90% of prompts are
`"very low"` volume is over-indexed on niche prompts and needs
high-volume head-term prompts added for commercial coverage. The
companion `peec-ai-tracking-strategy-builder` skill (§9.1) uses this
field directly in the volume-signal / allocation step.

---

## 8. Common recipes

The recipes below collectively cover Peec's six published primary use cases (per `docs.peec.ai/mcp/use-cases`): brand visibility monitoring (§8.1), competitive intelligence (§8.2a / §8.2b / §8.4 / §8.9), source authority audits (§8.2a / §8.10), prompt optimisation (§8.5, plus `/peec_prompt_grader`), action-driven SEO (§8.2b / §8.8, plus `/peec_source_authority`), and cross-platform benchmarking (§8.1 with `dimensions=[model_id]`). Recipe §8.7 (full project tune-up) wraps the atomics into a single overhaul workflow.

**Start here for the most common request type.** When a user says "give me a visibility report", "full report", "monthly report", or anything comparable, reach for the composite meta-recipe in **§8.0** first — it sequences the atomic recipes below into the three common depth tiers (quick / standard / deep).

**Composite recipes** answer recurring multi-step questions that the atomics only partially cover: **§8.8** is a local fallback for `get_actions` when the native tool is unavailable (e.g. schema-strict client strips params — see §7.12); **§8.9** is the full competitive gap analysis flow (strategic view, not just a URL list); **§8.10** is a source-authority audit (who's citing whom, and at what rate); **§8.11** is the safe test-entity lifecycle pattern for agent-driven experimentation.

### 8.0 "Give me a full visibility report" (meta-recipe)

This is the most common user request in practice. Users don't ask for a brand report or a domain report — they ask for a visibility report, full stop. The skill's atomic recipes (§8.1–§8.6) each answer one slice; a "full report" is a deliberate composition of several.

Three depth tiers map to the three common phrasings:

| User phrasing | Depth tier | Recipes to chain | Typical runtime |
|---|---|---|---|
| "quick check", "how are we doing" | **Quick** | §8.1 only (steps 1–4, skip step 5) | 2–3 calls |
| "visibility report", "full report", "monthly report" | **Standard** | §8.1 (all steps) + §8.1 with `dimensions=[topic_id]` + §8.2a + 1 sample chat via §8.3 | 7–10 calls |
| "deep dive", "audit", "everything we have" | **Deep** | Standard + §8.2b (URL gaps) + §8.6 (shopping queries) + 3 sample chats (one per active engine) | 12–15 calls |

**Standard-tier execution order:**

```
1. Follow §8.1 steps 1–5 → own-brand per-engine breakdown + full roster.
2. Repeat §8.1 step 4 with dimensions=[topic_id] → topic-level heat map.
   Zero-visibility topics are signal, not noise — they show which content
   verticals the brand is invisible on. Include them in the report,
   don't filter them out.
3. Follow §8.2a → domain-level source-authority gaps.
4. Pick one chat from the highest-mention-count engine and follow §8.3
   → one piece of qualitative colour (what the AI actually said).
5. Scale-normalise everything (§7.37) before writing the report:
   visibility and SoV × 100, sentiment as-is, position flagged as
   "rank among tracked brands" (§7.4).
```

**Date-range defaults by tier:**

| Tier | Window | Notes |
|---|---|---|
| Quick | 7–30 days | 7 only if chat volume is high enough to smooth. |
| Standard | 30 days | The baseline. Matches most clients' reporting cadence. |
| Deep | 90 days or "since project creation" | Watch §7.32 output-size caps — pair wider ranges with tighter `limit` values. |

Never pull 12-month at `limit=10000` without checking §7.32 — you'll hit the MCP output cap and the payload will silently save to a file instead of returning in-band.

**Output framing.** A full visibility report should always include: (1) the headline four-metric row for the own brand, (2) the competitive roster comparison, (3) the per-engine breakdown with the `is_active=false` engines flagged as inactive rather than zero (§7.1), (4) the topic heat map with zero-visibility topics called out as content gaps, (5) the domain-gap list with 3–5 actionable items, (6) one sample-chat quote showing the AI's actual framing. The atomic recipes give you the data; this meta-recipe gives you the narrative order.

---

### 8.1 "How visible is my brand this month?"

```
1. list_projects → pick project_id.
2. list_brands → note brand_id for is_own=true AND the competitor brand_ids.
3. list_models → note model_ids with is_active=true (see §7.1).
4. get_brand_report(project_id,
                    start_date=<30 days ago>, end_date=<today>,
                    filters=[{field: brand_id, operator: in, values: [own_id]}],
                    dimensions=[model_id])
   → own-brand breakdown per AI engine.
   (start_date and end_date are REQUIRED on every report tool — there is
   no "default to last 30 days" behaviour. Omitting them returns a
   schema error, not a default window.)
4b. Optionally, repeat step 4 with dimensions=[topic_id] instead of
    [model_id] → topic-level heat map. This reveals which content
    verticals the brand dominates vs. where it's invisible. Typically
    one of the most actionable dimensions in a standard report —
    consider it a near-default step, not an add-on.
5. get_brand_report(project_id,
                    start_date=<30 days ago>, end_date=<today>)
   → full-project roster, no dimension. This is the competitive
     benchmark: where does the own brand rank against each tracked
     competitor on visibility, SoV, sentiment, position?
6. Report the own brand's numbers alongside the roster view.
   Apply the scale rules (§7.37): multiply visibility and SoV by 100,
   keep sentiment as-is, flag position as "rank among tracked brands".
   Flag is_active=false engines as "inactive on plan, no data" rather
   than "zero visibility".
```

Why the two-call structure: step 4 answers "how are we doing where we show up?" (per-engine detail for our brand); step 5 answers "how do we compare to competitors?" (full-roster benchmark). Skipping step 5 is the most common mistake — a single own-brand number has no meaning without the competitive frame.

**Per-engine head-to-head competitive comparison.** The reports above give two useful views separately (own brand per engine, and all brands in one go) but no built-in way to read "how do we compare to competitor X on each engine". `brand_id` is **only valid as a filter, not as a dimension** — valid dimensions are `prompt_id, model_id, model_channel_id, tag_id, topic_id, date, country_code, chat_id`. To get a per-engine head-to-head, make two calls and pivot client-side:

```
# Call 1 — own brand, per engine
get_brand_report(project_id, start_date=…, end_date=…,
                 filters=[{field: brand_id, operator: in, values: [own_id]}],
                 dimensions=[model_id])
# Call 2 — competitor, per engine
get_brand_report(project_id, start_date=…, end_date=…,
                 filters=[{field: brand_id, operator: in, values: [competitor_id]}],
                 dimensions=[model_id])
→ two result sets, one row per engine per brand. Zip them by model_id
   into a small table (one row per engine, columns for own vs. competitor
   on visibility / SoV / sentiment).
```

This is the single most informative shape for a "we vs them" narrative — better than listing raw per-engine numbers and leaving the reader to do the arithmetic. When you're writing a competitive summary, reach for this pivot before writing any prose.

**Earlier drafts of this skill recommended `dimensions=[model_id, brand_id]`** for this pivot. That shape is rejected at the schema layer because `brand_id` isn't in the dimensions enum. The two-call pattern above replaces it.

**Default window:** 30 days. Shorter (7 days) is only useful if the project has high chat volume; longer (90 days) smooths signal but hides recent shifts. For anything longer than 30 days, read §7.32 on the output-size cap before pulling. For full-report composition, see §8.0.

### 8.2a Domain-level competitor analysis — "who's beating us at the source level?"

```
1. get_domain_report with filters=[{field: gap, operator: gte, value: 1}]
   → domains where competitors appear but we don't.
2. For each high-retrieval COMPETITOR-classified domain (§7.30), inspect
   which of our tracked brands are mentioned vs ours. This is the
   source-authority slice.
3. Decide: is this a "create content" gap (we should produce on our own
   domain) or a "get mentioned" gap (we should earn placement on theirs)?
```

Use this for source-authority audits and for feeding `/peec_source_authority`. Note the **full domain classification enum is 8 values, not 5** (§7.30) — don't filter only to `[CORPORATE, COMPETITOR, OWN]` or you'll silently exclude the `REFERENCE` and `INSTITUTIONAL` buckets that often matter most.

### 8.2b URL-level competitive gap analysis — "which specific pages should we target?"

This is a **sibling workflow to §8.2a, not a continuation of it.** An agent trying to chain the two by filtering domain results down to their own URLs via a second call will often find the competitor domain has *no* LISTICLE / COMPARISON classifications of its own — the gap signal lives at the URL level across all domains, not at the per-domain level. Run URL-level gap analysis as its own flow:

```
1. get_url_report with filters=[{field: gap, operator: gte, value: 2}]
   limit=20 (keep it bounded; see §7.32 output-size cap)
   → specific URLs where competitors appear multiple times and we don't.
2. Pick the subset with classifications you can meaningfully influence
   (LISTICLE, COMPARISON, HOW_TO_GUIDE, ARTICLE — see §7.31 for the full
   11-value enum).
3. get_url_content on each → actual markdown the AI engine is reading.
4. Analyse what gets a brand included: positioning, specificity of claims,
   structure of the listicle, the questions answered.
```

**Why `gap >= 2` and not `gap >= 1`:** `gap=1` surfaces pages where a single competitor appears once without the own brand — often just incidental coverage (a passing mention in a long article). `gap>=2` filters to pages where *multiple* competitors co-appear without the own brand, which is a stronger signal of a systematic editorial exclusion worth pursuing. Dial down to `gap>=1` for very new or low-data projects; dial up to `gap>=3` for mature projects with hundreds of retrievals per domain where the noise threshold is higher. Two is the pragmatic default.

The two flows answer different questions. §8.2a: "where is our source authority weak?" §8.2b: "which specific editorial placements should we pursue?" Most projects need both, run independently.

### 8.3 "What's the AI engine actually searching for?"

```
1. list_chats → pick one chat_id
2. list_search_queries(chat_id=...) → see the sub-queries the engine issued
3. get_chat → full response + brands_mentioned + sources
```

This workflow is what transforms Peec from a dashboard into a research tool. Don't skip it — it's where the real insight lives.

**Engine scope.** As documented in §7.41, `list_search_queries` returns
zero rows for **AI Overview (`google-0`), AI Mode (`google-1`), and
Copilot (`microsoft-0`)** — for those engines, step 2 will return
nothing; use the chat-level `sources` array from step 3 as the
retrieval signal instead. **ChatGPT (`openai-0`) and Grok (`xai-0`)**
are the confirmed positives. Other engines (Perplexity, Gemini, Claude,
etc.) haven't been tested — verify per-engine before relying on fanout
for any of them. Phrase findings to name the engines they actually
cover; don't say "what the AI searches for".

**How many sample chats for a report?** For the composite "full report" flow (§8.0): **1 chat** for a quick check (illustrative colour), **1 chat** for a standard report (from the highest-mention-count engine — the most representative slice), **3 chats for a deep dive** (one per active engine to capture per-platform variation). For a standalone research session rather than a report, pull as many as budget allows and compare fanout patterns across prompts.

### 8.4 Fix the competitor list after project creation

```
1. list_brands → note the auto-selected competitors
2. get_domain_report(high limit, exclude UGC) → find CORPORATE domains with
   high retrieved_percentage and mentioned_brand_ids that do NOT overlap
   with list_brands.
3. For each, create_brand with {name, domains: [domain], aliases: [...],
   optional regex}. Confirm with user first if multiple brands.
```

### 8.5 Add a tagged prompt set

```
1. create_tag(name="branded", color="blue") → note tag_id
2. For each desired prompt:
   create_prompt(text=..., country_code=..., topic_id=..., tag_ids=[tag_id])
3. Verify with list_prompts(tag_id=tag_id)
```

### 8.6 Audit what AI is recommending as products

```
1. list_shopping_queries(project_id, date range) → see actual SKU-level
   recommendations
2. Cross-reference with your product catalogue — are competitor products
   appearing where yours should be?
3. Use get_chat on the parent chat for context on why.
```

### 8.7 Full project tune-up (dry-run first)

A tune-up is a systematic overhaul of an under-performing Peec project: replacing wrong brands, retagging prompts, deleting non-commercial prompts, adding revenue-informed new prompts. Load the companion `peec-ai-project-tuneup` skill for the methodology.

Minimum viable sequence, all captured in a dry-run document *before* any writes:

```
# Capture current state (no writes)
list_brands, list_topics, list_tags, list_prompts — dump to an audit file
get_brand_report, get_domain_report, get_url_report — 30-day slices
list_chats + get_chat samples — qualitative

# Design target state (external research, no Peec calls)
# — external data: GSC queries, rank tracking, GA4 revenue, content inventory
# — produce: tag taxonomy, brand roster, prompt allocation table

# Dry run document — every call listed in execution order
# Execute in waves (see §6.5)

# Wave 1 — additive, zero risk
create_tag × N, create_brand × N

# Wave 2 — mutate existing
update_prompt (tags + topic) × N, delete_brand × N, delete_prompt × N

# Wave 3 — additive, final state
create_prompt × N (each with topic_id and tag_ids pre-resolved)

# Post-execution
list_prompts with an explicit high limit, OR paginate offset=0,100,…
  until rowCount=0 — confirm final prompt count equals
  (initial − deleted + created). See §7.18.
list_tags / list_brands → confirm final entity counts.
get_brand_report after 7 days → early signal.
```

Why dry-run first: deletions are destructive, tag-set replacements are not append, and the plan often changes after the client reviews it. A single dry-run doc is also a great handover artifact for the client.

**Count reconciliation as the completion check.** Track the arithmetic: initial prompt count − deletes + creates = expected final count. Verify with a paginated `list_prompts` read (see §7.18). This is faster and more reliable than spot-checking individual operations.

### 8.8 Approximate `get_actions` locally (fallback recipe)

Use this recipe when you can't rely on `get_actions` directly — schema-strict client strips params, scope=owned is flaky on the current project, or you want a single composite output instead of chaining 4–5 scoped calls. See §7.12 for when to prefer calling `get_actions` natively. This recipe approximates its output using the tools that do work. Also reach for it whenever a user asks "what specific actions should we take to improve visibility?" or anything that maps to Peec's Actions feature and you want a reproducible flow that doesn't depend on the tool's client-dependent behaviour.

```
# 1. Establish baseline and scope
list_projects → pick project_id
list_brands → own_id (is_own=true), competitor_ids
list_models → active_model_ids (is_active=true — §7.1)

# 2. Own-brand baseline (so recommendations are scale-aware)
get_brand_report(project_id, start_date, end_date,
                 filters=[{field: brand_id, operator: in, values: [own_id]}],
                 dimensions=[model_id])
→ current visibility, SoV, sentiment, position per engine.
  Apply §7.37 scaling before reading any numbers.

# 3. EDITORIAL gap (competitor-heavy pages where we're absent)
get_url_report(project_id, start_date, end_date,
               filters=[{field: gap, operator: gte, value: 2}],
               limit=20)
→ high-retrieval pages where ≥2 competitors appear and own brand doesn't.
  These are the EDITORIAL action candidates: pages to get listed on,
  outreach targets, PR/digital-PR opportunities.

# 4. OWNED gap (our own pages underperforming relative to peers)
# get_url_report has no url_classification filter — filter by the
# own brand's domain list instead (pulled from list_brands above).
get_url_report(project_id, start_date, end_date,
               filters=[{field: domain, operator: in,
                         values: [<own_brand.domains entries>]}],
               limit=20)
→ our own pages that are retrieved but rarely cited, or retrieved
  on fewer engines than comparable competitor pages.
  Cross-reference citation_rate (not retrieval_rate — §7.37 Rate type;
  values can exceed 1.0) against the leaders in the EDITORIAL gap set.
  Low citation_rate on OWN pages = OWNED action candidates:
  pages to restructure, clarify, or rewrite to earn citations.
  Note: own_brand.domains must include every TLD the company operates
  on (§7.10) — if .com is missing, the recipe will miss .com pages.

# 5. Domain-level source authority (who else is being cited heavily?)
get_domain_report(project_id, start_date, end_date,
                  filters=[{field: gap, operator: gte, value: 1}])
→ source-authority rivals. Used for context, not direct actions.

# 6. Synthesise into a prioritised list
- Group EDITORIAL gaps by likely outreach vector (listicle, comparison,
  how-to, reference source).
- Group OWNED gaps by page-type (category, product, blog, recipe — map
  to the classifications returned in step 4).
- Prioritise by retrieval volume × number of competitors present × gap
  delta (own 0 vs competitor N). Top 5–10 items become the action list.

# 7. Present with qualitative colour
Pull 1–2 chats via list_chats + get_chat that show the AI engine
choosing a competitor on a high-value prompt. This is what turns a
data list into an actionable narrative.
```

**What this recipe does NOT do** (and why that's OK): it doesn't produce Peec's exact `url_classification` action types (OWNED/EDITORIAL/TECHNICAL). It produces the two categories that matter most in practice (EDITORIAL, OWNED) and skips TECHNICAL — which covers crawlability/indexing issues that usually come from external SEO tooling (Screaming Frog, Sitebulb, searchVIU) anyway, not from Peec data.

**Expected runtime:** 6–9 MCP calls for a standard project. If the user then asks for "even more detail", chain §8.3 to pull sample chats from the action-list URLs so they can see the AI's actual framing of the competitive landscape.

### 8.9 Full competitive gap analysis (composite)

When a user asks "where are we losing to competitors?" or "what's the competitive landscape look like?" — this is the recipe. It combines roster benchmarking (§8.1 step 5), source authority (§8.2a), URL-level editorial gaps (§8.2b), and per-engine head-to-head (§8.1 head-to-head block) into a single flow that produces a three-layer narrative.

```
# Layer 1 — "How do we rank overall?"
get_brand_report (no dimension, no filter)
→ full-roster ranking on visibility / SoV / sentiment / position.
Identify the 2-3 competitors closest to or ahead of own brand.

# Layer 2 — "Where does the gap come from?"
For each identified competitor, run the §8.1 head-to-head pattern
(two calls, one filtered to own_id + dimensions=[model_id], one
filtered to competitor_id + dimensions=[model_id]; brand_id is a
filter, not a dimension). Pivot client-side by model_id.
Find the engines where the gap is widest (usually one or two dominate).

# Layer 3 — "What specifically is causing it?"
get_domain_report(filters=[{gap >= 1}], limit=20)
→ which source domains cite competitors but not us?
get_url_report(filters=[{gap >= 2}], limit=20)
→ which specific pages include competitors and exclude us?

# Optional Layer 4 — "What does a losing chat look like?"
list_chats(brand_id=closest_competitor_id, limit=10)
→ pick a chat where the competitor appears and own brand doesn't.
get_chat → read the actual AI response.
This reveals the narrative framing Peec doesn't visualise directly.
```

**Output framing.** Write this up as three stacked sections: (1) the roster gap ("we're at position 2.4; competitor X is at 1.6"), (2) the engine asymmetry ("the gap is concentrated on ChatGPT, not Grok or AI Overview"), (3) the concrete editorial and source-level drivers ("competitor X is cited on 4 high-retrieval listicles we're absent from"). Close with one AI-response excerpt that shows the actual prose the engine produced.

**Typical length:** 12–15 MCP calls. Budget 10 minutes of human-reading time for a written report at this depth.

### 8.10 Source authority audit (composite)

Answers "what sources is the AI reading, how authoritative are they, and which ones are citing competitors but not us?". Feed this into external content strategy (what to pitch, which outlets to court, what internal content to rewrite).

```
# 1. Top cited domains across the project
get_domain_report(project_id, start_date, end_date,
                  dimensions=[],
                  sort by retrieved_percentage desc, limit=30)
→ who are the AI engines actually reading?

# Note on the sort column: the undimensioned domain report returns
# retrieved_percentage (Ratio, 0–1), retrieval_rate (Rate, can exceed 1.0),
# and citation_rate (Rate) — but does NOT expose retrieval_count as a
# sortable column in the default response. Always sort the breadth view
# on retrieved_percentage. retrieval_count and citation_count DO appear on
# dimensioned domain responses, where you can sort on them directly.
# URL-level responses use a different column name — `retrievals` (integer)
# rather than `retrieval_count` — see §7.39 for the full mapping.
# If a sort-by-count call returns a schema error or empty rows, fall back
# to retrieved_percentage.

# 2. Classification mix
Same call; group rows by classification (§7.30 — 8 values).
Report the share per bucket: CORPORATE, COMPETITOR, OWN, UGC,
REFERENCE, INSTITUTIONAL, LISTICLE, OTHER (approximate names —
check §7.30 for the exact enum).
UGC + REFERENCE together often account for a large share of
retrievals — useful framing data.

# 3. Authority-vs-citation sanity check
Read citation_rate column (not retrieval_rate; §7.37 Rate type —
values can exceed 1.0). Sort descending.
Pair citation_rate against retrieved_percentage (the breadth metric
available on the default response).
Domains with high retrieved_percentage + low citation_rate are
"skimmed but not quoted" — weak authority signal despite frequent
retrieval.
Domains with low retrieved_percentage + high citation_rate are
"quoted when reached" — strong per-visit authority.
(If you need absolute counts instead of percentages, re-run step 1
with dimensions=[model_id] to expose retrieval_count and
citation_count — see §7.39.)

# 4. Gap layer — who's citing competitors but not us?
get_domain_report(filters=[{gap >= 1}], limit=20)
get_url_report(filters=[{gap >= 2}], limit=20)
Intersect with step 1: are any of the top-30 retrieved domains
also in the gap list? Those are the highest-leverage outreach
targets — they're already authoritative AND already citing
competitors.

# 5. Own-domain health check
# get_domain_report doesn't expose a classification filter, so scope
# by the own brand's domain list (from list_brands) instead.
get_domain_report(project_id, start_date, end_date,
                  filters=[{field: domain, operator: in,
                            values: [<own_brand.domains entries>]}])
→ our own domains' retrieval + citation rates over the period.
If citation_rate is low despite high retrieval_rate, we have an
internal content problem (§8.8 OWNED action path).
(The `classification` column in the response tells you how Peec
tagged each domain — CORPORATE/OWN/etc., §7.30 — but you can't
filter on it at query time; the own-domain list from list_brands
is the reliable scoping mechanism.)
```

**Key interpretive note.** Don't confuse `retrieval_rate` and `citation_rate`: both are Rate type (§7.37) and can exceed 1.0. A `citation_rate` of 1.8 on a domain means "on average, 1.8 distinct URLs from this domain are cited per chat that cites anything from this domain". It's a per-visit density measure, not a "share of chats" ratio. If you write "cited X% of the time", you'll be wrong in a way that superficially reads right.

**Deliverable shape.** Four stacked findings: (1) overall source authority landscape with classification breakdown, (2) own-domain citation health, (3) competitor-cited-not-us outreach list, (4) one surprise from the long tail (e.g. an INSTITUTIONAL domain that unexpectedly dominates citation_rate). This mirrors what a manual SEO source-authority audit would produce — but built from AI-engine retrieval data, not Google's index.

**Typical length:** 5–8 MCP calls. Shortest of the composite recipes.

### 8.11 Test entity lifecycle (safe experimentation pattern)

When an agent needs to demonstrate write-tool behaviour, stress-test schema edge cases, or run an exploratory workflow without polluting real data, use this lifecycle.

```
# 1. Capture baseline (exact snapshot)
list_brands, list_topics, list_tags, list_prompts (limit=10000 — §7.18)
→ save IDs and key fields to an in-memory baseline dict.

# 2. Prefix every test entity
create_brand(name="STRESSTEST-Rival")
create_tag(name="STRESSTEST-branded", color="blue")
create_topic(name="STRESSTEST-EN")
create_prompt(text="STRESSTEST: what are the best X in Y?", …)
The "STRESSTEST-" prefix is searchable, filterable, and impossible
to confuse with real entities. Pick any prefix; stick to one.

# 3. Run the experiment
Perform the writes, reads, verifications, etc.

# 4. Revert in reverse dependency order
delete_prompt(test prompt IDs) — must come before tag or topic delete
  if the prompts reference them (otherwise you orphan the reference).
delete_tag(test tag IDs)
delete_topic(test topic IDs)
delete_brand(test brand IDs)

# 5. Verify clean revert
list_brands, list_tags, list_topics, list_prompts (limit=10000)
→ assert counts match baseline; assert no "STRESSTEST-" prefix
remains anywhere.
```

**Why ordering matters.** Peec's soft-delete (§7.34) preserves historical chat data but removes the entity from `list_*` results. Deleting a tag while prompts still reference it leaves the prompt in a valid state (the tag's absence from `list_tags` doesn't break `list_prompts`), but it becomes impossible to audit what the test tag was attached to. Reverse-order deletes keep the audit trail intact.

**Baseline match ≠ zero residual.** A quiet bug: the prompt count returned to baseline but one of the baseline prompts had its `tag_ids` mutated during the test and wasn't reverted. Prefer capturing **all field values** (not just IDs) on any entity you mutate, and diff-verify those fields back to baseline after revert — not just counts. Count-only verification is necessary but not sufficient.

**When to use a separate test project instead.** For very-large-scale stress tests (>50 write operations) or anything that might hit rate limits, spin up a dedicated Peec project rather than prefixing entities in the live one. The test-prefix pattern is for single-session, <50-op experimentation.

---

## 9. Contributing

Spotted a discrepancy between this skill and Peec's actual behaviour? Discovered a new tool, parameter, or slash-command prompt? Hit a setup quirk in a client not yet covered? Please contribute back:

- **Issues and PRs:** [github.com/rebelytics/peec-ai-mcp](https://github.com/rebelytics/peec-ai-mcp)
- **Workflow:** See `CONTRIBUTING.md` in this repo for what's in scope and how to submit changes.

The goal is that anyone pulling this skill six months from now gets behaviour grounded in current reality, not an April 2026 snapshot. Contributions are the mechanism that keeps that promise.

---

## 10. License & attribution

**License:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Reuse, adapt, redistribute — just keep attribution.

**Attribution:**
> Peec AI MCP Companion Skill, maintained by Eoghan Henn (rebelytics.com), github.com/rebelytics/peec-ai-mcp.

**Not affiliated with Peec AI.** Peec's team has not reviewed or endorsed this skill.
