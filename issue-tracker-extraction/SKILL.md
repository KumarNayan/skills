---
name: issue-tracker-extraction
description: Fetch referenced issues from the tracker and normalise each issue's summary, description and acceptance criteria into a chunk.
version: 1.0.0
---

# issue-tracker-extraction

Retrieve the issues named in the run and normalise their acceptance criteria for downstream requirement synthesis.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. If no issue keys are provided, write `OK` with an empty payload and stop.
2. For each key, **fetch the issue** via your bound tracker tool (e.g. `jira_get_issue`). Do not report success for
   an issue you did not fetch. A failed fetch for one key is recorded per-issue and does not abort the others.
3. Extract, per issue: summary, description, and acceptance criteria (as a list of testable statements).

## Output schema (`payload`)
```json
{
  "issues": [
    { "key": "<KEY>", "summary": "", "description": "", "acceptanceCriteria": [ "" ], "fetchError": null }
  ]
}
```

## Output location
`<root>/workflow_output/00-inputs/issue-acs.json` (envelope). `status` is `PARTIAL` if any issue failed to fetch.

## Verification
Each requested key appears exactly once; any `fetchError` carries the verbatim tracker error; acceptance criteria are
verbatim from the issue, not paraphrased into requirements (that is a later stage).
