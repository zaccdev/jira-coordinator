# Team conventions (lead-authored guidance, read by the engine)

## Status meanings
- "Blocked - External": waiting on a vendor/third party; not a team blocker.
- "In Review": handed off, awaiting verification.
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
| In Review | 2 |
