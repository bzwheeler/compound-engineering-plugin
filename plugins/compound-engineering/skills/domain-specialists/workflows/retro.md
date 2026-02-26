---
name: retro
description: Feed PR review feedback back into domain specialist agent files
---

# Retro: Specialist Learning Loop

Read PR review comments, categorize the feedback, and propose updates
to relevant domain specialist agent files. This is the learning loop
that makes specialists improve over time.

## Workflow

### Step 1: Gather Review Feedback

```bash
# Get PR review comments
gh pr view <number> --json reviews,comments,reviewRequests \
  --jq '.reviews[].body, .comments[].body'

# Get inline review comments
gh api repos/{owner}/{repo}/pulls/<number>/comments \
  --jq '.[] | {path: .path, body: .body, diff_hunk: .diff_hunk}'

# Get changed files for specialist routing
gh pr view <number> --json files --jq '[.files[].path] | join(",")'
```

### Step 2: Match Feedback to Specialists

1. Read `compound-engineering.local.md` for `domain_specialists`
2. Get the list of changed files from the PR
3. Route feedback to matching specialists using the standard
   routing algorithm (file patterns, labels, keywords)
4. Group feedback by specialist

If no specialists match, suggest creating one based on the files
touched.

### Step 3: Categorize Feedback

For each piece of feedback, classify into:

| Category | Description | Action |
|----------|-------------|--------|
| Domain Knowledge | Reviewer shares domain-specific insight | Add to specialist's Expertise section |
| Common Pitfall | Reviewer catches a known gotcha | Add to specialist's Self-Review Checklist |
| Convention Violation | Code doesn't follow project patterns | Add to specialist's Conventions section |
| Style Issue | Formatting, naming, or structural feedback | Note but don't update specialist |
| Subjective | Reviewer preference, not a rule | Skip |

Focus on **domain knowledge**, **common pitfalls**, and **convention
violations** — these are the categories that compound specialist
expertise.

### Step 4: Propose Updates

For each specialist with actionable feedback:

1. Read the specialist's agent file
   (`.claude/agents/<specialist>.md`)
2. Identify where the learning fits:
   - New rule in **Conventions** section
   - New item in **Self-Review Checklist**
   - New pattern or anti-pattern in **Expertise** section
   - New entry in a **Known Gotchas** section
3. Draft the specific edit (addition, not replacement)

Present each proposed update one at a time:

```
## Update: graphql-specialist

Category: Common Pitfall
From: PR #1234 review comment by @reviewer

Learning: "Always use dataloaders for nested field resolution
to avoid N+1 queries. The `@strawberry.field` decorator alone
doesn't batch."

Proposed addition to Self-Review Checklist:
  - [ ] Nested field resolvers use dataloaders (not direct queries)

Apply this update? [y/n]
```

### Step 5: Apply Updates

For each approved update:

1. Edit the specialist's agent file with the new content
2. Keep updates minimal — add one rule or checklist item per
   learning, not paragraphs

After all updates are applied, summarize:

```
Retro complete for PR #1234:

  graphql-specialist: +2 checklist items, +1 convention
  database-specialist: +1 gotcha

Specialists updated. These learnings will improve future
implementations in these domains.
```

### Step 6: Suggest Compound (Optional)

If the PR involved a non-trivial fix or discovery:

```
This PR had significant learnings. Would you also like to run
/workflows:compound to document the solution for the whole team?
```
