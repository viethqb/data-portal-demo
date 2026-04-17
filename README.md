# Data Portal v2

Self-service Excel report platform built as a **ToolJet app**, powered by [pyDBAPI Report Engine](https://github.com/viethqb/pydbapi).

## Architecture

```text
                    ┌─ ToolJet Server ─────────────────────────────┐
                    │                                             │
End Users ────────→ │  ToolJet App (browser only talks to here)  │
  (browser)         │                                             │
                    │  ToolJet Database:                          │
                    │  ├── template_permissions (who sees what)   │
                    │  ├── export_log (audit trail)               │
                    │  └── schedulers (cron config)               │
                    │                                             │
                    │  Data Sources (server-side, internal):      │
                    │  ├── pyDBAPI (OAuth 2.0) → Report API       │
                    │  ├── MinIO (direct) → File upload/download  │
                    │  └── SMTP → Email delivery                  │
                    │                                             │
                    └──────────┬──────────────────────────────────┘
                               │ (internal network only)
                    ┌──────────▼──────────────────┐
                    │  pyDBAPI + MinIO + Redis     │
                    │  (not exposed to browser)    │
                    └─────────────────────────────┘
```

**Key design:**
- **Browser only connects to ToolJet** — pyDBAPI and MinIO are internal, never exposed
- **pyDBAPI Data Source**: OAuth 2.0 Client Credentials — token auto-managed server-side
- **MinIO Data Source**: File upload/download via server-side connection
- **File download**: MinIO Read Object → base64 → JS blob → browser download
- **User permissions**: ToolJet Database tables control which end users see which templates

## Prerequisites (on pyDBAPI Dashboard)

1. **Create a Report Module** — pairs SQL datasource with MinIO storage
2. **Create an AppClient** — System → Clients → Create (note `client_id` + `client_secret`)
3. **Assign Client to Module** — System → Clients → Edit client → select Report Modules → Save

## Permission Model

```text
pyDBAPI (backend security)          ToolJet (frontend permissions)
┌─────────────────────────┐        ┌──────────────────────────────┐
│ Client → Module access  │        │ ToolJet Groups:              │
│ (which modules the      │        │   admin    → all pages       │
│  client token can use)  │        │   da       → manage templates│
│                         │        │   end_user → export only     │
│ API returns ALL          │        │                              │
│ templates in module     │        │ ToolJet DB:                  │
│                         │        │   template_permissions table │
│                         │        │   → user_email + template_id │
│                         │        │   → end_user sees only       │
│                         │        │     assigned templates       │
│                         │        │                              │
│                         │        │ export_log table             │
│                         │        │   → tracks who exported what │
└─────────────────────────┘        └──────────────────────────────┘
```

## Documents

| File | Description |
|---|---|
| [00-setup.md](00-setup.md) | pyDBAPI setup, ToolJet datasource, ToolJet DB tables, token management |
| [pages/01-dashboard.md](pages/01-dashboard.md) | Dashboard — stats, recent exports, quick export |
| [pages/02-my-exports.md](pages/02-my-exports.md) | My Exports — filtered by permissions, generate + download |
| [pages/03-export-history.md](pages/03-export-history.md) | Export History — end_user sees own, admin sees all |
| [pages/04-templates.md](pages/04-templates.md) | Templates — CRUD + manage user permissions |
| [pages/05-schedulers.md](pages/05-schedulers.md) | Schedulers — cron schedule, email delivery |
| [pages/06-users.md](pages/06-users.md) | Users & Permissions — assign templates to users |

## ToolJet Database Tables

### `template_permissions`

| Column | Type | Description |
|---|---|---|
| `id` | serial | Primary key |
| `user_email` | varchar | ToolJet user email |
| `template_id` | varchar | pyDBAPI template ID |
| `created_at` | timestamp | Set in query: `new Date().toISOString()` |

### `schedulers`

| Column | Type | Description |
|---|---|---|
| `id` | serial | Primary key |
| `name` | varchar | Schedule name |
| `template_id` | varchar | pyDBAPI template ID |
| `module_id` | varchar | pyDBAPI module ID |
| `parameters` | varchar | JSON with date placeholders |
| `cron_expression` | varchar | Cron syntax (e.g., `0 9 * * 1`) |
| `timezone` | varchar | Default: Asia/Ho_Chi_Minh |
| `recipients` | varchar | Comma-separated emails |
| `is_active` | boolean | Pause/resume |
| `last_run_at` | timestamp | Last execution |
| `last_status` | varchar | success / failed |

### `export_log`

| Column | Type | Description |
|---|---|---|
| `id` | serial | Primary key |
| `user_email` | varchar | Who exported |
| `user_name` | varchar | Display name |
| `template_id` | varchar | Which template |
| `template_name` | varchar | Template name (snapshot) |
| `execution_id` | varchar | pyDBAPI execution ID |
| `parameters` | varchar | JSON string — filters used |
| `status` | varchar | pending / success / failed |
| `created_at` | timestamp | When exported |

## Page Map

```text
┌─ Navigation (sidebar) ─────────────────────┐
│                                             │
│  📊  Dashboard           ← All roles       │
│  📥  My Exports          ← All roles       │
│  📋  Export History       ← All roles       │
│  ──────────────────────                     │
│  📑  Templates           ← admin, da       │
│  ⏰  Schedulers          ← admin, da       │
│  ──────────────────────                     │
│  👥  Users & Permissions ← admin           │
│                                             │
└─────────────────────────────────────────────┘
```

## Roles (ToolJet Groups)

| Group | Pages | Template Access | Permissions |
|---|---|---|---|
| **admin** | All | See all, CRUD all | Manage user→template assignments |
| **da** | Dashboard, Exports, History, Modules, Templates, Schedulers | See all, CRUD all | Cannot manage user assignments |
| **end_user** | Dashboard, My Exports, History | Only assigned templates | Export + view own history |
