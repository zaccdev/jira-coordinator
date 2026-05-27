# Jira Coordinator — Design Spec

> _Sanitized for public sharing. All organization-specific data (projects, people, hosts, IDs, paths) is replaced with placeholders; real values live in the private internal copy._

**Date:** 2026-05-27
**Author:** Your Name (Systems Engineer, ExampleApp / your-site.atlassian.net)
**Status:** Approved design — ready for implementation planning

## 1. Purpose

An AI "project coordinator" that runs every working day to gather a team lead's
in-scope Jira tickets, surface what needs attention (deadlines, stale tickets,
unanswered comments, blockers, workload imbalance, release/deploy status), and
propose actions — without taking any write action until the lead approves each
one.

The coordinator's intelligence lives in **context files the lead grows over
time**; the skill is a thin, generic **engine** that reads those files. This
makes it portable: any team lead can adopt it with their own config and their
own Atlassian login.

## 2. Goals / Non-goals

**Goals**
- Daily digest of in-scope work, grouped by sub-team → person.
- Detect: at-risk/overdue deadlines, stale tickets, unanswered comments,
  blocked tickets, in-scope-but-unassigned, status stuck too long.
- Workload view (who is overloaded vs idle).
- **Release management**: (a) *watch* — track open deployment Service Requests
  in the daily digest; (b) *build* — a `/release-ticket` command that scaffolds
  a new deployment ticket (hybrid template + clone-from-last).
- **Dev → QA → Product handoff tracking** across multiple test environments
  (SBX / QA / STG / PROD-regions) via handoff signals.
- **Nickname resolution**: map one person to many nicknames; "assign to abc"
  resolves to the right Jira account.
- **Capacity & availability**: model each person's load (hybrid: estimates where
  present, else priority-weighted ticket count) + a manual availability register
  (leave / medical / holiday / meeting / outstation). Power an assignment advisor
  and a management-facing manpower report.
- **Reviewable write workflow**: every write renders an action-preview card for
  approval by default; an opt-in `auto` mode can skip review (with a stated risk).
- **Role-aware coordination**: store the lead's and members' roles +
  responsibilities so the agent understands who does code review, release
  sign-off, etc., and tailors routing and reporting accordingly.
- Be distributable as a Claude Code plugin so other team leads can adopt it.

**Non-goals (v1)**
- No autonomous writes to Jira.
- No analytics dashboards / charts.
- No cross-tool sync (Slack/Lark/email) in v1 — output is markdown + terminal.
  (Hooks for later.)
- No invented deadlines when no deadline signal exists — flag instead.

## 3. Environment (discovered 2026-05-27)

- Site: `your-site.atlassian.net`, cloudId `<YOUR_CLOUD_ID>`.
- 37 projects. Core product cluster = ExampleApp family.
- Work spreads across multiple projects but a knowable set of ~7 people.
- Custom, support-oriented workflows (e.g. "Escalate to Vendor Ops",
  "Pending QA/Vendor", "Onboarding Verification") — not vanilla To-Do/Doing/Done.
- Sparse due dates (≈3 of 30 recent tickets) → deadline logic cannot rely on the
  `duedate` field alone.

**First profile scope:** `PJ-CORE, PJ-VI, PJ-VL, PJ-GAM, PJ-LIST, PJ-CS2, PJ-CS, PJ-REC, REQ`
(ExampleApp core; PJ-SUP support board excluded for now).

**Release tickets** live separately in `OPS` (DevOps) as issue type
**Service Request**, with approval workflow
`Pending For Approval → To Do → In Verification → Done`. Naming/labels follow a
fixed convention (see §11). `OPS` is added to scope only for the release-watch
view, not the per-person work digest. PJ-OPS2 ("Fortress") deploy Tasks are a
secondary pattern, out of scope for v1.

## 4. Architecture

Two cleanly separated layers: a shareable **engine** and per-lead **config**.

### 4.1 Engine — the Claude Code plugin (identical for everyone)

```
jira-coordinator/                       (the plugin repo)
├── .claude-plugin/
│   ├── plugin.json                     plugin manifest
│   └── marketplace.json                self-hosted marketplace entry
├── skills/
│   ├── jira-coordinator/
│   │   ├── SKILL.md                    engine: orchestrates the daily run
│   │   ├── references/
│   │   │   ├── jql-recipes.md          how to build scope JQL from a profile
│   │   │   ├── signal-rules.md         definitions of each attention signal
│   │   │   └── digest-template.md      output structure
│   │   └── templates/                  blank profile files copied on init
│   │       ├── teams.example.yaml
│   │       ├── scope.example.yaml
│   │       ├── conventions.example.md
│   │       ├── projects.example.md
│   │       ├── release.example.yaml
│   │       ├── release-template.example.md
│   │       └── environments.example.yaml
│   ├── release-ticket/                 the /release-ticket command
│   │   ├── SKILL.md                    build a deployment SR (template+clone)
│   │   └── references/
│   │       └── clone-rules.md          how to find + bump the last release SR
│   └── jira/                           the /jira umbrella router
│       └── SKILL.md                    parse intent → dispatch (see §4.3)
└── README.md                           install + usage for other leads
```

The engine contains **zero team-specific data**.

### 4.2 Config — per-lead, lives in the lead's working project (git-ignored)

```
<lead-project>/
├── .jira-coordinator/
│   └── profiles/<team>/
│       ├── teams.yaml          teams, sub-teams (frontend/backend), members→accountId
│       ├── scope.yaml          project set + JQL templates + signal thresholds
│       ├── conventions.md      status meanings, SLA/deadline rules, "stale = N days"
│       ├── projects.md         glossary: what PJ-CORE/PJ-VI/PJ-CS/REQ mean
│       ├── release.yaml         OPS project, label scheme, service→repo map, mention groups
│       ├── release-template.md  deployment SR body skeleton (placeholders)
│       ├── environments.yaml    env list + status→stage→env handoff mapping
│       ├── availability.yaml    manual register: who is on leave/holiday/meeting/outstation
│       └── settings.yaml        behavior toggles: approval_mode (review|auto), output prefs
└── docs/standups/YYYY-MM-DD.md         digest output
```

Multiple profiles supported; the lead selects one at run time (defaults to the
sole profile if only one exists).

### 4.3 Command surface

Three entry points (the "both" model):

| Trigger | Behavior |
|---|---|
| `/jira-coordinator` | Runs the full daily digest directly. |
| `/release-ticket [version]` | Builds a deployment SR directly. |
| `/jira <intent>` | **Umbrella router.** Parses natural-language intent and dispatches: "standup"/"digest" → coordinator; "release vX.Y.Z" → release-ticket; "capacity"/"manpower"/"who's free" → manpower report (§6c); "assign … to \<name\>" → assignment advisor (§6c). **Fallthrough:** if the intent matches no known workflow, treat it as an ad-hoc Jira request and fulfill it via the MCP directly (e.g. `/jira comment on PJ-CORE-123 ...`), still under the §9 approve-before-write rule. |

The router is a thin dispatcher: it owns intent parsing only, then hands off to
the coordinator or release-ticket skill (or raw MCP). No workflow logic is
duplicated in it.

### 4.4 Auth

Each lead authenticates through their own Atlassian MCP connection. No
credentials, API keys, or accountIds are shared between leads. The coordinator
only ever sees what that lead's Jira permissions allow.

## 5. Config schemas

### teams.yaml
```yaml
team: ExampleApp Core
lead:
  name: Your Name
  accountId: <ACCOUNT_ID>
  role: BackOffice Backend Lead
  responsibilities: [code_review, release_signoff, assign_work]
subteams:
  backend:
    - { name: alice, accountId: "<resolved-on-init>", aliases: [wei, sc], wip_limit: 5,
        role: Backend Engineer, responsibilities: [implementation] }
    - { name: bob, accountId: "<resolved-on-init>", aliases: [jl], wip_limit: 5,
        role: Backend Engineer, responsibilities: [implementation] }
  frontend:
    - { name: Carol, accountId: "<resolved-on-init>", aliases: [abc, kn], wip_limit: 4,
        role: BackOffice Frontend Lead, responsibilities: [code_review, implementation] }
# accountIds resolved during init via lookupJiraAccountId.
# aliases  = nicknames the lead uses; resolver maps any alias -> this member.
# wip_limit = max concurrent active tickets before "overloaded" (capacity, §6c).
# role/responsibilities power role-aware routing (§6c) and reporting.
# responsibility tags (suggested vocab): implementation, code_review, pr_approval,
#   release_signoff, deployment, qa_signoff, assign_work, sprint_planning, vendor_liaison.
```

### settings.yaml
```yaml
# Behavior toggles for this profile.
approval_mode: review     # review = preview every write + ask (DEFAULT, SAFE);
                          # auto   = execute writes without asking. RISK: the agent
                          #          can comment/transition/reassign/create tickets
                          #          on its own. Misreads become live Jira changes.
auto_allow: []            # when approval_mode: auto, optionally limit auto-exec to
                          # specific action types, e.g. [comment]; others still prompt.
output: [markdown, terminal]   # where the digest lands
```

### scope.yaml
```yaml
projects: [PJ-CORE, PJ-VI, PJ-VL, PJ-GAM, PJ-LIST, PJ-CS2, PJ-CS, PJ-REC, REQ]
# base filter the engine ANDs with per-view JQL
base_jql: "project in ({projects}) AND statusCategory != Done"
thresholds:
  stale_days: 3            # no update in N working days => stale
  deadline_warning_days: 2 # due within N days => at-risk
include_unassigned: true   # surface in-scope unassigned tickets
overrides_allowed: true    # lead may pass raw JQL at run time
```

### conventions.md (free-form, lead-authored)
Documents: what each custom status means, the deadline fallback chain, SLA by
issue type, and any team norms. The engine reads this as guidance, not schema.

## 6. Daily run flow (SKILL.md)

1. **Select profile** (or the only one).
2. **Load context** from the profile files.
3. **Resolve scope** → build JQL from `scope.yaml` + roster → query Jira
   (`searchJiraIssuesUsingJql`). Page through results; large outputs are written
   to file and parsed, never dumped into context.
4. **Enrich** each ticket: status, assignee, last comment (author + age), last
   update age, due date / sprint end, blockers, links.
5. **Detect signals** (see §7).
6. **Group** by sub-team → person → workload.
7. **Emit digest** (see §8) to `docs/standups/YYYY-MM-DD.md` and terminal.
8. **Proposed actions** are drafts only. On explicit approval ("do #2, #4") the
   engine executes via the write tools (see §9).

## 6a. Release management

### Release watch (read, part of daily digest)
The daily run queries `OPS` for open deployment Service Requests related to the
profile's services and reports: version, target env/regions, approval status,
and flags any stuck at **Pending For Approval**. Driven by `release.yaml`.

### Release build — `/release-ticket` (write, on demand)
Hybrid template + clone-from-last:
1. **Skeleton** comes from `release-template.md` (the fixed body structure:
   "deploy tag {version} on {date}" → regional rollout sequence → release-report
   link → Deployment service block → Attn / QA mentions).
2. **Pre-fill** by cloning the most recent matching deployment SR (per
   `release.yaml` match rule, e.g. summary contains "APP-BO"): carry forward
   service→tag lines, mention groups, region sequence; bump version/tags/date
   from command args.
3. **Labels** set from `release.yaml` (`myapp`, region labels, automation marker).
4. Output a drafted ticket. Under §9, create via `createJiraIssue` **only on
   approval**. Never auto-creates.

### release.yaml (schema sketch)
```yaml
project: OPS
issue_type: Service Request
match_rule: 'summary ~ "APP-BO" AND issuetype = "Service Request"'  # find last release
labels: [myapp, n8n-automation]
region_sequence:                      # default rollout order + times
  - { region: Region1,   label: region-1, time: "10:00" }
  - { region: Region2,    label: region-2,     time: "11:00" }
  - { region: Region3,    label: region-3,     time: "14:00" }
  - { region: Region4,  label: region-4,   time: "15:00" }
  - { region: Region5, label: region-5,  time: "16:00" }
services:                             # service → gitlab repo for tag URLs
  service-a: "https://git.example.com/your-group/service-a"
  service-b: "https://git.example.com/your-group/service-b"
mentions:
  dev: ["<accountId>", ...]
  qa:  ["<accountId>", ...]
```

## 6b. Dev → QA → Product handoff

The coordinator models a ticket's lifecycle across environments and raises a
**handoff signal** when a ticket is ready to move but hasn't. Environments and
the status→stage mapping are lead-provided (deferred) and stored in
`environments.yaml`.

```yaml
# environments.yaml — VALUES BELOW ARE PLACEHOLDERS, lead confirms later
environments: [SBX, QA, STG, PROD]
stages:
  - { name: dev,        ready_status: "Integration Done", next: QA,  owner: backend }
  - { name: qa,         ready_status: "Ready for QA",      next: STG, owner: qa }
  - { name: staging,    ready_status: "QA Passed",         next: PROD, owner: qa }
  - { name: production, ready_status: "Approved for PROD", next: null, owner: devops }
```

Handoff signals emitted: a ticket sitting in a `ready_status` past its budget
("done dev N days, not picked up by QA"), or deployed to an env with no sign-off.
Exact status names and env count are confirmed when the lead supplies them
(§13); the engine treats `environments.yaml` as the source of truth.

## 6c. Nickname resolution, capacity & availability

### Name resolution
Any name token in a command ("assign to abc") resolves in this order:
1. Match against every member's `aliases` and `name` in `teams.yaml` (case-insensitive).
2. If no match, try `lookupJiraAccountId` on the literal token.
3. If still ambiguous or empty, ask the lead. Never assign to a guessed account.

### Capacity model (hybrid)
Per person, compute **active load** from their open, non-Done assigned tickets:
- If a ticket has a story-point / time estimate, use it.
- Otherwise weight by priority (Highest=3, High=2, Medium=1, Low=0.5).
- **Blocked** tickets are counted as "on plate but paused" — listed, but excluded
  from active load (they are not consuming current effort).

Load tier: compare active count (or weighted sum) against the member's `wip_limit`:
`free` (< 60%), `nearing` (60–100%), `overloaded` (> 100%).

### Availability register (`availability.yaml`)
A manual, lead- or member-maintained file of time-bound statuses:
```yaml
# availability.yaml
entries:
  - { who: Carol, type: annual_leave, from: 2026-05-28, to: 2026-05-30 }
  - { who: alice,      type: medical_leave, from: 2026-05-27, to: 2026-05-27 }
  - { who: bob,      type: outstation,    from: 2026-05-27, to: 2026-05-29, note: client site }
# types: annual_leave | medical_leave | holiday | meeting | outstation | available
```
A person with an active entry overlapping "today" is **unavailable**; the advisor
and reports factor this in (e.g. don't assign new work to someone on leave).

### Role-aware routing
Roles + `responsibilities` (teams.yaml) shape who work goes to:
- Tasks implying a responsibility route to people who hold it — e.g. "needs code
  review" → members with `code_review`; "release sign-off" → `release_signoff`.
- The advisor will not propose assigning a responsibility-tagged task to someone
  who lacks that tag without flagging it.
- The digest frames the lead's own duties by their role (e.g. surfaces items
  awaiting *their* code review / sign-off first).

### Assignment advisor
When asked to assign work to a person:
1. Resolve the name (above).
2. Check the work's implied responsibility against the target's `responsibilities`.
3. Check availability register for today.
4. Compute their load tier.
5. If unavailable, overloaded, or missing the needed responsibility, do NOT
   silently assign — present options as numbered cards (§9): assign anyway · put a
   named lower-priority ticket On Hold · re-rank their queue by priority · suggest
   a teammate with headroom (and the right role).
6. Execute only the chosen option on approval (reassign = `editJiraIssue`,
   On Hold = `transitionJiraIssue`), per §9.

### Manpower report (management-facing)
A presentable capacity snapshot, available as a digest section and on demand
(`/jira capacity` or `/jira manpower`):
- Per-person: load tier, active count, blocked count, availability status.
- Team totals: headroom remaining, % at/over capacity, count blocked on external
  parties (vendor/QA/product).
- An evidence line for management, e.g. "Team at 100% capacity; 6 tickets blocked
  on vendor; new product requests will queue until capacity frees."

## 7. Signal detection rules

| Signal | Rule |
|---|---|
| Overdue | due date / sprint end < today, not Done |
| At-risk deadline | due within `deadline_warning_days`, not progressing |
| Stale | no update in ≥ `stale_days` working days |
| Unanswered comment | latest comment author ≠ assignee, after assignee's last activity |
| Blocked | "is blocked by" link open, or blocked status/flag |
| In-scope unassigned | matches scope, assignee empty (if `include_unassigned`) |
| Stuck status | in same status beyond a per-status budget in conventions.md |
| Handoff pending | in a stage `ready_status` past budget, not advanced (§6b) |
| Release approval stuck | OPS deployment SR at "Pending For Approval" past budget (§6a) |
| Person overloaded | active load > `wip_limit` (§6c) |
| Assigned-but-away | open assigned ticket where assignee is unavailable today (§6c) |

**Deadline fallback chain:** `duedate` → active sprint end → SLA-by-issue-type
table (conventions.md) → flag "no deadline signal" (never invent one).

## 8. Digest format

Sections, in order:
1. **Needs your attention today** — ranked, with one-line why + ticket link.
2. **Per-person status** — grouped by sub-team; in-progress, blocked, stale.
3. **Manpower / capacity** — per-person load tier + availability, team headroom,
   external-blocked count, and the management evidence line (§6c).
4. **At-risk deadlines** — overdue + upcoming.
5. **Release / deploy watch** — release/deploy tickets and their blockers.
6. **Proposed actions** — numbered drafts: comment text, deadline change,
   reassignment, On Hold. Nothing executed yet.

## 9. Write safety

Hard rule in both skills: read tools (`searchJiraIssuesUsingJql`, `getJiraIssue`,
comment/transition *reads*) may be called freely. Write tools (`editJiraIssue`,
`addCommentToJiraIssue`, `transitionJiraIssue`, `createIssueLink`,
`createJiraIssue`) are governed by `settings.yaml.approval_mode`.

### Action-preview card (always rendered before a write)
Every proposed write is shown as a numbered card in the terminal/chat, so the lead
sees exactly what will happen:

```
[#2] COMMENT → PJ-CORE-123  "FeatureX Summary"
  As: Your Name
  Body:
    > Deployed v1.6.46 to QA; please verify by EOD.

[#3] TRANSITION → PJ-CORE-130   In Progress → On Hold
  Reason: blocked by vendor (PJ-VI-44)

[#4] REASSIGN → PJ-VI-51     bob → alice   (bob overloaded: 6/5)
```
Field shape by type: COMMENT shows author + full body; TRANSITION shows
`from → to` + reason; REASSIGN/EDIT shows `old → new`; CREATE shows project, type,
summary, and rendered body. After the cards: `approve <#s> / edit <#> / skip <#> /
approve all / cancel`.

### approval_mode
- **`review`** (default): render the cards and execute ONLY the items the lead
  approves this session. Approval never carries across runs.
- **`auto`**: execute writes without prompting. The skill MUST print a one-line
  risk banner each run ("⚠ auto mode: writes execute without review"). If
  `auto_allow` is set, only those action types auto-run; all others still prompt.
  `createJiraIssue` is never auto-run unless `create` is explicitly in `auto_allow`.

Regardless of mode, every executed write is echoed back with the resulting ticket
link so there is always an audit trail in the session.

## 10. Init / onboarding mode

On first run (no profile found) the skill enters init mode and does what was
done manually on 2026-05-27:
1. Calls `getAccessibleAtlassianResources` + `atlassianUserInfo`.
2. Lists the lead's projects and recent assignees.
3. Interactively scaffolds a profile from the templates, resolving accountIds
   via `lookupJiraAccountId`.
4. Captures the **lead's role** (and suggests responsibility tags from a default
   role→responsibilities map for confirmation), then each member's role.
5. Confirms `approval_mode` (defaults to `review`; explains the `auto` risk before
   accepting it).
No lead ever edits a blank file from scratch.

## 11. Distribution (Claude Code plugin)

Engine ships as a plugin via a shared `marketplace.json`. Other leads install
with a single command and receive updates. Their config profiles stay local and
git-ignored — never part of the plugin.

## 12. Scheduling

Once trusted, the lead uses `/schedule` to run the coordinator each working
morning (Mon–Fri). The scheduled run produces the digest file; the lead reviews
and approves actions manually. Automation produces; the human disposes.

## 13. Open items (resolved during implementation)

- Exact accountIds for roster members (resolved in init).
- Per-status time budgets for the "stuck status" signal (lead-authored).
- SLA-by-issue-type table (lead-authored in conventions.md).
- Sprint-field discovery (board/sprint API specifics) for deadline fallback.
- **Environment list + status→stage mapping** for handoff tracking (§6b) — lead
  supplies; spec values are placeholders.
- Release `match_rule` per product line (APP-BO vs vendor-service vs service-d).
- Per-member `wip_limit` values and nickname `aliases` (lead-authored, §6c).
- `availability.yaml` is maintained manually and kept current by the lead/members (§6c).
- Whether tickets carry usable story-point/time estimates (affects hybrid capacity
  accuracy; engine falls back to priority-weighted count when absent).
- Roles + responsibility tags for the lead and each member (lead-authored at init, §6c).
- `approval_mode` default is `review`; `auto`/`auto_allow` are opt-in with a stated risk (§9).

## 14. Milestones

1. Plugin scaffold (`plugin.json`, `marketplace.json`, two skill skeletons, README).
2. Profile schema + templates + init mode.
3. Scope resolution + Jira query + enrichment.
4. Signal detection + digest generation (read-only end to end).
5. Approve-and-execute write workflow: action-preview cards + `approval_mode`
   (review default / auto opt-in) (§9).
6. Release watch in digest (§6a) + handoff signals (§6b).
7. Roles + nickname resolution + capacity/availability + role-aware assignment
   advisor + manpower report (§6c).
8. `/release-ticket` command — hybrid template + clone-from-last.
9. Scheduling + multi-lead distribution polish.
