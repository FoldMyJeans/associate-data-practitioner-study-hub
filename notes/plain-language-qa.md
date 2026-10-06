# Questions I asked while studying

These are the "wait, but why" questions I had while studying for the Associate Data Practitioner exam. I came from BI and was new to Google Cloud, so most of them are me trying to map new products onto things I already knew. Each one has the plain answer that made it click.

## The big picture

### I come from BI. Is knowing BigQuery enough?

No. BigQuery covers the store and query end only. The exam guide covers the whole pipeline. These are the gaps a BI background usually has:

- **Ingest:** Pub/Sub for streaming, Cloud Storage as the landing place for batch files.
- **Transform tools:** Dataflow, Managed Service for Apache Spark (used to be Dataproc), and Cloud Data Fusion, not only BigQuery SQL.
- **Orchestration:** Managed Service for Apache Airflow (used to be Cloud Composer).
- **Storage choice:** when Bigtable beats BigQuery, when Spanner beats Cloud SQL.
- **IAM:** access at the dataset, table, and column level, which BI tools usually hide from you.

BigQuery SQL transfers directly and is the biggest chunk, but the exam guide expects you to reason about the whole pipeline.

### What are the layers of a data platform, and what do I already know them as?

All of it together is a cloud data platform. Each layer has one job. The Microsoft column is the stack I already knew.

| Layer | Job | Google Cloud | Microsoft |
|---|---|---|---|
| Access | who can do what | IAM | Entra ID and RBAC |
| Storage | raw files (the data lake) | Cloud Storage | Data Lake, Blob Storage |
| Ingestion | bring data in | Pub/Sub, Datastream, BigQuery Data Transfer Service | Event Hubs, Data Factory |
| Processing | clean, join, transform | Dataflow, Managed Service for Apache Spark, Dataform | Data Factory, Databricks, dataflows |
| Warehouse | tables and SQL | BigQuery | Synapse, Fabric Warehouse |
| Orchestration | schedule and chain jobs | Managed Service for Apache Airflow | Data Factory pipelines, Power Automate |
| BI | dashboards | Data Studio (used to be Looker Studio), Looker | Power BI |
| Tooling | command line | Cloud Shell with `bq` and `gcloud` | Azure CLI, PowerShell |

The usual shape: data lands in Cloud Storage, Dataflow cleans it into BigQuery, Airflow runs that on a schedule, and Data Studio reads the final tables. A company typically runs separate dev, test, and prod projects. You write and fix in dev, then promote finished pipelines to prod.

## Storage and databases

### Why does Google have so many database products?

Three things force the split. No single engine wins on all of them.

1. **Data shape.** Rows and columns (SQL), documents (Firestore), or wide-column key-value (Bigtable).
2. **Workload pattern.** Transactional means many small, fast reads and writes. Analytical means a few huge scans and aggregations. An engine tuned for millisecond single-row writes (Bigtable) is bad at scanning petabytes. One tuned for scanning petabytes (BigQuery) is bad at millisecond writes.
3. **Scale and consistency.** Cloud SQL is simple but regional. Spanner adds global scale with strong consistency, at higher complexity.

The decision tree is not product padding. Each requirement removes options until one fits.

### When do I use Cloud SQL, and when BigQuery?

**Cloud SQL is the cash register. BigQuery is the accountant.**

- **Cloud SQL:** an app saves or changes one record at a time. A customer places an order (insert a row, stock minus 1). A manager approves a leave request (update one row). A rider unlocks a bike (save one trip).
- **BigQuery:** people ask questions across many rows. "Total sales by region last year." "Busiest station across 83 million trips."

Real systems have both. The app writes to Cloud SQL, the data is copied to BigQuery (for example with Datastream), and the dashboard reads BigQuery.

In Power BI terms: Cloud SQL is the SQL Server database behind an app, and BigQuery is the dataset behind the report. You pay for a Cloud SQL instance while it runs, even when it is idle.

### Cloud SQL, AlloyDB, or Spanner?

All three are relational databases for apps.

| Product | Pick it when |
|---|---|
| Cloud SQL | The default choice for a normal app database |
| AlloyDB | PostgreSQL only. The CPU is maxed out, reporting slows the app, or you need analytics on live data |
| Spanner | Global scale, writes in many regions, 99.999% availability |

### What is Bigtable for?

Bigtable is a NoSQL wide-column database. It is built for a constant stream of new timestamped rows (time-series writes at massive volume) and for millisecond lookups by row key. The open source equivalents are HBase and Cassandra. It is not for SQL analysis and not for app documents.

### What is the difference between a bucket and a database?

**A bucket holds files. A database holds rows.**

| | Bucket | Database |
|---|---|---|
| Holds | files: CSV, photos, PDFs, anything | tables with columns and rows |
| You ask it | "give me `invoice.csv`" | "which invoices are over $500?" |
| Can it filter? | no, you get the whole file | yes, that is the point |

A bucket does not know what is inside its files. It is a cloud hard drive. The whole trick of a Lakehouse table (used to be BigLake) is making bucket files behave like database rows, which is why it needs a connection: a service account that reads the bucket on behalf of BigQuery.

### Is SQL enough for a data warehouse but not for a data lake?

Yes.

- **Warehouse (BigQuery):** the schema is already applied and someone already did the transform. SQL is enough.
- **Lake (Cloud Storage):** raw files in their native formats, such as JSON blobs, images, audio, PDFs, and CSVs with no schema. SQL cannot touch most of it. Exploring raw data, building features, and training models need Python and ML tooling: pandas, TensorFlow, Agent Platform (used to be Vertex AI), or Spark on Managed Service for Apache Spark (used to be Dataproc).

This is also why a lake needs discovery, governance, and metadata tooling. With no schema, you need extra tools just to know what is in there.

One nuance: BigQuery is not purely structured. Object tables blur the line (next question).

### What is a BigQuery object table, concretely?

Example: an online shop where vendors upload product photos to a Cloud Storage bucket, named like `sku_48213_1.jpg`. There are too many to review by hand.

1. **Cloud Storage** holds the raw images (unstructured data, the lake).
2. A **BigQuery object table** points at the bucket. BigQuery creates one row per image with metadata such as the URI, size, and content type. It stores no pixels.
3. A model called from BigQuery ML with `ML.PREDICT` (for example a remote model hosted on Agent Platform) classifies each image and returns labels like `blurry: 0.92` as normal columns.
4. You **join** that to the structured `products` table on the SKU taken from the file name.
5. The result is a "photo quality" column you can query. QA filters for bad SKUs before the listings go live.

BigQuery becomes the catalog and pointer layer for lake files. It is not an image processor.

### How do I permanently change or delete data in old Iceberg files?

Apache Iceberg is an open table format for files in a data lake.

**If you can read but not write** (someone else's bucket): run `CREATE TABLE ... AS SELECT` into your own project. The originals stay. You never get to delete data you do not control.

**If it is your own Iceberg table**, there are two separate operations that people confuse:

1. **Compaction** merges many small files into fewer big ones. It is housekeeping and improves query speed. The old files still exist and time travel still works.
2. **`expire_snapshots`** is the one that actually deletes. It drops snapshots older than a cut-off and removes data files that nothing points at anymore.

The catch: expiring snapshots ends time travel for that period, and you can no longer undo a bad write from before the cut-off. People compact and are then surprised that storage cost does not drop. Compaction is housekeeping. Expiry is deletion.

## Getting data in

### There are two ways to get a file from a website into BigQuery. Is the result the same?

Yes, same table. The route is different.

| | Through a bucket | Through the Cloud Shell disk |
|---|---|---|
| Steps | download, upload to a bucket, create the table from the bucket | `wget` and `unzip` in Cloud Shell, then `bq load` |
| File afterwards | kept in the bucket | sits on the Cloud Shell disk, and in a temporary practice project it is gone when the project ends |
| Size | practically unlimited | 5 GB |
| Run on a schedule | yes | no, typed by hand |
| Use for | real pipelines | quick one-off tests |

In real work, files land in a **bucket first** (the lake) and load from there into BigQuery (the warehouse). Cloud Shell is a borrowed library laptop: fine for doing work, not the place to keep files.

### A website's live clicks and logs need to reach BigQuery. Is that Datastream?

No, that is Pub/Sub. The trigger is the difference:

- **A row changed in a database:** Datastream.
- **An app emitted an event:** Pub/Sub.

Clicks, page views, and app logs never touch a database. The web server just emits them. The path is site, then Pub/Sub, then Dataflow, then BigQuery.

| Data | Lives where | Tool |
|---|---|---|
| Someone clicked "Add to cart" | nowhere, it is emitted | Pub/Sub |
| The order row now says `shipped` | PostgreSQL | Datastream |

A real Datastream example on the same site: orders live in Cloud SQL for PostgreSQL. Finance wants a near-live revenue dashboard, but nobody will let a BI tool run `SELECT *` against production every 15 minutes. Datastream reads the PostgreSQL change log and mirrors the order tables into BigQuery. Checkout stays fast and BigQuery stays close to live.

### Why would four different databases ever land in one BigQuery dataset?

Acquisitions. A company buys three others over five years. Each one already runs its own system: SQL Server, Oracle, MySQL, PostgreSQL. Nobody is going to migrate all four onto one system soon, because each business has to keep running. But the CFO wants one revenue number for the group, every day.

So you set up one Datastream stream per source, all landing in the same BigQuery dataset. Datastream maps each source's data types to a common set, which is what makes the four feeds compatible on arrival. One query unions the four `orders` tables and one dashboard reads it. The alternative was four reports that finance adds up in a spreadsheet.

### Should I clean each source first, or append first and then clean?

**Clean first, then append.** Raw sources cannot be appended as they are. The column names differ and the currencies differ, so the union would fail or lie.

But the per-source code stays **thin**. It only brings each source to a common shape: rename, cast, convert. All the real business logic (margin, year over year, dedupe) is written **once**, after the union.

```
4 sources -> raw.orders_a / _b / _c / _d -> stg.* (conform) -> curated.orders_all -> dashboard
                                              ^
                                         ref.fx_rates
```

Two details that matter:

- Convert currency with the rate of **that order's date**, not today's rate. Otherwise last year's revenue changes every morning.
- Add a `company` column in the union, or you lose track of which source a row came from.

The rule: per-source code does only what is different about that source. Anything common is written once, downstream. It is the same instinct as Power Query: conform in the queries, model once.

## ETL, ELT, and the tools

### ETL vs ELT: is there a real difference?

Yes. ETL means extract, transform, load. ELT means extract, load, transform. Power Query is ETL: it transforms before the data lands in the model.

Same task, two shapes. Take 500 million help desk tickets and build a summary by department:

```
ETL:  source -> a machine that cleans and aggregates -> warehouse gets 340 clean rows
ELT:  source -> warehouse gets all 500M raw rows -> SQL cleans -> 340 rows
```

| | ETL | ELT |
|---|---|---|
| Who does the compute | a machine you own and size | the warehouse |
| What lands | only clean data | everything, raw |
| Bug in your logic | re-extract from the source | re-run SQL on data you already have |
| Scale limit | that machine | the warehouse (huge) |

The replay row is the practical one. If the department mapping was wrong for six months, ELT still has the raw rows, so you fix the SQL and re-run.

ELT is the default because BigQuery is serverless and elastic. Use ETL when:

- sensitive fields must be masked or dropped **before** they land
- the target has a size limit (for example the 1 GB model cap in Power BI Pro)
- the volume is not worth storing raw
- the target is not a warehouse (an app or an API)

The test I use: where does the compute happen? Inside the warehouse is ELT. Somewhere else first is ETL.

### For dashboards on a huge table, do I build summary tables or query on demand?

**Build tables.** The cost math decides it, because BigQuery on-demand pricing bills by bytes scanned.

- **Query on demand:** the dashboard scans 1 PB. With 50 analysts and 10 loads a day each, that is 500 PB scanned daily. Not affordable, and minutes per load.
- **Pre-aggregated:** one scheduled job scans 1 PB once at 2 am and writes a 200 MB summary. Dashboards scan 200 MB, which is nearly free and instant.

```
raw watch events (trillions of rows)
   -> nightly scheduled job
daily_video_metrics (one row per video per day)
   -> dashboards
```

Add **incremental processing**: append only yesterday's partition. Re-aggregating five years of history every night is the rookie mistake.

### What is Apache Spark, and who uses it?

**Spark** is an engine for processing huge datasets with **code** (Python or Scala) instead of SQL. It splits the work across many machines. **PySpark** is the Python interface. **Managed Service for Apache Spark (used to be Dataproc)** is Google running Spark for you.

The split: SQL answers questions you can express as SQL. Spark handles what SQL cannot do well, such as ML, parsing messy text, custom row logic, and images or audio. Analysts mostly write SQL, data scientists often write Spark.

It shows up next to Lakehouse tables (used to be BigLake) because the same Parquet files in Cloud Storage can be read by both engines:

```
Parquet files in Cloud Storage
   |-- BigQuery  <- analysts, SQL
   |-- Spark     <- data scientists, Python
```

One copy, two tools. A Spark BigQuery connector also lets Spark read BigQuery tables directly.

### Dataprep vs Dataform: are they related?

No. They are two separate products that only share the "Data" prefix.

| | Dataprep | Dataform |
|---|---|---|
| What you do | click through a messy dataset in a visual grid and fix it step by step | write `.sqlx` files in a git repo |
| Saved as | a **recipe** (a list of wrangling steps) | SQL code with `${ref()}` dependencies |
| Runs where | on Dataflow, behind the scenes | inside BigQuery |
| Pattern | ETL (cleans before landing) | ELT (transforms after landing) |
| Who it is for | an analyst, no code | someone who writes SQL |
| Power BI analogy | the Power Query editor, where each click becomes an Applied Step | a chain of SQL views that reference each other |

Scenario: a vendor sends a CSV with dates in three formats. Someone who does not write code fixes it by clicking, saves the recipe, and runs it on each new file. That is Dataprep. The clean data is already in BigQuery and you need a daily summary table built from it. That is Dataform.

Keyword triggers: "recipes", "wrangling", "no-code", and "visual" point to Dataprep. "SQL workflow", "dependencies", "assertions", and "inside BigQuery" point to Dataform.

### What is Cloud Data Fusion?

A **visual, drag-and-drop pipeline builder**. You place boxes on a canvas: sources (databases, SaaS apps, files, other clouds), transforms, and sinks (BigQuery, Cloud Storage). Many connectors come ready-made as plugins.

- **Built on CDAP**, an open source project from Cask Data, a company Google acquired in 2018.
- **Runs the pipeline on Managed Service for Apache Spark.** You draw, it generates the Spark job.
- **Not serverless.** You create and pay for a Data Fusion instance that hosts the designer.
- Its strength is **hybrid and multicloud** integration, with many systems wired together.

The closest things I knew were Azure Data Factory and SSIS.

Compared with Dataprep: both are visual. Dataprep cleans **one dataset** with a recipe. Data Fusion **connects many systems** in a pipeline.

### Which tools are managed open source?

The pattern: the open source project is the engine, and Google runs the servers, patching, and scaling for you.

| Google service | Open source underneath | Relationship |
|---|---|---|
| Managed Service for Apache Spark | Apache Hadoop, Spark | hosted clusters running the real thing |
| Managed Service for Apache Airflow (used to be Cloud Composer) | Apache Airflow | hosted Airflow. Your DAG files are plain Airflow Python |
| Cloud Data Fusion | CDAP | managed CDAP with a Google UI and plugins |
| Dataflow | Apache Beam | Google built it, then open sourced the programming model as Beam. Beam is the SDK you write, Dataflow is one place to run it |
| Dataform | Dataform core | Google acquired the company. The SQLX compiler is open source, the managed service is Google's |
| Dataprep | none | a partner product (Trifacta, now Alteryx) |
| BigQuery, Pub/Sub, Bigtable, Datastream | none | Google proprietary |

Why it matters: "existing code" questions.

- Existing Spark or Hadoop jobs, move with minimal changes: Managed Service for Apache Spark.
- Existing Airflow DAGs: Managed Service for Apache Airflow.
- One portable pipeline for batch and streaming: Dataflow (Beam).
- Existing CDAP pipelines, hybrid: Data Fusion.

### Is masking personal data all Dataflow is for?

No, that is just the easiest use case to explain. Dataflow is managed Apache Beam: the same pipeline code runs over a fixed batch or an endless stream, and it scales by itself.

One story covers it. A coffee chain where every card reader sends a message on each sale:

1. **Streaming.** The dashboard updates as sales happen, not overnight.
2. **Windowing.** "Cups sold in the last 5 minutes." The stream never ends, so Dataflow chops it into time buckets.
3. **Enrich in flight.** The reader sends `shop_id: 17`. Dataflow adds the city as the message passes.
4. **Masking and filtering.** Strip the card number mid-flight. The stored copy never had it.
5. **Non-SQL logic.** Call a fraud model per sale and add a risk flag.
6. **Fan-out.** Read the message once, send it to the warehouse, the points app, and the alert system.
7. **Templates.** "Pub/Sub to BigQuery" is already written. Fill in a form, no code.
8. **Late data.** A shop's internet drops for seven minutes and the sales arrive all at once. Dataflow reads each sale's own timestamp and files it under the right minute.

Rule of thumb: if the answer could be a SQL query in BigQuery, it is not Dataflow. Reach for Dataflow when it is streaming, when the logic cannot be written in SQL, or when data must change before it lands.

### Is Dataflow a form? Where does the code live?

Dataflow is a **processing engine**. The form is only the easy front door. There are three ways in:

1. A **template**: fill in a form, Google wrote the logic.
2. A **template plus your own function**.
3. **Your own pipeline**, written in Java or Python with Apache Beam.

You never type code into the form. The form holds a **pointer**. For a template function you write a small JavaScript function, save it as a file, upload it to a Cloud Storage bucket, and give the form the bucket path and the function name. Dataflow fetches it at run time. For a full pipeline, the code lives in your repo. Running it stages a package to a bucket and hands it to Dataflow, because the workers cannot read your laptop.

The bucket is where code goes to be picked up. That idea keeps coming back in Google Cloud.

Naming trap: Google's Dataflow has nothing to do with a Power BI dataflow.

### Is Dataform a form?

No, it is **code**. The name is just the company Google acquired. You write `.sqlx` files in a git repo, edit them in a web editor, and Dataform works out the run order and executes it in BigQuery. A Dataform repo can also be linked to an external git host, so you can edit locally and review changes through pull requests.

Three unrelated products that share letters:

| Product | What it is |
|---|---|
| Dataform | SQL files in git, transforms inside BigQuery |
| Dataflow | Beam engine, batch and streaming, real code |
| Datastream | change data capture pipe, from a database into BigQuery |

### What is a .sqlx file, in plain terms?

A SQL file with a small label on top.

File 1 says "make a view" and then lists four fruits. File 2 says "make a table" and then adds up the fruits from file 1. The label (the `config` block) decides view or table. You never write `CREATE OR REPLACE VIEW`. The label does it.

The one special bit is `${ref("file_1")}`, which means "use the output of file 1". Writing it also tells Dataform that file 1 must run first. **You never write the order.** With 40 files, that is the entire value of the tool.

The possible blocks in a file are `config`, `js`, `pre_operations`, the SQL body, and `post_operations`.

### In Dataform, what is the difference between a workspace and a branch?

A workspace **contains** a branch, plus what git calls the working directory.

| | |
|---|---|
| Branch | the saved commit history |
| Workspace | that branch, plus your uncommitted edits, plus the editor |

On a laptop, `git checkout -b my-work` creates the branch and the folder you type in is the working directory. Dataform merges both into one thing in the browser. If you never commit, nothing reaches the branch, and a run executes straight from the workspace.

## Replication and change data capture

Change data capture (CDC) means copying every insert, update, and delete out of a source database as it happens, instead of re-exporting whole tables.

### What is a write-ahead log (WAL)?

The database writes down what it is **about to do** before doing it.

Take `UPDATE employees SET salary = 95000 WHERE id = 42`:

1. Write to the log: "about to change id 42 from 88000 to 95000". Flush it to disk.
2. Tell the client "done".
3. Update the real data file later, in a batch.

If the power dies between steps 2 and 3, the database replays the log on restart. Nothing is lost. Without the log, step 2 would have to wait for a slow random disk write on every statement. One fast append to the end of a file makes the database faster and safer at once.

Why data people care: that log is a complete, ordered record of every change, with before and after values. CDC tools just read it. The database was writing it anyway, which is why Datastream puts very little load on the source.

### Oracle, MySQL, PostgreSQL, SQL Server: are their change logs four different SQL languages?

No. They are four competing **database products**, like four web browsers. All of them store tables and speak SQL. Each vendor gave its change log a different name.

| Product | Its log mechanism | What you actually read |
|---|---|---|
| Oracle | **LogMiner** | a view you can query, with the reconstructed SQL text |
| MySQL | **binary log** | a binary file with before and after images of each row |
| PostgreSQL | **logical decoding** | the output of a decoder plugin, because the raw WAL is physical ("these bytes on this page changed") |
| SQL Server | **transaction log** | change tables that SQL Server fills from its transaction log |

At this level, only the name-to-engine mapping matters.

### Why is logical decoding off by default in PostgreSQL?

Because it is not free. It makes PostgreSQL **write more**.

The default WAL is physical and sized for crash recovery only. Turning on logical decoding (the `cloudsql.logical_decoding` flag on Cloud SQL) adds enough detail to rebuild row-level changes. That costs disk and write throughput. It also creates a retention duty: a replication slot holds log entries until a reader collects them, so a stopped stream can fill the disk.

Most databases never need it, so it ships off.

- Turn it on for: replication to a warehouse, upgrades between PostgreSQL versions with no downtime, event-driven triggers, audit trails.
- Leave it off for: a plain app database that nobody reports on.

You only pay for it when something downstream consumes the changes.

### Publication vs replication slot: what is the difference?

Think of a video streaming app. The **publication** is which shows you are allowed to watch. The **slot** is where you paused.

| Thing | Answers |
|---|---|
| Publication | which tables may be copied |
| Replication slot | how far the reader got |

"Replication" on its own is not an object you create. It is the general word for the whole activity.

### User vs superuser: why does it matter for replication?

A **superuser** skips all permission checks. It can drop any table, read any data, create users, and change server settings. A regular user can only do what was explicitly granted. It is the same idea as root or Administrator against a normal account.

Tutorials often let you run everything as the admin user because it is quick. In production you create a dedicated user for Datastream with only the rights it needs. If that password leaks, the damage is limited.

### Is Datastream a storage product?

No. **Datastream stores nothing.** It is a pipe. It reads a change, passes it along, and forgets it. You always give it a destination (BigQuery or Cloud Storage), and that is where the data actually sits.

| Holds data | Moves data |
|---|---|
| Cloud Storage, BigQuery, Bigtable, Cloud SQL | Datastream, Dataflow, Storage Transfer Service, `gcloud storage` |

The name misleads. Read it as a verb.

### Could Dataflow do the job of Datastream?

Not the important half. The usual way Dataflow reads a database is by **running a query**, such as "give me the rows modified since yesterday". That is the nightly export problem again: real load on the source, a snapshot only, and it misses anything that changed and changed back. Reading each vendor's change log is the job Datastream exists for.

| | Datastream | Dataflow |
|---|---|---|
| How it gets data | reads the change log | asks the database a question |
| Load on the source | very little | a real query |
| Sees every change? | yes | only the state at query time |
| Transforms? | no | yes, anything |

The common shape is a chain: Oracle, then Datastream, then Dataflow (mask or enrich), then BigQuery. Datastream gets the changes out. Dataflow decides what they look like.

## SQL and data quality

### How do I know which columns to GROUP BY?

Say the question out loud. The nouns after **"per"** or **"each"** are the GROUP BY columns.

| Question | GROUP BY |
|---|---|
| Visitors per channel | `channelGrouping` |
| Views per product | `v2ProductName` |
| Each person once per product | `fullVisitorId, v2ProductName` |
| Sales per country per month | `country, month` |

Check: finish the sentence "one row in my result is one ___". Then every other column in SELECT must be wrapped in an aggregate.

### Is "GROUP BY every column HAVING COUNT > 1" how duplicates are found in real life?

The idea yes, the form usually not.

- **Group on the business key, not every column.** Real duplicates often differ in an unimportant column (a `loaded_at` timestamp) and slip past an all-columns check. Decide what should be unique, such as `order_id`, and group on that.
- **A key check is stronger.** If the key has no duplicates, the full row cannot have any either. The reverse is not true.
- **Shortcuts.** Any exact duplicates? Compare `COUNT(*)` with `COUNT(DISTINCT TO_JSON_STRING(t))`. Remove exact copies with `SELECT DISTINCT *`. Keep the latest row per key with `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY loaded_at DESC) = 1`.
- **Automate it.** Pipelines run the check on every load, for example a Dataform `uniqueKey` assertion that fails the run.

One line to keep: group by the columns that should be unique, and HAVING COUNT(*) > 1 shows what is not.

### How do I know the data is not duplicated, and that it is real?

I ask five questions, in order.

1. **What is one row?** (the grain) One click, one order, or one order line? Nothing else makes sense until this is answered.
2. **What should make a row unique?** (the key) Orders: `order_id`. Order lines: `order_id` plus `line_number`.
3. **Is the key actually unique?** Group by the key with `HAVING COUNT(*) > 1`. Then look at a few offenders side by side.
4. **Why are they duplicated?** The cause decides the fix. A load was retried (drop exact copies). Each update was kept as a new version (keep the latest per key). A join fanned out (fix the join, do not delete). The event really happened twice (not a duplicate at all).
5. **Is it real?** Duplicates are one failure. Rows that are unique but wrong are the other.

| Check | Example |
|---|---|
| Ranges | a negative quantity, a date in the future, a 90-hour bike ride |
| Placeholders | the ten heaviest babies in a births table all have the exact same weight: that is a cap |
| Test data | rows created by test or QA accounts |
| Outliers | one visitor with 50,000 hits in a day is a bot |
| Row counts | 955 rows for 954 stations: the header was loaded as data |
| Units | revenue stored multiplied by 1,000,000 |
| Reconciliation | does the warehouse total match the finance report? |

Worked example: the dashboard says March revenue is $1.2M and Finance says $1.0M. The table is one row per order line. 4,000 keys have two rows, loaded 10 minutes apart on the same night: a failed load was re-run and appended. After dedupe, revenue is $1.02M. The rest is 30 test orders ($15K) and refunds recorded a month late.

Reconciliation is the check that convinces stakeholders.

### A source API returns null for a field that should never be null. Who fixes it?

"The API is buggy" is usually the wrong diagnosis. Two different problems look the same downstream.

- **Source data problem.** The field really is empty. The "required" rule was added later and old records were never backfilled, or the record was created by an import that skipped the form validation.
- **Transport problem.** The value exists but the API returns null. The API account is not allowed to read that field, or the value is filled in a moment after the record is created and you read it too early.

To tell them apart, open the record in the source system's own screen. Filled there but null in your pull means transport. Empty there too means a real data quality issue.

- A transport problem is fixed by the data engineer.
- A real source problem is **not** patched in the pipeline. Quietly filling in a default hides it. Send those rows to a quarantine table and report them to the owner of the source process.

This is what the raw zone is for. Validation catches the null before it reaches a trusted reporting table.

### A practice exercise had me count bike trips per station. What would a real company do with that?

The exercise has no business goal. It is a SQL tour. A real bike share operator would use it for:

1. **Rebalancing.** More starts than ends at a station means it runs out of bikes. More ends than starts means the docks fill up. Trucks move bikes every day.
2. **Where to add docks.** The busiest stations get bigger ones.
3. **Maintenance.** The busiest stations wear out first.

That is why starts and ends belong **side by side** with a JOIN (`station, starts, ends, gap`), not stacked with a UNION that mixes both with no label.

### Is hardcoding a correction into SQL good practice?

Business logic in SQL is fine. That is what ELT is. **Hardcoding `id = 2` is the problem.** It is a magic value buried in a query. Nobody knows why, who decided, or when. Six months later someone deletes it because it looks like test code.

The fix is to keep corrections in a table, not in the query:

```sql
-- sales_corrections (id, corrected_amount, reason, approved_by, applied_on)
SELECT s.* REPLACE(COALESCE(c.corrected_amount, s.amount) AS amount)
FROM their_dataset.sales s
LEFT JOIN my_dataset.sales_corrections c USING (id)
```

- `LEFT JOIN` keeps every sales row, even with no matching correction.
- `COALESCE(a, b)` takes the first value that is not NULL: the correction if one exists, the original otherwise.
- `s.* REPLACE(... AS amount)` takes every column but swaps out `amount`.

The rule: logic belongs in SQL, exceptions belong in data. If you are editing a query to change a number, that number should have been a row.

And first ask why the row is wrong at the source. A correction layer is right only when you cannot fix it upstream.

### Does that correction pattern still work at 50 billion rows?

**The join is fine. The rewrite is not.** A tiny corrections table costs almost nothing to join. The problem is that `CREATE TABLE AS SELECT` scans 50 billion rows and writes 50 billion rows every time you add one correction.

In order of preference:

1. **A view, not a table.** No copy and no storage, but every query pays the join. Fine if the table is partitioned and users filter to a date range.
2. **Partition, then rewrite only what is affected.** A `MERGE` with partition pruning touches gigabytes, not petabytes. This is the right answer if you own the table.
3. **A materialized view.** BigQuery keeps it refreshed. Good when the same corrected result is queried all the time.

The general rule at that scale: never rewrite the whole thing to change part of it.

## Security and governance

### What is PI, and what is PII?

**PII** is personally identifiable information: data that identifies a specific person, alone or combined. Examples are a government ID number, passport, email, phone, employee ID, address, or a photo.

**PI** is personal information, the broader term: any data about an identifiable person, even if it does not identify them alone. Examples are job title, salary, or badge swipe times. All PII is PI. Not all PI is PII.

Which word you see depends on the law. GDPR says "personal data". US guidance says PII. In practice people use them interchangeably.

The part that matters: PII is not a fixed list of columns. It is whether the **combination** identifies someone. A row of `department, office, login_hour` has no name, but if that department in that office has one person who starts at 08:00, the row is a named individual. Those columns are called **quasi-identifiers**. "We dropped the name column" is not anonymization.

On Google Cloud, Sensitive Data Protection (used to be Cloud DLP) scans and classifies columns. Knowledge Catalog (used to be Dataplex) is where you tag the asset and attach policy.

### On a bucket, what is the difference between an ACL and IAM?

An **ACL** (access control list) is a permission list on **one object**, meaning one file. **IAM** is one grant on the bucket or the project that covers everything in it.

| Bucket access mode | What applies |
|---|---|
| Uniform bucket-level access | IAM only |
| Fine-grained | IAM plus per-object ACLs |

Uniform is simpler to reason about and audit. Fine-grained exists for the case where single files need their own rules.

### What is Knowledge Catalog, and where do I see it?

Knowledge Catalog is one product for finding and understanding your own data. Its jobs:

- catalog and search
- a business glossary
- data profiling
- data quality scans
- lineage (where a table came from and what reads it)

It is not a page per table. But it surfaces inside BigQuery as the Lineage, Data profile, and Data quality tabs on a table.

What it is **not** for: moving data (that is Dataflow or Dataform) or sharing data with others outside your team (that is the job of BigQuery sharing (used to be Analytics Hub)).

## Machine learning basics

### How does a model actually predict? (logistic regression by hand)

Start with five customers whose outcome you already know.

| customer | orders_count | days_since_last | avg_amount | churned |
|---|---|---|---|---|
| A | 12 | 5 | 340 | 0 |
| B | 2 | 200 | 60 | 1 |
| C | 9 | 20 | 280 | 0 |
| D | 1 | 310 | 45 | 1 |
| E | 15 | 8 | 410 | 0 |

`churned` is the answer key (the **label**). It comes from your own history. No history, no model.

Training searches for weights that make this formula match the known answers:

```
z = w0 + w1 * orders + w2 * days_since_last + w3 * avg_amount
```

Say it lands on `z = -2.0 - 0.15 * orders + 0.03 * days - 0.004 * avg_amount`. The signs are the learned knowledge: more orders, less churn. A longer quiet period, more churn. A bigger spender, less churn.

`z` can be any number, and a probability must sit between 0 and 1. The **sigmoid** function squashes it: `p = 1 / (1 + e^-z)`.

- Check B (who did churn): `z = -2.0 - 0.3 + 6.0 - 0.24 = 3.46`, so `p = 0.97`.
- Check A (who stayed): `z = -2.0 - 1.8 + 0.15 - 1.36 = -5.01`, so `p = 0.007`.
- New customer F (3 orders, 150 days quiet, $80 average): `z = 1.73`, so `p = 0.85`. That is 85% likely to churn.

Nothing in the data says F will churn. The formula, built from A to E, says F looks like B and D.

In one line: SQL tells you who left. The model turns that history into a formula and applies it to people who have not left yet.

### Is a model always a straight-line formula?

No. The `w1 * x1 + w2 * x2` shape is what makes a model **linear**: each input pushes the answer by a fixed amount, on its own. It cannot learn "high spend matters only for new customers".

| Family | Examples | Shape it learns |
|---|---|---|
| Linear | linear regression, logistic regression | straight-line effects |
| Trees | decision tree, random forest, gradient boosted trees (XGBoost) | nested if/then splits |
| Neural networks | deep learning | arbitrary curves |
| Instance-based | k-nearest neighbors | "who does this resemble?" |

On tabular business data, gradient boosted trees usually win. Neural networks dominate images, audio, and text. Logistic regression stays popular because you can **read** it: the weights say "more orders, less churn" in plain sight. Boosted trees are more accurate and much harder to explain in a meeting.

### How do gradient boosted trees work? (worked by hand)

Problem: predict how many hours a new help desk ticket will take to resolve.

| ticket | priority | reassignments | actual_hours |
|---|---|---|---|
| T1 | 1 | 3 | 40 |
| T2 | 4 | 0 | 4 |
| T3 | 2 | 1 | 20 |
| T4 | 4 | 1 | 8 |
| T5 | 1 | 2 | 32 |

**The mechanism: each tree fixes the mistakes of the ones before it.**

**Step 0, start dumb.** Predict the average, 20.8, for everyone. The errors (actual minus prediction) are +19.2, -16.8, -0.8, -12.8, +11.2. Total size of the errors: 60.8.

**Step 1, a tree that predicts the error, not the hours.** Split on `priority <= 2`. T1, T3, T5 have an average error of +9.9. T2 and T4 have an average error of -14.8. Add half of that (a learning rate of 0.5, so the numbers move visibly):

- T1, T3, T5: `20.8 + 0.5 * 9.9 = 25.7`
- T2, T4: `20.8 + 0.5 * (-14.8) = 13.4`

New errors: +14.3, -9.4, -5.7, -5.4, +6.3. Total 41.1. Four tickets improved. T3 got worse, because the split lumped it in with T1 and T5.

**Step 2, a tree on the new errors.** Split on `reassignments >= 2`. T1 and T5 get +5.2. T2, T3, T4 get -3.4.

| ticket | prediction | actual | error |
|---|---|---|---|
| T1 | 30.9 | 40 | +9.1 |
| T2 | 10.0 | 4 | -6.0 |
| T3 | 22.3 | 20 | -2.3 |
| T4 | 10.0 | 8 | -2.0 |
| T5 | 30.9 | 32 | +1.1 |

Total error went 60.8, then 41.1, then 20.5. Tree 1 hurt T3, and tree 2 caught it. That is the self-correcting part.

The loop in three lines:

1. Look at how wrong you are.
2. Build a small tree that predicts that wrongness.
3. Add a fraction of it to your prediction, then go back to 1.

"Boosting" means many weak models stacked. "Gradient" means each one aims at the leftover error. Real jobs use a small learning rate (such as 0.1) with many trees.

The catch: you get a feature importance ranking, not readable weights. You cannot easily explain why one ticket got 44 hours. It is a permanent trade between accuracy and being able to defend the number.

### How do I tell if a model is any good without reading the code?

Nobody reads the rows, not even the data scientist. You ask five questions.

**1. What did you hold out?** The model must be scored on rows it never saw in training. Ask for both numbers.

| | Training score | Test score | Verdict |
|---|---|---|---|
| Healthy | 82% | 79% | fine |
| Overfit | 99% | 61% | it memorized |

The gap between training and test is the sign of **overfitting**. A single impressive number quoted alone is often the training score.

**2. What is the baseline?** The dumbest rule, scored the same way, such as "always predict the majority class". Fraud detection at 99.5% accuracy sounds excellent until you notice that 99.5% of transactions are not fraud, so "always say no fraud" scores the same.

**3. Would we have this column at prediction time?** This is **data leakage**. Someone adds the length of the resolution notes to a ticket model and accuracy jumps to 97%. But resolution notes only exist after the ticket is resolved. The model was reading the answer.

**4. How did you split, randomly or by date?** With time-based data, a random split trains on December and tests on November. The honest version trains on January to September and tests on October to December.

**5. Does the feature importance list make sense?** `reassignments` at the top fits what you know about the process. `ticket_id` at the top means something is badly wrong. This is where knowing the business beats knowing the math.

The short version: what is the test score, what is the baseline, and would we have every one of those columns at 9 am on a brand-new ticket?

### Can my BI measures become ML features?

Yes. Same logic, different shape. A **feature** is an input column for a model. Two things change, and the second one is the trap.

**1. Grain.** A dashboard measure aggregates up: "12% of tickets breached this month". A feature points down to one row: "this ticket's caller has a 23% breach rate". Same calculation, opposite direction.

**2. Point-in-time correctness.** A dashboard measure uses everything known today. A feature may only use what was known **when that row's event happened**. Get this wrong and you have leakage.

The rule: a feature can only use data that existed before the thing you are predicting. For every measure ask "would I have known this at prediction time?" Anything that touches the outcome, such as the resolution date, the closure code, or the final state, is out.

### Can all the feature engineering be done in SQL, in BigQuery?

For tabular data, yes, and it is the preferred path. Joins, aggregates, date math, `CASE` binning, and window functions are all native. Write one query, save it with `CREATE TABLE ... AS SELECT`, and point AutoML or BigQuery ML at the result. The data does not move: no CSV export and no second copy drifting from the first.

Where SQL runs out: images, audio, video, and some library-specific transforms.

A worked example with a self-join. The same `tickets` table is used three times under three aliases (short labels for which copy you mean): `i` is the ticket being described, `p` is that caller's earlier tickets, and `q` is the tickets open at that moment.

| number | caller | opened | closed | breached |
|---|---|---|---|---|
| T001 | jdoe | Jan 05 | Jan 06 | TRUE |
| T002 | jdoe | Jan 10 | Jan 11 | FALSE |
| T003 | asmith | Jan 18 | Jan 25 | TRUE |
| T004 | jdoe | Jan 20 | Jan 22 | TRUE |

Building the training row for T004:

- `p`: same caller and closed before Jan 20. That leaves T001 (TRUE) and T002 (FALSE). Past breach rate is 0.5.
- `q`: opened before Jan 20 and closed after it. Only T003. Open tickets at that moment: 1.

T004 itself is excluded from its own history, because it closed after it opened. That is point-in-time correctness in SQL. Only closed tickets can be training rows, because only they have a known answer.

### What is an embedding, and what is it for?

**An embedding turns text into a list of numbers that represents its meaning.** One row, one list (often 768 numbers), stored in one column.

| ticket | description | vector (4 of 768 shown) |
|---|---|---|
| T001 | "Server won't boot" | `[0.91, 0.12, 0.88, 0.04]` |
| T002 | "Machine fails to start" | `[0.89, 0.15, 0.85, 0.06]` |
| T003 | "Printer out of toner" | `[0.07, 0.94, 0.11, 0.81]` |

T001 and T002 share zero words but sit almost on top of each other. T003 is far away. **Close numbers mean close meaning.**

Four uses, all of them "find close numbers":

1. **Duplicate detection.** Compare a new ticket's vector to the open tickets. Keyword matching cannot catch the pair above.
2. **Clustering into themes.** K-means over thousands of ticket vectors produces groups nobody defined: VPN, printers, password resets.
3. **Semantic search.** "can't log in remotely" finds VPN tickets that never use the word "login".
4. **As features.** The numbers become input columns for a model.

It is also the engine behind retrieval-augmented generation (RAG): turn the question into a vector, find the nearest document vectors, and paste those documents into the prompt.

In BigQuery this stays in SQL. `ML.GENERATE_EMBEDDING` makes the vectors through a remote model on Agent Platform (used to be Vertex AI), `VECTOR_SEARCH` finds the nearest ones, and a k-means model groups them.

### How do I pick a threshold when the two kinds of mistake cost different amounts?

A classifier outputs a probability. The **threshold** is the cut-off where you call it a "yes". A **false positive** is a false alarm. A **false negative** is a miss.

The evaluation screens give you precision, recall, and the curves. They never give you money. So I build the cost chart myself from the test set predictions. With a miss costing $10,000 and a false alarm costing $1,500:

```sql
SELECT
  t AS threshold,
  COUNTIF(prob >= t AND actual = 0) * 1500     -- false alarms
  + COUNTIF(prob < t AND actual = 1) * 10000   -- misses
  AS total_cost
FROM predictions, UNNEST(GENERATE_ARRAY(0.05, 0.95, 0.05)) AS t
GROUP BY t
ORDER BY total_cost
```

One row per threshold, cheapest first. On a chart, put the threshold on x and dollars on y, and look for the bottom of the U.

When a miss costs much more than a false alarm, the best threshold drops below 0.5. Lowering the threshold buys recall and spends precision.

The two cost numbers are the hard part. They are a business claim that people will argue about, not data. Getting them agreed in writing is the real decision. The SQL takes ten minutes after that.

## Beyond the exam guide

None of this is in the Associate Data Practitioner exam guide. I asked anyway, and I kept the answers short.

### Is an AI agent just a foundation model plus instructions plus MCP?

Two-thirds right.

- **Foundation model:** yes, that is the brain.
- **Instructions:** they steer the model, but they are input, not a separate component.
- **MCP** (Model Context Protocol): one standard format for handing tools to a model. It is plumbing inside the **tools** component. An agent does not need MCP.

What the formula misses is the **orchestration layer**, the loop: the model decides, a tool runs, the result comes back to the model, and the model decides the next step. Without that loop, you have a model that makes one tool call and stops.

The honest version: agent = foundation model (with its goal and instructions) + tools + a loop that feeds tool results back to the model.

### When does a job actually deserve an agent?

A file converter is not an agent use case. One input, one fixed operation, one output, no decisions. That is a few lines of code. An agent would add delay, cost, and a model that can get it wrong.

**The rule: if you can draw the flowchart, write the code.** An agent earns its place only when the steps vary by input and cannot be known in advance.

Examples that pass the test:

- **Proposal intake.** A request arrives. The agent decides which product sheets are relevant, checks the deadline, and either drafts a response or routes it to a person with the reason.
- **Ticket triage.** The agent decides whether to look up the asset owner or check for a duplicate, does that, then decides again based on what came back.
- **Invoice exceptions.** The invoice does not match the purchase order. The agent works out why, and each cause needs a different system checked.

The common thread is branching that a person would otherwise do.

### What is Model Garden?

A **catalog** inside Agent Platform (used to be Vertex AI) that lists the models you can deploy. Each one has a card showing what it does and how to call it. It does not run anything itself. It is the shelf you pick from.

Three kinds of model sit on it:

- **Google's own**, such as Gemini (text and more), Imagen (images), Veo (video), and Chirp (speech).
- **Third-party models hosted by Google**, such as Llama and Mistral.
- **Open models you can tune and host yourself**, such as Gemma.

Example: you want a tool that drafts summaries of help desk tickets. You start with a fast, cheap model. A week later the summaries read too shallow, so you switch to a stronger model and change one model name in the config. The rest of the code does not change. That swap is the point of having a catalog.

### What is ADK?

**Agent Development Kit**: the code-first path for building agents. It is a library you install, most commonly used from Python, in place of clicking an agent together in a visual builder.

What it gives you:

- **The agent defined in code:** its instructions, which tools it may call, which model it uses.
- **Your own functions as tools.** You write a plain function and ADK exposes it to the model. The function's description text is how the model knows what the tool does.
- **Multi-agent structure.** A manager agent hands work to sub-agents, with explicit rules for who hands off to whom.
- **The loop is written for you.** ADK runs the decide, call, observe cycle.

The prompt does not go away. It lives inside the code as a string.

The trade-off: a visual builder is fast, but you inherit the reasoning loop someone else built. ADK makes you write the structure yourself, which is why it fits "must talk to our old internal system" or "must follow this exact approval sequence".

In Power Platform terms: the visual builder is Copilot Studio, and ADK is writing the bot in code.

### What is LangChain, and do I need it?

A free, open source **library** for building agents and apps on top of large language models. Think of it as a vendor-neutral cousin of ADK. Its selling point is **portability**: how much you would have to rewrite if you switched vendors. The model is a separate object in the code, so in theory you swap one line to change vendor.

Do you need it? Roughly:

- One vendor: use that vendor's own SDK or ADK.
- Several vendors: a thin routing layer may be enough.
- A full framework: only when you want its prebuilt pieces, knowing the cost is a layer between you and the model.

Portability is insurance against a vendor change. It costs an extra layer now to save a rewrite later. If you never switch, you paid for nothing.

The BI version of the same idea: a report with source field names hardcoded into 40 queries is not portable. Read everything from one staging table with generic column names and you swap one query.

### What is the equivalent of BigQuery ML on other platforms?

The idea of BigQuery ML: train and call the model **inside the warehouse, in SQL**, with no data movement.

| Platform | Equivalent | How close |
|---|---|---|
| Snowflake | Cortex: SQL functions for embeddings, sentiment, and text completion, plus a native vector type | very close |
| Databricks | AI functions in SQL (`ai_query`, `ai_classify`, `ai_analyze_sentiment`) plus Vector Search. Classic training is done in notebooks | close |
| AWS | Redshift ML: `CREATE MODEL` in SQL, with SageMaker doing the work behind the scenes | close for classic ML |
| Microsoft | A `PREDICT` function can score a model in SQL, but training lives in Azure ML | partial |

The pattern: most platforms converged on "call a model as a SQL function and keep the data where it is". In the Microsoft stack you still mostly move the data to the ML tool. In BigQuery ML the model comes to the data.
