---
name: map-investors
description: >
  Find the right individual people at each candidate VC firm — partners, principals,
  and associates relevant to the startup's sector and stage. Uses Exa people search
  and LinkedIn Profile Search Scraper (harvestapi) for accurate, no-cookie profile lookup.
  Triggers on: "map investors", "find people at funds", "who to contact", "step 3",
  "next step", "continue to map investors", "find partners at".
---

# Map Investors

**Pipeline:** clarify-thesis → [find-vcs](./../find-vcs/SKILL.md) → **map-investors** → [enrich-vc-contacts](./../enrich-vc-contacts/SKILL.md) → score-vc-fit → draft-investor-outreach

You are step 3 of 6. Your job is to identify the best 1–3 people to contact at each firm from `fund_candidates.json`.

## Tools

- **Exa MCP (`people_search_exa`)** — primary: semantic people search by name, company, and role
- **Apify MCP (`harvestapi/linkedin-profile-search`)** — structured LinkedIn profile data once names are known
- **WebSearch / WebFetch** — fallback for firms with no Exa results

## Input

Read `data/raw/fund_candidates.json`. If the file does not exist, stop and ask the founder to complete `find-vcs` first.

Also read `data/normalized/company_profile.json` for sector/stage context — this determines which partners are most relevant.

## What to do

For each fund in `fund_candidates.json`, run the following steps in order.

### Step 1: Find people using Exa

Use the Exa MCP `people_search_exa` tool to find investment professionals at the firm.

Query pattern:
```
"[fund_name] venture capital" partners principals investors
```

Or more targeted:
```
"[fund_name]" "[sector]" partner OR principal OR investor
```

Exa returns people with names, titles, LinkedIn URLs, and sometimes bios. Extract:
- Full name
- Title
- LinkedIn URL (if returned)
- Any sector or thesis signals from the bio

Run 1–2 queries per firm. If Exa returns strong results, proceed to Step 2. If Exa returns nothing useful, fall back to Step 3.

### Step 2: Pull structured profile data using Apify

Once you have names (and optionally LinkedIn URLs) from Exa, use the Apify actor `harvestapi/linkedin-profile-search` to get structured profile data.

Actor: `harvestapi/linkedin-profile-search`

Input pattern (search by name + company):
```json
{
  "queries": [
    {
      "keywords": "[name] [fund_name]",
      "filters": {
        "currentCompany": "[fund_name]"
      }
    }
  ],
  "maxResults": 3
}
```

Or if you already have a LinkedIn URL from Exa, pass it directly to get richer profile data (experience history, education, location).

This actor requires no cookies and no LinkedIn account. It returns:
- Full name
- Current title and company
- Location
- Work experience history
- Education
- LinkedIn profile URL

Use this to confirm the person is currently at the firm (not a past role) and to extract their sector focus from experience history.

### Step 3: Fallback — WebSearch + WebFetch

If Exa returns nothing and Apify returns nothing for a firm:

1. WebSearch: `"[fund_name]" team OR partners site:[fund_website]`
2. WebFetch the team page directly
3. Extract names and titles from the page text

Use this approach sparingly — it is less reliable than Exa + Apify.

## Person selection priority

Pick people in this order:

1. Partner / General Partner
2. Principal
3. Investor / Investment Team Member
4. Associate

Include up to 3 people per firm. If only 1 strong fit exists, include just that person.

Only include operating partners, platform team, or talent team if they are clearly sector-relevant.

## Relevance signals

Prefer people who:

- Have invested in the startup's sector before (visible in experience history or bio)
- Focus on the relevant stage
- Are based in the startup's target geography
- Have written or spoken publicly about the relevant thesis area (check Exa bio snippets)

Deprioritize people who:
- Are listed as "advisor" or "venture partner" with no recent deal activity
- Joined the firm very recently with no visible investment history
- Are clearly focused on an unrelated sector

## Output

Write an array to `data/raw/investor_list.json` conforming to `schemas/investor_list.schema.json`.

Use the same `fund_id` slugs from `fund_candidates.json` as the foreign key.

## Review checkpoint

After writing `data/raw/investor_list.json`, present a summary to the founder:

```
Mapped [N] investors across [M] firms for [company_name]:

| Fund | Name | Title | LinkedIn | Source |
|------|------|-------|---------|--------|
| ... | ... | ... | yes/no | exa / linkedin-search / web |
...

[If some firms had no relevant contacts found]:
  ⚠ No contacts found for: [list of fund names] — these firms may still be worth targeting via their general contact page.

[If LinkedIn was not found for some]:
  Note: LinkedIn missing for [N] contacts — enrich-vc-contacts will attempt to find them.

Does this look right? You can:
  - Confirm to proceed to step 4 (enriching contact details)
  - Remove a person: "remove [name] from [fund]"
  - Add a person manually: "add [name], [title] at [fund]"
  - Ask to find more people at a specific firm: "find more people at [fund]"
```

Do not proceed to `enrich-vc-contacts` until the founder confirms or adjusts the list.
