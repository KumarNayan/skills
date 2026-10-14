---
name: cdk-test-design
description: How to author CDK tests so they compile and stay in lockstep with the stack — infer the config from the repo's real reader against a real environment (never a fabricated fixture), bind calls/imports to the stack's actual constructor signature and exported names, and derive expected values from the config, not magic literals.
version: 1.2.0
---

# cdk-test-design

Authoring rules for the **fixtures and expectations** inside CDK assertion tests, so the test file
passes `tsc --noEmit` and never drifts from the stack it exercises. This complements
`cdk-assertion-testing` (which says to assert the *synthesised* CloudFormation, not mere
compilation) and defers to `cdk-ts-correctness` on any compile-rule conflict. It exists
because the generator repeatedly builds fixtures by **mutating `readonly` config** and asserts
**stale expected values**, so the test files fail to compile (`TS2540`) or fail at
`Template.fromStack()` — and the tests are never actually verified.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared
Memory*). Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree
lives at `<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a
sibling, never inside `workflow_output`). Read inputs with `read_file`; write outputs with
`write_file`. Never pass state through chat.

## Rules (each prevents a specific test-file defect seen in production runs)

1. **INFER the test config from the repo's real reader — do NOT hand-author a fixture literal.** The
   single biggest source of broken tests is a fabricated `BuildConfig` literal full of guessed values
   (`Environment: 'test-env'`, made-up `AWS_ACCOUNT`/`AWS_REGION`, invented key names) that drifts from
   the real interface (`TS2345`/`TS2741`/`TS2540`) and asserts values nothing produces. Instead, obtain
   the config the way the proven working repo does — **call the repo's actual config reader**:
   - Discover the reader's **exported name and parameter order** from the clone
     (`src/utils/**/Read*Config.ts`) — do not assume it; it varies (e.g.
     `readPaceMessageConfig(environment, stack_name, filename)` or
     `getPaceMessagePersistentConfig(stack_name, environment, envType)`).
   - Call it against a **REAL environment** discovered from the clone
     (`src/config/stacks/<env>/<Stack>/`) and/or `<root>/workflow_output/20-requirements/
     config-provisioning.json` — **never invent an environment** like `'test-env'`.
   - right: `const config = getPaceMessagePersistentConfig('PaceMessagePersistentStack', 'apn1nisintegration', envType);`
   - wrong: `const config: BuildConfig = { Environment: 'test-env', Common: { AWS_REGION: 'us-east-1', AWS_ACCOUNT: '123456789012' }, … };`
   This guarantees the fixture matches the interface exactly and every value is inferred from the clone's
   config files, not guessed. **Only** if the unit genuinely takes a plain props object and no reader/
   config exists, fall back to a hand-built literal — and then build it as a WHOLE typed object literal
   (never assign after construction → `TS2540`; vary via spread, see rule 2).

2. **To make a variant, SPREAD into a new object — never assign into the base.** Vary one field by
   spreading the base (and any nested group) into a fresh object; never write back to a readonly
   field or `delete` a key:
   - right: `const dark = { ...base, SessionStore: { ...base.SessionStore, shouldProvision: false } };`
   - wrong: `base.SessionStore.shouldProvision = false;`  // TS2540
   - wrong: `delete base.Environment;`                    // TS2703

3. **Include every REQUIRED config field so the stack constructs.** A fixture missing a required
   field fails `tsc` (`TS2741`/`TS2322`) or throws inside the constructor at `Template.fromStack()`.
   Take the field set and required/optional shape from the config **interface** and the config
   **reader** — the fixture must satisfy the same shape (see `cdk-ts-correctness` rule 6:
   interface ↔ reader ↔ tests are one shape). When a required nested group (DynamoDB/SQS/EventBridge/
   APIGateway/WAF) is present in the interface, provide it fully in the fixture.

4. **Import ONLY the stack's public surface.** Import the stack/construct and the config interface
   type from their public entrypoints — never deep-import private files, internal helpers, or test
   utilities that aren't exported. An import that doesn't resolve a type makes the file implicit-`any`
   or fails to compile.

5. **Derive expected values from reusable helpers, never magic literals.** Compute the expected
   resource names/properties the same way the stack does (from the fixture's own values or a shared
   helper), so a config change updates both sides at once. Hard-coded expected strings drift from the
   stack and break on the next legitimate change.

6. **Config is the authority — both stack and test must honour the config value (fix the DEVIATING
   side, in one pass).** The most common `hasResourceProperties` mismatch is the stack and test
   disagreeing on an encryption/mode/enum value. The tie-breaker is the **config** (`live.yml`/`dark.yml`
   under `src/config/stacks/<env>/<Stack>/`), not whichever side is easier to edit. Read the config
   value and assert the CloudFormation property it implies; if the stack synthesises something else,
   **fix the stack** (see `cdk-ts-correctness` rule 4 — realise the config value, via the L1
   escape hatch if the enum member is missing), do **not** rewrite the test to the stack's wrong output:
   - config `encryption: 'SQS_MANAGED'` ⇒ assert `SqsManagedSseEnabled: true` **and** ensure the stack
     emits it (`CfnQueue.addPropertyOverride('SqsManagedSseEnabled', true)`), **never** flip the test to
     `KmsMasterKeyId` because the stack wrongly used `QueueEncryption.KMS_MANAGED`.
   - config `encryption: 'KMS_MANAGED'` ⇒ assert `KmsMasterKeyId` and the stack uses
     `QueueEncryption.KMS_MANAGED`. Match the assertion to the **config's** intent, consistently on both
     sides — never leave stack and test asserting different encryptions (that cannot converge).

7. **The test file must `tsc --noEmit`-compile before it is useful.** Jest cannot run a suite that
   fails to compile (`Test suite failed to run`), so an uncompilable test is an **unverified** test.
   Mentally run `tsc` over the fixture and expectations before emitting (see
   `cdk-ts-correctness` self-check): no writes to `readonly`, no `delete`, optionals guarded,
   every imported name resolves a type, fixtures satisfy the config interface.

8. **Bind to the ACTUAL public surface — read it from the generated source, never assume.** Before any
   constructor call or import, open the generated **stack** file and read its real
   `constructor(scope, id, …)` parameter list (count, order, types), and open the **loader/index** and
   read its real `export` names. Construct and import **exactly** those:
   - If the constructor is `(scope, id, buildConfig, environment_type: string, props?: cdk.StackProps)`,
     call `new Stack(app, id, cfg, 'live', { env });` — if it is `(scope, id, buildConfig, props?)`,
     call `new Stack(app, id, cfg, { env });`. Pass the right number/types of args in the right order.
   - Import the exact exported identifiers (e.g. `deployPaceMessageStack`), **never** a guessed name
     (`loadPaceMessageStacks` → `TS2724`).
   Never assume the standard CDK `(scope, id, props)` shape and never invent an export name — a mismatch
   is `TS2345`/`TS2554`/`TS2724` and stops the whole suite compiling (→ tests unverified). If the stack's
   signature or exports look wrong versus the solution, record that as a **STACK** defect for CodeFixAgent
   — but the test must still compile against the **actual current** signature.

## Output
This skill changes **how** you write test fixtures and expectations; it does not add its own output
artifact. Emit your normal test-authoring chunk/envelope (see `cdk-assertion-testing`). Test files go
under `<root>/src`.

## Verification
`tsc --noEmit` passes on the test files (no `TS2540`/`TS2703`/`TS2741`/`TS2322`); `jest` **runs** every
suite (no "Test suite failed to run"); each fixture is a whole typed object literal (no post-construction
assignment to config); variants are made by spread; and every expected assertion matches the shape the
installed `aws-cdk-lib` actually synthesises for the current stack code.
