---
name: iam-resource-policy-least-privilege
description: Design and implement least-privilege resource policies scoped to the exact principal and source resource — never public or broad grants.
version: 1.0.0
---

# iam-resource-policy-least-privilege

Grant exactly the access the requirements state, and no more.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. From the integration design, list each principal→resource action that must be permitted.
2. Write a resource policy per grant that: names the specific service principal; restricts by the specific source
   resource (e.g. a rule/queue ARN or account/source condition the inputs define); allows only the required actions.
3. Never emit a wildcard principal, a public grant, or actions beyond what is required.

## Output schema (design/codegen chunk `payload`)
```json
{
  "grants": [ { "principal": "", "action": [ "" ], "resource": "", "condition": { }, "source": "<requirement ref>" } ],
  "files": [ { "path": "", "action": "create|modify" } ]
}
```

## Output location
Design: `30-design/`; code under `<root>/src` + chunk in `50-codegen/` (envelopes).

## Verification
No grant uses `*` principal or public access; every grant is scoped to a specific source and traces to a requirement;
actions are the minimal set. The chunk parses.
