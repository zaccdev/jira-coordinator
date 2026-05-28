---
name: jira-coordinator
description: Use when the user wants a daily Jira standup/digest, to see what needs attention across their team's tickets (deadlines, stale tickets, unanswered comments, blockers, workload), to check who has capacity / is overloaded / is on leave, to advise on assigning work to a person (by name or nickname), to produce a manpower report for management, to surface @-mentions of the lead in Jira and/or Confluence, or to watch release tickets. Reads a team profile from .jira-coordinator/profiles/ and reports via a dated markdown digest. Read + draft only — never writes to Jira without per-action approval.
---

# Jira Coordinator

Generate a daily coordinator digest for a team lead's in-scope Jira work, plus
capacity/availability analysis, an assignment advisor, and a manpower report.

Reference files (read as needed): `references/jql-recipes.md`,
`references/signal-rules.md`, `references/digest-template.md`,
`references/capacity-rules.md`, `references/mentions-rules.md`.

## Resolve cloudId at runtime (never hardcode)
Call `getAccessibleAtlassianResources` and use the returned `id` as `cloudId` for
all Jira calls. If the user has one site, use it; if several, ask which. You may
cache the chosen cloudId in the profile's `settings.yaml` (`cloud_id:`), but the
engine itself carries no organization-specific values.

## Write safety (HARD RULE)
Read tools may be called freely. Every write (`editJiraIssue`,
`addCommentToJiraIssue`, `transitionJiraIssue`, `createIssueLink`,
`createJiraIssue`) is first rendered as an ACTION-PREVIEW CARD, then governed by
`settings.yaml.approval_mode`.

Action-preview card (numbered), shown before any write:
```
[#2] COMMENT    -> KEY    As: <author>   Body: <full text>
[#3] TRANSITION -> KEY    <from> -> <to>  Reason: <why>
[#4] REASSIGN   -> KEY    <old> -> <new>  (<why>)
[#5] CREATE     -> PROJ   type / summary / rendered body
```
After the cards, prompt: `approve <#s> / edit <#> / skip <#> / approve all / cancel`.

approval_mode:
- `review` (DEFAULT): execute ONLY the items the user approves THIS session.
  Approval never carries across runs.
- `auto`: execute without prompting, but print a one-line risk banner first
  ("auto mode: writes execute without review"). If `auto_allow` is set, only those
  action types auto-run; others still prompt. NEVER auto-run `createJiraIssue`
  unless `create` is explicitly in `auto_allow`.

In every mode, echo each executed write back with its ticket link. When in doubt,
draft — do not write.

## Modes
Do Step 0–1 (profile + context) in every mode, then:
- **Digest** (default / "standup" / "digest"): Steps 2–7 — the full digest.
- **Manpower** ("capacity" / "manpower" / "who's free" / "who's overloaded"):
  Step 4b only, then print just the Manpower/capacity report from
  `references/capacity-rules.md`.
- **Assignment advisor** ("assign \<work\> to \<name\>"): resolve the name and run
  the advisor flow in `references/capacity-rules.md`; present options, execute only
  the approved one per the write rule.
- **Mentions** ("mentions" / "mention" / "tags" / "pings"): follow
  `references/mentions-rules.md`. Parse the trailing word for source
  (`jira` / `confluence`, default = BOTH) and optional `<N>d` lookback (default 7d).
  Read-only; no writes generated.

## Step 0: Select profile
1. List `.jira-coordinator/profiles/*/` (ignore `_fixture` unless asked).
2. If none exist -> go to "Init mode" below.
3. If one exists -> use it. If several -> ask which.

## Step 1: Load context
Read the profile's `teams.yaml` (incl. roles/responsibilities), `scope.yaml`,
`conventions.md`, `projects.md`, `release.yaml`, `environments.yaml`,
`availability.yaml`, and `settings.yaml` (approval_mode, output).

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
Render the "Proposed actions" as action-preview cards and apply `approval_mode`
(see Write safety). Execute only approved/auto-allowed items via the matching
write tool, then confirm each with its ticket link.

## Init mode (first run, no profile)
1. `getAccessibleAtlassianResources` + `atlassianUserInfo` to confirm site + user;
   record the cloudId.
2. Ask the team name; create `.jira-coordinator/profiles/<team>/` by copying the
   files from the plugin's `templates/` directory.
3. Help fill `teams.yaml`: for each member, resolve accountId via
   `lookupJiraAccountId`, and ask for nicknames (`aliases`), a `wip_limit`, and a
   `role`. Suggest `responsibilities` tags from a default role→responsibilities map
   (e.g. a "Lead" → code_review, release_signoff, assign_work) for confirmation.
4. Capture the lead's own `role` + responsibilities the same way.
5. Confirm `scope.yaml.projects` (the lead's real Jira project keys).
6. Confirm `settings.yaml.approval_mode` — default `review`; if the user wants
   `auto`, state the risk first (writes execute without review) and record it.
7. Tell the user which placeholders still need filling: `conventions.md` (status
   meanings, SLAs, budgets), `environments.yaml` (stage status names — handoff
   signals stay inactive until confirmed), and `availability.yaml` (kept current
   manually — empty register is fine to start).
No lead ever edits a blank file from scratch.
