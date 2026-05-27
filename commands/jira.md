---
description: Jira coordinator router — standup, release, capacity, assignment, or any ad-hoc Jira request.
---
You are the `/jira` router. Parse the user's intent from: $ARGUMENTS

Dispatch:
- "standup" / "digest" / "today" / empty -> invoke `jira-coordinator` (Digest mode).
- "release ..." / "deploy ..." -> invoke the `release-ticket` skill.
- "capacity" / "manpower" / "who's free" / "who's overloaded" -> invoke
  `jira-coordinator` (Manpower mode).
- "assign ... to <name>" -> invoke `jira-coordinator` (Assignment advisor mode);
  resolve the name/nickname, check capacity + availability, present options.
- Anything else (e.g. "comment on PROJ-123 ...", "create a bug in PROJ2 ...") ->
  treat as an ad-hoc Jira request and fulfill it directly via the Atlassian MCP.

HARD RULE: in all paths, every write (editJiraIssue, addCommentToJiraIssue,
transitionJiraIssue, createIssueLink, createJiraIssue) goes through the
action-preview card and `settings.yaml.approval_mode` (review by default). Do not
execute a write the user hasn't approved unless approval_mode/auto_allow permits it.
