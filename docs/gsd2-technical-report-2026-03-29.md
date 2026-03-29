# GSD2 Technical Report (Source-Level)

Date: 2026-03-29

## Purpose

This report consolidates the architecture, runtime behavior, prompt flow, persistence model, collaboration model, and porting implications for GSD2, based on source inspection and the discussion in this session.

The main goal is to explain how GSD2 really works, step-by-step, including ELI5 explanations and technical implementation references.

## Executive Summary

1. GSD2 is an orchestrated unit engine, not just a prompt pack. It has:
   - a loop and phase engine,
   - a dispatch rule table,
   - a prompt compiler,
   - a state derivation engine,
   - persistence and projection layers.
2. Current behavior is DB-first for operational state (SQLite), with markdown mostly acting as a projection and human-readable artifact layer.
3. Auto mode is guarded by lock and recovery mechanisms so only one session controls a project lane at a time.
4. Team collaboration is primarily milestone-lane parallelism, not true multi-writer, same-milestone real-time planning.
5. Porting functionality to Exaskill is feasible if you port the orchestration contracts (loop, dispatch, state, mutation API, verification), not only prompt text.

---

## High-Level Architecture

At runtime, GSD2 behaves like this:

1. Bootstrap auto session
2. Derive current state and phase
3. Resolve next dispatch rule
4. Run one unit in a fresh LLM session
5. Verify and finalize
6. Re-derive and repeat

Primary code entry points:

- Start/stop and session lifecycle: `src/resources/extensions/gsd/auto.ts:1057`
- Bootstrap flow: `src/resources/extensions/gsd/auto-start.ts:121`
- Main loop: `src/resources/extensions/gsd/auto/loop.ts:41`
- Phase handlers: `src/resources/extensions/gsd/auto/phases.ts:156`
- Dispatch table: `src/resources/extensions/gsd/auto-dispatch.ts:134`
- State derivation: `src/resources/extensions/gsd/state.ts:195`

---

## Locking and Crash Recovery (ELI5 + Technical)

### ELI5

Think of auto-mode like a forklift operating in a narrow aisle.

- A lock means "only one forklift in this aisle."
- If another forklift tries to enter, it is blocked.
- While working, the forklift writes a sticky note saying what box it is carrying.
- If power dies, next startup reads the sticky note and can recover context.

### Technical

There are two related mechanisms:

1. Session lock (true concurrency guard)
   - Acquired at auto start via `acquireSessionLock()`.
   - Uses OS-level locking (via `proper-lockfile`) on lock target directories.
   - Writes metadata to `.gsd/auto.lock`.
   - References:
     - `src/resources/extensions/gsd/session-lock.ts:252`
     - `src/resources/extensions/gsd/session-lock.ts:298`
     - `src/resources/extensions/gsd/session-lock.ts:312`

2. Crash lock metadata (recovery context)
   - Writes unit metadata (pid, unitType, unitId, sessionFile).
   - Read on next start to synthesize crash-recovery prompts.
   - References:
     - `src/resources/extensions/gsd/crash-recovery.ts:34`
     - `src/resources/extensions/gsd/crash-recovery.ts:63`

Validation and release behavior:

- Lock ownership is validated during loop iterations:
  - `src/resources/extensions/gsd/auto/loop.ts:97`
- Compromised lock handling and re-acquire logic:
  - `src/resources/extensions/gsd/session-lock.ts:422`
- Clean release on stop/pause:
  - `src/resources/extensions/gsd/session-lock.ts:490`

Parallel workers use per-milestone lock naming:

- `effectiveLockFile()` produces `auto-<milestone>.lock` in parallel mode.
- Reference: `src/resources/extensions/gsd/session-lock.ts:92`

---

## Full End-to-End Example (ELI5 + Technical)

Example scenario: user runs `/gsd auto` for a milestone that will implement login flow.

### Step 1: Start command enters auto lifecycle

- ELI5: "Turn on autopilot."
- Technical: `startAuto()` is called and routes either resume or fresh bootstrap.
- Reference: `src/resources/extensions/gsd/auto.ts:1057`

### Step 2: Acquire lock and prep environment

- ELI5: "Reserve exclusive control of this project lane."
- Technical:
  - Acquire session lock.
  - Ensure repo state and `.gsd` state setup.
  - Handle external-state migration/symlink.
- References:
  - `src/resources/extensions/gsd/auto-start.ts:137`
  - `src/resources/extensions/gsd/migrate-external.ts:36`
  - `src/resources/extensions/gsd/repo-identity.ts:367`

### Step 3: Open DB and migrate markdown if needed

- ELI5: "Open the project memory database; import old notes if needed."
- Technical:
  - Resolve DB path (including worktree-aware root DB behavior).
  - Open SQLite and initialize schema/migrations.
  - If markdown-only legacy state exists, run import.
- References:
  - `src/resources/extensions/gsd/bootstrap/dynamic-tools.ts:16`
  - `src/resources/extensions/gsd/gsd-db.ts:769`
  - `src/resources/extensions/gsd/gsd-db.ts:162`
  - `src/resources/extensions/gsd/md-importer.ts:693`
  - `src/resources/extensions/gsd/auto-start.ts:556`

### Step 4: Derive state

- ELI5: "Figure out where we are in the workflow."
- Technical:
  - `deriveState()` uses DB-first path when populated.
  - Falls back to filesystem for unmigrated/empty DB cases.
  - Computes phase and next action.
- References:
  - `src/resources/extensions/gsd/state.ts:195`
  - `src/resources/extensions/gsd/state.ts:279`

### Step 5: Enter main loop

- ELI5: "Repeat: choose next unit, run it, verify it."
- Technical: `autoLoop()` orchestrates derive, dispatch, guards, runUnit, finalize.
- Reference: `src/resources/extensions/gsd/auto/loop.ts:4`

### Step 6: Resolve dispatch rule for current phase

- ELI5: "Pick next job card based on state."
- Technical:
  - Rules evaluated in order, first match wins.
  - Example transitions:
    - `pre-planning` -> discuss/research/plan milestone
    - `planning` -> research/plan slice
    - `executing` -> execute-task
    - `summarizing` -> complete-slice
- References:
  - `src/resources/extensions/gsd/auto-dispatch.ts:134`
  - `src/resources/extensions/gsd/auto-dispatch.ts:734`

### Step 7: Build the unit prompt

- ELI5: "Assemble a focused brief with only relevant context."
- Technical:
  - Prompt templates are loaded and variable-substituted.
  - Context is inlined (milestone/slice/task artifacts, requirements, decisions).
  - Skill activation block is injected.
- References:
  - `src/resources/extensions/gsd/prompt-loader.ts:104`
  - `src/resources/extensions/gsd/auto-prompts.ts:901`
  - `src/resources/extensions/gsd/auto-prompts.ts:1026`
  - `src/resources/extensions/gsd/auto-prompts.ts:1093`
  - `src/resources/extensions/gsd/auto-prompts.ts:443`

### Step 8: Run unit in fresh session

- ELI5: "Use a fresh worker for this one job."
- Technical:
  - `newSession()` is created with timeout handling.
  - Prompt is sent.
  - Waits for one-shot `agent_end` resolve.
- References:
  - `src/resources/extensions/gsd/auto/run-unit.ts:24`
  - `src/resources/extensions/gsd/auto/resolve.ts:49`

### Step 9: Artifact verification and pre-verification post-unit work

- ELI5: "Check that worker actually produced the expected deliverable."
- Technical:
  - Expected artifacts mapped by unit type.
  - Missing artifacts can trigger retries and projection regeneration attempts.
- References:
  - `src/resources/extensions/gsd/auto-artifact-paths.ts:22`
  - `src/resources/extensions/gsd/auto-recovery.ts:171`
  - `src/resources/extensions/gsd/auto-post-unit.ts:433`

### Step 10: Verification gate

- ELI5: "Run tests/checks; if failing, request auto-fix retries."
- Technical:
  - Executes configured verification commands.
  - Writes verification evidence.
  - Returns continue/retry/pause.
- Reference: `src/resources/extensions/gsd/auto-verification.ts:49`

### Step 11: Post-verification sidecars and transitions

- ELI5: "Run hooks and side jobs, then continue to next unit."
- Technical:
  - Post-unit hooks, triage captures, quick-task dispatch are queued as sidecar items.
  - Loop continues until terminal condition (blocked/complete/stop).
- References:
  - `src/resources/extensions/gsd/auto-post-unit.ts:507`
  - `src/resources/extensions/gsd/auto/phases.ts:1210`

### Step 12: Milestone completion and merge

- ELI5: "When all slices done and validated, finish milestone and merge branch/worktree."
- Technical:
  - Complete/validate phase logic triggers merge and stop paths.
  - Merge conflict handling is explicit and stops for manual resolution.
- References:
  - `src/resources/extensions/gsd/auto/phases.ts:462`
  - `src/resources/extensions/gsd/auto/phases.ts:264`

---

## Prompt Stack and Behavior

### System prompt injection

- GSD appends its system contract (`prompts/system.md`) into agent system context on startup.
- Reference: `src/resources/extensions/gsd/bootstrap/system-context.ts:44`

### Core unit prompts used in auto mode

- Discuss: `src/resources/extensions/gsd/prompts/guided-discuss-milestone.md`
- Plan milestone: `src/resources/extensions/gsd/prompts/plan-milestone.md`
- Plan slice: `src/resources/extensions/gsd/prompts/plan-slice.md`
- Execute task: `src/resources/extensions/gsd/prompts/execute-task.md`

These prompts are not generic text only; they enforce tool-backed state mutation patterns:

- plan milestone prompt requires `gsd_plan_milestone` usage.
- plan slice prompt requires `gsd_plan_slice` usage.
- execute task prompt requires writing summary + `gsd_complete_task`.

References:

- `src/resources/extensions/gsd/prompts/plan-milestone.md:50`
- `src/resources/extensions/gsd/prompts/plan-slice.md:70`
- `src/resources/extensions/gsd/prompts/execute-task.md:71`

---

## State Machine Details

`deriveStateFromDb()` computes the active milestone/slice/task and phase by combining DB rows plus selected filesystem flags/artifacts.

Important derived phases include:

- `pre-planning`
- `needs-discussion`
- `planning`
- `evaluating-gates`
- `executing`
- `summarizing`
- `replanning-slice`
- `validating-milestone`
- `completing-milestone`
- `complete`
- `blocked`

References:

- DB-first derive path: `src/resources/extensions/gsd/state.ts:279`
- slice/task execution phase return: `src/resources/extensions/gsd/state.ts:814`
- quality gate phase return: `src/resources/extensions/gsd/state.ts:740`
- replanning phase return: `src/resources/extensions/gsd/state.ts:775`

---

## Persistence Model: SQLite + Markdown Projection

### SQLite layer

- Provider chain: `node:sqlite` -> `better-sqlite3`.
- WAL mode and schema migrations at DB init.
- Schema includes milestones, slices, tasks, evidence, quality gates, dependencies, artifacts, requirements, decisions, memories.

References:

- provider chain: `src/resources/extensions/gsd/gsd-db.ts:54`
- WAL: `src/resources/extensions/gsd/gsd-db.ts:165`
- schema tables: `src/resources/extensions/gsd/gsd-db.ts:252`, `src/resources/extensions/gsd/gsd-db.ts:274`, `src/resources/extensions/gsd/gsd-db.ts:299`, `src/resources/extensions/gsd/gsd-db.ts:374`

### DB-backed mutation tools

- `plan-milestone` writes rows in transaction, then renders roadmap.
- `plan-slice` writes slice/tasks/gates in transaction, then renders plan.
- `complete-task` writes completion + verification evidence, then renders summary and checkboxes.

References:

- `src/resources/extensions/gsd/tools/plan-milestone.ts:168`
- `src/resources/extensions/gsd/tools/plan-slice.ts:125`
- `src/resources/extensions/gsd/tools/complete-task.ts:129`

### Markdown as projection layer

- Projection renderer writes PLAN/ROADMAP/SUMMARY/STATE from DB state.
- Stale detection and repair exist for DB-vs-disk drift.

References:

- projection module: `src/resources/extensions/gsd/workflow-projections.ts:1`
- stale detection: `src/resources/extensions/gsd/markdown-renderer.ts:802`
- stale repair: `src/resources/extensions/gsd/markdown-renderer.ts:959`

### Direct write interception

- Direct writes to `.gsd/STATE.md` are blocked and redirected toward tool API usage.
- Reference: `src/resources/extensions/gsd/write-intercept.ts:19`

---

## Reliability and Guardrails

1. Session lock ownership checks each loop iteration
2. Worktree health checks before execute-task dispatch
3. Artifact presence verification per unit type
4. Stuck-loop detection with graduated recovery
5. Verification gate with retries and pause-on-exhaustion
6. Merge conflict hard stop with actionable user message

References:

- loop lock check: `src/resources/extensions/gsd/auto/loop.ts:97`
- worktree health: `src/resources/extensions/gsd/auto/phases.ts:877`
- artifact verification retry: `src/resources/extensions/gsd/auto-post-unit.ts:468`
- stuck detection: `src/resources/extensions/gsd/auto/phases.ts:577`
- verification retry: `src/resources/extensions/gsd/auto-verification.ts:192`
- merge conflict handling: `src/resources/extensions/gsd/auto/phases.ts:266`

---

## Collaboration Model and Team Implications

### What is shared vs local

Observed behavior and docs indicate runtime state is local/ignored, including DB files:

- `.gsd/gsd.db`, `.gsd/gsd.db-shm`, `.gsd/gsd.db-wal` in runtime exclusion paths.
- Reference: `src/resources/extensions/gsd/gitignore.ts:31`

Team docs describe selective sharing of planning artifacts in some modes:

- `docs/working-in-teams.md:40`
- `docs/working-in-teams.md:47`

But preferences now mark `git.commit_docs` deprecated and describe `.gsd` as externally managed/gitignored:

- `src/resources/extensions/gsd/preferences-validation.ts:675`
- `src/resources/extensions/gsd/docs/preferences-reference.md:141`

### Practical interpretation

GSD2 currently behaves most naturally as:

- single-writer per active lane (session lock),
- local runtime DB and metadata,
- milestone-lane collaboration rather than many concurrent writers on one milestone plan.

---

## Porting GSD2 Functionality to Exaskill

### Feasibility

Port is feasible, but only if orchestration contracts are preserved.

Do not port only prompt text. Port these subsystems:

1. Loop and phase orchestration
2. Dispatch rule engine
3. State derivation API
4. Mutation API with transactional semantics
5. Artifact verification and retry system
6. Verification gate and evidence recording
7. Locking/recovery semantics

### Minimal adapter surface for Exaskill

Suggested interfaces to implement behind Exaskill/Beads state:

- `deriveState(): GSDStateLike`
- `resolveDispatch(state): DispatchActionLike`
- `planMilestone(payload)`
- `planSlice(payload)`
- `completeTask(payload)`
- `completeSlice(payload)`
- `completeMilestone(payload)`
- `recordVerificationEvidence(payload)`
- `verifyExpectedArtifact(unitType, unitId)`
- `acquireSessionLock(project, lane)` / `releaseSessionLock(project, lane)`

If Exaskill uses Beads as canonical state, that should remain the only source of truth. Avoid dual canonical stores (Beads + GSD SQLite).

---

## Notable Consistency Gaps to Watch

There are signs of evolution and some drift between docs and runtime behavior (for example around `.gsd` sharing and `commit_docs` semantics). Treat source behavior as canonical for implementation decisions.

Key references:

- Team sharing docs: `docs/working-in-teams.md:40`
- Deprecation warning: `src/resources/extensions/gsd/preferences-validation.ts:675`
- External state migration/symlink behavior: `src/resources/extensions/gsd/migrate-external.ts:36`, `src/resources/extensions/gsd/repo-identity.ts:367`

---

## Appendix: Core Source Reference Index

### Runtime loop and phases

- `src/resources/extensions/gsd/auto.ts:1057`
- `src/resources/extensions/gsd/auto-start.ts:121`
- `src/resources/extensions/gsd/auto/loop.ts:41`
- `src/resources/extensions/gsd/auto/phases.ts:156`
- `src/resources/extensions/gsd/auto/run-unit.ts:24`
- `src/resources/extensions/gsd/auto/resolve.ts:49`

### Dispatch and prompt build

- `src/resources/extensions/gsd/auto-dispatch.ts:134`
- `src/resources/extensions/gsd/auto-dispatch.ts:734`
- `src/resources/extensions/gsd/auto-prompts.ts:843`
- `src/resources/extensions/gsd/auto-prompts.ts:901`
- `src/resources/extensions/gsd/auto-prompts.ts:1026`
- `src/resources/extensions/gsd/auto-prompts.ts:1093`
- `src/resources/extensions/gsd/prompt-loader.ts:104`

### Prompt templates

- `src/resources/extensions/gsd/prompts/system.md`
- `src/resources/extensions/gsd/prompts/guided-discuss-milestone.md`
- `src/resources/extensions/gsd/prompts/plan-milestone.md`
- `src/resources/extensions/gsd/prompts/plan-slice.md`
- `src/resources/extensions/gsd/prompts/execute-task.md`

### State and persistence

- `src/resources/extensions/gsd/state.ts:195`
- `src/resources/extensions/gsd/state.ts:279`
- `src/resources/extensions/gsd/gsd-db.ts:162`
- `src/resources/extensions/gsd/gsd-db.ts:769`
- `src/resources/extensions/gsd/md-importer.ts:693`

### Tool-backed mutations

- `src/resources/extensions/gsd/tools/plan-milestone.ts:168`
- `src/resources/extensions/gsd/tools/plan-slice.ts:125`
- `src/resources/extensions/gsd/tools/complete-task.ts:129`

### Projection and drift handling

- `src/resources/extensions/gsd/workflow-projections.ts:310`
- `src/resources/extensions/gsd/markdown-renderer.ts:802`
- `src/resources/extensions/gsd/markdown-renderer.ts:959`
- `src/resources/extensions/gsd/write-intercept.ts:19`

### Verification and post-unit pipeline

- `src/resources/extensions/gsd/auto-verification.ts:49`
- `src/resources/extensions/gsd/auto-post-unit.ts:234`
- `src/resources/extensions/gsd/auto-post-unit.ts:507`

### Locking and crash recovery

- `src/resources/extensions/gsd/session-lock.ts:252`
- `src/resources/extensions/gsd/session-lock.ts:390`
- `src/resources/extensions/gsd/session-lock.ts:422`
- `src/resources/extensions/gsd/session-lock.ts:490`
- `src/resources/extensions/gsd/crash-recovery.ts:34`
- `src/resources/extensions/gsd/crash-recovery.ts:63`

### Team/collaboration related

- `docs/working-in-teams.md:40`
- `docs/working-in-teams.md:47`
- `src/resources/extensions/gsd/gitignore.ts:31`
- `src/resources/extensions/gsd/preferences-validation.ts:675`
- `src/resources/extensions/gsd/docs/preferences-reference.md:141`
