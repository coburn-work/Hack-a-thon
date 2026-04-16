---
applyTo: "**/*.cs"
---

# Shamrock Foods C# Conventions

## Architecture Overview

- Each application consists of a Vue 3 front-end SPA and a C# (.NET) back-end API Gateway
- Microservices handle business logic and data access; API Gateways and SPAs communicate with microservices via NuGet gateway packages
- Orchestrators (`SFC.Orchestrations.*`) manage multi-step workflows and long-running processes; they consume microservices via NuGet gateways

## Standard C# Naming

- **PascalCase** for method names, public members, and component names
- **camelCase** for local variables
- **_camelCase** (underscore prefix) for private and protected fields (e.g., `_myField`)
- **I** prefix for interfaces (e.g., `IUserService`, `ICompanyServiceGateway`)
- Create interfaces for all services and data access layers to enable dependency injection and testability

## NuGet Gateway Pattern (§2)

- **NEVER** make direct HTTP calls between microservices or from Azure Functions to microservices
- **ALWAYS** use NuGet gateway packages (e.g., `ISalesOrderServiceGateway`)
- Register via `services.Add{Name}ServiceGateway(Configuration)` in Startup/Program
- Each NuGet gateway requires a `.Configure<{Name}ServiceGatewayOptions>()` call (chained in `Program.cs`) setting `BaseUri`, `AdResourceId`, and `WebRequestTimeoutInSeconds` from the consuming project's own settings/options object
- Developers do not manually bump NuGet package versions — CI/CD auto-increments on build

## BaseResult<T> Wrapper (§3)

- Microservices **always** return HTTP 200 OK with `BaseResult<T>` wrapper
- Check `result.IsNotOk()` — never use redundant `result.Payload == null` guard
- When `ResultType == Ok`, `Payload` is **guaranteed non-null**
- Set `Payload` before setting `ResultType = ResultTypes.Ok`
- Use `BaseResult<bool>` with `Payload = true` for success-only results
- Initialize result with `ResultType = ResultTypes.BadRequest` as default
- `ValidationMessage` has `Message` (description) and `Field` (property name) properties
- `ResultType` values: `Ok` (success), `BadRequest` (validation errors), `NotFound` (missing resource)
- Use `result.To<U>()` to convert between `BaseResult` types while preserving `ValidationMessages` and `ResultType`

## Entity Naming (§4.1–4.5)

| Representation | Pattern | Type | Example |
|---|---|---|---|
| Full object | `{Entity}` | Object/DTO | `Customer`, `SalesOrder` |
| Business key | `{Entity}Number` | string | `CustomerNumber`, `WarehouseNumber` |
| Surrogate key | `{Entity}Id` | int | `CustomerId`, `WarehouseId` |

- **No** `Entity` suffix on entity classes — use `Customer` not `CustomerEntity`
- DTO: `{Entity}Dto`; Request: `{Action}{Entity}RequestDto`; Response: `{Action}{Entity}ResponseDto`
- `{Entity}Number` is for **purely numeric strings** (e.g., `"008"`, `"123456"`) — not for alphanumeric codes
- Surrogate keys (database-generated identifiers such as identity integers or GUIDs with no business meaning) are **never exposed** in C# entities or APIs when a business identifier exists
- Use `Id` as the property name for a database-generated primary key on its own entity (e.g., a GUID or identity column); use `{Entity}Id` when referencing that key from another entity
- **No abbreviations** in C# property or JSON names — use `Quantity` not `Qty`, `Number` not `Nbr`, `Description` not `Desc`. Named exception: `SkuId`

## Async Naming (§4.9)

- **All** methods returning `Task` or `Task<T>` must use the `Async` suffix
- Exception: `[Function]` attribute controls the external name; the C# method still uses `Async`

## Database Access (§5)

- **Only microservices and orchestrators** access databases directly — API Gateways, standalone Azure Functions, and SPAs never touch DBs; orchestrators may only access their own database and must call other services' data via NuGet gateways
- **Entity Framework Core** is the default data access technology. Raw SQL and Dapper are **never permitted**
- Stored procedures are allowed for: existing SP modification, complex atomic Oracle logic, proven performance need
- All data access goes through the Repository project (`xxDbGateway` classes) — no SQL in Service or Controller layers

## Error Handling & Validation (§11)

- **No data annotations** for validation — all validation in service layer using `BaseResult.ValidationMessages`
- Controllers always return `Ok(result)` — never throw or return other status codes; exception: webhook endpoints subscribing to Event Grid may return non-2xx responses to leverage Event Grid's built-in retry mechanisms
- Service layer catches exceptions, sets `ResultType` and `ValidationMessages`
- Service layer should handle predictable but incorrect states (e.g., a downstream dependency found in a state that does not permit the requested operation) by setting `ResultType` and `ValidationMessages` rather than throwing

## Security (§13)

- API Gateways: OAuth2 RBAC via Azure Entra ID, `[Authorize]` with role policies
- Microservices: App Registration bearer tokens with `DEFAULT_ACCESS` role
- Service-to-service auth via `.AddAppRegistrationAuthentication()`

## Comments

- Comments explain **why**, not what — avoid redundant comments restating code
- Note usage and purpose when using external libraries or dependencies
- XML doc comments on public APIs with `<example>` and `<code>` where applicable

## Formatting

- Insert a newline before the opening curly brace of any code block
- Final return statement of a method is on its own line
- Prefer pattern matching and switch expressions where supported
- Use `nameof` instead of string literals when referring to member names

## AutoMapper (§7.4)

- Create AutoMapper profiles in a `Mappings/` folder named `[Domain]MappingProfile.cs`
- Profiles inherit from `BaseMapperProfile` (from `SFC.Core.Mapper`) and define mappings in the constructor using `CreateMap<TSource, TDestination>()`; call `IgnoreAllUnmappedProperties()` on each map to suppress unmapped property warnings by default
- Register profiles via `AddAutoMapper(AppDomain.CurrentDomain.GetAssemblies())` in `Program.cs`
- Use `using` aliases when namespace collisions exist (e.g., `using CoreBusinessUnit = SFC.Core.Contracts.EnumsAndConstants.BusinessUnit;`)

## Logging

- Use **Application Insights** for structured logging and telemetry
- Use appropriate log levels: `Information` for business events, `Warning` for recoverable issues, `Error` for failures
- Include correlation IDs for request tracking
- Let Application Insights handle unhandled exceptions globally — do not add a global catch-all

## Cross-References

- **DateTime handling:** When mapping dates to/from Adage Oracle, typically convert between UTC and Arizona Time; note that some Adage tables store values in the warehouse's local timezone. See `timezone-conventions` instructions for `SFC.Core.Extensions` methods and pitfalls.
- **Oracle data mapping:** When writing `xxDbGateway` classes or mapping Oracle columns to C# properties, see `csharp-oracle-data-mapping` instructions for type conversion rules (bool from Y/N, enum from status strings, trim CHAR columns).
- **ARM deployment:** When adding or modifying app settings, Event Grid subscriptions, connection strings, or Key Vault references, update **all four** ARM parameter files (dev/qa/uat/prod) in the project's `ARMTemplateParameterFiles/` directory.
- **Testing:** See `csharp-testing` instructions for unit test conventions.
- **Azure SQL:** See `azure-sql` instructions for SQL Database Project conventions.

## Workflow Reminder

When implementation is complete and deployment artifacts are needed, ask Copilot to generate the deployment package or use `/create-microservice` or `/create-orchestrator`.
