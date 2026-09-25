# Helen → Jordan lead intake (hourly)

You are running an automated, unattended job. No human is watching this session.
Follow these steps exactly, then stop. Never ask questions; when something is
ambiguous, **flag it** (step 6) and move on. Do not edit any repository or push code.

Hard rules:
- **Airtable is read-only.** Never create, update or delete Airtable records.
- **Never send email** from anyone's mailbox. Only read Gmail.
- Never modify or add to an existing Close lead. If the person already exists, do nothing.
- Only create a lead when every duplicate check in step 3 is clean.

## Constants

| Thing | Value |
|---|---|
| Jordan's Gmail (Composio account) | `gmail_dah-ceyx` (jkempster@smbdealhunter.xyz) |
| Helen's sending address | `helen@smbdealhunter.xyz` |
| Airtable base / table | `appDBhzdZfjoIFh2J` / `tblqt9BdHcwm0TzX6` ("All Clients") |
| Slack channel | `#helen-email-digest` = `C0BTCGZSF9R` |
| Jordan Kempster Close user id | `user_3IrYZLUcCBZrg2mb49wCybrxGBFTCxa5eqTHhE1ApuC` |
| Jordan's calendar link | https://calendly.com/jkempster-smbdealhunter/intro-call-with-smb-deal-hunter |
| Lead status "Potential" | `stat_v6aCXRI3yiAPBImnr8zJ1dyADO6m0mdPrhIvzK3Kdkj` |
| Timezone for dates | America/Denver (Jordan's) |

Close lead custom fields to set:

| Field | Id | Value |
|---|---|---|
| Lead Owner | `cf_iwYMbuKfpAL7hJicgF0ZwPDAan5eg3vdIh3WG4Bf7PL` | Jordan user id |
| Setter | `cf_GgXDBkiGJbOFFIOW43vcFWc0bXr706WQU5zxJMDIAAE` | Jordan user id |
| Initial Contact User | `cf_C6fOIXKxCBrG5PhlmEhcO6Rzu20G8XV6aA7wnADkH3P` | Jordan user id |
| Creation Date | `cf_RdUwNTlCpEBzzWDbmzUcPUwPMmQPmO6EZJLsH6olFhM` | Helen's reply timestamp (ISO 8601, UTC) |

Leave blank: Closer, SDR, Lead Source, Lead Quality, and every qualification
field unless the prospect's email states the answer outright (step 4e).

## Tools

- Gmail: Composio `GMAIL_FETCH_EMAILS`, `GMAIL_FETCH_MESSAGE_BY_THREAD_ID`, always with `account: "gmail_dah-ceyx"`.
- Close search: Close connector `lead_search` (`full_text`, `name`) and `search` (natural language), or Composio `CLOSE_MCP_LEAD_SEARCH` / `CLOSE_MCP_SEARCH`.
- Close writes: Composio `CLOSE_MCP_CREATE_LEAD` (supports `custom_fields`), `CLOSE_MCP_CREATE_CONTACT`, `CLOSE_MCP_CREATE_NOTE`, `CLOSE_MCP_CREATE_TASK`, `CLOSE_MCP_UPDATE_LEAD`
  (the `close_mcp` connection acts as Sheila). Email logging only exists as `CLOSE_CREATE_EMAIL`
  on the `close` connection (acts as Jay DeCristofaro) — that's expected.
- Airtable: Airtable connector `search_records` (read only).
- Slack: Composio `SLACK_SEND_MESSAGE`; history via `SLACK_FETCH_CONVERSATION_HISTORY` (or equivalent found with `COMPOSIO_SEARCH_TOOLS`).

## Step 1 — Find trigger emails

Search Jordan's mailbox:

```
from:helen@smbdealhunter.xyz cc:jkempster@smbdealhunter.xyz newer_than:1d
```

(also run the same query with `to:jkempster@smbdealhunter.xyz` in place of `cc:`
to catch replies where Helen put Jordan on the To line).

For each message, read the full message and keep it only if **Helen's own new
text** (above the quoted "On … wrote:" block) mentions Jordan being looped in to
talk to the person, e.g.:
- "@Jordan on our team can grab 15 minutes with you…"
- "Looping in Jordan from our team. @Jordan, do you mind finding 15 minutes to give X a call?"

Skip: plain forwards Helen sends only to Jordan ("Fwd:"), messages where the
@Jordan mention is only inside quoted text, and messages from anyone else.

If nothing qualifies, stop here (no Slack post).

## Step 2 — Skip already-handled messages

Read the last 2 days of `#helen-email-digest` history (`SLACK_FETCH_CONVERSATION_HISTORY`, `oldest` = now − 2 days). If a message there already
contains `gmail:<this message id>`, skip it: it was already created or flagged.

## Step 3 — Identify the prospect and check for duplicates

**Prospect** = the external recipient of Helen's reply (To/CC addresses that are
not `@smbdealhunter.xyz` / `@dylanhelen.com`). Get their name from the quoted
"On …, Name <email> wrote:" line or their signature. Also pull any phone number,
title, company and LinkedIn from their signature.
If there is not exactly one external recipient, or no usable name → **flag** (reason: "can't identify prospect").

Run ALL of these checks:

Close:
1. `lead_search(full_text=<email>)`
2. `lead_search(name=<full name>)`
3. If phone known: `lead_search(full_text=<10 digits, no punctuation>)` and `search("contacts with phone <+1XXXXXXXXXX>")`

Airtable "All Clients" (`search_records`, read only):
4. query = email, fields `["Email", "Partner Email"]`
5. query = full name, fields `["Name", "Additional Member Name", "Name of Partner"]`
6. If phone known: query = `+1XXXXXXXXXX`, fields `["Member Phone Number", "Partner Phone Number"]`

Decide:
- **Email or phone matches** any Close lead/contact or Airtable record → already exists → **do nothing** (no lead, no Slack post). Go to next message.
- **Full name matches exactly** (same first + last name, case-insensitive) in Close or Airtable **but email/phone do not** → **flag** (reason: "name matches existing record(s) but email differs", include links/record names). Do not create.
- Only surname or partial matches (e.g. other "Habib"s) → not a duplicate.
- All clean → step 4.

## Step 4 — Create the Close lead

a. `CLOSE_MCP_CREATE_LEAD`:
   - `name`: prospect full name
   - `status_id`: Potential
   - `custom_fields`: Lead Owner, Setter, Initial Contact User = Jordan; Creation Date = Helen's reply timestamp
   - `description`: company name if known (e.g. "Asset Manager, Giving Lotus Capital")

b. `CLOSE_MCP_CREATE_CONTACT` on that lead: name, title (if in signature),
   email (`type: office` if a company domain, else `home`), phone in `+1XXXXXXXXXX`
   format (`type: mobile`) if in signature, LinkedIn URL if present.

c. `CLOSE_MCP_CREATE_NOTE` (title "Helen email referral"), plain text:
   ```
   Source: Helen Guo email referral → looped in Jordan Kempster on <reply date>.

   Prospect's email (<original date>): <2-4 sentence summary of what they asked / want>

   Helen's reply (<reply date>, cc Jordan): "<Helen's new text verbatim>"

   Gmail thread: <display_url of Helen's reply>
   gmail:<Helen's reply message id>
   ```

d. Log both emails with `CLOSE_CREATE_EMAIL` (logging only; never use `outbox`/`scheduled`):
   1. The prospect's original email to Helen: `status: "inbox"`, `sender`: prospect,
      `to: ["helen@smbdealhunter.xyz"]`, subject, `body_text`: their message only
      (strip the quoted newsletter), `activity_at` / `date_created`: its timestamp, `contact_id`.
   2. Helen's reply: `status: "sent"`, `sender: "Helen Guo <helen@smbdealhunter.xyz>"`,
      `to`: prospect, `cc: ["jkempster@smbdealhunter.xyz"]`, same subject, `body_text`:
      Helen's new text, `activity_at` / `date_created`: its timestamp, `contact_id`.
   If the original email's timestamp isn't in the headers, use the date from the
   "On …, wrote:" line.

e. Qualification fields: only if the prospect's email says so outright. Example:
   "commercial cleaning company in 1-2 specific markets" → set
   `What does your search criteria look like?` (`cf_fMji3IHe85zXS7ze4egNyKjSmGvLgsC5HqVTGNdZ4mo`)
   via `CLOSE_MCP_UPDATE_LEAD`. Never guess liquidity, timeline, etc.

## Step 5 — Create Jordan's three tasks

`CLOSE_MCP_CREATE_TASK` with `assigned_to` = Jordan, `lead_id` = new lead, `contact_id` = new contact,
`due_date` as below, `send_notification: true`. Let D0 = today (America/Denver).

**Task 1**, due D0:
```
Day 1 (Helen referral): Respond TODAY. Call <First> at <phone or "no number yet — reply on Helen's thread and ask for best number"> or get a call booked. Calendar: https://calendly.com/jkempster-smbdealhunter/intro-call-with-smb-deal-hunter
```

**Task 2**, due D0 + 1 day:
```
Day 2 check-in: Call booked? Lead responded? If NOT → call again + send follow-up email on the same thread:

If you have their number and called:
Hi <First>,

Gave you a call yesterday but didn't reach you — wanted to follow up and try you again today.

If it's easier, grab 15 minutes here: https://calendly.com/jkempster-smbdealhunter/intro-call-with-smb-deal-hunter

If you don't have their number:
Hi <First>,

Sent over my availability yesterday but haven't seen a booked call come through yet, so wanted to follow up with it again: https://calendly.com/jkempster-smbdealhunter/intro-call-with-smb-deal-hunter

What's the best number to reach you at?
```

**Task 3**, due D0 + 5 days:
```
Day 5 breakup email — ONLY if Day 2 also went nowhere (still no response / no booked call). Email only, no call. Same thread:

Hi <First>,

Helen mentioned you were interested, but we haven't had a chance to connect yet. Should I stop following up?
```

## Step 6 — Slack

**Lead created** → `SLACK_SEND_MESSAGE` to `C0BTCGZSF9R`, `markdown_text`:
```
:new: **New Close lead from Helen's inbox: <Full Name>**
• Email: <email> | Phone: <phone or —>
• <title, company if known>
• Asked: <one-line summary>
• Helen looped in Jordan on <reply date>
• Close: <lead URL https://app.close.com/lead/<lead_id>/>
• Tasks for Jordan: Day 1 (<D0>), Day 2 (<D0+1>), Day 5 breakup (<D0+5>)
_gmail:<Helen's reply message id>_
```

**Flag** (not created) → same channel:
```
:warning: **Needs Sheila's review — lead NOT created: <name or email>**
• Reason: <reason>
• Possible matches: <Close lead links / Airtable client names>
• Gmail: <display_url>
_gmail:<Helen's reply message id>_
```

## Step 7 — Finish

Print a short summary: messages scanned, leads created (with links), flagged,
skipped as existing, skipped as already handled. If any tool call failed partway
through creating a lead, post a :warning: flag describing exactly what was and
wasn't created so a human can clean it up. Do not retry lead creation in that case.
