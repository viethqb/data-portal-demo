# Page 7: Users & Permissions

**Handle:** `permissions` | **Permissions:** admin only

Manage which end users can access which templates. Users are ToolJet workspace users (not pyDBAPI users).

## Wireframe

```text
┌─────────────────────────────────────────────────────────────────────┐
│  Users & Permissions                                                │
│  [Search users...________]                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────┬──────────────┬──────────┬──────────────────────┐│
│  │ Email          │ Name         │ Group    │ Assigned Templates   ││
│  ├────────────────┼──────────────┼──────────┼──────────────────────┤│
│  │ nguyen@co.com  │ Nguyen A     │ end_user │ Sales Q1, KPI Month  ││
│  │ tran@co.com    │ Tran B       │ end_user │ HR Report            ││
│  │ le@co.com      │ Le C         │ end_user │ (none)               ││
│  │ da@co.com      │ DA User      │ da       │ (all — DA role)      ││
│  │ admin@co.com   │ Admin        │ admin    │ (all — Admin role)   ││
│  └────────────────┴──────────────┴──────────┴──────────────────────┘│
│                                                                     │
│  Click a row to manage template assignments                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## User Detail Modal (on row click)

```text
┌──────────────────────────────────────────────────────────────────┐
│  Manage Permissions: nguyen@co.com                          [X]  │
│  Name: Nguyen A  |  Group: end_user                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Assigned Templates:                                             │
│                                                                  │
│  ┌──────────────────────┬──────────┬───────────────────────────┐ │
│  │ Template             │ Module   │ Actions                   │ │
│  ├──────────────────────┼──────────┼───────────────────────────┤ │
│  │ Sales Q1 Report      │ Finance  │ [Remove]                  │ │
│  │ KPI Monthly          │ Ops      │ [Remove]                  │ │
│  └──────────────────────┴──────────┴───────────────────────────┘ │
│                                                                  │
│  [+ Assign Template]                                             │
│                                                                  │
│  ── Assign Template Dropdown ──                                  │
│                                                                  │
│  Template: [Select template...          ▼]                       │
│  ┌──────────────────────────────────────┐                        │
│  │ Finance / Sales Q1 Report  (assigned)│                        │
│  │ Finance / Monthly Budget             │                        │
│  │ Ops / KPI Monthly         (assigned) │                        │
│  │ Ops / Inventory Snapshot             │                        │
│  │ HR / Employee Summary                │                        │
│  └──────────────────────────────────────┘                        │
│  [Add]                                                           │
│                                                                  │
│  ── Quick Actions ──                                             │
│  [Assign All Templates]  [Remove All]                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## Page Events

| Event | Action |
|---|---|
| On page load | Run: `listToolJetUsers`, `listAllTemplates`, `listAllPermissions` → `buildUserList` |

## Queries

> **Note on raw SQL:** ToolJet Database's built-in operations (`List rows`, `Create row`, …) do NOT accept raw SQL. For the delete-by-filter patterns below, either use the `Delete row` operation with filters, or add a **PostgreSQL data source** pointed at the ToolJet DB (`tooljet_db`) so you can run real SQL. Code blocks below show both options.

### Q1: `listToolJetUsers`

Get all ToolJet workspace users. The simplest approach uses ToolJet's internal admin API through a REST data source pointed at the ToolJet server.

| Property | Value |
|---|---|
| Data source | REST API — `http://tooljet-server:3000` (internal) or ToolJet Admin API token |
| Method | GET |
| URL | `/api/organization_users` (ToolJet workspace users endpoint) |
| Transform | `return data.organization_users.map(u => ({ email: u.email, name: \`${u.first_name || ''} ${u.last_name || ''}\`.trim(), group: (u.groups || []).map(g => g.name).join(',') }))` |

> **Alt (simpler, no admin API):** if the list of end users is small and static, maintain it as a second ToolJet DB table `portal_users` and query it like any other row list. Trade-off: manual sync when new ToolJet users are added.

### Q2: `listAllPermissions`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | **List rows** |
| Sort by | `user_email` ASC, then `created_at` ASC |

Returns every permission row. Merging happens client-side in `buildUserList` — cheaper than the N+1 count subquery that raw SQL would require.

### Q3: `getUserPermissions`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | **List rows** |
| Filter | `user_email` equals `{{variables.selectedUserEmail}}` |
| Sort by | `created_at` ASC |
| Run on demand | When user row clicked |

### Q4: `listAllTemplates`

| Property | Value |
|---|---|
| Data source | `pyDBAPI` (REST API) |
| Method | POST |
| URL | `/api/v1/report-modules/client/templates` |
| Body | `{"page": 1, "page_size": 100}` |
| Run on page load | Yes |

Used to populate the "Assign Template" dropdown and to resolve `template_id` → template name in the Assigned Templates table.

### Q5: `checkPermissionExists`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | **List rows** |
| Filter 1 | `user_email` equals `{{variables.selectedUserEmail}}` |
| Filter 2 | `template_id` equals `{{parameters.templateId}}` |

### Q6: `assignTemplate` (JavaScript)

```javascript
// Check duplicate before insert
const existing = await queries.checkPermissionExists.run({
  templateId: parameters.templateId,
});
if (existing.data && existing.data.length > 0) {
  return { skipped: true };
}

await queries.insertPermission.run({ templateId: parameters.templateId });
await queries.getUserPermissions.run();
await queries.listAllPermissions.run();        // refresh global counts
await queries.buildUserList.run();
return { skipped: false };
```

### Q7: `insertPermission`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | **Create row** |
| `user_email` | `{{variables.selectedUserEmail}}` |
| `template_id` | `{{parameters.templateId}}` |
| `created_at` | `{{new Date().toISOString()}}` |

### Q8: `removePermission`

| Property | Value |
|---|---|
| Data source | ToolJet Database |
| Table | `template_permissions` |
| Operation | **Delete row** |
| Filter | `id` equals `{{parameters.permissionId}}` |

| Event | Action |
|---|---|
| On success | Run `getUserPermissions` → `listAllPermissions` → `buildUserList` |

### Q9: `assignAllTemplates` (JavaScript)

```javascript
const templates = queries.listAllTemplates.data?.data || [];
const assigned = new Set((queries.getUserPermissions.data || []).map(p => p.template_id));

let added = 0;
for (const t of templates) {
  if (assigned.has(t.id)) continue;
  await queries.insertPermission.run({ templateId: t.id });
  added++;
}

await queries.getUserPermissions.run();
await queries.listAllPermissions.run();
await queries.buildUserList.run();
return { added };
```

### Q10: `removeAllPermissions`

Preferred: a JS query that lists → deletes each row via the `Delete row` operation. Avoids needing raw SQL.

```javascript
const rows = queries.getUserPermissions.data || [];
for (const r of rows) {
  await queries.removePermission.run({ permissionId: r.id });
}
await queries.getUserPermissions.run();
await queries.listAllPermissions.run();
await queries.buildUserList.run();
```

Alternative (if you have a PostgreSQL data source on `tooljet_db`):

```sql
DELETE FROM template_permissions
WHERE user_email = '{{variables.selectedUserEmail}}'
```

### Q11: `buildUserList` (JavaScript)

Merges ToolJet users with their permission counts. Runs once on page load and again whenever permissions change.

```javascript
const users = queries.listToolJetUsers.data || [];
const perms = queries.listAllPermissions.data || [];

const permCounts = {};
const permTemplates = {};
for (const p of perms) {
  permCounts[p.user_email] = (permCounts[p.user_email] || 0) + 1;
  if (!permTemplates[p.user_email]) permTemplates[p.user_email] = [];
  permTemplates[p.user_email].push(p.template_id);
}

// group is a comma-joined string like "admin,end_user" — split before comparing
return users.map(u => {
  const groups = (u.group || '').split(',').map(s => s.trim());
  const isAdminOrDA = groups.includes('admin') || groups.includes('da');
  return {
    ...u,
    templateCount: isAdminOrDA ? 'all' : (permCounts[u.email] || 0),
    templateIds: permTemplates[u.email] || [],
    isAdminOrDA,
  };
});
```

## Components

### C1: `headerText`

| Property | Value |
|---|---|
| Component | Text |
| Content | `Users & Permissions` |
| Font size | 24px, bold |

### C2: `searchUsers`

| Property | Value |
|---|---|
| Component | Text Input |
| Placeholder | `Search by email or name...` |
| Width | 320px |

Filter happens client-side in `userTable.Data`; no query needs to re-run.

### C3: `userTable`

| Property | Value |
|---|---|
| Component | Table |
| Data | `{{(queries.buildUserList.data || []).filter(u => !components.searchUsers.value \|\| u.email.includes(components.searchUsers.value) \|\| (u.name \|\| '').toLowerCase().includes(components.searchUsers.value.toLowerCase()))}}` |
| Loading | `{{queries.buildUserList.isLoading}}` |
| Show pagination | Yes (client-side) |

**Columns:**

| # | Header | Key | Type | Width | Config |
|---|---|---|---|---|---|
| 1 | Email | `email` | Default | 240px | — |
| 2 | Name | `name` | Default | 180px | — |
| 3 | Group | `group` | **Badges** | 160px | One badge per group, split the comma-joined string |
| 4 | Templates | `templateCount` | Default | 100px | Cell: `{{cellValue === 'all' ? 'all (role)' : cellValue}}` |
| 5 | Actions | — | Action button | 120px | See below |

**Action button: `Manage`:**

| Event | Action |
|---|---|
| On click | 1. Set variable `selectedUserEmail` = `{{rowData.email}}` |
| | 2. Run `getUserPermissions` |
| | 3. Show modal `userPermissionsModal` |

### C4: `userPermissionsModal`

| Property | Value |
|---|---|
| Component | Modal |
| Title | `Manage Permissions: {{variables.selectedUserEmail}}` |
| Size | Large |

**Children:**

| # | Component | ID | Type | Properties |
|---|---|---|---|---|
| 1 | Text | `upmHeaderInfo` | Text | Content: `Group: {{(queries.buildUserList.data || []).find(u => u.email === variables.selectedUserEmail)?.group \|\| ''}}`, font: 13px, muted |
| 2 | Text | `upmAdminHint` | Text | Content: `This user has the admin/DA role and automatically sees all templates. Assignments below are ignored for role-based access.`, color: amber, visible: `{{(queries.buildUserList.data || []).find(u => u.email === variables.selectedUserEmail)?.isAdminOrDA}}` |
| 3 | Text | `upmAssignedTitle` | Text | Content: `Assigned Templates`, font: 16px, bold |
| 4 | Table | `assignedTemplatesTable` | Table | Data: `{{queries.getUserPermissions.data}}`, see columns below |
| 5 | Text | `upmAssignTitle` | Text | Content: `Assign a Template`, font: 16px, bold |
| 6 | Dropdown | `upmTemplateToAssign` | Dropdown | Options: see below (already-assigned are disabled), searchInOptions: true |
| 7 | Button | `btnAssignTemplate` | Button | Label: `Assign`, variant: primary, disabled: `{{!components.upmTemplateToAssign.value}}` |
| 8 | Text | `upmQuickActionsTitle` | Text | Content: `Quick Actions`, font: 14px, bold |
| 9 | Button | `btnAssignAll` | Button | Label: `Assign All Templates`, variant: outline |
| 10 | Button | `btnRemoveAll` | Button | Label: `Remove All`, variant: destructive-outline |
| 11 | Button | `btnCloseUserModal` | Button | Label: `Close`, variant: ghost |

**`assignedTemplatesTable` columns:**

| # | Header | Key | Width | Config |
|---|---|---|---|---|
| 1 | Template | `template_id` | 260px | Cell: `{{queries.listAllTemplates.data?.data?.find(t => t.id === cellValue)?.name \|\| cellValue}}` |
| 2 | Module | `template_id` | 160px | Cell: `{{queries.listAllTemplates.data?.data?.find(t => t.id === cellValue)?.report_module_id \|\| '—'}}` |
| 3 | Assigned | `created_at` | 140px | Cell: `{{new Date(cellValue).toLocaleDateString()}}` |
| 4 | Actions | — | 100px | [Remove] button — Run `removePermission` with `permissionId: {{rowData.id}}` |

**`upmTemplateToAssign` options:**

```javascript
{{
  (queries.listAllTemplates.data?.data || []).map(t => {
    const assignedIds = new Set((queries.getUserPermissions.data || []).map(p => p.template_id));
    return {
      label: assignedIds.has(t.id) ? (t.name + '  (already assigned)') : t.name,
      value: t.id,
      disable: assignedIds.has(t.id),
    };
  })
}}
```

**Event bindings:**

| Component | Event | Action |
|---|---|---|
| `btnAssignTemplate` | On click | Run `assignTemplate` with `templateId: {{components.upmTemplateToAssign.value}}` → `setComponentValue(upmTemplateToAssign, '')` |
| `btnAssignAll` | On click | Show confirm "Assign all templates to this user?" → Run `assignAllTemplates` |
| `btnRemoveAll` | On click | Show confirm "Remove ALL template permissions from this user?" → Run `removeAllPermissions` |
| `btnCloseUserModal` | On click | Close `userPermissionsModal` |

## Event Flow

```text
1. Page load → listToolJetUsers + listAllTemplates + listAllPermissions → buildUserList → userTable
2. Type in searchUsers → userTable filters client-side
3. Click [Manage] on a row:
   a. selectedUserEmail = row.email → getUserPermissions → Show userPermissionsModal
4. In modal:
   a. Pick template → btnAssignTemplate → assignTemplate → refresh table + counts
   b. [Remove] row → removePermission → refresh
   c. [Assign All] → assignAllTemplates (skips already-assigned)
   d. [Remove All] → removeAllPermissions (JS loop over rows, no raw SQL needed)
5. [Close] → modal hides (selectedUserEmail persists; getUserPermissions cache stale until next Manage)
```

## Variables

| Variable | Type | Default |
|---|---|---|
| `selectedUserEmail` | String | `''` |

## Notes

- Admin and DA users don't need `template_permissions` entries — they see all templates via role check
- Only `end_user` group members need explicit template assignments
- The table shows admin/da users as "(all — role)" for clarity
- When a template is deleted from pyDBAPI, orphan `template_permissions` rows are harmless (the template won't appear in API results)
