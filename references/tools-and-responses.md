# The tool surface (§3–§5)

Part of the **peec-ai-mcp** skill (CC BY 4.0 — Eoghan Henn / [rebelytics.com](https://www.rebelytics.com)). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read before your first Peec tool call of the session — tool selection, write-safety classes, response format, pagination defaults, and the slash-command prompts.

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
| `list_models` | AI engine catalog (see §7: `is_active` filter is critical; now deprecated — prefer `list_model_channels`) |
| `list_model_channels` | Per-channel engine catalogue (regional/setting variants behind each model) — a later addition to the MCP surface; the preferred channel-resolution tool (see §7.1 / §7.6) |
| `list_chats` | Individual AI responses, filterable by brand/prompt/model |
| `get_chat` | Full chat payload (messages, sources, products, brands_mentioned) |
| `list_search_queries` | Sub-queries the AI engine fanned out to |
| `list_shopping_queries` | Shopping-mode queries + product listings |
| `get_brand_report` | Visibility, SoV, sentiment, position aggregates |
| `get_domain_report` | Source-domain retrieval + citation rates |
| `get_url_report` | Source-URL retrieval + citation rates + page classification |
| `get_url_content` | Scraped markdown of any indexed source URL |
| `get_actions` | Opportunity-scored recommendations — two-step workflow (`scope=overview` then drill down). Callable from pass-through clients despite empty declared schema; see §6.4 / §7.12 |

**Drift note on the counts.** The read-only table above now lists 16 tools: `list_model_channels` appeared on the MCP surface after this skill's initial tool census, and `list_models`' own tool description now marks it "Deprecated — prefer list_model_channels". The "27 tools" / "15 read-only" framing used throughout this skill predates that addition. As always, the authoritative catalogue is what the server announces via `tools/list` on connection (§7.25) — re-verify counts there before quoting them.

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

**Tag colour enum (22 values):** `gray`, `red`, `orange`, `yellow`, `lime`, `green`, `cyan`, `blue`, `purple`, `fuchsia`, `pink`, `emerald`, `amber`, `violet`, `indigo`, `teal`, `sky`, `rose`, `slate`, `zinc`, `neutral`, `stone`. The tool's JSON schema is the authoritative source if Peec adds colours later; this list matches the schema at last verification.

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
- **`totalCount` is now standard on `list_*` responses.** Alongside `rowCount`, paginated `list_*` tools carry a `totalCount` field with the full row count for the query — the "optional, on some report tools" framing above predates this. This enables a cheap **`limit=1` totalCount probe**: call the tool with `limit=1` and read `totalCount` to size a dataset (e.g. learn a project has 2,000+ fanout rows) for the cost of a single row, before deciding on a pagination or overflow strategy (§7.18, §7.32).
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
