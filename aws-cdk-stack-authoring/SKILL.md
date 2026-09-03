---
name: aws-cdk-stack-authoring
description: Author AWS CDK (TypeScript) stack and construct code in the repository's established style, from a design file map.
version: 1.0.0
---

# aws-cdk-stack-authoring

Write CDK TypeScript that matches the repo's discovered conventions and implements the design exactly.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read the assigned design chunks in `30-design/` and the reference patterns in `10-analysis/`.
2. For each file-map entry you own, create/modify the file **under `<root>/src`** (never under `workflow_output`,
   never `src/src`), following the repo's class structure and construct organisation.
3. Keep everything **configuration-driven**: read names, ARNs, and tunables from config/inputs; hardcode nothing.
4. Compile-check what you can locally before finishing.

## Output schema (codegen chunk `payload`)
```json
{
  "files": [ { "path": "<repo-relative>", "action": "create|modify", "summary": "" } ],
  "notes": [ "" ]
}
```

## Output location
Code under `<root>/src`; a chunk in `<root>/workflow_output/50-codegen/` (envelope).

## Verification
Files exist at their file-map paths under `src`; no hardcoded environment/account/region/name/ARN; style matches the
reference patterns. The chunk parses.
