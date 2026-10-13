---
name: cdk-typescript-correctness
description: Hard TypeScript/CDK compile-correctness rules every stack, config reader and test must obey — immutable config objects, valid CDK API and enums, guarded optionals, and declared types — so the generated app passes `tsc --noEmit` on the first alignment pass.
version: 2.2.0
---

# cdk-typescript-correctness

A compile-correctness checklist for generated AWS CDK (TypeScript). The generated code must pass
`tsc --noEmit` **before** it is useful; these rules exist because the generator repeatedly emits the
same handful of type errors. **On any conflict with a style/authoring skill, these rules win —
compiling code first.** Apply them in stack code, construct code, config readers, interfaces, and
tests alike.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared
Memory*). Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree
lives at `<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a
sibling, never inside `workflow_output`). Read inputs with `read_file`; write outputs with
`write_file`. Never pass state through chat.

## Rules (each prevents a specific error seen in production runs)

1. **Never assign to a `readonly` field (`TS2540`).** The config interfaces here declare fields
   `readonly` (e.g. `StackName`, `Environment`, `shouldProvision`, `busName`, `apiName`, the
   `DynamoDB`/`SQS`/`EventBridge`/`APIGateway`/`WAF` groups). Build each object **whole in a single
   object literal**; never set a property after construction — in readers **or tests**. To make a
   variant for a test case, **spread into a new object**, never mutate:
   - right: `const cfg = { ...base, SessionStore: { ...base.SessionStore, shouldProvision: false } };`
   - wrong: `cfg.SessionStore.shouldProvision = false;`

2. **Never `delete` an object property (`TS2703`).** `delete obj.x` fails on typed/readonly objects.
   Build a new object without the key instead:
   - right: `const { Environment, ...rest } = cfg;` (use `rest`)
   - wrong: `delete cfg.Environment;`

3. **Guard every optional before use (`TS18048` / `TS2532`).** Optional config fields (`DynamoDB?`,
   `SQS?`, `EventBridge?`, a looked-up `mainQueue`, etc.) may be `undefined`. Narrow first; only use
   `!` when a prior guard has proven presence:
   - right: `if (!infra.SQS?.Queues?.length) throw new Error('SQS config missing'); const q = infra.SQS.Queues[0];`
   - wrong: `const q = infra.SQS.Queues[0];`

4. **Use only real CDK API — validate against the installed `aws-cdk-lib` types, never memory.**
   - **API Gateway stage throttling is NOT a `throttle` property on `StageProps` (`TS2345`).** Do not
     pass `{ throttle: { rateLimit, burstLimit } }` to a `Stage`/`StageProps`. Set throttling via the
     stage/method options the installed version exposes — e.g. `StageOptions.throttlingRateLimit` and
     `throttlingBurstLimit`, or `MethodDeploymentOptions` under `deployOptions` /
     `addStage`/method settings. If the deployment uses `new apigateway.Stage(...)`, configure
     throttling through its supported `methodOptions`, not an invented `throttle` key.
   - **`QueueEncryption` members (`TS2551`):** use only members the installed `aws-cdk-lib` actually
     exports (`UNENCRYPTED`, `KMS_MANAGED`, `KMS`, and — only on newer versions — `SQS_MANAGED`). Do
     not assume `SQS_MANAGED` exists; if unsure, use `QueueEncryption.KMS_MANAGED`.
   - General: before using any enum member, static prop, or construct prop, confirm it exists in the
     installed types (check `node_modules/aws-cdk-lib`); never invent names.

5. **Imports of untyped modules (`TS7016`) — match the repo's module style, do NOT add a dependency.**
   An ES import of a module with no bundled types (e.g. `js-yaml`) makes the file implicit-`any` and
   fails strict mode. The repo's existing config readers already solve this with CommonJS `require`,
   which type-checks **without** any `@types/*` package. **Follow that pattern:**
   - **YAML specifically:** load it exactly as the repo's existing `Read*Config.ts` do —
     `const yaml = require('js-yaml');` then `yaml.load(...)`. **Never** `import * as yaml from
     'js-yaml'` / `import yaml from 'js-yaml'`: the repo ships no `@types/js-yaml`, it is not a declared
     dependency, and the `import` form is the only thing that fails `tsc` (the repo's own
     `require`-based readers compile cleanly).
   - **Any untyped module in general:** prefer the repo's `const x = require('<mod>');` convention over
     an ES import, since it needs no `@types/<mod>`. Do **not** "fix" `TS7016` by adding `@types/...`
     to `package.json` — Install Dependencies runs **before** the alignment loop, so a dep added
     mid-loop is never installed and `tsc` stays red. If `require` is genuinely unavailable, add a
     local `declare module '<name>';` ambient declaration instead.
     Never leave an import that resolves to implicit `any`, and never introduce an uninstallable dep.

6. **Keep interface ↔ reader ↔ tests in one shape.** The config interface, the config reader that
   builds it, and the test fixtures that stand in for it must use the **same field names and the same
   required/optional shape**. If you change the interface, update the reader and the tests in the
   **same pass** — drift here produces `TS2322`/`TS2741`/`TS2540` cascades.

## Alignment-loop IO contract (CodeFixAgent / alignment & validation agents)
The code-alignment loop's control files live under **`<root>/workflow_output/…`** — **never**
`<root>/<stage>/…`. When you read or write them, always include the `workflow_output/` segment and
use these **EXACT** paths (CDK folder numbers — `60-validation`, `50-codegen`):
- `<root>/workflow_output/60-validation/Validation-Result.json` — tsc/tests/synth/coverage exit codes
- `<root>/workflow_output/60-validation/Alignment-Result.json` — ALIGNED / NOT_ALIGNED verdict
- `<root>/workflow_output/50-codegen/Stage-Verdict.json` — the PASS/FAIL gate verdict (Resume signal)
- `<root>/workflow_output/60-validation/Fix-Report.json` — your own repair report

A path that omits `workflow_output/` (e.g. `<root>/60-validation/Validation-Result.json`) is **wrong**
and yields a false "file not found", which makes the repair step spuriously NO-OP and the loop never
converge. If a result file is genuinely absent at the correct path (e.g. the first pass, before any
validation has run), that is the only legitimate NO-OP — do not conclude "not found" from a path that
is missing the `workflow_output/` segment.

## Self-check before emitting
Mentally run `tsc --noEmit` over every file you produced and confirm:
- every assignment targets a **mutable** field (no writes to `readonly`); no `delete`;
- every optional access is **guarded**;
- every CDK prop / enum member **exists in the installed types**;
- every `import` resolves a **type** (no implicit `any`);
- interface, reader and tests agree on one shape.

## Output
This skill changes **how** you write code; it does not add an output artifact. Emit your normal
authoring chunk/envelope for your stage. Files go under `<root>/workflow_output/`.

## Verification
`tsc --noEmit` passes on the files you produced, with **none** of: `TS2540`, `TS2703`, `TS18048`,
`TS2532`, `TS2345` (`throttle`-on-`StageProps`), `TS2551` (bad `QueueEncryption` member), `TS7016`
(untyped import).
