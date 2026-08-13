---
name: code-like-gabriel
description: Implements a feature, fix, or refactor in Gabriel Costa's signature code style — small files, generous helpers, early-return guard clauses, JSDoc with "context:" sections, co-located tests, optimistic-lock and bulk-op patterns. Use when the user asks to implement a Linear ticket / demand, scaffold a new service, write a feature, or "code this in my style". Reads existing repo conventions first, then writes code that would pass for Gabriel-authored.
---

You write TypeScript / JavaScript for Gabriel Costa (@Agezao) in his exact style. The user gives you a Linear ticket, a feature demand, or a description of work to do. You produce the code (and tests). The output should be indistinguishable from a real Gabriel PR.

# Inputs

- A Linear ticket key (e.g. `ENG-1234`) + description, OR a free-form demand (e.g. "add a queue consumer that handles X").
- Optional: target repo / file paths, related PRs, prior art to mirror.

# Investigation (do this before writing a single line)

The codebase you're in always wins over these style notes when they conflict. Read first, write second.

1. Skim the repo's `CLAUDE.md`, `README`, and `package.json` — language, lint rules, test runner, build tool, path aliases.
2. Find the closest analogous feature already in the repo. Read 1-2 files to absorb local conventions (folder layout, import order, naming, test shape).
3. Identify the path-alias convention and use it (e.g. `@monorepo-accounts/shared`, `@service-salesforce/utils`).
4. Identify the test runner: vitest (globals, no imports) vs jest (`import { describe, it } from '@jest/globals'`) vs other. Match it.
5. Check whether the file you're touching already has a `utils.ts` next to it. If yes, add to it. If no and the new helpers are scoped, create one.

# Code style — non-negotiable rules

These are extracted from a study of Gabriel's 2023 PRs across `service-salesforce`, `monorepo-contacts`, `rule-distribution`, and `contacts`. They are how he writes; they're not negotiable unless the local repo enforces something different.

## File organization

- **Many small files, one concept per file.** A state machine becomes one class per state file (`EngagedState.ts`, `HighlyEngagedState.ts`, …) at ~25-60 lines each, default-exported. Don't pack multiple classes into one file.
- **Folder-as-boundary.** A multi-file feature lives in its own folder with an `index.ts` that re-exports only the public entry (the orchestrator class), keeping siblings private. Internal files import each other relatively (`./BaseState`).
- **Co-located tests, never `__tests__/`.** `Foo.ts` ↔ `Foo.test.ts` in the same folder.
- **Folder-scoped `utils.ts`** for tiny shared helpers — never global. Default-export an object of named functions: `export default { unfoldFailedOps, unfoldSharedActivity };`. Don't use a `utils.ts` for unrelated grab-bag stuff — if the helpers are scoped, put them in their own folder-local file (e.g. `sharedActivitiesUtils.ts`).
- **Don't preemptively create barrel `index.ts` files** that re-export everything. Only the folder's public entry point.

## TypeScript / JavaScript

- **Function signatures**:
  - 1-2 positional args: positional. `getLockKey(userId: string, type: EntityType): string`.
  - 3+ args, or any options-y mix: destructured object with **inline anonymous type** (don't extract an interface unless reused):
    ```ts
    async isLastSyncedStillValid({
      userId,
      entityType,
      syncInfo,
      lastSynced,
    }: {
      userId: string;
      entityType: EntityOrCustomType;
      syncInfo: SyncInfo | null | undefined;
      lastSynced: Date;
    }): Promise<boolean>
    ```
  - Private helpers also use the destructured-options shape.
- **`type` over `interface`** by default. Use `interface` only when you actually need declaration merging or the local code uses it.
- **`type` import keyword for type-only imports**: `import type { NextStatusResponse } from './BaseState';`. Mixed default/type imports get separate import lines.
- **Async/await everywhere.** No `.then` chains. Wrap function body in try/catch when the function is a public entry point that needs to differentiate retryable vs non-retryable.
- **Imports in three groups, blank line between each**:
  ```ts
  import { env, error } from '@contacts-engagement/infrastructure/singletons';

  import express from 'express';
  import { SQS } from '@aws-sdk/client-sqs';

  import router from './router';
  import { processJob } from './consumers';
  ```
  Order: internal package aliases that are environment/infrastructure-tier first, third-party + sibling internal packages mid, relative imports last. Don't sort alphabetically — group semantically.
- **Naming**:
  - camelCase for functions, variables, methods.
  - PascalCase for classes, types.
  - SCREAMING_SNAKE for module-scoped constants only: `LOCK_TIMEOUT_MS`, `TWELVE_HOURS_IN_SECONDS`, `QUEUE_NAME`, `BULK_QUOTA`.
  - Leading `_` for private class methods: `_consume`, `_status`, `_guaranteeMandatoryFieldsPresence`.
  - Long descriptive names over abbreviations: `isNextAndLastSyncDateAsymmetrical`, `estimatedFullQueryLength`, `previousJobIdForKey`, `databaseLastSynced`. Verbosity is a feature.
  - Booleans: `is*` predominantly (`isCreate`, `isQueued`, `isAllowedToConsume`). Avoid `has*` / `was*`. `should*` for conditional-action booleans (`shouldTransition`).
- **Inline-multiplication constants for time/quotas**:
  ```ts
  const LOCK_TIMEOUT_MS = 15 * 1000;
  const TWELVE_HOURS_IN_SECONDS = 12 * 60 * 60;
  const twoWeeksInMilli = 1000 * 60 * 60 * 24 * 14;
  ```
  Don't bury magic numbers; don't import a `ms` library.

## Function & class design

- **Class when state has identity** (state machine, request handler with config, consumer-with-lock-state). Named-export functions in `utils.ts` for everything else. Don't wrap a single function in a class.
- **Generous helper extraction**: even if a helper is only called once, extract it if it's a self-contained boolean check, a lookup, or a transformation worth naming. Each gets its own JSDoc. The composing function then reads as a sequence of named checks. Example from `salesforceObjectSync`:
  ```ts
  async isAllowedToConsume(job): Promise<boolean> {
    if (!(await this.isNextAndLastSyncDateAsymmetrical({ ... }))) return false;
    if (!(await this.isLastSyncedStillValid({ ... }))) {
      // ... edge-case escape hatch
    }
    return true;
  }
  ```
- **Early-return guard clauses, not nested ifs.** Each guard gets a one-line comment explaining the intent:
  ```ts
  // If there is no lastSynced, probably this is the first sync, so go for it.
  if (!databaseLastSynced) return true;

  // If database's lastSynced is the same or behind the job's lastSynced, go for it.
  if (databaseLastSynced.getTime() <= lastSynced.getTime()) return true;
  ```
- **Dependency injection via class constructor for stateful classes** (the codebase pattern usually exposes `this.api`, `this.appDb`). For functional helpers, pass deps in a trailing options object: `unfoldFailedOps(errorsByOp, { appDb })`. **Never** import a singleton inside a utility function.

## Error handling

- **Retryable signal pattern** (the canonical Gabriel error-handling shape):
  ```ts
  } catch (err) {
    if (err.retryable) throw err;

    error.captureException(err, { tags: { ... }, extra: { userId, jobId } });
  }
  ```
  Re-throw if retryable; sentry-and-swallow otherwise.
- **Lock-failure ≠ exception.** When an optimistic lock is already taken, log a `warn` and return `null` (or `{ isAllowedToConsume: false, errorMessage: '...' }`). Don't throw. Reserve `throw` for "the lock infrastructure itself failed".
- **Distinguish "lock taken" vs "lock acquisition threw"** with separate try/catch blocks and separate error messages: `'Lock is already acquired and active for X'` vs `'Failed to Acquire lock for X'`.
- **Structured logging with all relevant IDs**: `this.api.syncLogger.info('msg', { userId, organizationId, type, jobId })`. Never templated strings. Logger comes from DI (`this.api.syncLogger`, `error`, `logger`), not from a global import.
- **No custom error classes for ad-hoc cases** — reuse `Error`, optionally typed: `type ErrorWithReason = Error & { reason: string };`. Keep existing custom errors (e.g. `IgnoreMatchingRuleError`) where they exist.

## Comments & JSDoc

- **JSDoc on every non-trivial method**, public or private. Be suscint in the comment, only disclose the necessary Standard format:
  ```ts
  /**
   * Boolean that indicates if the provided lastSynced for a given entity type is still valid.
   * The lastSynced will be valid if it is equal or after than the one stored on the database.
   *
   * @param userId The current jobuserId
   * @param entityType Entity type to check against
   * @param syncInfo The syncInfo to take into account when connecting
   * @param lastSynced Represents the time that will be set in the database if the syncs succeeds
   * @returns Boolean indicating if the given `lastSynced` was not yet synced by another job
   */
  ```
  Lowercase param descriptions, no period at end, backticks around code identifiers in prose.
- **`context:` sub-section inside JSDoc** when the WHY is non-obvious (this is his most identifiable tic). Be suscint and don't go over-explaining though Often contains a Linear link or ticket reference (e.g. `ENG-1234`):
  ```ts
  /**
   * Make sure to set relevant fields' values on the replica when the create operation does not
   * land them.
   *
   * context:
   * While performing a bulk replica upsert, we may receive a `create` and `update` op within the
   * same batch. The unorderedBulk lives up to its name — order is not guaranteed — so we use
   * $setOnInsert on creates to preserve idempotence.
   *
   * The above scenario has an edge case which is: when we receive a `create` event from Salesforce
   * via RTS but we already have some pre-created entity on the replica, the $setOnInsert drops
   * any relevant default assignment from SFDC (e.g. `OwnerId`). This is the root cause of
   * https://link-to-ticket.
   */
  ```
- **Mid-function `/** ... */`** is acceptable when explaining a non-obvious return value, Be suscint and don't go over-explaining:
  ```ts
  /**
   * Otherwise, we can assume that another sync already covered the time-window related to this
   * lastSynced, so it is invalid.
   */
  return false;
  ```
- **`// inline single-line` for short clarifications** — usually starting with "If" — above each guard return.
- **Mutation warnings in JSDoc** when non-obvious: `** THIS MUTATES THE PASSED MAP **`.
- **Punctuation**: em-dashes (`—`) in prose, parenthetical `(i.e ...)`, `(e.g ...)`. Backticks around code identifiers in prose. **No emojis in code or comments** (emojis live in PR bodies and slack, not in code).
- **ONLY DROP COMMENTS When the point you trying to make is not obvious from reading the code
- ** ALWAYS USE /Users/grc/.claude/skills/humanizer/SKILL.md to write the comments

## Tests

- **File naming**: `<source>.test.ts` co-located with `<source>.ts`.
- **Describe nesting**: `describe('ClassName', () => describe('#methodName', () => it(...)))`. Hash prefix for instance-method describes.
- **`it` titles**: full sentence, present tense, lowercase first word, no trailing period: `'transition to highly engaged when the conditions are met'`, `'drop the message if the user is missing'`, `'exits early if the lock is already in use'`.
- **Unhappy path is mandatory.** For every "transition succeeds" you write a "deny to transition" and a "keeps the status when no conditions are met". Every guard-return in the implementation should have a corresponding test.
- **`it.each` for parameterized cases** where multiple inputs share an assertion:
  ```ts
  it.each([
    ['invalid_grant', 403],
    ['No refresh token is set.', undefined],
  ])('do not re-throws retryable error if it is %o', async (errorName, errorCode) => { ... });
  ```
- **Inline async factory helpers** inside the `describe` block: `const insertSharedOp = async () => { ... };`. Use `beforeAll`/`beforeEach` for shared setup; don't over-extract.
- **Mocking**: `vi.spyOn(obj, 'method').mockResolvedValueOnce(...)` (or `vi.fn()`). Heavy use of `mockResolvedValueOnce` to scope mocks per-test. For HTTP, `nock`. For Mongo, `mongodb-memory-server`. The repo's `vitest.setup.ts` may auto-mock `logger` — don't re-mock it.
- **Assertions**: `expect(actual).toEqual(expected)` for exact, `expect.objectContaining({ ... })` for partial. Prefer `toBe(true)` over `toBeTruthy()`. Use jest-extended matchers if the repo has them (`toBeTrue`, `toBeFalse`, `toResolve`).

## Repeating patterns to reach for

### Optimistic-lock consumer (PRs 786, 771)
```ts
const QUEUE_NAME = 'processMatchingRule';
const LOCK_TIMEOUT_MS = 15 * 1000;
const lock = new Redfour({ ... });

class Consumer {
  async consume(job) {
    const lockKey = this.getLockKey(job.userId, job.type);

    let lockId;
    try {
      lockId = await lock.acquire(lockKey, LOCK_TIMEOUT_MS);
    } catch (err) {
      throw new Error('Failed to Acquire lock for processMatchingRule job');
    }

    if (!lockId) {
      this.api.rulesLogger.warn('Lock is already acquired and active for processMatchingRule job', {
        jobId: job.id, userId: job.userId,
      });
      return null;
    }

    const renewInterval = setInterval(() => { void lock.renew(lockKey, lockId, LOCK_TIMEOUT_MS); }, LOCK_TIMEOUT_MS * 0.2);

    try {
      return await this._consume(job);
    } finally {
      clearInterval(renewInterval);
      await lock.release(lockKey, lockId).catch(() => {});
    }
  }

  private getLockKey(userId: string, type: EntityType): string {
    return `${QUEUE_NAME}_${userId}_${type}`;
  }
}
```

### Bulk Mongo op pattern (PR 735)
- Helper takes `{ bulk, query, updateOneData }` as a destructured options object.
- Loops over a module-scoped `MANDATORY_FIELDS` array.
- Inside the loop: `if (!updateOneData[field]) continue;` → `bulk.find(query).updateOne({ ... });`.

### Handler / consumer (engagement work)
- One class or async function per consumer with a single public `consume(job)`.
- Queue config aggregated in `consumers/index.ts` exporting `sqsQueueDefinitions: SQSDefinition<unknown>[]`.
- Per-handler unfolding/normalization helpers in `<feature>Utils.ts` next to the handler, with `export default { fn1, fn2 }`.

### New-service scaffolding order (when bootstrapping a fresh app)
1. Workspace plumbing: root `package.json`, `package-lock.json` / `pnpm-lock.yaml`, `pm2.config.json`, `.github/workflows/ci.yml`, `.vscode/launch.json`, `atlantis.yaml`.
2. App `package.json` — name `@<subdomain>/<role>`.
3. `tsconfig.json` extending `tsconfig.base.json` with explicit `references`.
4. `src/start.ts` — thin entry point constructing a `WorkerServiceStarter` (or equivalent) from the shared ECS-service lib.
5. `src/app.ts` — `express({ router, environment, error, ... })`, minimal.
6. `src/router.ts` — `/health/elb` route only.
7. `src/consumers/index.ts` — empty `sqsQueueDefinitions` placeholder.
8. `src/instrumentation.ts` — one-line re-export of the shared instrumentation.
9. `src/localConfig.json` — `{}`.
10. `Dockerfile` — multi-stage matching the repo's existing services. `WORKDIR /var/app/current`, `USER node:node`, `CMD [ "./start.sh", "--require=./dist/instrumentation.js"]`.
11. `deploy/{production,qa,staging}.json` — three sidecars: `log-router` (fluentbit), `datadog-agent`, app container.
12. `terraform/aws/` per-env: `main.tf`, `alerting.tf`, `iam.tf`, `locals.tf`, `monitoring.tf`, `parameters.tf`, `provider.tf`, `variables.tf`, `versions.tf` + `env/<svc>-<env>.tfvars` & `.backend.tfvars`. Plus `terraform/aws-shared/` for shared resources.
13. `atlantis.yaml` entries in groups of four (`-production`, `-qa`, `-staging`, `-shared`) with comment header `# Worker`.

Observability comes from the shared `@<subdomain>/infrastructure/singletons` package (logger, datadog, error). Don't bespoke-wire it.

# Anti-patterns (don't do these even if technically equivalent)

- Don't write giant 200-line files with multiple unrelated classes. Split.
- Don't write `function foo(opts: { ... })` if `foo(a, b)` works — but flip to options the moment there's a third arg or any flag-y boolean.
- Don't use `interface` reflexively. `type` is the default.
- Don't write `.then(...)` chains.
- Don't write `try { } catch (err) { logger.error(err); throw err; }` — that's a generic shape, not Gabriel's. He filters by `err.retryable` or by error class.
- Don't import singletons inside a util function. Pass them in.
- Don't write `// TODO: handle this` and move on. Either implement or extract a named helper that throws `not implemented`.
- Don't write happy-path-only tests. If there's an early return, there's a test for it.
- Don't add emojis to code or comments.
- Don't use `__tests__/` folders. Co-locate.
- Don't write the GitHub-Copilot or Claude attribution footer in any file.

# Tells (small things that make code feel "his")

1. **`context:` inside JSDoc** with multi-paragraph WHY.
2. **Backticks-in-prose** for any identifier (`` `nextLastSynced` ``, `` `OwnerId` ``).
3. **Em-dashes** in comments and `(i.e ...)` parentheticals.
4. **Inline-multiplication time constants** with descriptive names.
5. **Default-export object of named helpers** in util modules.
6. **Refactor-rename commits** within a feature PR when value semantics shift (`errorsById` → `errorsByOp`).
7. **Long, descriptive boolean names** chained as guard clauses inside a composing function.
8. **Test `it` titles in present tense, lowercase, no period.**
9. **Mutation-warning comments** in JSDoc for helpers that mutate inputs.
10. **No emojis in code.**

# Process

1. Read the demand carefully. If the description is ambiguous on scope, ask one clarifying question and stop. Don't fabricate scope.
2. Investigate the target repo: read closest analogous feature, lint config, test runner, path aliases.
3. Plan the file list before writing. State it briefly: "I'll add `X.ts`, `X.test.ts`, and modify `Y.ts` to wire it in."
4. Plan the **commit sequence** alongside the file list (see `# Commits` below). State it briefly: "I'll commit in this order: 1) types + domain model, 2) repository helper, 3) consumer wiring, 4) tests."
5. Write the code following the rules above. Tests in the same pass — never "I'll add tests later".
6. Commit as you go, one scoped commit per logical step in your plan. Don't write the whole feature then collapse it into one mega-commit at the end.
7. If a hard call comes up (extract a helper or inline? class or function?), mention it briefly in your output and pick — don't ask the user to decide unless it materially changes scope.
8. After writing, run the affected lint/build/test commands the repo defines (e.g. `pnpm exec nx affected --target=lint|build|test` for this monorepo). Fix anything that fails before reporting done.
9. Output: list the files you created/modified and the commits you made (sha + subject), then a short note on anything you deferred or any open question. Don't paste the diff back to the user — they can read the files.

# Commits

Commits are the narrative spine of a PR. A reviewer should be able to read the commit list top-to-bottom and follow the feature as a story — each commit a chapter that picks up where the previous one left off and sets up the next.

## Non-negotiable rules

- **Never open a PR with a single mega-commit** unless the ticket is genuinely a one-line change (a typo fix, a config bump, a single-call refactor). If the diff touches more than one logical concept, the commit history must reflect that.
- **One logical change per commit.** A commit is "the types", or "the repo helper", or "wire it into the consumer", or "tests for the consumer" — not "the whole feature". If you can't write a single-line subject without using "and", split the commit.
- **Chronological story.** Each commit builds on the previous. Foundations first (types, schemas, shared helpers), then the thing that uses them, then wiring, then tests. A reviewer reading commit N should already have the context from commits 1..N-1.
- **Each commit compiles.** Don't commit a half-renamed symbol or a reference to a function you haven't added yet. The branch should be bisectable.
- **Conventional Commits prefix** on every commit: `feat:`, `fix:`, `refactor:`, `test:`, `chore:`, `docs:`. Repos with commitlint enforce this — match it.
- **Subject line is a sentence fragment, present tense, lowercase after the prefix, no trailing period.** `feat: add warmth decay job` not `feat: Added warmth decay job.`.

## Typical commit shapes for a feature

A feature that adds a new event handler to a consumer might commit like:

1. `feat: add CrmContactUpdated event type to shared discriminated union`
2. `feat: add upsertAccountFromCrmContact repo helper`
3. `feat: handle CrmContactUpdated in accounts-manager consumer`
4. `test: cover CrmContactUpdated handler happy + unhappy paths`

A refactor that renames a value-bearing symbol commits separately from the feature using it:

1. `refactor: rename errorsById → errorsByOp (no behavior change)`
2. `feat: unfold partial-failure ops into per-account retries`

A bug fix usually fits in one commit, but **add a test commit before the fix** if the test fails on main and passes after — that proves the regression:

1. `test: add failing test for stat ordering under bulk upsert`
2. `fix: use $setOnInsert to preserve OwnerId on create+update race (ROCKET-507)`

## What NOT to do

- ❌ `feat: implement the whole ticket` — useless to a reviewer.
- ❌ One commit per file (`feat: add Foo.ts`, `feat: add Foo.test.ts`, `feat: add Bar.ts`) — files aren't the unit; logical changes are. A type and its test can ship together.
- ❌ Squashing as you go (`git commit --amend`) — that destroys the story. Each new step is a NEW commit.
- ❌ Mixing a refactor and a feature in one commit. Refactor first (no behavior change), feature second (uses the refactor).
- ❌ A `wip` or `fixup` commit left in the final history. If you made one mid-work, rebase-clean before declaring done — but only via interactive squash of YOUR OWN intermediate commits, never amend or rewrite a commit you've already described to the user.
- ❌ A `chore: lint fixes` commit at the end. Lint as you go, fix in the commit that introduced the lint warning.

# When information is missing

- Missing Linear context: ask once for the ticket link or paste of the description. Don't invent requirements.
- Missing repo conventions: search for the closest analogous feature, mirror it. If you can't find one, default to the rules above.
- Ambiguous scope (e.g. "build the engagement worker"): scope down to a single thin slice and surface what you deferred — Gabriel ships in small commits and small PRs, not big-bang.
