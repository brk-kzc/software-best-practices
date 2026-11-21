# Integration & E2E Testing with Coding Agents

When an agent writes your code, tests are no longer just a safety net. They are the agent's **feedback loop**. That changes two things:

1. The agent needs a **fast, reliable, readable** signal it can run on its own.
2. The agent must be **unable to game** that signal (by editing tests, skipping them, or over-mocking).

The examples below use TypeScript + Vitest + Playwright + Testcontainers + Postgres + Claude Code. Every piece has an equivalent in other stacks and agent tools.

---

## 1. Put the test commands in `CLAUDE.md` / `AGENTS.md`

Agents waste a lot of time guessing at test invocations. Spell them out.

```markdown
## Testing
- Unit (run constantly): `pnpm test:unit` · one file: `pnpm test:unit src/orders/price.test.ts`
- Integration (run before saying you're done): `pnpm test:int` (needs Docker; starts its own Postgres)
- E2E (critical flows only): `pnpm test:e2e` · one spec: `pnpm test:e2e tests/e2e/checkout.spec.ts`
- Reset local DB + seed: `pnpm db:reset`
- E2E failure artifacts: `test-results/<test>/trace.zip`, `screenshot.png`

## Rules
- Do NOT edit files in `tests/` or `*.test.ts` to make a failing test pass. If you think a test is wrong, stop and explain why.
- Never add `.skip`, `.only`, retries, or `sleep`/`waitForTimeout`.
- Don't mock our own modules or the database in integration tests. Only mock external HTTP (Stripe, SendGrid) via MSW.
- When finished, list which test commands you ran and their results.
```

**Tier the suite by speed:**

| Tier        | When it runs                        | Target time |
|-------------|-------------------------------------|-------------|
| Unit        | Constantly, in the agent's loop     | < 30 s      |
| Integration | Before the agent says "done"; in CI | < 5 min     |
| E2E         | In CI, or on request                | < 15 min    |

---

## 2. Make the tests hard to tamper with

The most common failure mode: when a test fails, the agent "fixes" the **test** instead of the code. It loosens assertions, adds `.skip`, mocks the thing being tested, or updates snapshots without looking.

### Block test edits during implementation

`.claude/settings.json`:

```json
{
  "permissions": {
    "deny": [
      "Edit(tests/**)", "Write(tests/**)",
      "Edit(**/*.test.ts)", "Write(**/*.test.ts)"
    ]
  }
}
```

Remove these rules for sessions where you *want* the agent writing tests.

### Fail CI on common cheats

`scripts/check-test-tampering.sh`:

```bash
#!/usr/bin/env bash
diff=$(git diff origin/main -- 'tests/**' '**/*.test.ts' '**/*.spec.ts')
echo "$diff" | grep -nE '^\+.*\.(skip|only)\(' && { echo "Added skip/only"; exit 1; }
echo "$diff" | grep -nE '^\+.*waitForTimeout\(' && { echo "Added hard wait"; exit 1; }
echo "$diff" | grep -nE '^-.*expect\(' >/dev/null && echo "WARNING: assertions removed, review carefully"
exit 0
```

### Split the work into two sessions

- **Session A (tests):** *"Write failing tests for [feature] based on this spec. Don't implement anything. Run them and confirm they fail for the right reason."* A human reviews and commits the tests.
- **Session B (implementation):** *"Make the tests in `tests/orders/` pass. You may not modify test files."*

### Check that the tests can actually fail

Coverage doesn't tell you a test works. After implementation:

> *"Temporarily break `calculateTotal` (e.g. drop the discount) and confirm which tests fail. Then revert."*

If nothing fails, the tests are hollow. For a more systematic check, run mutation testing (e.g. `npx stryker run`) on critical modules.

---

## 3. Don't let the agent say "done" while tests are red

`.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [{ "type": "command", "command": "./scripts/agent-verify.sh" }] }
    ]
  }
}
```

`scripts/agent-verify.sh`:

```bash
#!/usr/bin/env bash
out=$(pnpm -s test:unit --reporter=dot 2>&1 && pnpm -s typecheck 2>&1)
if [ $? -ne 0 ]; then
  echo "Tests or typecheck failing. Fix before finishing:" >&2
  echo "$out" | tail -40 >&2
  exit 2   # exit 2 = block the stop and send stderr back to the agent
fi
```

Keep this hook to the **fast** suite. Put integration and e2e in CI, or the agent waits minutes every time it tries to finish.

---

## 4. Integration tests against real dependencies

Agents over-mock by default, which produces tests that pass and prove nothing.

### Real Postgres via Testcontainers

`tests/setup/db.ts` (Vitest `globalSetup`):

```ts
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { execSync } from "node:child_process";

export default async function setup() {
  const pg = await new PostgreSqlContainer("postgres:16").start();
  process.env.DATABASE_URL = pg.getConnectionUri();
  execSync("pnpm db:migrate", { env: process.env, stdio: "inherit" });
  return async () => pg.stop();
}
```

### Isolate each test

Wrap each test in a transaction that's rolled back, so tests are isolated and order doesn't matter:

```ts
beforeEach(async () => { tx = await db.begin(); });
afterEach(async () => { await tx.rollback(); });
```

### Mock only third parties, at the network layer

```ts
server.use(
  http.post("https://api.stripe.com/v1/charges", () =>
    HttpResponse.json({ id: "ch_test", status: "succeeded" }))
);
```

### Make setup one command

The agent should be able to stand up the full stack itself: `make test-env-up` / `pnpm db:reset` plus seed scripts. A devcontainer makes this much more reliable.

### Contract tests at service boundaries

Use OpenAPI schema validation or Pact, so an agent changing one side can't silently break the other.

---

## 5. E2E tests with Playwright

Cover only the **critical user journeys** (sign-up, checkout, core workflow). Push edge cases down to integration and unit tests.

### Config

`playwright.config.ts`:

```ts
export default defineConfig({
  retries: process.env.CI ? 1 : 0,     // no retries locally, so the agent sees real flakes
  reporter: process.env.CI ? "github" : "line",
  use: {
    trace: "retain-on-failure",
    screenshot: "only-on-failure",
    testIdAttribute: "data-testid",
  },
  webServer: {
    command: "pnpm dev:test",
    url: "http://localhost:3000",
    reuseExistingServer: true,
  },
});
```

### Explore, then lock it in

1. Give the agent a browser (e.g. the Playwright MCP server).
2. *"Use the browser to complete checkout as the seeded user `test@example.com`. Note every step and which element you used."*
3. *"Turn that into `tests/e2e/checkout.spec.ts` using `getByRole` / `getByLabel` / `getByTestId` only. No CSS selectors, no `waitForTimeout`. Assert on what the user sees (confirmation text, URL), not internal state."*

### Debug from evidence, not guesses

> *"Read the screenshot and error context in `test-results/<test>/` and tell me what the page showed at the failure point before changing anything."*

### Remove sources of randomness

```ts
await page.clock.setFixedTime(new Date("2026-01-15T10:00:00Z"));
```

Also: seed random values, stub third-party network calls, and use fixed test accounts.

---

## 6. Handle flaky tests

A flaky test is worse with an agent than with a human. The agent will either chase a bug that isn't there or retry until it gets a pass.

- Tag flaky tests `@quarantine`, exclude them from the main run (`--grep-invert @quarantine`), and track them in an issue.
- Agent rule: *"If a test passes on rerun without code changes, report it as flaky. Don't keep rerunning it."*

---

## 7. Review a PR in this order

1. `git diff --stat`: were any test files touched that shouldn't be?
2. **The test diff**: removed `expect`s, broader mocks, snapshot updates?
3. The code.
4. For larger changes, a fresh review agent that didn't write the code:
   > *"Here's the spec and the diff. List behaviors in the spec that no test covers, and any test that would still pass if the feature were deleted."*

---

## The default loop

1. A human writes or approves the test scenarios.
2. The agent writes failing tests and a human checks them.
3. The agent implements until they pass, **without editing the tests**.
4. Integration and e2e tests run before merge.
5. Review the test diff first, then the code.
