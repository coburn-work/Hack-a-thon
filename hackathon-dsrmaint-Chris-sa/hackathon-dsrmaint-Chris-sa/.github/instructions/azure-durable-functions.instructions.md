---
description: "Use when writing Azure Durable Functions orchestrators, activities, triggers, or the SFC.Orchestrations project structure"
---

# Azure Durable Functions Conventions

## When to Use (§9.1)

- Multi-step workflows requiring orchestration
- Long-running processes (minutes to hours)
- Human approval steps
- Retry/compensation logic for distributed transactions

Do **not** use for simple event handlers (<5 seconds) or pure data transformations.

## Project Naming (§9.5)

- **`SFC.Orchestrations.{Name}`** — not `SFC.Services.{Name}`
- No `.Contracts`, `.Nuget`, or `.Repository` sub-projects
- Namespace: `SFC.Orchestrations.{Name}.Orchestrators`, `.Activities`, `.Triggers`, `.Models`, `.Functions`

```
SFC.Orchestrations.{Name}/
    Orchestrators/     <- orchestrator classes
    Activities/        <- activity functions
    Triggers/          <- Event Grid trigger functions
    Models/            <- activity input/output DTOs
    Functions/         <- standalone functions
    Program.cs
    host.json
    local.settings.json
```

## Orchestrator Rules (§9.2)

- Orchestrators **must be deterministic**: no `DateTime.Now`, no `Random`, no direct HTTP calls
- Use `context.CurrentUtcDateTime` instead of `DateTime.UtcNow`
- Use `context.CallActivityAsync()` for all non-deterministic operations
- Use `nameof()` for activity function names — never string literals

## Activity Pattern (§9.3)

- Activities perform all non-deterministic operations
- Activities may access the orchestration's own database directly; they call microservices via **NuGet gateway packages** — never direct HTTP access to another service's database (Adage Oracle or a microservice's Azure SQL)
- **Isolated Worker binding:** Use `[ActivityTrigger] T input` with the typed input directly — do **not** use `IDurableActivityContext` with `context.GetInput<T>()`
- Check `result.IsNotOk()` on gateway results
- Throw generic `Exception` for retriable failures (retry policy applies)
- Throw a scenario-specific terminating exception for non-retriable failures (orchestration fails immediately, alert fires) — see §18

## Retry Configuration (§9.4)

> **Isolated Worker Model:** The in-process `RetryOptions`/`CallActivityWithRetryAsync` API does not exist. Use `TaskOptions`/`TaskRetryOptions`/`RetryPolicy` as a **third argument** to `context.CallActivityAsync`.

```csharp
// Build once, reuse for every activity call in the orchestration.
var taskOptions = new TaskOptions(
    new TaskRetryOptions(
        new RetryPolicy(
            maxNumberOfAttempts: 5,
            firstRetryInterval: TimeSpan.FromSeconds(5),
            backoffCoefficient: 2.0,
            maxRetryInterval: TimeSpan.FromMinutes(5)),
        // Optional: exclude non-retriable exceptions from the retry loop.
        retryDetails => !retryDetails.IsCausedBy<SomeTerminatingException>()));

// ⚠️ Argument order: (activityName, input, options) — NOT (name, options, input).
var result = await context.CallActivityAsync<MyResult>(
    nameof(MyActivities.DoWork),
    input,
    taskOptions);
```

## Alerting & Non-Retriable Failures (§18)

- Define scenario-specific terminating exceptions (e.g., `TransferOrderNotFoundTerminatingException`); more than one per orchestrator is valid and preferred when distinct scenarios require different alert resolution steps
- Each exception should be named precisely enough that a Logic Monitor alert can link directly to a resolution procedure
- Include `ActivityName` and `CorrelationId` properties on every terminating exception
- Non-retriable: gateway returns `NotFound` for expected resource, business rule violation, data integrity error
- Retriable (transient): HTTP 500/503, timeout, throttled → throw generic `Exception`

## Management Endpoints (§18.3)

Orchestrator projects may expose management endpoints using the `DurableTaskClient` API (not the legacy `IDurableOrchestrationClient`). Use these when operational tooling or on-call runbooks require direct instance control:

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/orchestrations/failed` | GET | List failed instances |
| `/api/orchestrations/{instanceId}` | GET | Detailed status with history |
| `/api/orchestrations/{instanceId}/restart` | POST | Restart with original input |
| `/api/orchestrations/{instanceId}/terminate` | POST | Terminate stuck orchestration |

Use `[DurableClient] DurableTaskClient client` and `HttpRequestData req` in function signatures.

## Zero Downtime (§9.6)

- PROD only — DEV/QA/UAT do not require zero downtime
- **Warning:** Zero downtime strategies (deployment slots) are **not currently safe** for Azure Function Apps at Shamrock

## Cross-References

- **NuGet gateway:** Activities must call microservices via NuGet gateways — never direct DB access. See `csharp-conventions` instructions for the gateway pattern.
- **Event Grid:** Orchestrators are often triggered by Event Grid events via trigger functions. See `event-grid` instructions for event schema conventions (subject format, past-tense event types).
- **ARM deployment:** When adding Event Grid subscriptions or app settings for the orchestrator, update **all four** ARM parameter files (dev/qa/uat/prod) in `ARMTemplateParameterFiles/`.

## Prompt Cross-Reference

For scaffolding a new orchestrator project, use `/create-orchestrator`.
