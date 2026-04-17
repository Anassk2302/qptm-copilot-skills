# Quorum Architecture Layers — Debugging Trace Guide

## Overview

This reference explains how to trace issues through Quorum's architecture layers during systematic debugging. The key principle: **trace backward from the symptom to the root cause**, crossing layer boundaries methodically.

---

## Layer Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│ LAYER 1: UI (Web)                                                  │
│   Controllers → ViewModels → Views → JavaScript/AJAX               │
├────────────────────────────────────────────────────────────────────┤
│ LAYER 2: Service                                                   │
│   QPTMServiceCore → IQPTMService_* interfaces → Business Logic     │
├────────────────────────────────────────────────────────────────────┤
│ LAYER 3: Data Access                                               │
│   Q*DataAccess → DataAccessInterface → SQL/Stored Procs            │
├────────────────────────────────────────────────────────────────────┤
│ LAYER 4: Database                                                  │
│   Tables → Views → Stored Procedures → Triggers                    │
├────────────────────────────────────────────────────────────────────┤
│ LAYER 5: Batch Processing                                          │
│   QPS*Seg classes → Batch segments → Process queue                 │
├────────────────────────────────────────────────────────────────────┤
│ LAYER 6: Configuration (ClassicGUI)                                │
│   QVp* screens → Config tables (PAXREF_*, PACTRL_*)               │
└────────────────────────────────────────────────────────────────────┘
```

---

## Backward Trace Method

**Start at the symptom layer and trace backward:**

### Symptom is in the UI
```
UI shows wrong data
  → Is the ViewModel mapping wrong? (Controller → ViewModel)
  → Is the Service returning wrong data? (Service → Controller)
  → Is the DataAccess query wrong? (DataAccess → Service)
  → Is the DB data wrong? (Database → DataAccess)
  → Was the data populated incorrectly? (Batch → Database)
  → Is the configuration wrong? (ClassicGUI Config → Batch/Service)
```

### Symptom is in batch output
```
Batch produces wrong results
  → Is the batch segment logic wrong? (QPS*Seg class)
  → Was it called with wrong parameters? (Process queue / config)
  → Is the input data wrong? (Previous batch step / manual entry)
  → Is the configuration wrong? (Rule sets, transaction groups, etc.)
```

---

## Finding Code by Layer

### Layer 1: UI / Web Controllers

**Naming conventions:**
- Controllers: `*Controller.cs` in `Quorum.QPTM.Web.Core/Controllers/`
- Base class: `QPTMInterfaceControllerBase`
- ViewModels: `*VM.cs` in `Quorum.QPTM.Web.Core/ViewModels/`
- Security: `[QScreenSecurityObject(Constants.SecurityObjectIDs.*)]`

**How to find:**
```
Screen name → Controller class
  "CAS Summary Maintenance" → CASummaryMaintenanceController
  "Nomination Detail"       → NominationDetailController
  "Confirmation Entry"      → ConfirmationEntryController

Pattern: Remove spaces, add "Controller" suffix
```

**What to check:**
- Action methods (Query, Submit, Save, etc.)
- ViewModel population (what data is returned to the UI)
- Model binding (what data is received from the UI)
- Security attribute (access control)

### Layer 2: Service Layer

**Naming conventions:**
- Main service: `QPTMServiceCore` (partial classes split by domain)
- Interfaces: `IQPTMService_*` in `Quorum.QPTM.ServiceInterface/`
- Domain services: `Quorum.QPTM.ServiceCore.[Domain]/` projects
- Container: `QPTMServiceServerContainer` for DI registration

**How to find:**
```
Controller calls → Service method
  controller.Query() → service.GetSummarySchedulings()
  controller.Submit() → service.SubmitSchedulings()
  controller.PrelimCut() → service.DoPrelimCut()

Pattern: Controller action → IQPTMService_* interface method → QPTMServiceCore implementation
```

**Domain-specific service projects:**
| Domain | Project | Key Classes |
|--------|---------|-------------|
| Scheduling/CAS | `ServiceCore.Scheduling` | QPTMSchedulingService, TransGroupMgr |
| Nominations | `ServiceCore.Nomination` | NominationService |
| Allocations | `ServiceCore.Allocation` | AllocationService |
| Capacity Release | `ServiceCore.CapacityRelease` | CapacityReleaseService |
| Inventory | `ServiceCore.Inventory` | InventoryService |
| EDI | `ServiceCore.EDI` | EDIService |
| Rates | `ServiceCore.Rate` | RateService |
| RFS | `ServiceCore.RFS` | RFSService |

### Layer 3: Data Access

**Naming conventions:**
- Data access: `Q*DataAccess` in `Quorum.QPTM.DataAccess/`
- Interfaces: `Quorum.QPTM.DataAccessInterface/`
- Data objects: `*DOExt.cs` in `Quorum.QPTM.DataObject/`
- Helpers: `*CalcHelper`, `*Helper` classes

**How to find:**
```
Service calls → DataAccess class
  QPTMSchedulingService → QSchedulingDataAccess
  NominationService     → QNominationDataAccess
  AllocationService     → QAllocationDataAccess

Pattern: Service domain name → Q[Domain]DataAccess
```

**Critical rule:** Data objects have two file types:
- `*DO.cs` / `CodeGen/*.cs` — **AUTO-GENERATED. NEVER MODIFY.**
- `*DOExt.cs` — Extension files with business logic. **Edit these.**

### Layer 4: Database

**Naming conventions:**
| Prefix | Meaning | Example |
|--------|---------|---------|
| `CACTRL_` | CAS control tables | `CACTRL_SCHD_HDR`, `CACTRL_SUMMARY` |
| `CASTAG_` | CAS staging tables | `CASTAG_OBJ_NOM_TRANS_GRP` |
| `NNCTRL_` | Nomination control | `NNCTRL_NOM_HDR`, `NNCTRL_NOM_DTL` |
| `PACTRL_` | Pipeline admin control | `PACTRL_RULE_SET`, `PACTRL_TRANS_GRP` |
| `PAXREF_` | Pipeline admin cross-ref | `PAXREF_TRANS_GRP_TOS` |
| `SYSTBL_` | System tables | `SYSTBL_USER_BP_ACCESS` |

**How to find the right table:**
1. Use `mcp_quorum-metada_get_grid_all_details` to find the grid → DB table mapping
2. Use `mcp_quorum-metada_get_reg_sql_details` to find the SQL behind a screen
3. Follow DataAccess code to see which tables/procs are used

### Layer 5: Batch Processing

**Naming conventions:**
- Segments: `QPS*Seg` classes (e.g., `QPSNomClassificationAndTransGroupingSeg`)
- Process codes: Short codes like `CANOMCLTG`, `SCREDUCE`, `SCOAC`
- Queue table: `tProcessQueue` (check batch execution history)
- Message table: `tMessage` (check batch errors)

**How to find:**
```
Process code → Batch segment class
  CANOMCLTG → QPSNomClassificationAndTransGroupingSeg
  SCREDUCE  → QPSSchedulingReductionSeg
  SCOAC     → QPSStageOperationalAvailableCapacity

Use: mcp_quorum-metada_get_batch_process_all_details
  → Returns process definition, steps, and implementation classes
```

**Batch code is often NOT in the Web repo.** Search ADO:
```
→ mcp_ado_search_code
  searchText: "QPSNomClassificationAndTransGroupingSeg"
  → Find in Quorum.QPTM.Batch or related repos
```

### Layer 6: Configuration (ClassicGUI)

**When to check ClassicGUI:**
- Configuration screen not found in Web repo
- Tables with `PAXREF_*` prefix (cross-reference config tables)
- Rule set assignments, transaction group configuration
- Validation rule cross-references
- Code table maintenance

**How to find:**
```
→ mcp_ado_search_code
  searchText: "[table name]" in Quorum.QPTM.ClassicGUI repo
  
ClassicGUI patterns:
  Screens: QVp*.cpp / QVp*.h
  Tabs: QVp*_TabName.cpp
  Grid pattern: QDBVpTabAvailSel = Available/Selected grid
```

---

## CodeGen Boundaries

### CRITICAL: Never Modify Generated Code

```
Files in CodeGen/ folders     → AUTO-GENERATED. NEVER TOUCH.
Files ending in DO.cs         → Often generated. Check before editing.
Files ending in DOExt.cs      → SAFE TO EDIT. This is the extension pattern.
```

**If the bug appears to be in generated code:**
1. The issue is likely in the generation configuration, not the generated output
2. Look for `*DOExt.cs` that may override or extend the generated behavior
3. The fix should be in the extension file, not the generated file

---

## Cross-Repo Debugging

### When the Web Repo Isn't Enough

| Clue | Check This Repo |
|------|----------------|
| Batch process logic | `Quorum.QPTM.Batch` (via `mcp_ado_search_code`) |
| Config screen missing in Web | `Quorum.QPTM.ClassicGUI` (via `mcp_ado_search_code`) |
| QFC framework issue | `Quorum.QFC.*` packages (check NuGet sources) |
| EDI processing logic | `Quorum.QPTM.Batch` EDI segments |
| Shared stored procedures | Database scripts in deployment repos |

### Cross-Repo Search Strategy

```
1. Search local workspace first (grep_search, semantic_search)
2. If not found locally, search ADO repos:
   → mcp_ado_search_code searchText="[class/method/table name]"
3. Check commits for recent changes:
   → mcp_ado_repo_search_commits searchText="[WI number or keyword]"
```

---

## `.prompts/` Documentation Lookup

When investigating a feature, load the feature documentation:

```
Step 1: Check .prompts/QUICK_REFERENCE.md
  → Map keywords from the issue to a feature name

Step 2: Load ALL 3 docs for the feature (in parallel):
  → .prompts/domain/[feature].md         — Business rules & concepts
  → .prompts/architecture/[feature].md   — Code structure & data flow
  → .prompts/troubleshooting/[feature].md — Known issues & past WIs

Step 3: Search troubleshooting doc for:
  → Similar symptoms in "Common Issues"
  → Matching error messages
  → Historical WIs with same pattern
```

**Currently documented features:**
- Capacity Scheduling Allocations (CAS): `capacity-scheduling-allocations.md`
- Other features: Planned (see QUICK_REFERENCE.md for status)

---

## Common Debugging Shortcuts

| I need to find... | Do this |
|-------------------|---------|
| Controller for a screen | Search `*Controller.cs` in `Web.Core/Controllers/` |
| Service method | Search `IQPTMService_` interfaces in `ServiceInterface/` |
| Data access for a domain | Search `Q*DataAccess` in `DataAccess/` |
| DB table behind a grid | `mcp_quorum-metada_get_grid_all_details` |
| SQL behind a screen | `mcp_quorum-metada_get_reg_sql_details` |
| Batch process definition | `mcp_quorum-metada_get_batch_process_all_details` |
| Config screen in ClassicGUI | `mcp_ado_search_code` in ClassicGUI repo |
| Past fix for similar issue | `mcp_ado_search_workitem` + `mcp_ado_repo_search_commits` |
| Extension file for a DO | Search `*DOExt.cs` in `DataObject/` |
| Validation rules | Search `Validations.Rules.*` projects |
| Event handlers | Search `Events.*` projects |
