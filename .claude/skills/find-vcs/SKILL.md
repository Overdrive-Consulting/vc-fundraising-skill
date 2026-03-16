---
name: find-vcs
description: >
  Find venture capital firms that match a startup's fundraising brief.
  Searches VC databases and firm websites using WebSearch and Apify scraping.
  Triggers on: "find vcs", "find investors", "search for funds", "which vcs invest in",
  "step 2", "next step", "continue to find vcs".
---

# Find VCs

**Pipeline:** [clarify-thesis](./../clarify-thesis/SKILL.md) → **find-vcs** → [map-investors](./../map-investors/SKILL.md) → enrich-vc-contacts → score-vc-fit → draft-investor-outreach

You are step 2 of 6. Your job is to identify 15–30 candidate VC firms that plausibly match the startup's fundraising profile.

## Tools

- **Exa MCP (`web_search_exa`)** — primary VC database discovery
- **Exa MCP (`company_research_exa`)** — deep research on a specific firm: portfolio, thesis, recent activity
- **Exa MCP (`crawling_exa`)** — fetch and extract content from firm/portfolio pages
- **Apify MCP (`apify/cheerio-scraper`)** — fallback for structured portfolio page extraction (JS-light pages)
- **Apify MCP (`apify/website-content-crawler`)** — fallback for JS-heavy firm websites
- **WebSearch / WebFetch** — fallback for anything Exa doesn't cover

Prefer Exa over Apify for portfolio and firm research — it handles rendering and returns cleaner structured content. Use Apify only when Exa returns incomplete results for a specific page.

## Input

Read `data/normalized/company_profile.json`. If the file does not exist or required fields are null, stop and ask the founder to complete `clarify-thesis` first.

## Search strategy

### Step 0: Read the VC databases reference

Read [references/vc-databases.md](references/vc-databases.md) before searching. It lists recommended sources, search patterns, and Apify actors for each database.

### Step 1: Discover VC firms using Exa

Use `web_search_exa` to find candidate firms. Run at least 5 queries from different angles. Tailor to the startup's sector, stage, and geography.

| Source | Exa query |
|--------|-----------|
| OpenVC | `site:openvc.app [sector] [stage] investors` |
| Crunchbase | `site:crunchbase.com/organization [sector] VC fund [stage]` |
| Signal by NFX | `site:signal.nfx.com [sector] [stage]` |
| Notion VC lists | `"vc list" [sector] [stage] site:notion.so` |
| Airtable VC lists | `"vc list" [sector] [stage] site:airtable.com` |
| General | `[sector] seed fund OR venture capital investors [geography]` |
| Portfolio adjacency | `"[comparable_company]" investors OR "backed by" OR "portfolio"` |

Deduplicate by fund name across all queries.

### Step 2: Research each firm's portfolio using Exa

For each candidate firm found in Step 1, use `company_research_exa` to pull structured information:

```
company_research_exa("[fund_name] venture capital")
```

This returns: investment thesis, sector focus, stage, portfolio companies, check size signals, and recent activity — without needing to scrape individual pages.

Then use `crawling_exa` to fetch the firm's portfolio page directly for the full portfolio list:

```
crawling_exa("[fund_website]/portfolio")
```

or

```
crawling_exa("[fund_website]/companies")
```

Extract:
- Portfolio company names (to check for comparable_companies matches)
- Sector signals from portfolio descriptions
- Stage signals from portfolio company maturities
- Any stated check size or investment criteria

**Fallback to Apify** if Exa returns incomplete portfolio data for a specific firm:
- `apify/cheerio-scraper` — for structured portfolio grids on JS-light pages
- `apify/website-content-crawler` — for JS-heavy firm websites

### Step 3: Filter candidates

Exclude firms that are clearly out on:
- Stage (e.g. growth-stage only fund for a pre-seed company)
- Geography (e.g. LATAM-only fund for a US company)
- Sector (e.g. biotech-only fund for a SaaS company)
- Check size (e.g. $50M+ checks for a $500K raise)

Flag partial fits with a note in `possible_concerns`.

## What counts as a match

Evaluate each firm on:

- sector fit
- stage fit
- geography fit
- check size fit
- lead/follow fit
- portfolio adjacency (invested in comparable_companies or direct competitors)
- recent investment activity (last 24 months)
- thematic alignment with thesis_keywords
- explicit investor preferences from the profile

## Output

Write an array to `data/raw/fund_candidates.json` conforming to `schemas/fund_candidates.schema.json`.

Target 15–30 firms. Fewer is fine if quality is high. Do not pad with weak fits.

Assign each fund a `fund_id` as a lowercase hyphen-separated slug of the fund name (e.g. `sequoia-capital`). This is the canonical join key used in all later steps.

## Review checkpoint

After writing `data/raw/fund_candidates.json`, present a summary table to the founder:

```
Found [N] candidate VC firms for [company_name]:

| # | Fund | Stage | Sector focus | Check size | Confidence |
|---|------|-------|-------------|-----------|-----------|
| 1 | ... | ... | ... | ... | high/medium/low |
...

[If any partial fits]:
  ⚠ Partial fits flagged: [list with concerns]

[If evidence was thin for some firms]:
  Note: Evidence confidence is low for: [list] — worth verifying manually.

Does this list look right? You can:
  - Confirm to proceed to step 3 (mapping people at each firm)
  - Remove specific firms: "remove [fund name]"
  - Add a firm manually: "add [fund name] at [website]"
  - Narrow the list: "only keep firms with high confidence"
```

Do not proceed to `map-investors` until the founder confirms or adjusts the list.
