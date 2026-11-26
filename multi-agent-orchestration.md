# Multi-Agent Orchestration: Fan Out the Work, Not the Risk

Running five agents in parallel is easy. Making the five results merge into one coherent change is the actual engineering problem.

> **Last verified:** 26 Nov 2026
> **Status:** Stable guidance. Topology names are descriptive, not a product taxonomy.

## Contents

1. [When parallelism actually pays](#1-when-parallelism-actually-pays)
2. [Three topologies](#2-three-topologies)
3. [Slicing: by verifiable unit, not by file](#3-slicing-by-verifiable-unit-not-by-file)
4. [The worker brief](#4-the-worker-brief)
5. [Fan-in is where the cost lives](#5-fan-in-is-where-the-cost-lives)
6. [Who verifies what](#6-who-verifies-what)
7. [Failure modes](#7-failure-modes)
8. [A workflow you can run](#8-a-workflow-you-can-run)
9. [When it must be serial](#9-when-it-must-be-serial)
10. [Limitations](#10-limitations)
11. [Related reading](#11-related-reading)
12. [Sources](#12-sources)

---

## 1. When parallelism actually pays

Every worker pays a fixed setup cost: the system prompt, the tool definitions, reading the repository, rediscovering the conventions. Five workers cost roughly five times that — **you are not parallelising for free, you are buying wall-clock time with tokens.**

Parallelism pays when three things hold:

1. **The slices are genuinely independent** — no shared file, no shared type, no shared migration.
2. **Verification is cheap and local** — each slice can prove itself without the others.
3. **The contract is frozen up front** — the interface every slice must satisfy is decided before anyone starts.

| Task shape | Fan out? | Why |
|---|---|---|
| Ten endpoints following one existing pattern | **Yes** | Independent, individually testable, one contract |
| A mechanical rename or import rewrite across 40 files | **Yes** | Deterministic, verifiable by compile + grep |
| Five similar bug reports in different modules | **Yes** | Bounded scope each, independent tests |
| One feature that needs a design decision first | **No** | The decision is the work; workers will invent five different ones |
| A cross-cutting refactor of a shared type | **No** | Every slice touches the same files; conflicts are guaranteed |
| Hunting a flaky test | **No** | Diagnosis is serial; parallel guesses multiply noise |
| "Build the whole app" | **No** | No contract exists, so there is nothing to converge on |

> **The test before you fan out:** can you write the interface, the test command and the acceptance criteria for slice *i* without knowing what slices *j* and *k* will do? If not, you are not ready to parallelise — you are ready to plan.

## 2. Three topologies

| Topology | Shape | Coordination cost | Best for | Signature failure |
|---|---|---|---|---|
| **Fan-out / fan-in** | N workers, one integrator | Low per worker, high at merge | Independent slices of one contract | Five near-duplicate implementations |
| **Pipeline** | Each item flows through stages, no barrier | Low | Migrating 30 files with the same steps | The slowest item sets the pace |
| **Hierarchical** | Lead + specialists (research, implement, review) | Medium | Ambiguous problems needing exploration | The lead's summary loses the specialist's nuance |

**Fan-out/fan-in** is what most people mean by "multi-agent". Its economics depend entirely on how cheap the fan-in is, which depends on how clean the slices were.

**Pipeline** is underrated: no barrier between stages means one item finishing early does not wait for the slowest. It is the right shape for repetitive, staged work — analyse, transform, verify — per file or per module.

**Hierarchical** is the right shape when the problem is *finding* the work rather than doing it. A research subagent that reads forty files and returns a ten-line map is worth more than five implementers guessing where to start. Keep the lead's plan explicit, or the summarisation loses the detail that made the exploration worthwhile.

## 3. Slicing: by verifiable unit, not by file

The instinct is to slice by directory, because that is how the repository is laid out. It is usually the wrong cut, because directories are not independent — the shared type, the parent component and the migration are all in someone else's folder.

Slice by **what can be verified on its own**:

- ✅ "`POST /orders` accepts a cursor and returns 400 for `limit=101`" — one endpoint, one test file, one verifiable claim.
- ✅ "Every `*.sql` migration gets the standard header" — mechanical and checkable with grep.
- ❌ "The types" and "the code that uses the types" — the second slice is blocked on the first; this is serial work with extra steps.
- ❌ "Half the refactor" — no slice can pass the suite, so no slice can be verified.

Two rules that make slicing safe:

**Freeze the contract first.** Types, API shape, database schema, error format, naming. Publish it in the brief for every worker. Most merge pain in multi-agent work is not conflicting edits — it is five slightly different ideas about what the interface is.

**One owner per file.** Give each worker an explicit allowed-file list and a forbidden list. Two workers editing the same file is not a merge conflict; it is a silent last-writer-wins.

**One worktree per worker.** `git worktree add` per slice gives you isolated branches, isolated test runs, and a clean "throw the whole slice away" button. See [agent guardrails and security](agent-guardrails-and-security.md) for why the worktree is also a security boundary.

## 4. The worker brief

A worker starts with a cold context. Everything it knows is what the brief says plus what it reads. That makes the brief the highest-leverage artifact in the whole setup — and identical briefs across workers the main defence against five divergent conventions.

```markdown
# Slice: keyset pagination for GET /orders

## Contract (frozen — do not change)
- Query params: `limit` (1..100), `after` (opaque base64 of `created_at,id`)
- Response: `{ items: Order[], nextCursor: string | null }`
- Errors: 400 for limit > 100, 400 for a malformed cursor

## Allowed files
- src/orders/endpoints.ts
- src/orders/queries.ts
- tests/orders/pagination.test.ts
Forbidden: db/migrations/**, src/domain/**, package.json

## Definition of done
`./verify.sh` exits 0, and these tests exist and pass:
- Pagination_FirstPage_Returns100
- Pagination_NextPage_NoOverlap
- Pagination_LimitOver100_Returns400

## Do not
- Add dependencies. Rename public functions. Touch other slices' files.
- Edit an applied migration.

## Report back
Files changed, tests added, anything you could not do, and any assumption you had to make.
```

The "Report back" section matters more than it looks: it is what the integrator reads instead of the worker's full transcript. Ask for the four-line version, not a summary of its reasoning.

## 5. Fan-in is where the cost lives

Fan-out is cheap and satisfying; fan-in is where the schedule dies. Assume it is the hard part and budget for it.

- **Integrate serially, yourself.** Not with a "merge agent". An unreviewed model reconciling five diffs is a reviewer with no rubric and write access — the worst combination available.
- **Expect duplicate abstractions.** This is the signature failure: five workers, each needing a date formatter, each writing one, in five styles. Dedupe after merge, not during.
- **Assign shared code an owner.** Either a slice whose only job is the shared utility, or reserve it for the integration pass.
- **Re-run everything after merge.** Green slices do not imply a green integration; that is the difference between `A` and `A ∧ B`.
- **Keep the reports, not the transcripts.** Five transcripts will not fit in your attention or your context; five reports will.

> **Dedupe pass, then re-verify, then review.** In that order. Reviewing before dedupe wastes the review on code that is about to change.

## 6. Who verifies what

Layered verification, because each layer is blind to what the next one sees:

| Layer | Runs on | Catches | Blind to |
|---|---|---|---|
| Slice tests | Each worker, in its worktree | Does this slice do what its brief says | The other slices |
| Slice review | Fresh reviewer per slice (optional) | Slice-internal correctness | Cross-slice conflicts |
| Integration suite | After merge | Does the combined system work | Intent conformance |
| Integrated review | One reviewer, the merged diff | Duplication, contract drift, missing pieces | Problems in slices it never saw |
| Human pass | The running artifact | Everything above, plus judgement | — |

Two rules:

**Review the integration, not the slices** — unless the slices are large enough that a defect would be expensive to find later. Five parallel reviewers generate five times the findings and none of the cross-slice insight.

**Never let a worker be its own verifier.** A worker that graded its own slice is a claim, not evidence. See [verification-first AI coding](verification-first-ai-coding.md) for the ladder.

## 7. Failure modes

| Failure | Symptom | Mitigation |
|---|---|---|
| **Duplicate abstraction** | Five helpers doing the same thing | Freeze the contract with utilities named; dedupe pass; shared-code owner |
| **Contract drift** | Slices disagree on the error shape or the cursor format | Publish the contract in every brief; integration test on the contract |
| **Silent partial completion** | A worker "finishes" having implemented two of four criteria | Per-criterion tests; report-back template demands the gap list |
| **Token multiplier shock** | The bill or the limit arrives before the work does | Fewer, larger, well-specified slices; cheap worker model for the mechanical slices |
| **Runaway retries** | One worker loops on a failing test for 20 turns | Two-strike rule per slice: stop, report the blocker, let a human decide |
| **Confident merge** | Integration looks clean because nobody ran the suite | CI on the merged branch, no exceptions |
| **Convention entropy** | Five styles, five test names, five logging calls | Identical rubric and conventions section in every brief; linter enforces the rest |
| **Scope creep** | A worker "helpfully" refactors an unrelated module | Explicit allowed-file list; forbidden list; diff review on the integration |

Most of these are the same failure: **a slice boundary that only existed in your head.** Write it down in the brief, or the worker will invent its own.

## 8. A workflow you can run

Six steps, with the artifacts that make it repeatable:

1. **Decide the contract and the slice list.** Write `slices/NN-name/brief.md` for each. If you cannot write the done-criteria, the slice is not ready.
2. **Create one worktree per slice.** `git worktree add ../wt/01-pagination -b slice/01-pagination`.
3. **Dispatch the workers with identical rubrics.** Same conventions block in every brief; only the contract and the allowed files differ.
4. **Collect `report.md` per slice.** Files changed, tests added, gaps, assumptions. Not transcripts.
5. **Integrate serially.** Merge in dependency order, run the slice tests after each merge, dedupe shared code, then run the full suite.
6. **One integrated review, then the human pass.** Only then does it become a PR.

```text
slices/
├── 01-pagination/
│   ├── brief.md        # contract, allowed files, done criteria, prohibitions
│   └── report.md       # what was done, tests, gaps, assumptions
├── 02-cursor-encoding/
│   ├── brief.md
│   └── report.md
└── integration-checklist.md   # merge order, dedupe notes, suite runs
```

The `integration-checklist.md` is the artifact that makes the second run faster than the first. What went wrong at merge time is exactly what the next set of briefs should say.

## 9. When it must be serial

| Work | Why serial |
|---|---|
| Schema migrations | Ordering is the semantics; parallel migrations are a broken history |
| Cross-cutting refactors | Every slice touches every file |
| Anything touching auth, money or deletion | Blast radius; one author, one reviewer, one human |
| Urgent production fixes | Coordination overhead exceeds the work |
| Unclear requirements | Parallelism multiplies the ambiguity by N |
| Skill-shaped work (one hard algorithm) | Splitting it does not divide the difficulty, only the context |

A useful heuristic: **if two workers would need to talk to each other, they should be one worker.** Splitting is for work that genuinely does not need to negotiate.

## 10. Limitations

- **The token cost is real.** N workers cost roughly N times the setup, and orchestration adds a coordinator on top.
- **Wall-clock gains are smaller than the count implies.** Fan-in is serial, and the slowest slice gates the merge.
- **Slice independence is an aspiration.** In a real repository, shared code leaks in; the dedupe pass is not optional.
- **Reviews of large integrations degrade.** A 2,000-line merged diff gets a shallow review however good the rubric.
- **Topologies are not a substitute for a spec.** Five workers with a bad contract produce five coherent pieces of the wrong thing.
- **This is not a recommendation to parallelise by default.** For most single-developer work, one careful serial session beats four sloppy ones.

## 11. Related reading

- [Context Engineering for Coding Agents](context-engineering-for-coding-agents.md) — why subagents are a context firewall, and what each worker's cold start costs.
- [Verification-First AI Coding](verification-first-ai-coding.md) — the evidence ladder applied per slice, and the definition of done in the brief.
- [Asking an Agent for a Code Review](review-requests-for-coding-agents.md) — the rubric an integrated review should use.
- [Agent Guardrails and Security](agent-guardrails-and-security.md) — worktrees, sandboxes, and the blast radius of four parallel sessions.
- [AI Coding Workflows: Pitfalls, Philosophies, Tooling & Architecture](ai-coding-workflows-guide.md) — thin vertical slices, the two-strike rule, and where ADRs fit.
- [DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5.5.md) — a cheap worker model is what makes fan-out affordable.
- [All notes](README.md)

## 12. Sources

- Anthropic, [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) — the lead-agent + specialists pattern, and where it actually helps.
- Anthropic, [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — workflows vs agents, and the case for the simplest structure that works.
- Anthropic, [Effective context engineering for AI agents](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents) — subagent isolation and condensed returns.
- [git-worktree](https://git-scm.com/docs/git-worktree) — one isolated checkout per slice.
- [AGENTS.md](https://agents.md/) — nested instruction files, so a worker in `slices/02/` reads the rules for that slice.
- Claude Code: [Commands](https://code.claude.com/docs/en/commands) · [Memory and instruction files](https://code.claude.com/docs/en/memory)
