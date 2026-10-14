---
name: graphql-standards-digest
description: Turn the fetched GraphQL coding-standards bundle into a numbered, enforceable STD-n checklist that design, code, test and review steps apply and cite.
---

# GraphQL standards digest

Used by: Standards Digest Agent.

## Input
- `<root>/workflow_output/00-inputs/standards-fetch.json`: payload `{status, bundle_path, commit, ref, files[], lint_configs[]}`.
- If its status is not `OK`, write the output with status `BLOCKED` and the error, then stop.

## Procedure
1. Read the whole bundle at `bundle_path`. Every `## File: <path>` section is a source file.
2. Extract every enforceable statement: a rule, a convention, an example labelled "do"/"don't", or a lint rule enabled in a lint config. Skip narrative text and rationale.
3. Give each rule one category: `naming | nullability | pagination | error-model | deprecation | descriptions | versioning | auth | resolver | lambda | caching | logging | testing | performance | other`.
4. Severity: MUST, SHALL, NEVER, "required" or a lint rule at `error` is `must`. SHOULD, prefer, recommended or a lint `warn` is `should`.
5. Write `check_hint`: how to verify the rule mechanically. Use a regex over SDL or TS, a lint rule id, or a yes/no review question.
6. Ids run `STD-1`, `STD-2`, … in order of first appearance. Merge duplicates and keep every source.
7. Read the lint configs in `lint_configs[]` and record the enabled rules as STD rules with `source_file` = the config.

## Output: `<root>/workflow_output/00-inputs/standards-checklist.json`
```
payload: {
  standards_commit, standards_ref, source_files[], lint_configs[],
  rules[{id, category, rule, severity, source_file, source_heading, check_hint, example_good, example_bad}],
  counts{must, should}
}
```

## Never
- Add a rule that is not in the bundle, or "best practice" from memory.
- Drop a rule because it seems obvious.
- Rephrase a rule so its meaning changes.
