---
name: graphql-requirement-traceability
description: Merge Jira acceptance criteria and the Confluence solution design into one GraphQL requirement spec with an AC traceability matrix and classified open items.
---

# GraphQL requirement traceability

Used by: Requirement Synthesiser.

## Inputs
`00-inputs/issue-acs.json`, `solution-model.json`, `solution-design.md`, `standards-checklist.json`, `run-inputs.json`; `10-analysis/Repo-Profile.json`, `reference-patterns.json`.

## Procedure
1. One requirement per operation. Name, kind, arguments, return type and behaviour come from the design. Acceptance behaviour comes from the ACs. Every field carries `trace{ac_ids[], design_sources[]}`.
2. Environments: use `run-inputs.targetEnvironments` if present, otherwise `solution-model.target_environments`. Same rule for markets. Record the source.
3. Map every AC id exactly once in the matrix: design elements, planned artefacts (`sdl | resolver | lambda | cdk | config | test`), and status `covered | partial | uncovered`.
4. Open items: anything the inputs leave open, classified as `conflict | missing | open_question`.
   - **Blocking** when an AC cannot be implemented from the inputs; Jira and Confluence disagree on a contract detail (name, type, nullability, auth, behaviour, error semantics); an operation has no data source or access pattern; an open design question decides the contract; or the environment list is empty.
   - Everything else is non-blocking and gets a `proposed_assumption`.
5. If `solution-model.found` is false, build from the ACs only and add one blocking item: "no solution design; built from Jira acceptance criteria only".

## Outputs in `<root>/workflow_output/20-requirements/`
- `requirement-spec.json`: `{operations[{name, kind, arguments, return_type, behaviour, data_sources, caching, auth, errors, nfrs, trace}], types[], constraints[], out_of_scope[], environments, environments_source, markets, markets_source, additional_instructions}`
- `traceability.json`: `{matrix[{ac_id, ac_text, design_elements[], planned_artefacts[{kind, description}], status}]}`
- `open-items.json`: `{items[{id:"Q-n", type, description, sources[], blocking, proposed_assumption}], counts{blocking, non_blocking}}`

## Never
Take requirements from existing code, add scope, or resolve a conflict by picking a side.
