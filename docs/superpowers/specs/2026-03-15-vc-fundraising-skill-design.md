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
│   │   │   ├── thesis-intake.md             ← structured intake questions
│   │   │   ├── scoring-rubric.md            ← weighted scoring instructions
│   │   │   └── output-format.md             ← output shape definitions
│   │   ├── schemas/
│   │   │   ├── company_profile.schema.json
│   │   │   ├── fund_match.schema.json
│   │   │   └── investor_contact.schema.json
│   │   └── rubrics/
│   │       └── vc-fit-rubric.md             ← scoring weights and criteria
│   └── agents/
│       ├── thesis-normalizer.md
│       ├── fund-researcher.md
│       ├── investor-mapper.md
│       ├── contact-enricher.md
│       └── outreach-ranker.md
├── scripts/
│   ├── validate-output.ts
│   └── export-csv.ts
└── data/
    ├── raw/
    └── normalized/
```

---

## Workflow

Linear, founder-driven pipeline:

```
clarify-thesis → find-vcs → enrich-vc-contacts → score-vc-fit → draft-investor-outreach
```

Each step is handled by a dedicated subagent. The main `SKILL.md` orchestrates delegation.

---

## Apify Integration

The official Apify MCP server (`@apify/actors-mcp-server`) is installed and configured in `.claude/settings.json`. No custom MCP code is needed.

Subagents (`fund-researcher`, `contact-enricher`) instruct Claude to use the Apify MCP tools at runtime to discover and invoke the most relevant actors. If the user provides a pinned actor list, the skills reference those IDs directly.

**Pattern:**
1. Claude identifies candidate funds (via WebSearch + reasoning)
2. Apify scrapes firm/team/portfolio pages (via MCP)
3. Claude normalizes results against schemas
4. Claude ranks targets using scoring rubric
5. Output written to `data/normalized/`

**Apify use cases:**
- VC firm website scraping
- Team page scraping
- Portfolio page scraping
- Public work email discovery
- Recent investment extraction

**Not used for:** investor quality judgment — Claude handles that.

---

## Subagents

### `thesis-normalizer`
- **Input:** raw founder text
- **Output:** `company_profile.schema.json`-conformant JSON
- **Tools:** none (reasoning only)
- **Model:** sonnet

### `fund-researcher`
- **Input:** structured company profile
- **Output:** list of candidate funds with preliminary rationale
- **Tools:** WebSearch, WebFetch, Apify MCP
- **Model:** sonnet
- **Search dimensions:** sector, stage, geography, check size, lead behavior, recency, portfolio adjacency

### `investor-mapper`
- **Input:** fund list
- **Output:** relevant people per fund (partner/principal/investor/associate)
- **Tools:** WebSearch, WebFetch, Apify MCP
- **Model:** sonnet

### `contact-enricher`
- **Input:** investor list
- **Output:** `investor_contact.schema.json`-conformant JSON with confidence labels
- **Tools:** WebSearch, WebFetch, Apify MCP
- **Model:** sonnet
- **Policy:** public professional info only; inferred emails clearly labeled

### `outreach-ranker`
- **Input:** enriched + scored investor list
- **Output:** ranked list with per-investor outreach drafts
- **Tools:** none (reasoning only)
- **Model:** sonnet

---

## Scoring Rubric

Weighted score out of 100:

| Dimension | Weight |
|---|---|
| Sector fit | 25% |
| Stage fit | 20% |
| Geography fit | 10% |
| Check size fit | 10% |
| Lead behavior fit | 10% |
| Portfolio adjacency | 10% |
| Recency of activity | 10% |
| Partner relevance / contactability | 5% |

---

## Output Shape

### 1. Structured thesis summary
```json
{
  "company_name": "ExampleCo",
  "sector": ["climate fintech"],
  "stage": "seed",
  "geo": ["US", "Canada"],
  "check_size_target_usd": [500000, 2000000],
  "investor_preferences": {
    "lead": true,
    "thesis_keywords": ["carbon markets", "SMB finance", "climate data"],
    "avoid": ["late-stage only", "consumer-only"]
  }
}
```

### 2. Fund match
```json
{
  "fund_name": "Fund X",
  "score": 87,
  "why_match": [
    "Invests in seed climate software",
    "Led 4 North American deals in past 18 months",
    "Portfolio overlap with adjacent workflow tools"
  ],
  "evidence": {
    "website": "...",
    "recent_investments": ["..."]
  }
}
```

### 3. Investor contact
```json
{
  "name": "Jane Doe",
  "title": "Partner",
  "firm": "Fund X",
  "location": "New York",
  "linkedin": "...",
  "work_email": "jane@fundx.com",
  "email_confidence": 0.72,
  "source_type": "public/probabilistic",
  "reason_to_contact": "Leads seed climate infra investments"
}
```

### 4. Outreach (per investor)
- Subject line
- Short email
- LinkedIn DM
- Personalization hook
- CTA

---

## Intake Requirements

Collect or infer before running:

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

**Good clarifying questions** (ask only what changes search logic):
- Pre-seed, seed, or Series A?
- Lead investors or co-investors?
- Geography of customer base or HQ?
- Target check size?
- Sector: B2B, B2C, fintech, climate, devtools, deeptech, etc.?
- Active investors only (last 24 months)?
- Prioritize portfolio adjacency?

---

## Data Flow

```
[founder input]
      ↓
thesis-normalizer → data/normalized/company_profile.json
      ↓
fund-researcher   → data/raw/fund_candidates.json
      ↓
investor-mapper   → data/raw/investor_list.json
      ↓
contact-enricher  → data/normalized/investor_contacts.json
      ↓
outreach-ranker   → data/normalized/final_output.json
      ↓
scripts/export-csv.ts → data/normalized/output.csv
```

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
| Contact quality is inconsistent | Separate public/confirmed, inferred, and no-contact-found tiers |
| Over-scraping noise | Search narrowly after thesis normalization |
| Privacy / compliance | Public professional info only; disclose source and confidence |
| Generic VC matching | Score by weighted rubric + recency, not sector tags alone |
