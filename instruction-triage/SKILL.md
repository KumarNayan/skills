---
name: instruction-triage
description: Normalise free-text additional instructions into structured, actionable overrides and guidance for downstream stages.
version: 1.0.0
---

# instruction-triage

Turn free-text run instructions into structured guidance so later stages apply them deterministically.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. If the instructions are ABSENT (`NA`/empty), write `OK` with an empty payload and stop.
2. Classify each instruction as: an **override** (changes a default), a **constraint** (must/must-not),
   **scope** (in/out of scope), or **note**. Attach the stage it applies to when clear.

## Output schema (`payload`)
```json
{
  "overrides": [ { "stage": "", "instruction": "" } ],
  "constraints": [ "" ],
  "scope": { "inScope": [ "" ], "outOfScope": [ "" ] },
  "notes": [ "" ]
}
```

## Output location
`<root>/workflow_output/00-inputs/instructions.json` (envelope).

## Verification
Every instruction from the input appears in exactly one bucket; nothing is dropped or invented. The file parses.
