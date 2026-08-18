# Nickname resolution, roles, capacity, availability, assignment advisor, manpower report

## Name resolution (used wherever a person is named)
Given a token like "al":
1. Case-insensitive match against every member's `name` and `aliases` in `teams.yaml`.
2. If no match, try `lookupJiraAccountId` on the literal token.
3. If ambiguous (>1 match) or empty, ask the lead. NEVER assign to a guessed account.

## Roles & responsibilities (role-aware routing)
Each member (and the lead) has a `role` string and `responsibilities` tags in
`teams.yaml`. Use them to route work and frame reports:
- Infer the responsibility a piece of work implies (e.g. "tech design doc review"
  / "needs review" -> `code_review`; "sign off release" -> `release_signoff`).
- Prefer assignees who hold that responsibility. If the named target lacks it,
  flag this in the advisor options (don't silently assign).
- In the digest, surface items awaiting the LEAD's own responsibilities first
  (e.g. tickets needing their `code_review` or `release_signoff`).

## Capacity model (hybrid)
For each member, gather open non-Done tickets where they are assignee. Compute
**active load**:
- If a ticket has a story-point or time estimate, use that value.
- Else weight by priority: Highest=3, High=2, Medium=1, Low=0.5, Lowest=0.5.
- BLOCKED tickets ("is blocked by" unresolved, or blocked status/flag) are listed
  as "paused" and EXCLUDED from active load.

Load tier vs the member's `wip_limit` (interpret wip_limit as the active-load ceiling):
- `free` if load < 60% of wip_limit
- `nearing` if 60–100%
- `overloaded` if > 100%

## Availability
Read `availability.yaml`. A member is **unavailable today** if any entry has
`who` == member name AND `from <= today <= to`. Record the `type`
(annual_leave / medical_leave / holiday / meeting / outstation).

## Assignment advisor (when asked to assign work to someone)
1. Resolve the name. 2. Check the work's implied responsibility vs the target's
   `responsibilities`. 3. Check availability. 4. Compute load tier.
5. If unavailable OR overloaded OR missing the needed responsibility, do NOT
   silently assign. Present numbered options:
   (a) assign anyway; (b) put a specific lower-priority ticket On Hold
   (`transitionJiraIssue`); (c) re-rank their queue by priority (propose order);
   (d) name a teammate who is `free` AND holds the needed responsibility.
6. Execute only the approved option (reassign = `editJiraIssue`, On Hold =
   `transitionJiraIssue`) per the write-safety rule (action-preview card).

## Manpower report (management-facing)
- Per person: tier, active count, blocked count, availability.
- Team: total headroom (sum of remaining capacity), % members at/over capacity,
  count of tickets blocked on external parties (vendor / QA / product).
- Evidence line, e.g.: "Team at 100% capacity; 6 tickets blocked on vendor; new
  product requests will queue until capacity frees." State numbers, not opinions.
