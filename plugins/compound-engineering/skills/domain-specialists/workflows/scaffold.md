---
name: scaffold
description: Analyze repo structure and propose initial domain specialist definitions
---

# Scaffold Domain Specialists

Analyze a project's structure and propose initial domain specialist
definitions. Creates agent files and updates
`compound-engineering.local.md`.

## Workflow

### Step 1: Scan Project Structure

Identify domain boundaries by examining:

```bash
# Top-level source directories
ls -d src/*/ app/*/ lib/*/ 2>/dev/null

# Module-level directories (one level deeper)
ls -d src/*/*/ app/*/*/ 2>/dev/null

# Check for existing agents
ls .claude/agents/*.md 2>/dev/null
```

Also check for signals of domain structure:

- README sections describing architecture or modules
- CLAUDE.md references to domain areas
- Directory naming patterns (e.g., `billing/`, `auth/`, `graphql/`)
- Separate test directories mirroring source structure

### Step 2: Identify Domain Clusters

Group related directories into candidate domains. Look for:

- Directories that form a cohesive feature area (e.g.,
  `src/billing/`, `src/payments/`, `tests/billing/`)
- Shared naming conventions (e.g., everything with `graphql` in
  the path)
- Documentation that describes module boundaries
- Existing team or ownership structures (CODEOWNERS file)

### Step 3: Check Existing Agents

Read any existing `.claude/agents/*.md` files. For each:

- Note the agent name and described expertise
- Check if it already has frontmatter with `name` and `description`
- Identify which source directories it likely covers

### Step 4: Propose Specialists

For each identified domain cluster, propose a specialist definition:

```yaml
- agent: <domain>-specialist
  file_patterns: ["src/<domain>/**", "tests/<domain>/**"]
  labels: ["<domain>"]
  keywords: ["<key terms from the domain>"]
```

Present all proposals to the user:

```
Found 4 potential domain specialists:

1. billing-specialist
   Files: src/billing/**, src/payments/**, tests/billing/**
   Keywords: invoice, subscription, stripe, payment

2. graphql-specialist
   Files: src/graphql/**, **/schema.py
   Keywords: resolver, dataloader, mutation, subscription

3. auth-specialist
   Files: src/auth/**, src/permissions/**
   Keywords: authentication, authorization, oauth, jwt

4. search-specialist
   Files: src/search/**, src/indexing/**
   Keywords: elasticsearch, index, query, aggregation

Which would you like to create? (comma-separated numbers, or "all")
```

### Step 5: Create Agent Files

For each accepted specialist, create `.claude/agents/<name>.md` if
it doesn't already exist:

```markdown
---
name: <domain>-specialist
description: "<Domain> specialist for <project>. Expert in <key areas>."
model: inherit
---

# <Domain> Specialist

## Expertise

<Inferred from directory structure, README, and file contents>

## Key Files and Patterns

<List the main files and directories in this domain>

## Conventions

<Extracted from existing code patterns, CLAUDE.md, or style guides>

## Self-Review Checklist

- [ ] Changes follow domain conventions
- [ ] Tests cover the modified behavior
- [ ] No cross-domain dependencies introduced without justification
```

Read representative files from the domain to populate the expertise
and conventions sections with actual project patterns.

### Step 6: Update Configuration

Add all created specialists to `compound-engineering.local.md`
frontmatter under `domain_specialists`. If the file doesn't exist,
create it using the setup skill first.

Present the final configuration to the user for confirmation before
writing.
