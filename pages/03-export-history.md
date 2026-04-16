# Page 3: Export History

**Handle:** `history` | **Permissions:** All users (scoped by role)

## Layout

```text
┌─────────────────────────────────────────────────────────────────────┐
│  [headerText]                                                       │
│  [statusFilter]  [templateFilter]                                   │
├─────────────────────────────────────────────────────────────────────┤
│  [historyTable]                                                     │
│  [pagination]                                                       │
├─────────────────────────────────────────────────────────────────────┤
│  [errorDetailModal]                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Page Events

| Event | Action |
|---|---|
| On page load | Run: `listExportHistory`, `getUserRole` |

## Queries

### Q1a: `listHistoryAll` (for admin/da)

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `export_log` |
| Operation | List rows |
| Sort by | `created_at` DESC |
| Limit | 20 |
| Offset | `{{(variables.currentPage - 1) * 20}}` |
| Filter | (none — see all users) |

### Q1b: `listHistoryMine` (for end_user)

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `export_log` |
| Operation | List rows |
| Sort by | `created_at` DESC |
| Limit | 20 |
| Offset | `{{(variables.currentPage - 1) * 20}}` |
| Filter | `user_email` equals `{{globals.currentUser.email}}` |

### Q1: `listExportHistory` (JavaScript — picks query + client-side filter)

```javascript
const role = queries.getUserRole.data;
let data;

if (role === 'end_user') {
  await queries.listHistoryMine.run();
  data = queries.listHistoryMine.data || [];
} else {
  await queries.listHistoryAll.run();
  data = queries.listHistoryAll.data || [];
}

// Client-side filter by status and template name
const statusVal = components.statusFilter?.value;
const templateVal = components.templateFilter?.value;

if (statusVal) {
  data = data.filter(r => r.status === statusVal);
}
if (templateVal) {
  data = data.filter(r => (r.template_name || '').toLowerCase().includes(templateVal.toLowerCase()));
}

return data;
```

| Property | Value |
|---|---|
| Run on page load | Yes (after `getUserRole` completes) |

### Q2: `getExecutionDetail`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/execution-detail?exec_id={{variables.selectedExecId}}` |
| Run on demand | Row action click |

## Components

### C1: `headerText`

| Property | Value |
|---|---|
| Component | Text |
| Content | `Export History` |
| Font size | 24px, bold |

### C2: `statusFilter`

| Property | Value |
|---|---|
| Component | Dropdown |
| Label | `Status` |
| Width | 150px |
| Options | `[{label: "All", value: ""}, {label: "Success", value: "success"}, {label: "Failed", value: "failed"}, {label: "Pending", value: "pending"}]` |
| Default value | `""` |

| Event | Action |
|---|---|
| On select | Run `listExportHistory` |

### C3: `templateFilter`

| Property | Value |
|---|---|
| Component | Text Input |
| Placeholder | `Filter by template name...` |
| Width | 250px |

| Event | Action |
|---|---|
| On change (debounce 500ms) | Run `listExportHistory` |

### C4: `historyTable`

| Property | Value |
|---|---|
| Component | Table |
| Data | `{{queries.listExportHistory.data}}` |
| Show search | No (we have custom filters) |
| Show pagination | No (custom pagination below) |
| Loading state | `{{queries.listExportHistory.isLoading}}` |

**Columns:**

| # | Header | Key | Type | Width | Config |
|---|---|---|---|---|---|
| 1 | Template | `template_name` | Default | 200px | — |
| 2 | User | `user_name` | Default | 150px | **Visible:** `{{queries.getUserRole.data !== 'end_user'}}` |
| 3 | Status | `status` | **Select** | 100px | Options: `[{label:'success', value:'success', backgroundColor:'#10b981'}, {label:'failed', value:'failed', backgroundColor:'#ef4444'}, {label:'pending', value:'pending', backgroundColor:'#f59e0b'}]` |
| 4 | Parameters | `parameters` | Default | 200px | Cell: `{{cellValue \|\| '--'}}`, text truncate |
| 5 | Time | `created_at` | Default | 160px | Cell: `{{new Date(cellValue).toLocaleString()}}` |
| 6 | Actions | — | Action buttons | 120px | Two buttons: Download + Detail |

**Action button 1: Download**

| Property | Value |
|---|---|
| Label | `⬇` |
| Visible | `{{rowData.status === 'success' && rowData.execution_id}}` |

| Event | Action |
|---|---|
| On click | Run `downloadExecution` with `executionId: {{rowData.execution_id}}` |

**Action button 2: Detail / Error**

| Property | Value |
|---|---|
| Label | `👁` |
| Visible | `{{rowData.status === 'failed'}}` |

| Event | Action |
|---|---|
| On click | 1. Set variable `selectedExecId` = `{{rowData.execution_id}}` |
| | 2. Set variable `selectedExportRow` = `{{rowData}}` |
| | 3. Run `getExecutionDetail` |
| | 4. Show modal `errorDetailModal` |

### C5: Pagination (Container with buttons)

| Component | ID | Properties |
|---|---|---|
| Text | `pageInfo` | Content: `Page {{variables.currentPage}}` |
| Button | `btnPrev` | Label: `← Prev`, disabled: `{{variables.currentPage <= 1}}` |
| Button | `btnNext` | Label: `Next →`, disabled: `{{queries.listExportHistory.data.length < 20}}` |

| Button | Event | Action |
|---|---|---|
| `btnPrev` | On click | Set `currentPage` = `{{variables.currentPage - 1}}` → Run `listExportHistory` |
| `btnNext` | On click | Set `currentPage` = `{{variables.currentPage + 1}}` → Run `listExportHistory` |

### C6: `errorDetailModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `Export Detail` |
| Size | Medium |

**Children:**

| # | Component | ID | Properties |
|---|---|---|---|
| 1 | Text | `detailTemplate` | Content: `Template: {{variables.selectedExportRow.template_name}}` |
| 2 | Text | `detailUser` | Content: `User: {{variables.selectedExportRow.user_email}}` |
| 3 | Text | `detailTime` | Content: `Time: {{new Date(variables.selectedExportRow.created_at).toLocaleString()}}` |
| 4 | Text | `detailParams` | Content: `Parameters: {{variables.selectedExportRow.parameters \|\| 'none'}}`, font: mono |
| 5 | Text | `detailStatus` | Content: `Status: {{variables.selectedExportRow.status}}`, color: red if failed |
| 6 | Text | `detailError` | Content: `{{queries.getExecutionDetail.data?.error_message \|\| 'No error details'}}`, font: mono, color: red, background: muted, padding: 12px |
| 7 | Button | `btnRetryFromHistory` | Label: `Retry`, variant: primary, visible: `{{variables.selectedExportRow.status === 'failed'}}` |
| 8 | Button | `btnCloseDetail` | Label: `Close`, variant: outline |

| Button | Event | Action |
|---|---|---|
| `btnRetryFromHistory` | On click | Close modal → Switch page to `exports` (with template pre-selected if possible) |
| `btnCloseDetail` | On click | Close modal `errorDetailModal` |

## Variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `currentPage` | Number | `1` | Pagination offset |
| `selectedExecId` | String | `''` | For getExecutionDetail query |
| `selectedExportRow` | Object | `{}` | Full row data for detail modal |
