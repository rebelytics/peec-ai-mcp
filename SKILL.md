---
name: peec-ai-mcp
description: Companion skill for the Peec AI MCP server (https://api.peec.ai/mcp). Load when the user does Peec reporting, analysis, or multi-step work — visibility reports, per-engine comparisons, competitive gap analysis, source-authority audits, project tune-ups (brands, prompts, topics, tags), or Peec slash commands (`peec_weekly_pulse`, `peec_competitor_radar`, `peec_engine_scorecard`, `peec_topic_heatmap`, `peec_prompt_grader`, `peec_source_authority`, `peec_campaign_tracker`). Also load for Peec data interpretation (sentiment, position, visibility, share of voice, retrieval vs citation, `get_actions` two-step workflow, `list_prompts.volume` ordinals, `get_url_content` 5-day refresh cadence) or when combining two or more Peec tools. Skip for trivial single-tool lookups like `list_projects`, `list_brands`, `list_topics` where Peec's own tool descriptions suffice. Teaches agents the real behaviour of the Peec MCP, including gotchas the official docs omit or get wrong.
version: 2.3.0
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

**Coverage boundary.** This skill documents **the MCP surface**: the tools, their parameters and filters, the shapes they return, and how to read the metrics in them. It does **not** document the Peec web app's own UI and reporting screens, onboarding, plan or billing administration, any Peec capability with no MCP tool behind it, or tracking-strategy design (which prompts to track and why — that belongs to the Peec tracking-strategy companion skill). Silence in this file is not evidence a capability is absent: a companion skill is loaded precisely because the agent doesn't know the tool, so its silence reads as "no such feature" when it may only mean "not covered here". Check `tools/list` on connection and Peec's own docs before concluding something doesn't exist.

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

## Querying an opaque remote source — rules that aren't about Peec

Peec is one of several data sources an agent queries without being able to see inside it. The rules below are properties of that situation, not of Peec; each is stated here in its Peec-specific form.

1. **Run a control call before interpreting any absence.** An empty `rows: []` from a filtered call and a project with genuinely no data are byte-identical — §7.8 catalogues seven distinct causes that all look the same, and none of them errors. Before reporting "no mentions", "no citations", or "engine X returned nothing", re-run the same call with the filters dropped, or fire a `limit=1` `totalCount` probe on `list_chats` for the same date window (§4). Only once the unfiltered call proves the project has data in that window does the filtered emptiness mean anything. Two sharpenings: **a scope's presence in an enumeration is not evidence its data is processed** — a selectable date window, an engine in `list_models`, or a freshly-run wave can hold zero processed rows (the registry that lists scopes and the store that serves rows are different systems with a lag, cf. the ~24–48h post-write-wave gaps in §7.40), and an empty scope renders identically to a genuinely clean one, always as good news. So the strongest control is a **positive control on a known-populated scope through the identical call** — not merely an unfiltered call, which can succeed on older data and disguise the gap. And read the distribution of the zeros: zeros across *every* metric and dimension mean the scope is empty; a zero in *one* filtered slice means the filter matched nothing. A run or wave that completed within hours of the query is the leading suspect for an unprocessed scope — check its completion time before interpreting its numbers.

2. **Inspect the actual response shape on the first call; never script against an assumed envelope.** Peec's payload conventions are not uniform across the surface: chat payloads use camelCase where the report tools use snake_case (§7.29), empty collections come back as `null` on some fields and `[]` on others (§7.26), the domain and URL reports use different column names and types for the same concept (§7.39), and any response over the output cap is replaced by a file pointer rather than rows (§7.32). Call the tool once, read the real body, then write the loop.

3. **A filtered figure identical to the unfiltered one means the filter was ignored — not that the segment is the whole.** If `get_brand_report` with a `tag_id`, `topic_id` or `model_id` filter returns the same visibility, SoV and mention counts as the unfiltered project, treat the filter as unapplied until proven otherwise: check `visibility_total` per dimension cell (§7.7) and confirm the field is actually a supported filter for that tool. The report tools accept no classification filter at all (§7.31), so anything that looks like "filtered by classification" is client-side narrowing of a broad pull. The same "acceptance is not application" pattern shows up on writes, where `update_*` on unchanged values returns `{success: true}` with no diff (§7.35).

4. **Verify any count against an independent derivation before quoting it.** A single figure from one tool is not a fact about the project. Observed chat counts exceed `prompts × active_models × days` for structural reasons (§7.6), and the engine/channel catalogue drifts fast enough that any count in prose is a snapshot (§7.1). Derive the number a second way — or name the dataset behind it — before it reaches a human.

5. **Keep full-payload views to small limits; prefer count views where only a number is needed.** Size the dataset with the `limit=1` `totalCount` probe first, then choose in-band paging or deliberate overflow-and-grep (§7.32).

6. **An enumeration proves an item is addressable, not populated or current.** Listings like `list_brands`, `list_prompts`, `list_tags` and the dimension/filter sets in tool schemas are accretions — entities and fields stay listed after they stop being used (Peec's soft-deletes, §7.34/§7.38, are one instance of the general pattern). A zero-row probe against a listed entity or field has three readings — populated but not matching your filter, live but genuinely empty, or retired — and the API cannot distinguish them; only whoever configured the project can. Report an unpopulated listed item as an observation with a question attached ("these return nothing — still in use?"), never as a defect finding; choosing the defect reading commits someone else's time under the appearance of diligence. Where a working sibling entity covers the same concept, retired is the leading hypothesis, not broken.

7. **Human-authored metadata in a response is a claim, not behaviour.** Project names, prompt text, topic and tag names, and brand labels travel in the same envelope as machine-generated metrics and inherit their apparent authority while carrying none of their guarantees. A tag named "non-branded" is a claim about intent at tagging time, not proof of what the tagged prompts contain; a topic's name describes what it was built to hold, not what it holds now. Metadata is the artefact least likely to be updated when the configuration it describes changes, so the gap only widens. Where a label and the rows disagree, the rows win — verify any scope you take from a name or label against the actual rows before it reaches a deliverable, and report the drift to whoever owns the project.

8. **Before engineering a workaround for a predicate the tools can't express, check what the platform already computes.** Peec precomputes derived answers as first-class objects — `get_actions` recommendations, `domain_report.classification`, the gap filters (§6), retrieval/citation rates — and these are computed inside the platform, unbound by the tool surface's filter limits. A platform-computed figure also carries the vendor's definition and is comparable across runs, which an ad-hoc client-side reconstruction is not. The workaround is the interesting route and the precomputed answer is the boring one, so attention flows the wrong way by default; ask "has Peec already answered this?" before building around a missing filter.

    **The counterweight: a precomputed classification is a convenience for the vendor's own UI, not a gate for your analysis.** `domain_classification`, the `branded` system tag, `mentioned_brand_ids` — each is the platform's guess at a question the platform defined, and each is good enough to colour a dashboard while being wrong at the margin that decides an operator's list. Reach for the precomputed figure when you need the vendor's definition and cross-run comparability (rates, scores, recommendations); do **not** let one stand in for the operator's own roster or definition when the output is a list someone will act on. The safe shape is the same every time: **pull unfiltered, then apply the operator's definitions client-side**, and say which definition produced the number. §7.10 has the concrete case — filtering competitors out with `domain_classification` leaves real commercial rivals in the result set.

9. **A parameter that passes validation can still fail at execution — and the failure's blast radius misdirects.** Schema-level acceptance of a dimension, filter or value is a name check, not an engine guarantee; the two enforce different contracts (the §7.40 dimensioned-query failures are the local instance: accepted parameters, empty or null-labelled results). When a response comes back wholesale broken — every section erroring or empty, not just the slice you filtered — suspect your own parameter before suspecting the tool or the project: an error larger than its cause points the investigation at the wrong artefact by default.

10. **A default you didn't set is still a filter, and it appears in neither the request nor the response.** Peec's defaults are uneven, which is what makes them dangerous: the report tools take no default date window at all (omitting `date_from`/`date_to` is a schema error, not a silent 30 days — §8), while every paginated `list_*` tool defaults to `limit=100` and `offset=0`, silently truncating a project with more than 100 prompts, brands, tags or topics (§7.18). You cannot tell which regime you're in by re-reading the call or by inspecting the result — only by varying the parameter. Before quoting any total, count or "all of them" claim, issue the same call twice with the scoping parameter at its extremes and compare; if the figure moves, label every number with the scope that produced it. A `totalCount` probe, a row count, and an exhausted paging loop all report the scope you **requested**, never the scope Peec stores — and paging to the end is the most dangerous case, because it supplies the feeling of completeness that would otherwise prompt the check. A per-row timestamp older than your window (a chat date, a `first_seen`-style field) is no evidence at all about how wide the window was.

    And even a window you *did* set is still an aggregate that destroys recency — a distinct failure from the one above, and the one that actually bites here, since the report tools force `date_from`/`date_to` and so can never hide the window from you. Stating it correctly does not make the number current: a rate answers "how often, across this period" and is read as "how often, now", and the two diverge exactly when the condition has already changed — which is the case where acting on the number is most wasteful. A visibility collapse, sentiment dip or citation loss that ran through the first half of a 30-day window and recovered a fortnight before the report was written is arithmetically correct in the aggregate and operationally wrong in the recommendation: it sends someone hunting a live cause that no longer exists. **So before any rate — `visibility`, `share_of_voice`, `retrieval_rate`, `citation_rate`, `retrieved_percentage`, `sentiment` — carries a recommendation, segment the window.** Peec makes this cheap: `date` is in the `get_brand_report` dimensions enum, so one dimensioned re-run yields the series (mind the output cap on long ranges, §7.32, and the early-window dimension failures in §7.40). Where the dimension is unavailable or failing, fall back to disjoint `date_from`/`date_to` bands — 0–3, 3–7, 7–14, 14–21, 21–30 days — at one call per band; nested windows differenced against each other work the same way on any surface that takes a since-date. Two things belong in the deliverable: which band the finding actually lives in, and whether it is still occurring in the most recent band. A rate quoted without a recency band should not carry a recommendation to act. The bands also recover the mechanism the aggregate destroyed — a component that is present in the older bands and exactly zero in every band of 14 days or less names both the cause and roughly when it stopped. Two riders: check the denominator first (`visibility_total` per cell, §7.7), because a dramatic percentage over a handful of chats or retrievals is not a finding; and where a UI attaches a headline rate to a named prompt, page or domain, read it as scoped to that row until proven otherwise, not as a project total that happens to be displayed beside an example.

11. **Test an id before using it as a join key across two calls.** Peec's `or_`/`kw_`/`pr_`/`tp_`/`tg_`/`ch_` prefixes make every id look like a durable primary key, and nothing in a field's name, type or format distinguishes a stored identifier from a per-response handle. That promise is untested until the same scope is fetched twice and the id sets compared — and it already fails in one documented direction here, where `mentioned_brand_ids` carries `kw_` ids that no longer resolve through `list_brands` (§7.38). The domain and URL reports carry no row id at all. So key any before/after comparison on a **natural key** — `url`, `domain`, or a prompt's immutable `text` (§7.13) — and diff the substantive fields explicitly rather than inferring change from set membership. An unstable id fails silently in both directions: it manufactures a difference in exactly the operation used to detect difference, so the instrument's noise arrives wearing the costume of the signal.

12. **`mentioned_brand_ids` on a source row is PAGE-level, not answer-level — and it is substring-matched.** On `get_domain_report` and `get_url_report`, `mentioned_brand_ids` records the tracked brands whose name or alias appears **in the cited source page's own content**, not the brands that co-appeared in the answers citing it (§7.38). Peec's own docs say so ("which brands appear in the source content"), and the data agrees: a heavily-retrieved authority URL can carry an **empty** list, which is impossible at answer level. Two controls confirmed it on a live sample — of 18 pages where the own brand was present in the fetched HTML, Peec listed it on 17 (the miss carried no brands at all, i.e. unscraped); of ~130 "gap" candidates where the own brand was absent from `mentioned_brand_ids` and at least one competitor present, **0** had the own brand on the live page. So the default inverts: **a gap list built from this field is decisive about page content, not a hypothesis** — once you have run one positive and one negative control on the project in hand. Keep the controls (a sample of static fetches, which is also the bytes an AI crawler sees); they are cheap, and they still separate two findings with nothing in common: brand absent from the page, and brand present on the page but not selected into the answer, or present only under a legal-entity or alternative name — the first is a coverage problem, the second an authority and entity-disambiguation problem, and they call for opposite interventions. To tell page-level from answer-level in any tool: fetch a page you know does **not** mention the brand but whose citing answers do, and see which way the field falls. Second half, equally load-bearing: **detection is plain substring matching on the brand's name and aliases** — see §7.38 for the false-positive classes and the regex fix. Any brand whose name or alias is a dictionary word or a short acronym will pollute both the mention list and the gap list until a word-boundary regex is configured. The sample's disagreement rate is the list's error rate; measure it before the list drives spend.

**A portability test for new rules.** Before writing any new rule into this file, ask: could the sentence survive having "Peec" removed? If yes, it is a rule about querying any opaque data source, not about Peec — keep it phrased generically, so it can be lifted into whatever companion skill documents the next tool.

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
