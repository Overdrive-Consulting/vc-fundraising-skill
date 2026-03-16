---
name: enrich-vc-contacts
description: >
  Enrich public professional contact details for each mapped investor.
  Uses icypeas_official/bulk-email-finder for batch email lookup,
  and Exa for any missing LinkedIn URLs. Clearly labels all inferred data.
  Triggers on: "enrich contacts", "find emails", "get contact details", "step 4",
  "next step", "continue to enrich contacts".
---

# Enrich VC Contacts

**Pipeline:** clarify-thesis → find-vcs → [map-investors](./../map-investors/SKILL.md) → **enrich-vc-contacts** → [score-vc-fit](./../score-vc-fit/SKILL.md) → draft-investor-outreach

You are step 4 of 6. Your job is to enrich contact details for each person in `investor_list.json` — primarily emails, with LinkedIn URLs as fallback.

## Tools

- **Apify MCP (`icypeas_official/bulk-email-finder`)** — primary: batch email lookup by name + company domain
- **Exa MCP (`people_search_exa`)** — fill in any missing LinkedIn URLs
- **WebSearch / WebFetch** — fallback for checking firm bio pages

## Input

Read `data/raw/investor_list.json`. If the file does not exist, stop and ask the founder to complete `map-investors` first.

## Step 1: Prepare the prospect list

From `investor_list.json`, build a list of `{ name, domain }` pairs — one per investor.

Extract the domain from `fund_website` in `fund_candidates.json` (e.g. `sequoiacap.com` from `https://www.sequoiacap.com`). Strip `www.` and the protocol.

The actor accepts up to 5,000 rows. Format as CSV:
```
firstName,lastName,domain
Sarah,Smith,sequoiacap.com
James,Lee,benchmark.com
```

Split first and last name from the `name` field. If only a single name token is available, put it in `firstName` and leave `lastName` empty.

## Step 2: Run bulk email finder

Run Apify actor `icypeas_official/bulk-email-finder` with the prospect CSV.

The actor returns:
- `email` — found email address
- `confidence` — confidence score (0.0–1.0)
- `status` — `found`, `not_found`, or `risky`

Map results back to investors by `firstName + lastName + domain`.

Assign `contact_tier` based on the result:

| Actor status | confidence | contact_tier | email_confidence |
|-------------|-----------|-------------|-----------------|
| `found` | ≥ 0.8 | `confirmed` | actor value |
| `found` | < 0.8 | `inferred` | actor value |
| `not_found` or `risky` | — | `not_found` | null |

## Step 3: Fill missing LinkedIn URLs

For any investor still missing a `linkedin` URL after `map-investors`:

Use Exa MCP `people_search_exa`:
```
"[name]" "[fund_name]" venture capital
```

Extract the LinkedIn URL from the result. If not found, leave `linkedin: null` — the investor can still be contacted via their firm contact page.

## Step 4: Check firm bio pages (spot check only)

For investors where both email and LinkedIn are still missing after Steps 2–3:

WebSearch: `"[name]" site:[firm_website]`

Or WebFetch `{firm_website}/team` and scan for the person's name. Some firms list emails directly on their team page — if found, set `contact_tier: "confirmed"`.

This step is a spot check, not a full scrape. Do not spend time on firms where Step 2 already returned results.

## Contact information rules

**Allowed:**
- Email returned by `icypeas_official/bulk-email-finder` with status `found` → tier based on confidence
- Email found directly on a public firm page → `contact_tier: "confirmed"`, `email_confidence: 1.0`
- Public LinkedIn URL
- Firm bio page URL

**Not allowed:**
- Personal email addresses
- Non-public phone numbers
- Presenting a low-confidence email as confirmed

## Output

Write an array to `data/normalized/investor_contacts.json` conforming to `schemas/investor_contact.schema.json`.

Every record must include `contact_tier` and `source_type`, even if email is not found.
Records with `contact_tier: "not_found"` are still included — they are contacted via LinkedIn.

Set `source_type`:
- `"public"` — email found on a firm page
- `"probabilistic"` — email returned by bulk-email-finder
- `"not_found"` — no email found

## Review checkpoint

After writing `data/normalized/investor_contacts.json`, present a summary to the founder:

```
Enriched [N] contacts across [M] firms for [company_name]:

Contact quality breakdown:
  ✓ Confirmed email (high confidence): [N] contacts
  ~ Inferred email (lower confidence): [N] contacts
  ✗ No email found — LinkedIn only: [N] contacts

| Fund | Name | Email | Confidence | LinkedIn | Tier |
|------|------|-------|-----------|---------|------|
| ... | ... | ... | 0.92 | yes/no | confirmed |
...

[If many low-confidence emails]:
  Note: [N] emails have confidence below 0.8. Treat as best-guess — verify before sending or use LinkedIn as primary channel.

[If many not_found]:
  Note: [N] contacts have no email. LinkedIn DM is the recommended channel for these.

Does this look right? You can:
  - Confirm to proceed to step 5 (scoring fit and ranking)
  - Correct an email: "update email for [name] to [email]"
  - Remove a contact: "remove [name]"
  - Skip enrichment for a firm: "skip [fund name]"
```

Do not proceed to `score-vc-fit` until the founder confirms or adjusts the contacts.
