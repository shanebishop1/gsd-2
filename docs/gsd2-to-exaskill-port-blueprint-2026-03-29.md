# GSD2 -> Exaskill Port Blueprint

Date: 2026-03-29

## Objective

Define a concrete, low-risk port plan that reproduces GSD2 behavior inside Exaskill using Exaskill/Beads-native state, while preserving GSD2 orchestration guarantees.

## Scope

This blueprint ports runtime behavior contracts, not file layout or exact internal module names.

In scope:

1. Phase/loop orchestration
2. Dispatch rules
3. Canonical state derivation
4. Tool-backed transactional mutations
5. Artifact verification + retry policy
6. Verification gate + evidence capture
7. Session lock + crash recovery semantics

Out of scope:

- Reusing GSD2 SQLite as canonical store in Exaskill
- Reproducing every prompt string verbatim

## Ground Truth Contracts From GSD2

### 1) Single canonical state per lane

- GSD2 derives state DB-first with fallback paths and computes a phase for dispatch.
- Port requirement: Exaskill must expose exactly one canonical state source per lane (recommended: Beads).

References:

- `src/resources/extensions/gsd/state.ts:195`
- `src/resources/extensions/gsd/state.ts:279`

### 2) Deterministic dispatch by phase and guards

- GSD2 resolves the next unit by ordered rule evaluation.
- Port requirement: Exaskill dispatch must be deterministic, ordered, and guard-driven.

References:

- `src/resources/extensions/gsd/auto-dispatch.ts:134`
- `src/resources/extensions/gsd/auto-dispatch.ts:734`

### 3) One unit per fresh worker session

- GSD2 runs each unit in a fresh sub-session and waits for terminal event.
- Port requirement: Exaskill runtime must isolate unit execution context and capture a terminal result.

References:

- `src/resources/extensions/gsd/auto/run-unit.ts:24`
- `src/resources/extensions/gsd/auto/resolve.ts:49`

### 4) Tool-backed mutations are transactional

- Planning/completion update structured state first, then render artifacts.
- Port requirement: mutation tools must be atomic with rollback on failure.

References:

- `src/resources/extensions/gsd/tools/plan-milestone.ts:168`
- `src/resources/extensions/gsd/tools/plan-slice.ts:125`
- `src/resources/extensions/gsd/tools/complete-task.ts:129`

### 5) Artifact + verification gates control forward progress

- Missing artifacts or failed verification trigger retries/pause paths.
- Port requirement: Exaskill loop must gate transitions on explicit artifact and verification outcomes.

References:

- `src/resources/extensions/gsd/auto-artifact-paths.ts:22`
- `src/resources/extensions/gsd/auto-post-unit.ts:433`
- `src/resources/extensions/gsd/auto-verification.ts:49`

### 6) Session lock enforces single active orchestrator per lane

- GSD2 has a hard lock plus lock metadata for recovery.
- Port requirement: Exaskill needs a lock primitive with ownership checks and recovery metadata.

References:

- `src/resources/extensions/gsd/session-lock.ts:252`
- `src/resources/extensions/gsd/session-lock.ts:422`
- `src/resources/extensions/gsd/crash-recovery.ts:34`

## Exaskill Adapter Interfaces (Concrete)

Define these as the minimum integration contract:

```ts
type LaneId = string
type UnitType =
  | 'guided-discuss-milestone'
  | 'plan-milestone'
  | 'research'
  | 'plan-slice'
  | 'execute-task'
  | 'complete-slice'
  | 'complete-milestone'

interface OrchestrationState {
  laneId: LaneId
  phase:
    | 'pre-planning'
    | 'needs-discussion'
    | 'planning'
    | 'evaluating-gates'
    | 'executing'
    | 'summarizing'
    | 'replanning-slice'
    | 'validating-milestone'
    | 'completing-milestone'
    | 'complete'
    | 'blocked'
  activeMilestoneId?: string
  activeSliceId?: string
  activeTaskId?: string
  flags: Record<string, boolean>
}

interface DispatchAction {
  unitType: UnitType
  unitId?: string
  reason: string
}

interface ExaskillGsdAdapter {
  acquireSessionLock(projectId: string, laneId: LaneId): Promise<void>
  assertLockOwnership(projectId: string, laneId: LaneId): Promise<void>
  releaseSessionLock(projectId: string, laneId: LaneId): Promise<void>

  deriveState(laneId: LaneId): Promise<OrchestrationState>
  resolveDispatch(state: OrchestrationState): Promise<DispatchAction>

  buildUnitPrompt(action: DispatchAction, state: OrchestrationState): Promise<string>
  runUnitInFreshSession(action: DispatchAction, prompt: string): Promise<{ ok: boolean; outputRef?: string }>

  verifyExpectedArtifact(action: DispatchAction): Promise<{ ok: boolean; details?: string }>
  runVerificationGate(state: OrchestrationState): Promise<{ ok: boolean; retryable: boolean; details?: string }>
  recordVerificationEvidence(state: OrchestrationState, details: string): Promise<void>

  applyMutationTool(toolName: string, payload: unknown): Promise<{ ok: boolean; details?: string }>
  renderProjections(laneId: LaneId): Promise<void>

  writeRecoveryMetadata(info: Record<string, unknown>): Promise<void>
  readRecoveryMetadata(): Promise<Record<string, unknown> | null>
}
```

## Migration Plan (Phased)

### Phase 0: Harness and invariants

Deliverables:

- Implement adapter stubs with no-op persistence.
- Implement reference loop shell (`start`, `tick`, `stop`) with lock acquire/release.
- Add invariant checks:
  - exactly one lock owner per lane,
  - no phase transition without dispatch action,
  - no completion without artifact + verification pass.

Exit criteria:

- Loop can run dry in simulation mode across all phases.

### Phase 1: State + dispatch parity

Deliverables:

- Port phase derivation rules into Exaskill state model.
- Port ordered dispatch table and guard predicates.
- Snapshot tests for known workflow scenarios.

Exit criteria:

- For the same synthetic state fixtures, Exaskill dispatch matches expected GSD2-equivalent unit type.

### Phase 2: Mutation tool parity

Deliverables:

- Implement transactional handlers for:
  - plan milestone,
  - plan slice,
  - complete task,
  - complete slice,
  - complete milestone.
- Add idempotency keys for retry-safe writes.

Exit criteria:

- Replaying the same completion tool request does not duplicate rows/records.

### Phase 3: Unit execution + artifact/verification pipeline

Deliverables:

- Wire fresh-session unit execution.
- Implement expected-artifact mapping and retry policy.
- Implement verification gate with evidence storage and retry exhaustion behavior.

Exit criteria:

- Failed verification transitions to retry/pause correctly.

### Phase 4: Recovery and operator UX

Deliverables:

- Write/read crash metadata.
- On restart, synthesize recovery task prompt with previous unit context.
- Add lock compromise detection and graceful pause path.

Exit criteria:

- Simulated crash mid-unit resumes with actionable recovery context and no double-commit of mutation tools.

## Test Matrix (Must Pass Before Production)

1. Locking
   - dual orchestrator start on same lane: second denied.
   - stale lock metadata present but no owner: recover + continue.

2. Dispatch
   - each phase fixture maps to exactly one action.
   - guard failures route to pause/blocked path, not default execute.

3. Mutation atomicity
   - injected failure at mid-transaction leaves no partial state.
   - repeated request with same idempotency key is a no-op.

4. Artifact verification
   - expected artifact missing triggers retry path.
   - artifact present allows verification gate step.

5. Verification gate
   - failing command records evidence and retries up to policy limit.
   - exhaustion pauses lane with operator-visible reason.

6. Recovery
   - crash after tool write but before projection render repairs projections on restart.
   - crash before tool write does not mark task complete.

## Recommended Implementation Order

1. State derivation + dispatch table
2. Mutation tools with idempotency
3. Locking + crash metadata
4. Unit prompt/execution wrapper
5. Artifact verification + verification gate
6. Projection rendering and stale-repair jobs

Rationale: this order gives early deterministic behavior and minimizes risk of hidden state drift.

## Risks and Mitigations

- Dual source of truth risk (Beads + imported SQLite)
  - Mitigation: keep Beads canonical; import once; never dual-write canonically.

- Non-deterministic dispatch from ad hoc conditionals
  - Mitigation: single ordered dispatch table with test fixtures.

- Duplicate completion events under retries
  - Mitigation: idempotency keys on all mutation tools.

- Silent verification bypass
  - Mitigation: hard transition gate requiring explicit verification result object.

## Definition of Done

Port is considered complete when:

1. Exaskill can run full milestone lifecycle with deterministic phase transitions.
2. All mutation tools are transactional and idempotent.
3. Single-lane lock semantics prevent concurrent orchestrators.
4. Artifact + verification failures are observable and retry/pause correctly.
5. Recovery after simulated crashes is safe and operator-guided.
6. Projection artifacts can be regenerated from canonical Exaskill state.
