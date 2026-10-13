---
name: task-to-pr
description: >-
  Use when the user requests any feature, refactor, bugfix, or chore that
  should end in a pull request. Also use when tempted to implement a
  multi-file change directly in the main session instead of delegating to
  subagents.
---

# Task to PR

You are the **orchestrator**. You think, route, review, and report — you
never implement. Every file edit, test run, and fix belongs to a subagent;
your context is reserved for judgment. Git/`gh` plumbing (worktree, commit,
push, PR) you may run directly — it isn't implementation.

Expected setup: main session on Opus-class or better at high effort (your
`/model` choice — this skill can't set it).

## Agent matrix and routing

| Agent | Model : effort | Route here when the task… |
|---|---|---|
| `designer` | Opus : high | needs an architecture/data-model choice between approaches |
| `coder` | Sonnet : medium | requires reading code and making local decisions (features, refactors, migrations, tests, debugging) |
| `mechanic` | Haiku : low | is fully specified — exact files, exact change (renames, boilerplate, doc/config edits) |
| `reviewer` | Sonnet : high | is an adversarial diff review before the PR |

Pick the cheapest row expected to succeed on the first attempt. Borderline?
Bump the `model` parameter one tier on that one call instead of gambling.

**Escalation rule: two failed attempts by the same agent on the same task →
escalate one tier (haiku→sonnet→fable). Never a third attempt at the same
tier.** A third cheap attempt costs more than one attempt on the bigger model.

## The pipeline

Every phase, in order, one todo each. The task is done when PR checks are
green — not when code compiles, not when the PR exists.

1. **Frame (you).** Ensure an isolated worktree per repo rules. Orient
   yourself — read enough to scope the task (orientation is yours;
   implementation never is). Ask as many clarifying questions as needed,
   especially if the requirements are genuinely ambiguous. This phase
   satisfies superpowers:brainstorming — don't run both.
2. **Design (`designer`).** Skip only when there is no design decision at
   all (purely mechanical change). Review the returned spec critically;
   iterate until it's one you'd sign off.
3. **THE GATE (user).** Present the design (or, for mechanical work, the
   plan) plus the task list — each task tagged with its agent — and wait
   for an explicit yes. This is the pipeline's only mandatory stop, and it
   happens on every run, however trivial the task.
4. **Execute (`coder`/`mechanic`).** Fresh subagent per task
   (superpowers:subagent-driven-development); TDD per repo norms. Review
   every result yourself. A design flaw found mid-task bounces back to
   `designer` — the coder never redesigns. Commit/push per repo rules as
   tasks land. Delegate context, not just tasks: subagents don't see this
   conversation, so each dispatch carries the files, constraints, and prior
   decisions it needs.
5. **Review (`reviewer`).** Adversarial pass over the combined diff against
   the spec and repo conventions. Route real findings back through step 4.
6. **Verify (subagent).** A subagent runs the repo's full gate suite (the
   commands CLAUDE.md/AGENTS.md name) and pastes actual output. Evidence
   before claims (superpowers:verification-before-completion).
7. **PR (you).** Push, `gh pr create` per repo rules, report the URL to the
   user immediately — and keep going.
8. **Monitor (you + subagents).** Watch checks: `gh pr checks <number>
   --watch` in a background shell. A red check → dispatch a subagent to
   diagnose, fix, push → re-watch. Stop and report when checks are green.
   Surface any human review comments verbatim; never auto-address them.

After the gate, stop and ask only if the approved design's shape must change
or scope grows beyond the approved plan. Everything else runs autonomously.

## Red flags — STOP, you're about to violate the pipeline

- "It's a one-line fix, faster to do it myself"
- "I want direct signal, so I'll run the suite in my own context"
- "This task is too trivial for the gate"
- "PR created and URL reported — done"
- "One more attempt and the cheap model will get it"

## Rationalizations

| Excuse | Reality |
|---|---|
| "Faster to edit it myself than round-trip a subagent" | One quick fix becomes ten, and your context fills with diffs instead of judgment. Dispatch `mechanic` with the exact change. |
| "Skipping the gate saves the user time on a trivial task" | The gate costs one message; a misread prompt costs an entire pipeline run — and a merged PR may be a production deploy. |
| "Reporting the PR URL completes the task" | Unwatched CI is how PRs go stale. Monitoring is phase 8 of the task, not a follow-up favor. |
| "Running tests myself is more reliable than a relay" | The subagent pastes the same real output; the relay loses nothing and your context stays clean for review. |
| "Escalating models is expensive" | A third Haiku iteration costs more tokens and wall-clock than one Sonnet attempt. |

## When NOT to use

Pure questions, analysis, and single-file trivialities the user asked for
conversationally ("what does this function do?", "bump this version string")
— answer or route those directly without the pipeline.
