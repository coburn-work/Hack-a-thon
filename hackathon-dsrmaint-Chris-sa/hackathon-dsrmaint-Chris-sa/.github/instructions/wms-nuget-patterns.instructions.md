---
description: "Use when consuming WMS microservice NuGet packages, implementing WMS gateway providers, or configuring WMS URL Discovery"
---

# WMS NuGet Gateway Patterns

## WMS Monolith Avoidance (§10.1)

New Warehouse Transfer functionality does **NOT** use `Sfc.Wms.App.Api` (the legacy WMS monolith). Use dedicated microservices (`Sfc.Wms.Services.{Domain}`) and the new Transfer Gateway (`Sfc.Wms.Gateways.Transfer`).

## Per-Warehouse Deployment Model

WMS microservices are deployed as **per-warehouse instances**. URL resolution depends on which NuGet package pattern the service uses.

## Pattern A — New Provider Pattern (with URL Discovery) (§10.3)

NuGet packages that expose a `*WmsServiceGatewayProvider` extending `BaseWmsServiceGatewayProvider<T>`. URL Discovery is built in — caller must register its infrastructure.

**DI Registration:**
```csharp
// URL Discovery infrastructure
services.Configure<WmsUrlDiscoveryServiceOptions>(configuration.GetSection("WmsUrlDiscovery"));
services.AddSingleton<IWmsUrlDiscoveryService, WmsUrlDiscoveryService>();
services.Configure<RemoteCacheOptions>(configuration.GetSection("RemoteCache"));
services.AddSingleton<IRemoteCacheHelper, RemoteCacheHelper>();

// Gateway provider
services.Configure<AssignmentWmsServiceGatewayProviderOptions>(
    configuration.GetSection("WmsAssignmentProvider"));
services.AddScoped<AssignmentWmsServiceGatewayProvider>();
```

**Usage:**
```csharp
var gateway = _assignmentProvider.GetServiceGateway(warehouseNumber);
return await gateway.ProcessAssignmentAsync(request);
```

**Base class:** `BaseWmsServiceGatewayProvider<T>` from `SFC.WMS.Core.Gateway`
**Backing store:** Redis cache via `RemoteCacheHelper`

## Pattern B — Legacy Pattern (consumer-provided URL) (§10.3)

NuGet packages that expose gateway classes directly. Caller provides the base URL via configuration.

**DI Registration:**
```csharp
services.Configure<WmsServiceGatewayOptions>("Service", configuration.GetSection("WmsServiceOptions"));
services.AddScoped<IItemWmsServiceGateway, ItemWmsServiceGateway>(sp =>
    new ItemWmsServiceGateway(
        sp.GetRequiredService<IOptionsSnapshot<WmsServiceGatewayOptions>>().Get("Service"),
        sp.GetRequiredService<HttpClient>(),
        sp.GetRequiredService<IBearerTokenProvider>()));
// appsettings: "WmsServiceOptions": { "BaseUrl": "https://sfc-wms-items-055.azurewebsites.net" }
```

## How to Determine Which Pattern

Check the NuGet package: if it exposes a `*WmsServiceGatewayProvider`, use Pattern A. If it only exposes a direct gateway class, use Pattern B.

> **Production status:** URL Discovery is in-flight. Use legacy NuGets (Pattern B) until updated NuGets are verified. WMS gateways deployed per-warehouse use static config.

## Reference Implementations

| Pattern | Provider / Gateway | NuGet Package |
|---|---|---|
| New (URL Discovery) | `AssignmentWmsServiceGatewayProvider` | `Sfc.Wms.Services.Assignment` |
| New (URL Discovery) | `InboundContainerWmsServiceGatewayProvider` | `Sfc.Wms.Services.Inbound.Container` |
| Legacy (static URL) | `ItemWmsServiceGateway` | `Sfc.Wms.Services.Item` |
| Legacy (static URL) | `TrailerWmsServiceGateway` | `Sfc.Wms.Services.Trailer` |

## Thread Safety (§10.1–10.2)

- The legacy WMS monolith has a "Unit of Work" anti-pattern causing thread safety issues — **not applicable** to dedicated WMS microservices
- Always create **new** connections per operation — ADO.NET connection pooling handles concurrency automatically
- Never share `IWmsConnectionFactory` connections across parallel tasks
