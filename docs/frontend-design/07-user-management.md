# User & Permission Management

## Purpose

Allow Admin to manage users, assign roles, and organize users into groups.

## Users

- **Admin**: CRUD users and groups
- **DA**: View user list (for assigning to Export Configs)
- **End User**: No access

## Interface

### Users Tab (default)

- Search bar + Role filter + "+ Add User" button
- Table:

| Column     | Description                 |
| ---------- | --------------------------- |
| Name       | Avatar + name               |
| Email      | Login email                 |
| Role       | Admin / DA / End User (badge) |
| Groups     | Group list (tags)           |
| Status     | Active / Inactive           |
| Last Login | Last login time             |
| Actions    | Edit, Deactivate            |

### User Create/Edit Form (dialog)

- Name, Email, Password (create only)
- Role (dropdown: Admin / DA / End User)
- Groups (multi-select)
- Status (toggle Active/Inactive)

### Groups Tab

- Table: Name, Members (count), Actions (Edit, Delete)
- Create/Edit group form (dialog): Name, Description, Members (searchable checkbox list)

## Flow

1. Admin opens User Management
2. Users tab: create user → assign role + groups → Save
3. Groups tab: create group → add members → Save
4. Export config access control is managed on the Export Config page (Access section)