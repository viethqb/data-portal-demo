# Dashboard

## Purpose

Landing page after login. Shows system overview stats and recent export activity.

## Users

- **Admin/DA**: Full system stats + recent exports
- **End User**: Only their own recent exports

## Interface

### Shared Layout (all pages)

- **Top navigation bar**: Logo on the left, horizontal menu (Dashboard, Datasets, Templates, Export Configs, Export, Schedulers, Users, History) — visibility based on role, user avatar + logout on the right
- No sidebar — horizontal nav is sufficient for 8 items

### Dashboard Content

1. **4 stat cards** (Admin/DA only): Datasets, Templates, Exports, Schedulers — each card shows a number + label
2. **"Recent Exports" table**: Last 10 entries

| Column | Description            |
| ------ | ---------------------- |
| Config | Export config name     |
| User   | Who triggered it       |
| Status | Success / Failed       |
| Time   | Timestamp              |

## Flow

1. User logs in → lands on Dashboard
2. Click table row → navigate to Export History (detail)
3. Click stat card → navigate to corresponding management page