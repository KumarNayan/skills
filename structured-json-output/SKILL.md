---
name: structured-json-output
description: Emit only valid, schema-conformant JSON chunk files — never prose — with stable names and a per-folder manifest.
version: 1.0.0
---

# structured-json-output

Your deliverable is **machine-readable JSON on disk**, not chat text.

## Rules
- Write with `write_file`. The file must be **strictly valid JSON** — no markdown fences, no comments, no trailing commas.
- Conform to the output schema your task/skill defines. Include every required key; use `null` or `[]` for empties.
- One chunk per unit of work; **stable, predictable filenames** so downstream agents find them deterministically.
- When you emit several chunks, also write `_manifest.json`: `{ "chunks": [ ... ], "count": <n> }`.
- Wrap the content in the standard envelope.

Every skill in this repo writes its result as a single **envelope** object (the `payload` differs per skill):

```json
{
  "schemaVersion": "1.0",
  "stage": "<stage-folder, e.g. 00-inputs>",
  "agent": "<your agent name>",
  "status": "OK | PARTIAL | BLOCKED",
  "outputPath": "<absolute path you wrote>",
  "payload": { }
}
```

- `OK` — you did the whole job (an absent optional input is still `OK` with an empty payload).
- `PARTIAL` — you did part of it; put the reason (and any upstream error, verbatim) in `payload.errors[]`.
- `BLOCKED` — you cannot proceed; name the blocker in `payload.blockers[]` and write `SKIP_DOWNSTREAM`.


## Verification
Re-read each file you wrote and `JSON.parse` it. If it does not parse, rewrite it before finishing. Confirm the
manifest lists exactly the chunk files present.
