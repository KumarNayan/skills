---
name: workflow-run-contract
description: The shared run contract every agent follows: how to use the run root, the by-functionality folder tree, the chunked JSON envelope, and absent-input semantics.
version: 1.0.0
---

# workflow-run-contract

The rules every agent in this pipeline obeys so that a swarm of independent agents can coordinate purely through disk.

## Run root and shared memory
The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

The tree is organised **by functionality**, one folder per stage:

```
<root>/workflow_output/
  00-inputs/        10-analysis/     20-requirements/   30-design/
  40-gaps/          50-codegen/      60-validation/     90-summary/
  logs/<stage>/     README.md
<root>/src/         # the cloned repository checkout
```

You read only from **upstream** folders and write only into **your own** folder. Do not read another agent's chat
output — read its file.

## Output envelope
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


## Chunked outputs
Partition your output into **small JSON chunk files, one per unit of work** (one per input source, per requirement
group, per target file, per module) rather than one monolith. Write each chunk **as soon as it is ready** so
downstream agents can start early. Give each chunk a **stable, predictable name** inside your folder. Where you emit
more than one chunk, also write a `_manifest.json` in the folder: `{ "chunks": ["a.json", "b.json"] }`.

## Absent inputs
An input that is empty, `NA`, `null`, `undefined`, or still contains `{` or `}` is **ABSENT** — a recorded fact,
never a failure. Report `status: OK` with an empty payload.

## Verification before you finish
1. Your output file exists at the exact path you were given and is **valid JSON** (parse it back).
2. The envelope has all six keys and a valid `status`.
3. You wrote nothing outside your folder (code excepted, which goes under `<root>/src`).
