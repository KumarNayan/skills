---
name: standards-normalisation
description: Read the provided code-standards file and normalise it into a standards profile the code-generation and validation stages honour.
version: 1.0.0
---

# standards-normalisation

Convert a supplied coding-standards file into a structured profile.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. If no standards file is provided, write `OK` with an empty payload and stop.
2. Read the file (any text format). Extract concrete, enforceable rules — naming, structure, lint/formatting,
   testing, forbidden patterns — and normalise them.

## Output schema (`payload`)
```json
{
  "naming": [ "" ], "structure": [ "" ], "lintFormatting": [ "" ],
  "testing": [ "" ], "forbidden": [ "" ], "notes": [ "" ]
}
```

## Output location
`<root>/workflow_output/00-inputs/standards-profile.json` (envelope).

## Verification
Every rule is traceable to the source file; no invented conventions. The file parses.
