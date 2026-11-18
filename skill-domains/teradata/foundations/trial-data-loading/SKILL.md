---
name: trial-data-loading
description: >
  Use this skill whenever a user wants to find, explore, or load demo/sample datasets
  into a Teradata Trial or ClearScape Analytics Experience environment. This skill is
  SPECIFIC to Teradata Trial / ClearScape Analytics Experience environments — do NOT
  trigger for generic Teradata environments, other databases, or non-Teradata systems.
  Trigger phrases include: "load demo data", "find a dataset", "hydrate the database",
  "what data is available", "get sample data", "load some data to explore",
  "what datasets can I use", "I have a Teradata Trial", "ClearScape Analytics",
  "ClearScape experience", or any time a user wants to populate a fresh Teradata Trial
  environment before running analytics, demos, or ML pipelines.
  Also triggers when a user asks about demo_user, get_data, or ClearScape/Trial demo datasets.
  Use a connected database MCP (any tool that can execute SQL against Teradata) to run queries.
---

# Teradata Demo Data Hydration

This skill guides the agent through discovering available demo datasets and loading them
into a Teradata Trial (formerly ClearScape Analytics Experience) environment using the
`demo_user.get_data()` stored procedure.

## Overview

**This skill is exclusively for Teradata Trial / ClearScape Analytics Experience environments.**
These are Teradata-hosted cloud trial environments available at
[https://www.teradata.com/getting-started/demos/clearscape-analytics](https://www.teradata.com/getting-started/demos/clearscape-analytics).
This hydration framework does NOT exist in standard Teradata on-prem installations,
customer-managed VantageCloud deployments, or any other database platform.

These environments ship with a `demo_user` database containing a `get_data` stored
procedure that hydrates datasets from Google Cloud Storage on demand. A registry of
all available datasets lives in `gs_tables_db.ddl`.

**Typical use case:** A user has a fresh Teradata Trial environment with a database MCP
connected (e.g. tdsql-dev, teradata-mcp, or any MCP that can execute SQL against Teradata)
and wants to load data before running analytics, demos, or ML pipelines.

**MCP compatibility:** This skill works with any MCP server that can execute SQL against
Teradata — the tool names may vary (execute_query, execute_statement, run_sql, query, etc.).
Use whatever SQL execution tool is available in the current session.

---

## Activation

Invoke this skill when a user asks to:
- Find or load demo/sample datasets in a Teradata Trial or ClearScape Analytics Experience environment
- Discover what data is available to explore in their trial environment
- Hydrate a fresh Teradata Trial before running analytics, demos, or ML pipelines
- Use `demo_user`, `get_data`, or anything related to ClearScape demo datasets

Do NOT invoke for generic Teradata environments, standard on-prem installations, or
customer-managed VantageCloud deployments — the `get_data` framework only exists in
Teradata Trial / ClearScape Analytics Experience.

---

## Response Style

Default to plain English. Most users arriving at a Teradata Trial are exploring for the
first time and are not SQL or database experts.

**Opening each interaction:**
Briefly explain what you're about to do in one sentence before running any SQL. After
getting results, describe what they mean before showing raw output.

**Technical depth:**
Ask once, early: *"Would you like me to show the SQL as we go, or just show you the results?"*
- **Show SQL** when: the user asks for it, something fails, or they're doing hands-on setup.
- **Hide SQL** otherwise — summarize the action and its outcome.

**When in doubt:** assume the user wants to find the right dataset and start querying,
not learn stored procedure syntax.

---

## Step 1 — Verify Connectivity

**Before anything else**, confirm you have a way to execute SQL against Teradata. Check
for available tools:
- An MCP server with SQL execution capability (`execute_query`, `base_readQuery`,
  `execute_statement`, `run_sql`, `query`, or similar)
- A CLI tool (`bteq`, `tq`) with known connection parameters
- A programming library (`teradatasql`, JDBC `terajdbc`) with connection parameters

If more than one option is available, clarify the target system with the user.

---

## Step 2 — Verify This Is a Teradata Trial / ClearScape Environment

**This is a critical gate.** Run both checks before proceeding.

**2a. Confirm the environment type with the user.**
If they haven't explicitly said so, ask:
> "Just to confirm — are you working in a Teradata Trial or ClearScape Analytics Experience
> environment? This data hydration process is specific to those environments and won't work
> on standard Teradata installations."

If the user says no or is unsure, **stop here** and explain that this skill only applies to
Teradata Trial / ClearScape Analytics Experience environments, available at
teradata.com/getting-started. Do not proceed with any SQL.

**2b. Verify the framework exists in the database:**

```sql
SELECT DatabaseName, TableName, TableKind
FROM DBC.TablesV
WHERE DatabaseName = 'demo_user' AND TableName = 'get_data';
```

- **If found:** proceed to Step 3.
- **If not found:** The environment may not have fully initialized, or it may not be a
  Trial environment. Inform the user and suggest they verify at teradata.com/getting-started.

---

## Step 3 — Discover Available Datasets and What's Already Loaded

Run both queries together to cross-reference what's available vs. what's already present:

```sql
-- All available datasets in the registry
SELECT DISTINCT databasename
FROM gs_tables_db.ddl
ORDER BY databasename;
```

```sql
-- Datasets already loaded in this environment
SELECT DatabaseName
FROM DBC.DatabasesV
WHERE DatabaseName LIKE 'DEMO_%'
ORDER BY DatabaseName;
```

Use the second result to build an "already loaded" set. When presenting options to the
user in Step 4, note which datasets are already available so they can use them immediately.

Each registry name ends in `_local` or `_cloud`:
- **`_local`** — copies data into local Teradata tables (faster queries, uses local perm space)
- **`_cloud`** — creates foreign tables pointing to GCS (no local storage used, slower queries)

Recommend `_local` unless the user has storage constraints or explicitly wants cloud access.

See `reference/dataset-catalog.md` for the full categorized catalog to help users browse.

---

## Step 4 — Guide the User to the Right Dataset

Ask the user what kind of data or use case they have in mind, then match to a category:

| Domain | Example Datasets |
|--------|-----------------|
| 🏥 Healthcare | HealthcareFWA, HeartFailure, CancerPrediction, DiabetesPrediction, HospitalReadmission |
| 🏦 Financial | BankChurn, CreditCard, CreditRisk, GLM_Fraud, Insurance |
| 🛍️ Retail | Retail, RetailJourney, Grocery_Data, CustomerReviews, MarketingCamp |
| 📡 Telco / IoT | Telco, TelcoNetwork, 5G, IndoorSensor, EVCarBattery |
| 🚕 Transportation | AustinBikeShare, NYCTaxi, TrainDelay, AirPassengers |
| 🤖 ML / AI | BankChurnIVSM, AnomalyDetection, ModelOps, NLP, SLM_RAG |
| 📊 Forecasting | SalesForecasting, DemandForecast, ProphetSTO, UAF |
| 🏭 Manufacturing | GreenManufacturing, PredictiveMaintenance, Secom |
| 🏀 General / Fun | BasketBall, TitanicSurvival, HousingPrices, Car, TPCH_1Gb |

When a user describes their use case, suggest 2-3 relevant datasets and briefly explain
what each contains before asking them to choose.

---

## Step 5 — Load the Dataset (if not already present)

Before calling `get_data`, check whether the dataset is already loaded. A dataset named
`DEMO_AustinBikeShare_local` creates `DEMO_AustinBikeShare_db`:

```sql
SELECT DatabaseName
FROM DBC.DatabasesV
WHERE DatabaseName = 'DEMO_<DatasetName>_db';
```

- **If the database exists:** skip the load — inform the user it's already available and
  proceed to Step 6.
- **If not found:** load it:

```sql
CALL demo_user.get_data('DEMO_AustinBikeShare_local');
```

**Important notes:**
- The procedure **requires exactly one argument** — calling with no args throws an error.
- A successful call returns `status: success, rowcount: 0` — this is normal.
- The procedure creates **two databases**: `DEMO_<name>` (views) and `DEMO_<name>_db` (raw tables).
- After loading, verify with:

```sql
SELECT DatabaseName, CommentString
FROM DBC.DatabasesV
WHERE DatabaseName LIKE 'DEMO_%'
ORDER BY DatabaseName;
```

---

## Step 6 — Confirm What Was Loaded

List the tables in the new database so the user can see what's available:

```sql
SELECT TableName, TableKind, CommentString
FROM DBC.TablesV
WHERE DatabaseName = 'DEMO_<DatasetName>_db'
  AND TableKind = 'T'
ORDER BY TableName;
```

Then describe the key tables and their columns so the user can get started immediately.
Use `DBC.ColumnsV` to show column details for the most important tables if needed.

---

## Step 7 — Suggest Next Steps

After confirming the load, suggest what the user can do next based on the dataset:

- **Run exploratory queries** — row counts, distributions, sample data
- **Join to weather/reference data** (e.g., AustinBikeShare includes a Weather table)
- **Apply ML functions** — if the dataset is ML-oriented, point to Teradata analytic functions
- **Build a dashboard or report** — offer to create a visualization

---

## Multiple Datasets

Users can load multiple datasets — each `CALL get_data(...)` is independent. There is no
batch-load option; call the procedure once per dataset name.

---

## Removing a Dataset

```sql
CALL demo_user.remove_data('DEMO_AustinBikeShare_local');
```

This drops the databases created by the corresponding `get_data` call.

---

## Reference Files

| Need | Load this file |
|------|----------------|
| Full categorized list of all ~180 available datasets with descriptions | `reference/dataset-catalog.md` |
| Ready-to-paste prompts and domain-specific variants for users | `reference/prompt-runbook.md` |
