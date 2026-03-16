# vc-fundraising-skill

A Claude Code skill that helps founders research and target venture capital investors. Describe your startup and it walks you through finding aligned VC funds, mapping the right people, enriching contact details, scoring fit, and drafting personalized outreach — step by step.

## What it does

```
clarify-thesis → find-vcs → map-investors → enrich-vc-contacts → score-vc-fit → draft-investor-outreach
```

| Step | What happens | You review |
|------|-------------|-----------|
| 1. clarify-thesis | Asks targeted questions to build a structured fundraising brief | Confirm the brief before continuing |
| 2. find-vcs | Searches VC databases (OpenVC, Crunchbase, etc.) and scrapes firm websites | Approve / edit the list of 15–30 candidate firms |
| 3. map-investors | Finds the right partners and principals at each firm | Approve / edit the people list |
| 4. enrich-vc-contacts | Finds public emails and LinkedIn profiles | Review contact quality tiers |
| 5. score-vc-fit | Scores each firm on 8 dimensions and ranks the list | Confirm or reorder the ranking |
| 6. draft-investor-outreach | Drafts a tailored email + LinkedIn message for each investor | Edit and send |

## Requirements

Two API tokens are required before running the pipeline:

| Service | Used for | Get it |
|---------|---------|--------|
| **Apify** | Scraping VC firm websites, team pages, portfolio pages, LinkedIn profiles, and bulk email finding | [apify.com](https://apify.com) → Console → Settings → API tokens |
| **Exa** | Semantic people search — finding the right partners and principals at each firm | [exa.ai](https://exa.ai) → Dashboard → API keys |

Both tokens must be set before opening the repo in Claude Code. The pipeline will not work without them.

## Setup

### 1. Get your API tokens

- **Apify:** sign up at [apify.com](https://apify.com), go to Console → Settings → API tokens
- **Exa:** sign up at [exa.ai](https://exa.ai), go to Dashboard → API keys

### 2. Set your tokens

```bash
export APIFY_TOKEN=your_apify_token_here
export EXA_API_KEY=your_exa_api_key_here
```

Add both lines to your `.bashrc` or `.zshrc` to persist them across sessions:

```bash
echo 'export APIFY_TOKEN=your_apify_token_here' >> ~/.zshrc
echo 'export EXA_API_KEY=your_exa_api_key_here' >> ~/.zshrc
source ~/.zshrc
```

### 3. Open the repo in Claude Code

```bash
cd vc-fundraising-skill
claude
```

Claude Code will automatically pick up the Apify and Exa MCP servers from `.mcp.json`.

### 4. Start the pipeline

Say:

> "I'm fundraising and want to find relevant VCs. My company does [brief description]."

Or simply:

> "Start vc fundraising"

The pipeline will guide you through each step, pause for your review at every checkpoint, and save structured outputs to `data/` as you go.

## Outputs

All outputs are saved in `data/`:

| File | Contents | Step |
|------|---------|------|
| `data/normalized/company_profile.json` | Structured fundraising brief | After step 1 |
| `data/raw/fund_candidates.json` | Candidate VC firms with rationale | After step 2 |
| `data/raw/investor_list.json` | Individual investors at each firm | After step 3 |
| `data/normalized/investor_contacts.json` | Enriched contacts with emails + LinkedIn | After step 4 |
| `data/normalized/scored_funds.json` | Scored and ranked firms with breakdowns | After step 5 |
| `data/normalized/final_output.json` | Ranked outreach list with drafted messages | After step 6 |

The `final_output.json` file contains everything you need to send outreach — investor name, contact details, subject line, email body, and LinkedIn message.

## Notes

- Only public professional contact information is used
- Inferred emails are clearly labeled as `contact_tier: "inferred"` — verify before sending
- Investors with no email are still included for LinkedIn outreach
- Each step waits for your confirmation before proceeding
- You can re-run any step individually without restarting the whole pipeline

---

Built by [Overdrive](https://enteroverdrive.com)
