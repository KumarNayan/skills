---
name: requirement-synthesis
description: "Derive concrete resources, groups and configuration to implement from the ingested inputs, set the environments to generate config for from the solution doc, and emit per-group requirement chunks."
version: 1.1.0
---

# requirement-synthesis

Turn normalised inputs + repo analysis into concrete, verifiable requirements. Requirements come from the **inputs**
(solution model, issue ACs, instructions) — not from any pre-existing implementation.

The workflow's **run root** is `<root>`. Read with `read_file`, write with `write_file` as valid JSON.

## Procedure
1. Read every chunk in `00-inputs/` and `10-analysis/`.
2. Enumerate the concrete resources/groups to implement and their configuration, sourced from the solution model and
   issue ACs.
3. Set `environmentsToGenerate` from the solution model's `targetEnvironments` when present; otherwise, if the
   solution model's constraints enumerate the deployment environments, use that enumerated list verbatim (record its
   source, e.g. constraints[5]). Emit a **config-provisioning** requirement listing, for each provisioned stack, the
   config files that must exist — one per environment in `environmentsToGenerate`, per `env_type` in
   `configConvention`. Describe them via `configConvention` (path shape), never as a literal path.
4. Record a **blocking open item** only when NO environment list appears anywhere in the solution model (neither
   `targetEnvironments` nor an enumerated constraint list); never default to an invented environment. A stated
   constraint list is a valid source, not a blocker.
5. Emit one chunk per requirement group so downstream design/codegen can parallelise.

## Output schema (per-group chunk `payload`)
```json
{
  "group": "<name>",
  "resources": [ { "type": "", "nameTemplate": "<as inputs specify>", "config": {}, "source": "<input ref>" } ],
  "environmentsToGenerate": [ "<env from solution doc>" ],
  "constraints": [ "" ],
  "acceptance": [ "<testable statement>" ]
}
```

## Output location
`<root>/workflow_output/20-requirements/<group>.json` per group + `_manifest.json` (envelopes).

## Verification
Every requirement traces to an input chunk; `environmentsToGenerate` equals the solution model's `targetEnvironments`
(or is empty and flagged blocking); names are expressed as templates from the inputs, never hardcoded literals; each
group has a testable acceptance statement. Files parse.