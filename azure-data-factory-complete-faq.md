# Azure Data Factory: The Complete FAQ and Study Reference

**Purpose of this document.** This is a single, self-contained reference that explains Azure Data Factory (ADF) from first principles to production practice. It is written in question-and-answer form. Every answer explains **what** something is, **why** it exists, **how** it works internally, and **what it means for performance and cost**. It is designed to be loaded into a study tool such as NotebookLM so the tool can explain any topic clearly.

**How to read it.**
- Parts build on each other: fundamentals and components first, then runtimes, data movement, control flow, triggers, data flows, DevOps, security, monitoring, patterns, performance, troubleshooting and interview scenarios.
- Terms are defined the first time they appear and again in the glossary.
- Anything that is known to change over time (limits, defaults, product positioning, preview features) is marked **(version-sensitive)**. Always confirm those against the current Microsoft documentation before relying on them in production or an exam.

**Contents**
1. Fundamentals and architecture
2. Core building blocks (linked services, datasets, activities, pipelines)
3. Integration Runtime (Azure, Self-hosted, Azure-SSIS)
4. The Copy activity in depth (including data types and byte sizes)
5. Control flow activities and the expression language
6. Triggers
7. Mapping Data Flows
8. Source control and CI/CD
9. Security
10. Monitoring, logging and alerting
11. Error handling and reliability patterns
12. Common design patterns (incremental load, metadata-driven, CDC, SCD)
13. Integration with other Azure services (Databricks, Synapse, SQL, Fabric)
14. Performance tuning and cost optimization
15. Limits and quotas
16. Troubleshooting: common errors and causes
17. Scenario-based interview questions
18. Glossary
19. One-page cheat sheet

---

# Part 1: Fundamentals and Architecture

### Q1. What is Azure Data Factory and what problem does it solve?

Azure Data Factory is Microsoft's **cloud-based, serverless data integration and orchestration service**. It lets you build, schedule and monitor **data pipelines** that move data between many sources and destinations (on-premises and cloud) and coordinate transformations that run in other services or in ADF's own data flow engine.

**The problem it solves.** Real organizations keep data in dozens of places (SQL Server on-premises, SaaS applications, files, APIs, data lakes, warehouses). Moving and preparing that data by hand or with scattered scripts is fragile, hard to schedule, hard to monitor and hard to secure. ADF provides one managed place to:
- **Connect** to 100+ data stores through built-in connectors.
- **Move** data at scale using managed, parallel infrastructure.
- **Orchestrate** multi-step workflows with dependencies, branching, loops and retries.
- **Transform** data using visual data flows or by calling Databricks, Synapse, SQL, functions and more.
- **Schedule and monitor** everything in one place with alerts and logs.

Because it is serverless, you do not provision or patch servers; you pay for orchestration activity runs, data movement duration and data flow compute time.

### Q2. Is ADF an ETL tool or an orchestrator?

Both, but it is best understood as **primarily an orchestration and data movement service**, with a transformation capability bolted on.
- **Orchestration:** control flow (pipelines, dependencies, loops, conditions, triggers).
- **Data movement (EL):** the Copy activity is the workhorse; it extracts and loads data without needing a transformation engine.
- **Transformation (T):** **Mapping Data Flows** run visually designed transformations on managed Spark clusters. For heavy or code-based transformation, ADF usually **orchestrates** other engines such as Azure Databricks, Synapse Spark, stored procedures in a SQL engine, or Azure Functions.

The modern pattern is **ELT**: use ADF to land raw data quickly into a lake or warehouse, then transform inside a powerful engine (Databricks, Synapse, SQL) where the data already lives.

### Q3. What are the main components of ADF and how do they relate?

| Component | What it is | Analogy |
|---|---|---|
| **Pipeline** | A logical group of activities that performs a unit of work | A workflow or recipe |
| **Activity** | A single step in a pipeline (copy, lookup, data flow, notebook, etc.) | One step of the recipe |
| **Dataset** | A named view of data (a table, file, folder) that activities read or write | A pointer to the ingredients |
| **Linked service** | The connection information to a data store or compute service | A saved connection string |
| **Trigger** | What starts a pipeline run (schedule, window, event, manual) | The alarm clock |
| **Integration Runtime (IR)** | The compute infrastructure that actually runs activities and moves data | The kitchen and its cooks |
| **Data flow** | A visually designed transformation that runs on managed Spark | A transformation subroutine |
| **Parameters / variables** | Dynamic values passed in or stored during a run | Function arguments / local variables |
| **Factory** | The top-level ADF resource that contains everything | The whole restaurant |

**How they fit:** a **trigger** starts a **pipeline**; the pipeline's **activities** use **datasets** (which refer to **linked services**) to read and write data; the **Integration Runtime** provides the compute to do the work.

### Q4. What are the control plane and data plane in ADF?

- The **ADF service (control plane)** stores your pipeline definitions, schedules triggers, dispatches activities and records run history. It does not itself move your data.
- The **Integration Runtime (data plane)** is where data actually flows. For a Copy activity, data moves from source to sink **through the IR**, not through the ADF service. This is why IR placement (region, network) matters so much for performance, security and cost.

### Q5. What is the difference between ADF V1 and V2?

ADF V2 is the current generation and the only version you should use; V1 was a time-slice, dataset-availability-driven model that has been retired or is legacy. V2 introduced control flow, flexible triggers, parameters, the Integration Runtime concept, mapping data flows, SSIS lift-and-shift, and Git integration. Any modern ADF discussion means V2.

### Q6. How does ADF compare with SSIS, Synapse pipelines, Fabric Data Factory, Databricks and Airflow?

| Tool | Nature | Typical use |
|---|---|---|
| **ADF** | Cloud PaaS orchestration plus movement plus visual transformation | Cloud ELT orchestration, hybrid data movement, scheduling |
| **SSIS** | On-premises ETL engine inside SQL Server | Existing SQL Server ETL; can be lifted into ADF via the Azure-SSIS IR |
| **Azure Synapse pipelines** | Same pipeline engine as ADF embedded in Synapse workspaces | Orchestration inside a Synapse analytics solution |
| **Microsoft Fabric Data Factory** | Next-generation data integration inside Microsoft Fabric (pipelines and Dataflow Gen2) | New Fabric-based solutions **(version-sensitive; positioning evolves)** |
| **Azure Databricks** | Spark-based lakehouse platform | Heavy transformation, ML, streaming; commonly orchestrated by ADF |
| **Apache Airflow** | Code-first (Python) orchestrator | Complex, code-defined DAGs; ADF offers a managed Airflow option whose availability may change **(version-sensitive)** |

A very common architecture: **ADF orchestrates and ingests; Databricks transforms; Power BI consumes.**

### Q7. What is serverless and what does that imply operationally?

You do not manage VMs for the Azure Integration Runtime: capacity is allocated on demand and scales for you. You still design for throughput (DIUs, parallelism), network access (private endpoints) and cost. Self-hosted IR is the exception: you install and maintain it on your own machines.

### Q8. What is the authoring UI and what are its hubs?

ADF Studio (the web UI) has hubs:
- **Home:** quick starts and templates.
- **Author:** build pipelines, datasets, data flows, triggers.
- **Monitor:** pipeline runs, trigger runs, integration runtimes, alerts.
- **Manage:** linked services, integration runtimes, triggers, Git configuration, global parameters, credentials, managed private endpoints, ARM template settings.
- **Learning center:** documentation and tutorials.

---

# Part 2: Core Building Blocks

### Q9. What is a linked service?

A linked service is the **connection definition** to an external resource: it holds the endpoint, authentication method and which Integration Runtime to use (via `connectVia`). It is much like a connection string. There are two roles:
- **Data store linked services** (Azure SQL, ADLS, Blob, Snowflake, REST, SAP, and so on) describing where data lives.
- **Compute linked services** (Databricks, HDInsight, Azure Batch, Azure Functions, Azure ML) describing where transformations run.

Best practice: store secrets in **Azure Key Vault** and reference them from the linked service rather than embedding passwords. **Parameterize** linked services (for example the server or database name) so one definition can serve multiple environments or databases.

### Q10. What is a dataset?

A dataset is a **named reference to the data** an activity uses: which table, which file path, which format, which schema. It sits on top of a linked service (the *where*) and adds the *what* (the specific object and format such as DelimitedText, Parquet, JSON, Avro, ORC, Binary, Excel, XML).

Datasets can be **parameterized** (file name, folder path, table name, schema), which lets you build **generic, reusable** datasets used by many pipelines instead of hundreds of near-identical ones. The schema stored in a dataset is optional metadata; Copy activity can discover schema at runtime.

### Q11. What are the categories of activities?

1. **Data movement:** Copy activity.
2. **Data transformation:** Mapping Data Flow; Databricks Notebook / JAR / Python; HDInsight activities; Stored Procedure; Script; Azure Function; Azure Batch custom activity; Synapse notebook / Spark job; Azure Machine Learning execute pipeline.
3. **Control flow:** ForEach, If Condition, Switch, Until, Wait, Set Variable, Append Variable, Filter, Execute Pipeline, Lookup, Get Metadata, Validation, Web, Webhook, Fail.

Every activity has common settings: **name, description, timeout, retry, retry interval, secure input, secure output** and **dependencies** on other activities.

### Q12. What is a pipeline and what are its parts?

A pipeline is a **unit of orchestration**: a graph of activities connected by dependency conditions. It has:
- **Parameters** (read-only inputs supplied at run time).
- **Variables** (read/write values within the run).
- **Activities** and their dependency arrows.
- **Concurrency** (maximum simultaneous runs of this pipeline) and annotations.
- **Settings** such as elapsed time metric thresholds.

Good design principle: keep pipelines **small and single-purpose**, compose them using **Execute Pipeline**, and make them **parameterized and idempotent** (safe to re-run).

### Q13. What is the difference between parameters, variables and global parameters?

- **Pipeline parameters:** defined on a pipeline; **immutable** during the run; set by triggers or the caller; referenced as `@pipeline().parameters.name`.
- **Variables:** defined on a pipeline; **mutable** with Set Variable / Append Variable; types are String, Boolean, Array (and a pipeline-scoped nature). Variables are **scoped to the pipeline run**, so using Set Variable inside a **parallel ForEach** creates race conditions; set the ForEach to sequential or use a child pipeline.
- **Global parameters:** constants defined at the **factory** level, available to all pipelines as `@pipeline().globalParameters.name`; ideal for environment settings (environment name, base paths), and they can be overridden during CI/CD.
- **Dataset and linked service parameters:** make connection and data definitions dynamic.
- **Data flow parameters:** passed from the pipeline into a mapping data flow.

### Q14. What are system variables?

Built-in values you can use in expressions, such as `@pipeline().RunId`, `@pipeline().Pipeline` (name), `@pipeline().DataFactory`, `@pipeline().TriggerType`, `@pipeline().TriggerName`, `@pipeline().TriggerTime`, `@pipeline().GroupId`. Triggers add their own, for example `@trigger().scheduledTime`, `@trigger().startTime`, `@trigger().outputs.windowStartTime`, `@trigger().outputs.windowEndTime` (tumbling window), and `@triggerBody().fileName` / `@triggerBody().folderPath` (storage event trigger).

### Q15. What is dynamic content and the expression language?

Most fields in the UI can be set to **dynamic content**: an expression evaluated at run time, written after `@`. Examples:

```
@pipeline().parameters.tableName
@concat('raw/sales/', formatDateTime(utcnow(), 'yyyy/MM/dd'))
@activity('LookupWatermark').output.firstRow.WatermarkValue
@item().schemaName                                  // inside a ForEach
@equals(activity('GetMeta').output.exists, true)
@if(greater(int(variables('count')), 0), 'yes', 'no')
@{pipeline().DataFactory}-@{pipeline().Pipeline}    // string interpolation
```

Categories of functions: string (`concat`, `substring`, `replace`, `split`, `toLower`, `trim`), collection (`length`, `first`, `last`, `union`, `take`, `skip`, `contains`, `empty`, `createArray`), logical (`if`, `equals`, `and`, `or`, `not`, `greater`, `less`), conversion (`int`, `string`, `json`, `float`, `bool`, `base64`, `xml`), math (`add`, `sub`, `mul`, `div`, `mod`, `rand`), and date/time (`utcnow`, `addDays`, `addHours`, `formatDateTime`, `startOfDay`, `convertTimeZone`, `ticks`, `dayOfWeek`).

Tips: a literal string starting with `@` must be escaped as `@@`; string interpolation `@{...}` can be embedded in text; `utcnow()` is UTC, so convert time zones explicitly; remember arrays and objects need `json()` or `string()` conversions in some contexts.

### Q16. What are annotations and user properties?

- **Annotations** are labels on pipelines, datasets and other resources for filtering in the Monitor and organizing (for example `domain:finance`).
- **User properties** are key/value pairs on an activity (often the source table name) that appear as columns in Monitor, making it much easier to find which table or file a run processed.

### Q17. What does an ADF pipeline look like as JSON?

Everything authored in the UI is stored as JSON. A simplified Copy pipeline:

```json
{
  "name": "pl_copy_orders",
  "properties": {
    "parameters": { "loadDate": { "type": "string" } },
    "activities": [
      {
        "name": "Copy orders",
        "type": "Copy",
        "dependsOn": [],
        "policy": { "timeout": "0.02:00:00", "retry": 2, "retryIntervalInSeconds": 60 },
        "typeProperties": {
          "source": { "type": "AzureSqlSource",
                      "sqlReaderQuery": "SELECT * FROM dbo.Orders WHERE OrderDate >= '@{pipeline().parameters.loadDate}'" },
          "sink": { "type": "ParquetSink" },
          "enableStaging": false
        },
        "inputs":  [ { "referenceName": "ds_sql_orders",  "type": "DatasetReference" } ],
        "outputs": [ { "referenceName": "ds_adls_orders", "type": "DatasetReference" } ]
      }
    ]
  }
}
```

Knowing the JSON helps with Git diffs, CI/CD parameterization and troubleshooting.

---

# Part 3: Integration Runtime

### Q18. What is an Integration Runtime (IR)?

The IR is the **compute infrastructure** ADF uses to run activities. It provides: **data movement** (Copy), **data flow execution** (Spark), **activity dispatch** (calling Databricks, stored procedures and similar on compute services), and **SSIS package execution**. Every linked service has a `connectVia` property naming its IR (default: the auto-resolve Azure IR).

### Q19. What are the types of Integration Runtime?

| Type | Where it runs | Used for |
|---|---|---|
| **Azure IR** | Microsoft-managed, serverless | Cloud-to-cloud copy, data flows, dispatching activities to cloud services |
| **Azure IR with Managed Virtual Network** | Microsoft-managed inside an ADF-managed VNet | Private connectivity via managed private endpoints; data exfiltration control |
| **Self-hosted IR (SHIR)** | Your own Windows machines (on-premises or VMs) | Reaching on-premises or private-network sources; hybrid movement |
| **Azure-SSIS IR** | Dedicated Azure VM cluster managed by ADF | Running existing SSIS packages in the cloud |

### Q20. How does the Azure IR choose where to run (auto-resolve)?

The **AutoResolveIntegrationRuntime** automatically picks a region for Copy based on the sink's region (or the factory's region if it cannot determine one). For performance and egress cost, you generally want the IR **in the same region** as the data stores. You can create region-specific Azure IRs when you need to control data residency or minimize cross-region traffic. Data flow clusters run in the region configured on the Azure IR.

### Q21. What is a Self-hosted Integration Runtime and how does it work?

You install the IR software on a **Windows machine** that can reach the data source (for example a server inside your corporate network). The SHIR:
- Registers with the factory using an **authentication key**.
- Makes only **outbound HTTPS (port 443)** connections to the ADF service (via Azure Relay and service endpoints); **no inbound firewall ports** are required.
- Stores credentials **encrypted** on the node (using Windows DPAPI) and decrypts them locally when connecting to the source.
- Runs Copy and activity dispatch for sources that cloud-based IRs cannot reach.

**High availability and scale:** you can register **up to four nodes** in one logical SHIR for failover and more parallel capacity. Resource usage (CPU, memory, concurrent jobs) is visible in Monitor.

### Q22. Can a Self-hosted IR be shared across factories?

Yes. A SHIR can be **shared** with other data factories: the factory that owns it grants permission, and other factories create a **linked Integration Runtime** referencing it. This avoids installing the SHIR repeatedly and centralizes maintenance.

### Q23. What are the best practices for Self-hosted IR?

- Install on a **dedicated, appropriately sized machine** (not the source database server).
- Use **two or more nodes** for production availability; keep software **up to date** (auto-update can be enabled).
- Place it **close to the data source** on a fast network for performance.
- Use **machine-level security**: patching, restricted access, monitored services.
- Keep outbound allow-lists correct (service endpoints, Azure Relay).
- Watch concurrent jobs limit and CPU/memory metrics; scale out nodes rather than overloading one.
- It is not supported inside a managed virtual network IR configuration; they solve different problems.

### Q24. What is the Azure-SSIS Integration Runtime?

A fully managed cluster of Azure VMs dedicated to **executing SSIS packages**. It lets organizations **lift and shift** existing SSIS projects without rewriting them. Packages are typically stored in **SSISDB** (hosted in Azure SQL Database or Managed Instance) or in file shares / Azure Files, and are run through the **Execute SSIS Package** activity. You choose node size and count, edition, and can join a virtual network to reach on-premises data. It is billed while running, so **start and stop it on a schedule** to save cost.

### Q25. What is a Managed Virtual Network and managed private endpoints?

When an Azure IR is created in a **managed virtual network**, ADF provisions an isolated network for its compute. You create **managed private endpoints** to target resources (storage, SQL, Key Vault, and so on); the target resource owner must **approve** each private endpoint connection. Benefits: traffic stays on the Microsoft backbone over private links, you can disable public access on data stores, and data exfiltration risk is reduced. Notes: cluster start for data flows in managed VNet may be slower; not all connectors are supported; you need the IR type to be the managed VNet Azure IR (not the default auto-resolve IR created before enabling).

### Q26. What is Time-To-Live (TTL) on an Azure IR?

For **data flows**, each run normally starts a Spark cluster (a "cold start" taking a few minutes). A **TTL** on the Azure IR keeps the cluster warm for a set number of minutes after a run, so subsequent data flows reuse it and start in seconds. You pay while the cluster is alive, so TTL trades cost for speed. Useful when chaining many data flows or running them frequently; wasteful if runs are far apart. Copy activities on Azure IR do not need TTL.

### Q27. Which IR does a Copy activity use when source and sink use different IRs?

Copy runs on **one IR**: the one linked to the **source** if it is self-hosted (or private), otherwise determined by the sink and the auto-resolve rules. In general: when either side needs a SHIR, the SHIR runs the copy. Understanding this prevents surprises such as a copy unexpectedly using the wrong region.

---

# Part 4: The Copy Activity in Depth

### Q28. How does the Copy activity work internally?

1. The IR **reads** data from the source in parallel using the source's native protocol or API.
2. It applies **serialization/deserialization**, optional **compression**, **format conversion** and **column mapping** (using an internal, interim set of data types).
3. It **writes** to the sink, using bulk-load mechanisms where available.
4. Throughput depends on: source read speed, sink write speed, network bandwidth between IR and stores, the number of **DIUs** (Azure IR) or the **node resources** (SHIR), **parallelism**, and the formats and sizes of files/rows.

Copy runs data **through the IR**, so IR region and network are critical. It can copy between 100+ source/sink pairs and supports binary (as-is) copies where no parsing is needed, which is the fastest mode.

### Q29. What is a DIU?

A **Data Integration Unit** is a measure of the power (a combination of CPU, memory and network resource) of one Azure IR copy execution. More DIUs mean higher potential throughput, and cost is **DIU-hours** (DIUs multiplied by duration). You can set DIUs between 2 and 256, or leave it on **Auto**, where ADF picks based on the source/sink pair and data pattern. DIUs apply to the Azure IR only; a self-hosted IR is limited by its machines.

**Billing example (illustrative):** a copy that runs 30 minutes using 8 DIUs consumes 8 × 0.5 = **4 DIU-hours**. Doubling DIUs only helps if the bottleneck is the IR; if the source database or the network is the limit, extra DIUs just cost more with no speed-up.

### Q30. What is parallelism in Copy?

- **Degree of copy parallelism (`parallelCopies`):** the number of threads reading from the source and writing to the sink concurrently. By default ADF chooses it dynamically.
- For **file-based** sources, parallelism spreads across files (and for large single files, in chunks where supported).
- For **database** sources, parallel reads need **partition options** (below) so threads read distinct slices.

### Q31. What are source partition options for database copy?

To read a big table in parallel without overlap:
- **Physical partitions** (for sources that expose them, such as Oracle, SQL Server partitioned tables).
- **Dynamic range:** you give a **partition column** (an integer, date or datetime column) with lower and upper bounds; ADF divides the range into slices read in parallel. Bounds are used only to split, not to filter rows.
- **None:** single-threaded read.

Choose a partition column that is **evenly distributed and indexed**; a skewed column makes one slice do all the work.

### Q32. What is staged copy?

A staged copy writes data first to **interim Azure Blob / ADLS storage** and then loads from there into the sink using the sink's fastest bulk loader. It is used for sinks like **Azure Synapse (via COPY command or PolyBase)** and **Snowflake**, and when direct copy is not possible because of format or network restrictions. The tradeoff: extra storage I/O and a temporary storage cost, but much faster bulk loading for warehouses. Direct sink writes (row-by-row inserts) into warehouses are far slower.

### Q33. What are the sink behaviors for databases (insert, upsert, stored procedure)?

- **Insert:** append rows (default).
- **Upsert:** update matching rows by key columns and insert others (supported for sinks such as Azure SQL, Synapse, Snowflake and some others **(version-sensitive)**); you list the **key columns**.
- **Stored procedure:** call your own procedure with a table-valued parameter so you control merge logic and keep the load idempotent.
- **Pre-copy script:** SQL run before writing, for example `TRUNCATE TABLE staging.Orders`.
- **Write batch size** (rows per batch; default around 10,000 for SQL sinks) and **write batch timeout** tune insert behavior; **table option: auto create** creates the table based on source schema (convenient for staging, not recommended for curated tables because types may not be what you want).

### Q34. What are the sink behaviors for files?

- **Copy behavior:** *preserve hierarchy* (keep folder structure), *flatten hierarchy* (all files into one folder with generated names), or *merge files* (combine into a single file).
- **Max rows per file / file name prefix:** split large outputs into multiple files of controlled size.
- **Format settings:** compression codec (Snappy, gzip, ZipDeflate, bzip2 and others depending on format), header row, delimiters, encoding, quote characters.

Sizing guidance: for analytics in a lake, avoid writing thousands of tiny files; aim for files of roughly **100 MB to 1 GB** (Parquet) so downstream engines run efficiently.

### Q35. What is column mapping and schema drift handling in Copy?

By default Copy maps source columns to sink columns **by name** (case-insensitive for many stores). You can define explicit mapping in the UI, **import schemas**, or use dynamic mapping through an expression that carries a TabularTranslator JSON. If you leave mapping empty with schema-flexible sinks (for example copying a table to Parquet), ADF maps all columns automatically, which is useful for generic metadata-driven pipelines. For sinks that require a fixed schema, column count or type mismatches cause failures.

### Q36. What are the interim data types, and how big are they?

Copy converts source types to **ADF interim types** and then to sink types. Approximate in-memory sizes (based on .NET representations, which is what the copy engine uses; treat as a guide):

| Interim type | Approx. size | Notes |
|---|---|---|
| Boolean | 1 byte | |
| Byte / SByte | 1 byte | Unsigned / signed |
| Int16 / UInt16 | 2 bytes | |
| Int32 / UInt32 | 4 bytes | |
| Int64 / UInt64 | 8 bytes | |
| Single (float) | 4 bytes | About 7 digits of precision |
| Double | 8 bytes | About 15 to 16 digits of precision |
| Decimal | 16 bytes | 128-bit; up to 28 to 29 significant digits in the interim type; precision beyond that may be truncated or cause errors |
| Guid | 16 bytes | |
| Datetime | 8 bytes | |
| DatetimeOffset | About 10 bytes (16 in memory) | Includes time-zone offset |
| Timespan | 8 bytes | |
| String | Variable | UTF-16 in memory (about 2 bytes per character) |
| Byte array | Variable | Raw binary |

**Common SQL Server source sizes** (storage on disk): `tinyint` 1, `smallint` 2, `int` 4, `bigint` 8, `bit` 1 bit (packed), `real` 4, `float` 8, `date` 3, `datetime` 8, `datetime2` 6 to 8, `uniqueidentifier` 16, `money` 8, and `decimal(p,s)` 5 bytes (precision 1 to 9), 9 bytes (10 to 19), 13 bytes (20 to 28) or 17 bytes (29 to 38).

**Why this matters for performance and correctness:**
- **Row width** drives throughput: wide rows mean fewer rows per batch and more bytes over the network. Select only the columns you need.
- **Large objects** (`varchar(max)`, `nvarchar(max)`, `varbinary(max)`, XML) slow bulk loads and may force smaller batch sizes.
- **Decimal precision:** `decimal(38,s)` values can exceed the interim type's precision; cast to a lower precision in the source query when you know the real range.
- **Datetime handling:** `datetime2`/`datetimeoffset` to Parquet and between systems can produce time-zone and precision surprises (for example nanosecond or 100-ns precision); agree on UTC and precision upfront.
- **Parquet sinks:** column names with spaces or special characters often need to be renamed; use the data flow or mapping to clean names.
- Mismatched types are the single most common cause of mapping failures; define staging tables as permissive types (for example strings) when landing raw data, then convert in the transformation step.

### Q37. What is fault tolerance in Copy?

Copy can **skip incompatible rows** (type conversion problems, constraint violations such as primary key duplicates) instead of failing the entire run, and **log** the skipped rows to a storage path for review. It can also skip missing files. Use it carefully: silently skipping data can hide quality problems, so always inspect the logs and consider a threshold-based alert.

### Q38. What is data consistency verification?

For file-based, binary-style copies, Copy can **verify** that source and destination files match using checksums (supported for certain connectors). If verification fails, the run can fail or be logged. It adds some overhead but provides integrity assurance for critical transfers.

### Q39. Can Copy preserve file metadata and ACLs?

Yes for supported pairs (for example between ADLS Gen2, Blob and file systems), you can **preserve** attributes such as ACLs, owner, last-modified time and content type when copying in binary mode. This matters for migrations.

### Q40. How do you copy data from REST APIs?

Use the **REST** connector (or HTTP for files). Key settings: base URL, authentication (anonymous, basic, OAuth2 client credentials, managed identity, service principal), headers, request method and body, and **pagination rules** (next-page URL from a response property, offset or page number increments, or header-based). Respect API rate limits using lower parallelism and retries. For deeply nested JSON responses, use **collection reference** and mapping to flatten, or land raw JSON and flatten in Spark or a data flow.

### Q41. What file formats does Copy support and how do they compare?

| Format | Characteristics | Typical use |
|---|---|---|
| **DelimitedText (CSV)** | Row-based text; large; no types | Interchange, small data |
| **JSON** | Hierarchical text | APIs, logs |
| **Parquet** | Columnar, compressed, typed | Analytics landing, lake storage (preferred) |
| **ORC** | Columnar, Hive ecosystem | Hive/HDInsight |
| **Avro** | Row-based binary with schema | Event Hubs capture, streaming |
| **Binary** | No parsing; byte-for-byte | Fastest file movement |
| **Excel, XML** | Business formats | Source ingestion (sources only for Excel in many cases) |

Landing as Parquet usually reduces storage and speeds later processing, provided column names and types are clean.

---

# Part 5: Control Flow Activities and the Expression Language

### Q42. What does the Lookup activity do and what are its limits?

Lookup reads data from a dataset (a table query, file, or stored procedure) and returns it to the pipeline as **output**. It can return **the first row only** (`firstRow`) or **all rows** (array in `value`). **Limits:** it returns at most **5,000 rows** and about **4 MB** of output **(version-sensitive)**; for larger data use Copy or a data flow. Typical uses: read a control table, fetch the current watermark, get a list of tables to process.

Access results with `@activity('Lookup1').output.firstRow.ColumnName` or `@activity('Lookup1').output.value`.

### Q43. What does Get Metadata do?

It returns metadata about a file, folder or table: `exists`, `itemName`, `itemType`, `size`, `lastModified`, `childItems` (list of files and folders), `structure`, `columnCount`, `contentMD5`. Common uses: check whether a file exists, list files to process, check modification times. `childItems` output is size-limited (about 4 MB) so very large folders need other approaches (such as partition folders or storage listing via REST).

### Q44. How does the ForEach activity work and what are its pitfalls?

ForEach iterates over an **array** (`items`) and runs inner activities per element, referenced as `@item()`.
- **Sequential** (`isSequential: true`) runs one at a time; otherwise iterations run **in parallel** up to `batchCount` (default **20**, maximum **50**).
- **Pitfall 1: variables.** Pipeline variables are shared, so `Set Variable` inside a parallel ForEach causes **race conditions**. Use sequential mode, or do the per-item work in a **child pipeline** with parameters.
- **Pitfall 2: limits.** The items array has an upper bound (around 100,000 items) and nested loops are restricted. A ForEach inside a ForEach is not directly supported; the standard workaround is an **Execute Pipeline** activity calling a child pipeline that contains the inner ForEach **(version-sensitive; verify nesting rules in current docs)**.
- **Pitfall 3: unbounded parallel load.** Parallel iterations may overload a source database or API; tune `batchCount` to what the source can handle.

### Q45. What is the difference between If Condition, Switch, Filter and Until?

- **If Condition:** evaluates a boolean expression and runs the True or False branch.
- **Switch:** compares an expression to multiple cases (strings) and runs the matching branch, with a default.
- **Filter:** takes an array and a condition and **returns the subset** that satisfies it (a transformation of data, not a branch).
- **Until:** repeats activities **until** an expression is true (like a do-until), with a **timeout** (default up to 7 days) and typically a **Wait** inside; used for polling.

### Q46. What do Wait, Set Variable, Append Variable and Fail do?

- **Wait:** pauses the pipeline for N seconds.
- **Set Variable:** assigns a value (string/boolean/array) to a variable.
- **Append Variable:** adds an element to an array variable.
- **Fail:** deliberately fails the pipeline with a **custom message and error code**, useful for validation failures (for example "row count was zero") so that monitoring and alerts are meaningful.

### Q47. What is the Execute Pipeline activity?

It calls another pipeline (a **child**) from a parent, passing parameters. Options include **Wait on completion** (parent waits and fails if the child fails, or continues without waiting). It is the foundation of modular design: parent "orchestrator" pipelines call reusable child pipelines. A child pipeline can return values to the parent through its **pipeline return value** feature **(version-sensitive)**. Cross-factory calls are not native; use the REST API through a Web activity when needed.

### Q48. What are the Web and Webhook activities, and what is the difference?

- **Web activity:** calls a REST endpoint (GET, POST, and so on) and waits for the response, with a **short timeout (about 1 minute)** for the HTTP response. It supports authentication including **managed identity**. Use it to call APIs, trigger Logic Apps, start Azure resources, or call the ADF REST API itself.
- **Webhook activity:** calls an endpoint and passes a **callback URL**; the pipeline **waits** until the external system calls back to signal completion (or the timeout elapses). Use it for long-running external processes.

For long-running work, avoid holding a Web activity open: use Webhook, polling (Until + Web), or durable functions.

### Q49. What are Validation, Script and Stored Procedure activities?

- **Validation:** waits until a file or folder **exists** (and optionally meets a minimum size), polling at a sleep interval until timeout; good for "wait for the vendor file".
- **Script:** runs SQL scripts (DDL/DML/queries) on supported databases and can return results; it supports multiple statements.
- **Stored Procedure:** calls a stored procedure with parameters.

### Q50. How do activity dependencies work?

Activities connect with arrows labelled **Succeeded**, **Failed**, **Completed** (either succeeded or failed) or **Skipped**. When an activity has **multiple incoming dependencies**, they are combined with **AND** (all must be satisfied). To get "run if A **or** B failed", use separate paths or restructure, for example by sending both failure paths into a shared Fail/logging pipeline via Execute Pipeline. Understanding these semantics is critical for error handling (see Part 11).

### Q51. What are the common expression pitfalls?

- Types: `@activity('x').output.value` is an array; `length()` it, loop it, or index with `[0]`.
- Null handling: use `coalesce()` or check `empty()`; accessing a property of a missing object fails.
- Numbers from Lookup arrive as numbers; concatenating into SQL needs `string()`.
- SQL injection / quoting: building SQL with `concat` and unescaped quotes breaks on apostrophes; use parameters or `replace()` to escape `'`.
- Date formats: `formatDateTime` uses .NET custom patterns (`yyyy-MM-dd`, `HH` for 24-hour, `MM` for month versus `mm` for minute).
- Expressions in JSON bodies must produce valid JSON; wrap with `json()` where an object is required.

---

# Part 6: Triggers

### Q52. What types of triggers exist?

1. **Schedule trigger:** runs on a wall-clock schedule (minutes, hours, days, weeks, months) with start/end dates and a time zone.
2. **Tumbling window trigger:** fires for **fixed-size, non-overlapping, contiguous time windows**, maintaining state per window.
3. **Storage event trigger:** fires when a blob is **created or deleted** in Blob/ADLS Gen2 (through Event Grid).
4. **Custom event trigger:** fires on events from an **Event Grid custom topic**.
5. **Manual / on-demand:** started from the UI ("Trigger now"), REST API, SDK, PowerShell or another pipeline.
6. **Debug:** an interactive run from the authoring UI using the unpublished (working) version.

### Q53. How do schedule and tumbling window triggers differ?

| Feature | Schedule trigger | Tumbling window trigger |
|---|---|---|
| Pipeline relationship | Many-to-many | **One-to-one** (one pipeline per trigger) |
| Backfill (past windows) | **No** | **Yes** (start date in the past creates runs for each window) |
| Window awareness | No, only run time | Yes: `windowStartTime` and `windowEndTime` |
| Retry policy built in | No (use activity retry) | **Yes** (retry count and interval per window) |
| Concurrency control | No | **Yes** (`maxConcurrency`) |
| Dependencies | No | **Yes** (on other tumbling window triggers, including self-dependency) |
| Rerun a specific period | Manual | **Yes** (rerun a window) |
| Gaps/overlaps | Based on schedule | Contiguous, no gaps, no overlaps |

Use tumbling windows for **incremental, time-sliced, reprocessable loads** (for example "process each hour of data exactly once"), and schedule triggers for simple recurring runs.

### Q54. How do you use the window start and end in a pipeline?

Create pipeline parameters (for example `windowStart`, `windowEnd`) and set them in the trigger as `@trigger().outputs.windowStartTime` and `@trigger().outputs.windowEndTime`. Use them in the source query:

```sql
SELECT * FROM dbo.Orders
WHERE ModifiedDate >= '@{pipeline().parameters.windowStart}'
  AND ModifiedDate <  '@{pipeline().parameters.windowEnd}'
```

Use a half-open interval (>= start, < end) so rows on boundaries are neither missed nor duplicated.

### Q55. How does a storage event trigger work and what must be configured?

When a blob is created or deleted, **Azure Event Grid** sends an event to ADF, which starts the pipeline. You configure the storage account, container, optional `blobPathBeginsWith` / `blobPathEndsWith` filters (for example `.csv`), and whether to ignore empty blobs. Requirements: the **Microsoft.EventGrid resource provider** must be registered in the subscription; the person publishing needs appropriate permissions on the storage account; the storage account must be a supported type (general-purpose v2, ADLS Gen2, BlobStorage). The event provides `@triggerBody().folderPath` and `@triggerBody().fileName`.

**Caveats:** a "blob created" event may fire before a large upload is fully committed in some patterns (for example multipart or a writer that creates then appends), so many teams use a **marker or trigger file** (written last) or a Validation activity; events may be delivered more than once, so design **idempotent** pipelines.

### Q56. Why is my trigger not firing?

Checklist: the trigger must be **started (activated)** and the factory changes **published** (triggers run the *published* version, not the debug draft); schedule start date/time zone; for event triggers, Event Grid provider registration, permissions, path filters, and that events are actually emitted; for tumbling windows, check dependencies and `maxConcurrency` (a window can be waiting); verify in **Monitor, Trigger runs** for errors such as a failed pipeline start due to missing parameters.

### Q57. What happens to triggers during deployment?

Triggers must be **stopped before deploying changes** to them (an active trigger cannot be modified by an ARM deployment) and **restarted** afterward. CI/CD pre- and post-deployment scripts automate this (see Part 8).

### Q58. How do you schedule in a specific time zone, and how do you handle daylight saving time?

Schedule triggers allow a **time zone** selection and adjust for daylight saving time automatically for time zones that observe it (recurrences are interpreted in that zone). Tumbling windows use UTC-based windows; for local-time reporting convert time zones in expressions with `convertTimeZone`. A frequent source of bugs is mixing UTC (`utcnow()`) with local source timestamps.

---

# Part 7: Mapping Data Flows

### Q59. What is a Mapping Data Flow?

A visually designed, code-free transformation that ADF runs on **managed Apache Spark clusters** (provisioned on your Azure IR). You design the logic in a graph; ADF translates it into Spark code (the **data flow script**), runs it, and you pay for the **vCore-hours** the cluster is running. It supports large-scale transformations such as joins, aggregations, deduplication, SCD logic and schema-drift-tolerant processing without writing Spark.

### Q60. What are the main transformations?

| Transformation | Purpose |
|---|---|
| **Source / Sink** | Read from / write to a dataset (including inline datasets such as Delta) |
| **Select** | Rename, reorder, drop columns; rule-based mapping |
| **Derived Column** | Create or modify columns with expressions |
| **Filter** | Keep rows matching a condition |
| **Conditional Split** | Route rows to different streams by conditions |
| **Join** | Inner, left, right, full outer, custom/cross joins between streams |
| **Lookup** | Left-outer-style match against another stream, returning first match |
| **Exists** | Keep rows that exist or do not exist in another stream (semi/anti join) |
| **Union** | Combine streams by name or position |
| **Aggregate** | Group and aggregate |
| **Window** | Window functions (rank, lag, running totals) |
| **Rank / Surrogate Key** | Rank rows; generate incrementing keys |
| **Pivot / Unpivot** | Rows to columns and back |
| **Sort** | Order rows (use sparingly; sorting is expensive) |
| **Alter Row** | Mark rows for insert, update, delete or upsert on the sink |
| **Flatten / Parse / Stringify** | Handle nested JSON and arrays |
| **Assert** | Data-quality checks with row- or data-flow-level assertions |
| **External Call** | Call a REST endpoint per row or batch |
| **Cached sink / Cached lookup** | Store results in memory for lookup across streams |
| **New Branch / Flowlet** | Reuse a stream or encapsulate reusable logic |

### Q61. What are compute types and cluster sizing for data flows?

On the Azure IR you choose a **compute type** (General Purpose or Memory Optimized; the older Compute Optimized option is deprecated **(version-sensitive)**) and a **core count** (for example 8 plus 8 driver cores up to much larger options). More cores increase parallelism and cost. Choose **Memory Optimized** for joins, aggregations and wide data with heavy memory pressure; General Purpose for most workloads. Cluster **cold start** is typically a few minutes unless you use **TTL**.

### Q62. How do partitioning and optimization settings work in data flows?

Each transformation (especially sources and sinks) has an **Optimize** tab:
- **Use current partitioning** (default, recommended in most cases).
- **Single partition** (forces one partition; use only for small outputs or specific sinks).
- **Round robin** (even distribution).
- **Hash** (by columns; helpful for joins/group-by to reduce skew).
- **Dynamic range** and **fixed range** (range partitions on a column).
- **Key** (one partition per unique value).

Over-partitioning creates many small files; under-partitioning limits parallelism. For sinks writing files, setting a number of partitions controls the number of output files. Other tuning: **broadcast** small streams in joins (auto, fixed, off), filter and select columns **early**, avoid unnecessary **sorts**, and minimize **Derived Column** complexity.

### Q63. What is schema drift and how do data flows handle it?

Schema drift is when source columns change (new, missing or renamed) without notice. Data flows can **allow schema drift** (columns not in the defined schema are carried through), **infer drifted column types**, and use **rule-based mapping** and column patterns (for example `$$` and `columns()` in Derived Column) to apply logic to columns by pattern rather than by name. This enables generic transformations over evolving datasets.

### Q64. How do data flow data types compare in size?

Data flow types mirror Spark-style types: `short` (2 bytes), `integer` (4), `long` (8), `float` (4), `double` (8), `byte` (1), `boolean`, `decimal(p,s)` (up to 38 digits of precision), `date` (4), `timestamp` (8), `string` (variable UTF-8), `binary`, and complex types (`array`, `map`, `struct`). Choosing the smallest suitable numeric type reduces memory and shuffle volume; use `decimal` for money rather than `float`/`double`; cast dates and timestamps explicitly rather than keeping them as strings.

### Q65. What is debug mode in data flows?

Turning on **data flow debug** starts an interactive Spark cluster (billed while active; time-to-live default about 60 minutes) so you can **preview data** at each transformation and iterate quickly. It is for development; production runs (triggered) start their own cluster per the IR settings. Turn debug off when finished to avoid cost. Use the **debug row limit** (default 1,000 rows) to keep previews fast.

### Q66. How do you implement SCD Type 1 and Type 2 in a data flow?

- **Type 1 (overwrite):** read source and target, use **Exists/Lookup** to find existing keys, **Alter Row** with upsert policy, and **sink** with key columns and "Allow upsert".
- **Type 2 (history):** compare tracked columns using a **hash** (for example `md5` over tracked columns) between incoming and current rows; for changed rows, produce **two outputs**: one that closes the old row (`is_current = false`, `end_date = now`) and one that inserts the new version (`is_current = true`, `start_date = now`); use **Alter Row** to mark update/insert, and sink to the dimension table with the surrogate key as the key. Add a **Surrogate Key** transformation for new keys. Ensure deterministic timestamps and handle late-arriving data explicitly.

### Q67. When should I use a data flow versus Databricks, Synapse Spark or SQL?

| Choose | When |
|---|---|
| **Mapping Data Flow** | Moderate-complexity transformations; team prefers visual, low-code; need quick schema-drift handling; no existing Spark platform |
| **Azure Databricks** | Complex logic, ML, Delta Lake features (merge, optimize, streaming), existing lakehouse, full code control and best performance tuning |
| **SQL (stored procedures, T-SQL)** | Data already in a SQL engine and transformations are set-based; ELT inside the warehouse |
| **Copy activity only** | Pure movement with minimal change |

A frequent real-world architecture uses ADF **Copy** for ingestion, **Databricks** notebooks for transformation, and ADF for scheduling and monitoring. Data flows are billed per vCore-hour and have cold-start overhead, so they are rarely the cheapest for simple work.

---

# Part 8: Source Control and CI/CD

### Q68. Why integrate ADF with Git?

Without Git, ADF saves directly to the live service ("live mode"): no versioning, no branching, no review, and changes are overwritten. Git integration (Azure DevOps Git or GitHub) gives **version control, branching, pull requests, collaboration and rollback**, and is a prerequisite for CI/CD. You can save partial work in a branch without publishing.

### Q69. What is the collaboration branch and the publish branch?

- **Collaboration branch** (usually `main`): where reviewed changes are merged. Developers create **feature branches** from it and open pull requests.
- **Publish branch** (default `adf_publish`): when you click **Publish** in the UI from the collaboration branch, ADF validates, then generates **ARM templates** (`ARMTemplateForFactory.json` and parameters file) and commits them to the publish branch. Release pipelines deploy from there.

Only the **collaboration branch** can be published through the UI.

### Q70. What is the standard CI/CD flow?

1. Developer works in a feature branch of the **dev factory** (connected to Git).
2. Pull request is reviewed and merged to `main`.
3. **Publish** generates ARM templates (or an automated build step does it, see below).
4. A release pipeline (Azure DevOps or GitHub Actions) deploys the ARM template to **test** and **prod** factories, **overriding parameters** per environment (linked service endpoints, Key Vault URLs, global parameters, trigger settings).
5. Pre-deployment: **stop triggers**. Post-deployment: **delete removed objects** and **restart triggers**.

Only the dev factory is attached to Git; test and prod are normally deployed to (not edited directly).

### Q71. How can I automate publishing without clicking Publish?

Use the **`@microsoft/azure-data-factory-utilities`** npm package in the build pipeline. Its `validate` command checks resources and its `export` command generates the ARM templates from the Git repository, so the CI pipeline produces deployable artifacts automatically on merge, without a manual publish step **(version-sensitive; check current package usage)**.

### Q72. How are environment differences handled?

- **ARM template parameters:** by default, certain properties (linked service connection details, global parameters) are parameterized; you override them per environment in the release pipeline or parameter file. You can **customize which properties are parameterized** using an `arm-template-parameters-definition.json` file in the root of the collaboration branch.
- **Global parameters:** include them in the ARM template and override per environment.
- **Key Vault:** use the same secret **names** in each environment's vault and only change the vault URL.
- **Managed identity permissions** must be granted in each environment (they do not travel with the template).
- Parameterize linked service `connectVia` references if IR names differ.

### Q73. Why do I need pre- and post-deployment scripts?

ARM deployments are **incremental**: they add and update resources but **do not delete** resources removed in Git, and they fail when updating **active triggers**. The Microsoft-provided PowerShell script (`PrePostDeploymentScript.ps1`) stops triggers before deployment and, after deployment, **removes orphaned resources** and restarts triggers. Without this, deleted pipelines linger in production and trigger updates fail.

### Q74. What are best practices for ADF DevOps?

Use small, well-named pipelines; follow naming conventions; avoid hard-coded environment values; keep secrets in Key Vault; test in a non-production factory using realistic parameters; use pull-request validation (`validate` command); version global parameters and ARM overrides in source control; document deployment ordering (linked services and IRs before pipelines); and consider a **dedicated factory per environment** and sometimes per domain to limit blast radius.

### Q75. What is the difference between debug runs and triggered runs?

A **debug run** uses the working copy (unpublished changes, in your branch), runs on the debug session settings, is interactive and shows output per activity; it does not require publishing. A **triggered run** executes the **published** definition under a trigger or manual trigger and appears in Monitor as a pipeline run. Debug runs still incur real cost and make real writes to real data stores.

---

# Part 9: Security

### Q76. How does authentication to data stores work in ADF?

Common options, from most to least recommended:
1. **Managed identity** (system-assigned or user-assigned): no secrets to store or rotate; grant the identity access on the target (for example **Storage Blob Data Contributor** on storage; a contained database user created `FROM EXTERNAL PROVIDER` in Azure SQL).
2. **Service principal** with secret or certificate (secret stored in Key Vault).
3. **Key Vault-referenced secrets** for passwords and connection strings.
4. **Access keys, SAS tokens, account keys** (avoid for long-term use; broader access and manual rotation).

### Q77. What is the factory's managed identity?

Each factory gets a **system-assigned managed identity** automatically (same name as the factory); you can also attach **user-assigned managed identities** and define **credentials** in ADF that reference them. Use these identities to authenticate to Key Vault, storage, SQL, Databricks and Functions, and grant least-privilege roles.

### Q78. How do I use Azure Key Vault with ADF?

Create a Key Vault linked service (authenticated with the factory's managed identity), grant the identity **Get and List secrets** permission (access policy) or the appropriate RBAC role (Key Vault Secrets User), and reference secrets in other linked services (select "Azure Key Vault" for the password or connection string field). Benefits: central rotation, no secrets in Git or ARM templates, and the same secret names across environments for painless deployment.

### Q79. What RBAC roles control access to ADF itself?

- **Data Factory Contributor:** create and manage factories and their child resources (also used for publishing and deployment automation).
- **Contributor/Owner:** broader subscription-level permissions.
- **Reader:** view only (can monitor).
- Custom roles for fine-grained control (for example monitor-only, or trigger-run-only).

Separate duties: developers in dev, a deployment service principal in higher environments, operations with monitor rights.

### Q80. How is network traffic secured?

- **Managed virtual network IR + managed private endpoints** for private connectivity to PaaS services.
- **Self-hosted IR** for on-premises or private-network sources (outbound 443 only).
- **Private endpoint for the factory** itself (restricts portal/API access to private networks) and disabling public network access where required.
- **Firewall rules** on data stores allowing the IR (for Azure IR, trusted-service exceptions or managed private endpoints are preferred over IP lists).
- **Trusted Microsoft services** setting on storage to allow ADF with managed identity when appropriate.

### Q81. How is data encrypted?

Data in transit uses TLS; data at rest in Azure services is encrypted by default. For the factory's own metadata you can enable a **customer-managed key (CMK)** from Key Vault, which **must be configured when the factory is empty** (before creating resources) and changed carefully, since losing the key makes the factory inaccessible. Credentials in linked services are stored encrypted by ADF (or on the SHIR node).

### Q82. How do you keep sensitive values out of logs?

Enable **Secure Input** and **Secure Output** on activities (for example Web activities carrying tokens, or Lookups returning secrets). When enabled, the activity input/output is not shown in Monitor or logged. Never place secrets in plain pipeline parameters or variables, because those are visible in run history.

### Q83. How does ADF integrate with Microsoft Purview?

Connect a Purview account to the factory to **publish lineage** for Copy and data flow activities, helping data governance, impact analysis and cataloging across Azure data services.

---

# Part 10: Monitoring, Logging and Alerting

### Q84. What can I see in the Monitor hub?

- **Pipeline runs:** status (Queued, In progress, Succeeded, Failed, Canceled), duration, run ID, triggered-by, parameters, annotations, and a drill-down into **activity runs** (input, output, error, duration, user properties).
- **Trigger runs:** which trigger fired and whether the pipeline started.
- **Integration runtimes:** status and metrics, especially for SHIR (CPU, memory, queue length, concurrent jobs).
- **Data flow debug sessions, alerts and metrics.**

Monitor retains run data for **45 days**; for longer retention, send diagnostics to **Log Analytics**, storage or Event Hubs.

### Q85. What are the rerun options?

- **Rerun** the entire pipeline (new run ID, grouped with the original).
- **Rerun from failed activity:** resumes from the point of failure, skipping already-succeeded upstream activities.
- **Rerun from selected activity:** start from a chosen activity.

Because reruns re-execute activities, your pipelines should be **idempotent** (for example truncate-then-load or upsert) to avoid duplicates.

### Q86. How do I get long-term logs and analytics?

Enable **diagnostic settings** on the factory to send logs to a **Log Analytics workspace**. Choose the **resource-specific** destination tables (`ADFPipelineRun`, `ADFActivityRun`, `ADFTriggerRun` and others) rather than the older shared `AzureDiagnostics` table, for better query performance and schema. Example KQL for failures in the last 24 hours:

```kusto
ADFPipelineRun
| where TimeGenerated > ago(24h) and Status == "Failed"
| project TimeGenerated, PipelineName, RunId, Status, FailureType, Parameters
| order by TimeGenerated desc
```

Average duration by pipeline (to spot regressions):

```kusto
ADFPipelineRun
| where Status == "Succeeded" and TimeGenerated > ago(14d)
| summarize avgMinutes = avg(datetime_diff('minute', End, Start)) by PipelineName
| order by avgMinutes desc
```

### Q87. How do I set up alerts?

Use **Azure Monitor alerts** on ADF **metrics** (for example Failed pipeline runs, Failed activity runs, Failed trigger runs, IR CPU/memory for SHIR, and elapsed-time metrics) with action groups (email, SMS, Teams via Logic App, webhook). For data-quality or business-rule failures, use the **Fail activity** to raise a failure with a meaningful message so alerts carry context. You can also alert from Log Analytics queries.

### Q88. How do I build an operational audit trail?

A frequent pattern: at pipeline start and end, call a **Stored Procedure** (or Script) activity that writes to a **logging table**: pipeline name, run ID (`@pipeline().RunId`), trigger name, start and end times, rows read/written (from the Copy output: `@activity('Copy1').output.rowsCopied`), status and error message (`@activity('Copy1').error.message`). This provides business-friendly auditing independent of the 45-day Monitor retention.

### Q89. What useful Copy activity outputs can I read?

`rowsRead`, `rowsCopied`, `dataRead`, `dataWritten` (bytes), `copyDuration`, `throughput` (KB/s), `usedDataIntegrationUnits`, `usedParallelCopies`, `executionDetails` (per-stage timing, such as queue, transfer and source/sink breakdown), and `errors`. These are essential for performance tuning and auditing.

---

# Part 11: Error Handling and Reliability Patterns

### Q90. What are retry and timeout settings?

Each activity has a **timeout** (default is long, commonly 12 hours **(version-sensitive)**; set explicitly to a realistic limit), **retry** count and **retry interval** (seconds). Use retries for **transient** errors (throttling, brief network failures) with an interval that allows recovery. Do not rely on retries for logic errors; they just repeat the failure and cost money.

### Q91. How does ADF determine a pipeline's overall success or failure?

The pipeline result is evaluated from the **leaf activities** (activities with no downstream dependents) and how failures are handled. A common surprise: if an activity fails but its **Failed** path leads to a handler that succeeds, the pipeline may still be reported as **succeeded** (the failure was "handled"), depending on the pattern. To make failures visible, end the failure path with a **Fail** activity or a deliberate failing step.

### Q92. What are the standard error-handling patterns?

1. **Try-Catch-Proceed (do-if-else):** Activity A has a **Succeeded** path to the next step and a **Failed** path to an error handler; the pipeline continues or ends cleanly.
2. **Try-Catch-Fail:** Failure path logs the error and then runs a **Fail** activity so the overall pipeline is marked **Failed** and alerts fire.
3. **Do-If-Skip-Else:** Use a **Completed** dependency for cleanup tasks that must run regardless of success (for example delete temp files, stop a resource).
4. **Common failure handler pipeline:** all failures call a shared child pipeline that logs and notifies, using Execute Pipeline with the error message passed as a parameter.

Remember that multiple dependencies on an activity are **AND**-ed, so to catch failures from several activities, route each to the handler separately.

### Q93. How do you make pipelines idempotent?

Idempotent means running twice has the same effect as once. Techniques: **truncate and reload** a staging table before loading; use **upsert/MERGE** on business keys; write to **partitioned folders by run date** and overwrite that folder; use **watermark-based** loads that advance only after success; track processed files in a control table; make downstream transformations deterministic. Idempotency makes retries and reruns safe.

### Q94. How do you prevent overlapping runs?

Set the pipeline's **Concurrency** setting (maximum concurrent runs). Extra triggered runs queue (or are cancelled depending on the configuration and trigger type). Tumbling window triggers have `maxConcurrency` and window dependencies for orderly processing.

### Q95. How do you design for transient source errors and throttling?

Add retries with longer intervals, reduce ForEach `batchCount` and copy parallelism, stagger schedules, use **exponential backoff** with Until + Wait for APIs, respect rate-limit headers, and prefer bulk endpoints. For databases, avoid peak-business-hour loads or read from replicas.

---

# Part 12: Common Design Patterns

### Q96. What is the watermark (incremental load) pattern?

Load only **new or changed rows** since the last run, using a column such as `LastModified` (or an incrementing ID) as a **watermark**.

1. **Lookup** the **old watermark** from a control table (`etl.Watermark` holding table name and last value).
2. **Lookup** the **new watermark** from the source (`SELECT MAX(LastModified) FROM dbo.Orders`).
3. **Copy** rows with `LastModified > oldWatermark AND LastModified <= newWatermark`.
4. **Stored Procedure** to update the watermark to the new value **only after the copy succeeds**.

```sql
-- Copy source query (dynamic content)
SELECT * FROM dbo.Orders
WHERE LastModified >  '@{activity('LookupOldWM').output.firstRow.WatermarkValue}'
  AND LastModified <= '@{activity('LookupNewWM').output.firstRow.NewWatermark}'
```

Pitfalls: late-arriving or backdated updates (use an overlap window and deduplicate), non-monotonic watermark columns, time zones (store UTC), and deletes (a watermark cannot see hard deletes; use soft deletes, CDC or periodic reconciliation).

### Q97. How do I implement a metadata-driven (config-driven) framework?

Instead of building one pipeline per table, drive generic pipelines from a **control table**:

```sql
CREATE TABLE etl.PipelineConfig (
  ConfigId        INT IDENTITY PRIMARY KEY,
  SourceSchema    VARCHAR(128),
  SourceTable     VARCHAR(128),
  TargetFolder    VARCHAR(400),
  LoadType        VARCHAR(20),     -- 'FULL' or 'INCREMENTAL'
  WatermarkColumn VARCHAR(128) NULL,
  IsActive        BIT DEFAULT 1,
  BatchGroup      INT
);
```

Flow: **Lookup** active rows (watch the 5,000-row limit; filter by `BatchGroup` or chunk) → **ForEach** over the rows (parallel with a sensible `batchCount`) → **Execute Pipeline** (child) with the row values as parameters → child uses **parameterized datasets** and a **dynamic source query** to copy, logs success or failure, and updates watermarks. Benefits: add a table by inserting a row, consistent logging, far less duplicated JSON. Risks: harder debugging if over-generalized; always include good logging and per-table failure isolation.

### Q98. How do I process files as they arrive?

Options: a **storage event trigger** (react to blob creation), a **schedule/tumbling trigger** plus **Get Metadata** (list files, filter by `lastModified`), or **Copy activity with "modified between" filters** for files in a time window. After processing, **archive** the file (Copy to an archive folder, then **Delete** activity), or log processed file names in a control table to avoid reprocessing. For continuous, high-volume file ingestion into Delta, consider Databricks Auto Loader instead.

### Q99. How do I copy many tables in one pipeline?

Use **Lookup** (or `INFORMATION_SCHEMA` query) to produce a list of tables, then **ForEach** with **Copy** inside using a **parameterized source dataset** (schema and table parameters) and a **parameterized sink**. Make the sink path dynamic, for example `@concat('raw/', item().SourceSchema, '/', item().SourceTable, '/', formatDateTime(utcnow(),'yyyy/MM/dd'))`.

### Q100. What is Change Data Capture (CDC) in ADF context?

CDC captures **row-level changes** (including deletes) from a source. Options: **SQL Server / Azure SQL native CDC or Change Tracking** queried through Copy or Script activities (use LSN ranges or version numbers like a watermark), the **ADF Change Data Capture resource** (a top-level, near-real-time, low-code incremental replication feature for supported sources and sinks **(version-sensitive)**), or CDC tools landing change files into the lake which ADF or Databricks then merges. CDC handles deletes and updates reliably, which watermark columns often cannot.

### Q101. How do I design a typical lakehouse ingestion with ADF and Databricks?

1. **ADF Copy** from source systems into the **raw/bronze** zone of ADLS (Parquet or raw files), parameterized and metadata-driven, with logging.
2. **Databricks notebook activity** (or Job activity) transforms bronze to silver and gold (Delta tables, merges, data quality).
3. **Stored Procedure / Web activity** refreshes downstream models or Power BI datasets.
4. **Triggers** schedule the whole parent pipeline; **alerts** and logs provide observability.

Passing values to Databricks: set **Base parameters** on the Notebook activity (read in the notebook with widgets) and return results via `dbutils.notebook.exit(...)`, readable in ADF as `@activity('Notebook1').output.runOutput`.

### Q102. How do you handle file deletion or archival after a successful copy?

Add a **Delete** activity (or Copy then Delete) on the **Succeeded** path, enabling "recursive" and logging if needed; use an **idempotent** control table so a failure between copy and delete does not cause duplicates on rerun.

### Q103. How do you handle slowly changing dimensions without data flows?

Land changes in a **staging table** with Copy, then use a **Stored Procedure** or **Script** activity running a T-SQL `MERGE` (or the Type 2 pattern with `UPDATE` then `INSERT`) inside the warehouse. This is often faster and cheaper than a data flow for warehouse-resident data.

### Q104. How do I restart a long pipeline without redoing everything?

Use **Rerun from failed activity**, design pipelines as **independent child pipelines** per stage, keep a **run-state table** so a pipeline can skip stages already completed for a given batch ID, and keep each unit small enough to rerun cheaply.

---

# Part 13: Integration with Other Azure Services

### Q105. How does ADF run Databricks code?

Use the **Databricks Notebook**, **Python** or **JAR** activities (and a Databricks **Job** activity **(version-sensitive)**). The **Databricks linked service** needs the workspace URL and authentication (access token stored in Key Vault, or Microsoft Entra ID / managed identity). Compute options: an **existing interactive cluster**, an **existing instance pool**, or a **new job cluster** created per run (cheapest per DBU, isolated, recommended for production). Pass parameters via Base parameters, read return values from `runOutput`, and set retries and timeouts on the activity.

### Q106. How does ADF work with Synapse and SQL?

ADF loads into **Azure Synapse dedicated SQL pools** using the **COPY command** or PolyBase with staged copy, and calls stored procedures for transformation. For **Azure SQL Database / Managed Instance**, use Copy (bulk insert), Script/Stored Procedure activities, and managed identity authentication. Synapse workspaces include the same pipeline engine, so skills transfer.

### Q107. How does ADF integrate with Azure Functions and Logic Apps?

- **Azure Function activity:** call a function (HTTP trigger) with a method, headers and body, and use the response; subject to the HTTP timeout limit (about 230 seconds through the front end), so use durable functions with polling for long tasks.
- **Logic Apps:** called through a Web activity (HTTP trigger), commonly for email/Teams notifications and workflow integrations.

### Q108. How do I refresh a Power BI dataset from ADF?

Use a **Web activity** calling the Power BI REST API (`POST .../datasets/{id}/refreshes`) with an authenticated service principal or managed identity that has permission on the workspace, optionally polling the refresh status using Until + Web. This ties dataset refresh to the completion of upstream loads instead of fixed schedules.

### Q109. How does ADF relate to Microsoft Fabric?

**Microsoft Fabric Data Factory** is the data integration experience inside Fabric, with Fabric pipelines (similar concepts to ADF pipelines) and **Dataflow Gen2** (Power Query based). ADF remains a supported, widely used service; Microsoft provides paths to **bring ADF factories into Fabric** or migrate pipelines, but feature parity differs (for example the Self-hosted IR concept maps to the on-premises data gateway in Fabric, and some activities or connectors differ) **(version-sensitive; check current migration guidance)**. For a new greenfield solution already standardized on Fabric, Fabric pipelines are the natural choice; for existing Azure-based estates, ADF continues to work.

### Q110. How do I load data into Snowflake, SAP or other third-party systems?

Use the dedicated connectors (Snowflake with staged copy through Blob/ADLS; SAP connectors such as SAP table, SAP BW, SAP HANA and SAP CDC through the self-hosted IR **(version-sensitive)**). Check connector docs for supported authentication, staging, and the IR required. For connectors that do not exist, use REST/ODBC, a custom Azure Function, or Azure Batch custom activity.

---

# Part 14: Performance Tuning and Cost Optimization

### Q111. How do I diagnose a slow Copy activity?

Look at **Copy activity output** → `executionDetails` (queue time, transfer time, and the source and sink breakdown) and `throughput`. Identify the **bottleneck**: source read, network/IR, or sink write.
- **Source-bound:** add partitions or parallel reads; query only needed columns; avoid reading during contention; use indexed partition columns.
- **Sink-bound:** use staged copy and bulk loaders for warehouses, increase write batch size for SQL, scale the sink (DTUs, DWUs, vCores) temporarily, avoid indexes/triggers during loads, write larger Parquet files.
- **IR/network-bound:** increase DIUs (Azure IR); scale out SHIR nodes; move the IR closer to the data; avoid cross-region copies; use ExpressRoute or VPN throughput.
- **Format-bound:** prefer binary copy for file moves; choose Parquet over CSV.

### Q112. What are the general performance best practices?

- Make pipelines **parallel where independent** (ForEach with sensible batch counts, parallel branches).
- Use **partitioned reads** for big tables; **bulk loaders** for warehouses.
- Keep **IR and data stores in the same region**.
- Avoid many tiny files; target large, efficient files.
- Select only needed columns; push filters to the source.
- For data flows: use TTL for chained flows, appropriate partitioning, broadcast joins, and minimal sorts.
- Avoid unnecessary activities: each activity run has orchestration overhead and cost, so don't build chains of hundreds of tiny activities where one set-based statement would do.

### Q113. How is ADF priced?

The main meters **(version-sensitive; check the pricing page for rates)**:
1. **Orchestration:** per 1,000 **activity runs** (pipeline activities on Azure IR and on Self-hosted IR are priced separately), including trigger runs and debug runs.
2. **Data movement:** **DIU-hours** for Copy on the Azure IR (billed by duration with minute-level rounding), or an hourly rate for Copy on SHIR.
3. **Data flow:** **vCore-hours** of cluster time (General Purpose or Memory Optimized), including cluster start and TTL idle time and debug sessions.
4. **Azure-SSIS IR:** per-hour VM and licensing costs while running.
5. **Operations:** read/write and monitoring entity operations (small).
6. **Other:** managed VNet and private endpoints, data egress and the costs of the services ADF calls (Databricks DBUs and VMs, SQL compute, storage transactions).

### Q114. How do I reduce ADF cost?

- Right-size **DIUs** (do not over-provision); leave on Auto unless measurements show a benefit.
- Avoid **small, chatty pipelines** that create excessive activity runs; use ForEach with batching and set-based operations.
- Use **binary copy** and compressed Parquet to reduce I/O.
- Set **TTL** deliberately for data flows; turn off **debug** sessions; use data flows only where they pay off.
- **Stop Azure-SSIS IR** when idle (schedule start/stop).
- Use **job clusters** for Databricks activities (cheaper than all-purpose).
- Reduce **trigger frequency** to what the business needs.
- Co-locate resources to avoid egress; monitor cost with Cost Management tags and the consumption report in Monitor.

### Q115. What file size and layout should ADF produce for the lake?

Target roughly **100 MB to 1 GB** per Parquet file, partition folders by low-cardinality columns such as date (`year=2026/month=10/day=19`), and avoid partitioning on high-cardinality columns. Many tiny files hurt every downstream engine (Spark, Synapse serverless, Databricks) due to metadata and I/O overhead.

---

# Part 15: Limits and Quotas

### Q116. What are the key limits to remember?

Limits change and some can be raised through support requests **(version-sensitive; always confirm in the official "Azure subscription and service limits" documentation)**. Commonly cited values:

| Item | Typical limit |
|---|---|
| Lookup activity output | 5,000 rows and about 4 MB |
| Get Metadata child items output | About 4 MB |
| ForEach batch count | Default 20, maximum 50 |
| ForEach items | About 100,000 |
| Activities per pipeline | Around 120 (including inner activities of containers) |
| Parameters per pipeline | 50 |
| Web activity response timeout | About 1 minute |
| Until activity timeout | Up to 7 days |
| Self-hosted IR nodes per IR | 4 |
| DIU range for Copy | 2 to 256 |
| Monitor run history retention | 45 days |
| Concurrent pipeline/activity runs | Subscription and region-level limits apply |

Design around limits: chunk big Lookups, use child pipelines to stay under activity counts, and use pagination or Copy for large lists.

---

# Part 16: Troubleshooting: Common Errors and Causes

### Q117. The Self-hosted IR shows "Offline" or "Limited".

Check the IR service is running on the machine, outbound access on port 443 to ADF/Azure Relay endpoints, proxy settings, system clock accuracy, expired or re-generated authentication key, node updates, and that the machine has enough CPU/memory. For "Limited", a node may be unable to reach the cloud service or one node is down in a multi-node setup.

### Q118. "Unable to connect" or login errors to a data store.

Verify the **IR can reach the source** (firewall, private endpoint approval, public access disabled, correct IR selected in `connectVia`). For Azure SQL with managed identity, confirm the contained user was created and given roles. For storage with managed identity, confirm the RBAC role (data-plane role like Storage Blob Data Contributor, not just Contributor) has propagated. For Key Vault, confirm the access policy or role. Test the linked service connection button, then inspect the full error.

### Q119. Copy fails with a mapping or conversion error.

Compare source and sink types (precision, nullability, string length truncation). Common messages: string too long for the sink column, decimal precision overflow, invalid date formats, or null into NOT NULL. Fix by casting in the source query, widening sink columns, adjusting mapping, or enabling fault tolerance to log bad rows for analysis.

### Q120. Lookup fails because output is too large.

You exceeded the row or size limit. Select fewer columns, filter or chunk the query (for example `TOP`/`OFFSET`), use a **Copy** to a file and read it differently, or restructure so Lookup returns only keys and the child pipeline fetches the details.

### Q121. ForEach produces inconsistent variable values.

Parallel iterations overwrite a shared pipeline variable. Make the ForEach sequential, or move per-item logic into a child pipeline where parameters are isolated, or avoid variables and use activity outputs.

### Q122. Pipeline "succeeded" although something failed.

A failed activity had a **Failed**-dependency path to a handler that succeeded, so the overall pipeline succeeded. Add a **Fail** activity at the end of the failure path, or evaluate the logic of leaf activities.

### Q123. Dynamic content error: "The expression ... cannot be evaluated".

Common causes: referencing a property that does not exist (for example `firstRow` when the Lookup returned zero rows), wrong activity name (activity names are case-sensitive in expressions), type mismatches (array vs string), invalid quotes, or using an activity output from a branch that did not run. Add `if(empty(...))` guards and inspect the actual activity output JSON.

### Q124. Storage event trigger does not fire.

Check: **Microsoft.EventGrid** provider registered; the trigger is started and published; filters (`blobPathBeginsWith`, `blobPathEndsWith`) match the path format (including the container prefix `/container/blobs/...`); storage account type supports events; permissions for the publishing user; and the Event Grid subscription health in the storage account.

### Q125. Data flow is slow to start or slow overall.

Cold start (3 to 5 minutes) is normal without TTL. For slowness: check partitioning, skew on join keys, large sorts, cross joins, too small a cluster, or sinks that write single-row inserts. Use Monitor's data flow details (stage times, rows per partition) to find the bottleneck.

### Q126. Deployment fails or behaves unexpectedly in CI/CD.

Typical causes: triggers not stopped before deployment; missing parameter overrides (the target points to dev resources); linked services referencing secrets absent in the target Key Vault; managed identity not granted access in the target environment; deleted objects remaining because the cleanup script was not run; ARM template size/parameter limits (consider linked templates for very large factories).

### Q127. Copy to Parquet fails with column-name errors.

Parquet sinks in Copy are sensitive to column names containing spaces or special characters. Alias columns in the source query, use mapping to rename, or use a data flow that cleans names.

### Q128. REST API copy returns partial data or 429 errors.

Pagination rules are misconfigured or the API is throttling. Review pagination configuration, lower parallelism, add retries with longer intervals, and check API quotas and headers.

---

# Part 17: Scenario-Based Interview Questions

### Q129. Design an incremental load from an on-premises SQL Server to a data lake and then to a warehouse.

Install a **Self-hosted IR** (two nodes for HA) near the SQL Server. Build a **metadata-driven** pipeline: a Lookup reads active tables and watermark columns from a control table; a ForEach (parallel, moderate batchCount) calls a child pipeline per table. The child uses watermark logic (old and new watermark lookups), a **Copy** with a dynamic query writing **Parquet** to `raw/<table>/<yyyy>/<MM>/<dd>/`, then updates the watermark only after success. Log each table's run to an audit table. A downstream **Databricks** or **Synapse** step merges raw data into curated tables. Secrets live in **Key Vault**; the factory uses its **managed identity** for the lake. A **tumbling window** or schedule trigger runs the parent; alerts fire on failures.

### Q130. A nightly pipeline sometimes duplicates data after a rerun. What went wrong and how do you fix it?

Likely the load is not idempotent: it appends on every rerun. Fix by truncating the staging/target partition before the load, using upsert/MERGE on business keys, writing to a run-date folder with overwrite semantics, advancing watermarks only on success, and tracking processed batch IDs.

### Q131. How would you migrate hundreds of SSIS packages to Azure?

Use the **Azure-SSIS IR** for lift and shift: provision the IR (choose node size and count), deploy projects to SSISDB (Azure SQL MI or SQL Database) or file storage, run them with **Execute SSIS Package** activities, join a VNet for on-premises access, and schedule start/stop to control cost. Over time, modernize high-value packages into native ADF pipelines, Databricks or SQL-based ELT.

### Q132. A pipeline needs to wait for a vendor file before processing. How do you build it?

Use a **storage event trigger** with path filtering and a trigger/marker file, or a schedule trigger plus a **Validation** activity polling for the file with a timeout. Add a **Fail** activity or alert when the timeout expires, process with idempotent logic, then archive the file.

### Q133. How do you secure a hybrid ADF solution end to end?

Managed identity and Key Vault for secrets, **private endpoints** (managed VNet IR) for PaaS targets with public access disabled, **Self-hosted IR** for on-premises (outbound-only), least-privilege RBAC (separate dev/deploy/monitor roles), secure input/output on sensitive activities, diagnostic logs to Log Analytics, Purview for lineage, CMK if required, and separate factories per environment with deployments only through CI/CD.

### Q134. A copy from Azure SQL to ADLS is slower than expected. What do you check?

Read `executionDetails` and throughput, identify source or sink bound, enable **partition option (dynamic range)** on an indexed column, ensure IR and storage are in the same region, check source DB throttling (DTU/vCore limits), choose Parquet, tune DIUs and parallel copies, select only needed columns, and check for LOB columns.

### Q135. How do you pass data between activities and pipelines?

Within a pipeline: activity outputs (`@activity('x').output...`), variables, parameters. Between pipelines: Execute Pipeline parameters and return values; via storage or a control table for larger data. To external systems: Web/Webhook activities, and to Databricks: Base parameters and `runOutput`.

### Q136. What is the difference between a Linked Service parameter and a Dataset parameter, and why use both?

Linked service parameters make **connection** details dynamic (server, database name, endpoint), allowing one linked service to serve many databases. Dataset parameters make **object** details dynamic (schema, table, folder, file name). Used together they yield fully generic, reusable components for metadata-driven frameworks.

### Q137. How do you handle schema changes in the source without breaking pipelines?

For Copy to schema-flexible sinks (Parquet/Delta landing), omit explicit mapping so new columns flow through; land raw data first; use data flows with **schema drift** enabled for generic transformations; add schema validation (Assert or Script checks) and alerts for breaking changes; version raw layer schemas and evolve downstream models deliberately.

### Q138. How would you orchestrate dependencies between pipelines owned by different teams?

Use **tumbling window trigger dependencies** where windowed data readiness matters; a parent **orchestrator pipeline** with Execute Pipeline; event-based signaling (files or Event Grid custom events); or a control table with readiness flags polled via Until + Lookup. Define clear SLAs and publish run status for upstream/downstream visibility.

### Q139. How do you test ADF pipelines?

Use **debug runs** with small data sets and parameters; non-production factories with realistic volume tests; unit-style tests by isolating logic in stored procedures or notebooks that can be tested independently; validation queries (row counts, checksums) as pipeline steps; PR validation using the utilities package; and deployment to test before prod.

### Q140. When would you choose a tumbling window over a schedule trigger?

When each time slice must be processed **exactly once, in order, and be re-runnable** (for example hourly incremental loads with backfill), when you need **dependencies** between pipelines or self-dependency, built-in **retry** per window, or controlled **concurrency**. Choose a schedule trigger for simple, recurring, non-time-sliced jobs or when one trigger must start several pipelines.

---

# Part 18: Glossary

- **Activity:** one step in a pipeline.
- **ADF Studio:** the web authoring and monitoring interface.
- **Annotation:** label on resources used to filter and organize.
- **ARM template:** Azure Resource Manager JSON used to deploy ADF resources across environments.
- **Azure IR:** Microsoft-managed Integration Runtime (serverless).
- **Azure-SSIS IR:** managed cluster for running SSIS packages.
- **Backfill:** running a trigger for past time windows (tumbling windows).
- **Collaboration branch:** Git branch (usually main) where reviewed changes merge and from which publishing occurs.
- **Control table:** database table holding configuration or state (tables to load, watermarks, run logs).
- **Copy activity:** the data movement activity.
- **Data flow (Mapping Data Flow):** visual Spark-based transformation.
- **Dataset:** named reference to data within a linked service.
- **DIU:** Data Integration Unit, the power measure of an Azure IR copy.
- **Dynamic content:** run-time expressions in fields.
- **Flowlet:** reusable group of data flow transformations.
- **Global parameter:** factory-wide constant.
- **Idempotent:** safe to run multiple times with the same result.
- **Integration Runtime (IR):** compute infrastructure for ADF execution.
- **Linked service:** connection definition to a data store or compute.
- **Managed identity:** Entra ID identity automatically managed for the factory.
- **Managed private endpoint:** private link created by ADF in a managed VNet.
- **Metadata-driven pipeline:** generic pipeline controlled by configuration data.
- **Parameter:** immutable run-time input.
- **Pipeline:** graph of activities forming a unit of work.
- **Publish branch:** Git branch (adf_publish) holding generated ARM templates.
- **Schema drift:** unplanned changes in source columns.
- **Self-hosted IR (SHIR):** IR installed on your own machine for private/on-premises access.
- **Staged copy:** copy routed through interim storage for faster loading.
- **Tumbling window trigger:** fixed, contiguous, non-overlapping time-window trigger with state.
- **TTL:** time-to-live for keeping data flow clusters warm.
- **Variable:** mutable value within a pipeline run.
- **Watermark:** stored high-water mark used for incremental loads.

---

# Part 19: One-Page Cheat Sheet

**Mental model:** Trigger → Pipeline → Activities → Datasets → Linked Services → Integration Runtime does the work. Data flows through the **IR**, not the ADF service.

**IR choice:** Azure IR for cloud-to-cloud; Managed VNet IR for private PaaS access; Self-hosted for on-premises (outbound 443 only, up to 4 nodes); Azure-SSIS for SSIS lift and shift.

**Copy tuning:** find the bottleneck (source, network, sink); partition database reads (dynamic range on an indexed column); staged copy plus COPY/PolyBase for Synapse and Snowflake; Parquet over CSV; large files (100 MB to 1 GB); DIUs on Auto unless measured; select only needed columns.

**Control flow facts:** Lookup max 5,000 rows / 4 MB; ForEach default 20 parallel, max 50; never Set Variable inside a parallel ForEach; no direct nested ForEach (use a child pipeline); Web activity about 1 minute timeout (use Webhook or polling for long work); multiple dependencies are AND.

**Triggers:** Schedule (many-to-many, no backfill); Tumbling window (1:1, backfill, windows, retry, dependencies); Storage event (Event Grid, needs provider registration, use idempotent design); always publish and start triggers.

**Patterns:** Watermark incremental (advance watermark only on success); metadata-driven (control table, Lookup, ForEach, Execute Pipeline, parameterized datasets); Try-Catch-Fail (log then Fail activity); idempotent loads (truncate/overwrite/upsert).

**CI/CD:** Git on dev factory only; collaboration branch main; Publish creates ARM in adf_publish (or use the utilities npm package); override parameters per environment; pre-/post-deployment script to stop triggers, delete removed objects and restart triggers.

**Security:** Managed identity first; secrets in Key Vault; private endpoints and managed VNet; least-privilege RBAC; secure input/output; CMK only on an empty factory.

**Monitoring:** Monitor keeps 45 days; use diagnostic settings to Log Analytics (resource-specific tables); alert on metrics; log business audit info to a table (rows copied, run ID).

**Data types and sizes:** Int16 2 bytes, Int32 4, Int64 8, Single 4, Double 8, Decimal 16 (interim), Guid 16; SQL `decimal` 5/9/13/17 bytes by precision; use decimals for money, dates as date types, narrow row widths, watch LOB columns.

---

**Final note.** Azure Data Factory evolves continuously, and its positioning relative to Microsoft Fabric Data Factory is changing. Limits, defaults, preview features and pricing marked **(version-sensitive)** should be verified in the current Microsoft documentation before an interview, exam or production design.
