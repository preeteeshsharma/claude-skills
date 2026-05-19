---
name: java-api-client
description: Use when building a Java HTTP client that calls external APIs — especially when APIs are unreliable, need retry logic, or are part of a multi-step pipeline. Triggers on: OkHttp, retry backoff, polling async jobs, HTTP status code handling, abstract web client, pipeline orchestration.
---

# Java API Client — Resilient HTTP Integration

## Overview

External APIs fail. The client's job is to handle that gracefully without leaking retry logic into business code.

**Core principle:** Retry logic lives in the HTTP layer. Orchestration logic lives in the service layer. Never mix them.

## Pattern: Abstract WebClient

Centralise all HTTP + retry logic in one abstract class. Callers get typed methods, not raw HTTP.

```java
abstract class ApiClient {

    private static final int  MAX_RETRY_ATTEMPTS = 3;
    private static final long RETRY_BASE_MS      = 1_000;

    protected abstract <T> T execute(String url, String method,
                                     String body, Class<T> responseType);

    protected <T> T get(String url, Class<T> type) {
        return withRetry(() -> execute(url, "GET", null, type));
    }

    // Non-idempotent POST: if server accepted but response dropped,
    // retrying may cause a duplicate action (e.g., double-taking an order).
    // Call this only when the endpoint is safe to call twice.
    protected <T> T post(String url, String body, Class<T> type) {
        return withRetry(() -> execute(url, "POST", body, type));
    }

    private <T> T withRetry(Callable<T> call) {
        Exception last = null;
        for (int attempt = 0; attempt < MAX_RETRY_ATTEMPTS; attempt++) {
            try {
                return call.call();
            } catch (RetryableException e) {
                last = e;
                long backoff = (long) Math.pow(2, attempt) * RETRY_BASE_MS;
                long jitter  = (long) (Math.random() * RETRY_BASE_MS);
                sleep(backoff + jitter);
            } catch (Exception e) {
                throw new ApiException("Non-retryable: " + e.getMessage(), e);
            }
        }
        throw new ApiException("Exhausted retries", last);
    }

    private void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Concrete OkHttp Implementation

```java
class OkHttpApiClient extends ApiClient {

    private static final int TIMEOUT_SECONDS = 10;

    private final OkHttpClient http;
    private final Gson gson;
    private final String baseUrl;

    OkHttpApiClient(String baseUrl) {
        this.baseUrl = baseUrl;
        this.http = new OkHttpClient.Builder()
            .connectTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .readTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .writeTimeout(TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .build();
        this.gson = new Gson();
    }

    @Override
    protected <T> T execute(String path, String method, String body, Class<T> type) {
        Request.Builder builder = new Request.Builder().url(baseUrl + path);

        if ("POST".equals(method)) {
            RequestBody rb = body != null
                ? RequestBody.create(body, MediaType.get("application/json"))
                : RequestBody.create("", MediaType.get("application/json"));
            builder.post(rb);
        }

        try (Response response = http.newCall(builder.build()).execute()) {
            return handleResponse(response, type);  // delegates to status-code switch
        } catch (IOException e) {
            // Network failure is retryable — server may not have received the request
            throw new RetryableException("Network error: " + e.getMessage(), e);
        }
    }

    private <T> T parse(ResponseBody body, Class<T> type) {
        try {
            return gson.fromJson(body.string(), type);
        } catch (IOException e) {
            throw new ApiException("Failed to parse response body", e);
        }
    }
}
```

## HTTP Status Code Handling

Decide **in the HTTP layer** — callers should never see raw status codes.

```java
<T> T handleResponse(Response response, Class<T> type) {
    return switch (response.code()) {
        case 200, 201, 204      -> parse(response.body(), type);
        case 429                -> throw new RetryableException("rate limited");
        case 400, 404, 422      -> throw new ApiException("client error: " + response.code());
        case 500, 502, 503, 504 -> throw new RetryableException("server error: " + response.code());
        default                 -> throw new ApiException("unexpected: " + response.code());
    };
}
```

**4xx = your fault → fail fast, don't retry. 5xx = their fault → retry with backoff.**

## Polling for Async Job Completion

**REQUIRED SKILL:** See `java-patterns:polling` for the full polling pattern, terminal state handling, timeout behaviour, and how polling composes with retry.

Polling and retry are different things that compose — each status check call goes through `withRetry` internally; the sleep-and-loop is a separate concern.

## Pipeline Step Outcomes

**REQUIRED SKILL:** See `java-patterns:pipeline-orchestration` for the full pipeline pattern, state machine discipline, and failure mode shapes.

## Pipeline Step Outcomes — StepResult vs Exceptions

Pipeline and Chain of Responsibility look similar but differ on type flexibility: CoR handlers must share one return type; Pipeline stages can change the output type at each step. Use `StepResult<T>` to express this.

```java
// ✅ Typed result — each stage can produce a different T
sealed interface StepResult<T> permits StepResult.Ok, StepResult.Err {
    record Ok<T>(T value)        implements StepResult<T> {}
    record Err<T>(String reason) implements StepResult<T> {}
}

// Compiler enforces both cases — no default needed
return switch (runAssets(order)) {
    case StepResult.Ok<AssetData>  ok  -> runMetadata(order, ok.value());
    case StepResult.Err<AssetData> err -> failOrder(order, err.reason());
};

// ❌ Exceptions as control flow — hides which step failed and why
try {
    AssetData assets = runAssets(order);
    return runMetadata(order, assets);
} catch (AssetGenerationException e) {
    return failOrder(order, e.getMessage());
}
```

## Named Constants — Never Inline

```java
// ✅ Named — readable, tunable, no magic numbers
private static final int  MAX_RETRY_ATTEMPTS = 3;
private static final long RETRY_BASE_MS      = 1_000;
private static final int  MAX_POLL_ATTEMPTS  = 30;
private static final long POLL_INTERVAL_MS   = 2_000;

// ❌ Inline — impossible to reason about or tune
Thread.sleep((long) Math.pow(2, attempt) * 1000);
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Retry on 4xx | 4xx won't change — fail fast |
| No jitter on backoff | Thundering herd on outage recovery — always add jitter |
| Retry non-idempotent POSTs blindly | Flag the assumption; `POST /take` may double-take |
| Retry + pipeline logic in same class | Abstract WebClient handles retry; orchestrator handles pipeline |
| Acting on PENDING/IN_PROGRESS status | Only act on SUCCESS or FAILED |
| Hardcoded retry counts and intervals | Extract to named constants |
