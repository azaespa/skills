# atlassian-gad — Atlassian skill with Team Mirror

Query and update **Jira** and **Confluence** from GitHub Copilot chat (direct REST API, no MCP), **plus**: every ticket you create in another Jira project is automatically mirrored into your team's project (default **GAD**), so your team board shows everything you're working on, without you creating the same ticket twice.

Based on the [`atlassian` skill by stevez-manulife](https://github.com/stevez-manulife/copilot-skills/tree/main/skills/atlassian).

## How the mirror works

```
You: /atlassian-gad create a Task in ABC: "Fix login timeout"

→ ABC-123  Fix login timeout                 (the real ticket)
→ GAD-1540 [ABC-123] Fix login timeout       (the mirror, on your team board)
            Description starts with "Mirrors ABC-123" (clickable)
```

| | |
|---|---|
| Mirrored on | `create issue` in any project other than your mirror project |
| Mirror summary | `[ORIGINAL-KEY] original summary` |
| Mirror description | `Mirrors ORIGINAL-KEY` + the original description |
| Issue type / assignee | Copied when possible (falls back to `Task` / unassigned) |
| Synced afterwards | **Summary** and **status**, one-way (original → mirror), whenever you update/move the original **through this skill** |
| Not synced | Description, comments, priority, etc. |
| Jira issue link | None by design: the `[KEY]` prefix + description reference connect them |

Status sync matches the same status name first. If the mirror project doesn't have that status (e.g. original is "In Review"), it falls back by category: to-do → `Backlog`, in-progress → `In Progress`, done → `Done` (configurable).

Edited the original directly in Jira? Run `/atlassian-gad sync ABC-123`.

## Install

Paste the matching prompt into your Copilot chat (Agent mode):

**GitHub Copilot Desktop App**
```
Follow the install instructions at
https://github.com/azaespa/skills/blob/main/skills/atlassian-gad/copilot-app/install.prompt.md
```

**GitHub Copilot CLI**
```
Follow the install instructions at
https://github.com/azaespa/skills/blob/main/skills/atlassian-gad/copilot-cli/install.prompt.md
```

**Claude Code**
```
Follow the install instructions at
https://github.com/azaespa/skills/blob/main/skills/atlassian-gad/claude-code/install.prompt.md
```

If you already use the original `atlassian` skill, the installer reuses your existing credentials and only asks for the mirror project key. Both skills can be installed side by side.

## Usage

```
/atlassian-gad                       # menu
/atlassian-gad create issue          # create + auto-mirror
/atlassian-gad move ABC-123          # transition + sync mirror status
/atlassian-gad update ABC-123        # edit + sync mirror summary/status
/atlassian-gad mirror ABC-123        # mirror an existing ticket
/atlassian-gad sync ABC-123          # re-sync summary + status
/atlassian-gad mirrors               # list your open tickets with no mirror yet
/atlassian-gad my issues             # everything else works like the original skill
```

Say "no mirror" when creating a ticket to skip mirroring for that one. Set `"mirror_enabled": false` in `~/.copilot/atlassian-config.json` to switch it off entirely.

## Config

Stored in `~/.copilot/atlassian-config.json` (shared with the original skill). Mirror-specific keys:

```json
{
  "mirror_project_key": "GAD",
  "mirror_enabled": true,
  "mirror_status_fallback": { "new": "Backlog", "indeterminate": "In Progress", "done": "Done" },
  "atlassian_gad_source_raw_base": "https://raw.githubusercontent.com/azaespa/skills/main"
}
```

## Requirements

- Atlassian Cloud (`*.atlassian.net`)
- API token: https://id.atlassian.com/manage-profile/security/api-tokens
- Permission to create issues in your mirror project
