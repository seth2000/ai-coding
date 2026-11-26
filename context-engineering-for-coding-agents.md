# Context Engineering for Coding Agents: Spend the Attention Budget on Purpose

Most "the agent got dumber" stories are context stories. This is the working model I use to keep sessions sharp.

> **Last verified:** 26 Nov 2026
> **Status:** Stable guidance. Token figures are order-of-magnitude estimates, not measurements.

## Contents

1. [The window is a budget, not a shelf](#1-the-window-is-a-budget-not-a-shelf)
2. [What actually occupies it](#2-what-actually-occupies-it)
3. [Four levers](#3-four-levers)
4. [Instruction files: an index, not a manual](#4-instruction-files-an-index-not-a-manual)
5. [Read less: retrieval discipline](#5-read-less-retrieval-discipline)
6. [Tool output hygiene](#6-tool-output-hygiene)
7. [Isolate: subagents as context firewalls](#7-isolate-subagents-as-context-firewalls)
8. [Survive the long task: checkpoints and compaction](#8-survive-the-long-task-checkpoints-and-compaction)
9. [Prune the tool surface](#9-prune-the-tool-surface)
10. [Diagnose: symptom, cause, fix](#10-diagnose-symptom-cause-fix)
11. [A worked budget](#11-a-worked-budget)
12. [Limitations](#12-limitations)
13. [Related reading](#13-related-reading)
14. [Sources](#14-sources)

---

## 1. The window is a budget, not a shelf

A large context window is capacity, not attention. As the number of tokens grows, a model's ability to use any *particular* one of them degrades — a well-documented effect sometimes called **context rot**. The reason is structural: self-attention is quadratic in sequence length, so every added token dilutes the pairwise relationships the model relies on.

Two consequences follow, and they are the whole article:

1. **Bigger windows do not solve long tasks.** They move the failure later, and make it quieter.
2. **Context quality is an engineering variable you control.** You decide what enters, when, and in what form.

> **The goal is not a small prompt.** It is the smallest set of high-signal tokens that reliably produces the outcome — which is often *more* than a small prompt, just much less than an unfiltered one.

## 2. What actually occupies it

Rough costs for a typical coding-agent session, so you know where the money goes. Treat the numbers as order-of-magnitude; measure your own.

| Component | Typical size | Grows with | Under your control? |
|---|---|---|---|
| System prompt + tool definitions | 5–15k | Number of connected tools and servers | Yes, by pruning tools |
| Root instruction file (`AGENTS.md` / `CLAUDE.md`) | 1–3k | Number of rules you keep in one file | Yes |
| Task description | 0.5–2k | How much you paste instead of summarise | Yes |
| Scoped rules loaded on demand | 0.5–2k each | How often the agent wanders between areas | Partly |
| File reads | 0.2–2k per file | Reading whole files instead of ranges | Yes |
| Tool output (tests, builds, `git diff`) | 0.5–20k per call | `cat`, `npm install` chatter, full CI logs | Yes |
| Conversation history | 1–5k per turn | Every turn, including failures | Yes, by starting fresh |
| Agent reasoning / plan text | 0.5–3k per turn | Deliberation on hard problems | No |

The two silent killers are usually **tool output** and **history**. A single unfiltered `npm install` plus a failing test log can cost more than everything you wrote on purpose, and it stays in the window for the rest of the session.

## 3. Four levers

Every context decision is one of these:

| Lever | Question | Typical moves |
|---|---|---|
| **Exclude** | Does this need to be here at all? | Drop unused MCP servers; delete historical rules; do not paste the whole file |
| **Defer** | Can it be loaded only when relevant? | Pointers from the root file to ADRs; search-then-read; scoped rules |
| **Compress** | Can it be smaller and still sufficient? | `head`/`tail`, `--short`, `-q`, summary of a diff instead of the diff |
| **Externalise** | Can it live outside the window? | Write the plan to `NOTES.md`; save long output to a file and read a range |

**Exclude** is the only free lever, and it is the one people skip. **Externalise** is the only lever that survives a context reset, which makes it the most valuable on long tasks.

## 4. Instruction files: an index, not a manual

The root instruction file has one job: get a new session productive in under two minutes.

- **Aim for under 100 lines.** Past ~150, it gets skimmed like documentation — because it is.
- **Put in it:** how to build and test, the hard boundaries, the definition of done, and *pointers* to detail.
- **Keep out of it:** project history, style essays, framework tutorials, and anything that must never happen (that belongs in a linter, a hook, or CI).
- **Prefer pointers over copies.** Detail that lives in one place cannot contradict itself.

```markdown
# AGENTS.md (index form)

## Commands
Build: `pnpm build` · Test: `pnpm test` · Full check: `./verify.sh`

## Boundaries
- `src/domain/**` must not import `src/infra/**`.
- Applied migrations are immutable.

## Read before acting
- SQL or repositories -> `docs/adr/0002-data-access.md`
- Anything touching billing -> `docs/adr/0007-billing.md`

## Done
`./verify.sh` exits 0 on a clean checkout.
```

**Nested files beat one big file.** Most agents read the nearest instruction file in the directory tree, with the closest one winning. That gives you progressive disclosure for free: the data-access rules cost tokens only when the agent is actually in the data-access folder.

> **One line, one rule.** If a rule needs a paragraph of justification to be followed, it is a lint rule waiting to be written.

## 5. Read less: retrieval discipline

The agent's search behaviour is trainable in about three sentences in the root file:

```text
Before reading, locate: grep for the symbol, then read the smallest range that answers the question.
Read whole files only if they are under ~200 lines.
State the files you intend to touch before editing them.
```

Why each line earns its place:

- **Locate first.** Fifty search hits of one line each cost less than one wrong 900-line file, and they narrow the read.
- **Ranges over files.** A 1,000-line file is ~10–15k tokens; lines 120–180 are ~800 tokens and usually contain the answer.
- **Declare the plan first.** Five lines of intended file list is the cheapest possible way to catch "this edit will touch 40 files" before it happens.
- **Ask for the shape of the change, not the whole file back.** A summary of a diff is cheaper than the diff, and far cheaper than the file.

For "where is X handled?" questions across a large repo, do not run the search in the main session at all — hand it to a subagent (section 7) that reads forty files and returns ten lines.

## 6. Tool output hygiene

Tool output is the highest-leverage thing you can fix, because it is pure overhead and it persists.

| Instead of | Use |
|---|---|
| `cat large-file.json` | `head -c 2000`, or `jq '.[0:3]'` |
| `npm install` | `npm ci --silent` |
| Full CI log | The failing job's section, or `grep -A20 -i error build.log` |
| `git diff` on a large refactor | `git diff --stat` first, then the specific files |
| Verbose test runners | `pytest -x -q --tb=short`, `dotnet test --logger "console;verbosity=minimal"` |
| A 3,000-line build output | Redirect to `.tmp/build.log`, then read the range you need |

Two habits that pay for themselves:

- **Save, then slice.** Long output goes to a file outside version control; the agent reads the interesting range. The file survives compaction, the transcript does not.
- **Summarise after a dump.** "In two sentences: what did that output tell us, and what is the next command?" turns 5k tokens of noise into a durable conclusion.

## 7. Isolate: subagents as context firewalls

A subagent is a **context isolation primitive** before it is a parallelism trick. It can explore for tens of thousands of tokens and return a condensed result — often on the order of 1–2k tokens — while the parent session's window stays clean.

Good uses:

- **Research:** "Find every place that constructs a database connection and report the pattern, with file:line." Returns a list, not 40 files.
- **Review:** a fresh-context reviewer that has never seen the implementation is the single best defence against a misread requirement. See [verification-first AI coding](verification-first-ai-coding.md).
- **Bulk mechanical work:** a migration across 30 files, reported as a summary plus failures.

Bad uses:

- **Anything needing the parent's full conversation state.** The child starts with a clean window by design; that is the feature and the limitation.
- **One-line questions.** You pay the summary cost for no isolation benefit.

> **Context firewall, not delegation of accountability.** The parent still owns the decision. A subagent's conclusion is evidence to check, not an authority to trust.

## 8. Survive the long task: checkpoints and compaction

Every session that outlives its window needs a way to hand off to the next one. Three techniques, in order of reliability:

**1. Structured notes (best).** The agent maintains `NOTES.md` — progress, decisions, next step — and reads it at the start of a new session. Bulletproof, because it fails loudly: a stale note is visible, a lossy summary is not.

```markdown
# Task: keyset pagination for GET /orders

## Goal
Cursor paging with no OFFSET; 100 max page size.

## Done
- Migration V002 with the composite index
- Repository method + unit tests

## In progress
- `OrderEndpoints.List` still passes `page`/`pageSize`; needs the cursor wire format

## Next step
1. Change the handler to accept `?after=<base64(created_at,id)>`
2. Add the 400 test for `limit=101`

## Decisions
- Cursor is opaque base64 JSON; keeps the API free to change the encoding.
- Kept OFFSET for admin lists under 200 rows.

## Commands
- `docker compose up -d postgres && ./scripts/test-integration.sh`
```

**2. Compaction (good, lossy).** Summarise the history and restart with the summary. Keep the decisions, the open bug, and the most recently touched files; drop redundant tool output. Do it *before* the window is full and quality has already degraded — compacting at the point of failure preserves the failure.

**3. Fresh session (best for clarity, worst for continuity).** One task per session. When a session has served its task, abandon it rather than nursing it.

> **Order of operations:** write `NOTES.md` **first**, then compact or clear. The note is what makes the reset safe, and it costs a few hundred tokens.

## 9. Prune the tool surface

Every connected MCP server contributes tool descriptions to every request, whether or not you use it on that task. Bloat has a second cost beyond tokens: overlapping tools create ambiguous choice points, and an agent that hesitates between two similar tools is an agent that picks wrong.

A short hygiene routine:

- **Per project, not per machine.** A three-server setup for the repo you are in beats twelve servers "just in case".
- **Prefer a shell over a bespoke tool** when the CLI is stable. It is one tool you already pay for, and it composes.
- **Delete what you cannot name the purpose of.** If you cannot say which task needed it last week, it is noise.
- **Check the loaded tool list, not the intended one.** Harnesses usually expose a command that shows what is actually in context.

## 10. Diagnose: symptom, cause, fix

| Symptom | Likely cause | Fix |
|---|---|---|
| Repeats a rule you gave in the first message | The rule is 40k tokens back, or buried in a long root file | Move it up, shorten it, or make a linter enforce it |
| Contradicts an earlier decision | The decision lived only in chat, and compaction dropped it | Write decisions to an ADR or `NOTES.md` |
| Invents APIs and functions that do not exist | No grounding — it never read the definition | Require a search and a read before writing; name the file in the instruction |
| Answers degrade after `/compact` | The summary lost the subtle constraint | Write a handoff note before compacting |
| Edits are small and timid despite a clear ask | Tool output is flooding the window; little room left for reasoning | Filter output, save to a file, start a fresh session |
| Constantly re-reads the same files | No map of the repo in the root file | Add a short "where things live" table |
| Slow, expensive turns that change nothing | Tool bloat, or too many parallel explorations in one session | Prune servers, move research to subagents |
| Confidently wrong about the requirement | The requirement arrived as a paragraph | Restate as executable checks (see the verification article) |

## 11. A worked budget

A realistic mid-size task, aiming to stay under roughly a third of a 200k window so reasoning still has room:

| Item | Cost | Running total |
|---|---|---|
| System prompt + tool definitions (6 servers) | ~12k | 12k |
| Root `AGENTS.md` (80 lines) | ~1.5k | 13.5k |
| Task + acceptance criteria | ~1k | 14.5k |
| Targeted reads (6 files, ranges only) | ~5k | 19.5k |
| Test output, filtered (8 runs) | ~4k | 23.5k |
| Failure-and-fix history (12 turns) | ~18k | 41.5k |
| Plan and summaries | ~3k | 44.5k |

Now the same session done badly: whole-file reads (~40k), one unfiltered `npm install` (~8k), two full CI logs (~30k), and no pruning of the unused MCP servers (~9k extra). That is ~132k before the model has done anything difficult — and the remaining room is exactly where the reasoning was supposed to go.

## 12. Limitations

- **Token counts are estimates.** Measure with your harness's own reporting rather than trusting this article's table.
- **Aggressive pruning has a cost too.** Removing context the agent needed produces confident wrong answers, which are harder to spot than slow ones.
- **Compaction quality is model-dependent,** and there is no universal "keep these five files" rule.
- **Subagent isolation is not free.** Each one costs a full session setup, and summaries lose detail the parent may later need.
- **Tooling moves fast.** Specific commands and thresholds here are point-in-time; the levers are the part that lasts.

## 13. Related reading

- [AI Coding Workflows: Pitfalls, Philosophies, Tooling & Architecture](ai-coding-workflows-guide.md) — where the lean root file and the ADR pattern come from.
- [Verification-First AI Coding](verification-first-ai-coding.md) — the reviewer session this article's isolation argument depends on.
- [DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5.5.md) — why long-context work is where a stronger model earns its price.
- [Agent Guardrails and Security](agent-guardrails-and-security.md) — what to do when the context itself is hostile.
- [Asking an Agent for a Code Review](review-requests-for-coding-agents.md) — what to put in a reviewer's window, and what to keep out.
- [Multi-Agent Orchestration](multi-agent-orchestration.md) — the subagent firewall turned into a working topology.
- [All notes](README.md)

## 14. Sources

- Anthropic, [Effective context engineering for AI agents](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents) — attention budget, just-in-time retrieval, compaction, structured notes, subagent summaries.
- [Context rot](https://research.trychroma.com/context-rot) — why recall degrades as context grows.
- [AGENTS.md](https://agents.md/) — the open convention, nested-file precedence, and the "README for agents" framing.
- [Model Context Protocol](https://modelcontextprotocol.io/) — what MCP servers add to the tool surface.
- Claude Code: [Commands](https://code.claude.com/docs/en/commands) · [Memory and instruction files](https://code.claude.com/docs/en/memory)
- [Writing tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — why bloated, overlapping tool sets cause bad decisions.
