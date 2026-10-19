# Power BI: The Complete FAQ and Study Reference

**Purpose of this document.** This is a single, self-contained reference that explains Power BI from first principles to production practice. It is written in question-and-answer form. Every answer explains **what** something is, **why** it exists, **how** it works internally, and **what it means for performance and maintainability**. It is designed to be loaded into a study tool such as NotebookLM so the tool can explain any topic clearly.

**Examples in this document** use a deliberately generic retail-style model (tables such as `Sales`, `Product`, `Customer`, `Date`, `Store`) so the ideas apply to any business domain.

**How to read it.**
- Parts build on each other: fundamentals, then data preparation (Power Query), then modeling, then DAX, then visuals, then the Power BI service, then security, performance, governance and interview preparation.
- Terms are defined the first time they appear and again in the glossary.
- Anything that is known to change over time (licensing, limits, preview features, product naming) is marked **(version-sensitive)**. Always confirm those against the current Microsoft documentation before relying on them in production or an exam.

**Contents**
1. Fundamentals and architecture
2. Licensing, capacities and Fabric
3. Getting data: connectivity and storage modes
4. Power Query and the M language
5. Data modeling (star schema, relationships, VertiPaq, data types and byte sizes)
6. DAX fundamentals
7. DAX patterns and time intelligence
8. Reports, visuals and interactivity
9. The Power BI service (workspaces, refresh, gateways, apps, deployment)
10. Security (RLS, OLS, governance)
11. Performance optimization
12. Advanced topics (composite models, aggregations, incremental refresh, XMLA, dataflows, paginated reports, embedding)
13. Developer tooling and DevOps
14. Troubleshooting: common errors and causes
15. Scenario-based interview questions
16. Glossary
17. One-page cheat sheet

---

# Part 1: Fundamentals and Architecture

### Q1. What is Power BI and what problem does it solve?

Power BI is Microsoft's business intelligence platform for connecting to data, shaping it, modeling it, analyzing it with a calculation language, and presenting it as interactive reports and dashboards that can be shared securely.

**The problem it solves.** Business data is scattered across databases, spreadsheets, files, cloud applications and APIs, and decision makers need trustworthy, current, interactive answers without waiting for a developer for each question. Power BI provides one governed pipeline from raw data to insight: **connect, transform, model, visualize, share, and secure**, with self-service capability for analysts and enterprise controls for IT.

### Q2. What are the main components of Power BI?

| Component | What it is | Typical user |
|---|---|---|
| **Power BI Desktop** | Free Windows application for authoring: Power Query, data modeling, DAX and report design | Analysts, developers |
| **Power BI service** (app.powerbi.com) | Cloud service for publishing, sharing, refreshing, collaboration, dashboards and administration | Everyone |
| **Power BI Mobile** | Apps for iOS, Android and Windows for viewing content | Consumers |
| **On-premises data gateway** | Bridge allowing the service to reach on-premises data | IT |
| **Power BI Report Server** | On-premises report server for organizations that cannot use the cloud | IT, regulated environments |
| **Report Builder** | Tool for authoring paginated (pixel-perfect) reports | Report developers |
| **Power BI Embedded** | APIs and capacity for embedding analytics into custom applications | Developers, ISVs |
| **Third-party/external tools** | Tabular Editor, DAX Studio, ALM Toolkit and similar | Advanced developers |

### Q3. What are the building blocks of Power BI content?

- **Semantic model** (formerly called a **dataset**): the data plus the model (tables, relationships, measures, security) that reports query. Microsoft renamed datasets to semantic models; older materials use the old name.
- **Report:** one or more pages of visuals built on a semantic model.
- **Dashboard:** a single-page canvas in the service of **tiles** pinned from one or more reports; it is not created in Desktop.
- **Dataflow:** reusable Power Query transformations that run in the service and store results for reuse.
- **Paginated report:** a fixed-layout report in RDL format for printing and exports.
- **App:** a packaged, published collection of dashboards and reports for broad distribution.
- **Workspace:** a container in the service for collaboration on content.

### Q4. What is the typical development workflow?

1. **Get data** from sources and apply **Power Query** transformations (cleaning, shaping).
2. **Model** the data: relationships, star schema, date table, hierarchies, formats.
3. Write **DAX measures** for business calculations.
4. **Design the report**: pages, visuals, slicers, navigation, themes.
5. **Publish** to a workspace in the service.
6. Configure **refresh** (and a gateway if needed), **security** (RLS), **sharing** (apps, permissions).
7. **Monitor** usage and refresh, and iterate with version control and deployment pipelines.

### Q5. What is the engine behind Power BI?

Power BI semantic models run on the **Analysis Services Tabular engine** (the same engine as SQL Server Analysis Services Tabular and Azure Analysis Services), whose in-memory columnar storage engine is called **VertiPaq**. Two engines cooperate when you query:
- The **storage engine (SE)** scans compressed columns very quickly and in parallel, handling simple aggregations and filters.
- The **formula engine (FE)** evaluates the DAX logic itself, is **single-threaded**, and handles complex expressions.

Efficient models and measures push as much work as possible to the storage engine. This distinction explains most DAX performance behavior (Part 11).

### Q6. What is the difference between a report and a dashboard?

| | Report | Dashboard |
|---|---|---|
| Pages | Multiple | Single canvas |
| Source | One semantic model | Tiles from many reports and models |
| Interactivity | Rich (slicers, drillthrough, cross-filter) | Limited (tiles link back to reports) |
| Authoring | Desktop or service | Service only |
| Alerts | No | Data alerts on supported tiles |
| Purpose | Exploration and analysis | At-a-glance monitoring |

### Q7. What are the typical data-flow architectures?

- **Self-service/Import:** Desktop imports data into a semantic model; the report is published and refreshed on a schedule.
- **Enterprise:** data warehouse or lakehouse (star schema) feeds a certified, shared semantic model; many thin reports connect to it (**live connection** to the shared model).
- **DirectQuery/Direct Lake:** reports query the source live with no import copy (subject to source performance).
- **Dataflow-first:** reusable Power Query logic in dataflows feeds multiple semantic models.

The best practice for scale is **separate the semantic model from the reports**, so one governed model serves many reports.

---

# Part 2: Licensing, Capacities and Fabric

### Q8. What licenses and capacity types exist?

Licensing changes frequently **(version-sensitive; check current pricing and rules)**. The core concepts:

| Option | What it provides |
|---|---|
| **Free license** | Author in Desktop; use "My workspace" for personal content; cannot share with others or consume shared content in shared workspaces unless the content sits in qualifying capacity |
| **Pro (per user)** | Publish, share and collaborate in shared workspaces; consume content shared by others; subject to Pro size and refresh limits |
| **Premium Per User (PPU)** | Premium features for individual licensed users (larger models, more refreshes, paginated reports, deployment pipelines, XMLA, AI features) |
| **Capacity (Fabric F SKUs; legacy Premium P SKUs; Embedded A SKUs)** | Dedicated compute and storage shared by many users; at larger sizes, viewers may not need individual Pro licenses |

Important notes: **Fabric capacity (F SKUs)** is the current capacity model; legacy **Premium (P) SKUs** are being phased out for new purchases **(version-sensitive)**. On capacities of a certain size and above (historically P1 or F64 equivalents), **free-licensed users can view content** in capacity workspaces, while creators still need Pro or PPU for authoring and sharing. **Embedded (A SKUs)** serve application embedding scenarios.

### Q9. What do Premium or Fabric capacities add beyond Pro?

Larger semantic model sizes (large model storage format), more frequent scheduled refreshes, longer refresh timeouts, **incremental refresh with real-time partitions (hybrid tables)**, **paginated reports**, **deployment pipelines**, **XMLA read/write endpoint**, **dataflow computed and linked entities**, **automatic aggregations**, enhanced refresh API, AI capabilities, higher concurrency, dedicated performance isolation, and more. Exact entitlements depend on the SKU and change over time **(version-sensitive)**.

### Q10. What is Microsoft Fabric and how does it relate to Power BI?

Microsoft Fabric is a unified analytics platform that includes Data Factory, data engineering (Spark notebooks, lakehouses), data warehousing, real-time analytics, data science, and **Power BI** as its BI layer, all on a common storage layer called **OneLake** (Delta/Parquet-based). Power BI is part of Fabric: Power BI items live in Fabric workspaces, capacities are Fabric capacities, and new capabilities such as **Direct Lake** connect Power BI directly to Delta tables in OneLake. Power BI remains usable standalone with Pro and PPU licenses.

### Q11. What is Direct Lake?

Direct Lake is a Fabric storage mode in which the semantic model **reads Delta/Parquet files in OneLake directly into memory** on demand, without a scheduled import copy and without translating each query into SQL as DirectQuery does. The idea: **import-like speed with DirectQuery-like freshness**. Under some conditions (for example unsupported features, or exceeding capacity guardrails) queries can **fall back to DirectQuery** mode, which is slower. Behavior and the exact variants (Direct Lake on SQL endpoint versus on OneLake) evolve **(version-sensitive)**.

### Q12. What are the main limits to remember for Pro versus capacity?

Common documented values **(version-sensitive; verify)**: semantic model size limit of about **1 GB per model for Pro** (compressed, in the service), much larger on capacities with large model format; **8 scheduled refreshes per day on Pro** versus up to **48 on capacity**; refresh timeout of **2 hours on Pro** versus up to **5 hours on capacity**; per-user storage limits for Pro. Design to the limits by using incremental refresh, aggregations and star schemas.

---

# Part 3: Getting Data: Connectivity and Storage Modes

### Q13. What data sources can Power BI connect to?

Hundreds of connectors: files (Excel, CSV, JSON, XML, Parquet, PDF, folders, SharePoint), relational databases (SQL Server, Azure SQL, Oracle, PostgreSQL, MySQL, Snowflake, Databricks, Synapse, BigQuery, Redshift), OLAP (Analysis Services), online services (Dynamics 365, Salesforce, SharePoint lists, Google Analytics), web and OData and REST (through the Web connector), Power BI semantic models and dataflows, Microsoft Fabric items (lakehouse, warehouse), and generic ODBC/OLE DB.

### Q14. What are the storage modes?

| Mode | How it works | Strengths | Weaknesses |
|---|---|---|---|
| **Import** | Data is loaded and **compressed into VertiPaq in memory**; queries run against the in-memory copy | Fastest queries; full DAX and Power Query features; works offline from the source | Data is a snapshot (needs refresh); size limits; refresh time |
| **DirectQuery** | No data stored; each visual sends **queries to the source** at report time | Always current; no model size limit; data stays at source | Slower and dependent on source performance; some DAX/Power Query limits; more load on source |
| **Dual** | A table acts as **Import or DirectQuery depending on the query** | Efficient dimension tables in composite models | Requires composite model design |
| **Live connection** | Report connects to an external model (SSAS/AAS or a Power BI semantic model); no local model | Reuse a governed model; centralized logic | Cannot modify the model (except report-level measures) |
| **Direct Lake** | Reads Delta files in OneLake on demand | Import-like speed and freshness | Fabric only; fallback behavior; feature limits |

**Rule of thumb:** use **Import** unless you have a real reason not to (data volume beyond limits, real-time requirements, data sovereignty). Many "must be real time" requirements are actually satisfied by frequent refresh or by incremental refresh.

### Q15. What is a composite model?

A model that mixes storage modes in one semantic model (for example Import for small dimension tables, DirectQuery for a huge fact table, or multiple DirectQuery sources). It enables **aggregation tables** and combining sources. The tradeoffs are complexity and the **limited-relationship** behavior between different storage-mode "islands" (see Part 5 on limited relationships).

### Q16. What is the difference between Import and DirectQuery in terms of how a visual is answered?

- **Import:** the visual generates a DAX query; the storage engine scans compressed in-memory columns; results return in milliseconds to seconds.
- **DirectQuery:** the visual generates a DAX query, the engine **translates it to the source's native SQL**, the source executes it, and the engine post-processes results. Every slicer click can fire multiple SQL queries at the source, so source indexing, views, and capacity matter, as does report design (fewer visuals per page).

### Q17. What are the DirectQuery best practices?

Use a **star schema** or well-indexed views at the source; keep the source query logic simple (avoid complex custom SQL in Power Query); disable or limit cross-highlighting between visuals; use **Query reduction** settings (apply-button for slicers and filters); limit the number of visuals per page; consider **aggregations** and **composite models**; avoid high-cardinality distinct counts; tune **maximum connections per data source**; ensure source statistics and indexes exist; use materialized views or summary tables for heavy calculations.

### Q18. What is a live connection to a Power BI semantic model, and why use it?

A report can connect live to a published semantic model, inheriting its tables, measures and security. It supports a **hub-and-spoke** design: one governed model, many reports authored by different teams, consistent numbers everywhere. Users need **Build permission** on the model. A related feature, **DirectQuery for Power BI semantic models and Analysis Services**, lets you extend a shared model with extra tables or measures in a composite model.

### Q19. What are the data source credentials and privacy levels?

Each data source has credentials (Windows, Basic, OAuth, Organizational account, anonymous, key) stored per data source in Desktop and in the service. **Privacy levels** (Public, Organizational, Private) control whether data from one source may be sent to another during a query (for example during merges or folded queries) to prevent leaking sensitive data into a less trusted source. Conflicts cause the **Formula.Firewall** error (Part 14).

---

# Part 4: Power Query and the M Language

### Q20. What is Power Query?

Power Query is the data connection and transformation engine in Power BI (and Excel and dataflows). You build **applied steps** in a visual editor; each step becomes a line of **M code**. Power Query is where you **clean and shape** data before it reaches the model: remove columns, change types, filter rows, merge, append, pivot, unpivot, split, group, and more.

**Principle:** do transformations **as far upstream as possible** (ideally in the source or warehouse). Do them in Power Query when upstream is not possible; do not do them in DAX calculated columns if Power Query or the source can do it more efficiently.

### Q21. What is the M language?

M is the functional language behind Power Query. A query is a `let ... in` expression: a sequence of named steps where each step refers to the previous one, and the `in` clause returns the final result.

```m
let
    Source   = Sql.Database("myserver", "SalesDb"),
    FactSales = Source{[Schema = "dbo", Item = "FactSales"]}[Data],
    Filtered = Table.SelectRows(FactSales, each [OrderDate] >= #date(2024, 1, 1)),
    Typed    = Table.TransformColumnTypes(Filtered, {{"SalesAmount", Currency.Type}, {"OrderDate", type date}}),
    Renamed  = Table.RenameColumns(Typed, {{"SalesAmount", "Sales Amount"}})
in
    Renamed
```

Key points: M is **case-sensitive**; `each` is shorthand for a function with parameter `_`; steps are **lazily evaluated**; it is a functional language (no loops; use `List.Generate`, `List.Accumulate`, recursion or `Table.TransformColumns`).

### Q22. What is query folding and why is it important?

**Query folding** is Power Query's ability to translate your transformation steps into a **single native query** (for example SQL) executed **at the source**, so the source does the heavy work and only the result comes back. It massively improves refresh performance and is required for effective **incremental refresh** and efficient **DirectQuery**.

**How to check:** right-click a step and choose **View Native Query** (if enabled, the step folds); or use Query Diagnostics. A step that breaks folding forces all later steps to run in the Power Query mashup engine on the gateway/service machine, which is slower and uses memory.

**Common folding breakers:** adding an Index column, `Table.Buffer`, merging queries from different sources, many custom M functions, certain pivot/unpivot variations, some data type conversions, using native SQL queries (steps after a native query typically do not fold unless supported), and referencing parameters in ways the connector cannot translate. **Best practice:** do the foldable steps (filter, select columns, joins within the same source, group by) **first**, and place non-foldable steps at the end.

### Q23. What is the difference between Reference and Duplicate for a query?

- **Duplicate** copies the query definition; the new query re-runs all steps from the source independently (may query the source twice).
- **Reference** creates a new query that **starts from the output of the original**, so it depends on it. Referencing avoids repeating shared steps in your definition, though without folding awareness the source can still be queried multiple times at refresh depending on the engine's evaluation.

### Q24. What is the difference between Merge and Append?

- **Append** stacks rows from tables with similar columns (like SQL `UNION ALL`).
- **Merge** joins tables on matching columns (like SQL joins: left outer, right outer, full outer, inner, left anti, right anti).

Choose Merge sparingly for large tables; prefer relationships in the model to join fact and dimension tables. Use Merge to fetch attributes into a dimension or to do anti-join checks.

### Q25. What are parameters, functions and how do you reuse logic?

- **Parameters** are named values (text, number, date, list) that queries and data source settings can reference; useful for server names, database names, file paths and incremental refresh (`RangeStart`, `RangeEnd`).
- **Custom functions:** convert a query into a function (parameterized) to apply the same logic to many inputs, such as combining many files from a folder.
- **Dataflows** centralize reusable Power Query logic in the service.

### Q26. How do you handle errors and nulls in Power Query?

Use **Replace Errors**, `try ... otherwise` in M (`try Number.From([Amount]) otherwise null`), conditional columns, and **Keep Errors** to inspect error rows. Profile data using **Column quality/distribution/profile** views (and set profiling to the entire dataset, not just the first 1,000 rows). Distinguish `null` (no value) from empty string `""` and from zero.

### Q27. What are the most important Power Query transformations?

Remove/choose columns, filter rows, change data type (always set types explicitly and early where possible), split column, merge columns, replace values, fill down/up, **unpivot columns** (turn wide columns such as month columns into rows, essential for analytic models), pivot, group by, add custom/conditional/index columns, transpose, promote headers, trim/clean text, extract text, date/time functions, and append/merge.

### Q28. What is the "Enable load" and "Include in report refresh" setting?

- **Enable load** off: the query is used only as an intermediate step and is **not loaded into the model** (staging queries). Turn it off for helper queries to reduce model size.
- **Include in report refresh** off: the query does not refresh on refresh (rare use).

Name helper queries clearly (for example prefix with `stg_`) and disable load for them.

### Q29. How do you combine many files from a folder?

Use the **Folder** (or SharePoint folder) connector, filter to the file type, and use **Combine Files**, which builds a sample-file query and a function that is applied to every file. Keep file structures consistent, filter file names early, and consider moving large file collections into a lake or database; thousands of files refresh slowly.

### Q30. How do you create a date table in Power Query or DAX?

Date table in **DAX**:

```dax
Date =
VAR MinDate = DATE(2020, 1, 1)
VAR MaxDate = DATE(2030, 12, 31)
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month", FORMAT([Date], "MMM"),
    "Year-Month", FORMAT([Date], "yyyy-MM"),
    "Quarter", "Q" & QUARTER([Date]),
    "Weekday Number", WEEKDAY([Date], 2),
    "Weekday", FORMAT([Date], "ddd")
)
```

Then **mark it as a date table**, sort text columns by their numeric columns (Month by Month Number), and relate it to fact tables. In Power Query, `List.Dates` plus conversion to a table does the same. For enterprise models, build the date table once in the warehouse.

### Q31. What is incremental refresh at the Power Query level?

You define two **datetime parameters named exactly `RangeStart` and `RangeEnd`**, filter a datetime column in the fact query with `>= RangeStart` and `< RangeEnd`, then define an **incremental refresh policy** on the table (store the last N years, refresh the last N days). The service creates **partitions** and refreshes only recent ones. It depends on **query folding** so that filters are pushed to the source. Details in Part 12.

---

# Part 5: Data Modeling

### Q32. What is a star schema and why does Power BI want it?

A **star schema** has one or more central **fact tables** (events or transactions with numeric measures and foreign keys, such as `Sales`) surrounded by **dimension tables** (descriptive attributes, such as `Product`, `Customer`, `Date`, `Store`), each related one-to-many to the fact. It is the recommended design because:
- Relationships are simple and filters flow predictably from dimensions to facts.
- VertiPaq compresses narrow, repetitive dimension data and numeric facts extremely well.
- DAX is simpler and faster.
- It avoids ambiguity and the pitfalls of wide, flat or heavily snowflaked tables.

A **snowflake** schema normalizes dimensions into sub-dimensions; in Power BI it is usually better to **flatten** it (merge in Power Query or the source) into a single dimension. A single huge **flat table** works for tiny models but wastes memory and slows queries as it grows.

### Q33. What are fact and dimension tables in practice?

- **Fact table:** many rows, mostly numeric and key columns. Example columns: `OrderDate`, `ProductKey`, `CustomerKey`, `Quantity`, `SalesAmount`, `Cost`. Grain (level of detail) must be clearly defined (for example one row per order line).
- **Dimension table:** fewer rows, descriptive attributes, unique key. Example: `Product` with `ProductKey`, `Name`, `Category`, `Brand`.
- **Rule:** columns you aggregate (sum, average) belong in facts; columns you slice by (group, filter) belong in dimensions.

### Q34. What are relationships, cardinality and cross-filter direction?

A **relationship** links a column in one table to a column in another so filters propagate.

- **Cardinality:** one-to-many (the standard, from the dimension "one" side to the fact "many" side), one-to-one, many-to-one, and many-to-many.
- **Cross-filter direction:** **Single** (filters flow from the "one" side to the "many" side) is the default and recommended. **Both** (bidirectional) lets filters flow both ways but can create ambiguity, slower queries and unexpected results.
- **Active vs inactive:** only one active relationship can exist between two tables; others are inactive and used on demand through `USERELATIONSHIP` in DAX.
- The "one" side column must contain **unique values** (no duplicates and, for strong relationships, no blanks issue). If the dimension key has duplicates, the relationship cannot be one-to-many.

### Q35. What are the dangers of bidirectional filtering?

It can introduce **ambiguous filter paths** (multiple routes between tables), create **circular dependencies**, slow queries (more work for the engine), and produce confusing totals. Prefer single direction and solve needs with DAX (`CROSSFILTER`, `TREATAS`, `CALCULATE`) or a proper **bridge table** design. Use bidirectional filtering deliberately and rarely, for example in a **many-to-many with a bridge table** or for RLS propagation in specific designs.

### Q36. How do you handle many-to-many relationships?

Preferred approach: **bridge (factless or mapping) table** with two one-to-many relationships (dimension A to bridge, dimension B to bridge) and filter direction configured on the bridge where needed. Power BI also supports **native many-to-many cardinality** between two columns, which is convenient but easy to misuse, because it does not guarantee unique keys and can produce unexpected duplicates or blank-handling behavior. Choose the bridge table for clarity in complex cases.

### Q37. What are role-playing dimensions?

One dimension used in multiple roles by a fact table, the classic example being a `Date` table used for `OrderDate`, `ShipDate` and `DueDate`. Options: **multiple relationships** from `Sales` to `Date` (only one active; others inactive, used with `USERELATIONSHIP`), or **duplicate the Date table** (a separate `Ship Date` table) so each role is a clean active relationship and filterable in visuals simultaneously.

```dax
Sales by Ship Date =
CALCULATE ( [Total Sales], USERELATIONSHIP ( Sales[ShipDate], 'Date'[Date] ) )
```

### Q38. What is a limited (weak) relationship?

A relationship that does **not guarantee uniqueness** on the "one" side, such as many-to-many relationships and relationships **across different storage-mode islands** (for example Import to DirectQuery, or different DirectQuery sources). Limited relationships do not do the same referential-integrity elimination, produce different blank behavior, and do not support `RELATED`. Be aware of them when designing composite models.

### Q39. What is the difference between a calculated column, a measure and a calculated table?

| | Calculated column | Measure | Calculated table |
|---|---|---|---|
| Evaluated | At **refresh**, row by row | At **query time**, in filter context | At refresh |
| Stored in model | **Yes** (uses memory) | **No** (just a formula) | Yes |
| Context | Row context | Filter context | Depends on expression |
| Responds to slicers | No (fixed values) | **Yes** | No |
| Best for | Attributes used for slicing/sorting/grouping, relationship keys | Aggregations, ratios, KPIs, time intelligence | Date tables, bridge tables, helper tables |

**Guidance:** prefer **measures** for anything you aggregate; use calculated columns only when you need a value for slicing, filtering, sorting or relationships, and push to Power Query or the source whenever possible because those compress better and do not slow refresh as DAX calculated columns do.

### Q40. What is VertiPaq and how does its compression work?

VertiPaq is the in-memory **columnar** storage engine. Each column is stored and compressed independently using a mix of encodings:
- **Value encoding:** for numeric columns, values are stored as offsets/bit-packed integers relative to a base, shrinking size when the range is small.
- **Dictionary (hash) encoding:** distinct values are stored once in a dictionary, and the column stores small integer **indexes** into it. Great for columns with repeating values (text and numbers with low cardinality).
- **Run-length encoding (RLE):** consecutive repeated values are stored as (value, count). Very effective when data is **sorted** or repeating.

Data is split into **segments** (about 1 million rows each by default in Power BI Desktop and the service, larger in some capacities), and each segment is compressed separately. Metadata (dictionaries, relationships, hierarchies) takes additional space.

**What this means in practice:**
- **Cardinality (number of distinct values) is the biggest driver of size.** A column of 10 million rows with 5 distinct values compresses to almost nothing; the same column with 10 million distinct values barely compresses.
- **High-cardinality columns** (GUIDs, transaction IDs, precise timestamps, free-text comments, long numeric decimals) are the usual cause of bloated models.
- Narrow, low-cardinality dimension attributes are cheap.
- Removing unused columns and splitting high-cardinality columns saves the most memory.
- Sorted data compresses better with RLE (the engine chooses its own ordering; you cannot directly control it in Desktop).

### Q41. What are the Power BI data types and how large are they?

| Power BI type | Underlying type | Raw size | Notes |
|---|---|---|---|
| **Whole number** | 64-bit integer | 8 bytes (uncompressed) | Range about ±9.2 quintillion; compresses very well by value encoding when range is small |
| **Decimal number** | 64-bit floating point (double) | 8 bytes | About 15 significant digits; floating-point rounding can occur; poor compression for many distinct values |
| **Fixed decimal number** | 64-bit integer scaled (19 digits, 4 decimal places) | 8 bytes | Range about ±922 trillion with 4 decimals; **exact** with no floating-point drift; usually compresses better than Decimal; ideal for currency |
| **Date/Time** | Floating-point number (days with fractional time) | 8 bytes | Integer part = date, fraction = time; high cardinality if time is included |
| **Date** | Date portion of Date/Time | Stored as the integer part | Low cardinality: at most one distinct value per day |
| **Time** | Time portion | Stored as the fraction | |
| **Text** | Unicode string (UTF-16) | About 2 bytes per character plus dictionary overhead | Dictionary encoded; cost grows with distinct values and length; maximum string length is very large (hundreds of millions of characters) but long strings are expensive |
| **True/False** | Boolean | Effectively 1 bit after compression | Extremely cheap |
| **Binary** | Byte sequence | Variable | Rarely used in models |

**Comparison to common source sizes** (on disk in SQL Server): `tinyint` 1 byte, `smallint` 2, `int` 4, `bigint` 8, `real` 4, `float` 8, `decimal` 5/9/13/17 bytes by precision, `date` 3, `datetime` 8, `datetime2` 6 to 8, `uniqueidentifier` 16, `money` 8. Note that **after VertiPaq compression, the model's size depends mostly on cardinality and encoding, not on the source type size**: an `int` column with few distinct values can occupy a tiny fraction of its raw 4 bytes per row.

**Performance guidance:**
- Prefer **Fixed decimal** over Decimal for money (exact and usually smaller).
- Keep **integers** for keys, and use integer surrogate keys for relationships rather than text or GUID keys (smaller dictionaries, faster joins).
- **Split Date/Time** into a Date column and a Time column (or round time to hour/minute) to reduce cardinality.
- Avoid text for numbers and dates.
- Reduce decimal precision where business-acceptable (round before load) to lower cardinality.
- Mark GUIDs and long text columns as unnecessary: remove them if unused, or keep them only in DirectQuery/detail tables.

### Q42. How do you shrink a model?

1. **Remove unused columns and tables** (the single biggest win).
2. **Reduce cardinality:** split datetime, round decimals, bucket or group where acceptable, remove GUIDs.
3. **Use the right data types** (Fixed decimal, integer keys).
4. **Filter rows** to the needed history; use incremental refresh.
5. **Avoid calculated columns** where Power Query or source can do the work.
6. **Disable Auto date/time** (it creates a hidden date table per date column).
7. **Aggregate** at a higher grain if detail is not needed; use aggregation tables for detail.
8. Use **star schema**, narrow facts, and avoid wide text columns.
9. Analyze with **VertiPaq Analyzer** (in DAX Studio, Bravo or Tabular Editor) to find the largest columns, and with Best Practice Analyzer.

### Q43. What are hierarchies, sort by column, and display folders?

- **Hierarchy:** an ordered set of columns (Year > Quarter > Month > Day) enabling drill-down.
- **Sort by column:** sorts a text column by another column (Month name by Month number); a data model setting that avoids alphabetical month order.
- **Display folders** group fields in the field list; **hide** technical columns (keys) and show only friendly fields.
- Set **Summarization** to "Don't summarize" for numeric columns that should not be aggregated (like keys), and set **data categories** (City, Country, Image URL) and **formats** (currency, percent).

### Q44. Why mark a table as a Date table and why disable Auto date/time?

DAX **time intelligence** functions rely on a proper date table: a column of **contiguous, unique dates covering full years**, **marked as a date table**. Marking tells the engine which column to use for time intelligence and disables the auto-generated behavior. **Auto date/time** creates a hidden date hierarchy table for every date column, bloating the model and creating inconsistent time logic; **turn it off** in options and use one shared Date table.

### Q45. What is a snowflake vs flattened dimension decision?

If a dimension is split (Product → Subcategory → Category), you can either keep them as separate related tables (snowflake) or merge them into one `Product` table with `Subcategory` and `Category` columns. Flattening is typically preferred in Power BI: simpler relationships, simpler DAX, better compression, fewer joins at query time. Keep a snowflake only when sub-dimensions are shared with other facts or very large.

### Q46. What are role-based modeling best practices?

Name tables and columns for business users; hide keys and technical columns; set descriptions; format measures; group measures in a dedicated **measure table** (or display folders); set correct data types and summarization; use consistent naming (singular dimension names, plural fact names or a clear convention); document assumptions; keep one fact grain per table; and avoid mixing different grains in one fact table.

### Q47. How do you handle slowly changing dimensions (SCD) in the model?

Model **Type 2** history as separate rows with a surrogate key and validity dates in the dimension (`ProductKey`, `ProductID`, `ValidFrom`, `ValidTo`, `IsCurrent`). The fact stores the **surrogate key** valid at transaction time, so each transaction reports against the attributes true at that time. For "current attribute" reporting, add a current-attribute column or filter `IsCurrent = TRUE`. Type 1 simply overwrites. The warehouse typically handles SCD logic; Power BI consumes the result.

### Q48. What are common modeling anti-patterns?

Flat wide single-table models; snowflaked dimensions; bidirectional relationships everywhere; many-to-many without need; text keys or GUID relationships; storing calculations as calculated columns instead of measures; implicit measures (dragging numeric columns directly) instead of explicit measures; auto date/time left on; unnecessary columns; mixed grains in a fact table; circular relationships; role-playing dates handled with one table and hacks; and no documentation or naming standard.

---

# Part 6: DAX Fundamentals

### Q49. What is DAX?

**Data Analysis Expressions (DAX)** is the formula language of Power BI, Analysis Services and Power Pivot. It is used for **measures**, **calculated columns**, **calculated tables**, **row-level security filters** and **queries**. DAX looks like Excel formulas but is a relational, context-driven language: results depend on **filter context** and **row context**.

### Q50. What is filter context?

**Filter context** is the set of filters currently applied to the model when a calculation is evaluated: from slicers, visual axis/legend fields, report/page/visual filters, row and column headers in a matrix, relationships, and `CALCULATE` modifiers. A measure like `SUM(Sales[SalesAmount])` returns a different value in each cell of a matrix because each cell has its own filter context (for example Category = Bikes, Year = 2025).

### Q51. What is row context?

**Row context** exists when a formula is evaluated **for each row of a table**: in **calculated columns** and inside **iterator functions** (`SUMX`, `AVERAGEX`, `FILTER`, `ADDCOLUMNS`). It lets you refer to the current row's column values (`Sales[Quantity] * Sales[UnitPrice]`). **Row context does not automatically filter other tables or apply relationships to aggregates** unless converted by context transition.

### Q52. What is context transition?

When `CALCULATE` (or a measure reference, which has an implicit `CALCULATE`) is evaluated **inside a row context**, the current row's values become **filters** (row context is converted into an equivalent filter context). Example: a calculated column in `Customer`:

```dax
Customer Total Sales = [Total Sales]   -- measure referenced in row context
```

For each customer row, the measure is computed with a filter on that customer, because of context transition. This is powerful and also a common source of slowness and confusion with iterators over large tables (each row triggers a context transition).

### Q53. What does CALCULATE do and why is it the most important function?

`CALCULATE(<expression>, <filter1>, <filter2>, ...)` **evaluates an expression in a modified filter context**. Filters can add, replace or remove existing filters. It is the foundation of time intelligence, percent-of-total, conditional totals and more.

```dax
Total Sales = SUM ( Sales[SalesAmount] )

Bike Sales = CALCULATE ( [Total Sales], Product[Category] = "Bikes" )

All Category Sales = CALCULATE ( [Total Sales], REMOVEFILTERS ( Product[Category] ) )
```

Rules: a boolean filter like `Product[Category] = "Bikes"` is shorthand for `FILTER(ALL(Product[Category]), Product[Category] = "Bikes")` and **replaces** any existing filter on that column; filters from different columns combine with **AND**; use `KEEPFILTERS` to intersect rather than replace.

### Q54. What are ALL, ALLEXCEPT, ALLSELECTED, REMOVEFILTERS and VALUES?

- **ALL(Table or Column):** removes filters (and as a table function returns all rows ignoring filters).
- **REMOVEFILTERS(...):** same effect as `ALL` but only usable as a `CALCULATE` modifier, clearer in intent.
- **ALLEXCEPT(Table, Col1, ...):** removes all filters on the table except on the listed columns.
- **ALLSELECTED(...):** removes filters from the **current visual** but keeps filters from slicers and outer context (used for "percent of visible total").
- **VALUES(Column):** returns the distinct values currently visible in filter context (plus a blank row if there are unmatched relationship values); **DISTINCT** ignores that blank row.

```dax
Sales % of Total = DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALL ( Product ) ) )

Sales % of Visible Total = DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALLSELECTED ( Product ) ) )
```

### Q55. What are iterator functions?

Functions ending in **X** (`SUMX`, `AVERAGEX`, `MINX`, `MAXX`, `COUNTX`, `RANKX`, `CONCATENATEX`) iterate over a table row by row, evaluating an expression for each row (creating row context), and then aggregate. Use them when the calculation must be done per row before aggregating:

```dax
Total Revenue = SUMX ( Sales, Sales[Quantity] * Sales[UnitPrice] )
```

Performance: iterators over large tables with complex expressions are slower than simple aggregations; prefer storing precomputed columns or simple expressions when possible, and avoid nested iterators over big tables and unnecessary context transitions inside them.

### Q56. What is the difference between SUM and SUMX?

`SUM(Column)` aggregates a single column directly (fast, storage-engine friendly). `SUMX(Table, Expression)` evaluates an expression **per row** and then sums, needed for products across columns (price × quantity) or conditional per-row logic. `SUM(col)` is internally a simple `SUMX(table, col)` but optimized.

### Q57. What are variables in DAX and why use them?

`VAR ... RETURN` defines named values evaluated **once** and usable multiple times, improving **readability and performance** (no repeated computation), and making debugging easier. A variable's value is computed in the filter context where it is defined, not where it is used.

```dax
Sales YoY % =
VAR CurrentSales = [Total Sales]
VAR PriorSales   = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
RETURN
    DIVIDE ( CurrentSales - PriorSales, PriorSales )
```

### Q58. What are the common logical and information functions?

`IF`, `SWITCH` (cleaner than nested IFs; use `SWITCH(TRUE(), cond1, result1, ...)` for conditions), `AND`/`OR` (and `&&`, `||`), `IFERROR`, `ISBLANK`, `COALESCE`, `HASONEVALUE`, `SELECTEDVALUE(column, default)` (returns the single selected value or a default), `ISFILTERED`, `ISCROSSFILTERED`, `ISINSCOPE` (detect the level of a hierarchy in a matrix, useful for different logic in totals).

### Q59. How do you handle division and blanks?

Use `DIVIDE(numerator, denominator, [alternate])`, which returns blank (or the alternate result) on division by zero, and is safer and cleaner than `IF` checks. Remember that DAX distinguishes **BLANK** (no value) from zero; many visuals hide rows with blank measures, which is often desirable and is also why measures returning 0 instead of blank can bloat visuals.

### Q60. What is the difference between COUNT, COUNTA, COUNTROWS, DISTINCTCOUNT?

- `COUNT(column)`: counts non-blank numeric values.
- `COUNTA(column)`: counts non-blank values of any type.
- `COUNTROWS(table)`: counts rows in a table (most common and clear).
- `DISTINCTCOUNT(column)`: counts unique values (can be expensive on high-cardinality columns).
- `DISTINCTCOUNTNOBLANK`: ignores blanks.

### Q61. What are RELATED, RELATEDTABLE and LOOKUPVALUE?

- **RELATED(column):** fetches a value from the "one" side table for the current row (follows a relationship from many to one); used in calculated columns and iterators.
- **RELATEDTABLE(table):** returns the rows on the "many" side related to the current row.
- **LOOKUPVALUE:** finds a value by matching search columns without needing a relationship (slower; avoid when a relationship exists).

### Q62. What are TREATAS, USERELATIONSHIP and CROSSFILTER?

- **TREATAS(table, columns):** applies a set of values as a filter to columns in another table **without a physical relationship** (a virtual relationship).
- **USERELATIONSHIP:** activates an inactive relationship for the duration of a `CALCULATE`.
- **CROSSFILTER:** changes the cross-filter direction (or disables the relationship) for a calculation.

### Q63. What are table functions like FILTER, ADDCOLUMNS, SUMMARIZE, SUMMARIZECOLUMNS, TOPN, GENERATE?

- **FILTER(table, condition):** returns rows satisfying the condition, evaluated row by row. **Avoid `FILTER` over an entire large fact table** inside `CALCULATE` when a simple column filter works; filter dimension columns instead.
- **ADDCOLUMNS(table, name, expression, ...):** adds calculated columns to a table expression.
- **SUMMARIZE / SUMMARIZECOLUMNS:** group by columns and compute aggregates (use `SUMMARIZECOLUMNS` for queries and `ADDCOLUMNS(SUMMARIZE(...))` for measures; it behaves differently with filter context).
- **TOPN(n, table, orderBy, direction):** top N rows by an expression.
- **GENERATE / GENERATEALL:** cross apply (like `CROSS APPLY`).
- **UNION, INTERSECT, EXCEPT, DISTINCT, VALUES, CROSSJOIN, SELECTCOLUMNS, DATATABLE, ROW.**

### Q64. What is the difference between a measure's evaluation in the total row versus detail rows?

Totals are evaluated in the **total row's filter context** (no row-level filters), not by summing the visible rows. So a ratio measure shows the ratio of totals, not the sum of ratios. For non-additive measures (distinct counts, averages, ratios), totals can look "wrong" if you expected a sum of rows. When you need the total to equal the sum of rows, use an iterator over the displayed level:

```dax
Total of Displayed =
SUMX ( VALUES ( Product[ProductName] ), [Margin %] * [Total Sales] )
```

and use `ISINSCOPE` to vary behavior by level.

### Q65. What are common DAX best practices?

Use **explicit measures**, not implicit aggregation; keep measure logic in a dedicated measures table; use **variables**; fully qualify columns (`Table[Column]`) and never qualify measures with a table (`[Measure]`); use `DIVIDE`; format code consistently; avoid calculated columns for aggregations; avoid bidirectional relationships; prefer simple column filters in `CALCULATE`; avoid iterating large tables unnecessarily; comment with `//` or `/* */`; test with DAX Query View or DAX Studio.

---

# Part 7: DAX Patterns and Time Intelligence

### Q66. What are the requirements for DAX time intelligence?

A **Date table** with unique, contiguous dates covering **complete years** (Jan 1 to Dec 31, or fiscal-year boundaries), **marked as a date table**, and a relationship from a fact date column to the Date table's date column. Functions like `TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD`, `DATESYTD` depend on this.

### Q67. How do you compute YTD, QTD, MTD and fiscal YTD?

```dax
Sales YTD   = TOTALYTD ( [Total Sales], 'Date'[Date] )
Sales QTD   = TOTALQTD ( [Total Sales], 'Date'[Date] )
Sales MTD   = TOTALMTD ( [Total Sales], 'Date'[Date] )
Sales FYTD  = TOTALYTD ( [Total Sales], 'Date'[Date], "6/30" )   -- fiscal year ending June 30
```

Equivalent using `CALCULATE ( [Total Sales], DATESYTD ( 'Date'[Date] ) )`.

### Q68. How do you compute year-over-year and growth?

```dax
Sales PY     = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
Sales YoY    = [Total Sales] - [Sales PY]
Sales YoY %  = DIVIDE ( [Sales YoY], [Sales PY] )
```

`DATEADD('Date'[Date], -1, YEAR)` is a more general alternative (also for months, quarters, days). `PARALLELPERIOD` shifts whole periods. Be careful that `SAMEPERIODLASTYEAR` returns the same *selected* period shifted; at month level it returns the same month last year.

### Q69. How do you compute a running (cumulative) total?

```dax
Running Total =
VAR MaxDate = MAX ( 'Date'[Date] )
RETURN
    CALCULATE ( [Total Sales], FILTER ( ALL ( 'Date'[Date] ), 'Date'[Date] <= MaxDate ) )
```

### Q70. How do you compute a moving average?

```dax
Sales 3M Avg =
DIVIDE (
    CALCULATE ( [Total Sales], DATESINPERIOD ( 'Date'[Date], MAX ( 'Date'[Date] ), -3, MONTH ) ),
    3
)
```

For a true average of monthly totals when partial periods exist, count the distinct months in the window instead of using 3.

### Q71. How do you rank and do Top N?

```dax
Product Rank = RANKX ( ALL ( Product[ProductName] ), [Total Sales], , DESC, DENSE )

Top 5 Products Sales =
CALCULATE (
    [Total Sales],
    KEEPFILTERS ( TOPN ( 5, ALL ( Product[ProductName] ), [Total Sales] ) )
)
```

For user-driven Top N, use the visual-level **Top N filter** or a parameter table (what-if) feeding the `n` value.

### Q72. How do you calculate "percent of total" at different levels?

```dax
% of Grand Total     = DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALL ( Sales ) ) )
% of Category Total  = DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALLEXCEPT ( Product, Product[Category] ) ) )
% of Visible Total   = DIVIDE ( [Total Sales], CALCULATE ( [Total Sales], ALLSELECTED () ) )
```

### Q73. How do you create dynamic measure selection (a measure switcher)?

Create a small **disconnected table** listing the measure names and a `SWITCH` measure using `SELECTEDVALUE`:

```dax
Selected Measure =
SWITCH (
    SELECTEDVALUE ( 'Metric'[Metric Name], "Sales" ),
    "Sales",  [Total Sales],
    "Profit", [Total Profit],
    "Units",  [Total Units]
)
```

Newer alternatives: **field parameters** (a built-in feature that lets users switch fields or measures in a visual) and **calculation groups** (reusable calculation items such as YTD, PY, YoY applied to any measure, created with Tabular Editor or newer Desktop capabilities **(version-sensitive)**).

### Q74. What are calculation groups?

A feature that lets you define **calculation items** (for example "Current", "YTD", "Prior Year", "YoY %") that **modify any measure** placed in a visual, removing the need to write separate time-intelligence variants of every measure. They use `SELECTEDMEASURE()`. They reduce measure count and keep logic consistent, but require care with format strings and precedence when multiple calculation groups are used.

### Q75. What are field parameters?

A feature (Modeling, New parameter, Fields) that creates a table allowing report users to **switch which fields (columns or measures) a visual displays** using a slicer. It generates a DAX table of fields with the `NAMEOF` pattern. It gives users flexible views without bookmarks and duplicate visuals.

### Q76. How do you build a what-if parameter?

Modeling, **New parameter, Numeric range** creates a generated table and a measure returning the selected value; use that measure in calculations (for example `Adjusted Sales = [Total Sales] * (1 + 'Growth %'[Growth % Value])`). Useful for scenario analysis.

### Q77. How do you handle semi-additive measures such as inventory balances?

Balances cannot be summed across time. Use functions that take the **last value** in the period:

```dax
Closing Stock =
CALCULATE (
    SUM ( Inventory[Quantity] ),
    LASTDATE ( 'Date'[Date] )
)
```

or `CLOSINGBALANCEMONTH`, `LASTNONBLANKVALUE`, or `CALCULATE ( ..., LASTNONBLANK ( 'Date'[Date], CALCULATE ( COUNTROWS ( Inventory ) ) ) )` when some dates have no snapshot.

### Q78. How do you calculate distinct counts and "new versus returning" customers?

```dax
Customers = DISTINCTCOUNT ( Sales[CustomerKey] )

New Customers =
VAR FirstOrderDates =
    ADDCOLUMNS ( VALUES ( Sales[CustomerKey] ), "FirstOrder", CALCULATE ( MIN ( Sales[OrderDate] ), ALL ( 'Date' ) ) )
RETURN
    COUNTROWS ( FILTER ( FirstOrderDates, [FirstOrder] >= MIN ( 'Date'[Date] ) && [FirstOrder] <= MAX ( 'Date'[Date] ) ) )
```

Returning customers = customers in the period minus new customers. For large models, precompute the first-order date as a customer attribute in the model.

### Q79. How do you do an ABC / Pareto analysis?

Rank products by sales, compute cumulative percentage of total in rank order, and classify (A = first 80%, B = next 15%, C = rest). It combines `RANKX`, a running total over the ranked table (using `TOPN` or filtering by rank), and `SWITCH(TRUE())` classification. Often built with a helper table or visual calculation for clarity.

### Q80. What are visual calculations?

A newer feature **(version-sensitive)** that lets you write DAX calculations that operate **directly on the data in a visual** (using functions like `RUNNINGSUM`, `MOVINGAVERAGE`, `PREVIOUS`, `FIRST`, `LAST`), referencing visual fields rather than the model. They simplify common calculations (running totals, percent of parent) without model changes, but remain specific to that visual.

### Q81. How do you debug DAX?

Use **DAX Query View** in Desktop (or DAX Studio) to run `EVALUATE` queries; break measures into variables and return intermediate results; use `ROW("Value", [Measure])`; check filter context with `ISFILTERED`/`VALUES` debugging measures; use **Performance Analyzer** to copy a visual's DAX query and test it in isolation; and use **Server Timings** in DAX Studio to see storage-engine vs formula-engine time.

---

# Part 8: Reports, Visuals and Interactivity

### Q82. What are the key report design principles?

Start from the **audience and decisions**, not the data: one clear purpose per page; most important KPIs top-left; consistent layout, fonts and color; limit colors and use them for meaning; choose the right visual for the question (comparison, trend, composition, distribution, relationship); avoid clutter and 3D/decorative charts; use whitespace; add clear titles and tooltips; ensure **accessibility** (contrast, alt text, tab order, not color-only encoding); and design for performance (fewer visuals per page).

### Q83. How do I choose the right chart?

| Question | Good choices |
|---|---|
| Compare categories | Bar/column chart (sorted) |
| Trend over time | Line or area chart |
| Part of a whole (few parts) | Stacked bar, donut/pie with few slices, treemap |
| Distribution | Histogram-style column, box plot (custom), scatter |
| Relationship between two measures | Scatter or bubble |
| Progress to target | Gauge (sparingly), bullet chart, KPI visual |
| Ranking | Sorted bar chart |
| Detail lookup | Table or matrix |
| Geographic | Map, filled map, shape map |
| Flow or variance | Waterfall, ribbon chart, decomposition tree |

### Q84. What are slicers and how do they behave?

Slicers filter the report interactively (list, dropdown, between/range, relative date, hierarchy, buttons). **Sync slicers** across pages through the View menu. Slicer selections are filter context. Avoid high-cardinality slicers (thousands of items) in DirectQuery; use search or dropdown, and consider the **Apply** button (query reduction) for heavy models.

### Q85. What are the filter levels (visual, page, report, drillthrough)?

Filters apply at the **visual**, **page**, **report** (all pages) and **drillthrough** levels. Filter types: **basic**, **advanced**, **Top N**, **relative date**, and **relative time**. Report-level filters apply everywhere; **"Filters on all pages"** is for global constraints. Hidden filters are not security: users with edit access can change them (use RLS for security).

### Q86. What are drill down, drill up, drillthrough and cross-filtering?

- **Drill down/up:** navigate hierarchy levels inside the same visual (Year → Quarter → Month).
- **Expand/show next level** in matrices and charts.
- **Drillthrough:** right-click a data point to jump to a **detail page** pre-filtered to that context; configured by adding drillthrough fields to a target page.
- **Cross-filtering/highlighting:** selecting a data point in one visual filters or highlights other visuals on the page; control with **Edit interactions**.

### Q87. What are bookmarks and how are they used?

Bookmarks capture a **report state** (filters, slicers, visibility, selection, sort order, focus mode) and can be triggered by buttons to create navigation, toggles (show/hide visuals), view switchers (chart vs table), and storytelling. Use them thoughtfully: many bookmarks complicate maintenance. Field parameters or page navigation can often replace bookmark-heavy designs.

### Q88. What are tooltips and report page tooltips?

Hover **tooltips** show extra fields. **Report page tooltips** are small report pages used as rich tooltips (mini charts), configured by setting a page's type to Tooltip and assigning it in a visual's tooltip settings. Use them to add context without clutter.

### Q89. What is conditional formatting?

Format colors, data bars, icons, web URLs or font colors **by rules, color scales, field values or DAX measures**. Using a **measure returning a color or text** gives full flexibility (traffic lights, thresholds relative to targets). Keep color meaning consistent and accessible (do not rely on red/green alone).

### Q90. What AI and advanced visuals exist?

**Key influencers**, **decomposition tree**, **Q&A**, **smart narrative**, **anomaly detection** on line charts, **forecasting** on line charts, **clustering** on scatter charts, **small multiples**, **ribbon**, **waterfall**, **funnel**, **scatter with play axis**, and **Copilot** for natural-language report and summary generation **(version-sensitive; availability depends on license/capacity and tenant settings)**. Use them to complement, not replace, carefully designed core visuals.

### Q91. What are custom visuals?

Visuals from **AppSource** or developed with the Power BI visuals SDK (TypeScript). **Certified** custom visuals meet security and quality checks. Tenant admins can control which custom visuals are allowed. Be aware of performance and security implications of uncertified visuals, and export limitations with some visuals.

### Q92. What are themes?

A **JSON theme file** defines colors, fonts and visual defaults across a report for consistent branding (import via View, Themes). Maintain a corporate theme file in source control and apply to every report.

### Q93. What is a paginated report and when should I use it?

A **paginated report** (RDL, built in Report Builder or Power BI Report Builder) produces **pixel-perfect, print-ready, multi-page output** such as invoices, statements, regulatory forms and large tabular exports with precise layout. It handles very large tables across pages and is exportable to PDF, Excel, Word and more. Interactive Power BI reports suit exploration; paginated reports suit operational and printable documents. Paginated reports in the service require qualifying capacity or PPU **(version-sensitive)**.

### Q94. What is a mobile layout?

A separate layout designed for phones (portrait canvas) created in Desktop (View, Mobile layout); you place and resize existing visuals for phone use. Test with the Power BI mobile app.

### Q95. How do you export data and what controls exist?

Users can export summarized or underlying data (CSV/Excel), export reports to PDF/PowerPoint, and use Analyze in Excel (connected to the semantic model). Admins control export through **tenant settings** and sensitivity labels; row-level security applies to exports based on the user's access.

### Q96. What is Analyze in Excel and PivotTables on Power BI semantic models?

Excel can connect live to a published semantic model, letting users build PivotTables with the governed measures and security. It is an effective way to give finance users familiar tools while preserving a single source of truth.

---

# Part 9: The Power BI Service

### Q97. What are workspaces and roles?

A **workspace** is a collaborative container for semantic models, reports, dashboards and dataflows. **Roles:**
- **Admin:** full control, manage access and settings.
- **Member:** edit content, publish, and manage permissions (except admin-level actions); can publish apps (depending on settings).
- **Contributor:** create and edit content but cannot publish apps or manage access.
- **Viewer:** view only (subject to RLS).

**Important:** **RLS applies only to Viewers.** Admins, Members and Contributors with edit rights can see all data in the model, so never rely on workspace roles to restrict data for them.

### Q98. What is an app and how does it differ from sharing?

An **app** packages selected content from a workspace and distributes it to **audiences** with controlled navigation; consumers get a clean, read-only experience without access to the workspace itself. **Direct sharing** gives access to individual items via links. Best practice for broad distribution: **publish an app** from a workspace, keep the workspace for developers.

### Q99. How do I refresh data?

- **Scheduled refresh:** configured in the semantic model settings (frequency and times); limited per day by license (Pro lower, capacity higher **(version-sensitive)**).
- **On-demand refresh:** manual Refresh now.
- **API/automation refresh:** REST API, PowerShell, Power Automate, or the **enhanced refresh API** (capacities) for partition-level control.
- **Automatic page refresh** (DirectQuery reports; fixed interval or change detection) and **dashboard tile refresh**.

A refresh **replaces** the in-memory data with new data (full refresh) unless incremental refresh is configured. Failures notify the owner; check the **refresh history** for errors.

### Q100. What is an on-premises data gateway?

A gateway is software installed on a machine in your network that lets the Power BI service **securely reach on-premises (or private-network) data sources** for refresh and DirectQuery/live connections, using **outbound connections only** (no inbound ports). Types:
- **Standard (enterprise) mode:** shared, supports Import refresh, DirectQuery and live connections; can be clustered for high availability; recommended for production.
- **Personal mode:** single user, Import refresh only; not for production sharing.
- **Virtual network (VNet) data gateway:** a Microsoft-managed gateway for sources inside an Azure virtual network, no machine to maintain.

**Best practices:** install on a dedicated, well-resourced machine (not a laptop or the database server); use **clusters** (multiple members) for availability; keep the gateway **updated**; store the **recovery key** securely; place it close to the data source; monitor performance; map each data source in the service with the right credentials; use service accounts rather than personal credentials; and be aware that **Power Query mashup work runs on the gateway machine** for non-folding queries, consuming its CPU and memory.

### Q101. What network requirements does a gateway have?

Outbound HTTPS (port 443) and additional outbound ports to Azure Service Bus used for communication (commonly 5671-5672 and 9350-9354) **(version-sensitive; confirm in the current documentation)**; no inbound ports; correct DNS and proxy configuration; and time synchronization.

### Q102. Why do I need to configure data source credentials in the service after publishing?

Credentials are not published with the file. For each data source (cloud or via gateway), set authentication and privacy level in the semantic model's settings. Cloud sources use OAuth or stored credentials; on-premises sources require a gateway data source mapping. Credentials expire (especially OAuth tokens) and must be refreshed, a common cause of refresh failures.

### Q103. What are deployment pipelines?

A service feature (capacity or PPU) providing **Development, Test and Production stages** for content promotion: compare stages, deploy selected items forward, and apply **deployment rules** (switch data sources and parameters per stage). They support controlled release processes without manual republishing. They complement (not replace) Git integration and external CI/CD.

### Q104. What is endorsement, and what are certified and promoted content?

**Promoted** content is marked by its owner as ready for use; **Certified** content is approved by designated reviewers per organizational criteria. Endorsement helps users find trusted semantic models and reports in the **OneLake data hub** (formerly the datasets hub) and reduces duplicate, conflicting models.

### Q105. What are dashboards, tiles and data alerts?

A **dashboard** holds tiles pinned from reports (visual, live page, text, image, video, web content). Tiles link back to the source report. **Data alerts** notify users when a card, KPI or gauge tile crosses a threshold. **Subscriptions** email report snapshots on a schedule. **Q&A** lets users type natural-language questions against the underlying model (Q&A availability and feature direction evolve **(version-sensitive)**).

### Q106. What are usage metrics and monitoring tools?

Workspace **usage metrics** reports show report views and viewers; the **Activity log** (admin) records user and admin actions; the **Capacity Metrics app** monitors capacity utilization; the **admin monitoring workspace** and **Audit** logs provide tenant visibility; **refresh history** tracks refresh outcomes; and Azure Log Analytics integration can capture semantic model engine events **(version-sensitive)**.

### Q107. How do you share a semantic model for reuse?

Publish it to a workspace, grant users **Build** permission, and let them create reports with a **live connection** (or composite model on top). Certify the model, document its measures, and apply RLS centrally. Separate **model workspace** (owned by BI team) from **report workspaces** (owned by business teams) to prevent uncontrolled duplicates.

### Q108. What is a Power BI Report Server?

An on-premises server (a superset of SQL Server Reporting Services) for hosting Power BI reports, paginated reports and KPIs behind your firewall. It uses its own Desktop build ("Power BI Desktop for Report Server") and receives updates on a different, slower cadence, so features available in the cloud service may be missing. Licensing is via SQL Server Enterprise with Software Assurance or Premium capacity entitlement **(version-sensitive)**.

---

# Part 10: Security

### Q109. What is Row-Level Security (RLS)?

RLS restricts **which rows of data** a user can see, by defining **roles** with **DAX filter expressions** on tables. Users assigned to a role only see rows where the expression is true; the filter propagates through relationships to related tables (from the "one" side to the "many" side by default).

**Static RLS:** a fixed rule per role, for example `[Region] = "West"` on the `Store` table.

**Dynamic RLS:** one role whose rule depends on the **logged-in user**, using `USERPRINCIPALNAME()` (or `USERNAME()`) and a **security mapping table**:

```dax
-- On the 'UserSecurity' mapping table (UserEmail, Region)
[UserEmail] = USERPRINCIPALNAME ()
```

with a relationship from `UserSecurity[Region]` to `Store[Region]` (and cross-filter set so the filter propagates appropriately). Dynamic RLS scales: add rows to the mapping table instead of creating roles.

### Q110. How do I implement and test RLS?

1. In Desktop, **Modeling, Manage roles**: create roles and DAX filters.
2. Test with **View as** (choose a role and optionally a user).
3. Publish, then in the service **Security** for the semantic model: add users or groups to roles.
4. Test again in the service with **Test as role**.

Key facts: RLS applies to **Viewers** (and to users accessing via Build permission); **workspace Admin/Member/Contributor** bypass it; RLS filters are applied **before** measures calculate; **performance** can suffer with complex RLS expressions, so keep them simple and index/shape the security tables; and RLS with DirectQuery can be done in the model or pushed to the source (for example with SSO to the source).

### Q111. What are RLS pitfalls?

Forgetting that workspace editors bypass RLS; not testing with real users; unions of roles (a user in multiple roles sees the **union** of allowed rows); security on dimension tables only (the fact must be reachable through relationships with correct direction); using bidirectional filtering to propagate security in the wrong direction; using `USERNAME()` where `USERPRINCIPALNAME()` is needed (formats differ between Desktop and service); and exposing sensitive data through columns that are not row-filtered or through `Analyze in Excel` for users with Build permission without proper design.

### Q112. What is Object-Level Security (OLS)?

OLS hides **entire tables or columns** (and their metadata) from specific roles so users cannot see or query them at all. It is configured with external tools such as **Tabular Editor** (or XMLA) and applies to roles; combined with RLS for full control. Measures referencing hidden objects may error for those users, so test.

### Q113. What other security and governance features exist?

- **Sensitivity labels** (Microsoft Purview Information Protection) that classify content and can persist into exports.
- **Tenant settings** (admin portal) controlling export, sharing, publish to web, custom visuals, external sharing, and more.
- **Microsoft Entra ID** (formerly Azure AD) authentication, conditional access and MFA.
- **Private links** and network isolation options for the service **(version-sensitive)**.
- **Bring your own key (BYOK)** for encryption at rest for capacities.
- **Audit logs** and activity monitoring.
- **Row-level and object-level security**, workspace roles, app audiences, item-level permissions.
- **Publish to web** creates **public** links that anyone on the internet can see: usually disabled for organizations.

### Q114. How is guest/external user access handled?

External users (B2B guests in Entra ID) can access content shared with them if tenant settings allow, typically requiring licenses (their own or yours). Combine with RLS using their principal names, and use apps/audiences. Review tenant settings for external sharing and consider governance policies.

### Q115. What is the difference between "Publish to web" and sharing with a link?

**Publish to web** generates a public URL that exposes the report to anyone without authentication and bypasses RLS and licensing; it should be used only for deliberately public data. **Sharing** (link or direct access) requires sign-in and respects permissions and RLS.

---

# Part 11: Performance Optimization

### Q116. How do I systematically find the cause of a slow report?

Work from observation to cause:
1. Use **Performance Analyzer** (Desktop, View) to record page load: for each visual it shows **DAX query**, **visual display** and **other** time. Identify the slowest visuals.
2. Copy the slow visual's DAX query into **DAX Studio** (or DAX Query View) and run with **Server Timings** to see storage-engine vs formula-engine time and the number of SE queries.
3. Check the **model** (size, cardinality, relationships) with **VertiPaq Analyzer**.
4. Check the **report design** (too many visuals, heavy slicers, unfiltered tables with thousands of rows).
5. Check **refresh** performance separately (Power Query folding, source speed).
Then fix the specific bottleneck instead of guessing.

### Q117. What model-level optimizations matter most?

- **Star schema**; narrow fact tables; remove unneeded columns and rows.
- **Reduce cardinality** (split datetime, round decimals, drop GUIDs/long text).
- Proper **data types** (integer keys, fixed decimal).
- **Disable Auto date/time**; use a single marked date table.
- Prefer **measures** to calculated columns; avoid bidirectional/many-to-many relationships.
- Use **aggregation tables** and **incremental refresh** for large data.
- Set **Dual** storage mode for dimensions in composite models.
- Keep relationship columns as **integers** and sorted logically.

### Q118. What DAX-level optimizations matter most?

- Use **variables** to avoid repeated evaluation.
- Filter **columns, not whole tables**: `CALCULATE([Sales], Product[Category] = "Bikes")` is better than filtering the fact table with `FILTER(Sales, ...)`.
- Avoid unnecessary **iterators over large tables** and **context transition inside iterators** over many rows.
- Prefer **`DIVIDE`**, **`SELECTEDVALUE`**, **`COUNTROWS`** over heavy alternatives.
- Avoid `DISTINCTCOUNT` on very high-cardinality columns where possible (or pre-aggregate).
- Reduce the number of **storage-engine queries** and avoid **callbacks** (formula engine callbacks from the SE) such as `IF` with complex branches inside iterators.
- Avoid heavy `FORMAT` and string operations inside large iterators.
- Keep measures **simple and composable**.

### Q119. What report-level optimizations matter?

Fewer visuals per page (each visual runs queries); avoid very large tables/matrices showing thousands of rows (use Top N or filters, or drillthrough to detail); avoid high-cardinality slicers; use **Apply buttons** (Query reduction) for slicers and filters; limit cross-highlighting where unnecessary; avoid complex custom visuals; use **bookmarks and field parameters** wisely; leverage **Optimize ribbon** and pause visuals during design; use **page-level filters** to restrict default data; and use **Performance Analyzer** after every major change.

### Q120. What are DirectQuery-specific performance steps?

Optimize the **source** (indexes, partitions, columnstore, statistics, materialized views); use simple views for Power BI instead of complex native SQL; reduce the number of visuals and slicers; use aggregation tables in a composite model; limit **bidirectional filters** and many-to-many; keep joins on integer keys; tune **Maximum connections per data source** (default modest) and consider capacity limits on parallel queries; monitor query load with source tools (for example SQL Profiler or Query Store).

### Q121. What speeds up refresh?

Ensure **query folding**; filter and select early; remove unused columns; use **incremental refresh**; load **dimensions and facts** from the warehouse with minimal transformations; avoid expensive Power Query operations (nested merges, custom functions per row, Table.Buffer misuse); consider **dataflows** to stage cleaned data once for multiple models; schedule refreshes outside source peak hours and stagger them; and use **enhanced refresh** to parallelize partitions on capacities.

### Q122. What is the Optimize ribbon?

A Desktop ribbon with **Pause visuals** (prevents queries while you rearrange the report), **Optimization presets**, and shortcuts to Performance Analyzer and DAX Query View. Use "pause visuals" during layout changes to avoid constantly re-querying.

### Q123. What is the effect of storage engine versus formula engine on tuning?

If most time is in the **storage engine** (many fast parallel scans), improve data model structure and reduce data scanned (cardinality, aggregations). If most time is in the **formula engine** (single-threaded), simplify DAX: reduce iterators, reduce callbacks and nested complexity, materialize intermediate results with variables, or precompute in the model. A very high number of SE queries often signals inefficient iterator patterns.

---

# Part 12: Advanced Topics

### Q124. What is incremental refresh and how do I configure it?

Incremental refresh refreshes only recent data and keeps history partitions untouched, shrinking refresh time and source load.

**Setup:**
1. Create Power Query parameters **`RangeStart`** and **`RangeEnd`** (type Date/Time; the names are reserved).
2. Filter the fact table's date/datetime column: `>= RangeStart and < RangeEnd` (half-open interval to avoid duplicates).
3. Ensure the filter **folds** to the source.
4. Right-click the table, choose **Incremental refresh**, and set **Store rows** (history window, for example 5 years) and **Refresh rows** (recent window, for example 10 days). Options: **Detect data changes** (refresh a partition only if a "last modified" column's max changed), **Only refresh complete periods**, and (on capacity) **Get the latest data in real time with DirectQuery** (hybrid table).
5. Publish. The service creates partitions (years, quarters, months, days) and refreshes according to the policy.

**Notes:** the policy applies only in the service (Desktop loads only the filtered parameter range); a semantic model with incremental refresh cannot be downloaded as a PBIX after publishing in the usual way; source data must support the date filter; changes to historical rows outside the refresh window will not appear unless you refresh those partitions (use enhanced refresh or XMLA) or detect data changes.

### Q125. What are aggregation tables?

An **aggregation table** is a pre-summarized table (for example sales by date, product category and region) used to answer high-level queries quickly while detailed queries fall through to the **detail table** (typically DirectQuery). **User-defined aggregations:** you create an Import-mode aggregation table, configure **Manage aggregations** (mapping columns to GroupBy, Sum, Count, Min, Max on the detail table), and Power BI automatically routes eligible queries to the aggregation. **Automatic aggregations** (capacities) use machine learning to create and maintain aggregations from query patterns **(version-sensitive)**. Dimension tables are usually set to **Dual** so they work with both.

### Q126. What is a hybrid table?

A table with an incremental refresh policy where the **most recent partition is DirectQuery** (real-time) and older partitions are Import. It delivers fresh data for current periods with fast, compressed history. Requires capacity **(version-sensitive)**.

### Q127. What is the XMLA endpoint?

An open-platform connectivity endpoint (capacity/PPU) that exposes Power BI semantic models as **Analysis Services** databases, enabling third-party tools (SSMS, Tabular Editor, DAX Studio, ALM Toolkit, Excel) to **read and write** models, script deployments, manage partitions, run advanced refresh and use enterprise ALM practices. It requires the XMLA read/write setting to be enabled by the admin.

### Q128. What are dataflows?

**Dataflows** are Power Query transformations authored and executed **in the service** (Power Query Online), producing reusable tables stored in a managed lake (Gen1) or in Fabric items such as lakehouses and warehouses (**Dataflow Gen2**). Benefits: centralize and reuse cleaning logic across many semantic models, decouple extraction from modeling, and reduce load on sources (extract once, reuse many). Considerations: they add a layer to maintain, performance features (linked/computed entities, enhanced compute) depend on capacity, and heavy transformation is often better in a warehouse or Spark **(version-sensitive; Gen1 vs Gen2 capabilities differ)**.

### Q129. What is the difference between a dataflow and a semantic model?

A **dataflow** prepares and stores **clean tables** (ETL, no relationships/measures). A **semantic model** adds **relationships, measures, hierarchies and security** (the analytical layer). A common pattern: dataflow or warehouse for preparation, semantic model for analytics, reports on top.

### Q130. How does embedding work?

Embed Power BI content in custom apps, portals and websites using the JavaScript client library and REST APIs.
- **User owns data (embed for your organization):** viewers authenticate with their own Entra ID identity and need appropriate licenses.
- **App owns data (embed for your customers):** the application authenticates with a **service principal or master account** and generates **embed tokens**; end users do not need Power BI licenses; requires an **Embedded (A) or Fabric/Premium capacity**; RLS is passed via the token's identity.
- **Secure embed** and **Publish to web** (public) are separate options.
Use cases include ISV analytics products and internal portals. Capacity sizing and token management are key operational concerns.

### Q131. What are Power BI REST API and automation options?

REST APIs and PowerShell cmdlets let you automate: trigger and monitor refreshes, manage workspaces and permissions, deploy reports, clone content, retrieve activity logs, manage gateways and embed tokens. Combined with **Power Automate**, **Azure DevOps** or **GitHub Actions**, this supports CI/CD, governance automation and bulk administration. With Fabric, **Semantic Link** (Python) and notebooks allow programmatic analysis and management of semantic models **(version-sensitive)**.

### Q132. What are Metrics (Goals) and scorecards?

A feature to define **business metrics with targets and owners**, track progress, and connect to report visuals for automatic updates, grouped into scorecards for goal tracking **(version-sensitive; naming and availability evolve)**.

### Q133. What is Analysis Services (SSAS/AAS) relevance now?

Azure Analysis Services and SSAS Tabular are the origin of the Power BI engine. Microsoft positions Power BI semantic models (with Premium/Fabric capacity) as the strategic enterprise semantic layer, and offers migration paths from AAS. Existing SSAS/AAS models can be consumed by Power BI through **live connection** or DirectQuery.

---

# Part 13: Developer Tooling and DevOps

### Q134. What external tools do Power BI professionals use?

| Tool | Purpose |
|---|---|
| **DAX Studio** | Write and run DAX queries, Server Timings, query plans, VertiPaq Analyzer |
| **Tabular Editor** | Edit the model metadata: measures, calculation groups, perspectives, OLS, Best Practice Analyzer, scripting, bulk changes |
| **ALM Toolkit** | Compare and merge semantic model schemas for deployments |
| **Bravo** | Analyze model size, format DAX, manage date tables |
| **SSMS / Azure Data Studio** | Connect via XMLA, run DMVs and scripts, process partitions |
| **Power BI Helper / Measure Killer** | Documentation and unused-object analysis (community tools) |
| **Performance Analyzer** | Built into Desktop for visual-level timings |

### Q135. What is the PBIP project format and Git integration?

The **Power BI Project (PBIP)** format saves a report and semantic model as **folders of text files** (the model in **TMDL**, Tabular Model Definition Language; the report in a JSON-based format such as PBIR) instead of a single binary PBIX, enabling **meaningful source control diffs, branching, pull requests and collaborative development** **(version-sensitive; formats and GA status evolve)**. Fabric workspaces can integrate with Git (Azure DevOps or GitHub) to sync items.

### Q136. What does a good Power BI DevOps process look like?

Use PBIP/Git (or at least versioned PBIX with naming conventions); separate dev, test and prod workspaces; use **deployment pipelines** (with rules for data sources and parameters) or automated release pipelines via REST/XMLA; separate **shared semantic models** from **reports**; run **Best Practice Analyzer** and performance checks in CI; document changes; protect production with permissions; and monitor refresh and usage.

### Q137. How do I document and govern a model?

Add **descriptions** to tables, columns and measures; hide technical fields; use consistent naming; keep a data dictionary generated from the model (Tabular Editor or scripts); certify trusted models; track lineage in the service (**Lineage view** and **impact analysis**); establish naming standards, review checklists and ownership.

---

# Part 14: Troubleshooting: Common Errors and Causes

### Q138. Refresh fails with a credentials or "data source" error.

Credentials expired or changed; the data source in the service is not mapped to the gateway; privacy level mismatch; OAuth token expired; the service account password rotated; firewall rules changed. Re-enter credentials in the semantic model settings, verify the gateway data source, and test connection.

### Q139. The gateway is offline or refresh times out.

Check the gateway service is running and has internet access, the machine has adequate CPU/memory, the gateway is updated, and the cluster has available members. Heavy Power Query transformations that do not fold run on the gateway and may overwhelm it. Refresh timeouts relate to limits (2 hours Pro, longer on capacity **(version-sensitive)**): reduce data, enable incremental refresh, and fix folding.

### Q140. "The key didn't match any rows in the table" or errors after a source change.

A column or table was renamed or removed at the source, so a Power Query step now references something that does not exist. Fix the step, or make the query resilient (for example using `Table.SelectColumns` with `MissingField.Ignore`).

### Q141. "Formula.Firewall: Query references other queries or steps, so it may not directly access a data source".

The privacy-level firewall blocked combining data from sources with incompatible privacy levels in a single step. Fixes: set consistent privacy levels, restructure queries so data access and combination occur in separate queries, or (carefully, with security review) ignore privacy levels. The error is a protection against leaking data between sources.

### Q142. "A circular dependency was detected".

Two calculated columns, tables or relationships reference each other directly or indirectly (for example calculated columns in two tables each depending on the other, or bidirectional relationships combined with certain calculated columns). Break the cycle by converting one calculation to a measure, moving logic to Power Query, or removing a bidirectional filter.

### Q143. "A table of multiple values was supplied where a single value was expected".

A measure or column expression returned multiple values where DAX needed a scalar, such as using `Table[Column]` in a measure without an aggregation, or using `VALUES` when multiple values are in context. Use `SELECTEDVALUE`, an aggregation (`MAX`), or iterate with `SUMX`/`MAXX`.

### Q144. Visuals show blanks or "(Blank)" categories.

Unmatched keys (facts with no dimension row produce a blank member), missing relationships, inactive relationship, filter direction not propagating, or a measure that returns BLANK in that context. Add an "Unknown" row to dimensions in the source and ensure referential integrity.

### Q145. Totals do not match the sum of the rows.

Measures are evaluated in the total's filter context, which is correct for ratios, averages, distinct counts and non-additive measures. If an additive total is expected, use an iterator over the displayed level with `SUMX(VALUES(...), [Measure])` and consider `ISINSCOPE`.

### Q146. "Resource exceeded" or "There's not enough memory to complete this operation".

A query used more memory than allowed (for example huge distinct counts, big iterators, materializing large tables, or too many visuals). Reduce cardinality, optimize measures, filter data, use aggregations, or move to larger capacity.

### Q147. Time intelligence returns wrong or blank results.

The Date table is not marked, is missing dates, is not contiguous or the relationship uses a datetime column with time portions; auto date/time interfering; or using the fact table's date column instead of the Date table's column in the function. Fix the date table and use `'Date'[Date]`.

### Q148. Incremental refresh is slow or not working.

The date filter does not **fold**, so the whole table is loaded and then filtered; `RangeStart`/`RangeEnd` data types are wrong; the model was not published to the service; the policy window does not match data; or the source column used for filtering is not indexed. Check **View Native Query** on the filter step.

### Q149. DirectQuery visuals are slow or time out.

Source indexes or statistics are missing, queries are complex, too many visuals or slicers, high-cardinality fields, bidirectional filters, or the source is under load. Optimize the source, simplify the model, use aggregations/composite models, and reduce visual count.

### Q150. Report works for me but a colleague sees no data or an error.

Check RLS roles and membership, Build permission, workspace role versus app audience, whether they have a license for the capacity type, row-level filtering on an unmatched value, or missing access to the underlying source in DirectQuery/SSO scenarios.

---

# Part 15: Scenario-Based Interview Questions

### Q151. How would you design a semantic model for sales analytics from scratch?

Define the **business questions and grain** (for example one row per order line); source from a warehouse star schema if available; build `Sales` (fact) with integer keys and numeric measures; `Product`, `Customer`, `Store`, and a shared `Date` dimension; flatten snowflakes; use single-direction one-to-many relationships; mark the date table; create a measure table with explicit measures (`Total Sales`, `Total Cost`, `Margin`, `Margin %`, time intelligence variants); hide keys; set formats; disable auto date/time; define RLS if needed; remove unused columns; test performance; publish as a certified shared model; build thin reports with live connections.

### Q152. A report takes 30 seconds to load. What do you do?

Run **Performance Analyzer** to find slow visuals; copy the DAX query into **DAX Studio** with Server Timings to see storage versus formula engine time; check the model size and cardinality with VertiPaq Analyzer; look for iterators with context transition, `FILTER` on big tables, bidirectional filters, high-cardinality distinct counts, and too many visuals/slicers. Fix with model simplification, aggregations, better measures, fewer visuals, query reduction, and (for DirectQuery) source tuning. Re-measure after each change.

### Q153. When would you use DirectQuery instead of Import?

When data volume exceeds Import limits, when data must be real-time and cannot be satisfied by frequent or incremental/hybrid refresh, when data must remain in the source for regulatory reasons, or when row-level security is implemented in the source. Otherwise choose Import for performance and flexibility, possibly with aggregations/composite models for scale.

### Q154. How would you implement dynamic RLS for a sales hierarchy where managers see their team?

Create a **security mapping table** (user, subordinate or territory) or a **parent-child hierarchy** flattened using `PATH`, `PATHITEM` and `PATHCONTAINS`; write a role filter such as `PATHCONTAINS ( Employee[Path], LOOKUPVALUE ( Employee[EmployeeKey], Employee[Email], USERPRINCIPALNAME () ) )` on the dimension; relate to the fact so filters propagate; test with View as; remember that workspace editors bypass RLS and verify performance.

### Q155. How do you handle a fact table with 500 million rows?

Keep the detailed fact in the source (DirectQuery) or a lakehouse (Direct Lake), build an **aggregation table** (Import) at a commonly queried grain, set dimensions to **Dual**, use **incremental refresh** if Import, reduce columns and cardinality, push heavy calculations to the warehouse, and design reports to default to aggregated views with drillthrough to detail.

### Q156. What is your approach to a "single source of truth" in a large organization?

Centralize **certified shared semantic models** owned by a BI team, with documented measures and RLS; allow business users to build **reports (thin) on top of them** with live connections or composite models; use **deployment pipelines**, **endorsement**, **lineage** and governance policies; discourage multiple copies of the same data; and provide training and a data catalog.

### Q157. How do you compare two periods dynamically (user picks Period A and Period B)?

Create two **disconnected date slicers (or parameter tables)**, and measures that compute the metric using `TREATAS` to apply selected periods:

```dax
Sales Period A =
CALCULATE ( [Total Sales], TREATAS ( VALUES ( 'Period A'[Date] ), 'Date'[Date] ) )

Sales Period B =
CALCULATE ( [Total Sales], TREATAS ( VALUES ( 'Period B'[Date] ), 'Date'[Date] ) )

Difference = [Sales Period A] - [Sales Period B]
```

### Q158. What is the difference between a measure and a calculated column in terms of when you would choose each, with an example?

A **calculated column** `Profit Margin Category` (High/Medium/Low per product) is needed for slicing and grouping, so it is a column (ideally computed in Power Query or the warehouse). A **measure** `Total Profit` must respond to filters and aggregate correctly, so it is a measure. If you computed profit as a calculated column and summed it, it would work but waste memory and could not respond to complex contexts like percent of total as flexibly.

### Q159. How would you migrate a large Excel-based reporting process to Power BI?

Inventory reports and logic; move source data to a database/warehouse or SharePoint/OneDrive as a stepping stone; recreate logic with Power Query and DAX (convert complex formulas into measures); build a star schema model; replicate key reports; run parallel validation against Excel results; use **Analyze in Excel** to give finance users PivotTables on the governed model; train users; and decommission Excel processes gradually.

### Q160. How do you ensure data quality in Power BI?

Validate upstream (warehouse constraints, tests), add data-quality checks in Power Query (null/duplicate/range checks, error rows), reconcile totals with source systems, create **audit pages/cards** (row counts, last refresh time, max date), use **assertions** in the pipeline, certify models, and monitor refresh failures and anomalies.

### Q161. What is the difference between Power BI Desktop, the service and Report Server?

Desktop authors; the service hosts, shares, refreshes and secures in the cloud; Report Server hosts on-premises with a separate Desktop version and slower feature cadence. Choose based on data sovereignty and feature needs.

### Q162. How do you design for multiple currencies or languages?

Use a **currency dimension and exchange rate fact** with date-based rates; compute converted amounts in measures (or pre-convert in ETL); use **calculation groups** or a currency slicer with dynamic format strings; for languages use **translations** (perspectives and metadata translations via tools) and report-level labels driven by a parameter table.

### Q163. How do you secure a report published for external customers?

Use **embed for your customers (app owns data)** with a service principal and RLS identities in the embed token, on an Embedded or Fabric capacity; isolate customer data via RLS or separate workspaces/models; avoid Publish to web; apply sensitivity labels and audit; monitor capacity and tokens.

---

# Part 16: Glossary

- **Aggregation table:** pre-summarized table used to accelerate queries on a large detail table.
- **App:** published package of workspace content for audiences.
- **Auto date/time:** feature creating hidden date tables for each date column; usually disabled.
- **Bridge table:** table resolving many-to-many relationships between two dimensions.
- **Calculated column:** column computed at refresh using row context and stored in the model.
- **Calculation group:** set of reusable calculation items modifying measures.
- **Cardinality:** number of distinct values in a column; also relationship type (one-to-many, and so on).
- **Composite model:** model mixing storage modes or sources.
- **Context transition:** conversion of row context into filter context by CALCULATE.
- **Cross-filter direction:** direction in which a relationship propagates filters.
- **Dataflow:** reusable Power Query transformations hosted in the service.
- **Dimension table:** descriptive table used to slice and filter facts.
- **DirectQuery:** storage mode that queries the source at report time.
- **Direct Lake:** Fabric storage mode reading Delta files in OneLake.
- **Dual:** storage mode acting as Import or DirectQuery depending on the query.
- **Fact table:** table of transactions or events with measures and foreign keys.
- **Filter context:** set of filters active when DAX is evaluated.
- **Formula engine / storage engine:** the two engine components that evaluate DAX.
- **Gateway:** software enabling the service to reach private/on-premises data.
- **Grain:** level of detail a fact table row represents.
- **Import:** storage mode that loads data into compressed in-memory VertiPaq.
- **Incremental refresh:** policy-based partition refresh limited to recent data.
- **Live connection:** report connected to an external semantic model with no local model.
- **M:** the Power Query formula language.
- **Measure:** DAX expression evaluated at query time in filter context.
- **OLS / RLS:** object-level / row-level security.
- **PBIP:** Power BI project folder format for source control.
- **Query folding:** pushing Power Query steps to the source as a native query.
- **Role-playing dimension:** one dimension used in several roles (for example Order Date, Ship Date).
- **Row context:** the current row during row-by-row evaluation.
- **Semantic model:** the model (data, relationships, measures, security) that reports use; formerly dataset.
- **Star schema:** a fact table surrounded by dimension tables.
- **VertiPaq:** the in-memory columnar compression engine.
- **Workspace:** container for collaborating on content in the service.
- **XMLA endpoint:** open endpoint exposing models to Analysis Services tools.

---

# Part 17: One-Page Cheat Sheet

**Architecture:** Desktop authors; service shares/refreshes/secures; gateway reaches on-premises; semantic model (VertiPaq + DAX) powers reports. Separate shared models from thin reports.

**Storage modes:** Import (default, fastest) → DirectQuery (live, slower) → Dual (dimensions in composite) → Live connection (shared model) → Direct Lake (Fabric, Delta in OneLake).

**Modeling rules:** star schema; one-to-many single-direction relationships; flatten snowflakes; integer surrogate keys; one marked Date table; Auto date/time off; hide keys; explicit measures in a measures table; avoid bidirectional and many-to-many unless needed.

**Size drivers:** cardinality first; remove unused columns; split datetime; Fixed decimal for money; avoid GUID/long text; calculated columns cost memory, measures do not.

**Types and bytes:** Whole number 8 bytes (int64); Decimal 8 bytes (double, 15 digits, drift possible); Fixed decimal 8 bytes (exact, 4 decimals); Date/Time 8 bytes; Text about 2 bytes per character plus dictionary; Boolean about 1 bit compressed. After compression, size depends on distinct values, not raw type size.

**DAX essentials:** filter context vs row context; CALCULATE modifies filters; context transition (measure in row context); ALL/REMOVEFILTERS/ALLEXCEPT/ALLSELECTED; variables; DIVIDE; avoid FILTER over big fact tables; iterators (X functions) only when needed; time intelligence needs a marked contiguous Date table.

**Power Query:** fold early (View Native Query); set types explicitly; unpivot wide data; disable load for staging queries; parameters RangeStart/RangeEnd for incremental refresh; Formula.Firewall = privacy levels.

**Service:** roles Admin/Member/Contributor/Viewer; **RLS affects Viewers only**; publish apps for consumers; scheduled refresh limits depend on license; standard gateway in a cluster for production; deployment pipelines for dev/test/prod.

**Performance workflow:** Performance Analyzer → DAX Studio (Server Timings) → VertiPaq Analyzer → fix model/DAX/visual count → re-measure.

**Security:** RLS (static/dynamic with USERPRINCIPALNAME), OLS via Tabular Editor, sensitivity labels, tenant settings; never use Publish to web for private data.

---

**Final note.** Power BI is updated monthly and its licensing, Fabric integration, preview features and limits change often. Items marked **(version-sensitive)** should be verified in the current Microsoft documentation before an interview, exam or production design.
