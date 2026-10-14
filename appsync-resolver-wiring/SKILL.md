---
name: appsync-resolver-wiring
description: Design and implement AppSync resolver wiring and the CDK Lambda/data-source/IAM/observability constructs for a GraphQL field, in whichever resolver style the repo uses.
---

# AppSync resolver and CDK wiring

Used by: Resolver & Lambda Designer (design), Schema & Resolver Agent and CDK Wiring Agent (code).

## Resolver style: follow `Repo-Profile.resolver_style` (per domain when MIXED)
- **DIRECT_LAMBDA**: Lambda data source plus a resolver with no mapping templates. Reuse the domain's existing Lambda data source when the repo routes several fields through one Lambda (dispatch on `info.fieldName`). Otherwise create one data source per Lambda, as the examples do.
- **VTL**: request/response templates stored where the examples keep theirs, named like them. Use `$util.toJson`, `$util.error` and `$ctx.result` exactly as the examples do.
- **APPSYNC_JS**: `request`/`response` functions in a `.js`/`.ts` resolver file, `runtime: FunctionRuntime.JS_1_0_0`, bundled the way the examples are. Pipeline resolvers reuse existing `AppsyncFunction`s when the design calls for the same step.

Deviate only when the design requires something the style cannot do (for example batch invoke). Justify it in the design.

## CDK wiring (code goes only in files Repo-Profile names, or new files matching example locations)
- **Lambda construct**: the same construct class as the examples (`NodejsFunction` / `Function` / `DockerImageFunction`), `runtime: NODEJS_20_X`, the same bundling options, memory and timeout from the design (taken from config if the repo does so).
- **Docker vs standard**: Docker only when the design needs native dependencies, a package over the zip limit, or the domain already ships Docker Lambdas. Justify the choice.
- **Env vars**: populated from the YAML config loader. Never literals for names, ARNs, endpoints, hosts or TTLs.
- **Observability**: `tracing: Tracing.ACTIVE` (X-Ray) and the Datadog layer, extension and tags exactly as the examples apply them; log retention as the examples set it.
- **IAM**: least privilege. Grant only the actions the data-access design lists, on resources resolved from config (table ARN plus `/index/*` only if an index is queried; bucket prefix ARNs; ES domain ARN with `es:ESHttpGet`/`ESHttpPost` as needed). Use the repo's grant helpers (`table.grantReadData`, …) when examples do.
- **Networking**: VPC, subnets and security groups for Redis/ES exactly as existing Lambdas that reach the same cluster.
- **Construct ids and names**: follow the example naming; ids stay stable across environments.

## Design-mode output: `30-design/resolver-lambda-design.json`
`{resolvers[{field, type_name, style, data_source, files[], justification}], lambdas[{name, kind, justification, construct_file, handler_file, service_files[], memory, timeout, env_vars[{name, config_key}], tracing, datadog, vpc}], cdk_changes[{file, change, ac_ids, std_ids}], file_plan[{path, action, purpose, ac_ids, std_ids}]}`

## Never
Hardcode an environment, account, region, ARN or name; recreate a resource that already exists; or touch unrelated stacks.
