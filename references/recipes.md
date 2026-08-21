# Common recipes (§8)

Part of the **peec-ai-mcp** skill (CC BY 4.0 — Eoghan Henn / [rebelytics.com](https://www.rebelytics.com)). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read when running a composite analysis — full visibility report, per-engine comparison, competitive gap analysis, source-authority audit, safe test-entity lifecycle, and the other multi-step workflows.

**Contents:**

- 8. Common recipes
  - 8.0 "Give me a full visibility report" (meta-recipe)
  - 8.1 "How visible is my brand this month?"
  - 8.1b Combined / group visibility — how to compute a multi-brand visibility figure correctly
  - 8.2a Domain-level competitor analysis — "who's beating us at the source level?"
  - 8.2b URL-level competitive gap analysis — "which specific pages should we target?"
  - 8.3 "What's the AI engine actually searching for?"
  - 8.4 Fix the competitor list after project creation
  - 8.5 Add a tagged prompt set
  - 8.6 Audit what AI is recommending as products
  - 8.7 Full project tune-up (dry-run first)
  - 8.8 Approximate `get_actions` locally (fallback recipe)
  - 8.9 Full competitive gap analysis (composite)
  - 8.10 Source authority audit (composite)
  - 8.11 Test entity lifecycle (safe experimentation pattern)

---

## 8. Common recipes

The recipes below collectively cover Peec's six published primary use cases (per `docs.peec.ai/mcp/use-cases`): brand visibility monitoring (§8.1), competitive intelligence (§8.2a / §8.2b / §8.4 / §8.9), source authority audits (§8.2a / §8.10), prompt optimisation (§8.5, plus `/peec_prompt_grader`), action-driven SEO (§8.2b / §8.8, plus `/peec_source_authority`), and cross-platform benchmarking (§8.1 with `dimensions=[model_id]`). Recipe §8.7 (full project tune-up) wraps the atomics into a single overhaul workflow.

**Start here for the most common request type.** When a user says "give me a visibility report", "full report", "monthly report", or anything comparable, reach for the composite meta-recipe in **§8.0** first — it sequences the atomic recipes below into the three common depth tiers (quick / standard / deep).

**Composite recipes** answer recurring multi-step questions that the atomics only partially cover: **§8.8** is a local fallback for `get_actions` when the native tool is unavailable (e.g. schema-strict client strips params — see §7.12); **§8.9** is the full competitive gap analysis flow (strategic view, not just a URL list); **§8.10** is a source-authority audit (who's citing whom, and at what rate); **§8.11** is the safe test-entity lifecycle pattern for agent-driven experimentation.

**Reporting convention — split prompt-level stats branded vs non-branded by default.** Before reporting any prompt-level proportion (visibility, mention rate, fanout coverage, "N of M prompts show X"), check `list_tags` for a branded/non-branded taxonomy on the project. If the tags exist, report the split unprompted — don't wait to be asked. A blended aggregate hides the number stakeholders actually want: e.g. "89% of prompts show third-party directory mentions in fanout" can decompose into ~91% on non-branded prompts but only ~60% on branded ones — materially different stories, and the non-branded (discovery) rate is usually the one that matters strategically. Reserve the blended figure for projects whose tag taxonomy doesn't support the split, and say so when you do.

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

### 8.1b Combined / group visibility — how to compute a multi-brand visibility figure correctly

**Use when:** the user wants a single visibility number for a group of brands (own brand + sister brands + acquired brands), e.g. *"what's our group visibility?"*, *"combined share for our brand family"*, *"Portfolio X total mention share."*

**The wrong answer (common):** sum per-brand visibility percentages across the group. *"Own brand 35% + sister 31% = group 66%"* — **mathematically wrong** in every case where a single chat can mention more than one of the grouped brands, because the rate metrics share a denominator and overlap on any chat where two brands co-appear. The sum is always an upper bound; the actual combined figure is between the single-brand max (35% here) and the sum. See §7.37 on Ratio-type metrics and the hard rule in `peec-ai-tracking-strategy-builder` §14.2.

**The right answer:** query Peec directly with a combined-brand filter.

```
get_brand_report(
  project_id,
  filters=[{field: "brand_id", operator: "in", values: [brand_id_1, brand_id_2, brand_id_3]}],
  start_date, end_date,
  dimensions=[]      # undimensioned — or add model_id for per-engine split
)
→ visibility = chats in which ANY of the filtered brands appeared / total chats
```

Peec handles the chat-level union internally. The returned `visibility` is the correct combined-group figure — 35–66% in the example above, not 66%. The call also returns SoV, sentiment, position, mention counts for the group as a whole.

**Alternative: compute the union manually.** Where the `in` filter isn't usable (old client shim, a brand entity not yet created in Peec, computing across project boundaries):

```
1. list_chats(project_id, brand_id=brand_id_1, limit=10000) → set A of chat_ids
2. list_chats(project_id, brand_id=brand_id_2, limit=10000) → set B of chat_ids
3. list_chats(project_id, brand_id=brand_id_3, limit=10000) → set C of chat_ids
4. union = A ∪ B ∪ C
5. total_chats = total in window (undimensioned brand report total, any brand filter)
6. combined_visibility = |union| / total_chats
```

The manual computation gives the same answer as the combined-filter call, at the cost of three more API round trips. Prefer the combined-filter call when Peec supports it.

**Applies to every rate metric, not just visibility.** The same problem — denominator shared across the cohort, overlap on chats where multiple brands co-appear — appears for any "% of chats" metric: SoV, retrieval share, citation share, mention rate. Apply the combined-filter pattern (or the union-of-chat-IDs pattern) for all of them.

**Does NOT apply to:** sentiment and position, which are per-mention aggregates without a shared denominator. Averaging sentiment across a group is a different kind of wrong (unweighted mean of unequal mention counts) but the overlap error described here doesn't hit them. Compute group sentiment / position as weighted means over the union of mentions, not as cross-brand sums.

**Reporting rule:** when a deliverable contains a group visibility figure, include a one-sentence provenance note naming the brand IDs that went into the filter (or the union) — so the stakeholder or a reviewing agent can check the composition. *"Group visibility (OwnBrand + SisterBrand + AcquiredBrand, April 1–30): 52%"* is defensible; *"Group visibility: 52%"* with no composition note is not.

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

A tune-up is a systematic overhaul of an under-performing Peec project: replacing wrong brands, retagging prompts, deleting non-commercial prompts, adding revenue-informed new prompts. Load the companion `peec-ai-tracking-strategy-builder` skill for the methodology.

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

**What this recipe does NOT do** (and why that's OK): it doesn't produce Peec's exact `url_classification` action types (OWNED/EDITORIAL/TECHNICAL). It produces the two categories that matter most in practice (EDITORIAL, OWNED) and skips TECHNICAL — which covers crawlability/indexing issues that usually come from external SEO tooling (Screaming Frog, Sitebulb, or equivalent crawlers) anyway, not from Peec data.

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
