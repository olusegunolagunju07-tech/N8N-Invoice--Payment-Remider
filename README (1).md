# Invoice / Payment Reminder Automation (n8n)

An n8n workflow that checks a Google Sheet every morning for unpaid invoices, writes a polite reminder with an LLM (tone depends on how late the payment is), emails it through Gmail, and records the reminder in the sheet so nobody is emailed twice in a short window.

## How it works

```
Schedule Trigger → Get Unpaid Rows → Calculate Days Overdue → IF → Message a model → Send Gmail → Update Row
```

| # | Node | What it does |
|---|------|--------------|
| 1 | **Schedule Trigger** | Runs daily at 9:00 (n8n timezone, e.g. Africa/Lagos). |
| 2 | **Get Unpaid Rows** (Google Sheets, Get Row(s)) | Reads invoice rows where `Status` is `Unpaid`. |
| 3 | **Calculate Days Overdue** (Code) | Adds `daysOverdue` and `daysSinceReminder` to each row. |
| 4 | **IF Overdue & Not Recently Reminded** | True branch only if `daysOverdue > 0` **and** `daysSinceReminder >= 5`. |
| 5 | **Message a model** (Gemini) | Writes the email body, with tone set by lateness. |
| 6 | **Send Gmail** | Sends the email to the client with a subject line containing the due date. |
| 7 | **Update Row** (Google Sheets) | Sets `LastReminded` to today and increments `ReminderCount`. |

Rows on the IF node's false branch are ignored for that run.

## Google Sheet format

Row 1 must contain these headers, spelled exactly like this:

| Client | Email | Amount | DueDate | Status | LastReminded | ReminderCount |
|--------|-------|--------|---------|--------|--------------|---------------|
| Acme Ltd | client@example.com | 250000 | 2026-11-05 | Unpaid | | 0 |

- **DueDate / LastReminded:** use `yyyy-mm-dd`. Other formats (for example `30/11/2026`) may be parsed wrongly by the Code node.
- **Status:** `Unpaid` or `Paid`. Change it to `Paid` when a client pays so reminders stop.
- **LastReminded:** leave blank for new invoices. The workflow fills it in.
- **ReminderCount:** start at `0`.

A ready-made sample is in `invoice-reminders-sample.csv` (import via File → Import → Upload in Google Sheets).

## Setup

1. Import the sheet, or create your own with the columns above.
2. In n8n, import `invoice-reminder-workflow.json` (Workflows → Import from File).
3. Connect credentials:
   - **Google Sheets** (Get Unpaid Rows and Update Row)
   - **Gmail** (Send Gmail)
   - **LLM** (Gemini, OpenAI, Anthropic or any chat model of your choice)
4. In **both** Google Sheets nodes, set the Document (paste the sheet URL if "From list" is unavailable) and the Sheet tab.
5. Test with your own email address in every row before using real client emails.
6. Publish/activate the workflow so the schedule runs.

## Node configuration

### Get Unpaid Rows
- Resource: Sheet Within Document, Operation: **Get Row(s)**
- Filter: Column `Status`, Value `Unpaid`

> Make sure the filter is actually set. Without it, every row (including `Paid` ones) flows through the workflow.

### Calculate Days Overdue (Code)

```javascript
const today = new Date();
today.setHours(0, 0, 0, 0);
const msDay = 86400000;

return $input.all().map(item => {
  const d = item.json;
  const due = new Date(d.DueDate);
  const last = d.LastReminded ? new Date(d.LastReminded) : null;
  item.json.daysOverdue = Math.floor((today - due) / msDay);
  item.json.daysSinceReminder = last ? Math.floor((today - last) / msDay) : 999;
  return item;
});
```

A blank `LastReminded` counts as 999 days, meaning "never reminded".

### IF Overdue & Not Recently Reminded
- `{{ $json.daysOverdue }}` is greater than `0`
- AND `{{ $json.daysSinceReminder }}` is greater than or equal to `5`

Change the `5` to adjust how often a client can be reminded.

### Message a model (LLM prompt)

```
You are writing a payment reminder email on behalf of the Accounts Team.

Client: {{ $json.Client }}
Amount due: {{ $json.Amount }}
Original due date: {{ $json.DueDate }}
Days overdue: {{ $json.daysOverdue }}
Reminders already sent: {{ $json.ReminderCount || 0 }}

Tone rules:
- Under 7 days overdue: gentle and friendly, assume it was an oversight.
- 7 to 14 days overdue: polite but clear that payment is needed soon.
- Over 14 days overdue: firm but professional, ask for payment or a response within 3 business days.

Output rules:
- Return only the email body in plain text. No subject line, no placeholders, no markdown.
- Start with "Dear {{ $json.Client }},"
- Mention the amount and how many days overdue it is.
- Under 120 words.
- End with "Kind regards," followed by "Accounts Team".
```

The prompt field must be in **Expression** mode so the `{{ }}` values are filled in.

### Send Gmail
- To: `{{ $('IF Overdue & Not Recently Reminded').item.json.Email }}`
- Subject: `Payment reminder: invoice due {{ $('IF Overdue & Not Recently Reminded').item.json.DueDate }}`
- Message (Gemini output): `{{ $json.content.parts.map(p => p.text).filter(Boolean).join('') }}`
  - If you use the Basic LLM Chain node instead of the Gemini node, use `{{ $json.text }}`.
- The Gemini output also contains a `thoughtSignature` field. It is internal reasoning metadata and can be ignored.

### Update Row
- Operation: **Update Row(s)**
- Column to match on: `row_number`
- Values to update (only these three; remove all other columns so nothing gets overwritten):
  - `row_number`: `{{ $('IF Overdue & Not Recently Reminded').item.json.row_number }}`
  - `LastReminded`: `{{ $now.toFormat('yyyy-MM-dd') }}`
  - `ReminderCount`: `{{ Number($('IF Overdue & Not Recently Reminded').item.json.ReminderCount || 0) + 1 }}`

It reads from the IF node (not the previous node) because the Gmail node's output replaces the row data.

## Testing

Use sample rows covering each case:

| Case | Expected result |
|------|-----------------|
| Unpaid, overdue, never reminded | Email sent, `LastReminded` set, `ReminderCount` +1 |
| Unpaid, overdue, reminded within the last 5 days | Skipped |
| Unpaid, overdue, last reminded 5 or more days ago | Email sent |
| Unpaid, not yet due | Skipped |
| Paid | Skipped (requires the Status filter) |

Run nodes one at a time first (Execute step), check each output, then run the full workflow. Re-running the full workflow on the same day should send nothing new, because everyone reminded today has `daysSinceReminder` of 0.

## Troubleshooting

| Problem | Likely cause and fix |
|---------|----------------------|
| Operation field is red after import | Pick **Get Row(s)** / **Update Row(s)** from the dropdown again. |
| "From list" is greyed out for Document | Use **By URL** or **By ID**. Enable the Google Drive API for your credential if you want list browsing. |
| Gmail sends an empty email | The Message expression does not match the LLM node's output shape. See the Send Gmail section. |
| Paid rows appear in the workflow | The Status filter is missing on Get Unpaid Rows. |
| Wrong `daysOverdue` values | `DueDate` is not in `yyyy-mm-dd` format. |
| Wrong row gets updated | Update Row is not matching on `row_number`. |
| Update Row blanks other columns | Remove the extra columns (Client, Email, Amount, and so on) from Values to Update. |
| Schedule never fires | The workflow is not published/active. |

## Limitations and ideas

- Amounts appear as plain numbers (for example `60000`). Add a currency symbol in the sheet or the prompt if you want one.
- There is no cap on `ReminderCount`. Add a condition (for example `ReminderCount < 4`) to stop emailing after a set number of reminders, or to escalate to a human.
- Add a payment link or bank details to the prompt so clients can pay straight away.
- Add a Slack or email summary of who was reminded each day.
- Add error handling (an Error Trigger workflow) so you hear about failed sends.
