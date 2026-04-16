---
description: "Generate a complete PkMS/WMS Oracle release package with InstallWMS.bat, install.sql, backout.sql, and rollback files"
agent: "agent"
---

# Generate PkMS/WMS Oracle Release Package

You are generating a complete PkMS/WMS Oracle release package. Follow the `oracle-release-pkms` instructions for all conventions.

## Required Information

Ask the user for any missing information:

1. **Objects being modified** — list of each object with its type:
   - Package specs (`.PKS`) / bodies (`.PKB`)
   - Standalone procedures/functions
   - Tables being altered
   - New tables/views/indexes
2. **Whether jobs need disabling** — determines if `disable_jobs.sql` / `enable_jobs.sql` are needed

## Generation Steps

### Step 1: Copy `InstallWMS.bat`

Copy from `SamplePKMSRelease/InstallWMS.bat` — no changes needed.

### Step 2: Copy utility scripts

Copy from `SamplePKMSRelease/`:
- `count_invalids_before.sql`
- `compile_invalids.sql`
- `count_invalids_after.sql`

### Step 3: Generate `install.sql`

Create the main install script containing all DDL/DML/package changes. Must end with `EXIT`.

For compiled objects: full `CREATE OR REPLACE` source.
For DDL: `ALTER TABLE`, `CREATE TABLE`, etc. wrapped in idempotent exception handlers where possible.

### Step 4: Generate `backout.sql`

Create the backout script. For each compiled object:
```sql
@@rollback/{SCHEMA}.{OBJECT_NAME}.{EXT}.bak
```
For DDL changes: compensating `ALTER TABLE DROP COLUMN`, `DROP TABLE`, etc.
Must end with `EXIT`.

### Step 5: Create `.bak` rollback files

For each compiled object, save pre-change source in `rollback/{SCHEMA}.{OBJECT_NAME}.{EXT}.bak`.
These files must **NOT** contain EXIT.

### Step 6: Generate job scripts (if needed)

If jobs need disabling, create `disable_jobs.sql` and `enable_jobs.sql`.

## Key Differences from Adage

- **Flat directory** — all files alongside `InstallWMS.bat` (no `scripts/` subdirectory)
- **Exact filenames**: `install.sql` and `backout.sql` (lowercase)
- `disable_jobs.sql` / `enable_jobs.sql` are **optional**

## Validation Checklist

- [ ] All files are ASCII-only
- [ ] All files in flat directory (no subdirectories except `rollback/`)
- [ ] `install.sql` and `backout.sql` end with EXIT
- [ ] No `.bak` file contains EXIT
- [ ] Every compiled object has a corresponding `.bak` in `rollback/`
- [ ] `backout.sql` references every `.bak` via `@@rollback/`
- [ ] Changes are idempotent where possible
