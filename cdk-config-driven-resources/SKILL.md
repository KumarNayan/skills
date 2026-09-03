---
name: cdk-config-driven-resources
description: Create resources by iterating configuration groups so adding a configured item needs no code change — nothing about the resources is hardcoded.
version: 1.0.0
---

# cdk-config-driven-resources

Every resource is produced from configuration, generically. Adding another configured entry must create its resource
through the same loop with no code edit.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read the resource config groups from the requirements/solution model.
2. Iterate the configured entries; for each flagged for provisioning, create the resource with its **configured**
   properties (names via the configured template, plus tunables like retention, timeouts, counts, policies).
3. Never write a hardcoded per-resource definition; never hardcode environment/account/region/names/ARNs.

## Output schema (design/codegen chunk `payload`)
```json
{
  "generator": "<construct/loop description>",
  "configSource": "<path in config the loop reads>",
  "properties": [ "<configured property honoured>" ],
  "files": [ { "path": "", "action": "create|modify" } ]
}
```

## Output location
Design: `30-design/`; code under `<root>/src` + chunk in `50-codegen/` (envelopes).

## Verification
A search of the generated code finds **no** hardcoded resource names/ARNs/env; every configured property in the
requirements is honoured by the loop; a hypothetical new config entry would be provisioned with no code change.
