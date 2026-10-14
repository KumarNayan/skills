---
name: graphql-data-access-and-caching
description: Design DynamoDB / S3 / Elasticsearch access, the Redis cache strategy (key, TTL, invalidation) and the least-privilege IAM it needs for a GraphQL operation, using the repo's shared clients.
---

# Data access and caching for GraphQL operations

Used by: Data Access & Cache Designer (design), Lambda Handler Agent (code).

## Data access, per store
- **DynamoDB**: name the table (as a config key), key or index, `Query`/`GetItem`/`BatchGetItem`/`TransactWrite`, projection and pagination (map `LastEvaluatedKey` to the GraphQL cursor if the SDL uses connections). `Scan` is not allowed unless the design requires it and the standards allow it.
- **S3**: bucket (config key), key prefix, operation and content type. Use presigned URLs only if the design asks.
- **Elasticsearch/OpenSearch**: index (config key), query DSL shape, size/from or search_after, and the fields returned.
- Every access lists the exact IAM actions and the resource expression, built from config keys.

## Redis cache (follow `reference-patterns.cache_pattern` first)
- **Key**: the repo's key format. If none exists, use `<service>:<domain>:<operation>:v<version>:<market>:<stable hash of normalised args>`. Include every argument and identity attribute that changes the result. Never include secrets.
- **TTL**: a config key per environment (and market if values differ). Never a literal.
- **Read-through**: get, then on a miss load and set with the TTL. Serialise the way the repo does.
- **Invalidation**: mutations that change cached data delete or version-bump every affected key pattern. List them. For an UPDATE that changes a cached response shape, bump the key version so old entries are never read with the new shape.
- **Redis down**: behave as the repo does (usually fail open: log and go to the source).
- **Stampede**: add a lock or jitter only if the repo or the design does.

## Output (design mode): `30-design/data-cache-design.json`
`{data_access[{operation, store, config_keys[], access, pagination, iam[{actions[], resource_expression}]}], cache{used, key_template, version, ttl_config_key, invalidation[{trigger, key_patterns[]}], down_behaviour}, config_entries[{key, purpose, per_environment, per_market, value_source}], file_plan[{path, action, purpose, ac_ids, std_ids}]}`
