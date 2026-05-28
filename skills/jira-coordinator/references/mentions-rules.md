# Mentions mode — surface @-tags awaiting the lead

Read-only mode. No writes are ever generated. Output prints to terminal and
also writes `docs/mentions/{today}.md`.

## Inputs
- Source: `jira` | `confluence` | `both` (default if user did not specify).
- Lookback window: parse `<N>d` or `<N> days` from the user's input; default `7d`.
- Lead identity: prefer `teams.yaml.lead.accountId`. If absent, call
  `atlassianUserInfo` once and use its `account_id`. Also use `displayName` for
  human-readable framing.
- Cloud ID: from `settings.yaml.cloud_id` if present, else resolve at runtime
  (`getAccessibleAtlassianResources`).

## Step M1 — Confluence side (when source ∈ {confluence, both})
Single CQL call:
```cql
mention = currentUser() AND lastModified > now("-{N}d") ORDER BY lastModified DESC
```
Tool: `searchConfluenceUsingCql`, `responseContentFormat: "markdown"`,
`maxResults <= 25`. For each result keep: `title`, `space.key`, `lastModified`,
`url`, `excerpt` (if available). No enrichment needed — CQL already proves the
mention exists.

## Step M2 — Jira side (when source ∈ {jira, both})

Jira has NO direct JQL operator for "@-mentioned me in a comment". So:

1. **Bound the scan** with the profile's `scope.yaml.base_jql`, narrowed by
   `updated >= -{N}d`. If `base_jql` contains `{projects}`, substitute it the
   same way the digest does. Order by `updated DESC`, `maxResults <= 50`.
   Fetch fields: `["summary","status","assignee","updated"]`.
2. **Enrich** the top results with `getJiraIssue`, requesting
   `fields: ["comment","updated","assignee"]` and `responseContentFormat: "adf"`
   (ADF is required to read `mention` nodes). Cap enrichment at 30 tickets —
   if more match, note "scan truncated" in the output.
3. **Scan comments**. For each comment, walk the ADF tree and collect every
   `{type: "mention", attrs: {id: <accountId>}}` node. A comment is a HIT iff:
   - It contains a mention with `id == lead.accountId`, AND
   - Its `created` timestamp is later than the lead's own latest activity on
     this ticket (latest of: lead's prior comment, or the comment we are
     scanning if the lead authored it — exclude self-mentions).
4. **Deduplicate** by `(ticketKey, commentId)`. Sort newest-first.

For each HIT keep: `ticketKey`, `summary`, `status`, `assignee.displayName`,
`comment.author.displayName`, `comment.created`, a one-line excerpt of the
comment text (strip ADF marks down to plain text, trim to ~120 chars),
and a deep link `<jira-base>/browse/{ticketKey}?focusedCommentId={commentId}`.

## Step M3 — Render

Write `docs/mentions/{YYYY-MM-DD}.md` AND print to terminal. Sections:

```
# Pings — {YYYY-MM-DD}
Window: last {N} days · Source: {jira|confluence|both}

## Jira mentions  ({hit-count}{, truncated if applicable})
- [{KEY}] {summary} · {status} · @-mentioned by {author} at {created}
  > {excerpt}
  {focused-comment link}

## Confluence mentions  ({hit-count})
- [{space.key}] {title} · last modified {lastModified}
  {url}
```

If a section has 0 hits, print: `_No new mentions in this window._`

## Hard constraints
- Read tools only. Even if the user says "reply to them", switch to digest's
  Proposed-actions flow — never auto-reply.
- If `atlassianUserInfo` returns 401, stop and tell the user to re-auth the
  Atlassian MCP; do not proceed with a guessed accountId.
- Token budget: never echo enrichment payloads back into context. Parse with
  python/jq, keep only the fields above.
