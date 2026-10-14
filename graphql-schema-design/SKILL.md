---
name: graphql-schema-design
description: Design and implement AppSync GraphQL SDL changes that follow the standards checklist and the repo's SDL style, with every change classified as additive, dangerous, deprecating or breaking.
---

# GraphQL schema design

Used by: Schema Contract Designer (design mode), Schema & Resolver Agent (code mode), Gap Resolver.

## Design mode: produce `30-design/schema-design.json` and `30-design/schema-proposed.graphql`
1. Start from the requirement spec and the design's `sdl_snippets`. Where the design gives SDL, use it as written, then apply the checklist and record each adjustment with its STD id.
2. Apply the checklist first, then the repo's `sdl_style` where the checklist is silent. Typical rule areas:
   - naming of types, fields, enums, inputs and payloads;
   - nullability (non-null only where the design guarantees a value);
   - pagination (connection/cursor vs limit/offset);
   - mutations take a single input object and return a payload type, if the standards say so;
   - descriptions on every new type and field;
   - auth directives matching the design's auth modes.
3. Target file: the existing schema file of the same domain from Repo-Profile. Create a new file only when the repo keeps one file per domain and no file exists for this domain, and only if `schema_composition` will pick it up. Record how.
4. For an UPDATE, classify every change against the current schema:

   | Change | Class |
   |---|---|
   | new type, new field, new optional argument | additive |
   | new enum value, new union member, new interface implementation | dangerous (clients may break on exhaustive switches) |
   | field kept and marked `@deprecated(reason: "...")` with a replacement | deprecating |
   | remove or rename a field, type or argument; change a field type; output non-null → nullable; input nullable → non-null; new required argument; remove an enum value | breaking |

   Breaking changes are forbidden by default. Replace each one with the standards' deprecation path: add the new field or argument, keep the old one working and mark it `@deprecated` with a reason naming the replacement. Only if no deprecation path exists, keep the change and set `requires_breaking_approval=true`.
5. Validate the proposed SDL. Concatenate it with the current schema files and the AppSync scalar/directive prelude (AWSDateTime, AWSJSON, @aws_iam, @aws_cognito_user_pools, …), then `buildSchema` with the repo's `graphql` package.

`schema-design.json` payload:
`{target_files[], delta_sdl, changes[{element, before, after, class, std_ids, ac_ids}], breaking_changes[{element, why_unavoidable, mitigation}], requires_breaking_approval, standards_self_check[{rule_id, result: pass|fail|n.a., note}]}`

## Code mode: edit the schema files
- Apply `schema-proposed.graphql` to the target files in the repo's style: ordering, comment style, blank lines.
- Never delete or rename existing fields unless `gap-resolutions.json` records an approved breaking change.
- After editing, run the same `buildSchema` validation and fix until it passes.
