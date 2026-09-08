---
name: cloudformation-synth-validation
description: Run install, type-check, the test suite and cdk synth; parse the results; and (for a fix agent) drive repairs until green or blocked.
version: 1.0.0
---

# cloudformation-synth-validation

Validate the generated code for real, and report actionable results.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. In `<root>/src`: install dependencies, run the type-checker, run the test suite, and run `cdk synth`.
2. Capture each step's pass/fail and the salient error lines. `cdk synth` failing is a hard failure.
3. If you are the **fix** agent: read the failures, repair type errors, failing assertions and synth errors in
   `<root>/src`, then re-validate — loop until green or `BLOCKED` (a failure you cannot resolve without new input).
4. Validation runs as a deterministic script with a hard `timeout` per step and non-interactive flags (`CI=true`, `npm ci --no-audit --no-fund`, `npm test -- --ci --watchAll=false`, `cdk synth --no-lookups`); it always exits and writes `60-validation/validation.json`. Agents READ that file — they do not run the toolchain themselves (a build must never be able to hang the run).

## Output schema (`payload`)
```json
{
  "steps": { "install": "pass|fail", "typecheck": "pass|fail", "tests": "pass|fail", "synth": "pass|fail" },
  "errors": [ { "step": "", "message": "" } ],
  "fixesApplied": [ "" ]
}
```

## Output location
`<root>/workflow_output/50-codegen/validation.json` (validation) or `fixes.json` (fix agent), as envelopes; logs to
`<root>/workflow_output/60-validation/`.

## Verification
The reported step statuses match the actual command exit codes; if `status` is `OK`, `synth` and `tests` are `pass`.
The file parses.
