---
name: score-vc-fit
description: >
  Score and rank VC firms and target investors based on fit with the startup's fundraising brief.
  Uses a weighted 8-dimension rubric. Produces a prioritized outreach list with clear reasoning.
  Triggers on: "score vcs", "rank investors", "evaluate fit", "step 5",
  "next step", "continue to score", "prioritize investors".
---

# Score VC Fit

**Pipeline:** clarify-thesis → find-vcs → map-investors → [enrich-vc-contacts](./../enrich-vc-contacts/SKILL.md) → **score-vc-fit** → [draft-investor-outreach](./../draft-investor-outreach/SKILL.md)

You are step 5 of 6. Your job is to turn the candidate list into a prioritized outreach list using a structured scoring rubric.

## Tools

None. This step is reasoning only — scoring based on data already collected in previous steps.

## Input

Read all three files:
- `data/normalized/company_profile.json` — the startup's fundraising profile
- `data/raw/fund_candidates.json` — candidate firms with rationale
- `data/normalized/investor_contacts.json` — enriched contacts

If any file is missing, stop and tell the founder which step to run first.

## Scoring rubric

Score each fund out of 100 using 8 weighted dimensions:

| Dimension | Max points | What to assess |
|-----------|-----------|----------------|
| Sector fit | 25 | Does the fund actively invest in this exact sector/subsector? |
| Stage fit | 20 | Does the fund's sweet spot match the raise stage? |
| Geography fit | 10 | Is the fund active in the startup's HQ geography or customer geography? |
| Check size fit | 10 | Does the fund's typical check size match the target? |
| Lead behavior fit | 10 | Does the fund's lead/follow style match the founder's preference? |
| Portfolio adjacency | 10 | Has the fund invested in comparable companies or adjacent spaces? |
| Recency | 10 | Has the fund made investments in the last 24 months (not just deploying old fund)? |
| Partner relevance | 5 | Is the identified contact person well-matched to this deal? |

**Score total** = sum of non-null sub-scores, rescaled to 100:
`score_total = (sum_of_non_null / score_max_possible) * 100`

**Missing evidence handling:**
- If evidence for a dimension is unavailable, score it `null` — do not score it 0
- `score_max_possible` = sum of max points for all non-null dimensions
- Note missing dimensions in `score_notes`
- If more than 3 dimensions are null, flag the record as low-confidence

**Score interpretation:**
- 90–100: excellent fit — prioritize first
- 75–89: strong fit — high priority
- 60–74: possible fit — worth pursuing if list is short
- below 60: weak fit — deprioritize or remove

**Scoring rules:**
- Be conservative — do not over-score weak evidence
- Do not reward prestige; reward actual fit evidence
- Penalize clear mismatches even if other dimensions are strong
- Distinguish "famous fund" from "right fund"

## Output

Write an array to `data/normalized/scored_funds.json` conforming to `schemas/fund_match.schema.json`.

Sort by `score_total` descending. Join investor_contact objects into each fund record using `fund_id`.

## Review checkpoint

After writing `data/normalized/scored_funds.json`, present the ranked list to the founder:

```
Ranked [N] VC firms for [company_name]:

| Rank | Fund | Score | Top contact | Sector | Stage | Concerns |
|------|------|-------|------------|--------|-------|---------|
| 1 | ... | 87/100 | ... | ✓ | ✓ | none |
| 2 | ... | 74/100 | ... | ✓ | ~ | check size unclear |
...

Score breakdown for top 5:
  [Fund name] — [score]/[max_possible]: sector [N], stage [N], geography [N], ...
  Score notes: [score_notes]

[If any funds scored below 60]:
  ⚠ [N] firms scored below 60 and will be deprioritized. Remove them entirely? (y/n)

[If many null dimensions]:
  Note: [N] firms had incomplete evidence — scores may shift with more research.

Does this ranking look right? You can:
  - Confirm to proceed to step 6 (drafting outreach messages)
  - Move a firm up/down: "prioritize [fund]" or "deprioritize [fund]"
  - Remove a firm: "remove [fund]"
  - Ask for more detail on a score: "explain score for [fund]"
```

Do not proceed to `draft-investor-outreach` until the founder confirms or adjusts the ranking.
