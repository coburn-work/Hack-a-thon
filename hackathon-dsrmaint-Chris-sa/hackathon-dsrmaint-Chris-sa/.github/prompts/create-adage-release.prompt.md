---
description: "Generate a complete Adage Oracle release package with Install.bat, scripts, rollback files, and deployment scripts"
agent: "agent"
---

# Generate Adage Oracle Release Package

You are generating a complete Adage Oracle release package. Follow the `oracle-release-adage` instructions for all conventions.

## Required Information

Ask the user for any missing information:

1. **FR identifier** (e.g., `FR-006`) — used in prerequisite script naming and REVISIONS header
2. **CR/Work Item number** (e.g., `437525`) — used in `disable_jobs.sql` and `enable_jobs.sql` as a NUMBER
3. **Objects being modified** — list of each object with its type:
   - Package specs (`.PKS`)
   - Package bodies (`.PKB`)
   - Standalone procedures/functions
   - Tables being altered
   - New tables/views/indexes
4. **Developer initials** (e.g., `RKT`) — for REVISIONS header entries
5. **Prerequisites** — any config rows, ALTER TABLE, or seed data needed

## Generation Steps

### Step 1: Create `Install.bat`

Copy the template from `SampleAdageRelease/Install.bat`. Update only:
- The `SET` variable header (one variable per object, PKS before PKB)
- The `do_install` subroutine (file_check and execute each variable)
- Leave the environment menu boilerplate unchanged

### Step 2: Generate `disable_jobs.sql`

Copy template structure from `SampleAdageRelease/scripts/disable_jobs.sql`. Set:
- `cr_number NUMBER := {WorkItemId};`
- `referenced_list VARCHAR2(4000) := '{comma-separated object names, NO SPACES}';`

Cross-check: every object in a CREATE OR REPLACE or ALTER TABLE must be in `referenced_list`.

### Step 3: Generate `enable_jobs.sql`

Copy template structure. Set `cr_number NUMBER := {WorkItemId};` (must match disable_jobs.sql).

### Step 4: Generate per-object `.sql` files

For each modified compiled object, create `{SCHEMA}.{OBJECT_NAME}.{EXT}.sql` with:
- Standard header (COLUMN identify, SPOOL, SET commands, separator)
- Object announcement (`SELECT '***...***' as ACTION FROM DUAL;`)
- Full `CREATE OR REPLACE` source
- Standard footer (show error, SPOOL OFF, EXIT)

### Step 5: Generate prerequisites script (if needed)

Create `FR{NNN}_prerequisites.sql` with idempotent DML/DDL (wrap in EXCEPTION handlers).
Create `FR{NNN}_prerequisites_rollback.sql` with compensating statements.

### Step 6: Generate `Rollback.sql`

Standard header with spool name `ADAGE_Rollback_Log`. For each compiled object:
```sql
SELECT '***   ROLLBACK {ACTION} {SCHEMA}.{OBJECT}   ***' as ACTION FROM DUAL;
@@rollback/{SCHEMA}.{OBJECT}.{EXT}.bak
```
If prerequisites exist, include `@@FR{NNN}_prerequisites_rollback.sql`.
Standard footer (show error, SPOOL OFF, EXIT).

### Step 7: Create `.bak` rollback files

For each compiled object, retrieve the current source from the workspace and save as `scripts/rollback/{SCHEMA}.{OBJECT}.{EXT}.bak`. These files must NOT contain EXIT.

### Step 8: Generate index files (if new indexes created)

For each new index, create `ADAGE.{INDEX_NAME}.IDX` in `ADAGEAZP/ADAGE/INDEX/` with DROP INDEX + CREATE INDEX with full STORAGE clause.

### Step 9: Copy utility scripts

Note that `Count_Invalids_Before.sql`, `compile_invalids.sql`, and `Count_Invalids_After.sql` should be copied from `SampleAdageRelease/scripts/`.

## Validation Checklist

- [ ] All files are ASCII-only (no em-dashes, box-drawing, or special characters)
- [ ] PKS files listed before PKB files in Install.bat
- [ ] `referenced_list` contains ALL modified objects with no spaces after commas
- [ ] `cr_number` is declared as NUMBER and matches in both disable/enable scripts
- [ ] Every compiled object has a corresponding `.bak` in `rollback/`
- [ ] Rollback.sql references every `.bak` via `@@rollback/`
- [ ] Prerequisites are idempotent
- [ ] No `.bak` file contains EXIT
- [ ] All top-level `.sql` files end with EXIT
