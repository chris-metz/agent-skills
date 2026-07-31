---
name: abstraction-policy
description: Find duplicated code that could be generalized into a single shared abstraction
disable-model-invocation: true
---

# Abstraction Policy

- Spawn multiple agents to analyze the codebase in parallel
- Each agent finds repeated code (copy-paste blocks, near-identical functions differing only in constants or types, parallel branches of the same logic, re-implemented helpers)
- Agents communicate and verify candidates across the entire repository to confirm the duplicates really share one intent, not just a surface shape
- For each confirmed cluster, propose the abstraction: where the single definition should live, its signature, and every call site that would reference it
- Call out duplication that should stay duplicated — coincidental similarity, divergent lifecycles, or abstractions that would cost more than they save
- The goal is a policy the repo can follow: what to unify now, what to leave alone, and where new code should look before writing its own version.
