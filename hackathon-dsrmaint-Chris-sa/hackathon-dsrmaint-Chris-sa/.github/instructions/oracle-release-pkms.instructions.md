---
description: "Use when creating PkMS WMS Oracle release packages, install.sql, backout.sql, InstallWMS.bat scripts, or PkMS rollback files"
---

# PkMS/WMS Oracle Release Package Conventions

## Release Package Structure (§14.2)

**All files in a flat directory** alongside `InstallWMS.bat` — no `scripts/` subdirectory:

```
InstallWMS.bat            <- copy from SamplePKMSRelease (no changes needed)
install.sql               <- all DDL/DML/package changes
backout.sql               <- compensating SQL to undo changes
count_invalids_before.sql <- copy from SamplePKMSRelease (no changes needed)
compile_invalids.sql      <- copy from SamplePKMSRelease (no changes needed)
count_invalids_after.sql  <- copy from SamplePKMSRelease (no changes needed)
disable_jobs.sql          <- (if jobs need disabling during install)
enable_jobs.sql           <- (if jobs were disabled)
rollback/
    {SCHEMA}.{OBJECT_NAME}.{EXT}.bak     <- pre-change source of each compiled object
```

## Key Differences from Adage Releases

- **Flat directory** — no `scripts/` subdirectory; `InstallWMS.bat` uses `%~dp0%` as `drctr`
- **Exact filenames**: `install.sql` and `backout.sql` (lowercase) — the installer looks for these exact names
- `disable_jobs.sql` / `enable_jobs.sql` are **optional** (skipped silently if not present)
- Invalids scripts are **local** to the release directory (not shared network paths)

## Execution Order Per Instance

1. Disable jobs (`disable_jobs.sql` if present)
2. Count invalids before (`count_invalids_before.sql`)
3. Run `install.sql` (install) **or** `backout.sql` (backout)
4. Compile invalids (`compile_invalids.sql`)
5. Count invalids after (`count_invalids_after.sql`)
6. Enable jobs (`enable_jobs.sql` if present)

## Rollback of Compiled Objects

- For every modified compiled object: save pre-change source in `rollback/` as `{SCHEMA}.{OBJECT_NAME}.{EXT}.bak` (e.g., `SHAMLIVE.PKG_INVENTORY.PKB.bak`)
- `backout.sql` uses `@@rollback/{SCHEMA}.{OBJECT_NAME}.{EXT}.bak` — never rely on manual steps
- `.bak` files must **not** contain `EXIT`
- `install.sql` and `backout.sql` must end with `EXIT`
- Both must be **idempotent** where possible

## PkMS Instance Registry

| Env | Instances |
|---|---|
| DEV | SWMDAD1 (001), VSWMDMD1 (007), VPKMSD1 (008), SWMDVD1 (010), VPKMSCOD (026), SWMBILD1 (056) |
| QA | SWMDAQ1, VSWMDMQ1, VPKMSQ1, SWMDVQ1, VPKMSCOQ, SWMBILQ1 |
| UAT | SWMDAU1, VSWMDMS1, VPKMSCS1, SWMDVU1, VSWMCOS, SWMBILU1 |
| PROD | SWMSDAP, SWMSDMP, PKMSDB, SWMSDVP, SWMSCOP, SWMSBILP |

## Encoding

All files must be **ASCII-only** (code-points 0–127). No em-dashes, box-drawing characters, or arrows.

## Prompt Cross-Reference

For a guided step-by-step release generation workflow, use `/create-pkms-release` with the object list.
