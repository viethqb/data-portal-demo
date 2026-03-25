# Email Scheduler Management

## Purpose

Allow Admin/DA to schedule automatic emails with Excel file attachments exported from an Export Config.

## Users

- **Admin**: CRUD all schedulers
- **DA**: CRUD own schedulers
- **End User**: No access (receives emails only)

## Interface

### List Page

- Search bar + Status filter + "+ New Scheduler" button
- Table:

| Column        | Description                    |
| ------------- | ------------------------------ |
| Name          | Scheduler name                 |
| Export Config | Config used                    |
| Schedule      | Frequency (human-readable)     |
| Recipients    | Number of recipients           |
| Status        | Active / Paused / Failed       |
| Last Run      | Time + result of last run      |
| Next Run      | Next scheduled run             |
| Actions       | Edit, Pause/Resume, Run Now, Delete |

### Create/Edit Form (single page)

**Section 1 — Basic**
- Name, Export Config (dropdown)

**Section 2 — Filter Override**
- Table of filters from export config, allows overriding default values
- File name pattern (supports `{{YYYY-MM}}` tokens)

**Section 3 — Schedule**
- Preset buttons: Daily / Weekly / Monthly
- If Weekly: day-of-week checkboxes
- Time picker + Timezone
- Shows next 5 scheduled runs

**Section 4 — Email**
- Recipients: tag-style input (type email, autocomplete from user list)
- Subject (text input, supports `{{tokens}}`)
- Body (textarea, supports `{{tokens}}`)

**"Send Test"** + **"Save"** buttons

### Run History (in scheduler detail page)

Table:

| Column  | Description                     |
| ------- | ------------------------------- |
| #       | Run number                      |
| Time    | Run time                        |
| Status  | Sent / Failed                   |
| File    | File size                       |
| Actions | View detail, Retry (if failed)  |

## Flow

1. DA clicks "+ New Scheduler"
2. Selects Export Config → overrides filters if needed
3. Configures schedule (weekly/daily/monthly) + time
4. Enters recipients + subject + body
5. Send Test → verifies email → Save