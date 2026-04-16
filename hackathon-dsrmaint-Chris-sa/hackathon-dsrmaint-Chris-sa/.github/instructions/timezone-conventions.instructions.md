---
description: "Use when handling DateTime conversions between microservices and Adage Oracle, Arizona Time, UTC, or SFC.Core.Extensions timezone methods"
---

# Timezone Conventions

## The Rule (§12.1)

- **Microservices and Azure Functions:** Always UTC
- **Adage Oracle:** Arizona Time (MST = UTC-7, no DST, year-round)
- **Never** use `DateTime.Now` — always `DateTime.UtcNow` or `context.CurrentUtcDateTime` (Durable Functions)

## Date and DateTime Naming (§12.2)

- Properties representing a **date only** (no time component): suffix with `Date`, type `DateOnly` (.NET 6+) or `SfcDate` (.NET 4.7.2)
- Properties representing a **date and time**: suffix with `DateTime`, type `DateTime`

## SFC.Core.Extensions Methods (§12.6)

**Package:** `SFC.Core` (already referenced in all microservices)
**Namespace:** `SFC.Core.Extensions`

| Method | Direction | Use When |
|---|---|---|
| `ConvertFromUtcToArizonaTime()` | UTC → Arizona | Writing to Adage / querying Adage with date parameters |
| `ConvertFromArizonaTimeToUtc()` | Arizona → UTC | Converting Arizona Time values to UTC |
| `ConvertAdageDbDateTimeToUtc()` | Adage → UTC | Reading dates from Adage Oracle (adds 7 hours, sets Kind=Utc) |
| `SetAzureDbDateTimeKindToUtc()` | Kind fix | Azure SQL returns Unspecified but is UTC — sets Kind only, no time change |
| `ConvertFromUtcToOtherTimeZone()` | UTC → any | Multi-region warehouses (e.g., Boise with DST) |
| `ConvertFromOtherTimeZoneToUtc()` | any → UTC | Multi-region warehouses |

## Reading from Adage (§12.4)

```csharp
using SFC.Core.Extensions;

// Query: convert UTC to Arizona Time for WHERE clause
var startArizona = startDateUtc.ConvertFromUtcToArizonaTime();

// Result: convert Adage datetime (Arizona, Unspecified) to UTC
OrderDate = ((DateTime)r.OrderDate).ConvertAdageDbDateTimeToUtc();
```

## Writing to Adage (§12.4)

```csharp
using SFC.Core.Extensions;

// Convert UTC to Arizona for Adage storage
var revisionDateTimeArizona = order.RevisionDateTime.ConvertFromUtcToArizonaTime();
command.Parameters.Add("p_revision_date", OracleDbType.Date, revisionDateTimeArizona, ParameterDirection.Input);
```

## Reading from Azure SQL (§12.6)

```csharp
// Azure SQL returns Unspecified but is actually UTC — fix Kind only
var createdDateUtc = createdDate.SetAzureDbDateTimeKindToUtc();
```

## Common Pitfalls (§12.8)

| Pitfall | Wrong | Correct |
|---|---|---|
| Using local time | `DateTime.Now` | `DateTime.UtcNow` |
| Comparing mixed timezones | `adageDate > DateTime.UtcNow` | `adageDate.ConvertAdageDbDateTimeToUtc() > DateTime.UtcNow` |
| Arizona DST assumption | `TimeZoneInfo.FindSystemTimeZoneById("Mountain Standard Time")` | `TimeZoneInfo.FindSystemTimeZoneById("US Mountain Standard Time")` |
| Ambiguous parameter | `ProcessOrder(DateTime orderDate)` | `ProcessOrder(DateTime orderDateUtc)` — name indicates timezone |

## Arizona Time Key Facts

- Arizona TZID: **`"US Mountain Standard Time"`** (not `"Mountain Standard Time"`)
- Always UTC-7 year-round — **no DST**
- Phoenix (008) and Marana (055): Arizona Time
- Albuquerque (002) and Boise (012): Mountain Time (with DST) — use `ConvertFromUtcToOtherTimeZone()`

## Oracle DATE Column Conventions

- Adage stores dates in Arizona Time in `DATE` columns (which include time component)
- `Oracle DATE` → C# `DateTime` (use `ConvertAdageDbDateTimeToUtc()`)
- `Oracle DATE` (nullable) → C# `DateTime?`
- Always trim `CHAR` column strings from Oracle: `((string)r.COLUMN)?.Trim()`
