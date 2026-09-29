# Data-literacy gotchas (§7)

Part of the **peec-ai-mcp** skill (CC BY 4.0 — Eoghan Henn / [rebelytics.com](https://www.rebelytics.com)). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read the contents list below before reporting any numbers, and read in full every entry that touches a tool or field in your workflow. The Pre-flight checklist in SKILL.md compresses the always-on subset; it does not replace the entries.

**Contents:**

- 7. Data-literacy gotchas (READ THIS BEFORE REPORTING)
  - 7.1 `list_models` returns 16 engines (schema `model_id` enum is 19); only `is_active: true` are tracked
  - 7.2 `*-scraper` models measure consumer behaviour; raw model IDs measure API responses
  - 7.3 Sentiment formula is not `sentiment_sum / sentiment_count`
  - 7.4 `position` in brand reports = position among *tracked* brands, not overall position
  - 7.5 Domain report is a *retrieval* report, not a *mention* report
  - 7.6 Observed chat counts exceed `prompts × active_models × days`
  - 7.7 `visibility_total` varies per dimension cell
  - 7.8 Empty results have seven possible causes
  - 7.9 Auto-selected competitors on project creation are often wrong
  - 7.10 `classification=COMPETITOR` in domain report ≠ commercial competitor
  - 7.11 Write-operation consent, verification, and safe-experimentation patterns
  - 7.12 `get_actions` MCP schema is empty — the tool is still callable, but reviewers will strip params
  - 7.13 `update_prompt` cannot change prompt text
  - 7.14 `update_prompt.tag_ids` is full replacement, not append
  - 7.15 `create_prompt` has no `language` field — only `country_code`
  - 7.16 `create_prompt.text` max length is 200 characters
  - 7.17 `update_prompt` accepts `topic_id: null` to detach
  - 7.18 Pagination: most `list_*` tools default to 100, but `list_projects` and `list_models` don't paginate at all
  - 7.19 `update_brand` triggers background metric recalculation
  - 7.20 `create_topic` has optional `country_code` and 64-char `name` limit
  - 7.21 `update_brand.regex` accepts `null` to clear an existing pattern
  - 7.22 Deprecated fields to avoid
  - 7.23 HTTP API has features the MCP doesn't expose (yet)
  - 7.24 Rate limits: 200 requests/minute per project
  - 7.25 Authoritative source of truth for the tool catalogue
  - 7.26 Empty collections are encoded inconsistently (`null` vs `[]`)
  - 7.27 `create_prompt` rejects duplicates with a distinct error
  - 7.28 `list_models` display names lag behind upstream model IDs
  - 7.29 Chat payload uses camelCase where other endpoints use snake_case
  - 7.30 `get_domain_report.classification` has 8 values, not 5
  - 7.31 `get_url_report.classification` has 11 values (10 observed + 1 schema-declared) — and isn't filterable server-side
  - 7.32 MCP output size limit — large responses auto-save to file
  - 7.33 Parameter-fuzz error-layer catalogue — know which layer rejected your call
  - 7.34 Soft-delete is not idempotent — second delete returns "not found"
  - 7.35 `update_*` on identical values silently succeeds
  - 7.36 Input-constraint asymmetry across `create_*` tools
  - 7.37 Peec report tools return FIVE different metric types — treat each column by its type, not by name
  - 7.38 Aggregated reports may reference brand IDs that `list_brands` no longer returns
  - 7.39 `get_domain_report` and `get_url_report` use different column names and types for the same concept
  - 7.40 `get_brand_report` dimension columns may return null labels (observed on `model_id`)
  - 7.41 `list_search_queries` (fanout) returns zero for AI Overview, AI Mode, and Copilot — other engines vary
  - 7.42 `list_prompts.volume` is a string ordinal, not an integer; `volume_status` is not exposed
  - 7.43 An ended trial or inactive project stays fully readable — but reports nothing about its own date range

---

## 7. Data-literacy gotchas (READ THIS BEFORE REPORTING)

These are things the official documentation either omits or gets wrong. Ignoring them produces confident-sounding but materially wrong analysis.

### 7.1 `list_models` returns 16 engines (schema `model_id` enum is 19); only `is_active: true` are tracked

The MCP's `model_id` filter/dimension enum on report tools listed 19 values at initial verification (adds `claude-haiku-4.5`, `claude-sonnet-4`, `grok-4`, `google-ai-mode-scraper`, `google-ai-overview-scraper`, `microsoft-copilot-scraper` on top of the legacy set). `list_models` returns 16 of these — the 3 omitted are engines that exist in the enum but aren't yet surfaced through the listing tool. Filter on `is_active: true` before building engine breakdowns. On lower-tier plans users select a subset of engines; the others return empty data. On a TRIAL-tier project, `is_active: true` typically holds for only 3 engines out of 16. Higher tiers unlock more. Empty results for an inactive model look identical to "no data exists" — there's no error.

**Practical implication.** If you build a report from the `model_id` filter enum (19 values) instead of from `list_models` (16 values), three of those engines will return clean empty envelopes regardless of plan tier — treat them the same way you'd treat any other inactive engine: skip, don't report as "zero visibility". If the user asks about one of the three non-listed engines (`claude-sonnet-4`, `claude-haiku-4.5`, `grok-4` on most projects), tell them it's not available via the MCP's listing surface even though the enum accepts it.

**Drift — the catalogue grew and `list_models` is now deprecated.** A later re-verification against the live schemas found: the `model_id` enum has grown to **24 values** (adding, among others, `grok-4.3`, `qwen-3-6-plus`, `qwen-3-7-plus`, `amazon-rufus-scraper`, `deepseek-v4-pro`); `list_models` now returns **18 rows**, and its own tool description marks it **"Deprecated — prefer `list_model_channels`"** — a tool that has been added to the MCP surface (§3). The structural lessons above are unchanged (the filter enum is a superset of the listing tool's output; `is_active` gates real data), but the specific counts in this section are snapshots. **Re-verify any engine/channel count against the live tool schema before quoting it** — the catalogue is the fastest-drifting part of the surface, and Amazon Rufus / Qwen-class engines arriving unannounced shows new providers can appear without notice.

**Engine availability by country is not answerable from the docs — check the project.** Which engines Peec offers in a given market is a recurring question (typically in the form "can you track DeepSeek / Qwen in these two markets?"), and a search of the vendor documentation does not settle it: no published per-country engine matrix was found, and the absence of one is not evidence that availability is uniform. **Record this as unverified rather than asserting it either way.** The only reliable answer is empirical and per-project: run `list_model_channels(project_id)` — or `list_models(project_id, is_active=true)` on older surfaces — on a project configured for that market and report what comes back, naming the project and date the answer came from. An engine's presence in the `model_id` enum says nothing about its availability in a given country, on a given plan, or on a given project (§7.8 cause 4).

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

**Why "well-known brands" in particular** — the brief phrase above covers a broader pattern worth naming directly. High-salience parametric fallback affects any query where the model has strong priors from its training data:

- **Well-known brands** (the original case) — branded queries almost always trigger parametric responses on ChatGPT and, variably, Grok. The model has enough training data on the brand to answer without retrieving. The typical shape: close to every sampled branded ChatGPT chat carries `sources: []`, while the same prompts on retrieval-first engines populate sources for a meaningful share of chats — the gap is the engine's architecture, not the brand's web presence.
- **Well-known standards or concepts** — named technical or management standards, legal frameworks, established methodologies. The model answers from training even when the brand in question is a small specialist organisation attached to that standard.
- **Commercial categories well-represented in pre-training** — cosmetics, hotels, consumer electronics, household appliances. Commercial queries can trigger parametric answers because the model has enough contemporary text about the category to answer without fetching.
- **Regulated verticals** — pharma, gambling, alcohol and tobacco, some financial products, some legal-services jurisdictions. The sharpest case of the pattern, plus a layer of refusal behaviour (§7.8 cause 7) that has to be sampled for separately.

**Per-engine architecture** — the pattern is not symmetric across engines. ChatGPT (scraper and API) and Grok can skip retrieval entirely on high-salience queries; Perplexity, AI Overview, AI Mode, and Copilot are retrieval-first architectures and almost always populate `sources`. Claude falls between the two depending on the query. When a domain report looks sparse, don't assume "the brand isn't retrieved anywhere" — it may only mean "ChatGPT answered parametrically", and the retrieval-first engines in the roster still have useful retrieval data.

**Strategic implication.** `get_domain_report` systematically under-represents parametric-heavy engines in the retrieval numbers while `get_brand_report` counts their mentions normally. When the two tell different stories — brand has high visibility but low retrieval — check the per-engine split before concluding anything about content authority. The `peec-ai-tracking-strategy-builder` §11.13 pattern library entry covers the strategic response (shift content-strategy focus to retrieval-based engines, set long-horizon expectations for parametric ones, prioritise training-data-influencing work for the parametric channel).

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

- **Model channels multiply the count.** Each model (e.g. `gpt-4o`) can have multiple channels (`openai-0`, `openai-1`, etc.) representing different regional/setting variants. `list_chats` filters by `model_id`, but each model may emit several chats per day from different channels. The full `model_channel_id` enum observed on report tools at initial verification is: `openai-0, openai-1, openai-2, perplexity-0, perplexity-1, google-0, google-1, google-2, google-3, anthropic-0, anthropic-1, deepseek-0, meta-0, xai-0, xai-1, microsoft-0` (16 channels across 8 providers). Later re-check: the enum has since grown to **18 channels**, adding `qwen-0` and `amazon-0` (Amazon Rufus) — and the new `list_model_channels` tool (§3) is now the preferred way to resolve the live channel set rather than reading the enum.

  **Resolving a channel id to a human-readable name.** No tool takes a channel id and returns a label. The two routes that work:

  1. **From the data itself** — `model_channel_name` is returned as a column on the report tools, but **only when `model_channel_id` is one of the requested dimensions**. Add the dimension to the pull you were making anyway and the labels arrive with the rows, no join required. This is the route to prefer, because it labels the rows you actually have.
  2. **From `list_model_channels`** — the live configuration for the project. Correct for current channels; see the caveat below.

  **Caveat: `list_model_channels` describes the present, and historical rows outlive it.** A channel that has been retired from a project's configuration disappears from `list_model_channels` while every historical chat and report row carrying it remains. Observed: `list_model_channels` no longer returned `xai-0` for a project in which every Grok chat of the tracked period carries `model_channel_id: xai-0`. A label lookup keyed on the current channel list therefore leaves an entire provider's rows unlabelled — and, worse, invites the reading that those rows are corrupt rather than merely older than the configuration. Label historical rows from the data (route 1), and keep a hardcoded fallback map for the common ids:

  ```
  openai-0  → ChatGPT
  google-0  → Google AI Overview
  xai-0     → Grok
  ```

  Extend it from `model_channel_name` values as you meet them; never infer a name from the id's provider prefix alone, since a provider can expose several channels (`google-0` AI Overview vs `google-1` AI Mode) that mean materially different things.
- **Back-fill on acceptance.** The "start running immediately" language suggests newly accepted prompts may be run multiple times shortly after acceptance to populate initial data.
- **Error retries.** Failed prompt runs may re-run on the same day without being deduped in the chat count.

Practical guidance: don't reconstruct "how many chats are expected" from simple arithmetic — it won't match. Use `list_chats` directly to get the observed count.

**Empirical anchor (TRIAL tier).** A TRIAL-tier project with 70 prompts and 3 active engines produced **452 chats in its first 24-hour window** — 2.15× the simple prediction of 210 (70 × 3). Consistent with the model-channel multiplication hypothesis above. Use this as a rough expectation anchor when setting user expectations for day-1 chat volume on TRIAL-tier projects, but treat the multiplier as project-specific — it will vary with which engines are active (channel count differs per engine) and any acceptance back-fill in play.

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
7. **Regulated-vertical content-policy refusal.** Certain engines (most commonly `chatgpt-scraper`, but also occasionally Claude and Gemini) produce refusal responses for regulated-category prompts — pharma, supplements, gambling, regulated finance, weapons, age-restricted products — where the engine's content policy blocks a substantive answer. The `get_chat` response looks populated (the message body has text) but reads as "I can't help with that" or similar. Aggregate-level, it looks like a chat that produced no brand mentions; qualitatively it's a structural absence, not an evidence-based one. Indicators: short message bodies, refusal-language patterns ("can't help", "not able to provide", "recommend consulting a professional"), absence of topical content. In regulated verticals, sample at least 8 chats per engine on regulated-category prompts to estimate the refusal rate before relying on that engine's data for strategic conclusions. Filter refusals out of visibility/SoV denominators — they're neither parametric nor retrieval, they're refusal, and they mean something different strategically (parametric = model knows your brand from training; refusal = model will never mention your brand regardless of what you do).

Peec returns a clean empty array (or counts the chat as a non-mention) in all seven cases — no error, no warning. Before reporting "no data", validate filter IDs by listing first (which excludes soft-deleted entities, so a missing ID in `list_*` is itself a signal), sanity-check the date range against the platform's real coverage window, and for low-visibility chats spot-check `get_chat` payloads for empty bodies and refusal patterns.

**Run the control call first.** The cheapest discriminator across causes 1–5 is to re-issue the same call with every filter dropped — or a `limit=1` `totalCount` probe on `list_chats` for the same date window (§4). If the unfiltered call also returns nothing, the project has no data in that window and the filter is irrelevant; if it returns rows, the emptiness is a property of the filter, and the seven-cause list tells you which. Never let a filtered absence stand as evidence of absence without that control.

**Trailing and interior data gaps are states, not events.** A related diagnosis that looks like cause 1 but isn't: a window of zero chats at the tail of a query range. When data ends before the query window does, probe `list_chats` `totalCount` for the trailing window — zero chats means the project stopped collecting (plan/trial state); non-zero chats with zero rows on another surface (e.g. fanout) means that surface lags, not the project (§7.41). And don't let "stopped collecting" harden into project lore: on one TRIAL-tier professional-services project, collection paused for five days and then resumed, leaving a clean interior gap in the time series. Re-probe in later sessions before describing a project as ended, and flag interior gaps explicitly in any time-series reporting — date-windowed rates computed across a gap will undercount for that period.

### 7.9 Auto-selected competitors on project creation are often wrong

When a user creates a project, Peec auto-suggests competitors based on algorithmic similarity (brands that co-appear in at least two tracked prompts, per Peec's `/identifying-your-competitors` docs). In practice these skew toward **information sites** (forums, publications, wiki-likes) rather than actual commercial competitors. In multiple observed projects, Peec auto-selected content sites (reference databases, community forums, industry publications) while ignoring the actual commercial competition (direct retail rivals, marketplaces, adjacent brands) that the domain report reveals once data accumulates.

**Skill recipe:** after project creation, run `list_brands` + `get_domain_report` side-by-side. Any high-retrieval `CORPORATE`-classified domain *not* in `list_brands` is a candidate competitor. Use `create_brand` with `domains` + optional `regex` to add it.

### 7.10 `classification=COMPETITOR` in domain report ≠ commercial competitor

In `get_domain_report`, the `classification` column can be `COMPETITOR` — but this just means "domain of a tracked competitor brand", not "direct commercial rival". Don't use it to identify competitive threats; use mention overlap + domain retrieval volume instead.

**`domain_classification` is Peec's guess — never the competitor gate, and never the basis for a landscape share.** The failure is asymmetric and therefore easy to miss. Excluding the own-brand and competitor classes (e.g. `domain_classification not_in [OWN, COMPETITOR]`, rendered as *You* / *Competitor* in the UI) does reliably remove the **own** brand's domains — the `OWN` class is driven by an explicit `domains` array you control (above), so it behaves. It does **not** remove real commercial competitors: any rival shop that is not on the tracked brand roster, and several that are, land in the generic corporate class and survive the filter. The typical result: genuine competing shops come through a `not_in [OWN, COMPETITOR]` filter as `CORPORATE`. The resulting list looks like a clean "third-party sources only" set and is not one.

Two rules follow:

- **Gate on the operator's roster, not the platform's class.** Pull the report **unfiltered**, then apply the client's own competitor list (and own-domain list) client-side. A competitor roster is a commercial judgement about who competes for the same customer; no vendor classifier has that input.
- **Compute landscape shares unfiltered.** "Own vs competitor vs third-party share of citations" computed over a classification-filtered pull is arithmetically sound and substantively wrong — the denominator silently excludes whatever the classifier happened to file elsewhere. Filter after the aggregation, never before it.

Same caution for the `domains` array in the other direction: a competitor brand with domains left unset is classified by content, not by ownership. Check `list_brands` coverage before reading any classification-derived split.

**Multi-TLD brands and the `OWN` classification.** `classification=OWN` is driven by the `domains` array on the `is_own=true` brand row in `list_brands`, not by name matching or corporate-ownership inference. A brand with multiple TLDs (e.g. example.de, example.com, example.nl) will show `OWN` **only for the specific TLDs listed in that array** — all other TLDs of the same commercial entity fall back to `CORPORATE` (or whatever other classification applies). This is a project configuration completeness issue, not a classification bug. Agents auditing a domain report should check `list_brands(is_own=true).domains` first; if the array doesn't cover every TLD of the own brand, advise the user to update it via `update_brand` before drawing conclusions about "own vs. competitor" domain share.

### 7.11 Write-operation consent, verification, and safe-experimentation patterns

The MCP server exposes **12 write/destructive tools** — 8 `create_*`/`update_*` and 4 `delete_*`. Every write tool is flagged `readOnlyHint: false` in the server's tool annotations, and every `delete_*` tool is flagged `destructiveHint: true`. Clients that honour MCP annotations (Claude, Cursor, and most others) prompt for confirmation before the call runs. Writes are consequential — some are destructive, at least one is billable (`create_prompt` consumes plan credits), and `update_brand` triggers background metric recalculation with a 409-class concurrent-edit window (see §7.19). Agents should therefore:

- **Confirm with the user before any mutation — but treat batch authorisation as one consent event.** If the user says *"go ahead and create these 20 prompts and then delete these 5 tags"* as a single instruction, don't re-prompt on each individual call. If the plan changes mid-run (new entities, new destructive steps that weren't in the original batch), fall back to per-step confirmation. Rule of thumb: each batch's confirmation covers only the writes explicitly enumerated at batch approval time.
- **Prefer `list_*` for verification after a write.** Writes return either `{id: "…"}` for creates or `{success: true}` for updates/deletes — never an echo of the mutated record. Follow up with the matching `list_*` call if you need to confirm state, especially after bulk waves.
- **Gate on credit balance before bulk `create_prompt`.** `create_prompt` is the only credit-consuming call. On TRIAL or small paid plans, check the remaining balance in the Peec UI before dispatching a wave of prompt creates — there's no `get_credit_balance` endpoint, and credit-exhaustion errors look like billing-shaped rejections rather than schema errors.
- **For experimentation, create a dedicated test project in Peec** so writes don't pollute production data. Peec's soft-delete model means test clutter accumulates server-side even after "deletion" (§7.34) — isolating experimentation to a throwaway project keeps the production roster clean.

**Historical note.** Earlier versions of this skill flagged Peec's docs as incorrectly describing the MCP as "read-only". Peec has since corrected their documentation to enumerate the full write surface. The skill content here has been retained because the consent-and-verification guidance is the useful part — the "docs are wrong" framing is no longer accurate.

### 7.12 `get_actions` MCP schema is empty — the tool is still callable, but reviewers will strip params

**Status update (re-verified):** `get_actions` is reliably callable from Cowork and other pass-through clients. Earlier versions of this skill described the tool as "broken" — that was a client-behaviour issue, not a server-side outage. The server-side two-step workflow works; the agent just has to be aware that the MCP tool's JSON schema is empty and some clients will strip parameters before sending.

**Behaviour by scope, re-confirmed:**

| Call | Status | Returns |
|---|---|---|
| `get_actions(scope=overview)` | ✓ works | Navigation metadata — counts and breakdowns by `url_classification` and `domain`, used to plan drill-down calls |
| `get_actions(scope=owned)` | ⚠︎ intermittent — has returned HTTP 422 at `POST .../get-action-owned` in otherwise-healthy sessions | Own-domain URL actions (no extra params required per schema) |
| `get_actions(scope=editorial, url_classification=<value>)` | ✓ works | Third-party editorial pages at that URL classification |
| `get_actions(scope=reference, domain=<value>)` | ✓ works | Reference-site (e.g. wikipedia.org) actions for that domain |
| `get_actions(scope=ugc, domain=<value>)` | ✓ works | UGC-site (e.g. reddit.com, youtube.com) actions for that domain |

The two-step workflow is: call `scope=overview` first to see which slices have data, then drill down by scope + the corresponding extra parameter. Full recipe in §6.4.

**Known client-layer caveat.** The tool's declared JSON schema is still empty (`{properties: {}, type: "object"}`). Schema-strict MCP clients strip all parameters before sending, and the server then rejects with *"No matching discriminator: scope"*. If you see that error, you're on a client that's stripping params — switch clients or use the §8.8 workaround. Cowork and other pass-through clients send the params through and the call succeeds.

**`scope=owned` 422 caveat.** Confirmed intermittent on at least one production project: `scope=overview`, `scope=editorial`, `scope=reference`, and `scope=ugc` all returned 200 OK in the same session where `scope=owned` returned HTTP 422 at `POST .../get-action-owned`. The error surface (path exposed, no schema-mismatch details) points to a server-side issue on the owned endpoint, not client-side parameter stripping. If you hit it, the overview response already includes the OWNED opportunity score, relative-strength tier, and gap percentage per `url_classification` — that's usually enough for the strategic read without the drill-down text. For the full owned-URL gap list, fall back to §8.8's local approximation (filter `get_url_report` with the `gap` filter against the own brand's domains).

**When to use §8.8 (approximate locally) instead:**
- Running on a schema-strict client that strips params.
- Want a single composite output without chaining 4–5 scoped calls.
- Need the OWNED view on a project where `scope=owned` is returning 422 — see caveat above.

For most pass-through client sessions, prefer calling `get_actions` directly now. Report persistent `scope=owned` 422s and the empty tool schema to `support@peec.ai`.

**Data vs interpretation — the two trust surfaces in one payload.** `get_actions` returns two categories of content on every row, and they do not carry the same reliability:

- **Data surface (reliable):** named URLs, named domains, gap percentages, opportunity scores, slice classifications (`url_classification`, `domain`, OWNED / EDITORIAL / REFERENCE / UGC). Peec knows which pages got retrieved and how often; trust the data.
- **Interpretation surface (not reliable):** the generated action text ("Incorporate X into the homepage", "Mention your brand favorably on Reddit", "Take inspiration from Y's category pages"). Peec generates these algorithmically without any knowledge of the brand's positioning, sister-brand relationships, platform norms, or commercial context. The interpretation is generic and frequently wrong in specific ways.

Anti-pattern actions to flag automatically and filter out before propagating to deliverables or writes:

- **Homepage vocabulary stuffing.** "Incorporate '[niche vocabulary]' into the homepage copy." The homepage is the primary brand-positioning surface; stuffing it with vertical-specific vocabulary dilutes brand identity regardless of the opportunity score. The underlying signal (homepage doesn't register for the queried topic) is legitimate; the action is not. Propose a dedicated landing page or content hub instead.
- **"Mention your brand favorably" on Reddit / UGC.** Always wrong. Platform anti-pattern that gets brands shadow-banned and generates negative community sentiment. The underlying signal (Reddit / forum thread is a high-retrieval surface where the brand is absent) is legitimate; the action framing is a platform violation. The correct action is organic community engagement over time, or no action.
- **Inspiration from sister-brand pages.** When the "competitor" named in the action is a brand inside the same group or portfolio, the action is noise — not a competitive signal. Peec's brand model can't see corporate-family relationships. Filter these out at the Analyse step.
- **PR targeting to comparison / `best-of` / "top 10" domains without legitimacy check.** Peec classifies comparison and listicle sites as `EDITORIAL` or `COMPARISON` based on surface markers, but in commercial verticals many of these are affiliate-driven comparison plays, not genuine editorial publications. See the EDITORIAL/COMPARISON default below and the legitimacy verification chain in `peec-ai-tracking-strategy-builder` §13.17.

Rule: **always extract the signal before acting on the interpretation**. When using `get_actions` output in a deliverable or as input to a Phase B deck, record separately (1) the signal (named URL, domain, opportunity score, gap %, slice classification), (2) the action as Peec phrases it, (3) whether the action survives commercial common-sense review. Actions that fail review should have their signal preserved — the data isn't wrong, the interpretation is. Do not let filtered actions pass through to writes or stakeholder output.

**EDITORIAL / COMPARISON classification — affiliate-driven by default in commercial verticals.** Peec's `EDITORIAL` and `COMPARISON` classifications (on `get_actions` targets and on `get_url_report.classification`) are based on surface markers — article structure, editorial-looking content, visible comparison tables — and cannot see the affiliate monetisation layer underneath. In commercial verticals with high affiliate base rates, assume these targets are affiliate-driven until a browser check proves otherwise. Verticals where this default applies:

- **E-commerce** across categories, in every market with a mature comparison-site scene ("best X", "X compared", "top 10 X" domains and their local-language equivalents).
- **High-commission categories** — supplements, gambling, finance affiliate stacks, VPN, hosting, web-tool roundups.
- **SaaS review sites** — most "best X software" roundups carry affiliate links.

Verification indicators (all three needed to classify as *genuine* editorial, not just "looks editorial"):

- **No affiliate parameters on outbound links** (`?aff=…`, `?a_aid=…`, `/ref/…`, partner-id fragments, redirects through adcell / awin1 / tradedoubler / daisycon / ShareASale / Impact / CJ / Rakuten).
- **No commission / affiliate disclosure copy** ("as an affiliate", "partner programme", "we may earn a commission", and the local-language equivalents).
- **Visible editorial byline and editorial separation** from commercial interests (publication masthead, editorial ethics statement, non-commercial about page).

When the verification fails, the action shape flips from PR outreach to **network-join**. The affiliate network — adcell, AWIN, Tradedoubler, Daisycon, ShareASale, Impact, CJ, Rakuten — is the vehicle, not the specific URL. Joining the relevant programme places the brand across every comparison site on that network that covers the category, at scale; site-by-site PR is the hard path. See `peec-ai-tracking-strategy-builder` §13.17.3 and §13.17.4 for the full critical-filter procedure and action-shape routing.

**Two independent confirmed instances in a vertical flip the prior for that vertical.** Once the affiliate pattern is confirmed twice in a vertical, subsequent EDITORIAL / COMPARISON targets in that vertical should be routed to network-join by default, with the browser check escalated to confirmation only — not to establish the prior. Log the vertical-level prior in the findings file so later loops inherit it.

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

Unlike some other AI-visibility tools where custom prompt tracking exposes separate language and locale fields, Peec's `create_prompt` takes **only** a `country_code` (two-letter ISO, e.g. `DE`, `GB`, `US`) and the prompt text. Language is inferred from the text itself. Practical implication: when building multi-market prompt sets, you manage language at the level of the prompt text (write the German prompt in German; write the French prompt in French). There's no separate field to set.

The allowed country codes are constrained to a fixed enum of **92 ISO 3166-1 alpha-2 codes** at last verification, covering most of Europe, the Americas, Middle East/North Africa, and Asia-Pacific. If your target country isn't on the list, the call will fail validation.

### 7.16 `create_prompt.text` max length is 200 characters

`create_prompt` enforces `minLength: 1, maxLength: 200` on the `text` field. Budget your prompt text accordingly — long-form analytical prompts will be rejected. For very long prompts, break them into focused shorter ones.

### 7.17 `update_prompt` accepts `topic_id: null` to detach

Setting `topic_id: null` (not empty string, not omitted) removes the prompt's topic assignment. Useful when restructuring topics. To move a prompt from one topic to another, pass the new `topic_id` directly — no detach step needed.

### 7.18 Pagination: most `list_*` tools default to 100, but `list_projects` and `list_models` don't paginate at all

`list_brands`, `list_prompts`, `list_tags`, `list_topics`, `list_chats`, `list_search_queries`, `list_shopping_queries` accept `limit` (default 100, max 10000) and `offset` (default 0). The 100-item default is a **silent truncation trap** — a project with more than 100 prompts, brands, tags, or topics will have its state misread if you don't pass an explicit `limit` or chain `offset` calls.

`list_projects` and `list_models` have **no pagination parameters at all**. `list_projects` accepts only `include_inactive` (boolean); `list_models` accepts only `project_id`. These tools return the full set in a single call, so the truncation trap doesn't apply there — but if you're coding a generic paginator, special-case them.

**Per-tool limit caps differ — later drift.** The "max 10000" above is no longer uniform: `list_search_queries` now hard-caps `limit` at **1000**, and its description explicitly says not to request 10000 but to paginate with `offset`. Other `list_*` tools retained the 10000 cap at last check. Treat every quantitative cap in this section as a snapshot — **check the live tool schema before relying on any max value**, and expect per-tool divergence rather than a single global cap. The `limit=1` totalCount probe (§4) pairs well with this: size the dataset first, then pick a limit/offset plan that fits the tool's actual cap.

**Two safe patterns for full-state reads on paginated tools:**

1. **Explicit large limit** — `list_prompts(project_id, limit=10000)` for any inventory task. The server caps at 10000 but that's almost always enough for a single project.
2. **Paginate until empty** — call with `offset=0, limit=100`, then `offset=100, limit=100`, … until `rowCount=0`. This is the pattern used during post-tune-up verification to prove no prompts were missed.

Both work; prefer the explicit-limit form for single-call simplicity, and the paginated form when you want an audit trail that explicitly proves completeness.

**Boundary behaviour:** `limit=0` returns a clean empty envelope rather than rejecting; `limit=1` works as expected; `limit=999999` is **silently capped** to the server's 10000 maximum without error. `limit=-1` and non-integer `limit` values are rejected client-side by the schema. The silent cap is the one to remember — an agent asking for "everything" by passing a huge number will silently get a truncated result, not an error. (On tools with a lower cap — e.g. `list_search_queries` at 1000 — over-limit values are rejected by the schema rather than silently capped; either way, don't assume one request returns everything.)

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

Peec's API has sunset several fields. Don't rely on them, and if you find older tooling that does, flag for upgrade:

- `citation_avg`, `usage_count`, `usage_rate` — deprecated 2026-03-19 (v0.12.0) per Peec's API changelog; still returning but scheduled for removal. The modern equivalents are retrieval/citation counts split per entity.
- `normalizedUrl` — **removed** (not just deprecated) 2026-03-06 (v0.9.0) from the Get URLs Report endpoint. Any tooling still referencing this field will silently get `undefined`. Treat the raw `url` as canonical.

If a Peec API field appears in a response but isn't documented in the current docs, check the changelog before using it — Peec's deprecation pattern is to leave the field returning for a grace period before removing it.

### 7.23 HTTP API has features the MCP doesn't expose (yet)

Peec's HTTP API (separate from the MCP server, same platform) exposes functionality that is **not** callable via the MCP surface at last verification. Specifically:

- **Prompt and topic suggestions** with accept/reject endpoints — Peec can suggest prompts or topics based on project context, but these live in the HTTP API, not the MCP.
- **Model channel listing** (`list-model-channels`) — exposes finer detail about which models/channels a plan has access to. (Update: this one has since crossed over — it's now exposed via MCP as `list_model_channels`, and `list_models` is marked deprecated in its favour. See §3 / §7.1.)

Why this matters for the agent: do **not** hallucinate MCP tools for these capabilities. If a user asks for "prompt suggestions", the MCP has no tool for that; either direct them to the Peec UI (where the feature lives) or, if they want programmatic access, to the HTTP API. The MCP surface announced via `tools/list` is the authoritative set for MCP agents — and, as the `list_model_channels` crossover shows, HTTP-only features can migrate into it over time.

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

Peec's own documentation at `https://docs.peec.ai/mcp/tools` now enumerates all 27 tools — 15 read and 12 write (grouped under a single "write tools" heading that covers `create_*`, `update_*`, and `delete_*`). Earlier versions of this skill flagged that page as omitting `list_search_queries`, `list_shopping_queries`, and the entire write/destructive surface; those omissions have since been corrected upstream. (And the catalogue keeps moving: `list_model_channels` was added later — see §3.)

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

**Drift — a `domain_classification` filter field has since been observed on `get_url_report`.** Treat the filter enum above as a snapshot and re-read the live schema: a later live run accepted `{field: domain_classification, operator: not_in, value: [...]}` on the URL report, taking the *domain*-level classes of §7.30 (surfaced in the UI as *You*, *Competitor*, and so on) rather than the URL-level classes of this section. So the URL report now has **two** classification concepts with opposite filterability: `url_classification` (column only, filter client-side) and `domain_classification` (filterable server-side). Don't confuse them, and read §7.10 before using the filterable one for anything: it removes the own brand's domains reliably and real competitors not at all, which makes it unsafe as a competitor gate or as the basis of a landscape share.

Recipe §8.2b (URL-level gap analysis) depends on picking among these values after the fact; an agent that narrows to `[LISTICLE, COMPARISON]` only will miss gap rows with classifications like `ARTICLE` or `HOW_TO_GUIDE` that are equally relevant to AI-search visibility. Pick the slice deliberately from the full enum, don't default to the two obvious values.

### 7.32 MCP output size limit — large responses auto-save to file

**High severity — silent response-shape change.**

MCP tool results are subject to a token cap (empirically ~130K characters). When a response exceeds the cap, the Cowork runtime (and equivalent runtimes in other clients) auto-saves the payload to a file and returns a file pointer in place of the data. The call still "succeeds" — but the in-band shape the agent receives is different from a normal columnar-JSON envelope.

Observed triggers:
- `get_domain_report(limit=10000, 12-month range)` — exceeded cap, saved to file.
- `get_url_report(limit=1000, 12-month range)` — exceeded cap, saved to file.
- `list_search_queries(limit=1000)` on a fanout-heavy project (~210K chars per page) — exceeded cap, saved to file.

Implications for an agent:
1. **Large responses will not arrive in-band.** If your agent parses the first tool result as columnar JSON unconditionally, a file-pointer envelope will look like a parse failure.
2. **The file-pointer "success" path looks different from an empty-result path.** Don't conflate the two.
3. **Chained recipes can silently fragment across file drops.** A `get_domain_report → filter → get_url_report` workflow that pulls a wide date range at high limit can truncate silently if either call overflows.

**Defensive pattern — three layers:**
- Cap `limit` around 200–500 for analytical pulls. Only reach for 10000 when you specifically need the full dataset and have verified your date range is narrow enough.
- **Narrow the date range before widening the limit.** A 30-day window at limit=10000 is safer than a 12-month window at limit=1000.
- **Recognise the file-pointer envelope shape.** If the tool result isn't columnar JSON, check for a file path in the response before treating it as an error or as "no data".

This is MCP runtime behaviour, not Peec-server behaviour. It applies to any tool whose response is large enough to exceed the cap.

**Processing the overflow file.** When the payload does land in a file, the file itself becomes the dataset — but how you can process it depends on where it landed:

- **Check reachability before planning the analysis.** In split host/sandbox environments (e.g. Cowork, where shell commands run in an isolated sandbox while file tools operate on the host), the overflow file is typically saved under the **host's** temp directory — a path the sandboxed shell cannot reach. `bash`/`jq`/`python` pipelines against it will fail with "no such file", while host-side file tools (Read, Grep, or their equivalents) can access it fine. Verify which execution surface can reach the saved path *before* designing the processing strategy.
- **The saved JSON is a single line** — line-oriented tooling (line counts, Grep's count mode, Read with line offsets) is useless against it directly. The workable primitive is **match-only grep** (`-o`) with offset/head-limit probing: a pattern match with `offset=N` that returns `k < head_limit` lines means exactly `N + k` total matches. This gives exact counts and targeted extraction at trivial context cost — e.g. probing with `offset=300` and getting 23 lines back proves exactly 323 matches, without ever loading the payload.
- **Two-stage grep converts the file into a line-based derivative.** A full-row `-o` grep over the single-line JSON whose match output itself exceeds the output cap gets persisted as a *new* tool-results file — and that second-stage file is **line-based** (one match per line). All line-oriented tooling works on it: Read pages through it with offset/limit, count mode counts correctly, and further `-o` extractions (e.g. pulling the leading `pr_…` prompt IDs off each row) are cheap. The pattern: (1) full-row `-o` grep on the single-line overflow JSON → persisted line-based match file; (2) run cheap count/extract passes against that file instead of the original. In production this turned a ~420K-char two-page fanout payload into a 300-odd-row match file that could be term-split and ID-extracted without re-reading the originals.
- **For anything beyond counting, copy the file into a scratch directory and parse it with a script.** Grep primitives are right for sizing and spot-extraction; they are the wrong tool once the task is a join, a group-by, or a shortlist that will be revisited. The durable pattern is: copy the saved payload to a writable scratch path the execution surface can reach, then `json.load` it and work on the parsed rows (zip `columns` with each `rows` entry — §4). Benefits compound: the parse is exact rather than pattern-matched, intermediate results can be written alongside as small files, and the dataset survives a context compaction that would otherwise force a re-pull. `get_url_report(limit=1000)` over a multi-week window lands around 150 KB and is a routine trigger. Nothing about this is Peec-specific — it is the standard handling for any overflow-prone endpoint, so an agent that has done it once on another MCP already knows the moves.
- **Persist every result a later step will need.** When two calls have to be joined — the classic being a report keyed on `prompt_id` plus a `list_prompts` call to label those IDs (§8.12) — write **both** payloads to files as they arrive. A compaction between the pull and the analysis otherwise costs a full re-pull, and on a project whose data window has closed (§7.43) a re-pull is not always as cheap as it sounds.
- **Deliberate overflow is a legitimate strategy.** When you only need pattern counts or examples from a large dataset, intentionally triggering the overflow (e.g. `list_search_queries(limit=1000)`) and grepping the saved file is far cheaper than paging hundreds of KB of rows through the conversation context. Size the dataset first with the `limit=1` totalCount probe (§4), then decide: in-band paging for small sets, overflow-and-grep for large ones.

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

**High severity — load-bearing interpretation rule.** Previous skill versions described a "three-scale" problem. Fresh testing revealed it's actually **five** distinct metric types, and the difference between them matters a lot. Getting this wrong produces report numbers that are off by 100× or that make no sense at all.

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

**Read-time vs write-time checks — run this rule twice, not once.** Applying the table above at the moment you read a number from Peec is necessary but not sufficient. Metric-type context decays rapidly as numbers move through intermediate workspaces (notebooks, spreadsheets, prose drafts) and into deliverables, and by the time a value is typeset in bold serif on a slide it reads as "just a number" — the Rank-vs-Score context has usually evaporated. The specific, recurring failure mode: a `position` of 2.0 presented next to a competitor's 2.8 reads as if the bigger number "wins", inverting reality. The fix is to re-run the metric-type check at write-time, not only at read-time. Pre-flight checklist item #12 ("Write-time metric-type re-check") formalises this; apply it before any Peec value is placed in a slide, email, report, or chart, whether the author is you or a collaborator. For Rank values specifically, prefer ordinal labels (`#1` / `#2` / `#3`) or an explicit "lowest = best" caption over raw decimal values in stakeholder-facing presentations — raw decimals invite the inversion.

**Caveat on Ratio denominators — engine-returned empty responses inflate "non-mention" counts.** `visibility` and `share_of_voice` are Ratios whose denominator is "chats in scope". That denominator includes chats where the engine returned an empty or placeholder response body (see §7.8 cause 6) — i.e. the engine didn't fail to surface the brand, it failed to answer at all. Where empty-response frequency is non-trivial, either (a) filter those chats out of the denominator before reporting, or (b) surface an "engine no-answer rate" alongside visibility so the reader can see the distinction. The headline number alone will understate true brand visibility by roughly the empty-response rate.

### 7.38 `mentioned_brand_ids` is page-level and substring-matched — and may reference brand IDs that `list_brands` no longer returns

**What the field measures.** `get_domain_report` and `get_url_report` include a `mentioned_brand_ids` array (and `mentioned_brand_count` column) listing every tracked brand whose name or alias was found **in the cited source page's own content**. It is a property of the *page*, not of the answers that cited it. Peec's own documentation says as much — the URL table's Mentions column is "which brands were mentioned on this specific URL", and the URL detail page's Brands Mentioned is "which brands appear in the source content".

An earlier revision of this skill read the field as answer-level (brands co-appearing in the responses that cited the URL). That was wrong, and it inverted the advice built on it. The cheapest disproof is the empty list: a top-authority URL retrieved hundreds of times in a week can carry `mentioned_brand_ids: []`, which cannot happen at answer level, because the answers citing it plainly mention brands. Two controls settle it on any project: pages where the own brand is present in the fetched HTML carry it in the field almost without exception (a miss typically carries no brands at all, i.e. the page has not been scraped), and candidates where the own brand is absent from the field and a competitor present essentially never carry the own brand on the live page.

**Consequence for gap analysis.** Because the field reports page content, "own brand absent, competitor present" is a direct statement about the page — usable as a prefilter for editorial-gap and outreach target lists rather than merely a hypothesis about them. Run one positive and one negative control per project before leaning on it (fetch a handful of pages the field says carry the brand, and a handful it says do not), then trust it. Do not carry this reading across to other platforms: a same-named field elsewhere may well be answer-level, and the two measure different objects.

**Detection is plain substring matching on name and aliases — not word-boundary aware.** Any brand whose name or alias is a common word or another organisation's acronym generates large volumes of false mentions:

- A short brand token that occurs inside longer unrelated English words will match on every page containing that word. The shape: a brand token that is also a common word matches on a large slice of unrelated pages, and is the *only* brand listed on every one of them.
- A short acronym alias collides with unrelated organisations using the same letters — especially across languages, where a public body or an unrelated company may share the initials. A single such alias can dominate an entire gap pool.

**Fix:** configure a word-boundary regex for any such brand (`\bBRAND\b`, plus exclusion of the colliding context or domain where an acronym is shared). **Cheap detector:** compute the share of sampled URLs where one short-named brand is the *only* brand listed. A brand that appears alone on a large, topically unrelated slice of the corpus is matching a word, not being mentioned. Run this before any gap list is built, because false mentions suppress gap rows (the brand looks present) while false competitor mentions manufacture them.

**Unresolvable IDs.** This array can also contain brand IDs that **do not appear in the current `list_brands` output** — typically 1–3 extra IDs.

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
zero rows no matter the date range or filter combination. Verified
across five production projects:

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
last verified. Peec may expand fanout coverage without
announcement — re-probe the three zero-row engines on each major
project-tune-up session, and treat any positive result as a bonus
capability to integrate into §8.3. The authoritative check is always
a live per-engine call, not a memory of the last table state.

**Later re-verification.** Per the cadence note above: on a
production project with ChatGPT, AI Overview, and Copilot active,
100% of ~2,000 fanout rows over a six-week window carried
`chatgpt-scraper` / `openai-0`; AI Overview and Copilot contributed
zero. The table above still holds.

**End-of-data vs ingestion-lag check.** When fanout data ends before
the query window does, don't assume fanout-specific lag — probe
`list_chats` `totalCount` for the trailing window. Zero chats means
the project stopped collecting (plan/trial state); non-zero chats
with zero fanout rows means a fanout-specific ingestion lag. And
treat "stopped collecting" as a state, not a terminal event: a
TRIAL-tier project observed with a trailing zero-chat window resumed
collection five days later, leaving a clean interior gap (see §7.8).
Re-probe on later sessions before reporting a project as ended, and
flag interior gaps in any time-series built on fanout or chat data.

### 7.42 `list_prompts.volume` is a string ordinal, not an integer; `volume_status` is not exposed

`list_prompts` returns a `volume` column per prompt — the
search-volume signal that was previously only visible in the Peec UI is now
pullable via MCP. But two details about this field catch agents out:

**1. `volume` is a string ordinal, not a number.** The published schema
describes `volume` as a 1–5 integer. In practice the column is populated
with lowercase string ordinals: `"very low"`, `"low"`, `"medium"`,
`"high"`, `"very high"`. Code that assumes a numeric type will either
coerce to NaN or throw. Treat the field as an enum of 5 ordered strings
and map to integers client-side if you need numeric sorting.

Observed distribution across tested projects: the modal
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

### 7.43 An ended trial or inactive project stays fully readable — but reports nothing about its own date range

**A subscription state is not an access state.** When a trial ends, the project stops collecting new data; it does not stop serving the data it already holds. Verified four months after a trial ended, on a project `list_projects` reported as `TRIAL_ENDED`: the report tools, `list_chats` and `get_chat` all returned normally, with full answer text, sources carrying citation counts and positions, brand mentions and fan-out queries. Nothing was degraded, redacted or truncated. So before telling a user their data is gone because the plan lapsed, **probe the API** — the claim is cheap to check and expensive to get wrong.

Two flags are needed to see such a project at all, and both default to off:

- **`list_projects(include_inactive=true)`** — without it, an ended-trial project is simply absent from the listing, which reads exactly like "no such project" or "wrong account". With it, the project appears with its `status` (e.g. `TRIAL_ENDED`).
- **`list_chats(..., include_archived_prompts=true)`** — prompts belonging to a lapsed project are archived, and their chats are excluded from the default response. Without the flag the project looks live but empty, which is the §7.8 failure mode in its most convincing form.

**Retention after trial end is not documented. Pull promptly.** The observation above establishes that the data survived four months; it establishes nothing about month five. Treat any read from a lapsed project as a window that may close without notice: extract what the work needs in one pass and persist it to files (§7.32), rather than planning a workflow that re-queries the project over weeks.

**The project will not tell you its own date range.** A lapsed project exposes no "data available from X to Y" field, and the report tools require explicit `date_from`/`date_to` (§8) — so an over-wide range returns rows only for the period that has data, with no signal about where that period starts or ends, and a mis-centred range returns an empty envelope indistinguishable from the seven causes in §7.8. **Discover the window with a `week`-dimension probe:** request a report over a deliberately generous range with `date` (or `week`, where the dimension set offers it) as the dimension and a small limit, and read the first and last populated buckets off the result. That is one call, and it converts a guess into a measured window that every subsequent pull can be scoped to. The same probe is the honest way to answer "how much data is there?" for a live project whose start date nobody remembers.

**Labelling caveat.** Historical rows in such a project can carry model-channel ids that the project's *current* `list_model_channels` output no longer includes — see §7.6 for the resolution routes and the fallback id→name map. The configuration moved on; the rows did not.

---
