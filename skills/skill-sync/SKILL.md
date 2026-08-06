---
name: skill-sync
description: Check every forked skill in this repo against its upstream source, report what changed there, and merge what's worth taking.
disable-model-invocation: true
---

# Skill Sync

This repo's skills are part home-grown, part forked from other people's repos and
then modified. `provenance/manifest.yaml` records which is which. This skill
answers one question: **has anything worth taking landed upstream since we last
looked?**

## The model

For every forked skill, `provenance/upstream/<name>/` holds a verbatim copy of the
upstream files as they were at `fork.commit`. That snapshot is the pivot of a
three-way comparison:

| comparison | meaning |
|---|---|
| `diff(snapshot, local)` | **our delta** — what we changed. Recomputed every run, so it can never go stale. |
| `diff(snapshot, upstream@HEAD)` | **their delta** — what's on offer. |
| the two overlapping | **conflict** — a human decision, not a merge. |

Never diff local against upstream-HEAD directly. That conflates our changes with
theirs and will attribute their edits to us.

## Run the check

1. **Read `provenance/manifest.yaml`.** Skills with `origin: own` are skipped
   entirely — they have no upstream.

   `yq` is available for querying and for writing edits back. Keep `date`
   values quoted — unquoted they parse as `!!timestamp`, not strings, and
   string operations on them fail.

2. **Refresh the source clones.** Cache them under `provenance/.cache/<source>`
   (gitignored):

   ```bash
   git -C provenance/.cache/<source> fetch --quiet origin || \
     git clone --quiet <sources.<source>.clone> provenance/.cache/<source>
   ```

   Use a real clone, not raw-URL fetches — the history is what lets you resolve
   renames and find fork points.

3. **Resolve today's upstream path per file.** Upstream repos restructure;
   `mattpocock/skills` moves skills between buckets. Do not trust the `upstream:`
   path in the manifest as current — verify it, and follow the file if it moved:

   ```bash
   git -C provenance/.cache/<source> log --follow --format='%H %ci %s' -- <upstream-path>
   ```

   If the path is gone at HEAD, the skill was renamed, retired, or absorbed into
   another one. Find where it went before reporting it as deleted.

4. **Compute both deltas** per file, against the snapshot. Note that a file may
   carry its own `fork_commit` override — the snapshots aren't all from the same
   upstream commit.

5. **Classify each skill:**

   - **unchanged** — upstream identical to snapshot. Say nothing beyond a tick.
   - **clean** — upstream changed only lines our delta doesn't touch. Mergeable.
   - **conflict** — both deltas touch the same region. Needs a decision.
   - **restructured** — the skill was split, renamed, absorbed, or retired
     upstream. Never a merge; always a decision.

6. **Filter out noise.** Pure reformatting (emphasis markers, line rewrapping,
   changeset bookkeeping, version bumps) is not an update. Mention it once as
   noise, don't itemise it.

7. **Check `reviews`** in the manifest. If an upstream change was already
   surfaced and consciously declined in an earlier run, mark it as such rather
   than re-pitching it from scratch.

## Report

Lead with a table: one row per forked skill, its classification, and a one-line
gist of what changed upstream. Then go skill by skill through everything that
isn't `unchanged` — show the actual upstream diff, say how it interacts with our
delta, and give a recommendation. One skill at a time; wait for the decision
before moving to the next.

Do not apply anything unprompted.

## After a decision

**Taken** — apply the change, re-resolve our local delta onto the new upstream
text, then:

- overwrite `provenance/upstream/<name>/…` with the new upstream files
- bump `fork.commit` and `fork.date`
- update the `upstream:` path if it moved
- rewrite `local_changes` to describe the delta as it now stands
- clear any `upstream_status` / `decision_pending` the change resolved

**Declined** — change nothing but the manifest. Append to `reviews` what was
offered, what was declined, and why, so the next run doesn't re-litigate it.

Either way, append a `reviews` entry recording the date and each source's HEAD
at the time of the run.

## Adding a new forked skill

Snapshot it at the exact commit it was taken from — not upstream HEAD, unless
those are genuinely the same. If the fork point is unknown, find it: diff the
local file against each historical upstream version of that path and take the
closest match. The commit date in our git log is a hint, not evidence; a skill
can be committed here weeks after it was copied.
