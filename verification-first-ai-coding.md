# Verification-First AI Coding: Turning "Done" Into Evidence

Why an agent's confidence is not a signal, and what to put in its place.

> **Last verified:** 26 Nov 2026
> **Status:** Stable guidance. Tool names are examples; the checks are the point.

## Contents

1. ["Done" is a claim, not a fact](#1-done-is-a-claim-not-a-fact)
2. [The evidence ladder](#2-the-evidence-ladder)
3. [Make the requirement executable](#3-make-the-requirement-executable)
4. [Testing traps that are specific to agents](#4-testing-traps-that-are-specific-to-agents)
5. [Verify the running system, not just the test run](#5-verify-the-running-system-not-just-the-test-run)
6. [Independent review: the second pair of eyes](#6-independent-review-the-second-pair-of-eyes)
7. [Put the definition of done in the build](#7-put-the-definition-of-done-in-the-build)
8. [A minimal workflow](#8-a-minimal-workflow)
9. [Limitations](#9-limitations)
10. [Related reading](#10-related-reading)
11. [Sources](#11-sources)

---

## 1. "Done" is a claim, not a fact

A completion message is a report an agent writes **about its own work, produced by the same process that did the work**. That is not evidence, and treating it as evidence is the single most expensive habit in AI-assisted development.

Three failure modes account for most of the damage, and none of them is visible in the summary:

| Failure mode | What it looks like | Why prose review misses it |
|---|---|---|
| **Plausible but wrong** | The code reads well and implements a reasonable interpretation of the wrong requirement | The diff is internally consistent, so it passes a skim |
| **Right but breaking something else** | The new behaviour works; an unrelated path that depended on the old behaviour does not | The agent never looked at that path, so it did not mention it |
| **A no-op** | A stub, an unreachable branch, a flag that is never set, a handler that is never registered | The diff *looks* like a feature; nothing executes it |

The common thread: verification has to come from a mechanism the agent does not control in the same turn. A test the agent wrote *and* ran *and* reported is one mechanism, weakly; a test that failed before the change and passes after it is a much stronger one.

> **Rule:** never accept a claim you cannot re-run.

## 2. The evidence ladder

Not all evidence is equal. Rank it, and decide in advance which level counts as "done" for each kind of change.

| Level | Evidence | What it actually proves | What it misses |
|---|---|---|---|
| **L0** | "It builds" | Syntax and types line up | Everything behavioural |
| **L1** | The agent says it works, in prose | That the agent can produce confident prose | Everything |
| **L2** | Tests the agent wrote and ran in the same session | Self-consistency | A misread requirement — the test encodes the same misreading |
| **L3** | The pre-existing suite passes on a clean checkout | No regression in behaviour the suite covers | Anything the suite never covered |
| **L4** | A test that **failed before** the change passes **after** it | The change causes the new behaviour | Whether the behaviour is the one the user wanted |
| **L5** | An independent party reproduces it from a clean environment, **plus** a runtime check on the real artifact | The system does it, not the harness | Judgement about *whether it should* |

Two thresholds worth writing into the project's instruction file:

- **Bug fixes: L4.** The failing test is the bug report. If nobody can write one, the bug is not understood yet.
- **Anything a user touches: L5.** "The tests pass" has never once told you what the screen looks like.

L2 is not acceptance. It is a useful smoke signal and a bad finish line.

> **Why L4 beats L2:** writing a test *after* the implementation means the test is fitted to the code. Writing it *before* means the code has to fit the test. Same keystrokes, opposite direction of causality.

## 3. Make the requirement executable

A test is a specification with a compiler. Prose acceptance criteria have too many readings, and the agent will pick one — usually a defensible one you did not intend.

| Vague criterion | Executable version |
|---|---|
| "Handle large inputs" | Importing a 50 MB CSV completes in under 30 s and stays under 512 MB RSS |
| "Validate email addresses" | `a@b` is rejected, `a@b.co` is accepted, and the error names the field |
| "Paginate the orders list" | `limit=101` returns 400; a full page then a cursor returns the next page with no overlap and no gaps |
| "Make it faster" | The p95 of `GET /orders` on the seeded 100k-row dataset is under 150 ms, measured by the existing load script |
| "Handle errors gracefully" | A downstream 500 returns 502 with a request id and logs one structured error line, not a stack trace |

Three rules that make this stick:

1. **Write the checks first, and run them.** They should fail for the *right* reason. If the failure message does not match the bug, you are testing the wrong thing.
2. **Keep the criteria in the repo** (`SPEC.md` next to the change, or the PR description) so a fresh session can reload them after a context reset.
3. **One criterion, one check.** Bundling three assertions into one test makes a failure ambiguous, and ambiguity is what sends agents into fix loops.

## 4. Testing traps that are specific to agents

Agents optimise for the goal you state. If the stated goal is "make the tests pass", you have created a very efficient attacker of your own test suite.

| Trap | How it shows up | Countermeasure |
|---|---|---|
| **Assertion softening** | `assertEquals(3, total)` becomes `assertTrue(total > 0)` to get to green | Treat edits to existing assertions as production changes: require a reason, review the test diff as closely as the code diff |
| **Mock tautology** | The thing under test is mocked out, so the test only proves the mock was called | Keep unit tests for pure logic; test real SQL, real HTTP and real serialisation against real dependencies (containers) |
| **Coverage theatre** | 100% line coverage over getters and generated code | Track **mutation score** on the modules that matter; a surviving mutant is an assertion that does not assert |
| **Silent skipping** | `.skip`, `@Disabled`, `-k "not slow"`, a test file that is no longer collected | CI asserts the expected test count; new skip markers require a reason and an expiry date |
| **Retry-until-green** | Flaky test retried three times, so the flake is now load-bearing | Quarantine and fix flaky tests; never let a retry be the thing that makes acceptance pass |
| **Snapshot churn** | Snapshot files regenerated to match the broken output | Snapshot updates are reviewed diffs, not side effects; a snapshot change is an assertion change |
| **Green on the wrong suite** | Tests are filtered by tag, run against a stale build, or the runner silently exits 0 with zero tests | Assert the run collected ≥ N tests, where N is the baseline count |
| **Environment drift** | Passes locally because a dev server or a seeded DB was already running | Run on a clean checkout against a fresh database from migrations |
| **Wrong failure** | The "failing test" that motivates the fix fails for an unrelated reason (import error, missing fixture) | Read the failure message; if it is not the bug, the test is not the reproduction |

The pattern behind every row: the agent is not malicious, it is responsive. Remove the incentive to fake the signal and the behaviour changes on its own.

## 5. Verify the running system, not just the test run

The suite is a model of the system. The system is the thing. For anything a user touches, do one manual pass per slice — it is the cheapest bug filter that exists.

A pass that takes five minutes and catches real problems:

- [ ] Fresh database, created by migrations only, with realistic seed data (including empty and one-row cases).
- [ ] Walk each acceptance criterion by hand **once**, including one failure path: bad input, unauthorised, not found, conflict.
- [ ] UI: capture before/after screenshots at a fixed viewport. Layout and copy bugs are invisible to assertions.
- [ ] Check the log output: one structured line per request, no duplicated handlers, no swallowed exceptions.
- [ ] Check the artefact that actually ships (built bundle, published image, the deployed commit hash) — not the dev server.

For cross-service changes, contract tests earn their keep: they pin the shape that two teams agreed on, so an agent cannot quietly change one side.

## 6. Independent review: the second pair of eyes

A reviewer with the **same context** as the author inherits the author's blind spots. That is true for models and for humans.

What a useful review session looks like:

- **Fresh context.** New session, no history. Give it: the diff, the acceptance criteria, the test output, and the relevant ADR.
- **A different model or tool** when you can afford it. Decorrelated failures are the entire value.
- **Ask for findings, not approval.** "What is missing, what is wrong, what would break?" gets you a list. "Is this good?" gets you a nod.
- **Review the tests too.** A diff of `*_test.*` files is where the fake signals live.
- **The reviewer may not have written the code.** Self-review in the same turn is theatre.

For high-risk diffs — authentication, authorisation, money, migrations, anything that deletes data — a human signs off, every time, however good the tests are.

## 7. Put the definition of done in the build

If a rule matters, it must be able to fail a build. A paragraph in `AGENTS.md` is a suggestion; a red CI job is a fact.

The strongest cheap pattern is **one script that both you and CI run**, so local and CI evidence cannot drift:

```bash
#!/usr/bin/env bash
# verify.sh - the definition of done, executable.
set -euo pipefail

git diff --check                      # whitespace errors, leftover conflict markers
./scripts/lint.sh                     # formatter check + linters + type check
./scripts/test.sh                     # unit tests
./scripts/test-integration.sh         # needs Docker; real database, real HTTP
./scripts/assert-tests-ran.sh --min 412   # a silently skipped suite is a failure
./scripts/check-migrations.sh         # applied migrations are immutable
./scripts/check-new-skips.sh          # new .skip / @Disabled needs a reason + expiry
```

Then in the root instruction file, exactly one line:

```text
Done = `./verify.sh` exits 0 on a clean checkout. Nothing else counts as done.
```

Two related habits:

- **Compare test counts, not just exit codes.** An empty suite exits 0.
- **Never fix CI by weakening the check.** Widening a threshold or adding a skip is a decision that needs a name attached to it.

## 8. A minimal workflow

Six steps. The order matters more than the tools.

1. **Restate the requirement as 1–5 executable checks.** Get them agreed before any code exists.
2. **Ask for the failing tests first.** Run them yourself. Confirm each failure message is the actual bug.
3. **Let the agent implement,** with the failing tests as the target. New dependencies need a one-line justification.
4. **Run `verify.sh` on a clean checkout.** Check the test count moved the way you expected.
5. **Send the diff to a fresh-context reviewer** with the criteria and the test output.
6. **Do the human pass** on the running artifact. Merge.

When a fix fails twice, stop re-prompting. Revert to the last clean commit, find the cause yourself, and hand back a short, targeted instruction — the two-strike rule, described in the [workflows guide](ai-coding-workflows-guide.md).

## 9. Limitations

- **Checks only confirm what you specified.** A misread requirement passes every test you wrote, cheerfully and in green.
- **Mutation testing is slow and noisy.** Run it on critical modules, not on the whole repo.
- **E2E suites are expensive and flaky.** The flake budget is real; a suite people distrust is worse than no suite.
- **Reviewers share blind spots.** Different model families help; they do not remove the problem.
- **Verification can cost more than implementation.** A one-line change rarely justifies a five-level evidence ladder. Scale the ladder to the blast radius.
- **This note describes practice, not proof.** None of it is a guarantee against a determined wrong answer.

## 10. Related reading

- [AI Coding Workflows: Pitfalls, Philosophies, Tooling & Architecture](ai-coding-workflows-guide.md) — the two-strike rule, the guardrail hierarchy, and the ADR template this article leans on.
- [Context Engineering for Coding Agents](context-engineering-for-coding-agents.md) — why a reviewer with fresh context sees things a long session cannot.
- [DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5.5.md) — the "escalate when tests fail" routing idea this article makes measurable.
- [Agent Guardrails and Security](agent-guardrails-and-security.md) — what to do when the change is not merely wrong but dangerous.
- [Asking an Agent for a Code Review](review-requests-for-coding-agents.md) — the reviewer session, specified as a prompt with an output contract.
- [Multi-Agent Orchestration](multi-agent-orchestration.md) — running the evidence ladder once per slice, then again on the merge.
- [All notes](README.md)

## 11. Sources

### Tools and documentation

- [Mutation testing](https://en.wikipedia.org/wiki/Mutation_testing) — the concept behind the "surviving mutant" check.
- [Stryker Mutator](https://stryker-mutator.io/) — mutation testing for JavaScript/TypeScript, C#, Scala and more.
- [Playwright: visual comparisons](https://playwright.dev/docs/test-snapshots) — snapshot-based UI checks.
- [Testcontainers](https://testcontainers.com/) — real databases and services in integration tests.
- [git-diff](https://git-scm.com/docs/git-diff) — `--check` for whitespace errors and conflict markers.
- [AGENTS.md](https://agents.md/) — the convention for the "Done = `./verify.sh`" line.
- Claude Code: [Commands](https://code.claude.com/docs/en/commands) · [Memory and instruction files](https://code.claude.com/docs/en/memory)

### Practice notes

- [Agent Guardrails and Security](agent-guardrails-and-security.md) — the enforcement side of the same problem.
- Anthropic, [Effective context engineering for AI agents](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents) — why a separate reviewer session is not just a nicety.
