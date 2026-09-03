---
name: requirement-synthesis
description: Derive concrete resources, groups and configuration to implement — plus the cross-cutting constraints the inputs state — into per-group requirement chunks.
version: 1.0.0
---

# requirement-synthesis

Turn normalised inputs + repo analysis into concrete, verifiable requirements. Requirements come from the **inputs**
(solution model, issue ACs, instructions) — not from any pre-existing implementation.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read every chunk in `00-inputs/` and `10-analysis/`.
2. Enumerate the concrete resources/groups to implement and their configuration, sourced from the solution model and
   issue ACs.
3. Capture cross-cutting constraints stated by the inputs (e.g. configuration-driven / no hardcoding, separation of
   concerns between stacks, explicitly out-of-scope items).
4. Emit **one chunk per requirement group** so downstream design/codegen can parallelise.

## Output schema (per-group chunk `payload`)
```json
{
  "group": "<name>",
  "resources": [ { "type": "", "nameTemplate": "<as inputs specify>", "config": { }, "source": "<input ref>" } ],
  "constraints": [ "" ],
  "acceptance": [ "<testable statement>" ]
}
```

## Output location
`<root>/workflow_output/20-requirements/<group>.json` per group + `_manifest.json` (envelopes).

## Verification
Every requirement traces to an input chunk; naming is expressed as **templates from the inputs**, never hardcoded
literals; each group has at least one testable acceptance statement. Files parse.
