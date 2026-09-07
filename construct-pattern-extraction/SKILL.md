---
name: construct-pattern-extraction
description: Read the target repository and extract the construct, naming, configuration, import, IAM and testing patterns the generated code must imitate.
version: 1.0.0
---

# construct-pattern-extraction

Learn the repository's house style from its existing/reference implementations so new code matches it. Pre-existing
code for the same feature is **context, not requirements**.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared Memory*).
Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree lives at
`<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a sibling, never inside
`workflow_output`). Read inputs with `read_file`; write outputs with `write_file`. Never pass state through chat.

## Procedure
1. Read `<root>/workflow_output/00-inputs/solution-model.json`. If it has `references[]`, resolve each `location` under `<root>/src` and study those modules first; only if none are given, discover reference implementations across `<root>/src`.
2. Explore `<root>/src`: locate reference implementations, config loaders, type definitions, and existing tests.
3. Extract the reusable patterns: how constructs are created and organised; how names are templated; how
   configuration is read; how resources are imported across stacks; how IAM/resource policies are written; how tests
   are structured.
4. Record each pattern with a concrete example reference (file + brief snippet), not an abstraction only.

## Output schema (`payload`)
```json
{
  "structure": { "buildTool": "", "testTool": "", "layout": [ "" ] },
  "patterns": [
    { "aspect": "construct|naming|config|import|iam|testing", "rule": "", "example": { "file": "", "snippet": "" } }
  ]
}
```

## Output location
`<root>/workflow_output/10-analysis/reference-patterns.json` and/or `repo-conventions.json` (envelope), per your
task assignment.

## Verification
Every pattern cites a real file in `<root>/src`; the build/test tooling is correctly identified. The file parses.
- The reference implementation is a source of PATTERNS only. Never copy its files, business logic, or resource definitions into generated code — derive the pattern and re-apply it to the actual requirements. Every reported pattern cites a real file under `<root>/src`.
