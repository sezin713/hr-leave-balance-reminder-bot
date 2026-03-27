# HR Leave Balance Reminder Bot

An n8n automation workflow that monitors employee leave balances, calculates burn rates, and sends personalised reminders before entitlements expire — keeping employees informed and HR teams ahead of unused-leave liability.

---

## What It Does

- Runs automatically on a **monthly schedule**
- Reads employee leave data from **Google Sheets** (or a connected HRIS)
- Calculates each employee's **burn rate** — how fast they need to use remaining leave before renewal
- Filters out inactive employees and new hires (< 90 days tenure)
- Sends **personalised email reminders** at three urgency levels: Critical, Warning, and Low
- Logs every notification run to a **Google Sheets audit log**
- Posts a **summary digest** to Slack or Microsoft Teams after each run

---

## How It Works — Burn Rate Logic

Burn rate measures how many leave days an employee must consume **per day** to exhaust their balance by renewal.

```
Burn Rate = leaveBalance / daysUntilRenewal
```

| Alert Level | Burn Rate    | Meaning |
|-------------|-------------|---------|
| 🔴 Critical  | ≥ 0.5       | Employee must take > 1 day every 2 days — immediate action needed |
| 🟠 Warning   | 0.2 – 0.49  | Balance is running down fast — start planning now |
| 🟡 Low       | 0.1 – 0.19  | Mild risk — a friendly nudge is enough |
| ⏭️ No alert  | < 0.1       | Balance is fine — no notification sent |

**Renewal date** is calculated per employee from their hire date anniversary. The engine also enforces data-quality guards: employees with invalid dates, balances exceeding their entitlement, zero balance, or fewer than 90 days of tenure are silently skipped.

---

## Prerequisites

- **n8n** (self-hosted v1.x+ or n8n Cloud)
- A **Google account** with access to Google Sheets (OAuth2 credential configured in n8n)
- A **Gmail account** (OAuth2 credential configured in n8n) — or swap to Outlook/SMTP
- Optional: Slack workspace token or Microsoft Teams webhook for summary notifications

---

## Quick Start — Google Sheets

### 1. Prepare your spreadsheet

Create a Google Sheet with two tabs:

**Tab 1 — `hr_mock_data`** (employee data)

| Column | Type | Example |
|--------|------|---------|
| `id` | string | `EMP001` |
| `name` | string | `Jane Smith` |
| `email` | string | `jane@company.com` |
| `department` | string | `Engineering` |
| `hireDate` | ISO date | `2021-06-15` |
| `leaveBalance` | integer | `12` |
| `leaveTotal` | integer | `25` |
| `status` | string | `active` |

**Tab 2 — `Log`** (auto-populated by the bot)

| Column | Description |
|--------|-------------|
| `timestamp` | ISO timestamp of the run |
| `employeeID` | Employee ID |
| `employeeName` | Full name |
| `department` | Department |
| `leaveBalance` | Remaining days at time of alert |
| `renewalDate` | Next renewal date |
| `daysUntilRenewal` | Days remaining until renewal |
| `burnRate` | Calculated burn rate |
| `alertLevel` | `critical` / `warning` / `low` |
| `emailSubject` | Subject line sent |

### 2. Import the workflow

1. In n8n, go to **Workflows → Import from file**
2. Upload `hr-leave-balance-bot.json`

### 3. Connect credentials

| Node | Credential needed |
|------|------------------|
| 02 — Fetch Employee Data | Google Sheets OAuth2 |
| 05 — Send Email | Gmail OAuth2 |
| 06 — Log to Google Sheets | Google Sheets OAuth2 |
| 07a — Notify via Slack *(optional)* | Slack OAuth2 |
| 07b — Notify via Teams *(optional)* | Microsoft Teams webhook |

### 4. Point to your spreadsheet

Open nodes **02** and **06**, click the **Document** selector, and choose your spreadsheet and the correct sheet tab.

### 5. Activate

Toggle the workflow to **Active**. It will run on the first day of every month at 9 PM server time.

---

## Data Source Integrations

The workflow ships with Google Sheets as the default data source. Node **02 — Fetch Employee Data** is the only node you need to swap to connect a different system.

### Google Sheets (default)
Use the built-in `n8n-nodes-base.googleSheets` node. Set up OAuth2 credentials in n8n and point the node at your spreadsheet.

### BambooHR
Replace node 02 with an **HTTP Request** node:
```
GET https://api.bamboohr.com/api/gateway.php/{subdomain}/v1/employees/directory
Authorization: Basic base64(apiKey:x)
Accept: application/json
```
Map the response fields: `id`, `displayName`, `workEmail`, `department`, `hireDate`.
Fetch leave balances with a second HTTP Request to `/v1/employees/{id}/time_off/calculator`.

### Workday
Replace node 02 with an **HTTP Request** node pointed at your Workday REST API endpoint:
```
GET https://{tenant}.workday.com/api/v1/workers?format=json
Authorization: Bearer {token}
```
Use the Workday `timeOffBalance` object to populate `leaveBalance` and `leaveTotal`.

### Personio
Replace node 02 with an **HTTP Request** node:
```
GET https://api.personio.de/v1/company/employees
Authorization: Bearer {token}
```
Map `id`, `first_name + last_name`, `email`, `department`, `hire_date`, and the time-off balance fields from the response.

### Hibob
Replace node 2 with an **HTTP Request** node:
```
GET https://api.hibob.com/v1/people
Authorization: Basic {serviceUserToken}
```
Map `id`, `displayName`, `email`, `work.department`, `work.startDate`, and balance fields from `timeOff.policy.balance`.

> **Tip:** After swapping the data source, make sure the field names flowing into node 03 match the expected keys: `id`, `name`, `email`, `department`, `hireDate`, `leaveBalance`, `leaveTotal`, and `status`. Use an **n8n Set** node after your data source if you need to rename fields.

---

## Alert Logic

### Thresholds

```
Burn Rate = leaveBalance / daysUntilRenewal
```

| Level    | Condition       | Subject line prefix |
|----------|-----------------|---------------------|
| 🔴 Critical | burnRate ≥ 0.5  | `Action Required: N Leave Days Expiring on DATE` |
| 🟠 Warning  | burnRate ≥ 0.2  | `Reminder: N Leave Days Remaining — Expiring DATE` |
| 🟡 Low      | burnRate ≥ 0.1  | `Friendly Reminder: N Leave Days Left Before DATE` |
| ⏭️ Skip     | burnRate < 0.1  | No email sent |

### Automatic filters (employees skipped silently)

- `status` is not `active`
- `leaveBalance` or `leaveTotal` is not a valid integer
- `leaveTotal` is 0 or negative
- `leaveBalance` is 0 or negative
- `leaveBalance` exceeds `leaveTotal`
- `hireDate` is not a valid date
- Tenure is less than 90 days (new hires)

### Results ordering

Employees are processed in **descending burn rate order** — the most urgent cases are emailed first.

---

## Output Integrations

### Gmail (default)
Node **05 — Send Email** uses the `n8n-nodes-base.gmail` node with OAuth2. Each email is sent individually to the employee's address with a personalised body.

### Outlook / Microsoft 365
Swap node 05 for `n8n-nodes-base.microsoftOutlook`. Use the same `$json.email`, `$json.emailSubject`, and `$json.emailBody` expressions — no other changes needed.

### SMTP (any provider)
Swap node 05 for `n8n-nodes-base.emailSend`. Configure host, port, and credentials in the node settings.

### Slack (summary only)
Node **07a — Notify via Slack** posts a summary digest to `#hr-alerts` after all emails are sent. Add your Slack OAuth2 credential and set the channel name in the node.

### Microsoft Teams (summary only)
Node **07b — Notify via Teams** posts the same summary digest to a Teams channel. Configure your team and channel ID using the list selectors in the node.

> **Important:** Only activate **one** of nodes 07a and 07b per run, or both if you use both platforms. The workflow runs both in parallel after the logging step.

---

## Node Structure

```
01 — Monthly Schedule
        │
        ▼
02 — Fetch Employee Data          ← swap for BambooHR / Workday / Personio / Hibob
        │
        ▼
03 — Calculate Burn Rate          ← filters, calculates burnRate, assigns alertLevel
        │
        ▼
04 — Build Email Content          ← generates personalised subject + body per level
        │
        ▼
05 — Send Email                   ← Gmail (swap for Outlook / SMTP)
        │
        ▼
06 — Log to Google Sheets         ← appends one row per notification to audit log
        │
       ┌┴─────────────────┐
       ▼                   ▼
07a — Notify via Slack   07b — Notify via Teams
```

---

## Configuration

All configuration is done directly in n8n — there are no environment variables or config files.

| Setting | Where to change | Default |
|---------|----------------|---------|
| Schedule frequency | Node 01 → Rule → Interval | Monthly at 9 PM |
| Data source spreadsheet | Node 02 → Document selector | Demo spreadsheet |
| Data source sheet tab | Node 02 → Sheet Name selector | `hr_mock_data` |
| Email sender credential | Node 05 → Credentials | Gmail OAuth2 |
| Log spreadsheet | Node 06 → Document selector | Demo spreadsheet |
| Log sheet tab | Node 06 → Sheet Name selector | `Log` |
| Slack channel | Node 07a → Channel | `#hr-alerts` |
| Teams team + channel | Node 07b → Team / Channel | *(empty — fill in)* |
| Critical threshold | Node 03 → JS code → `burnRate >= 0.5` | 0.5 |
| Warning threshold | Node 03 → JS code → `burnRate >= 0.2` | 0.2 |
| Low threshold | Node 03 → JS code → `burnRate >= 0.1` | 0.1 |
| Minimum tenure (days) | Node 03 → JS code → `tenureDays < 90` | 90 |

---

## Troubleshooting

**No emails sent — workflow runs but produces no output**
- Check that the Google Sheet tab name matches exactly (case-sensitive)
- Verify the sheet has a header row and at least one row with `status = active`
- Confirm the `hireDate` column uses ISO format (`YYYY-MM-DD`)
- Check that no employees have tenure < 90 days (all would be skipped)

**`leaveBalance` or `leaveTotal` shows as NaN**
- Ensure these columns contain plain integers, not text with units (e.g., `12 days` will fail — use `12`)

**Gmail credential error**
- Re-authorise the Gmail OAuth2 credential in n8n Settings → Credentials
- Make sure the Google Cloud project has the Gmail API enabled

**Google Sheets 403 error**
- Re-authorise the Google Sheets OAuth2 credential
- Confirm the service account or OAuth user has edit access to the spreadsheet

**Slack message not sent**
- The Slack node (07a) requires a Bot Token credential with `chat:write` scope
- Confirm the bot has been invited to the `#hr-alerts` channel (`/invite @your-bot`)

**Teams message not sent**
- Select your team and channel using the list dropdowns in node 07b (IDs must be resolved via the UI, not typed manually)

**Burn rate thresholds feel off**
- Adjust the three `if (burnRate >= ...)` conditions in the node 03 code block directly to match your organisation's leave policy

---

## Changelog

### v1.0.0 — 2025
- Initial release
- Google Sheets data source with OAuth2
- Three-tier burn rate alert engine (critical / warning / low)
- Personalised Gmail notifications per employee
- Audit log appended to Google Sheets after each run
- Parallel Slack and Microsoft Teams summary digest nodes
- Sticky-note annotations for all major sections
