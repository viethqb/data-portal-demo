# Export Config Management

## Purpose

Allow DA to create export configurations: select one or more Datasets + a Template, configure how each dataset's data is written into the template, set default filters, and assign access to End Users.

## Users

- **Admin**: CRUD all configs
- **DA**: CRUD own configs
- **End User**: No access (uses configs via the Export page)

## Interface

### List Page

- Search bar + "+ New Export Config" button
- Table:

| Column      | Description                    |
| ----------- | ------------------------------ |
| Name        | Config name                    |
| Datasets    | Source datasets (comma-separated) |
| Template    | Template used                  |
| Status      | Active / Draft                 |
| Last Export | Last export time               |
| Actions     | Edit, Delete                   |

### Create/Edit Form (single page, scrollable sections)

**Section 1 — Basic Info**
- Name, Description, Status (Active/Draft)

**Section 2 — Template**
- Template (dropdown)

**Section 3 — Datasets & Mapping**

A repeatable block — click "+ Add Dataset" to add more. Each block contains:

- Dataset (dropdown)
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

- Remove button (to delete this dataset block)

Each dataset can write to a different sheet in the same template, or to different row ranges within the same sheet.

**Section 4 — Filter Defaults**
- Combined table of filters from all selected datasets, each row:

| Dataset          | Filter     | Default Value       | User Editable |
| ---------------- | ---------- | ------------------- | ------------- |
| Doanh thu PB     | start_date | First day of month  | Yes           |
| Doanh thu PB     | end_date   | Today               | Yes           |
| Doanh thu PB     | department | ALL                 | Yes           |
| NV Chi Nhanh     | branch     | ALL                 | Yes           |

**Section 5 — Access Control**
- Radio: "All End Users" or "Specific Users/Groups"
- If Specific: searchable tag input to select users/groups

**"Preview Export"** + **"Save"** buttons

## Flow

1. DA clicks "+ New Export Config"
2. Fills in name → selects Template
3. Adds one or more Datasets, each with its own sheet target + column mapping
4. Sets filter defaults for all datasets + marks which filters End Users can edit
5. Selects users/groups with access
6. Preview → Save
