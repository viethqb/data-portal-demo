# Page 4: Templates

**Handle:** `templates` | **Permissions:** admin, da

Manages templates, sheet mappings, format config, and user permission assignments. This is the most complex page — uses a list/detail toggle pattern.

## Layout

```text
┌─ List View (showDetail = false) ────────────────────────────────────┐
│  [headerText]                                                       │
│  [searchInput]  [moduleFilter]  [btnCreateTemplate]                 │
│  [templateTable]                                                    │
├─ Detail View (showDetail = true) ───────────────────────────────────┤
│  [btnBackToList]  [detailTitle]                                     │
│  [tabGroup]                                                         │
│    ├─ Tab: Config    → [configForm]  [btnSaveConfig]                │
│    ├─ Tab: Files     → [fileList]  [fileUpload]  [fileDeleteBtn]   │
│    ├─ Tab: Mappings  → [mappingsList]  [mappingFormModal]           │
│    ├─ Tab: Format    → [formatEditor]  [btnSaveFormat]              │
│    ├─ Tab: Permissions → [permTable]  [addUserModal]                │
│    └─ Tab: Test      → [testParamsInput]  [btnTestGenerate]         │
└─────────────────────────────────────────────────────────────────────┘
```

## Page Events

| Event | Action |
|---|---|
| On page load | Run: `listAllTemplates`, `listModulesForFilter` |

## Variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `showDetail` | Boolean | `false` | Toggle list/detail view |
| `selectedModuleId` | String | `''` | Module of selected template |
| `selectedTemplateId` | String | `''` | Template being viewed/edited |
| `isEditingMapping` | Boolean | `false` | Create vs edit mapping modal |
| `editMappingData` | Object | `{}` | Pre-fill mapping form for edit |
| `testExportStatus` | String | `idle` | For test tab polling |
| `testExecutionId` | String | `''` | For test download |
| `testErrorMessage` | String | `''` | Test error display |
| `fileToDelete` | String | `''` | MinIO object name to delete |

---

## Part 1: List View Queries

### Q1: `listAllTemplates`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 100, "name__ilike": {{components.searchInput.value ? `"${components.searchInput.value}"` : 'null'}}, "module_id": {{components.moduleFilter.value ? `"${components.moduleFilter.value}"` : 'null'}}}` |
| Run on page load | Yes |

### Q2: `listModulesForFilter`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules` |
| Body | `{"page": 1, "page_size": 100}` |
| Run on page load | Yes |
| Transform | `return data.data.map(m => ({label: m.name, value: m.id}))` |

### Q3: `deleteTemplate`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules/{{variables.selectedModuleId}}/templates/delete?tid={{variables.selectedTemplateId}}` |
| Run on demand | Delete button |

| Event | Action |
|---|---|
| On success | Run `listAllTemplates` → Show alert "Template deleted" |

## Part 1: List View Components

### C1: `searchInput`

| Property | Value |
|---|---|
| Component | Text Input |
| Placeholder | `Search templates...` |
| Width | 300px |

| Event | Action |
|---|---|
| On change (debounce 500ms) | Run `listAllTemplates` |

### C2: `moduleFilter`

| Property | Value |
|---|---|
| Component | Dropdown |
| Options | `{{[{label: 'All', value: ''}].concat(queries.listModulesForFilter.data || [])}}` |
| Default | `''` |

| Event | Action |
|---|---|
| On select | Run `listAllTemplates` |

### C3: `btnCreateTemplate`

| Property | Value |
|---|---|
| Component | Button |
| Label | `+ Create Template` |
| Variant | Primary |

| Event | Action |
|---|---|
| On click | Show modal `createTemplateModal` |

### C4: `templateTable`

| Property | Value |
|---|---|
| Component | Table |
| Data | `{{queries.listAllTemplates.data.data}}` |
| Visible | `{{!variables.showDetail}}` |

**Columns:**

| # | Header | Key | Width | Config |
|---|---|---|---|---|
| 1 | Name | `name` | 200px | Bold text |
| 2 | Module | `report_module_id` | 150px | Map to name from `listModulesForFilter` data: `{{queries.listModulesForFilter.data?.find(m => m.value === cellValue)?.label \|\| cellValue}}` |
| 3 | Template File | `template_path` | 150px | Cell: `{{cellValue \|\| '(blank)'}}` |
| 4 | Output Sheet | `output_sheet` | 120px | Cell: `{{cellValue \|\| 'Full file'}}` |
| 5 | Active | `is_active` | 80px | Badge: true→green "Active", false→gray "Off" |
| 6 | Actions | — | 120px | Two buttons |

**Action button 1: Configure ⚙**

| Event | Action |
|---|---|
| On click | 1. Set `selectedTemplateId` = `{{rowData.id}}` |
| | 2. Set `selectedModuleId` = `{{rowData.report_module_id}}` |
| | 3. Run `getTemplateDetail` |
| | 4. Set `showDetail` = `true` |

**Action button 2: Delete 🗑**

| Event | Action |
|---|---|
| On click | 1. Set `selectedTemplateId` = `{{rowData.id}}` |
| | 2. Set `selectedModuleId` = `{{rowData.report_module_id}}` |
| | 3. Show confirm dialog → Run `deleteTemplate` |

---

## Part 2: Detail View Queries

### Q4: `getTemplateDetail`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/template-detail?tid={{variables.selectedTemplateId}}` |
| Run on demand | When entering detail view |

### Q5: `updateTemplate`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules/{{variables.selectedModuleId}}/templates/update` |
| Body | (see Config tab and Format tab below) |
| Run on demand | Save buttons |

| Event | Action |
|---|---|
| On success | Run `getTemplateDetail` → Show alert "Saved" |

### Q6: `createMapping`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules/{{variables.selectedModuleId}}/templates/{{variables.selectedTemplateId}}/mappings/create` |
| Body | `{{JSON.stringify(components.mappingForm.formData)}}` |
| Run on demand | Save button in mapping modal |

| Event | Action |
|---|---|
| On success | Close `mappingFormModal` → Run `getTemplateDetail` |

### Q7: `updateMapping`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules/{{variables.selectedModuleId}}/templates/{{variables.selectedTemplateId}}/mappings/update` |
| Body | `{{JSON.stringify({id: variables.editMappingData.id, ...components.mappingForm.formData})}}` |
| Run on demand | Save button in mapping modal (edit mode) |

| Event | Action |
|---|---|
| On success | Close `mappingFormModal` → Run `getTemplateDetail` |

### Q8: `deleteMapping`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/modules/{{variables.selectedModuleId}}/templates/{{variables.selectedTemplateId}}/mappings/delete?mapping_id={{parameters.mappingId}}` |
| Run on demand | Delete button on mapping card |

| Event | Action |
|---|---|
| On success | Run `getTemplateDetail` |

### Q9: `listBuckets`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/buckets/{{parameters.datasourceId}}` |
| Run on demand | When module detail loaded |

### Q10: `listFiles`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/files/{{parameters.datasourceId}}/{{parameters.bucket}}` |
| Run on demand | When bucket selected |

### Q11: `listSheets`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/sheets/{{parameters.datasourceId}}/{{parameters.bucket}}/{{parameters.filePath}}` |
| Run on demand | When file selected |

### Q12–Q14: Permissions queries

### Q12: `listTemplatePermissions`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | List rows |
| Filter | `template_id` equals `{{variables.selectedTemplateId}}` |
| Run on demand | When Permissions tab opened |

### Q13: `addPermission` (JavaScript — check duplicate then insert)

```javascript
// Check if already exists
const existing = queries.listTemplatePermissions.data?.filter(
  p => p.user_email === parameters.email
);
if (existing && existing.length > 0) {
  return { skipped: true };
}
await queries.insertPermission.run({ email: parameters.email });
await queries.listTemplatePermissions.run();
return { skipped: false };
```

### Q13b: `insertPermission`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | Create row |
| `user_email` | `{{parameters.email}}` |
| `template_id` | `{{variables.selectedTemplateId}}` |

### Q14: `removePermission`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | Delete row |
| Filter | `id` equals `{{parameters.permissionId}}` |

| Event | Action |
|---|---|
| On success | Run `listTemplatePermissions` |

### Q15–Q16: Test Generate

### Q15: `testGenerate`

Same as `generateReport` on page 02, but uses `variables.selectedTemplateId`.

### Q16: `pollTestExecution`

Same as `pollExecution` on page 02, but uses `testExportStatus` variable.

---

## Part 2: Detail View Components

### C5: `btnBackToList`

| Property | Value |
|---|---|
| Component | Button |
| Label | `← Back to List` |
| Variant | Ghost |
| Visible | `{{variables.showDetail}}` |

| Event | Action |
|---|---|
| On click | Set `showDetail` = `false` |

### C6: `detailTitle`

| Property | Value |
|---|---|
| Component | Text |
| Content | `Template: {{queries.getTemplateDetail.data.name}}` |
| Font size | 20px, bold |
| Visible | `{{variables.showDetail}}` |

### C7: `tabGroup`

| Property | Value |
|---|---|
| Component | Tabs |
| Tabs | `Config`, `Files`, `Mappings`, `Format`, `Permissions`, `Test` |
| Default tab | `Config` |
| Visible | `{{variables.showDetail}}` |

---

### Tab: Config

**Components inside Config tab:**

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Text Input | `cfgName` | Text Input | Label: `Name`, default: `{{queries.getTemplateDetail.data.name}}` |
| 2 | Textarea | `cfgDesc` | Textarea | Label: `Description`, default: `{{queries.getTemplateDetail.data.description}}`, rows: 2 |
| 3 | Dropdown | `cfgTemplateFile` | Dropdown | Label: `Template File`, options: from `listFiles` query, value: `{{queries.getTemplateDetail.data.template_path}}` |
| 4 | Text Input | `cfgOutputPrefix` | Text Input | Label: `Output Prefix`, default: `{{queries.getTemplateDetail.data.output_prefix}}` |
| 5 | Multiselect | `cfgOutputSheet` | Multiselect | Label: `Output Sheet`, options: from `listSheets` query, value: parsed from comma-separated string |
| 6 | Checkbox | `cfgRecalc` | Checkbox | Label: `Recalc (LibreOffice)`, default: `{{queries.getTemplateDetail.data.recalc_enabled}}` |
| 7 | Checkbox | `cfgActive` | Checkbox | Label: `Active`, default: `{{queries.getTemplateDetail.data.is_active}}` |
| 8 | Button | `btnSaveConfig` | Button | Label: `Save Config`, variant: primary |

**`btnSaveConfig` event:**

| Event | Action |
|---|---|
| On click | Run `updateTemplate` with body: |

```json
{
  "id": "{{variables.selectedTemplateId}}",
  "name": "{{components.cfgName.value}}",
  "description": "{{components.cfgDesc.value || null}}",
  "template_path": "{{components.cfgTemplateFile.value}}",
  "output_prefix": "{{components.cfgOutputPrefix.value}}",
  "output_sheet": "{{(components.cfgOutputSheet.value || []).join(',') || null}}",
  "recalc_enabled": {{components.cfgRecalc.value}},
  "is_active": {{components.cfgActive.value}}
}
```

---

### Tab: Files

Manage Excel template files in MinIO. Uses the **MinIO Data Source** (not pyDBAPI).

**Queries (MinIO Data Source):**

### `listTemplateFiles`

| Property | Value |
|---|---|
| Data source | `MinIO` |
| Operation | **List Objects in a Bucket** |
| Bucket | `report-templates` (or from module config) |
| Prefix | (optional, e.g., `templates/`) |
| Run on demand | When Files tab opened |

### `uploadTemplateFile`

| Property | Value |
|---|---|
| Data source | `MinIO` |
| Operation | **Put Object** |
| Bucket | `report-templates` |
| Object Name | `{{components.uploadPrefix.value}}{{components.filePicker1.file.name}}` |
| Upload data | `{{components.filePicker1.file.base64Data}}` |
| Content Type | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| Run on demand | Upload button click |

| Event | Action |
|---|---|
| On success | Run `listTemplateFiles` → Show alert "File uploaded" → `components.filePicker1.clearFiles()` |

### `deleteTemplateFile`

| Property | Value |
|---|---|
| Data source | `MinIO` |
| Operation | **Remove Object** |
| Bucket | `report-templates` |
| Object Name | `{{variables.fileToDelete}}` |
| Run on demand | Delete button click |

| Event | Action |
|---|---|
| On success | Run `listTemplateFiles` → Show alert "File deleted" |

### `readTemplateFile`

| Property | Value |
|---|---|
| Data source | `MinIO` |
| Operation | **Read Object** |
| Bucket | `report-templates` |
| Object Name | `{{parameters.fileName}}` |
| Run on demand | Download button click |

### `downloadTemplateFile` (JavaScript)

```javascript
const fileResp = await queries.readTemplateFile.run({ fileName: parameters.fileName });
const base64 = fileResp.data;
const byteChars = atob(base64);
const byteArray = new Uint8Array(byteChars.length);
for (let i = 0; i < byteChars.length; i++) byteArray[i] = byteChars.charCodeAt(i);
const blob = new Blob([byteArray], { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' });
const url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = parameters.fileName.split('/').pop();
a.click();
URL.revokeObjectURL(url);
```

**Components inside Files tab:**

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Text Input | `uploadPrefix` | Text Input | Label: `Folder prefix`, placeholder: `templates/`, default: `''` |
| 2 | **File Picker** | `filePicker1` | File Picker | Label: `Select .xlsx file`, accept: `.xlsx,.xls`, max size: 10MB, use drop zone: Yes |
| 3 | Button | `btnUpload` | Button | Label: `Upload`, variant: primary, disabled: `{{!components.filePicker1.file}}`, loading: `{{queries.uploadTemplateFile.isLoading}}` |
| 4 | Table | `fileTable` | Table | Data: `{{queries.listTemplateFiles.data}}`, loading: `{{queries.listTemplateFiles.isLoading}}` |

**File Picker config:**

| Property | Value |
|---|---|
| Accept file types | `.xlsx, .xls` |
| Max file size | `10485760` (10MB) |
| Max file count | 1 |
| Enable parsing | No |
| Use drop zone | Yes |
| Use file picker | Yes |

**`btnUpload` event:**

| Event | Action |
|---|---|
| On click | Run `uploadTemplateFile` |

**`fileTable` columns:**

| # | Header | Key | Type | Width | Config |
|---|---|---|---|---|---|
| 1 | File Name | `name` | Default | 300px | — |
| 2 | Size | `size` | Default | 100px | Cell: `{{Math.round(cellValue / 1024)}} KB` |
| 3 | Modified | `lastModified` | Default | 150px | Cell: `{{new Date(cellValue).toLocaleString()}}` |
| 4 | Actions | — | Action buttons | 150px | Download + Delete |

**Action button: Download**

| Property | Value |
|---|---|
| Label | `⬇ Download` |

| Event | Action |
|---|---|
| On click | Run `downloadTemplateFile` with `fileName: {{rowData.name}}` |

**Action button: Delete**

| Property | Value |
|---|---|
| Label | `🗑 Delete` |

| Event | Action |
|---|---|
| On click | 1. Set variable `fileToDelete` = `{{rowData.name}}` |
| | 2. Show confirm → Run `deleteTemplateFile` |

**Tab on enter event:** Run `listTemplateFiles`

---

### Tab: Mappings

**Components inside Mappings tab:**

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Button | `btnAddMapping` | Button | Label: `+ Add Mapping`, variant: primary |
| 2 | ListView | `mappingsList` | ListView | Data: `{{queries.getTemplateDetail.data.sheet_mappings}}`, row height: auto |

**`btnAddMapping` event:**

| Event | Action |
|---|---|
| On click | 1. Set `isEditingMapping` = `false` |
| | 2. Set `editMappingData` = `{}` |
| | 3. Show modal `mappingFormModal` |

**ListView children per row:**

| # | Component | Content |
|---|---|---|
| 1 | Text (bold) | `{{listItem.sheet_name}} : {{listItem.start_cell}}` |
| 2 | Text (badges) | `{{listItem.write_mode}}` · `Order: {{listItem.sort_order}}` · `Gap: {{listItem.gap_rows}}` · `{{listItem.write_headers ? 'Headers' : ''}}` |
| 3 | Text (code, muted) | `{{listItem.sql_content}}` (max 3 lines, overflow hidden) |
| 4 | Text (small) | Format summary: `{{listItem.format_config ? JSON.stringify(listItem.format_config) : 'No format'}}` |
| 5 | Button | `Edit` → Set `isEditingMapping=true`, `editMappingData={{listItem}}` → Show `mappingFormModal` |
| 6 | Button | `Delete` → Run `deleteMapping` with `mappingId: {{listItem.id}}` |

### `mappingFormModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `{{variables.isEditingMapping ? 'Edit' : 'Add'}} Sheet Mapping` |
| Size | Large |

**Form `mappingForm` inside modal:**

| # | Component | ID | Type | Default (create) | Default (edit) |
|---|---|---|---|---|---|
| 1 | Text Input | `mfSheetName` | Text Input | `Sheet1` | `{{variables.editMappingData.sheet_name}}` |
| 2 | Text Input | `mfStartCell` | Text Input | `A1` | `{{variables.editMappingData.start_cell}}` |
| 3 | Dropdown | `mfWriteMode` | Dropdown | `rows` | `{{variables.editMappingData.write_mode}}` |
| | | | | Options: `[{label:'Rows',value:'rows'},{label:'Single',value:'single'}]` | |
| 4 | Number Input | `mfSortOrder` | Number | `0` | `{{variables.editMappingData.sort_order}}` |
| 5 | Number Input | `mfGapRows` | Number | `0` | `{{variables.editMappingData.gap_rows}}` |
| 6 | Checkbox | `mfWriteHeaders` | Checkbox | `true` | `{{variables.editMappingData.write_headers}}` |
| 7 | **Code Editor** | `mfSqlContent` | Code Editor | `SELECT 1` | `{{variables.editMappingData.sql_content}}` |
| | | | | Mode: **SQL**, showLineNumber: Yes | |
| 8 | **Code Editor** | `mfFormatConfig` | Code Editor | `{}` | `{{JSON.stringify(variables.editMappingData.format_config \|\| {}, null, 2)}}` |
| | | | | Mode: **JSON**, showLineNumber: Yes | |
| 9 | Text Input | `mfDescription` | Text Input | `''` | `{{variables.editMappingData.description}}` |
| 10 | Button | `btnSaveMapping` | Button | Label: `Save`, variant: primary | |
| 11 | Button | `btnCancelMapping` | Button | Label: `Cancel`, variant: outline | |

**`btnSaveMapping` event:**

| Event | Action |
|---|---|
| On click | If `isEditingMapping`: run `updateMapping`. Else: run `createMapping` |

**Body assembled from form:**

```javascript
{
  "sheet_name": components.mfSheetName.value,
  "start_cell": components.mfStartCell.value,
  "write_mode": components.mfWriteMode.value,
  "sort_order": components.mfSortOrder.value,
  "gap_rows": components.mfGapRows.value,
  "write_headers": components.mfWriteHeaders.value,
  "sql_content": components.mfSqlContent.value,
  "format_config": JSON.parse(components.mfFormatConfig.value || '{}'),
  "description": components.mfDescription.value || null
}
```

---

### Tab: Format

Template-level default format config (inherited by all mappings).

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | **Code Editor** | `fmtEditor` | Code Editor | Mode: **JSON**, showLineNumber: Yes, default: `{{JSON.stringify(queries.getTemplateDetail.data.format_config \|\| {}, null, 2)}}` |
| 2 | Text | `fmtReference` | Text | Static reference text (see below), font: 12px, muted |
| 3 | Button | `btnSaveFormat` | Button | Label: `Save Format`, variant: primary |

**Reference text:**

```text
Format Config Reference:
{
  "auto_fit": true,               // Auto column widths
  "auto_fit_max_width": 50,       // Max width cap
  "wrap_text": true,              // Wrap text all cells
  "header": {                     // Header row style
    "font": {"name":"Calibri","size":11,"bold":true,"color":"FFFFFF"},
    "fill": {"bg_color":"002060"},
    "border": {"style":"thin","color":"000000"},
    "alignment": {"horizontal":"center"},
    "number_format": "#,##0"
  },
  "data": { ... },                // Data rows style (same structure)
  "column_widths": {"A":15,"B":25}// Manual column widths
}
```

**`btnSaveFormat` event:**

| Event | Action |
|---|---|
| On click | Run `updateTemplate` with body: `{"id": "{{selectedTemplateId}}", "format_config": {{components.fmtEditor.value}}}` |

---

### Tab: Permissions

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Button | `btnAddUser` | Button | Label: `+ Add User`, variant: primary |
| 2 | Table | `permTable` | Table | Data: `{{queries.listTemplatePermissions.data}}` |

**Tab on enter event:** Run `listTemplatePermissions`

**`permTable` columns:**

| # | Header | Key | Width | Config |
|---|---|---|---|---|
| 1 | Email | `user_email` | 250px | — |
| 2 | Added | `created_at` | 150px | Cell: `{{new Date(cellValue).toLocaleDateString()}}` |
| 3 | Actions | — | 100px | Button: `Remove` |

**Remove button event:** Run `removePermission` with `permissionId: {{rowData.id}}`

**`btnAddUser` event:** Show modal `addUserModal`

### `addUserModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `Add User Permission` |
| Size | Medium |

| # | Component | ID | Properties |
|---|---|---|---|
| 1 | Text Input | `addUserEmail` | Label: `User Email`, placeholder: `user@company.com` |
| 2 | Button | `btnConfirmAddUser` | Label: `Add`, variant: primary |
| 3 | Button | `btnCancelAddUser` | Label: `Cancel`, variant: outline |

**`btnConfirmAddUser` event:**

| Event | Action |
|---|---|
| On click | Run `addPermission` with `email: {{components.addUserEmail.value}}` → Close modal |

---

### Tab: Test

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | **Code Editor** | `testParams` | Code Editor | Mode: **JSON**, showLineNumber: Yes, default: `{}` |
| 2 | Button | `btnTestGenerate` | Button | Label: `Generate Test Report`, variant: primary, loading: `{{queries.testGenerate.isLoading}}` |
| 3 | Spinner | `testSpinner` | Spinner | Visible: `{{variables.testExportStatus === 'running'}}` |
| 4 | Text | `testResult` | Text | Visible: `{{variables.testExportStatus !== 'idle'}}` |
| 5 | Button | `btnTestDownload` | Button | Label: `Download`, visible: `{{variables.testExportStatus === 'success'}}` |

**`btnTestGenerate` body:**

```json
{
  "parameters": {{components.testParams.value || '{}'}}
}
```

**`btnTestDownload` event:** Run `downloadExecution` with `executionId: {{variables.testExecutionId}}`

---

## Event Flow Summary

```text
List View:
1. Page load → listAllTemplates + listModulesForFilter
2. Search/filter → re-run listAllTemplates
3. Click [⚙] → getTemplateDetail → showDetail=true

Detail View:
4. Config tab: edit fields → btnSaveConfig → updateTemplate
5. Files tab (MinIO Data Source):
   a. listTemplateFiles on tab enter → fileTable shows files
   b. filePicker1 select → btnUpload → uploadTemplateFile → refresh list
   c. [Delete] → confirm → deleteTemplateFile → refresh list
   d. [Download] → readTemplateFile (MinIO) → JS blob → browser download
6. Mappings tab:
   a. [+ Add Mapping] → mappingFormModal (create) → createMapping
   b. [Edit] on card → mappingFormModal (edit) → updateMapping
   c. [Delete] → deleteMapping
7. Format tab: edit JSON → btnSaveFormat → updateTemplate
8. Permissions tab:
   a. listTemplatePermissions on tab enter
   b. [+ Add User] → addUserModal → addPermission
   c. [Remove] → removePermission
9. Test tab: enter params → testGenerate → poll → download
10. [← Back] → showDetail=false
```
