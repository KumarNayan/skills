---
name: gap-analysis
description: "Compute coverage gaps between requirements and design, gate on missing or mis-pathed config data files and on an unnamed target environment, and answer whether a human is needed before code generation."
version: 1.1.0
---

# gap-analysis

Decide if the run can proceed to codegen unattended.

The workflow's **run root** is `<root>`. Read with `read_file`, write with `write_file` as valid JSON.

## Procedure
1. Read requirement chunks and the design + design-validation chunks.
2. Compute the gap set: requirements without adequate design coverage, plus high-severity design findings.
3. **Config gates** — each is HIGH (increment `counts.high`, set `needsReview=true`):
    - (a) the solution doc names no target environment (`environmentsToGenerate` empty / `targetEnvironmentsStated`
      false) — config cannot be generated;
    - (b) any stack × environment (in `environmentsToGenerate`) × `env_type` missing a config file in the design/file
      map;
    - (c) any config path left as a placeholder or missing the repo's `src/` prefix — the path shape must come from
      `configConvention.pathTemplate`.
4. Set `needsReview` = true iff any HIGH gap remains.

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
`needsReview` is true iff `counts.high > 0`; the three config gates were evaluated; every gap references a real
requirement or config file. The file parses.