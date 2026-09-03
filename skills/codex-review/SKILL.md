---
name: codex-review
description: Second-opinion code review by Codex (a different model family) with Claude's assessment underneath; nothing gets implemented
argument-hint: "[ref] [#issue]"
allowed-tools: Bash(codex login status), Bash(codex exec:*)
disable-model-invocation: true
---

# Codex Review

A second opinion from a different model family. Codex reviews the change, this skill shows its findings verbatim and adds an assessment per finding. The report is the deliverable: which findings get fixed is the user's call, taken afterwards.

Arguments: `$ARGUMENTS` — up to two optional tokens in any order: a git ref (the fixed point) and an issue reference (`#123`, `123`, or an issue URL).

## 1. Preflight

- `codex login status` succeeds. Otherwise stop and report; Codex is the point of this skill, a Claude sub-agent is no substitute.
- Fix the base: with a ref, `git merge-base <ref> HEAD` (the ref must `git rev-parse`); without, `HEAD`.
- Scope is everything since the base, committed or not: `git diff <base>`, the untracked files from `git ls-files --others --exclude-standard`, and the commit list `git log <base>..HEAD --oneline`.
- Done when the scope is non-empty. An empty scope stops here.

## 2. Fetch the issue

Codex runs read-only without network, so the issue text travels inside the brief.

- Issue given: fetch it via `docs/agents/issue-tracker.md` if the repo has one ("fetch the relevant ticket"), otherwise `gh issue view <n> --comments`.
- No issue given: scan the commit messages in scope and the branch name for issue references. Exactly one hit: use it and say so. Otherwise ask the user which issue applies; "none" is a valid answer.
- Done when the issue's title, body and decision-carrying comments are on hand, or the user has said there is none.

## 3. Write the brief

Write the brief to `codex-brief.md` in the session scratchpad (quoting stays out of the shell), in the conversation's language. Sections, in order:

1. **Role**: independent reviewer of a change another agent wrote. Read-only; the repo is checked out. Run the git commands yourself and read whatever surrounding code the judgement needs.
2. **Scope**: the three commands from step 1, verbatim.
3. **Spec** (only with an issue): the issue text in a fenced block. Task: check the change against it — requirements missing or partial, behaviour that deviates, behaviour the issue never asked for.
4. **Focus**: defects the author would fix. Correctness, edge cases, error handling, data integrity, concurrency, security, breaking changes for existing callers, missing or weak tests. Out of scope: style, naming, formatting, anything a linter enforces.
5. **Output format**, verbatim:

   ```
   ## Summary
   Two to four sentences.

   ## Findings
   Ordered by severity. Per finding:
   ### F<n>: <title>
   - File: <path>:<line>
   - Severity: high | medium | low
   - What: ...
   - Why it matters: ...
   - Suggested fix: ...
   Write "No findings." when there are none.

   ## Spec check
   (only with an issue) Per requirement: met | missing | deviates, quoting the issue line.
   ```

## 4. Run Codex

```
codex exec --ephemeral -s read-only -o "<scratchpad>/codex-review.md" - < "<scratchpad>/codex-brief.md" > "<scratchpad>/codex-review.log" 2>&1
```

Run it in the background and wait: a large diff at high reasoning effort takes minutes, beyond the foreground limit. The assessment in step 6 starts from Codex's findings, so the wait is idle. Model, reasoning effort and service tier come from `~/.codex/config.toml`; the command carries no model flags.

Done when the process has exited and `codex-review.md` is non-empty. On a non-zero exit or an empty file: show the last lines of the log and stop.

## 5. Present the findings

Under a heading `Codex findings`: the output file verbatim — same order, same wording, nothing dropped, nothing merged. One line of provenance above it, taken from the log header: model, reasoning effort, scope, duration.

## 6. Assess

Under a heading `Assessment`, per finding F<n>, at most five lines:

1. **Steelman**: the strongest case for the finding, after reading the referenced code. You most likely wrote this code and know why it looks the way it does; that context helps the reader and it makes you defend, which is why the steelman comes first.
2. **Verdict**: agree, partly or disagree, with the reason in one sentence.
3. **Would change**: yes or no, and in one sentence what the change would be.

Close with one line naming the findings you would change and the ones you would leave, then ask which ones to implement.

## 7. Stop

The report is the deliverable and the user's decision is the next instruction. Edits, commits and test runs wait for it.

## Why

- **A different model family.** A reviewer from the author's family shares the author's blind spots and prefers the author's choices. Codex does not.
- **Verbatim.** Rewording or reranking by the author is where findings get buried. Codex speaks in its own voice; the assessment is a separate section.
- **Steelman first.** The assessment comes from the author. Arguing for the finding before against it keeps the second opinion from being talked away.
- **Nothing implemented.** The user decides with both views on the table.
