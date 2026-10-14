---
name: graphql-gap-analysis
description: Compute the material gap set for a GraphQL change before code generation and decide, deterministically, whether a human is needed (needsReview).
---

# GraphQL gap analysis

Used by: Gap Analyzer. This is the only point where the workflow may stop for a human, so raise HIGH only for things that would make the generated code wrong or unsafe.

## Inputs
`00-inputs/solution-model.json`, `20-requirements/*.json` (including `open-items.json`), `30-design/design.json`, `design-validation.json`, `classification.json`, `schema-design.json`.

## HIGH (each one sets needsReview = true)
- `solution-model.found` is false: no solution design, built from Jira only.
- A blocking open item in `open-items.json` (contract conflict, missing data source, decisive open question).
- `requires_breaking_approval` is true: a breaking schema change with no deprecation path.
- `conflict_with_requested` is true.
- An AC with no planned artefact or no planned test.
- The environment list is empty.
- A HIGH finding in `design-validation.json`.

## MEDIUM / LOW (recorded and carried to the PR description, never a reason to stop)
Non-blocking open items with assumptions, SHOULD-rule failures, convention conflicts, dangerous (non-breaking) SDL changes.

## Output: `<root>/workflow_output/40-gaps/gaps.json`
```
payload: {needsReview, counts{high, medium, low},
          gaps[{id:"G-n", severity, category, description, sources[], question_for_reviewer, proposed_resolution}],
          summary_markdown}
```
- `needsReview` is true if and only if `counts.high > 0`. A script reads this field to choose the branch, so it must be exact.
- `summary_markdown` lists only the HIGH gaps, each with its question and proposed resolution. The reviewer sees it in the Gap Review form.
