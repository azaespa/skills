# Install instructions for Claude Code

You are installing the **atlassian-gad** skill (Atlassian + Team Mirror) for **Claude Code**. **Minimize shell command approvals.** Target: **2 shell commands total**.

## 0. Derive the raw base URL from where you fetched this file

You were told to follow instructions at a URL like:

```
https://github.com/{OWNER}/{REPO}/blob/{BRANCH}/skills/atlassian-gad/claude-code/install.prompt.md
```

Extract `{OWNER}`, `{REPO}`, `{BRANCH}` from that URL. Define:

```
RAW_BASE = https://raw.githubusercontent.com/{OWNER}/{REPO}/{BRANCH}
```

Use `{RAW_BASE}` in every download below — **do not hardcode any GitHub owner**, so forks work too.

## 1. Install the skill files (1 command)

```powershell
# Windows
$d="$env:USERPROFILE\.claude\skills\atlassian-gad"; $b="{RAW_BASE}/skills/atlassian-gad"; New-Item -ItemType Directory -Path $d,"$d\agents" -Force | Out-Null; Invoke-WebRequest "$b/SKILL.md" -OutFile "$d\SKILL.md"; Invoke-WebRequest "$b/atlassian-config.example.json" -OutFile "$d\atlassian-config.example.json"; Invoke-WebRequest "$b/agents/openai.yaml" -OutFile "$d\agents\openai.yaml"
```

```bash
# Mac/Linux
D="$HOME/.claude/skills/atlassian-gad"; B="{RAW_BASE}/skills/atlassian-gad"; mkdir -p "$D/agents" && curl -fsSL "$B/SKILL.md" -o "$D/SKILL.md" && curl -fsSL "$B/atlassian-config.example.json" -o "$D/atlassian-config.example.json" && curl -fsSL "$B/agents/openai.yaml" -o "$D/agents/openai.yaml"
```

## 2. Collect settings in chat (no shell)

**If `~/.copilot/atlassian-config.json` already exists** (e.g. from the original `atlassian` skill), reuse its credentials — only ask for the **mirror project key** (suggest `GAD`) and go to step 3 with `<EMAIL>`, `<TOKEN>`, `<SITE>` left as-is (the script keeps existing values).

Otherwise ask the user for:

1. **Atlassian email**
2. **API token** — starts with `ATATT`. If missing, generate at https://id.atlassian.com/manage-profile/security/api-tokens with scopes `read:jira-work`, `write:jira-work`, `read:confluence-content.all`, `write:confluence-content` (90-day expiry).
3. **Site URL** — e.g. `your-org.atlassian.net`
4. **Mirror project key** — the team project your tickets get mirrored into (default `GAD`)

## 3. Merge config + fetch cloudId + verify (1 combined command)

Substitute `<EMAIL>`, `<TOKEN>`, `<SITE>`, `<MIRROR>`, then run **one** script. It **merges** into the existing config (never drops other keys) and never prints the token.

```powershell
# Windows
$e="<EMAIL>"; $t="<TOKEN>"; $s="<SITE>"; $m="<MIRROR>"
$p="$env:USERPROFILE\.copilot\atlassian-config.json"; New-Item -ItemType Directory -Path (Split-Path $p) -Force | Out-Null
$c = if (Test-Path $p) { Get-Content $p -Raw | ConvertFrom-Json } else { [pscustomobject]@{} }
function Set-K($k,$v){ $c | Add-Member -NotePropertyName $k -NotePropertyValue $v -Force }
if ($e -notlike "<*") { Set-K email $e }; if ($t -notlike "<*") { Set-K api_token $t }; if ($s -notlike "<*") { Set-K site $s }
$auth = "Basic " + [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("$($c.email):$($c.api_token)"))
$h = @{ Authorization=$auth; "User-Agent"="atlassian-skill" }
$cid = (Invoke-RestMethod "https://$($c.site)/_edge/tenant_info" -Headers $h).cloudId
$me  = Invoke-RestMethod "https://$($c.site)/rest/api/3/myself" -Headers $h
$prj = Invoke-RestMethod "https://$($c.site)/rest/api/3/project/$m" -Headers $h
Set-K token $auth; Set-K cloudId $cid; Set-K mirror_project_key $m; Set-K atlassian_gad_source_raw_base "{RAW_BASE}"
if ($null -eq $c.mirror_enabled) { Set-K mirror_enabled $true }
if ($null -eq $c.mirror_status_fallback) { Set-K mirror_status_fallback @{ new="Backlog"; indeterminate="In Progress"; done="Done" } }
($c | ConvertTo-Json -Depth 10) | Set-Content $p -Encoding UTF8
Write-Host "OK site=$($c.site) user=$($me.displayName) cloudId=$cid mirror=$m ($($prj.name))"
```

```bash
# Mac/Linux
E="<EMAIL>"; T="<TOKEN>"; S="<SITE>"; M="<MIRROR>"
python3 -c "
import json,base64,os,urllib.request
p=os.path.expanduser('~/.copilot/atlassian-config.json'); os.makedirs(os.path.dirname(p),exist_ok=True)
c=json.load(open(p)) if os.path.exists(p) else {}
for k,v in (('email','$E'),('api_token','$T'),('site','$S')):
    if not v.startswith('<'): c[k]=v
auth='Basic '+base64.b64encode(f\"{c['email']}:{c['api_token']}\".encode()).decode()
h={'Authorization':auth,'User-Agent':'atlassian-skill'}
def get(u): return json.loads(urllib.request.urlopen(urllib.request.Request(u,headers=h)).read())
s=c['site']; cid=get(f'https://{s}/_edge/tenant_info')['cloudId']
me=get(f'https://{s}/rest/api/3/myself'); prj=get(f'https://{s}/rest/api/3/project/$M')
c.update(token=auth,cloudId=cid,mirror_project_key='$M',atlassian_gad_source_raw_base='{RAW_BASE}')
c.setdefault('mirror_enabled',True); c.setdefault('mirror_status_fallback',{'new':'Backlog','indeterminate':'In Progress','done':'Done'})
json.dump(c,open(p,'w'),indent=2)
print(f\"OK site={s} user={me['displayName']} cloudId={cid} mirror=$M ({prj['name']})\")
"
```

## 4. Show status summary (no shell)

```
✅ atlassian-gad skill installed and configured

  Site:    <SITE>
  Account: <displayName>
  Mirror:  <MIRROR> (<project name>)

Try these next:
  /atlassian-gad create issue        — create a ticket (auto-mirrored to <MIRROR>)
  /atlassian-gad mirror ABC-123      — mirror an existing ticket
  /atlassian-gad sync ABC-123        — re-sync summary + status to the mirror
  /atlassian-gad mirrors             — find your tickets that aren't mirrored yet
  /atlassian-gad my issues           — list Jira issues assigned to you
```

If step 3 fails with 401/403, the token or scopes are wrong — send them to https://id.atlassian.com/manage-profile/security/api-tokens. A 404 on `/project/<MIRROR>` means the key is wrong or they can't see that project.