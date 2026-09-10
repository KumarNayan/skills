---
name: alignment-verification
description: "Verify a produced artifact is faithfully aligned to its source of truth (a captured doc vs the live page, or generated code vs the solution + reference skeleton), emit an itemised ALIGNED/NOT_ALIGNED verdict, and no-op once ALIGNED so a bounded loop converges."
version: 1.0.0
---

# alignment-verification

Used inside a bounded **repeat loop** to drive a produced artifact toward its source of truth and then stop. The
caller's prompt names the two sides and the paths; this skill fixes the procedure, the verdict schema, and the
idempotency rule so every iteration behaves identically.

The workflow's **run root** is `<root>`. Read with `read_file`, write with `write_file` as valid JSON. The Confluence
page (when a capture is being verified) is reached with the `confluence_get_page` tool wired to you — never invent its
contents.

## Idempotency gate (ALWAYS run first)
1. Read the verdict file the caller names (e.g. `00-inputs/solution-capture-alignment.json` or
   `60-validation/Alignment-Result.json`).
2. If it exists and `verdict == "ALIGNED"`, **do nothing and return OK** — this is a no-op retry. Never re-verify,
   never re-fetch, never rewrite. The loop runs a fixed budget of attempts; once ALIGNED is written, every later
   attempt must be a no-op.

## Procedure (only when not already ALIGNED)
1. Re-read the **source of truth** the caller names:
   - *Document capture:* re-fetch the Confluence solution page with `confluence_get_page`.
   - *Code parity:* the solution contract (`Solution-In-Memory.json`) plus the reference skeleton, `configFiles` and
     `configSchema` in `Repo-Profile.json`.
2. Re-read the **produced artifact** the caller names (the captured `solution_doc.json`; or the generated code tree +
   config files on disk).
3. **Diff** the artifact against the source of truth and itemise every divergence. Flag, at minimum:
   - *Document capture:* a dropped or altered table, a paraphrase, invented content, a mis-parsed field, or a
     `targetEnvironments` list / `references[]` that does not match the page.
   - *Code parity:* a missing per-environment config file, a config file at the wrong level, config schema drift
     (renamed key, changed nesting, changed type), a spurious path prefix, a flattened concern, or a reference
     name/domain reused instead of a solution name. Verify layout/schema and solution coverage only — never require
     the artifact to copy the reference's content or names.
4. **Repair in place** every divergence you found (rewrite the captured doc, or correct the code/config), then set
   `verdict` from the post-repair state: `ALIGNED` only when zero divergences remain, otherwise `NOT_ALIGNED` with the
   residual items listed.

## Output schema (`payload`)
```json
{
  "verdict": "ALIGNED",
  "checked": "solution-capture | code-parity",
  "mismatches": [
    { "severity": "high|medium|low", "kind": "", "where": "", "detail": "", "repaired": true }
  ]
}
```
`mismatches` is `[]` when `verdict` is `ALIGNED`. Every item names what diverged, where, and whether this iteration
repaired it.

## Output location
Write the envelope to the exact verdict path the caller names — commonly:
- `<root>/workflow_output/00-inputs/solution-capture-alignment.json` (document capture), or
- `<root>/workflow_output/60-validation/Alignment-Result.json` (code parity).
When you repaired the artifact, also rewrite the artifact file in place (`solution_doc.json`, or the code/config files).

## Verification
The idempotency gate ran first and no-oped on a prior ALIGNED. The verdict file parses and carries a `verdict` and a
`mismatches` array consistent with it (empty iff ALIGNED). Every divergence you reported is itemised with a location;
nothing was weakened or deleted merely to reach ALIGNED.
