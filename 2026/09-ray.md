# 2026-09 — Ray

## Summary

September work focused on Ray Core scheduling and actor retry semantics. One Placement Group recovery fix is under upstream review, while a separate actor restart / per-call retry mismatch has a local implementation and focused tests ready for a Draft PR.

## Contributions

### Ray #65147 / PR #65970 — Placement Group recovery retry starvation

**Problem**

Repeated failed Placement Group recovery attempts could retain highest scheduling priority and delay unrelated pending placement groups.

**Work**

- Traced the GCS recovery and pending-queue scheduling path.
- Identified the fresh retry-state reset after failed `RESCHEDULING` attempts.
- Reused the existing exponential backoff for failed recovery retries while preserving top priority for fresh recovery.
- Added a RED → GREEN manager-level regression.
- Published a technical write-up explaining the failure mechanism and scheduling-policy tradeoff.

**Evidence**

- Issue: [ray-project/ray#65147](https://github.com/ray-project/ray/issues/65147)
- Pull request: [ray-project/ray#65970](https://github.com/ray-project/ray/pull/65970)
- Technical write-up: [Recovery Is Not Progress](https://medium.com/@tpa33.36.09.013/recovery-is-not-progress-5975d5aa4153)

**Status**

Under upstream review. The current discussion is whether strict recovery priority should remain across every failed retry or yield during backoff.

---

### Ray #44719 — Actor restart × per-call retry policy mismatch

**Problem**

Restart-time task disposition and retry accounting can use different `max_task_retries` policies when the actor handle default and per-call override differ, causing inconsistent fast-fail / buffering behavior.

**Work**

- Traced the restart and retry-policy paths.
- Identified the handle-level vs task-level policy mismatch.
- Reproduced both mixed-policy cases.
- Implemented a queue-owned fix preserving ordering semantics.
- Added focused regression coverage.

**Evidence**

- Issue: [ray-project/ray#44719](https://github.com/ray-project/ray/issues/44719)

**Status**

Implementation and focused tests are ready locally; Draft PR not yet opened.
