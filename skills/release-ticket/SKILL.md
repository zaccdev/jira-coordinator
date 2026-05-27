---
name: release-ticket
description: Use when the user wants to create or prepare a Jira release/deployment ticket (a Service Request in the configured release project) — e.g. "build a release ticket for app-bo v1.6.46". Clones the last matching deployment SR and bumps version/tags/date using a template. Drafts the ticket; creates it only after the user approves.
---

# Release Ticket Builder

Reference: `references/clone-rules.md`. Config: the active profile's `release.yaml`
and `release-template.md`.

## Resolve cloudId at runtime
Call `getAccessibleAtlassianResources` and use the returned `id` (or the cloudId
cached in `settings.yaml`). Never hardcode an organization's cloudId or site.

## Write safety (HARD RULE)
Render the drafted ticket as a CREATE action-preview card and obey
`settings.yaml.approval_mode`. Never call `createJiraIssue` until the user approves
in THIS session — and never auto-create unless `create` is explicitly in
`auto_allow` (see the jira-coordinator skill's "Write safety").

## Steps
1. Select the active profile (as in jira-coordinator Step 0).
2. Parse the user's request for product line + version + target env + date.
   Ask for any of these that are missing.
3. Follow `references/clone-rules.md` to find the last release and build the body.
4. Present the FULL drafted ticket: project, issue type, summary
   (e.g. "[PROD] Deployment for app-bo v1.6.46"), labels, and the rendered body.
5. On approval, call `createJiraIssue` with those fields and return the new ticket
   link. On rejection or edits, revise and re-present.
