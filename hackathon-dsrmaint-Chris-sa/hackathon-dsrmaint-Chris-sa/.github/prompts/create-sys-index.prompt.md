---
description: "Generate an AI Navigation Index (sys-index) for a large source file to enable efficient Copilot navigation"
agent: "agent"
---

# Generate AI Navigation Index (sys-index)

You are generating a sys-index file for a large source file. Follow the conventions in architecture-guide.md §17.

## Required Information

Ask the user for any missing information:

1. **Source file path** — the workspace-relative path to the file being indexed (e.g., `ADAGEAZP/ADAGE/PACKAGE_BODY/ADAGE.SOE_PROCESS_ORDERS.PKB`)

## Naming Convention

```
sys-index-{system}-{component}.md
```

| Segment | Description | Example |
|---|---|---|
| `{system}` | Application/schema (lowercase, hyphenated) | `adage`, `pkms`, `salesorder` |
| `{component}` | File/module (lowercase, hyphenated) | `soe-process-orders`, `pkg-inventory` |

Place the file in the `Feature Plan` workspace folder root.

## Generation Process

### Step 1: Read file header

Read the first ~100 lines to capture:
- Package/class name and purpose
- Global variable declarations
- Type declarations

### Step 2: Discover structure

Search for procedure and function boundaries:
- For PL/SQL: search for `PROCEDURE ` and `FUNCTION ` declarations
- For C#: search for method signatures
- Record line numbers for each

### Step 3: Build section map

Read representative chunks around each procedure/function boundary to understand purpose and group into logical sections.

### Step 4: Generate the sys-index

## Required Sections

### Header (required)

```markdown
# {COMPONENT_NAME} — AI Navigation Index

**Purpose:** Pre-digested structural reference for `{FILE_PATH}`.
Allows Copilot to orient within the file without loading all {LINE_COUNT} lines.

**File:** `{WORKSPACE_FOLDER}/{RELATIVE_PATH}`
**Total Lines:** ~{LINE_COUNT}
**Current Version:** {VERSION} ({DATE})
```

### Section 1: Entry Points (always include for executable units)

Table of public entry points with:
- Line number (approx.)
- Signature summary
- Callers
- One-line description

### Section 2: Section Map (always include)

Numbered table mapping line ranges to logical sections.

### Section 3: Procedure/Function Index (when >5 program units)

Every procedure/function in declaration order with line number, signature, and purpose.

### Section 4: External Dependencies (when file calls other packages/services)

Tables of: packages called, tables written (DML), tables queried (SELECT).

### Section 5: Global Variables (PL/SQL packages with globals)

Row-type globals, scalar globals with types, defaults, and roles.

### Section 6: State Machines (when column/variable encodes states)

Values, names, setters, meanings, resolution paths.

## Maintenance Footer

```markdown
*Last updated: {DATE} — next update when source file changes >50 lines of drift.*
```

## After Generation

Update the `README.md` file directory table to include the new sys-index file.
