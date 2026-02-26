---
date: 2026-02-22
topic: auto-compound-enrichment
---

# Auto-Compound: Tiered Processing with Institutional Enrichment

## What We're Building

Improving the `workflows:auto-compound` skill to minimize unnecessary token usage while maximizing knowledge capture. The key insight: most of the cost in auto-compound comes from context synthesis (Phase 2), which fetches full PR details and runs LLM extraction. By triaging PRs early — before synthesis — we can skip already-documented topics, process novel ones normally, and selectively enrich complex/high-value PRs with institutional context from the archeologist agent.

The date range improvement was already addressed by the existing `--since`/`--until` flags on the command — no changes needed there.

## Why This Approach

Three approaches were considered:

- **Tiered Processing (chosen)** — Lightweight triage after enumeration classifies each PR as SKIP/STANDARD/ENRICH before any expensive synthesis. Maximum token savings.
- **Front-loaded Knowledge Load** — Run learnings-researcher upfront for a full knowledge index. Simpler but higher upfront cost; doesn't save synthesis tokens.
- **Status Quo +** — Expand existing Phase 4 deduplication to include institutional docs. Minimal changes but doesn't save synthesis tokens since dedup happens after the expensive work.

Tiered Processing wins because it cuts cost at the right stage: before synthesis, not after.

## Key Decisions

- **Triage after Phase 1**: After enumerating PRs (titles + labels only — cheap), run a lightweight keyword check against `docs/solutions/` and `docs/institutional/` using Grep before fetching full PR details.
- **Three tiers**: SKIP (high confidence already documented) / STANDARD (novel topic, synthesize normally) / ENRICH (novel + complex/high-value → invoke archeologist before synthesis).
- **Archeologist scope**: Optional enrichment phase, per-PR, only for ENRICH-classified PRs. The archeologist's findings are passed as additional context into synthesis (Phase 2), enriching the extraction rather than replacing it.
- **Deduplication stays**: Phase 4 deduplication remains as a safety net for cases where triage didn't catch overlaps.

## Resolved Questions

- **ENRICH criteria**: LLM-as-judge (Haiku). Collect PR metadata — diff size, comment count, description, linked issue descriptions — then use a cheap Haiku call to judge whether archeologist enrichment is warranted. More nuanced than static thresholds.
- **SKIP threshold + triage method**: Combined triage approach. Grep `docs/solutions/` and `docs/institutional/` first (free). Pass grep results + PR metadata into a single Haiku call per PR that handles both SKIP assessment and ENRICH classification simultaneously. One cheap Haiku call per PR replaces multiple decision points.
- **Archeologist output integration**: Two outputs from each ENRICH PR: (1) fold archeologist findings into synthesis fields (root cause, prevention, investigation) for richer solution docs; (2) invoke archivist to persist findings to `docs/institutional/`. Future auto-compound runs find these during their own triage, compounding the value forward.

## Next Steps

→ `/workflows:plan` for implementation details
