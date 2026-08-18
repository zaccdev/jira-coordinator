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
Open deployment Service Requests (from `release.yaml.project`) for the profile's
services: version, target env/regions, status. Flag any at a `watch_status_budgets`
status past budget.

## 6. Proposed actions
Numbered drafts only. Each: action type (comment / due-date change / reassign /
transition / On Hold), target KEY, and exact proposed content. NOTHING is executed here.
