---
name: graphql-solution-design-extraction
description: Resolve, fetch and normalise a Confluence solution design for a GraphQL / AppSync change into solution-model.json, with a section-level source on every extracted item.
---

# GraphQL solution design extraction

Used by: Solution Design Fetcher, Solution Capture Verify Agent, Gap Resolver (when a reviewer supplies a design reference).

## 1. Resolve the page (stop at the first rule that yields a page)
1. The design reference handed to you:
   - numeric page id;
   - URL containing `/pages/<id>/` or `pageId=<id>`: use the id;
   - URL `/display/<SPACE>/<Title+With+Plus>`: use `title` + `space_key`, decoding `+` and `%xx`;
   - text `Title | SPACEKEY`.
2. `payload.confluence_refs` in `<root>/workflow_output/00-inputs/issue-acs.json`, in order.
3. `confluence_search` with `text ~ "<TICKET-KEY>" AND type = page`.
4. `confluence_search` with the ticket summary as plain text.

Accept a search hit only if its body mentions the ticket key or its title clearly matches the ticket summary. Otherwise put every hit in `candidates_considered` and do not pick one.

## 2. Fetch
- `confluence_get_page(page_id, convert_to_markdown=true, include_metadata=true)`.
- `confluence_get_page_children(parent_id, include_content=true, convert_to_markdown=true, limit=50)`. Page with `start` while 50 come back. Keep children that carry design content (API, schema, data model, flows, NFRs, environments). Record skipped children by title.
- Write `<root>/workflow_output/00-inputs/solution-design.md`, one `## Page: <title> (<id>, v<version>)` header per page, content verbatim. Tables stay tables.

## 3. Extract to `<root>/workflow_output/00-inputs/solution-model.json`
Envelope `{schemaVersion, stage:"00-inputs", agent, status, payload}`. Payload:

```
found, page_ids[], page_versions{}, source_resolution: input|jira|search_key|search_summary|reviewer,
proposed_operations[{name, kind: query|mutation|subscription, arguments[{name,type,description}],
                     return_type, behaviour, source}],
sdl_snippets[{sdl, source}],               // SDL blocks copied verbatim
types_and_fields[{type, kind: type|input|enum|interface|union, fields[{name,type,description}], source}],
data_sources[{kind: dynamodb|s3|elasticsearch|redis|http|other, name, access_pattern, source}],
caching{strategy, key, ttl, invalidation, source},
auth{modes[], rules[], source},           // @aws_iam, cognito groups, API key, lambda auth
error_handling[{case, error_type, message, source}],
nfrs[{text, source}], sequence_or_flows[{text, source}],
target_environments[], target_environments_stated, target_environments_source,
target_markets[], target_markets_source,
references[{path_or_module, purpose, source}],  // recorded, never resolved
open_questions[{text, source}], candidates_considered[]
```

Rules:
- `source` is `"<page_id>#<section heading>"` on every item.
- Keep types exactly as written: `String!`, `[Price!]!`, connection types. Never "fix" nullability.
- Environments and markets may be listed anywhere (constraints, deployment, scope). Any explicit list counts. A bracketed `[a/b/c]` becomes `["a","b","c"]`. Placeholder text means "not stated".
- Never add an operation, field, rule or value that is not on the page.
- No page resolved: `found=false`, status `PARTIAL`. Never guess a design.

## 4. Verify before finishing
- Re-open `solution-design.md`: the table row counts match the page.
- Every operation and type has a `source`.
- Every `sdl_snippets[].sdl` is byte-identical to the page.
