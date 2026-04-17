# Systematic Debugging for Quorum QPTM

**An AI-powered debugging methodology that turns hours of investigation into minutes.**

---

## What Is This?

This is a Copilot skill that transforms how you debug issues in the Quorum QPTM codebase. Instead of ad-hoc searching and guessing, it enforces a proven 4-phase investigation process — and it has direct access to your Azure DevOps work items and Quorum database metadata while it does it.

**Think of it as a senior engineer who:**
- Reads the entire work item history before touching code
- Checks if someone already solved this exact problem
- Knows every grid, every batch process, every code table in the system
- Traces issues through all 6 architecture layers automatically
- Never guesses — only proposes fixes backed by evidence

---

## Quick Start

### Option 1: Slash Command
Type in GitHub Copilot Chat:
```
/systematic-debugging
```
Then describe your issue.

### Option 2: Just Ask
The skill auto-activates when you use words like:
> *"debug"*, *"investigate"*, *"root cause"*, *"error"*, *"bug"*, *"fix"*, *"troubleshoot"*, *"wrong data"*, *"not working"*, *"failed"*

**Examples:**
```
"Help me debug WI #54321 — scheduled quantities are wrong"
"Investigate why the CAS summary grid shows zero for receipt quantities"
"The CANOMCLTG batch process failed last night, help me find root cause"
"Why is the nomination not being classified into any transaction group?"
```

---

## The 4-Phase Process

Every investigation follows the same disciplined flow. No shortcuts, no guessing.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Phase 1: EVIDENCE COLLECTION                              │
│   "What exactly is broken? Show me the data."               │
│                                                             │
│   • Reads the full work item (title, repro, comments)       │
│   • Resolves screen/grid metadata from the database         │
│   • Traces the data flow through all architecture layers    │
│   • Checks configuration and app settings                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Phase 2: PATTERN ANALYSIS                                 │
│   "Has anyone seen this before?"                            │
│                                                             │
│   • Searches ADO for similar past work items                │
│   • Loads feature documentation (domain + architecture +    │
│     troubleshooting) from .prompts/                         │
│   • Compares working vs broken cases                        │
│   • Checks batch process configuration                     │
│   • Searches code across ALL repos (Web, Batch, ClassicGUI) │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Phase 3: HYPOTHESIS TESTING                               │
│   "Here's my theory. Let me prove it."                      │
│                                                             │
│   • Tests ONE variable at a time                            │
│   • Creates a failing test before writing the fix           │
│   • Enforces the 3-fix circuit breaker:                     │
│     "3 failed attempts? STOP. Reassess everything."         │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Phase 4: IMPLEMENTATION & DOCUMENTATION                   │
│   "Fix the root cause, not the symptom. Then document it."  │
│                                                             │
│   • Implements the fix following Quorum code patterns       │
│   • Verifies with tests (no regressions)                   │
│   • Documents root cause directly in the ADO work item     │
│   • Updates troubleshooting docs for future investigators   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Features

### 1. Live Azure DevOps Integration

The skill connects directly to your Azure DevOps instance. No copy-pasting, no tab-switching.

| What It Does | How It Helps |
|---|---|
| **Reads work items** | Pulls title, description, repro steps, area path — understands the issue in full context |
| **Reads comments** | Checks what others have already investigated — avoids duplicate work |
| **Searches past WIs** | Finds similar resolved bugs — "this was fixed 6 months ago in WI #41200" |
| **Searches code across repos** | Looks in Batch, ClassicGUI, and other repos you don't have open locally |
| **Finds past fix commits** | Shows exactly what code change fixed a similar issue before |
| **Documents findings** | Writes structured root cause analysis back to the work item when done |

### 2. Live Quorum Metadata Lookup

The skill queries your QPTM database metadata in real-time. It knows your system's configuration.

| What It Does | How It Helps |
|---|---|
| **Resolves grid definitions** | "CAS Summary grid" → column names, DB field mappings, underlying tables |
| **Reads registered SQL** | Shows the exact SQL query behind any screen or grid |
| **Looks up code tables** | Checks what valid values exist and what's actually in the DB |
| **Inspects picklists** | Understands dropdown configuration, filter criteria, data sources |
| **Maps batch processes** | Shows every step, parameter, and implementation class for any batch process |
| **Checks app config** | Looks up feature flags and environment-specific settings |

### 3. 6-Layer Architecture Tracing

The skill understands Quorum's full architecture stack and traces issues through every layer:

```
Layer 1: UI (Controllers, ViewModels, Views, JavaScript)
Layer 2: Service (QPTMServiceCore, IQPTMService_* interfaces)
Layer 3: Data Access (Q*DataAccess, DataAccessInterface)
Layer 4: Database (Tables, Views, Stored Procedures)
Layer 5: Batch Processing (QPS*Seg classes, Process Queue)
Layer 6: Configuration (ClassicGUI screens, PAXREF_/PACTRL_ tables)
```

It knows the naming conventions at every layer:
- `*Controller.cs` → `IQPTMService_*` → `Q*DataAccess` → `CACTRL_*` tables
- `*DOExt.cs` for extensions (never touches `CodeGen/` folders)
- `QPS*Seg` for batch segments → `tProcessQueue` for execution history

### 4. Cross-Repository Awareness

Not everything lives in the Web repo. The skill automatically:
- Searches **ClassicGUI** when a configuration screen is missing from Web
- Searches **Batch** repo when investigating batch process logic
- Checks **QFC framework** packages when the issue seems framework-level
- Uses `mcp_ado_search_code` to search any repo in your ADO organization

### 5. Feature Documentation Integration

The skill automatically loads domain knowledge from `.prompts/` docs:

```
.prompts/QUICK_REFERENCE.md           → Maps keywords to features
.prompts/domain/[feature].md          → Business rules & concepts
.prompts/architecture/[feature].md    → Code structure & data flow
.prompts/troubleshooting/[feature].md → Known issues & historical WIs
```

As you add documentation for more features (Nominations, Confirmations, Allocations, etc.), the skill automatically gets smarter about those domains.

### 6. The 3-Fix Circuit Breaker

A built-in safety mechanism:

> **If 3 fix attempts fail, the skill STOPS and forces a complete reassessment.**

This prevents the most common debugging anti-pattern: making random changes hoping something sticks. When the circuit breaker triggers, the skill:
1. Discards all assumptions
2. Returns to Phase 1 with fresh eyes
3. Questions whether the issue is even in the layer it was looking at
4. Reads the critical code path line-by-line (no skimming)

---

## Real-World Examples

### Example 1: "WI #54321 — CAS scheduled quantities are wrong"

```
Phase 1: Evidence
  → Reads WI #54321: "Receipt scheduled quantities show 0 for contract X on gas day 2026-04-10"
  → Reads WI comments: Developer A checked CASTAG table, data exists
  → Resolves CAS Summary grid metadata: maps to CACTRL_SUMMARY view
  → Gets registered SQL behind the grid: finds the query has a cycle filter

Phase 2: Patterns
  → Searches ADO: finds WI #41200 "CAS quantities missing after cycle change" — RESOLVED
  → Loads CAS troubleshooting doc: matches "Issue 2: Incorrect Scheduled Quantities"
  → Compares gas day 04-09 (works) vs 04-10 (broken): CACTRL_SCHD_HDR missing for 04-10

Phase 3: Hypothesis
  → Theory: Classification batch didn't run for 04-10
  → Evidence: tProcessQueue shows CANOMCLTG ran but with wrong cycle parameter
  → Creates test: verifies classification with correct cycle → data appears

Phase 4: Fix
  → Root cause: Batch job configuration had stale cycle ID after DST change
  → Fix: Update batch parameter configuration
  → Documents root cause in WI #54321 with full trace
```

### Example 2: "The dropdown on the nomination screen is missing a TOS code"

```
Phase 1: Evidence
  → Gets picklist details for TOS dropdown: source is code table QF_TOS_CD
  → Gets code table values: TOS code "FT-NEW" exists in code table
  → Gets picklist filter: filtered by TSP_NO and ACTIVE_FLG = 'Y'

Phase 2: Patterns
  → Checks code table values with filter: FT-NEW has ACTIVE_FLG = 'N'
  → Root cause found in Phase 2 — no need for Phase 3

Phase 4: Fix
  → Configuration issue: TOS code not activated for this TSP
  → Documents: "Activate FT-NEW in code table for TSP 42 via ClassicGUI maintenance screen"
```

### Example 3: "Batch process CANOMCLTG fails with error"

```
Phase 1: Evidence
  → Gets batch process details: QPSNomClassificationAndTransGroupingSeg, 3 steps
  → Gets process step details: Step 2 "ClassifyNominations" has a DB timeout parameter
  → Reads error from tMessage: "Timeout expired" on step 2

Phase 2: Patterns
  → Searches ADO: 4 past WIs with "CANOMCLTG timeout" — all resolved by increasing timeout
  → But this one has 500K nominations (10x normal) — different root cause

Phase 3: Hypothesis
  → Theory: Query in ClassifyNominations isn't using index for large datasets
  → Searches code via ADO: finds the SQL in Batch repo
  → SQL missing index hint for high-volume scenarios
  → Creates test with 500K rows: confirms timeout

Phase 4: Fix
  → Adds index hint to classification query
  → Documents in WI with performance comparison
```

---

## Prerequisites

### MCP Servers Required

The skill uses two MCP servers configured in `.vscode/mcp.json`:

1. **Azure DevOps MCP** (`@azure-devops/mcp`)
   - Provides: Work item search, code search, commit search, WI comments
   - Setup: `npx -y @azure-devops/mcp@latest [org-name]`
   - Auth: Azure DevOps PAT or Azure CLI login

2. **Quorum Metadata MCP** (`quorum-metadata-sup`)
   - Provides: Grid metadata, registered SQL, code tables, batch process config
   - Setup: Local executable (`QuorumMetadataMCP.exe`)
   - Config: `dbconfig.json` with environment connection strings

Both servers should be configured and running. The skill will indicate when it's trying to use a tool that isn't available.

### Feature Documentation (Optional but Recommended)

The skill gets more powerful as you add feature documentation to `.prompts/`:
- Currently documented: **Capacity Scheduling Allocations (CAS)**
- Planned: Nominations, Confirmations, Allocations, Capacity Release, Contracts, Inventory, EDI, Rates

Each feature has 3 docs (domain, architecture, troubleshooting). See `.prompts/QUICK_REFERENCE.md` for the full map.

---

## Tips for Best Results

1. **Give it a WI number** — The more context the skill has, the faster it finds root cause. A WI number unlocks title, repro steps, comments, and linked items automatically.

2. **Be specific about symptoms** — "Quantities are wrong" is okay. "Receipt scheduled quantity shows 0 instead of 5000 for contract 12345 on gas day 2026-04-10 cycle 1" is much better.

3. **Mention the screen name** — If you say "CAS Summary Maintenance," the skill immediately resolves the grid, columns, and underlying SQL.

4. **Let it trace** — Don't jump to "I think it's in the service layer." Let the skill trace through all layers systematically. The bug is often not where you think it is.

5. **Trust the circuit breaker** — If it stops after 3 attempts, that's a feature, not a failure. It means the problem needs a fundamentally different approach.

---

## File Structure

```
.github/skills/systematic-debugging/
├── SKILL.md                                    ← Main skill (auto-loaded by Copilot)
├── README.md                                   ← This file (user guide)
└── references/
    ├── ado-integration.md                      ← ADO MCP tool reference & examples
    ├── metadata-integration.md                 ← Metadata MCP tool reference & resolution chains
    └── quorum-architecture-layers.md           ← Layer tracing guide & naming conventions
```

The skill loads `SKILL.md` automatically. Reference docs are loaded progressively when the skill needs deeper detail about a specific integration.

---

## FAQ

**Q: Does this replace the Work Item Investigation workflow?**
No. The Work Item Investigation (`.prompts/WORK_ITEM_INVESTIGATION.md`) handles *understanding what the issue is*. This skill handles *finding why it happens and how to fix it*. They work together: WI Investigation → Systematic Debugging.

**Q: What if the MCP servers aren't running?**
The skill still works — it just can't auto-query ADO or metadata. It will fall back to local code search, file reading, and manual investigation. You lose the "superpowers" but keep the methodology.

**Q: Does it work for ClassicGUI issues?**
Yes. While the skill runs in the Web repo, it searches ClassicGUI code via `mcp_ado_search_code` and knows which configuration tables map to ClassicGUI-only screens.

**Q: Can it debug batch process issues?**
Yes. It queries batch process definitions, steps, and parameters via the Metadata MCP, and searches batch segment implementation code via ADO code search.

**Q: Will it modify generated code?**
Never. The skill respects the CodeGen boundary — it uses `*DOExt.cs` extension patterns and will flag if a fix attempt would touch generated files.

**Q: How does it handle environment-specific issues?**
It checks app configuration via `get_app_configuration_details` and notes that metadata results are environment-specific. If behavior differs between environments, it flags this as a configuration investigation path.
