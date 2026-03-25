# Dataset Management

## Purpose

Allow Admin/DA to create and manage datasets. Each dataset = one SQL query + a list of filter parameters.

## Users

- **Admin**: CRUD all datasets
- **DA**: CRUD own datasets
- **End User**: No access

## Interface

### List Page

- Search bar + "+ New Dataset" button
- Table:

| Column   | Description                      |
| -------- | -------------------------------- |
| Name     | Dataset name                     |
| Database | Source DB (badge)                |
| Status   | Active / Inactive                |
| Used In  | Number of export configs using it |
| Modified | Last updated date                |
| Actions  | Edit, Delete                     |

### Create/Edit Form (single page, no tabs)

All on one scrollable page:

1. **Basic Info**: Name, Description, Database Source (dropdown), Status (toggle)
2. **SQL Editor**: Code editor with syntax highlighting. Filter parameters use `{{param_name}}` syntax
3. **Filter Parameters**: Auto-detected table from SQL, each row contains:
   - Parameter name (auto-detected)
   - Label (text input)
   - Type (dropdown: Text / Number / Date / Select)
   - Default value (text input)
4. **"Preview" button**: Runs the query, shows first 50 rows below
5. **"Save" button**

## Flow

1. DA clicks "+ New Dataset"
2. Fills in name, selects database, writes SQL
3. System auto-detects `{{params}}` → displays filter table
4. DA configures label, type, default for each filter
5. Clicks Preview → reviews results → Save