# Page 1: Dashboard

**Handle:** `dashboard` | **Home page:** Yes | **Permissions:** All users

## Layout

```text
┌─────────────────────────────────────────────────────────────────────┐
│  [headerText]                                                       │
├─ Row 1: Stat Cards ─────────────────────────────────────────────────┤
│  [cardModules]  [cardTemplates]  [cardExports]  [cardSchedulers]   │
├─ Row 2: Recent Exports ─────────────────────────────────────────────┤
│  [sectionTitle] .................. [btnViewAll]                      │
│  [recentTable]                                                      │
├─ Row 3: Quick Export ───────────────────────────────────────────────┤
│  [quickExportList]                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Page Events

| Event | Action |
|---|---|
| On page load | Run queries: `countModules`, `countTemplates`, `recentExports`, `quickExportTemplates`, `getUserRole` |

## Queries

### Q1: `countModules`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules` |
| Body | `{"page": 1, "page_size": 1}` |
| Run on page load | Yes |
| Transform | — |

### Q2: `countTemplates`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 1}` |
| Run on page load | Yes |

### Q3a: `recentExportsAll` (for admin/da)

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `export_log` |
| Operation | List rows |
| Sort by | `created_at` DESC |
| Limit | 10 |
| Filter | (none) |

### Q3b: `recentExportsMine` (for end_user)

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `export_log` |
| Operation | List rows |
| Sort by | `created_at` DESC |
| Limit | 10 |
| Filter | `user_email` equals `{{globals.currentUser.email}}` |

### Q3: `recentExports` (JavaScript — picks the right query)

```javascript
const role = queries.getUserRole.data;
if (role === 'end_user') {
  await queries.recentExportsMine.run();
  return queries.recentExportsMine.data;
} else {
  await queries.recentExportsAll.run();
  return queries.recentExportsAll.data;
}
```

| Property | Value |
|---|---|
| Run on page load | Yes (after `getUserRole` completes) |

### Q4: `quickExportTemplates`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 6, "is_active": true}` |
| Run on page load | Yes |

### Q5: `getUserRole` (shared — defined in 00-setup.md)

| Property | Value |
|---|---|
| Data source | JavaScript |
| Run on app load | Yes |

```javascript
const groups = (globals.currentUser.groups || []).map(g => g.name);
if (groups.includes('admin')) return 'admin';
if (groups.includes('da')) return 'da';
return 'end_user';
```

## Components

### C1: `headerText` — Page Title

| Property | Value |
|---|---|
| Component | Text |
| Content | `Dashboard` |
| Font size | 24px, bold |
| Margin bottom | 16px |

### C2–C5: Stat Cards

Use the **Statistics** component (built-in ToolJet component with primary/secondary values).

#### C2: `statModules`

| Property | Value |
|---|---|
| Component | **Statistics** |
| Primary value label | `Modules` |
| Primary value | `{{queries.countModules.data.total \|\| 0}}` |
| Hide secondary value | Yes |
| Loading state | `{{queries.countModules.isLoading}}` |

#### C3: `statTemplates`

| Property | Value |
|---|---|
| Component | **Statistics** |
| Primary value label | `Templates` |
| Primary value | `{{queries.countTemplates.data.total \|\| 0}}` |
| Hide secondary value | Yes |

#### C4: `statExports`

| Property | Value |
|---|---|
| Component | **Statistics** |
| Primary value label | `Recent Exports` |
| Primary value | `{{queries.recentExports.data.length \|\| 0}}` |
| Hide secondary value | Yes |

#### C5: `statSchedulers`

| Property | Value |
|---|---|
| Component | **Statistics** |
| Primary value label | `Schedulers` |
| Primary value | `0` |
| Hide secondary value | Yes |

### C6: `sectionTitle` — Recent Exports Header

| Property | Value |
|---|---|
| Component | Text |
| Content | `Recent Exports` |
| Font size | 18px, bold |

### C7: `btnViewAll`

| Property | Value |
|---|---|
| Component | Button |
| Label | `View All →` |
| Variant | Link / Ghost |
| Position | Right-aligned, same row as sectionTitle |

| Event | Action |
|---|---|
| On click | Switch page → `history` |

### C8: `recentTable`

| Property | Value |
|---|---|
| Component | Table |
| Data | `{{queries.recentExports.data}}` |
| Show pagination | No |
| Show search | No |
| Row height | Compact |

**Columns:**

| # | Header | Key | Type | Width | Config |
|---|---|---|---|---|---|
| 1 | Template | `template_name` | Default (text) | 200px | — |
| 2 | User | `user_name` | Default (text) | 150px | Visibility: `{{queries.getUserRole.data !== 'end_user'}}` |
| 3 | Status | `status` | **Select** | 100px | Options: `[{label:'success', value:'success', backgroundColor:'#10b981'}, {label:'failed', value:'failed', backgroundColor:'#ef4444'}, {label:'pending', value:'pending', backgroundColor:'#f59e0b'}]` |
| 4 | Parameters | `parameters` | Default (text) | 200px | Cell value: `{{cellValue \|\| '--'}}`, truncate |
| 5 | Time | `created_at` | Default (text) | 150px | Cell value: `{{new Date(cellValue).toLocaleString()}}` |
| 6 | Actions | — | Action button | 80px | See below |

**Action button config:**

| Property | Value |
|---|---|
| Label | `⬇` |
| Visible | `{{rowData.status === 'success'}}` |

| Event | Action |
|---|---|
| On click | Run `downloadExecution` with parameter `executionId: {{rowData.execution_id}}` |

### C9: `quickExportList`

| Property | Value |
|---|---|
| Component | ListView |
| Data | `{{queries.quickExportTemplates.data.data}}` |
| Row height | 140 |
| Columns | 3 |
| Show border | Yes |

**Children per row:**

| Component | ID | Content | Style |
|---|---|---|---|
| Text | `qeTitle` | `{{listItem.name}}` | Font 16px, bold |
| Text | `qeDesc` | `{{listItem.description \|\| ''}}` | Font 12px, muted, max 2 lines |
| Button | `qeBtn` | `Export Now` | Full width, primary color |

**Button event:**

| Event | Action |
|---|---|
| On click | 1. Set variable `selectedTemplate` = `{{listItem}}` |
| | 2. Switch page → `exports` |

## Variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `selectedTemplate` | Object | `{}` | Passed to My Exports page |
