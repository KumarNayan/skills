---
name: lambda-graphql-handler
description: Implement Node.js 20 TypeScript Lambda handlers and services behind AppSync resolvers — validation, typed responses, error model, structured logging and data access through the repo's shared clients.
---

# Lambda GraphQL handler

Used by: Resolver & Lambda Designer (design), Lambda Handler Agent (code).

## Shape (follow `reference-patterns.handler_layering`)
- **handler**: parse the AppSync event (`arguments`, `identity`, `source`, `info.fieldName`, `request.headers`), build the context (requestId, market, environment), call the service, and map the result or error.
- **service**: business rules from the ACs, cache read-through and orchestration.
- **repository**: data access only, through the shared clients in `Repo-Profile.shared_libs`. Never construct a new SDK or Redis client when a shared one exists.

## Required behaviour
1. **Input validation** with the repo's validation helper or library, before any I/O. Invalid input raises the standards' validation error type.
2. **Typed contracts**: TypeScript interfaces for arguments and responses that match the SDL exactly, including nullability. No `any` in new code unless the examples use it at that boundary.
3. **Errors**: the standards' error model. In Direct Lambda, throw the repo's error classes; AppSync maps them to `errorType`/`message`. Keep the error types the design lists. Never leak stack traces or internal ids.
4. **Logging**: the repo's logger with its mandatory fields (requestId, fieldName, market, latency) at the levels the examples use. Never log secrets, tokens or full PII payloads.
5. **Caching**: through the shared Redis helper, as the data-access-and-caching design specifies (read-through for queries, invalidation for mutations).
6. **Config**: read env vars populated by CDK from the YAML config. No literal names, endpoints or TTLs.
7. **Performance**: create clients outside the handler as the examples do; run independent fetches in parallel; respect timeouts from config.
8. **Docker Lambdas**: follow the repo's Dockerfile pattern (base image, build stage, CMD).

## Code-mode output: `<root>/workflow_output/50-codegen/handlers.json`
`payload: {files[{path, change: added|modified, purpose, ac_ids, std_ids}], exported_signatures[{file, export, signature}]}`
`exported_signatures` lets the Test Authoring Agent write tests without guessing.
