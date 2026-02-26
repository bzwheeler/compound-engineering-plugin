---
name: workflows:retro
description: Feed PR review feedback back into domain specialist agent files to improve future work
argument-hint: "[PR number or URL]"
---

# Retro

Feed PR review feedback back into domain specialist agent files.
This is the learning loop that makes specialists improve over time.

## PR

<pr_target> #$ARGUMENTS </pr_target>

**If the PR target is empty, ask the user:** "Which PR would you like
to run a retro on? Provide a PR number or URL."

Do not proceed without a PR target.

## Execution

Load and follow the retro workflow from the `domain-specialists` skill:

1. Gather all review comments from the PR
2. Match feedback to domain specialists using routing config from
   `compound-engineering.local.md`
3. Categorize each comment (domain knowledge, common pitfall,
   convention violation, style, subjective)
4. Propose updates to relevant specialist agent files
5. Apply approved updates
6. Summarize changes

See `domain-specialists` skill [retro workflow](../skills/domain-specialists/workflows/retro.md) for the detailed process.

## When to Run

Run after a PR receives human review feedback:

```
/workflows:work → PR created → Human reviews → /workflows:retro
     ↑                                              ↓
     └── Specialist gets smarter ←──────────────────┘
```

## Related Commands

- `/workflows:compound` — Document a solved problem for the team
- `/workflows:review` — Run multi-agent code review
