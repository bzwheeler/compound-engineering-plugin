---
name: auto-compound
description: >
  Batch-extract knowledge from historical merged PRs into structured
  docs/solutions/ entries. This skill should be used when a team wants
  to retroactively build a knowledge base from past pull requests.
disable-model-invocation: true
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
preconditions:
  - gh CLI installed and authenticated
  - Current directory is a git repo hosted on GitHub
---

# Auto-Compound: Extract Knowledge from Historical PRs

Build a searchable knowledge base from merged pull requests. Each
qualifying PR becomes a structured `docs/solutions/` entry with YAML
frontmatter, following the same format as `/workflows:compound`.

## Prerequisites Check

Before starting, verify the environment:

```bash
# Verify gh is authenticated
gh auth status

# Get repo owner/name
gh repo view --json nameWithOwner --jq '.nameWithOwner'

# Check API rate limit budget
gh api rate_limit --jq '.rate | "Budget: \(.remaining)/\(.limit) requests, resets at \(.reset | strftime("%H:%M:%S"))"'
```

If `gh auth status` fails, stop and tell the user:
"gh CLI must be authenticated. Run `gh auth login` first."

---

## Phase 1: Enumerate & Fetch PRs

### 1a. List merged PRs matching filters

Build the `gh pr list` command from the parsed arguments:

```bash
gh pr list --state all \
  --search "is:merged merged:${SINCE}..${UNTIL}" \
  --limit ${LIMIT} \
  --json number,title,mergedAt,labels,author \
  --jq 'sort_by(.mergedAt) | reverse'
```

If `--label` was provided, add it to the search string:
`--search "is:merged merged:${SINCE}..${UNTIL} label:${LABEL}"`

If `--author` was provided, add:
`--search "is:merged merged:${SINCE}..${UNTIL} author:${AUTHOR}"`

Display the count: "Found N merged PRs matching filters."

If 0 PRs found, stop with: "No merged PRs found for the given filters."

---

### Phase 1.5: Triage

Classify each PR as **SKIP**, **STANDARD**, or **ENRICH** before fetching
full PR details. This prevents expensive API calls and synthesis work for
PRs whose topics are already documented.

**Step 1: Keyword extraction (free)**

For each PR, extract 3–5 keywords from title and labels:
- Convert to lowercase
- Remove stopwords (the, a, an, fix, update, add, remove, change, etc.)
- Prefer nouns and technical terms
- Example: "Fix N+1 query in billing invoices" → `billing`, `n+1`, `query`,
  `invoices`

**Step 2: Parallel grep of existing knowledge bases (free)**

For each keyword, run in parallel:

```bash
grep -rl "<keyword>" docs/solutions/ 2>/dev/null | head -5
grep -rl "<keyword>" docs/institutional/ 2>/dev/null | head -5
```

If neither directory exists (first run), skip grep — all PRs default to
STANDARD.

Record filenames (not content) for use in Step 4.

**Step 3: Lightweight metadata fetch (1 API call per PR)**

Fetch only classification-relevant fields — NOT the expensive full fetch:

```bash
gh pr view ${PR_NUM} --json \
  title,body,additions,deletions,comments,closingIssuesReferences,labels
```

Extract:
- `body` (first 300 characters only)
- `additions + deletions` (total diff size)
- `len(comments)` (review comment count)
- `len(closingIssuesReferences) > 0` (has linked issues)

**Step 4: Haiku batch classification**

Batch 5–10 PRs per Task call using `model: haiku`. For each PR, pass:
- Title, labels, body snippet (≤300 chars)
- Diff stats (additions + deletions total)
- Comment count
- Grep match results (filenames only — not content)

Haiku prompt:

```
You are classifying merged PRs for knowledge extraction. For each PR,
output JSON.

SKIP: The topic is clearly already documented in existing solution files.
  - Requirement: grep matches exist AND topic overlap is high confidence (>0.85)
  - Default to STANDARD if uncertain — prefer to process than to miss

ENRICH: Topic is novel AND complex enough to warrant institutional research.
  - Signals: diff >200 lines, >5 review comments, has linked issues,
    or label contains bug/incident/breaking-change/security
  - ENRICH should be <20% of batch — reserve for genuinely high-value PRs

STANDARD: Everything else — novel topic, normal complexity, process normally.

For each PR, output one JSON object per line:
{"pr": <number>, "tier": "SKIP|STANDARD|ENRICH", "confidence": 0.0-1.0, "reason": "<15 words>"}
```

**Error handling:** If the Haiku classification call fails for a batch,
default all PRs in that batch to STANDARD. Never SKIP on classification
error — prefer false negatives to missed docs.

**Step 5: Record tier in checkpoint**

Add a `tier` field to each PR entry in `auto-compound-progress.json`
(see Phase 1d). Preserve tier decisions so resuming a run does not
re-classify already-triaged PRs.

Display triage summary: "Triage: N SKIP, N STANDARD, N ENRICH"

---

### 1b. Fetch rich context per PR

Process only **STANDARD** and **ENRICH** PRs. Skip PRs classified as SKIP
entirely — record them as `triage_skip` in the progress file (no further
API calls or synthesis).

For each STANDARD or ENRICH PR, fetch two data sources:

**PR metadata** (title, body, files, reviews, linked issues):
```bash
gh pr view ${PR_NUM} --json \
  title,body,labels,files,reviews,comments,commits,\
  closingIssuesReferences,additions,deletions,mergedAt,mergedBy
```

**Inline review comments** (code-level feedback -- richest signal):
```bash
gh api repos/${OWNER}/${REPO}/pulls/${PR_NUM}/comments \
  --paginate --jq '.[].body'
```

### 1c. Rate limit management

- Process PRs in batches of 10
- After each batch, check remaining budget:
  ```bash
  gh api rate_limit --jq '.rate.remaining'
  ```
- If remaining < 100, display:
  "Rate limit low (N remaining). Pausing until reset."
  Then wait for the reset time before continuing.
- Display progress: "Processing PR N/TOTAL..."

### 1d. Checkpoint

After each batch of 10 PRs, write a progress file:

```bash
# Write to .claude/auto-compound-progress.json
{
  "repo": "owner/repo",
  "started_at": "2026-01-15T10:00:00Z",
  "filters": { "since": "...", "until": "...", "label": "...", "limit": 50 },
  "processed": [
    { "pr": 1234, "tier": "STANDARD", "outcome": "created", "file": "docs/solutions/..." },
    { "pr": 1235, "tier": "ENRICH", "outcome": "created", "file": "docs/solutions/..." },
    { "pr": 1236, "tier": "SKIP", "outcome": "skipped", "reason": "triage_skip" },
    { "pr": 1237, "tier": "STANDARD", "outcome": "skipped", "reason": "insufficient_context" }
  ],
  "remaining": [1240, 1241, 1242]
}
```

Existing checkpoints without a `tier` field should be treated as STANDARD
for all unprocessed PRs (backward compatibility).

If a progress file already exists for this repo + filter combination,
ask the user: "Found progress from a previous run (N PRs already
processed). Resume from where it left off?"

---

## Phase 2: Context Synthesis

For each fetched PR (STANDARD and ENRICH only), determine if it contains
a documentable problem and synthesize the context. See
[context-synthesis.md](./references/context-synthesis.md) for detailed
extraction rules.

ENRICH PRs proceed through steps 2a–2c normally, then continue to steps
2d–2f for archeologist enrichment before generating the final document.

### 2a. Minimum viable context check

A PR must have at least ONE of:
- A body with > 50 characters describing a problem or fix
- At least one non-trivial review comment (> 20 chars, not just "LGTM"
  or approval text)
- A linked issue (`closingIssuesReferences`) with a description

PRs failing this check are recorded as `skipped: insufficient_context`.

### 2b. Problem detection

Determine if the PR addresses a **problem** (bug fix, performance fix,
incident response) vs. a **non-problem** (feature addition, refactor,
dependency update, docs-only change).

Signals that indicate a problem-solving PR:
- Labels containing: bug, fix, hotfix, incident, patch, regression
- Title/body containing: fix, bug, broken, crash, error, fail, issue,
  regression, revert, patch, resolve
- Linked issues that describe error symptoms
- Review comments pointing out what was wrong

PRs classified as non-problems are recorded as
`skipped: non_problem_pr` with the detected type (feature, refactor,
dependency, docs).

### 2c. Synthesize structured context

For qualifying PRs, extract these fields from the PR artifacts:

| Field | How to extract |
|-------|---------------|
| **Module** | Common directory prefix of changed files (e.g., `src/billing/` -> "Billing") |
| **Symptoms** | Error messages or behavior described in PR body or linked issue |
| **Root cause** | Corrections described in review comments, or explanation in PR body |
| **Solution** | Summary of what the diff changed + PR description of the fix |
| **Prevention** | Review comments containing "should always", "next time", "going forward" |
| **Investigation** | Failed attempts mentioned in PR body or review thread discussions |

### 2d. Archeologist enrichment (ENRICH tier only)

For PRs classified as ENRICH, invoke the `institutional-archeologist`
agent after step 2c synthesis is complete. Pass:
- The extracted module and symptom keywords for this PR
- Any `closingIssuesReferences` issue numbers

The archeologist will:
1. Check `docs/institutional/` for existing research (gap-fill if found)
2. Search GitHub Issues and PRs for prior discussions
3. Search Slack/Notion if configured in `compound-engineering.local.md`

**Error handling:** If the archeologist fails (timeout, API error, empty
findings), record `enrich_partial` in the checkpoint for this PR and
continue with the STANDARD synthesis output. Do not fail the whole batch.

If `--dry-run` mode is active, skip archeologist invocation entirely —
show a note "Would invoke institutional-archeologist for ENRICH PR #N"
in the preview output.

### 2e. Fold archeologist findings into synthesis

Merge the archeologist's output into the already-extracted synthesis fields:

| Synthesis field | How to enrich |
|----------------|---------------|
| **Root cause** | Deepen with "prior discussions found X" if archeologist surfaced relevant context |
| **Prevention** | Enrich with "team noted going forward..." from prior decisions in institutional docs or Slack threads |
| **Investigation** (What Didn't Work) | Add prior failed approaches found in GitHub issues or Slack if they match this PR's problem domain |
| **Solution** | Note if the approach aligns with or departs from prior team decisions |

Only incorporate findings that are genuinely relevant to this specific PR.
Do not pad the document with tangentially related history.

### 2f. Invoke archivist to persist findings

Call `institutional-archivist` with the archeologist's full findings
output, the planning topic keywords for this PR, and the list of sources
searched.

The archivist will write a structured digest to `docs/institutional/`.
Future auto-compound runs will find these files during Phase 1.5 Step 2
(grep), compounding the institutional value forward.

**Error handling:** If the archivist fails, log the error and continue.
Archivist failure does not block document generation.

If `--dry-run` mode is active, skip archivist invocation entirely.

---

## Phase 3: YAML Classification

For each synthesized context, classify into the compound-docs YAML
schema. See [classification.md](./references/classification.md) for
the full enum lists and mapping heuristics.

### 3a. Generate frontmatter

Produce YAML frontmatter matching this structure:

```yaml
---
module: [extracted module name]
date: [PR merge date, YYYY-MM-DD]
problem_type: [enum from classification.md]
component: [enum from classification.md]
symptoms:
  - [symptom 1]
  - [symptom 2]
root_cause: [enum from classification.md]
resolution_type: [enum from classification.md]
severity: [enum from classification.md]
source_pr: [PR number]
tags: [keyword1, keyword2]
---
```

The `source_pr` field is new -- it links the doc back to its origin PR.

### 3b. Validate enum values

Every enum field MUST match one of the allowed values exactly
(case-sensitive). If a field cannot be confidently classified:

- In `--dry-run` mode: mark the field as "uncertain" in the preview
- In normal mode: use the closest match and add a tag
  `needs-review-classification`

### 3c. Confidence assessment

For each PR, assess overall classification confidence:
- **High**: PR body clearly describes a bug fix with symptoms and
  solution; review comments confirm the approach
- **Medium**: PR body mentions a fix but details are sparse; review
  comments provide some context
- **Low**: PR body is vague; classification relies heavily on file
  paths and labels

PRs with low confidence are still processed but tagged
`needs-review-classification` in the frontmatter tags.

---

## Phase 4: Deduplication

Before writing any doc, check for duplicates in existing
`docs/solutions/` entries. This phase remains as a safety net for Phase
1.5 triage false negatives (STANDARD PRs that overlap with existing docs).
SKIP PRs never reach this phase.

### 4a. Search existing docs

```bash
# Find all existing solution docs
find docs/solutions/ -name "*.md" -type f 2>/dev/null
```

If no `docs/solutions/` directory exists, skip dedup entirely (first
run -- all PRs are new).

### 4b. Frontmatter match (exact)

For each existing doc, extract its YAML frontmatter and compare:
- If `module` + `problem_type` + `root_cause` all match the new
  PR's classification = **duplicate**. Skip this PR.
- If `source_pr` field matches = **already processed**. Skip.

### 4c. Keyword overlap (fuzzy)

Extract keywords from the new PR's symptoms and tags. Search existing
docs for overlapping terms:

```bash
# Search for symptom keywords in existing docs
grep -rl "keyword" docs/solutions/
```

If multiple existing docs share > 3 keywords with the new PR and are
in the same module, flag as potential duplicate. In normal mode, skip
and note "possible duplicate of [existing-file]". The user can review
in the summary report.

---

## Phase 5: Document Generation

For each PR that passes dedup checks, generate a solution document.

### 5a. Determine file path

Map `problem_type` to directory:

| problem_type | Directory |
|-------------|-----------|
| build_error | `docs/solutions/build-errors/` |
| test_failure | `docs/solutions/test-failures/` |
| runtime_error | `docs/solutions/runtime-errors/` |
| performance_issue | `docs/solutions/performance-issues/` |
| database_issue | `docs/solutions/database-issues/` |
| security_issue | `docs/solutions/security-issues/` |
| ui_bug | `docs/solutions/ui-bugs/` |
| integration_issue | `docs/solutions/integration-issues/` |
| logic_error | `docs/solutions/logic-errors/` |
| developer_experience | `docs/solutions/developer-experience/` |
| workflow_issue | `docs/solutions/workflow-issues/` |
| best_practice | `docs/solutions/best-practices/` |
| documentation_gap | `docs/solutions/documentation-gaps/` |

Generate filename: kebab-case from module + key symptom.
Example: `billing-n-plus-one-query.md`

### 5b. Fill the resolution template

Use the compound-docs resolution template format:

```markdown
---
[YAML frontmatter from Phase 3, including source_pr field]
---

# Troubleshooting: [Clear Problem Title]

## Problem
[1-2 sentence description from PR body/linked issue]

## Environment
- Module: [module name]
- Affected Component: [component description]
- Date: [merge date]

## Symptoms
- [symptom 1]
- [symptom 2]

## What Didn't Work
[From review comment discussions about alternative approaches,
or "Direct solution: identified and fixed on first attempt."]

## Solution
[Summary of the actual fix from the diff + PR description]

## Why This Works
[Root cause explanation from review comments or PR body]

## Prevention
[Prevention guidance from review comments, or general best practice]

## Related Issues
- Source PR: #[PR number]
[Cross-references from Phase 4 if any related docs found]
```

### 5c. Dry-run vs. write

If `--dry-run`:
- Display the proposed file path and YAML frontmatter for each PR
- Do NOT create any files or directories
- Show totals at the end: "Would create N docs, skip M PRs"

If normal mode:
- Create directory: `mkdir -p docs/solutions/[category]/`
- Write the file
- Display: "Created: docs/solutions/[category]/[filename].md (from PR #N)"

---

## Phase 6: Summary Report

After all PRs are processed, produce a summary report.

### 6a. Display summary

```
## Auto-Compound Summary

**Repo**: [owner/repo]
**Date range**: [since] to [until]
**PRs found**: [total]
**PRs processed**: [processed count]

### Triage Results
- SKIP (triage): N  — already documented, skipped before full fetch
- STANDARD: N       — processed normally
- ENRICH: N         — processed with institutional archeologist

### Processing Results
- Created: N new solution docs
- Created (with enrichment): N new solution docs (ENRICH tier)
- Skipped (duplicate, Phase 4): N
- Skipped (insufficient context): N
- Skipped (non-problem PR): N
- Failed (classification): N
- Failed (enrich_partial): N  — enrichment failed, STANDARD synthesis used

### New Documents Created
| PR | Title | Tier | Category | File |
|----|-------|------|----------|------|
| #NNNN | [title] | STANDARD | [category] | [file path] |
| #NNNN | [title] | ENRICH | [category] | [file path] |

### Needs Manual Review
| PR | Title | Reason |
|----|-------|--------|
| #NNNN | [title] | [reason] |

### Suggested Next Steps
- Review generated docs for accuracy
- git add docs/solutions/ && git commit -m "Add auto-compounded solution docs"
```

### 6b. Write report file

Write the summary to `docs/auto-compound-report-{YYYY-MM-DD}.md`.

### 6c. Clean up checkpoint

If all PRs were processed successfully, remove the progress file:
`.claude/auto-compound-progress.json`

---

## Error Handling

- **gh command fails**: Display the error, skip that PR, continue
  with the next one. Record as `failed: api_error` in progress.
- **Triage classification fails**: Default the entire affected batch to
  STANDARD. Never SKIP on classification error — prefer false negatives
  to missed docs.
- **Classification fails**: Record as `skipped: classification_failed`.
  Include in "Needs Manual Review" section of summary.
- **Schema validation fails**: Do not write the file. Record as
  `failed: schema_validation`. Include the invalid field in summary.
- **Disk write fails**: Stop the batch and preserve the checkpoint
  file so the user can resume later.
- **Archeologist fails (ENRICH tier)**: Fall back to STANDARD synthesis
  output for that PR. Record `enrich_partial` in the checkpoint. Do not
  fail the batch.
- **Archivist fails**: Log the error and continue. Archivist failure
  does not block document generation or the batch.
