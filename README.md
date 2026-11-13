# AI Renewal & Revenue Risk Management (n8n)

An n8n workflow that scores every account's renewal risk with transparent rules (usage drop, support tickets, payment status, time to renewal), then asks an AI Agent (Gemini) to write a retention plan and a friendly customer message. High-risk accounts are sent to Slack.

▶ **Demo video:** https://drive.google.com/file/d/1HcZLpLKXVNRoTnFyO2o-XeLOOqsQfJFH/view

## How it works

1. Mock accounts arrive (ARR, days to renewal, usage change, tickets in 30 days, payment status)
2. A Code node scores risk from 0 to 100 and lists the warning signs (rules, not AI)
3. The AI Agent writes a one-sentence recommendation for the account manager and a short message draft for the customer
4. A Code node validates the output. High risk accounts go to Slack
5. Every account is saved to Postgres with score, level and factors

```
Manual Trigger -> Mock Accounts -> Score Risk -> AI Agent (Gemini)
-> Parse Advice -> High Risk?
   yes -> Slack Alert -> Save Risk
   no  -> Save Risk
```

## AI Agent

AI Agent node without tools (JSON answer). The score is deterministic, so it is auditable. The AI only writes the advice.

## Tech stack

n8n, AI Agent, Gemini, Customer Success, Retention, Slack, Postgres

## Setup

1. Copy the JSON from `workflow.md` into a file named `renewal-risk-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `revenue-risk`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: 4 mock inputs run through all branches

## Database

```sql
create table if not exists public.renewal_risks (
  id bigint generated always as identity primary key,
  account text,
  arr numeric,
  risk_score numeric,
  risk_level text,
  days_to_renewal numeric,
  risk_factors text,
  recommendation text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| Contoso Retail (usage -45%, 7 tickets, payment failed) | High risk, Slack alert |
| Adventure Works (usage +12%, on time) | Low risk |
| Litware Inc (usage -20%, 3 tickets, late payment) | Medium risk |
| Proseware (no warning signs, renews in 14 days) | Low risk |

## Notes

- Replace Mock Accounts with usage, support and payment data from your systems for client use
- The customer message draft avoids internal risk scores by design
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
