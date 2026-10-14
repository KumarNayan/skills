---
name: env-market-config-authoring
description: Add per-environment and per-market YAML config entries for a GraphQL change, deterministically, in the repo's exact layout and key style, with no placeholder values.
---

# Environment and market config authoring

Used by: Config Authoring Agent. Your only job is YAML config. Do not write `.ts` code.

## Inputs
`30-design/design.json` → `config_entries`; `20-requirements/requirement-spec.json` → `environments`, `markets`; `10-analysis/Repo-Profile.json` → `config{yaml_root, layout_pattern, key_style, env_files, market_files, loader_file}`.

## Procedure
1. Write a short script (Python or Node) and run it. Do not hand-edit many files.
2. Expand `layout_pattern` for every environment × market (or every environment if config is not per market) into concrete file paths. They must already exist, or match the pattern exactly.
3. For each file, add every required key at the nesting the loader expects, in the repo's key style. Preserve comments, ordering and indentation: use a round-trip YAML library (ruamel.yaml or `yaml` in Node with `keepSourceTokens`) or a careful text insert.
4. Values:
   - Environment-specific values (table names, ARNs, endpoints, account ids) come only from values already present in the same environment's or market's files. For example, reuse the existing table-name prefix, or the account from a sibling resource ARN.
   - Shared tunables (TTL, timeout, memory) come from the design.
   - If a value cannot be derived, leave the key out of that file and record it as unresolved. Never write a placeholder, TODO or invented value.
5. Verify: re-parse every touched file; count files and keys written against environments × markets × config_entries; confirm the loader reads the new keys at the path you wrote them (read the loader code).

## Output: `<root>/workflow_output/50-codegen/config.json`
`payload: {files[{path, keys_added[]}], expected_count, written_count, unresolved[{key, environment, market, reason}]}` — status `PARTIAL` if anything is unresolved.
