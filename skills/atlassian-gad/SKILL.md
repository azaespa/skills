---
name: atlassian-gad
description: Query and update Jira issues and Confluence pages using the Atlassian Cloud REST API, and automatically mirror every ticket you create in another Jira project into your team's project (default GAD) so the team board always reflects your work. Keeps the mirror's summary and status in sync when you update the original through this skill. No MCP required. Uses ~/.copilot/atlassian-config.json.
disable-model-invocation: true
allowed-tools: PowerShell(*) Bash(curl:*) Bash(python:*) Bash(python3:*)
---

# Atlassian Jira & Confluence Skill — with Team Mirror

> **Scope: Atlassian Cloud only** (`*.atlassian.net`). Not compatible with Data Center or Server.

Direct REST API access to Jira and Confluence — no MCP server, no proxies. Uses an API token via HTTP Basic auth.

This is a variant of the original [`atlassian` skill](https://github.com/stevez-manulife/copilot-skills/tree/main/skills/atlassian). It adds the **Team Mirror** behaviour described below; everything else works the same.

## Menu

When the user invokes `/atlassian-gad` without a specific request, show:

```
Atlassian (Team Mirror) — What would you like to do?

JIRA
  1. my issues          — list my open Jira issues
  2. issue <KEY-123>    — get details of a specific issue
  3. create issue       — create a new Jira issue (auto-mirrored to <MIRROR>)
  4. update <KEY-123>   — update summary, description, or status (mirror synced)
  5. comment <KEY-123>  — add a comment to an issue
  6. move <KEY-123>     — transition issue to a new status (mirror synced)
  7. search <jql>       — search issues with custom JQL

TEAM MIRROR
  8. mirror <KEY-123>   — create the <MIRROR> mirror for an existing ticket
  9. sync <KEY-123>     — re-sync summary + status from original to its mirror
 10. mirrors            — list my open tickets outside <MIRROR> that have no mirror yet

CONFLUENCE
 11. search confluence <query> — search Confluence pages
 12. page <pageId>      — get a specific page by ID
 13. create page        — create a new Confluence page

SETUP
 14. setup              — configure credentials + mirror project (first-time or reset)
 15. status             — show current config (site, cloudId, mirror project)
 16. update             — download the latest version of this skill

Just type a number or describe what you want naturally.
```

Replace `<MIRROR>` with `mirror_project_key` from the config (default `GAD`).

## Config

File location (shared with the original `atlassian` skill — both can be installed side by side):
- **Windows:** `%USERPROFILE%\.copilot\atlassian-config.json`
- **Mac/Linux:** `~/.copilot/atlassian-config.json`

Schema:

```json
{
  "email":     "you@yourcompany.com",
  "api_token": "ATATT3x...",
  "site":      "your-org.atlassian.net",
  "token":     "Basic <base64(email:api_token)>",
  "cloudId":   "<uuid>",

  "mirror_project_key": "GAD",
  "mirror_enabled": true,
  "mirror_status_fallback": { "new": "Backlog", "indeterminate": "In Progress", "done": "Done" },
  "atlassian_gad_source_raw_base": "https://raw.githubusercontent.com/<owner>/<repo>/<branch>"
}
```

- `email`, `api_token`, `site` are source-of-truth. `token` and `cloudId` are derived and cached.
- `mirror_project_key` — your team's Jira project. Missing ⇒ treat as `GAD`.
- `mirror_enabled` — set `false` to turn mirroring off without uninstalling. Missing ⇒ `true`.
- `mirror_status_fallback` — which mirror status to use when the original's status name doesn't exist in the mirror project, keyed by Jira status category (`new` / `indeterminate` / `done`). Missing ⇒ use the defaults shown above.
- `atlassian_gad_source_raw_base` — where `/atlassian-gad update` downloads from. Deliberately a different key from the original skill's `source_repo_raw_base`, so updating one skill never overwrites the other.

**Never print `api_token` or `token` back to the user.**

### First-time setup

If config is missing or the user types `setup`, ask in chat for `email`, `api_token`, `site`, and the **mirror project key** (suggest `GAD`). Then run one PowerShell/bash script to fetch `cloudId`, compute `token`, **merge** these fields into the existing config file (do not drop unknown keys — the original `atlassian` skill shares it), and verify with `GET /rest/api/3/myself` and `GET /rest/api/3/project/{mirror_project_key}`. See the install prompts under `copilot-app/`, `copilot-cli/`, `claude-code/`.

If the config already exists but has no `mirror_project_key`, just ask for the mirror project key and merge it in.

### Expired token (401/403)

> **First, rule out a malformed cached header.** If you get `401` on `/myself`, empty `issues[]` from `/search/jql`, or `404` on a valid `/issue/{KEY}`, the cached `token` is likely missing the `Basic ` prefix — this authenticates as **anonymous**, not expired. Re-derive it: `token = "Basic " + Base64(email + ":" + api_token)` and retry before assuming the token is expired. (The PowerShell snippet below self-heals this automatically.)

1. Tell the user: "Your Atlassian API token appears expired or invalid."
2. Point them at: https://id.atlassian.com/manage-profile/security/api-tokens
3. After they generate a new one, they can either edit `api_token` in the config file directly (leave `token` blank — the skill will re-derive it), or run `/atlassian-gad setup` again.

### Update the skill (`/atlassian-gad update`)

Re-download the latest `SKILL.md` from the repo/branch it was installed from. **Preserves config** — only overwrites the skill file.

1. Read `atlassian_gad_source_raw_base` from the config. If missing, ask the user for the raw base URL (e.g. `https://raw.githubusercontent.com/azaespa/skills/main`) and save it back to the config.
2. Detect where the skill was installed (based on the path this SKILL.md was loaded from):
   - Copilot CLI: `~/.agents/skills/atlassian-gad/SKILL.md`
   - Copilot Desktop App (Win): `%APPDATA%\com.github.githubapp\app-skills\atlassian-gad\SKILL.md`
   - Copilot Desktop App (Mac): `~/Library/Application Support/com.github.githubapp/app-skills/atlassian-gad/SKILL.md`
   - Claude Code: `~/.claude/skills/atlassian-gad/SKILL.md`
3. Re-download `{atlassian_gad_source_raw_base}/skills/atlassian-gad/SKILL.md` to that location.

```powershell
$c = Get-Content "$env:USERPROFILE\.copilot\atlassian-config.json" -Raw | ConvertFrom-Json
Invoke-WebRequest "$($c.atlassian_gad_source_raw_base)/skills/atlassian-gad/SKILL.md" -OutFile "<INSTALL_DIR>\SKILL.md"
Write-Host "✅ SKILL.md updated. Start a new chat to load the new version."
```

## How to call the API

**No MCP.** Direct HTTPS calls to `https://{site}/rest/api/3/*` (Jira) or `https://{site}/wiki/api/v2/*` (Confluence) with the Basic auth header from the config file.

### PowerShell preamble (Windows — preferred). Start every script with this.

```powershell
$cfgPath = "$env:USERPROFILE\.copilot\atlassian-config.json"
$c = Get-Content $cfgPath -Raw | ConvertFrom-Json
# Self-heal: a token missing the "Basic " prefix authenticates as ANONYMOUS -> silent 401s / empty results.
if (-not $c.token -or -not $c.token.StartsWith("Basic ")) {
  $b64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("$($c.email):$($c.api_token)"))
  $c | Add-Member -NotePropertyName token -NotePropertyValue "Basic $b64" -Force
  ($c | ConvertTo-Json -Depth 10) | Set-Content $cfgPath -Encoding UTF8
}
$h  = @{ Authorization = $c.token; "User-Agent" = "atlassian-skill"; Accept = "application/json" }
$b  = "https://$($c.site)"
$mp = if ($c.mirror_project_key) { $c.mirror_project_key } else { "GAD" }
$mirrorOn = ($c.mirror_enabled -ne $false)
function Jira($method, $path, $body) {
  if ($null -ne $body) {
    $json = $body | ConvertTo-Json -Depth 50 -Compress
    Invoke-RestMethod -Method $method -Uri "$b$path" -Headers $h -ContentType "application/json; charset=utf-8" -Body ([Text.Encoding]::UTF8.GetBytes($json))
  } else {
    Invoke-RestMethod -Method $method -Uri "$b$path" -Headers $h
  }
}
```

Example — my open issues:

```powershell
(Jira Post "/rest/api/3/search/jql" @{ jql = "assignee = currentUser() AND resolution = Unresolved"; fields = @("summary","status","priority"); maxResults = 20 }).issues |
  % { "$($_.key) [$($_.fields.status.name)] $($_.fields.summary)" }
```

### Python (Mac / Linux fallback)

```python
import json, urllib.request, os
c = json.load(open(os.path.expanduser("~/.copilot/atlassian-config.json")))
h = {"Authorization": c["token"], "User-Agent": "atlassian-skill",
     "Accept": "application/json", "Content-Type": "application/json"}
def jira(method, path, body=None):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(f"https://{c['site']}{path}", data, h, method=method)
    raw = urllib.request.urlopen(req).read().decode()
    return json.loads(raw) if raw else None
MP = c.get("mirror_project_key") or "GAD"
```

Port the mirror recipes below to Python 1:1 when on Mac/Linux.

**Rules:**
- Always send `User-Agent` — Atlassian's edge returns 403 without it.
- Jira REST **v2 is gone (410)** — use `/rest/api/3/*` only.
- `GET /rest/api/3/search` is removed — use `POST /rest/api/3/search/jql` with a JSON body.
- Descriptions and comments in v3 require **ADF** (see below), not plain text.
- In PowerShell, always use `ConvertTo-Json -Depth 50` for ADF payloads — the default depth (2) silently truncates them.
- In PowerShell strings write `"$($key)?fields=..."`, never `"$key?fields=..."`.

## Jira REST endpoints

| Action | Method | Endpoint | Notes |
|---|---|---|---|
| Search issues | POST | `/rest/api/3/search/jql` | body: `{jql, fields, maxResults}` |
| Get issue | GET | `/rest/api/3/issue/{issueKey}` | |
| Create issue | POST | `/rest/api/3/issue` | body: `{fields:{project,summary,issuetype,description(ADF)}}` |
| Update issue | PUT | `/rest/api/3/issue/{issueKey}` | body: `{fields:{...}}` |
| Add comment | POST | `/rest/api/3/issue/{issueKey}/comment` | body: `{body: <ADF>}` |
| List transitions | GET | `/rest/api/3/issue/{issueKey}/transitions` | returns transition IDs + target status |
| Transition issue | POST | `/rest/api/3/issue/{issueKey}/transitions` | body: `{transition:{id:"<id>"}}` |
| Project (issue types) | GET | `/rest/api/3/project/{projectKey}` | `.issueTypes[]` with `name`, `subtask` |
| Current user | GET | `/rest/api/3/myself` | verification / whoami |

### Common JQL

- My open issues: `assignee = currentUser() AND resolution = Unresolved`
- By project: `project = PROJ AND resolution = Unresolved`
- In Review: `assignee = currentUser() AND status = "In Review"`
- Recent: `assignee = currentUser() ORDER BY updated DESC`

## Team Mirror

Goal: every ticket the user creates **outside** the mirror project (`$mp`, default `GAD`) gets a twin in `$mp`, so the team board shows the work without anyone creating two tickets by hand.

### Mirror conventions (do not change — sync relies on them)

| Mirror field | Value |
|---|---|
| Project | `$mp` |
| Summary | `[<ORIGINAL-KEY>] <original summary>` — e.g. `[ABC-123] Fix login timeout` |
| Description (ADF) | First paragraph: `Mirrors <ORIGINAL-KEY>` with the key linked to `https://{site}/browse/<ORIGINAL-KEY>`, then a rule, then the original description |
| Issue type | Same name as the original if `$mp` has it (non-subtask). Sub-tasks and unknown types ⇒ `Task` |
| Assignee | Same as the original, if assignable in `$mp`; otherwise left unassigned |

There is **no Jira issue link** between the two tickets — the `[KEY]` summary prefix and the `Mirrors KEY` description line are the only connection. That is intentional (user preference).

### When to mirror

- **create issue** in any project other than `$mp`: create the original first, then immediately run **Create mirror** for it. Report both keys: `Created ABC-123 → mirrored as GAD-1540`.
- **create issue** in `$mp` itself: do not mirror.
- Skip mirroring if `mirror_enabled` is `false`, or the user explicitly says "no mirror" / "don't mirror" for that ticket.
- If the mirror fails, **never delete or roll back the original**. Report the error and tell the user they can retry with `mirror <KEY>`.
- Never mirror a ticket whose summary already starts with `[` + a Jira key + `]` and lives in `$mp` (that's a mirror itself).

### When to sync

Only the **summary** and **status** are synced, one-way, original ⇒ mirror.

- **update <KEY>** that changes the summary of a non-`$mp` ticket ⇒ after updating the original, **Find mirror** and set its summary to `[<KEY>] <new summary>`.
- **move <KEY>** / **update <KEY>** that changes status of a non-`$mp` ticket ⇒ after transitioning the original, run **Sync status**.
- **sync <KEY>** ⇒ run both summary and status sync, even if nothing changed through the skill (use this after editing the original directly in Jira).
- Description, comments, priority, etc. are **not** synced.
- If no mirror is found during a sync, say so and offer `mirror <KEY>` — don't create one silently.

### Recipe: Find mirror

```powershell
$origKey = "ABC-123"
$jql = 'project = {0} AND summary ~ "\"{1}\"" ORDER BY created ASC' -f $mp, $origKey
$mirror = (Jira Post "/rest/api/3/search/jql" @{ jql = $jql; fields = @("summary","status"); maxResults = 20 }).issues |
  ? { $_.fields.summary.StartsWith("[$origKey]") } | Select-Object -First 1
if (-not $mirror) {
  $jql = 'project = {0} AND text ~ "\"Mirrors {1}\"" ORDER BY created ASC' -f $mp, $origKey
  $mirror = (Jira Post "/rest/api/3/search/jql" @{ jql = $jql; fields = @("summary","status"); maxResults = 5 }).issues | Select-Object -First 1
}
$mirror.key   # $null => no mirror
```

### Recipe: Create mirror

Run **Find mirror** first; if one already exists, report it and stop (no duplicates).

```powershell
$origKey = "ABC-123"
$o = Jira Get "/rest/api/3/issue/$($origKey)?fields=summary,issuetype,assignee,description,project"
if ($o.fields.project.key -eq $mp) { throw "$origKey is already in $mp - nothing to mirror." }

$types = (Jira Get "/rest/api/3/project/$mp").issueTypes | ? { -not $_.subtask } | % { $_.name }
$typeName = if (-not $o.fields.issuetype.subtask -and $types -contains $o.fields.issuetype.name) { $o.fields.issuetype.name } else { "Task" }

$ref = @{ type = "paragraph"; content = @(
  @{ type = "text"; text = "Mirrors " },
  @{ type = "text"; text = $origKey; marks = @(@{ type = "link"; attrs = @{ href = "$b/browse/$origKey" } }) }
) }
$fullDesc = @{ type = "doc"; version = 1; content = @($ref) }
if ($o.fields.description -and $o.fields.description.content) {
  $fullDesc.content = @($ref, @{ type = "rule" }) + @($o.fields.description.content)
}
$refOnly = @{ type = "doc"; version = 1; content = @($ref) }

$base = @{ project = @{ key = $mp }; summary = "[$origKey] $($o.fields.summary)"; issuetype = @{ name = $typeName } }
# Attempts, most complete first. Copied ADF can fail (e.g. media from the other ticket); assignee may not be assignable in $mp.
$attempts = @()
if ($o.fields.assignee) { $attempts += ,($base + @{ description = $fullDesc; assignee = @{ accountId = $o.fields.assignee.accountId } }) }
$attempts += ,($base + @{ description = $fullDesc })
$attempts += ,($base + @{ description = $refOnly })

$created = $null; $lastErr = $null
foreach ($f in $attempts) {
  try { $created = Jira Post "/rest/api/3/issue" @{ fields = $f }; break }
  catch { $lastErr = $_.ErrorDetails.Message; if (-not $lastErr) { $lastErr = $_.Exception.Message } }
}
if ($created) { "Mirrored $origKey -> $($created.key)  $b/browse/$($created.key)" } else { "Mirror FAILED for ${origKey}: $lastErr" }
```

After creating the mirror, run **Sync status** once so it starts in the matching column (e.g. original already "In Progress").

### Recipe: Sync summary

```powershell
$o = Jira Get "/rest/api/3/issue/$($origKey)?fields=summary"
Jira Put "/rest/api/3/issue/$($mirror.key)" @{ fields = @{ summary = "[$origKey] $($o.fields.summary)" } } | Out-Null
```

### Recipe: Sync status

Match order: (1) a transition into a status with the **same name** as the original's; (2) the `mirror_status_fallback` status for the original's status **category**; (3) any transition into a status of the same category.

```powershell
$o  = Jira Get "/rest/api/3/issue/$($origKey)?fields=status"
$target = $o.fields.status.name
$cat    = $o.fields.status.statusCategory.key            # new | indeterminate | done
$m  = Jira Get "/rest/api/3/issue/$($mirror.key)?fields=status"
$fallbacks = @{ new = "Backlog"; indeterminate = "In Progress"; done = "Done" }
if ($c.mirror_status_fallback) { $c.mirror_status_fallback.PSObject.Properties | % { $fallbacks[$_.Name] = $_.Value } }
$fb = $fallbacks[$cat]

if ($m.fields.status.name -ieq $target -or ($m.fields.status.name -ieq $fb -and $m.fields.status.statusCategory.key -eq $cat)) {
  "$($mirror.key) already in '$($m.fields.status.name)'"
} else {
  $tr = (Jira Get "/rest/api/3/issue/$($mirror.key)/transitions").transitions
  $pick = $tr | ? { $_.to.name -ieq $target } | Select-Object -First 1
  if (-not $pick) { $pick = $tr | ? { $_.to.name -ieq $fb } | Select-Object -First 1 }
  if (-not $pick) { $pick = $tr | ? { $_.to.statusCategory.key -eq $cat } | Select-Object -First 1 }
  if ($pick) {
    Jira Post "/rest/api/3/issue/$($mirror.key)/transitions" @{ transition = @{ id = $pick.id } } | Out-Null
    "$($mirror.key): '$($m.fields.status.name)' -> '$($pick.to.name)' (original $origKey is '$target')"
  } else {
    "Could not sync status: no transition from '$($m.fields.status.name)' fits '$target'. Available: " + (($tr | % { $_.to.name }) -join ", ")
  }
}
```

### Recipe: `mirrors` (find unmirrored tickets)

1. Search `reporter = currentUser() AND project != <MP> AND resolution = Unresolved ORDER BY created DESC` (max 50).
2. Search `project = <MP> AND reporter = currentUser() AND summary ~ "\"<KEY>\""` for each, or a single `project = <MP> AND reporter = currentUser() AND resolution = Unresolved` and match the `[KEY]` prefixes client-side (preferred — fewer calls).
3. List those without a mirror and offer to mirror them all or a selection. Only create after the user confirms.

## Confluence REST endpoints

Confluence uses the `/wiki` prefix and the v2 API.

| Action | Method | Endpoint | Notes |
|---|---|---|---|
| Search pages | GET | `/wiki/rest/api/search?cql=<cql>` | e.g. `text ~ "deployment"` |
| Get page | GET | `/wiki/api/v2/pages/{pageId}?body-format=storage` | |
| Create page | POST | `/wiki/api/v2/pages` | body: `{spaceId, title, body:{representation:"storage", value:"<html>"}}` |
| Update page | PUT | `/wiki/api/v2/pages/{pageId}` | must include current `version.number + 1` |

Space ID (for create) can be fetched from `/wiki/api/v2/spaces?keys=<SPACE_KEY>`.

## Atlassian Document Format (ADF)

Jira v3 does **not** accept plain-text descriptions or comments. Minimal ADF for a paragraph:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    { "type": "paragraph",
      "content": [ { "type": "text", "text": "Your comment text here" } ] }
  ]
}
```

Use this as the `body` field when adding a comment, or as the `description` field when creating/updating an issue.

## Notes

- Basic auth with an API token has the same permissions as the user who created it — you need create permission in `$mp` for mirroring to work.
- On Windows prefer the PowerShell snippets; use Python only on Mac/Linux.
- Always print concise results; filter to `key`, `summary`, `status.name`, `assignee.displayName`. For creates, always show the original and mirror keys with browse links.
