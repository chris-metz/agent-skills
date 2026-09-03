# Brief: comment drift sweep

You audit the comments in one shard of a repository, read-only, with the repository checked out. Every comment in the files below gets a verdict: it matches the code, or it drifted. Report drift only.

## Files

{{FILES}}

## What to judge

A comment makes claims: what the code does, why, what it refers to, what remains to be done. Check each claim against the code: the function body; the callers when the comment names them; the referenced symbol when it names one (grep the repository); the file header, and skip the whole file when it announces itself as generated. In scope: doc comments above functions, components, types and modules; inline comments; JSX comments; TODO, FIXME and HACK notes.

Pass without a report: license headers, tool directives (`eslint-disable`, `@ts-ignore`, `prettier-ignore`, `go:generate` and kin), commented-out code, and comments that are true but redundant, vague or badly written. Style is a different job.

## Drift categories

- **contradiction**: the comment claims something the code contradicts: parameters, return value, behaviour, conditions, side effects, "only used by", "always", "never". A reader who trusted it would write wrong code.
- **dangling**: the comment names a function, component, prop, file, flag, route or config key that no longer exists or was renamed, or points somewhere ("see X", "handled in Y") where nothing of the sort remains.
- **resolved**: a TODO, FIXME, HACK or "temporary workaround" whose condition the code shows to be settled: the fix is in, the workaround's target is gone. A ticket number alone proves nothing either way; the code is the evidence.

Every finding cites the comment line and the code line that contradicts it. A hunch without a line stays out.

## Output

```
## Shard: <n> files read, <m> comments judged

### D1: <path>:<line>
- Category: contradiction | dangling | resolved
- Comment: "<verbatim, trimmed to the claim>"
- Code: <what the code actually does>, <path>:<line>
- Wrong side: comment | code | unclear
- Suggestion: delete | rewrite to "<text>" | check the code

"No findings." when nothing drifted.
```

Done when every file is read in full and every comment in it has a verdict; the header counts are the proof.
