# LLM Discord Alerts Setup Guide

## Overview

This guide shows you how to set up Discord notifications for Claude (or any LLM assistant) to automatically post updates when completing tasks, closing issues, deploying infrastructure, or finishing any work.

## Benefits

- **Real-time updates**: Get notified immediately when Claude completes tasks
- **Team visibility**: Keep your team informed of progress without manual updates
- **Audit trail**: Permanent record of completed work in Discord
- **Context**: Each notification includes git context, commit info, and issue links

## Prerequisites

- Discord server with webhook permissions
- Git repository for your project
- Bash/shell access

## Step 1: Create Discord Webhook

1. Open your Discord server
2. Go to **Server Settings → Integrations → Webhooks**
3. Click **"New Webhook"** or **"Create Webhook"**
4. Configure the webhook:
   - **Name**: `Claude Task Updates` (or your preferred name)
   - **Channel**: Select where notifications should appear
5. Click **"Copy Webhook URL"**
6. Save the URL for the next step

## Step 2: Download the Notification Script

Save this script as `.discord-notify.sh` in your project root:

```bash
#!/bin/bash
# Discord notification script for LLM task completions

set -e

# Load webhook URL from config or environment
if [ -f .discord-webhook ]; then
    DISCORD_WEBHOOK=$(cat .discord-webhook)
elif [ -n "$DISCORD_WEBHOOK" ]; then
    DISCORD_WEBHOOK="$DISCORD_WEBHOOK"
else
    echo "Error: Discord webhook not configured"
    echo "Create .discord-webhook file with your webhook URL or set DISCORD_WEBHOOK env var"
    exit 1
fi

# Get task details from arguments
TASK_TITLE="${1:-Task completed}"
TASK_DESCRIPTION="${2:-}"
TASK_STATUS="${3:-✅}"
ISSUE_NUMBER="${4:-}"

# Get git context
BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
LAST_COMMIT=$(git log -1 --pretty=format:"%h - %s" 2>/dev/null || echo "No recent commit")
REPO_URL=$(git remote get-url origin 2>/dev/null | sed 's/\.git$//' || echo "")

# Build embed description
EMBED_DESC="**Branch:** \`${BRANCH}\`\n**Last Commit:** ${LAST_COMMIT}"

if [ -n "$TASK_DESCRIPTION" ]; then
    EMBED_DESC="${EMBED_DESC}\n\n${TASK_DESCRIPTION}"
fi

# Add issue link if provided
if [ -n "$ISSUE_NUMBER" ]; then
    EMBED_DESC="${EMBED_DESC}\n\n[View Issue #${ISSUE_NUMBER}](${REPO_URL}/issues/${ISSUE_NUMBER})"
fi

# Determine color based on status
COLOR=3066993  # Green
if [[ "$TASK_STATUS" == "⚠️" ]]; then
    COLOR=16776960  # Yellow
elif [[ "$TASK_STATUS" == "❌" ]]; then
    COLOR=15158332  # Red
fi

# Send notification
curl -X POST "$DISCORD_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d "{
    \"content\": \"${TASK_STATUS} **Claude Task Update**\",
    \"embeds\": [{
      \"title\": \"${TASK_TITLE}\",
      \"description\": \"${EMBED_DESC}\",
      \"color\": ${COLOR},
      \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)\"
    }]
  }" \
  2>/dev/null || echo "Failed to send Discord notification"

echo "✓ Discord notification sent"
```

## Step 3: Make Script Executable

```bash
chmod +x .discord-notify.sh
```

## Step 4: Configure Webhook URL

Choose one of these methods:

### Option A: Store in File (Recommended)

```bash
# Save webhook URL to .discord-webhook file
echo "YOUR_DISCORD_WEBHOOK_URL" > .discord-webhook
```

### Option B: Environment Variable

```bash
# Add to your shell profile (.bashrc, .zshrc, etc.)
export DISCORD_WEBHOOK="YOUR_DISCORD_WEBHOOK_URL"
```

## Step 5: Add to .gitignore

Prevent the webhook URL from being committed:

```bash
# Add to your .gitignore
echo ".discord-webhook" >> .gitignore
```

**Optional**: Also gitignore the script if you want it local-only:
```bash
echo ".discord-notify.sh" >> .gitignore
```

## Step 6: Test the Setup

Send a test notification:

```bash
./.discord-notify.sh "Test Notification" "This is a test of the Discord notification system" "✅"
```

You should see a message appear in your Discord channel!

## Usage

### Basic Notification

```bash
./.discord-notify.sh "Task Title" "Task description" "✅"
```

### With Issue Number

```bash
./.discord-notify.sh "Completed Issue #123" "Added new feature" "✅" "123"
```

### Status Indicators

- `✅` - Success (green)
- `⚠️` - Warning (yellow)
- `❌` - Error (red)

### Example Use Cases

**Issue completion:**
```bash
./.discord-notify.sh "Closed Issue #42" "Fixed authentication bug" "✅" "42"
```

**Deployment:**
```bash
./.discord-notify.sh "Deployed to Production" "Version 2.5.0 deployed successfully" "✅"
```

**Build failure:**
```bash
./.discord-notify.sh "Build Failed" "Tests failing on main branch" "❌"
```

**Infrastructure update:**
```bash
./.discord-notify.sh "AWS Infrastructure Updated" "Deployed new Lambda functions" "✅"
```

## Integration with Claude

When working with Claude, instruct it to use the notification script:

**Example prompt:**
> "When you complete this task, notify Discord using the `.discord-notify.sh` script with an appropriate message."

Claude will automatically call the script after completing work, for example:

```bash
./.discord-notify.sh "Completed Issue #129" "Added integration tests, updated CHANGELOG, bumped version to 2.5.0" "✅" "129"
```

## Integration with CI/CD

### GitHub Actions

Add to your workflow:

```yaml
- name: Notify Discord
  if: success()
  run: |
    echo "${{ secrets.DISCORD_WEBHOOK }}" > .discord-webhook
    ./.discord-notify.sh "CI Pipeline Passed" "All tests passed on main" "✅"
  env:
    DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
```

### Git Hooks

Add to `.git/hooks/post-commit`:

```bash
#!/bin/bash
if [ -f .discord-notify.sh ]; then
    ./.discord-notify.sh "New Commit" "$(git log -1 --pretty=%B)" "✅"
fi
```

## Troubleshooting

### Script says "Discord webhook not configured"

**Solution**: Create `.discord-webhook` file or set `DISCORD_WEBHOOK` environment variable.

### No message appears in Discord

**Checks:**
1. Verify webhook URL is correct
2. Check the webhook hasn't been deleted in Discord
3. Run script with `-x` flag for debugging: `bash -x ./.discord-notify.sh "Test" "Test" "✅"`

### Permission denied

**Solution**: Make script executable: `chmod +x .discord-notify.sh`

### Webhook URL accidentally committed

**Solution:**
1. Remove from git: `git rm --cached .discord-webhook`
2. Add to `.gitignore`
3. Regenerate webhook URL in Discord (delete old one, create new)
4. Update local `.discord-webhook` file with new URL

## Security Best Practices

1. **Never commit webhook URLs** to version control
2. **Use `.gitignore`** to prevent accidental commits
3. **Regenerate webhooks** if accidentally exposed
4. **Limit webhook permissions** to specific channels only
5. **Consider using environment variables** in production environments

## Advanced Configuration

### Custom Colors

Edit the `COLOR` variable in the script:

```bash
COLOR=3066993   # Green (default success)
COLOR=16776960  # Yellow (warnings)
COLOR=15158332  # Red (errors)
COLOR=3447003   # Blue (info)
COLOR=10181046  # Purple (special events)
```

### Multiple Webhooks

For different channels:

```bash
# Development notifications
echo "DEV_WEBHOOK_URL" > .discord-webhook-dev

# Production notifications
echo "PROD_WEBHOOK_URL" > .discord-webhook-prod

# Use specific webhook
DISCORD_WEBHOOK=$(cat .discord-webhook-dev) ./.discord-notify.sh "Dev Update" "..." "✅"
```

### Rich Embeds

Modify the JSON payload in the script to add:
- Thumbnails
- Images
- Multiple fields
- Footers
- Author information

See [Discord Embed Documentation](https://discord.com/developers/docs/resources/channel#embed-object) for details.

## Support

For issues or questions:
- Check the [Discord API Documentation](https://discord.com/developers/docs/intro)
- Review the [Discord Webhook Guide](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)
- Test webhook with [Discord Webhook Tester](https://discohook.org/)

## Example Notification

When properly configured, you'll see messages like this in Discord:

```
✅ Claude Task Update

Completed Issue #129
Branch: feature/integration-tests
Last Commit: abc1234 - Add integration tests for remote execution

Added integration tests, updated CHANGELOG, bumped version to 2.5.0

View Issue #129
```

---

**Last Updated**: 2025-11-22
**Version**: 1.0
