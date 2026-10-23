# Production Debugging and Root-Cause Analysis

Use this playbook for Java service incidents. The objective is not merely to restore service, but to preserve evidence, distinguish symptoms from causes, and prevent recurrence.

## Incident workflow

1. **Scope impact:** affected endpoints, tenants, regions, error rates, latency percentiles, and start time.
2. **Mitigate safely:** rollback, reduce traffic, disable a feature, or shed load if evidence supports it.
3. **Preserve evidence:** JFR, thread dumps, GC logs, heap data where safe, request traces, and deployment/configuration changes.
4. **Build hypotheses:** rank causes by evidence and testability.
5. **Find the earliest failure:** trace the first incorrect state or failed dependency, not only the final exception.
6. **Fix the invariant:** correct the source of the problem rather than hiding it in a catch block.
7. **Verify:** regression test, load test, failure injection, and post-deployment monitoring.
8. **Prevent:** add alerts, limits, runbook steps, and ownership.

## RCA template

### Summary
One or two sentences describing customer impact and the technical failure.

### Impact
Duration, affected users/requests, data integrity implications, and SLO effect.

### Timeline
Detection, mitigation, diagnosis, fix, rollout, and recovery.

### Root cause
Evidence-backed mechanism. Distinguish direct cause from contributing conditions.

### Why defenses failed
Missing limit, alert, test, idempotency, validation, or rollback mechanism.

### Corrective actions
Owner, due date, validation method, and priority.

## Common diagnosis patterns

### Q1. How do you distinguish a symptom from a root cause?

A symptom is the observed failure; a root cause explains the mechanism that produced it. “Heap OOM” is a symptom category; an unbounded cache retaining requests may be the root cause.

Use evidence to connect the mechanism to the impact. Avoid calling the first plausible hypothesis the root cause before ruling out alternatives.

### Q2. Why should an incident not begin with random JVM flag changes?

Random changes alter multiple variables and can erase evidence, mask the symptom, or worsen another constraint. Establish a baseline and gather data before tuning.

Change one variable at a time, preserve the previous configuration, and define rollback criteria.

### Q3. What should be done when a remote request times out after a write?

Treat the outcome as potentially unknown. Check idempotency keys, durable operation state, provider reconciliation, and whether the server may have committed the write before the response was lost.

Do not blindly retry a non-idempotent operation.

### Q4. What is the value of a thread dump?

It shows thread stacks and states at one moment, often including lock ownership/waiting. Repeated dumps show whether threads progress, block on the same path, or form a deadlock.

Correlate with queue, connection-pool, and request metrics; a dump alone does not measure CPU attribution.

### Q5. How do you write an actionable postmortem?

Focus on system conditions and corrective actions rather than blame. Include evidence, impact, contributing factors, detection gaps, and owners/dates for preventive work.

Actions should be testable and observable. “Be more careful” is not an engineering control.

## Example RCA: executor queue saturation

**Impact:** Request latency and timeout rate increased during downstream slowness.

**Evidence:** Queue depth rose continuously, workers were blocked on HTTP calls, CPU stayed low, and thread dumps showed waits in the client layer.

**Root cause:** Unbounded task submission combined with missing end-to-end deadlines allowed waiting tasks to accumulate.

**Mitigation:** Shed non-critical work and reduce ingress rate.

**Correction:** Add bounded concurrency, request deadlines, explicit rejection behavior, and downstream bulkheads.

**Prevention:** Alert on queue wait time and saturation; load-test dependency slowdown and verify recovery.

## Example RCA: duplicate payment after retry

**Impact:** Some requests generated duplicate provider-side operations.

**Evidence:** The first request timed out after reaching the provider; retry used a new operation identity.

**Root cause:** Timeout was treated as definitive failure and the operation lacked idempotency.

**Correction:** Persist a stable idempotency key and reconcile unknown outcomes before retrying.

**Prevention:** Add failure-injection tests for response loss after remote commit and monitor duplicate-operation rates.
