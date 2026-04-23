# CLAUDE.md — Security-first template

> **Shared at Claude Code · Ahmedabad, April 2026**
> Companion to the talk *"From prompt to production: an IAM agent that unlocks Okta — in chat, no ticket."*
>
> **Author:** Pratik Bhatt — [linkedin.com/in/bhattpratikm](https://linkedin.com/in/bhattpratikm)
> **Scope:** Global instructions (`~/.claude/CLAUDE.md`). Project-level `CLAUDE.md` files override anything here on conflict.
> **Before you use this:** grep for `<YOUR-*>` placeholders and replace. Read `README.md` first.

---

## 1. Response style

- Be concise, direct, technically precise. No flattery, no hedging filler, no restating the prompt.
- Honest assessment over agreement. Push back on bad ideas with reasoning.
- Lead with the answer; supporting detail after. Never bury the conclusion.
- When uncertain, say so and state what would resolve the uncertainty. Do not invent APIs, flags, or citations.
- Prefer code + short rationale over long prose explanations. Show, don't essay.

## 2. Coding standards

**Go (primary)**
- `gofmt`, `goimports`, `go vet`, `staticcheck` clean before handing back.
- Errors: wrap with `%w`; one sentence, lowercase, no trailing punctuation; sentinel errors only when callers need to `errors.Is`.
- Context: `ctx context.Context` is always the first parameter on any call that does I/O, locks, or can block.
- No `init()` for anything observable. No global mutable state.
- Prefer small interfaces defined at the consumer, not the producer.
- Concurrency: channels for ownership transfer, mutexes for shared state. Always explain which and why in a comment if non-obvious.
- Dependencies: standard library first. New deps need a one-line justification.

**Python (secondary — LLM SDKs, infra glue, legacy integrations)**
- Python 3.12+, type hints mandatory on public functions, `ruff` + `mypy --strict` clean.
- `pyproject.toml` only; no `setup.py`.
- `httpx` over `requests`; `pydantic` for boundaries; `structlog` for logging.
- Async by default for I/O-bound code in new services.

**General**
- **No silent fallback.** Config missing → fail loud at startup, not at request time.
- No `TODO` without a ticket reference: `// TODO(<YOUR-TICKET-PREFIX>-123): ...`.
- Files >400 lines or functions >50 lines need a justification comment or a split.

## 3. Security & data handling (non-negotiable)

Mapped to ISO 27001:2022 Annex A controls where relevant.

- Never hardcode secrets, tokens, or tenant URLs in code, commit messages, or examples. Use env vars or secret manager refs. **[A.8.24]**
- Never paste proprietary, client, or regulated data into a public prompt or third-party URL rewriter/proxy. **[A.5.10, A.8.12]**
- Redact PII, auth tokens, and session IDs in logs. Treat identity-system events as sensitive by default. **[A.8.15]**
- For IAM-adjacent code: least-privilege, deny-by-default, explicit allowlist. Justify every permission. **[A.5.15, A.8.2]**
- Data residency: default to your mandated region (`<YOUR-REGION>`, e.g. `europe-west2`); flag every cross-region call. **[A.5.34]**
- Flag risky operations explicitly before executing: schema migrations, mass deletes, production writes, key rotations, IAM changes, webhook re-registrations. **[A.8.32]**
- **Log first. Alert later. Tune alerts last.** Every auditable event lands in durable storage before any real-time notification is designed. **[A.8.15, A.8.16]**
- **Fail loud, not silent.** A misconfigured service is safer when it refuses to start than when it silently degrades.

## 4. Security patterns from production

Six patterns that survived first contact with users probing a production LLM agent (8,400 users, live April 2026). If you're shipping something with tool-calling access to a sensitive system, start here.

1. **Regex guard before the model sees input.** Block adversarial strings (injection patterns, role-override attempts, data-exfil prefixes) *before* they hit the LLM. Catches the cheap stuff cheaply. Does not replace downstream defences.

2. **Tool-result sanitiser.** Injections hide in *tool output*, not user input. Base64 payload in a profile field. Zero-width characters in a display name. Sanitise what comes *back* from every tool before the model ever sees it.

3. **Post-LLM classifier.** Run a second lightweight check on the model's response for social engineering, off-topic drift, and capability probing. Regex misses emotional-pressure injections — the classifier catches them.

4. **Persistent action limiters.** In-memory counters die on pod restart. Rate-limit actions in durable storage (Firestore, Redis with persistence) — or don't bother.

5. **Audit privilege-graph traversal as its own event type.** `manager_of(manager_of(X))` is a high-signal probe. Log every hop to a dedicated store. Graduate alerts by risk: HIGH → page; MEDIUM → batched review; LOW → log only.

6. **Rich error context.** Don't cap error payloads at 500 characters — you lose the signal for future classifiers and for incident review. Store 5,000+.

## 5. Spec-first development

80% of production code should come from a committed spec, not a one-shot prompt. Template:

```markdown
# Design: <feature>

## Purpose       <one sentence — what problem this solves>
## Scope         in / out of scope
## Constraints   locked decisions (non-negotiable)
## Data model    exact schemas, field types, nullability
## Risks         threats and mitigations

# Commit this file BEFORE any code lands.
```

**Trust the spec, not the diff.** When a generated diff drifts from the spec, the spec wins and the prompt gets re-run. The spec is the contract; the diff is one attempt at meeting it.

## 6. Testing expectations

- **TDD required** for: business logic, parsers/validators, auth/authz decisions, anything touching money or access, anything with tool-calling access to a sensitive system.
- **TDD optional** for: throwaway scripts, one-shot data pulls, prototype spikes explicitly marked as such.
- **Adversarial tests mandatory for every LLM surface.** Direct prompt injection, injection via tool output, social engineering, capability probing, privilege-graph traversal, refusal-case regressions.
- Integration tests at the service boundary (HTTP, pub/sub, DB) — not mocked-to-death unit tests pretending to be integration tests.
- A bug fix ships with a regression test that fails without the fix. No exceptions.
- Coverage is a floor (70%), not a goal. Don't game it.

## 7. Commits, branches, PRs

- Conventional Commits: `feat|fix|chore|refactor|docs|test|build|ci|perf|revert(scope): subject`.
- Subject ≤72 chars, imperative mood, no period.
- Body explains *why*, not *what*. The diff is the *what*.
- One logical change per commit. `git rebase -i` before pushing shared branches.
- Branch: `<YOUR-HANDLE>/<ticket-or-topic>`. Feature work uses worktrees (see §9).
- PRs: describe problem, approach, and what you chose not to do. Include rollback plan for anything production-facing.

## 8. Documentation

- ADR (Architecture Decision Record) required for: language/framework choice, auth model, storage engine, external dependency, anything hard to reverse. Format: context · decision · consequences · alternatives rejected. Store in `/docs/adr/NNNN-title.md`.
- Public functions get doc comments. Private functions only when non-obvious.
- Every repo has a `README.md` answering: what is this, how do I run it, how do I test it, who owns it, how do I deploy it.

## 9. Plugin precedence

When multiple skills or agents could fire for the same task, these win. Do not invoke the skipped entries.

| Task                       | Winner                                                                | Skip                                           |
| -------------------------- | --------------------------------------------------------------------- | ---------------------------------------------- |
| TDD                        | `superpowers:test-driven-development` + `tdd-guide` agent             | `everything-claude-code:tdd-workflow`, `:tdd`  |
| Planning                   | `superpowers:brainstorming` → `writing-plans` → `executing-plans`     | `everything-claude-code:plan`                  |
| Verification               | `superpowers:verification-before-completion`                          | all other verification skills                  |
| Continuous learning        | `everything-claude-code:continuous-learning-v2`                       | `v1` (deprecated)                              |
| Code review (analysis)     | `code-reviewer` agent                                                 | —                                              |
| Receiving review feedback  | `superpowers:receiving-code-review`                                   | —                                              |
| Commit messages            | `caveman-commit`                                                      | —                                              |
| Feature work isolation     | `superpowers:using-git-worktrees`                                     | —                                              |
| Guardrails during edits    | `karpathy-guidelines` (always on for write/edit/refactor)             | —                                              |

**Caveman mode:** OFF for code, commit bodies, security warnings, multi-step destructive ops. ON elsewhere.

## 10. CodeGraph

**If `.codegraph/` exists in the project:** use codegraph tools instead of grep / file scanning.

| Tool                   | Use for                                         |
| ---------------------- | ----------------------------------------------- |
| `codegraph_search`     | Find symbols by name                            |
| `codegraph_context`    | Pull relevant context for a task                |
| `codegraph_callers`    | What calls a function                           |
| `codegraph_callees`    | What a function calls                           |
| `codegraph_impact`     | Blast radius before editing a symbol            |
| `codegraph_node`       | Symbol details + source                         |

**If `.codegraph/` missing:** only offer to run `codegraph init -i` when the repo has >50 source files, the current task is exploration-heavy, and the user hasn't already declined this session.

## 11. Web fetching

- Use Claude's built-in `web_fetch` for URL content — it returns cleaned markdown.
- Do **not** route URLs through third-party rewriters/proxies. Reasons: data egress to an uncontrolled service, added latency, no audit trail. Especially important for any URL containing tenant identifiers, query-string tokens, or internal domains.
- If `web_fetch` fails on a page, try once with a different extraction mode before falling back.

## 12. Long outputs — chunk and checkpoint

Long single-turn outputs are the #1 cause of `Stream idle timeout` errors in Claude Code. Reduce their frequency:

- For plans, ADRs, or MD documents longer than ~200 lines, write to disk in sections with short confirm pauses.
- When populating a filled template, write section-by-section. Confirm progress between sections.
- For long refactors, prefer multiple small edits over one giant rewrite — easier rollback, more stable streams.
- If a tool call returns a very large result (logs, full-file reads >2000 lines), summarise or filter in the *next* step before reasoning over it in-turn.

## 13. Templates — auto-consult on matching intent

Two structured templates live in the same directory as this file.

### `PROJECT_TEMPLATE.CLAUDE.md`
Read when: starting a new project, planning greenfield work, creating a project-level CLAUDE.md, architecting from scratch.

### `TROUBLESHOOTING_TEMPLATE.CLAUDE.md`
Read when: a bug / regression / incident, "why is X slow/broken/failing/flaky" questions, auditing or improving an existing repo.

### Re-reading rule
Consult these files the first time a matching intent appears in a session, **and again whenever the conversation pivots**. Re-read if more than a few exchanges have passed or if the session has covered unrelated work in between.
