---
name: pipeline-orchestration
description: Use when building a multi-step pipeline where each step depends on the prior result — async job orchestration, ingestion pipelines, ordered workflows with failure modes. Triggers on: pipeline pattern, step result, state machine, fail forward, ordered steps, StepResult.
---

# Pipeline Orchestration

## Overview

A pipeline is a sequence of stages where the output of one stage becomes the input of the next.
Each stage can change the data type — that's what distinguishes Pipeline from Chain of Responsibility.

**Core principle:** The pipeline governs HOW data flows. A state machine governs WHAT transitions are legal. Use both together.

## Pipeline vs Chain of Responsibility vs Decorator

| Pattern | Type contract | Primary question | Early exit |
|---|---|---|---|
| **Pipeline** | Each stage can change `T` | How is data transformed? | Failure case |
| **CoR** | Same type in/out per handler | Which handler owns this? | Normal case |
| **Decorator** | Same interface wraps same type | What behaviour to add? | Rarely |

**Use Pipeline when:** data accumulates or changes shape across stages.
**Use CoR when:** you're routing to one handler that owns the request.

## StepResult\<T\> — Typed Step Outcomes

Don't use exceptions for expected failures between pipeline steps. Exceptions are for unexpected failures. Expected failures (a job fails, an API returns an error) are part of the normal flow.

```java
// Each stage returns a typed result — T changes per stage
sealed interface StepResult<T> permits StepResult.Ok, StepResult.Err {
    record Ok<T>(T value)        implements StepResult<T> {}
    record Err<T>(String reason) implements StepResult<T> {}
}
```

**Why sealed:** compiler enforces exhaustiveness — add a new case and forget to handle it, it won't compile. No `default` needed.

```java
// ✅ Typed, exhaustive, intent is explicit
return switch (runAssets(order)) {
    case StepResult.Ok<AssetData>  ok  -> runMetadata(order, ok.value());
    case StepResult.Err<AssetData> err -> failOrder(order, err.reason());
};

// ❌ Exceptions as control flow — hides which step failed, why, and what data survived
try {
    AssetData assets = runAssets(order);
    return runMetadata(order, assets);
} catch (AssetGenerationException e) {
    return failOrder(order, e.getMessage());
}
```

## State Machine Discipline

Ordered pipelines have strict transition rules. Break one → unrecoverable.

```
PENDING → STEP_1_QUEUED → STEP_1_DONE → STEP_2_QUEUED → STEP_2_DONE → SUBMITTED
                               ↓ FAILED                      ↓ FAILED
                          FAILED (no data)            FAILED (step-1 data included)
```

**Rules:**
- **Advance only on success** — never proceed to the next step if the current one failed
- **Fail forward, not backward** — submit what you have and stop; don't retry earlier steps
- **No partial recovery** — once you've committed an outcome, don't amend it
- **Each step idempotent where possible** — safe to call twice if unsure whether first call landed

## Failure Modes — Shape Depends on Which Step Failed

A pipeline with N steps can fail at any step, and the failure payload differs at each:

```java
// Step 1 fails → no data from any step
FailedOrder failedAtAssets(OrderId id, String reason) {
    return new FailedOrder(id, reason, null, null); // no assets, no metadata
}

// Step 2 fails → include data from step 1, exclude step 2
FailedOrder failedAtMetadata(OrderId id, AssetData assets, String reason) {
    return new FailedOrder(id, reason, assets, null); // assets included, no metadata
}

// All succeed → full payload
ShippableOrder success(OrderId id, AssetData assets, Metadata metadata) {
    return new ShippableOrder(id, assets, metadata);
}
```

Use sealed types to make the three outcomes unambiguous:

```java
sealed interface OrderOutcome permits ShippableOrder, FailedOrder {}
```

## Composing Polling Inside a Pipeline Stage

**REQUIRED SKILL:** See `java-patterns:polling` for the full polling implementation.

Each pipeline stage that triggers an async job follows this sub-pattern — the orchestrator never sees the polling loop, it just gets a `StepResult`:

```java
StepResult<AssetData> runAssetGeneration(Order order) {
    String jobId = client.queueAssetJob(order.id());   // POST — queues the job
    JobStatus status = pollUntilDone(jobId);           // polls until SUCCESS/FAILED
    if (status == JobStatus.FAILED) {
        return new StepResult.Err<>("Asset generation failed");
    }
    return new StepResult.Ok<>(client.getAssetData(order.id()));
}
```

## Orchestrator — Wires Stages Together

```java
OrderOutcome process(Order order) {
    client.take(order.id());  // must happen first — marks order as in-progress

    return switch (runAssetGeneration(order)) {
        case StepResult.Ok<AssetData> assets ->
            switch (runMetadataGeneration(order)) {
                case StepResult.Ok<Metadata> meta ->
                    new ShippableOrder(order.id(), assets.value(), meta.value());
                case StepResult.Err<Metadata> err ->
                    failedAtMetadata(order.id(), assets.value(), err.reason());
            };
        case StepResult.Err<AssetData> err ->
            failedAtAssets(order.id(), err.reason());
    };
}
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Exceptions for expected step failures | Use `StepResult<T>` — exceptions hide which step failed |
| Proceeding after a failed step | Check `StepResult` before advancing; sealed types force this |
| Same failure payload regardless of which step failed | Shape the payload per step — carry what you have |
| Retrying an earlier step after a later step fails | Fail forward — submit what you have and stop |
| Confusing Pipeline with CoR | CoR: same type, one handler owns it. Pipeline: type changes, all stages run |
| Mixing polling logic into the orchestrator | Polling is a stage-internal detail — return `StepResult` from the stage |
