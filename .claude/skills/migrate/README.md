# Migration Skill

Intelligently migrate from previous AI assistant installations (clawdbot, OpenClaw, custom setups) to NanoClaw.

## Features

- **🔍 Smart Discovery**: Scans source installation to find credentials, templates, repositories, and scheduled jobs
- **🎯 Pattern Recognition**: Identifies services by content (Jira, Zoho, GitHub, etc.) without hardcoded lists
- **✅ Selective Migration**: User chooses exactly what to migrate
- **🧪 Validation**: Tests credentials where possible
- **📊 Comprehensive Reporting**: Detailed migration report with rollback instructions

## Usage

```bash
/migrate
```

The skill will:

1. **Discover** your previous installation
2. **Scan** for credentials, templates, repos, scheduled jobs, skills
3. **Present** findings with confidence levels
4. **Ask** what you want to migrate
5. **Execute** migration with validation
6. **Generate** detailed report

## What Gets Migrated

### Credentials
- Auto-detects: Jira, Zoho, GitHub, and generic OAuth/API keys
- Tests connections where possible
- Stores in `~/.claude/secrets/`

### Templates & Helpers
- Jira templates, Zoho scripts, custom helpers
- Transforms hardcoded paths
- Makes scripts executable
- Stores in `~/.claude/<category>/`

### Repositories
- Detects Git repos (Obsidian vaults, dotfiles, etc.)
- Offers clone from remote or copy local
- Verifies key files exist
- Stores in `~/.claude/<repo-name>/`

### Scheduled Jobs
- Finds crontab entries and job definitions
- Transforms to NanoClaw format
- Asks about context requirements (group vs isolated)
- Registers via `schedule_task` MCP

### Skills
- Detects custom skills
- Analyzes compatibility
- Migrates compatible ones
- Generates porting guide for incompatible skills

## Example

```
$ /migrate

🔍 Analyzing source installation...

Found clawdbot at: /root/clawd/
├── 5 credential files
├── 3 template directories
├── 1 Git repository (obsidian-vault)
├── 4 scheduled jobs
└── 2 custom skills

[Interactive selection follows]

📦 Migrating...
  ✅ Jira → ~/.claude/secrets/jira.json (tested, connected)
  ✅ Zoho → ~/.claude/secrets/zoho.json (tested, connected)
  ✅ jira-templates → ~/.claude/jira-templates/ (4 files)
  ✅ obsidian-vault → cloned from GitHub
  ✅ PR checker → scheduled every 30 min

📝 Migration complete!
   Report: ~/.claude/migration-report-2026-02-23.md
```

## Output

After migration, you'll have:

- `~/.claude/secrets/` - Migrated credentials
- `~/.claude/<category>/` - Templates and helpers
- `~/.claude/<repo>/` - Cloned/copied repositories
- `~/.claude/nanoclaw-paths.md` - Path reference document
- `~/.claude/migration-report-<date>.md` - Detailed migration report with:
  - What was migrated successfully
  - What needs manual review
  - Test results for credentials
  - Scheduled task IDs
  - Rollback instructions
  - Porting guides for incompatible skills

## Requirements

- Read access to source installation (may need sudo for root-owned files)
- Git installed (for cloning repositories)
- Optional: `gh` CLI for GitHub operations

## Safety

- Non-destructive: never modifies source files
- All migrations copy/clone, never move
- User controls what gets migrated
- Comprehensive audit trail
- Rollback instructions provided

## Contributing

Found a service pattern that should be auto-detected? Submit a PR to improve the pattern matching logic in the SKILL.md.
