# Learning notes 2: ETL, automation and BigQuery hands-on

## Extract, Transform and Load (ETL)

The third and largest of the three patterns. This is where the **compute engines** live.

Topics covered here:
1. Baseline extract, transform and load architecture diagram.
2. **GUI tools** on Google Cloud for ETL data pipelines.
3. **Batch data processing using Dataproc**.
4. **Dataproc Serverless for Spark** for ETL.
5. **Streaming data processing** options on Google Cloud.
6. The role **Bigtable** plays in data pipelines.

Two distinctions to watch for: **Dataproc vs Dataproc Serverless** (cluster you manage vs no cluster), and **batch vs streaming**, which runs through all of it. Topic 5 is where **Pub/Sub and Dataflow** properly arrive, the path for the website-clicks case (see the Datastream vs Pub/Sub note in open questions). Bigtable has been in the storage decision tree since learning notes 1 but has not had a job until now.

## Hands-on: running a Spark job with no cluster to load an Avro file into BigQuery

Run a **Spark job with no cluster** to move an Avro file from Cloud Storage into BigQuery.

⚠️ **`bq load` could do this in one line.** The point is not the result, it is the **mechanism**: Spark, running serverless, driven by a pre-built template.

### What "serverless Spark" means here

Classic Dataproc: create a cluster → wait for machines to boot → run the job → delete the cluster. **Serverless for Apache Spark:** submit the job, Google finds the machines. Same Spark, no infrastructure to size, patch, or remember to delete.

### The four steps

**1. Environment (Cloud Shell)**

```bash
gcloud compute networks subnets update default --region=us-west1 --enable-private-ip-google-access
gsutil mb -p $PROJECT gs://$PROJECT              # staging bucket
gsutil mb -p $PROJECT gs://$PROJECT-bqtemp       # BigQuery scratch bucket
bq mk -d loadavro                                # empty dataset
```

| Command | Why |
|---|---|
| `enable-private-ip-google-access` | Lets machines without a public IP reach Google services, the Spark workers need it |
| `gsutil mb` | **m**ake **b**ucket |
| **Two buckets, not one** | Staging holds the data + Spark's own files; `-bqtemp` is scratch space BigQuery needs while building the table. Different jobs, don't merge |

**2. Assets (inside the VM, via SSH)**: `wget` the Avro and the templates zip, `gcloud storage cp` the Avro **into the bucket** (Spark reads from the bucket, not from the VM), unzip, `cd dataproc-templates/python`.

**3. Run it**

```bash
export GCP_PROJECT=<project>
export REGION=us-west1
export GCS_STAGING_LOCATION=gs://<project>
export JARS=gs://<sample-bucket>/spark-bigquery_2.12-20221021-2134.jar

./bin/start.sh \
-- --template=GCSTOBIGQUERY \
    --gcs.bigquery.input.format="avro" \
    --gcs.bigquery.input.location="gs://<project>" \
    --gcs.bigquery.input.inferschema="true" \
    --gcs.bigquery.output.dataset="loadavro" \
    --gcs.bigquery.output.table="campaigns" \
    --gcs.bigquery.output.mode=overwrite \
    --gcs.bigquery.temp.bucket.name="<project>-bqtemp"
```

Reads as a sentence: *take avro files from this bucket, work out the schema yourself, write to dataset `loadavro` table `campaigns`, overwrite, use bqtemp as scratch.*

`JARS` is the **spark-bigquery connector**: Spark cannot write to BigQuery natively, this library teaches it how.

**Three layers stacked, only the top one is yours:** a shell command → runs `start.sh` → which submits PySpark → which runs on serverless Spark. You never see or write the Python; the template holds it.

**4. Confirm**: `bq query --use_legacy_sql=false 'SELECT * FROM \`loadavro.campaigns\`;'`

The returned columns (`created_at`, `period`, `campaign_name`, `amount`, `advertising_channel`, `bid_type`, `id`) were **never typed by anyone**: they came out of the Avro file's own schema. That is `inferschema=true` working.

### Concepts clarified during this exercise

**A Spark job** = one piece of work handed to Spark: *read this, do this, write there*. Submit, it runs, it finishes. Like a script, except Spark splits the work across many machines running at once. 800 GB of logs → 10 GB each to 80 machines → each counts its chunk → partial counts summed.

**Avro** = a binary table format that **carries its own schema inside the file** (once, in a header, not per row).

The real use case is **systems handing data to each other**. Send a CSV and the receiver needs a separate spec doc, which drifts from the file over time. Send Avro and there is nothing to send separately and nothing to drift. **Schema evolution:** add a `discount` column and old readers ignore the field they don't know; with CSV an extra column shifts everything and silently corrupts the load.

**Row-based vs the schema property are two separate things:**

| | Layout | Best at |
|---|---|---|
| CSV | Text rows, no types | Handing a file to a human |
| **Avro** | **Row by row**, binary, schema in a header | Data **in motion**: feeds, streams, messages |
| Parquet | **Column by column**, binary | Data **at rest**: sitting in a bucket to be queried |

Avro writes one complete record at a time, which fits streaming (Datastream emits Avro or JSON for exactly this reason). Parquet wants a big batch before it can write efficiently.

**Compute Engine / SSH**: Compute Engine is Google's VMs, a rented computer in their datacenter. **SSH** (Secure Shell) is the standard way to get a terminal on a remote machine. The VM here is only a workbench, a place to hold files and type commands. It does **not** do the processing; the serverless Spark environment does.

**Two terminals, don't mix them:** *Cloud Shell* (a small free VM every account gets) vs *SSH-in-browser* (the workbench VM, provisioned for this exercise). Steps 2 to 4 all run in the SSH window.

**Environment variables**: a named value the terminal remembers, so commands look it up instead of you retyping. `start.sh` reads `GCP_PROJECT`, `REGION`, `GCS_STAGING_LOCATION`, `JARS` rather than taking them as arguments. They live only in **that terminal session**; `export` is what makes them visible to programs you launch.

**Where the table actually lives**: in **BigQuery**, not in the terminal. `bq` is a *client*: it sends the query over the network, BigQuery runs it, the answer prints back. The same table is visible in BigQuery Studio. The terminal was one of several doors.

```
campaigns.avro (bucket)
  → Spark reads it, reads its schema
  → creates loadavro.campaigns in BigQuery
  → bq query asks BigQuery to show it
```

### Expected noise

- `WARN FileStreamSink: Assume no metadata directory`: harmless, ignore.
- "Batch job has failed", this is expected here. Wait, then run the command again.

### Batch vs Streaming, Pub/Sub and Dataflow

**The split that runs through the whole ETL topic:**

| | Batch processing | Streaming processing |
|---|---|---|
| Data | A **fixed set** of stored data | A **continuous flow** from various sources |
| Suited to | **Payroll, billing** | **Fraud detection, intrusion detection**: real-time |

Payroll does not need to know about a paycheque three seconds after the hour is worked. Fraud detection is worthless if it tells you tomorrow.

#### The streaming ETL shape

```
event data → Pub/Sub → Dataflow → BigQuery (analytics, near real-time insights)
             (ingest)  (transform  → Bigtable (NoSQL storage)
                        + enrich)
```

This is the properly drawn answer to the earlier "website clicks into BigQuery" question (see open questions: **a row changed in a database → Datastream; an app emitted an event → Pub/Sub**).

#### Pub/Sub, the fan-out hub

Pub/Sub acts as a **central hub**: it receives events from various sources and distributes them to the systems that care.

**Example, an HR onboarding process:** an event `New employee` (or `New contractor`) arrives, and Pub/Sub delivers it to:
- access card activation
- facilities
- account provisioning

**Why this matters:** the HR system does not know those three systems exist. It publishes "new employee" and stops caring. Add a fourth consumer next year, laptop ordering, and **nothing in HR changes**; the new system just subscribes.

That is **decoupled, asynchronous communication** with reliable delivery. The alternative is a direct integration from HR to every downstream system, rebuilt every time one is added.

#### Dataflow

Built on the **Apache Beam** programming framework. The headline: **one unified framework for both batch AND streaming**. Write the pipeline once, it handles either, that is what "unified approach simplifies development" means.

- Languages: **Java, Python, Go**
- Features: **pipeline runner**, **serverless execution**, **templates**, **notebooks**
- Integrates with the other Google Cloud services

**The code shape** (Beam, streaming Pub/Sub → BigQuery):

```python
ReadFromPubSub()   →   beam.Map(parse_fn)   →   WriteToBigQuery()
```

`ReadFromPubSub` retrieves messages, `beam.Map()` applies the parsing transformation, `WriteToBigQuery` loads the result, **creating the table if necessary and appending** new data. Read, transform, write: genuinely most Dataflow pipelines.

#### Dataflow templates

- **Reusable pipelines** for recurring work
- **Separate the pipeline design from its deployment**: easier to manage and update
- **Parameters** customise one pipeline for different inputs
- Deployable through various methods
- **Google provides pre-built templates** for common scenarios

Same idea as the `GCSTOBIGQUERY` Dataproc template used in the Serverless Spark exercise: supply parameters rather than write code. (Different product, same pattern, do not confuse Dataflow templates with Dataproc templates.)

### Bigtable in Data Pipelines

An excellent choice for **streaming data pipelines that require millisecond-level latency analytics**.

- **Wide-column data model** with **column families**: flexible schema design
- **Row keys serve as the index**: fetch by key, fast
- **High throughput, low latency**
- Suited to: **time series data, IoT, financial data, machine learning**, especially with large datasets

**Where it sits:** `Pub/Sub → Dataflow → BigQuery (analytics) or Bigtable (NoSQL)`.

The split with BigQuery: BigQuery answers *"how many last quarter"* (ad-hoc scans). Bigtable answers *"what is this sensor's current reading"*, one known key, sub-10ms, millions of times a second. Consistent with the storage decision tree from learning notes 1 (analytical + NoSQL → Bigtable).


### Spark batch, more detail

**Three runtimes**, not just serverless: **Compute Engine** (VM clusters, can use HDFS on persistent disks), **GKE** (containers), **Serverless** (no cluster at all). Cloud Storage replaces disk-based HDFS in all of them, so disk-based HDFS is no longer needed.

**Spark's four components** (worth knowing that all four belong to Spark):

| Component | For |
|---|---|
| Spark SQL | structured data |
| Spark Streaming | real-time data |
| MLlib | machine learning |
| GraphX | graph processing |

Languages: R, SQL, Python, Scala, Java.

**Serverless has two execution modes:**

| Mode | How you start it | For |
|---|---|---|
| **Batches** | `gcloud` command | automated / scheduled jobs |
| **Interactive sessions** | JupyterLab (local or in Google Cloud) | exploration, notebook development |

A session's life cycle: create (runtime version, network) → active, kernel idle/busy → shutdown, manually or by inactivity timeout.

**Supporting services behind serverless Spark:** the **Spark history server** and **Dataproc Metastore** hold job history and table metadata after the ephemeral cluster is gone. That is the answer to "where did my job logs go when the cluster was deleted".

### GUI tools

**Dataprep (by Trifacta)**: serverless, no-code wrangling.
- Connects to a source, you click columns and apply **pre-built transformation functions**, chained into a **recipe**.
- **Visual preview**: you see the effect of a transform *before* applying it.
- **Intelligent suggestions**: it spots patterns and offers "extract this value", "replace this pattern".
- ⚠️ **It executes on Dataflow.** Dataprep is the front end; Dataflow is the engine. That is *why* it counts as serverless. Scheduling and monitoring included.

**Cloud Data Fusion**: GUI enterprise data integration.
- Drag-and-drop **Studio** canvas, pre-built transformations, no coding.
- Connects **on-premises and cloud** sources, the "hybrid / multicloud" trigger word.
- **Extensible with custom plugins** (CDAP).
- ⚠️ **It executes on Hadoop/Spark clusters.** That is *why* it is not serverless, a cluster is running underneath.
- Example pipeline: two SAP tables → **Join** → one leg to a Cloud Storage bucket, the other leg through **Add-Datetime** → BigQuery. Data can be previewed at each stage.

**The pairing worth remembering:** both are GUI front ends over an engine you already know, Dataprep over **Dataflow**, Data Fusion over **Spark**. That single fact explains the serverless row of the table below.

### Summary, the four ETL services

| | **Dataprep** | **Cloud Data Fusion** | **Dataproc** | **Dataflow** |
|---|---|---|---|---|
| **Applications** | Data wrangling | Data integration | ETL workloads | ETL workloads |
| **Integrations** | Cloud Storage, BigQuery, others | hybrid, multicloud environments | Cloud Storage, BigQuery, other OSS | **Pub/Sub**, Cloud Storage, BigQuery, other OSS |
| **Open source** | - | **CDAP** | **Apache Hadoop, Spark**, other OSS | **Apache Beam** |
| **Velocity** | **Batch only** | Batch, stream | Batch, stream | Batch, stream **(recommended)** |
| **Serverless** | **yes** | **no** | **no** (except Serverless Spark) | **yes** |

⚠️ **Two rows that are easy to miss:**
- **Dataprep is batch only.** The only one of the four that cannot stream. "Wrangling" **and** "streaming" in the same requirement rules it out.
- **Serverless is only Dataprep and Dataflow.** Cloud Data Fusion is a flat no; Dataproc is no except for Serverless Spark.

Only Dataflow lists **Pub/Sub** as an integration, consistent with it being the streaming default.

**Trigger words:**

| Requirement says | Service |
|---|---|
| "wrangling", cleaning messy data visually | **Dataprep** |
| "hybrid", "multicloud" integration, CDAP | **Cloud Data Fusion** |
| "existing Hadoop / Spark workloads" | **Dataproc** |
| "batch and streaming", a new pipeline | **Dataflow** |

## Hands-on: streaming taxi ride data into BigQuery with a Dataflow template and charting it

NYC taxi fleet, monitored in near real time. This version does **not** use Pub/Sub:

```
CSV in Cloud Storage → Dataflow template "Cloud Storage Text to BigQuery (Stream)" + JavaScript UDF → BigQuery taxirides.realtime → Data Studio dashboard
```

- **Step 1:** dataset `taxirides`, table `realtime`, schema defined up front, **partitioned on `timestamp`**.
- **Step 2:** copy three files into your bucket: `schema.json` (column types), `transform.js` (the UDF that turns each CSV line into a row), `rt_taxidata.csv` (the data).
- **Step 3:** Dataflow job from a **template**: no Beam code, only parameters. 1 worker, max 2, e2-medium.
- **Steps 4 to 5:** query while rows arrive; per-minute rides, revenue, passengers; save the query.
- **Step 6:** **stop the job.** A streaming job never ends on its own and bills until stopped.
- **Steps 7 to 8:** Data Studio combo chart from the saved query; time series chart from a custom query.

### Gotchas hit

- **First job failed: "Startup of the worker pool in us-central1-f failed to bring up any of the desired 1 workers."** Job status said Running, but Current workers stayed 0 and the graph boxes turned Failed. Cause: no e2-medium capacity in that zone. Fix: Stop → Cancel, **Clone**, new name `streaming-taxi-pipeline-2`, set **Worker zone** `us-central1-a` (Optional Parameters, not the Regional endpoint dropdown). Ran first time.
- **"Job successfully created" ≠ working.** Proof is Current workers = 1 and no red logs.
- **Form field names differ from the exercise instructions** (e.g. "The GCS location of the text you'd like to process"); `gs://` is pre-filled, paste paths without it.
- **Default machine type is N1**; unchecking the box still shows N1 until you change Series to E2.
- **Gemini Cloud Assist panel** opens from the icon on an error row; Cancel it, not needed.
- **Chart types is its own row** in the Data Studio properties panel; the search box there searches settings, not charts.
- **Data is from 2022**: a replayed sample file, not live traffic.

### The pipeline graph, read

`TextIO.Read / ReadFromSource` → `ApplyUDFTransformation` (transform.js) → `ConvertJsonToTableRow` → `InsertIntoBigQuery`, with a side path `WrapInsertionErrors → Flatten → WriteFailedRecords` so bad rows are set aside instead of crashing the job. Beam's read → transform → write, plus a dead-letter path.

## Automation Patterns and Options for Pipelines

The pipelines are built. This part is about **who presses run**: on a clock, in a sequence, or when something happens.

Topics covered here:
1. **Automation patterns** and options for pipelines.
2. **Cloud Scheduler** and **Workflows**.
3. **Cloud Composer**.
4. **Cloud Run functions**.
5. **Eventarc**.

**The split to watch for, anchored to Power Automate:**

| Trigger | Power Automate equivalent | Google Cloud |
|---|---|---|
| On a clock | Scheduled cloud flow | **Cloud Scheduler** |
| Run steps in order, with branching and retries | The flow's action sequence | **Workflows** (light) / **Cloud Composer** (heavy, Apache Airflow) |
| Something happened | Automated flow ("when a file is created") | **Eventarc** routes the event → **Cloud Run functions** does the small job |

The tiebreaker worth knowing is **Workflows vs Cloud Composer**: both orchestrate steps, but Composer is managed Airflow for complex, many-task DAGs, and costs an always-on environment. The same "simplest thing that works" logic applies as with `bq load` vs Serverless Spark.

### Automation patterns

Two examples, one per trigger type:

| | **Scheduled ELT** | **Event-driven ETL** |
|---|---|---|
| Trigger | A defined schedule | A file uploaded to Cloud Storage |
| Work | Extract from BigQuery → transform with **Dataform** → load back into BigQuery | Batch process with **Dataproc** |
| Lands in | BigQuery | Cloud Storage |

The ELT one never leaves the warehouse (compute happens inside BigQuery, the ELT test from the Dataform notes). The ETL one does its work outside, on Dataproc, before landing.

**The service map:**

| Need | Service |
|---|---|
| Scheduled or one-off jobs | **Cloud Scheduler**, **Cloud Composer** |
| Orchestration (many steps, dependencies) | **Cloud Composer** |
| React to an event | **Cloud Run functions**, **Eventarc** |

⚠️ Composer appears in two rows: it can schedule *and* orchestrate. Scheduler only fires on a clock, it does not manage steps. **Workflows** is not named in this map. It shows up under Cloud Scheduler below.

### Cloud Scheduler

Invokes workloads at **recurring intervals**. You set both the **frequency** and the **time of day**. Power Automate equivalent: the Recurrence trigger.

**What it can trigger:**
- HTTP/S calls
- App Engine HTTP calls
- Pub/Sub messages
- Workflows

It does not run the work itself. It fires a trigger at something else that does.

#### Example: scheduling a Dataform SQL workflow

Cloud Scheduler job → runs a **Workflows** definition written in a **YAML** config file → that workflow makes two calls to Dataform:

1. **Create a compilation result.** Dataform compiles the SQLX project into the actual SQL it will run (the "compile" step you saw in the Dataform exercise before executing).
2. **Create a workflow invocation** using that compilation result. This is the actual run.

**Included tags** limit which parts of the project execute. Tag the daily tables `daily`, and the scheduled job runs only those, not the whole project.

⚠️ So "Scheduler + Workflows" is already a pair: Scheduler is the clock, Workflows is the list of steps. That is also why Workflows appears in the trigger list above.

### Cloud Composer

A **central orchestrator** across Google Cloud, **on-premises and multicloud**. It is **managed Apache Airflow** (see the open source table in open questions).

**Airflow's three building blocks:**

| Term | Meaning |
|---|---|
| **Operator** | A ready-made template for one kind of action: "load a GCS file into BigQuery", "run a BigQuery query", "submit a Dataproc job" |
| **Task** | One use of an operator in your pipeline, with its parameters filled in |
| **Dependency** | Which task must finish before which starts |

Plus built-in **triggering, monitoring, logging**, and **error handling and retries** per task.

#### DAG: directed acyclic graph

The pipeline definition, written in **Python**.
- **Graph**: tasks are boxes, dependencies are arrows.
- **Directed**: arrows go one way (A then B).
- **Acyclic**: no loops. Nothing can depend back on itself, so there is always a clear start and end.

Same idea Dataform used: the `${ref()}` dependencies formed a DAG, which is how it knew the table waited for the view. Dataform infers the DAG from SQL; in Airflow you **write the arrows yourself**.

**Lifecycle:** write the DAG in Python with operators → **deploy** it to Composer (drop the file in the environment's DAGs folder in Cloud Storage) → Composer **parses and schedules** it → Composer **runs** the tasks with retries and monitoring.

#### Example: a data analytics DAG

```
GCS file → load into BigQuery → JOIN with an existing BigQuery table → insert into a new table → Dataproc transform
```

Shape of the Python (illustrative):
```python
load    = GCSToBigQueryOperator(...)       # file from Cloud Storage into BigQuery
join    = BigQueryInsertJobOperator(...)   # JOIN, write results to a new table
process = DataprocSubmitJobOperator(...)   # further transform on Spark

load >> join >> process                    # the dependencies: the arrows of the DAG
```

**The point of the example:** three different services (Cloud Storage, BigQuery, Dataproc) in one pipeline with one place to watch it. That is what "orchestrator" means. Workflows could chain these too; Composer wins when there are many tasks, complex dependencies, systems outside Google Cloud, or a team that already writes Airflow.

⚠️ **Cost trade-off:** a Composer environment is an always-on Airflow cluster, billed while idle. Two steps on a schedule does not need it; Scheduler + Workflows is the simpler answer.

**Power Automate analogy:** a DAG is a flow's action sequence with "run after" settings, retries per action, and run history. The difference is the flow is drawn, the DAG is Python code in a file.

### Cloud Run functions

**A small piece of code that runs when an event happens.** Serverless: no machine to manage, it spins up per event and you pay per run. Multiple languages (Python, Node.js, Go, Java, and others).

**Event sources:**
- HTTP requests
- Pub/Sub messages
- Cloud Storage changes (file uploaded, deleted)
- Firestore updates
- Custom events through **Eventarc**

**Power Automate analogy:** an automated flow ("when a file is created in a folder"), except the action is your own code.

#### Example: file upload starts a Dataproc job

```
file uploaded to Cloud Storage → Cloud Run function catches the event → calls the Dataproc API → Dataproc runs a workflow template with the file as input → result saved to Cloud Storage
```

The function does almost nothing itself: it catches the event and passes the file name to Dataproc. Dataproc does the heavy work. Same idea as Composer: **the trigger is not the worker.**

This is the event-driven ETL example from the automation patterns section above, now with the missing piece named: the function is what connects "file landed" to "Dataproc runs".

⚠️ **Dataproc workflow template ≠ Dataflow template ≠ Workflows.** A Dataproc workflow template is a saved set of Spark/Hadoop jobs, optionally with a cluster that is created for the run and deleted after.

**When not to use it:** long or heavy processing. Functions are for short glue code; hand the real work to BigQuery, Dataproc or Dataflow.

### Eventarc

**An event router.** It does not run code; it catches an event from a source and delivers it to a target.

- **Sources:** Google Cloud services, third-party systems, custom events via Pub/Sub.
- **Targets:** Cloud Run functions, and others.
- **CloudEvents format:** every event arrives in one standard message shape, whatever produced it, so targets don't need custom parsing per source.
- **Loosely coupled:** the source doesn't know who reacts. Add a new reaction without touching the source (same idea as the pub/sub pattern).
- Can react to **audit log** events, so almost anything that happens in Google Cloud can become a trigger.

#### Example: react to a BigQuery insert

```
rows inserted into a BigQuery table → Cloud Audit Log event → Eventarc catches it → target does something
```

Possible actions: rebuild a dashboard, retrain an ML model, any custom action.

**Why it matters:** BigQuery has no "when a row is inserted" trigger of its own. The audit log is the side door, and Eventarc watches it.

**Power Automate analogy:** the trigger part of an automated flow ("when an item is created"), separated out as its own service. Cloud Run functions is the action.

### Summary, the four automation options

| | **Cloud Scheduler** | **Cloud Composer** | **Cloud Run functions** | **Eventarc** |
|---|---|---|---|---|
| Trigger | Scheduled or manual | Scheduled or manual | **Event-driven** | **Event-driven** |
| Coding effort | **Low** (YAML) | **Medium** (Python) | Multiple languages | **Language agnostic** |
| Serverless | yes | **no** | yes | yes |

⚠️ **Composer is the only one that is not serverless**, the always-on Airflow environment from earlier. Same shape as the ETL summary, where Data Fusion and Dataproc were the non-serverless ones.

**Trigger words:**

| Requirement says | Service |
|---|---|
| "run every night at 2am", simple | **Cloud Scheduler** |
| "complex dependencies", "Airflow", "many tasks across systems" | **Cloud Composer** |
| "run code when a file is uploaded" | **Cloud Run functions** |
| "route events", "CloudEvents", "loosely coupled", "BigQuery insert / audit log" | **Eventarc** |

## BigQuery and SQL hands-on

Six hands-on exercises in BigQuery and SQL, from first queries to a small report.

## Hands-on: querying bike trips in BigQuery, then moving the results into Cloud SQL

London bikeshare trips, 83,434,866 rows, one row per trip.

```
BigQuery (analyze 83M trips) → CSV to laptop → Cloud Storage bucket (loading dock) → Cloud SQL MySQL (edit single rows)
```

- **Cloud SQL instance first:** it takes 5 to 10 minutes to build, so start it before the BigQuery part. Enterprise edition, Development preset (4 vCPU, 16 GB), MySQL 8.0, ID `my-demo`. A bigger preset gets the practice project shut down.
- **BigQuery part:** `SELECT` one column (83,434,866 rows) → `WHERE duration>=1200` (seconds, so 20+ min: 26,441,016 rows, ~30%) → `GROUP BY start_station_name` (954 rows, same as DISTINCT) → `COUNT(*) AS num ... ORDER BY num DESC`. Busiest start station: Hyde Park Corner, 671,688.
- **Export:** Save results → **Local download → CSV** (up to 10 MB). Other options: Google Drive CSV (1 GB), straight to **Cloud Storage** (the real-world choice, skips the laptop), or a **BigQuery table**.
- **Bucket:** name = project ID (bucket names are globally unique). Defaults: Standard class, US multi-region, public access prevention on.
- **Cloud SQL part:** Cloud Shell → `gcloud sql connect my-demo --user=root` (password does not echo) → `CREATE DATABASE bike;` → `USE bike;` + `CREATE TABLE london1 (start_station_name VARCHAR(255), num INT);` and `london2` → console **Import** CSV from the bucket → `DELETE`, `INSERT INTO`, `UNION`.

### Concepts clarified

- **Cloud SQL = cash register, BigQuery = accountant.** An app saves one order, one leave request, one bike unlock → Cloud SQL. "Total sales by region last year", "busiest station across 83M trips" → BigQuery. Real systems have both: app writes to Cloud SQL, data is copied (e.g. Datastream) to BigQuery, dashboard reads BigQuery.
- **Instance → database → table** (Cloud SQL) vs **project → dataset → table** (BigQuery). Excel: instance = folder, database = workbook, table = sheet. BigQuery has no instance because it is serverless.
- **Project** = top container for services, billing, access. Like a Power Platform environment. `bigquery-public-data` is Google's project, shared publicly.
- **Public datasets:** Google downloads open data (here Transport for London's files), loads it into BigQuery, pays for storage; you pay only for queries. Same idea as Analytics Hub: query in place, no copy.
- **Starring** a project is only a bookmark. The full `project.dataset.table` name in backticks works without it.
- **Query results** sit in a temporary table for ~24h (rerun is free), but that is not a table you own.
- **Cloud SQL imports only from a bucket**, not from a laptop.
- **MySQL needs the schema first** (`CREATE TABLE` with column types); data must fit the types.
- **`INSERT INTO`** = SQL's version of Power Apps `Patch()`: one new row.
- **UNION dedups whole rows only.** Hyde Park appears twice: same name, but starts 671,688 ≠ ends 671,680. UNION stacks rows, it does not connect the tables. The exercise's query mixes starts and ends without a label; real fix is a `'start'`/`'end'` label column, or a JOIN on station name for `station | starts | ends`.

### Gotchas hit

- **Header row imported as data.** `COUNT(*)` gave 955 not 954. The header `start_station_name,num` loaded as a row; the text "num" in an INT column **silently became 0**. No error ≠ correct load: always check row counts after a load. Fix here: `DELETE ... WHERE num = 0`, which is only safe because no real station has 0 trips; run the same WHERE as a SELECT first.
- **CSV file names don't match in the exercise instructions** (`start_station_name.csv` vs `start_station_data.csv`). Use whatever was uploaded.
- **Cloud SQL page:** skip the top "Create free instance" form (8 vCPU / 64 GB trial). Use **Create Development instance**, then switch edition to Enterprise and version to MySQL 8.0.
- **BigQuery console is newer than the exercise instructions:** no "+ Add data" in Explorer. Skipped starring, ran queries with full names.
- **Console confirmed the rename:** Cloud SQL showed "Knowledge Catalog integration", the new name for what these notes call Dataplex.
## Hands-on: loading a file into BigQuery from the console and querying it

Load your own data into BigQuery and query it. The previous exercise only read Google's tables.

```
file in Google's bucket (baby-names/yob2014.txt) → create dataset babynames → create table names_2014 with schema → query it
```

- **Step 2:** public query `bigquery-public-data.samples.natality`, 10 heaviest babies. Validator said **3.49 GB** before running (the dry run estimate).
- **Step 3:** Explorer → ⋮ next to project → **Create dataset** `babynames`. BigQuery's version of `CREATE DATABASE`.
- **Step 4:** ⋮ next to dataset → **Create table** from Google Cloud Storage, CSV, table `names_2014`, schema **Edit as text** `name:string,gender:string,count:integer`.
- **Step 6:** top 5 boys' names 2014: **Noah 19,286**, Liam, Mason, Jacob, William. Scanned **622 KB**.

### Concepts clarified

- **Cost = bytes scanned, shown before you run.** On-demand is roughly $6.25 per TB; first 1 TB of queries per month free. 3.49 GB ≈ 2 cents. The practice project is a **Sandbox** (free mode, no billing).
- **LIMIT does not lower cost.** 10 heaviest babies still reads every row's weight. Selecting fewer columns lowers cost; **Preview tab is free**.
- **Two baby things, unrelated:** natality = Google's project, dataset `samples`. `babynames` = new empty dataset in my project, filled from a bucket file.
- **Schema = `column:type`, matched to the file by position.** Line `Noah,M,19000` → name, gender, count. Wrong order fails the load. Type matters for sorting: as text "9000" sorts above "19000".
- **Auto detect** lets BigQuery guess the types. Fine for exploring; defining it yourself is safer.
- **No header in this file**, so no junk row. If a file has one, set **Header rows to skip = 1** (Advanced options). The MySQL import in the Cloud SQL exercise had no such setting and loaded the header as a 0 row.
- **Table name without a project** (`babynames.names_2014`) means your own project. Other projects need the full `project.dataset.table`.
- **Data quality:** all top 10 natality rows weigh exactly 18.0007436923 lb, a cap or placeholder value, plus nulls in state. Question outliers before trusting a "heaviest" answer.

### Gotchas hit

- **File picker said "No buckets found."** It only lists buckets in your own project; the source bucket is Google's. Type the path into the field instead of browsing.
- **Cloud Shell setup in the exercise instructions is not needed**; everything is in the console.
## Hands-on: loading and querying a BigQuery table from the command line with bq

Same idea as the console exercise, typed as `bq` commands. The file comes from the internet instead of a bucket.

```
ssa.gov website → Cloud Shell disk → your dataset babynames → table names2010 → query → delete
      wget + unzip                        bq mk + bq load          bq query     bq rm -r
```

| Command | Does | Console equivalent |
|---|---|---|
| `bq show project:dataset.table` | schema, row count, size (free) | table Details tab |
| `bq head -n 5 dataset.table` | first rows (free) | Preview tab |
| `bq query --use_legacy_sql=false '...'` | run SQL | Run button |
| `bq ls` / `bq ls project:` | list datasets in mine / another project | Explorer pane |
| `bq mk babynames` | create dataset | Create dataset |
| `bq load babynames.names2010 yob2010.txt name:string,gender:string,count:integer` | create table + load in one step | Create table form |
| `bq rm -r babynames` | delete dataset **and** its tables | Delete |

- **Shakespeare:** `bigquery-public-data:samples.shakespeare`, 164,656 rows, one per word per play. `LIKE "%raisin%"` found praising, raising, raisins... `= "huzzah"` returned nothing, which is correct.
- **Names 2010:** 34,111 rows (the exercise text says 34,073; the government revises past years, so a fresh download differs).
- **Three ways to access BigQuery:** Web UI, `bq` command line tool, REST API. GLib and GStreamer are unrelated Linux libraries, not ways into BigQuery.

### Concepts clarified

- **Colon vs dot:** `bq` commands write `project:dataset.table`; inside SQL it is all dots.
- **`--use_legacy_sql=false` is a flag, not SQL.** BigQuery has two dialects: legacy SQL (old, `[project:dataset.table]`) and GoogleSQL / standard SQL (current, backticks). The console defaults to standard; the `bq` tool has defaulted to legacy. Other ways to get standard SQL: `#standardSQL` as the first line of the query, or set it once in `~/.bigqueryrc` (`[query]` then `--use_legacy_sql=false`). Simple SELECT/WHERE/ORDER BY/LIMIT works in both dialects, so the flag only matters when syntax differs.
- **String matching is case-sensitive:** `praising` and `Praising` came back as separate rows. Use `LOWER(word)` to merge.
- **Cloud Shell** = a small free temporary VM in the browser, 5 GB home disk. Scratch workspace like a library laptop, not storage. Real files go in a bucket, real tables in BigQuery.
- **Bucket ≠ dataset.** Bucket holds files (Cloud Storage), dataset holds tables (BigQuery). This exercise used no bucket at all.
- **Who names the columns:** the file has no header, only values. The government's `NationalReadMe.pdf` in the zip says the positions are name, sex, count; I named them in the `bq load` schema (`gender`, not "sex"). Before loading any file: `head` it and read its docs (header? delimiter? what each position means).
- **Two routes, same table:**

| | Via bucket | Via Cloud Shell disk |
|---|---|---|
| File afterwards | kept | gone when the VM or practice project ends |
| Size | practically unlimited | 5 GB |
| Automate on a schedule | yes | no, typed by hand |
| Use for | real pipelines (bucket = data lake, BigQuery = warehouse) | quick one-off tests |

- **Sandbox tables expire after 60 days** (`bq show` showed Expiration 2 months out).
- **`bq rm -r`:** without `-r`, `bq` refuses to delete a dataset that still has tables. Safety catch.
- **Rarest names all have count 5:** the government omits names given to fewer than 5 babies, for privacy.

### Gotchas hit

- **`wget` from ssa.gov worked** this time. It can be blocked by the site; if so, the file needs another route (e.g. a bucket copy).
- **The zip now runs to `yob2025.txt`**, more years than the exercise text shows.

## Hands-on: checking raw web data for duplicates and answering business questions in SQL

Google Merchandise Store, Google Analytics logs, `data-to-insights.ecommerce`. One row ≈ one hit by one visitor.

```
all_sessions_raw (has duplicates) → find them → all_sessions (clean copy) → business questions
```

| Step | Query idea | Result |
|---|---|---|
| Duplicates in raw | `GROUP BY` all 32 columns `HAVING COUNT(*) > 1` | **615** duplicated records (some stored 2, 3, 5 times) |
| Duplicates in clean | `GROUP BY 1..12` (12 key columns) `HAVING row_count > 1` | **0 rows** = pass |
| Visitors | `COUNT(*)` vs `COUNT(DISTINCT fullVisitorId)` | 21,493,109 rows, 389,934 visitors (~55 rows each) |
| Channels | unique visitors `GROUP BY channelGrouping` | Organic Search 211,993 (over half), Direct 75,688, Referral 57,308, Paid Search 11,865, Display 3,067 |
| Product list | `GROUP BY v2ProductName` | **633** products |
| Top viewed | `WHERE type = 'PAGE'` ... `ORDER BY product_views DESC LIMIT 5` | |
| Unique views | `WITH` per person + product, then count | |
| Orders, units, avg | `COUNT(productQuantity)`, `SUM(productQuantity)`, SUM / COUNT | YouTube Bottle Infuser ≈ 9.38 per order; the most viewed product also got the most orders |

### Concepts clarified

- **Look before querying:** Schema tab = columns and types, Details = row count and size (5.63 GB is a size, not rows), Preview = sample rows, free. Nobody memorizes column names: Schema tab, docs / data dictionary, autocomplete. Power BI equivalent: the Data pane.
- **WHERE vs HAVING:** WHERE filters rows before grouping; HAVING filters groups after, so it can see `COUNT(*)`.
- **More GROUP BY columns = stricter piles.** Rows share a pile only if they match in every listed column. Grouping by all columns means only exact copies group together.
- **0 rows from a duplicate check is a pass:** every pile has 1 row, so `HAVING > 1` keeps nothing.
- **Key check beats all-columns check.** If the 12 key columns have no duplicates, all 32 can't either (more columns only split piles). The reverse is not true: same visitor + product + time with a different city is 0 duplicates by all columns but 1 by the key, which is the inconsistency you want to catch.
- **Real-world dedup:** group on the business key (e.g. `order_id`), not everything. Shortcuts: `COUNT(*)` vs `COUNT(DISTINCT TO_JSON_STRING(t))`; remove with `SELECT DISTINCT *`; keep latest with `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY loaded_at DESC) = 1`. Automate it in the pipeline, e.g. Dataform `uniqueKey` assertion.
- **Why duplicates matter:** GA stores money × 1,000,000 (304,320,000 = $304.32). A duplicated transaction row doubles revenue in a plain SUM.
- **`GROUP BY 1,2,3`** = column positions in the SELECT.
- **`#` starts a comment** in BigQuery SQL.
- **A column alias is only a label.** Step 4 named `COUNT(*)` "product_views" but it counted every hit type; `WHERE type = 'PAGE'` later made it true.
- **COUNT(DISTINCT x)** counts each person once. Sara browsing 30 pages then coming back direct = 1 visitor in Organic Search and 1 in Direct, so channel totals (~404k) exceed the overall 389,934.
- **ORDER BY channelGrouping DESC** sorts by name Z→A, not by size. Use `ORDER BY unique_visitors DESC` for biggest first.
- **WITH (CTE)** = a named temporary result inside one query. Its columns are whatever its SELECT returns; the next SELECT uses the name in FROM like a real table; nothing is stored. Same result possible with a subquery in FROM, or here simply `COUNT(DISTINCT fullVisitorId) ... GROUP BY v2ProductName`. WITH pays off when logic has 3 to 4 steps. Like a DAX `VAR ... RETURN`.
- **`COUNT(*)` counts every row; `COUNT(column)` skips nulls.** `productQuantity` is null on non-purchase rows, so `COUNT(productQuantity)` = orders and `SUM(productQuantity)` = units. Sara 1 + Tom 5 + Ali 4 = 3 orders, 10 units.
- **avg_per_order divides by orders, not rows.** Rows for one product: blank, 2, blank, 5. `COUNT(*)` = 4 views, `COUNT(productQuantity)` = 2 orders, `SUM` = 7 units, avg = 7 / 2 = 3.5. Dividing by all rows (7 / 4) would be units per view, a different number. A blank (NULL) quantity = the visitor looked and did not buy.
- **What to GROUP BY: the words after "per" or "each".** "Each person once per product" → `GROUP BY fullVisitorId, v2ProductName` (the step 8 WITH). Check: finish "one row in my result = one ___".
- **`fullVisitorId`** = anonymous browser cookie ID. **`v2ProductName`** = product name, "v2" because GA replaced an older field.

### Gotchas hit

- **New BigQuery Studio UI:** no "+ Add data > Star a project by name", and Explorer search for `data-to-insights` found 0 results even with "Search all projects in your organization" on. Workaround: type `` `data-to-insights.ecommerce.all_sessions_raw` `` in a query tab and **Ctrl+click** it. The table opens with Schema / Details / Preview, and the project then shows in Explorer.

## Hands-on: fixing broken SQL queries one error at a time in BigQuery

Same store data, table `data-to-insights.ecommerce.rev_transactions` (sessions that include transactions). Scenario: a new analyst hands over broken queries; fix them one error at a time.

| Step | Business question | Errors fixed on the way |
|---|---|---|
| 2 | Unique visitors who reached checkout | no columns in SELECT, typo in the project name, legacy SQL table name, missing comma, COUNT without GROUP BY, COUNT without DISTINCT, no filter |
| 3 | Cities with the most visitors and transactions | empty GROUP BY, plain column next to an aggregate, sort, calculated average, WHERE on an aggregate |
| 4 | Distinct products per category | GROUP BY with no aggregate, COUNT without DISTINCT, NULL product names |

Where each chain ends:

```sql
-- Step 2
SELECT COUNT(DISTINCT fullVisitorId) AS visitor_count, hits_page_pageTitle
FROM `data-to-insights.ecommerce.rev_transactions`
WHERE hits_page_pageTitle = "Checkout Confirmation"
GROUP BY hits_page_pageTitle

-- Step 3
SELECT geoNetwork_city,
  SUM(totals_transactions) AS total_products_ordered,
  COUNT(DISTINCT fullVisitorId) AS distinct_visitors,
  SUM(totals_transactions) / COUNT(DISTINCT fullVisitorId) AS avg_products_ordered
FROM `data-to-insights.ecommerce.rev_transactions`
GROUP BY geoNetwork_city
HAVING avg_products_ordered > 20
ORDER BY avg_products_ordered DESC

-- Step 4
SELECT COUNT(DISTINCT hits_product_v2ProductName) AS number_of_products, hits_product_v2ProductCategory
FROM `data-to-insights.ecommerce.rev_transactions`
WHERE hits_product_v2ProductName IS NOT NULL
GROUP BY hits_product_v2ProductCategory
ORDER BY number_of_products DESC
LIMIT 5
```

### Concepts clarified

- **Legacy vs standard SQL, from the table name alone:** square brackets + colon `[data-to-insights:ecommerce.rev_transactions]` = legacy. Backticks + dots = standard. A `#standardSQL` header with a legacy table name fails.
- **Runs ≠ useful.** `SELECT fullVisitorId FROM ...` works but dumps every row with repeats in no order. A useful query usually has an aggregation, a LIMIT or an ORDER BY.
- **Missing comma = silent alias.** `SELECT fullVisitorId hits_page_pageTitle` means `fullVisitorId AS hits_page_pageTitle`: 1 column of visitor IDs with the wrong label, no error. `AS` is optional in SQL.
- **Plain column next to an aggregate:** must be in GROUP BY or wrapped in an aggregate (SUM, COUNT, AVG, MAX, MIN). San Jose rows with transactions 1, 2, 4: `GROUP BY geoNetwork_city` makes 1 row, `SUM` gives 7, a plain `totals_transactions` has 3 values for 1 cell, so it errors.
- **"How many different X" = `COUNT(DISTINCT X)`.** Office rows Pen, Pen, Pen, Notebook: `COUNT` = 4, `COUNT(DISTINCT)` = 2. Same bug as `COUNT(fullVisitorId)` in Step 2.
- **WHERE vs HAVING, by run order.** SQL runs FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY, not in written order. WHERE filters single rows before any SUM or alias exists. HAVING filters the grouped results. BigQuery lets HAVING and ORDER BY use a SELECT alias (`HAVING avg_products_ordered > 20`); WHERE never can.
- **Alias** = a name created with `AS`, for a calculation (`SUM(...) AS total_products_ordered`) or for a renamed real column (`fullVisitorId AS visitor`). Neither name exists in the table, so `WHERE visitor = ...` fails.
- **You query tables, not datasets.** A dataset is only a folder. Several tables in one query = `JOIN`, even across datasets or projects.

## Hands-on: building a report table in Data Studio on top of a BigQuery table

**Data Studio** = Google's free dashboard tool, the Power BI equivalent. Renamed **Looker Studio** in 2022; the app banner now says "Looker Studio is now called Data Studio". Not the same product as **Looker**.

| | Data Studio (Looker Studio) | Data Studio Pro | Looker |
|---|---|---|---|
| What | self-service dashboards | same app + team workspaces, Cloud project ownership, support | enterprise BI platform |
| Cost | free | $9 per user per project per month | paid licence |
| Who builds | anyone, drag and drop | same | data team defines a LookML semantic model first |
| Power BI match | Power BI Desktop reports | Power BI Pro workspaces | a governed shared semantic model |

```
BigQuery data-to-insights.ecommerce.sales_report → Data Studio BigQuery connector → report table
```

`sales_report` = one row per product: name, productSKU, stockLevel, total_ordered, restockingLeadTime, ratio, sentimentScore, sentimentMagnitude.

- **Step 1:** `lookerstudio.google.com` → Create a report → BigQuery connector → Shared projects → practice project → shared project name `data-to-insights` → `ecommerce` → `sales_report` → Add → Add to report. Add a chart → Table, drag **ratio** into Dimension, click the **123** icon on its chip → Data type → Numeric → **Percent**. Then delete the table.
- **Step 2:** rename the report `Ecommerce Product Operations Report`; text box `Product Inventory Watchlist` at 32px; Insert → Table with dimension productSKU, metrics stockLevel, ratio, restockingLeadTime, sort ratio descending, Style → Wrap text; then add **name** above productSKU and **total_ordered** below restockingLeadTime; View.
- The last part is described as adding an interactive filter, but it only adds a dimension and a metric. No filter control gets built.

### Concepts clarified

- **Dimension vs metric.** Dimension = what you group by, one row per value (text, dates, IDs; SQL `GROUP BY`; Power BI Rows / Axis). Metric = a number that gets aggregated (SUM, COUNT, AVG; Power BI Values). Example: make **name** the dimension instead of SKU, and a product with sizes S, M, L at stock 2, 5, 3 becomes 1 row with stockLevel 10. In `sales_report` every SKU is 1 row, so nothing visibly sums.
- **Changing ratio to Percent** is display formatting in the report, like a Power BI format string. The BigQuery column does not change.
- **Real-life role:** this is BI / reporting work (data analyst or BI developer). The data engineer builds `sales_report` and the pipeline that refreshes it, the same split as the M / dataflow side vs the visuals in Power BI.

### Gotchas hit

- **Searching "Data Studio" in the Cloud console** opens the **Data Studio Pro** product page. The practice account has no permission to create it. Do not start the trial; go to `lookerstudio.google.com`.
- **Stale temporary account.** Incognito still held the previous exercise's temporary account, so "Verify it's you" asked for the old account and the new password failed. Fix: close every incognito window, open a fresh one, sign in again from the practice project's console link.
- **Check the avatar before connecting data.** A photo = a personal account; the practice account shows a plain icon. Connect BigQuery only as the practice account.
- **Actions outside the instructions can get the practice account blocked.** Stick to the steps.

## Product renames and small differences worth knowing

BigLake is now **Lakehouse for Apache Iceberg**. Iceberg itself has its own section below.

| | Earlier | Now |
|---|---|---|
| Metadata management zones | Landing → Raw → Curated (3) | Raw → Curated (2) |
| Connection type when creating a BigQuery connection | searched as **Vertex AI** | searched as **Agent Platform** |
| Airflow product | Cloud Composer | managed service for Apache Airflow |

The 3-zone version is the fuller one. Both versions are worth knowing.

## Lakehouse for Apache Iceberg

### Three words, three ideas

| | What it is | What it can't do |
|---|---|---|
| **Lake** (bucket of files) | cheap, any format, any tool reads it | no `UPDATE`, no transactions, no schema enforcement |
| **Warehouse** (BigQuery) | real tables, fast, safe | data is locked inside BigQuery |
| **Lakehouse** | warehouse behaviour **on** lake storage | - |

### Apache Iceberg is a *specification*, not a library

It is a written standard for a metadata layer that sits beside your Parquet files. Like
PDF: Adobe wrote the format down, and Chrome, Word and Preview all implement it. Engines
implement Iceberg (BigQuery/Lakehouse, Spark, Snowflake, Trino, Databricks), and PyIceberg
is *an* implementation, not the thing itself.

**Why that matters:** the files in `gs://my-lake/sales/` are not Google's. Point Spark at
the same folder and it reads the same snapshots. Nobody owns your data.

### The folder layout, with real contents

```
gs://my-lake/sales/
├── data/      fileA.parquet     (id 1|100, 2|300, 3|250)  <- the rows, Parquet only
│              fileB.parquet     (id 2|500)                 <- new row written by an UPDATE
│              delete-1.parquet  (fileA, the id 2 row)      <- marks the old row as deleted
└── metadata/  v1.metadata.json  { columns, current-snapshot: snap-1 }
               v2.metadata.json  { columns, current-snapshot: snap-2 }
               snap-1  { data-files: [fileA] }
               snap-2  { data-files: [fileA, fileB], delete-files: [delete-1], parent: 1 }
```

**Three layers, three jobs:** `.parquet` holds the rows; a snapshot lists which files are in
that version; `vN.metadata.json` holds the columns and names the current snapshot.
(Simplified: a real snapshot points to a manifest list, which points to manifest files,
and those list the data files.)

### What an UPDATE actually does

`UPDATE sales SET amount = 500 WHERE id = 2` never edits fileA in place. It can run in one
of two ways:

- **Merge-on-read** (shown above): write a small delete file that marks the old `id=2` row,
  write fileB with the new row, then a new snapshot listing all three, then
  `v2.metadata.json` pointing at it. Readers apply the delete file, so `id=2` is read once,
  as 500.
- **Copy-on-write:** write a new copy of fileA with the changed row. The new snapshot lists
  the new file instead of fileA.

Either way:

- The old fileA stays on disk until its snapshot expires.
- fileB is **the same format** as fileA (Parquet). Iceberg only adds JSON/Avro metadata.
- The extra file is **not a second table**: same table, extra file.
- `v1` is never edited, so **time travel** = read `v1` instead and `id=2` is 300 again.
### Why not just rewrite the table?

1. **Size**: one row in a 40 TB table would mean reading and rewriting 40 TB.
2. **Readers mid-write**: an in-place rewrite shows a half-finished table; the pointer swap
   is atomic, so a query sees v1 or v2, never something between.
3. **History**: rewriting destroys the old state.
4. **Concurrent writers**: two appenders don't clobber each other; two rewriters do.

The cost is deferred, not avoided: many small files slow queries down, so you run
**compaction** to merge them. You still pay the rewrite, just batched, on your schedule.

### One snapshot per *commit*

Not per row, not per column. Two separate `UPDATE` statements = two snapshots; one statement
changing two columns = one snapshot; a `BEGIN TRANSACTION ... COMMIT` wrapping both = one
snapshot. Hence: 10,000 single-row UPDATEs = 10,000 tiny files. Batch them.

### Metadata cache and the external-vs-Lakehouse split

(Already covered in the BigLake section above: file size, row count, min/max
column stats, staleness configurable 30 min to 7 days, no cost estimation or preview.)

## Extra notes on Datastream, Lakehouse and Dataform

### Datastream, the allowlist follows the region

The Datastream IP allowlist depends on the region. `us-east4` and `us-west1` each have
their own five IPs, so a stream in a different region needs a different allowlist:
`<the five allowlisted IPs for your region>`.
The publication and slot names and staleness 0 do not change with the region.

### Lakehouse, two additions

- **Navigation trap:** the first step is to search *Agent Platform*. Searching it in the **console's
  global search bar** lands on the Agent Platform product page, which is wrong. The search
  must happen inside BigQuery's **+ Add data** panel, or skip it entirely: Explorer tree →
  **Connections** → ⋮ → **Create connection**. (Same shape as the Data Studio Pro trap.)
- **Add policy tag only appears in Edit schema mode.** The read-only Schema tab lets you tick
  fields but shows no tag button.

**A policy tag denies by default.** Access is a separate IAM role, **Fine-Grained Reader**
(`roles/datacatalog.categoryFineGrainedReader`), granted *on the tag*. Nobody holds it in
the practice project, including the table owner, which is why `SELECT *` fails while
`SELECT * EXCEPT(...)` succeeds. In production you grant it to the one group that needs the
columns and everyone else keeps querying the same table minus those fields.

**Do policy tags survive a schema change?** I did not test this hands-on. Check the
BigQuery docs before you change the schema of a tagged table, and re-check the tags after.
Teams usually define tags in Terraform and re-apply them on every schema change rather than
clicking them in.

### Dataform, one UI note

The initialized workspace ships with sample `first_view.sqlx` / `second_view.sqlx` files, ignore them, and
watch for a stray `: definitions` folder if a path gets typed with a colon.

**The idiom, seen three times in one session:** create a robot → grant the robot roles →
the robot does the work. Datastream's robot reads the database, Lakehouse's robot reads the
bucket, Dataform's robot writes to BigQuery. Recognise it on sight.


---
