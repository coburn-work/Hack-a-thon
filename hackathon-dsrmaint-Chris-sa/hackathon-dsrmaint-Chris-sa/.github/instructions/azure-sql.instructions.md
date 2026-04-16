---
description: "Use when working with Azure SQL Database Projects, SQL Server schemas, stored procedures, or SQL migration scripts"
---

# Azure SQL Database Project Conventions

## Schema & Organization

- Use schema-based organization for database objects
- Use version-controlled `.sql` files for all schema changes
- Use stored procedures for batch operations or scenarios where Entity Framework performs poorly

## Naming Conventions

- **PascalCase** for table names, column names, indexes, and constraints
- **Singular nouns** for table names (e.g., `Customer`, `SalesOrder`)
- Use meaningful, descriptive names for all database objects

## Data Access

- **No inline SQL in C# code** — all data access goes through Entity Framework Core or stored procedures
- Optimize queries for performance — avoid N+1 patterns and unnecessary full-table scans

## Cross-References

- **C# data access:** See `csharp-conventions` §5 for the database access rules and Repository pattern.
- **Oracle databases:** For Oracle-specific conventions, see `csharp-oracle-data-mapping` and `oracle-plsql` instructions.
