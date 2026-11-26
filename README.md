# AI Coding Notes

Practical, opinionated notes on building real software with AI coding agents — workflows, verification, context, review, orchestration, security, and model choice.

These are **working notes, not documentation**. Each one records what I currently believe, what I checked, and what I have not proven. Where a claim is a community report rather than a reproduced result, the note says so.

## Contents

| Note | What it covers | Kind |
|---|---|---|
| **[AI Coding Workflows](ai-coding-workflows-guide.md)** | The common failure modes, the Tech Lead vs Product Manager split, a lean `AGENTS.md` with progressive disclosure and ADRs, and a worked .NET + PostgreSQL example | Guide |
| **[Verification-First AI Coding](verification-first-ai-coding.md)** | Turning "done" into evidence: the evidence ladder, executable acceptance criteria, agent-specific test traps, independent review, and one `verify.sh` that CI also runs | Guide |
| **[Context Engineering for Coding Agents](context-engineering-for-coding-agents.md)** | Treating context as a fixed budget: what fills the window, four levers, retrieval discipline, subagents as context firewalls, checkpoints and compaction | Guide |
| **[Agent Guardrails and Security](agent-guardrails-and-security.md)** | The agent as a confused deputy: untrusted inputs, prompt injection in real repos, sandboxing, egress control, credentials, supply chain, and the merge gate | Guide |
| **[Asking an Agent for a Code Review](review-requests-for-coding-agents.md)** | Review requests that produce findings instead of approval: the six-part request, an output contract, evidence-seeking questions, a layer split, and four copyable prompts | Guide |
| **[Multi-Agent Orchestration](multi-agent-orchestration.md)** | When fan-out pays, three topologies, slicing by verifiable unit, the worker brief, fan-in as the real cost, and the failure modes that come with N workers | Guide |
| **[DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5-5.md)** | Prices, context, benchmarks and community tests, plus a planner/worker routing strategy with an honest limitations section | Research note |

## Start here

- **Your AI-assisted project is drifting and you are not sure why** → [AI Coding Workflows](ai-coding-workflows-guide.md), then [Verification-First AI Coding](verification-first-ai-coding.md).
- **The agent was great last week and seems to be getting worse** → [Context Engineering](context-engineering-for-coding-agents.md). It is almost always context, not the model.
- **You are about to point an agent at a repo with real credentials in it** → [Agent Guardrails and Security](agent-guardrails-and-security.md) first, everything else after.
- **You want a second opinion on a diff you do not want to read twice** → [Asking an Agent for a Code Review](review-requests-for-coding-agents.md).
- **You are about to run several agents at the same time** → [Multi-Agent Orchestration](multi-agent-orchestration.md) before you open the second terminal.
- **You are choosing models, or deciding whether the expensive one is worth it** → [DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5-5.md).

The six guides form one argument: **the workflow sets the constraints, verification makes them checkable, context keeps the session able to follow them, and review is how someone other than the author enforces them.** Orchestration is that same loop with several workers; security decides what happens when a constraint fails.

## Conventions

- **One topic per file.** A note that needs two theses is two notes.
- **Every note states its date.** "Last verified" means the links and tool behaviour were checked then — not that the advice is permanently true.
- **Every note has a limitations section.** If it does not, the note is not finished.
- **Claims are labelled.** Benchmarks and prices are sourced and linked; community anecdotes are marked as reported, and kept out of factual tables.
- **Examples are fictional.** `ShopLite`, the repositories and the projects are illustrative; no private code is reproduced here.
- **Code blocks are tested by reading, not by CI.** Treat snippets as sketches to adapt, not files to paste.

## Status

Living documents. Notes get revised when a practice stops working or a number changes; the `Last verified` line is the only promise about freshness.

Corrections are welcome — especially reproduced benchmark numbers, or a guardrail that turns out to be bypassable. Open an issue with the specifics, or a PR that changes one section and says why.

## License

[MIT](LICENSE). Take the text, take the snippets, adapt them, publish your own version — keep the copyright notice and you are done.

## About

Maintained by [@seth2000](https://github.com/seth2000). Repo: [seth2000/ai-coding](https://github.com/seth2000/ai-coding).

AI tooling changes fast. Every note here should be read with its date attached, and re-checked before it becomes load-bearing in someone else's process.
