---
description: "Use when writing ERP API Gateway controllers, services, ViewModel mapping, or BFF patterns in SFC.ERP.* projects"
---

# API Gateway Conventions

## Purpose (§7.1)

API Gateways (`SFC.ERP.{Name}`) serve as:
- **Backend for Frontend (BFF):** Tailored APIs for specific SPAs
- **Result Unwrapper:** Convert `BaseResult<T>` → HTTP status codes for SPAs
- **ViewModel Mapper:** Map microservice DTOs → SPA-specific ViewModels
- **Security Boundary:** Authenticate users, authorize requests

API Gateways do **NOT**:
- Contain business logic
- Access databases directly (use NuGet gateways to call microservices)
- Transform data beyond DTO → ViewModel mapping

## Microservice NuGet Package Structure (§7.5)

Each microservice exposes two packages to consuming API Gateways:

- **Contracts Package**: `SFC.Services.{Domain}.Contracts`
  - DTOs at namespace `SFC.Services.{Domain}.Contracts.DTOs`
  - DTOs may be nested by entity: `SFC.Services.{Domain}.Contracts.DTOs.{EntityName}` (e.g., `Customer.CustomerV4Dto`)
  - Always import from Contracts when mapping microservice DTOs to API Gateway models

- **NuGet Gateway Package**: `SFC.Services.{Domain}.Nuget`
  - Gateway interface at `SFC.Services.{Domain}.Nuget.Interfaces` (e.g., `ICompanyServiceGateway`)
  - The gateway interface is the **source of truth** for all available methods and their exact signatures
  - Always inject the gateway interface into service classes

**Enum Namespace Conflicts:** When the same enum name exists in multiple packages (e.g., `BusinessUnit` in both `SFC.Core.Contracts` and a domain package), use namespace aliases:
```csharp
using CoreBusinessUnit = SFC.Core.Contracts.EnumsAndConstants.BusinessUnit;
using CompanyBusinessUnit = SFC.Services.Company.Contracts.Enums.BusinessUnit;
```
Domain-specific DTOs typically use enums from their own Contracts package.

## GetResult() Pattern (§7.2)

Controllers inherit `BaseController` and use `GetResult()` to unwrap `BaseResult<T>`:

```csharp
[ApiController]
[Route("transfer-orders")]
[Authorize]
public class TransferOrdersController : BaseController
{
    [HttpPost]
    public async Task<ActionResult<TransferOrderViewModel>> CreateTransferOrder(
        [FromBody] CreateTransferOrderRequest request)
    {
        var result = await _transferOrderService.CreateTransferOrderAsync(request);
        return GetResult(result);  // BaseResult → HTTP status codes
    }
}
```

`GetResult()` maps: `Ok` → 200, `BadRequest` → 400, `NotFound` → 404, `Unauthorized` → 401, `Forbidden` → 403.

## Service Layer Pattern (§7.3)

Gateway services: map ViewModel → DTO, call microservice via NuGet gateway, map DTO → ViewModel:

```csharp
public async Task<BaseResult<TransferOrderViewModel>> CreateTransferOrderAsync(
    CreateTransferOrderRequest request)
{
    var result = new BaseResult<TransferOrderViewModel> { ResultType = ResultTypes.BadRequest };

    var dto = _mapper.Map<CreateSalesOrderRequestDto>(request);
    var msResult = await _salesOrderGateway.CreateSalesOrderAsync(dto);

    if (msResult.IsNotOk())
    {
        result.ValidationMessages = msResult.ValidationMessages;
        result.ResultType = msResult.ResultType;
        return result;
    }

    result.Payload = _mapper.Map<TransferOrderViewModel>(msResult.Payload);
    result.ResultType = ResultTypes.Ok;
    return result;
}
```

## Authentication (§13.1–13.2)

- API Gateways use **OAuth2 RBAC via Azure Entra ID** for user-facing auth
- Use `SFCAuthorizeAttribute` from `SFC.Core.Web` for controller authorization
- `[Authorize(Policy = "...")]` for RBAC role checks
- ERP Gateways own **no databases** — data flows only through NuGet gateways

## ViewModel Mapping

- Use `IMapper` (AutoMapper or equivalent) for DTO ↔ ViewModel conversions
- ViewModels are SPA-specific — may combine data from multiple microservice DTOs
- Keep mapping thin — no business logic in mapper profiles

## Search/Filter Model Requirements (§7.6)

**All search/filter model properties MUST be nullable.** ASP.NET Core treats non-nullable reference types as required, causing 400 errors when optional criteria are omitted.

✅ **Search/Filter Models** — all nullable:
```csharp
public class CustomerSearchFilter
{
  public string? CustomerNumber { get; set; }
  public string? CustomerName { get; set; }
  public List<string>? Tags { get; set; } = new List<string>();
}
```

❌ **Wrong** — non-nullable causes "required field" validation errors:
```csharp
public class CustomerSearchFilter
{
  public string CustomerNumber { get; set; }  // Required by ASP.NET Core!
}
```

**Model type guidelines:**
- **Entity models (Create/Update):** Non-nullable for required fields, nullable for optional
- **Search/Filter models:** ALL properties nullable
- **Response models:** Match the source DTO nullability exactly

## Cross-References

- **Vue.js SPA:** The SPA calls these gateway endpoints. See `vue-spa` instructions for the `@sfc-enterprise-ui/fetch-api` pattern and Vue.js 3 Composition API conventions.
- **NuGet gateway:** All microservice calls go through NuGet gateways. See `csharp-conventions` for the gateway pattern.
