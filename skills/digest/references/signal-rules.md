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
