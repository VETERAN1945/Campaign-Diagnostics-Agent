# Campaign Diagnostic Agent

A ReAct agent built on n8n that investigates "why did campaign X's metrics drop" on its own: it looks at the stats → forms hypotheses → checks them one tool at a time → delivers a diagnosis. The route isn't fixed in advance — each step depends on what the previous tool returned.

Built with n8n (AI Agent node) + Google Gemini (`gemini-3.1-flash-lite`) + Google Sheets as the data source.

## How it works

The agent runs a "reason → act → observe" loop (ReAct): at each step the model decides which tool to call, reads the result, and picks the next move based on it. So it takes different paths on different input data:

```
CR collapses while clicks stay stable:   stats → info → check_landing → 404  → diagnosis "dead landing page"
revenue drops while CR stays stable:     stats → info → payout_history → 30→15 → diagnosis "payout was cut"
```

Same agent, same prompt — two different investigations.

## Architecture

```
When chat message received
        │
        ▼
     AI Agent ──── Chat Model: Google Gemini (gemini-3.1-flash-lite)
        │
        └── Tools:
            ├── get_campaign_stats   (Google Sheets → daily_stats)    — daily dynamics, used FIRST
            ├── get_campaign_info    (Google Sheets → campaigns)      — offer_id and landing_url
            ├── check_landing        (HTTP Request)                   — landing page HTTP status (200 / 4xx / 5xx)
            └── get_payout_history   (Google Sheets → payout_history) — payout changes per offer
```

Safeguards against the loop running away:
- `Max Iterations = 8` on the AI Agent node (hard stop at the engine level);
- a "max 6 tool calls" limit in the system prompt (soft stop at the model level).

## Data

Spreadsheet `Campaign_Analytics`, three sheets:

| Sheet | Columns |
|-------|---------|
| `daily_stats` | campaign_id, date, clicks, conversions, cr, revenue |
| `campaigns` | campaign_id, name, offer_id, landing_url |
| `payout_history` | offer_id, date, old_payout, new_payout |

Two "failures" are deliberately planted in the data to test the agent:

- **C003 — dead landing page.** Clicks stay stable, but starting June 8 the CR collapses from ~3% to ~0.4%. The `landing_url` returns a real 404.
- **C004 — payout cut.** Clicks and CR stay stable, but starting June 9 revenue drops by half. The landing page is alive (200), while in `payout_history` the offer's payout was cut from 30 to 15.

Plus a red herring: a harmless payout increase on a different offer in `payout_history` — a check that the agent doesn't grab the first change it sees.

## Setup

1. Import `Campaign_Diagnostic_Agent.json` into your n8n instance.
2. Upload `Campaign_Analytics.xlsx` to Google Drive and open it as a Google Sheet.
3. Connect your credentials: Google Gemini (API key from Google AI Studio) and Google Sheets OAuth2. Set the `documentId` of your spreadsheet in the three Sheets nodes.
4. Open the chat on the "When chat message received" node and start asking.

## Test prompts

| Prompt | What it checks | Expected result |
|--------|----------------|-----------------|
| Why did conversions drop on campaign C003? | the basic ReAct chain | finds the 404, diagnosis "dead landing page" |
| Why did revenue drop on campaign C004? | doesn't stop after the first hypothesis | sees 200, moves on to payouts, diagnosis "payout was cut" |
| What's wrong with campaign C001? | doesn't hallucinate on healthy data | honestly answers "no anomalies" |

## A note on secrets

An n8n workflow export does **not** contain key values — only references to credentials (`id` + `name`). The keys themselves live in the n8n instance and never end up in Git. Don't commit `.env` files or real API keys. If the production spreadsheet holds private data, put an anonymized copy in the repo instead.

## The project's key takeaway

If the agent took the wrong branch — you **fix the prompt, not the node graph**. The agent's route is a consequence of how the model read the methodology; fixing the prompt fixes the reasoning itself and carries over to every future investigation, whereas fixing the nodes would only patch one special case.<img width="2033" height="1196" alt="company" src="https://github.com/user-attachments/assets/379f5412-316e-4848-9536-4e8884e4f28f" />
