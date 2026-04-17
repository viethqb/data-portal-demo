# Setup — ToolJet + pyDBAPI Integration

## Prerequisites

1. **ToolJet** instance running (self-hosted Docker or ToolJet Cloud)
2. **pyDBAPI** deployed with Report Engine, MinIO, Redis

## Part 1: pyDBAPI Setup (one-time, on Dashboard)

### 1. Create a Report Module

Dashboard → **Report Management → Modules → Create Module**

- **SQL Datasource**: PostgreSQL/MySQL containing your data
- **MinIO Datasource**: MinIO for template/output storage
- **Template Bucket**: e.g., `report-templates`
- **Output Bucket**: e.g., `report-output`

### 2. Create an AppClient

Dashboard → **System → Clients → Create**

- **Name**: e.g., `tooljet-reports`
- **Client ID**: e.g., `tooljet-reports`
- **Client Secret**: set a strong secret — note it down
- Optionally set **Token Expiration** (default 3600s = 1 hour)

### 3. Assign Client to Module

Dashboard → **System → Clients → click client → Edit**

- Scroll to **Report Modules** at the bottom
- Select the module(s) → **Save**

## Part 2: ToolJet Setup

### Step 1: Create ToolJet Database Tables

In ToolJet: **ToolJet Database** (left sidebar database icon)

**Table: `template_permissions`**

| Column | Type | Constraints |
|---|---|---|
| `id` | serial | Primary key |
| `user_email` | varchar | Not null |
| `template_id` | varchar | Not null |
| `created_at` | timestamp | Set in query: `new Date().toISOString()` |

> Duplicate prevention is handled at application level (check before insert) since ToolJet DB UI does not support unique constraints.

**Table: `export_log`**

| Column | Type | Constraints |
|---|---|---|
| `id` | serial | Primary key |
| `user_email` | varchar | Not null |
| `user_name` | varchar | |
| `template_id` | varchar | Not null |
| `template_name` | varchar | |
| `execution_id` | varchar | |
| `parameters` | varchar | JSON string |
| `status` | varchar | Default: `pending` |
| `created_at` | timestamp | Set in query: `new Date().toISOString()` |

**Table: `schedulers`**

| Column | Type | Constraints |
|---|---|---|
| `id` | serial | Primary key |
| `name` | varchar | Not null |
| `description` | varchar | |
| `template_id` | varchar | Not null |
| `template_name` | varchar | |
| `module_id` | varchar | Not null |
| `parameters` | varchar | JSON with date placeholders |
| `cron_expression` | varchar | Not null |
| `timezone` | varchar | Default: `Asia/Ho_Chi_Minh` |
| `recipients` | varchar | Comma-separated emails |
| `email_subject` | varchar | |
| `email_body` | varchar | |
| `is_active` | boolean | Default: true |
| `last_run_at` | timestamp | |
| `last_status` | varchar | |
| `last_error` | varchar | |
| `created_at` | timestamp | Set in query: `new Date().toISOString()` |
| `updated_at` | timestamp | Set in query: `new Date().toISOString()` |

### Step 2: Create REST API Data Source (OAuth 2.0)

**Settings → Data Sources → + Add Data Source → REST API**

| Field | Value |
|---|---|
| Name | `pyDBAPI` |
| Base URL | `http://pydbapi-host:8000` |
| Authentication | **OAuth 2.0** |
| Grant Type | **Client Credentials** |
| Access Token URL | `http://pydbapi-host:8000/api/token/generate` |
| Client ID | `tooljet-reports` (from step 2 above) |
| Client Secret | (the secret you set above) |
| Scopes | (leave empty) |

Add default header:

| Key | Value |
|---|---|
| Content-Type | `application/json` |

Click **Save & Test**.

**How it works:**
- ToolJet calls the token endpoint automatically using Client Credentials grant
- Token is stored **server-side** by ToolJet — never exposed to the browser
- ToolJet auto-refreshes the token when it expires
- All queries through this data source include `Authorization: Bearer <token>` automatically
- No JavaScript `fetch`, no `localStorage`, no manual token management needed

### Step 3: Create MinIO Data Source

**Settings → Data Sources → + Add Data Source → MinIO**

| Field | Value |
|---|---|
| Name | `MinIO` |
| Host | `minio-host` (same as pyDBAPI's MinIO) |
| Port | `9000` |
| Access Key | MinIO access key (e.g., `minioadmin`) |
| Secret Key | MinIO secret key |

Click **Save & Test**.

This data source is used for **template file management** — upload, list, delete `.xlsx` files. Report generation still goes through pyDBAPI API.

**Available MinIO operations:**

| Operation | Use Case |
|---|---|
| **List Buckets** | Show available buckets |
| **List Objects in a Bucket** | Browse template files (with prefix filter) |
| **Put Object** | Upload new `.xlsx` template |
| **Remove Object** | Delete template file |
| **Read Object** | Download file content (server-side, for blob download) |

### Step 4: Create SMTP Data Source (for Scheduler emails)

**Settings → Data Sources → + Add Data Source → SMTP**

| Field | Value |
|---|---|
| Name | `SMTP` |
| Host | `smtp.gmail.com` (or your SMTP server) |
| Port | `587` |
| Username | `noreply@company.com` |
| Password | App password or SMTP password |

> Optional — only needed if using the Scheduler email delivery feature.

### Step 5: Create the App

**Apps → + Create New App → Blank App**

- App name: `Data Portal`
- Navigation: **Side menu** with icons

### Step 6: Create User Groups

**Settings → Groups**

| Group | Description |
|---|---|
| `admin` | All pages, manage user→template permissions |
| `da` | Manage templates/mappings/schedulers |
| `end_user` | Export assigned templates, view own history |

### Step 7: Create 6 Pages

| # | Page Name | Handle | Home | Permissions |
|---|---|---|---|---|
| 1 | Dashboard | `dashboard` | ✅ | All users |
| 2 | My Exports | `exports` | | All users |
| 3 | Export History | `history` | | All users |
| 4 | Templates | `templates` | | admin, da |
| 5 | Schedulers | `schedulers` | | admin, da |
| 6 | Users & Permissions | `permissions` | | admin |

## Security Model

```text
┌─ Browser ────────────────────────────────────────────┐
│  Only connects to ToolJet                            │
│  Never sees: pyDBAPI URL, MinIO URL, tokens, secrets │
└──────────┬───────────────────────────────────────────┘
           │ HTTPS
┌──────────▼───────────────────────────────────────────┐
│  ToolJet Server                                      │
│                                                       │
│  pyDBAPI Data Source (OAuth 2.0)                     │
│  • Client ID + Secret encrypted, server-side         │
│  • Token auto-managed, never reaches browser         │
│                                                       │
│  MinIO Data Source (direct)                           │
│  • Access Key + Secret encrypted, server-side        │
│  • Read Object → base64 → blob download in browser   │
│  • Put Object → upload from File Picker              │
│                                                       │
│  SMTP Data Source                                     │
│  • Email delivery for scheduled reports              │
└──────────┬───────────────────────────────────────────┘
           │ Internal network
┌──────────▼───────────────────────────────────────────┐
│  pyDBAPI + MinIO + Redis (not exposed to browser)    │
└──────────────────────────────────────────────────────┘
```

## Shared Queries & Helpers

These queries are defined **once** at the app level and referenced from multiple pages. Add them to the app (Global Data Sources → pyDBAPI / MinIO / JavaScript) so every page can call them via `queries.<name>.run(...)`.

| Name | Type | Used by | Purpose |
|---|---|---|---|
| `getExecutionForDownload` | pyDBAPI REST | Download flow | Fetch execution detail → get `output_minio_path` |
| `readReportFile` | MinIO Read Object | Download flow | Read file content from MinIO output bucket |
| `downloadExecution` | JavaScript | Dashboard, My Exports, History, Templates Test tab | Orchestrates get-detail → read-minio → blob download |
| `getUserRole` | JavaScript | All pages | Resolve current user's role from groups |

See full definitions in **Download Helper** and **Role Check Helper** below.

## Common Patterns

### pyDBAPI Queries

All pyDBAPI API calls use the `pyDBAPI` REST API data source. Authentication is handled automatically by OAuth 2.0 — no manual `Authorization` header needed.

Example query `listTemplates`:

| Field | Value |
|---|---|
| Data Source | `pyDBAPI` |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 100}` |

No `Authorization` header — ToolJet injects it server-side.

### Download Helper

Generated reports are stored in MinIO. Download flow goes entirely through **ToolJet server** — browser never connects to pyDBAPI or MinIO directly.

```text
Browser ←──file──── ToolJet Server ←──read object──── MinIO (internal)
          (blob)         │
                         ├── Step 1: pyDBAPI query → get output_minio_path
                         ├── Step 2: MinIO Read Object → get file content
                         └── Step 3: JS blob → trigger browser download
```

**Step 1:** Create **pyDBAPI query** `getExecutionForDownload`:

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | GET |
| URL | `/api/v1/report-modules/client/execution-detail?exec_id={{parameters.execId}}` |

**Step 2:** Create **MinIO query** `readReportFile`:

| Property | Value |
|---|---|
| Data source | `MinIO` |
| Operation | **Read Object** |
| Bucket | `{{parameters.bucket}}` |
| Object Name | `{{parameters.objectName}}` |

**Step 3:** Create **JavaScript query** `downloadExecution`:

```javascript
// 1. Get execution detail → find MinIO path
const execResp = await queries.getExecutionForDownload.run({
  execId: parameters.executionId,
});
const minioPath = execResp.data?.output_minio_path;
if (!minioPath) throw new Error('No output file');

// 2. Parse "bucket/path/to/file.xlsx"
const [bucket, ...pathParts] = minioPath.split('/');
const objectName = pathParts.join('/');
const fileName = pathParts[pathParts.length - 1];

// 3. Read file from MinIO (ToolJet server fetches, returns to browser)
const fileResp = await queries.readReportFile.run({
  bucket: bucket,
  objectName: objectName,
});

// 4. Convert to blob and trigger download
const base64 = fileResp.data; // MinIO Read Object returns base64
const byteChars = atob(base64);
const byteNumbers = new Array(byteChars.length);
for (let i = 0; i < byteChars.length; i++) {
  byteNumbers[i] = byteChars.charCodeAt(i);
}
const byteArray = new Uint8Array(byteNumbers);
const blob = new Blob([byteArray], {
  type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
});

const url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = fileName;
a.click();
URL.revokeObjectURL(url);
```

**Security:** Browser only connects to ToolJet. No pyDBAPI URL, MinIO URL, tokens, or credentials in the browser.

### Current ToolJet User

```javascript
{{globals.currentUser.email}}          // Email
{{globals.currentUser.firstName}}      // First name
{{globals.currentUser.groups}}         // Array of group objects
```

### Role Check Helper

Create **JavaScript query** `getUserRole` (run on app load):

```javascript
const groups = (globals.currentUser.groups || []).map(g => g.name);
if (groups.includes('admin')) return 'admin';
if (groups.includes('da')) return 'da';
return 'end_user';
```

Use in component visibility:

```javascript
{{queries.getUserRole.data === 'end_user'}}   // true for end users
{{queries.getUserRole.data !== 'end_user'}}    // true for admin/da
```

### ToolJet DB: Get Permitted Template IDs

Create **ToolJet Database query** `getMyPermissions`:

| Field | Value |
|---|---|
| Data Source | ToolJet Database |
| Table | `template_permissions` |
| Operation | List rows |
| Filter | `user_email` equals `{{globals.currentUser.email}}` |

### ToolJet DB: Log an Export

Create **ToolJet Database query** `logExport`:

| Field | Value |
|---|---|
| Data Source | ToolJet Database |
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

### ToolJet DB: Update Export Status

Create **ToolJet Database query** `updateExportStatus`:

| Field | Value |
|---|---|
| Data Source | ToolJet Database |
| Table | `export_log` |
| Operation | Update row |
| Filter | `execution_id` equals `{{parameters.executionId}}` |
| Column `status` | `{{parameters.status}}` |

### Error Handling

On every query failure → Show Alert (Error): `{{queries.<name>.rawData?.detail || 'Request failed'}}`
