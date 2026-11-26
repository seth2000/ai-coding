# Asking an Agent for a Code Review: Prompts That Produce Findings

Reviewer agents fail in two directions at once — they approve too easily, and when they do complain, they complain about the wrong things. Both are prompt-design problems.

> **Last verified:** 26 Nov 2026
> **Status:** Stable guidance. Template wording is a starting point; the output contract is the part that matters.

## Contents

1. [Two failure directions](#1-two-failure-directions)
2. [The minimum viable review request](#2-the-minimum-viable-review-request)
3. [The output contract](#3-the-output-contract)
4. [Turn opinions into evidence](#4-turn-opinions-into-evidence)
5. [What context to give, and what to withhold](#5-what-context-to-give-and-what-to-withhold)
6. [Layer the review: don't spend tokens on lint](#6-layer-the-review-dont-spend-tokens-on-lint)
7. [Calibrate the reviewer, not the prompt](#7-calibrate-the-reviewer-not-the-prompt)
8. [Four prompts you can copy](#8-four-prompts-you-can-copy)
9. [Anti-patterns](#9-anti-patterns)
10. [Limitations](#10-limitations)
11. [Related reading](#11-related-reading)
12. [Sources](#12-sources)

---

## 1. Two failure directions

| Failure | What it looks like | Root cause |
|---|---|---|
| **Approval bias** | "This looks good overall, a few minor suggestions" — on a diff with an unhandled null and no authz check | The prompt asked for a judgement, and judgement defaults to agreeable |
| **Coverage mode** | Forty comments: naming, spacing, "consider extracting a helper", zero on correctness | No rubric, so every observation has equal weight |
| **Diff-only tunnel vision** | Detailed review of the changed lines, silent about the code path the change broke | It was never told what the change was *supposed* to do |
| **Reviewer turns author** | It rewrites the implementation instead of describing the problem | You asked it to "fix anything it finds" |

These are not model defects you can wait out. A reviewer with no rubric optimises for the most visible thing — style — because style is verifiable in one line, and correctness is not.

> **The ask determines the answer.** "Is this good?" and "What would break in production?" are the same diff and completely different reviews.

## 2. The minimum viable review request

Six parts. If a review request is missing more than one, expect a vague answer.

1. **Scope** — what is under review, and what is explicitly out of scope (other branches, uncommitted experiments, the legacy module).
2. **Artifact** — the diff itself, not a summary of it. Small enough to fit; split it if not.
3. **Intent** — the acceptance criteria, so "wrong" has a definition. Reviewers without intent can only check internal consistency.
4. **Rubric** — the ordered list of what counts as a finding, and what does not.
5. **Output contract** — the exact shape of the answer: label, severity, `file:line`, scenario, verification step.
6. **Prohibitions** — do not edit, do not approve, do not comment on anything a linter decides.

Two of these are usually missing, and they are intent and the rubric. Without them the model can only tell you whether the code is *tidy*, which is exactly what CI already tells you.

## 3. The output contract

Ask for a shape you can act on, and refuse prose. A review you cannot triage is a review you will skim.

Adapted from [Conventional Comments](https://conventionalcomments.org/) — the labels are worth stealing because they make "blocking vs not" explicit and are machine-parseable:

```text
issue (blocking): Order total overflows for quantities above 2^31
file: src/orders/Total.ts:41
scenario: quantity = 3_000_000_000, unitPrice = 2 -> total returns a negative number
verify by: add a test `Total_BillionQuantity_DoesNotOverflow`
```

Rules that keep the output honest:

- **Every finding cites `file:line`.** A finding without a location is a feeling.
- **Every finding names a failing scenario.** "This could be a problem" is not a finding; "this returns −2 for inputs above 2³¹" is.
- **Every finding ships a verification step** — a test to write, a command to run, an input to try. This is what makes the review auditable instead of rhetorical.
- **Silence must be explicit.** "No findings in categories 2 and 5" is information. An empty list is ambiguous.
- **No approve.** Remove the option entirely. A model that can approve will approve.

| Weak output | Usable output |
|---|---|
| "Error handling could be improved." | "`issue (blocking): fetchOrders ignores a 409 and shows the empty state; verify by stubbing 409 in the handler test." |
| "Consider adding tests." | "`todo: the 400 path for limit=101 has no test; verify by adding it to OrderEndpointsTests`." |
| "This looks correct." | "`question: what happens when two cursors arrive out of order? — would settle with the tie-break test`." |

## 4. Turn opinions into evidence

The single most useful trick: **ask for the failing test, not the worry.** A test is falsifiable; an opinion is not.

| Instead of | Ask |
|---|---|
| "Are there any bugs?" | "Write the test that fails because of a bug in this diff. If there is none, say so." |
| "Is this robust?" | "Give me the input that makes this throw. Only use the public interface." |
| "Did I miss anything?" | "List the acceptance criteria, then mark each CONFORMS / PARTIAL / NOT MET with `file:line`." |
| "Is this secure?" | "For each new entry point, state where authz is enforced, or say `not enforced`." |
| "Is the test good?" | "Which assertion in this test would still pass if the feature were reverted?" |

That last one is the highest-value question in agentic review, because weakened assertions are the most common fake signal in AI-written code. A reviewer that can answer it is doing something you cannot do by skimming.

> **Make the reviewer prove it can fail.** If a reviewer cannot articulate a scenario where the change misbehaves, it has not reviewed the change; it has read it.

## 5. What context to give, and what to withhold

**Give:**

- The diff, in full, for the scope you asked about.
- The acceptance criteria or the ticket, verbatim.
- The test command and its summary output.
- The relevant ADR, or the one interface it must not break.
- The repository conventions that are *not* enforced by tooling.

**Withhold:**

- **Your own conclusion.** "I think this is fine, just double-check" is an instruction to agree.
- **The implementation chat.** The reviewer inherits the author's framing, including its mistakes.
- **Everything else.** Unrelated files, other branches, and the whole session history are noise that dilutes the rubric.

**Fresh context or no review.** A reviewer that watched the code being written is reviewing its own work. Use a new session, and ideally a different model family — that is where the decorrelated errors come from. See [context engineering](context-engineering-for-coding-agents.md) for why a fresh window is qualitatively different, not just emptier.

**Chunk by concern, not by file.** A 1,500-line diff produces a shallow review of everything. Three reviews of 400 lines each — intent, failure paths, tests — produce three findings that matter.

## 6. Layer the review: don't spend tokens on lint

| Layer | Owner | Catches | Does not catch |
|---|---|---|---|
| Format, style, dead code | Formatter + linter | Consistency | Anything semantic |
| Types | Compiler | Shape mismatches | Wrong shapes that type-check |
| Behaviour | Test suite | Regression in covered behaviour | Uncovered paths, misread requirements |
| Intent conformance | Agent review + criteria | Wrong-but-plausible implementations | Whether the requirement was right |
| Failure paths, contracts, security | Agent review with a rubric | Unhandled states, broken compat, missing authz | Abuse you did not think of |
| Design, naming, boundaries | Human | Everything above, plus taste | — |

**The rule:** if a deterministic check can decide it, do not ask a model about it. Not because the model is wrong, but because each wasted finding costs credibility — and a reviewer you have learned to ignore is worse than no reviewer.

Run `./verify.sh` **before** the review, so the reviewer never spends its budget telling you the build is broken.

## 7. Calibrate the reviewer, not the prompt

A review process you cannot measure degrades into ritual. Two cheap metrics:

- **Precision:** of the findings raised in the last ten reviews, how many did you accept? Below roughly half, the rubric is too broad.
- **Recall proxy:** how often does a reviewer finding correspond to something that would have shipped and hurt? Keep a list. It is the only evidence that review is paying for itself.

Then version the rubric. Put it in the repo:

```text
docs/review-rubric.md
```

Not in your shell history. A rubric in the repository can be reviewed, diffed, and improved when a bug escapes — "add a line for the class of bug we just shipped" is the same move as [treating every repeat mistake as a missing guardrail](ai-coding-workflows-guide.md).

> **One escaped bug, one rubric line.** That is the whole improvement loop.

## 8. Four prompts you can copy

### 8.1 Diff review with a rubric

```text
You are reviewing a diff in a repository you have not seen. You did not write this code.

## Artifact
Diff: <git diff main...HEAD>
Acceptance criteria: <paste>
Test command and result: <paste the command and the summary lines>

## Rubric (in priority order)
1. Intent: does the diff implement the acceptance criteria, and nothing else?
2. Correctness: wrong results, unhandled states, empty/null, off-by-one, concurrency.
3. Failure paths: swallowed errors, unbounded retries, partial writes.
4. Contracts: API and schema changes, backwards compatibility, migrations.
5. Security: authz on new entry points, unvalidated input, secrets, injection.
6. Tests: would each new test still pass if the feature were reverted?

## Output
One line per finding, in Conventional Comments form:
  <label> (blocking|non-blocking): <subject>
  file: <path:line>
  scenario: <the concrete input or sequence that misbehaves>
  verify by: <a command or test to write>
Use labels: issue, todo, question, suggestion.
Say "no findings in category N" for every clean category.
List anything in the diff the criteria did not ask for.

## Prohibitions
Do not edit files. Do not rewrite the implementation. Do not comment on formatting,
naming or style — a linter owns those. Do not approve; there is no approve.
```

### 8.2 Adversarial review ("break it")

```text
Your goal is to find one input, sequence or environment in which this change fails.

Deliver, in order:
1. The three most plausible failure scenarios, ranked by impact x likelihood.
2. For the top one: the exact test (path, name, code) and the expected failure message.
3. Which files a fix would have to touch. Do not write the fix.

Constraints: use only the public interface. Do not modify files.
If you cannot find a failure scenario, say "no failure found" and list what you tried.
```

### 8.3 Criteria conformance

```text
Here are the acceptance criteria and the diff. Add no criteria of your own.

For each criterion output exactly one of CONFORMS / PARTIAL / NOT MET,
with file:line evidence, and for PARTIAL or NOT MET the precise gap.

Then output a separate list: everything in the diff that the criteria did not ask for.
Then: any criterion that cannot be verified from the diff alone, and what would verify it.
```

### 8.4 Security-only pass

```text
Review only for security; ignore style, naming and performance.

For each new or changed entry point report:
- where authorization is enforced (file:line) or "not enforced"
- input validation at the boundary (or "absent")
- secrets in code, logs, error messages or URLs (or "none found")
- injection surface: SQL, shell, template, path traversal
- any new dependency: what it does, why it is needed, or "unjustified"

Rate each finding: blocking | should-fix | note.
Where a control does not apply, write "not applicable" rather than omitting it.
```

## 9. Anti-patterns

| Anti-pattern | Why it fails |
|---|---|
| "Review this" | No scope, no rubric, no contract — you get style commentary |
| "Is this good?" | Invites agreement; the model will find something nice to say |
| "Find all bugs" | Unbounded; produces speculation that costs you triage time |
| "Be thorough" | Increases volume, not precision |
| "Fix anything you find" | The reviewer becomes the author, and you lose the only independent check you had |
| Reviewing in the authoring session | Same context, same blind spots |
| Reviewing a summary instead of the diff | You are reviewing the agent's description of its own work |
| Asking about formatting | The linter already answered; you just taught the reviewer to nitpick |
| Accepting findings without reproducing them | Half of them are wrong, and you cannot tell which half |

## 10. Limitations

- **A rubric is a ceiling.** The reviewer finds what your rubric names; categories you never wrote are categories you never get.
- **Sycophancy is reduced, not removed.** Fresh context and a no-approve contract help; they do not make the model adversarial by nature.
- **Precision is expensive to measure.** It needs you to record outcomes for a while before the numbers mean anything.
- **Reviewer and author share training data,** so shared blind spots persist even across model families.
- **Review is not verification.** A finding is a hypothesis until you reproduce it — see [verification-first AI coding](verification-first-ai-coding.md).
- **Templates age.** The wording here is a starting point; the output contract is the durable part.

## 11. Related reading

- [Verification-First AI Coding](verification-first-ai-coding.md) — why the reviewer must be independent, and what "reproduce it" means.
- [AI Coding Workflows: Pitfalls, Philosophies, Tooling & Architecture](ai-coding-workflows-guide.md) — the two-strike rule, and the guardrail hierarchy the rubric feeds into.
- [Context Engineering for Coding Agents](context-engineering-for-coding-agents.md) — fresh sessions, chunking, and what a reviewer's window costs.
- [Multi-Agent Orchestration](multi-agent-orchestration.md) — when the review itself fans out into parallel reviewers.
- [Agent Guardrails and Security](agent-guardrails-and-security.md) — the security pass, and what to do when a finding is a real incident.
- [DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5-5.md) — using a cheap model as the first-pass reviewer and escalating the diff that matters.
- [All notes](README.md)

## 12. Sources

- [Conventional Comments](https://conventionalcomments.org/) — the label/decoration format used in the output contracts above.
- [Google eng-practices: How to do a code review](https://google.github.io/eng-practices/review/reviewer/) — the human standard this borrows from, especially "what to look for" and how to write comments.
- Claude Code: [Commands](https://code.claude.com/docs/en/commands) · [Memory and instruction files](https://code.claude.com/docs/en/memory)
- [AGENTS.md](https://agents.md/) — where the review rubric should live, so a new session always loads it.
- [OWASP Top 10 for LLM and GenAI](https://genai.owasp.org/initiative/owasp-top-10-for-llm-and-genai/) — context for the security-only pass.
