# Databricks: The Complete FAQ and Study Reference

**Purpose of this document.** This is a single, self-contained reference that explains Databricks from first principles to production practice. It is written in question-and-answer form. Every answer explains **what** something is, **why** it exists, **how** it works internally, and **what it means for performance and cost**. It is designed to be loaded into a study tool such as NotebookLM so the tool can explain any topic clearly.

**How to read it.**
- Each part is self-contained, but later parts build on earlier ones (Spark and Delta first, then governance, compute, pipelines, performance).
- Terms are defined the first time they appear and again in the glossary at the end.
- Where a product name, default value or feature is known to change over time, it is marked **(version-sensitive)**. Always confirm those against the current Databricks documentation before relying on them in production or an exam.

**Contents**
1. Platform fundamentals
2. Apache Spark essentials (the engine underneath)
3. File formats, Parquet and data types (with byte sizes)
4. Delta Lake in depth
5. Unity Catalog and governance
6. Compute: clusters, SQL warehouses, serverless, Photon, billing
7. Ingestion, streaming and the Medallion Architecture
8. Lakeflow Declarative Pipelines (formerly Delta Live Tables)
9. Orchestration, CI/CD and deployment
10. Databricks SQL, BI and analytics
11. Performance tuning playbook
12. Security, networking and operations
13. Azure Databricks specifics
14. Machine learning and AI on Databricks (overview)
15. Troubleshooting: common errors and their causes
16. Scenario-based interview questions
17. Glossary
18. One-page cheat sheet

---

# Part 1: Platform Fundamentals

### Q1. What is Databricks, and what problem does it solve?

Databricks is a cloud data and AI platform founded by the original creators of Apache Spark. It runs on Azure, AWS and Google Cloud as a managed service. It brings together data engineering, SQL analytics, streaming, data science, machine learning and governance in one place.

**The problem it solves.** Historically, organizations ran two separate systems:
- A **data lake** (cheap object storage holding raw files) used by data scientists and engineers. It is flexible and cheap but has no transactions, weak governance and poor SQL performance.
- A **data warehouse** used by analysts and BI tools. It is fast and reliable but expensive, mostly limited to structured data, and weaker for machine learning.

Keeping both meant copying data between them, maintaining two security models, and constantly fighting data that was out of sync. Databricks proposes a **lakehouse**: keep one copy of data in open formats on cheap cloud storage, and add the reliability, performance and governance of a warehouse on top.

### Q2. What is the Lakehouse architecture, and what are its layers?

A lakehouse is an architecture that provides warehouse-like capabilities (ACID transactions, schema control, fast SQL, governance) directly on data stored in low-cost, open-format lake storage. On Databricks it is made of three cooperating layers:

1. **Storage layer.** Your own cloud object storage (Azure Data Lake Storage Gen2, Amazon S3, Google Cloud Storage). Data is stored as Parquet files.
2. **Table/transaction layer: Delta Lake.** An open table format that adds a transaction log on top of Parquet, giving ACID transactions, versioning (time travel), schema enforcement and efficient updates and deletes.
3. **Governance layer: Unity Catalog.** A central place for permissions, auditing, lineage and data discovery.

On top of these sit the compute engines (Spark, Photon, SQL warehouses) and the workloads (ETL, BI, streaming, ML).

### Q3. How do a data lake, a data warehouse and a lakehouse compare?

| Aspect | Data Lake | Data Warehouse | Lakehouse |
|---|---|---|---|
| Data types | Everything (raw files) | Mostly structured | Everything |
| Storage format | Open files (Parquet, CSV, JSON) | Proprietary | Open (Parquet + Delta log) |
| ACID transactions | No | Yes | Yes (through Delta) |
| Schema control | Weak (schema-on-read) | Strong (schema-on-write) | Both available |
| BI/SQL performance | Weak | Strong | Strong |
| ML and data science | Good | Limited | Good |
| Cost | Low | High | Low |
| Vendor lock-in | Low | High | Low (open formats) |

### Q4. What is the difference between the control plane and the compute plane?

The architecture separates *managing* the platform from *processing* your data.

- **Control plane.** Hosted and managed by Databricks in its own cloud account. It contains the web application, notebook and job services, the cluster manager, the REST APIs, and workspace metadata (notebook code, job definitions, and so on). It does **not** store your business data.
- **Compute plane.** Where your data is actually processed.
  - **Classic compute plane:** virtual machines run inside *your* cloud subscription and your virtual network. They read data from your storage. You pay the cloud provider for the VMs and Databricks for the software (DBUs).
  - **Serverless compute plane:** Databricks runs and manages the compute in its own account. You do not choose VM types or manage networking for the VMs. Startup is much faster.

Your data stays in your own storage in both models. The practical difference is who operates the infrastructure.

### Q5. What is a Databricks workspace, and how does it relate to the account and metastore?

- An **account** is the top-level organizational container (billing, users, workspaces, Unity Catalog metastores). It is managed through the account console.
- A **workspace** is an environment where users work: notebooks, jobs, clusters, SQL warehouses, dashboards. One account can have many workspaces (for example dev, test, prod).
- A **metastore** is the Unity Catalog container for metadata and governance. You typically create **one metastore per cloud region** and attach many workspaces in that region to it, so governance is shared across workspaces.

### Q6. What are the main persona areas inside Databricks?

- **Data Engineering:** notebooks, jobs, Lakeflow Declarative Pipelines, Auto Loader, Delta Lake.
- **Databricks SQL:** SQL editor, SQL warehouses, dashboards, alerts, and AI/BI features for analysts.
- **Machine Learning / AI:** MLflow, feature engineering, model serving, vector search, AI functions.
- **Governance and administration:** Unity Catalog, access control, lineage, system tables, audit logs.

### Q7. What is the Databricks Runtime (DBR)?

The Databricks Runtime is the set of software that runs on every cluster node: Apache Spark plus Databricks optimizations, Delta Lake, Photon (optionally), Python/R/Scala/Java runtimes, and common libraries. There are variants such as Databricks Runtime ML (adds machine learning libraries and GPU support). Runtimes are versioned; **Long-Term Support (LTS)** versions get extended support and are recommended for production, because non-LTS versions are retired quickly. Pin production jobs to an LTS runtime so behavior does not change unexpectedly.

---

# Part 2: Apache Spark Essentials (The Engine Underneath)

Understanding Spark is essential because almost every Databricks performance question reduces to how Spark executes work.

### Q8. What is Apache Spark and why is it fast?

Spark is a distributed data-processing engine. It splits data into **partitions** and processes partitions in parallel across many machines. It is faster than older systems like Hadoop MapReduce mainly because it keeps intermediate results in **memory** where possible instead of writing them to disk between every step, and because it optimizes the whole query plan before running it.

### Q9. What are the driver and executors?

- The **driver** is the process that runs your program's main logic, builds the execution plan, and coordinates the work. It holds the Spark session.
- **Executors** are worker processes that run **tasks** on partitions and hold cached data. Each executor has cores (parallel task slots) and memory.
- The **cluster manager** allocates resources. On Databricks this is handled for you.

**Why it matters:** operations like `collect()` and `toPandas()` pull data from executors into the single driver's memory. Doing this with large data causes driver out-of-memory failures.

### Q10. What are RDDs, DataFrames and Datasets?

- **RDD (Resilient Distributed Dataset):** the original low-level API. A distributed collection of objects with no schema. Spark cannot optimize it much.
- **DataFrame:** a distributed table with named, typed columns. Spark understands the structure, so the optimizer (Catalyst) can reorder and optimize operations. This is the standard API in Python, SQL, Scala and R.
- **Dataset:** a typed API available in Scala and Java only.

**Recommendation:** use DataFrames or Spark SQL. Avoid RDDs unless you truly need low-level control, because you lose optimization.

### Q11. What is lazy evaluation? What is the difference between transformations and actions?

- **Transformations** describe a computation (`select`, `filter`, `join`, `groupBy`). They are **lazy**: nothing runs when you call them. Spark only records them in a plan.
- **Actions** trigger actual execution (`count`, `show`, `collect`, `write`, `save`).

**Why lazy?** Because Spark can look at the *whole* chain of operations, optimize it (for example push filters earlier, prune unneeded columns), and then execute it efficiently. A practical consequence: an error in a transformation may only appear when an action runs, and calling the same action twice recomputes everything unless you cache.

### Q12. What are narrow and wide transformations?

- **Narrow transformation:** each output partition depends on a single input partition (`filter`, `select`, `map`, `withColumn`). No data needs to move between machines. These are cheap and are pipelined together.
- **Wide transformation:** output partitions depend on data from many input partitions (`groupBy`, `join`, `distinct`, `orderBy`, `repartition`). Data must be redistributed across the network. This redistribution is called a **shuffle**.

### Q13. What is a shuffle and why is it expensive?

A shuffle redistributes rows across the cluster so that rows with the same key end up on the same partition. It is expensive because it involves:
1. Writing intermediate data to local disk on the map side.
2. Transferring data over the network.
3. Reading and sorting or hashing on the reduce side.
4. Possible spill to disk if memory is insufficient.

Shuffle boundaries also split a job into separate **stages**. Most Spark tuning is about reducing the number of shuffles and the amount of data shuffled.

### Q14. How are jobs, stages and tasks related?

- An **action** triggers a **job**.
- A job is divided into **stages** at shuffle boundaries.
- A stage is divided into **tasks**, one per partition. A task is the smallest unit of work, run by one executor core.

So if a stage has 200 partitions, it has 200 tasks. If the cluster has 40 cores, they run in about 5 waves.

### Q15. What is the Catalyst optimizer and what is Tungsten?

- **Catalyst** is Spark's query optimizer. It converts your query into a logical plan, applies rule-based and cost-based optimizations (predicate pushdown, column pruning, constant folding, join reordering), and then selects a physical plan.
- **Tungsten** is the execution engine improvements: compact binary memory layout, off-heap memory, cache-friendly algorithms and whole-stage code generation (turning many operators into one tight compiled function).

You can inspect the plan with `df.explain()` or `EXPLAIN` in SQL.

### Q16. What is Adaptive Query Execution (AQE)?

AQE re-optimizes the plan **while the query runs**, using real statistics collected at shuffle boundaries. It is enabled by default in modern Spark and Databricks Runtime. Its three main features:
1. **Coalescing shuffle partitions:** merges many tiny post-shuffle partitions into fewer reasonably sized ones.
2. **Switching join strategies:** for example, converting a sort-merge join to a broadcast join when one side turns out to be small at runtime.
3. **Skew join handling:** splitting an oversized partition into smaller sub-partitions so one slow task does not stall the stage.

### Q17. What are the join strategies in Spark?

| Strategy | When used | Cost profile |
|---|---|---|
| **Broadcast hash join** | One side is small enough to send to every executor | Fastest; no shuffle of the large side |
| **Sort-merge join** | Both sides large (default for large joins) | Shuffles and sorts both sides |
| **Shuffle hash join** | Medium-size side, hash-able | Shuffles both sides; builds a hash table |
| **Broadcast nested loop / cartesian** | Non-equality joins or no condition | Very expensive; avoid |

The auto-broadcast threshold (`spark.sql.autoBroadcastJoinThreshold`) defaults to 10 MB in open-source Spark **(version-sensitive; Databricks and AQE may adapt this)**. You can force it with a hint: `broadcast(df)` or `/*+ BROADCAST(t) */`.

### Q18. How are partitions determined, and what are good sizes?

- When reading files, Spark creates input partitions of up to `spark.sql.files.maxPartitionBytes` (default **128 MB**).
- After a shuffle, the number of partitions is `spark.sql.shuffle.partitions` (default **200**), unless AQE coalesces them.
- A practical target is partitions of roughly **100 to 200 MB** each. Too few and large partitions risk memory spill and underuse cores. Too many tiny partitions cause scheduling overhead.

### Q19. What is the difference between `repartition` and `coalesce`?

- `repartition(n)` performs a **full shuffle** and can increase or decrease partitions, producing evenly sized partitions. It can also repartition by column: `repartition(n, "col")`.
- `coalesce(n)` can only **decrease** partitions and avoids a full shuffle by merging neighbours, but resulting partitions may be uneven.

Use `coalesce` for cheaply reducing output file count. Use `repartition` when you need balance or key-based distribution.

### Q20. What is data skew and how do you spot it?

Skew means some keys have far more rows than others, so one partition (and one task) is far bigger than the rest. Symptoms: in the Spark UI, almost all tasks of a stage finish quickly while one or two run for a very long time; a stage seems stuck at the last few tasks (for example 199 of 200); one task shows very high shuffle read or spill.

Fixes: rely on AQE skew handling; **salt** the hot key (append a random suffix to spread it, and replicate the other side); filter out null or default placeholder keys before joining; broadcast the smaller side; pre-aggregate before the join.

### Q21. What is spill and what causes executor out-of-memory (OOM) errors?

**Spill** happens when data for a task does not fit in execution memory, so Spark writes it to disk temporarily. It works but is slow. OOM happens when memory is truly exhausted. Common causes: skewed partitions that are too large, partitions too big (too few of them), broadcasting a table that is too large, huge `collect()` to the driver, exploding arrays producing many rows, or too many cores per executor sharing too little memory. Remedies: increase partitions, fix skew, use a larger-memory instance type, reduce broadcast size.

### Q22. How does Spark memory work at a high level?

Executor memory is divided into regions. **Unified memory** (controlled by `spark.memory.fraction`, about 60% of the usable heap by default) is shared between **execution** (shuffles, joins, sorts, aggregations) and **storage** (cached data), and each can borrow from the other. The remainder holds user objects and internal metadata. There is also overhead memory outside the heap for things like Python processes and native code. You rarely tune these directly on Databricks, but understanding them explains spill and OOM behavior.

### Q23. What is the Spark UI and what should I look at?

The Spark UI is the main diagnostic tool. Key tabs:
- **Jobs / Stages:** durations, task counts, and task-level metrics. Look for skew (a max task time far above the median), spill, and large shuffle read/write.
- **SQL / DataFrame:** the executed query plan with metrics per operator; shows which join strategy was used and how many rows each step produced.
- **Storage:** what is cached and how much memory it uses.
- **Executors:** per-executor memory, garbage collection time, task failures, lost executors.
- **Environment:** the configuration actually in effect.

### Q24. When should I use `cache()` or `persist()`?

Only when the same DataFrame is reused several times in one application and is expensive to recompute. Caching is lazy (it fills on the first action). Caching data used once, or huge data that exceeds memory, wastes resources. Always `unpersist()` when done. On Databricks there is also an automatic **disk cache** (see Q93).

### Q25. Why are Python UDFs slow and what are the alternatives?

Regular Python UDFs run row by row and require serializing data between the JVM and a Python process, and the optimizer treats them as a black box. Preference order: (1) **built-in Spark SQL functions**, which are optimized and Photon-compatible; (2) **Pandas (vectorized) UDFs**, which process batches using Apache Arrow with far lower overhead; (3) plain Python UDFs as a last resort. Scala/Java UDFs avoid the Python serialization cost but are still opaque to the optimizer.

---

# Part 3: File Formats, Parquet and Data Types (With Byte Sizes)

### Q26. What is Parquet and why is it the foundation of Delta Lake?

Parquet is an open-source **columnar** file format. Instead of storing data row by row, it stores each column's values together. Benefits:
- **Column pruning:** a query selecting 3 of 100 columns reads only those 3 columns from disk.
- **Better compression:** values in one column are similar, so encodings and codecs work very well.
- **Predicate pushdown:** each file and row group stores min/max statistics per column, so readers can skip data that cannot match a filter.
- **Efficient analytics:** aggregations over a column touch contiguous memory.

### Q27. How is a Parquet file structured?

A file is divided into **row groups** (large horizontal slices of rows, commonly tens to hundreds of MB). Within each row group, each column is stored as a **column chunk**, split into **pages**. The **footer** at the end of the file holds the schema and statistics (min, max, null count) for each column chunk. Readers read the footer first, decide which row groups and columns they need, and fetch only those byte ranges.

Common encodings include dictionary encoding (repeated values stored once plus small integer references), run-length encoding and bit-packing. Compression codecs include Snappy (default in many Spark setups, fast), Zstd (better ratio, increasingly popular) and Gzip (higher ratio, slower).

### Q28. How do Parquet, CSV, JSON, Avro and ORC compare?

| Format | Layout | Schema | Best for | Weakness |
|---|---|---|---|---|
| **CSV** | Row, text | None (all text unless inferred) | Simple exchange, small files | Slow, large, no types, parsing ambiguity |
| **JSON** | Row, text | Implicit, nested | APIs, semi-structured logs | Verbose, slow to scan |
| **Avro** | Row, binary | Embedded | Streaming/Kafka, write-heavy | Poor for analytic column scans |
| **ORC** | Columnar | Embedded | Hive ecosystem | Less common outside Hive |
| **Parquet** | Columnar | Embedded | Analytics and lakehouse | Not ideal for tiny frequent row writes |
| **Delta** | Parquet + log | Enforced | Reliable lakehouse tables | Requires Delta-aware engines (though UniForm helps) |

### Q29. What are the main Spark SQL data types and how much space do they take?

These are the in-memory sizes in Spark. Choosing the smallest type that safely fits your data reduces memory use, shuffle volume and I/O.

| Type | Size | Notes |
|---|---|---|
| `BOOLEAN` | 1 byte in memory | Bit-packed (about 1 bit) in Parquet files |
| `TINYINT` | 1 byte | Range -128 to 127 |
| `SMALLINT` | 2 bytes | Range -32,768 to 32,767 |
| `INT` | 4 bytes | About ±2.1 billion |
| `BIGINT` | 8 bytes | About ±9.2 quintillion |
| `FLOAT` | 4 bytes | Single precision, about 7 significant digits |
| `DOUBLE` | 8 bytes | Double precision, about 15 to 16 significant digits |
| `DECIMAL(p,s)` | Up to 16 bytes | Precision up to 38. Small precision (up to 18 digits) is stored compactly as a 64-bit value internally |
| `DATE` | 4 bytes | Days since 1970-01-01 |
| `TIMESTAMP` | 8 bytes | Microseconds since epoch, interpreted in UTC with session time zone for display |
| `TIMESTAMP_NTZ` | 8 bytes | Timestamp without time zone |
| `STRING` | Variable | UTF-8 bytes; length-dependent, plus overhead |
| `BINARY` | Variable | Raw bytes |
| `ARRAY`, `MAP`, `STRUCT` | Variable | Nested types; cost is the sum of their elements plus overhead |
| `VARIANT` | Variable | Semi-structured type for JSON-like data **(version-sensitive)** |

**Important nuance on disk vs memory.** Parquet stores values using physical types: `INT32`, `INT64`, `FLOAT`, `DOUBLE`, `BYTE_ARRAY`, `BOOLEAN`. A `TINYINT` or `SMALLINT` is stored physically as `INT32`, but encoding and compression shrink it dramatically on disk, so the on-disk saving is smaller than the in-memory saving. Decimals use `INT32` for precision up to 9, `INT64` for up to 18, and a fixed-length byte array above that.

### Q30. How should I choose data types for performance and cost?

- Use `INT` instead of `BIGINT` when values are guaranteed to fit. Halving a column from 8 to 4 bytes halves its in-memory and shuffle footprint.
- Use `DATE` rather than `STRING` for dates. Dates take 4 bytes and enable range filtering, statistics-based file skipping and date functions. A string like `'2026-10-19'` takes 10 bytes and compares as text.
- Use `DECIMAL(p,s)` for money, never `FLOAT` or `DOUBLE`. Binary floating-point cannot represent many decimal fractions exactly, causing rounding errors.
- Avoid very long strings in columns you filter, join or Z-order on. Delta statistics for strings are truncated (about the first 32 characters), which weakens skipping for long, similar strings.
- Prefer numeric surrogate keys for joins over long string keys. They are smaller and faster to hash and compare.
- Avoid unnecessary wide tables with hundreds of columns: statistics are collected on only the first 32 columns by default (see Q38).

---

# Part 4: Delta Lake in Depth

### Q31. What is Delta Lake?

Delta Lake is an open-source storage layer that brings reliability to data lakes. A Delta table is simply:
1. A folder of **Parquet data files**, and
2. A **`_delta_log`** folder holding the transaction log.

Delta is the default table format on Databricks. It supports ACID transactions, scalable metadata, time travel, schema enforcement and evolution, efficient `UPDATE`/`DELETE`/`MERGE`, and unified batch and streaming.

### Q32. How does the transaction log work?

Every change to the table (insert, update, delete, schema change, `OPTIMIZE`) produces a new numbered commit file in `_delta_log`, such as `00000000000000000012.json`. A commit records **actions**, such as:
- `add`: a data file was added, with its path, size and statistics (min/max/null counts per column).
- `remove`: a data file is no longer part of the table (it is not physically deleted yet).
- `metaData`: the schema, partition columns and table properties.
- `protocol`: the minimum reader/writer versions required.
- `commitInfo`: who did what and when (this feeds `DESCRIBE HISTORY`).

To read the table, an engine replays the log to figure out which files make up the current version. To avoid replaying thousands of JSON files, Delta writes a **checkpoint** (a Parquet file summarizing the table state) every 10 commits by default. Readers load the latest checkpoint and then only the commits after it.

**Why this design is powerful:** the state of the table is defined by the log, not by directory listing. Files can be added to storage without becoming part of the table until the commit succeeds, which is what makes writes atomic.

### Q33. How does Delta provide ACID guarantees on object storage?

- **Atomicity:** a write becomes visible only when its single commit file is successfully written to the log. If the job fails partway, the data files it wrote are orphaned and never referenced, so no partial result is visible.
- **Consistency:** schema enforcement and constraints stop invalid data from being committed.
- **Isolation:** readers see a consistent **snapshot** of a particular table version while writers work. Writers use **optimistic concurrency control**: each writer assumes it will not conflict, does its work, and at commit time checks whether anything conflicting was committed since it started. If so, it fails (or retries) with a concurrency exception.
- **Durability:** the log and data live in durable cloud storage.

### Q34. What are Delta isolation levels?

- **WriteSerializable** (default): ensures that the committed write operations are serializable, but reads may see a state that never existed as a single serial order of writes. It permits more concurrency and works for most workloads.
- **Serializable:** the strongest level; every read and write acts as if executed one after another. It results in more conflicts.

Set per table with `delta.isolationLevel`.

### Q35. What concurrency exceptions can occur and why?

Because of optimistic concurrency, conflicting commits raise exceptions such as `ConcurrentAppendException` (another writer added files that your operation read), `ConcurrentDeleteReadException` (another operation deleted a file you read), `ConcurrentDeleteDeleteException` (both deleted the same file), `MetadataChangedException` (schema or properties changed concurrently) and `ProtocolChangedException`.

How to reduce them: add **partition or clustering predicates** to `MERGE`/`UPDATE`/`DELETE` conditions so operations touch disjoint files; schedule maintenance (`OPTIMIZE`) away from heavy writers; and use features that enable **row-level concurrency** (deletion vectors with Liquid Clustering or Photon-enabled workloads) **(version-sensitive)**, which detect conflicts at the row level instead of the file level.

### Q36. What is time travel and what are its limits?

Because old files are retained for a while, you can query earlier table versions:

```sql
SELECT * FROM main.sales.orders VERSION AS OF 12;
SELECT * FROM main.sales.orders TIMESTAMP AS OF '2026-10-01';
DESCRIBE HISTORY main.sales.orders;
RESTORE TABLE main.sales.orders TO VERSION AS OF 12;
```

**Limits.** Two retention settings control it:
- `delta.logRetentionDuration` (default **30 days**): how long commit history is kept.
- `delta.deletedFileRetentionDuration` (default **7 days**): how long removed data files are kept before `VACUUM` is allowed to delete them.

After `VACUUM` deletes files, versions that depended on them can no longer be read. Time travel is for recovery, debugging and audit, not long-term archiving. `RESTORE` is itself a new commit; it does not erase history.

### Q37. What do OPTIMIZE, ZORDER and VACUUM do?

- **OPTIMIZE** performs **bin-packing**: it rewrites many small files into fewer larger ones (target about 1 GB per file by default **(version-sensitive; file size can be auto-tuned by table size)**). It is idempotent and does not break readers, because it is just another commit that adds new files and removes old ones. It fixes the **small-file problem**.
- **ZORDER BY (col1, col2)** is a multi-dimensional clustering technique applied during `OPTIMIZE`. It co-locates rows with similar values of those columns into the same files, so **data skipping** can skip far more files when you filter on them. It works best on a small number of high-cardinality columns that you filter on frequently. Effectiveness drops as you add more columns. You cannot Z-order by partition columns.
- **VACUUM** physically deletes data files that are no longer referenced by the current table version **and** are older than the retention threshold. It saves storage; it does **not** speed up queries. It never deletes `_delta_log`.

```sql
OPTIMIZE main.sales.orders ZORDER BY (customer_id);
VACUUM main.sales.orders DRY RUN;               -- preview what would be deleted
VACUUM main.sales.orders RETAIN 168 HOURS;      -- 7 days
```

Do not reduce retention below 7 days unless you are certain that no long-running reader, streaming job or time-travel need depends on older files. Databricks blocks very short retention by default (`spark.databricks.delta.retentionDurationCheck.enabled`).

### Q38. What is data skipping and how does it use statistics?

Each `add` action in the log records per-file statistics: number of records, and min, max and null count for columns. When you run a query with a filter such as `WHERE order_date = '2026-10-01'`, Delta compares the filter with each file's min/max and **skips files that cannot contain matches**, without reading them. This is why data layout (clustering, Z-order) matters: if values are scattered across all files, every file's min/max range is wide and nothing can be skipped.

By default, statistics are collected for the **first 32 columns** (`delta.dataSkippingNumIndexedCols`). Place frequently filtered columns early in the schema, or adjust the setting. Long string values are truncated in statistics.

### Q39. What is Liquid Clustering and why is it recommended over partitioning and Z-ORDER?

Liquid Clustering is Delta's modern data layout feature. You declare clustering keys when creating (or later altering) the table:

```sql
CREATE TABLE main.sales.orders (...) CLUSTER BY (customer_id, order_date);
ALTER TABLE main.sales.orders CLUSTER BY (region);   -- change keys without rewriting everything
OPTIMIZE main.sales.orders;                          -- applies clustering incrementally
```

Advantages:
- **Flexible:** you can change clustering keys later without rewriting the whole table.
- **Incremental:** `OPTIMIZE` only clusters data that is not yet clustered, instead of reprocessing everything like Z-ORDER.
- **No over-partitioning problems:** it avoids tiny partitions and the small-file issues caused by choosing a bad partition column.
- Works well with high-cardinality columns and skewed data.
- **Automatic liquid clustering** (`CLUSTER BY AUTO`) lets Databricks choose and evolve keys from query patterns **(version-sensitive)**.

Restrictions to remember: clustering is applied when `OPTIMIZE` runs (manually or through predictive optimization), a limited number of keys is allowed (up to 4), and liquid clustering cannot be combined with partitioning or `ZORDER` on the same table. Databricks recommends it as the default layout choice for new tables.

### Q40. When should I still use traditional partitioning?

Rarely. Partitioning physically splits data into one folder per distinct value (for example `order_date=2026-10-19/`). It helps when a table is very large (a common guideline is **1 TB or more**) and each partition holds at least about **1 GB**, and when you almost always filter on a **low-cardinality** column such as date or region. Partitioning on a high-cardinality column (`customer_id`) creates millions of tiny folders and files, which badly hurts performance. Partition columns also cannot be changed later without rewriting the table.

### Q41. What is the small-file problem?

Many tiny files cause heavy overhead: each file requires a metadata operation, a storage request (listing, open, read), and a task or part of one. A table with 100,000 files of 100 KB each is much slower than the same data in 100 files of 100 MB. Causes: frequent small streaming or micro-batch writes, over-partitioning, many small appends, and too many shuffle partitions at write time.

Fixes: `OPTIMIZE` regularly; enable **optimized writes** (`delta.autoOptimize.optimizeWrite`, which shuffles data before writing to produce better-sized files) and **auto compaction** (`delta.autoOptimize.autoCompact`, which compacts small files after writes, targeting around 128 MB); use Liquid Clustering; batch streaming writes using `availableNow` triggers or longer intervals. Managed tables with **predictive optimization** (Q60) handle much of this automatically.

### Q42. What is schema enforcement versus schema evolution?

- **Schema enforcement (default).** Delta rejects writes whose columns or data types do not match the table's schema. This protects data quality.
- **Schema evolution.** You explicitly allow the schema to change. For appends, use `.option("mergeSchema", "true")`; for `MERGE`, use `MERGE WITH SCHEMA EVOLUTION` or the relevant session setting; for overwrite changes, use `overwriteSchema`.

The principle: schema changes should be a deliberate choice rather than an accident. Delta also supports `ALTER TABLE ADD COLUMNS`, and with **column mapping** enabled you can **rename or drop columns** as metadata-only operations without rewriting data.

### Q43. What constraints and special column types does Delta support?

- `NOT NULL` constraints and `CHECK` constraints (for example `CHECK (amount >= 0)`): writes violating them fail.
- **Generated columns:** values computed from other columns (for example a date derived from a timestamp).
- **Identity columns:** auto-generated unique surrogate keys (`GENERATED ALWAYS AS IDENTITY`). Values are unique and increasing but **not guaranteed to be contiguous**.
- Primary key and foreign key constraints in Unity Catalog are **informational only** (not enforced), but help tools understand relationships and optimizers.

### Q44. How do upserts and deletes work? What is MERGE?

`MERGE INTO` matches source rows to target rows on a condition and performs insert, update or delete atomically in one transaction:

```sql
MERGE INTO main.sales.customers AS t
USING updates AS s
  ON t.customer_id = s.customer_id
WHEN MATCHED AND s.is_deleted = true THEN DELETE
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
WHEN NOT MATCHED BY SOURCE THEN DELETE;   -- optional: remove rows missing from source
```

How it works internally: Delta finds files in the target that contain matching rows (using data skipping where possible), rewrites those files with changes applied, and commits `add` and `remove` actions. This is why a `MERGE` can be expensive: updating one row may rewrite an entire large file (unless **deletion vectors** are enabled).

Performance tips: make the join condition selective and include partition/clustering predicates; ensure the source has unique keys (duplicate source matches cause an error because one target row would be matched multiple times); keep the source small or pre-deduplicated.

### Q45. What are deletion vectors?

Without them, deleting or updating a single row forces a rewrite of the whole Parquet file that contains it. With deletion vectors, Delta writes a small side file that **marks rows as deleted** instead of rewriting the data file. The actual rewrite is deferred to later compaction (`OPTIMIZE`) or `REORG TABLE ... APPLY (PURGE)`. This makes `DELETE`, `UPDATE` and `MERGE` much faster on large files, at the cost of a small amount of extra read-time work to apply the vectors. It is enabled by default for new tables in recent runtimes **(version-sensitive)**. Note: if you must physically purge deleted data (for example for privacy compliance), run the purge and then `VACUUM`.

### Q46. How do SCD Type 1 and Type 2 work in Delta?

- **SCD Type 1:** overwrite old values with new ones (no history). Implemented with `MERGE ... WHEN MATCHED THEN UPDATE`.
- **SCD Type 2:** keep history by closing the old row (`is_current = false`, set `end_date`) and inserting a new row with `start_date` and `is_current = true`. You can write this `MERGE` logic yourself, or in pipelines use the declarative **AUTO CDC** flow (previously `APPLY CHANGES INTO`) with `STORED AS SCD TYPE 2`, which handles ordering, late and out-of-order data, and history columns for you **(version-sensitive naming)**.

### Q47. What is Change Data Feed (CDF)?

When you set `delta.enableChangeDataFeed = true`, Delta records row-level changes with a `_change_type` column: `insert`, `update_preimage` (before), `update_postimage` (after) and `delete`, plus the commit version and timestamp. Downstream jobs read only what changed instead of rescanning the table:

```sql
SELECT * FROM table_changes('main.sales.orders', 5, 8);
```

```python
spark.readStream.option("readChangeFeed", "true").table("main.sales.orders")
```

Use cases: incremental propagation from Silver to Gold, auditing, and feeding external systems. CDF is only available for changes made after it is enabled.

### Q48. What is the difference between shallow clone, deep clone and copying a table?

- **Deep clone:** copies metadata **and** data files into a new location. Independent of the source.
- **Shallow clone:** copies only metadata; the clone references the source's data files. It is instant and costs almost no storage. Changes to the clone write new files; but if the source is vacuumed, the clone's references can break.
- Use clones for testing, backups and experimentation, and for creating dev copies of production data without full duplication.

### Q49. What are table properties, protocol versions and `DESCRIBE` commands?

Delta tables carry a **protocol version** (minimum reader and writer versions) that gates features such as deletion vectors, column mapping and CDF. Enabling an advanced feature may upgrade the protocol, which can make the table unreadable by older clients; some upgrades are not reversible easily. Useful commands:

```sql
DESCRIBE DETAIL main.sales.orders;     -- size, number of files, location, properties
DESCRIBE HISTORY main.sales.orders;    -- audit trail of operations
DESCRIBE EXTENDED main.sales.orders;   -- schema and metadata
SHOW TBLPROPERTIES main.sales.orders;
```

### Q50. What is UniForm?

Delta Universal Format (UniForm) lets a Delta table also expose metadata in **Apache Iceberg** (and Hudi) formats, so Iceberg-compatible engines can read it without copying data. Databricks also supports managed Iceberg tables and Iceberg REST catalog access in Unity Catalog **(version-sensitive)**.

### Q51. What are the best practices for writing to Delta?

- Prefer `saveAsTable` or `INSERT` into managed tables with a three-level name.
- Avoid overwriting entire tables when `MERGE` or partition-replace (`replaceWhere`) is enough. `replaceWhere` atomically replaces only rows matching a predicate.
- Use `mode("append")` for incremental loads; `mode("overwrite")` replaces everything and records it as a new version (old data stays available for time travel until vacuumed).
- Keep file sizes healthy using optimized writes, auto compaction and regular `OPTIMIZE`.
- Never directly modify or delete files inside a Delta table folder with file tools; always use SQL or Delta APIs so the log stays consistent.

---

# Part 5: Unity Catalog and Governance

### Q52. What is Unity Catalog?

Unity Catalog (UC) is Databricks' unified governance layer for data and AI assets. It provides centralized **access control**, **auditing**, **lineage**, **data discovery** and **sharing** across all workspaces attached to a metastore. It governs tables, views, volumes, functions, ML models, and access to external storage.

### Q53. What is the Unity Catalog object hierarchy?

**Metastore → Catalog → Schema → (Table, View, Materialized View, Volume, Function, Model)**

Tables are referenced with a **three-level namespace**: `catalog.schema.table`. A typical design uses a catalog per environment or domain (`dev`, `prod`, `finance`), and schemas per layer or subject (`bronze`, `silver`, `gold`). Other top-level securables live beside the catalogs: **storage credentials**, **external locations**, **connections** (for federation), **shares**, **recipients** and **clean rooms**.

### Q54. How does permission management work?

Permissions are granted with SQL `GRANT`/`REVOKE` (or the UI/API) to users, groups or service principals. Important privileges:

| Privilege | Meaning |
|---|---|
| `USE CATALOG` / `USE SCHEMA` | Required to access objects inside (a "door key") |
| `SELECT` | Read a table or view |
| `MODIFY` | Insert, update, delete on a table |
| `CREATE TABLE`, `CREATE SCHEMA`, etc. | Create objects |
| `EXECUTE` | Run a function or use a model |
| `READ VOLUME` / `WRITE VOLUME` | Access files in a volume |
| `MANAGE` | Manage grants and the object |
| `ALL PRIVILEGES` | Everything applicable |

**Inheritance:** privileges granted on a catalog or schema apply to child objects. To read `main.sales.orders` a user needs `USE CATALOG` on `main`, `USE SCHEMA` on `main.sales`, and `SELECT` on the table (or granted at a higher level). Every securable has an **owner** with full control. Best practice: grant to **groups**, not individuals.

```sql
GRANT USE CATALOG ON CATALOG main TO `analysts`;
GRANT USE SCHEMA ON SCHEMA main.sales TO `analysts`;
GRANT SELECT ON SCHEMA main.sales TO `analysts`;
```

### Q55. What are row filters and column masks?

- A **row filter** is a function attached to a table that decides which rows a given user can see (for example, a regional manager sees only their region).
- A **column mask** is a function that transforms a column's value for users who lack permission (for example, show only the last 4 digits of a card number or return `NULL`).

Both are enforced at query time inside the engine, so you manage one copy of data with different views per audience. **Attribute-based access control (ABAC)** using governed tags to apply policies at scale is also available **(version-sensitive)**. Dynamic views are an older alternative that achieves similar results using functions like `is_account_group_member()`.

### Q56. What are storage credentials and external locations?

- A **storage credential** wraps a cloud identity that can access storage (on Azure: a managed identity through an **Access Connector for Azure Databricks**; on AWS: an IAM role).
- An **external location** pairs a storage path (for example `abfss://container@account.dfs.core.windows.net/path`) with a storage credential, and then permissions on that location can be granted to users.

This lets Unity Catalog govern access to cloud storage without distributing storage keys to users.

### Q57. What is the difference between managed and external tables?

| | Managed table | External table |
|---|---|---|
| Storage location | Chosen and controlled by Unity Catalog (metastore, catalog or schema storage root) | You specify `LOCATION` |
| `DROP TABLE` | Removes metadata and (after a retention window) data | Removes metadata only; files remain |
| Lifecycle management | Full: predictive optimization, automatic maintenance | Limited |
| Recovery | `UNDROP TABLE` within about 7 days | Re-register the table at the same location |
| Best for | Default choice for new data | Data that other systems own or write, or existing data you must not own |

Managed tables are recommended because Databricks can optimize layout, collect statistics and clean up automatically.

### Q58. What are Volumes?

Volumes are Unity Catalog objects that govern access to **non-tabular files** (CSV, JSON, images, PDFs, model files, libraries). They are addressed as `/Volumes/<catalog>/<schema>/<volume>/<path>`. **Managed volumes** store files in UC-managed storage; **external volumes** point to an existing external location path. Volumes replace older patterns such as mounting storage (`dbutils.fs.mount`) or putting files in the DBFS root, because they bring permissions, lineage and auditing to files.

### Q59. What is data lineage and what are system tables?

- **Lineage:** Unity Catalog automatically captures how data flows between tables, views, notebooks, jobs, dashboards and models, down to **column level**. It helps with impact analysis ("what breaks if I change this column?") and root-cause tracing.
- **System tables:** read-only tables in the `system` catalog exposing operational data: audit logs (`system.access.audit`), billing usage (`system.billing.usage`), lineage, compute, query history and more. Use them for cost monitoring, security reviews and usage analytics.
- **Information schema:** every catalog has an `information_schema` for querying metadata about tables, columns, and privileges.

### Q60. What is predictive optimization?

A Unity Catalog feature that automatically runs maintenance operations such as `OPTIMIZE`, `VACUUM` and statistics collection (`ANALYZE`) on **managed tables** when its models predict a benefit. It removes the need to schedule maintenance jobs yourself and tends to cut storage cost and speed up reads. It is typically enabled at the account, catalog or schema level **(version-sensitive; check availability and defaults)**.

### Q61. What is Delta Sharing?

An open protocol for sharing live data securely **without copying it**. A **provider** creates a **share** (a set of tables, views, volumes or notebooks) and grants it to a **recipient**. Recipients can be on Databricks (Databricks-to-Databricks sharing, governed by UC) or use any tool that supports the open protocol (pandas, Spark, Power BI, Tableau and others). Access is controlled and auditable, and the provider can revoke access at any time. Cross-region or cross-cloud sharing may incur data egress costs.

### Q62. What is Lakehouse Federation?

It lets you **query external databases in place** (for example SQL Server, PostgreSQL, MySQL, Snowflake, BigQuery, Redshift) through Unity Catalog without ingesting the data. You create a **connection** and a **foreign catalog**, and the external tables appear under UC with governance and lineage. It is ideal for exploration and light queries; for heavy, repeated analytics, ingest the data into Delta instead.

### Q63. What is the legacy Hive metastore and how does it differ from Unity Catalog?

The Hive metastore is the older, workspace-local metadata store (`hive_metastore` catalog) with a two-level namespace (`schema.table`) and weaker governance (no centralized permissions across workspaces, limited lineage). Databricks strongly recommends migrating to Unity Catalog. Tools like `SYNC` and upgrade wizards help migrate tables.

### Q64. What are the cluster access modes in relation to Unity Catalog?

- **Standard** (previously "shared"): multiple users can share the cluster safely with UC enforcement and isolation between users.
- **Dedicated** (previously "single user"): assigned to one user or group; supports a wider set of workloads, including ML libraries and RDD APIs, with UC enforcement.
- Legacy "no isolation" modes do not support Unity Catalog governance.

Serverless compute and SQL warehouses are always UC-enabled.

---

# Part 6: Compute

### Q65. What are the types of compute in Databricks?

- **All-purpose clusters:** interactive, used for development, exploration and collaboration in notebooks. Can be shared and kept running. Higher DBU price.
- **Job clusters (job compute):** created automatically when a job run starts and terminated when it ends. Isolated and cheaper per DBU. **Recommended for production.**
- **SQL warehouses:** compute optimized for SQL and BI queries. Types: **Serverless**, **Pro** and **Classic**.
- **Serverless compute for notebooks and jobs:** no cluster setup; Databricks manages it; fast startup and autoscaling.
- **Instance pools:** pre-warmed idle VMs to reduce cluster start time (classic compute).

### Q66. What are the main cluster settings?

- **Databricks Runtime version** (choose LTS for production).
- **Node types** for driver and workers (general purpose, memory optimized, compute optimized, storage optimized, GPU).
- **Single node vs multi-node:** single node has only a driver (good for small data and ML on one machine).
- **Autoscaling:** minimum and maximum workers.
- **Auto-termination:** minutes of inactivity before shutdown.
- **Spot (Azure Spot / AWS Spot) instances** with on-demand fallback.
- **Access mode** and **Photon** toggle.
- **Spark configuration, environment variables, init scripts, libraries, tags.**
- **Cluster policies** to enforce limits and defaults.

### Q67. What is autoscaling and when does it help or not?

Autoscaling adds workers when there are pending tasks and removes them when the cluster is idle, between your min and max. It helps with variable workloads and interactive use. It helps less for short jobs (scale-up takes time), for strongly skewed workloads (adding nodes does not help a single hot partition), and for streaming where steady state is predictable. Enhanced autoscaling in pipelines is more intelligent for streaming workloads.

### Q68. What are cluster policies?

Policies are rules that restrict or pre-configure how clusters can be created: allowed node types, maximum size, mandatory auto-termination, required tags (for cost allocation), fixed runtime versions. They are the main governance and cost-control tool for compute. Administrators grant users the ability to create only policy-compliant clusters.

### Q69. What is Photon?

Photon is Databricks' native **vectorized query engine written in C++**. It accelerates Spark SQL and DataFrame operations (scans, joins, aggregations, Delta reads and writes) without any code change, and is compatible with Spark APIs. It is on by default on SQL warehouses and serverless. It does **not** accelerate Python UDFs, RDD code or workloads dominated by non-SQL work. Photon carries a higher DBU rate, so it pays off when it reduces runtime enough (typically on large scans, joins and aggregations).

### Q70. How does Databricks billing work?

You pay two things for classic compute:
1. **DBUs (Databricks Units):** a unit of processing capability per hour. Price per DBU depends on the **workload type (SKU)** (jobs compute, all-purpose compute, SQL, pipelines, serverless, model serving) and the tier. Billed per second.
2. **Cloud infrastructure:** VMs, disks, networking, and storage billed directly by your cloud provider.

For **serverless**, both are bundled into a single DBU-based price, and Databricks manages the infrastructure. Job compute is substantially cheaper per DBU than all-purpose compute, so moving scheduled workloads off all-purpose clusters is one of the easiest savings. Use **tags** and the `system.billing.usage` table to attribute and monitor spend.

### Q71. What are the best practices for cost control?

- Set **auto-termination** on all-purpose clusters (for example 20 to 30 minutes).
- Run production on **job compute** or serverless, not all-purpose clusters.
- Use **spot instances** for fault-tolerant workers; keep the driver on-demand.
- Enforce **cluster policies** and **tags**.
- Right-size using Spark UI metrics (CPU utilization, memory, spill). Bigger clusters do not fix skew or inefficient code.
- Use Photon where it pays off; use `availableNow` streaming instead of 24/7 clusters for non-real-time needs.
- Compact files and prune data so queries scan less.
- Monitor with budgets, system tables and dashboards.

### Q72. How do libraries and init scripts work?

- **Libraries** (PyPI, Maven, wheel/JAR files) can be installed at cluster level, job level, or notebook level with `%pip install`. Notebook-scoped libraries affect only that session. Store wheels and JARs in Unity Catalog volumes.
- **Init scripts** are shell scripts that run on each node at startup (installing system packages, configuring agents). They are supported in volumes and workspace files; legacy DBFS-based init scripts are deprecated **(version-sensitive)**. Overusing them slows startup and complicates reproducibility.

---

# Part 7: Ingestion, Streaming and the Medallion Architecture

### Q73. What is the Medallion Architecture?

A layered data design that improves quality step by step:
- **Bronze:** raw data exactly as received, append-only, with ingestion metadata (source file, load time). It preserves history so you can **reprocess** later if logic changes.
- **Silver:** cleaned, validated, deduplicated, type-cast and conformed data; joins across sources; business keys resolved. Often the most reusable layer.
- **Gold:** aggregated, business-ready tables (metrics, dimensional models, feature tables) optimized for BI, reporting and ML.

Why it works: each layer has one clear purpose, errors can be traced to a layer, and Silver/Gold can always be rebuilt from Bronze. It is a guideline, not a law; some designs add layers or skip one.

### Q74. What is Auto Loader and why use it instead of `spark.read`?

Auto Loader (source name `cloudFiles`) **incrementally** ingests new files arriving in cloud storage. It remembers which files it has processed in its **checkpoint**, so each file is processed exactly once even across restarts.

```python
(spark.readStream.format("cloudFiles")
  .option("cloudFiles.format", "json")
  .option("cloudFiles.schemaLocation", "/Volumes/main/bronze/_schemas/events")
  .option("cloudFiles.inferColumnTypes", "true")
  .load("/Volumes/main/landing/events/")
  .writeStream
  .option("checkpointLocation", "/Volumes/main/bronze/_checkpoints/events")
  .trigger(availableNow=True)
  .toTable("main.bronze.events"))
```

Benefits:
- **Scales to millions of files**, with two discovery modes: **directory listing** (default, uses incremental listing) and **file notification** (uses cloud events and queues, such as Azure Event Grid and Queue Storage).
- **Schema inference and evolution**: stores inferred schema in `schemaLocation`. For JSON/CSV it infers all columns as strings unless `inferColumnTypes` is true. `cloudFiles.schemaEvolutionMode` controls behavior: `addNewColumns` (default; the stream fails once, then picks up new columns on restart), `rescue`, `failOnNewColumns` and `none`.
- **Rescued data column** (`_rescued_data`) captures fields that do not match the schema instead of losing them.
- Exactly-once processing and fault tolerance.

### Q75. What is the difference between Auto Loader and COPY INTO?

Both are idempotent (do not reload files already loaded). `COPY INTO` is a SQL command suited to **thousands** of files and simpler, scheduled batch loads. Auto Loader scales to **millions** of files, supports streaming, richer schema evolution and file notification mode, and is generally recommended for new ingestion. Managed ingestion connectors in **Lakeflow Connect** also exist for databases and SaaS sources **(version-sensitive)**.

### Q76. How does Structured Streaming work?

Structured Streaming treats a live stream as an **unbounded table** to which new rows keep being appended. You write a query as if on a static table, and Spark runs it incrementally in **micro-batches**.

Key concepts:
- **Sources:** Kafka, Event Hubs, Auto Loader (files), Delta tables.
- **Sinks:** Delta tables (most common), Kafka, `foreachBatch` (custom logic per micro-batch such as `MERGE`).
- **Checkpoint:** stores offsets (what has been read), commits (what has been written) and state. It enables fault tolerance and exactly-once results with replayable sources and idempotent sinks such as Delta. **Never share a checkpoint between two streams, and do not delete it unless you intend to reprocess from scratch.**
- **Output modes:** `append` (only new rows), `update` (changed rows), `complete` (entire result table, for aggregations).
- **Watermarks:** tell Spark how late data may arrive, so it can finalize windows and drop old state. Essential for stateful aggregations and stream-stream joins to prevent unbounded state growth.

### Q77. What do the streaming triggers mean?

- **Default (unspecified) / `processingTime="10 seconds"`:** runs micro-batches continuously or at a fixed interval.
- **`availableNow=True`:** processes all currently available data (in multiple batches if large) then stops. This is the best option for **cost-effective scheduled incremental batch**, because the cluster only runs while there is work.
- **`once=True`:** older single-batch version; use `availableNow` instead.
- **Continuous / real-time modes** exist for ultra-low-latency use cases **(version-sensitive)**.

Useful Delta source options: `maxFilesPerTrigger` and `maxBytesPerTrigger` limit batch size; `skipChangeCommits` and `ignoreDeletes` handle upstream updates or deletes that would otherwise fail the stream.

### Q78. How do you do upserts from a stream?

Use `foreachBatch` to run a `MERGE` for each micro-batch:

```python
def upsert(batch_df, batch_id):
    batch_df.createOrReplaceTempView("updates")
    batch_df.sparkSession.sql("""
        MERGE INTO main.silver.customers t
        USING updates s ON t.id = s.id
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *""")

(spark.readStream.table("main.bronze.customers")
   .writeStream.foreachBatch(upsert)
   .option("checkpointLocation", "/Volumes/main/silver/_chk/customers")
   .start())
```

Ensure each micro-batch is deduplicated by key before the merge.

### Q79. How should I think about batch versus streaming?

Streaming on Databricks is not only for real-time. With `availableNow`, the same streaming code becomes an **incremental batch** that processes only new data and keeps state in checkpoints, which is usually cheaper and simpler than re-running full batch loads. Choose continuous streaming only when latency requirements (seconds to minutes) demand it.

---

# Part 8: Lakeflow Declarative Pipelines (Formerly Delta Live Tables)

**Naming note (version-sensitive).** This feature was originally called **Delta Live Tables (DLT)** and has since been renamed **Lakeflow Declarative Pipelines / Lakeflow Spark Declarative Pipelines**, with the core engine contributed to Apache Spark as *Spark Declarative Pipelines*. APIs may appear as `import dlt` (older) or `from pyspark import pipelines as dp` (newer). The concepts below are the same.

### Q80. What are declarative pipelines and why use them?

Instead of writing the orchestration steps, you declare **what each table should contain** with SQL or Python, and the framework figures out dependency order, incremental processing, checkpoints, retries, scaling and cleanup. Benefits: less boilerplate, automatic dependency graph, built-in data quality rules, built-in monitoring (event log), and simplified CDC and SCD handling.

### Q81. What are streaming tables and materialized views in a pipeline?

- **Streaming table:** a Delta table that is updated **incrementally** from an append-only streaming source; each row is processed once. Ideal for ingestion (Bronze) and append-style transformations.
- **Materialized view:** a table defined by a query whose results are **kept up to date** by the system, incrementally refreshed where possible. Ideal for Silver/Gold aggregations and joins.
- **Temporary views** are private intermediate steps within the pipeline.

### Q82. What are expectations?

Expectations are declarative **data-quality rules** attached to a table, with three behaviors on violation:
- **warn** (`expect`): keep the row, record the metric.
- **drop** (`expect_or_drop`): remove invalid rows.
- **fail** (`expect_or_fail`): stop the update.

```python
import dlt

@dlt.table
@dlt.expect("has_id", "order_id IS NOT NULL")
@dlt.expect_or_drop("valid_amount", "amount > 0")
@dlt.expect_or_fail("valid_date", "order_date <= current_date()")
def silver_orders():
    return dlt.read_stream("bronze_orders")
```

Metrics on passed and failed records are recorded in the pipeline **event log**, which you can query for data-quality dashboards.

### Q83. How does a pipeline handle CDC?

The AUTO CDC flow (formerly `APPLY CHANGES INTO`) takes a change feed (inserts, updates, deletes, with a sequencing column) and maintains a target table, handling out-of-order events and optionally maintaining SCD Type 1 or Type 2 history.

### Q84. What are pipeline update modes and operational concepts?

- **Triggered** pipelines run to completion and stop (good for scheduled, cost-efficient processing); **continuous** pipelines run constantly for low latency.
- **Development vs production mode:** development keeps clusters alive and does not retry aggressively for faster iteration.
- **Full refresh** rebuilds a table from scratch; use carefully with streaming tables because the source history may no longer be available.
- The **event log** records lineage, data-quality results and performance metrics.

---

# Part 9: Orchestration, CI/CD and Deployment

### Q85. What are Jobs (Lakeflow Jobs / Workflows)?

Jobs schedule and orchestrate multi-step workloads. A job has **tasks** (notebook, SQL, Python script, Python wheel, JAR, dbt, pipeline, other job, if/else condition, for-each loop) arranged as a **DAG** with dependencies. Features: schedules (cron) and triggers (file arrival, table update, continuous), parameters, retries, timeouts, concurrency limits, notifications and alerts, **task values** to pass small results between tasks, and **repair run** to rerun only failed tasks and their dependents.

### Q86. How do I pass parameters to notebooks and between tasks?

- Notebook **widgets** (`dbutils.widgets.text("env", "dev")`; read with `dbutils.widgets.get("env")`) receive job parameters.
- **Job parameters** apply across tasks; **task parameters** are task-specific; dynamic value references such as `{{job.run_id}}` can be used.
- **Task values** (`dbutils.jobs.taskValues.set/get`) pass small values from one task to later tasks.

### Q87. What are Databricks Asset Bundles?

Asset Bundles are the recommended **infrastructure-as-code** approach: you define jobs, pipelines, clusters, dashboards and other resources in YAML files in a Git repo (`databricks.yml`), then validate and deploy them to dev, staging and prod targets using the Databricks CLI:

```bash
databricks bundle validate
databricks bundle deploy -t dev
databricks bundle run my_job -t dev
```

Benefits: repeatable deployments, code review, environment parity, and CI/CD integration with GitHub Actions, Azure DevOps or Jenkins.

### Q88. How should dev, test and prod be structured?

Common approaches: separate **workspaces** per environment, separate **catalogs** per environment (`dev`, `test`, `prod`), or both. Code lives in Git; deployment is automated through CI/CD; production jobs run as **service principals** with write permission to prod data, while developers have read-only prod access. Use parameters (catalog name, paths) so the same code runs in every environment.

### Q89. What are Git folders (Repos)?

Git folders connect a workspace to GitHub, Azure DevOps, GitLab or Bitbucket so you can clone, branch, commit, pull and push from the UI. Notebooks and files are versioned as normal source files. Best practice: develop on feature branches in personal Git folders; production runs from a deployed bundle or a controlled branch, not someone's personal copy.

### Q90. What other interfaces exist for automation?

The **Databricks CLI**, the **REST API**, SDKs (Python, Java, Go), the **Terraform provider**, and `dbutils` utilities inside notebooks (`dbutils.fs`, `dbutils.secrets`, `dbutils.widgets`, `dbutils.notebook.run`, `dbutils.jobs`).

---

# Part 10: Databricks SQL, BI and Analytics

### Q91. What is Databricks SQL and what are SQL warehouses?

Databricks SQL is the analytics experience for SQL users: SQL editor, query history, dashboards, alerts and BI connectivity. A **SQL warehouse** is the compute behind it. Sizes follow T-shirt sizing (2X-Small up to very large). **Serverless** warehouses start in seconds and scale automatically with Databricks managing the infrastructure and have intelligent workload management. **Pro** adds extra features over Classic. Multi-cluster scaling (min and max clusters) handles many concurrent users: it scales **out** for concurrency, whereas a bigger size scales **up** for individual query speed.

### Q92. How do I choose warehouse size?

Larger sizes make a single large query faster (more parallelism) but cost more per hour. More clusters handle more concurrent queries. Start small, observe the **query profile** and queue times, and scale up for slow heavy queries or scale out for queuing under concurrency. Short auto-stop (a few minutes) keeps cost down; serverless makes this cheap because restart is fast.

### Q93. What caching exists in Databricks?

1. **Spark cache** (`cache()`/`persist()`): user-controlled, in memory (and disk), per application.
2. **Disk cache** (formerly Delta cache): automatically caches remote Parquet/Delta file data on **local NVMe SSDs** of workers, so repeated reads avoid cloud storage latency. Works on specific instance types and is enabled by default on many.
3. **Query result cache** (Databricks SQL): returns stored results when the same query runs again on unchanged data.

### Q94. What are AI/BI dashboards and Genie?

**AI/BI dashboards** are Databricks-native dashboards built on SQL datasets. **Genie** lets business users ask questions in natural language over curated data ("Genie spaces"); results depend on good table and column descriptions and curated instructions **(version-sensitive)**. Good metadata (comments, tags) in Unity Catalog improves both discovery and AI-generated answers.

### Q95. How does Power BI connect to Databricks?

Through the built-in Databricks connector (using a SQL warehouse via server hostname and HTTP path) in **Import** or **DirectQuery** mode, authenticated with Microsoft Entra ID (OAuth), personal access tokens or service principals. Best practices: point Power BI at **Gold** tables or views, use a serverless/Pro SQL warehouse, push filters and aggregations down to the warehouse, keep Gold tables optimized (clustering, compaction), and use Import with scheduled refresh for slower-changing data.

### Q96. What views exist and when to use each?

- **Standard view:** a saved query; no stored data; recomputed on each read.
- **Temporary view:** exists only in the session.
- **Materialized view:** stores results and refreshes them; faster reads, extra storage and refresh cost.
- **Metric views** define reusable business metrics and dimensions semantically **(version-sensitive)**.

---

# Part 11: Performance Tuning Playbook

### Q97. What is a systematic approach to performance tuning?

Work from observation to cause to fix; avoid guessing.
1. **Measure:** find the slow stage in the Spark UI or the slow operator in the query profile.
2. **Classify:** is it I/O (scanning too much), shuffle (too much data moved), skew (one slow task), spill (memory pressure), or compute (UDFs, CPU bound)?
3. **Fix the cause:** reduce data scanned, reduce shuffle, fix skew, resize partitions or memory.
4. **Re-measure.**

### Q98. How do I reduce the amount of data scanned?

- Use **columnar formats** and select only needed columns (column pruning).
- Filter early and on **clustering/partition columns** so data skipping works.
- Use **Liquid Clustering** (or Z-ORDER) on frequently filtered columns.
- Keep statistics fresh and put filter columns within the first 32 indexed columns.
- Compact small files; avoid `SELECT *` on wide tables.
- Pre-aggregate in Silver/Gold so BI queries scan summaries, not raw data.

### Q99. How do I make a join faster?

1. **Broadcast** the small table (`/*+ BROADCAST(dim) */`).
2. **Filter and project before** the join.
3. Check for **skew** and use AQE skew handling, salting, or null-key filtering.
4. Ensure join keys have **matching data types** (a type mismatch can prevent efficient planning or cause implicit casts).
5. Avoid cross joins and non-equi joins on large data.
6. Make sure statistics exist so the optimizer picks a good strategy (`ANALYZE TABLE ... COMPUTE STATISTICS`, or rely on predictive optimization).

### Q100. How do I tune shuffle partitions?

With AQE enabled, partition coalescing handles most cases. If a stage has too many tiny tasks, reduce `spark.sql.shuffle.partitions`; if tasks are huge and spill, increase it. Target about 100 to 200 MB per shuffle partition. Setting `spark.sql.shuffle.partitions` to `auto` on Databricks lets the platform choose **(version-sensitive)**.

### Q101. What is a quick list of anti-patterns?

- `collect()` or `toPandas()` on large data.
- Python UDFs where built-ins exist.
- Row-by-row loops over a DataFrame in Python.
- Partitioning small tables or partitioning on high-cardinality columns.
- `SELECT *` into wide joins.
- Running production on all-purpose clusters.
- Many tiny files and never running `OPTIMIZE`.
- Caching everything.
- Using `FLOAT`/`DOUBLE` for money, or `STRING` for dates and numbers.
- Overwriting whole tables for small changes.
- Sharing checkpoint locations.

### Q102. A job that ran fine yesterday is slow today. How do I diagnose it?

1. Open the **Spark UI** (or query profile) and find the slowest stage or operator.
2. Compare to a previous run: did **input volume** change? Did small files accumulate?
3. Check **skew** (max vs median task time) and **spill**.
4. Look at the **query plan** (`EXPLAIN`): did the join strategy change (a broadcast no longer applying because a table grew)?
5. Check cluster health: lost spot nodes, autoscaling delays, a different runtime version, or contention with other workloads.
6. Check table health: file counts via `DESCRIBE DETAIL`, last `OPTIMIZE` date.
Only then decide on a fix (compaction, clustering change, resizing, code change).

---

# Part 12: Security, Networking and Operations

### Q103. How do I manage secrets?

Use **secret scopes**: either Databricks-backed or backed by **Azure Key Vault**. Read secrets with `dbutils.secrets.get(scope="kv", key="db-password")`. Values are redacted if printed in notebook output. Never hardcode credentials in notebooks or commit them to Git. Prefer identity-based access (managed identities, service principals, OAuth) over long-lived keys wherever possible.

### Q104. What are service principals and why use them for jobs?

A service principal is a non-human identity for automation. Running production jobs as a service principal rather than as an individual means jobs do not break when someone leaves or changes roles, permissions are scoped and auditable, and personal credentials are not embedded in pipelines.

### Q105. How are users and groups managed?

Identity is managed at the **account level** and synced from the identity provider (Microsoft Entra ID on Azure) using SCIM provisioning and SSO. Permissions are assigned to groups. Roles to know: **account admin**, **metastore admin**, **workspace admin**, and object owners.

### Q106. What is the difference between DBFS and Unity Catalog Volumes?

DBFS (Databricks File System) is the older abstraction over workspace storage. The **DBFS root is accessible by all users in the workspace**, which makes it a poor fit for governed data, and DBFS mounts bypass Unity Catalog governance. Volumes are the governed replacement: permissions, lineage and auditing for files. Use Volumes for new work and avoid storing production data in DBFS root.

### Q107. How is network security handled?

Typical enterprise controls: **VNet injection** (deploy the classic compute plane into your own virtual network), **secure cluster connectivity** (no public IPs on cluster nodes), **private link / private endpoints** for front-end (user access) and back-end (compute-to-control plane) traffic, **IP access lists**, firewall rules or network security groups, and serverless **network policies** to control egress. Data at rest is encrypted by default; **customer-managed keys** are available for managed services and storage.

### Q108. What auditing and monitoring is available?

Audit logs (in `system.access.audit` and delivered to storage), lineage, billing usage tables, job run history and alerts, cluster event logs, Spark UI, query history, and pipeline event logs. Use SQL alerts and job notifications for operational monitoring.

### Q109. What is a Databricks personal access token (PAT) and what is better?

A PAT is a long-lived token for API/CLI access tied to a user. Because they are long-lived and personal, prefer **OAuth** (user-to-machine or machine-to-machine with service principals) and short-lived credentials for automation.

### Q110. What are clean rooms?

Clean rooms provide a secure environment for multiple parties to collaborate on combined data analysis without exposing each party's raw data to the others, built on Delta Sharing and serverless compute **(version-sensitive)**.

---

# Part 13: Azure Databricks Specifics

### Q111. How does Azure Databricks integrate with Azure services?

- **Azure Data Lake Storage Gen2 (ADLS):** the primary storage, accessed through `abfss://container@account.dfs.core.windows.net/path`. ADLS Gen2 uses a **hierarchical namespace**, which makes directory operations fast and atomic; this matters for Delta performance and commits.
- **Microsoft Entra ID:** single sign-on, SCIM provisioning, service principals and managed identities.
- **Azure Key Vault:** backs secret scopes.
- **Azure Data Factory / Synapse pipelines:** can run Databricks notebooks, jobs and Delta operations as pipeline activities.
- **Event Hubs:** Kafka-compatible streaming source (use the Kafka connector).
- **Power BI:** native connector.
- **Azure Monitor / Log Analytics:** diagnostic logs and monitoring.
- **Azure Purview / Microsoft Purview:** governance integration for cataloging.

### Q112. How does Unity Catalog access Azure storage securely?

Create an **Access Connector for Azure Databricks** (a managed identity resource), grant it the **Storage Blob Data Contributor** role on the storage account, create a **storage credential** in Unity Catalog that references the connector, then create **external locations** and managed storage roots for the metastore, catalogs or schemas. Users never see or need storage keys.

### Q113. What are the Azure-specific compute and billing notes?

VMs are billed by Azure; DBUs by Databricks (visible in Azure cost management). Azure pricing tiers affect features: Unity Catalog and fine-grained access control are Premium-tier features (Premium is the current default) **(version-sensitive)**. Azure Spot VMs can be used for workers. The **managed resource group** (created automatically with a workspace) holds the cluster VMs, disks, and networking resources.

### Q114. What are the common ways to authenticate to storage from Databricks?

From most to least recommended: (1) **Unity Catalog storage credentials and external locations** (managed identity through Access Connector); (2) service principal with OAuth secrets stored in Key Vault-backed secret scopes (legacy approach); (3) SAS tokens or account keys (avoid; keys are broad and long-lived). Avoid mounting storage with `dbutils.fs.mount` for new work.

---

# Part 14: Machine Learning and AI on Databricks (Overview)

### Q115. What is MLflow?

MLflow is an open-source platform for the ML lifecycle, integrated into Databricks. Components: **Tracking** (log parameters, metrics, artifacts per run), **Models** (a standard packaging format), **Model Registry** (versioned models with aliases, governed in Unity Catalog), and evaluation and tracing tools for LLM and agent applications. Databricks notebooks can autolog experiments automatically.

### Q116. What other ML/AI capabilities exist?

- **Feature engineering in Unity Catalog:** governed feature tables and online feature serving that avoid training-serving skew.
- **Model Serving:** managed real-time endpoints for custom models, foundation models and external model providers.
- **AutoML:** automated baseline models with generated notebooks.
- **Vector Search:** managed vector database for retrieval-augmented generation (RAG).
- **AI Functions** such as `ai_query` and task-specific functions that call LLMs directly from SQL.
- **Agent frameworks and evaluation** for building and testing AI agents.
- **Distributed training** support and GPU clusters.

Product names in this area change quickly **(version-sensitive)**; verify current names in the documentation.

---

# Part 15: Troubleshooting: Common Errors and Their Causes

### Q117. "Table or view not found" or permission errors in Unity Catalog. What do I check?

Verify the three-level name and the current catalog/schema (`USE CATALOG`, `USE SCHEMA`); confirm the user has `USE CATALOG`, `USE SCHEMA` and `SELECT`; confirm the cluster access mode supports Unity Catalog; check the workspace is attached to the right metastore and that the catalog is bound to the workspace.

### Q118. "AnalysisException: A schema mismatch detected when writing to the Delta table."

Schema enforcement caught a difference between the incoming data and the table. Decide whether the change is intended; if so use `mergeSchema` (or `MERGE WITH SCHEMA EVOLUTION`), otherwise fix the source types. Beware of columns arriving as strings from CSV/JSON.

### Q119. "ConcurrentAppendException" or similar conflicts.

Two operations touched overlapping data. Add partition/clustering predicates so operations are disjoint, serialize conflicting jobs, retry on failure, or use row-level concurrency features.

### Q120. "FileNotFoundException ... underlying files have been updated" or missing file after VACUUM.

A reader or stream referenced files that were removed (by `OPTIMIZE` then `VACUUM`, or by external modification). Causes: retention too short, long-running queries, or streaming checkpoints that lag far behind. Increase retention, avoid manual file deletion, and for caches run `REFRESH TABLE`.

### Q121. Out of memory errors.

Driver OOM: remove `collect`/`toPandas` on large data, avoid huge broadcast variables. Executor OOM: fix skew, increase partitions, use more memory per core (memory-optimized nodes), reduce broadcast size, avoid exploding huge arrays.

### Q122. "Multiple sources found for ..." or "Cannot resolve column".

Usually a naming ambiguity after a join (duplicate column names) or a typo. Alias DataFrames and select explicit columns.

### Q123. Streaming job fails after changing logic.

Checkpoint state is tied to query structure. Some changes (like altering stateful aggregation or schema of state) are incompatible with an existing checkpoint; start with a new checkpoint (and backfill carefully) rather than deleting blindly. Auto Loader schema evolution intentionally fails once when new columns appear; restarting picks them up.

### Q124. Merge fails with "multiple source rows matched".

The source has duplicate keys. Deduplicate the source (for example by latest timestamp using a window function) before the merge.

### Q125. Job is slow only on the last few tasks.

Classic skew. See Q20 and Q99.

---

# Part 16: Scenario-Based Interview Questions

### Q126. Design a pipeline that ingests daily files from a vendor into a reporting layer.

Land files in a Unity Catalog **volume** or external location. Use **Auto Loader** with `availableNow` into a **Bronze** streaming table (keep raw data plus `_metadata.file_path` and load time; use rescued data column). Use a declarative pipeline or job to build **Silver** (type casting, deduplication, validation with expectations; `MERGE` or AUTO CDC for updates). Build **Gold** aggregates as materialized views or tables with Liquid Clustering. Run as a **job** under a **service principal** on job compute, with alerts. Secure with Unity Catalog grants; monitor with the event log and system tables. Power BI reads Gold through a SQL warehouse.

### Q127. A table has 500,000 tiny files and queries are slow. What do you do?

Run `OPTIMIZE` (and compare `DESCRIBE DETAIL` before and after). Find the cause of small files (streaming micro-batches, over-partitioning) and fix it: enable optimized writes and auto compaction, change to Liquid Clustering, or use `availableNow` triggers. Enable predictive optimization for managed tables. Schedule `VACUUM` after `OPTIMIZE` (respecting retention) to reclaim storage.

### Q128. How do you recover accidentally deleted data?

Use `DESCRIBE HISTORY` to find the version before the bad operation, then `RESTORE TABLE ... TO VERSION AS OF n` (works only if the files are still within retention). For a dropped managed Unity Catalog table, use `UNDROP TABLE` within the recovery window. Prevent recurrence with tighter permissions and use of clones for risky changes.

### Q129. How do you implement SCD Type 2 for a customer dimension?

Use an AUTO CDC flow with `STORED AS SCD TYPE 2` and a sequencing column, or write a `MERGE` that: (1) detects changed rows by comparing hash of tracked columns, (2) updates the current row setting `is_current = false` and `end_date`, and (3) inserts the new version with `start_date` and `is_current = true`. The usual trick is to union the "new version" rows with a copy keyed to match the closing action so a single `MERGE` does both.

### Q130. How do you secure sensitive columns like national ID or email?

Apply **column masks** so only authorized groups see real values, use **row filters** for regional restrictions, tag columns with governed tags (PII), restrict `SELECT` to groups, audit access through `system.access.audit`, and keep raw PII in restricted Bronze/Silver schemas while exposing masked views in Gold.

### Q131. When would you choose a broadcast join versus letting Spark decide?

Let AQE and the optimizer decide by default. Force a broadcast hint when you know one side is small and statistics are missing or misleading (for example after heavy filtering that the optimizer cannot predict), and the table comfortably fits in executor memory.

### Q132. Explain how you would reduce the cost of a nightly job.

Move it to job compute or serverless; use spot workers; right-size using metrics; use `availableNow` incremental logic so only new data is processed; compact files and use clustering so scans are smaller; consider Photon if the job is SQL/join heavy; schedule to avoid keeping clusters idle; add tags and monitor `system.billing.usage`.

### Q133. Explain the difference between `DELETE`, `TRUNCATE`, `DROP` and `VACUUM` in Delta.

`DELETE` removes rows matching a predicate and records a new version (data still recoverable by time travel). `TRUNCATE` removes all rows quickly. `DROP TABLE` removes the table (managed: data removal after retention; external: only metadata). `VACUUM` physically deletes unreferenced old files and ends time travel into them.

### Q134. How do you handle late-arriving or duplicate events in streaming?

Use watermarks to bound lateness; deduplicate with `dropDuplicatesWithinWatermark` or by key within `foreachBatch`; use event-time columns rather than arrival time; use AUTO CDC sequencing to resolve out-of-order updates.

### Q135. Which Databricks features would you use to give analysts safe, fast access?

Unity Catalog groups and grants on Gold schemas, row filters/column masks where needed, a serverless SQL warehouse, optimized Gold tables (clustering, statistics), certified tags and comments for discovery, AI/BI dashboards or Power BI connected to the warehouse, and cost control through warehouse auto-stop and query limits.

---

# Part 17: Glossary

- **ACID:** Atomicity, Consistency, Isolation, Durability; the guarantees of reliable transactions.
- **Access Connector (Azure):** managed identity resource used by Unity Catalog to reach storage.
- **AQE:** Adaptive Query Execution; runtime re-optimization of Spark plans.
- **Auto Loader:** incremental file ingestion source (`cloudFiles`).
- **Bin-packing:** combining small files into larger ones (OPTIMIZE).
- **Bronze/Silver/Gold:** raw, cleaned, business-ready layers of the Medallion Architecture.
- **Catalyst:** Spark's query optimizer.
- **CDF:** Change Data Feed; row-level change tracking in Delta.
- **Checkpoint (Delta log):** Parquet snapshot of table state to speed log replay.
- **Checkpoint (streaming):** stored progress and state of a streaming query.
- **Control plane / compute plane:** Databricks-managed services versus where data is processed.
- **DBU:** Databricks Unit; billing unit of processing per hour.
- **Deletion vector:** side file marking rows as deleted without rewriting data files.
- **Delta Lake:** open table format (Parquet plus transaction log).
- **Delta Sharing:** open protocol for sharing live data.
- **Driver / Executor:** coordinator and worker processes in Spark.
- **External location:** UC object tying a storage path to a storage credential.
- **Lakehouse:** architecture combining lake storage with warehouse capabilities.
- **Lazy evaluation:** transformations only execute when an action runs.
- **Liquid Clustering:** flexible incremental data layout feature replacing partitioning and Z-ORDER.
- **Managed table:** table whose storage is controlled by Unity Catalog.
- **Materialized view:** query result stored and refreshed by the system.
- **Medallion Architecture:** layered data quality design.
- **MERGE:** atomic upsert/delete command.
- **Metastore:** top-level Unity Catalog container for metadata.
- **MLflow:** open-source ML lifecycle platform.
- **Narrow / wide transformation:** without / with data shuffle.
- **Optimistic concurrency:** assume no conflict, verify at commit.
- **OPTIMIZE:** file compaction (and clustering) command.
- **Parquet:** open columnar file format.
- **Photon:** native vectorized query engine.
- **Predicate pushdown:** applying filters at the storage-reading level.
- **Predictive optimization:** automatic maintenance of managed tables.
- **Row filter / column mask:** fine-grained access controls in Unity Catalog.
- **Service principal:** non-human identity for automation.
- **Shuffle:** redistribution of data across partitions and nodes.
- **Skew:** uneven distribution of data across partitions.
- **Spill:** writing intermediate data to disk when memory is insufficient.
- **Storage credential:** UC object representing a cloud identity for storage access.
- **Structured Streaming:** Spark's incremental streaming engine.
- **Time travel:** querying earlier versions of a Delta table.
- **Unity Catalog:** centralized governance layer.
- **VACUUM:** deletes unreferenced old data files.
- **Volume:** UC object governing non-tabular files.
- **Watermark:** bound on lateness of streaming event-time data.
- **Z-ORDER:** multi-dimensional data clustering within OPTIMIZE.

---

# Part 18: One-Page Cheat Sheet

**Architecture:** Delta = Parquet files + `_delta_log`. Unity Catalog = governance (`catalog.schema.table`). Control plane = Databricks-managed; compute plane = where data is processed.

**Delta maintenance:** `OPTIMIZE` compacts; `VACUUM` deletes old files (default 7 days); time travel depends on retention (log 30 days, files 7 days); new tables: prefer Liquid Clustering over partitioning and Z-ORDER; partition only huge tables (1 TB or more, 1 GB or more per partition) on low-cardinality columns.

**Ingestion:** Auto Loader for files (`cloudFiles`, `availableNow`); Medallion = Bronze raw, Silver clean, Gold business-ready; declarative pipelines for quality rules (expect, drop, fail).

**Compute and cost:** job compute for production; auto-terminate interactive clusters; serverless for fast start; Photon for SQL/join-heavy work; spot for workers; policies and tags for control.

**Spark performance:** shuffles are the main cost; broadcast small tables; trust AQE; hunt skew in the Spark UI (one long task); built-in functions beat Python UDFs; pandas UDFs beat plain UDFs; avoid `collect()` on large data; target 100 to 200 MB partitions.

**Data types:** `INT` 4 bytes, `BIGINT` 8, `DATE` 4, `TIMESTAMP` 8, `DOUBLE` 8, `DECIMAL` up to 16; use `DECIMAL` for money and `DATE` for dates; keep filter columns within the first 32 columns for statistics.

**Governance:** grants to groups; `USE CATALOG` + `USE SCHEMA` + `SELECT`; managed tables and volumes by default; row filters and column masks for sensitive data; lineage and audit in system tables.

**Operations:** run production as service principals; secrets in secret scopes (Key Vault-backed on Azure); deploy with Asset Bundles through CI/CD; never share streaming checkpoints; never edit Delta files manually.

---

**Final note.** Databricks evolves quickly. Product names (Delta Live Tables to Lakeflow Declarative Pipelines), feature defaults (deletion vectors, predictive optimization, serverless capabilities, automatic liquid clustering) and limits change between releases. Items marked **(version-sensitive)** should be verified in the current Databricks documentation before an interview, exam or production design.
