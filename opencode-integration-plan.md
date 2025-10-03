# Integration Plan: Unify my-opencode into spec-kit's-opencode

Status: Proposed (no code changes applied)
Owner: alperenyagci
Decision Summary:

- Spec‑Kit is the single source of truth
- Use all subagents from my‑opencode
- Preserve thoughts/ workflow: keep a standalone `/research_codebase` command that writes to thoughts/; `/plan` must always refer to relevant thoughts research if present (no migration of legacy content)
- No wrappers or env toggles
- Produce per‑feature `research.md` (narrative) with file:line references

## Objectives

- Unify the development workflow under Spec‑Kit commands and guardrails.
- Integrate my‑opencode’s subagent strengths into Spec‑Kit’s planning, analysis, and implementation commands.
- Preserve thoughts/ as an auxiliary research space referenced by `/plan`.
- Keep outputs consistent with Spec‑Kit defaults (Markdown artifacts only).

## Workflow Diagram (Mermaid)

```mermaid
flowchart LR

  %% Commands
  subgraph Commands
    SPEC[/specify/]
    PLAN[/plan/]
    TASKS[/tasks/]
    ANALYZE[/analyze/]
    IMPLEMENT[/implement/]
    RC[/research_codebase/]
  end

  SPEC --> PLAN --> TASKS --> ANALYZE --> IMPLEMENT

  %% Subagents (documentarians)
  subgraph Subagents
    L[[codebase-locator]]
    A[[codebase-analyzer]]
    PF[[codebase-pattern-finder]]
    TL[[thoughts-locator]]
    TA[[thoughts-analyzer]]
    WS[[web-search-researcher (optional)]]
  end

  %% Plan uses subagents in Phase 0 (research) to produce narrative research.md
  PLAN -- "Phase 0 research" --> TL
  PLAN -- "Phase 0 research" --> L
  PLAN -. optional .-> TA
  PLAN --> A
  PLAN --> PF

  %% Standalone research command uses subagents heavily, writes to thoughts/
  RC --> TL
  RC --> TA
  RC --> L
  RC --> A
  RC --> PF
  RC -. optional .-> WS

  %% Artifacts
  PLAN -->|writes| RMD[research.md]
  RC -->|writes| TR[thoughts/.../research/*.md]

  %% Analyze remains read-only; may optionally consult locator/analyzer
  ANALYZE -. may use .-> L
  ANALYZE -. may use .-> A

  %% Implement executes tasks; no subagents introduced here in this plan

  %% Styling
  classDef cmd fill:#eef,stroke:#335,stroke-width:1px,color:#000;
  classDef agent fill:#efe,stroke:#353,stroke-width:1px,color:#000;
  classDef artifact fill:#ffe,stroke:#aa6,stroke-width:1px,color:#000;
  class SPEC,PLAN,TASKS,ANALYZE,IMPLEMENT,RC cmd;
  class L,A,PF,TL,TA,WS agent;
  class RMD,TR artifact;
```

## Scope

- Repository areas touched:
  - `spec-kit's-opencode/.opencode/command/{plan.md, analyze.md, implement.md, research_codebase.md}`
  - `spec-kit's-opencode/.opencode/agent/*` (new)
  - `.specify/scripts/bash/*` (read‑only; keep as control plane)
  - `AGENTS.md` (project‑level memory, optional docs note)
- Out of scope:
  - Migration of existing thoughts/ documents (we keep them as‑is and reference when relevant)
  - Altering Spec‑Kit’s core bash scripts beyond their current behavior
  - Adding wrappers, toggles, or alternate command names

## Deliverables

- Validate and synchronize existing subagent files under `spec-kit's-opencode/.opencode/agent/*.md`:
  - Confirm `codebase-locator.md`, `codebase-analyzer.md`, and `codebase-pattern-finder.md` mirror the latest intent from the legacy my-opencode counterparts.
  - Ensure thoughts workflow agents `thoughts-locator.md` and `thoughts-analyzer.md` remain in parity with the historical behavior.
  - Optional: verify presence and freshness of `web-search-researcher.md` (best-effort, offline-safe) and document any gaps.
- Comparison workflow:
  - If a historical `my-opencode/agent/*.md` source exists (local checkout, archive, or commit history), diff each agent against the corresponding file under `spec-kit's-opencode/.opencode/agent/`.
  - Capture unresolved deltas in an integration log (e.g., `notes/opencode-agent-diffs.md`) with action items for follow-up porting.
  - When historical sources are unavailable, record that status and flag the agent for exploratory validation during rollout.
- New main command:
  - `research_codebase.md` (copy from `my-opencode/command/research_codebase.md`) — standalone, writes to thoughts/
- Command alignment (no new gates or artifacts):
  - `plan.md`: Research-first policy — ALWAYS check for a `thoughts/` directory and consult existing research; run `thoughts-locator` (and optionally `thoughts-analyzer`) to incorporate relevant insights into `research.md` (no additional gating beyond Spec‑Kit templates)
  - `analyze.md`: Keep read‑only cross‑artifact analysis as in Spec‑Kit; may link to thoughts/ research when present (non‑gating)
  - `implement.md`: Follow tasks.md execution per Spec‑Kit defaults; may use `research.md` as context (no Edit Waypoints or anchor gating)

## Non‑Goals

- Migrating existing thoughts/ content (we will reference it when relevant)
- Introducing parallel command names or compatibility wrappers
- Changing Spec‑Kit’s `specify → plan → tasks → analyze → implement` flow order

## Agent Mapping

- Code agents (drop‑in):
  - `codebase-locator` → inventory code surfaces and entry points
  - `codebase-analyzer` → trace data flows and call graphs with file:line anchors
  - `codebase-pattern-finder` → surface analogous implementations
- Thoughts agents (for external insights):
  - `thoughts-locator` → discover relevant documents under thoughts/
  - `thoughts-analyzer` → extract decisions, trade‑offs, constraints from thoughts docs
- Optional:
  - `web-search-researcher` used only when network is available or explicitly requested; include citations only

## Research.md Structure

Narrative sections (rendered in this order, with `##` headings):

- Overview and Scope — summarize feature intent, context, and guardrails inherited from spec/tasks.
- Code Surface Map — list entry points, primary modules, and key files with bullet summaries; each list item includes at least one `path:line` anchor.
- Data Flow Traces — enumerate at least one non-trivial control/data flow per entry surface; include anchors and describe pre/post state deltas.
- Similar Implementations & Patterns — connect to analogous implementations; cite anchors that justify reuse or divergence.
- External Insights — integrate `thoughts-*` results; for each linked `thoughts/` document include permalink, anchor (heading slug), and a one-line takeaway; if none exist, write “No external insights found (checked YYYY-MM-DD)”.
- Risks, Unknowns, Assumptions — highlight gaps or dependencies with owner + mitigation proposal when possible.
- (Optional) Citations list — bullet list of key file:line references collected during research.

Schema rules:

 
 
- Cross-links: each External Insight entry back-links into earlier sections via inline references (e.g., “(see Code Surface Map §API Gateway)”) to maintain traceability between thoughts research and in-repo findings.
- Front matter (YAML) optional; if present, include `research_version: 1` and `generated_by: plan` for downstream automation.

## Command Enhancements (No wrappers, Spec‑Kit as control plane)

### plan.md — Phase 0: Research & Discovery (research-first)

Inputs: `.specify/scripts/bash/setup-plan.sh --json` → FEATURE_SPEC, IMPL_PLAN, SPECS_DIR, BRANCH (absolute paths).
Steps:

0. Check for a `thoughts/` directory at repo root.
   - If present: Run `thoughts-locator` to find recent and topic‑relevant research docs (esp. those produced by `/research_codebase`); optionally run `thoughts-analyzer` to extract decisions/constraints.
   - If absent: Proceed, but note in External Insights: “No thoughts directory found (checked YYYY-MM-DD)”.
1. Run `thoughts-locator` to find recent and topic‑relevant research docs under thoughts/ (esp. those produced by `/research_codebase`).
2. Optionally run `thoughts-analyzer` on top K findings to extract decisions, constraints, and links for the “External Insights” section.
3. Run `codebase-locator` to identify code surfaces and plausible entries.
4. Run `codebase-analyzer` (top N surfaces) to extract flows and anchors.
5. Run `codebase-pattern-finder` to list analogous implementations and references.
6. Synthesize narrative sections (including External Insights) and write `specs/<feature>/research.md`.

 

### analyze.md — Read‑Only Reality Check

Inputs: `spec.md`, `plan.md`, `tasks.md`, `research.md` (narrative), constitution.
Checks:

- Terminology/entities across artifacts vs the documented surfaces/flows (may validate against live code with subagents)
- File paths in tasks vs actual repo (validate existence via subagents)
- Constitution alignment (keep existing severity rules)
  Behavior:
- Report structured findings. If `research.md` is missing critical sections, note it with remediation (non‑blocking).
- Optionally surface a non‑gating “Related thoughts research” subsection with links and deltas from `research.md`

 

 

### research_codebase — Standalone Research Command

Purpose: Ad‑hoc, subagent‑heavy exploration for complex questions; writes narrative research to thoughts/ (and may embed the same JSON block as a sidecar for reuse).
Location: `spec-kit's-opencode/.opencode/command/research_codebase.md`
Behavior:

- Use `codebase-locator`, `codebase-analyzer`, `codebase-pattern-finder`, and `thoughts-*` agents as orchestrated in your legacy command
- Output to `thoughts/shared/research/YYYY-MM-DD-topic.md` (or user‑scoped)
- Include links and, optionally, a JSON appendix compatible with `research.md` schema for easy import
- `/plan` must always refer to these outputs (if present) in its External Insights section

## Validation & Tooling

- Offline behavior: `web-search-researcher` is best‑effort; treat web search as optional enrichment only.

## Success Criteria

- `/plan` produces a narrative `research.md` with file:line references
- `/analyze` remains read‑only cross‑artifact per Spec‑Kit defaults
- `/implement` executes tasks per Spec‑Kit defaults
- Thoughts workflow preserved; `my-opencode/` can be deleted after parity

## Risks & Mitigations

- Task path inaccuracies → use `codebase-locator` and report diffs
- Network limits → treat web search as optional enrichment only

## Rollout Plan

- Phase A: Add subagents under `.opencode/agent/*` (parity with my‑opencode)
- Phase B: Provide `/research_codebase` (standalone; writes to thoughts/)
- Phase C: Validate end‑to‑end on a fresh feature: `/specify` → `/plan` → `/tasks` → `/analyze` → `/implement`
- Phase D: Remove `my-opencode/` after parity confirmed

---

Appendix: Implementation Checklist

- [ ] Validate existing `spec-kit's-opencode/.opencode/agent/*.md` contents against historical my-opencode sources (capture unresolved diffs)
- [ ] Ensure `research_codebase.md` exists and matches legacy behavior (writes to thoughts/)
- [ ] Sanity‑run end‑to‑end on a new feature; confirm artifacts align with Spec‑Kit defaults
- [ ] Delete `my-opencode/` after validation
