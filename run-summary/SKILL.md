---
name: run-summary
description: "Assemble the final human-readable run summary from every stage folder: what was built, coverage, validation results, and blockers."
version: 1.0.0
---

# run-summary

Close the run with one readable summary sourced from the shared memory.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read the manifests and key chunks across `00-inputs` → `60-validation`, plus the validation logs.
2. Summarise: inputs ingested (and which were absent), requirements derived, files created/modified, tests authored,
   validation/synth results, and any unresolved gaps or blockers.
3. Do not re-run work or re-derive facts — report what the stage files say.

## Output schema (`payload`)
```json
{
  "inputs": { "present": [ "" ], "absent": [ "" ] },
  "requirements": { "count": 0, "groups": [ "" ] },
  "generated": { "files": [ "" ], "tests": [ "" ] },
  "validation": { "typecheck": "", "tests": "", "synth": "" },
  "blockers": [ "" ]
}
```

## Output location
`<root>/workflow_output/90-summary/summary.json` (envelope).

## Verification
Every figure traces to a stage file; absent inputs are reported as absent, not omitted; validation reflects the
`60-validation` logs. The file parses.
