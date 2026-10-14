---
name: appsync-repo-discovery
description: Discover the real layout of an AWS AppSync + CDK + Lambda (TypeScript) repository at run time and write Repo-Profile.json, the repo map every later step reads paths from.
---

# AppSync repository discovery

Used by: Repo Structure Discovery Agent. Nothing about the layout is known in advance. Never assume `lib/`, `src/`, `graphql/` or any other conventional folder.

## Inputs
- `<root>/src`: the checkout.
- `<root>/workflow_output/10-analysis/repo-census.json`: a deterministic inventory (schema files, yaml files, tests, regex matches with file and line). Start here, then confirm every finding by opening the files.

## Procedure
1. Read the root listing, `package.json` (scripts, workspaces, dependencies), `cdk.json` (`app`, `context`), every `tsconfig*.json` and the jest config (`roots`, `testMatch`, `collectCoverageFrom`, `coverageThreshold`).
2. **CDK entry.** Follow `cdk.json.app` to the app file and list every stack it instantiates (class name and file).
3. **API construct.** Find `new GraphqlApi` / `CfnGraphQLApi` and how the schema is attached (`SchemaFile.fromAsset`, `Definition.fromFile`, concatenation at synth time, inline). Record every schema file and how they are composed.
4. **Resolver style.** Inspect three or more existing resolvers:
   - `DIRECT_LAMBDA`: Lambda data source with no request/response mapping.
   - `VTL`: `MappingTemplate.*` or `.vtl` files.
   - `APPSYNC_JS`: `FunctionRuntime.JS_1_0_0` with `.js`/`.ts` resolver code.
   - `MIXED`: record which style each domain uses.
5. **Trace one real operation end to end:** schema field → resolver registration → data source → Lambda construct → handler → service → repository → shared clients → config loader → tests. Repeat for a second operation of the other kind (query/mutation).
6. **Shared libraries.** Find the Redis, DynamoDB, S3 and Elasticsearch/OpenSearch clients, the logger, error classes, the input-validation helper and the config loader. Record the import path used by handlers.
7. **Config.** Find the YAML root, the layout pattern (for example `<yaml_root>/<environment>/<market>.yml`), the loader file and how the CDK app and Lambdas read config. List the existing environment and market files.
8. **Synth selection.** Find how an environment and market are chosen at synth time (`tryGetContext('env')`, `process.env.STAGE`, npm scripts that pass `-c`). Record the context keys.
9. **Scripts.** Map `build`, `test`, `lint`, `schema_validate` and `synth` to the actual script names in `package.json`, or null.
10. **Tests.** Record where handler, service and CDK assertion tests live and which mocking libraries they use (aws-sdk-client-mock, jest.mock, ioredis-mock).

## Output: `<root>/workflow_output/10-analysis/Repo-Profile.json`
```
payload: {
  resolver_style, resolver_style_evidence[], resolver_style_by_domain{},
  schema_files[], schema_composition,
  cdk{app_entry, stacks[{name,file}], api_construct_file, resolver_registration_files[],
      lambda_construct_pattern{file, standard_vs_docker, bundling, env_var_source, tracing, datadog}},
  handlers_root, lambda_layout,
  example_operations[{name, kind, domain, schema_file, resolver_files[], cdk_registration_file,
                      data_source, handler_file, service_files[], test_files[]}],
  shared_libs{redis, dynamodb, s3, elasticsearch, logger, errors, config_loader, validation},
  config{yaml_root, layout_pattern, loader_file, env_files{}, market_files{}, key_style},
  tests{roots[], test_match, handler_test_examples[], cdk_assertion_examples[], mocking_libraries[]},
  npm_scripts{build, test, lint, schema_validate, synth},
  synth_context_keys{environment, market},
  package_manager, node_version, typescript_version,
  notes[], evidence[{claim, file, line}]
}
```
- Every path keeps the `src/` prefix exactly as on disk.
- Pick example operations that are most similar to the change in `00-inputs/solution-model.json`: same domain first, then the same resolver style and operation kind.
- If something cannot be found, set it to null and add a `notes[]` entry saying what you searched. Never invent a path.
