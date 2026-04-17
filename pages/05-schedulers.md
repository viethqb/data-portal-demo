# Page 5: Schedulers

**Handle:** `schedulers` | **Permissions:** admin, da

Automated report generation on a schedule with email delivery. Uses **ToolJet Database** for schedule config + **ToolJet Workflows** for execution.

## Architecture

```text
┌─ ToolJet App (this page) ──────────────────────────────────────┐
│  CRUD schedule configs in ToolJet DB table `schedulers`        │
│  [Run Now] button → triggers workflow manually                 │
└────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─ ToolJet Workflow: "report-scheduler" ─────────────────────────┐
│  Trigger: Scheduler (every 5 min) or Manual                    │
│                                                                │
│  Start → getActiveSchedulers (ToolJetDB)                       │
│    → Loop each scheduler:                                      │
│        → isDue? (JS: check cron vs now)                        │
│            ├─ Yes → generateReport (pyDBAPI REST)              │
│            │    → pollUntilDone (JS loop)                       │
│            │    → readReportFile (MinIO Read Object)            │
│            │    → sendEmail (SMTP)                              │
│            │    → updateLastRun (ToolJetDB)                     │
│            └─ No → skip                                        │
│  → Response                                                    │
└────────────────────────────────────────────────────────────────┘
```

## ToolJet Database Table: `schedulers`

Create in **ToolJet Database**:

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | serial | Primary key | |
| `name` | varchar | Not null | Schedule name |
| `description` | varchar | | Optional description |
| `template_id` | varchar | Not null | pyDBAPI template ID |
| `template_name` | varchar | | Snapshot for display |
| `module_id` | varchar | Not null | pyDBAPI module ID |
| `parameters` | varchar | | JSON string, with date placeholders |
| `cron_expression` | varchar | Not null | Cron syntax (e.g., `0 9 * * 1`) |
| `timezone` | varchar | Default: `Asia/Ho_Chi_Minh` | |
| `recipients` | varchar | | Comma-separated emails |
| `email_subject` | varchar | | Subject line with placeholders |
| `email_body` | varchar | | Body text with placeholders |
| `is_active` | boolean | Default: true | Pause/resume |
| `last_run_at` | timestamp | | Last execution time |
| `last_status` | varchar | | success / failed |
| `last_error` | varchar | | Error message if failed |
| `created_at` | timestamp | Set in query: `new Date().toISOString()` | |
| `updated_at` | timestamp | Set in query: `new Date().toISOString()` | |

## Layout

```text
┌─────────────────────────────────────────────────────────────────────┐
│  [headerText]                                                       │
│  [statusFilter]  [btnCreateScheduler]                               │
├─────────────────────────────────────────────────────────────────────┤
│  [schedulerTable]                                                   │
├─────────────────────────────────────────────────────────────────────┤
│  [schedulerFormModal]                                               │
│  [confirmDeleteModal]                                               │
└─────────────────────────────────────────────────────────────────────┘
```

## Page Events

| Event | Action |
|---|---|
| On page load | Run: `listSchedulers`, `listAllTemplates` |

## Variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `isEditing` | Boolean | `false` | Create vs edit mode |
| `editSchedulerData` | Object | `{}` | Pre-fill form for edit |
| `schedulerToDelete` | Number | `null` | ID for delete confirm |
| `schedulerParamRows` | Array | `[]` | Rows of `{id, key, type, value}` for dynamic parameter form |
| `cronNextRuns` | Array | `[]` | Preview of next run times |

## Queries

### Q1: `listSchedulers`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `schedulers` |
| Operation | List rows |
| Sort by | `name` ASC |
| Run on page load | Yes |

### Q2: `listAllTemplates`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 100}` |
| Run on page load | Yes |
| Transform | `return data.data.map(t => ({label: t.name, value: t.id, module_id: t.report_module_id}))` |

### Q3: `createScheduler`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `schedulers` |
| Operation | Create row |

| Column | Value |
|---|---|
| `name` | `{{components.sfName.value}}` |
| `description` | `{{components.sfDescription.value}}` |
| `template_id` | `{{components.sfTemplate.value}}` |
| `template_name` | `{{components.sfTemplate.selectedOption.label}}` |
| `module_id` | `{{queries.listAllTemplates.data.find(t => t.value === components.sfTemplate.value)?.module_id}}` |
| `parameters` | `{{queries.sfBuildParametersJSON.data}}` (JSON string with placeholders, see Dynamic Parameter Form below) |
| `cron_expression` | `{{components.sfCron.value}}` |
| `timezone` | `{{components.sfTimezone.value}}` |
| `recipients` | `{{components.sfRecipients.value}}` |
| `email_subject` | `{{components.sfEmailSubject.value}}` |
| `email_body` | `{{components.sfEmailBody.value}}` |
| `is_active` | `true` |

| Event | Action |
|---|---|
| On success | Close `schedulerFormModal` → Run `listSchedulers` → Show alert "Scheduler created" |

### Q4: `updateScheduler`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `schedulers` |
| Operation | Update row |
| Filter | `id` equals `{{variables.editSchedulerData.id}}` |

Same columns as Q3, plus `updated_at` = `{{new Date().toISOString()}}`.

| Event | Action |
|---|---|
| On success | Close `schedulerFormModal` → Run `listSchedulers` → Show alert "Scheduler updated" |

### Q5: `deleteScheduler`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `schedulers` |
| Operation | Delete row |
| Filter | `id` equals `{{variables.schedulerToDelete}}` |

| Event | Action |
|---|---|
| On success | Run `listSchedulers` → Show alert "Scheduler deleted" |

### Q6: `toggleScheduler`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `schedulers` |
| Operation | Update row |
| Filter | `id` equals `{{parameters.schedulerId}}` |
| Set `is_active` | `{{parameters.newStatus}}` |
| Set `updated_at` | `{{new Date().toISOString()}}` |

| Event | Action |
|---|---|
| On success | Run `listSchedulers` |

### Q7: `runNow`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/{{parameters.moduleId}}/templates/{{parameters.templateId}}/generate` |
| Body | `{"parameters": {{parameters.resolvedParams}}}` |

| Event | Action |
|---|---|
| On success | Show alert "Report generation started" → Update `last_run_at` and `last_status` in ToolJet DB |
| On failure | Show alert "Failed: {{queries.runNow.rawData?.detail}}" |

## Components

### C1: `headerText`

| Property | Value |
|---|---|
| Component | Text |
| Content | `Schedulers` |
| Font size | 24px, bold |

### C2: `statusFilter`

| Property | Value |
|---|---|
| Component | Dropdown |
| Options | `[{label: "All", value: ""}, {label: "Active", value: "true"}, {label: "Paused", value: "false"}]` |
| Default | `""` |

| Event | Action |
|---|---|
| On select | Run `listSchedulers` with filter `is_active` if value not empty |

### C3: `btnCreateScheduler`

| Property | Value |
|---|---|
| Component | Button |
| Label | `+ Create Scheduler` |
| Variant | Primary |

| Event | Action |
|---|---|
| On click | Set `isEditing` = `false`, `editSchedulerData` = `{}` → Show `schedulerFormModal` |

### C4: `schedulerTable`

| Property | Value |
|---|---|
| Component | Table |
| Data | `{{queries.listSchedulers.data}}` |
| Loading | `{{queries.listSchedulers.isLoading}}` |

**Columns:**

| # | Header | Key | Type | Width | Config |
|---|---|---|---|---|---|
| 1 | Name | `name` | Default | 180px | Bold |
| 2 | Template | `template_name` | Default | 160px | — |
| 3 | Schedule | `cron_expression` | Default | 120px | Font: mono |
| 4 | Timezone | `timezone` | Default | 140px | — |
| 5 | Status | `is_active` | **Boolean** (toggle) | 80px | — |
| 6 | Last Run | `last_run_at` | Default | 150px | Cell: `{{cellValue ? new Date(cellValue).toLocaleString() : '--'}}` |
| 7 | Last Status | `last_status` | **Select** | 100px | Options: `[{label:'success', value:'success', backgroundColor:'#10b981'}, {label:'failed', value:'failed', backgroundColor:'#ef4444'}]` |
| 8 | Actions | — | Action buttons | 180px | Edit, Run Now, Delete |

**Toggle column (Status) event:**

| Event | Action |
|---|---|
| Cell value changed | Run `toggleScheduler` with `schedulerId: {{rowData.id}}`, `newStatus: {{!rowData.is_active}}` |

**Action button: Edit ⚙**

| Event | Action |
|---|---|
| On click | Set `isEditing` = `true`, `editSchedulerData` = `{{rowData}}` → Show `schedulerFormModal` |

**Action button: Run Now ▶**

| Event | Action |
|---|---|
| On click | Run `runNow` with `moduleId: {{rowData.module_id}}`, `templateId: {{rowData.template_id}}`, `resolvedParams: {{resolveParameters(rowData.parameters)}}` |

> **Note:** `resolveParameters` is a JavaScript helper that replaces date placeholders. See Date Placeholders section below.

**Action button: Delete 🗑**

| Event | Action |
|---|---|
| On click | Set `schedulerToDelete` = `{{rowData.id}}` → Show `confirmDeleteModal` |

### C5: `schedulerFormModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `{{variables.isEditing ? 'Edit' : 'Create'}} Scheduler` |
| Size | Large |

**Modal on open event:**

| Event | Action |
|---|---|
| On show | Parse existing parameters into rows: run `sfLoadParamsForEdit` (see below) |

**Children:**

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Text Input | `sfName` | Text Input | Label: `Name *`, default: `{{variables.editSchedulerData.name \|\| ''}}` |
| 2 | Textarea | `sfDescription` | Textarea | Label: `Description`, rows: 2, default: `{{variables.editSchedulerData.description \|\| ''}}` |
| 3 | Dropdown | `sfTemplate` | Dropdown | Label: `Template *`, options: `{{queries.listAllTemplates.data}}`, default: `{{variables.editSchedulerData.template_id}}`, searchInOptions: true |
| 4 | **(See: Dynamic Parameter Form)** | — | — | Replaces the old Code Editor |
| 5 | Dropdown | `sfCronPreset` | Dropdown | Label: `Schedule preset`, options: see Cron Preset Dropdown below, value: `''` |
| 6 | Text Input | `sfCron` | Text Input | Label: `Cron Expression *`, placeholder: `0 9 * * 1`, default: `{{variables.editSchedulerData.cron_expression \|\| ''}}`, font: mono |
| 7 | Text | `sfCronNextRuns` | Text | Content: `{{variables.cronNextRuns.length > 0 ? 'Next runs: ' + variables.cronNextRuns.join(', ') : 'Enter a cron expression or pick a preset to see next run times'}}`, font: 11px, muted |
| 8 | Dropdown | `sfTimezone` | Dropdown | Label: `Timezone`, options: timezone list, default: `{{variables.editSchedulerData.timezone \|\| 'Asia/Ho_Chi_Minh'}}`, searchInOptions: true |
| 9 | Text Input | `sfRecipients` | Text Input | Label: `Recipients (comma-separated emails)`, placeholder: `user1@co.com, user2@co.com`, default: `{{variables.editSchedulerData.recipients \|\| ''}}` |
| 10 | Text Input | `sfEmailSubject` | Text Input | Label: `Email Subject`, placeholder: `Weekly Report - {{YYYY-MM-DD}}`, default: `{{variables.editSchedulerData.email_subject \|\| ''}}` |
| 11 | Textarea | `sfEmailBody` | Textarea | Label: `Email Body`, rows: 4, default: `{{variables.editSchedulerData.email_body \|\| ''}}` |
| 12 | Button | `btnSaveScheduler` | Button | Label: `Save`, variant: primary |
| 13 | Button | `btnCancelScheduler` | Button | Label: `Cancel`, variant: outline |

---

### Dynamic Parameter Form (replaces `sfParameters` Code Editor)

Same Postman-style row editor pattern as the Test tab on page 04 — lets the scheduler builder add parameters one row at a time with a type picker, while still supporting date placeholders like `{{TODAY}}` / `{{FIRST_DAY_OF_MONTH}}` in String / Date values.

#### Layout

```text
┌─ Parameters ────────────────────────────────────────────────┐
│ [x] Simple form    ( ) Raw JSON                              │
├─ Simple form (default) ─────────────────────────────────────┤
│  Key            Type     Value                               │
│  [date_from]   [Date ▼] [{{FIRST_DAY_OF_MONTH}}] [📅] [🗑]  │
│  [region   ]   [String▼] [north                 ]       [🗑]│
│  [limit    ]   [Number▼] [1000                  ]       [🗑]│
│  [+ Add parameter]    [Insert placeholder ▼]                 │
├─ Raw JSON mode ─────────────────────────────────────────────┤
│  { "date_from": "{{FIRST_DAY_OF_MONTH}}", ... }              │
└─────────────────────────────────────────────────────────────┘
```

#### Components (inside the modal)

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Toggle | `sfParamsMode` | Toggle | Label: `Raw JSON`, value: false |
| 2 | ListView | `sfParamsList` | ListView | Data: `{{variables.schedulerParamRows}}`, visible: `{{!components.sfParamsMode.value}}`, row height: auto |
| 3 | Button | `btnAddSfParam` | Button | Label: `+ Add parameter`, variant: outline, visible: `{{!components.sfParamsMode.value}}` |
| 4 | Dropdown | `sfInsertPlaceholder` | Dropdown | Label: `Insert placeholder`, options: see Date Placeholders below (value = placeholder text), visible: `{{!components.sfParamsMode.value}}` |
| 5 | Code Editor | `sfParamsRaw` | Code Editor | Mode: JSON, visible: `{{components.sfParamsMode.value}}`, default: built from `variables.schedulerParamRows` |

**Row children (per row):**

Each row carries `{id, key, type, value}`. Like page 04's Test tab, pass `{{self.value}}` into the helper to capture the component that fired the event.

| # | Component | Properties |
|---|---|---|
| Text Input | Value: `{{listItem.key}}`, on blur → `sfUpdateParam` with `{id: listItem.id, field:'key', value: {{self.value}}}` |
| Dropdown | Options: `[{label:'String',value:'string'},{label:'Number',value:'number'},{label:'Boolean',value:'boolean'},{label:'Date',value:'date'},{label:'JSON',value:'json'}]`, value: `{{listItem.type}}`, on change → `sfUpdateParam` with `{id: listItem.id, field:'type', value: {{self.value}}}` |
| Dynamic value input (switched by `listItem.type`) — fires `sfUpdateParam` with `{id, field:'value', value: {{self.value}}}` on change | See page 04 Test tab row children — same 5 variants (Text / Number / Checkbox / Date / Code Editor) |
| Button | Label: `🗑`, variant: ghost → `sfDeleteParam` with `{id: listItem.id}` |

#### Helpers

**`sfLoadParamsForEdit` (on modal show)** — parses the stored JSON string back into rows:

```javascript
const raw = variables.editSchedulerData.parameters;
if (!raw) { await actions.setVariable('schedulerParamRows', []); return; }

let obj;
try { obj = typeof raw === 'string' ? JSON.parse(raw) : raw; }
catch (e) { obj = {}; }

const rows = Object.entries(obj).map(([k, v]) => {
  let type = 'string';
  const isPlaceholder = typeof v === 'string' && /\{\{[A-Z_]+\}\}/.test(v);
  if (typeof v === 'number') type = 'number';
  else if (typeof v === 'boolean') type = 'boolean';
  else if (typeof v === 'object' && v !== null) type = 'json';
  else if (isPlaceholder && /DATE|DAY|TODAY|YESTERDAY|MONTH|WEEK|YEAR/.test(v)) type = 'date';
  return { id: Date.now() + '_' + Math.random(), key: k, type, value: v };
});
await actions.setVariable('schedulerParamRows', rows);
```

**`btnAddSfParam` on click:**

```javascript
const rows = [...(variables.schedulerParamRows || [])];
rows.push({ id: Date.now() + '_' + Math.random(), key: '', type: 'string', value: '' });
await actions.setVariable('schedulerParamRows', rows);
```

**`sfUpdateParam` (parameters: `id`, `field`, `value`):**

```javascript
const rows = (variables.schedulerParamRows || []).map(r =>
  r.id === parameters.id ? { ...r, [parameters.field]: parameters.value } : r
);
await actions.setVariable('schedulerParamRows', rows);
```

**`sfDeleteParam` (parameter: `id`):**

```javascript
await actions.setVariable(
  'schedulerParamRows',
  (variables.schedulerParamRows || []).filter(r => r.id !== parameters.id)
);
```

**`sfParamsMode` on change** — seed Raw JSON editor from rows when switching mode (same pattern as page 04 Test tab):

```javascript
if (components.sfParamsMode.value) {
  const obj = {};
  for (const row of (variables.schedulerParamRows || [])) {
    if (!row.key) continue;
    obj[row.key] = row.value;
  }
  await actions.setComponentValue('sfParamsRaw', JSON.stringify(obj, null, 2));
}
```

**`sfInsertPlaceholder` on select** — appends the picked placeholder token to the **last focused** Text Input / Date value cell. Simpler alternative: copy the token to clipboard and show a toast "Placeholder copied — paste into a value field".

**`sfBuildParametersJSON` (JavaScript — called on save)** — returns a **JSON string** because the DB column is `varchar`:

```javascript
if (components.sfParamsMode.value) {
  // Raw mode: validate then passthrough
  try {
    JSON.parse(components.sfParamsRaw.value || '{}');
    return components.sfParamsRaw.value || '{}';
  } catch (e) {
    throw new Error('Invalid JSON in parameters: ' + e.message);
  }
}

const out = {};
for (const row of (variables.schedulerParamRows || [])) {
  if (!row.key) continue;
  let v = row.value;
  if (typeof v === 'string' && /\{\{[A-Z_]+\}\}/.test(v)) {
    // Keep placeholders as strings even if type is Date/Number — resolver handles them at run time
    out[row.key] = v;
    continue;
  }
  switch (row.type) {
    case 'number':  v = Number(v); break;
    case 'boolean': v = v === true || v === 'true'; break;
    case 'json':    v = typeof v === 'string' ? JSON.parse(v || 'null') : v; break;
    default:        v = String(v ?? '');
  }
  out[row.key] = v;
}
return JSON.stringify(out);
```

---

### Cron Preset Dropdown

`sfCronPreset` options (apply the selected preset into `sfCron`):

```javascript
[
  {label: '— Custom —',               value: ''},
  {label: 'Daily at 07:00',           value: '0 7 * * *'},
  {label: 'Daily at 09:00',           value: '0 9 * * *'},
  {label: 'Weekdays 09:00',           value: '0 9 * * 1-5'},
  {label: 'Monday 09:00',             value: '0 9 * * 1'},
  {label: 'First day of month 08:00', value: '0 8 1 * *'},
  {label: 'Every hour (on the hour)', value: '0 * * * *'},
  {label: 'Every 15 minutes',         value: '*/15 * * * *'},
]
```

**`sfCronPreset` on change:**

| Event | Action |
|---|---|
| On select | If value not empty: `setComponentValue(sfCron, value)` → Run `sfPreviewNextRuns` |

**`sfCron` on change:**

| Event | Action |
|---|---|
| On change (debounce 300ms) | Run `sfPreviewNextRuns`. Also if value no longer matches the preset → set `sfCronPreset` to `''` (Custom) |

**`sfPreviewNextRuns` (JavaScript)** — computes the next 3 run times in the chosen timezone.

```javascript
// Minimal cron matcher (minute hour dom month dow). No seconds, no special chars beyond * / , -
function matchField(val, part) {
  if (part === '*') return true;
  return part.split(',').some(seg => {
    let step = 1, range = seg;
    if (seg.includes('/')) { [range, step] = seg.split('/'); step = Number(step); }
    if (range === '*') return val % step === 0;
    if (range.includes('-')) {
      const [lo, hi] = range.split('-').map(Number);
      return val >= lo && val <= hi && ((val - lo) % step === 0);
    }
    return Number(range) === val;
  });
}

function nextRuns(cron, tz, count = 3) {
  const parts = (cron || '').trim().split(/\s+/);
  if (parts.length !== 5) return [];
  const [m, h, dom, mon, dow] = parts;
  const out = [];
  // Scan forward minute-by-minute up to 1 year (525,600 iterations worst case) — fine in JS
  let d = new Date();
  d.setSeconds(0, 0); d.setMinutes(d.getMinutes() + 1);
  const end = new Date(d.getTime() + 366 * 24 * 60 * 60 * 1000);
  while (d < end && out.length < count) {
    const local = new Date(d.toLocaleString('en-US', {timeZone: tz}));
    if (
      matchField(local.getMinutes(), m) &&
      matchField(local.getHours(), h) &&
      matchField(local.getDate(), dom) &&
      matchField(local.getMonth() + 1, mon) &&
      matchField(local.getDay(), dow)
    ) {
      out.push(local.toLocaleString('sv-SE', {timeZone: tz}).slice(0, 16));
    }
    d = new Date(d.getTime() + 60 * 1000);
  }
  return out;
}

return nextRuns(components.sfCron.value, components.sfTimezone.value || 'Asia/Ho_Chi_Minh', 3);
```

On success → `setVariable('cronNextRuns', data)`. On empty/invalid → return `[]` so the hint text falls back to the placeholder message.

> **Trade-off:** the inline matcher is ~25 lines and covers the common cases (ranges, steps, lists). If you need special tokens like `L`, `W`, `#`, bundle `cron-parser` as a ToolJet library dependency instead.

**Timezone dropdown options** (common timezones):

```javascript
[
  {label: "UTC", value: "UTC"},
  {label: "Asia/Ho_Chi_Minh (ICT +7)", value: "Asia/Ho_Chi_Minh"},
  {label: "Asia/Tokyo (JST +9)", value: "Asia/Tokyo"},
  {label: "Asia/Singapore (SGT +8)", value: "Asia/Singapore"},
  {label: "America/New_York (EST -5)", value: "America/New_York"},
  {label: "America/Los_Angeles (PST -8)", value: "America/Los_Angeles"},
  {label: "Europe/London (GMT)", value: "Europe/London"},
  {label: "Europe/Berlin (CET +1)", value: "Europe/Berlin"},
  {label: "Australia/Sydney (AEST +10)", value: "Australia/Sydney"},
]
```

**`btnSaveScheduler` event chain:**

1. Run `sfBuildParametersJSON` — validates + stringifies parameters. On failure → show the error alert and stop.
2. If `isEditing`: run `updateScheduler`. Else: run `createScheduler`.

**`btnCancelScheduler` event:**

| Event | Action |
|---|---|
| On click | Close `schedulerFormModal` |

### C6: `confirmDeleteModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `Delete Scheduler` |
| Size | Small |

| Component | Properties |
|---|---|
| Text | `Are you sure you want to delete this scheduler?` |
| Button "Delete" | Variant: destructive → Run `deleteScheduler` → Close modal |
| Button "Cancel" | Variant: outline → Close modal |

## Date Placeholders

Parameters can include date placeholders that are resolved at execution time.

| Placeholder | Resolves To | Example |
|---|---|---|
| `{{TODAY}}` | Current date | `2026-04-17` |
| `{{YESTERDAY}}` | Previous day | `2026-04-16` |
| `{{FIRST_DAY_OF_WEEK}}` | Monday of current week | `2026-04-14` |
| `{{LAST_DAY_OF_WEEK}}` | Sunday of current week | `2026-04-20` |
| `{{FIRST_DAY_OF_MONTH}}` | 1st of current month | `2026-04-01` |
| `{{LAST_DAY_OF_MONTH}}` | Last day of month | `2026-04-30` |
| `{{FIRST_DAY_OF_YEAR}}` | Jan 1st | `2026-01-01` |
| `{{YYYY-MM-DD}}` | Current date (for email subject) | `2026-04-17` |
| `{{YYYY-MM}}` | Current month | `2026-04` |

**Resolver function** (used in Workflow JavaScript node):

```javascript
function resolveParameters(paramsJson) {
  const now = new Date();
  const yyyy = now.getFullYear();
  const mm = String(now.getMonth() + 1).padStart(2, '0');
  const dd = String(now.getDate()).padStart(2, '0');
  const today = `${yyyy}-${mm}-${dd}`;

  const yesterday = new Date(now);
  yesterday.setDate(yesterday.getDate() - 1);
  const yd = `${yesterday.getFullYear()}-${String(yesterday.getMonth()+1).padStart(2,'0')}-${String(yesterday.getDate()).padStart(2,'0')}`;

  const dow = now.getDay();
  const monday = new Date(now);
  monday.setDate(now.getDate() - (dow === 0 ? 6 : dow - 1));
  const sunday = new Date(monday);
  sunday.setDate(monday.getDate() + 6);

  const firstOfMonth = `${yyyy}-${mm}-01`;
  const lastOfMonth = new Date(yyyy, now.getMonth() + 1, 0);
  const lastOfMonthStr = `${yyyy}-${mm}-${String(lastOfMonth.getDate()).padStart(2,'0')}`;

  const replacements = {
    '{{TODAY}}': today,
    '{{YESTERDAY}}': yd,
    '{{FIRST_DAY_OF_WEEK}}': monday.toISOString().split('T')[0],
    '{{LAST_DAY_OF_WEEK}}': sunday.toISOString().split('T')[0],
    '{{FIRST_DAY_OF_MONTH}}': firstOfMonth,
    '{{LAST_DAY_OF_MONTH}}': lastOfMonthStr,
    '{{FIRST_DAY_OF_YEAR}}': `${yyyy}-01-01`,
    '{{YYYY-MM-DD}}': today,
    '{{YYYY-MM}}': `${yyyy}-${mm}`,
  };

  let result = paramsJson || '{}';
  for (const [key, val] of Object.entries(replacements)) {
    result = result.replaceAll(key, val);
  }
  return JSON.parse(result);
}
```

---

## ToolJet Workflow: `report-scheduler`

> **Note:** ToolJet Workflows is a paid feature. If not available, use Option B (external cron) below.

### Setup

1. Go to **Workflows** in ToolJet sidebar → **Create New Workflow** → name: `report-scheduler`
2. Add **Trigger** → **Scheduler** → Interval mode: every **5 minutes** (or Cron: `*/5 * * * *`)
3. Build the flow below

### Workflow Nodes

```text
[Start] → [getActiveSchedulers] → [filterDueSchedulers] → [Loop] → [Response]
                                                              │
                                                    For each scheduler:
                                                    [generateReport]
                                                        → [pollLoop]
                                                        → [readReportFile]
                                                        → [sendEmail]
                                                        → [updateLastRun]
```

### Node 1: `getActiveSchedulers` (ToolJetDB)

| Property | Value |
|---|---|
| Node type | ToolJetDB |
| Table | `schedulers` |
| Operation | List rows |
| Filter | `is_active` equals `true` |

### Node 2: `filterDueSchedulers` (JavaScript)

```javascript
// Check which schedulers are due based on cron_expression
// Simple approach: compare last_run_at with cron interval
const schedulers = getActiveSchedulers.data;
const now = new Date();

return schedulers.filter(s => {
  if (!s.cron_expression) return false;

  // If never run, it's due
  if (!s.last_run_at) return true;

  const lastRun = new Date(s.last_run_at);
  const diffMinutes = (now - lastRun) / 60000;

  // Parse simple cron patterns
  const parts = s.cron_expression.split(' ');
  const [minute, hour, dayOfMonth, month, dayOfWeek] = parts;

  // Check if current time matches cron
  if (minute !== '*' && now.getMinutes() !== parseInt(minute)) return false;
  if (hour !== '*' && now.getHours() !== parseInt(hour)) return false;
  if (dayOfMonth !== '*' && now.getDate() !== parseInt(dayOfMonth)) return false;
  if (dayOfWeek !== '*' && now.getDay() !== parseInt(dayOfWeek)) return false;

  // Prevent re-run within 4 minutes (since workflow runs every 5 min)
  if (diffMinutes < 4) return false;

  return true;
});
```

### Node 3: `Loop` over due schedulers

| Property | Value |
|---|---|
| Node type | Loop |
| Data | `{{filterDueSchedulers.data}}` |

**Inside Loop — Looped function:** Chain of nodes:

#### Loop Step 1: `generateReport` (REST API — pyDBAPI)

| Property | Value |
|---|---|
| Data source | `pyDBAPI` |
| Method | POST |
| URL | `/api/v1/report-modules/{{value.module_id}}/templates/{{value.template_id}}/generate` |
| Body | `{"parameters": {{resolveParameters(value.parameters)}}}` |

> `resolveParameters` is the JavaScript function defined above, called inline or in a preceding JS node.

#### Loop Step 2: `pollLoop` (JavaScript)

```javascript
const execId = generateReport.data.execution_id;
let status = 'pending';
let attempts = 0;

while (status === 'pending' || status === 'running') {
  if (attempts++ > 60) break; // 2 min timeout
  await new Promise(r => setTimeout(r, 2000));

  const resp = await datasources.pyDBAPI.GET(
    `/api/v1/report-modules/client/execution-detail?exec_id=${execId}`
  );
  status = resp.data.status;
}

return { execution_id: execId, status, output_minio_path: resp?.data?.output_minio_path };
```

#### Loop Step 3: `readReportFile` (MinIO) — if success

| Property | Value |
|---|---|
| Data source | `MinIO` |
| Operation | **Read Object** |
| Bucket | parsed from `pollLoop.data.output_minio_path` (first segment) |
| Object Name | parsed path (remaining segments) |

> Workflow runs server-side — MinIO is accessible from ToolJet server.

#### Loop Step 4: `sendEmail` (SMTP) — if success and recipients exist

| Property | Value |
|---|---|
| Data source | SMTP (configure separately) |
| To | `{{value.recipients}}` |
| Subject | `{{resolveParameters(value.email_subject)}}` |
| Body | `{{resolveParameters(value.email_body)}}` |
| Attachment | `{{readReportFile.data}}` (base64 file content) |

> If SMTP data source doesn't support attachments, include a note in the email body asking recipients to download from the Data Portal app.

#### Loop Step 5: `updateLastRun` (ToolJetDB)

| Property | Value |
|---|---|
| Node type | ToolJetDB |
| Table | `schedulers` |
| Operation | Update row |
| Filter | `id` equals `{{value.id}}` |
| Set `last_run_at` | `{{new Date().toISOString()}}` |
| Set `last_status` | `{{pollLoop.data.status}}` |
| Set `last_error` | `{{pollLoop.data.status === 'failed' ? pollLoop.data.error_message : null}}` |

### Node 4: `Response`

```javascript
return {
  processed: filterDueSchedulers.data.length,
  timestamp: new Date().toISOString(),
};
```

---

## Alternative: External Cron (no Workflow license)

If ToolJet Workflows (paid feature) is not available, use an external cron service:

```python
# scheduler_runner.py — runs as a separate service
import json
import time
from datetime import datetime

import requests
from apscheduler.schedulers.blocking import BlockingScheduler

TOOLJET_DB_URL = "https://tooljet-host/api/tooljet-db"
PYDBAPI_URL = "http://pydbapi-host:8000"
CLIENT_ID = "tooljet-reports"
CLIENT_SECRET = "your-secret"

def get_token():
    resp = requests.post(f"{PYDBAPI_URL}/api/token/generate",
        data={"grant_type": "client_credentials",
              "client_id": CLIENT_ID, "client_secret": CLIENT_SECRET})
    return resp.json()["access_token"]

def run_due_schedulers():
    # 1. Read active schedulers from ToolJet DB
    # 2. Check which are due (cron match)
    # 3. For each due: generate → poll → read file → send email with attachment
    # 4. Update last_run_at/status in ToolJet DB
    pass

scheduler = BlockingScheduler()
scheduler.add_job(run_due_schedulers, 'interval', minutes=5)
scheduler.start()
```

The ToolJet page CRUD works the same regardless — it only manages the `schedulers` table.

## Event Flow

```text
Page:
1. Page load → listSchedulers + listAllTemplates
2. [+ Create] → open schedulerFormModal (create mode) → fill form → Save → createScheduler
3. [Edit ⚙] → open schedulerFormModal (edit mode) → edit → Save → updateScheduler
4. [Toggle] → toggleScheduler (flip is_active)
5. [Run Now ▶] → runNow (pyDBAPI generate) → update last_run/status
6. [Delete 🗑] → confirmDeleteModal → deleteScheduler

Workflow (background, every 5 min):
7. getActiveSchedulers → filterDueSchedulers → Loop:
   generate → poll → readReportFile → email (with attachment) → updateLastRun
```
