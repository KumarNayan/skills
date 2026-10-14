---
name: graphql-validation-loop
description: One iteration of a bounded validate-and-fix loop for a GraphQL/AppSync/CDK change — run the repo's real checks, fix failures minimally, and no-op once green so the loop converges.
---

# GraphQL validation loop

Used by: GraphQL Validation & Fix Agent, inside a `repeat` node with a fixed budget.

## Protocol
1. Read `<root>/workflow_output/60-validation/iteration-latest.json`. If `payload.verdict` is `GREEN`, return it unchanged. This iteration is a no-op.
2. If `<root>/workflow_output/50-codegen/_codegen.json` has status `BLOCKED`, write a `BLOCKED` iteration and return. There is nothing to validate.
3. Source `<root>/workflow_output/node-env.sh` if it exists. In `<root>/src`, run the checks using the script names in `Repo-Profile.npm_scripts`. Write logs to `<root>/workflow_output/logs/60-validation/iter-<n>-<check>.log`.
   - build / typecheck;
   - tests with `--coverage --coverageReporters=json-summary`;
   - GraphQL schema validation (the repo's script, or `buildSchema` over all schema files plus the AppSync scalar/directive prelude);
   - GraphQL lint with the configs in `00-inputs/standards/lint/`, if a matching linter is installed;
   - `cdk synth --quiet -o <temp dir outside src>` per environment × market using `synth_context_keys`;
   - standards MUST rules over `git diff HEAD` plus untracked files;
   - every AC id appears in at least one test name.
4. Fix every failure with minimal edits to files this run created or modified. Order: compile errors first, then tests, then synth, then lint and standards.
5. Re-run only the failing checks to confirm, then write the iteration file.

## Never
Delete, skip or weaken tests; lower coverage thresholds; add `@ts-ignore`, `eslint-disable` or `any` to silence errors; edit files outside this run's change set; commit.

## Output
Let `n` = (number of `iteration-*.json` files, excluding `iteration-latest.json`) + 1. Write `iteration-<n>.json` and overwrite `iteration-latest.json`:
`payload: {iteration, verdict: GREEN|RED|BLOCKED, checks[{name, status, summary, log}], fixes[{file, change, reason}], remaining[]}`
