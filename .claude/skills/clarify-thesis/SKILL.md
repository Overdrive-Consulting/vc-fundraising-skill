---
name: clarify-thesis
description: >
  Turn a startup description into a structured fundraising brief for VC targeting.
  Ask only the clarifying questions that materially affect investor targeting.
  Triggers on: "start fundraising", "find investors", "raise capital", "clarify thesis",
  "I'm fundraising", "help me raise", "find VCs", "we're raising a round".
---

# Clarify Thesis

**Pipeline:** clarify-thesis → [find-vcs](./../find-vcs/SKILL.md) → map-investors → enrich-vc-contacts → score-vc-fit → draft-investor-outreach

You are step 1 of 6. Your job is to turn a founder's description into a structured fundraising brief so the pipeline has a clean, complete profile to work from.

## Environment

Before starting, check that APIFY_TOKEN is set (needed for later steps):
```
test -n "$APIFY_TOKEN" && echo "set" || echo "missing"
```
If missing, ask the founder to run `export APIFY_TOKEN=<your_token>` before proceeding past step 2.

## Tools

None. This step is conversation + reasoning only.

## Input

The founder's description of their company — can be a deck summary, website copy, raw notes, or a direct message. If nothing is provided, ask: "Tell me about your company and what you're raising."

## Intake conversation

Collect the following fields. Ask only if the answer is not inferable from what the founder has shared.

**Required fields** (block progress if missing):

| Field | Good clarifying question |
|-------|--------------------------|
| Company name | — |
| One-line thesis | "How would you describe what your company does in one sentence?" |
| Stage | "What stage are you raising at — pre-seed, seed, or Series A?" |
| Geography | "Where is your company based, and which geographies are your target customers?" |
| Sector / sub-sector | "What sector does your company operate in — e.g. B2B SaaS, fintech, climate, devtools, deeptech?" |
| Business model | "How does the company make money?" |
| Target check size | "What check size are you targeting from a lead investor?" |
| Lead vs. follow | "Are you looking for investors that lead rounds, co-lead, or are strong co-investors acceptable?" |

**Recommended fields** (proceed with a warning if missing):

| Field | Good clarifying question |
|-------|--------------------------|
| Comparable companies | "What companies would you compare yourselves to — especially ones that have raised from VCs you admire?" |
| Must-have investor traits | "Are there traits you need in an investor — e.g. deep sector expertise, operator background, specific geography?" |
| Must-avoid investor traits | "Any investor types or firms you want to avoid?" |
| Traction summary | "What does your traction look like — ARR, users, pilots, contracts, notable customers?" |
| Warm intro preference | "Do you have warm intro access to any VC networks, or are you open to cold outreach?" |

**Good clarifying questions:**
- One at a time
- Specific, not open-ended
- Only when the answer would materially change the investor targeting

**Bad clarifying questions:**
- Repeating what the founder already said
- Nice-to-know but not targeting-relevant
- Vague discovery questions

## Output

Write a JSON file to `data/normalized/company_profile.json` conforming to `schemas/company_profile.schema.json`.

If a recommended field is missing, include it as `null` and note it in `open_questions`.
If you infer something not explicitly stated, include it in `assumptions_made`.

## Review checkpoint

After writing `data/normalized/company_profile.json`, present the structured brief to the founder for review:

```
Here's your fundraising brief:

  Company: [company_name]
  Thesis: [one_line_thesis]
  Stage: [stage]
  Sector: [sector]
  Geography: [geo]
  Check size: [check_size_target_usd]
  Lead preference: [lead]
  Comparables: [comparable_companies]
  Traction: [traction_summary]

[If any required fields are null]:
  ⚠ Missing required fields: [list]. Please provide these before we continue.

[If any assumptions were made]:
  Assumptions I made: [assumptions_made]

[If any recommended fields are null]:
  Note: [open_questions] — we can proceed without these but targeting may be broader.

Does this look right? Confirm to move to step 2 (finding VC firms), or correct anything above.
```

Do not proceed to `find-vcs` until the founder confirms or corrects the brief.
