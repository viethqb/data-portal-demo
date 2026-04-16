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

## Queries

### `listToolJetUsers`

Get all ToolJet workspace users. Use ToolJet's built-in workspace users API or `tj_get_users` query:

```javascript
// JavaScript query — fetch ToolJet users from workspace API
const users = tooljet.users; // or use ToolJet's user management API
return users.map(u => ({
  email: u.email,
  name: `${u.firstName || ''} ${u.lastName || ''}`.trim(),
  group: (u.groups || []).map(g => g.name).join(', '),
}));
```

### `listAllPermissions` (ToolJet Database)

```sql
SELECT tp.*, 
  (SELECT COUNT(*) FROM template_permissions tp2 
   WHERE tp2.user_email = tp.user_email) as template_count
FROM template_permissions tp
ORDER BY tp.user_email, tp.created_at
```

### `getUserPermissions` (ToolJet Database)

```sql
SELECT * FROM template_permissions
WHERE user_email = '{{variables.selectedUserEmail}}'
ORDER BY created_at
```

### `listAllTemplates` (pyDBAPI)

```
POST /api/v1/report-modules/client/templates
Body: {"page": 1, "page_size": 100}
```

Used to populate the "Assign Template" dropdown.

### `checkPermissionExists` (ToolJet Database)

| Property | Value |
|---|---|
| Table | `template_permissions` |
| Operation | List rows |
| Filter 1 | `user_email` equals `{{variables.selectedUserEmail}}` |
| Filter 2 | `template_id` equals `{{parameters.templateId}}` |

### `assignTemplate` (JavaScript)

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
return { skipped: false };
```

### `insertPermission` (ToolJet Database)

| Property | Value |
|---|---|
| Table | `template_permissions` |
| Operation | Create row |
| `user_email` | `{{variables.selectedUserEmail}}` |
| `template_id` | `{{parameters.templateId}}` |

### `removePermission` (ToolJet Database)

```sql
DELETE FROM template_permissions WHERE id = {{parameters.permissionId}}
```

### `assignAllTemplates` (JavaScript)

```javascript
const templates = queries.listAllTemplates.data?.data || [];
const email = variables.selectedUserEmail;

for (const t of templates) {
  await queries.assignTemplate.run({ templateId: t.id });
}

await queries.getUserPermissions.run(); // refresh
```

### `removeAllPermissions` (ToolJet Database)

```sql
DELETE FROM template_permissions
WHERE user_email = '{{variables.selectedUserEmail}}'
```

## Components

### Users Table

| Property | Value |
|---|---|
| Data | Merged: ToolJet users + permission counts |

**Merge logic** (JavaScript query `buildUserList`):

```javascript
const users = queries.listToolJetUsers.data || [];
const perms = queries.listAllPermissions.data || [];

// Count permissions per user
const permCounts = {};
const permTemplates = {};
for (const p of perms) {
  permCounts[p.user_email] = (permCounts[p.user_email] || 0) + 1;
  if (!permTemplates[p.user_email]) permTemplates[p.user_email] = [];
  permTemplates[p.user_email].push(p.template_id);
}

return users.map(u => ({
  ...u,
  templateCount: permCounts[u.email] || 0,
  isAdminOrDA: u.group.includes('admin') || u.group.includes('da'),
}));
```

| Column | Source | Format |
|---|---|---|
| Email | `email` | Text |
| Name | `name` | Text |
| Group | `group` | Badge |
| Templates | `templateCount` | Number, or "all" for admin/da |
| Actions | — | [Manage] button |

**[Manage] button:**
1. Set `selectedUserEmail = rowData.email`
2. Run `getUserPermissions`
3. Open `userPermissionsModal`

### User Permissions Modal

**Assigned Templates Table:**

| Column | Source | Format |
|---|---|---|
| Template | `template_id` | Map to template name from `listAllTemplates` |
| Module | — | Map template → module |
| Actions | — | [Remove] button |

**Assign Template Section:**

- Dropdown listing all templates from `listAllTemplates`
- Already-assigned templates shown as disabled
- [Add] button → run `assignTemplate` → refresh

**Quick Actions:**
- [Assign All] → run `assignAllTemplates`
- [Remove All] → run `removeAllPermissions` → refresh

## Variables

| Variable | Type | Default |
|---|---|---|
| `selectedUserEmail` | String | `''` |

## Notes

- Admin and DA users don't need `template_permissions` entries — they see all templates via role check
- Only `end_user` group members need explicit template assignments
- The table shows admin/da users as "(all — role)" for clarity
- When a template is deleted from pyDBAPI, orphan `template_permissions` rows are harmless (the template won't appear in API results)
