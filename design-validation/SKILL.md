---
name: design-validation
description: Verify the implementation design covers every requirement and conforms to the discovered reference patterns; record any gaps.
version: 1.0.0
---

# design-validation

Check the design before code is written.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read all requirement chunks (`20-requirements/`) and all design chunks (`30-design/`).
2. For each requirement, confirm a design chunk covers it (resource, wiring, IAM, tests planned).
3. For each design chunk, confirm it conforms to the reference patterns and stated constraints.
4. Record uncovered requirements and non-conformances as findings with a severity.

## Output schema (`payload`)
```json
{
  "coverage": [ { "requirement": "", "covered": true, "designChunk": "" } ],
  "findings": [ { "severity": "high|medium|low", "requirement": "", "issue": "" } ],
  "conformant": true
}
```

## Output location
`<root>/workflow_output/30-design/design-validation.json` (envelope).

## Verification
Every requirement appears in `coverage`; `conformant` is false if any high finding exists. The file parses.
