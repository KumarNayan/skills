---
name: gap-analysis
description: "Compute coverage gaps between requirements and design, gate on an unnamed target environment or a missing/mis-pathed config CONTENT template (never on config .yml files being absent from the design fileMap — that is the contract), and answer whether a human is needed before code generation."
version: 1.1.0
---

# gap-analysis

Decide if the run can proceed to codegen unattended.

The workflow's **run root** is `<root>`. Read with `read_file`, write with `write_file` as valid JSON.

## Procedure
1. Read requirement chunks and the design + design-validation chunks.
2. Compute the gap set: requirements without adequate design coverage, plus high-severity design findings.
3. **Config gates** — each is HIGH (increment `counts.high`, set `needsReview=true`). Config `.yml` files are
   generated later, at codegen, by the Config Authoring Agent, so they are **deliberately NOT present in the design
   fileMap** — do NOT gate on their absence there (that is the design contract, not a gap). Gate only on:
   - (a) the solution doc names no target environment (`environmentsToGenerate` empty / `targetEnvironmentsStated`
     false) — config cannot be generated. If `environmentsToGenerate` is populated, this gate does NOT fire.
   - (b) a config-driven stack for which the design provides **no config CONTENT template in any accepted form** — the
     Config Authoring Agent would have nothing to expand. A content template counts whether it is a standalone
     template object (StackName + Environment + Common + resource blocks), a `configStructure` in the
     config-provisioning requirement, OR config metadata embedded in the construct designs (e.g. queue/table/bus
     metadata with a `configConvention.schema` to shape it). Embedded metadata IS a valid content template — do NOT
     raise a gap merely because there is no *standalone YAML* file or template.
   - (c) a config CONTENT template whose `configConvention.pathTemplate` is left as a placeholder or drops the repo's
     `src/` prefix. (This checks the path *shape*, not the existence of generated files.)
4. Genuine per-stack env_type coverage (e.g. a `*PersistentStack` missing its `dark` requirement) is already asserted
   by **design-validation**; carry any HIGH finding it emitted into the gap set rather than re-deriving it from the
   fileMap here.
5. Set `needsReview` = true iff any HIGH gap remains.

## Output schema (`payload`)
```json
{
  "gaps": [ { "id": "", "severity": "high|medium|low", "requirement": "", "description": "" } ],
  "needsReview": false,
  "counts": { "high": 0, "medium": 0, "low": 0 }
}
```

## Output location
`<root>/workflow_output/40-gaps/gaps.json` (envelope). A downstream script reads `payload.needsReview` to route the
branch — your chat text is not the routing signal, so make the file correct.

## Verification
`needsReview` is true iff `counts.high > 0`; the config gates were evaluated as defined above; **no gap was raised
merely because config `.yml` files are absent from the design fileMap** (that is the contract, not a gap), and none
was raised because a content template is embedded-metadata rather than standalone YAML; when
`environmentsToGenerate` is populated, gate (a) did not fire. Every gap references a real requirement, a real
design-validation finding, or a real config-template defect. The file parses.