# Quick reference sheets

Part of the [handbook](README.md). Practice questions and the interactive version: [ADP Study Hub](https://claude.ai/artifact/BKoxgMe73TZksDEshXkXGY).

**In this chapter**

- [If the question says X, think Y](#s-if-then)
- [Tool choosers](#s-choosers)
- [Look-alike names](#s-lookalikes)

<a id="s-if-then"></a>

## If the question says X, think Y

Read this the night before: find the phrase in the question, then jump to the section if the answer is not yet automatic.

### Qualifier words

| The question says | Think |
|---|---|
| "Most efficient", "minimize complexity", "without custom code" | The dedicated service built for exactly that job, set up in the console, with the fewest parts [see](00-start-here.md#s-patterns) |
| "Fully managed" or "serverless" | BigQuery, Dataflow, Pub/Sub, scheduled queries: no servers to size [see](00-start-here.md#s-patterns) |

### Getting data in and preparing it

| The question says | Think |
|---|---|
| Copy files, under about 1 TB, one time | `gcloud storage` [see](01-data-preparation-and-ingestion.md#s-transfer-tools) |
| Over 1 TB, recurring, or the files sit in Amazon S3 or Azure | Storage Transfer Service [see](01-data-preparation-and-ingestion.md#s-transfer-tools) |
| Upload would take over a week, or the network is weak | Transfer Appliance, a device Google ships to you [see](01-data-preparation-and-ingestion.md#s-transfer-tools) |
| Pull Google Ads, Salesforce or S3 data into BigQuery on a schedule, no code | BigQuery Data Transfer Service [see](01-data-preparation-and-ingestion.md#s-transfer-tools) |
| Move a live database to Cloud SQL or AlloyDB with little downtime | Database Migration Service [see](01-data-preparation-and-ingestion.md#s-transfer-tools) |
| Every insert, update and delete in a database must reach BigQuery | Datastream, which reads the database log [see](01-data-preparation-and-ingestion.md#s-transfer-tools) |
| Pub/Sub into BigQuery. Nothing changes / a light change, no code / complex logic | BigQuery subscription / Google-provided Dataflow template / custom Dataflow pipeline [see](01-data-preparation-and-ingestion.md#s-streaming) |
| Some messages fail again and again | Dead-letter topic on the subscription [see](01-data-preparation-and-ingestion.md#s-streaming) |
| Personal data must never be stored | ETL: remove or mask it with Dataflow or Data Fusion before the load [see](01-data-preparation-and-ingestion.md#s-etl-elt) |
| Keep the raw data, analysts write SQL | ELT: load first, transform in BigQuery, Dataform for chains of steps [see](01-data-preparation-and-ingestion.md#s-etl-elt) |
| Drag and drop, no code, many connectors | Cloud Data Fusion [see](01-data-preparation-and-ingestion.md#s-processing-tools) |
| Existing Spark or Hadoop jobs, or the question says "Dataproc" | Managed Service for Apache Spark (used to be Dataproc) [see](01-data-preparation-and-ingestion.md#s-processing-tools) |
| A JSON file holds one big array and fails to load | Convert to newline-delimited JSON; Avro or Parquet carry their own schema [see](01-data-preparation-and-ingestion.md#s-file-formats) |
| Must survive the loss of a region | Dual-region or multi-region Cloud Storage bucket [see](01-data-preparation-and-ingestion.md#s-location-types) |

### Analysis and ML

| The question says | Think |
|---|---|
| Filter on `COUNT` or `SUM` / filter on `ROW_NUMBER` | `HAVING` / `QUALIFY` [see](02-analysis-and-presentation.md#s-sql) |
| A total comes out too high after a join | Fan-out: reduce the many side to one row per key first [see](02-analysis-and-presentation.md#s-sql) |
| Items are a list inside each order row | `UNNEST`, and `COUNT(*)` then counts items [see](02-analysis-and-presentation.md#s-nested) |
| Make a query cheaper, someone suggests `LIMIT` | `LIMIT` still reads the whole table; filter the partition column and select fewer columns [see](02-analysis-and-presentation.md#s-cost-speed) |
| Queries always filter on a date / on customer or country | Partition by the date / cluster by those columns [see](02-analysis-and-presentation.md#s-cost-speed) |
| Same aggregation, new rows all day, result must be current | Materialized view [see](02-analysis-and-presentation.md#s-cost-speed) |
| Governed metrics defined once, many teams, LookML | Looker [see](02-analysis-and-presentation.md#s-looker) |
| Free, quick, self-service report, mixed sources | Data Studio (used to be Looker Studio) [see](02-analysis-and-presentation.md#s-looker) |
| Predict a number / a category / groups with no labels / a forecast | `LINEAR_REG` / `LOGISTIC_REG` / `KMEANS` / `ARIMA_PLUS` with `ML.FORECAST` [see](02-analysis-and-presentation.md#s-bqml) |
| Translate, transcribe or detect objects with no training data | Pre-trained API; Gemini over table rows is a remote model [see](02-analysis-and-presentation.md#s-ai-options) |
| Evaluation score is near perfect | Suspect data leakage, a feature known only after the outcome [see](02-analysis-and-presentation.md#s-ml-project) |

### Orchestration

| The question says | Think |
|---|---|
| One SQL statement on a timer | Scheduled query [see](03-pipeline-orchestration.md#s-orchestrators) |
| Several products, dependencies, waits, backfills | Managed Service for Apache Airflow (used to be Cloud Composer) [see](03-pipeline-orchestration.md#s-orchestrators) |
| A few API calls in order, serverless, "simplest" | Workflows, or the product's own scheduler [see](03-pipeline-orchestration.md#s-orchestrators) |
| Process each file as soon as it lands | Eventarc trigger to a Cloud Run function (used to be Cloud Functions) [see](03-pipeline-orchestration.md#s-event-driven) |
| Alert me / exact error / which step is slow | Cloud Monitoring / Cloud Logging / Dataflow job graph [see](03-pipeline-orchestration.md#s-monitoring) |

### Access, protection and recovery

| The question says | Think |
|---|---|
| Analyst runs queries on one dataset, "least privilege" | Job User on the project plus Data Viewer on that dataset [see](04-data-management.md#s-iam) |
| One place to audit / one object differs / never public | Uniform access / fine-grained with ACLs / public access prevention [see](04-data-management.md#s-bucket-access) |
| Different rows for different people, in BigQuery / in Looker / without table access | Row-level policy / `access_filter` / authorized view [see](04-data-management.md#s-protect) |
| Rotate or disable my own key / key must stay outside Google Cloud | CMEK in Cloud KMS / CSEK, not for BigQuery [see](04-data-management.md#s-encryption) |
| Read about monthly / quarterly / less than yearly | Nearline / Coldline / Archive [see](04-data-management.md#s-storage-classes) |
| Unknown access pattern / delete after N days / nobody may delete | Autoclass / lifecycle rule / locked retention policy [see](04-data-management.md#s-storage-classes) |
| Bad delete in Cloud SQL / zone outage / heavy reads | Backup or point-in-time recovery / high availability / read replica [see](04-data-management.md#s-recovery) |
| Give partners live data with no copies / find tables and lineage | BigQuery sharing (used to be Analytics Hub) / Knowledge Catalog (used to be Dataplex) [see](04-data-management.md#s-sharing-catalog) |

<a id="s-choosers"></a>

## Tool choosers

Each table answers one choice: find the tool by what the question needs, then check the "Not when" column for the trap.

### Transfer and loading

Details: [see](01-data-preparation-and-ingestion.md#s-transfer-tools).

| Tool | Use it when | Not when |
|---|---|---|
| `gcloud storage` | Files go to Cloud Storage, under about 1 TB, one quick copy | Recurring, huge or cross-cloud transfers |
| Storage Transfer Service | Over 1 TB, scheduled, or from S3, Azure, URLs, on-premises file systems | The source is a SaaS app or a live database |
| Transfer Appliance | Upload would take over a week, or bandwidth is poor | Small data, or a fast network |
| `bq load` or client library | You hold a file and want one BigQuery table | Scheduled pulls from SaaS apps |
| BigQuery Data Transfer Service | Scheduled, no-code loads from Google Ads, Salesforce, S3 and more | Moving a database, or custom file formats |
| Database Migration Service | Move a live database to Cloud SQL or AlloyDB, little downtime | Feeding changes to analytics |
| Datastream | Every database change must reach BigQuery or Cloud Storage | Moving the app database; web click events |

### Processing and transformation

Details: [see](01-data-preparation-and-ingestion.md#s-processing-tools) and [see](01-data-preparation-and-ingestion.md#s-streaming).

| Tool | Use it when | Not when |
|---|---|---|
| BigQuery SQL | Data is already in BigQuery and SQL can do the job | Streaming, or logic SQL cannot express |
| Dataform | A chain of dependent SQL steps with tests and a schedule | Loading data into BigQuery |
| Dataflow | Streaming, or batch plus stream, with custom logic or a template | One SQL query would do |
| Managed Service for Apache Spark (used to be Dataproc) | Spark or Hadoop code already exists | New SQL-only work |
| Cloud Data Fusion | Visual, no-code pipelines, many connectors, ETL | A quick SQL transformation |
| BigQuery subscription | Pub/Sub messages fit the table unchanged | Messages need reshaping |
| Google-provided Dataflow template | Light change on the way, no code to write | Complex joins or windows |

### Orchestration

Details: [see](03-pipeline-orchestration.md#s-orchestrators) and [see](03-pipeline-orchestration.md#s-event-driven).

| Tool | Use it when | Not when |
|---|---|---|
| Scheduled query | One SQL statement on a timer | Steps depend on each other |
| Cloud Scheduler | You need a clock that calls an HTTP target or Pub/Sub | You need steps or retries between steps |
| Dataform schedule | Many SQL tables built in order | The work is not SQL in BigQuery |
| Managed Service for Apache Spark workflow templates (used to be Dataproc workflow templates) | A fixed chain of Spark jobs on a temporary cluster | Other products take part |
| Workflows | A few API calls in order, serverless | You need backfills |
| Managed Service for Apache Airflow (used to be Cloud Composer) | Many products, dependencies, backfills | One product's own scheduler is enough |
| Eventarc with a Cloud Run function (used to be Cloud Functions) | Run small code when a file lands or a message arrives | Heavy processing inside the function |

### Storage and databases

Details: [see](01-data-preparation-and-ingestion.md#s-storage-products).

| Product | Use it when | Not when |
|---|---|---|
| Cloud Storage | Raw files, images, backups, a data lake | You need row-by-row SQL lookups |
| BigQuery | Large-scale SQL analysis and reports | Many tiny fast app transactions |
| Cloud SQL | A standard MySQL, PostgreSQL or SQL Server app database | Global scale or huge write volume |
| AlloyDB | PostgreSQL apps that outgrow Cloud SQL | A small simple database |
| Spanner | Strong consistency at global scale | A regional app Cloud SQL already handles |
| Bigtable | Huge writes, millisecond lookups by row key, time series | Ad hoc SQL or joins |
| Firestore | Mobile and web apps, flexible fields, live sync | Heavy analytics |

### BigQuery speed and cost

Details: [see](02-analysis-and-presentation.md#s-cost-speed).

| Feature | Use it when | Not when |
|---|---|---|
| Partitioning | Queries filter on a date, timestamp or integer column | Queries never filter on it; the table already exists |
| Clustering | Queries filter on a few other columns | No filter on those columns |
| Standard view | Hide columns or share a saved query | You need stored, faster results |
| Materialized view | The same aggregation, kept current as rows arrive | A heavy report can be a little old, so a scheduled query is enough |
| Scheduled query into a table | A heavy report that may be a little old | Results must be current |
| Dry run | You want the bytes estimate before paying | You need the results |
| Selecting only needed columns | Read less data | You rely on `LIMIT` to cut cost |

### ML options

Details: [see](02-analysis-and-presentation.md#s-ai-options) and [see](02-analysis-and-presentation.md#s-bqml).

| Option | Use it when | Not when |
|---|---|---|
| Pre-trained API | A common task such as translation, speech or vision, no training data | The task is specific to your data |
| Gemini through a remote model | Summaries or text drafting over BigQuery rows | A fixed, structured answer from a specialized API |
| BigQuery ML | Labeled table data in BigQuery, a SQL team | Data lives outside BigQuery and cannot move |
| `KMEANS` in BigQuery ML | No labels, you want groups | You have a known answer column |
| AutoML in Agent Platform (used to be Vertex AI) | Labeled tabular or image data, little code | A SQL team can train in BigQuery |
| Custom training | You need full control of the model code | AutoML or BigQuery ML already works |

### Access and protection

Details: [see](04-data-management.md#s-iam), [see](04-data-management.md#s-bucket-access) and [see](04-data-management.md#s-protect).

| Control | Use it when | Not when |
|---|---|---|
| Predefined role at the lowest scope | Least privilege on a dataset or bucket | A basic role looks easier |
| Uniform bucket-level access | One place to audit, IAM only | One object needs its own ACL |
| Fine-grained access with ACLs | Single objects must differ from the rest | You want the simplest audit trail |
| Public access prevention | Nothing may ever be made public | You need a public website bucket |
| Signed URL | Time-limited access for someone with no Google account | The person has a Google identity |
| Row-level access policy | BigQuery users see different rows | The users all come in through Looker |
| Authorized view | Share part of a table with people who lack table access | You only need a different role |
| Looker `access_filter` | Per-person rows inside Looker, from a user attribute | Plain BigQuery users |

### Recovery

Details: [see](04-data-management.md#s-recovery) and [see](01-data-preparation-and-ingestion.md#s-location-types).

| Feature | Use it when | Not when |
|---|---|---|
| Automated or on-demand backup | Restore a Cloud SQL database to a past backup | You need a minute-exact restore |
| Point-in-time recovery | Undo a bad change at a known minute | Backups or the log are off |
| High availability | Automatic failover for a zone outage | Someone deleted rows (the standby copies it) |
| Read replica | Take read load, or keep a copy in another region | You expect automatic failover |
| Object versioning | Recover overwritten or deleted objects | Surviving a region loss |
| Soft delete | Bring back deleted or overwritten objects within 7 days by default | After the window |
| Dual-region bucket | Survive a region loss with no action | Recovering deleted objects |
| Retention policy with Bucket Lock | Nobody may delete for a set time | You need to delete on request |

### Storage classes and expiry

Details: [see](04-data-management.md#s-storage-classes).

| Option | Use it when | Not when |
|---|---|---|
| Standard | Read often, or kept a few days | Data sits untouched for months |
| Nearline | Read about once a month, 30 day minimum | Read all the time |
| Coldline | Read about once a quarter, 90 day minimum | Read monthly |
| Archive | Read less than once a year, 365 day minimum | Deleted early or read often |
| Autoclass | Unknown or changing access pattern | A lifecycle rule on a known age is enough |
| Lifecycle rule | Delete or cool objects after N days | Nobody may delete (use retention) |
| BigQuery expiration | Drop old tables, partitions or dataset defaults | You need the data for audit |

<a id="s-lookalikes"></a>

## Look-alike names

Names that sound alike get swapped in wrong options. Use the last column to separate them fast.

| Name | What it is in one line | Do not confuse with |
|---|---|---|
| Dataflow | Serverless Apache Beam pipelines for batch and streaming | Dataform (SQL only) and Data Fusion (drag and drop) |
| Dataform | SQL files that build chained tables inside BigQuery, ELT | Dataflow, which moves and reshapes data in code |
| Managed Service for Apache Spark (used to be Dataproc) | Managed Spark and Hadoop, clusters or serverless | Dataflow, which uses Beam instead |
| Managed Service for Apache Spark workflow templates (used to be Dataproc workflow templates) | A graph of Spark jobs that runs on a temporary cluster | Workflows, which chains API calls |
| Workflows | Serverless chain of API calls with retries | Spark workflow templates, and Managed Service for Apache Airflow (used to be Cloud Composer), which handles backfills |
| Cloud Data Fusion | Visual pipelines and Wrangler for no-code cleaning | Dataflow, which needs Beam code or a template |
| Datastream | Change data capture from a database into BigQuery or Cloud Storage | Database Migration Service, which moves the database itself |
| Database Migration Service | Moves a live database to Cloud SQL or AlloyDB | Datastream, which feeds analytics |
| Knowledge Catalog (used to be Dataplex) | Search, lineage, profile and quality scans for data you own | BigQuery sharing, which sends data out |
| BigQuery sharing (used to be Analytics Hub) | Publish a dataset in a listing, subscribers get a linked dataset, no copy | Knowledge Catalog, which shares nothing |
| Looker | Enterprise BI where LookML defines metrics once | Data Studio (used to be Looker Studio) |
| Data Studio (used to be Looker Studio) | Free drag-and-drop self-service reports | Looker, which has LookML and folder access |
| Storage Transfer Service | Bulk copy of files into Cloud Storage from S3, Azure, URLs, on-premises | BigQuery Data Transfer Service, which loads into BigQuery |
| BigQuery Data Transfer Service | Scheduled no-code loads from SaaS apps and clouds into BigQuery | Storage Transfer Service, which lands in Cloud Storage |
| Transfer Appliance | A device Google ships, you fill it, and Google loads the data | Storage Transfer Service, which needs the network |
| Cloud Scheduler | A clock that calls an HTTP target or Pub/Sub | A scheduled query, which runs the SQL itself |
| Scheduled query | BigQuery runs one SQL statement on a timer | Cloud Scheduler, which runs nothing itself |
| Managed Service for Apache Airflow (used to be Cloud Composer) | Python DAGs across many products, with backfills | Cloud Scheduler, which is only a clock |
| Eventarc | Routes an event to a destination; it runs no code | Pub/Sub, a message bus; the Cloud Run function (used to be Cloud Functions) is what runs code |
| Pub/Sub | Topics and subscriptions that carry messages | Eventarc, which can use a topic as its source |
| CMEK | Your key, kept in Cloud KMS, works with BigQuery and Cloud Storage | CSEK, where the key stays outside Google |
| CSEK | Your key, kept outside Google, sent with each request | CMEK; BigQuery does not accept CSEK |
| GMEK | Google creates, stores and rotates the key; the default | CMEK, where you control the key |
| Uniform bucket-level access | IAM only, ACLs off for the whole bucket | Fine-grained, which also uses ACLs |
| Fine-grained access | IAM plus ACLs on single objects | Uniform, and public access prevention, a separate switch |
| Standard view | A saved query that runs every time | Materialized view, which stores a refreshed result |
| Materialized view | A stored result that BigQuery keeps current | A scheduled query table, only as new as its last run |
| Table partitioning | Splits a table by a column so filters skip slices | Clustering, which sorts rows by up to four columns |
| Clustering | Sorts rows by up to four columns inside the table | Partitioning, which splits the table into slices |
| Window `PARTITION BY` | Groups rows inside `OVER()` for one calculation | Table partitioning, which is storage layout |
| `ML.PREDICT` | Applies a model to new rows and keeps your columns | `ML.EVALUATE`, which scores the model |
| `ML.FORECAST` | Future values from a time-series model, no input table | `ML.PREDICT`, which needs input rows |
| `ML.EVALUATE` | Scores a model on held-back test rows | `ML.PREDICT`, which gives answers for new rows |
| High availability | A standby in another zone with automatic failover | A read replica, which serves reads and is promoted by hand |
| Read replica | A read-only copy that can sit in another region | High availability, which cannot serve reads |

---

Previous: [Domain 4: Data management](04-data-management.md)
