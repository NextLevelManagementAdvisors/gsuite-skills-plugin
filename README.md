# gsuite-skills — Claude Code Plugin

Google Workspace skills for Claude Code. Install this plugin to get Drive organization capabilities alongside the Google Workspace MCP.

## Skills included

### `/organize-drive`
Rule-based Google Drive file organizer. Three modes:
- **PLAN** — dry-run: shows every proposed move before touching anything
- **APPLY** — executes moves, creates folders, logs to audit Sheet
- **UNDO** — reverses any run from the audit log

Trigger: *"organize my Drive"*, *"sort Drive files"*, *"clean up my Drive"*, *"archive old Drive files"*

Requires the Google Workspace MCP (wired automatically on plugin install).

## Install

```bash
/plugins add https://github.com/NextLevelManagementAdvisors/gsuite-skills-plugin
```

## License

MIT
