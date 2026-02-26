---
name: workflows:auto-compound
description: Batch-extract knowledge from historical merged PRs into docs/solutions/
argument-hint: "[--since YYYY-MM-DD] [--until YYYY-MM-DD] [--label bug] [--author user] [--limit N] [--dry-run]"
---

# /workflows:auto-compound

Retroactively build a searchable knowledge base from merged pull requests.

Load the `auto-compound` skill and execute its 6-phase workflow.

## Arguments

Parse these from `$ARGUMENTS`:

| Flag | Default | Description |
|------|---------|-------------|
| `--since` | 90 days ago | Start of date range (YYYY-MM-DD) |
| `--until` | today | End of date range (YYYY-MM-DD) |
| `--label` | (none) | Filter PRs by label (comma-separated) |
| `--author` | (none) | Filter PRs by author |
| `--limit` | 50 | Maximum number of PRs to process |
| `--dry-run` | false | Preview what would be created without writing files |

<pr_filters> $ARGUMENTS </pr_filters>

## Execution

Follow the 6-phase workflow defined in the `auto-compound` skill:

1. **Enumerate** merged PRs matching filters
2. **Synthesize** problem context from each PR's artifacts
3. **Classify** into compound-docs YAML schema
4. **Deduplicate** against existing `docs/solutions/`
5. **Generate** solution docs for qualifying PRs
6. **Report** summary of everything processed

Start with Phase 1 now.
