---
name: appsync-pattern-extraction
description: Extract the GraphQL, resolver, Lambda, cache, config, logging, error and test patterns new code must imitate from an AppSync/CDK repo, with file evidence and conflicts against the standards checklist.
---

# AppSync pattern extraction

Used by: GraphQL Pattern Extractor. These are patterns to imitate, never code to copy.

## Inputs
`10-analysis/Repo-Profile.json`, `00-inputs/standards-checklist.json`, `00-inputs/solution-model.json`. If the solution model lists `references[]`, study those modules first. Otherwise study `example_operations`.

## Extract, each with `{description, evidence[{file,line}], snippet (≤25 lines)}`
- **naming**: types, inputs, enums, payloads, operations, files, folders, construct ids, handler names.
- **sdl_style**: descriptions (`"""` vs `#`), nullability habits, pagination (connections or limit/offset), input objects for mutations, payload types, auth directives, `@deprecated` usage, `extend type` usage.
- **resolver_registration**: how a field is wired (data source reuse or one per Lambda; resolver id naming; pipeline functions).
- **lambda_construct**: construct class, runtime, bundling, env var population, X-Ray, Datadog layer/extension/tags, timeout, memory, log retention, IAM grant helpers.
- **handler_layering**: handler → service → repository split and AppSync event parsing.
- **input_validation**: library and where validation happens.
- **cache_pattern**: shared Redis helper, key format, TTL source (config key), read-through helper, invalidation on writes, behaviour when Redis is down.
- **data_access**: per store (DynamoDB, S3, Elasticsearch): client import, query style, pagination.
- **error_model**: error classes, `errorType` values, how errors surface to AppSync.
- **logging_fields**: logger, mandatory fields (requestId, fieldName, market, …).
- **config_pattern**: how a new key is added for every environment and market and how it reaches CDK and Lambdas.
- **test_patterns**: handler/service test structure, AWS SDK mocking, Redis mocking, CDK assertion test (Template.fromStack) and fixtures.

## Conflicts
Compare each pattern with the checklist and record
`conflicts_with_standards[{rule_id, observed_pattern, location, resolution:"standard wins for new code"}]`.

## Output: `<root>/workflow_output/10-analysis/reference-patterns.json`
`payload: {patterns{...}, conflicts_with_standards[]}`

Paths keep the `src/` prefix exactly as on disk. Never cite a pattern without a real file and line.
