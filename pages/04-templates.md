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
| On page load | Run: `initFormatConstants`, `listModulesForFilter`, `buildListTemplatesBody` → `listAllTemplates` |

## Variables

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `showDetail` | Boolean | `false` | Toggle list/detail view |
| `selectedModuleId` | String | `''` | Module of selected template |
| `selectedTemplateId` | String | `''` | Template being viewed/edited |
| `isEditingMapping` | Boolean | `false` | Create vs edit mapping modal |
| `editMappingData` | Object | `{}` | Pre-fill mapping form for edit |
| `formatConfig` | Object | `{}` | Template-level format editor state (Tab: Format) |
| `mappingFormatConfig` | Object | `{}` | Mapping-level format override state (mapping modal) |
| `mappingUseCustomFormat` | Boolean | `false` | Whether mapping overrides template format |
| `testParamRows` | Array | `[]` | Test tab dynamic parameter rows `{id, key, type, value}` |
| `resolvedTestParams` | Object | `{}` | Built test payload, set by `tpBuildPayload` before `testGenerate` |
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
| Body | `{{queries.buildListTemplatesBody.data}}` (see Q1a) |
| Run on page load | Yes (after `buildListTemplatesBody` resolves) |

### Q1a: `buildListTemplatesBody` (JavaScript)

Builds the request body as a real object so user-typed quotes / special chars in the search input cannot break JSON. Triggered before `listAllTemplates` on every search/filter change.

```javascript
const body = { page: 1, page_size: 100 };
const search = (components.searchInput?.value || '').trim();
if (search) body.name__ilike = search;
const moduleId = components.moduleFilter?.value || '';
if (moduleId) body.module_id = moduleId;
return body;
```

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
| On change (debounce 500ms) | Run `buildListTemplatesBody` → Run `listAllTemplates` |

### C2: `moduleFilter`

| Property | Value |
|---|---|
| Component | Dropdown |
| Options | `{{[{label: 'All', value: ''}].concat(queries.listModulesForFilter.data || [])}}` |
| Default | `''` |

| Event | Action |
|---|---|
| On select | Run `buildListTemplatesBody` → Run `listAllTemplates` |

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
| Body | `{{queries.buildMappingPayload.data}}` (run `buildMappingPayload` first, see below) |
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
| Body | `{{queries.buildMappingPayload.data}}` (adds `id` when `isEditingMapping`, see below) |
| Run on demand | Save button in mapping modal (edit mode) |

| Event | Action |
|---|---|
| On success | Close `mappingFormModal` → Run `getTemplateDetail` |

### Q6a: `buildMappingPayload` (JavaScript)

Single source of truth for both create and update bodies. Called from `btnSaveMapping` before running `createMapping` / `updateMapping`.

```javascript
const body = {
  sheet_name: components.mfSheetName.value,
  start_cell: components.mfStartCell.value,
  write_mode: components.mfWriteMode.value,
  sort_order: components.mfSortOrder.value,
  gap_rows: components.mfGapRows.value,
  write_headers: components.mfWriteHeaders.value,
  sql_content: components.mfSqlContent.value,
  format_config: variables.mappingUseCustomFormat ? variables.mappingFormatConfig : null,
  description: components.mfDescription.value || null,
};
if (variables.isEditingMapping) body.id = variables.editMappingData.id;
return body;
```

**`btnSaveMapping` event chain:**

1. Run `buildMappingPayload`
2. If `isEditingMapping` → run `updateMapping`, else → run `createMapping`

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

**Modal on open event:**

| Event | Action |
|---|---|
| On show | 1. Set `mappingFormatConfig` = `{{variables.editMappingData.format_config || {}}}` |
| | 2. Set `mappingUseCustomFormat` = `{{!!(variables.editMappingData.format_config && Object.keys(variables.editMappingData.format_config).length > 0)}}` |

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
| 8 | Toggle | `mfUseCustomFormat` | Toggle | Label: `Override template format`, value: `{{variables.mappingUseCustomFormat}}` | |
| 9 | Container | `mfFormatContainer` | Container | Visible: `{{variables.mappingUseCustomFormat}}` — contains the same visual format editor as Tab: Format, but bound to `variables.mappingFormatConfig` and using prefix `mmFmt*` for component IDs | |
| 10 | Text | `mfFormatDefaultNote` | Text | Content: `Using template default format_config. Turn on "Override template format" to customize.`, visible: `{{!variables.mappingUseCustomFormat}}`, muted | |
| 11 | Text Input | `mfDescription` | Text Input | `''` | `{{variables.editMappingData.description}}` |
| 12 | Button | `btnSaveMapping` | Button | Label: `Save`, variant: primary | |
| 13 | Button | `btnCancelMapping` | Button | Label: `Cancel`, variant: outline | |

**`mfUseCustomFormat` on change:**

| Event | Action |
|---|---|
| On change | Set `mappingUseCustomFormat` = `{{components.mfUseCustomFormat.value}}`. If toggled off → set `mappingFormatConfig` = `{}`. |

**Reused editor inside `mfFormatContainer`:** clone the General / Header Format / Data Format / Column Widths sections defined in Tab: Format. Bind each component to `variables.mappingFormatConfig` instead of `variables.formatConfig`, and call `fmtPatch` with `target: 'mappingFormatConfig'`. Use component-ID prefix `mmFmt*` (e.g., `mmFmtHeaderFontName`) to avoid collision with the template-level editor on the same page.

**`btnSaveMapping` event:**

| Event | Action |
|---|---|
| On click | If `isEditingMapping`: run `updateMapping`. Else: run `createMapping` |

**Body assembled from form (no JSON.parse — `mappingFormatConfig` is already an object):**

```javascript
{
  "sheet_name": components.mfSheetName.value,
  "start_cell": components.mfStartCell.value,
  "write_mode": components.mfWriteMode.value,
  "sort_order": components.mfSortOrder.value,
  "gap_rows": components.mfGapRows.value,
  "write_headers": components.mfWriteHeaders.value,
  "sql_content": components.mfSqlContent.value,
  "format_config": variables.mappingUseCustomFormat ? variables.mappingFormatConfig : null,
  "description": components.mfDescription.value || null
}
```

---

### Tab: Format

Template-level default format config (inherited by all mappings). Uses a **visual form editor** matching the pyDBAPI Dashboard (`FormatConfigEditor.tsx`) — no raw JSON. State is kept in `variables.formatConfig` (Object) and sent directly to the API.

**Tab on enter event:**

| Event | Action |
|---|---|
| On tab enter | Set variable `formatConfig` = `{{queries.getTemplateDetail.data.format_config || {}}}` |

**Layout (inside a Container `fmtContainer`):**

```text
┌─ General ────────────────────────────────────────┐
│ [x] Auto-fit columns     Max width: [ 50 ]      │
│ [x] Wrap text (global)                           │
├─ Header Format ──────────────────────────────────┤
│ Font: [Calibri ▼]  Size: [11 ▼]  [B] [I]         │
│ Text color:   [ Color ▼] [FFFFFF]                │
│ Background:   [ Color ▼] [002060]                │
│ Border:       [Thin ▼]   [ Color ▼] [000000]     │
│ H-Align: [←][↔][→][⇔]  V-Align: [Top ▼] [x] Wrap │
│ Number format: [General ▼]  or  [ custom...... ] │
├─ Data Format ────────────────────────────────────┤
│ (same structure as Header)                       │
├─ Column Widths ──────────────────────────────────┤
│ [A] [15.0] [🗑]                                   │
│ [B] [25.0] [🗑]                                   │
│ [+ Add column width]                             │
└──────────────────────────────────────────────────┘
[Clear All]               [Save Format]
```

#### General section

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Checkbox | `fmtAutoFit` | Checkbox | Label: `Auto-fit columns`, value: `{{variables.formatConfig.auto_fit ?? true}}` |
| 2 | Number Input | `fmtAutoFitMax` | Number | Label: `Max width`, value: `{{variables.formatConfig.auto_fit_max_width ?? 50}}`, min: 1, step: 1, disabled: `{{!components.fmtAutoFit.value}}` |
| 3 | Checkbox | `fmtWrapTextGlobal` | Checkbox | Label: `Wrap text (global)`, value: `{{variables.formatConfig.wrap_text ?? false}}` |

On change for each → patch `variables.formatConfig` via small JS (see "Binding helper" below).

#### Header Format section (inside collapsible container `fmtHeaderCollapse`)

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Dropdown | `fmtHeaderFontName` | Dropdown | Label: `Font`, options: `{{queries.initFormatConstants.data.FONT_NAMES}}`, value: `{{variables.formatConfig.header?.font?.name || 'Calibri'}}` |
| 2 | Dropdown | `fmtHeaderFontSize` | Dropdown | Label: `Size`, options: `{{queries.initFormatConstants.data.FONT_SIZES}}`, value: `{{variables.formatConfig.header?.font?.size || 11}}` |
| 3 | Toggle | `fmtHeaderBold` | Toggle | Icon: `Bold`, value: `{{variables.formatConfig.header?.font?.bold ?? false}}` |
| 4 | Toggle | `fmtHeaderItalic` | Toggle | Icon: `Italic`, value: `{{variables.formatConfig.header?.font?.italic ?? false}}` |
| 5 | Dropdown | `fmtHeaderFontColorPreset` | Dropdown | Label: `Text color`, options: `{{queries.initFormatConstants.data.PRESET_COLORS}}`, value: `{{variables.formatConfig.header?.font?.color}}` |
| 6 | Text Input | `fmtHeaderFontColorHex` | Text Input | Placeholder: `Hex`, max length: 6, width: 80px, value: `{{variables.formatConfig.header?.font?.color || ''}}` |
| 7 | Dropdown | `fmtHeaderBgPreset` | Dropdown | Label: `Background`, options: `{{queries.initFormatConstants.data.PRESET_COLORS}}`, value: `{{variables.formatConfig.header?.fill?.bg_color}}` |
| 8 | Text Input | `fmtHeaderBgHex` | Text Input | Placeholder: `Hex`, max length: 6, width: 80px |
| 9 | Dropdown | `fmtHeaderBorderStyle` | Dropdown | Label: `Border`, options: `{{queries.initFormatConstants.data.BORDER_STYLES}}`, value: `{{variables.formatConfig.header?.border?.style || ''}}` |
| 10 | Dropdown | `fmtHeaderBorderColorPreset` | Dropdown | Label: `Border color`, options: `{{queries.initFormatConstants.data.PRESET_COLORS}}`, value: `{{variables.formatConfig.header?.border?.color}}` |
| 11 | Text Input | `fmtHeaderBorderColorHex` | Text Input | Placeholder: `Hex`, max length: 6, width: 80px |
| 12 | Dropdown | `fmtHeaderHAlign` | Dropdown | Label: `H-Align`, options: `[{label:'Default',value:''},{label:'Left',value:'left'},{label:'Center',value:'center'},{label:'Right',value:'right'},{label:'Justify',value:'justify'}]`, value: `{{variables.formatConfig.header?.alignment?.horizontal || ''}}` |
| 13 | Dropdown | `fmtHeaderVAlign` | Dropdown | Label: `V-Align`, options: `[{label:'Default',value:''},{label:'Top',value:'top'},{label:'Center',value:'center'},{label:'Bottom',value:'bottom'}]`, value: `{{variables.formatConfig.header?.alignment?.vertical || ''}}` |
| 14 | Checkbox | `fmtHeaderWrapText` | Checkbox | Label: `Wrap text`, value: `{{variables.formatConfig.header?.alignment?.wrap_text ?? false}}` |
| 15 | Dropdown | `fmtHeaderNumFormatPreset` | Dropdown | Label: `Number format`, options: `{{queries.initFormatConstants.data.NUMBER_FORMATS}}`, value: `{{variables.formatConfig.header?.number_format || 'General'}}` |
| 16 | Text Input | `fmtHeaderNumFormatCustom` | Text Input | Placeholder: `Custom e.g. #,##0.00_);[Red](#,##0.00)`, width: 300px |

#### Data Format section

Identical structure to Header — use prefix `fmtData*` instead of `fmtHeader*`. Bind to `variables.formatConfig.data.*`.

#### Column Widths section (inside collapsible container `fmtColWidthsCollapse`)

Use a ListView bound to `variables.formatConfig.column_widths` converted to array:

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | ListView | `fmtColWidthsList` | ListView | Data: `{{Object.entries(variables.formatConfig.column_widths || {}).map(([col, w]) => ({col, width: w}))}}`, row height: auto |
| 2 | Button | `fmtAddColWidth` | Button | Label: `+ Add column width`, variant: outline |

**Row children (per column width):**

| # | Component | Properties |
|---|---|---|
| Text Input | Value: `{{listItem.col}}`, width: 60px, max length: 3, uppercase. On blur → JS `fmtUpdateColWidth` with `{oldKey: listItem.col, newKey: value, width: listItem.width}` |
| Number Input | Value: `{{listItem.width}}`, width: 100px, step: 0.5. On change → JS `fmtUpdateColWidth` with `{oldKey: listItem.col, newKey: listItem.col, width: value}` |
| Button | Label: `🗑`, variant: ghost. On click → JS `fmtDeleteColWidth` with `{key: listItem.col}` |

#### Bottom buttons

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Button | `btnClearFormat` | Button | Label: `Clear All`, variant: outline |
| 2 | Button | `btnSaveFormat` | Button | Label: `Save Format`, variant: primary |

#### Constants (define once on page load)

Create a JS query `initFormatConstants` run on page load:

```javascript
return {
  FONT_NAMES: ['Arial','Calibri','Cambria','Consolas','Courier New','Georgia',
               'Helvetica','Segoe UI','Tahoma','Times New Roman','Trebuchet MS','Verdana']
    .map(n => ({label: n, value: n})),
  FONT_SIZES: [8,9,10,11,12,14,16,18,20,24,28,36].map(s => ({label: String(s), value: s})),
  PRESET_COLORS: [
    {label:'None', value:''},
    {label:'Black', value:'000000'}, {label:'White', value:'FFFFFF'},
    {label:'Red', value:'FF0000'}, {label:'Dark Red', value:'C00000'},
    {label:'Orange', value:'FF6600'}, {label:'Yellow', value:'FFFF00'},
    {label:'Light Yellow', value:'FFFFCC'},
    {label:'Green', value:'00B050'}, {label:'Dark Green', value:'006100'},
    {label:'Light Green', value:'C6EFCE'},
    {label:'Blue', value:'0070C0'}, {label:'Dark Blue', value:'002060'},
    {label:'Light Blue', value:'DDEBF7'},
    {label:'Purple', value:'7030A0'},
    {label:'Gray 25%', value:'D9D9D9'}, {label:'Gray 50%', value:'808080'},
    {label:'Gray 80%', value:'333333'},
  ],
  BORDER_STYLES: [
    {label:'None', value:''}, {label:'Thin', value:'thin'},
    {label:'Medium', value:'medium'}, {label:'Thick', value:'thick'},
    {label:'Dashed', value:'dashed'}, {label:'Dotted', value:'dotted'},
    {label:'Double', value:'double'},
  ],
  NUMBER_FORMATS: [
    {label:'General', value:'General'},
    {label:'#,##0', value:'#,##0'}, {label:'#,##0.00', value:'#,##0.00'},
    {label:'0%', value:'0%'}, {label:'0.00%', value:'0.00%'},
    {label:'yyyy-mm-dd', value:'yyyy-mm-dd'},
    {label:'dd/mm/yyyy', value:'dd/mm/yyyy'},
    {label:'yyyy-mm-dd hh:mm', value:'yyyy-mm-dd hh:mm:ss'},
    {label:'@ (Text)', value:'@'},
  ],
};
```

Reference these fields directly as `{{queries.initFormatConstants.data.FONT_NAMES}}` etc. — no need for separate page variables. Run this query once on page load; it has no side effects so it can stay cached for the session.

#### Binding helper: `fmtPatch` (JavaScript query)

Every on-change handler calls this with a dot-path and a value. Avoids writing 30+ inline handlers.

```javascript
// parameters: { target: 'formatConfig' | 'mappingFormatConfig', path: 'header.font.bold', value: true }
const targetVar = parameters.target || 'formatConfig';
const path = parameters.path;
const value = parameters.value;

const current = JSON.parse(JSON.stringify(
  targetVar === 'mappingFormatConfig' ? variables.mappingFormatConfig : variables.formatConfig
));

const keys = path.split('.');
let obj = current;
for (let i = 0; i < keys.length - 1; i++) {
  const k = keys[i];
  if (obj[k] == null || typeof obj[k] !== 'object') obj[k] = {};
  obj = obj[k];
}
const lastKey = keys[keys.length - 1];

// Empty string / null / undefined → delete the key to keep config clean
if (value === '' || value == null) {
  delete obj[lastKey];
} else {
  obj[lastKey] = value;
}

// Prune empty nested objects ({}) recursively so the payload stays minimal
const prune = (o) => {
  Object.keys(o).forEach(k => {
    if (o[k] && typeof o[k] === 'object' && !Array.isArray(o[k])) {
      prune(o[k]);
      if (Object.keys(o[k]).length === 0) delete o[k];
    }
  });
};
prune(current);

await actions.setVariable(targetVar, current);
```

Example bindings (on change):

| Component | Action |
|---|---|
| `fmtAutoFit` | Run `fmtPatch` with `target:'formatConfig', path:'auto_fit', value: components.fmtAutoFit.value` |
| `fmtHeaderFontName` | Run `fmtPatch` with `path:'header.font.name', value: components.fmtHeaderFontName.value` |
| `fmtHeaderBold` | Run `fmtPatch` with `path:'header.font.bold', value: components.fmtHeaderBold.value` |
| `fmtHeaderBgPreset` | Run `fmtPatch` with `path:'header.fill.bg_color', value: components.fmtHeaderBgPreset.value` then also sync `fmtHeaderBgHex.value` |
| `fmtHeaderBgHex` | Run `fmtPatch` with `path:'header.fill.bg_color', value: components.fmtHeaderBgHex.value.toUpperCase()` |
| `fmtHeaderNumFormatPreset` | Run `fmtPatch` with `path:'header.number_format', value: components.fmtHeaderNumFormatPreset.value` |
| `fmtHeaderNumFormatCustom` | Run `fmtPatch` with `path:'header.number_format', value: components.fmtHeaderNumFormatCustom.value` (only fires when non-empty) |

Preset ↔ hex mirroring: when the preset dropdown changes, also `setComponentValue(fmtHeaderBgHex, value)` and vice-versa — a tiny JS wrapper can do both in one call.

#### Column widths helpers

**`fmtAddColWidth` on click** → JS:

```javascript
const current = {...(variables.formatConfig.column_widths || {})};
// Pick next unused uppercase letter A-Z, fallback to 'NEW'
const used = new Set(Object.keys(current));
let next = 'NEW';
for (let c = 65; c <= 90; c++) {
  const letter = String.fromCharCode(c);
  if (!used.has(letter)) { next = letter; break; }
}
current[next] = 15;
await actions.setVariable('formatConfig', {...variables.formatConfig, column_widths: current});
```

**`fmtUpdateColWidth` (JS query, parameters: `oldKey`, `newKey`, `width`)**:

```javascript
const current = {...(variables.formatConfig.column_widths || {})};
const key = (parameters.newKey || '').toUpperCase().trim();
if (!key) return;
if (parameters.oldKey !== key) delete current[parameters.oldKey];
current[key] = Number(parameters.width) || 0;
await actions.setVariable('formatConfig', {...variables.formatConfig, column_widths: current});
```

**`fmtDeleteColWidth` (JS query, parameter: `key`)**:

```javascript
const current = {...(variables.formatConfig.column_widths || {})};
delete current[parameters.key];
await actions.setVariable('formatConfig', {...variables.formatConfig, column_widths: current});
```

#### Save & Clear

**`btnSaveFormat` event:**

| Event | Action |
|---|---|
| On click | Run `updateTemplate` with body: `{{ {id: variables.selectedTemplateId, format_config: variables.formatConfig} }}` |

No `JSON.parse` / `JSON.stringify` — the variable already holds the object. ToolJet serializes it as JSON when sending.

**`btnClearFormat` event:**

| Event | Action |
|---|---|
| On click | Show confirm dialog → Set variable `formatConfig` = `{}` → Reset all form component values |

---

#### Alternative: embed the pyDBAPI editor as a Custom Component

If maintaining 30+ ToolJet components is heavy, wrap `FormatConfigEditor.tsx` (from pyDBAPI) as a ToolJet **Custom Component**:

1. Package `FormatConfigEditor` + its `Button`, `Input`, `Select`, `Toggle` dependencies as a single bundle.
2. Expose `value` and `onChange` as ToolJet Custom Component props.
3. In this tab, replace the whole form with a single `<CustomComponent id="fmtEditor" />`:
   - `exposedVariables`: `{ value: variables.formatConfig }`
   - On event `onChange` → Run `fmtPatch` with the full new object (or simpler: `setVariable('formatConfig', payload)`).

Trade-off: avoids duplicating 30 components but introduces a React bundle dependency. Pick native ToolJet components if the team is not comfortable bundling React code.

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

Replaces the old JSON Code Editor with a **dynamic key-value form** (Postman-style). Each row is a parameter with a **type selector** so values are serialized correctly (string / number / boolean / date / JSON). Keeps a fallback "Raw JSON" toggle for advanced nested payloads.

**Tab on enter event:**

| Event | Action |
|---|---|
| On tab enter | If `variables.testParamRows` is empty: set `testParamRows = []`, set `testExportStatus = 'idle'`, clear `testExecutionId` / `testErrorMessage` |

#### Layout

```text
┌─ Parameters ────────────────────────────────────────────────┐
│ [x] Simple mode   ( )Raw JSON                                │
├─ Simple mode (default) ─────────────────────────────────────┤
│  Key            Type       Value                             │
│  [start_date ] [Date ▼  ] [2026-04-01      ] [🗑]            │
│  [region     ] [String▼ ] [north           ] [🗑]            │
│  [limit      ] [Number▼ ] [1000            ] [🗑]            │
│  [+ Add parameter]                                           │
├─ Raw JSON mode ─────────────────────────────────────────────┤
│  { "start_date": "2026-04-01", ... }                         │
└─────────────────────────────────────────────────────────────┘
[Generate Test Report]
```

#### Components

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Toggle | `testParamsMode` | Toggle | Label: `Raw JSON`, value: false (default simple mode) |
| 2 | ListView | `testParamsList` | ListView | Data: `{{variables.testParamRows}}`, visible: `{{!components.testParamsMode.value}}` |
| 3 | Button | `btnAddTestParam` | Button | Label: `+ Add parameter`, variant: outline, visible: `{{!components.testParamsMode.value}}` |
| 4 | Code Editor | `testParamsRaw` | Code Editor | Mode: JSON, showLineNumber: Yes, visible: `{{components.testParamsMode.value}}` |
| 5 | Button | `btnTestGenerate` | Button | Label: `Generate Test Report`, variant: primary, loading: `{{queries.testGenerate.isLoading}}` |
| 6 | Spinner | `testSpinner` | Spinner | Visible: `{{variables.testExportStatus === 'running'}}` |
| 7 | Text | `testResult` | Text | Visible: `{{variables.testExportStatus !== 'idle'}}` |
| 8 | Button | `btnTestDownload` | Button | Label: `Download`, visible: `{{variables.testExportStatus === 'success'}}` |

#### Variable

Add to page variables:

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `testParamRows` | Array | `[]` | Rows of `{id, key, type, value}` for the simple editor |

#### ListView row children (`testParamsList`)

Each row carries `{id, key, type, value}`:

| # | Component | Properties |
|---|---|---|
| Text Input | Label: `Key`, value: `{{listItem.key}}`, on blur → `tpUpdateParam` with `{id: listItem.id, field: 'key', value: text}` |
| Dropdown | Label: `Type`, options: `[{label:'String',value:'string'},{label:'Number',value:'number'},{label:'Boolean',value:'boolean'},{label:'Date',value:'date'},{label:'JSON',value:'json'}]`, value: `{{listItem.type}}` |
| Dynamic value input (one of the below, switched by `listItem.type`) | |
| &nbsp;&nbsp;Text Input (type=string) | Value: `{{listItem.value}}` |
| &nbsp;&nbsp;Number Input (type=number) | Value: `{{Number(listItem.value) || 0}}` |
| &nbsp;&nbsp;Checkbox (type=boolean) | Value: `{{listItem.value === true || listItem.value === 'true'}}` |
| &nbsp;&nbsp;Date Picker (type=date) | Value: `{{listItem.value}}`, format: `YYYY-MM-DD` |
| &nbsp;&nbsp;Code Editor (type=json) | Mode: JSON, value: `{{typeof listItem.value === 'string' ? listItem.value : JSON.stringify(listItem.value)}}` |
| Button | Label: `🗑`, variant: ghost → `tpDeleteParam` with `{id: listItem.id}` |

#### JS helpers

**`btnAddTestParam` on click:**

```javascript
const rows = [...(variables.testParamRows || [])];
rows.push({ id: Date.now() + '_' + Math.random(), key: '', type: 'string', value: '' });
await actions.setVariable('testParamRows', rows);
```

**`tpUpdateParam` (parameters: `id`, `field`, `value`):**

```javascript
const rows = (variables.testParamRows || []).map(r =>
  r.id === parameters.id ? { ...r, [parameters.field]: parameters.value } : r
);
await actions.setVariable('testParamRows', rows);
```

**`tpDeleteParam` (parameter: `id`):**

```javascript
await actions.setVariable(
  'testParamRows',
  (variables.testParamRows || []).filter(r => r.id !== parameters.id)
);
```

**`testParamsMode` on change** — seed the raw editor from rows when switching to Raw JSON so the user doesn't lose what they typed:

```javascript
if (components.testParamsMode.value) {
  // Simple → Raw: dump rows into editor
  const obj = {};
  for (const row of (variables.testParamRows || [])) {
    if (!row.key) continue;
    obj[row.key] = row.value;
  }
  await actions.setComponentValue('testParamsRaw', JSON.stringify(obj, null, 2));
}
// Raw → Simple: rows are preserved; user can keep editing.
```

**`tpBuildPayload` (called before `btnTestGenerate`):**

```javascript
// If Raw JSON mode, parse the code editor. Else, build from rows.
if (components.testParamsMode.value) {
  try {
    return JSON.parse(components.testParamsRaw.value || '{}');
  } catch (e) {
    throw new Error('Invalid JSON: ' + e.message);
  }
}

const out = {};
for (const row of (variables.testParamRows || [])) {
  if (!row.key) continue;
  let v = row.value;
  switch (row.type) {
    case 'number':  v = Number(v); break;
    case 'boolean': v = v === true || v === 'true'; break;
    case 'date':    v = String(v || ''); break;
    case 'json':    v = typeof v === 'string' ? JSON.parse(v || 'null') : v; break;
    default:        v = String(v ?? '');
  }
  out[row.key] = v;
}
return out;
```

**`btnTestGenerate` event chain:**

1. Run `tpBuildPayload` → `variables.resolvedTestParams = queries.tpBuildPayload.data`
2. Run `testGenerate` with body: `{{ {parameters: variables.resolvedTestParams} }}`

On failure of `tpBuildPayload` (Raw JSON parse error) → Show alert with the error message.

**`btnTestDownload` event:** Run `downloadExecution` with `executionId: {{variables.testExecutionId}}`

---

## Event Flow Summary

```text
List View:
1. Page load → initFormatConstants + listModulesForFilter + (buildListTemplatesBody → listAllTemplates)
2. Search/filter → buildListTemplatesBody → listAllTemplates
3. Click [⚙] → getTemplateDetail → showDetail=true

Detail View:
4. Config tab: edit fields → btnSaveConfig → updateTemplate
5. Files tab (MinIO Data Source):
   a. listTemplateFiles on tab enter → fileTable shows files
   b. filePicker1 select → btnUpload → uploadTemplateFile → refresh list
   c. [Delete] → confirm → deleteTemplateFile → refresh list
   d. [Download] → readTemplateFile (MinIO) → JS blob → browser download
6. Mappings tab:
   a. [+ Add Mapping] → mappingFormModal open → reset mappingFormatConfig
      → fill form (incl. visual format editor if "Override" is on)
      → btnSaveMapping → buildMappingPayload → createMapping
   b. [Edit] on card → mappingFormModal (edit) → load mappingFormatConfig
      from row → buildMappingPayload → updateMapping
   c. [Delete] → deleteMapping
7. Format tab:
   a. On tab enter → init formatConfig from getTemplateDetail
   b. Each field change → fmtPatch → mutates variables.formatConfig
   c. btnSaveFormat → updateTemplate with body {id, format_config}
   d. btnClearFormat → confirm → reset formatConfig = {}
8. Permissions tab:
   a. listTemplatePermissions on tab enter
   b. [+ Add User] → addUserModal → addPermission
   c. [Remove] → removePermission
9. Test tab:
   a. Add parameter rows (key / type / value) OR toggle Raw JSON
   b. btnTestGenerate → tpBuildPayload → testGenerate → poll → download
10. [← Back] → showDetail=false
```
