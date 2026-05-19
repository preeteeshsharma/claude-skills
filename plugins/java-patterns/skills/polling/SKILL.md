---
name: polling
description: Use when waiting for an async job or background process to reach a terminal state by repeatedly checking its status. Triggers on: poll until done, async job status, job queue, wait for completion, terminal state, SUCCESS FAILED status check.
---

# Polling for Async Job Completion

## Overview

Polling and retry are different things that compose. Retry handles a single call failing. Polling handles a job that's running and not yet done.

**Core principle:** Poll until terminal. Each status check is itself retried. Never act on intermediate state.

## The Pattern

```java
private static final int  MAX_POLL_ATTEMPTS = 30;
private static final long POLL_INTERVAL_MS  = 2_000;

// Poll until terminal (SUCCESS or FAILED) — sleep between checks
JobStatus pollUntilDone(String jobId) {
    for (int i = 0; i < MAX_POLL_ATTEMPTS; i++) {
        JobStatus status = client.getJobStatus(jobId); // retried internally by ApiClient
        if (status.isTerminal()) return status;
        sleep(POLL_INTERVAL_MS);
    }
    throw new TimeoutException("Job " + jobId + " did not reach terminal state");
}
```

## Terminal vs Intermediate States

```java
enum JobStatus {
    PENDING, IN_PROGRESS,   // intermediate — keep polling
    SUCCESS, FAILED;        // terminal — act on these

    public boolean isTerminal() {
        return this == SUCCESS || this == FAILED;
    }
}
```

**Never act on `PENDING` or `IN_PROGRESS`.** Only `SUCCESS` and `FAILED` are safe to hand to the orchestrator.

## Polling vs Retry — They Compose

```
pollUntilDone(jobId)
    └── for each attempt:
            client.getJobStatus(jobId)   ← this call is retried on network/5xx failure
                └── withRetry(() -> http.get("/jobs/" + jobId))
```

- **Retry** handles: network error, 5xx, transient failure on a single call
- **Polling** handles: job still running, not yet done
- Each poll attempt goes through retry internally — they compose cleanly

## Named Constants — Never Inline

```java
// ✅ Named — tunable, readable
private static final int  MAX_POLL_ATTEMPTS = 30;
private static final long POLL_INTERVAL_MS  = 2_000;

// ❌ Inline — opaque
for (int i = 0; i < 30; i++) {
    Thread.sleep(2000);
}
```

## Timeout Handling

When max attempts exhausted, the job didn't finish in time. The orchestrator decides what to submit — don't silently swallow the timeout.

```java
// ✅ Throw — let the orchestrator decide the outcome
throw new TimeoutException("Job " + jobId + " did not complete in time");

// ❌ Return a fake status — hides the real situation
return JobStatus.FAILED;
```

Flag to the interviewer: should a timed-out job be treated as `FAILED`? That's an explicit assumption worth naming.

## Inside a Pipeline Stage

Polling is a stage-internal detail. The orchestrator never sees the loop — it just gets a `StepResult`.

```java
StepResult<AssetData> runAssetGeneration(Order order) {
    String jobId = client.queueAssetJob(order.id());
    JobStatus status = pollUntilDone(jobId);          // ← polling here
    if (status == JobStatus.FAILED) {
        return new StepResult.Err<>("Asset job failed");
    }
    return new StepResult.Ok<>(client.getAssetData(order.id()));
}
```

**REQUIRED SKILLS:**
- `java-patterns:java-api-client` — for retry, HTTP client, `withRetry`
- `java-patterns:pipeline-orchestration` — for `StepResult<T>`, orchestrator pattern

## Common Mistakes

| Mistake | Fix |
|---|---|
| Acting on `PENDING`/`IN_PROGRESS` | Only act on terminal states |
| No timeout on the poll loop | Always set `MAX_POLL_ATTEMPTS` — jobs can hang |
| Silently returning `FAILED` on timeout | Throw `TimeoutException` — let the orchestrator decide |
| Hardcoding poll count and interval | Named constants only |
| Confusing polling with retry | Retry = single call failing. Polling = job not done yet |
| Exposing the poll loop to the orchestrator | Keep it inside the stage method — return `StepResult` |
