# Template Management

## Purpose

Allow Admin/DA to upload and manage Excel template files. Templates are stored in MinIO.

## Users

- **Admin**: CRUD all templates
- **DA**: CRUD own templates
- **End User**: No access

## Interface

### List Page

- Search bar + "+ Upload Template" button
- Table:

| Column   | Description                         |
| -------- | ----------------------------------- |
| Name     | Template name                       |
| File     | Original filename + size            |
| Sheets   | Number of sheets                    |
| Version  | Current version                     |
| Used In  | Number of export configs using it   |
| Uploaded | Upload date                         |
| Actions  | Download, Upload New Version, Delete |

### Upload Dialog

- Drag & drop area (.xlsx, .xls, max 10MB)
- After file selection: shows filename + size
- Fields: Name, Description
- "Upload" button

### Detail Page (click template name)

- Basic info (name, description, file info)
- Version history table: Version, Date, Notes, Actions (Download / Revert)
- "Upload New Version" button

## Flow

1. DA clicks "+ Upload Template"
2. Drags & drops Excel file → fills in name, description → Upload
3. To update → clicks "Upload New Version" → selects new file → version auto-increments