---
name: graphql-standards-review
description: Review the final GraphQL change diff against the STD-n standards checklist and the AC traceability matrix, and report violations and untested acceptance criteria with exact file and line references.
---

# GraphQL standards review

Used by: Standards Review Agent, after the deterministic Validation Run.

## Inputs
`60-validation/Validation-Result.json`, `00-inputs/standards-checklist.json`, `20-requirements/traceability.json`, `50-codegen/_codegen.json`, `10-analysis/reference-patterns.json` (conflicts), plus `git -C <root>/src diff HEAD` and every untracked file.

## Procedure
1. For each STD rule, decide whether it applies to the changed lines (SDL, resolver, Lambda, CDK, config or tests). For each applicable rule, check every changed hunk using `check_hint`.
2. Record each violation with the exact file and line (in the new file), the issue and a concrete fix.
3. Map every AC id to the tests whose names start with it (`file::test name`). An AC with no such test goes in `untestedAcs`.
4. List repo-convention conflicts where the new code followed the standard instead of the repo. These go in the PR description.
5. Do not fix anything. Report only.

## Output: `<root>/workflow_output/60-validation/standards-review.json`
`payload: {violations[{std_id, severity: must|should, file, line, issue, fix}], mustViolations, shouldViolations, untestedAcs[], acCoverage[{ac_id, tests[]}], conventionConflicts[], summary_markdown}`
