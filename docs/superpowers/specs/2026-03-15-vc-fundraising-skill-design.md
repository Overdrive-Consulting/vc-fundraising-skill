# VC Fundraising Skill — Design Spec
**Date:** 2026-03-15
**Status:** Approved

---

## Overview

A Claude Code skill that helps founders research and target venture capital investors. The founder provides their company thesis; the skill identifies aligned VC funds, enriches investor contacts, scores fit, and drafts personalized outreach.

---

## Repo Structure

```
vc-fundraising-claude/
├── CLAUDE.md
├── README.md
├── .claude/
│   ├── settings.json                        ← configures Apify MCP server
│   ├── skills/vc-fundraising/
│   │   ├── SKILL.md                         ← entry point, delegation logic
│   │   ├── prompts/
│   │   │   ├── thesis-intake.md             ← interactive intake questions run by SKILL.md before invoking thesis-normalizer
│   │   │   ├── scoring-rubric.md            ← weighted scoring instructions
│   │   │   └── output-format.md             ← output shape definitions
│   │   ├── schemas/
│   │   │   ├── company_profile.schema.json
│   │   │   ├── fund_candidates.schema.json  ← output of fund-researcher
│   │   │   ├── investor_list.schema.json    ← output of investor-mapper
│   │   │   ├── fund_match.schema.json       ← output of fund-scorer (scored fund)
│   │   │   ├── investor_contact.schema.json ← output of contact-enricher
│   │   │   └── final_output.schema.json     ← output of outreach-ranker
│   │   └── rubrics/
│   │       └── vc-fit-rubric.md             ← scoring weights, criteria, missing-evidence rules
│   └── agents/
│       ├── thesis-normalizer.md
│       ├── fund-researcher.md
│       ├── investor-mapper.md
│       ├── contact-enricher.md
│       ├── fund-scorer.md
│       └── outreach-ranker.md
├── scripts/
│   ├── validate-output.ts     ← validates JSON files against schemas; exits non-zero on failure
│   └── export-csv.ts          ← exports data/normalized/final_output.json to data/normalized/output.csv
└── data/
    ├── raw/
    │   ├── fund_candidates.json
    │   └── investor_list.json
    └── normalized/
        ├── company_profile.json
        ├── investor_contacts.json
        ├── scored_funds.json
        └── final_output.json
```

---

## Workflow

Linear, founder-driven pipeline. Each step maps 1:1 to a subagent.

| Step | Subagent | Reads | Writes |
|---|---|---|---|
| clarify-thesis | thesis-normalizer | founder text (interactive) | `data/normalized/company_profile.json` |
| find-vcs | fund-researcher | `company_profile.json` | `data/raw/fund_candidates.json` |
| map-investors | investor-mapper | `fund_candidates.json` | `data/raw/investor_list.json` |
| enrich-vc-contacts | contact-enricher | `investor_list.json` | `data/normalized/investor_contacts.json` |
| score-vc-fit | fund-scorer | `company_profile.json` + `fund_candidates.json` + `investor_contacts.json` | `data/normalized/scored_funds.json` |
| draft-investor-outreach | outreach-ranker | `scored_funds.json` | `data/normalized/final_output.json` |

**Note:** `find-vcs` (user-facing step name) is implemented by `fund-researcher` + `investor-mapper` in sequence. SKILL.md runs both when the user invokes `find-vcs`.

---

## Apify Integration

The official Apify MCP server (`@apify/actors-mcp-server`) is installed and configured in `.claude/settings.json`. No custom MCP code is needed.

### settings.json template

```json
{
  "mcpServers": {
    "apify": {
      "command": "npx",
      "args": ["-y", "@apify/actors-mcp-server"],
      "env": {
        "APIFY_TOKEN": "${APIFY_TOKEN}"
      }
    }
  }
}
```

`APIFY_TOKEN` must be set in the shell environment or a `.env` file before running Claude Code.

### Recommended Apify actors (defaults)

Subagent prompts reference these actor IDs as defaults; the user may override with a pinned list.

| Use case | Recommended actor |
|---|---|
| VC firm website scraping | `apify/website-content-crawler` |
| Team / people page extraction | `apify/web-scraper` |
| Portfolio page extraction | `apify/cheerio-scraper` |
| Public email discovery | `apify/hunter-email-finder` (or equivalent) |
| LinkedIn-like public profiles | `apify/linkedin-profile-scraper` (or equivalent) |

If a recommended actor is unavailable, subagents use the Apify MCP `actor-search` tool to find an alternative.

### Apify pattern

1. Claude identifies candidate funds (WebSearch + reasoning)
2. Apify scrapes firm/team/portfolio pages (via MCP)
3. Claude normalizes results against schemas
4. Claude ranks targets using scoring rubric
5. Output written to `data/normalized/`

**Not used for:** investor quality judgment — Claude handles that.

---

## Subagents

### `thesis-normalizer`
- **Input:** structured text collected by `thesis-intake.md` prompt; SKILL.md runs the intake conversation with the user first, then passes the collected fields as plain text to this subagent
- **Output:** `company_profile.schema.json`-conformant JSON written to `data/normalized/company_profile.json`
- **Tools:** none (reasoning only)
- **Model:** sonnet
- **Re-prompting:** does not re-prompt the user; if a required field is missing, it marks it `null` and SKILL.md asks the user to supply it before continuing

### `fund-researcher`
- **Input:** `data/normalized/company_profile.json`
- **Output:** `fund_candidates.schema.json`-conformant array written to `data/raw/fund_candidates.json`
- **Tools:** WebSearch, WebFetch, Apify MCP
- **Model:** sonnet
- **Search dimensions:** sector, stage, geography, check size, lead behavior, recency, portfolio adjacency

### `investor-mapper`
- **Input:** `data/raw/fund_candidates.json`
- **Output:** `investor_list.schema.json`-conformant array written to `data/raw/investor_list.json`
- **Tools:** WebSearch, WebFetch, Apify MCP
- **Model:** sonnet
- **Focus:** partner, principal, investor, associate (in priority order)

### `contact-enricher`
- **Input:** `data/raw/investor_list.json`
- **Output:** `investor_contact.schema.json`-conformant array written to `data/normalized/investor_contacts.json`
- **Tools:** WebSearch, WebFetch, Apify MCP
- **Model:** sonnet
- **Policy:** public professional info only; all records include `contact_tier` field

### `fund-scorer`
- **Input:** `data/normalized/company_profile.json` + `data/raw/fund_candidates.json` + `data/normalized/investor_contacts.json`
- **Output:** `fund_match.schema.json`-conformant array written to `data/normalized/scored_funds.json`
- **Tools:** none (reasoning only)
- **Model:** sonnet
- **Scoring:** applies weighted rubric; produces sub-scores + total

### `outreach-ranker`
- **Input:** `data/normalized/scored_funds.json`
- **Output:** `final_output.schema.json`-conformant array written to `data/normalized/final_output.json`
- **Tools:** none (reasoning only)
- **Model:** sonnet
- **Produces:** ranked list with per-investor outreach drafts

---

## Scoring Rubric

Weighted score out of 100. Each dimension produces a sub-score (0–max), which sum to the total.

| Dimension | Weight | Max points |
|---|---|---|
| Sector fit | 25% | 25 |
| Stage fit | 20% | 20 |
| Geography fit | 10% | 10 |
| Check size fit | 10% | 10 |
| Lead behavior fit | 10% | 10 |
| Portfolio adjacency | 10% | 10 |
| Recency of activity | 10% | 10 |
| Partner relevance / contactability | 5% | 5 |

### Missing evidence handling

If `fund-researcher` cannot find evidence for a dimension:
- Score that dimension `null` (not 0)
- Do not renormalize weights
- The `fund-scorer` must note the missing dimension in `score_notes`
- `validate-output.ts` accepts `null` sub-scores but flags records where more than 3 dimensions are null

---

## Schemas

### `company_profile.schema.json`
```json
{
  "company_name": "string",
  "one_line_thesis": "string",
  "sector": ["string"],
  "stage": "pre-seed | seed | series-a",
  "geo": ["string"],
  "business_model": "string",
  "check_size_target_usd": [number, number],
  "investor_preferences": {
    "lead": "boolean",
    "thesis_keywords": ["string"],
    "avoid": ["string"],
    "warm_intro_preferred": "boolean"
  },
  "comparable_companies": ["string"]
}
```

### `fund_candidates.schema.json` (array of)
```json
{
  "fund_id": "string (slugified fund name, e.g. 'sequoia-capital')",
  "fund_name": "string",
  "website": "string",
  "location": "string",
  "stage_focus": ["string"],
  "sector_focus": ["string"],
  "check_size_range_usd": [number, number],
  "lead_behavior": "lead | co-lead | follow | unknown",
  "recent_investments": ["string"],
  "preliminary_rationale": "string"
}
```

`fund_id` is assigned by `fund-researcher` as a lowercase, hyphen-separated slug of `fund_name`. It is the canonical join key used throughout the pipeline.

### `investor_list.schema.json` (array of)
```json
{
  "fund_id": "string (foreign key → fund_candidates.fund_id)",
  "name": "string",
  "title": "string",
  "firm": "string",
  "location": "string",
  "linkedin": "string | null",
  "firm_profile_url": "string | null"
}
```

### `investor_contact.schema.json` (array of)
```json
{
  "fund_id": "string (foreign key → fund_candidates.fund_id)",
  "name": "string",
  "title": "string",
  "firm": "string",
  "location": "string",
  "linkedin": "string | null",
  "firm_profile_url": "string | null",
  "work_email": "string | null",
  "email_confidence": "number (0.0–1.0) | null",
  "source_type": "public | probabilistic | not_found",
  "contact_tier": "confirmed | inferred | not_found",
  "reason_to_contact": "string"
}
```

**Contact tiers:**
- `confirmed` — email found directly on a public professional page
- `inferred` — email constructed from pattern (firstname@firm.com); confidence < 1.0
- `not_found` — no email found; record still included with null email fields

### `fund_match.schema.json` (array of)
```json
{
  "fund_id": "string (foreign key → fund_candidates.fund_id)",
  "fund_name": "string",
  "score_total": "number (0–100, rescaled to 100 from evidenced dimensions only)",
  "score_max_possible": "number (sum of weights for non-null dimensions, e.g. 75 if 3 dimensions null)",
  "score_breakdown": {
    "sector_fit": "number | null",
    "stage_fit": "number | null",
    "geography_fit": "number | null",
    "check_size_fit": "number | null",
    "lead_behavior_fit": "number | null",
    "portfolio_adjacency": "number | null",
    "recency": "number | null",
    "partner_relevance": "number | null"
  },
  "score_notes": "string",
  "why_match": ["string"],
  "risks": ["string"],
  "investors": ["investor_contact object (joined by fund_id)"]
}
```

**Score computation:** `score_total` = sum of non-null sub-scores, rescaled to 100 by: `(sum_of_non_null / score_max_possible) * 100`. `score_max_possible` is the sum of the max points for all non-null dimensions. Both values are stored so consumers can assess confidence. `fund-scorer` documents the effective max in `score_notes` (e.g., "Scored out of 75 — geography and recency data unavailable").

### `final_output.schema.json` (array of)
```json
{
  "priority_rank": "number",
  "fund_id": "string",
  "fund_name": "string",
  "score_total": "number",
  "score_max_possible": "number",
  "investor_name": "string",
  "investor_title": "string",
  "contact_tier": "confirmed | inferred | not_found",
  "work_email": "string | null",
  "linkedin": "string | null",
  "outreach_subject_line": "string",
  "outreach_email_body": "string",
  "outreach_linkedin_dm": "string",
  "outreach_personalization_hook": "string",
  "outreach_cta": "string"
}
```

Outreach fields are flattened (prefixed `outreach_`) so `export-csv.ts` maps each field directly to a CSV column with no nesting. Column names match field names exactly.

---

## Data Flow

```
[founder conversation via thesis-intake.md]
      ↓
thesis-normalizer → data/normalized/company_profile.json
      ↓
fund-researcher   → data/raw/fund_candidates.json
      ↓
investor-mapper   → data/raw/investor_list.json
      ↓
contact-enricher  → data/normalized/investor_contacts.json
      ↓
fund-scorer       → data/normalized/scored_funds.json
      ↓
outreach-ranker   → data/normalized/final_output.json
      ↓
export-csv.ts     → data/normalized/output.csv
```

---

## Intake Requirements

SKILL.md runs `thesis-intake.md` interactively. It collects the following fields before invoking `thesis-normalizer`. Missing required fields block progress; recommended fields proceed with a warning.

| Field | Required |
|---|---|
| Company name | yes |
| One-line thesis | yes |
| Stage | yes |
| Geography (HQ + customer base) | yes |
| Sector / sub-sector | yes |
| Business model | yes |
| Target check size | yes |
| Lead vs. follow preference | yes |
| Comparable companies | recommended |
| Must-have investor traits | recommended |
| Must-avoid investor traits | recommended |
| Warm intro preference | recommended |

**Good clarifying questions** (ask only what changes search logic):
- Pre-seed, seed, or Series A?
- Lead investors or co-investors?
- Geography of customer base or HQ?
- Target check size?
- Sector: B2B, B2C, fintech, climate, devtools, deeptech, etc.?
- Active investors only (last 24 months)?
- Prioritize portfolio adjacency?

---

## Scripts

### `validate-output.ts`
- Validates JSON files in `data/raw/` and `data/normalized/` against their corresponding schemas
- Exits with code 1 and prints validation errors if any file fails
- Exits with code 0 and prints "All outputs valid" if all pass
- Run after any pipeline step: `npx ts-node scripts/validate-output.ts`

### `export-csv.ts`
- Reads `data/normalized/final_output.json`
- Flattens each record into a flat CSV row using the `final_output.schema.json` column mapping
- Writes `data/normalized/output.csv`
- Run after the full pipeline: `npx ts-node scripts/export-csv.ts`

---

## CLAUDE.md Rules

```markdown
# VC Fundraising Repo Rules

You help identify venture capital investors that fit a startup's fundraising profile.

- Ask clarifying questions only when missing information materially affects investor targeting.
- Prefer structured outputs over long prose.
- Prefer recent, evidence-backed investor activity.
- Only use public professional contact information.
- Clearly label inferred emails as inferred.
- Rank investors by fit, not just by sector keywords.
- Prefer partners and principals over generic firm contacts.
- When drafting outreach, keep tone concise, specific, and credible.
- Avoid hypey language and generic praise.
```

---

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Bad thesis in → bad investor list out | Force structured intake first; block on missing required fields |
| Contact quality is inconsistent | Three-tier system: confirmed / inferred / not_found; all tiers included in output |
| Over-scraping noise | Search narrowly after thesis normalization; use pinned actor IDs when provided |
| Privacy / compliance | Public professional info only; disclose source type and email confidence on every record |
| Generic VC matching | Score by weighted rubric + recency, not sector tags alone |
| Apify actor discovery fails at runtime | Recommended default actor IDs documented; subagents fall back to actor-search tool |
| Partial pipeline failure | Pipeline continues with null fields; validate-output.ts flags records with 3+ null score dimensions |
| Missing evidence in scoring | Null sub-scores allowed; noted in score_notes; not treated as 0 |
