# Hidden features & power moves (§6)

Part of the **peec-ai-mcp** skill (CC BY 4.0 — Eoghan Henn / [rebelytics.com](https://www.rebelytics.com)). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read when a task needs gap filters, mentioned_brand_count, brand regex, the get_actions two-step workflow, wave-based bulk writes, or scraped page content via get_url_content.

---

## 6. Hidden features & power moves

### 6.1 Gap filter on domain/URL reports

The `filters` param on `get_domain_report` and `get_url_report` accepts a special `gap` operator not shown in the visible schema hints:

```json
{"field": "gap", "operator": "gte", "value": 1}
```

Returns domains/URLs where **competitors appear but the user's own brand doesn't**. Essential for competitive content audits.

"Appear" here means **in the cited page's own content**, not in the answers citing it (§7.38) — so a gap row is a claim about the page, usable directly once the per-project controls are run. Because that detection is plain substring matching on brand names and aliases, a brand whose name or alias is a dictionary word or a shared short acronym silently corrupts the filter in both directions: false own-brand mentions hide real gaps, false competitor mentions invent them. Check the roster for collisions and set word-boundary regexes (§6.3) **before** filtering on `gap`.

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

This workflow is reliably callable from Cowork and other pass-through MCP clients despite the tool's declared JSON schema still being empty. Schema-strict clients strip the `scope` parameter before sending — on those, the server rejects with *"No matching discriminator: scope"*. See §7.12 for the full behavioural matrix and when to fall back to the local approximation recipe in §8.8.

### 6.5 Wave-based execution for bulk config changes

When a tune-up touches many entities (tags, brands, prompts), execute in three waves rather than one sprint:

1. **Wave 1 — additive:** `create_tag`, `create_brand`. Zero risk; reversible by deletion. Confirm new entities appear in Peec UI before moving on.
2. **Wave 2 — mutation:** `update_prompt`, `update_topic`, `delete_brand`. Changes existing state. Pause to re-check UI.
3. **Wave 3 — creation:** `create_prompt`. Final bulk add. Resolve any new tag IDs from Wave 1 and substitute into the `tag_ids` array before calling.

Between waves, run the appropriate `list_*` tool to verify state. Use this pattern for any tune-up of more than ~20 operations.

**Intra-wave parallelism is safe for independent writes to different entities.** Within a single wave, operations that don't reference each other's outputs *and* don't target the same entity (e.g. 10 independent `create_tag` calls, or a batch of `create_prompt` calls whose tag IDs were resolved upstream) can be issued in parallel. Batches of 5–10 parallel MCP calls stay well under the published rate limit of **200 requests/minute per project** (see §7.24 and the official `/ratelimits` docs page). For inter-wave ordering, keep the strict sequence; for intra-wave independent writes to different entities, batch freely. **Do not parallelise multiple writes to the same entity** — especially `update_brand` calls against the same brand ID, which triggers a background recalculation and rejects concurrent updates with 409 (see §7.19).

**Pre-fetch tag and topic IDs once per wave.** When a wave will issue many writes that reference the same set of tag or topic IDs (common in Wave 3 prompt creates), call `list_tags` and `list_topics` once at the start of the wave and hold the ID map in-memory for the whole batch. Inter-wave refetches are only required if an intervening wave created new tags/topics.

The companion [`peec-ai-tracking-strategy-builder`](https://github.com/rebelytics/peec-ai-tracking-strategy-builder) skill codifies this into a full methodology — load it when the user wants to overhaul a Peec project, not just query it.

### 6.6 Scraped content via `get_url_content`

After `get_url_report`, feed interesting URLs into `get_url_content` to pull the actual markdown the AI engine was reading. Pass the URL verbatim — trailing slashes and scheme changes break the lookup.

`get_url_content` has **two distinct failure modes** that an agent has to dispatch on:

- **Failure mode A — URL not indexed by Peec at all:** the call returns an error response with the text `"URL not found"`. This is a hard miss — no retry helps. Skip and continue.
- **Failure mode B — URL indexed but not yet scraped:** the call returns a success envelope with `content: null`. Scraping runs up to 24h after first encounter. A retry later in the day often succeeds — queue for a deferred re-run rather than skipping.

An agent looping over `get_url_report` → `get_url_content` must branch on these two cases, not conflate them. The error-vs-null distinction is the reliable signal.

**Response payload fields.** A successful `get_url_content` response carries, in addition to the markdown body:

- `content_updated_at` — ISO timestamp of the last scrape for this URL. Use this, not "now", as the as-of date when quoting the page back to a user.
- `classification` — **domain-level** classification of the source (same enum as §7.30: `CORPORATE`, `EDITORIAL`, `INSTITUTIONAL`, `UGC`, `REFERENCE`, `COMPETITOR`, `OWN`, `OTHER`).
- `url_classification` — **page-level** classification of this specific URL (same enum as §7.31: `HOMEPAGE`, `CATEGORY_PAGE`, `PRODUCT_PAGE`, `LISTICLE`, `COMPARISON`, `PROFILE`, `DISCUSSION`, `HOW_TO_GUIDE`, `ARTICLE`, `OTHER`, `ALTERNATIVE`).

The dual classification matters: a page can be `EDITORIAL` at the domain level and `COMPARISON` at the page level, and the two enums never overlap. Don't conflate them; they answer different questions (who publishes this vs. what kind of page is it).

**5-day refresh cadence.** Source-URL page content is re-scraped every ~5 days, **not daily** (confirmed by Peec staff in the MCP challenge Slack channel and verified empirically across 8 URL samples — `content_updated_at` values cluster into 5-day buckets). Implications:

- For time-sensitive analysis (e.g. "what is this page saying right now?"), the content may be up to 5 days stale. Surface `content_updated_at` alongside any quoted content so the user knows the as-of date.
- Re-running `get_url_content` more frequently than every 5 days returns the same payload — don't build re-fetch loops tighter than that.
- A URL that returned `content: null` yesterday may return populated content today (first-encounter scrape can happen on any day); but once scraped, subsequent scrapes only happen on the 5-day cadence.

**Failure mode C — content is silently incomplete.** `get_url_content`
extracts page content via Mozilla Readability + Turndown GFM (per the
tool's own documentation). Readability is designed to isolate the
"main article" from surrounding chrome, which means it can — and
empirically does — silently strip mid-page sections that follow
promotional embeds, lazy-loaded blocks, or unusual HTML structures.
The output looks complete (no error, no `content: null`), but content
is missing. Observed in production: an editorial listicle page with
multiple regional / category sections separated by promotional
embeds; the cached markdown jumped from one section header to a
later one, omitting the section in between — including its full
brand list. A directly browser-verified view of the same URL showed
the missing section in place. The cache and the brand-detection
layer (`mentioned_brand_ids`) ran on the same truncated content, so
both reported the contained brands as absent.

**When this matters:** any time a downstream action depends on a
specific brand, fact, or item being present in the cached content. In
particular:

- Editorial **gap analysis** ("brand X is absent from this URL across
  N retrievals") — a `mentioned_brand_ids` set computed from
  Readability-stripped content can show a brand absent that is in fact
  prominently on the page.
- Pitch-target verification — proposing outreach to an editor on the
  basis that the brand is missing from their listicle, when the brand
  is in the actual rendered listicle but in the section Readability
  removed.
- Content-recipe analysis — drawing conclusions about what's "on" a
  page from an extraction that silently lost a section.

**Rule:** for action-driving claims based on URL content (especially
"brand X absent" gap claims that drive a stakeholder action item),
treat `get_url_content` output as a starting point, not as ground
truth. Browser-verify the URL in the rendered page before promoting
the gap claim to a deck action item or a pitch target. The cost of
one browser-verification call per candidate URL is dramatically lower
than the cost of an embarrassing wrong-action-item that turns out to
be based on a missing section.

This applies upstream of the deck-ready phase — see
peec-ai-tracking-strategy-builder §15.3 for the gate that codifies
the verification step at deck level.

---
