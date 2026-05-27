# Clone-and-bump rules for /release-ticket

Hybrid: the skeleton comes from the profile's `release-template.md`; field values
are pre-filled by cloning the most recent matching release SR.

## Find the last release
Pick the product line from the user's request (e.g. "app-bo"). Look up
`release.yaml.match_rules[<product>]` and run it via `searchJiraIssuesUsingJql`
(maxResults 1). Fetch its body with `getJiraIssue`
(`fields: ["summary","description","labels"]`, `responseContentFormat: "markdown"`).

## Carry forward vs bump
- CARRY: region sequence, dev/QA mentions, service list, pre-deployment structure.
- BUMP from user args: `{version}`, `{date}`, `{weekday}`, per-service `{tag}` values,
  and the release-report link (new release version page).

## Build the body
Fill `release-template.md` placeholders:
- `{region_sequence}` -> numbered lines from `release.yaml.region_sequence`
  ("1. Region1 - 10:00am").
- `{service_blocks}` -> for each service in args: name + `(tag: X)` + repo tag URL
  from `release.yaml.services[name] + "/-/tags/" + tag`.
- `{dev_mentions}` / `{qa_mentions}` -> from `release.yaml.mentions` (or carried).

## Labels
`release.yaml.labels` + the region labels for the regions being deployed.
