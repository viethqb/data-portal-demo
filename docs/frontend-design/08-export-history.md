# Export History

## Purpose

View history of all exports (manual + scheduler). Allows re-downloading files and viewing errors for failed exports.

## Users

- **Admin**: View all exports
- **DA**: View exports from configs they manage
- **End User**: View own exports only

## Interface

### List Page

- Filters: Search, Export Config (dropdown), Status (dropdown), Source (dropdown), Date range
- Table:

| Column  | Description                 |
| ------- | --------------------------- |
| ID      | e.g. #EXP-1247              |
| Config  | Export config name          |
| Source  | Manual / Scheduler          |
| User    | Who triggered it            |
| Status  | Success / Failed            |
| File    | File size (if success)      |
| Time    | Export timestamp             |
| Actions | Download, View Detail       |

### Detail Page (click View Detail)

**On Success:**
- Info: Config, Dataset, Template, User, Duration
- Filter values used
- Result: row count, file name, file size
- Download + Re-export buttons

**On Failed:**
- Basic info (same as above)
- Error: type + message
- Log viewer (monospace, scrollable)
- Retry button

## Flow

1. User opens Export History → filters as needed
2. Clicks row → views detail
3. Success → Download file
4. Failed → views log → Retry