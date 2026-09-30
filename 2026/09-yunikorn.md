# 2026-09 — Apache YuniKorn

## Summary

September work covered deletion recovery, autoscaling lifecycle correctness, flaky-test reliability, and preemption review. Two fixes were merged, one k8shim fix remains under review, and one Core fix is ready locally.

## Contributions

### YUNIKORN-3382 / PR #1093 — Failed Pod deletion re-drive with UID fencing

**Problem**

After scheduler-originated release state had already been established, a failed Kubernetes Pod DELETE could return without any component retaining responsibility for completing the deletion. Delayed retries also had to remain bound to the original Pod identity rather than a same-name replacement.

**Work**

- Traced the failed release path and identified the loss of deletion responsibility after the initial Kubernetes DELETE failure.
- Added a single Task-local retry owner with retry-until-resolved semantics using Kubernetes `wait.Backoff`.
- Added UID-fenced deletion and identity-aware Conflict reconciliation to protect replacement Pods.
- Added regression coverage for ownership, retry lifecycle, UID fencing, Conflict handling, and completion boundaries.

**Evidence**

- Issue: [YUNIKORN-3382](https://issues.apache.org/jira/browse/YUNIKORN-3382)
- Pull request: [apache/yunikorn-k8shim#1093](https://github.com/apache/yunikorn-k8shim/pull/1093)

**Status**

Reviewer feedback addressed and all CI checks pass; awaiting upstream re-review or approval.

---

### YUNIKORN-3448 — Autoscaling advertisement lifecycle

**Problem**

An ask that had already advertised scale-up demand could lose policy headroom without withdrawing the previous advertisement, while `scaleUpTriggered` then suppressed re-advertisement after headroom returned.

**Work**

- Reproduced the lifecycle on current master and traced it through outstanding-request collection.
- Implemented a local Core fix using the existing `SKIPPED` transition to withdraw stale demand and re-arm a later `FAILED` advertisement.
- Added deterministic coverage for `FAILED` → `SKIPPED` → `FAILED`, duplicate suppression, accounting, and policy-headroom boundaries.

**Evidence**

- Issue: [YUNIKORN-3448](https://issues.apache.org/jira/browse/YUNIKORN-3448)

**Status**

Issue claimed and investigation posted upstream. Local implementation and focused tests are ready; PR not yet opened.

---

### YUNIKORN-3470 — Webservice shutdown flake

**Problem**

Incomplete finite HTTP response-body cleanup could leave connections crossing test-server lifetimes, producing shutdown timeouts and follow-up EOF failures.

**Work**

- Traced the response and connection lifecycle, then applied a test-only fix that consumes affected finite response bodies before close.
- Stress validation improved from 37/500 failures on the clean control to 0/500 on the patched version.

**Evidence**

- Issue: [YUNIKORN-3470](https://issues.apache.org/jira/browse/YUNIKORN-3470)
- Pull request: [apache/yunikorn-core#1168](https://github.com/apache/yunikorn-core/pull/1168)

**Status**

Merged upstream.

---

### YUNIKORN-3468 / PR #1105 — Quota-preemption E2E readiness flake

**Problem**

The quota-preemption E2E test could fail during setup because the initial Pod readiness timeout was too short.

**Work**

- Increased the readiness timeout from 5s to 30s.
- Validated the focused and quota-preemption E2E suites.

**Evidence**

- Pull request: [apache/yunikorn-k8shim#1105](https://github.com/apache/yunikorn-k8shim/pull/1105)

**Status**

Merged upstream.

---

### PR #1161 — Preemption victim scan tradeoff review

**Problem**

The current early `break` can stop victim scanning after one candidate exceeds the queue's guaranteed-resource bound, potentially skipping later viable candidates. Replacing it with unconditional `continue` improves completeness but may increase scan cost.

**Work**

- Reviewed the victim-selection path and discussed the correctness and efficiency tradeoff.
- Ran a focused microbenchmark: tens of microseconds for 10 candidates and about 1.8 ms for 1,000 candidates.
- Noted that the benchmark is not representative of production scheduler latency and that neither first-candidate termination nor unconditional full scanning is obviously sufficient.

**Evidence**

- Pull request: [apache/yunikorn-core#1161](https://github.com/apache/yunikorn-core/pull/1161)

**Status**

Provided technical review and benchmark evidence; no implementation submitted.
