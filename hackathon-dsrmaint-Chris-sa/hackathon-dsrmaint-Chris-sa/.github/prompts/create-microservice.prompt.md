---
description: "Scaffold a new Shamrock Foods microservice with API, Contracts, Repository, and Nuget projects"
agent: "agent"
---

# Scaffold New Microservice

You are scaffolding a new Shamrock Foods microservice. Follow the `csharp-conventions` instructions for all C# conventions.

## Required Information

Ask the user for any missing information:

1. **Domain name** (e.g., `Purchasing`, `DeliveryLogistics`) — determines project naming
2. **Database type** — Oracle (Adage/PkMS) and/or Azure SQL
3. **Initial endpoints** — what API operations are needed
4. **Events consumed/published** — Event Grid integration needs

## Project Structure

```
SFC.Services.{Domain}.sln
SFC.Services.{Domain}.API/
    Controllers/
    Services/
    Startup.cs / Program.cs
    ARMTemplateParameterFiles/
        SFC.Services.{Domain}.Api.dev.parameters.V2.json
        SFC.Services.{Domain}.Api.qa.parameters.V2.json
        SFC.Services.{Domain}.Api.uat.parameters.V2.json
        SFC.Services.{Domain}.Api.prod.parameters.V2.json
SFC.Services.{Domain}.Contracts/
    Dtos/
    Enums/
SFC.Services.{Domain}.Repository/
    {Database}DbGateway.cs
    Models/
SFC.Services.{Domain}.Nuget/
    I{Domain}ServiceGateway.cs
    {Domain}ServiceGateway.cs
    ServiceCollectionExtensions.cs
```

## Checklist (Appendix A)

- [ ] Create `SFC.Services.{Domain}.API` project (.NET 6+)
- [ ] Create `SFC.Services.{Domain}.Contracts` project (DTOs, enums)
- [ ] Create `SFC.Services.{Domain}.Repository` project (database access via EF Core)
- [ ] Create `SFC.Services.{Domain}.Nuget` project (gateway interface + implementation)
- [ ] All controller actions return `Ok(result)` with `BaseResult<T>`
- [ ] Initialize `BaseResult<T>` with `ResultType = ResultTypes.BadRequest`
- [ ] Validation in service layer using `ValidationMessages` — no data annotations
- [ ] App Registration auth with `DEFAULT_ACCESS` role
- [ ] Application Insights → shared `sfc-shared01-{env}-ai`
- [ ] Create ARM parameter files for all 4 environments (dev/qa/uat/prod)
- [ ] NuGet gateway: `services.Add{Domain}ServiceGateway(Configuration)` extension method
- [ ] Repository naming: `sfc-services-{domain}` (kebab-case, compound words as single word)
- [ ] Azure SQL database: `{domain}-{env}-db`

## Key Patterns

- **NuGet gateway:** Consumers inject `I{Domain}ServiceGateway` — never direct HTTP
- **BaseResult<T>:** HTTP 200 always; `IsNotOk()` check; Payload guaranteed non-null when Ok
- **Entity naming:** `{Entity}` (no suffix), `{Entity}Number` (string business key), `{Entity}Id` (int surrogate)
- **Database access:** EF Core preferred; all SQL in Repository project only; `{Database}DbGateway` classes
- **Async suffix:** All methods returning `Task`/`Task<T>` use `Async` suffix
