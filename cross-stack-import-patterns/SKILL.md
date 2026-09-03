---
name: cross-stack-import-patterns
description: Design and implement how resources owned by one stack are imported by another (by ARN/exported reference) instead of being recreated.
version: 1.0.0
---

# cross-stack-import-patterns

Preserve stack separation: where a resource is owned by another stack, **import** it; never recreate it.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. From requirements + reference patterns, identify resources this stack must reference but does not own.
2. Use the repo's established import mechanism (import-by-ARN, exported value, or SSM reference — whichever the
   reference stacks use). Match that mechanism exactly.
3. When designing: record each import as a design chunk. When implementing: write the import code under `<root>/src`.

## Output schema (design `payload`)
```json
{
  "imports": [ { "logicalName": "", "resourceType": "", "identifierSource": "arn|export|ssm|config", "mechanism": "" } ],
  "targetFiles": [ "<repo-relative .ts>" ]
}
```

## Output location
Design: `<root>/workflow_output/30-design/`. Implementation: files under `<root>/src` + a codegen chunk.

## Verification
No imported resource is also created; the import mechanism matches the reference stacks; identifiers come from
configuration/inputs, not hardcoded literals.
