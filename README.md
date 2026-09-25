# closeprofile

Automations that create Close lead profiles.

## Helen → Jordan lead intake

When Helen replies to a prospect and CCs Jordan Kempster with a hand-off like
"@Jordan on our team can grab 15 minutes with you…", an hourly Claude routine:

1. Finds the reply in Jordan's Gmail (Composio, read only).
2. Checks Close (email, name, phone) and the Airtable "All Clients" table
   (email, partner email, name, additional member / partner name, phones; read only).
3. If the person is new, creates a Close lead (status Potential, owner/setter/initial
   contact = Jordan, Creation Date = Helen's reply date), a contact, a summary note,
   logs the prospect's email and Helen's reply, and adds Jordan's Day 1 / Day 2 /
   Day 5 follow-up tasks with the email templates.
   If the person already has a Close lead, no new lead is created; Jordan's three
   follow-up tasks are added to the existing lead instead.
4. Posts one line, "Hourly Close Lead Creation Report", to `#helen-email-digest`,
   with one thread reply per person (new lead / existing lead re-referred / needs review).
   Nothing is posted in hours with no activity. Ambiguous cases (same name but a different
   email, an Airtable client with no Close lead, several matching leads) are flagged as
   ":warning: Needs Sheila's review" and nothing is created.

The full step-by-step instructions the routine follows are in [RUNBOOK.md](RUNBOOK.md).
The routine's prompt is a copy of that file. If you change one, update the other
(claude.ai → Routines → "Helen → Jordan lead intake (hourly)").

### Requirements

- Routine connectors: **Composio** (Jordan's Gmail `gmail_dah-ceyx`, Slack, Close `close` + `close_mcp`),
  **Airtable**, **Close**.
- Only new hand-offs from the last 24 hours are picked up on each run; the report
  threads double as the processed log (`gmail:<message id>` footer).
