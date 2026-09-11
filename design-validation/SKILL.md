---
name: design-validation
description: "Verify the implementation design covers every requirement, conforms to the reference patterns, and enumerates a config data file for every stack, environment and env_type at the configConvention path."
version: 1.1.0
---

# design-validation

Check the design before code is written.

The workflow's **run root** is `<root>`. Read with `read_file`, write with `write_file` as valid JSON.

## Procedure
1. Read all requirement chunks (`20-requirements/`) and all design chunks (`30-design/`).
2. For each requirement, confirm a design chunk covers it (resource, wiring, IAM, tests planned).
3. For each design chunk, confirm it conforms to the reference patterns and stated constraints.
4. **Config check (hard):** for each config-driven stack, verify the file map contains one config file for every
   environment in `environmentsToGenerate` and every `env_type` the stack requires — taken from
   `requiredConfigFiles` / `configConvention.env_types` (authoritatively, per stack), NOT from a sampled reference
   file — each at a concrete path expanded from `configConvention.pathTemplate` (repo `src/` prefix preserved, real
   environment and stack substituted; no placeholder left, no missing `src/`). A missing config file, a placeholder
   path, or a path that drops the repo's `src/` prefix is a HIGH-severity finding and sets `conformant=false`. If
   `environmentsToGenerate` is empty, that itself is HIGH (config cannot be generated).
5. **env_type completeness (hard):** every `*PersistentStack` MUST have BOTH a `live.yml` and a `dark.yml` entry in
   the file map (for each environment). A persistent stack whose file map carries only `live.yml` is a HIGH finding
   ("dark.yml missing for <stack>") and sets `conformant=false` — this is the exact regression where a sparse `dark`
   in the reference gets dropped. Cross-check the count against `requiredConfigFiles`: fileMap config entries must
   equal Σ stacks × required env_types × `environmentsToGenerate`.

## Output schema (`payload`)
```json
{
  "coverage": [ { "requirement": "", "covered": true, "designChunk": "" } ],
  "findings": [ { "severity": "high|medium|low", "requirement": "", "issue": "" } ],
  "conformant": true
}
```

## Output location
`<root>/workflow_output/30-design/design-validation.json` (envelope).

## Verification
Every requirement appears in `coverage`; the config check ran for every stack × `environmentsToGenerate` × `env_type`;
every `*PersistentStack` has both `live.yml` and `dark.yml` in the file map (else a HIGH finding);
`conformant` is false if any high finding exists. The file parses.