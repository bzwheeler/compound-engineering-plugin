# Fork Context For AI Agents

Last updated: 2026-02-22 (auto-compound tiered enrichment)

## TL;DR

This fork is rebased on `upstream/main`. Its intended delta includes:
institutional knowledge agents (Archeologist + Archivist), domain-specialist
learning loop, configurable artifact paths for plan/brainstorm/todo workflows,
and tiered enrichment for `auto-compound` (triage before full PR fetch).

## Repository Relationship Snapshot

- Fork remote: `origin` -> `https://github.com/bzwheeler/compound-engineering-plugin.git`
- Upstream remote: `upstream` -> `https://github.com/EveryInc/compound-engineering-plugin.git`
- Current local branch: `feat/domain-specialists`
- Local `main` equals `upstream/main`:
  - `main...upstream/main` = `0 ahead / 0 behind`
- Local feature branch delta:
  - `codex/configurable-temp-artifact-paths...upstream/main` = local feature work in progress
- Remote state (before push):
  - `origin/main...upstream/main` = `0 ahead / 12 behind`

Interpretation:
- Sync/rebase is complete locally.
- Only one feature commit remains on top of upstream.
- `origin` needs a push if remote branches should reflect this sync.

## Intended Fork Delta

The fork introduces these non-upstream capabilities:

- **Auto-compound tiered enrichment** (`feat/domain-specialists`, 2026-02-22):
  - **Phase 1.5 Triage** inserted between PR enumeration and full fetch: keyword extraction → parallel grep of `docs/solutions/` + `docs/institutional/` → lightweight `gh pr view` (1 call/PR) → Haiku batch classification (5–10 PRs/call) → SKIP / STANDARD / ENRICH
  - **SKIP** PRs (topic already documented, confidence >0.85) bypass full fetch entirely — `triage_skip` in checkpoint; no synthesis run
  - **ENRICH** PRs (novel + complexity signals: diff >200 lines, >5 comments, linked issues, or bug/incident/security label) invoke `institutional-archeologist` after Phase 2c, fold findings into synthesis fields (root_cause, prevention, investigation), then call `institutional-archivist` to persist `docs/institutional/` — compounding triage accuracy on future runs
  - Failure modes are soft: classification error → STANDARD; archeologist/archivist error → `enrich_partial` fallback, batch continues
  - Phase 6 summary now separates triage-skip counts from Phase 4 dedup counts
  - Checkpoint schema gains `tier` field per PR; old checkpoints default to STANDARD

- **Institutional knowledge pipeline** (`feat/domain-specialists`, v2.36.0):
  - New agent: `agents/research/institutional-archeologist.md` — searches external sources (Slack, GitHub Issues/PRs, Notion) before planning; checks `docs/institutional/` first and gap-fills rather than re-excavating; hands off to archivist
  - New agent: `agents/workflow/institutional-archivist.md` — distills findings into `docs/institutional/*.md` with YAML frontmatter (`topics`, `sources`, `last_searched_date`, `key_decisions`, `open_questions`) for grep-first retrieval
  - `workflows:plan` + `workflows:brainstorm` — both now run archeologist+archivist in parallel with existing research agents
  - `learnings-researcher` — extended to search `docs/institutional/` alongside `docs/solutions/`
  - `compound-engineering.local.md` — new `institutional_knowledge` config block (enable/disable, `data_sources` list, `institutional_dir` override)

- New commands:
  - `plugins/compound-engineering/commands/workflows/auto-compound.md`
  - `plugins/compound-engineering/commands/workflows/retro.md`
- New skills:
  - `plugins/compound-engineering/skills/auto-compound/SKILL.md`
  - `plugins/compound-engineering/skills/auto-compound/references/classification.md`
  - `plugins/compound-engineering/skills/auto-compound/references/context-synthesis.md`
  - `plugins/compound-engineering/skills/domain-specialists/SKILL.md`
  - `plugins/compound-engineering/skills/domain-specialists/workflows/retro.md`
  - `plugins/compound-engineering/skills/domain-specialists/workflows/scaffold.md`
- Related integration/docs changes:
  - `workflows:work`, `workflows:review`, `workflows:compound` wiring
  - `setup` skill updates for specialist configuration
  - compound-docs schema/reference updates
  - plugin/readme/changelog/version metadata updates

- Configurable temporary artifact paths via `compound-engineering.local.md`:
  - New optional frontmatter:
    - `artifact_paths.root`
    - `artifact_paths.todos`
    - `artifact_paths.brainstorms`
    - `artifact_paths.plans`
  - Path resolution behavior:
    1. Per-path override (`artifact_paths.<kind>`)
    2. Derived from `artifact_paths.root`
    3. Backward-compatible defaults (`todos`, `docs/brainstorms`, `docs/plans`)
  - Updated workflows/skills now resolve and use configured paths:
    - `workflows:brainstorm`
    - `workflows:plan`
    - `workflows:review`
    - `deepen-plan`
    - `resolve_todo_parallel`
    - `triage`
    - `file-todos`
    - `brainstorming`
    - `setup` (now offers artifact-path configuration)

## Future Work

### Knowledge Base Search Scaling (not yet implemented)

**Context:** Phase 1.5 Triage uses `grep -rl` across `docs/solutions/` and
`docs/institutional/` to find keyword matches before Haiku classification.
Grep performance is not the concern — it stays fast even at 500+ files.
The real degradation is **semantic quality**: as the corpus grows, keyword
collisions increase (many unrelated docs match generic terms like "billing"
or "query"), false SKIP rates rise, and synonym gaps persist ("N+1 query"
vs. "eager loading" are the same problem but grep never connects them).

**Recommended progression:**

1. **~150 files — generated index (no new dependencies):**
   Auto-compound writes `docs/solutions/.index.json` after each run:
   ```json
   [
     { "file": "billing/n-plus-one-query.md", "module": "billing",
       "tags": ["n+1", "query", "eager-loading"],
       "symptoms": ["slow response", "high db cpu"] }
   ]
   ```
   Phase 1.5 Step 2 queries this single file with `jq` instead of grepping
   hundreds of individual docs. Supports richer metadata (canonical problem
   names, manually curated synonyms) without touching source files.
   The auto-compound write step (Phase 5) and the archivist should both
   update the index when creating new entries.

2. **~500+ files — local vector store (adds a dependency):**
   Embed all docs at write time using the Anthropic embeddings API
   (or a local model). Store in sqlite-vec or similar. Phase 1.5 replaces
   grep with a semantic similarity query — catches synonyms, related
   concepts, and cross-domain patterns. Requires: an embedding build step
   triggered by `auto-compound` and `workflows:compound`, and a
   `compound-engineering.local.md` flag to opt in (preserves the plugin's
   zero-dependency default for smaller repos).

**Constraint to preserve:** The plugin currently requires only `gh` CLI.
Any solution that adds a mandatory runtime dependency (vector DB, embedding
model) should be opt-in via `compound-engineering.local.md`, not the default.

**Signal to act:** Monitor false SKIP rates in the Phase 6 summary report.
When "Skipped (duplicate, Phase 4)" counts rise while "SKIP (triage)"
counts also rise, the triage step is over-skipping — keyword collisions are
causing it to SKIP PRs that should have been processed. That's the trigger
to move to the index or vector approach.

## Quick Verification Commands

```bash
git fetch upstream origin
git rev-list --left-right --count main...upstream/main
git rev-list --left-right --count feat/domain-specialists...upstream/main
git diff --name-only upstream/main..feat/domain-specialists
```
