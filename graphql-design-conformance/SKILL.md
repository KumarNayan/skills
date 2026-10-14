---
name: graphql-design-conformance
description: Validate a GraphQL / AppSync design against the acceptance criteria, the standards checklist and the discovered repo patterns, and assemble the consolidated design.json and reviewer-ready design-review.md.
---

# GraphQL design conformance

Used by: Design Validation Agent (runs after the other design workers have written their chunks).

## Inputs
In `30-design/`: `classification.json`, `schema-design.json`, `schema-proposed.graphql`, `resolver-lambda-design.json`, `data-cache-design.json`. Also `20-requirements/*.json`, `00-inputs/standards-checklist.json`, `10-analysis/Repo-Profile.json` and `reference-patterns.json`.

## Checks (verify independently; never copy a self-check)
1. Every AC maps to at least one planned artefact and at least one planned test.
2. Every design element traces to an AC or a design source.
3. Every MUST rule is `pass` or `n.a.`, with evidence.
4. Every `file_plan` path either exists, or sits in a folder an example operation uses for the same kind of file. Nothing under `workflow_output`, no `src/src`.
5. Resolver style matches Repo-Profile, or the deviation is justified.
6. Every env var and IAM resource reads from a config key, and every config key appears in `config_entries`.
7. `config_entries` cover every environment × market in `requirement-spec.json`.
8. Breaking changes and `conflict_with_requested` are listed.
9. The proposed SDL builds together with the current schema (run the validation).

## Outputs
- `30-design/design.json`: the consolidated design the coders read:
  `{change_type, operations, sdl{target_files, delta_sdl}, resolvers, lambdas, data_access, cache, iam, config_entries, file_plan (merged, de-duplicated, one owner worker per file), test_plan[{ac_id, test_file, test_name:"AC-n: …", kind: handler|service|cdk-assertion, mocks}], breaking_changes, convention_conflicts}`
- `30-design/design-validation.json`: `{conformant, findings[{severity: high|medium|low, area, detail, ref}], ac_coverage[], standards_check[]}`
- `30-design/design-review.md`: a readable summary — operations, SDL delta, traceability table, standards table, conflicts, breaking changes. It is the record a reviewer reads; it is not a gate.
