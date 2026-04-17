# Quorum Metadata MCP Integration — Systematic Debugging

## Overview

The Quorum Metadata MCP (`quorum-metadata-sup`) provides tools to resolve UI screen/grid definitions, registered SQL, code tables, batch process configuration, and app settings directly from the QPTM database. This is essential for debugging UI issues, understanding what's behind a screen, and tracing batch process behavior.

**MCP Server**: `quorum-metadata-sup` (configured in `.vscode/mcp.json`)
**Tool Prefix**: `mcp_quorum-metada_*`
**Environment**: Database-specific (currently `MSSQL_CORE_HD_UPS_DEVA1` per mcp.json)

> **Note**: Metadata results are environment-specific. If an issue is env-dependent, verify against the correct environment's metadata.

---

## Resolution Chains

The Metadata MCP tools form resolution chains — start with what you know and drill down:

### Chain 1: Screen → Grid → Columns → DB Table

```
User says: "The CAS Summary grid shows wrong quantities"

Step 1: Get grid details
  → mcp_quorum-metada_get_grid_all_details
  → Input: grid name or partial name (e.g., "CASummary")
  → Returns: Grid ID, all column definitions, underlying table/view, column-to-DB mappings

Step 2: Get the SQL behind the grid
  → mcp_quorum-metada_get_reg_sql_details
  → Input: SQL key or statement name from grid details
  → Returns: Full registered SQL statement, parameters, source tables

Step 3: Now you know:
  - Which DB table/view the grid reads from
  - Which columns map to which DB fields
  - What SQL query populates the data
  → Trace the issue: is the SQL wrong? Is the DB data wrong? Is the mapping wrong?
```

### Chain 2: Dropdown → Picklist → Code Table → Values

```
User says: "The dropdown is missing an option" or "Wrong options in the dropdown"

Step 1: Get picklist configuration
  → mcp_quorum-metada_get_picklist_all_details
  → Input: picklist name or field name
  → Returns: Picklist ID, source (code table or custom SQL), filter criteria

Step 2: Get code table definition
  → mcp_quorum-metada_get_codetable_all_details
  → Input: code table name from picklist source
  → Returns: Code table structure, columns, relationships

Step 3: Get actual values in the DB
  → mcp_quorum-metada_get_codetable_values
  → Input: code table name
  → Returns: Live values from the database

Step 4: Now you know:
  - What values SHOULD appear (code table definition)
  - What values DO appear (live DB query)
  - Why a value is missing (filter criteria, inactive flag, etc.)
```

### Chain 3: Batch Process → Steps → Parameters

```
User says: "The batch process produced wrong data" or "CANOMCLTG failed"

Step 1: Get batch process definition
  → mcp_quorum-metada_get_batch_process_all_details
  → Input: process code (e.g., "CANOMCLTG", "SCREDUCE", "SCOAC")
  → Returns: Process definition, all steps, parameters, lookup tables, dependencies

Step 2: Get specific step details
  → mcp_quorum-metada_get_process_step_all_details
  → Input: process code + step number
  → Returns: Step implementation class, parameters, execution order, dependencies

Step 3: Get parameter definitions
  → mcp_quorum-metada_get_process_parameter_details
  → Input: parameter name or process code
  → Returns: Parameter type, default value, valid range, lookup source

Step 4: Get parameter type/display details
  → mcp_quorum-metada_get_parameter_type_details
  → Input: display control type code
  → Returns: Display control configuration (QARCH_CODE_DISPLAY_CTRL)

Step 5: Now you know:
  - What the batch process is supposed to do (step by step)
  - What parameters it expects
  - What class implements each step (→ search in code)
  - Whether it was run with correct parameters
```

---

## Tool Reference

### `mcp_quorum-metada_get_grid_all_details`

**Use**: Resolve grid/screen IDs, column definitions, DB table mappings
**When**: Debugging UI display issues, wrong data in grids, column mapping problems
**Returns**: QGridBase data — grid ID, columns, DB field mappings, sort order, visibility

**Common scenarios:**
- "Data shows wrong in the grid" → Check column-to-DB field mappings
- "Column is missing" → Check grid column visibility configuration
- "Grid shows stale data" → Check underlying SQL/view

---

### `mcp_quorum-metada_get_reg_sql_details`

**Use**: Get registered SQL statements (QSqlStatementKey/QSqlStatementBase)
**When**: Need to see the exact SQL that populates a grid or screen
**Returns**: SQL text, parameters, source tables/views

**Common scenarios:**
- "Query returns wrong results" → Read the actual SQL, check WHERE clause
- "Data is filtered incorrectly" → Check SQL parameters and conditions
- "Performance is slow on this screen" → Examine the SQL for N+1 or missing indexes

---

### `mcp_quorum-metada_get_picklist_all_details`

**Use**: Get picklist/dropdown field configuration (QPickFieldFor)
**When**: Dropdown has wrong values, missing options, or incorrect filtering
**Returns**: Picklist source (code table or SQL), filter criteria, dependencies

---

### `mcp_quorum-metada_get_codetable_all_details`

**Use**: Get code table definitions (QFCodeTable/QFCodeTableFor)
**When**: Need to understand what valid code values exist
**Returns**: Code table structure, columns, relationships, display format

---

### `mcp_quorum-metada_get_codetable_values`

**Use**: SELECT actual code table values from the database
**When**: Need to see live data — what values are currently in the DB
**Returns**: Current rows in the code table

**Common scenarios:**
- "Status code not recognized" → Check if code exists in code table
- "New type not appearing" → Check if it's been added to the code table
- "Value shows as blank" → Check code table for display text mapping

---

### `mcp_quorum-metada_get_app_configuration_details`

**Use**: Look up application configuration key/values
**When**: Behavior seems config-dependent or environment-specific
**Returns**: Config key, value, description

**Common scenarios:**
- "Feature works in DEV but not UAT" → Check config differences
- "Ratcheting not enabled" → Check ratchet config flag
- "Auto-balance not working" → Check auto-balance config setting

---

### `mcp_quorum-metada_get_batch_process_all_details`

**Use**: Get full batch process definition including steps, params, and lookup tables
**When**: Batch process fails, produces wrong data, or needs configuration review
**Returns**: Process definition, steps, parameters, dependencies

---

### `mcp_quorum-metada_get_process_step_all_details`

**Use**: Get details of a specific process step within a batch process
**When**: Narrowed issue to a specific step in a multi-step batch process
**Returns**: Step class, parameters, execution order, step-specific configuration

---

### `mcp_quorum-metada_get_process_parameter_details`

**Use**: Get parameter definitions (QARCH_CTRL_PARAM)
**When**: Checking if batch was run with correct parameters
**Returns**: Parameter type, default value, valid range, lookup source

---

### `mcp_quorum-metada_get_parameter_type_details`

**Use**: Get display control type details (QARCH_CODE_DISPLAY_CTRL)
**When**: Understanding how a parameter is presented/validated in the UI
**Returns**: Display control configuration

---

### `mcp_quorum-metada_list_data_sources`

**Use**: List available database data sources/environments
**When**: Need to verify which environment metadata is coming from
**Returns**: List of configured data sources

---

### `mcp_quorum-metada_refresh_cache`

**Use**: Refresh the metadata cache
**When**: Metadata seems stale or you've made DB changes that should be reflected
**Returns**: Cache refresh confirmation

---

### `mcp_quorum-metada_get_cache_stats`

**Use**: Check cache statistics
**When**: Diagnosing metadata staleness or performance issues
**Returns**: Cache hit/miss rates, age of cached data

---

## Debugging Patterns by Symptom

### "Grid shows wrong data"
1. `get_grid_all_details` → identify grid and column mappings
2. `get_reg_sql_details` → read the SQL that populates the grid
3. Compare SQL results vs expected data
4. Check if SQL has correct WHERE clause / parameters

### "Dropdown is empty or has wrong values"
1. `get_picklist_all_details` → identify picklist source
2. `get_codetable_all_details` → check code table definition
3. `get_codetable_values` → check actual values in DB
4. Compare expected vs actual values

### "Batch process produces wrong results"
1. `get_batch_process_all_details` → understand process structure
2. `get_process_step_all_details` → identify which step produces the data
3. `get_process_parameter_details` → verify parameters are correct
4. Search for step implementation class in code

### "Feature behaves differently per environment"
1. `get_app_configuration_details` → compare config across environments
2. `list_data_sources` → verify which environment you're querying
3. `get_codetable_values` → compare code table data across environments
