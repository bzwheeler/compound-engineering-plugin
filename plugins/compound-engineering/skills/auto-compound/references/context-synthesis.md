# Context Synthesis Reference

How to build a structured problem narrative from PR artifacts.

## Source Priority

For each field, extract from the highest-priority source available:

### Module Name
1. **File paths** (best): Find the common directory prefix of changed
   files. Example: if all changed files are under `src/billing/`,
   the module is "Billing".
2. **PR labels**: Labels like `area/billing` or `module:auth`.
3. **PR title**: Extract the domain noun (e.g., "Fix invoice
   calculation" -> "Invoice" or "Billing").

### Symptoms / Problem Description
1. **Linked issue body** (`closingIssuesReferences`): Issues often
   contain the clearest problem description with error messages.
2. **PR body**: Look for sections like "Problem", "Bug", "Issue",
   "Before this change", or text following "Fixes #", "Resolves #".
3. **PR title**: Often a compressed symptom description.
4. **Review comments**: Reviewers sometimes restate the problem
   more clearly than the PR author.

### Investigation / What Didn't Work
1. **Review comment threads**: Multi-reply threads often contain
   "I tried X but it didn't work because..." or "The first approach
   was Y but we went with Z instead".
2. **PR body**: Look for "Tried", "Attempted", "Previous approach",
   "Alternative considered".
3. **Commit messages**: Sequential commits sometimes show iteration
   (e.g., "try fix with X", "actually fix with Y").

If no investigation is found, use: "Direct solution: The problem was
identified and fixed on the first attempt."

### Root Cause
1. **Review comments with corrections**: Comments that say "the
   actual issue is...", "the root cause was...", "this happened
   because...".
2. **PR body**: Look for "Root cause", "The issue was", "This was
   caused by", "The problem occurred because".
3. **Diff context**: What was changed often implies the root cause
   (e.g., adding an index implies `missing_index`, adding
   `.includes()` implies `missing_include`).

### Solution
1. **PR body**: The "Solution" or "Changes" or "What this PR does"
   section.
2. **Diff summary**: `additions`, `deletions`, `files` fields give
   scope. File names indicate what was changed.
3. **Commit messages**: Often describe each step of the fix.

### Prevention
1. **Review comments**: Look for "should always", "next time",
   "going forward", "to prevent this", "we should".
2. **PR body**: "Prevention" or "Follow-up" sections.
3. If nothing found, omit this section or write a generic prevention
   note based on the root cause category.

## Minimum Context Thresholds

| Signal | Threshold | If missing |
|--------|-----------|------------|
| PR body | > 50 characters, non-template | Weak context |
| Review comments | >= 1 comment > 20 chars, not "LGTM" | No reviewer signal |
| Linked issue | Has body with > 30 chars | No problem description |
| Changed files | >= 1 file | Invalid PR |

A PR needs at least ONE signal above threshold to proceed.

## Problem Detection Heuristics

### Strong problem signals (any one is sufficient)
- Label matches: `bug`, `fix`, `hotfix`, `incident`, `patch`,
  `regression`, `revert`, `security`
- Title starts with: "Fix", "Bug", "Hotfix", "Revert", "Patch"
- Body contains linked issue that describes an error or failure
- Body contains error messages or stack traces

### Weak problem signals (need 2+ to classify as problem)
- Title contains: "fix", "resolve", "correct", "handle", "prevent"
- Body mentions: "broken", "crash", "error", "fail", "wrong"
- Review comments discuss what was "broken" or "incorrect"
- Small diff (< 50 lines) suggesting a targeted fix

### Non-problem signals (skip unless overridden)
- Labels: `feature`, `enhancement`, `refactor`, `docs`, `chore`,
  `dependency`, `ci`
- Title starts with: "Add", "Implement", "Update", "Refactor",
  "Bump", "Upgrade", "Remove", "Rename"
- All changed files are in `docs/`, `test/`, `.github/`, or config
- PR body describes new functionality rather than fixing existing
