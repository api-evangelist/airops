---
name: aeo-opportunities
description: "Discovers AEO optimization opportunities and creates action grids. Use when a user wants to find content to refresh, topics to create content for, or citation opportunities to pursue for better AI visibility. Guides through three strategies: content refresh, content creation, and offsite citations. Requires AirOps MCP connection."
---

# AEO Opportunities & Actions

Discover how AI assistants cite your brand, find optimization opportunities, and create action grids to improve your AI visibility.

## Usage

```
/aeo-opportunities
```

Run the skill and follow the interactive flow. You will select a Brand Kit, scan for opportunities across three strategies, then create action grids for the most promising opportunities.

## Workflow

### Step 1: Brand Kit Selection

If a `brand_kit_id` is already known from the conversation (e.g. the user previously called `get_brand_kit` or mentioned a brand kit by name), use it and skip to Step 2.

Otherwise, call `list_brand_kits` to retrieve available brand kits. Present the list to the user and ask them to select one. If only one Brand Kit exists, confirm it with the user and proceed.

Store the selected `brand_kit_id` — it is required for all subsequent calls.

### Step 2: Quick Diagnostic Scan

Before running the scan, tell the user the default analysis period and let them choose:

*"The analysis will compare the last 30 days against the prior 30 days to detect trends. Would you like to use this default, or pick a different time frame?"*

Options:
- **Last 30 days** (default) — good for recent changes
- **Last 90 days** — better for trend detection
- **Custom range** — ask for `start_date` and `end_date` (ISO 8601)

If the user picks a non-default period, pass `start_date` and `end_date` to **all** data calls throughout the workflow — `list_pages`, `list_aeo_prompts`, `list_aeo_citations`, `list_aeo_domains`, and `query_analytics` all accept these parameters.

**Important: Use `brand_mentioned: false` for all prompt-based calls** (list_aeo_prompts, query_analytics). Branded prompts ("What is [BrandName]?") inflate mention rates because AI almost always mentions the brand when it's in the question. Non-branded gives the true picture. Only include branded if the user explicitly asks for a branded vs non-branded comparison.

Run these calls **in parallel** to detect which strategy has the strongest signal:

**Call A1 — losing AI visibility:**
```
list_pages(brand_kit_id, smart_filter: "losing_ai_visibility", per_page: 5)
```

**Call A2 — citation rate decline:**
```
list_pages(brand_kit_id, smart_filter: "citation_rate_decline", per_page: 5)
```

**Call A3 — almost page one:**
```
list_pages(brand_kit_id, smart_filter: "almost_page_one", per_page: 5)
```

**Call A4 — losing clicks:**
```
list_pages(brand_kit_id, smart_filter: "losing_clicks", per_page: 5)
```

**Call A5 — rankings slipping:**
```
list_pages(brand_kit_id, smart_filter: "rankings_slipping", per_page: 5)
```

**Call B — Content Creation signal:**
```
list_aeo_prompts(brand_kit_id, brand_mentioned: false, filters: [{field: "mention_rate", operator: "LEQ", value: 5}], sort: "-prompt_volume", per_page: 5)
```

**Call C — Offsite signal:**
```
list_aeo_citations(brand_kit_id, filters: [{field: "domain_category", operator: "IN", value: "Communities,Social"}], sort: "-citation_count", per_page: 5)
```

#### Detect GSC & GA4 availability

Use the **Call A1** response to detect integration status. The `list_pages` response includes GSC metrics (`position`, `clicks`, `impressions`) and GA4 metrics (`traffic`, `sessions`, `engagement`) — these fields return `null` when the integration is not connected.

Check the first returned page:
- **GSC connected** = `position` or `clicks` is non-null
- **GA4 connected** = `traffic` or `sessions` is non-null

Store these as flags (`has_gsc`, `has_ga4`) — they determine which sub-strategies and metrics are available.

If GSC is not connected, A2-A5 depend on GSC fields (`position`, `position_diff`, `clicks_diff`) and will return empty results — this is expected. Discard those results and note that these sub-strategies are unavailable.

Tell the user what integrations were detected:
- Both connected: *"GSC and GA4 are connected — full Page360 data is available. You'll get the richest analysis with organic search position, traffic, and citation data combined."*
- GSC only: *"GSC is connected but GA4 is not. Search position and click data is available, but traffic and engagement metrics are missing. Consider connecting GA4 for richer prioritization."*
- GA4 only: *"GA4 is connected but GSC is not. Traffic data is available, but search position data is missing. The position and click-based strategies won't be available. Consider connecting GSC."*
- Neither: *"Neither GSC nor GA4 is connected. Analysis will be based on AEO citation data only. Connect GSC and GA4 for much richer opportunities — especially 'almost page one' and traffic-based prioritization."*

#### Present diagnostic results

| Strategy | Signal | Count | Example |
|---|---|---|---|
| Content Refresh — losing AI visibility | Citations dropped ≥20% | (total from A1) | Top page URL |
| Content Refresh — citation rate decline | Citation rate dropped, position stable | (total from A2, or "N/A — no GSC") | Top page URL |
| Content Refresh — almost page one | Pages ranking 10-20 | (total from A3, or "N/A — no GSC") | Top page URL |
| Content Refresh — losing clicks | Clicks dropped, position stable | (total from A4, or "N/A — no GSC") | Top page URL |
| Content Refresh — rankings slipping | Position worsened ≥25% | (total from A5, or "N/A — no GSC") | Top page URL |
| Content Creation | Low-mention high-volume prompts | (total from B) | Top prompt text |
| Offsite / Citations | Community & social citations | (total from C) | Top citation URL |

Recommend the strategy with the strongest signal (highest count or most actionable results). Ask the user to confirm or pick a different strategy.

### Step 3: Deep Discovery

Run the discovery calls for the chosen strategy.

#### If Content Refresh:

1. `list_pages(brand_kit_id, smart_filter: "losing_ai_visibility", per_page: 20)` — pages losing citations
2. **If `has_gsc`:** `list_pages(brand_kit_id, smart_filter: "almost_page_one", per_page: 20)` — pages ranking 10-20 in search (requires GSC position data). **Skip if GSC is not connected** — this filter depends on `position` which will be null.
3. **If `has_gsc`:** Optionally also try `list_pages(brand_kit_id, smart_filter: "rankings_slipping", per_page: 10)` — pages declining in SERP position (bonus signal for content that needs refreshing).
4. For the top 5 pages from each list, call `get_page_prompts(brand_kit_id, page_id)` to understand which AI questions cite them.
5. **If `has_ga4`:** Use `traffic` and `sessions` from the page data to prioritize — pages with higher traffic that are losing visibility are more urgent to fix.

Group results by action type:
- **losing_ai_visibility** — pages with declining citations (always available)
- **almost_page_one** — pages near first-page ranking (GSC required — skip if not connected)
- **visibility_gap** — pages cited by prompts but with low mention rates (always available)

If neither GSC nor GA4 is connected, Content Refresh still works but is limited to citation-based signals only. Let the user know what additional opportunities they'd unlock by connecting these integrations.

#### If Content Creation:

1. `list_aeo_prompts(brand_kit_id, brand_mentioned: false, filters: [{field: "mention_rate", operator: "LEQ", value: 5}], sort: "-prompt_volume", per_page: 20)` — uncovered high-volume queries
2. `query_analytics(brand_kit_id, brand_mentioned: false, dimensions: ["competitor"], metrics: ["mention_rate"])` — which competitors dominate which topics
3. **Refresh vs Create check (mandatory):** For every keyword identified, call `list_pages` to search for existing content — filter by `primary_keyword` AND by `url` containing relevant terms. If a page already exists, it's a **refresh** opportunity, not a creation one. Only classify as new content creation when you've confirmed no page exists. Show your work: state what you searched for and what you found.

Prioritize by `relative_volume_score` — a prompt with 0.9 volume where you're not mentioned is a much bigger opportunity than one with 0.1 volume.

Group results by action type:
- **popular_query_gap** — high-volume prompts with zero or near-zero mention rate and no existing page (confirmed via search)
- **competitor_dominated_keyword** — prompts where competitors have significantly higher mention rates
- **blue_ocean** — prompts with low mention rates across all brands (no dominant player)

#### If Offsite / Citations:

1. `list_aeo_citations(brand_kit_id, filters: [{field: "domain_category", operator: "IN", value: "Communities,Social,Media"}], sort: "-citation_count", per_page: 20)` — community and social citations
2. `list_aeo_domains(brand_kit_id, sort: "-citation_count", per_page: 20)` — top non-owned citing domains
3. For the top 5 citations, call `get_aeo_citation(brand_kit_id, citation_id)` to see which prompts drive them

Group results by action type:
- **content_outreach** — non-Reddit URLs with high citation potential
- **community_outreach** — Reddit URLs specifically (required by validation)
- **highly_cited_thread** — threads with high citation rates
- **rising_third_party_content** — content with increasing citation rates over time

### Step 4: Present Opportunities

Present a table of discovered opportunities, grouped by action type. Include key metrics. Adapt columns based on available integrations.

**Skip root/home pages** — filter out any URL that is just the domain root (path is `/` or empty). Home pages are not actionable content optimization targets.

#### How to interpret the numbers

When presenting results, don't just show raw numbers — explain the pattern. Lead with mention rate and share of voice, follow with citation rate as supporting context.

**AI search metric patterns:**

| Pattern | What it means | Action |
|---|---|---|
| High mention, low citation | Brand is known but not linked to | Create more authoritative, linkable content so AI has something to cite |
| Low sentiment | AI talks about the brand negatively | Flag for messaging/PR review |
| Declining mention + stable citation | Losing mindshare but content still works | Brand awareness campaign needed |
| Declining citation + stable mention | Losing authority but still known | Content refresh — update outdated pages so AI trusts them again |

**Single-signal page patterns:**

| Pattern | What it means | Action |
|---|---|---|
| Losing AI citations (`losing_ai_visibility`) | AI is dropping references to this page | Content is going stale for AI. Refresh with current data, stats, examples. Check what AI is citing instead. |
| Losing clicks, stable position (`losing_clicks`) | Ranking hasn't changed but fewer people click | Something else is eating clicks — likely AI overviews or featured snippets. Review meta title/description. |
| Rankings slipping (`rankings_slipping`) | Position declining over time | Classic SEO issue — competitors may have fresher/better content. Audit and refresh. |
| Almost page one (`almost_page_one`) | Ranking #11-20 | Quick win — small improvements could push to page 1. Strengthen content, add internal links. |
| Citation rate decline, stable SEO (`citation_rate_decline`) | AI losing faith while Google still ranks it | Early warning. AI updates faster than Google. Refresh content now before both decline. |

**Multi-signal page patterns (when GSC + GA4 available):**

| AEO signal | GSC signal | GA4 signal | What's happening | Action |
|---|---|---|---|---|
| Citations dropping | Clicks dropping | Traffic dropping | Losing relevance across the board | Major content refresh or rewrite. Check what competitors are winning instead. |
| Citations dropping | Clicks stable | Traffic stable | AI moving away but Google still sends traffic | Page works for traditional search but isn't structured for AI citation. Update to be more authoritative/quotable. |
| Citations stable | Clicks dropping | Traffic dropping | AI still cites you but Google traffic falling | Traditional SEO issue — position may be slipping, or AI overviews are taking clicks. |
| Citations rising | Clicks stable | Traffic rising | AI is sending more traffic via citations | Page is winning in AI search. Study what makes it work and replicate the pattern. |
| Citations stable | Clicks rising | Traffic rising | Google sending more traffic, AI unchanged | Traditional SEO win. Focus AI optimization effort elsewhere. |
| No citations | High clicks | High traffic | Google loves this page but AI doesn't cite it | Opportunity — optimize this high-traffic page to be more citable by AI. |

**Benchmarks:** A 5-15% mention rate for non-branded prompts is good. Below 5% means significant room for improvement. Always factor in `relative_volume_score` when prioritizing — a high-volume prompt where you're not mentioned matters far more than a low-volume one.

**Content Refresh example (with GSC + GA4):**

| # | Action Type | URL | Citations | Position | Traffic | Questions |
|---|---|---|---|---|---|---|
| 1 | losing_ai_visibility | /blog/example | -15% | 8.3 | 1,200/mo | "What is…", "How to…" |
| 2 | almost_page_one | /docs/guide | 5 | 11.2 | 850/mo | "Best way to…" |

**Content Refresh example (AEO only — no GSC/GA4):**

| # | Action Type | URL | Citations Change | Prompts | Questions |
|---|---|---|---|---|---|
| 1 | losing_ai_visibility | /blog/example | -15% | 12 | "What is…", "How to…" |
| 2 | visibility_gap | /docs/guide | — | 8 | "Best way to…" |

**Content Creation example:**

| # | Action Type | Keyword | Volume | Mention Rate | Competitors |
|---|---|---|---|---|---|
| 1 | popular_query_gap | "ai seo tools" | 85 | 2% | — |
| 2 | competitor_dominated_keyword | "brand monitoring" | 72 | 3% | Competitor A (45%) |

**Offsite example:**

| # | Action Type | URL | Citation Rate | Domain | Questions |
|---|---|---|---|---|---|
| 1 | community_outreach | reddit.com/r/… | 12% | reddit.com | "What tools…" |
| 2 | highly_cited_thread | forum.example… | 8% | forum.example | "How do you…" |

Ask the user to confirm which opportunities to include in the action grid, or accept all.

### Step 5: Create Action Grid(s)

Map the confirmed opportunities into action grid rows. **Critical rules:**

1. **All row values must be strings** — convert numbers with `.toString()` or string interpolation
2. **One grid per action_type** — mixing types in one grid will fail validation
3. **Max 100 rows per grid**
4. **`questions` field** — comma-separated string of prompt texts, not an array
5. **`community_outreach` rows require Reddit URLs** — validate before including

For each action type with confirmed rows, call:

```
create_action_grid(brand_kit_id, action_type, rows, grid_name)
```

Use a descriptive `grid_name` like `"Content Refresh - Losing Visibility - Feb 2025"`.

#### Row Schema Reference

**Content Refresh grids:**

| action_type | Required | Optional |
|---|---|---|
| `losing_ai_visibility` | `url` | `folder`, `citation_loss_percent`, `questions` |
| `almost_page_one` | `url` | `folder`, `average_serp_position` |
| `visibility_gap` | `url` | `folder`, `questions` |

**Content Creation grids:**

| action_type | Required | Optional |
|---|---|---|
| `popular_query_gap` | `keyword` | `relative_volume_score`, `questions` |
| `competitor_dominated_keyword` | `keyword` | `mention_rate_percentage`, `mention_rate_gap_percentage`, `questions`, `competitors` |
| `blue_ocean` | `keyword` | `questions` |

**Offsite grids:**

| action_type | Required | Optional |
|---|---|---|
| `content_outreach` | `url` | `questions`, `competitors` |
| `community_outreach` | `url` | `questions`, `thread` |
| `highly_cited_thread` | `cited_url` | `citation_rate_percent`, `domain_url`, `questions` |
| `rising_third_party_content` | `cited_url` | `early_citation_rate_percent`, `recent_citation_rate_percent`, `domain_name`, `domain_url`, `questions` |

#### Field Mapping

**Content Refresh** (from `list_pages` / `get_page_prompts`):
- `url` → row `url`
- `folder_name` → row `folder`
- `citations_count_diff` → row `citation_loss_percent` (as string)
- `position` → row `average_serp_position` (as string)
- prompt `text` values → row `questions` (comma-separated string)

**Content Creation** (from `list_aeo_prompts` / `query_analytics`):
- prompt `keyword` or `text` → row `keyword`
- `relative_volume_score` → row `relative_volume_score` (as string)
- prompt `text` values → row `questions` (comma-separated string)
- competitor names from analytics → row `competitors` (comma-separated string)
- competitor `mention_rate` → row `mention_rate_percentage` (as string)

**Offsite** (from `list_aeo_citations` / `list_aeo_domains` / `get_aeo_citation`):
- citation `url` → row `url` or `cited_url`
- `citation_rate` → row `citation_rate_percent` (as string)
- `domain_name` → row `domain_name`
- domain `url` → row `domain_url`
- prompt texts from `get_aeo_citation` → row `questions` (comma-separated string)

### Step 6: Present Results

For each created grid, display:
- Grid name
- Action type
- Number of rows
- Grid URL (from the `create_action_grid` response)

Tell the user: **"Your action grids are ready. Open them in AirOps to run power agents on each row — they will generate optimized content, outreach drafts, or refresh recommendations based on the opportunities found."**

## Guidelines

- **Default to non-branded** — Always use `brand_mentioned: false` for prompt-based calls unless the user explicitly asks for branded analysis
- **Skip root pages** — Never include root/home page URLs in recommendations or action grids
- **Refresh before create** — Always check if a page exists before classifying an opportunity as content creation. This is mandatory, not optional.
- **Prioritize by volume** — Use `relative_volume_score` to rank opportunities. High volume + low mention = biggest opportunity.
- **Don't report raw numbers without context** — Always pair metrics with trends and benchmarks. "12% mention rate" means nothing without knowing the direction and what's normal.
- **Don't mix up mention and citation** — Mention = named in the response, citation = linked to. A brand can be mentioned without being cited.
- **Prefer parallel calls** — Steps 2 and 3 contain independent API calls that should run concurrently
- **Respect rate limits** — Use `per_page` parameters to limit result sets; start with 20 and paginate only if the user wants more
- **Be transparent about signals** — If a strategy shows no results, say so and suggest another
- **Validate before creating grids** — Confirm the row data matches the schema before calling `create_action_grid`
- **Name grids descriptively** — Include the strategy, action type, and date in the grid name
