# Export Config Management

## Purpose

Allow DA to create export configurations: select a Dataset + Template, configure how data is written into Excel, set default filters, and assign access to End Users.

## Users

- **Admin**: CRUD all configs
- **DA**: CRUD own configs
- **End User**: No access (uses configs via the Export page)

## Interface

### List Page

- Search bar + "+ New Export Config" button
- Table:

| Column      | Description          |
| ----------- | -------------------- |
| Name        | Config name          |
| Dataset     | Source dataset       |
| Template    | Template used        |
| Status      | Active / Draft       |
| Last Export | Last export time     |
| Actions     | Edit, Delete         |

### Create/Edit Form (single page, scrollable sections)

**Section 1 — Basic Info**
- Name, Description, Status (Active/Draft)

**Section 2 — Data Source**
- Dataset (dropdown)
- Template (dropdown)

**Section 3 — Data Mapping**
- Target Sheet (dropdown of sheets in template)
- Start Row (row number where data writing begins)
- Column Mapping table:

| Column (template) | Field (dataset) | Format      |
| ------------------ | --------------- | ----------- |
| A                  | order_id        | Number      |
| B                  | customer_name   | Text        |
| C                  | amount          | #,##0       |

- Cell Values table (for headers/filter values):

| Cell | Value          |
| ---- | -------------- |
| B2   | {{start_date}} |
| D2   | {{end_date}}   |

**Section 4 — Filter Defaults**
- Table of filters (from dataset), each row:

| Filter     | Default Value       | User Editable |
| ---------- | ------------------- | ------------- |
| start_date | First day of month  | Yes           |
| end_date   | Today               | Yes           |
| department | ALL                 | Yes           |

**Section 5 — Access Control**
- Radio: "All End Users" or "Specific Users/Groups"
- If Specific: searchable tag input to select users/groups

**"Preview Export"** + **"Save"** buttons

## Flow

1. DA clicks "+ New Export Config"
2. Fills in name → selects Dataset + Template
3. Configures column mapping (column → field)
4. Sets filter defaults + marks which filters End Users can edit
5. Selects users/groups with access
6. Preview → Save