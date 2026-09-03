---
name: retry-and-failure
description: Retry transient failures with backoff, and when a step still fails, record a PARTIAL/BLOCKED result with the error verbatim — never fake success.
version: 1.0.0
---

# retry-and-failure

How every agent handles a step that can fail (a fetch, an install, a command).

## Procedure
1. Retry a **transient** failure (network, rate limit, timeout) a small bounded number of times with exponential
   backoff. Do not retry a deterministic failure (bad input, 404) — record it.
2. When a step ultimately fails, capture the **verbatim** error message.
3. Downgrade honestly: a job that partly succeeded is `PARTIAL`; a job that cannot proceed is `BLOCKED`. Never report
   `OK` for work you did not complete, and never fabricate content to cover a failed fetch.

## Output
Fold the outcome into your envelope: `status` reflects reality; `payload.errors[]` (or `payload.blockers[]`) carries
each verbatim error with the step name and how many retries were attempted.

## Verification
No `OK` envelope hides a failed step; every recorded error is the real message, not a paraphrase.
