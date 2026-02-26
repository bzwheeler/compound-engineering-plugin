---
title: "feat: Auto-Compound Tiered Processing with Institutional Enrichment"
type: feat
status: completed
date: 2026-02-22
brainstorm: docs/brainstorms/2026-02-22-auto-compound-enrichment-brainstorm.md
---

# feat: Auto-Compound Tiered Processing with Institutional Enrichment

## Overview

Improve the `workflows:auto-compound` skill to minimize unnecessary token and API usage while maximizing knowledge capture quality. The optimization targets the most expensive phase — context synthesis (Phase 2) — by adding a lightweight triage step immediately after PR enumeration.

The core change: after listing PRs (cheap), classify each as **SKIP / STANDARD / ENRICH** before fetching full PR details or running synthesis. SKIP PRs are already documented — skip them entirely. ENRICH PRs are novel and complex — invoke the `institutional-archeologist` before synthesis to deepen the output. STANDARD PRs proceed as before.

## Problem Statement / Motivation

The current skill fetches full PR context and runs LLM synthesis for every PR, regardless of whether the topic is already documented. This wastes:
- **GitHub API calls**: Full PR fetch (reviews, files, comments) for PRs that will just be deduplicated in Phase 4
- **Synthesis tokens**: LLM extraction for topics that already exist in `docs/solutions/`
- **Missed enrichment**: High-value, complex PRs get no richer treatment than trivial ones

The deduplication in Phase 4 is a good safety net but arrives too late — the expensive work is already done.

## Proposed Solution

Insert **Phase 1.5: Triage** between existing Phase 1 (enumeration) and Phase 1b (full PR fetch). This restructures the workflow into:

1. **Phase 1a** — Enumerate PRs (title, labels, mergedAt) — unchanged
2. **Phase 1.5 (NEW)** — Triage: grep existing docs + lightweight metadata fetch + Haiku classification → SKIP / STANDARD / ENRICH
3. **Phase 1b (modified)** — Full context fetch **only for STANDARD and ENRICH PRs** (skip SKIP entirely)
4. **Phase 2 (modified)** — For ENRICH PRs, invoke archeologist after synthesis; invoke archivist to persist findings
5. **Phases 3–6** — Unchanged

## Technical Considerations

### Phase 1.5: Triage — Step by Step

**Step 1: Keyword extraction (free)**
For each PR, extract 3–5 keywords from title and labels (lowercase, stopwords removed). These are used for all subsequent checks.

**Step 2: Parallel grep of existing knowledge bases (free)**
```bash
# Run for each keyword, in parallel
grep -rl "<keyword>" docs/solutions/ 2>/dev/null | head -5
grep -rl "<keyword>" docs/institutional/ 2>/dev/null | head -5
```
If neither directory exists (first run), skip grep — all PRs default to STANDARD.

**Step 3: Lightweight metadata fetch (1 API call per PR)**
Fetch only what's needed for classification — NOT the expensive full fetch:
```bash
gh pr view ${PR_NUM} --json title,body,additions,deletions,comments,closingIssuesReferences,labels
```
Extract: `body` (first 300 chars), `additions + deletions`, `len(comments)`, has `closingIssuesReferences`.

**Step 4: Haiku batch classification**
Batch 5–10 PRs per Task call with `model: haiku`. Pass for each PR:
- Title, labels, body snippet (300 chars max)
- Diff stats (additions + deletions)
- Comment count
- Grep match results (filenames only — not content)

Request structured output per PR:
```json
{
  "pr": 1234,
  "tier": "SKIP|STANDARD|ENRICH",
  "confidence": 0.0-1.0,
  "reason": "one sentence"
}
```

**Tier definitions (for Haiku prompt):**
- **SKIP**: Grep found matches AND title/description topic clearly overlaps with an existing solution doc. High confidence (>0.85) required.
- **ENRICH**: Topic appears novel AND at least one complexity signal: diff >200 lines, comments >5, has linked issues, or has label `bug`/`incident`/`breaking-change`/`security`.
- **STANDARD**: Novel topic with no strong complexity signals.

**Record tier in checkpoint** (`auto-compound-progress.json`) so resuming a run preserves tier decisions.

### Phase 1b (modified): Selective Full Fetch

The existing full fetch logic moves here with one addition:
```
Skip PRs classified as SKIP — do not fetch, record as skipped: triage_skip
Process STANDARD and ENRICH PRs through normal full fetch
```

### Phase 2 (modified): Archeologist Enrichment for ENRICH PRs

After the existing synthesis steps (2a–2c), add:

**2d. Archeologist enrichment (ENRICH tier only)**
Invoke `institutional-archeologist` agent with:
- The extracted module + symptom keywords for this PR
- Any `closingIssuesReferences` issue numbers

The archeologist will:
1. Check `docs/institutional/` for existing research (gap-fill if found)
2. Search GitHub Issues and PRs for prior discussions (always available)
3. Search Slack/Notion if configured in `compound-engineering.local.md`

**2e. Fold findings into synthesis**
Merge archeologist output into the already-extracted synthesis fields:
- **Root cause**: deepen with "prior discussions found X" if archeologist has context
- **Prevention**: enrich with "team noted going forward..." from prior decisions
- **Investigation** ("What Didn't Work"): add prior failed approaches if found
- **Solution context**: note if approach aligns with or departs from prior decisions

**2f. Invoke archivist to persist**
Call `institutional-archivist` with the archeologist's findings. The archivist writes to `docs/institutional/`. Future auto-compound runs will find these files during Step 2 (grep) in Phase 1.5, compounding the value forward.

### Rate Limit Impact

Phase 1.5 adds one lightweight `gh pr view` call per PR in the STANDARD/ENRICH path. SKIP PRs incur no additional calls. The net effect: fewer total API calls because SKIP PRs never reach the expensive Phase 1b full fetch.

| Scenario | Before | After |
|----------|--------|-------|
| Already-documented PR | 2 API calls + synthesis | 1 lightweight call (skipped after) |
| Standard novel PR | 2 API calls + synthesis | 1 lightweight + 2 full + synthesis (same) |
| Complex novel PR | 2 API calls + synthesis | 1 lightweight + 2 full + synthesis + archeologist |

## System-Wide Impact

- **Interaction graph**: Phase 1.5 inserts between 1a and 1b. Phase 2 branches on tier. Phase 4 dedup remains as safety net.
- **Error propagation**: If the Haiku classification call fails, default the affected batch to STANDARD (never SKIP on error — prefer false negatives to missed docs).
- **State lifecycle risks**: If ENRICH processing fails mid-PR (archeologist timeout etc.), the PR should fall back to STANDARD synthesis rather than failing entirely. Record `enrich_partial` in checkpoint.
- **API surface parity**: The `auto-compound.md` command wrapper requires no changes — date range flags already exist.
- **Checkpoint compatibility**: Progress file gains a `tier` field per PR. Existing checkpoints without `tier` should be treated as STANDARD for all unprocessed PRs.

## Acceptance Criteria

- [x] Phase 1.5 runs after Phase 1a enumeration and before Phase 1b full fetch
- [x] SKIP PRs are never passed to Phase 1b or synthesis; recorded as `triage_skip` in checkpoint and summary
- [x] Classification uses a Haiku sub-agent (not the main model); batches 5–10 PRs per call
- [x] SKIP requires confidence > 0.85 AND matching grep results; defaults to STANDARD on ambiguity
- [x] ENRICH PRs invoke `institutional-archeologist` after Phase 2c synthesis
- [x] Archeologist findings are folded into synthesis fields (root_cause, prevention, investigation)
- [x] `institutional-archivist` is invoked for each ENRICH PR to persist `docs/institutional/` entries
- [x] If archeologist/archivist fails, PR falls back to STANDARD synthesis (no hard failure)
- [x] Phase 4 deduplication remains unchanged as a safety net
- [x] Summary report shows SKIP triage counts separately from Phase 4 skip counts
- [x] `--dry-run` mode shows tier classification but does not invoke archeologist or write files
- [x] If `docs/solutions/` and `docs/institutional/` don't exist, all PRs default to STANDARD/ENRICH

## Implementation Plan

### Files to Change

**Primary change:**
- `plugins/compound-engineering/skills/auto-compound/SKILL.md`

**What changes in SKILL.md:**

1. **Restructure Phase 1** into Phase 1a (enumeration — current Phase 1a content) and Phase 1b (full fetch — current Phase 1b content with SKIP filter added)

2. **Add Phase 1.5: Triage** between 1a and 1b with the full step-by-step described above

3. **Update Phase 2 header** to note the ENRICH branch (steps 2d–2f added after 2c)

4. **Update Phase 4 header** to note it remains as a safety net for triage false negatives

5. **Update Phase 6 summary** to include triage stats: skipped (triage_skip), previously called "skipped (duplicate)" should clarify it's Phase 4 dedup

6. **Update Phase 1d checkpoint** schema to add `tier` field per PR

**No changes needed:**
- `commands/workflows/auto-compound.md` — date range flags already present
- Reference files (`context-synthesis.md`, `classification.md`) — unchanged
- Any other plugin files

### Pseudo-code for SKILL.md additions

**Phase 1.5 triage prompt for Haiku:**
```
You are classifying merged PRs for knowledge extraction. For each PR, output JSON.

SKIP: The topic is clearly already documented in existing solution files.
  - Requirement: grep matches exist AND topic overlap is high confidence (>0.85)
  - Default to STANDARD if uncertain — we prefer to process than to miss

ENRICH: Topic is novel AND complex enough to warrant external institutional research.
  - Signals: diff >200 lines, >5 review comments, has linked issues, bug/incident/security label
  - ENRICH should be <20% of batch — reserve for genuinely high-value PRs

STANDARD: Everything else — novel topic, normal complexity, process normally.

For each PR:
{"pr": <number>, "tier": "SKIP|STANDARD|ENRICH", "confidence": 0.0-1.0, "reason": "<15 words>"}
```

## Dependencies & Risks

**Dependencies:**
- `institutional-archeologist` and `institutional-archivist` agents (already exist at `agents/research/institutional-archeologist.md` and `agents/workflow/institutional-archivist.md`)
- `gh` CLI (already required)
- Haiku model access via Task tool

**Risks:**
- Haiku classification may over-SKIP in dense codebases where many old docs share keywords → mitigated by high confidence threshold (0.85) and Phase 4 safety net
- Archeologist adds latency for ENRICH PRs → acceptable since these are explicitly flagged as high-value
- Archeologist may return empty findings for novel topics → handled gracefully, proceed with STANDARD synthesis

## References & Research

### Internal References

- Auto-compound skill: `plugins/compound-engineering/skills/auto-compound/SKILL.md`
- Command wrapper: `plugins/compound-engineering/commands/workflows/auto-compound.md`
- Archeologist agent: `plugins/compound-engineering/agents/research/institutional-archeologist.md`
- Archivist agent: `plugins/compound-engineering/agents/workflow/institutional-archivist.md`
- Context synthesis ref: `plugins/compound-engineering/skills/auto-compound/references/context-synthesis.md`
- Classification ref: `plugins/compound-engineering/skills/auto-compound/references/classification.md`
- Brainstorm: `docs/brainstorms/2026-02-22-auto-compound-enrichment-brainstorm.md`

### Design Decisions (from brainstorm)

- **Triage method**: Combined grep + Haiku-as-judge (not pure grep, not full LLM)
- **ENRICH criteria**: LLM-assesses diff size, comment count, description, linked issues
- **SKIP threshold**: Confidence >0.85 required; defaults to STANDARD on ambiguity
- **Archeologist scope**: Per-PR, only for ENRICH tier
- **Archivist output**: Fold into synthesis fields + persist to `docs/institutional/`
