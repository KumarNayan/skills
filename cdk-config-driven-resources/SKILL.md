---
name: cdk-config-driven-resources
description: "Create resources by iterating configuration groups so adding a configured item needs no code change, and materialise the per-environment config data files themselves from the configConvention."
version: 1.1.0
---

# cdk-config-driven-resources

Every resource is produced from configuration, generically. Adding another configured entry must create its resource
through the same loop with no code edit — and the per-environment config **data files** the loop reads must
themselves be generated.

The workflow's **run root** is `<root>`. Code goes under `<root>/src`. Read with `read_file`, write with `write_file`.

## Procedure
1. Read the resource config groups from the requirements, and `configConvention` from `10-analysis`.
2. Iterate the configured entries; for each flagged for provisioning, create the resource with its **configured**
   properties (names via the configured template, plus tunables like retention, timeouts, counts, policies).
3. **Materialise every config data file** required by `requiredConfigFiles` in
   `20-requirements/config-provisioning.json` — one per `env_type` (from `configConvention.env_types`, per stack) for
   every environment in `environmentsToGenerate`. A `*PersistentStack` therefore gets BOTH `live.yml` and `dark.yml`
   for each environment; a compute stack gets `live.yml`. Do NOT emit only `live`. Populate each file with real values
   matching `configConvention.schema`, sourced from the requirements / solution model (StackName, Environment,
   Common.AWS_REGION/AWS_ACCOUNT, each resource group with `shouldProvision` + `metadata`). `dark.yml` is a distinct
   variant, not a copy of `live.yml` — carry its own `shouldProvision`/metadata differences from the reference dark
   schema. A config file is a **required deliverable — never a note**.
4. Generate the config loader to read via `configConvention.loaderReadPath` (do not prepend `src/` in the read
   string — the file LIVES at the `src/` path but is READ via the loader path).
5. Never hardcode environment/account/region/names/ARNs; never write a per-resource hardcoded definition.

## Output schema (design/codegen chunk `payload`)
```json
{
  "generator": "<construct/loop description>",
  "configSource": "<requirements path the loop reads>",
  "configFilesWritten": [ "<path from file map, under src/config/...>" ],
  "files": [ { "path": "", "action": "create|modify" } ]
}
```

## Output location
Code + config files under `<root>/src`; a chunk in `<root>/workflow_output/50-codegen/` (envelope).

## Verification
A search of generated code finds **no** hardcoded resource names/ARNs/env; every config data file in
`requiredConfigFiles` exists on disk at its `src/config/...` path with real values (no placeholder path, no missing
`src/`, none deferred to a note); every `*PersistentStack` has BOTH `live.yml` and `dark.yml` for every environment
(a persistent stack with only `live` is a defect); the loader reads via `configConvention.loaderReadPath` and branches
on `environment_type` for `dark`; a hypothetical new config entry would be provisioned with no code change.