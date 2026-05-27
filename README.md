# Jira Coordinator

An AI **project-coordinator plugin for [Claude Code](https://claude.com/claude-code)**.
It acts like a daily project coordinator on top of your Jira: gathers your team's
in-scope tickets, surfaces what needs attention, tracks capacity and availability,
helps assign work, and prepares release tickets — all under a strict
**draft-then-approve** rule, so it never changes Jira without your confirmation.

## Requirements
- Claude Code with the Atlassian (Rovo) MCP connected (`/mcp` to check).

## Install
```
/plugin marketplace add https://github.com/zaccdev/jira-coordinator
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
- `/release-ticket app-bo v1.6.46` — draft a deployment ticket from the last one.
- `/jira capacity` (or `manpower`) — management-facing manpower report: who's free,
  nearing, or overloaded; team headroom; and an evidence line you can show upward.
- `/jira assign a tech-design doc to <name>` — resolves the nickname, checks that
  person's load + availability first, and proposes options if they're full or away.
- `/jira <anything>` — router: standup, release, capacity, assignment, or an
  ad-hoc Jira request (e.g. `/jira comment on PROJ-123 ...`).

## Roles, nicknames, capacity & availability
- In `teams.yaml`, give each person a `role` + `responsibilities` (e.g.
  `code_review`, `release_signoff`), `aliases` (nicknames), and a `wip_limit`.
  "assign to al" resolves `al` to the right Jira account, and the coordinator
  routes work by responsibility (e.g. review tasks go to reviewers).
- Load is measured (estimates where present, else priority-weighted ticket count;
  blocked tickets are paused, not counted) against `wip_limit`.
- Keep `availability.yaml` current with leave / medical / holiday / meeting /
  outstation entries so the advisor and manpower report know who's actually around.

## Write safety
By default the plugin reads freely but renders every write as an **action-preview
card** and executes nothing until you approve it in the session. You can change
this in `settings.yaml`:
- `approval_mode: review` (default) — preview + approve every write.
- `approval_mode: auto` — execute without prompting. ⚠ Risk: the agent can
  comment/transition/reassign on its own; misreads become live Jira changes. Use
  `auto_allow: [comment]` to auto-run only safe action types; `create` is never
  automatic unless you explicitly list it.

## How it works

The plugin is a thin, generic **engine**; all team-specific knowledge lives in
**config profiles** on your machine and is never shared. Jira access is through
the Atlassian MCP, authenticated as you — no credentials are committed or shared.

```
engine (shared, in this repo)        config (private, on your machine — git-ignored)
  skills/        SKILL.md logic         .jira-coordinator/profiles/<team>/
  commands/      /jira entry points       teams.yaml, scope.yaml, conventions.md, ...
  templates/     blank profile files
```

## Privacy
Your team configuration (rosters, project scope, availability) lives under
`.jira-coordinator/profiles/` and is git-ignored. The engine ships with **no**
organization-specific data — only placeholder templates you fill in locally.

## Automate the morning digest
Use Claude Code's `/schedule` to run `/jira-coordinator` each working morning. The
scheduled run produces the digest file; you review it and approve any actions
manually.

## License
MIT — see [LICENSE](./LICENSE).
