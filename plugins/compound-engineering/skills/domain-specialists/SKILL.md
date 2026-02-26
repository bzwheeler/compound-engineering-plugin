---
name: domain-specialists
description: Route tasks to project-specific domain specialist agents based on file patterns, labels, and keywords. Use when workflow commands need to detect and delegate to domain experts.
---

# Domain Specialists

Route tasks to project-specific domain specialist agents during the
plan/work/review cycle. Specialists are project-level or user-level
agent files (`.claude/agents/<name>.md`) with routing triggers
configured in `compound-engineering.local.md`.

## Configuration

Domain specialists are declared in the `compound-engineering.local.md`
YAML frontmatter alongside `review_agents`:

```yaml
---
review_agents: [security-sentinel, performance-oracle]
domain_specialists:
  - agent: billing-specialist
    file_patterns: ["src/billing/**", "src/payments/**"]
    labels: ["billing", "payments"]
    keywords: ["invoice", "subscription", "stripe"]
  - agent: graphql-specialist
    file_patterns: ["src/**/graphql/**", "**/*schema*.py"]
    labels: ["graphql", "schema"]
    keywords: ["resolver", "dataloader", "mutation"]
---
```

Each specialist entry has:

| Field | Required | Description |
|-------|----------|-------------|
| `agent` | Yes | Name of the agent file (resolves to `.claude/agents/<agent>.md`) |
| `file_patterns` | No | Glob patterns matching files in the specialist's domain |
| `labels` | No | Issue/PR labels that activate this specialist |
| `keywords` | No | Words in descriptions that trigger routing |

At least one trigger field (`file_patterns`, `labels`, or `keywords`)
must be present.

## Routing Algorithm

To detect which specialists match the current context:

1. Read `compound-engineering.local.md` frontmatter
2. If `domain_specialists` is missing or empty, skip routing
3. Gather context signals:
   - **Files**: from `git diff --name-only`, plan references, or
     PR changed files
   - **Labels**: from `gh pr view --json labels` or issue metadata
   - **Keywords**: from task description, plan title, or issue title
4. For each specialist in `domain_specialists`:
   - Check `file_patterns` against the file list (glob matching)
   - Check `labels` against current labels (case-insensitive)
   - Check `keywords` against the description text
     (case-insensitive substring match)
   - A specialist **activates** if ANY trigger type matches (OR logic)
5. Return all matching specialists, ordered by number of trigger
   matches (most specific first)

If no specialists match, proceed without specialist delegation.
Multiple specialists can match simultaneously.

## Invoking Specialists

Once detected, specialists are spawned via the Task tool using their
agent name as the `subagent_type`:

```
Task(
  subagent_type: "<specialist.agent>",
  description: "<specialist.agent> working on <task>",
  prompt: "<context and instructions>"
)
```

The specialist agent file at `.claude/agents/<agent>.md` provides the
agent's system prompt, domain knowledge, conventions, and checklist.

### For Implementation (in `/workflows:work`)

```
Task(
  subagent_type: "billing-specialist",
  description: "billing-specialist implements payment validation",
  prompt: "Implement the following task: <task details from plan>.
           Files involved: <file list>.
           Follow your domain conventions and checklist."
)
```

### For Review (in `/workflows:review`)

```
Task(
  subagent_type: "billing-specialist",
  description: "billing-specialist domain review",
  prompt: "Review this PR from your domain expertise perspective.
           Focus on domain-specific patterns, correctness, and
           potential issues. PR diff: <diff content>"
)
```

## Agent File Discovery

Specialists are resolved from multiple locations (highest priority
first):

1. **Project**: `.claude/agents/<agent>.md` in the repo
2. **User**: `~/.claude/agents/<agent>.md`
3. **Plugin**: agents bundled with installed plugins

This follows the "Grow Your Own Garden" principle — projects cultivate
their own specialists, users carry personal ones across projects, and
plugins provide generic starting points.

## Creating Specialists

Use the [scaffold workflow](./workflows/scaffold.md) to analyze a
repo and propose initial specialist definitions, or create agents
manually:

1. Create `.claude/agents/<name>.md` with domain expertise
2. Add an entry to `domain_specialists` in
   `compound-engineering.local.md`
3. Test routing: the specialist should activate when working on files
   in their domain

## The Learning Loop

After a PR gets human review, run `/workflows:retro <PR>` to feed
review feedback back into specialist agent files. See the
[retro workflow](./workflows/retro.md).

```
Work on issue → Specialist implements → PR reviewed by humans
     ↑                                          ↓
     └── Specialist gets smarter ←── /retro feeds back learnings
```

Each review cycle compounds the specialist's knowledge, making
future implementations better.
