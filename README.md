# Jira Coordinator

An AI **project-coordinator plugin for [Claude Code](https://claude.com/claude-code)**.
It acts like a daily project coordinator on top of your Jira: gathers your team's
in-scope tickets, surfaces what needs attention, tracks capacity and availability,
helps assign work, and prepares release tickets — all under a strict
**draft-then-approve** rule, so it never changes Jira without your confirmation.

> ⚠️ **Status: work in progress.** This repository currently contains only the
> project scaffold. The plugin engine is being built from a design that lives
> outside this repo.

## What it will do

- **Daily digest** — pulls your in-scope tickets and reports, grouped by sub-team
  and person, what needs attention today.
- **Attention signals** — overdue / at-risk deadlines, stale tickets, unanswered
  comments, blocked work, in-scope-but-unassigned, status stuck too long.
- **Capacity & availability** — models each person's workload (estimates where
  present, otherwise priority-weighted ticket count; blocked items are treated as
  paused) against a per-person WIP limit, plus a manual availability register
  (leave / holiday / meeting / etc.).
- **Assignment advisor** — resolve a person by name or **nickname**, check whether
  they have capacity or are away, and propose options before assigning.
- **Manpower report** — a management-facing capacity snapshot: who's free vs
  overloaded, team headroom, and how much work is blocked on external parties.
- **Release tickets** — watch in-flight deployment tickets, and a builder command
  that drafts a new one from a template + your last release.

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

## Requirements

- Claude Code with the Atlassian MCP connected.

## Privacy

Your team configuration (rosters, project scope, availability) lives under
`.jira-coordinator/profiles/` and is git-ignored. The engine ships with **no**
organization-specific data — only placeholder templates you fill in locally.

## License

MIT — see [LICENSE](./LICENSE).
