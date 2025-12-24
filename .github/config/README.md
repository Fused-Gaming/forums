# Forum Configuration

This directory contains JSON configuration files for managing the FUSED GAMING community forums.

## Files

### `labels.json`
Defines all labels used across the forum for categorization and triage.

| Field | Description |
|-------|-------------|
| `name` | Label name |
| `description` | Label description |
| `color` | Hex color (without #) |
| `scope` | Where label applies: `public`, `internal`, or `both` |

### `categories.json`
Defines discussion categories for both public and internal forums.

| Field | Description |
|-------|-------------|
| `name` | Category display name |
| `emoji` | Category emoji icon |
| `description` | Category description |
| `slug` | URL-friendly identifier |
| `is_answerable` | Whether Q&A mode is enabled |
| `maintainers_only` | Whether only maintainers can post |
| `notify_discord` | Whether to send Discord notifications |
| `subcategories` | Array of subcategories |

### `notifications.json`
Configures Discord webhook notifications.

| Field | Description |
|-------|-------------|
| `webhook_secret` | GitHub secret name for webhook URL |
| `notify_on.discussion_created` | Categories that trigger notifications |
| `notify_on.labels` | Labels that trigger notifications |
| `embed_colors` | Discord embed colors per category |
| `channel_routing` | Discord channel routing hints |

## Secrets Required

Set these in GitHub repo Settings → Secrets → Actions:

- `DISCORD_WEBHOOK_PUBLIC` - Public forum notifications
- `DISCORD_WEBHOOK_INTERNAL` - Internal/escalation notifications

## Workflows

- `sync-labels.yml` - Syncs labels from config on push
- `discussion-notify.yml` - Sends Discord notifications based on config
