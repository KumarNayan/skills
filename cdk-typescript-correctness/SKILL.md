---
name: cdk-typescript-correctness
description: Hard TypeScript/CDK compile-correctness rules every stack, config reader and test must obey — immutable config objects, valid CDK API and enums, guarded optionals, and declared types — so the generated app passes `tsc --noEmit` on the first alignment pass.
version: 2.0.0
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

5. **Declare types for every import (`TS7016`).** An import of a module with no bundled types (e.g.
   `js-yaml`) makes the file implicit-`any` and fails under strict mode. Fix it at the source:
   - add the matching `@types/...` package (e.g. `@types/js-yaml`) to `devDependencies` in the
     generated/updated `package.json` so Install Dependencies resolves it; **and/or**
   - prefer a typed alternative already present in the repo; **and/or**
   - if neither is possible, add a local `declare module '<name>';` ambient declaration.
   Never leave an import that resolves to implicit `any`.

6. **Keep interface ↔ reader ↔ tests in one shape.** The config interface, the config reader that
   builds it, and the test fixtures that stand in for it must use the **same field names and the same
   required/optional shape**. If you change the interface, update the reader and the tests in the
   **same pass** — drift here produces `TS2322`/`TS2741`/`TS2540` cascades.

## Self-check before emitting
Mentally run `tsc --noEmit` over every file you produced and confirm:
- every assignment targets a **mutable** field (no writes to `readonly`); no `delete`;
- every optional access is **guarded**;
- every CDK prop / enum member **exists in the installed types**;
- every `import` resolves a **type** (no implicit `any`);
- interface, reader and tests agree on one shape.

## Output
This skill changes **how** you write code; it does not add an output artifact. Emit your normal
authoring chunk/envelope for your stage. Files go under `<root>/src`.

## Verification
`tsc --noEmit` passes on the files you produced, with **none** of: `TS2540`, `TS2703`, `TS18048`,
`TS2532`, `TS2345` (`throttle`-on-`StageProps`), `TS2551` (bad `QueueEncryption` member), `TS7016`
(untyped import).
