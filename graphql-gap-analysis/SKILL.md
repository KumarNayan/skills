---
name: graphql-gap-analysis
description: Compute the material gap set for a GraphQL change before code generation (including missing inputs) and decide, deterministically, whether a human is needed (needsReview).
---

# GraphQL gap analysis

Used by: Gap Analyzer. This is the only point where the workflow may stop for a human, so raise HIGH only for things that would make the generated code wrong or unsafe. It also covers the input checks (ticket, acceptance criteria, standards) that used to have their own gate.

## Inputs
`00-inputs/issue-acs.json`, `standards-fetch.json`, `standards-checklist.json`, `solution-model.json`; `20-requirements/*.json` (including `open-items.json`); `30-design/design.json`, `design-validation.json`, `classification.json`, `schema-design.json`.

## HIGH (each one sets needsReview = true)
Inputs:
- The Jira ticket could not be fetched (`issue-acs.json` status BLOCKED), or it has no acceptance criteria. Ask the reviewer to supply the criteria in the resolutions file.
- The standards could not be fetched (`standards-fetch.json` status not OK) or the checklist is empty. Ask whether to proceed on the repo's own conventions only.
- `solution-model.found` is false: no solution design, built from Jira only. Ask for a design reference, or confirmation to proceed from Jira only.

Contract and design:
- A blocking open item in `open-items.json` (contract conflict, missing data source, decisive open question).
- `requires_breaking_approval` is true: a breaking schema change with no deprecation path.
- `conflict_with_requested` is true.
- An AC with no planned artefact or no planned test.
- The environment list is empty.
- A HIGH finding in `design-validation.json`.

## MEDIUM / LOW (recorded and carried to the run summary, never a reason to stop)
Non-blocking open items with assumptions, SHOULD-rule failures, convention conflicts, dangerous (non-breaking) SDL changes.

## Output: `<root>/workflow_output/40-gaps/gaps.json`
```
payload: {needsReview, counts{high, medium, low},
          gaps[{id:"G-n", severity, category, description, sources[], question_for_reviewer, proposed_resolution}],
          summary_markdown}
```
- `needsReview` is true if and only if `counts.high > 0`. A script reads this field to choose the branch, so it must be exact.
- Every HIGH gap has a concrete `proposed_resolution`. If the reviewer leaves a gap out of the resolutions file, that proposal is applied and recorded as an assumption.
- `summary_markdown` is what the reviewer sees in the Gap Review form. List only the HIGH gaps, each as:
  `G-n — <description>` / `Question: …` / `Proposed resolution: …`.
  End with a ready-to-copy resolutions-file template: one line `G-n: ` per HIGH gap, plus `design_reference: ` if the design was missing and `acceptance_criteria: ` if the ticket had none.

## Resolutions file the reviewer uploads (read by the Gap Resolver)
Plain text, markdown, YAML or JSON with:
- `G-n: <answer>` for any gap the reviewer answers;
- optional `design_reference: <page id | URL | Title | SPACEKEY>`;
- optional `acceptance_criteria:` followed by `AC-1: …`, `AC-2: …` lines (Given/When/Then allowed).
