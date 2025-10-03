# Integration Plan: Unify my-opencode into spec-kit's-opencode

Status: Proposed (no code changes applied)
Owner: alperenyagci
Decision Summary:
- Spec‑Kit is the single source of truth
- Use all subagents from my‑opencode
- No thoughts/ usage and no migration of any legacy thoughts content
- No wrappers or env toggles
- Produce per‑feature `research.md` with a machine‑readable JSON appendix

## Objectives
- Unify the development workflow under Spec‑Kit commands and guardrails.
- Integrate my‑opencode’s subagent strengths into Spec‑Kit’s planning, analysis, and implementation commands.
- Eliminate the separate thoughts/ knowledge silo entirely.
- Enhance automation by appending a structured JSON appendix to `specs/<feature>/research.md`.

## Scope
- Repository areas touched:
  - `spec-kit's-opencode/.opencode/command/{plan.md, analyze.md, implement.md}`
  - `spec-kit's-opencode/.opencode/agent/*` (new)
  - `.specify/scripts/bash/*` (read‑only; keep as control plane)
  - `AGENTS.md` (project‑level memory, optional docs note)
- Out of scope:
  - Any thoughts/ folder creation or migration
  - Altering Spec‑Kit’s core bash scripts beyond their current behavior
  - Adding wrappers, toggles, or alternate command names

## Deliverables
- New subagent files under `spec-kit's-opencode/.opencode/agent/`:
  - `codebase-locator.md` (copy from `my-opencode/agent/codebase-locator.md`)
  - `codebase-analyzer.md` (copy from `my-opencode/agent/codebase-analyzer.md`)
  - `codebase-pattern-finder.md` (copy from `my-opencode/agent/codebase-pattern-finder.md`)
  - Renamed, retargeted doc agents (replace thoughts/*):
    - `artifact-locator.md` (from `my-opencode/agent/thoughts-locator.md`, targets Spec‑Kit artifacts only)
    - `artifact-analyzer.md` (from `my-opencode/agent/thoughts-analyzer.md`, targets Spec‑Kit artifacts only)
  - Optional: `web-search-researcher.md` (best‑effort, offline‑safe)
- Command enhancements:
  - `plan.md`: Phase 0 runs subagents, writes narrative + JSON appendix to `research.md` and enforces acceptance gates
  - `analyze.md`: Adds read‑only “source reality check” against `research.json` data
  - `implement.md`: Derives “Edit Waypoints” from `research.json`; blocks tasks lacking anchors
- JSON schema for the `research.md` appendix (machine‑readable block)
- Brief documentation note in `AGENTS.md` on the JSON appendix (optional)

## Non‑Goals
- Maintaining a thoughts/ directory or migrating legacy thoughts content
- Introducing parallel command names or compatibility wrappers
- Changing Spec‑Kit’s `specify → plan → tasks → analyze → implement` flow order

## Agent Mapping and Retargeting
- Code agents (drop‑in):
  - `codebase-locator` → inventory code surfaces and entry points
  - `codebase-analyzer` → trace data flows and call graphs with file:line anchors
  - `codebase-pattern-finder` → surface analogous implementations
- Doc agents (renamed, no thoughts/):
  - `artifact-locator` (was thoughts‑locator) → index Spec‑Kit artifacts only:
    - `specs/**/{spec.md, plan.md, research.md, tasks.md}`
    - `AGENTS.md`, `.specify/memory/constitution.md`, `docs/**`, `CHANGELOG.md`
  - `artifact-analyzer` (was thoughts‑analyzer) → extract decisions, trade‑offs, constraints from those artifacts
- Optional:
  - `web-search-researcher` used only when network is available or explicitly requested; include citations only

## Research.md Structure (with JSON Appendix)
Narrative sections:
- Overview and Scope
- Code Surface Map (entries, modules, key files)
- Data Flow Traces (file:line anchors)
- Similar Implementations/Patterns
- Risks, Unknowns, Assumptions
- Evidence Index (files and lines referenced)

JSON appendix (single fenced block at end):
```json
{
  "schema_version": "1.0",
  "feature": {
    "branch": "string",
    "feature_dir": "string",
    "spec_path": "string",
    "plan_path": "string"
  },
  "code_surfaces": [
    { "path": "string", "role": "module|service|cli|endpoint|lib", "symbols": ["string"], "entry": true }
  ],
  "flows": [
    { "from": {"path": "string", "symbol": "string"}, "to": {"path": "string", "symbol": "string"}, "via": {"pattern": "string"}, "notes": "string" }
  ],
  "patterns": [
    { "name": "string", "refs": [ { "path": "string", "lines": [1,2] } ], "rationale": "string" }
  ],
  "risks": [
    { "id": "R1", "severity": "LOW|MEDIUM|HIGH|CRITICAL", "summary": "string", "evidence": [ { "path": "string", "lines": [1,2] } ] }
  ],
  "open_questions": [ { "id": "Q1", "text": "string" } ],
  "artifact_index": [
    { "type": "spec|plan|tasks|research|constitution|agents|docs|changelog", "path": "string", "title": "string", "date": "string" }
  ],
  "evidence_index": [ { "path": "string", "lines": [1,2,3] } ]
}
```

## Command Enhancements (No wrappers, Spec‑Kit as control plane)

### plan.md — Phase 0: Research & Discovery
Inputs: `.specify/scripts/bash/setup-plan.sh --json` → FEATURE_SPEC, IMPL_PLAN, SPECS_DIR, BRANCH (absolute paths).
Steps:
1) Run `artifact-locator` to build `artifact_index` (no thoughts/; only Spec‑Kit artifacts and project docs).
2) Run `codebase-locator` to identify code surfaces and plausible entries.
3) Run `codebase-analyzer` (top N surfaces) to extract flows and anchors.
4) Run `codebase-pattern-finder` to list analogous implementations and references.
5) Synthesize narrative sections and JSON appendix; write to `specs/<feature>/research.md`.
6) Validate the JSON schema and Phase 0 acceptance gates; mark progress in plan.

Phase 0 acceptance gates:
- JSON present and parseable
- ≥ 1 `code_surfaces` entry with `entry=true`
- ≥ 1 `flows` item
- ≥ 1 `risks` item
- `evidence_index` includes ≥ 5 unique file:line anchors

### analyze.md — Read‑Only Reality Check
Inputs: `spec.md`, `plan.md`, `tasks.md`, `research.md` (parse JSON), constitution.
Checks:
- Terminology/entities across artifacts vs `code_surfaces`/`flows`
- File paths in tasks vs actual repo and surfaces
- Constitution alignment (keep existing severity rules)
Behavior:
- Report structured findings. If JSON missing/invalid → CRITICAL finding.

### implement.md — Execution Targeting
Inputs: `.specify/scripts/bash/check-prerequisites.sh --json --require-tasks` + artifacts.
Per task:
- Map task paths to `code_surfaces`; derive Edit Waypoints `{path, symbol, approx_lines}`
- If absent, run `codebase-analyzer` to locate anchors; if still absent → block with actionable error
Execution:
- Respect phases, dependencies, and [P] parallelism in tasks.md; mark completion by updating tasks file

## Validation & Tooling
- JSON schema validation in Plan Phase 0; re‑validate in Analyze/Implement
- Evidence integrity: verify referenced files exist and line ranges are valid
- Offline behavior: `web-search-researcher` is best‑effort; never blocks gates

## Success Criteria
- `/plan` produces `research.md` with valid JSON and passes gates
- `/analyze` cross‑checks artifacts with code reality and constitution deterministically
- `/implement` computes Edit Waypoints from JSON; blocks unsafe edits with clear diagnostics
- No thoughts/ usage; no migration required; `my-opencode/` can be deleted without loss

## Risks & Mitigations
- Large codebases → limit analyzer to top N hotspots (configurable; default N=20)
- Schema drift → include `schema_version`; centralize JSON validation logic in the command prompt
- Task path inaccuracies → auto‑map through `code_surfaces` and report diffs before blocking
- Network limits → treat web search as optional enrichment only

## Rollout Plan
- Phase A: Add subagents under `.opencode/agent/*` with renames (no behavior change yet)
- Phase B: Enhance `plan.md` to generate `research.md` + JSON and enforce gates
- Phase C: Enhance `analyze.md` to read JSON and perform source reality checks
- Phase D: Enhance `implement.md` to require Edit Waypoints derived from JSON
- Phase E: Validate on a fresh feature: `/specify` → `/plan` → `/tasks` → `/analyze` → `/implement`
- Phase F: Remove `my-opencode/` after parity confirmed

## Open Decisions (to finalize before edits)
- JSON schema finalized as listed with `schema_version = "1.0"`
- Analyzer hotspot budget (default N=20)
- Minimum evidence anchors threshold (default 5)
- Optional: include `last_seen_commit` per `code_surfaces` entry for stability

---

Appendix: Implementation Checklist
- [ ] Copy subagents to `spec-kit's-opencode/.opencode/agent/` and rename thoughts‑* → artifact‑*
- [ ] Update `plan.md` to run subagents and write `research.md` with JSON appendix and gates
- [ ] Update `analyze.md` to validate artifacts against code using `research.json`
- [ ] Update `implement.md` to compute Edit Waypoints and enforce presence
- [ ] Sanity‑run end‑to‑end on a new feature; confirm gates and reports
- [ ] Delete `my-opencode/` after validation
