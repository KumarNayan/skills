---
name: graphql-change-classification
description: Decide from schema evidence whether a GraphQL change CREATEs new operations or UPDATEs existing ones, per operation and overall.
---

# GraphQL change classification

Used by: API Change Classifier (the first worker in the design stage).

## Inputs
The requested change type and target API name (from `00-inputs/run-inputs.json`), `20-requirements/requirement-spec.json`, and the schema files and resolver registrations in `10-analysis/Repo-Profile.json`.

## Procedure
1. Build an index of the current schema: every field of `Query`, `Mutation` and `Subscription`, including `extend type` blocks in every schema file, and every named type, input and enum. Record file and line. Prefer a small node script using the repo's own `graphql` package (`parse` + visit) over grep.
2. For each operation and type in the requirement spec, and for the target API name, record `exists` with file and line, plus the current signature when it exists.
3. Decide:
   - **UPDATE** when any targeted operation or type already exists. New operations added alongside are still handled as part of the update.
   - **CREATE** when none exist.
4. If the requested change type is `create` or `update` and the evidence disagrees, follow the evidence and set `conflict_with_requested=true`. The gap analysis raises this as a high gap.

## Output: `<root>/workflow_output/30-design/classification.json`
```
payload: {decision: CREATE|UPDATE, requested_change_type, conflict_with_requested,
          operations[{name, kind, exists, schema_file, line, current_signature, resolver_file}],
          existing_types[], new_types[], reasoning}
```
