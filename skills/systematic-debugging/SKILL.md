---
name: systematic-debugging
description: >
  Systematic debugging methodology for Quorum QPTM issues. Use when investigating bugs, errors,
  incorrect data, failed batch processes, wrong scheduled quantities, nomination issues, CAS problems,
  validation failures, or any root cause analysis. Integrates Azure DevOps work item search and
  Quorum Metadata MCP for grid/screen/batch process resolution. Follows a strict 4-phase process:
  Evidence Collection → Pattern Analysis → Hypothesis Testing → Implementation.
  Keywords: debug, investigate, root cause, error, bug, issue, fix, troubleshoot, diagnose, trace,
  wrong data, incorrect, missing, failed, broken, not working.
---

# Systematic Debugging for Quorum

## The Iron Law

> **No fix without root cause. No root cause without evidence.**

Never propose a fix based on a guess. Never claim root cause without traced evidence through the code.
If you've made 3 failed fix attempts, STOP — you don't understand the problem yet.

---

## Quick Decision Table

| Situation | Start At |
|-----------|----------|
| User gives a WI number | Phase 0 (WI Investigation) → Phase 1 |
| User describes a symptom (wrong data, error, etc.) | Phase 1 directly |
| You've already investigated and need to compare patterns | Phase 2 |
| You have a theory and need to verify | Phase 3 |
| Root cause confirmed, need to implement fix | Phase 4 |

---

## Phase 0: Work Item Investigation (Pre-Debugging)

**Trigger**: User mentions a WI number, ticket, or bug ID.

> **Follow `.prompts/WORK_ITEM_INVESTIGATION.md` for this phase.**

This skill picks up AFTER WI investigation is complete. The handoff is:
- WI Investigation gives you: **what** the issue is, **who** is affected, **where** it happens
- Systematic Debugging gives you: **why** it happens and **how** to fix it

---

## Phase 1: Evidence Collection

**Goal**: Gather all observable facts before forming any theory.

### Step 1.1: Understand the Symptom

Extract the precise symptom — not "it's broken" but:
- What value is wrong? What was expected vs actual?
- What screen/grid/report shows the problem?
- What gas day, cycle, TSP, contract, location is affected?
- What error message (exact text)?
- When did it start? What changed?

### Step 1.2: Resolve Screen & Grid Metadata

If the issue involves a UI screen or grid, resolve the metadata immediately:

```
→ mcp_quorum-metada_get_grid_all_details
  Use: Get grid ID, column definitions, underlying SQL, DB table mappings
  When: User mentions a screen name, grid, or "the data in the grid is wrong"

→ mcp_quorum-metada_get_reg_sql_details
  Use: Get the registered SQL statement behind a grid/screen
  When: You need to see what query populates a grid

→ mcp_quorum-metada_get_picklist_all_details
  Use: Get picklist/dropdown configuration
  When: Issue involves wrong dropdown values or missing options

→ mcp_quorum-metada_get_codetable_all_details / get_codetable_values
  Use: Look up code table definitions and actual values in DB
  When: Issue involves code values, status codes, type codes
```

### Step 1.3: Get Work Item Context (if not done in Phase 0)

```
→ mcp_ado_wit_get_work_item
  Use: Get full WI details (title, description, repro steps, area path, module)
  Params: id=[WI number], expand=all

→ mcp_ado_wit_list_work_item_comments
  Use: Read past investigation notes — someone may have already found the root cause
  Params: workItemId=[WI number]

→ mcp_ado_wit_list_work_item_revisions
  Use: Track what was already tried, state changes, reassignments
  Params: workItemId=[WI number]
```

### Step 1.4: Trace the Data Flow

Follow the data through Quorum's architecture layers:

```
UI Screen → Controller → Service → DataAccess → Database (→ Batch Process)
```

See [./references/quorum-architecture-layers.md](./references/quorum-architecture-layers.md) for:
- How to identify which layer owns the bug
- Naming conventions for finding the right files
- CodeGen boundaries (never modify CodeGen/ folders)

### Step 1.5: Check Configuration

```
→ mcp_quorum-metada_get_app_configuration_details
  Use: Look up app config keys/values (feature flags, settings)
  When: Behavior seems config-dependent or environment-specific
```

### Phase 1 Checklist

Before moving to Phase 2, you MUST have:
- [ ] Exact symptom with expected vs actual values
- [ ] Screen/grid metadata resolved (if UI issue)
- [ ] WI comments and history reviewed
- [ ] Data flow traced to the layer where the problem originates
- [ ] Relevant configuration checked

**If you can't check all boxes, gather more evidence. Do NOT proceed to guessing.**

---

## Phase 2: Pattern Analysis

**Goal**: Compare the broken case against working cases and historical knowledge.

### Step 2.1: Search Past Work Items

```
→ mcp_ado_search_workitem
  Use: Search for similar past WIs by symptoms, keywords, error messages
  Params: searchText="[symptom keywords]", $top=10

  Example searches:
  - "scheduled quantity incorrect CAS" — for CAS quantity issues
  - "nomination not classified" — for classification failures
  - "batch process failed CANOMCLTG" — for specific batch failures
  - "[exact error message text]" — for specific error messages
```

### Step 2.2: Load Feature Documentation

Check `.prompts/QUICK_REFERENCE.md` to map keywords → feature, then load ALL 3 docs:

```
.prompts/domain/[feature].md           — Business rules & concepts
.prompts/architecture/[feature].md     — Code structure & data flow
.prompts/troubleshooting/[feature].md  — Known issues & historical WIs
```

**Search the troubleshooting doc** for:
- Similar symptoms in "Common Issues" section
- Matching error messages
- Historical WIs with the same pattern

### Step 2.3: Check Batch Process Configuration

If the issue involves batch processing or data that batch processes produce:

```
→ mcp_quorum-metada_get_batch_process_all_details
  Use: Get batch process definition, parameters, steps, lookup tables
  When: Issue involves data produced by a batch process (CAS staging, nomination processing, etc.)

→ mcp_quorum-metada_get_process_step_all_details
  Use: Get detailed step information within a batch process
  When: Need to understand which step in a multi-step process is failing

→ mcp_quorum-metada_get_process_parameter_details
  Use: Get parameter definitions for a batch process
  When: Checking if batch was run with correct parameters
```

### Step 2.4: Compare Working vs Broken

Find a working case with similar characteristics:
- Same TSP, different gas day (that works)
- Same gas day, different contract (that works)
- Same screen, different query parameters (that works)

Compare the data at each layer to isolate where divergence begins.

### Step 2.5: Search Code Across Repos

```
→ mcp_ado_search_code
  Use: Search for relevant code in ADO repos (ClassicGUI, Batch, other repos not in local workspace)
  When: Issue may involve code outside the current Web repo (batch process logic, ClassicGUI config screens)

→ mcp_ado_repo_search_commits
  Use: Find commits that fixed similar issues in the past
  When: Looking for how a similar bug was resolved before
```

### Phase 2 Checklist

Before moving to Phase 3, you MUST have:
- [ ] Searched past WIs for similar patterns
- [ ] Loaded and reviewed feature documentation (if available)
- [ ] Compared working vs broken case (identified divergence point)
- [ ] Checked batch process configuration (if applicable)
- [ ] Identified the specific layer and code area where the bug lives

---

## Phase 3: Hypothesis Testing

**Goal**: Verify ONE theory at a time with the smallest possible change.

### The Rules

1. **One variable at a time** — Change exactly one thing. If you change two things and the fix works, you don't know which one fixed it.

2. **Smallest possible change** — If you think the issue is in a SQL query, prove it with a diagnostic query first. Don't rewrite the service layer.

3. **Failing test first** — Before implementing a fix, create a unit test that demonstrates the bug:
   ```
   [Test] method that sets up the failing scenario
   → Assert the WRONG behavior currently happening
   → This test should PASS now (proving the bug exists)
   → After fix, change assertion to expected behavior → should PASS
   ```

4. **Follow testing guidelines** — See `Quorum.QPTM.UnitTests/TESTING_GUIDELINES.md` for project-specific test patterns, build commands, and best practices.

### The 3-Fix Circuit Breaker

> **If 3 fix attempts fail, STOP. You don't understand the problem.**

When triggered:
1. Discard all assumptions
2. Return to Phase 1 with fresh eyes
3. Question the architecture — maybe the issue is in a different layer entirely
4. Read the code more carefully (don't skim, read every line in the critical path)
5. Check if the issue is actually in generated code, configuration, or a different repo

### Hypothesis Template

For each hypothesis, document:
```
HYPOTHESIS: [What you think is wrong]
EVIDENCE FOR: [Facts supporting this theory]
EVIDENCE AGAINST: [Facts that don't fit]
TEST: [How to verify — specific file, line, query, or test]
RESULT: [What happened when you tested]
```

---

## Phase 4: Implementation & Documentation

**Goal**: Fix the root cause and document everything.

### Step 4.1: Implement the Fix

- Fix the ROOT CAUSE, not the symptom
- Follow existing code patterns (see [./references/quorum-architecture-layers.md](./references/quorum-architecture-layers.md))
- Never modify `CodeGen/` folders — use `*DOExt.cs` extension pattern
- New `.cs` files MUST be added to `.csproj` manually (MSBuild doesn't auto-discover)
- Build with MSBuild, not `dotnet build`

### Step 4.2: Verify the Fix

1. Run the failing test → now passes
2. Run existing tests → no regressions
3. Verify the original symptom is resolved
4. Check edge cases identified during investigation

### Step 4.3: Document Root Cause

```
→ mcp_ado_wit_add_work_item_comment
  Use: Add investigation findings and root cause to the WI
  Include:
  - Root cause (one sentence)
  - What was wrong (specific code/config/data)
  - What was changed (files modified)
  - How it was verified (tests, manual verification)
```

### Step 4.4: Update Troubleshooting Docs

If this is a new pattern not already in the troubleshooting docs:
- Add to `.prompts/troubleshooting/[feature].md` under "Historical Work Items"
- Include: WI number, symptom, root cause, resolution
- This helps future investigations find this pattern in Phase 2

---

## Red Flags — Stop and Reassess

| Red Flag | What It Means |
|----------|---------------|
| Fix works in one environment but not another | Config difference — check `get_app_configuration_details` |
| Data looks correct in DB but wrong on screen | Grid/SQL mismatch — check `get_grid_all_details` and `get_reg_sql_details` |
| Batch process succeeds but data is wrong | Logic issue in batch segment — search batch code via `mcp_ado_search_code` |
| Issue only happens for specific TSP/contract | Configuration issue — check rule sets, TOS filters, transaction groups |
| Can't find the screen/config in Web repo | May be ClassicGUI-only — search via `mcp_ado_search_code` in ClassicGUI repo |
| Fix works but breaks something else | Incomplete understanding — return to Phase 1, map ALL consumers |
| Generated code seems wrong | NEVER modify CodeGen/ — the issue is in generation config or in your extension |

---

## Anti-Patterns

**DO NOT:**
- Propose a fix without tracing through the code
- Search for similar code patterns instead of reading the actual failing code path
- Skip reading WI comments (someone may have already found the answer)
- Ignore batch process configuration when the issue is about processed data
- Assume the bug is in the layer closest to the symptom (UI bug may be a DB issue)
- Make multiple changes at once
- Skip writing a test because "it's a simple fix"

**DO:**
- Read every error message completely
- Trace the full data flow before theorizing
- Search ADO for past WIs with similar symptoms
- Resolve grid/screen metadata before debugging UI issues
- Use the Metadata MCP to understand what's behind a screen
- Create a failing test before implementing a fix
- Document root cause in the WI

---

## MCP Tool Quick Reference

### Azure DevOps MCP (`mcp_ado_*`)

| Tool | Use Case |
|------|----------|
| `mcp_ado_wit_get_work_item` | Get WI details (expand=all for full info) |
| `mcp_ado_wit_list_work_item_comments` | Past investigation notes |
| `mcp_ado_wit_list_work_item_revisions` | State change history |
| `mcp_ado_search_workitem` | Search similar WIs by keywords |
| `mcp_ado_wit_get_work_items_batch_by_ids` | Batch-fetch linked WIs |
| `mcp_ado_search_code` | Search code across ADO repos |
| `mcp_ado_repo_search_commits` | Find past fix commits |
| `mcp_ado_wit_add_work_item_comment` | Document root cause |

### Quorum Metadata MCP (`mcp_quorum-metada_*`)

| Tool | Use Case |
|------|----------|
| `get_grid_all_details` | Grid ID, columns, DB table mappings |
| `get_reg_sql_details` | Registered SQL behind screens |
| `get_picklist_all_details` | Picklist/dropdown configuration |
| `get_codetable_all_details` | Code table definitions |
| `get_codetable_values` | Code table values from DB |
| `get_app_configuration_details` | App config key/values |
| `get_batch_process_all_details` | Batch process definition & params |
| `get_process_step_all_details` | Process step details |
| `get_process_parameter_details` | Process parameter details |
| `get_parameter_type_details` | Display control type details |
| `list_data_sources` | Available data sources |
| `refresh_cache` | Refresh metadata cache |

See detailed integration guides:
- [./references/ado-integration.md](./references/ado-integration.md)
- [./references/metadata-integration.md](./references/metadata-integration.md)
- [./references/quorum-architecture-layers.md](./references/quorum-architecture-layers.md)
