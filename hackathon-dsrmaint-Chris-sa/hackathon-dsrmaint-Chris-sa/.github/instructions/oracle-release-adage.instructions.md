---
description: "Use when creating Adage Oracle release packages, Install.bat scripts, adage.sql, Rollback.sql, disable_jobs.sql, enable_jobs.sql, .bak rollback files, or prerequisite DML/DDL scripts"
---

# Adage Oracle Release Package Conventions

## Release Package Structure (§14.1)

```
Install.bat                              <- copy from SampleAdageRelease, update variable header
scripts/
    Count_Invalids_Before.sql            <- copy from SampleAdageRelease/scripts/ (do not modify)
    compile_invalids.sql                 <- copy from SampleAdageRelease/scripts/ (do not modify)
    Count_Invalids_After.sql             <- copy from SampleAdageRelease/scripts/ (do not modify)
    disable_jobs.sql                     <- set cr_number (NUMBER) and referenced_list
    enable_jobs.sql                      <- set cr_number (must match disable_jobs.sql)
    FR{NNN}_prerequisites.sql            <- prerequisite DML/DDL (idempotent)
    {object1}.sql                        <- one per modified object (with standard header/footer)
    Rollback.sql                         <- compensating SQL with @@rollback/<name>.bak references
    rollback/
        {object1}.bak                    <- pre-change source of each compiled object
```

## Install.bat Variable Header

```bat
@echo off
SET  drctr=%~dp0scripts
SET prerequisites=FR{NNN}_prerequisites.sql
SET objectSpec=ADAGE.{PACKAGE}.PKS.sql
SET objectBody=ADAGE.{PACKAGE}.PKB.sql
SET backoutfilename=Rollback.sql
SET /A adage_count=1
SET /A counter=1
SET /A fileexists=0
```

- `drctr` is **always** `%~dp0scripts` — never hardcoded UNC paths
- **PKS before PKB**: All package specs must be listed and executed before any package bodies
- Copy the environment menu boilerplate (~400 lines) from `SampleAdageRelease/Install.bat` — modify only the variable header and `do_install` file list

## Job Management Scripts

**`disable_jobs.sql`** — two customization points:
- `cr_number NUMBER := {WorkItemId};` — the numeric Azure DevOps work item ID (column is `NUMBER`, use `TO_CHAR(cr_number)` for string concatenation)
- `referenced_list VARCHAR2(4000) := 'OBJ_A,OBJ_B,OBJ_C';` — **no spaces** after commas — every table, package, procedure, function, and view being modified

**`enable_jobs.sql`** — one customization point:
- `cr_number NUMBER := {WorkItemId};` — must match `disable_jobs.sql`

## SQL Script Standards

**Standard header** (every authored `.sql` file):
```sql
COLUMN identify NEW_VALUE thisSpool
SELECT REPLACE (global_name, '.SHAMROCKFOODS.COM') || '.' || TO_CHAR (SYSDATE, 'YYYYMMDD.HH24MISS') || '.' || USER identify FROM GLOBAL_NAME;
SPOOL {ScriptName}_Log.sql.&&thisSpool..lst;
SET TIME ON
SET TIMING ON
SET TRIMSPOOL ON
SET DEFINE OFF
-------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------------------------
```

**Object announcement** before each DDL/DML:
```sql
SELECT '****************   CREATE OR REPLACE PACKAGE BODY ADAGE.{NAME}    ****************' as ACTION FROM DUAL;
```

**Section separator** between objects:
```sql
-------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------------------------
```

**Standard footer** (every authored `.sql` file):
```sql
-------------------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------------------------------------------
show error
SPOOL OFF
EXIT
```

- **`EXIT` rule**: Top-level scripts (`.sql`, `Rollback.sql`) must end with `EXIT`. Sub-scripts called via `@@` (`.bak` files, prerequisite rollback files) must **NOT** contain `EXIT`.

## Rollback Conventions

- Every compiled object gets a `.bak` file in `scripts/rollback/` with pre-change source
- `Rollback.sql` uses `@@rollback/<OBJECT_NAME>.bak` — never rely on manual retrieval
- `Rollback.sql` spool name: `ADAGE_Rollback_Log`
- `do_backout` uses `pushd %drctr%` / `popd` so `@@rollback/` paths resolve correctly

## Prerequisite Scripts (§15.5)

- Place in `FR{NNN}_prerequisites.sql`; rollback in `FR{NNN}_prerequisites_rollback.sql`
- Both must be **idempotent** (wrap INSERT in `EXCEPTION WHEN DUP_VAL_ON_INDEX`, ALTER in `EXCEPTION WHEN col_exists`)
- Config keys: `{FEATURE}_{PURPOSE}` in UPPERCASE (e.g., `TRANSFER_FEATURE_ENABLED`)
- Config read pattern: `SELECT NVL(MAX(sfc_config_val), 'N') INTO l_var FROM adage.sfc_config_tbl WHERE sfc_config_key = 'KEY';`

## ALTER TABLE with Triggers (§14.1 step 9)

When ALTER TABLE is needed on a table with triggers:
1. Dynamically discover enabled triggers via `ALL_TRIGGERS`
2. Disable them
3. Perform ALTER TABLE
4. Re-enable only the triggers that were enabled before step 1
- Do **not** hard-code trigger names — always discover at deployment time

## Index Files for database-adage repo (§14.1 step 8)

When creating new indexes: generate `ADAGE.{INDEX_NAME}.IDX` in `ADAGEAZP/ADAGE/INDEX/` with `DROP INDEX` + `CREATE INDEX` with full STORAGE clause.

## Encoding

All files must be **ASCII-only** (code-points 0–127). No em-dashes, box-drawing characters, or arrows. Use `-`, `->`, `<-` instead.

## Adage Instance Registry

| Env | Instances |
|---|---|
| DEV | ADAGEDEV (002), ADAGECOD (003), ADAGENMD (009), ADAGEBLD (040), VADGCOD2 (032) |
| QA | VADGAZQ1, VADGCOQ1, VADGNMQ1, VADGBLQ1 |
| UAT | VADGAZU1, VADGCOU1, VADGNMU1, VADGBLU1 |
| PROD | ADAGEPHX, ADAGEDEN, ADAGEBL, ADAGEABQ (always ALL) |

## Prompt Cross-Reference

For a guided step-by-step release generation workflow, use `/create-adage-release` with the FR number, CR number, and object list.
