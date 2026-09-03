---
name: gap-resolution
description: Apply the reviewer's answers to the open gaps exactly, and record what remains unresolved so code generation can proceed deterministically.
version: 1.0.0
---

# gap-resolution

Fold human decisions back into the run.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read `40-gaps/gaps.json` and the reviewer's answers handed to you (resolutions text; a
   `proceedWithoutResolving` flag).
2. For each gap, apply the reviewer's decision verbatim — do not reinterpret. Mark it resolved with the decision, or
   leave it open.
3. If gaps remain open and `proceedWithoutResolving` is false, set `proceedBlocked: true`.

## Output schema (`payload`)
```json
{
  "resolutions": [ { "gapId": "", "decision": "", "resolved": true } ],
  "unresolved": [ "<gapId>" ],
  "proceedBlocked": false
}
```

## Output location
`<root>/workflow_output/40-gaps/gap-resolutions.json` (envelope).

## Verification
Every gap from `gaps.json` is either resolved or listed in `unresolved`; `proceedBlocked` is true only when open gaps
remain and the reviewer did not authorise proceeding. The file parses.
