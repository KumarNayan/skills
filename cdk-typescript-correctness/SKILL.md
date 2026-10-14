---
name: cdk-typescript-correctness
description: Hard TypeScript/CDK compile-correctness rules every stack, config reader and test must obey — immutable config objects, valid CDK API and enums, guarded optionals, and declared types — so the generated app passes `tsc --noEmit` on the first alignment pass.
version: 2.5.0
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
     exports (`UNENCRYPTED`, `KMS_MANAGED`, `KMS`, and — only on newer versions — `SQS_MANAGED`).
     Do not assume `SQS_MANAGED` exists. **But when a missing member is the one the config asks for,
     do NOT substitute a different member — honour the config via the escape hatch (next bullet).**
   - **Config-value ↔ CDK-member fidelity — realise what the config says, never a different value.**
     When config carries an encryption/mode/enum-like value, build the resource so the **synthesised
     CloudFormation matches that value** — read the value from config, never hardcode it, and map it to
     the **same-named** CDK member. If the installed `aws-cdk-lib` does **not** export that member
     (e.g. older libs lack `QueueEncryption.SQS_MANAGED`, `TS2551`), realise the intended CloudFormation
     property through the repo's **existing L1 escape-hatch convention**
     `(resource.node.defaultChild as <CfnType>).addPropertyOverride('<CfnProp>', <value>)` — exactly as
     the repo already does elsewhere (e.g. `stacks/ConfigServiceStack/ConfigServiceStack.ts` uses
     `CfnFunction.addPropertyOverride`). **Never swap in a different member just to compile** (e.g.
     `SQS_MANAGED`→`KMS_MANAGED`): it changes the resource's behaviour and breaks the config-faithful
     test. Worked example — config `encryption: 'SQS_MANAGED'` ⇒ CloudFormation
     `SqsManagedSseEnabled: true`:
     - right: `const q = new sqs.Queue(this, id, { /* …no encryption prop… */ }); (q.node.defaultChild as sqs.CfnQueue).addPropertyOverride('SqsManagedSseEnabled', true);`
     - wrong: `encryption: sqs.QueueEncryption.KMS_MANAGED`  // synthesises KmsMasterKeyId, not SqsManagedSseEnabled
   - **On ANY uncertainty, check the CLONED REPO — it is the source of truth, not memory and not the
     compiler's suggestion.** Before choosing an enum member / API / prop, resolve the two questions
     from the clone:
     1. **Which aws-cdk version?** Read `<cdir>/node_modules/aws-cdk-lib/package.json` → `version`
        (fallback `<cdir>/package.json` dependencies). Use **that** installed version's API — never
        assume a newer one and never add/upgrade a dep mid-loop (Install Dependencies already ran, so a
        bump won't install and `tsc` stays red).
     2. **Which members actually exist?** Grep the installed type definitions for the real exports, e.g.
        `grep -RnE 'SQS_MANAGED|KMS_MANAGED|UNENCRYPTED' node_modules/aws-cdk-lib/aws-sqs/lib/queue.d.ts`
        (or the relevant `*.d.ts`). Use only members that appear there.
     Then decide: if the member the **config** asks for exists in the installed version → use it; if it
     does **not** exist → realise the config's intent via the L1 escape hatch (above), **never** the
     compiler's "Did you mean 'X'?" suggestion (that proposes the nearest-spelled member, e.g.
     `KMS_MANAGED` for a missing `SQS_MANAGED`, which is a *different* resource behaviour). Never invent
     names, and never silently downgrade/substitute a config-requested value to satisfy the compiler.

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

7. **Entrypoint & wiring completeness — infer the stack set from the artifacts, never hardcode.**
   The authoritative list of stacks to generate and wire is **derived from the pipeline's own
   outputs**, not from the prompt or memory: read `requiredConfigFiles[].stack` in
   `<root>/workflow_output/20-requirements/config-provisioning.json` (and the FileMap in
   `<root>/workflow_output/30-design/`). For **every** stack in that list: (a) its loader must export
   the deploy function(s), **and** (b) `bin/Main.ts` must `import` that function and contain a
   `case "<stack name exactly as the artifact spells it>":` that calls it. A stack that appears in the
   requirements/design but has no `Main.ts` case (or no loader fn) is a defect — add it in the same
   pass. Never invent, drop, or guess a stack name; take it verbatim from the artifact.

8. **Canonical, convention-correct naming — inferred from requirements/design/analysis.** Do not
   re-case or re-style any name; derive every identifier and filename from the pipeline's artifacts
   and the reference repo:
   - **Identifiers** (stack/construct/class/interface/reader/test-type names): use the **exact
     spelling and casing** given in `20-requirements`/`30-design` (e.g. the `StackName`/FileMap
     entries). Never alter case (do not turn a given `Xyz` into `XYZ` or vice-versa) and keep **one**
     casing across the stack files, interface, reader, config paths, tests, loader and `Main.ts`.
   - **File/directory style:** follow the repo convention recorded in `<root>/workflow_output/
     10-analysis/` (conventions / reference-pattern output) and mirror the nearest existing sibling
     files (how the repo already names `Read*Config.ts`, `*StackLoader.ts`, test files, config dirs).
   - When unsure, copy the pattern from the reference file identified in `10-analysis`; never invent a
     new convention.

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
- interface, reader and tests agree on one shape;
- every stack in `requiredConfigFiles` has a loader fn AND a `Main.ts` import+case;
- every identifier/filename matches the artifact's exact casing and the repo's naming convention.

## Output
This skill changes **how** you write code; it does not add an output artifact. Emit your normal
authoring chunk/envelope for your stage. Files go under `<root>/src`.

## Verification
`tsc --noEmit` passes on the files you produced, with **none** of: `TS2540`, `TS2703`, `TS18048`,
`TS2532`, `TS2345` (`throttle`-on-`StageProps`), `TS2551` (bad `QueueEncryption` member), `TS7016`
(untyped import); every requirements/design stack is wired in `Main.ts`; and all names match the artifacts' casing + the repo convention (no re-cased or invented names).
