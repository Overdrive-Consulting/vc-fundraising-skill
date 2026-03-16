---
name: draft-investor-outreach
description: >
  Draft personalized investor outreach emails and LinkedIn messages using the startup thesis,
  investor fit rationale, and specific personalization hooks. Saves a final ranked outreach
  list and exports to CSV. Triggers on: "draft outreach", "write emails", "draft messages",
  "step 6", "next step", "continue to draft", "write investor emails", "draft linkedin messages".
---

# Draft Investor Outreach

**Pipeline:** clarify-thesis → find-vcs → map-investors → enrich-vc-contacts → [score-vc-fit](./../score-vc-fit/SKILL.md) → **draft-investor-outreach**

You are step 6 of 6. Your job is to draft a tailored email and LinkedIn message for each investor in the ranked list, then save the complete output.

## Tools

None. This step is reasoning + writing only.

## Input

Read `data/normalized/scored_funds.json`. If the file does not exist, stop and ask the founder to complete `score-vc-fit` first.

Also read `data/normalized/company_profile.json` for the startup voice and thesis.

Draft outreach only for firms and contacts that the founder confirmed in step 5. Skip any firms that were removed.

## Message principles

Every message must:
- Be tailored to this specific investor — not a template with a name swapped in
- Reference one real, specific reason this investor was selected
- Explain the company clearly in 1–2 sentences
- Make a simple, clear ask
- Avoid hype, generic praise, and fake familiarity

## Tone

- Concise
- Confident but not arrogant
- Specific — no filler
- Founder-like — sounds like a person wrote it
- Respectful of the investor's time

## Personalization rules

**Good personalization hooks:**
- "You invested in [portfolio company] — we're solving the same problem from the infrastructure layer"
- "Your thesis post on [topic] directly maps to what we're building"
- "You led [comparable company]'s seed — our model is similar but in [market]"
- Portfolio adjacency that is genuinely relevant, not superficial

**Bad personalization:**
- "I've long admired your work"
- "You're one of the top funds in [sector]"
- "I noticed you invest in B2B SaaS" (too generic)
- Anything that could apply to 50 other investors

## Output format

For each investor, write:

| Field | Description |
|-------|-------------|
| `personalization_hook` | The specific angle used (1 sentence) |
| `email_subject` | Short, specific, non-salesy |
| `email_body` | 3–5 sentences: who we are, why this investor, traction signal, ask |
| `linkedin_message` | Shorter version — 2–3 sentences max |
| `call_to_action` | Clear and simple: "Happy to share a brief deck — would a 20-min call make sense?" |
| `optional_follow_up_angle` | One angle for a follow-up if no reply in 1 week |

**Email rules:**
- Subject line: max 8 words, no clickbait, no ALL CAPS
- Body: 3–5 sentences. Don't over-explain. One traction signal if available.
- End with a simple, low-friction CTA

**LinkedIn message rules:**
- Max 300 characters
- No long background — get to the point immediately
- Still include one real reason for reaching out

**Rules:**
- Do not fabricate traction, portfolio connections, or relationships
- Do not claim a warm intro if none exists
- If there is not enough personalization data for a meaningful hook, note it and draft a simpler but honest message
- Keep messages distinct — each should feel written for that investor

## Saving output

Write the final ranked outreach array to `data/normalized/final_output.json` conforming to `schemas/final_output.schema.json`.

Field names map directly to CSV columns — use exact field names from the schema.

## Final summary to founder

After writing `data/normalized/final_output.json`, present the output:

```
Done. Here's your investor outreach list for [company_name]:

[N] personalized outreach drafts saved to data/normalized/final_output.json

Preview of top 5:

--- #1: [investor_name], [fund_name] (score: [score_total]/[score_max_possible]) ---
Subject: [outreach_subject_line]
Email:
[outreach_email_body]

LinkedIn:
[outreach_linkedin_dm]

Hook: [outreach_personalization_hook]
---

[Repeat for top 5]

Contact quality breakdown:
  ✓ [N] confirmed emails
  ~ [N] inferred emails (marked clearly in the file)
  ✗ [N] LinkedIn-only contacts

Next steps:
  1. Review and edit messages in data/normalized/final_output.json
  2. Use your email tool (Instantly, Apollo, Gmail) to send
  3. For inferred emails — verify before sending or use LinkedIn as primary channel
  4. For investors with no email — send via LinkedIn DM

Want any messages revised? Tell me which investor and what to change.
```

## CSV export

After the founder approves the messages, write `data/normalized/output.csv` by flattening `final_output.json`.

CSV column order:

```
priority_rank, fund_name, score_total, score_max_possible, investor_name, investor_title,
contact_tier, work_email, linkedin, outreach_subject_line, outreach_email_body,
outreach_linkedin_dm, outreach_personalization_hook, outreach_cta
```

Rules:
- Wrap any field containing commas or newlines in double quotes
- Escape internal double quotes as `""`
- Include a header row
- One row per investor (same as `final_output.json`)
- Do not include `fund_id` in the CSV — it is internal only

Confirm to the founder: `data/normalized/output.csv` written — [N] rows.

After writing the CSV, remind them:
- `data/normalized/output.csv` is ready to drop into Notion, Airtable, or an email tool
- `data/normalized/final_output.json` has the full structured output for re-processing
- All inferred emails are marked `contact_tier: "inferred"` — verify before sending
- Re-run `clarify-thesis` at any time to start a fresh campaign with a new fundraising profile
