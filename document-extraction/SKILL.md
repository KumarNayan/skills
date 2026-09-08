---
name: document-extraction
description: "Fetch the solution design document from its source and normalise it into a structured solution model, including any reference-implementation pointers and the target environments the document names."
version: 1.1.0
---

# document-extraction

Turn a referenced solution design document into a normalised, structured model on disk. You **transcribe and
attribute**; you do not design.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*). Treat
it as `<root>` verbatim. Read inputs with `read_file`; write your output with `write_file` as valid JSON. Never pass
state through chat.

## Procedure
1. If the document reference is ABSENT, write `OK` with an empty payload and stop.
2. If PRESENT, **actually fetch it** via your bound document tool (never report success without fetching). A failed
   fetch is `PARTIAL` with the error verbatim.
3. Normalise the retrieved content into the solution model below — capture what the document states, never invent.
4. Capture any **reference-implementation** section into `references[]` verbatim (the in-repo path(s)/module(s) and
   what each teaches). Record the pointer only; do not read that code (the analysis stage does).
5. Capture the **target deployment environment(s)** the document names into `targetEnvironments` (verbatim, concrete
   names). If none are named, set `targetEnvironments: []` and `targetEnvironmentsStated: false` — never invent one.

## Output schema (`payload`)
```json
{
  "source": { "ref": "<page id/title/url>", "fetchedAt": "<iso>" },
  "architecture": "<narrative + component list>",
  "resources": [ { "type": "<as stated>", "name": "<as stated>", "config": {} } ],
  "namingConventions": [ "<pattern as stated>" ],
  "references": [ { "location": "<in-repo path/module>", "kind": "in-repo-path", "whatToLearn": "<...>" } ],
  "targetEnvironments": [ "<environment name exactly as stated>" ],
  "targetEnvironmentsStated": true,
  "constraints": [ "<config-driven / no-hardcode / scope as stated>" ],
  "openQuestions": []
}
```

## Output location
`<root>/workflow_output/00-inputs/solution-model.json` (as an envelope).

## Verification
The file parses; every `resources[]` and `references[]` entry traces to the document; `targetEnvironments` holds only
names the document states (or is `[]` with `targetEnvironmentsStated:false`); nothing is fabricated.