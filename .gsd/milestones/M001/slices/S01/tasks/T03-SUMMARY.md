---
id: T03
parent: S01
milestone: M001
key_files:
  - src/resources/extensions/gsd/prompts/plan-milestone.md
  - src/resources/extensions/gsd/prompts/guided-plan-milestone.md
  - src/resources/extensions/gsd/prompts/plan-slice.md
  - src/resources/extensions/gsd/prompts/replan-slice.md
  - src/resources/extensions/gsd/prompts/reassess-roadmap.md
  - src/resources/extensions/gsd/auto-post-unit.ts
  - src/resources/extensions/gsd/tests/prompt-contracts.test.ts
  - src/resources/extensions/gsd/tests/rogue-file-detection.test.ts
key_decisions:
  - Treat `gsd_plan_milestone` and future DB-backed planning tools as the planning source of truth in prompts, while preserving markdown templates only as output-shaping guidance rather than manual write instructions.
  - Extend rogue-file detection by checking for planning-state presence in milestone and slice DB rows instead of inventing a separate planning completion status model just for enforcement.
  - Keep verification honest by recording both the passing repo-local TS harness command and the still-failing bare `node --test` rogue-detection command, since the latter reflects an existing test-runtime mismatch rather than a T03 implementation bug.
duration: ""
verification_result: mixed
completed_at: 2026-03-23T15:39:21.178Z
blocker_discovered: false
---

# T03: Migrate planning prompts to DB-backed tool guidance and extend rogue detection to roadmap/plan artifacts

**Migrate planning prompts to DB-backed tool guidance and extend rogue detection to roadmap/plan artifacts**

## What Happened

I executed the T03 contract against the current repo state instead of the planner snapshot. First I verified the slice plan’s observability section already contained the required failure-path coverage, then read the five planning prompts, `auto-post-unit.ts`, and the existing prompt/rogue test files. The root gap was straightforward: milestone and adjacent planning prompts still contained direct file-writing language, while rogue-file detection only covered execute-task and complete-slice summary artifacts. I updated `plan-milestone.md` and `guided-plan-milestone.md` so they now route milestone planning through `gsd_plan_milestone` and explicitly forbid manual roadmap writes. I also updated `plan-slice.md`, `replan-slice.md`, and `reassess-roadmap.md` so those planning-era prompts consistently treat DB-backed tool state as the source of truth and stop implying that direct roadmap/plan edits are acceptable. On the enforcement side, I extended `detectRogueFileWrites()` in `src/resources/extensions/gsd/auto-post-unit.ts` to flag direct `ROADMAP.md` writes for `plan-milestone` when no milestone planning state exists in DB, and direct slice `PLAN.md` writes for `plan-slice` / `replan-slice` when no matching slice planning state exists. I preserved the existing execute-task and complete-slice logic. I then expanded `prompt-contracts.test.ts` with explicit assertions that the milestone and adjacent planning prompts reference the tool path and forbid manual roadmap/plan writes, and expanded `rogue-file-detection.test.ts` with positive/negative cases for roadmap and slice-plan rogue detection. The first verification run exposed two concrete issues only: my initial prompt assertions were too broad and matched the new explicit prohibition text, and I incorrectly imported a non-existent `updateMilestone` export. I fixed those specific problems by tightening the prompt assertions to test for the explicit prohibition language and switching the DB setup to `upsertMilestonePlanning()`. After that, the adapted task-level test command passed cleanly.

## Verification

I ran the task-level verification under the repository’s actual TypeScript harness: `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/prompt-contracts.test.ts src/resources/extensions/gsd/tests/rogue-file-detection.test.ts`, and all 32 assertions passed. I also ran the literal slice-plan verification pieces individually. `node --test src/resources/extensions/gsd/tests/prompt-contracts.test.ts` now passes directly. `node --test src/resources/extensions/gsd/tests/rogue-file-detection.test.ts` still fails before reaching the test logic because `auto-post-unit.ts` imports `.js` sibling modules from TypeScript sources and direct `node --test` cannot resolve them without the repo’s resolver import; this is the same repo-local harness mismatch previously documented in T02, not a regression introduced by this task. Observability expectations for T03 are now met: prompt regressions fail explicitly in `prompt-contracts.test.ts`, and rogue roadmap/plan bypasses are surfaced immediately by `detectRogueFileWrites()` and its regression tests.

## Verification Evidence

| # | Command | Exit Code | Verdict | Duration |
|---|---------|-----------|---------|----------|
| 1 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/prompt-contracts.test.ts src/resources/extensions/gsd/tests/rogue-file-detection.test.ts` | 0 | ✅ pass | 519ms |
| 2 | `node --test src/resources/extensions/gsd/tests/prompt-contracts.test.ts` | 0 | ✅ pass | 107ms |
| 3 | `node --test src/resources/extensions/gsd/tests/rogue-file-detection.test.ts` | 1 | ❌ fail | 103ms |


## Deviations

Used the repository’s existing TypeScript resolver harness for the authoritative task-level verification because `rogue-file-detection.test.ts` cannot run truthfully under bare `node --test` in this source tree. No functional deviation from the task scope otherwise.

## Known Issues

Direct `node --test src/resources/extensions/gsd/tests/rogue-file-detection.test.ts` still fails with `ERR_MODULE_NOT_FOUND` on `.js` sibling imports from TypeScript sources (`auto-post-unit.ts` → `state.js`) unless the repo resolver import is used. This harness mismatch predates this task and remains for T04 to account for when running the integrated slice suite. No T03-specific functional failures remain under the repo’s actual TS harness.

## Files Created/Modified

- `src/resources/extensions/gsd/prompts/plan-milestone.md`
- `src/resources/extensions/gsd/prompts/guided-plan-milestone.md`
- `src/resources/extensions/gsd/prompts/plan-slice.md`
- `src/resources/extensions/gsd/prompts/replan-slice.md`
- `src/resources/extensions/gsd/prompts/reassess-roadmap.md`
- `src/resources/extensions/gsd/auto-post-unit.ts`
- `src/resources/extensions/gsd/tests/prompt-contracts.test.ts`
- `src/resources/extensions/gsd/tests/rogue-file-detection.test.ts`
