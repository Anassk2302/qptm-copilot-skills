# ADO MCP Integration — Systematic Debugging

## Overview

The Azure DevOps MCP (`@azure-devops/mcp`) provides tools to search work items, read investigation history, search code across repos, and document findings. This reference shows how to use each tool during debugging phases.

**MCP Server**: `ado` (configured in `.vscode/mcp.json`)
**Tool Prefix**: `mcp_ado_*`

---

## Phase 1: Evidence Collection

### Get Work Item Details

```
Tool: mcp_ado_wit_get_work_item
Params:
  id: [work item number]
  expand: "all"    ← ALWAYS use "all" to get relations, comments, full fields

Extract:
  - System.Title → symptom summary
  - System.Description → full problem statement
  - Microsoft.VSTS.TCM.ReproSteps → how to reproduce
  - System.AreaPath → identifies feature/module
  - Custom.Module → QPTM module (Scheduling, Nominations, etc.)
  - System.Tags → additional categorization
  - Relations → linked WIs (parent, child, related)
```

### Read Past Investigation Notes

```
Tool: mcp_ado_wit_list_work_item_comments
Params:
  workItemId: [work item number]

Why: Someone may have already investigated and documented findings in comments.
Check for:
  - SQL queries already run
  - Configuration already checked
  - Theories already tested (avoid repeating failed attempts)
  - Root cause already identified but not yet fixed
```

### Track What Was Already Tried

```
Tool: mcp_ado_wit_list_work_item_revisions
Params:
  workItemId: [work item number]

Why: Revision history shows state changes, reassignments, and field updates.
Check for:
  - Who investigated before and when
  - State transitions (Active → Resolved → Reopened = fix didn't work)
  - Field changes that reveal investigation progress
```

---

## Phase 2: Pattern Analysis

### Search Similar Past Work Items

```
Tool: mcp_ado_search_workitem
Params:
  searchText: "[symptom keywords]"
  $top: 10

Strategy: Use specific Quorum domain terms for better results.
```

#### Example Searches by Domain

**CAS / Scheduling Issues:**
```
"scheduled quantity incorrect CAS"
"nomination not classified transaction group"
"OAC operational available capacity wrong"
"reduction algorithm CAS"
"CANOMCLTG batch failed"
"SCREDUCE process error"
"ratchet quantity previous cycle"
```

**Nomination Issues:**
```
"nomination validation failed"
"nomination submit error"
"NNSUBMIT batch process"
"nomination cycle gas day"
"nomination detail quantity"
```

**Confirmation Issues:**
```
"confirmation variance"
"confirmed quantity mismatch"
"CFPROCESS batch"
```

**General Patterns:**
```
"[exact error message text]"           ← Best: search by exact error
"batch process failed [process code]"  ← Search by process code
"[table name] data incorrect"          ← Search by DB table
"[screen name] not showing data"       ← Search by screen name
```

### Batch-Fetch Linked Work Items

```
Tool: mcp_ado_wit_get_work_items_batch_by_ids
Params:
  ids: [comma-separated WI IDs from relations]

When: The WI has related/linked items. Fetch them all to understand the full picture.
Common link types:
  - Parent/Child → feature hierarchy
  - Related → similar issues
  - Predecessor/Successor → sequential issues
```

---

## Phase 3: Hypothesis Testing

### Search Code Across ADO Repos

```
Tool: mcp_ado_search_code
Params:
  searchText: "[class name, method name, table name, or error message]"

When to use:
  - Issue may involve code OUTSIDE the local Web repo
  - Need to find batch process implementation (Quorum.QPTM.Batch repo)
  - Need to find ClassicGUI screen code (Quorum.QPTM.ClassicGUI repo)
  - Looking for how a DB table is populated by another system
```

#### Common Cross-Repo Searches

```
# Find batch segment implementation
"QPSNomClassificationAndTransGroupingSeg"

# Find ClassicGUI screen for a config table
"PAXREF_VALD_RULE_RFS_ACTN"

# Find all consumers of a stored procedure
"sp_CAS_GetSummary"

# Find where a specific error message is thrown
"exact error message text in quotes"

# Find service implementation by interface method
"ClassifyNominations"
```

### Search Past Fix Commits

```
Tool: mcp_ado_repo_search_commits
Params:
  searchText: "[WI number, class name, or fix description]"

When: Looking for how a similar bug was resolved before.
Search by:
  - Past WI number (commits often reference WI in message)
  - Class or file name that was modified
  - Feature keywords
```

---

## Phase 4: Documentation

### Document Root Cause in Work Item

```
Tool: mcp_ado_wit_add_work_item_comment
Params:
  workItemId: [work item number]
  text: "[investigation summary - see template below]"
```

#### Comment Template

```markdown
## Root Cause Investigation

**Root Cause**: [One-sentence summary of what was wrong]

**Details**:
- Layer: [Controller / Service / DataAccess / Database / Batch / Configuration]
- File(s): [Specific files where the bug was found]
- Code: [Brief description of the code defect]

**Evidence**:
- [What data/trace confirmed this root cause]
- [Working vs broken comparison that proved it]

**Fix**:
- [What was changed]
- [Files modified]

**Verification**:
- [Test(s) added/modified]
- [Manual verification steps]

**Related WIs**: [Any linked WIs found during investigation]
```

---

## Tips

1. **Always expand=all** when getting a WI — you need relations and full fields
2. **Search by exact error message first** — it's the most precise search
3. **Check comments before investigating** — avoid duplicating work
4. **Use process codes for batch issues** — e.g., "CANOMCLTG", "SCREDUCE", "SCOAC"
5. **Search ClassicGUI repo** when you can't find configuration screens in Web repo
6. **Document your findings** even if you don't have a fix yet — it helps the next investigator
