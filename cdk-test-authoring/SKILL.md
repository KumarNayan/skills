---
name: cdk-test-authoring
description: How to build CDK test fixtures so the test file compiles and stays in lockstep with the stack — whole immutable config objects (vary by spread, never assign readonly), all required fields present, public surface only, and expected values that follow whatever the stack actually emits.
version: 1.1.0
---

# cdk-test-authoring

Authoring rules for the **fixtures and expectations** inside CDK assertion tests, so the test file
passes `tsc --noEmit` and never drifts from the stack it exercises. This complements
`cdk-assertion-testing` (which says to assert the *synthesised* CloudFormation, not mere
compilation) and defers to `cdk-typescript-correctness` on any compile-rule conflict. It exists
because the generator repeatedly builds fixtures by **mutating `readonly` config** and asserts
**stale expected values**, so the test files fail to compile (`TS2540`) or fail at
`Template.fromStack()` — and the tests are never actually verified.

The workflow's **run root** is handed to you in your task text (the one line printed by *Init Shared
Memory*). Treat it as `<root>` verbatim — never search for or re-derive it. The shared-memory tree
lives at `<root>/workflow_output/<stage-folder>/`; the repository checkout lives at `<root>/src` (a
sibling, never inside `workflow_output`). Read inputs with `read_file`; write outputs with
`write_file`. Never pass state through chat.

## Rules (each prevents a specific test-file defect seen in production runs)

1. **Build every fixture as a WHOLE typed object literal from the config interface — never mutate.**
   The config interfaces declare their fields `readonly`, so assigning a property after construction
   fails `tsc` with `TS2540`. Construct each fixture in a single object literal typed to the config
   interface:
   - right: `const cfg: PaceMessageConfig = { StackName: 'PaceMessageStack', Environment: 'dev', /* …all fields… */ };`
   - wrong: `const cfg = baseCfg; cfg.Environment = 'qa';`  // TS2540

2. **To make a variant, SPREAD into a new object — never assign into the base.** Vary one field by
   spreading the base (and any nested group) into a fresh object; never write back to a readonly
   field or `delete` a key:
   - right: `const dark = { ...base, SessionStore: { ...base.SessionStore, shouldProvision: false } };`
   - wrong: `base.SessionStore.shouldProvision = false;`  // TS2540
   - wrong: `delete base.Environment;`                    // TS2703

3. **Include every REQUIRED config field so the stack constructs.** A fixture missing a required
   field fails `tsc` (`TS2741`/`TS2322`) or throws inside the constructor at `Template.fromStack()`.
   Take the field set and required/optional shape from the config **interface** and the config
   **reader** — the fixture must satisfy the same shape (see `cdk-typescript-correctness` rule 6:
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
   **fix the stack** (see `cdk-typescript-correctness` rule 4 — realise the config value, via the L1
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
   `cdk-typescript-correctness` self-check): no writes to `readonly`, no `delete`, optionals guarded,
   every imported name resolves a type, fixtures satisfy the config interface.

## Output
This skill changes **how** you write test fixtures and expectations; it does not add its own output
artifact. Emit your normal test-authoring chunk/envelope (see `cdk-assertion-testing`). Test files go
under `<root>/src`.

## Verification
`tsc --noEmit` passes on the test files (no `TS2540`/`TS2703`/`TS2741`/`TS2322`); `jest` **runs** every
suite (no "Test suite failed to run"); each fixture is a whole typed object literal (no post-construction
assignment to config); variants are made by spread; and every expected assertion matches the shape the
installed `aws-cdk-lib` actually synthesises for the current stack code.
