---
name: reset
description: >
  Delete pipeline data files to start fresh or re-run from a specific step.
  Supports full reset or partial reset from a given step onwards.
  Triggers on: "reset", "start over", "clear data", "reset pipeline",
  "reset from step", "delete data", "fresh start", "wipe data".
---

# Reset Pipeline Data

Delete data files so the pipeline can be re-run from scratch or from a specific step.

## Files by step

| Step | Files produced | Path |
|------|---------------|------|
| 1 — clarify-thesis | company_profile.json | `data/normalized/` |
| 2 — find-vcs | fund_candidates.json | `data/raw/` |
| 3 — map-investors | investor_list.json | `data/raw/` |
| 4 — enrich-vc-contacts | investor_contacts.json | `data/normalized/` |
| 5 — score-vc-fit | scored_funds.json | `data/normalized/` |
| 6 — draft-investor-outreach | final_output.json, output.csv | `data/normalized/` |

## Reset modes

Ask the founder which reset they want before doing anything:

```
What would you like to reset?

  1. Full reset — delete all pipeline data (start completely fresh)
  2. From step 2 — keep company profile, re-run VC search onwards
  3. From step 3 — keep company profile + VC list, re-run people mapping onwards
  4. From step 4 — keep everything through map-investors, re-run contact enrichment onwards
  5. From step 5 — keep everything through enrichment, re-run scoring onwards
  6. From step 6 — keep everything through scoring, re-draft outreach only

Reply with the step number or "full".
```

Wait for the founder to reply before deleting anything.

## Files to delete per mode

| Mode | Files to delete |
|------|----------------|
| Full reset (1) | All files listed above |
| From step 2 | fund_candidates.json, investor_list.json, investor_contacts.json, scored_funds.json, final_output.json, output.csv |
| From step 3 | investor_list.json, investor_contacts.json, scored_funds.json, final_output.json, output.csv |
| From step 4 | investor_contacts.json, scored_funds.json, final_output.json, output.csv |
| From step 5 | scored_funds.json, final_output.json, output.csv |
| From step 6 | final_output.json, output.csv |

## Execution

After the founder confirms the reset mode, run the corresponding shell commands.

**Full reset:**
```bash
rm -f data/raw/fund_candidates.json \
       data/raw/investor_list.json \
       data/normalized/company_profile.json \
       data/normalized/investor_contacts.json \
       data/normalized/scored_funds.json \
       data/normalized/final_output.json \
       data/normalized/output.csv
```

**From step 2:**
```bash
rm -f data/raw/fund_candidates.json \
       data/raw/investor_list.json \
       data/normalized/investor_contacts.json \
       data/normalized/scored_funds.json \
       data/normalized/final_output.json \
       data/normalized/output.csv
```

**From step 3:**
```bash
rm -f data/raw/investor_list.json \
       data/normalized/investor_contacts.json \
       data/normalized/scored_funds.json \
       data/normalized/final_output.json \
       data/normalized/output.csv
```

**From step 4:**
```bash
rm -f data/normalized/investor_contacts.json \
       data/normalized/scored_funds.json \
       data/normalized/final_output.json \
       data/normalized/output.csv
```

**From step 5:**
```bash
rm -f data/normalized/scored_funds.json \
       data/normalized/final_output.json \
       data/normalized/output.csv
```

**From step 6:**
```bash
rm -f data/normalized/final_output.json \
       data/normalized/output.csv
```

Only delete files that actually exist — skip any that are already missing without error.

## After reset

Confirm to the founder what was deleted and what step to run next:

```
Reset complete. Deleted:
  - [list of deleted files]

Kept:
  - [list of kept files]

Next step: run [step name] to continue.
```
