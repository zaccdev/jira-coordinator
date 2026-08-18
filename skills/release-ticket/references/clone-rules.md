# Clone-and-bump rules for /release-ticket

Hybrid: the skeleton comes from the profile's `release-template.md`; field values
are pre-filled by cloning the most recent matching release SR.

## Find the last release
Pick the product line from the user's request (e.g. "gabo", "ga-bo-sas-service").
Look up `release.yaml.match_rules[<product>]` and run it via
`searchJiraIssuesUsingJql` (maxResults 1). Fetch its body + custom fields with
`getJiraIssue` (`fields: ["summary","description","labels","customfield_10087",
"customfield_10171"]`, `responseContentFormat: "markdown"`, `expand: "names"`).

## Carry forward vs bump
- CARRY: region sequence, dev/QA mentions, service list, branch notes, Purpose,
  Project & Environment checkbox selection.
- BUMP from user args: `{version}`, `{date}`, `{weekday}`, per-service `{tag}`
  values, and the release-report link (new fix-version page id).

## Required custom fields (createJiraIssue 400s without these)
The DevOps "Service Request" type requires:
- `release.yaml.required_fields.purpose` (customfield_10087) — free text, default
  `release.yaml.purpose_default` ("Deployment").
- `release.yaml.required_fields.project_environment` (customfield_10171) — pass the
  `{id}` objects from `release.yaml.env_options` for the regions being deployed,
  e.g. `[{"id":"10367"},{"id":"10369"}]`.

## Build the body
Fill `release-template.md` placeholders:
- `{region_sequence}` -> numbered lines from `release.yaml.region_sequence`
  ("1. APAC - 10:00am"), only the regions being deployed.
- `{release_report_link}` -> `release.yaml.release_report_link` with the new
  fix-version id.
- `{service_blocks}` -> for each service in args:
  `"{name} (tag: {tag})"` then a `"tag : {repo}/-/tags/{tag_prefix}{tag}"` line,
  using `release.yaml.services[name].repo` and `.tag_prefix`. List multiple tags
  (e.g. `1.6.46, 1.6.46-cv`) with one tag line each.
- `{branch_notes}` -> `release.yaml.branch_notes` (omit if the deploy has no -cv
  / vectorcore split).
- `{dev_mentions}` / `{qa_mentions}` -> from `release.yaml.mentions` (or carried).
  Use account ids where present so @-mentions notify rather than render as text.

## Labels
`release.yaml.labels` + the region labels for the regions being deployed
(`region_sequence[].label`, skipping null).
