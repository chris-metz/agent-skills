---
name: dead-code
description: Find dead code in the repository
disable-model-invocation: true
---

# Dead Code Sweep

- Spawn multiple agents to analyze the codebase in parallel
- Each agent identifies potential dead code candidates (unreferenced files, unused exports, unreachable branches, orphaned dependencies)
- Agents communicate and verify candidates across the entire repository to ensure accuracy
- The goal is to provide a comprehensive report of dead code that can be safely removed, improving code maintainability and reducing technical debt.
