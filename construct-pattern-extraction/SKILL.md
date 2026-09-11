---
name: construct-pattern-extraction
description: "Read the target repository and extract the construct, naming, config, import, IAM and testing patterns the generated code must imitate, and emit a configConvention describing how config files are laid out."
version: 1.1.0
---

# construct-pattern-extraction

Learn the repository's house style from its existing/reference implementations so new code matches it. Pre-existing
code for the same feature is **context, not requirements**.

The workflow's **run root** is `<root>` (the line printed by *Init Shared Memory*). Read with `read_file`, write with
`write_file` as valid JSON.

## Procedure
1. Read `<root>/workflow_output/00-inputs/solution-model.json`. If it lists `references[]`, resolve each in-repo path
   under `<root>/src` and study those modules first; otherwise discover reference implementations across `<root>/src`.
2. Extract the reusable patterns — construct creation/organisation, naming, config loading, cross-stack imports,
   IAM/resource policies, testing — each with a concrete `file:` citation.
3. Express **every** path repo-relative from the checkout root, preserving the `src/` prefix exactly as the file
   citations show. Never strip `src/` in the layout summary.
4. Emit a **`configConvention`** object describing HOW config files are laid out — the *form* only, never a concrete
   environment:
    - `pathTemplate`: the repo's physical config location, WITH `src/`, using `<environment>/<StackName>/<env_type>`
      placeholders as the repo shows (take the physical prefix from the actual repo — the reference document's diagram
      may omit `src/`; the repo is authoritative).
    - `envTypes`: the environment types the reference uses. **Enumerate the COMPLETE distinct set — never
      sample.** Read `<root>/workflow_output/10-analysis/config-envtype-census.json` (the deterministic census that
      globs every `config/stacks/*/*/*.yml`) and copy its `allEnvTypes`, `envTypesByStackType` and `envTypesByStack`.
      `dark.yml` is SPARSE in the reference (present only for some stacks/environments), so opening one example config
      file will miss it — a miss here silently drops every `dark.yml` downstream. Carry `envTypesByStack` through so
      per-stack differences (e.g. persistent stacks that have `dark`, compute stacks that do not) survive.
    - `loaderReadPath`: the loader's read string verbatim (CWD-relative, may omit `src/`). The file is CREATED at
      `pathTemplate` and READ via `loaderReadPath` — never conflate the two.
    - `schema`: required keys, common keys, and the resource-group shape the config files follow.

## Output schema (`payload`)
```json
{
  "structure": { "buildTool": "", "testTool": "", "layout": [ "src/..." ] },
  "patterns": [ { "aspect": "construct|naming|config|import|iam|testing", "rule": "", "example": { "file": "src/...", "snippet": "" } } ],
  "configConvention": {
    "pathTemplate": "src/config/stacks/<environment>/<StackName>/<env_type>.yml",
    "envTypes": [ "live", "dark" ],
    "envTypesByStackType": { "PersistentStack": [ "live", "dark" ], "ComputeStack": [ "live" ] },
    "envTypesByStack": { "<StackName>": [ "live", "dark" ] },
    "loaderReadPath": "./config/stacks/<environment>/<StackName>/<env_type>.yml",
    "schema": { "required": [ "StackName", "Environment", "Common" ], "common": [ "AWS_REGION", "AWS_ACCOUNT" ], "resourceBlocks": "groups[]{name,shouldProvision,metadata}" }
  }
}
```

## Output location
`<root>/workflow_output/10-analysis/reference-patterns.json` (envelope).

## Verification
Every pattern cites a real file under `<root>/src`; all reported paths keep the repo's `src/` prefix;
`configConvention.pathTemplate` is physical (with `src/`) and distinct from `loaderReadPath`; `configConvention`
contains no concrete environment (only placeholders). `envTypes` equals the census `allEnvTypes` and INCLUDES `dark`
whenever the census does; `envTypesByStack` is present. The file parses.