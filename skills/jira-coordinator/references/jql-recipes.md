# Building scope JQL from a profile

Inputs: `scope.yaml` (`projects`, `base_jql`, `thresholds`, `include_unassigned`)
and `teams.yaml` (roster accountIds).

## Project clause
Substitute `{projects}` in `base_jql` with the comma-joined `projects` list.
Example result:
`project in (PROJ1,PROJ2,PROJ3) AND statusCategory != Done`

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
