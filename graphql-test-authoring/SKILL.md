---
name: graphql-test-authoring
description: Write Jest (ts-jest) tests for a GraphQL change — one or more per acceptance criterion named with the AC id, AWS SDK and Redis mocked the repo's way, plus CDK assertion tests for new or changed resources.
---

# GraphQL test authoring

Used by: Test Authoring Agent (runs after the code workers).

## Inputs
`30-design/design.json` → `test_plan`; `50-codegen/handlers.json` → `exported_signatures`; `10-analysis/reference-patterns.json` → `test_patterns`; `10-analysis/Repo-Profile.json` → `tests{roots, test_match, mocking_libraries}`.

## Rules
1. **Per AC**: at least one test whose name starts with the AC id, for example `it('AC-2: returns the cached price when present', …)`. Cover the AC's Given/When/Then literally: the given state is in the mocks, the when is the handler call with an AppSync-shaped event, and the then is the asserted response or error.
2. **Mocks**: use the libraries the repo already uses. Typically `aws-sdk-client-mock` for v3 clients, and `jest.mock` of the shared Redis helper or `ioredis-mock`. Never call real AWS or Redis. Reset mocks between tests.
3. **Cache behaviour**: when the design caches, test hit, miss-then-set (with the TTL from config) and, for mutations, invalidation. When the repo fails open, also test Redis-down.
4. **Errors**: test every error case the design lists, asserting the error type the standards define.
5. **CDK assertions**: `Template.fromStack(...)` tests for each new or changed resource — Lambda (runtime nodejs20.x, memory, timeout, env var names, tracing Active), data source, resolver (type name, field name, data source, and templates or code when not Direct Lambda), and IAM statements (actions and resources). Use `Match.objectLike` and `Match.arrayWith`. Build the stack with the same test fixture or config the existing CDK tests use.
6. **Placement**: where the jest config discovers tests (`roots` + `testMatch`), next to existing tests of the same kind. Verify each path matches before finishing; a test jest cannot find does not count.
7. **Coverage**: aim for the minimum coverage on changed source files. Never weaken an assertion, use `.skip`/`.only`, or rely on snapshots alone.

## Output: `<root>/workflow_output/50-codegen/tests.json`
`payload: {tests[{file, name, ac_id, kind: handler|service|cdk-assertion}], ac_without_test[], paths_verified_against_jest_config: true|false}`
