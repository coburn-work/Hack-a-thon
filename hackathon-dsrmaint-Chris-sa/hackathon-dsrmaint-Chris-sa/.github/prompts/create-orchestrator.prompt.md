---
description: "Scaffold a new SFC.Orchestrations.* Azure Durable Functions project with orchestrator, activities, triggers, and ARM templates"
agent: "agent"
---

# Scaffold New Orchestrator

You are scaffolding a new Azure Durable Functions orchestrator project. Follow `azure-durable-functions` and `csharp-conventions` instructions.

## Required Information

Ask the user for any missing information:

1. **Orchestrator name** (e.g., `WarehouseTransfer`) — determines `SFC.Orchestrations.{Name}`
2. **Trigger event(s)** — which Event Grid events start the orchestrator
3. **Activities needed** — the workflow steps (each becomes an activity function)
4. **Microservices consumed** — which NuGet gateways are needed
5. **Human approval steps** — any `WaitForExternalEvent` requirements

## Project Structure

```
SFC.Orchestrations.{Name}/
    SFC.Orchestrations.{Name}.csproj
    Orchestrators/
        {Name}Orchestrator.cs
    Activities/
        {Name}Activities.cs
    Triggers/
        {EventName}Trigger.cs
    Models/
        {Activity}Request.cs
        {Activity}Response.cs
    Functions/
        OrchestrationManagementFunctions.cs
    Exceptions/
        {Name}TerminatingException.cs
    Program.cs
    host.json
    local.settings.json
    ARMTemplateParameterFiles/
        SFC.Orchestrations.{Name}.dev.parameters.V2.json
        SFC.Orchestrations.{Name}.qa.parameters.V2.json
        SFC.Orchestrations.{Name}.uat.parameters.V2.json
        SFC.Orchestrations.{Name}.prod.parameters.V2.json
```

## Checklist (Appendix A)

- [ ] Create `SFC.Orchestrations.{Name}` project — **not** `SFC.Services`
- [ ] No `.Contracts`, `.Nuget`, or `.Repository` sub-projects
- [ ] `Orchestrators/` with deterministic orchestrator (no `DateTime.Now`, use `context.CurrentUtcDateTime`)
- [ ] `Activities/` with activity functions using `nameof()` references
- [ ] `Triggers/` for Event Grid trigger functions
- [ ] `Models/` for activity input/output DTOs
- [ ] Register NuGet gateways in `Program.cs` (`services.Add{Name}ServiceGateway`)
- [ ] Register activity classes as scoped services
- [ ] `host.json` with Durable Task hub name
- [ ] App Registration auth with `DEFAULT_ACCESS` role
- [ ] Application Insights → shared `sfc-shared01-{env}-ai`
- [ ] ARM parameter files for all 4 environments
- [ ] Event Grid subscription(s) in ARM parameter files
- [ ] Custom terminating exception: `{Name}TerminatingException` with `ActivityName` and `CorrelationId`
- [ ] Management endpoints: GET failed, GET status, POST restart, POST terminate
- [ ] Repository naming: `sfc-orchestrations-{name}` (kebab-case)

## Key Patterns

### Orchestrator Skeleton

```csharp
[Function(nameof({Name}Orchestrator))]
public async Task<{Name}Result> {Name}Orchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var input = context.GetInput<{TriggerEvent}>();
    var startTime = context.CurrentUtcDateTime;  // Deterministic

    // Build retry options once, reuse for all activity calls
    var taskOptions = new TaskOptions(
        new TaskRetryOptions(
            new RetryPolicy(
                maxNumberOfAttempts: 5,
                firstRetryInterval: TimeSpan.FromSeconds(5),
                backoffCoefficient: 2.0,
                maxRetryInterval: TimeSpan.FromMinutes(5)),
            retryDetails => !retryDetails.IsCausedBy<{Name}TerminatingException>()));

    // Activity calls — argument order: (activityName, input, options)
    var result = await context.CallActivityAsync<TOutput>(
        nameof({Name}Activities.{ActivityMethod}), input, taskOptions);

    return new {Name}Result { Success = true };
}
```

### Terminating Exception

```csharp
public class {Name}TerminatingException : Exception
{
    public string ActivityName { get; }
    public string CorrelationId { get; }

    public {Name}TerminatingException(string activityName, string correlationId, string message)
        : base(message)
    {
        ActivityName = activityName;
        CorrelationId = correlationId;
    }
}
```

### Management Endpoints

Include `OrchestrationManagementFunctions.cs` with:
- `GET /api/orchestrations/failed` — list failed instances
- `GET /api/orchestrations/{instanceId}` — detailed status
- `POST /api/orchestrations/{instanceId}/restart` — restart with original input
- `POST /api/orchestrations/{instanceId}/terminate` — terminate stuck instance
