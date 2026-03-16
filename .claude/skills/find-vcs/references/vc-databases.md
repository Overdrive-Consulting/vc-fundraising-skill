# VC Databases Reference

Sources and recommended Apify actors for finding VC firms. Used by `find-vcs` and `map-investors`.

---

## OpenVC

**URL:** https://openvc.app/vcs  
**What it has:** Curated list of VC funds with sector, stage, geography, check size, and direct application links. Self-reported by funds — generally accurate for active investors.  
**Best for:** Pre-seed and seed stage funds; funds actively looking for deal flow; geography-specific filtering.

**Search approach:**
- Browse by sector filter: `https://openvc.app/vcs?sector=[sector]`
- WebSearch: `site:openvc.app/vcs [sector] [stage]`
- WebFetch the results page directly and extract fund names, websites, and focus areas

**Apify actor:** `apify/cheerio-scraper` or `apify/website-content-crawler`  
**Scrape target:** `https://openvc.app/vcs` — paginated list with fund cards  
**Fields to extract:** fund name, website, focus sectors, stages, check size, geography, application link

---

## Crunchbase

**URL:** https://www.crunchbase.com  
**What it has:** Comprehensive fund database with investment history, portfolio companies, founding year, AUM estimates, partner names, and recent deals. Most reliable source for portfolio adjacency research.  
**Best for:** Verifying portfolio companies; checking recency of investments; finding comparable-company investors.

**Search approach:**
- WebSearch: `site:crunchbase.com/organization [fund name]` for specific funds
- WebSearch: `crunchbase "[sector]" "seed fund" OR "venture capital"` for discovery
- WebFetch individual fund pages: `https://www.crunchbase.com/organization/[fund-slug]`

**Apify actor:** `apify/website-content-crawler`  
**Scrape target:** Individual fund pages — `/organization/[slug]` — for portfolio and investment data  
**Note:** Crunchbase blocks aggressive scraping. Use WebFetch for individual pages rather than bulk scraping. Rate-limit requests.  
**Fields to extract:** fund description, investment stages, sectors, portfolio companies, recent investments, lead investor flag

---

## PitchBook

**URL:** https://pitchbook.com  
**What it has:** Most comprehensive VC/PE database. Fund size, LP info, deal history, partner backgrounds, return data. Paywalled for most data.  
**Best for:** Verifying check size ranges; finding fund vintage and deployment status; Series A+ research.

**Search approach:**
- Public pages are limited. Use WebSearch: `site:pitchbook.com [fund name]` for any indexed content
- Many PitchBook profiles are partially public — WebFetch the fund page for visible fields
- For paywalled content: fall back to Crunchbase or fund website

**Apify actor:** `apify/website-content-crawler`  
**Scrape target:** Public-facing fund pages only — do not attempt to bypass the paywall  
**Fields to extract:** fund name, website link, publicly visible investment focus, any portfolio companies listed

---

## Airtable VC Lists

**What they are:** Community-maintained spreadsheets of VC funds, often sector- or stage-specific. Published as public Airtable bases.  
**Best for:** Niche sectors (climate, web3, femtech, etc.) where curated community lists exist; finding funds that don't appear in mainstream databases.

**Search approach:**
- WebSearch: `"vc list" "[sector]" site:airtable.com`
- WebSearch: `"investor list" "[stage]" "[sector]" airtable`
- Common examples:
  - Climate/cleantech VCs: `"climate vc" list airtable`
  - Diversity-focused funds: `"diverse investors" list airtable`
  - Pre-seed specialists: `"pre-seed" investors airtable`

**Apify actor:** `apify/website-content-crawler` or `apify/cheerio-scraper`  
**Scrape target:** Public Airtable share URLs — `https://airtable.com/[base_id]/[view]`  
**Fields to extract:** fund name, website, focus, stage, check size, contact info (if listed)  
**Note:** Airtable public views render with JavaScript — `website-content-crawler` handles this better than `cheerio-scraper`.

---

## Notion VC Databases

**What they are:** Community-curated Notion pages listing VC funds, often with filtering by sector, stage, or geography. Frequently updated by founders and operators.  
**Best for:** Finding newer or emerging funds; sector-specific lists; funds actively seeking intros.

**Search approach:**
- WebSearch: `"vc list" "[sector]" site:notion.so`
- WebSearch: `"investor database" "[stage]" notion`
- Common examples:
  - `"pre-seed investors" list notion.so`
  - `"seed funds" "[sector]" notion.so`
  - `"vc database" geography notion.so`

**Apify actor:** `apify/website-content-crawler`  
**Scrape target:** Public Notion share URLs — `https://[workspace].notion.site/[page]`  
**Fields to extract:** fund name, website, focus sectors, stages, check size, contact or application link  
**Note:** Notion pages load with JavaScript. Use `website-content-crawler` with a reasonable wait time for JS rendering.

---

## VC firm websites (direct)

**What they are:** The fund's own website, which typically has portfolio, team, and thesis pages.  
**Best for:** Confirming sector/stage fit; extracting recent portfolio companies; identifying thesis language that maps to the startup.

### Primary: Exa `company_research_exa` + `crawling_exa`

Use `company_research_exa("[fund_name] venture capital")` first — it returns a structured summary of the firm's thesis, portfolio, stage focus, and recent activity without needing to navigate individual pages.

Then use `crawling_exa` to fetch the portfolio page directly:
```
crawling_exa("{fund_website}/portfolio")
crawling_exa("{fund_website}/companies")
```

Exa handles JS rendering and returns clean content. Prefer this over Apify for portfolio pages.

### Fallback: Apify

Use only when Exa returns incomplete results for a specific firm:
- `apify/cheerio-scraper` — faster for structured portfolio grids on JS-light pages
- `apify/website-content-crawler` — for complex JS-heavy firm websites

**Fields to extract from portfolio page:** company names, sectors, stages implied  
**Fields to extract from about/thesis page:** investment focus, check size, stage, geography

---

## LinkedIn (public profiles)

**What it has:** Individual investor profiles — current role, past roles, investment focus stated in bio, activity/posts.  
**Best for:** Confirming a person is currently at a firm; extracting sector focus from experience history; finding personalization hooks.

### Primary: Exa `people_search_exa`

Use the Exa MCP `people_search_exa` tool first. It does semantic people search without scraping LinkedIn directly.

Query pattern:
```
"[fund_name] venture capital" partners principals investors
```

Returns: name, title, LinkedIn URL, bio snippet. Fast, reliable, no account required.

### Secondary: Apify `harvestapi/linkedin-profile-search`

Actor: `harvestapi/linkedin-profile-search`  
**No cookies or LinkedIn account required.**

Use after Exa to get structured profile data (experience history, education, location) from names or LinkedIn URLs returned by Exa.

Input:
```json
{
  "queries": [{ "keywords": "[name] [fund_name]", "filters": { "currentCompany": "[fund_name]" } }],
  "maxResults": 3
}
```

Returns: full name, current title, company, location, work experience history, education, LinkedIn URL.

### Fallback: WebSearch

```
"[name]" "[fund name]" site:linkedin.com/in
```

Use only if Exa and Apify both return nothing.

---

## If an Apify actor is unavailable

Use the Apify MCP `actor-search` tool to find an alternative:

```
Search for: "web scraper [target site type]"
```

Or fall back to WebSearch + WebFetch for the specific pages needed. Most fund pages are readable via WebFetch without scraping.

---

## Source priority by use case

| Use case | Primary source | Fallback |
|----------|---------------|---------|
| Initial fund discovery | Exa `web_search_exa` (OpenVC, Crunchbase, Notion/Airtable) | WebSearch |
| Portfolio research per firm | Exa `company_research_exa` + `crawling_exa` | Apify `cheerio-scraper` |
| Portfolio adjacency check | Exa `company_research_exa` | Crunchbase via `crawling_exa` |
| Check size / stage verification | Exa `company_research_exa` | Crunchbase, PitchBook |
| Finding people at funds | Exa `people_search_exa` | `harvestapi/linkedin-profile-search` |
| Verifying current role | `harvestapi/linkedin-profile-search` | WebFetch LinkedIn profile |
| Recency of investment activity | Exa `company_research_exa` | Crunchbase recent deals |
| Niche sector funds | Exa `web_search_exa` (Airtable/Notion lists) | OpenVC sector filter |
