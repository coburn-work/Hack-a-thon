---
description: "Use when mapping Oracle Adage columns to C# properties, writing AdageDbGateway or xxDbGateway queries, or converting Oracle data types to C# types"
---

# C# ↔ Oracle Data Mapping Conventions

## Data Type Mapping (§4.6)

| Oracle Type | Logical Meaning | C# Type | Mapping |
|---|---|---|---|
| `VARCHAR2(1)` `'Y'`/`'N'` | Boolean flag | `bool` | `((string)r.COL)?.Trim() == "Y"` |
| `NUMBER(1)` `0`/`1` | Boolean flag | `bool` | `((int?)r.COL ?? 0) == 1` |
| `VARCHAR2` discrete values | Status/type | `enum` | `MapStatus((string)r.COL)` switch expression |
| `NUMBER(11)` (integers) | Quantity/count | `int` | Direct cast |
| `NUMBER(18,3)` (decimals) | Quantity with fractions | `double` | Direct cast |
| `NUMBER(15,2)` (money) | Currency amounts | `decimal` | Direct cast |
| `NUMBER(11)` (ID/key) | Primary/foreign key | `int` or `long` | Direct cast |
| `DATE` | Date with time | `DateTime` | `.ConvertAdageDbDateTimeToUtc()` (see timezone-conventions) |
| `DATE` (nullable) | Optional date | `DateTime?` | `.ConvertAdageDbDateTimeToUtc()` |
| `VARCHAR2(n)` / `CHAR(n)` | String | `string` | **Always trim**: `((string)r.COL)?.Trim()` |

## Business-Friendly Naming (§4.7)

- C# properties use **business names**, not database column names
- **No abbreviations**: `Quantity` not `Qty`, `Number` not `Nbr`, `Location` not `Locn`
- Use `[Column("db_column_name")]` attribute to map between C# and Oracle names
- Named exception: `SkuId` is retained as-is
- Surrogate keys (database-generated identifiers with no business meaning) **are** exposed in C# entities and DTOs for backend record lookups, but are **never surfaced in UI-facing ViewModels or SPA APIs** — end users have no need for them

| DB Column | C# Property |
|---|---|
| `RECORD_ID` | `RecordId` (exposed in entities/DTOs, not ViewModels) |
| `in_whs_no` | `WarehouseNumber` |
| `IN_WHITM_NQOH` | `QuantityOnHand` |
| `so_cust_no` | `CustomerNumber` |
| `SO_ORD_NO` | `OrderNumber` |

## xxDbGateway Pattern (§4.8)

**Naming:** `{DatabaseName}DbGateway` (e.g., `AdageDbGateway`, `PkmsDbGateway`, `InventoryDbGateway`)

**Responsibilities:**
- Execute queries and stored procedures via EF Core `DbContext`
- Map database results to C# entities with proper type conversions
- **No business logic** — pure data access

**Location:** `SFC.Services.{Domain}.Repository/` project

```csharp
public class AdageDbGateway
{
    private readonly AdageDbContext _context;

    public AdageDbGateway(AdageDbContext context)
    {
        _context = context;
    }

    public async Task<List<ProductStockingStatus>> GetByWarehouseNumbersAsync(
        string productNumber, string[] warehouseNumbers)
    {
        return await _context.Set<WarehouseItem>()
            .Include(w => w.Extension)
            .Include(w => w.Warehouse)
            .Where(w => w.ProductNumber == productNumber
                && warehouseNumbers.Contains(w.Warehouse.WarehouseNumber))
            .Select(w => new ProductStockingStatus
            {
                WarehouseNumber = w.Warehouse.WarehouseNumber.Trim(),
                QuantityOnHand = (double)w.QuantityOnHand,
                StockingStatus = MapStockingStatus(w.Extension.StockingStatusCode.Trim()),
                TransferEligible = w.Extension.IsTransferEligible.Trim() == "1"
            })
            .ToListAsync();
    }
}
```

## Entity Example with Column Annotations

```csharp
using System.ComponentModel.DataAnnotations.Schema;

[Table("SO_DTL_TBL")]
public class SalesOrderLine
{
    [Column("SO_DTL_KEY")]
    public long LineKey { get; set; }          // Surrogate key — no business equivalent

    [Column("SO_ORD_NO")]
    public string OrderNumber { get; set; }    // Business identifier

    [Column("in_whs_no")]
    public string WarehouseNumber { get; set; } // Business identifier

    [Column("SO_DTL_ORDQT")]
    public double OrderedQuantity { get; set; } // Business-friendly name

    [Column("SO_DTL_HOLDF")]
    public bool IsOnHold { get; set; }          // Boolean from NUMBER 0/1
}
```

## Cross-References

- **Timezone:** Most Oracle `DATE` columns in Adage store Arizona Time. When reading into C#, use `ConvertAdageDbDateTimeToUtc()`. See `timezone-conventions` instructions. **Exception:** A small number of columns (e.g., Delivery Schedule dates) store values in the corresponding warehouse's local time rather than Arizona Time — verify the column's time zone convention before applying a conversion.
- **PL/SQL calling:** When C# calls Adage procedures, use parallel associative array parameters (ODP.NET limitation). See `oracle-plsql` instructions §15.7 for the interop pattern.
