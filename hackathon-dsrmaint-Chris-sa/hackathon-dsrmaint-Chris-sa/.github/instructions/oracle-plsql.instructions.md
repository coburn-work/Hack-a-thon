---
applyTo: "**/*.{PKB,PKS,pkb,pks}"
---

# Shamrock Foods Oracle PL/SQL Standards

## Version History (§15.1)

- **No** `--` inline comments for change documentation — all changes go in the header `REVISIONS:` block
- Increment minor version (e.g., `1.05` → `1.06`)
- Use ISO 8601 date format (`YYYY-MM-DD`), developer initials, and Epic-FR prefix (e.g., `431229-001:`)
- Multi-change entries: list each on its own `- ` prefixed line, indented under Description — never semicolons on one line
- Entries must be chronologically ascending

## Logging (§15.2)

- **Always** use `adage.logger($$plsql_unit, 'event_text')` — never insert directly into `sfc_adagelog_tbl`
- `$$plsql_unit` resolves to package name at compile time — include procedure name in event text
- `sfc_adagelog_tbl` has only 3 columns: `APPLICATION VARCHAR2(30)`, `EVENT_DATE DATE`, `EVENT VARCHAR2(255)` — no LOG_LEVEL, LOG_SOURCE, LOG_MESSAGE columns exist
- Exception logging must include `SQLERRM` and collapsed backtrace:

```sql
adage.logger($$plsql_unit, 'proc_name FAIL: ' || SQLERRM ||
   REPLACE(REPLACE(UPPER(DBMS_UTILITY.format_error_backtrace), CHR(10)), 'ORA-06512: ', ','));
```

## Object Naming (§15.3)

| Object | Suffix | Example |
|---|---|---|
| Table | `_tbl` | `transfer_config_tbl` |
| View | `_vw` | `transfer_status_vw` |
| Trigger | `_trg` | `transfer_audit_trg` |
| Index | `_IDX{N}` | `SO_DTL_EXT_IDX4` |
| Bitmap idx | `_BMIDX{N}` | `IN_WHITM_TBL_BMIDX1` |
| Type | `_typ` / `_tab` / `_obj` | |

- **No** `SFC_` prefix on new objects — the schema qualifier identifies ownership
- Procedures, functions, and packages have **no** suffix

## Variable Naming (§15.4)

| Scope | Prefix | Example |
|---|---|---|
| Local | `l_` | `l_transfer_enabled` |
| Constant | `c_` | `c_gl_cmp_key` |
| Global | `g_` | `g_transfer_event` |
| IN param | `p_` | `p_so_dtl_key` |
| OUT param | `o_` | `o_result_code` |
| IN OUT param | `io_` | `io_allocation_qty` |
| Exception | `e_` | `e_transfer_not_found` |

- **Always** declare: `c_gl_cmp_key CONSTANT VARCHAR2(2) := 'SF';` — never use literal `'SF'` inline
- Declare all variables in the `IS`/`AS` block before `BEGIN` — no inline declarations
- Order: constants → `%ROWTYPE` → local scalars → cursors → collections

## Error OUT Parameters (§15.7)

- Initialize `errCode := 0;` as the **first executable statement** after `BEGIN`
- On failure: `errCode := -1; errMsg := SQLERRM || backtrace;`
- Use `RAISE_APPLICATION_ERROR(-20001, 'message')` for validation failures — not direct assignment with RETURN
- Every C#-callable procedure needs a top-level `EXCEPTION WHEN OTHERS` block with ROLLBACK, errCode/errMsg assignment, and logger call

## Package Structure (§15.6)

- All `TYPE` declarations above any `PROCEDURE`/`FUNCTION` in package spec
- Private variables/procedures at top of package body, before public implementations
- New packages must be **stateless** — no mutable `g_` globals (immutable `c_` constants are fine)

## Composite Keys & Index Usage (§15.7, §15.12)

- `SO_DTL_KEY` is **not globally unique** — always include all composite key columns: `GL_CMP_KEY`, `SO_BRNCH_KEY`, `SO_HDR_KEY`, `SO_DTL_KEY`
- After validating branch via `so_brnch_tbl`, use the local variable (`l_so_brnch_key`) in all WHERE clauses
- Include **all leading columns** of composite indexes in WHERE/JOIN predicates — never filter on trailing column only

## Associative Array Iteration (§15.11)

- **Always** use `FIRST .. LAST` with empty-collection guard — never `1 .. COUNT`

```sql
IF l_keys.COUNT > 0 THEN
    FOR i IN l_keys.FIRST .. l_keys.LAST LOOP
        process_key(l_keys(i));
    END LOOP;
END IF;
```

## Schema Ownership (§15.15)

- **`ADAGE`** schema: core business logic, transactional DML, domain packages
- **`SVC_{DOMAIN}`** schema: microservice-facing views and API-callable procedures
- `SVC_*` reads from `ADAGE` tables; writes go through `ADAGE` package procedures

## Oracle 12c Constraints (§15.16)

- **No** `BOOLEAN` columns — use `VARCHAR2(1)` with `'Y'`/`'N'` or `NUMBER(1)` with `1`/`0`
- `JSON_OBJECT()` is **SQL-only** — use string concatenation in PL/SQL bodies
- `SYS_GUID()` formatting requires `SELECT ... INTO ... FROM DUAL`

## Tablespace & Storage (§15.10)

- Indexes → `ADAGE_INDEX`; all other objects → `ADAGE_DATA`
- CREATE INDEX must include: `PCTFREE 10; INITRANS 2; MAXTRANS 255; STORAGE(INITIAL 1M, NEXT 1M, PCTINCREASE 0, MAXEXTENTS UNLIMITED); NOLOGGING` then `ALTER INDEX ... LOGGING`

## Column Widths (§15.13)

- `VARCHAR2(2)` = GL_CMP_KEY; `VARCHAR2(4)` = warehouse/branch; `VARCHAR2(10)` = product/customer/header key; `VARCHAR2(20)` = order number/references
- Common mistake: `VARCHAR2(5)` for warehouse — canonical width is `VARCHAR2(4)`

## Cross-References

- **C# data mapping:** When writing procedures callable from C#, use parallel associative array parameters (ODP.NET limitation — no RECORD/OBJECT types). See `csharp-oracle-data-mapping` instructions for C# type mapping from Oracle columns.
- **C#-callable procedures:** Use `errCode OUT NUMBER, errMsg OUT VARCHAR2` pattern. Initialize `errCode := 0`. Schema-level types `T_NUMARR`, `T_VARCHARARR`, `T_DATEARR` for `SYS_REFCURSOR` output. Package-spec `INDEX BY PLS_INTEGER` arrays for input parameters.
- **Adage date conventions:** Adage stores dates in Arizona Time (UTC-7, no DST). When C# microservices consume Adage dates, they convert via `SFC.Core.Extensions` — see `timezone-conventions` instructions.

## Workflow Reminder

When your PL/SQL changes are complete, generate the Adage release package by asking Copilot or using `/create-adage-release`.
