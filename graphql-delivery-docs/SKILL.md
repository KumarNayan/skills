---
name: graphql-delivery-docs
description: Produce the delivery documents for a GraphQL change (API doc, changelog entry, PR description, Jira comment) from the stage files, and post the Jira update; reports the true validation status.
---

# GraphQL delivery docs and Jira update

Used by: Docs & PR Writer, Jira Update Agent.

## Documents in `<root>/workflow_output/70-delivery/` (Docs & PR Writer)
- **api-doc.md**: for each new or changed operation: its signature (SDL), its arguments (type, nullability, description), a sample query and a sample response consistent with the tests, the errors (errorType and when), and caching behaviour.
- **changelog-entry.md**: in the repo's changelog style. If the repo has a CHANGELOG or API docs folder, also add the entry there with a minimal edit.
- **pr-description.md**:
  1. Summary: what changed and why (ticket key and summary)
  2. Change type: CREATE or UPDATE, and any breaking, deprecating or dangerous changes
  3. AC traceability table: AC | artefacts | tests
  4. Standards: ref and commit SHA, MUST/SHOULD violation counts, convention conflicts and how the standard won
  5. Validation, quoted exactly from `Validation-Result.json`: each check's status, coverage on changed files against the minimum, and the synth combinations. If anything is red, say RED plainly at the top.
  6. Gaps and assumptions: every item from `40-gaps/gaps.json` and `gap-resolutions.json`
  7. Config: keys added per environment and market, and anything unresolved
  8. Rollback: revert the commit; redeploy the previous CDK version; cache keys to invalidate or the key version to roll back; config keys to remove
- **jira-comment.md**: 5–10 lines: change type, operations, AC coverage summary, validation status. No internal paths, no secrets.

## Jira update (Jira Update Agent)
1. Read the branch and commit: `git -C <root>/src rev-parse --abbrev-ref HEAD` and `git -C <root>/src rev-parse HEAD`.
2. `jira_add_comment(issue_key, comment)` with jira-comment.md, the branch, the commit and the base branch for the pull request.
3. Transition only if `run-inputs.transitionJiraOnCompletion` is true and a transition name is given: call `jira_get_transitions`, match the name case-insensitively, then `jira_transition_issue(issue_key, transition_id)`. If nothing matches, do not transition; record the available names.
4. Write `70-delivery/jira-update.json`: `{comment_posted, comment_id, transitioned, transition_name, available_transitions[], branch, commit}`.
