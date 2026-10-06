# Learning notes 1: data engineering foundations

## 2026 Product Renames (read this first)

Google renamed several products in 2026, going generic-first. The notes below and
Google's own pages may use either form. Recognise both.

| Old name (used in notes below) | New name (2026) |
|---|---|
| Dataproc / Dataproc Serverless / Serverless for Apache Spark | Managed Service for Apache Spark |
| Cloud Composer | Managed Service for Apache Airflow |
| Dataplex | Knowledge Catalog |
| Looker Studio | Data Studio (renamed *back*, 2026) |
| Vertex AI Agent Builder | Agent Platform |
| Agentspace | Gemini Enterprise (app) |
| NotebookLM | Gemini Notebook |
| Analytics Hub | BigQuery sharing (renamed 2025) |
| Cloud Functions | Cloud Run functions (renamed 2024) |

- The service, pricing and behaviour did not change, only the label.
- **Looker Studio is the odd one out**: it went Data Studio → Looker Studio (2022) → Data Studio (2026). Still not the same product as **Looker**.
- Most of this file was written before the renames and uses the old names.

## Data Pipeline Stages: Ingest, Transform, Store

**Ingest**: data becomes a data source, available downstream.
- Data source = raw, unprocessed data; any system/app/platform that creates, stores, or shares data.
- Cloud Storage, data lake, holds various data source types.
- Pub/Sub, asynchronous messaging, delivers data from external systems.

**Transform**: adjust, modify, join, or customize a data source to match a downstream requirement.
- Three patterns: EL (extract, load), ELT (extract, load, transform), ETL (extract, transform, load).
- Each pattern gets its own section later in my notes.

**Store**: final stage, deposit data in its final form.
- Data sink = final stop; where processed/transformed data is stored for future use, analysis, decisions.
- BigQuery, serverless data warehouse.
- Bigtable, highly scalable NoSQL database.

## Data Formats: Structured vs Unstructured

**Unstructured data**: non-tabular form (documents, images, audio files).
- Usually suited for Cloud Storage.
- BigQuery can also store unstructured data, via object tables.

**Structured data**: stored in tables, rows, and columns.

## Google Cloud Storage & Database Products

**Cloud Storage**
- Accessed via HTTP requests, including ranged GETs for partial retrieval.
- Only key is the object name; object itself is unstructured bytes (plus metadata).
- Objects up to 5 TB each.
- Built for availability, durability, scalability, consistency.
- Ideal for static websites, images, videos, blobs, any unstructured data.
- 4 storage classes, differentiated by expected access frequency: Standard, Nearline, Coldline, Archive.

**Structured data services** (no one-size-fits-all, pick based on workload):
- **Cloud SQL**: managed relational database service.
- **AlloyDB**: fully managed, high-performance PostgreSQL service.
- **Spanner**: fully managed relational DB; strong consistency + horizontal scalability.
- **Firestore**: fast, serverless NoSQL document database; automatic scaling, high performance, easy app dev.
- **BigQuery**: fully managed, serverless enterprise data warehouse for analytics.
- **Bigtable**: high-performance NoSQL; fast key-value lookup, consistent sub-10ms latency.

## Data Lake vs Data Warehouse

**Data lake**: vast repository for raw, unprocessed data (unstructured, semi-structured, structured).
- Centralized storage, flexible use cases: data science, applications, business decisions.

**Data warehouse**: structured repository for pre-processed, aggregated data from multiple sources.
- For long-term business analysis: efficient querying and reporting.
- Often standalone, independent of other data storage systems.

**Comparison table:**

| | Data lake | Data warehouse |
|---|---|---|
| Data format | Native (unstructured, semi-structured, structured) | Schema (structured or semi-structured) |
| Data type | Raw | Pre-processed and aggregated from multiple sources |
| Purpose | Data science, applications, business decisions | Long-term business analysis |
| Dependencies | Tools/processes for data discovery, governance, security, metadata management | Standalone |
| Service | Cloud Storage | BigQuery |

## Metadata Management: Dataplex

> **Renamed: Dataplex is now Knowledge Catalog.** Newer Google material uses Knowledge Catalog for the same thing, and the Cloud SQL console shows "Knowledge Catalog integration". The same material also says **Managed Apache Spark** (= Dataproc / Serverless for Apache Spark) and **Agent Platform** (probably Vertex AI). Either name can show up, learn both.

**Dataplex**: comprehensive data management solution; centrally discover, manage, monitor, govern distributed data across an organization.
- Breaks down data silos, centralizes security and governance, while still enabling distributed ownership.
- Enables search/discovery of data based on business context.
- Built-in data intelligence, open-source tool support, partner ecosystem, helps trust data and speed up time to insights.
- Standardizes and unifies metadata, security policies, governance, classification, and data lifecycle management across distributed data.

**Common use case, group and share data by readiness, using Dataplex (3 zones):**

| Zone | Access | State | Storage |
|---|---|---|---|
| Landing zone | Data engineers | Ingested data, limited access | Bucket(s) |
| Raw zone | Data engineers + data scientists | Cleaned, immutable | Bucket + Dataset |
| Curated zone | All users | Processed, source of trust | Dataset(s) |

Example (retail): POS exports land raw in a Cloud Storage bucket (Landing, engineers only) → engineers clean/dedupe into a BigQuery dataset (Raw, engineers + data scientists can explore/feature-engineer) → aggregated into a final reporting table (Curated, source of trust for all users/dashboards). Equivalent to a BI extract → staging → presentation-layer pattern.

## Sharing Data Outside Your Organization

Sharing a BigQuery dataset (tables, views, ML models, routines) with an external org is harder than internal sharing. Naive approaches and their problems:
- **Export and copy** → data freshness (copy goes stale).
- **ETL process** into external org's BigQuery → pipeline management overhead.
- **Onboard external users in IAM** → complex permissions, no visibility into how they use the data.

**Underlying challenges this creates:**
- Data freshness
- Pipeline management
- No data usage visibility
- Complex permissions

## Sharing Datasets Using Analytics Hub

Cross-org data sharing considerations: security/permissions, destination options for data pipelines, data freshness/accuracy, usage monitoring.

**Analytics Hub**: built to meet these challenges.
- Create a rich data ecosystem by **publishing** and **subscribing** to analytics-ready datasets.
- Data shared **in place** (no copy/export), providers control and monitor how their data is used.
- Self-service access to trusted data assets, including data provided by Google.
- Enables **monetizing** data assets, Analytics Hub removes the need to build monetization infrastructure yourself.

## Hands-on: three ways to load data into BigQuery

NYC taxi 2018 trips. Three ingestion methods practiced.

**Step 1, create dataset:** project ⋮ → Create dataset → ID `nyctaxi`.

**Step 2, load local CSV (console):** dataset → Create Table → Source: Upload, format CSV → Table `2018trips` → Schema: **Auto detect** → Create. Result: 10,018 rows.

**Step 3, load from Cloud Storage (CLI):** Cloud Shell (terminal icon, top-right toolbar):
```
bq load --source_format=CSV --autodetect --noreplace \
  nyctaxi.2018trips \
  gs://<source-bucket>/nyc_tlc_yellow_trips_2018_subset_2.csv
```
- `--noreplace` = **append**. Append is also the default for `bq load`; `--replace` is what overwrites the table.
- `gs://` = Cloud Storage path; no local download needed.
- Result: 20,024 rows (10,018 + 10,006). Same dataset split into 2 files purely to practise 2 methods.

**Step 4, CTAS (DDL):**
```sql
CREATE TABLE nyctaxi.january_trips AS
SELECT * FROM nyctaxi.2018trips
WHERE EXTRACT(MONTH FROM pickup_datetime) = 1;
```

### SQL learnings from this exercise

**Schema Auto detect**: BigQuery infers column names/types from header + sample rows. Every table needs a schema; the toggle is auto vs manual, not on/off. Use auto for quick loads/exploration; use manual in production where a wrong guess matters (e.g. `"01"` typed as INTEGER loses the leading zero) and where you want bad data caught on load.

**fare_amount vs total_amount**: `fare_amount` = metered fare only; `total_amount` = full bill (fare + tip + tolls + taxes).

**Window functions + QUALIFY**: `MAX(x) OVER()` computes an aggregate across all rows but *keeps every row*, stamping the result onto each one (unlike GROUP BY, which collapses). `QUALIFY` filters on window-function results, which `WHERE` cannot do.
```sql
SELECT fare_amount, pickup_datetime
FROM nyctaxi.2018trips
QUALIFY EXTRACT(YEAR FROM pickup_datetime)
      = MAX(EXTRACT(YEAR FROM pickup_datetime)) OVER()
ORDER BY fare_amount DESC
LIMIT 5
```

**SQL logical execution order** (explains why QUALIFY exists, WHERE runs at step 2, window functions aren't computed until step 5):
```
FROM → WHERE → GROUP BY → HAVING → window functions → QUALIFY → SELECT → ORDER BY → LIMIT
```

**LIMIT behavior:**
- `LIMIT 5` alone → "any 5 rows", no ordering guarantee, engine genuinely stops early. Tables have no inherent order; this is for peeking at data, not answering questions.
- `ORDER BY x DESC LIMIT 5` → full scan still required (the max could be the last row read), but the *sort* is cheap: engine keeps a running top-5 in memory (top-K), never sorts the whole set.

**DDL vs DML vs Query**: all SQL, different jobs:
- DDL (definition): `CREATE TABLE`, `ALTER`, `DROP`: structure.
- DML (manipulation): `INSERT`, `UPDATE`, `DELETE`: data.
- Query: `SELECT`: reads.

**CTAS (`CREATE TABLE ... AS SELECT`)**: the SELECT defines the new table's schema *and* contents. It is a **frozen snapshot**; it does not stay in sync with the source. This is the "T" in ELT.

**Table vs View:**

| | CREATE TABLE AS | CREATE VIEW |
|---|---|---|
| Stored | Actual copied data | Just the SQL text |
| Storage cost | Yes | None |
| Freshness | Frozen snapshot | Always live (re-runs each query) |
| Query speed | Fast (pre-computed) | Slower (recomputes) |

Power BI analogy: table ≈ Import mode, view ≈ DirectQuery.

**Dataset naming is convention only**: `landing.x`, `raw.y`, `curated.z` are just `dataset.table`. SQL attaches no meaning to the names; the zone semantics live in team discipline (and in Dataplex, which can enforce real governance on them).

### Writing scalable SQL in BigQuery

BigQuery bills by **bytes scanned per column touched**. At 100M+ rows:
1. **Avoid `SELECT *`**: columnar storage means unused columns are free if you don't select them. Biggest cost lever.
2. **Avoid double scans**: a scalar subquery that recomputes an aggregate scans the table again; `QUALIFY` + window function does it in one pass.
3. **Don't wrap a partition column in a function**: `EXTRACT(YEAR FROM ts) = 2018` can defeat **partition pruning**. Use a range instead: `ts >= '2018-01-01' AND ts < '2019-01-01'`.
4. **Partition by date, cluster by frequently-filtered columns** for tables queried repeatedly.

**Aggregate once, read many times**: the core pattern for dashboards at scale. Don't let 50 analysts each scan a petabyte; run one scheduled job that writes a small pre-aggregated table, and point dashboards at that. At scale, process **incrementally** (yesterday's partition only), never re-aggregate all history nightly.

Raw → curated is typically a chain of scheduled CTAS statements (orchestrated by Cloud Composer/Airflow), which is what a "data pipeline" concretely *is*.

## BigQuery Deep Dive

- Fully managed, serverless enterprise data warehouse for analytics.
- Built-in features: machine learning, geospatial analysis, business intelligence.
- Scans terabytes in seconds, petabytes in minutes.
- Great for OLAP (online analytical processing), big data exploration/processing.
- Well-suited for reporting with BI tools.

**Access options:**
1. Google Cloud console SQL editor.
2. `bq` command-line tool (part of Google Cloud SDK).
3. REST API, supports 7 programming languages.

**Organization:**
- Tables organized into **datasets**, scoped to a Google Cloud project.
- Reference construct: `project.dataset.table`.
- Access control via IAM, at dataset/table/view/column level.
- Need at least read permission on a table/view to query it.

## Data Replication and Migration

Topics covered here:
1. Baseline Google Cloud data replication and migration architecture.
2. `gcloud` command-line tool, options and use cases.
3. Storage Transfer Service, functionality and use cases.
4. Transfer Appliance, functionality and use cases.
5. Datastream, features and deployment.


### 1. Replication and Migration Architecture

The replicate/migrate stage is the front door to **ingest**: bringing data from external or internal systems into Google Cloud for further refinement, then transform, then store.

**Sources**: on-premises or multi-cloud: file systems, object stores, HDFS, relational databases.
**Destinations for file/object transfer**: Cloud Storage or BigQuery.

**Two distinct problem families:**

| | Tools | Lands in |
|---|---|---|
| Move files/objects | `gcloud storage`, Storage Transfer Service, Transfer Appliance, Datastream | Cloud Storage, BigQuery |
| Migrate a database workload | **Database Migration Service** (Oracle, MySQL, PostgreSQL, SQL Server); **Dataflow** templates for NoSQL / non-relational / complex migrations | Cloud SQL, AlloyDB, BigQuery |

**Three timing modes:**

| Mode | Meaning |
|---|---|
| One-off transfer | Move it once |
| Scheduled replication | Re-run on a cadence |
| Change data capture (CDC) | Stream only changes, continuously, tail the DB transaction log (Datastream) |

CDC note: instead of re-querying the source table nightly, read the database's own write-ahead log and forward each INSERT/UPDATE/DELETE as it happens. Near-real-time, near-zero load on the source, which is why a DBA will allow it when they'd refuse a nightly full extract.

**Bandwidth math, the decision rule.** Ease of migration depends on data size *and* network bandwidth:
- 1 TB over **100 Gbps** ≈ **2 minutes**
- 1 TB over **100 Mbps** ≈ **30 hours**

Same data, 1000x difference. Therefore:
- **Smaller datasets** → `gcloud storage` (manual, scriptable) or Storage Transfer Service (managed, scheduled, retries).
- **Larger datasets** → **Transfer Appliance**, a faster *offline* transfer: Google ships physical hardware, you fill it and ship it back. 500 TB on a 100 Mbps link would take ~20 months online; the appliance makes it a week of shipping.

### 2. `gcloud storage` command

CLI for transferring **small to medium** datasets to Cloud Storage. Ad-hoc, run by a person.

- Sources: on-prem file systems, object stores, HDFS.
- `cp` is the workhorse verb (same as Unix `cp`, one end is a bucket):

```bash
gcloud storage cp myfile.csv gs://my-bucket/
gcloud storage cp -r ./exports gs://my-bucket/exports/     # whole folder
```

- If it must run unattended on a schedule, that's Storage Transfer Service, not this.
- **Naming:** older docs say `gsutil cp`: legacy tool, same job. `gcloud storage` is the modern, faster replacement.

### 3. Storage Transfer Service and Transfer Appliance

**Storage Transfer Service**: moves **large** datasets into Cloud Storage, managed and scheduled.
- Sources: on-premises and **multicloud** file systems, object stores, **Amazon S3** and **Azure Blob Storage** named explicitly, and HDFS.
- High transfer speeds, **up to tens of gigabits per second**.
- Supports **scheduled** transfers; managed retries/resume.
- Signal worth knowing: "data in S3/Azure, want it in Cloud Storage" → STS.

**Transfer Appliance**: Google's **offline** solution for massive datasets.
- Google provides the hardware, you copy data onto it, you ship it back.
- Comes in **multiple sizes**.
- Two triggers, either/or: **limited bandwidth**, or a **very large transfer** (months over the wire).

**The three file-transfer tools, settled:**

| Tool | Trigger |
|---|---|
| `gcloud storage` | You, once, by hand |
| Storage Transfer Service | Unattended, scheduled, or source is S3/Azure |
| Transfer Appliance | The network can't do it in reasonable time |

## Choosing a Structured Data Storage Option (decision tree)

**Structured data → workload type → SQL/NoSQL → product**

- **Transactional workload**
  - SQL
    - Local/regional scalability → **Cloud SQL**
    - High PostgreSQL scalability → **AlloyDB**
    - Global scalability → **Spanner**
  - NoSQL → **Firestore**
- **Analytical workload**
  - SQL → **BigQuery**
  - NoSQL → **Bigtable**

### 4. Datastream

Continuous replication of **relational** databases (on-prem or multi-cloud) into Google Cloud.
Sources: **Oracle, MySQL, PostgreSQL, SQL Server**. Destinations: **Cloud Storage or BigQuery**.

**Two CDC modes, your choice:**
- **Historical backfill**: read the entire existing table once, then follow changes. The initial load is the expensive part; the ongoing stream is a trickle.
- **Changes only**: propagate new changes from now on. Use when history already arrived another way (appliance seed, existing export).

**Selective replication, schema, table, or column level.** You are not forced to mirror the whole database. Pick `sales.orders` and `sales.order_lines`; inside those, exclude columns. Real use: replicate `employee_id`, `department`, `hire_date` and leave SIN / home address / salary on-prem. Sensitive columns never enter Google Cloud, so they can't leak from it, the compliance conversation ends there.

**The mechanism, tap the write-ahead log (WAL).** Same idea, four different engine mechanisms (memorize):

| Source | Log mechanism |
|---|---|
| Oracle | **LogMiner** |
| MySQL | **binary log** (binlog) |
| PostgreSQL | **logical decoding** |
| SQL Server | **transaction logs** |

INSERT / UPDATE / DELETE events are captured, processed, and transformed into **Avro or JSON**, ready for storage, typically BigQuery tables. Near-real-time, near-zero load on the source.

**Event message anatomy, three parts:**
1. **Generic metadata**: source table, timestamps, related context.
2. **Source-specific metadata**: database name, schema, table, **change type (INSERT/UPDATE/DELETE)**, engine-specific identifiers. Supports data lineage tracking.
3. **Payload**: the actual changed data, key-value by column name.

Why the change type matters: a nightly extract gives you *current state*; a change stream gives you *every state the row passed through*. That is how you build a real slowly-changing dimension, or answer "how long did this ticket sit in Pending?" when the source never stored that.

**Unified data types, the headline feature.** Four vendors spell one number four ways. Datastream normalizes at the wire:

```
Oracle NUMBER      ┐
MySQL DECIMAL      ├─→ Datastream: decimal ─→ Avro decimal
PostgreSQL NUMERIC ┤                          JSON number
SQL Server DECIMAL ┘                          BigQuery NUMERIC (native)
```

Why it matters: two source engines (say an acquisition on Oracle, you on SQL Server) feeding one BigQuery dataset. Without unified typing you write per-source `CAST` logic in every query and find rounding differences in a revenue report three months later. Same class of problem as merging an ODBC and a REST source in Power Query and having one side arrive silently as text.

**Four deployment shapes:**
1. **Direct → BigQuery**: the default, nothing in between.
2. **→ Dataflow → BigQuery**: custom processing: reshape, enrich, or mask in transit.
3. **Event-driven architecture**: the change stream *is* the trigger; something reacts within seconds.
4. **Dataflow templates**: pre-built pipelines for replication and migration tasks.

### Summary, the four migration/replication options

| Tool | Fits |
|---|---|
| `gcloud storage` | Smaller **online** transfers |
| Storage Transfer Service | Larger **online** transfers |
| Transfer Appliance | Massive **offline** migrations |
| Datastream | Continuous **online replication of structured data**: batch *and* streaming velocities |

Choose on: **data size, transfer type, data availability requirements.**

## Reading and Writing SQL

### How to read a query, bottom first, then FROM before SELECT

SQL is *written* SELECT-first but must be *read* FROM-first. That mismatch is why `o.amt_eur` is confusing on first pass: you hit a nickname before the line that defines it. (Same reason `QUALIFY` exists, see the logical execution order note above.)

Order to read any query:
1. **Line 1**: what am I building? (`CREATE TABLE x AS`)
2. **The bottom `SELECT`**: how many ingredients are there? A 300-line query is usually 6 small ones stacked; the bottom tells you which `WITH` blocks are actually used.
3. **Per block: the `FROM` / `JOIN` lines first**: this is where table nicknames are defined, and where you learn how many tables are involved.
4. **Then that block's `SELECT`**: now the `o.x * r.y` expressions read plainly.

**Aliases**: `FROM raw.orders_a o` means "call this `o` for the rest of the query". Pure typing shortcut. Always defined in `FROM`/`JOIN`, never in `SELECT`, so a confusing `x.something` always has its definition further down.

### JOIN vs UNION, the distinction everyone trips on

- **JOIN**: adds **columns** sideways. Orders + rates → orders with a rate attached.
- **UNION**: stacks **rows** downward. Company A's rows + Company B's rows → all rows.

A JOIN on a lookup table is a **VLOOKUP**: for each row on the left, go find the matching row on the right and bring a column back.

Both sides of a UNION must already be identical in shape (same columns, same order, same types). **Making them identical is the entire job of the staging layer.**

### UNION vs UNION ALL

`UNION ALL` keeps every row. `UNION DISTINCT` (plain `UNION` in most other databases; BigQuery makes you write `ALL` or `DISTINCT`) silently drops exact duplicates, rows identical across *every* column.

Failure case: two different customers each order $300 on the same day, and the SELECT has no order_id. The rows are indistinguishable, `UNION DISTINCT` deletes one, revenue is $300 short, nothing warns you. **Use `UNION ALL` on anything financial** unless you specifically want dedupe.

### CAST, changes the data type, not the display

`CAST(88 AS STRING)` → `"88"`. Looks the same, behaves differently: `88 + 1 = 89`, but `"88" + 1` errors. A UNION requires matching types in each position, so mismatched sources need one side cast.

### Date joins and the silent-zero-rows trap

If both columns are real `DATE` types, display format (`yyyy-mm-dd` vs `mm/dd/yyyy`) is **irrelevant**: that's presentation, not storage. SQL compares actual dates.

If either is stored as `STRING`, they must match character for character. `"2026-03-01"` vs `"03/01/2026"` → the join matches nothing, returns **zero rows, no error**. Silent and expensive.

Fix once, in staging: `PARSE_DATE('%m/%d/%Y', order_dt_text) AS order_date`.

**How to check a column's type** (looking at the values tells you nothing):

| Where | How |
|---|---|
| BigQuery console | Click the table → **Schema** tab |
| BigQuery SQL | `SELECT column_name, data_type FROM raw.INFORMATION_SCHEMA.COLUMNS WHERE table_name = 'orders_a'` |
| Python / pandas | `df.dtypes`: **`object` means text**; a real date reads `datetime64[ns]` |

Quick visual tell: in BigQuery's grid (and Excel), real numbers and dates are **right**-aligned, text is **left**-aligned. A date hugging the left edge is text pretending to be a date. Good for a glance, not proof, check the schema before trusting a join.

Related trap already noted above: schema auto-detect guessing `"01"` as INTEGER.

### Also: dataset prefixes are just names

`stg.orders_a`, `raw.orders_b`, `ref.fx_rates` are `dataset.object`. SQL attaches **no** meaning to `stg` / `raw` / `ref`: pure convention. `banana.orders_a` works identically.

## Hands-on: copying a PostgreSQL table into BigQuery with Datastream

Replicated a Cloud SQL Postgres table into BigQuery and watched INSERT / UPDATE / DELETE flow through. **You only ever wrote to Postgres. BigQuery filled itself in, both times.** That is the whole point of the exercise.

### What actually got built

**2 connection profiles + 1 stream.** Not two streams, the profiles are saved logins that do nothing on their own; the stream is the one job that uses both.

| | postgres-cp | bigquery-cp |
|---|---|---|
| Role | Source | Destination |
| Needs IP, port, user, password | Yes | **No** |
| Why | Separate database with its own login | Same project, permission comes from IAM |

### Step 1, make a Postgres that is willing to be tailed

```bash
gcloud services enable sqladmin.googleapis.com

POSTGRES_INSTANCE=postgres-db
DATASTREAM_IPS=<allowlisted Datastream IPs for your region, comma-separated>
gcloud sql instances create ${POSTGRES_INSTANCE} \
    --database-version=POSTGRES_14 --cpu=2 --memory=10GB \
    --authorized-networks=${DATASTREAM_IPS} \
    --region=us-east4 --root-password <your password> \
    --database-flags=cloudsql.logical_decoding=on
```

Everything in that command is Google's order form except **two lines**:

- **`--authorized-networks`**: a new Cloud SQL instance accepts connections from **nobody**. Those 5 IPs are **Datastream's own machines** in `us-east4` (published by Google, fixed per region). Datastream runs as a fleet; you can't predict which machine handles your stream at a given moment, so all 5 must be allowed. If only one were listed, the stream would break the first time Google switched machines, silently, at 2am. *They are not 5 databases and not 5 destinations. Data goes to exactly one place: BigQuery.*
- **`cloudsql.logical_decoding=on`**: the CDC switch. **Off by default everywhere.** Postgres normally keeps its change log private; this writes it in a form an outsider can read. No Datastream without it.

Note `PRIMARY_ADDRESS` from the output, needed for the connection profile.

**Then, inside psql** (`gcloud sql connect postgres-db --user=postgres`, password `<your password>`): created schema `test`, table `example_table`, 4 rows. Plus `ALTER TABLE ... REPLICA IDENTITY DEFAULT`: tells Postgres to identify rows by primary key in the change log, without which UPDATEs and DELETEs don't replicate properly.

**Then the three replication permissions:**

```sql
CREATE PUBLICATION test_publication FOR ALL TABLES;
ALTER USER POSTGRES WITH REPLICATION;
SELECT PG_CREATE_LOGICAL_REPLICATION_SLOT('test_replication', 'pgoutput');
```

| Line | Plain meaning |
|---|---|
| Publication | **What** may be read, the list of tables allowed out |
| `WITH REPLICATION` | **Who** may read the change log |
| Replication slot | **How far** the reader got, a bookmark |

**Publication vs slot** (confusingly similar names, both typed into the UI later):
Postgres writes changes as a numbered list, `#1 insert id=1`, `#2 insert id=2`, `#5 update id=2`, `#6 delete id=4`. If Datastream has read up to #4, the slot stores **4**. Postgres wants to delete old entries to save space; the slot says *don't bin #5 onward, nobody has read them*. Offline for an hour → changes pile up → Datastream resumes at #5, nothing lost. Without the slot they'd be gone.

**Production trap (the flip side):** a stream that's stopped and never deleted leaves the slot demanding retention forever. Postgres hoards change records waiting for a reader that never comes, until the disk fills and the database goes down.

### Step 2, the UI

`postgres-cp` (IP, port 5432, `postgres`/`<your password>`, database `postgres`, encryption None, IP allowlisting) → **Run Test** proves the IPs and password work. Then `bigquery-cp` (name + region only).

Stream `test-stream`: source `postgres-cp` → **slot `test_replication`, publication `test_publication`, schema `test`** → destination `bigquery-cp` → **staleness limit 0 seconds** (apply changes immediately, don't batch, the cost-vs-freshness knob) → Run Validation → Create & Start.

### Steps 3 and 4, the proof

Step 3: the 4 pre-existing rows appeared in BigQuery on their own, **that is backfill**, nobody asked for a copy. Step 4: back in **Postgres**, `INSERT 0 3`, `UPDATE 7` (`int_col*2`), `DELETE 1`. BigQuery then showed 6 rows, `-987 → -1974`, `2786 → 5572`, `abc` gone, **that is CDC**.

The two steps are deliberately the two modes: backfill, then changes.

**Where the event structure showed up concretely**: the BigQuery table had the 4 source columns **plus** a `datastream_metadata` RECORD column. Payload and metadata, side by side, exactly as described in my Datastream notes above.

### Gotchas hit

- **`Password:` fails instantly without letting you type**: leftover newlines from an earlier multi-line paste sit in the input buffer and get eaten by the prompt as an empty password. Press Enter a few times to flush, then retry. (`gcloud sql users set-password postgres --instance=postgres-db --password=<your password>` resets it if needed.)
- **`Not found: Dataset ...:test was not found in location US`**: the query ran in `US` while the data lives in `us-east4`. Once the dataset exists, opening it from the explorer and clicking **Query** targets the right location automatically.
- **Preview tab shows "There is no data to display"** even when rows exist, Preview can't see rows still streaming in. Run a `SELECT` instead. This is expected.

### Where these fields come from in a real job

| Field | Who gives it to you |
|---|---|
| Hostname / IP | DBA, or read it off the Cloud SQL page |
| Port | Standard per engine, Postgres 5432, MySQL 3306, SQL Server 1433 |
| Username / password | **Request a dedicated service account.** Never reuse a person's login |
| Encryption | Security team. `None` is a practice shortcut; real work uses SSL |
| IP allowlisting | Send the network/DBA team Google's published IP list |

The realistic version of Step 1 is an email: *"I need a read-only Postgres account for a replication job, logical decoding enabled, and these 5 IPs allowlisted."* The form is the easy part; getting those approved is the job.

**Most common real failure:** the DBA provisions the account without the `REPLICATION` permission. Run Test passes, the stream fails later with a permissions error.

## Extract and Load

The first of the three transform patterns introduced at the start of these notes (EL / ELT / ETL).

Topics covered here:
1. Baseline extract and load architecture diagram.
2. `bq` command-line tool, options.
3. **BigQuery Data Transfer Service**: functionality and use cases.
4. **BigLake**: functionality and use cases, as a **non** extract-load pattern.

⚠️ **Name trap:** *BigQuery Data Transfer Service* is not *Storage Transfer Service*. STS moves files into Cloud Storage; BQ DTS loads into BigQuery.

BigLake is flagged up front as the "don't load at all" option, query files where they sit. Related to the object tables note above.

### 1. Extract and Load Architecture

**The pattern:** bring data into BigQuery with **no upfront transformation**. Transform later, in SQL, once it's landed. Greatly simplifies ingestion.

**Two families of tools, genuinely different mechanics:**

| | Copies the data in | Leaves it where it is |
|---|---|---|
| Tools | `bq load`, **BigQuery Data Transfer Service** | **External tables**, **BigLake tables** |
| Result | A real BigQuery table | A pointer to files in a bucket |

The second column is the "eliminates the need for data copying" claim. Both families support **scheduling**.

**Formats, the lists are not symmetric:**

| Direction | Formats |
|---|---|
| **Load into** BigQuery | Avro, Parquet, ORC, CSV, JSON, **Firestore exports** |
| **Export from** BigQuery | CSV, JSON, Avro, Parquet |

⚠️ **ORC loads in but does not export out.** Firestore exports likewise are import-only. This asymmetry is a common trap.

Exportable artifacts include **query results** as well as table data.

Practical note: **Parquet** is columnar and carries its own types, so a date stays a date. CSV is plain text, every column is a guess on arrival, which is where schema auto-detect problems come from (see the `"01"` note above).

**Two ways to load:**

1. **The UI**: select files, specify format, auto-detect schema. Simple, but needs a person clicking. (Used in the NYC taxi exercise.)
2. **`LOAD DATA` SQL statement**: more control, built for automation and for appending or overwriting existing tables.

```sql
LOAD DATA INTO nyctaxi.trips
FROM FILES (format='PARQUET', uris=['gs://my-bucket/trips/*.parquet']);
```

`LOAD DATA OVERWRITE` replaces the table instead of appending.

**Why `LOAD DATA` matters:** it's SQL, so it can be scheduled, version-controlled, and run by a pipeline. Same manual-vs-unattended split as `gcloud storage` vs Storage Transfer Service.

### 2. The `bq` command-line tool

Part of the Google Cloud SDK. A programmatic way to interact with BigQuery, Linux-shaped verbs.

**`bq mk`**: make BigQuery objects (datasets, tables). The command form of "Create dataset" in the UI:

```bash
bq mk nyctaxi
```

**`bq load`**: load data into a table (used in the NYC taxi exercise). Key parameters:

```bash
bq load \
  --source_format=CSV \
  --skip_leading_rows=1 \
  --schema=./trips_schema.json \
  nyctaxi.2018trips \
  gs://my-bucket/trips/*.csv
```

| Parameter | What it does |
|---|---|
| `--source_format=CSV` | Which format the source is. Defaults to CSV, so often omitted |
| `--skip_leading_rows=1` | **Skip the header row**: without it the column names load as a data row |
| `--schema=file.json` | Define the table structure yourself, instead of `--autodetect` |
| `gs://.../*.csv` | **Wildcard**: load multiple files from Cloud Storage in one command |
| `--noreplace` | Append (also the default); `--replace` overwrites (from the taxi exercise) |

**The wildcard is the practical one.** The taxi exercise ran the command twice for two files; `*.csv` does both, or 900 daily files, in one. That's the shape of a real daily load: files land in a bucket, one command sweeps them all.

**Schema file vs `--autodetect`**: same trade-off as the taxi exercise, now as a flag. A `.json` schema file can be version-controlled, reviewed, and fails loudly when the source changes shape. Auto-detect guesses silently. Autodetect for exploring, schema file for production.

### 3. BigQuery Data Transfer Service

Loads **structured** data into BigQuery from:
- **SaaS applications**
- **Object stores**
- **Other data warehouses**

**Characteristics:**
- **Scheduling**: recurring *or* on-demand transfers.
- Configuration options for data source details and destination settings.
- **Managed and serverless**: no infrastructure to manage.
- **No code**: setup and management through configuration, not scripts.

⚠️ **Name trap, restated:** *BigQuery Data Transfer Service* ≠ *Storage Transfer Service*.

| | Storage Transfer Service | BigQuery Data Transfer Service |
|---|---|---|
| Moves | Files/objects | Structured data |
| Into | Cloud Storage | BigQuery |
| Typical source | S3, Azure Blob, HDFS, on-prem file systems | SaaS apps, object stores, other warehouses |

Position in the EL pattern: the **scheduled, no-code** member of the "copies the data in" family, as opposed to `bq load` (manual/scripted) and external/BigLake tables (no copy at all).

### 4. External Tables and BigLake, querying without loading

BigQuery's data access extends beyond its own storage.

**Three ways to analyze structured data, the core decision:**

| | Load into BigQuery | External table | BigLake table |
|---|---|---|---|
| Data movement | **Yes**: a copy | **No** | **No** |
| Performance | High | Slower | **High** |
| Security | Full | Basic | **Column-level and row-level** |
| Suited for | Everyday high-performance analytics | **Less frequent access** | Enterprise data lake |

BigLake is pitched as "best of both worlds": high-performance analytics on data in Cloud Storage, without loading it.

**What each can read:**

| | Sources |
|---|---|
| **External tables** | Cloud Storage, **Google Sheets**, Bigtable |
| **BigLake tables** | Cloud Storage, **and other cloud providers' object stores** (multi-cloud) |

**Google Sheets external tables**: give BigQuery the Sheets URL and format, and the sheet behaves as a table. Bridges the gap between a hand-maintained spreadsheet and the warehouse.
*Real use:* finance maintains a cost-centre → department mapping sheet by hand; join it in SQL to a 500M-row fact table. They keep editing the sheet, the query picks up changes with no reload.

**Limitations (worth knowing):**

| | Slower performance | No cost estimation | No table preview | No query caching |
|---|---|---|---|---|
| External table | ✔ | ✔ | ✔ | ✔ |
| BigLake table | - | ✔ | ✔ | - (has metadata caching) |

"No cost estimation" = BigQuery can't tell you what a query will scan before running it, because it doesn't know what's in the bucket until it looks.

**BigLake underlying tech:** unified interface to query the data lake and other sources without moving or copying. Uses **Apache Arrow** for efficient data handling, plus fine-grained security and metadata caching. Standard SQL, `SELECT`, joins, exactly as with native tables.

#### The metadata cache, why BigLake is fast

Stores details **about** the external data, e.g. for Parquet files in Cloud Storage: **file size, row count, column statistics (min/max)**.

What that buys:
- Skip listing all objects in the bucket
- **Prune files and partitions** faster
- Enable **dynamic predicate pushdown**
- Spark can read the same statistics via the Spark-BigQuery connector

Same instinct as partition pruning (see the scalable-SQL notes above), don't read what can't match. A query for `WHERE date = '2026-09-01'` skips any file whose max date is in August, without opening it.

**Staleness is configurable: 30 minutes to 7 days.** Refresh automatically or manually.

#### Security, the bigger differentiator

| | External table | BigLake table |
|---|---|---|
| Permissions needed | On the **table** *and* on the **underlying data source** | On the **table** only |
| How | User is granted both, separately | Access **delegated through a service account** |
| Result | Complex access management | Table access **decoupled** from data source |

The delegation is what makes column-level and row-level security possible: BigQuery becomes the only door to the data, so it can hide a column or filter rows. With a plain external table the user has bucket access anyway, so the bucket is a way around any table-level rule.

**Summary:** both query data outside BigQuery, but BigLake supports more formats and storage locations (**including multi-cloud object stores**) and adds fine-grained security. External tables are **simpler to set up** but lack those controls. BigLake = performance + security + flexibility, aimed at enterprise data lake use cases.

## Hands-on: hiding columns of a bucket CSV with a Lakehouse (BigLake) table

⚠️ **Naming:** the newer name is **"Lakehouse"**, my notes above say **"BigLake"**. Same product, Google renamed it. Both names appear in the console (the table gets a `Lakehouse` label).

**What the exercise proves in one line:** you can hide specific *columns* of a plain CSV file sitting in a bucket, and BigQuery will enforce it, even against a project owner.

### The six steps and what each is really for

| Step | What it does | Concept |
|---|---|---|
| 1 | Create a **connection resource** | Creates a robot (service account) |
| 2 | Grant **that robot** `Storage Object Viewer` | The robot can read the bucket. **You can't** |
| 3 | Create the Lakehouse table on `customer.csv` | Table points at the file *and* names the connection |
| 4 | Query it, all 13 columns | Delegation working: you read a file you have no access to |
| 5 | Policy-tag `address`, `postal_code`, `phone` | **The actual lesson** |
| 6 | Upgrade a plain external table | Attaching a connection is the only difference |

The single checkbox **"Create a Lakehouse table using a Cloud Resource connection"** is the entire difference between an external table and a Lakehouse table.

### Step 5, the payoff

After tagging three columns, the same query that worked five minutes earlier fails:

```
Access Denied: BigQuery: User has neither fine-grained reader nor masked get
permission to get data protected by policy tag "biglake-taxonomy-... : biglake-policy"
on columns ...biglake_table.address, ...biglake_table.phone, ...biglake_table.postal_code
```

**Denied to a project owner.** The policy tag overrides project-level roles. Two roles could unlock it: **Fine-Grained Reader** (see real values) or **Masked Reader** (see them masked).

Then this works, returning 10 columns:

```sql
SELECT * EXCEPT(address, phone, postal_code)
FROM `PROJECT.demo_dataset.biglake_table`
```

**Why the whole setup was necessary:** you personally have no bucket access, only the service account does. So BigQuery is the *only* door to that data, which is what lets it decide what you see. **Important warning:** after migrating users to Lakehouse tables, *remove their direct Cloud Storage permissions*. Otherwise they open the CSV in the bucket and every policy is theatre.

### Step 6, the upgrade, and what the commands do

```bash
export PROJECT_ID=$(gcloud config get-value project)

bq mkdef --autodetect --connection_id=$PROJECT_ID.US.my-connection \
  --source_format=CSV "gs://$PROJECT_ID/invoice.csv" > /tmp/tabledef.json

bq show --schema --format=prettyjson demo_dataset.external_table > /tmp/schema

bq update --external_table_definition=/tmp/tabledef.json --schema=/tmp/schema \
  demo_dataset.external_table
```

| Command | What it does |
|---|---|
| `bq mkdef` | Writes a config describing "these files, read via *this connection*" |
| `bq show --schema` | Saves the table's existing column list |
| `bq update` | Applies both to the existing table |

**Why the middle command exists:** `bq update` replaces the *whole* definition. Without handing the schema back, `--autodetect` would guess and you'd lose the typed columns.

**No rebuild, no reload.** Attaching a connection is all that separates the two table types.

**Verify:** table Details → External Data Configuration → a `Connection ID` row appears, and the table gains the **Lakehouse** label. If `Last modified` still equals `Created`, the update didn't run.

### Console gotchas hit

- **"+ Add data" wasn't where the instructions expect** (Steps 1 to 2). Did both in Cloud Shell instead, same API, same result:
  ```bash
  bq mk --connection --location=US --project_id=$PROJECT_ID \
    --connection_type=CLOUD_RESOURCE my-connection
  bq show --format=prettyjson --connection $PROJECT_ID.US.my-connection   # read serviceAccountId
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:<serviceAccountId from the previous command> \
    --role=roles/storage.objectViewer
  ```
- **Create table form defaults to "Native table"**: must be switched to **External Table** manually.
- **Live sighting of the metadata cache:** after a successful query, BigQuery shows a banner *"Metadata caching is disabled. You can accelerate queries over external tables by enabling metadata caching."*, the BigLake feature from my notes above, off by default.

## Extract, Load and Transform (ELT)

The second of the three transform patterns. Load first, transform **inside** BigQuery afterwards.

Topics covered here:
1. Baseline extract, load and transform architecture diagram.
2. A common **ELT pipeline** on Google Cloud.
3. **BigQuery SQL scripting and scheduling** capabilities.
4. **Dataform**: functionality and use cases.

Connects directly to the earlier note: *"raw → curated is typically a chain of scheduled CTAS statements."* This section is that pattern, formalized. Dataform is the tool for managing those chains as a real project (dependencies, version control, testing) instead of scheduling 40 CTAS statements by hand.

### 1. ELT Architecture

**The pattern: data is loaded into BigQuery FIRST.** Transformations happen afterwards, inside BigQuery.

```
structured data → BigQuery STAGING tables → transform in BigQuery → PRODUCTION tables
```

Already built by hand in the multi-source union example above: `raw.orders_a` → `stg.orders_a` → `curated.orders_all`. Same shape, Google's names for the layers (staging → production).

**Four ways to do the transform step:**

| Option | What it is |
|---|---|
| **Procedural languages / SQL** | Write the transform as SQL |
| **Scheduled queries** | The same SQL, run on a regular basis |
| **Scripting / programming languages (Python)** | For logic SQL cannot express |
| **Dataform** | Simplifies transformation beyond basic programming options, SQL workflows |

**Why load first, then transform**: the whole argument for ELT: BigQuery is a serverless engine that scans terabytes in seconds. Transforming *before* loading means doing that work on some smaller machine you have to own and size. Dump the raw data in and let BigQuery's processing power do it.

Difference from ETL: same three letters, the **T** just moves.

### 2. SQL Scripting and Scheduling with BigQuery

**Procedural language support**: execute multiple SQL statements in sequence **with shared state**. Enables:
- Automating tasks like table creation
- Complex logic with `IF` and `WHILE`
- **Transactions** for data integrity
- **Declare variables**, and reference **system variables**

Plain SQL is one question, one answer. Procedural SQL is a *script*, do this, remember the result, then do that with it. What makes it procedural:

| Keyword | What it adds |
|---|---|
| `DECLARE id STRING` | A **variable**: plain SQL has none |
| `SET id = GENERATE_UUID()` | Assign to it |
| `BEGIN ... END` | A **block** that runs in order |
| `CALL create_customer()` | Run a saved block by name |

#### UDFs, user-defined functions

Your own function, callable in any query.

- Written in **SQL or JavaScript**
- **Persistent** (saved in a dataset, reusable) or **temporary** (one query only)
- ⚠️ **Use SQL UDFs when possible**: the recommendation, and one worth remembering
- **JavaScript UDFs** exist for the flexibility of **external libraries**
- **Community-contributed UDFs** are available for reuse

#### Stored procedures

**Pre-compiled collections of SQL statements** that encapsulate complex logic.

- Benefits: **reusability**, **parameterization** (input values), **transaction handling**, maintainability
- Called from applications *or* from within SQL scripts, promotes modular design
- Can accept input values and return values as output
- Can access or modify data across **multiple datasets**

```sql
CREATE OR REPLACE PROCEDURE dataset_name.create_customer()
BEGIN
  DECLARE id STRING;
  SET id = GENERATE_UUID();
  INSERT INTO dataset_name.customers (customer_id) VALUES(id);
  SELECT FORMAT("Created customer %s", id);
END;

CALL dataset_name.create_customer();
```

**Stored procedures for Apache Spark**: BigQuery can run PySpark, Java or Scala inside a stored procedure. Defined in the **BigQuery PySpark editor**, or via `CREATE PROCEDURE`. The code sits **inline in the SQL editor** or in a **Cloud Storage file**. So a Spark job becomes callable from SQL.

#### Remote functions

Integrate with **Cloud Run functions** for transformations needing real programming logic (Python).

**The Python lives in Cloud Run, not BigQuery.** BigQuery only stores a pointer:

```sql
CREATE FUNCTION dataset_name.object_length(signed_url STRING) RETURNS INT64
REMOTE WITH CONNECTION `<location>.<connection>`
OPTIONS(
  endpoint = "https://[...].cloudfunctions.net/object_length",
  max_batching_rows = 1
);
```

Then `SELECT object_length(uri) FROM ...` feels like any SQL function, but every call goes out over HTTP.

Note `REMOTE WITH CONNECTION`: **the same kind of connection resource created in the Lakehouse exercise**. Same mechanism: a service account BigQuery uses to reach something outside itself. There a bucket, here a function. `RETURNS INT64` is required because BigQuery can't see inside your Python.

**The three tiers, in order of preference:**

| Tier | Use when |
|---|---|
| **SQL UDF** | Anything SQL can express, fastest, first choice |
| **JavaScript UDF** | Needs an external library |
| **Remote function** | Needs Python, an external API, or real programming, slowest, last resort |

#### Jupyter notebooks + BigQuery DataFrames

A **notebook** is a document made of code cells: run one cell, its output (table, chart, number) appears underneath, then write the next. Saves **both code and outputs**, so it doubles as a record, a re-runnable job, and something a colleague can check.

Features listed: join/aggregate datasets, parse complex structures with SQL or Python functions, **schedule notebook execution**, integrated BigQuery DataFrames (no setup), and `matplotlib` / `seaborn` for visualization.

**The key line:** `import bigframes.pandas as bf`. Normal Python loads data into the machine's memory, 500 GB won't fit. **BigQuery DataFrames** looks like pandas but runs the work *in BigQuery*, so it handles **datasets that exceed runtime memory**.

Where they live: the **Notebooks** node in the BigQuery Studio explorer (also Colab, Vertex AI Workbench, VS Code).

⚠️ Scheduling a notebook is an *option*, not what a notebook is. Most are never scheduled, they're exploration. Production work usually gets rewritten as a script or moved into Dataform, because notebooks are messy (cells run out of order, dead experiments left in).

#### Saved and scheduled queries

- **Save** queries, **manage versions**, **share** them with others
- **Schedule**: set frequency, start and end times, and a **result destination**

This is the plumbing behind "one job runs at 3am and writes the small table" from the scalable-SQL notes.

#### Why this isn't enough, the argument for Dataform

After a scheduled query runs, you typically still need to:
- **Trigger subsequent SQL scripts**
- **Run data quality tests** on the output
- **Configure security measures**

A scheduled query runs one query and stops. It can do none of those. **Dataform is recommended for more complex SQL workflows**: that gap is the whole reason it exists.

### 3. Dataform

**A serverless framework** that simplifies development and management of **ELT pipelines using SQL**, transforming data within BigQuery, ensuring data quality and providing documentation.

**What it unifies, three things that would otherwise need multiple tools and manual processes:**

| | Without Dataform |
|---|---|
| **Transformation** | Defining tables by hand |
| **Assertion** | Testing data quality separately |
| **Automation** | Managing code and scheduling pipelines manually |

All three are time-consuming and error-prone when handled separately.

**How Dataform and BigQuery work together:**

```
develop SQL + JavaScript
   ↓
Dataform: real-time compilation (dependency checks, error handling)
   ↓
BigQuery: executes the SQL workflows - transformation and materialization
   ↓
on-demand, or scheduled
```

#### Workspace structure

| File / folder | Holds |
|---|---|
| `definitions/` | the **`.sqlx`** files |
| `includes/` | **JavaScript** files (reusable functions) |
| `.gitignore` | managing git commits |
| `package.json` / `package-lock.json` | JavaScript dependencies |
| `workflow_settings.yaml` | project **compilation settings** |
| `README.md` etc. | custom files |

#### The `.sqlx` file, five blocks, in order

```
config           → metadata and data quality tests
js               → reusable JavaScript functions
pre_operations   → SQL statements run BEFORE the main body
[main SQL body]  → the core logic
post_operations  → SQL statements run AFTER
```

#### Reusability, replacing boilerplate

A sprawling `CASE` statement categorizing countries into regions becomes:

```
${mapping.region("country")}
```

Write the mapping once in `includes/`, call it everywhere. Improves readability and maintainability by reducing boilerplate.

#### Four configuration types, worth knowing

| Type | What it does |
|---|---|
| **`declaration`** | Reference an **existing** BigQuery table (your sources) |
| **`table`** | CREATE OR REPLACE a table from a `SELECT` |
| **`incremental`** | Create a table, then **update it with new data** on later runs |
| **`view`** | CREATE OR REPLACE a view, **optionally materialized** |

`incremental` is the practically important one: "append yesterday's partition" rather than rebuilding five years of history nightly, exactly the incremental-processing rule from the scalable-SQL notes.

#### Assertions and operations

- **Assertions** = **data quality tests**, written in **SQL or JavaScript**. Ensure consistency and accuracy; flexible enough for complex checks. e.g. "customer_id is never null" → the pipeline fails loudly instead of shipping bad rows to production.
- **Operations** = **custom SQL statements** run **before, after, or during** pipeline execution.

Together these are the answer to the gap the previous section ended on, a scheduled query can't test its output or trigger the next step.

#### Dependencies, three mechanisms

| Mechanism | How |
|---|---|
| **Implicit declaration** | `ref("table_name")` directly in your SQL, Dataform reads it and infers the order |
| **Explicit declaration** | List them in the config block's **`dependencies` array** |
| **`resolve()`** | Reference a table **WITHOUT creating a dependency**: same name resolution, no ordering edge |

`resolve()` vs `ref()` is the subtle distinction worth remembering.

Dataform compiles user-defined table definitions into executable SQL (e.g. a `customer_details` table created or replaced from a `customer_source` table via `SELECT`), manages the dependencies between them, and orchestrates execution.

#### The workflow graph

SQL workflows are best visualized as a graph. An example:

```
customer_source (declaration)
   ↓
customer_intermediate (table - pre-processed source)
   ↓
customer_rowConsistency (assertions - data quality checks)
   ↓
   ├── customer_ml_training (operation, on validated data)
   └── customer_prod_view (view)
```

Note the assertion sits **in the middle of the graph**, gating everything downstream, not bolted on at the end.

#### Scheduling and execution, two paths

| Path | Mechanisms |
|---|---|
| **Internal triggers** | Manual execution in the Dataform UI; scheduled configurations within Dataform itself |
| **External triggers** | **Cloud Scheduler**, **Cloud Composer** |

**Either way, all workflows execute within BigQuery**: BigQuery's central role in the whole pattern.

## Hands-on: a two-step SQL pipeline in Dataform

The smallest possible pipeline, two steps, built to watch Dataform work out the run order by itself.

```
quickstart-source (view: 4 fruits with counts)
        ↓  ${ref("quickstart-source")}
quickstart-table (table: SUM(count) per fruit)
```

### Repository vs workspace

| | |
|---|---|
| **Repository** | The **git repo**. Shared, holds the finished code. Region set at creation (`europe-west1` here) |
| **Workspace** | Your **personal sandbox**, effectively your own branch. Edit freely; nothing hits the repo until you commit |

Same relationship as a GitHub repo and a local clone. **INITIALIZE WORKSPACE** is what creates the default structure, `definitions/`, `includes/`, `.gitignore`, `workflow_settings.yaml`: i.e. the layout from my Dataform notes above, for real.

The workspace UI shows a **"Commit N changes"** button, confirming it really is git underneath.

### The two files

`definitions/quickstart-source.sqlx`:
```
config { type: "view" }

SELECT "apples" AS fruit, 2 AS count
UNION ALL SELECT "oranges", 5
UNION ALL SELECT "pears", 1
UNION ALL SELECT "bananas", 0
```
A fake data source, no bucket, no database, four rows typed by hand and `UNION ALL`'d.

`definitions/quickstart-table.sqlx`:
```
config { type: "table" }

SELECT fruit, SUM(count) as count
FROM ${ref("quickstart-source")}
GROUP BY 1
```

**`${ref("quickstart-source")}` is the point of the entire exercise.** Not a table name, a reference. Dataform reads it and infers the dependency.

**Visible proof:** with the table file open, the Metadata pane shows **`Dependencies: dataform.quickstart-source`**. Never typed. Derived from the `ref()`.

### The three IAM roles Dataform's service account needs

Copy the service account from the repository-creation screen (`service-<project-number>@gcp-sa-dataform.iam.gserviceaccount.com`), you navigate away before you need it in the grant step.

```bash
export PROJECT_ID=$(gcloud config get-value project)
SA=service-<PROJECT_NUMBER>@gcp-sa-dataform.iam.gserviceaccount.com
for ROLE in bigquery.jobUser bigquery.dataEditor bigquery.dataViewer; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:$SA --role=roles/$ROLE
done
```

| Role | Why |
|---|---|
| **BigQuery Job User** | Run queries at all |
| **BigQuery Data Editor** | Create and write the tables |
| **BigQuery Data Viewer** | Read the source data |

**Skip the "Grant all" button** on the repository-created screen, it only adds `jobUser`, one of the three.

UI path for the same thing: ☰ → **IAM & Admin → IAM** → **VIEW BY PRINCIPALS** → **+ GRANT ACCESS** → paste the SA → add the three roles → Save. Same page used for Storage Object Viewer in the Lakehouse exercise, one place for every permission in a project, whatever the service.

### Execution and the proof

**START EXECUTION** → authenticate with user credentials → **Select actions to execute** → *Select all* → OK → START EXECUTION → Allow.

Execution log, Status **Success**:

| Time | Action |
|---|---|
| 8:38:**17** | `quickstart-source` (the view) |
| 8:38:**17** | `first_view` |
| 8:38:**20** | `quickstart-table` ← waited 3 seconds |
| 8:38:**20** | `second_view` |

**The table ran 3 seconds after the view, because it depended on it.** The two `_view` files (leftover samples from workspace initialization) had no dependencies and went immediately. None of that ordering was specified, one `${ref()}` produced it.

Output lands in a BigQuery dataset called **`dataform`** (the default repository setting).

### Gotchas hit

- **The Create-file dialog pre-fills `definitions/`**: type only the filename after it, or you get `definitions/definitions/...`.
- **An error in the compiled-queries pane before permissions are granted is expected**: ignore it.
- **The IAM page caches**: a freshly granted role may not appear until you refresh. The `gcloud` output is the truth; check the final policy block it prints.

### Third time for the service account pattern

Datastream → Lakehouse → Dataform. Every time: **create a robot, give the robot roles, the robot does the work.** It is the core Google Cloud idiom, and worth recognising on sight.
