---
name: comment-drift
description: Sweep the code for comments that no longer match the code they describe and report them; fixing is a separate decision
argument-hint: "[path...] [ref]"
disable-model-invocation: true
---

# Comment Drift

Code moves, prose stays, and the next reader trusts the prose. This skill sweeps every comment in scope with parallel sub-agents, verifies what they flag, and reports it. The report is the deliverable: which findings get fixed is the user's call, taken afterwards.

Arguments: `$ARGUMENTS` — optional, any order: paths (limit the sweep to those subtrees) and one git ref (limit it to files changed since `git merge-base <ref> HEAD`, committed or not). Without arguments the whole repository is in scope.

## 1. Inventory

Files in scope come from `git ls-files` (with a ref: `git diff --name-only <base>` plus `git ls-files --others --exclude-standard`), restricted to source code in the repository's languages. Generated, vendored and minified code is out: by path here (`__generated__`, `*.g.dart`, `*.freezed.dart`, `*.pb.go`, `*.min.*`, `vendor/`), by header later (a shard agent skips a file that announces itself as generated). Take line counts with `wc -l`.

Done when the list with line counts is on hand. An empty list stops here.

## 2. Shard

Cut the list into shards of about 5,000 lines along directory boundaries, so related files share a shard and cross-references resolve locally: two shards at least, ten at most, larger shards beyond that.

Done when every file sits in exactly one shard.

## 3. Sweep

For each shard, spawn one general-purpose sub-agent whose prompt is [`brief.md`](brief.md) with `{{FILES}}` replaced by the shard's file list. Launch them all in one message so they run in parallel; save each result as `shard-<n>.md` in the session scratchpad.

With a ref, spawn one more agent with the identifiers the diff removed or renamed (functions, components, props, files, flags, routes). It greps the comments of the whole repository for them and reports hits as `dangling` in the brief's finding format: the comments that go stale with a change sit mostly outside the changed files.

Done when every agent has returned and each shard result's header counts (files read, comments judged) cover the shard.

## 4. Verify

Every finding cites a comment line and a code line. Read both. The finding stays when the code says what the finding claims; it goes otherwise, as does a duplicate from another shard (delegate in batches when the list runs long). A finding where the comment looks right and the code wrong stays: that is a suspected bug and the most valuable kind of hit.

Done when every surviving finding has been read against its cited lines.

## 5. Report

Prose in the conversation's language, findings in the brief's format, renumbered D1…Dn: grouped by category in the order `contradiction`, `dangling`, `resolved`, sorted by path within a group. One header line above: scope, files read, comments judged, findings after verification, findings dropped by verification.

Close with one line naming the findings you would fix and the ones you would leave, then ask which to apply.

## After the report

When the user picks findings: a comment that restates the code is deleted; one carrying a reason the code cannot show is rewritten to what is true now; a suspected bug stays as it is and is named in the answer, the fix being a separate task. This pass touches comments only.
