---
name: document-extraction
description: Fetch a solution design document from its source and normalise it into a structured solution model chunk (architecture, resource tables, diagrams, naming, configuration).
version: 1.0.0
---

# document-extraction

Turn a referenced solution design document into a normalised, structured model on disk. You **transcribe and
attribute**; you do not design.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. If the document reference is ABSENT, write `OK` with an empty payload and stop.
2. If PRESENT, **actually fetch it** via your bound document tool (e.g. `confluence_get_page` by page id, or title +
   space key). Reporting success without fetching is forbidden. A failed fetch is `PARTIAL` with the error verbatim.
3. Normalise the retrieved content into the solution model below — capture what the document states, do not invent.

## Output schema (`payload`)
```json
{
  "source": { "ref": "<page id/title/url>", "fetchedAt": "<iso>" },
  "architecture": "<narrative + component list>",
  "resources": [ { "type": "<as stated>", "name": "<as stated>", "config": { } } ],
  "namingConventions": [ "<pattern as stated>" ],
  "diagrams": [ { "kind": "mermaid|text", "content": "<verbatim>" } ],
  "constraints": [ "<config-driven / no-hardcode / scope as stated>" ],
  "openQuestions": [ ]
}
```

## Output location
`<root>/workflow_output/00-inputs/solution-model.json` (as an envelope).

## Verification
The file parses; every `resources[]` entry traces to the document; nothing is fabricated. If the fetch failed,
`status` is `PARTIAL` and the error text is present verbatim.
