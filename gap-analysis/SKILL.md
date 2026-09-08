---
name: gap-analysis
description: Compute coverage gaps between requirements and design, and answer whether a human is needed before code generation.
version: 1.0.0
---

# gap-analysis

Decide if the run can proceed to codegen unattended.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read requirement chunks and the design + design-validation chunks.
2. Compute the gap set: requirements without adequate design coverage, plus any high-severity design findings.
3. Decide `needsReview`: true if any material (high) gap remains, else false.
4. A config-driven stack lacking its per-environment config data file is a HIGH-severity gap: increment `counts.high` (which forces `needsReview=true`).

## Output schema (`payload`)
```json
{
  "gaps": [ { "id": "", "severity": "high|medium|low", "requirement": "", "description": "" } ],
  "needsReview": false,
  "counts": { "high": 0, "medium": 0, "low": 0 }
}
```

## Output location
`<root>/workflow_output/40-gaps/gaps.json` (envelope).

## Final answer
Your agent's final response text must be exactly `true` or `false` — the value of `needsReview` — so the branch node
can route on it.

## Verification
`needsReview` is true iff `counts.high > 0`; every gap references a real requirement. The file parses.
