---
name: cdk-assertion-testing
description: Author CDK assertion tests that synthesise CloudFormation and assert the resources and properties the requirements specify — compilation alone is insufficient.
version: 1.0.0
---

# cdk-assertion-testing

Prove the generated infrastructure is correct by asserting the **synthesised CloudFormation**, not just that it compiles.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read the requirement chunks (for the acceptance statements) and the design/codegen chunks (for what was built).
2. Author tests under `<root>/src` using the repo's test framework and CDK assertions library.
3. For each requirement, assert the synthesised template: resource creation, resource name/properties, wiring,
   retry/DLQ, and IAM scoping — using the assertions API (template match / resource-count / has-resource-properties).
4. Follow the reference tests' structure and helpers.

## Output schema (codegen chunk `payload`)
```json
{
  "testFiles": [ { "path": "", "requirementsCovered": [ "" ] } ],
  "assertions": [ { "requirement": "", "resourceType": "", "asserts": "" } ]
}
```

## Output location
Tests under `<root>/src`; a chunk in `<root>/workflow_output/50-codegen/tests.json` (envelope).

## Verification
Every requirement acceptance statement maps to at least one assertion; tests assert synthesised resources/properties
(not mere compilation); tests follow the repo's testing patterns. The chunk parses.
