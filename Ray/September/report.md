# September 2026 — Ray

**Reporting period:** 2026-09-01 to 2026-10-10

## Summary

Code Review: 2

The reporting period focused on Ray Core scheduling, recovery, and actor retry semantics, with additional correctness reviews covering GCS restart reconciliation and Ray Data error-contract behavior. One Placement Group recovery fix is under upstream review, while a separate actor restart / per-call retry fix is open and ready for review.

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

### Ray #44719 / PR #66818 — Actor restart × per-call retry policy mismatch

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
- Pull request: [ray-project/ray#66818](https://github.com/ray-project/ray/pull/66818)

**Status**

Open; ready for review.

---

## Code Reviews

### [Ray #66326 — GCS restart reconciliation gate](https://github.com/ray-project/ray/pull/66326)

- Reviewed the restart reconciliation path and found that the reconciliation gate could be cleared after a non-OK `ReleaseUnusedActorWorkers` result.
- Constructed a deterministic failure path where the node remains alive, cleanup is not established as successful, but a subsequent scheduling retry sends a new worker lease.
- Asked whether reconciliation should remain active until cleanup succeeds or node removal is confirmed.

**Status:** Review comment posted; upstream discussion pending.

---

### [Ray #66878 — JSONL exception subtype preservation](https://github.com/ray-project/ray/pull/66878)

- Reviewed the new JSON/JSONL error-context handling and noticed that the broad `ValueError` wrapper also catches `UnicodeDecodeError`.
- Reproduced the behavior in both chunk-size estimation and direct JSONL reading: invalid UTF-8 that previously raised `UnicodeDecodeError` becomes a plain `ValueError`.
- Asked whether this normalization is intentional or whether the original subtype should be preserved consistently with the Arrow fallback.

**Status:** Review comment posted; upstream discussion pending.
