# Jira Coordinator Implementation Plan

> _Sanitized for public sharing. All organization-specific data (projects, people, hosts, IDs, paths) is replaced with placeholders; real values live in the private internal copy._

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a distributable Claude Code plugin that acts as a daily Jira project coordinator (digest + signals + capacity/availability + assignment advisor + manpower report), resolves people by nickname, watches/builds release tickets, and exposes a `/jira` router — all under a strict draft-then-approve write rule.

**Architecture:** A plugin (`jira-coordinator`) whose user-facing triggers are thin **slash commands** in `commands/`, which invoke heavier **skills** in `skills/` that hold the engine logic. All team-specific knowledge lives in per-lead **config profiles** under `.jira-coordinator/profiles/<team>/` (git-ignored), never in the plugin. Jira access is via the already-connected Atlassian (Rovo) MCP, authenticated as each lead.

**Tech Stack:** Claude Code plugin format (`.claude-plugin/plugin.json`, `commands/*.md`, `skills/*/SKILL.md`), YAML config, Atlassian Rovo MCP tools (`searchJiraIssuesUsingJql`, `getJiraIssue`, `createJiraIssue`, `editJiraIssue`, `addCommentToJiraIssue`, `transitionJiraIssue`, `lookupJiraAccountId`). Validation via `jq`, `python3 -c "import yaml"`, and `claude plugin validate`.

**Spec:** `design/ARCHITECTURE.md`

**Conventions used throughout this plan**
- cloudId: `<YOUR_CLOUD_ID>` (site `your-site.atlassian.net`).
- "Validate" steps must be run and must show the expected output before moving on.
- "Acceptance" steps require actually invoking the workflow in a Claude Code session that has the Atlassian MCP connected; read-only acceptance never writes to Jira.
- Commit after every task.

---

## File Structure

```
jira/                                         ← plugin repo root (this directory)
├── .claude-plugin/
│   ├── plugin.json                           plugin manifest
│   └── marketplace.json                      self-hosted marketplace entry
├── .gitignore                                ignore per-lead config + outputs
├── README.md                                 install + usage for other leads
├── commands/
│   ├── jira.md                               /jira <intent> router (thin)
│   ├── jira-coordinator.md                   /jira-coordinator (thin)
│   └── release-ticket.md                     /release-ticket (thin)
├── skills/
│   ├── jira-coordinator/
│   │   ├── SKILL.md                          digest engine + init + capacity/advisor
│   │   └── references/
│   │       ├── jql-recipes.md                build scope JQL from a profile
│   │       ├── signal-rules.md               attention-signal definitions
│   │       ├── digest-template.md            output structure
│   │       └── capacity-rules.md             nickname resolve, load model, advisor, manpower
│   └── release-ticket/
│       ├── SKILL.md                          build a deployment SR
│       └── references/
│           └── clone-rules.md                find + bump the last release SR
├── templates/                                blank profile files copied on init
│   ├── teams.example.yaml
│   ├── scope.example.yaml
│   ├── conventions.example.md
│   ├── projects.example.md
│   ├── release.example.yaml
│   ├── release-template.example.md
│   ├── environments.example.yaml
│   └── availability.example.yaml
├── .jira-coordinator/
│   └── profiles/
│       └── _fixture/                         test fixture profile (committed)
│           ├── teams.yaml
│           ├── scope.yaml
│           ├── conventions.md
│           ├── projects.md
│           ├── release.yaml
│           ├── release-template.md
│           ├── environments.yaml
│           └── availability.yaml
└── docs/
    ├── superpowers/{specs,plans}/            (existing)
    └── standups/                             digest output (git-ignored)
```

Per-lead real profiles live under `.jira-coordinator/profiles/<team>/` and are git-ignored. Only `_fixture/` is committed (for acceptance tests).

---

## Task 0: Repo + plugin skeleton

**Files:**
- Create: `.gitignore`
- Create (dirs): `.claude-plugin/`, `commands/`, `skills/jira-coordinator/references/`, `skills/release-ticket/references/`, `templates/`, `.jira-coordinator/profiles/_fixture/`, `docs/standups/`

- [ ] **Step 1: Initialize git and create the directory tree**

```bash
cd ~/jira
git init
mkdir -p .claude-plugin commands \
  skills/jira-coordinator/references skills/release-ticket/references \
  templates .jira-coordinator/profiles/_fixture docs/standups
```

- [ ] **Step 2: Create `.gitignore`**

```
# Per-lead config profiles are private — never commit (except the test fixture)
.jira-coordinator/profiles/*
!.jira-coordinator/profiles/_fixture/

# Daily digest output is personal/ephemeral
docs/standups/*
!docs/standups/.gitkeep

# OS noise
.DS_Store
```

- [ ] **Step 3: Keep the standups dir tracked but empty**

```bash
touch docs/standups/.gitkeep
```

- [ ] **Step 4: Validate the tree exists**

Run: `find . -type d -not -path './.git/*' | sort`
Expected: lists `.claude-plugin`, `commands`, `skills/jira-coordinator/references`, `skills/release-ticket/references`, `templates`, `.jira-coordinator/profiles/_fixture`, `docs/standups`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: scaffold jira-coordinator plugin repo"
```

---

## Task 1: Plugin manifest + marketplace entry

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Write `plugin.json`**

```json
{
  "name": "jira-coordinator",
  "version": "0.1.0",
  "description": "AI project coordinator for Jira: daily digest, deadline/people/workload signals, release-ticket watch and builder, all under a draft-then-approve write rule. Powered by the Atlassian MCP.",
  "author": { "name": "Your Name", "email": "you@example.com" },
  "keywords": ["jira", "atlassian", "project-management", "release", "devops"]
}
```

- [ ] **Step 2: Write `marketplace.json`** (self-hosted so other leads can `plugin marketplace add` this repo)

```json
{
  "name": "jira-coordinator-marketplace",
  "owner": { "name": "Your Name", "email": "you@example.com" },
  "plugins": [
    {
      "name": "jira-coordinator",
      "source": "./",
      "description": "AI project coordinator for Jira (digest, signals, release tickets)."
    }
  ]
}
```

- [ ] **Step 3: Validate both files are well-formed JSON with required keys**

Run:
```bash
jq -e '.name and .version and .description' .claude-plugin/plugin.json
jq -e '.name and .plugins[0].name == "jira-coordinator"' .claude-plugin/marketplace.json
```
Expected: each prints `true` (exit 0). If `jq` errors, the JSON is malformed — fix.

- [ ] **Step 4: Validate with Claude Code if available**

Run: `claude plugin validate . 2>/dev/null || echo "validator not available, jq checks suffice"`
Expected: either a success message, or the fallback echo (jq checks in Step 3 are then authoritative).

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "feat: add plugin manifest and marketplace entry"
```

---

## Task 2: Profile templates

These are the blank files `init` mode copies for a new lead. Each is valid and self-documenting.

**Files:**
- Create: `templates/teams.example.yaml`, `templates/scope.example.yaml`, `templates/conventions.example.md`, `templates/projects.example.md`, `templates/release.example.yaml`, `templates/release-template.example.md`, `templates/environments.example.yaml`

- [ ] **Step 1: `templates/teams.example.yaml`**

```yaml
# Who is on the team. accountIds are resolved during `init` via lookupJiraAccountId.
# aliases  = nicknames you use for this person; "assign to <alias>" resolves here.
# wip_limit = max concurrent active tickets before this person is "overloaded".
team: ExampleApp Core
lead:
  name: Your Name
  accountId: "REPLACE_WITH_ACCOUNT_ID"
subteams:
  backend:
    - { name: alice, accountId: "REPLACE_WITH_ACCOUNT_ID", aliases: [wei, sc], wip_limit: 5 }
    - { name: bob, accountId: "REPLACE_WITH_ACCOUNT_ID", aliases: [jl], wip_limit: 5 }
  frontend:
    - { name: Carol, accountId: "REPLACE_WITH_ACCOUNT_ID", aliases: [abc, kn], wip_limit: 4 }
```

- [ ] **Step 2: `templates/scope.example.yaml`**

```yaml
# Which tickets are "in scope" for the daily digest.
projects: [PJ-CORE, PJ-VI, PJ-VL, PJ-GAM, PJ-LIST, PJ-CS2, PJ-CS, PJ-REC, REQ]
base_jql: "project in ({projects}) AND statusCategory != Done"
thresholds:
  stale_days: 3
  deadline_warning_days: 2
include_unassigned: true
overrides_allowed: true
```

- [ ] **Step 3: `templates/conventions.example.md`**

```markdown
# Team conventions (lead-authored guidance, read by the engine)

## Status meanings
- "Escalate to Vendor Ops": waiting on the vendor; not a team blocker.
- "Pending QA/Vendor": handed off, awaiting external verification.
<!-- Add the custom statuses your projects use and what they mean. -->

## Deadline fallback chain
1. Issue `duedate` field.
2. Active sprint end date.
3. SLA-by-issue-type table below.
4. If none: flag "no deadline signal" — never invent a date.

## SLA by issue type (working days from creation)
| Issue type | SLA |
|---|---|
| Bug (Production) | 2 |
| Bug | 5 |
| Story | 10 |
| Task | 10 |

## Per-status time budgets (for the "stuck status" signal, working days)
| Status | Budget |
|---|---|
| In Progress | 5 |
| Pending For Approval | 2 |
```

- [ ] **Step 4: `templates/projects.example.md`**

```markdown
# Project glossary

- PJ-CORE — ExampleApp (core aggregation API)
- PJ-VI — ExampleApp Vendor Integration
- PJ-VL — ExampleApp Vendor Line
- PJ-GAM — ExampleApp Gamification
- PJ-LIST — ExampleApp Centralized Game List
- PJ-CS2 / PJ-CS — ExampleApp Customer / Client Support
- REQ — ExampleApp Feature Request Form
- PJ-REC — ExampleApp Reconciliation
- OPS — DevOps (release / deployment Service Requests)
```

- [ ] **Step 5: `templates/release.example.yaml`**

```yaml
# Configuration for release watch + /release-ticket.
project: OPS
issue_type: Service Request
# How to find the most recent release to clone for a given product line:
match_rules:
  APP-BO: 'project = OPS AND summary ~ "APP-BO" AND issuetype = "Service Request" ORDER BY created DESC'
  vendor-service: 'project = OPS AND summary ~ "vendor-service" AND issuetype = "Service Request" ORDER BY created DESC'
  service-d: 'project = OPS AND summary ~ "service-d" AND issuetype = "Service Request" ORDER BY created DESC'
labels: [myapp, n8n-automation]
region_sequence:
  - { region: Region1,   label: region-1, time: "10:00" }
  - { region: Region2,    label: region-2,     time: "11:00" }
  - { region: Region3,    label: region-3,     time: "14:00" }
  - { region: Region4,  label: region-4,   time: "15:00" }
  - { region: Region5, label: region-5,  time: "16:00" }
services:
  service-a: "https://git.example.com/your-group/service-a"
  service-b: "https://git.example.com/your-group/service-b"
  service-c: "https://git.example.com/your-group/service-c"
mentions:
  dev: ["REPLACE_WITH_ACCOUNT_ID"]
  qa:  ["REPLACE_WITH_ACCOUNT_ID"]
# Watch: flag release SRs stuck at these statuses past the budget (working days).
watch_status_budgets:
  "Pending For Approval": 2
```

- [ ] **Step 6: `templates/release-template.example.md`**

```markdown
Please assist in deploying tag **{version}** on {date} ({weekday}) on the following sequence:

{region_sequence}

{release_report_link}

**Deployment service**

Pre-Deployment action
{pre_deployment_actions}

{service_blocks}

Attn: {dev_mentions}

QA: {qa_mentions}
```

- [ ] **Step 7: `templates/environments.example.yaml`**

```yaml
# Dev -> QA -> Product handoff model.
# PLACEHOLDER VALUES — confirm real status names with the lead before relying on handoff signals.
environments: [SBX, QA, STG, PROD]
stages:
  - { name: dev,        ready_status: "Integration Done", next: QA,   owner: backend, budget_days: 2 }
  - { name: qa,         ready_status: "Ready for QA",      next: STG,  owner: qa,      budget_days: 3 }
  - { name: staging,    ready_status: "QA Passed",         next: PROD, owner: qa,      budget_days: 2 }
  - { name: production, ready_status: "Approved for PROD", next: null, owner: devops,  budget_days: 1 }
```

- [ ] **Step 7b: `templates/availability.example.yaml`**

```yaml
# Manual availability register. Keep current; entries overlapping today mark a
# person unavailable (the assignment advisor and manpower report use this).
# types: annual_leave | medical_leave | holiday | meeting | outstation | available
entries:
  - { who: Carol, type: annual_leave,  from: 2026-05-28, to: 2026-05-30 }
  - { who: alice,      type: medical_leave, from: 2026-05-27, to: 2026-05-27 }
  - { who: bob,      type: outstation,    from: 2026-05-27, to: 2026-05-29, note: client site }
```

- [ ] **Step 8: Validate every YAML template parses**

Run:
```bash
for f in templates/*.yaml; do python3 -c "import yaml,sys; yaml.safe_load(open('$f')); print('OK', '$f')"; done
```
Expected: one `OK templates/...` line per YAML file, no traceback.

- [ ] **Step 9: Commit**

```bash
git add templates/
git commit -m "feat: add profile templates for init mode"
```

---

## Task 3: Test fixture profile

A committed, realistic profile so skills can be acceptance-tested without a real lead's private config.

**Files:**
- Create: `.jira-coordinator/profiles/_fixture/{teams.yaml,scope.yaml,conventions.md,projects.md,release.yaml,release-template.md,environments.yaml}`

- [ ] **Step 1: Copy templates into the fixture and make scope tiny for fast tests**

```bash
cp templates/teams.example.yaml        .jira-coordinator/profiles/_fixture/teams.yaml
cp templates/scope.example.yaml        .jira-coordinator/profiles/_fixture/scope.yaml
cp templates/conventions.example.md    .jira-coordinator/profiles/_fixture/conventions.md
cp templates/projects.example.md       .jira-coordinator/profiles/_fixture/projects.md
cp templates/release.example.yaml      .jira-coordinator/profiles/_fixture/release.yaml
cp templates/release-template.example.md .jira-coordinator/profiles/_fixture/release-template.md
cp templates/environments.example.yaml .jira-coordinator/profiles/_fixture/environments.yaml
cp templates/availability.example.yaml .jira-coordinator/profiles/_fixture/availability.yaml
```

- [ ] **Step 2: Narrow the fixture scope to the 3 most active projects (fast, real data)**

Edit `.jira-coordinator/profiles/_fixture/scope.yaml` so `projects` is exactly:

```yaml
projects: [PJ-CORE, PJ-VI, PJ-CS]
```

- [ ] **Step 3: Validate fixture YAML parses**

Run:
```bash
for f in .jira-coordinator/profiles/_fixture/*.yaml; do python3 -c "import yaml; yaml.safe_load(open('$f')); print('OK','$f')"; done
```
Expected: `OK` for each of `teams.yaml`, `scope.yaml`, `release.yaml`, `environments.yaml`.

- [ ] **Step 4: Commit**

```bash
git add -f .jira-coordinator/profiles/_fixture/
git commit -m "test: add committed fixture profile for acceptance tests"
```

---

## Task 4: jira-coordinator skill — reference docs

Write the three reference files the skill leans on. These hold the precise rules so `SKILL.md` stays short.

**Files:**
- Create: `skills/jira-coordinator/references/jql-recipes.md`
- Create: `skills/jira-coordinator/references/signal-rules.md`
- Create: `skills/jira-coordinator/references/digest-template.md`

- [ ] **Step 1: `references/jql-recipes.md`**

````markdown
# Building scope JQL from a profile

Inputs: `scope.yaml` (`projects`, `base_jql`, `thresholds`, `include_unassigned`)
and `teams.yaml` (roster accountIds).

## Project clause
Substitute `{projects}` in `base_jql` with the comma-joined `projects` list.
Example result:
`project in (PJ-CORE,PJ-VI,PJ-CS) AND statusCategory != Done`

## Roster clause (optional, for the per-person view)
`assignee in (accountId1, accountId2, ...)` built from all `subteams` members + `lead`.

## Recency
Always fetch with `ORDER BY updated DESC` and request only needed fields to keep
output small:
`["summary","status","issuetype","assignee","project","priority","duedate","updated","labels"]`

## Large output handling (REQUIRED)
The Atlassian MCP truncates large results to a file. Always:
1. Request `responseContentFormat: "markdown"` and a bounded `maxResults` (<= 50).
2. If the result is written to a file, parse it with `python3`/`jq` — never echo the
   whole file into context. Extract only the fields listed above.

## Comments + last activity (enrichment)
For each in-scope ticket needing detail, call `getJiraIssue` with
`fields: ["comment","updated","assignee","status","duedate"]` and read the latest
comment's author + timestamp.
````

- [ ] **Step 2: `references/signal-rules.md`**

```markdown
# Attention-signal definitions

Thresholds come from `scope.yaml.thresholds`, `conventions.md` (per-status budgets,
SLA table), and `environments.yaml` (stage budgets).

| Signal | Rule |
|---|---|
| Overdue | effective deadline < today AND statusCategory != Done |
| At-risk deadline | effective deadline within `deadline_warning_days` AND not progressing |
| Stale | now - `updated` >= `stale_days` working days |
| Unanswered comment | latest comment author != assignee AND comment time > assignee's last activity |
| Blocked | "is blocked by" link unresolved, or a blocked status/flag |
| In-scope unassigned | matches scope AND assignee empty (only if `include_unassigned`) |
| Stuck status | time in current status > that status's budget in conventions.md |
| Handoff pending | in a stage `ready_status` longer than its `budget_days`, not advanced |
| Release approval stuck | release SR at a `watch_status_budgets` status past its budget |
| Person overloaded | active load > member `wip_limit` (see capacity-rules.md) |
| Assigned-but-away | open assigned ticket whose assignee is unavailable today (availability.yaml) |

## Effective deadline (fallback chain)
1. `duedate`. 2. active sprint end. 3. SLA-by-issue-type (conventions.md) applied to
created date. 4. none -> emit "no deadline signal", do NOT invent a date.

## Working-days note
"Working days" excludes Sat/Sun. Use the lead's locale if conventions.md states one;
otherwise assume Mon-Fri.
```

- [ ] **Step 3: `references/digest-template.md`**

```markdown
# Daily digest output structure

Write to `docs/standups/{YYYY-MM-DD}.md` AND print to terminal. Sections, in order:

## 1. Needs your attention today
Ranked list. Each line: `[SIGNAL] KEY — one-line why — assignee — link`.

## 2. Per-person status
Grouped by sub-team, then person. Under each person: in-progress / blocked / stale
counts and the specific tickets.

## 3. Manpower / capacity
Per person: load tier (free / nearing / overloaded), active count, blocked count,
and availability status (from availability.yaml). Team totals: headroom, % at/over
capacity, count blocked on external parties. End with the management evidence line
(see capacity-rules.md).

## 4. At-risk deadlines
Overdue first, then upcoming within `deadline_warning_days`. Show effective deadline
and its source (duedate / sprint / SLA).

## 5. Release / deploy watch
Open OPS deployment SRs for the profile's services: version, target env/regions,
status. Flag any at a `watch_status_budgets` status past budget.

## 6. Proposed actions
Numbered drafts only. Each: action type (comment / due-date change / reassign /
transition / On Hold), target KEY, and exact proposed content. NOTHING is executed here.
```

- [ ] **Step 3b: `references/capacity-rules.md`**

````markdown
# Nickname resolution, capacity, availability, assignment advisor, manpower report

## Name resolution (used wherever a person is named)
Given a token like "abc":
1. Case-insensitive match against every member's `name` and `aliases` in `teams.yaml`.
2. If no match, try `lookupJiraAccountId` on the literal token.
3. If ambiguous (>1 match) or empty, ask the lead. NEVER assign to a guessed account.

## Capacity model (hybrid)
For each member, gather open non-Done tickets where they are assignee. Compute
**active load**:
- If a ticket has a story-point or time estimate, use that value.
- Else weight by priority: Highest=3, High=2, Medium=1, Low=0.5, Lowest=0.5.
- BLOCKED tickets ("is blocked by" unresolved, or blocked status/flag) are listed
  as "paused" and EXCLUDED from active load.

Load tier vs the member's `wip_limit` (interpret wip_limit as the active-load ceiling):
- `free` if load < 60% of wip_limit
- `nearing` if 60–100%
- `overloaded` if > 100%

## Availability
Read `availability.yaml`. A member is **unavailable today** if any entry has
`who` == member name AND `from <= today <= to`. Record the `type`
(annual_leave / medical_leave / holiday / meeting / outstation).

## Assignment advisor (when asked to assign work to someone)
1. Resolve the name. 2. Check availability. 3. Compute load tier.
4. If unavailable OR overloaded, do NOT silently assign. Present numbered options:
   (a) assign anyway; (b) put a specific lower-priority ticket On Hold
   (`transitionJiraIssue`); (c) re-rank their queue by priority (propose order);
   (d) name a teammate in the same sub-team who is `free`.
5. Execute only the approved option (reassign = `editJiraIssue`, On Hold =
   `transitionJiraIssue`) per the write-safety rule.

## Manpower report (management-facing)
- Per person: tier, active count, blocked count, availability.
- Team: total headroom (sum of remaining capacity), % members at/over capacity,
  count of tickets blocked on external parties (vendor / QA / product).
- Evidence line, e.g.: "Team at 100% capacity; 6 tickets blocked on vendor; new
  product requests will queue until capacity frees." State numbers, not opinions.
````

- [ ] **Step 4: Validate the reference docs exist and are non-empty**

Run: `wc -l skills/jira-coordinator/references/*.md`
Expected: FOUR files (jql-recipes, signal-rules, digest-template, capacity-rules),
each with a non-trivial line count (> 10).

- [ ] **Step 5: Commit**

```bash
git add skills/jira-coordinator/references/
git commit -m "feat: add jira-coordinator reference docs (jql, signals, digest, capacity)"
```

---

## Task 5: jira-coordinator skill — SKILL.md (digest engine + init mode)

**Files:**
- Create: `skills/jira-coordinator/SKILL.md`

- [ ] **Step 1: Write `SKILL.md`**

````markdown
---
name: jira-coordinator
description: Use when the user wants a daily Jira standup/digest, to see what needs attention across their team's tickets (deadlines, stale tickets, unanswered comments, blockers, workload), to check who has capacity / is overloaded / is on leave, to advise on assigning work to a person (by name or nickname), to produce a manpower report for management, or to watch release tickets. Reads a team profile from .jira-coordinator/profiles/ and reports via a dated markdown digest. Read + draft only — never writes to Jira without per-action approval.
---

# Jira Coordinator

Generate a daily coordinator digest for a team lead's in-scope Jira work, plus
capacity/availability analysis, an assignment advisor, and a manpower report.

cloudId: `<YOUR_CLOUD_ID>` (site your-site.atlassian.net).
Reference files (read as needed): `references/jql-recipes.md`,
`references/signal-rules.md`, `references/digest-template.md`,
`references/capacity-rules.md`.

## Write safety (HARD RULE)
Read tools may be called freely. Write tools (`editJiraIssue`,
`addCommentToJiraIssue`, `transitionJiraIssue`, `createIssueLink`,
`createJiraIssue`) may be called ONLY after the user approves a specific numbered
item from the "Proposed actions" section in THIS session. Approval never carries
across runs. When in doubt, draft — do not write.

## Modes
Do Step 0–1 (profile + context) in every mode, then:
- **Digest** (default / "standup" / "digest"): Steps 2–7 — the full digest.
- **Manpower** ("capacity" / "manpower" / "who's free" / "who's overloaded"):
  Step 4b only, then print just the Manpower/capacity report from
  `references/capacity-rules.md`.
- **Assignment advisor** ("assign \<work\> to \<name\>"): resolve the name and run
  the advisor flow in `references/capacity-rules.md`; present options, execute only
  the approved one per the write rule.

## Step 0: Select profile
1. List `.jira-coordinator/profiles/*/` (ignore `_fixture` unless asked).
2. If none exist -> go to "Init mode" below.
3. If one exists -> use it. If several -> ask which.

## Step 1: Load context
Read the profile's `teams.yaml`, `scope.yaml`, `conventions.md`, `projects.md`,
`release.yaml`, `environments.yaml`, `availability.yaml`.

## Step 2: Resolve scope and query
Build JQL per `references/jql-recipes.md`. Call `searchJiraIssuesUsingJql` with
`responseContentFormat: "markdown"`, bounded `maxResults`, and only the fields
listed in jql-recipes. If output is written to a file, parse with python3/jq —
never echo the whole file.

## Step 3: Enrich
For tickets that look relevant (recently updated, near deadline, or commented),
call `getJiraIssue` for the latest comment + activity timestamps.

## Step 4: Detect signals
Apply every rule in `references/signal-rules.md` using the profile thresholds.

## Step 4b: Capacity & availability
Per `references/capacity-rules.md`, compute each member's active load (hybrid),
load tier, and today's availability from `availability.yaml`. This feeds the
Manpower/capacity digest section and the assignment advisor.

## Step 5: Release watch
Query `release.yaml.project` for open Service Requests matching the profile's
services; apply `watch_status_budgets`.

## Step 6: Emit digest
Write `docs/standups/{today}.md` and print it, following
`references/digest-template.md`. The "Proposed actions" section lists numbered
drafts only.

## Step 7: Await approval
If the user approves specific numbered actions, execute ONLY those via the
matching write tool, then confirm what was written with ticket links.

## Init mode (first run, no profile)
1. `getAccessibleAtlassianResources` + `atlassianUserInfo` to confirm site + user.
2. Ask the team name; create `.jira-coordinator/profiles/<team>/` by copying the
   files from the plugin's `templates/` directory.
3. Help fill `teams.yaml`: for each member, resolve accountId via
   `lookupJiraAccountId`, and ask for any nicknames (`aliases`) and a `wip_limit`.
4. Confirm `scope.yaml.projects` (offer the ExampleApp core set as default).
5. Tell the user which placeholders still need filling: `conventions.md` (status
   meanings, SLAs, budgets), `environments.yaml` (stage status names — handoff
   signals stay inactive until confirmed), and `availability.yaml` (kept current
   manually — empty register is fine to start).
````

- [ ] **Step 2: Validate frontmatter has name + description**

Run:
```bash
python3 -c "
import re,sys
t=open('skills/jira-coordinator/SKILL.md').read()
m=re.match(r'^---\n(.*?)\n---', t, re.S); assert m, 'no frontmatter'
import yaml; fm=yaml.safe_load(m.group(1))
assert fm.get('name')=='jira-coordinator', fm.get('name')
assert fm.get('description'), 'no description'
print('OK frontmatter:', fm['name'])
"
```
Expected: `OK frontmatter: jira-coordinator`.

- [ ] **Step 3: Acceptance (read-only) — run the digest against the fixture**

In a Claude Code session with the Atlassian MCP connected, invoke the skill and
point it at the `_fixture` profile. Verify:
- It builds JQL `project in (PJ-CORE,PJ-VI,PJ-CS) AND statusCategory != Done`.
- It writes `docs/standups/{today}.md` containing all six sections from
  `digest-template.md` (including "Manpower / capacity").
- It makes ZERO write-tool calls.

Run (to confirm the file was produced):
```bash
F=docs/standups/$(date +%F).md
ls docs/standups/ && grep -c "Proposed actions" "$F" && grep -c "Manpower" "$F"
```
Expected: today's file exists, with exactly one "Proposed actions" heading and one
"Manpower" heading.

- [ ] **Step 3b: Acceptance — capacity & advisor**

In the connected session, also verify:
- `who's overloaded` → prints the Manpower report only, with per-person tiers and
  the management evidence line; no digest file rewrite required.
- `assign a tech-design doc to abc` → resolves `abc` to Carol via aliases,
  reports his current load tier + availability, and presents the advisor options
  (assign / put a lower-priority ticket On Hold / re-rank queue / suggest a free
  teammate) as numbered drafts — making no write call. (If `availability.yaml`
  marks him away on the run date, the away note appears; otherwise the load tier
  drives the recommendation.)

- [ ] **Step 4: Commit**

```bash
git add skills/jira-coordinator/SKILL.md
git commit -m "feat: add jira-coordinator digest skill with init mode"
```

---

## Task 6: release-ticket skill

**Files:**
- Create: `skills/release-ticket/references/clone-rules.md`
- Create: `skills/release-ticket/SKILL.md`

- [ ] **Step 1: `references/clone-rules.md`**

````markdown
# Clone-and-bump rules for /release-ticket

Hybrid: the skeleton comes from the profile's `release-template.md`; field values
are pre-filled by cloning the most recent matching release SR.

## Find the last release
Pick the product line from the user's request (e.g. "APP-BO"). Look up
`release.yaml.match_rules[<product>]` and run it via `searchJiraIssuesUsingJql`
(maxResults 1). Fetch its body with `getJiraIssue`
(`fields: ["summary","description","labels"]`, `responseContentFormat: "markdown"`).

## Carry forward vs bump
- CARRY: region sequence, dev/QA mentions, service list, pre-deployment structure.
- BUMP from user args: `{version}`, `{date}`, `{weekday}`, per-service `{tag}` values,
  and the release-report link (new PJ-CORE version page).

## Build the body
Fill `release-template.md` placeholders:
- `{region_sequence}` -> numbered lines from `release.yaml.region_sequence`
  ("1. Region1 - 10:00am").
- `{service_blocks}` -> for each service in args: name + `(tag: X)` + GitLab tag URL
  from `release.yaml.services[name] + "/-/tags/" + tag`.
- `{dev_mentions}` / `{qa_mentions}` -> from `release.yaml.mentions` (or carried).

## Labels
`release.yaml.labels` + the region labels for the regions being deployed.
````

- [ ] **Step 2: `skills/release-ticket/SKILL.md`**

````markdown
---
name: release-ticket
description: Use when the user wants to create or prepare a Jira release/deployment ticket (a OPS Service Request) — e.g. "build a release ticket for APP-BO v1.6.46". Clones the last matching deployment SR and bumps version/tags/date using a template. Drafts the ticket; creates it only after the user approves.
---

# Release Ticket Builder

cloudId: `<YOUR_CLOUD_ID>`.
Reference: `references/clone-rules.md`. Config: profile's `release.yaml` and
`release-template.md`.

## Write safety (HARD RULE)
Never call `createJiraIssue` until the user approves the drafted ticket in THIS
session.

## Steps
1. Select the active profile (as in jira-coordinator Step 0).
2. Parse the user's request for product line + version + target env + date.
   Ask for any of these that are missing.
3. Follow `references/clone-rules.md` to find the last release and build the body.
4. Present the FULL drafted ticket: project, issue type, summary
   (e.g. "[PROD] Deployment for APP-BO v1.6.46"), labels, and the rendered body.
5. On approval, call `createJiraIssue` with those fields and return the new ticket
   link. On rejection or edits, revise and re-present.
````

- [ ] **Step 3: Validate both frontmatters**

Run:
```bash
python3 -c "
import re,yaml
for p in ['skills/release-ticket/SKILL.md']:
    t=open(p).read(); m=re.match(r'^---\n(.*?)\n---', t, re.S); assert m, p
    fm=yaml.safe_load(m.group(1)); assert fm['name']=='release-ticket' and fm['description']
    print('OK', fm['name'])
"
wc -l skills/release-ticket/references/clone-rules.md
```
Expected: `OK release-ticket` and a non-trivial line count for clone-rules.

- [ ] **Step 4: Acceptance (read-only draft) — dry-run a release draft**

In a connected session, ask the skill to build a release ticket for "APP-BO v1.6.46"
using the `_fixture` profile. Verify it:
- finds the most recent APP-BO deployment SR (e.g. one of the `[PROD] Deployment for
  APP-BO ...` tickets),
- produces a summary `[PROD] Deployment for APP-BO v1.6.46`,
- renders the region sequence + service blocks from config,
- STOPS and asks for approval (makes no `createJiraIssue` call).

- [ ] **Step 5: Commit**

```bash
git add skills/release-ticket/
git commit -m "feat: add release-ticket builder skill (template + clone-last)"
```

---

## Task 7: Slash commands (thin entry points)

In Claude Code, `commands/*.md` are what the user types. Each is a thin prompt that
invokes a skill; the `/jira` router parses intent and dispatches.

**Files:**
- Create: `commands/jira-coordinator.md`, `commands/release-ticket.md`, `commands/jira.md`

- [ ] **Step 1: `commands/jira-coordinator.md`**

```markdown
---
description: Run the daily Jira coordinator digest for your team profile.
---
Invoke the `jira-coordinator` skill to produce today's digest. If the user passed
arguments, treat them as a profile name or focus hint: $ARGUMENTS
```

- [ ] **Step 2: `commands/release-ticket.md`**

```markdown
---
description: Build a Jira release/deployment ticket (clones the last one, bumps version).
---
Invoke the `release-ticket` skill. Treat the arguments as the product line and
version (e.g. "APP-BO v1.6.46"): $ARGUMENTS
```

- [ ] **Step 3: `commands/jira.md`** (umbrella router)

```markdown
---
description: Jira coordinator router — standup, release, workload, or any ad-hoc Jira request.
---
You are the `/jira` router. Parse the user's intent from: $ARGUMENTS

Dispatch:
- "standup" / "digest" / "today" / empty -> invoke `jira-coordinator` (Digest mode).
- "release ..." / "deploy ..." -> invoke the `release-ticket` skill.
- "capacity" / "manpower" / "who's free" / "who's overloaded" -> invoke
  `jira-coordinator` (Manpower mode).
- "assign ... to <name>" -> invoke `jira-coordinator` (Assignment advisor mode);
  resolve the name/nickname, check capacity + availability, present options.
- Anything else (e.g. "comment on PJ-CORE-123 ...", "create a bug in PJ-VI ...") ->
  treat as an ad-hoc Jira request and fulfill it directly via the Atlassian MCP.

HARD RULE: in all paths, never call a Jira write tool (editJiraIssue,
addCommentToJiraIssue, transitionJiraIssue, createIssueLink, createJiraIssue)
until the user approves the specific action in this session.
```

- [ ] **Step 4: Validate command frontmatter**

Run:
```bash
for f in commands/*.md; do
  python3 -c "
import re,yaml,sys
t=open('$f').read(); m=re.match(r'^---\n(.*?)\n---', t, re.S)
assert m, 'no frontmatter in $f'
assert yaml.safe_load(m.group(1)).get('description'), 'no description in $f'
print('OK', '$f')
"
done
```
Expected: `OK commands/jira.md`, `OK commands/jira-coordinator.md`, `OK commands/release-ticket.md`.

- [ ] **Step 5: Acceptance — router dispatch**

In a connected session: `/jira release APP-BO v1.6.46` should reach the release-ticket
draft flow; `/jira standup` should reach the digest; `/jira comment on PJ-CORE-1 saying hi`
should draft a comment and ask for approval (no write).

- [ ] **Step 6: Commit**

```bash
git add commands/
git commit -m "feat: add /jira router and /jira-coordinator, /release-ticket commands"
```

---

## Task 8: README + scheduling guide + distribution

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write `README.md`**

````markdown
# Jira Coordinator

An AI project-coordinator plugin for Claude Code. It runs a daily Jira digest,
surfaces what needs attention (deadlines, stale tickets, unanswered comments,
blockers, workload), watches release tickets, and builds new deployment tickets —
all under a strict **draft-then-approve** write rule. Jira access uses the
Atlassian (Rovo) MCP, authenticated as you; no credentials are shared.

## Requirements
- Claude Code with the Atlassian (Rovo) MCP connected (`/mcp` to check).

## Install (other team leads)
```
/plugin marketplace add <git-url-or-path-to-this-repo>
/plugin install jira-coordinator@jira-coordinator-marketplace
```

## First run
Run `/jira-coordinator`. With no profile yet, it enters **init mode**: it reads
your accessible Jira, helps you create `.jira-coordinator/profiles/<team>/` from
the templates, and resolves your teammates' accountIds. Fill the placeholders in
`conventions.md` (status meanings, SLAs) and `environments.yaml` (dev→QA→prod
status names) when ready — handoff signals stay off until `environments.yaml` is
confirmed.

## Daily use
- `/jira-coordinator` — full digest (written to `docs/standups/YYYY-MM-DD.md`).
- `/release-ticket APP-BO v1.6.46` — draft a deployment ticket from the last one.
- `/jira capacity` (or `manpower`) — management-facing manpower report: who's free,
  nearing, or overloaded; team headroom; and an evidence line you can show upward.
- `/jira assign a tech-design doc to abc` — resolves the nickname, checks that
  person's load + availability first, and proposes options if they're full or away.
- `/jira <anything>` — router: standup, release, capacity, assignment, or an
  ad-hoc Jira request (e.g. `/jira comment on PJ-CORE-123 ...`).

## Nicknames, capacity & availability
- In `teams.yaml`, give each person `aliases` (nicknames) and a `wip_limit`.
  "assign to abc" resolves `abc` to the right Jira account.
- The coordinator measures load (estimates where present, else priority-weighted
  ticket count; blocked tickets are paused, not counted) against `wip_limit`.
- Keep `availability.yaml` current with leave / medical / holiday / meeting /
  outstation entries so the advisor and manpower report know who's actually around.

## Write safety
The plugin reads freely but will not edit, comment, transition, or create any
Jira issue until you approve the specific numbered action in the session.

## Your config is private
`.jira-coordinator/profiles/` and `docs/standups/` are git-ignored. The plugin
(engine) carries zero team data and is the only thing shared.

## Automate the morning digest
Use Claude Code's `/schedule` to run `/jira-coordinator` each working morning. The
scheduled run produces the digest file; you review it and approve any actions
manually.
````

- [ ] **Step 2: Validate the install/usage commands are all documented**

Run:
```bash
grep -Eq "plugin install jira-coordinator" README.md \
 && grep -q "/jira-coordinator" README.md \
 && grep -q "/release-ticket" README.md \
 && grep -q "draft-then-approve\|draft" README.md \
 && grep -qi "capacity\|manpower" README.md \
 && grep -qi "availability" README.md \
 && echo "README OK"
```
Expected: `README OK`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README with install, usage, safety, scheduling"
```

---

## Task 9: Full-plugin validation pass

**Files:** none (verification only)

- [ ] **Step 1: Validate the whole plugin**

Run: `claude plugin validate . 2>/dev/null || echo "validator unavailable"`
Expected: success, or fallback echo.

- [ ] **Step 2: Confirm every command maps to a real skill**

Run:
```bash
ls skills/*/SKILL.md && ls commands/*.md
```
Expected: skills `jira-coordinator`, `release-ticket`; commands `jira`,
`jira-coordinator`, `release-ticket`.

- [ ] **Step 3: Confirm git ignores private config but tracks the fixture**

Run:
```bash
git check-ignore .jira-coordinator/profiles/realteam 2>/dev/null; echo "---"
git ls-files .jira-coordinator/profiles/_fixture/ | head
```
Expected: first command echoes the path (ignored); second lists the fixture files (tracked).

- [ ] **Step 4: End-to-end acceptance (read-only)**

In a connected session, run `/jira standup` against the `_fixture` profile end to
end and confirm a complete digest (six sections, incl. Manpower) is written with
zero Jira writes. Run `/jira capacity` and confirm the manpower report prints with
per-person tiers + evidence line. Run `/jira assign a tech-design doc to abc` and
confirm nickname resolution + away/overloaded options as drafts. Then run
`/release-ticket APP-BO v1.6.46` and confirm it drafts and waits for approval.

- [ ] **Step 5: Final commit / tag**

```bash
git add -A
git commit -m "chore: jira-coordinator v0.1.0 complete" --allow-empty
git tag v0.1.0
```

---

## Deferred items (tracked in spec §13, not built in v1)
- Real environment status names + handoff mapping (lead supplies → fill `environments.yaml`).
- Per-status budgets + SLA table tuning in `conventions.md`.
- Sprint-field discovery for the deadline fallback's "active sprint end" step.
- PJ-OPS2 "Fortress" deploy pattern (secondary; out of v1 scope).
- Confluence / email output targets (spec §2 non-goals for v1).
- Real `wip_limit` values + nicknames per member (lead-authored).
- `availability.yaml` upkeep is manual; calendar-MCP auto-sync is a future option.
- Estimate-field discovery (story points / time) to improve hybrid capacity accuracy.
```
