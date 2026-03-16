---
name: export-csv
description: >
  Convert any or all pipeline JSON files to CSV. Handles flattening of nested objects.
  Saves CSV files alongside the JSON files in the same directory.
  Triggers on: "export csv", "convert to csv", "export data", "download csv",
  "export all", "get csv", "export contacts", "export funds", "export results".
---

# Export CSV

Convert pipeline JSON files to CSV. Each file gets its own CSV saved in the same folder.

## Available exports

| File | CSV output | Contents |
|------|-----------|---------|
| `data/normalized/company_profile.json` | `data/normalized/company_profile.csv` | Single-row summary of the fundraising brief |
| `data/raw/fund_candidates.json` | `data/raw/fund_candidates.csv` | One row per candidate VC firm |
| `data/raw/investor_list.json` | `data/raw/investor_list.csv` | One row per mapped investor |
| `data/normalized/investor_contacts.json` | `data/normalized/investor_contacts.csv` | One row per enriched contact with email/LinkedIn |
| `data/normalized/scored_funds.json` | `data/normalized/scored_funds.csv` | One row per fund with score breakdown flattened |
| `data/normalized/final_output.json` | `data/normalized/final_output.csv` | One row per investor with outreach drafts (same as output.csv from step 6) |

## What to ask

If the founder does not specify which file(s) to export, ask:

```
Which files would you like to export to CSV?

  1. All available files
  2. fund_candidates — candidate VC firms
  3. investor_list — mapped people at each firm
  4. investor_contacts — enriched contacts with emails
  5. scored_funds — ranked firms with score breakdown
  6. final_output — full outreach list with drafted messages
  7. company_profile — your fundraising brief

Reply with a number, a comma-separated list (e.g. "3,4"), or "all".
```

Only export files that exist. Skip and note any that are missing.

## CSV column definitions

### fund_candidates.csv

```
fund_id, fund_name, website, location, stage_focus, sector_focus,
check_size_min_usd, check_size_max_usd, lead_behavior,
recent_investments, preliminary_rationale, possible_concerns, evidence_confidence
```

- `stage_focus` and `sector_focus`: join array items with ` | ` (e.g. `seed | series-a`)
- `recent_investments`: join array with ` | `
- `check_size_range_usd[0]` → `check_size_min_usd`, `[1]` → `check_size_max_usd`

### investor_list.csv

```
fund_id, name, title, firm, location, linkedin, firm_profile_url, why_relevant
```

### investor_contacts.csv

```
fund_id, name, title, firm, location, linkedin, firm_profile_url,
work_email, email_confidence, source_type, contact_tier, reason_to_contact
```

### scored_funds.csv

Flatten `score_breakdown` into individual columns:

```
fund_id, fund_name, score_total, score_max_possible,
score_sector_fit, score_stage_fit, score_geography_fit,
score_check_size_fit, score_lead_behavior_fit, score_portfolio_adjacency,
score_recency, score_partner_relevance,
score_notes, why_match, risks
```

- `score_breakdown.sector_fit` → `score_sector_fit`, etc.
- `why_match`: join array with ` | `
- `risks`: join array with ` | `
- Do not include the nested `investors` array — use `investor_contacts.csv` for that

### final_output.csv

```
priority_rank, fund_name, score_total, score_max_possible,
investor_name, investor_title, contact_tier, work_email, linkedin,
outreach_subject_line, outreach_email_body, outreach_linkedin_dm,
outreach_personalization_hook, outreach_cta
```

This is the same as `output.csv` written at the end of step 6.

### company_profile.csv

Single row. Flatten the object:

```
company_name, one_line_thesis, sector, stage, geo, business_model,
check_size_min_usd, check_size_max_usd, total_round_target_usd,
traction_summary, comparable_companies,
investor_lead, investor_thesis_keywords, investor_avoid, warm_intro_preferred,
assumptions_made, open_questions
```

- Arrays: join with ` | `
- `check_size_target_usd[0]` → `check_size_min_usd`, `[1]` → `check_size_max_usd`
- `investor_preferences.lead` → `investor_lead`, etc.

## How to write the CSV

Always write the Python conversion script to a `.py` file first, then execute it. Never use a bash heredoc (`<<'EOF'`) to run Python inline — it triggers a false-positive safety warning due to curly braces in dict comprehensions.

**Correct pattern:**
1. Write the script to `scripts/export_csv.py` using the file-write tool
2. Run it: `python3 scripts/export_csv.py`
3. Delete the script after it runs (optional)

**Wrong pattern** (do not use):
```bash
python3 - <<'EOF'
...script with {curly braces}...
EOF
```

## CSV formatting rules

- Always include a header row
- Wrap any field containing commas, newlines, or double quotes in double quotes
- Escape internal double quotes as `""`
- Use UTF-8 encoding
- Null values → empty cell (not the string "null")

## After export

List each file written with its row count:

```
Exported:
  ✓ data/raw/fund_candidates.csv — 24 rows
  ✓ data/normalized/investor_contacts.csv — 67 rows
  ✓ data/normalized/final_output.csv — 67 rows

Skipped (file not found):
  - data/normalized/scored_funds.json
```
