---
title: "Synapse → Fabric Warehouse: The Migration Checklist I Wish I'd Had"
date: 2026-11-17
draft: false
description: "What we had to change when migrating an enterprise data warehouse from Azure Synapse Dedicated SQL Pool to Microsoft Fabric Warehouse."
tags: ["Microsoft Fabric", "Fabric Warehouse", "Synapse", "Migration", "T-SQL"]
series: ["Fabric Warehouse in Practice"]
---

*Fabric Warehouse in Practice #1*

We migrated an enterprise data warehouse from Azure Synapse Dedicated SQL Pool to Microsoft Fabric Warehouse, using the Fabric Migration Assistant plus a set of our own scripts. Fabric Warehouse is highly compatible with Synapse, and the T-SQL looks almost identical, so it's tempting to copy the DDL across and fix whatever breaks. Here's the checklist we ended up with, so you can fix things up front instead. Some of it applies to SQL Server → Fabric migrations too, not just Synapse.

## 0. Before you run Migration Assistant

The Migration Assistant converts what it can see: the DDL in your database project. Before and after it, we still needed a set of Python scripts.

**Clean up first.** Migration is the cheapest moment to drop dead code. We went through sandbox schemas and removed everything nobody used *before* building the project. Less to convert, less to test.

**Fix what the tool misses.** A few patterns weren't covered (at the time of our migration), so we pre-processed them with regex:

- `CAST(x AS NVARCHAR)` **without a length** (the default is 30, so we used `VARCHAR(60)`)
- `NVARCHAR(MAX)` → `VARCHAR(MAX)`
- `SYSNAME` columns and variables
- Synapse-only statements such as `SET RESULT_SET_CACHING OFF`

## 1. Collation: decide before you create the warehouse

This one comes before any DDL. Our Synapse pool ran on a **case-insensitive** collation (`SQL_Latin1_General_CP1_CI_AS`). Fabric Warehouse defaults to `Latin1_General_100_BIN2_UTF8`, which is **case-sensitive**:

```sql
SELECT * FROM dbo.Customer WHERE Status = 'active';  -- won't find 'Active'
```

On the default collation, every filter, join on a text key and `GROUP BY` over text can quietly return different results than in Synapse. For a migration, you almost certainly want `Latin1_General_100_CI_AS_KS_WS_SC_UTF8`.

The catch: collation is set **when the warehouse is created**. When we started, the only way was the REST API. Now it's a workspace setting, so set it *before* you create the warehouse, not after you notice the first missing rows. It can't be changed later. Same old lesson as with SQL Server installations: **collation is a day-zero decision.**

## 2. Data types: what to convert

| Synapse | Fabric Warehouse | Why |
|---|---|---|
| `NVARCHAR(n)` | `VARCHAR(2*n)` (what Migration Assistant generates) | No Unicode types. Strings are stored as UTF-8 |
| `NCHAR(n)` | `CHAR(n)` (we deliberately did **not** follow the 2×n rule here, see below) | Same reason |
| `DATETIME` / `SMALLDATETIME` | `DATETIME2(6)` | Not supported as column types |
| `DATETIMEOFFSET` | `DATETIME2(6)` | Not supported, and **the time zone offset is lost**: convert to UTC first |
| `DATETIME2(7)` | `DATETIME2(6)` | Maximum precision is 6, and bare `DATETIME2` fails too |
| `TIME` | `TIME(6)` | Same precision rule |
| `TINYINT` | `SMALLINT` / `INT` | Not supported |
| `XML`, `TEXT`, `NTEXT` | `VARCHAR(MAX)` | Not supported. For `XML`, you also lose the XML functions |

⚠️ **Don't trust the 2×n rule blindly, even though Migration Assistant uses it.** `NVARCHAR(n)` counts UTF-16 code units, `VARCHAR(n)` counts UTF-8 bytes, and the ratio depends on the character:

- ASCII → 1 byte
- Accented Latin letters (`č`, `ü`) → 2 bytes
- **`€`, `„“`, `–`, `…`, `™`** → **3 bytes** for a single UTF-16 unit

Text pasted from Word or email is full of these. A column that was full in Synapse can fail to load in Fabric after the "correct" automatic conversion. The only guaranteed-safe size is `3*n`, but `NVARCHAR(4000)` then exceeds the 8000-byte limit and becomes `VARCHAR(MAX)`. Treat the generated DDL as a starting point and **adjust the lengths if necessary**.

💡 **For `NCHAR`, we went the other way: `NCHAR(n)` → `CHAR(n)`.** `CHAR` is fixed-length, so applying 2×n would pad every value: a currency code in `CHAR(6)` is stored as `'CZK   '`. Comparisons ignore trailing spaces, but exports, concatenations and `LEN` vs. `DATALENGTH` checks don't. Our `NCHAR` columns held ASCII codes (currencies, countries, flags), so `CHAR(n)` fits exactly. If an `NCHAR` column can contain non-ASCII characters, `CHAR(n)` is too short, so check the data first.

## 3. Watch for types you didn't declare

Converting table DDL isn't enough. In `SELECT INTO` and CTAS, Fabric infers column types from the **expressions**, and some built-in functions return types Fabric can't store:

```sql
CREATE TABLE #a AS SELECT GETDATE() AS a;
-- Msg 24574: The data type 'datetime' in column 'a'
-- is not supported in this edition of SQL Server.
```

`GETDATE()` returns `DATETIME`, and so does `DATEADD(day, -1, GETDATE())`, because `DATEADD` keeps the type of its input. We wrapped every such expression in an explicit cast:

```sql
CREATE TABLE #a AS SELECT CAST(GETDATE() AS DATETIME2(6)) AS a;  -- OK
```

The same applies to `JSON_VALUE()` (returns `nvarchar`) and to `.name` columns from catalog views (`sysname`). Rule of thumb: **every expression in `SELECT INTO` / CTAS gets an explicit CAST.**

## 4. Remove distributions, partitions and indexes

`DISTRIBUTION = HASH(...)`, `CLUSTERED COLUMNSTORE INDEX` and partitioning clauses are gone. Fabric manages storage itself (Delta/Parquet). Delete them from the DDL; you don't need to translate them into anything else. The same goes for query hints like `WITH (NOLOCK)`. We removed them all.

The one exception is **partitioning on large tables**. Where we relied on it for performance (typically date-based fact tables), we replaced it with **data clustering**, Fabric's way of co-locating rows with similar values so queries filtering on those columns read less data:

```sql
CREATE TABLE dbo.FactSales (...)
WITH (CLUSTER BY (TransactionDate));
```

## 5. Identity and surrogate keys

When we started the migration, Fabric supported `IDENTITY`, but **you couldn't insert explicit values into an identity column**. That ruled it out for us: a migration has to carry over existing surrogate keys 1:1, otherwise all references in facts and dimensions break. So we kept generating keys in our own ETL logic.

Since then, `SET IDENTITY_INSERT` has been supported, so if you're starting a migration today, `IDENTITY` is a valid option again. Just note that **identity columns must be `BIGINT`** ([docs](https://learn.microsoft.com/en-us/fabric/data-warehouse/identity)). That matches the rule we followed anyway: **all surrogate keys in new tables are `BIGINT`**, so there's no mix of int and bigint across joins.

## 6. Distributed vs. non-distributed: temp tables and system views

Fabric has two kinds of `#temp` tables, and **the default is not the one you usually want**:

```sql
-- Non-distributed (mdf-backed): what you get without the WITH clause
CREATE TABLE #t (Id BIGINT, Name VARCHAR(200));

-- Distributed (Parquet-backed): requires explicit syntax
CREATE TABLE #t (Id BIGINT, Name VARCHAR(200))
WITH (DISTRIBUTION = ROUND_ROBIN);
```

The Synapse habit of writing `CREATE TABLE #t (...)` silently gives you the non-distributed one, and that one has real limits: **`INSERT INTO #t SELECT ...` is supported only on distributed temp tables**, and its size is limited. Global `##temp` tables don't exist at all, and you can't create a view over a temp table. We add `WITH (DISTRIBUTION = ROUND_ROBIN)` to every temp table and use non-distributed ones only rarely.

**System catalog views live on the non-distributed side too.** `sys.*` objects can't be combined with your user tables in one query: no joins, and no `INSERT INTO dbo.MyTable SELECT ... FROM sys.objects` either. That breaks anything that reads metadata *into* tables, such as permission audits or object inventories. Our workaround: **copy the system views into regular user tables with a pipeline** (a Copy activity reads the catalog view and writes to a warehouse table), then join those.

**The trap (new in Fabric):** in Synapse, a procedure could create the same temp table in two different `IF` branches. In Fabric it fails, even though only one branch can ever run:

```sql
-- Fails
IF @Mode = 1
    CREATE TABLE #t (...) WITH (DISTRIBUTION = ROUND_ROBIN);
ELSE
    CREATE TABLE #t (...) WITH (DISTRIBUTION = ROUND_ROBIN);

-- Works: create once, fill in the branches
CREATE TABLE #t (...) WITH (DISTRIBUTION = ROUND_ROBIN);
IF @Mode = 1 INSERT INTO #t SELECT ...;
ELSE         INSERT INTO #t SELECT ...;
```

## 7. External tables → views + OPENROWSET

Fabric Warehouse has no external tables. We replaced them with views over `OPENROWSET`, which read files from OneLake or storage directly.

## 8. Moving the data: you're on your own

Migration Assistant moves the **schema**, not the data. For the data itself, you have several options: a **Fabric Copy job**, a **custom Fabric pipeline**, or a combination of exports and SQL.

What we did:

1. **Export from Synapse to Parquet files in ADLS Gen2** with a pipeline. Why not Delta? Synapse pipelines can't write Delta without a Spark job, and Parquet was simple and fast enough for a one-off transfer.
2. **Shortcut** from a Fabric Lakehouse to the exported data in ADLS. No copying, the data is immediately visible in OneLake.
3. **`INSERT INTO ... SELECT FROM OPENROWSET`** in the Warehouse, reading the Parquet files through the shortcut:

```sql
INSERT INTO dbo.FactSales
SELECT *
FROM OPENROWSET(
    BULK 'https://onelake.dfs.fabric.microsoft.com/<workspace>/<lakehouse>.Lakehouse/Files/export/FactSales/*.parquet',
    FORMAT = 'PARQUET'
) AS src;
```

Step 3 was **surprisingly fast**. The bottleneck was step 1, the export out of Synapse. For the largest tables, we **partitioned the export in the pipeline** (several parallel copies, each over its own key or date range) instead of exporting the whole table in one go.

💡 In hindsight, it's worth trying **CETAS** (`CREATE EXTERNAL TABLE AS SELECT`) in Synapse to export straight to Parquet. It uses the parallelism of the MPP engine itself. In our environment, CETAS would only write **CSV, not Parquet**, and under migration time pressure we didn't dig into why. CSV wasn't an option for us (types, encoding, delimiters in text), so we went the pipeline route. But if CETAS to Parquet works for you, it can save a lot of time on a large DWH.

Whichever route you take, make **data quality checks part of the migration**, not an afterthought. We compared row counts and key amounts per table between Synapse and Fabric before switching any consumer over.

## 9. The good news: what Fabric has that Synapse didn't

Some workarounds you can finally delete:

- ✅ `DROP TABLE IF EXISTS`
- ✅ **Default values for procedure parameters**, so you can call a procedure without passing every parameter
- ✅ **Cross-database queries** between warehouses and lakehouse SQL endpoints **in the same workspace**

## Takeaway

Most of these changes are mechanical, which means you can **automate** them: a script over your DDL that looks for `NVARCHAR`, `DATETIME`, `TINYINT`, `DISTRIBUTION` and uncast `SELECT INTO` expressions catches most of the problems before deployment. Migration Assistant does a lot of the work, but not all of it.

*More from the Fabric Warehouse in Practice series is coming soon.*

