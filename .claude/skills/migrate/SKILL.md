---
name: migrate
description: Discover and migrate configuration, credentials, and data from previous assistant installations (clawdbot, OpenClaw, custom setups). Uses intelligent discovery to find resources without hardcoded assumptions.
---

# Assistant Migration

Migrate from previous AI assistant installations to NanoClaw using intelligent discovery. This skill scans for credentials, templates, repositories, scheduled jobs, and skills, then guides you through selective migration.

## Principles

- **Discovery over assumption**: Scan the source environment to find what exists
- **Pattern recognition**: Identify services by content, not just filenames
- **User control**: Present findings and let user choose what to migrate
- **Validation**: Test credentials where possible
- **Documentation**: Generate comprehensive migration report

## Phase 1: Source Discovery

### 1.1 Detect Source Installation

Check common installation locations:

```bash
# Check standard paths
for path in /root/clawd /root/.clawdbot ~/.openclaw ~/.clawdbot ~/clawd; do
  if [ -d "$path" ]; then
    echo "Found: $path"
  fi
done
```

If none found, use `AskUserQuestion`:

```
Question: Where is your previous assistant installation located?
Options:
  - /root/clawd (standard clawdbot)
  - /root/.clawdbot (alternative clawdbot)
  - ~/.openclaw (OpenClaw)
  - Custom path (I'll specify)
```

Store the source path as `SOURCE_PATH` for subsequent steps.

### 1.2 Discover Credentials

Find all credential files:

```bash
# Find JSON files in credential directories
find $SOURCE_PATH -type f -name "*.json" \
  \( -path "*/secrets/*" -o -path "*/credentials/*" -o -path "*/.config/*" \) \
  2>/dev/null
```

For each file found, use the `Read` tool to parse and categorize:

**Pattern matching for service identification:**

```
File contains "atlassian.net" in url field → Jira (high confidence, testable)
File contains "zohoapis.com" or "zoho.com" → Zoho (high confidence, testable)
File has token starting with "ghp_" or "github_pat_" → GitHub PAT (high confidence, testable)
File has "client_id" + "refresh_token" → OAuth service (medium confidence, not testable)
File has keys matching "token|key|secret|password" → Unknown credential (low confidence, not testable)
```

Build an inventory:

```json
{
  "credentials": [
    {
      "path": "/root/.clawdbot/secrets/jira.json",
      "service": "Jira",
      "confidence": "high",
      "testable": true,
      "keys": ["url", "email", "token", "project"]
    },
    {
      "path": "/root/.clawdbot/secrets/zoho.json",
      "service": "Zoho Books",
      "confidence": "high",
      "testable": true,
      "keys": ["client_id", "refresh_token", "api_domain"]
    }
  ]
}
```

### 1.3 Discover Templates & Helpers

Find template and helper directories:

```bash
# Find template directories
find $SOURCE_PATH -type d \( -name "*template*" -o -name "*templates" \) 2>/dev/null

# Find script directories
find $SOURCE_PATH -type d \( -name "*script*" -o -name "*helper*" \) 2>/dev/null
```

For each directory, list files and infer purpose:

```bash
# Count files and estimate purpose
ls $DIR/*.md $DIR/*.sh $DIR/*.py 2>/dev/null | wc -l
# Sample first file to understand content
head -20 $DIR/$(ls $DIR | head -1)
```

Categorize by naming patterns:
- `jira-*` → Jira templates
- `zoho-*` → Zoho helpers
- `notion-*` → Notion templates
- Generic → Unknown helpers

### 1.4 Discover Repositories

Find Git repositories:

```bash
# Find all .git directories
find $SOURCE_PATH -type d -name ".git" 2>/dev/null | sed 's|/.git||'
```

For each repository, gather metadata:

```bash
cd $REPO_PATH
# Get remote URL
git remote get-url origin 2>/dev/null
# Check if it's a known service (GitHub, GitLab, Bitbucket)
# Check current branch
git branch --show-current
# Count commits
git log --oneline | wc -l
```

Identify special repositories:
- Contains "obsidian" or "vault" → Knowledge base
- Contains "notes" or "journal" → Daily notes
- Contains "dotfiles" → Configuration repo

### 1.5 Discover Scheduled Jobs

Check multiple sources:

```bash
# User crontab
crontab -l 2>/dev/null

# System timers (if accessible)
systemctl list-timers --all 2>/dev/null | grep -E 'claw|assistant|bot'

# Job definition files
find $SOURCE_PATH -name "*cron*" -o -name "*schedule*" -o -name "*job*" 2>/dev/null
```

Parse cron format:

```
*/30 * * * * → Every 30 minutes
0 9 * * * → Daily at 9am
0 9 * * 1-5 → Weekdays at 9am
0 0 1 * * → Monthly on 1st
```

For job definition files (JSON/YAML), parse structure to extract:
- Schedule (cron expression, interval, one-time)
- Command or prompt
- Context requirements (needs history or isolated)

### 1.6 Discover Custom Skills

Find skills directory:

```bash
find $SOURCE_PATH -type d -name "skills" 2>/dev/null
```

List custom skills (exclude default ones):

```bash
# Default skills to exclude
DEFAULTS="setup,debug,customize,add-telegram,add-discord"

# Find custom skills
ls $SKILLS_DIR | grep -vE "^(setup|debug|customize|add-telegram|add-discord)$"
```

For each custom skill, analyze compatibility:

```bash
# Check SKILL.md for tool usage
grep -E "(Bash|Read|Write|Edit|WebSearch|WebFetch)" $SKILL_DIR/SKILL.md

# Flag incompatible tools
grep -E "(Gateway|Sessions|cron tool|message tool)" $SKILL_DIR/SKILL.md
```

## Phase 2: Present Findings

Use `AskUserQuestion` to show discoveries and get selection.

### 2.1 Credentials Selection

Present discovered credentials grouped by confidence:

```
Found 5 credential files:

High Confidence (tested on migration):
├── Jira - awesomemotive.atlassian.net
├── Zoho Books - OAuth credentials
└── GitHub - Personal Access Token

Medium Confidence (OAuth-like, not testable):
└── OpenAI API - api.openai.com

Low Confidence (unknown):
└── credentials.json - has token/key fields

Which would you like to migrate?
```

Options:
- All high-confidence (recommended)
- Select individually
- All credentials
- Skip credential migration

If "Select individually", ask for each service.

### 2.2 Templates & Helpers Selection

Present discovered directories:

```
Found 3 template/helper directories:

├── jira-templates/ (4 files: bug.md, task.md, story.md, naming-convention.md)
├── zoho-helpers/ (2 scripts: zoho-books.sh, zoho-record.py)
└── notion-templates/ (5 markdown files)

Which would you like to migrate?
```

Options:
- All directories
- Select by directory
- Skip templates

### 2.3 Repositories Selection

Present discovered repositories:

```
Found 2 Git repositories:

├── obsidian-vault/ (GitHub: AhmedTheGeek/obsidian-vault, 245 commits)
└── daily-notes/ (No remote, 147 markdown files, 2.3MB)

How should we migrate these?
```

For each repo, offer options:
- Clone from Git remote (if available)
- Copy local directory
- Skip this repository

### 2.4 Scheduled Jobs Selection

Present discovered jobs with context:

```
Found 4 scheduled jobs:

├── Every 30 min - "Check for new PRs assigned to me" (cron)
├── Daily 9am - "Daily standup summary" (cron)
├── Monthly 1st - "Zoho Books reminder" (cron)
└── One-time - "Review WPChat PR" (already past due, paused)

For each job that runs:
- Does it need conversation history? (context_mode: group)
- Or is it standalone? (context_mode: isolated)
```

For each job, ask about context_mode.

### 2.5 Skills Selection

Present custom skills with compatibility:

```
Found 2 custom skills:

Compatible (ready to use):
└── jira-create-ticket (uses Read, Write, Bash)

Needs Porting (uses unavailable tools):
└── telegram-notify (uses 'message' tool, needs rewrite for nanoclaw)

Which would you like to migrate?
```

Options:
- All compatible skills
- Skip skills (I'll add them manually later)

## Phase 3: Migration Execution

### 3.1 Migrate Credentials

For each selected credential file:

```bash
# Create secrets directory
mkdir -p ~/.claude/secrets
chmod 700 ~/.claude/secrets

# Determine service name (lowercase, hyphenated)
# jira.json, zoho-books.json, github.json, openai.json, unknown-1.json

# Copy file
cp $SOURCE_FILE ~/.claude/secrets/$SERVICE_NAME.json
chmod 600 ~/.claude/secrets/$SERVICE_NAME.json
```

**Test if possible:**

```bash
# Jira
curl -s -u "$(jq -r .email ~/.claude/secrets/jira.json):$(jq -r .token ~/.claude/secrets/jira.json)" \
  "$(jq -r .url ~/.claude/secrets/jira.json)/rest/api/3/myself" | jq -r .displayName

# GitHub (requires gh CLI)
export GITHUB_TOKEN=$(jq -r .token ~/.claude/secrets/github.json)
gh auth status

# Zoho (OAuth - test with simple API call)
ACCESS_TOKEN=$(jq -r .access_token ~/.claude/secrets/zoho.json)
curl -s -H "Authorization: Zoho-oauthtoken $ACCESS_TOKEN" \
  "https://www.zohoapis.com/books/v3/settings/preferences?organization_id=..." | jq -r .message
```

Record result:
- ✅ Migrated and tested successfully
- ⚠️ Migrated (not testable, manual verification needed)
- ❌ Migration failed

### 3.2 Migrate Templates & Helpers

For each selected directory:

```bash
# Determine category from directory name
# jira-templates → ~/.claude/jira-templates
# zoho-helpers → ~/.claude/zoho-helpers
# Generic fallback → ~/.claude/<dirname>

# Copy directory
mkdir -p ~/.claude/$CATEGORY
cp -r $SOURCE_DIR/* ~/.claude/$CATEGORY/

# Make scripts executable
find ~/.claude/$CATEGORY -type f \( -name "*.sh" -o -name "*.py" \) -exec chmod +x {} \;

# Check for hardcoded paths and offer to transform
if grep -r "/root/clawd\|/root/.clawdbot" ~/.claude/$CATEGORY 2>/dev/null; then
  # Use AskUserQuestion
  # "Found hardcoded paths. Transform /root/clawd → ~/.claude?"
  # If yes: find ~/.claude/$CATEGORY -type f -exec sed -i 's|/root/clawd|~/.claude|g' {} \;
fi
```

### 3.3 Migrate Repositories

For repositories with Git remotes:

**Clone fresh from remote:**

```bash
# Check if SSH keys exist
if [ ! -f ~/.ssh/id_rsa ] && [ ! -f ~/.ssh/id_ed25519 ]; then
  # Offer HTTPS fallback with token
  # If GitHub and we have token: Use https://$TOKEN@github.com/...
fi

git clone $REMOTE_URL ~/.claude/$REPO_NAME

# Verify key files
# For obsidian: Check for Todo.md or daily notes
# For general repos: Just confirm .git exists
```

For local-only repositories:

```bash
# Copy with rsync to preserve .git
rsync -av $SOURCE_REPO/ ~/.claude/$REPO_NAME/
```

### 3.4 Migrate Scheduled Jobs

For each selected job:

Transform cron expression to nanoclaw format, then use `schedule_task` MCP:

```typescript
schedule_task({
  prompt: "Check for new PRs assigned to me and notify if found",
  schedule_type: "cron",
  schedule_value: "*/30 * * * *",
  context_mode: "group"  // or "isolated" based on user selection
})
```

For jobs that are past due or one-time, create them as paused:

```typescript
// Create task, then immediately pause it
// User can review and resume manually
```

Record task IDs for the report.

### 3.5 Migrate Skills

For compatible skills:

```bash
# Copy skill directory
cp -r $SOURCE_SKILLS/$SKILL_NAME ~/.claude/skills/

# Skills are auto-discovered, no registration needed
```

For incompatible skills, generate porting guide in the migration report.

### 3.6 Update Path Reference

Create or update `~/.claude/nanoclaw-paths.md`:

```markdown
# NanoClaw Paths - Migrated from $SOURCE_TYPE

## Migration Date
$DATE

## Credentials
- Jira: ~/.claude/secrets/jira.json ✅
- Zoho: ~/.claude/secrets/zoho.json ✅
- GitHub: gh CLI authenticated ✅

## Templates & Helpers
- Jira templates: ~/.claude/jira-templates/ (4 files)
- Zoho helpers: ~/.claude/zoho-helpers/ (2 scripts)

## Repositories
- Obsidian vault: ~/.claude/obsidian-vault/ (cloned from GitHub)
- Todo file: ~/.claude/obsidian-vault/AM/👨💻 Todo.md

## Scheduled Tasks
- PR checker: Every 30 min (task ID: abc-123, context_mode: group)
- Daily standup: Weekdays 9am (task ID: def-456, context_mode: isolated)

## Skills
- jira-create-ticket: ~/.claude/skills/jira-create-ticket/ ✅

## Old Paths → New Paths
- /root/.clawdbot/secrets/ → ~/.claude/secrets/
- /root/clawd/jira-templates/ → ~/.claude/jira-templates/
- /root/clawd/obsidian-vault/ → ~/.claude/obsidian-vault/
```

## Phase 4: Verification & Report

### 4.1 Test Migrated Resources

Run tests on each migrated component:

**Credentials:**
```bash
# Re-run test commands from 3.1
# Record pass/fail for each
```

**Repositories:**
```bash
# Verify key files exist
test -f ~/.claude/obsidian-vault/AM/👨💻\ Todo.md && echo "✅ Todo file found"

# Check git remote
cd ~/.claude/obsidian-vault && git remote -v
```

**Scheduled tasks:**
```bash
# Use list_tasks MCP to verify they're registered
list_tasks
```

**Skills:**
```bash
# Verify skill files exist
test -f ~/.claude/skills/$SKILL_NAME/SKILL.md && echo "✅ Skill installed"
```

### 4.2 Generate Migration Report

Create comprehensive report at `~/.claude/migration-report-$DATE.md`:

```markdown
# Migration Report - $DATE

## Source
Migrated from: $SOURCE_TYPE at $SOURCE_PATH

## Summary

✅ Successfully Migrated:
- 3 credentials (Jira, Zoho, GitHub)
- 2 template directories
- 1 repository (obsidian-vault)
- 2 scheduled tasks
- 1 skill

⚠️ Manual Review Required:
- 2 credentials (not testable, verify manually)
- 1 scheduled task (paused, was past due)

❌ Not Migrated:
- 1 skill (incompatible, needs porting)

## Credentials

### ✅ Jira
- Location: ~/.claude/secrets/jira.json
- Status: Tested successfully, connected as Ahmed Hussein
- URL: awesomemotive.atlassian.net

### ✅ Zoho Books
- Location: ~/.claude/secrets/zoho.json
- Status: Tested successfully, OAuth tokens valid
- Organization: Geekology FZ - LLC

### ✅ GitHub
- Status: gh CLI authenticated as AhmedTheGeek
- Scopes: repo, read:org, workflow

### ⚠️ OpenAI API
- Location: ~/.claude/secrets/openai.json
- Status: Migrated, not tested (manual verification needed)

## Templates & Helpers

### jira-templates/
- Location: ~/.claude/jira-templates/
- Files: bug.md, task.md, story.md, naming-convention.md
- Transformed paths: /root/clawd → ~/.claude

### zoho-helpers/
- Location: ~/.claude/zoho-helpers/
- Files: zoho-books.sh, zoho-record.py (made executable)

## Repositories

### obsidian-vault
- Location: ~/.claude/obsidian-vault/
- Source: Cloned from git@github.com:AhmedTheGeek/obsidian-vault.git
- Todo file: ~/.claude/obsidian-vault/AM/👨💻 Todo.md ✅

## Scheduled Tasks

### PR Review Checker
- Schedule: Every 30 minutes (cron: */30 * * * *)
- Context: group (has conversation history)
- Task ID: abc-123
- Status: Active

### Daily Standup
- Schedule: Weekdays at 9am (cron: 0 9 * * 1-5)
- Context: isolated (standalone)
- Task ID: def-456
- Status: Active

### Monthly Zoho Reminder
- Schedule: 1st of month at 7am (cron: 0 7 1 * *)
- Context: isolated
- Task ID: ghi-789
- Status: Paused (review and resume manually)

## Skills

### ✅ jira-create-ticket
- Location: ~/.claude/skills/jira-create-ticket/
- Compatibility: Full (uses Read, Write, Bash)
- Status: Ready to use

### ❌ telegram-notify
- Status: Not migrated (uses incompatible tools)
- Porting guide: See section below

## Porting Guide for Incompatible Skills

### telegram-notify
This skill uses the `message` tool which is not available in nanoclaw.

**What it did:**
- Sent notifications to Telegram via message tool

**How to port:**
1. Install Telegram integration: `/add-telegram`
2. Replace `message` tool calls with `send_message` MCP
3. Update channel JID format to `tg:$CHAT_ID`

**Original SKILL.md:** [preserved at ~/.claude/skills-backup/telegram-notify-original.md]

## Next Steps

1. ✅ All credentials tested and working
2. ⚠️ Review paused scheduled task: `ghi-789` - Resume if needed
3. ⚠️ Test OpenAI credential manually
4. 📖 Path reference documented at: ~/.claude/nanoclaw-paths.md
5. 🎉 Migration complete!

## Rollback Instructions

If you need to revert:

```bash
# Remove migrated credentials
rm -rf ~/.claude/secrets/

# Remove migrated templates
rm -rf ~/.claude/jira-templates/ ~/.claude/zoho-helpers/

# Remove migrated repos
rm -rf ~/.claude/obsidian-vault/

# Cancel scheduled tasks
cancel_task abc-123
cancel_task def-456
cancel_task ghi-789

# Remove skills
rm -rf ~/.claude/skills/jira-create-ticket/
```

---

Report generated: $TIMESTAMP
```

### 4.3 Present Summary to User

Show concise summary:

```
🎉 Migration Complete!

Migrated successfully:
✅ 3 credentials (all tested, working)
✅ 2 template directories
✅ 1 repository (obsidian-vault from GitHub)
✅ 2 active scheduled tasks
✅ 1 skill

Needs attention:
⚠️ 1 credential (manual verification needed)
⚠️ 1 scheduled task (paused, review required)

Not migrated:
❌ 1 skill (incompatible, porting guide available)

Full report: ~/.claude/migration-report-$DATE.md
Path reference: ~/.claude/nanoclaw-paths.md
```

## Usage Examples

### Example 1: Full Migration

```
User: /migrate
Agent: [Scans /root/clawd]
Agent: Found 5 credentials, 3 template directories, 1 repo, 4 scheduled jobs. Show details?
User: Yes
Agent: [Presents detailed findings]
Agent: [User selects what to migrate]
Agent: [Executes migration with progress updates]
Agent: Migration complete! Report at ~/.claude/migration-report-2026-02-23.md
```

### Example 2: Partial Migration (Credentials Only)

```
User: /migrate
Agent: [Discovers resources]
Agent: Which would you like to migrate?
User: Just credentials
Agent: [Migrates credentials, tests them]
Agent: 3 credentials migrated successfully
```

### Example 3: Custom Source Path

```
User: /migrate
Agent: Where is your previous installation?
User: Custom path → /home/ahmed/old-assistant
Agent: [Scans custom path]
Agent: [Continues with migration]
```

## Troubleshooting

### Permission Denied on Source Files

If source files are owned by root:

```bash
# Use sudo to read, then write to accessible location
sudo cat /root/.clawdbot/secrets/jira.json > /tmp/jira.json
cp /tmp/jira.json ~/.claude/secrets/jira.json
rm /tmp/jira.json
```

Or ask user to provide file contents directly via AskUserQuestion.

### Git Clone Fails (SSH)

If SSH keys are missing:

```
Use AskUserQuestion: "Git clone requires SSH keys. Options:"
- Set up SSH keys now (guide me through it)
- Use HTTPS with token instead (I'll provide GitHub token)
- Skip repository migration (I'll do it manually)
```

### Credential Test Fails

If a credential test fails but file exists:

```
Report as: ⚠️ Migrated (test failed, may need manual reconfiguration)
Include error message in report
Don't block migration - continue with other items
```

## Notes

- This skill is non-destructive: never modifies source files
- All migrations copy/clone, never move
- User has full control over what gets migrated
- Comprehensive reporting for audit trail
- Rollback instructions provided in report
