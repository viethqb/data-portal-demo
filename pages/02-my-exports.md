# Page 2: My Exports

**Handle:** `exports` | **Permissions:** All users | **Primary page for End Users**

## Layout

```text
┌─────────────────────────────────────────────────────────────────────┐
│  [headerText]                                                       │
│  [searchInput]  [moduleFilter]                                      │
├─────────────────────────────────────────────────────────────────────┤
│  [templateList]                                                     │
│    ┌──────────────────┐  ┌──────────────────┐                      │
│    │ [cardTitle]       │  │ [cardTitle]       │                      │
│    │ [cardDesc]        │  │ [cardDesc]        │                      │
│    │ [btnExportNow]    │  │ [btnExportNow]    │                      │
│    └──────────────────┘  └──────────────────┘                      │
├─────────────────────────────────────────────────────────────────────┤
│  [exportModal]                                                      │
│    [modalHeader]                                                    │
│    [filterForm]                                                     │
│    [btnGenerate]                                                    │
│    [statusSection]                                                  │
│    [successSection] / [failSection]                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Page Events

| Event | Action |
|---|---|
| On page load | Run: `listAllTemplates`, `getMyPermissions`, `getUserRole`, `listModulesForFilter` |

## Queries

### Q1: `listAllTemplates`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 100, "is_active": true}` |
| Run on page load | Yes |

### Q2: `getMyPermissions`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | List rows |
| Filter | `user_email` equals `{{globals.currentUser.email}}` |
| Run on page load | Yes |

### Q3: `getVisibleTemplates`

| Property | Value |
|---|---|
| Data source | JavaScript |
| Run on page load | Yes (after Q1 + Q2 complete) |

```javascript
const allTemplates = queries.listAllTemplates.data?.data || [];
const role = queries.getUserRole.data;

if (role !== 'end_user') return allTemplates;

const permittedIds = new Set(
  (queries.getMyPermissions.data || []).map(p => p.template_id)
);
return allTemplates.filter(t => permittedIds.has(t.id));
```

### Q4: `listModulesForFilter`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules` |
| Body | `{"page": 1, "page_size": 100}` |
| Run on page load | Yes |
| Transform | `return data.data.map(m => ({label: m.name, value: m.id}))` |

### Q5: `generateReport`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/{{variables.selectedTemplate.report_module_id}}/templates/{{variables.selectedTemplate.id}}/generate` |
| Body | `{"parameters": {{JSON.stringify(components.filterForm.formData || {})}}}` |
| Run on demand | Button click |

| Event | Action |
|---|---|
| On success | 1. Run `logExport` → 2. Run `pollLoop` |
| On failure | Set variable `exportStatus` = `failed`, `errorMessage` = `{{queries.generateReport.rawData?.detail}}` |

### Q6: `pollExecution`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/execution-detail?exec_id={{queries.generateReport.data.execution_id}}` |
| Run on demand | Called by `pollLoop` |

### Q7: `pollLoop`

| Property | Value |
|---|---|
| Data source | JavaScript |
| Run on demand | After generateReport success |

```javascript
await actions.setVariable('exportStatus', 'running');
await queries.pollExecution.run();
const status = queries.pollExecution.data?.status;

if (status === 'success') {
  await queries.updateExportStatus.run({
    status: 'success',
    executionId: queries.generateReport.data.execution_id,
  });
  await actions.setVariable('exportStatus', 'success');
  await actions.setVariable('executionId', queries.generateReport.data.execution_id);
  return;
}

if (status === 'failed') {
  await queries.updateExportStatus.run({
    status: 'failed',
    executionId: queries.generateReport.data.execution_id,
  });
  await actions.setVariable('exportStatus', 'failed');
  await actions.setVariable('errorMessage', queries.pollExecution.data?.error_message);
  return;
}

await new Promise(r => setTimeout(r, 2000));
return await queries.pollLoop.run();
```

### Q8: `logExport`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `export_log` |
| Operation | Create row |

| Column | Value |
|---|---|
| `user_email` | `{{globals.currentUser.email}}` |
| `user_name` | `{{globals.currentUser.firstName}} {{globals.currentUser.lastName}}` |
| `template_id` | `{{variables.selectedTemplate.id}}` |
| `template_name` | `{{variables.selectedTemplate.name}}` |
| `execution_id` | `{{queries.generateReport.data.execution_id}}` |
| `parameters` | `{{JSON.stringify(components.filterForm.formData || {})}}` |
| `status` | `pending` |
| `created_at` | `{{new Date().toISOString()}}` |

### Q9: `updateExportStatus`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `export_log` |
| Operation | Update row |
| Filter | `execution_id` equals `{{parameters.executionId}}` |
| Set `status` | `{{parameters.status}}` |

## Components

### C1: `headerText`

| Property | Value |
|---|---|
| Component | Text |
| Content | `My Exports` |
| Font size | 24px, bold |

### C2: `searchInput`

| Property | Value |
|---|---|
| Component | Text Input |
| Placeholder | `Search templates...` |
| Width | 300px |

| Event | Action |
|---|---|
| On change (debounce 500ms) | Run `getVisibleTemplates` (client-side filter) |

### C3: `moduleFilter`

| Property | Value |
|---|---|
| Component | Dropdown |
| Label | `Module` |
| Options | `{{[{label: 'All Modules', value: ''}].concat(queries.listModulesForFilter.data || [])}}` |
| Default value | `''` (All) |
| Width | 200px |

| Event | Action |
|---|---|
| On select | Run `listAllTemplates` with body including `"module_id": "{{components.moduleFilter.value}}"` → then run `getVisibleTemplates` |

### C4: `templateList`

| Property | Value |
|---|---|
| Component | ListView |
| Data | `{{queries.getVisibleTemplates.data}}` |
| Row height | 180 |
| Columns | 2 |
| Show border | Yes |
| Border radius | 8px |

**Children per row:**

| # | Component | ID | Properties |
|---|---|---|---|
| 1 | Text | `tplName` | Content: `{{listItem.name}}`, font: 16px bold |
| 2 | Text | `tplDesc` | Content: `{{listItem.description \|\| 'No description'}}`, font: 13px, color: muted, max-lines: 2 |
| 3 | Text | `tplMeta` | Content: `Output: {{listItem.output_sheet \|\| 'Full file'}}`, font: 11px, color: muted |
| 4 | Button | `btnExportNow` | Label: `Export Now`, variant: primary, full width |

**Button event:**

| Event | Action |
|---|---|
| On click | 1. Set variable `selectedTemplate` = `{{listItem}}` |
| | 2. Set variable `exportStatus` = `idle` |
| | 3. Set variable `errorMessage` = `''` |
| | 4. Show modal `exportModal` |

### C5: `exportModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `Export: {{variables.selectedTemplate.name}}` |
| Size | Large |
| Close on click outside | No |
| Show footer | No |

**Children layout inside modal:**

```text
┌─ Modal Body ────────────────────────────────────────────────┐
│  [tplInfoText]                                              │
│  ─── Filters ───                                            │
│  [filterForm]                                               │
│  [btnReset]  [btnGenerate]                                  │
│  ─── Status ───                                             │
│  [spinnerRunning]  [textRunning]     (visible when running) │
│  [iconSuccess] [textSuccess]         (visible when success) │
│  [btnDownload] [btnAgain]            (visible when success) │
│  [iconFail] [textError]              (visible when failed)  │
│  [btnRetry] [btnClose]               (visible when failed)  │
└─────────────────────────────────────────────────────────────┘
```

### C5a: `tplInfoText`

| Property | Value |
|---|---|
| Component | Text |
| Content | `Module: {{variables.selectedTemplate.report_module_id}}` |
| Font size | 13px, muted |

### C5b: `filterForm`

| Property | Value |
|---|---|
| Component | Form |
| Show submit button | No (we use custom button) |

**Children inside form** (common filter fields):

| # | Component | ID | Properties | Visible |
|---|---|---|---|---|
| 1 | **Date Range Picker** | `dateRange` | Label: `Date Range`, format: `YYYY-MM-DD`, defaultStartDate: 30 days ago, defaultEndDate: today | Always |
| 2 | Dropdown | `department` | Label: `Department`, placeholder: `All`, options: dynamic or static | Optional |
| 3 | Text Input | `search` | Label: `Search`, placeholder: `Keyword` | Optional |

> **Note:** Filter fields depend on the template's SQL parameters. Access date range values via `{{components.dateRange.startDate}}` and `{{components.dateRange.endDate}}`. For advanced: parse template description JSON to render dynamic fields.

### C5c: `btnReset`

| Property | Value |
|---|---|
| Component | Button |
| Label | `Reset Defaults` |
| Variant | Outline |
| Visible | `{{variables.exportStatus === 'idle'}}` |

| Event | Action |
|---|---|
| On click | `await components.filterForm.resetForm()` |

### C5d: `btnGenerate`

| Property | Value |
|---|---|
| Component | Button |
| Label | `Generate Report` |
| Variant | Primary |
| Loading | `{{queries.generateReport.isLoading}}` |
| Disabled | `{{variables.exportStatus === 'running'}}` |
| Visible | `{{variables.exportStatus === 'idle'}}` |

| Event | Action |
|---|---|
| On click | Run `generateReport` |

### C5e: Running Status

| Component | ID | Properties | Visible |
|---|---|---|---|
| Spinner | `spinnerRunning` | Size: small | `{{variables.exportStatus === 'running'}}` |
| Text | `textRunning` | Content: `Generating report...` | `{{variables.exportStatus === 'running'}}` |

### C5f: Success Section

| Component | ID | Properties | Visible |
|---|---|---|---|
| Text | `textSuccess` | Content: `✅ Report generated successfully!`, color: green | `{{variables.exportStatus === 'success'}}` |
| Button | `btnDownload` | Label: `Download`, variant: primary | `{{variables.exportStatus === 'success'}}` |
| Button | `btnAgain` | Label: `Generate Again`, variant: outline | `{{variables.exportStatus === 'success'}}` |

| Button | Event | Action |
|---|---|---|
| `btnDownload` | On click | Run `downloadExecution` with `executionId: {{variables.executionId}}` |
| `btnAgain` | On click | Set variable `exportStatus` = `idle` |

### C5g: Failure Section

| Component | ID | Properties | Visible |
|---|---|---|---|
| Text | `textError` | Content: `❌ {{variables.errorMessage}}`, color: red, font: mono | `{{variables.exportStatus === 'failed'}}` |
| Button | `btnRetry` | Label: `Retry`, variant: primary | `{{variables.exportStatus === 'failed'}}` |
| Button | `btnCloseFail` | Label: `Close`, variant: outline | `{{variables.exportStatus === 'failed'}}` |

| Button | Event | Action |
|---|---|---|
| `btnRetry` | On click | Set `exportStatus` = `idle` |
| `btnCloseFail` | On click | Close modal `exportModal` |

## Variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `selectedTemplate` | Object | `{}` | Template being exported |
| `exportStatus` | String | `idle` | `idle` → `running` → `success` / `failed` |
| `executionId` | String | `''` | For download after success |
| `errorMessage` | String | `''` | Error text on failure |

## Event Flow

```text
1. Page loads → listAllTemplates + getMyPermissions + getUserRole + listModulesForFilter
2. getVisibleTemplates merges → templateList shows filtered cards
3. User clicks "Export Now" → sets selectedTemplate → opens exportModal
4. User fills filterForm → clicks "Generate"
5. generateReport runs → on success:
   a. logExport writes to ToolJet DB export_log
   b. pollLoop starts polling pollExecution every 2s
6. On success: updateExportStatus → show download button
7. On failure: updateExportStatus → show error message
8. User clicks "Download" → downloadExecution opens file
```
