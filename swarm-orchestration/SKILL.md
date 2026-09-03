---
name: swarm-orchestration
description: How an orchestrator fans out independent workers in parallel, assembles their file outputs, and handles partial/blocked results without doing the workers' work itself.
version: 1.0.0
---

# swarm-orchestration

You are an **orchestrator**. You delegate and assemble; you do **not** do the workers' work.

## Procedure
1. Read your upstream inputs (manifests + chunks) with `read_file`.
2. Identify the independent units of work. Hand each to exactly one sub-agent, giving it: its input value(s), its
   **full absolute output path**, and the output shape it must produce.
3. **Run independent workers in parallel.** Only serialise a worker that truly depends on another's output.
4. Wait for every worker, then **assemble** their file outputs into your stage envelope — read their chunks from
   disk, do not trust a worker that claims success without a written file.
5. Never silently substitute for a failed worker. Propagate its `PARTIAL`/`BLOCKED` into your envelope.

## Delegation contract you impose on each worker
- Do exactly one job; write exactly one (or a defined set of) JSON chunk(s) to the given path.
- Fetch when the input is present; an absent input is `OK` with no content.
- Return the envelope as raw JSON, never a prose summary.

## Output
Write your stage envelope (see workflow-run-contract) to your stage folder; `payload` lists the worker chunks and a
rolled-up status. If any worker is `BLOCKED` on something the run cannot continue without, set your status `BLOCKED`.

## Verification
Every worker you delegated to has a corresponding file on disk; your manifest lists them; your rolled-up status
reflects the worst worker status.
