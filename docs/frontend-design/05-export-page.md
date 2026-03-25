# Export Page

## Purpose

Allow End Users to select a report, customize filters, and download an Excel file. This is the primary page End Users interact with daily — must be as simple as possible.

## Users

- **End User**: View and export configs they have access to
- **DA/Admin**: View and test all configs

## Interface

### Report List

- Search bar
- List of report cards, each containing:
  - Report name
  - Short description
  - **"Export"** button

### On "Export" Click → Opens Dialog

- Filter form (pre-filled with default values from config):
  - Only shows filters with `User Editable = Yes`
  - Each filter renders by type: Date picker, Select, Text input...
- "Reset to defaults" button
- **"Export file"** button

### Export States

- **Processing**: Progress bar + steps (Querying → Writing → Done)
- **Success**: File name + size + Download button
- **Failed**: Error message + Retry button

## Flow

1. End User opens Export page → sees list of reports they have access to
2. Clicks "Export" on desired report
3. Dialog opens → adjusts filters (or keeps defaults) → clicks "Export file"
4. Waits for progress → Downloads file