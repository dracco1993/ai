---
name: creating-issues
description: >-
  Use when the user asks to file, create, write up, or open one or more
  GitHub issues (a single request or a bulleted list of them), or says
  "make an issue for this" about a bug, idea, or follow-up.
---

# Creating Issues

You are the **orchestrator**. You split, route, relay questions, and file
cross-references. Research and drafting belong to one `opus` subagent per
issue; your context is reserved for the user conversation. No code is written
by anyone: no repo edits, no branches, no PRs. A throwaway script in the
scratchpad to confirm a fact for the issue body is the only exception.

**Ambiguity is a question for the user, never a default.** A subagent cannot
reach the user, so it returns questions to you and you ask them. Better one
extra round than a confident issue about the wrong thing.

## The pipeline

1. **Split (you).** One issue per distinct ask. For a list, show the user
   your item split (`A: …`, `B: …`) and confirm it before dispatching. A
   bullet can be two issues or half of one.
2. **Dispatch (you).** One `Agent` per item, `model: "opus"`, `name` set to
   the item key, all in one message. Each gets the prompt template below,
   filled in. Never share one agent across items.
3. **Relay questions (you).** Each agent returns either `QUESTIONS` or
   `DRAFT`. Collect every `QUESTIONS` block and ask the user before any
   re-dispatch: one `AskUserQuestion` call per item (max 4 questions per
   call, so split an item's questions across calls; option labels are 1 to
   5 words, put the agent's long option text in the description). Then
   re-dispatch a fresh
   agent per item with the same template plus the previous agent's `NOTES`
   and the user's answers (subagents cannot be resumed here). Repeat until
   every item has a `DRAFT`.
4. **Preview gate (user).** Show each `DRAFT` (title, labels, body) to the
   user. Wait for approval or edits. Edits go back the same way: fresh
   dispatch with `NOTES` + edits. This stop happens on every issue, however
   small.
5. **File (agent).** On approval, re-dispatch with the approved DRAFT and the
   word FILE. It runs
   `gh issue create --title … --label … --body-file <scratchpad path>` and
   reports the URL and number.
6. **Link (you).** If drafts referenced sibling items as `{{issue:KEY}}`,
   swap in the real numbers and `gh issue edit N --body-file`. Report all
   URLs.

## Subagent prompt template

```
You are drafting ONE GitHub issue in the repo at <cwd>. Read the repo's
agent instructions file (AGENTS.md / CLAUDE.md) first. `gh` is authenticated.

REQUEST: <the item text, plus any user context already known>
PRIOR NOTES: <NOTES block from the previous round, or "none">
ANSWERS / EDITS: <user's answers or edits from the previous round, or "none">
APPROVED DRAFT: <"none", or the word FILE followed by the verbatim approved
title / labels / body>
SIBLINGS in this batch (reference as {{issue:KEY}}, never guess numbers):
<KEY: one-line summary, or "none">

HARD RULES
- Do not write, edit, or create code, branches, commits, or PRs. Repo tree
  stays clean. A throwaway script under <scratchpad> to confirm a fact for
  the issue body is the only allowed code.
- Do not run `gh issue create/edit/delete` until told "FILE".
- Every ambiguity that would change what gets built is a question for the
  user, not an assumption. Return QUESTIONS before writing any body.

RESEARCH (light)
- Format: read the 5 most recent issues (`gh issue view`), match their body
  shape and tone. Labels: `gh label list`, pick from that list only.
- Duplicates/related: `gh issue list --state all --search "<keywords>"`.
  A likely duplicate is a QUESTION for the user, not a reason to stop.
- Codebase: locate the files, routes, tables, and existing patterns the
  issue touches. Record paths as pointers.
- Web search only if a standard, vendor API, or library behavior matters.

RETURN FORMAT: exactly one of these two blocks.

QUESTIONS
- <question>; options: <a> / <b> / <c>  (or "free text")
- ...
NOTES
<everything you found: file paths, related issue numbers, label picks,
format observations. The next round starts from this, so be complete.>
(Include only questions whose answers change the issue. No draft yet.)

DRAFT
title: <imperative, specific, under 80 chars>
labels: <from gh label list>
body-file: <scratchpad path you wrote>
body:
<full markdown body, in the repo's prevailing shape. Required parts:
what and why (the problem or ask, as the user framed it, with their
answers folded in); pointers, a list of file paths/routes/tables found in
research; related issues by number, or "{{issue:KEY}}" for siblings;
open questions, only those the user chose to leave open; and the closing
line "No code was changed as part of filing this issue.">

The body states the problem and where it lives. It does not dictate the
implementation: no helper names, no test file names, no chosen design
unless the user chose it.

If ANSWERS / EDITS is present: fold them in and return DRAFT (re-verify
pointers against the code; do not re-ask answered questions).
If APPROVED DRAFT says FILE: write that body verbatim to the scratchpad, run
gh issue create --title --label --body-file, report URL and number.
```

## Red flags — STOP

- "I'll pick a sensible default and list it under assumptions"
- "The request is clear enough to skip the questions round"
- "I'll show the user the URL, the draft is fine"
- Body contains a `## Recommendation` naming functions or tests to write
- Inline `--body "…"` instead of `--body-file` (shell quoting has bitten before)
- One agent handling two list items "since they're related"

## Rationalizations

| Excuse | Reality |
|---|---|
| "Listing assumptions is as good as asking" | Assumptions get built. The user said: ask, don't guess. |
| "Preview slows a trivial issue down" | The preview is one message; a wrong issue is a wasted implementation. |
| "Proposing the design helps the implementer" | It anchors them to a guess made without the user. Pointers yes, design no. |
| "Duplicate found, so skip the issue" | Whether to add to the existing issue or file anew is the user's call. |

## When NOT to use

The user is asking a question about the codebase, or wants the change made
now (use task-to-pr). Editing or commenting on an existing issue is a direct
`gh` call, not this pipeline.
