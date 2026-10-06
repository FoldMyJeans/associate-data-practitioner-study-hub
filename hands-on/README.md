# What I learned working hands-on with Google Cloud

These are notes from hands-on exercises I worked through on Google Cloud, in the order I did them. For each one I wrote down what I built, what it taught me, and what tripped me up. The console changes often, so written steps can be out of date, and the notes below point out where.

These are notes, not walkthroughs. I do not list every step.

The domains I refer to come from the public exam guide:

1. Data preparation and ingestion
2. Data analysis and presentation
3. Data pipeline orchestration
4. Data management

## Summary

| Exercise | Domain | What you build |
|---|---|---|
| Three ways to load files into BigQuery | 1 | One BigQuery dataset filled three ways: console upload, command line, and SQL |
| Copy a PostgreSQL table into BigQuery and keep it in sync | 1 | A stream that copies a PostgreSQL table into BigQuery and keeps it in sync |
| A table over a bucket file with locked columns | 4 | A table over a file in a bucket, with sensitive columns locked |
| A two-step SQL pipeline in Dataform | 3 | A two-step SQL pipeline that works out its own run order |
| Run a Spark job with no cluster | 1 and 3 | A Spark job with no cluster that loads an Avro file into BigQuery |
| A streaming job that feeds a live chart | 1, 2 and 3 | A streaming job from a template that feeds a live dashboard |
| Move query results from BigQuery into a MySQL database | 1 and 2 | Query results moved from BigQuery to a bucket to a MySQL database |
| Load and query a table in the BigQuery console | 1 and 2 | Your own table, loaded from a file with a typed schema, then queried |
| Load and query a table with `bq` commands | 1 and 2 | The same idea, done with `bq` commands |
| Find duplicates and answer business questions on web store data | 2 | A duplicate check on raw web data, then business questions in SQL |
| Fix broken queries one error at a time | 2 | Broken queries fixed one error at a time |
| A report page on a BigQuery table in Data Studio | 2 | A report page on top of a BigQuery table |
| Train a purchase prediction model in SQL | 2 | A model in SQL that predicts which visitors will buy |
| Prototype a gen AI agent without code | 2 | A prototype gen AI agent, built without code |
| Analyze text with a ready-made model | 1 and 2 | Calls to a ready-made text analysis model |
| Train a loan default model with no code | 2 | A no-code model that predicts loan default |

## Three ways to load files into BigQuery

**What you build**

A small dataset of taxi trips, loaded three ways. One CSV goes in through the console, a second CSV is appended from a Cloud Storage bucket with the `bq` command line tool, and a third table is created from a query.

**What it taught me**

- Domain 1: the three common ways into BigQuery. Console upload, `bq load` from a `gs://` path, and SQL.
- The shape of a load command. A `gs://` path means BigQuery reads the file straight from Cloud Storage, with no download.

  ```
  bq load --source_format=CSV --autodetect --noreplace \
    my_dataset.my_table \
    gs://my-bucket/my-file.csv
  ```

  `--noreplace` appends to the table. `--replace` overwrites it.
- Schema auto detect. BigQuery guesses column names and types from the header and some sample rows. Every table has a schema, so the real choice is who writes it. Auto detect is fine for exploring. A schema you define yourself is safer in production, where a wrong guess matters. Example: a code like `01` read as a number loses its leading zero.
- CTAS, short for `CREATE TABLE ... AS SELECT`. The query defines both the columns and the rows of the new table. The result is a frozen copy and does not stay in sync with the source. This is the "T" (transform) in ELT.

  ```sql
  CREATE TABLE shop.orders_2024 AS
  SELECT *
  FROM shop.orders
  WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'
  ```

- Table versus view. A table stores copied data, costs storage, and is fast to read. A view stores only the SQL text, is always current, and recomputes on every query. In Power BI terms, a table is like Import mode and a view is like DirectQuery.
- DDL, DML and queries are all SQL with different jobs. DDL defines structure (`CREATE`, `ALTER`, `DROP`). DML changes data (`INSERT`, `UPDATE`, `DELETE`). A query (`SELECT`) reads.
- Window functions and `QUALIFY`. `MAX(x) OVER()` computes a value across all rows and keeps every row, unlike `GROUP BY`, which collapses them. `QUALIFY` filters on a window result, which `WHERE` cannot do, because `WHERE` runs before window functions exist. The logical order is: FROM, WHERE, GROUP BY, HAVING, window functions, QUALIFY, SELECT, ORDER BY, LIMIT.
- Cost. On-demand BigQuery bills by bytes scanned in the columns you touch. So avoid `SELECT *`, partition by date, cluster by the columns you filter on most, and filter a partition column with a plain range instead of wrapping it in a function.
- Domain 3 link: "aggregate once, read many times". One scheduled job writes a small summary table and dashboards read that. A chain of scheduled CTAS statements, run by a tool like Managed Service for Apache Airflow (used to be Cloud Composer), is what a simple data pipeline looks like in practice.
- Dataset names such as `landing`, `raw` and `curated` are only a naming habit. SQL gives them no meaning. Knowledge Catalog (used to be Dataplex) is where real governance on those zones lives.

**Things that tripped me up**

- Nothing in the console went wrong in this exercise. The traps were in the SQL.
- `LIMIT 5` alone means "any 5 rows". Tables have no built-in order, so this is for peeking, not for answering a question.
- `ORDER BY x DESC LIMIT 5` still reads the whole column. The top value could be in the last row read.
- A CTAS table looks like the source on day one and then quietly goes stale.

## Copy a PostgreSQL table into BigQuery and keep it in sync

**What you build**

A Cloud SQL for PostgreSQL database with one small table, and a Datastream stream that copies that table into BigQuery. Then you insert, update and delete rows in PostgreSQL and watch BigQuery follow by itself. You only ever write to PostgreSQL. That is the whole point of the exercise.

**What it taught me**

- Domain 1: replication with Datastream, and its two modes. Backfill copies the rows that already exist. CDC (change data capture) reads the database's change log and replays each insert, update and delete at the destination. The exercise shows them in that order.
- The moving parts are two connection profiles and one stream. A profile is only a saved set of connection details and does nothing by itself. The stream is the job that uses both.
- The source profile needs a host, port, user and password, because the database is a separate system with its own login. The BigQuery profile needs none of that. It is in the same project, so permission comes from IAM.
- What a source database must allow before it can be replicated:
  - A network allowlist. A new Cloud SQL instance accepts connections from nobody. Google publishes a list of Datastream addresses per region, and all of them must be allowed, because Datastream runs on a fleet of machines.
  - Logical decoding switched on. It is off by default. It makes PostgreSQL write its change log in a form an outside reader can use.
  - A publication, which says what may be read (the list of tables).
  - The replication permission on the user, which says who may read.
  - A replication slot, which records how far the reader got. Think of it as a bookmark. If the reader is offline for an hour, the database keeps the unread changes and nothing is lost.
- Replica identity tells PostgreSQL to identify rows by primary key in the change log. Updates and deletes need it to replicate correctly.
- The staleness limit on the BigQuery side is the cost versus freshness setting. Zero means apply changes right away instead of batching them.
- The BigQuery table ends up with the source columns plus a `datastream_metadata` record column. That is the payload and the metadata of each change event, side by side.
- For real work: ask for a dedicated account for the replication job, never a person's login. No encryption is a practice shortcut. Real connections use SSL.

**Things that tripped me up**

- The database password prompt failed at once, before I could type. Leftover line breaks from an earlier multi-line paste were sitting in the terminal and got read as an empty password. Pressing Enter a few times and trying again fixed it.
- A query failed with a "dataset was not found in location US" error. The query ran in the US location while the data lived in the practice project's region. Opening the dataset from the Explorer and starting the query from there picks the right location.
- The Preview tab said there was no data even though rows existed. Preview cannot see rows that are still streaming in. Run a `SELECT` instead. This is expected.
- I did this exercise twice and the region was different the second time. The allowlist is per region, so always use the address list for the region you are working in on that run.
- A trap for real systems: a stream that is stopped and never deleted leaves its slot asking the database to keep changes forever. The change log grows until the disk fills.

## A table over a bucket file with locked columns

**What you build**

A connection resource, a Lakehouse (used to be BigLake) table over a CSV file in a bucket, policy tags on a few sensitive columns, and an in-place upgrade of a plain external table to a Lakehouse table.

**What it taught me**

- Domain 4: column-level security on data that lives in files, not in BigQuery storage.
- External table versus Lakehouse table. Both point at files in Cloud Storage. The only difference is that a Lakehouse table names a connection. BigQuery then reads the file through the connection's service account (a non-human account that a service uses), so the person running the query needs no access to the bucket.
- Why that matters. If users cannot reach the bucket, BigQuery is the only door to the data, and that is what lets BigQuery decide which columns each person sees.
- A policy tag is a label you attach to a column. It denies access by default, even to a project owner. Access comes from a separate role granted on the tag: Fine-Grained Reader to see real values, or Masked Reader to see masked values.
- Once columns are tagged, a `SELECT *` fails for anyone without the role. Leaving the tagged columns out works:

  ```sql
  SELECT * EXCEPT(address, phone, postal_code)
  FROM my_dataset.customers
  ```

- After moving people to Lakehouse tables, remove their direct Cloud Storage permissions. Otherwise they can open the file in the bucket and the column rules protect nothing.
- Upgrading an external table needs no rebuild and no reload. Three `bq` commands do it: `bq mkdef` writes a definition that says "these files, read through this connection", `bq show --schema` saves the current column list, and `bq update` applies both. The saved schema matters because `bq update` replaces the whole definition. Without it, auto detect would guess the types again.
- Metadata caching for external tables exists and is off by default. BigQuery shows a banner about it after a query.

**Things that tripped me up**

- The "+ Add data" button was not where the written steps show it. I created the connection and granted the bucket role from Cloud Shell instead, and that worked. Another route that worked: in the Explorer tree, open Connections, then the three-dot menu, then Create connection.
- On my second run the written steps told me to search for Agent Platform (used to be Vertex AI) to find the connection type. Typing that into the console's main search bar opens the Agent Platform product page, which is the wrong place. The search belongs inside BigQuery's Add data panel.
- The create table form defaults to a native table. You have to switch it to External table yourself.
- The Add policy tag button only appears in Edit schema mode. The read-only Schema tab lets you tick fields but shows no tag button.
- Naming: newer material says Lakehouse and older material says BigLake. It is the same product, and both names still show up in the console.
- The two steps that carry the real lesson are the query and the policy tags. Do not skip them.
- To confirm the upgrade worked, open the table details. A connection ID appears under the external data configuration and the table is marked as a Lakehouse table.

## A two-step SQL pipeline in Dataform

**What you build**

The smallest possible pipeline in Dataform: one view with a few hand-typed rows, and one table that sums the view. You run it and watch Dataform decide the order by itself.

**What it taught me**

- Domain 3: Dataform is the tool for SQL pipelines inside BigQuery, with version control built in.
- Repository versus workspace. The repository is the shared git repo that holds the finished code. A workspace is your personal sandbox, like your own branch. Nothing reaches the repository until you commit.
- Initializing a workspace creates the standard layout: a `definitions` folder for your SQLX files, an `includes` folder, and a settings file.
- A SQLX file is SQL with a small `config` block on top that says what to create, such as a view or a table.
- `${ref("name")}` is the idea the whole exercise exists for. It is a reference, not a table name. Dataform reads it and works out that one step depends on another.

  ```sql
  config { type: "table" }

  SELECT product, SUM(quantity) AS total_quantity
  FROM ${ref("orders_view")}
  GROUP BY product
  ```

  With that file open, the metadata pane lists the dependency, even though I never typed it anywhere else. In the execution log the table started a few seconds after the view it depends on.
- Dataform's service account needs three BigQuery roles: Job User to run queries, Data Editor to create and write tables, and Data Viewer to read source data.
- Output lands in a dataset named `dataform` unless you change the repository settings.
- The pattern to recognize: create a service account, give it roles, and it does the work. Datastream, Lakehouse connections and Dataform all work this way.

**Things that tripped me up**

- The service account name is shown on the repository-created screen. Copy it then. You leave that screen before the step that needs it.
- The quick "Grant all" button on that screen added only one of the three roles when I used it. I granted all three myself.
- The create file dialog already contains `definitions/`. Type only the file name after it, or you get a folder inside a folder. A path typed with a colon also creates a stray folder.
- An error in the compiled queries pane before the roles are granted is expected. Ignore it.
- The IAM page caches. A role I had just granted did not show until I refreshed. The output of the `gcloud` command is the reliable record.
- A new workspace comes with two sample view files. Ignore them. They also show up in the execution log.

## Run a Spark job with no cluster

**What you build**

A Spark batch job that runs with no cluster and copies an Avro file from Cloud Storage into a BigQuery table. The job comes from a prebuilt template, so you pass settings and write no Spark code.

**What it taught me**

- Domain 1: where Spark fits. Managed Service for Apache Spark (used to be Dataproc) is the natural choice when there are existing Hadoop or Spark workloads.
- Cluster versus serverless. With a cluster you create machines, wait, run the job, and delete the cluster. With serverless you submit the job and Google finds the machines. It is the same Spark with nothing to size, patch or remember to delete.
- A plain `bq load` could do this job in one line. The point is the mechanism, not the result.
- Templates. You supply parameters such as input format, input location, output dataset, output table and write mode. Dataflow has templates too. They are a different product with the same idea, so do not mix them up.
- Spark cannot write to BigQuery by itself. A connector library (a JAR file) teaches it how.
- The job uses two buckets with different jobs: one for staging the data and Spark's own files, one as temporary space that BigQuery needs while it builds the table.
- Private Google Access lets machines with no public IP reach Google services. The Spark workers need it.
- File formats:

  | Format | Layout | Best at |
  |---|---|---|
  | CSV | Text rows, no types | Handing a file to a person |
  | Avro | Row by row, binary, schema stored in the file header | Data in motion: feeds, streams, messages |
  | Parquet | Column by column, binary | Data at rest that will be queried |

  Because Avro carries its own schema, the columns of the new table came from the file. Nobody typed them.
- Serverless Spark has two modes. Batches are started by a command and suit scheduled jobs, which is the Domain 3 angle. Interactive sessions run from a notebook and suit exploration.
- `bq` is only a client. The table lives in BigQuery and is just as visible in the console.

**Things that tripped me up**

- There are two terminals and they are easy to mix up. Cloud Shell is the small free machine every account gets. The SSH window belongs to a VM set up for the exercise. The later steps all run in the SSH window.
- The VM is only a workbench for files and commands. It does not do the processing. Spark reads from the bucket, not from the VM, so the file has to be copied into the bucket first.
- Environment variables live only in the terminal session where you set them. Open a new window and they are gone.
- Expected noise: a warning about a missing metadata directory is harmless. A "batch job has failed" message can also appear, and that is expected too. Wait a little and run the command again.

## A streaming job that feeds a live chart

**What you build**

A partitioned BigQuery table, a Dataflow streaming job started from a Google template, and a Data Studio (used to be Looker Studio) chart on top. The job reads a CSV file of taxi rides from Cloud Storage, passes each line through a small JavaScript function, and writes rows to BigQuery while you query them.

**What it taught me**

- Domain 3: a Dataflow template lets you run a pipeline with parameters only. No Apache Beam code is written.
- Domain 1: the destination table is created first with a defined schema and is partitioned on the timestamp column. A UDF (user-defined function) turns each raw line into a row.
- The job graph is Beam's standard shape: read, transform, write. It also has a side path that sets failed rows aside instead of crashing the job. That side path is called a dead-letter path.
- A streaming job never ends by itself and bills until you stop it. Stopping the job is a step of its own for that reason.
- This version of the exercise does not use Pub/Sub. The more common streaming shape is Pub/Sub for ingestion, Dataflow for transformation, and BigQuery for analysis.
- Domain 2: the dashboard reads from a saved query, so the chart updates as rows arrive.

**Things that tripped me up**

- My first job failed because no worker could start in the zone it picked. The status still said Running, but the worker count stayed at 0 and the boxes in the graph turned to Failed. I stopped the job, cloned it under a new name, and set a different worker zone. The zone setting is under the optional parameters, not the regional endpoint dropdown.
- "Job successfully created" does not mean it works. The proof is one current worker and no red log lines.
- Field names in the template form differ from the written steps. The `gs://` prefix is already filled in, so paste paths without it.
- The default machine series was N1. Unticking the default box still showed N1 until I changed the series to E2.
- A Gemini Cloud Assist panel opens if you click the icon on an error row. It is not needed. Cancel it.
- In Data Studio the chart types sit in their own row of the properties panel. The search box there searches settings, not charts.
- The data is a replayed sample file from 2022, not live traffic.
- I ran out of time before the last chart step. Budget time for it.

## BigQuery and SQL exercises

The next six exercises are all about querying BigQuery data and reporting on it. General tips for that kind of work:

- If you can explain every query in them, you have the core skills: filtering, grouping, counting distinct values, and reading an error message.
- Before writing a query, open the Schema, Details and Preview tabs of the table. Most wrong answers start with a wrong guess about a column.
- Decide what one row of your result should mean before you write `GROUP BY`.
- When time is limited, run the main queries first and explore afterward.

## Move query results from BigQuery into a MySQL database

**What you build**

You query a large public bike share table in BigQuery, export the results as CSV files, put them in a Cloud Storage bucket, and import them into a Cloud SQL MySQL database. Then you add, delete and combine rows there.

**What it taught me**

- Domain 1: choosing between Cloud SQL and BigQuery. Cloud SQL is the cash register and BigQuery is the accountant. An app that saves one order or one bike unlock at a time writes to Cloud SQL. A question like "busiest station across 83 million trips" belongs in BigQuery. Real systems often have both, with data copied from one to the other.
- The hierarchy. Cloud SQL is instance, database, table. BigQuery is project, dataset, table. BigQuery has no instance because it is serverless.
- Public datasets. Google loads open data and pays for the storage, and you pay only for your queries. It is the same idea as BigQuery sharing (used to be Analytics Hub): you query the data in place and no copy is made.
- Ways to save query results: a small local CSV download (up to 10 MB), a larger CSV to Google Drive, a file in Cloud Storage, or a new BigQuery table. Cloud Storage is the realistic choice because it skips the laptop.
- Bucket names are globally unique.
- Cloud SQL imports from a bucket, and MySQL needs the table and its column types created first. The data must fit those types.
- Domain 2: SQL basics. `WHERE`, `GROUP BY`, `COUNT(*)`, `ORDER BY`, and `UNION`.
- `UNION` removes duplicates only when the whole row matches. It stacks rows and does not connect tables. If two lists share a name but have different numbers, both rows stay. To tell them apart, add a label column, or use a `JOIN` to put the numbers side by side.
- Query results sit in a temporary table for about a day, so running the same query again is free. That is not a table you own.
- Starring a project is only a bookmark. The full `project.dataset.table` name in backticks works without it.

**Things that tripped me up**

- The Cloud SQL instance takes 5 to 10 minutes to build. Start it first and do the BigQuery part while it builds.
- The header row of my CSV was imported as data. My row count was one too high, and the header text in a number column silently became 0. No error does not mean a correct load, so check row counts after every load. Before running a `DELETE` to clean up, run the same `WHERE` as a `SELECT` to see what it would remove.
- The CSV file names in the written steps do not match each other. Use the name of the file you actually uploaded.
- The Cloud SQL create page opens with a large free trial form at the top. I used the Development preset instead and then set the edition and version the exercise asks for. A bigger preset can get a practice project shut down.
- The BigQuery console is newer than the written steps. There was no "+ Add data" in the Explorer. I skipped starring the public project and used full table names.
- The Cloud SQL page showed "Knowledge Catalog integration". Knowledge Catalog (used to be Dataplex) is the current name.

## Load and query a table in the BigQuery console

**What you build**

Your own dataset and table. You load a text file from a public bucket, type the schema yourself, and query the result. The earlier exercise only read Google's tables.

**What it taught me**

- Domain 1: creating a dataset and loading a file from Cloud Storage with a defined schema.
- A schema written as text is a list of `column:type` pairs, matched to the file by position. The wrong order breaks the load. Types matter for sorting: as text, "9000" sorts above "19000".
- If a file has a header row, set "header rows to skip" to 1 in the advanced options. The MySQL import in the previous exercise had no such setting.
- Domain 2: cost is bytes scanned, and the console shows the estimate before you run the query. On-demand pricing is roughly 6.25 US dollars per TB, and the first 1 TB of queries each month is free. The practice project runs in the free sandbox mode.
- `LIMIT` does not lower cost. A "top 10" still reads the whole column. Selecting fewer columns lowers cost. The Preview tab is free.
- A table name with no project in front means your own project. Tables in other projects need the full `project.dataset.table` name.
- Data quality. In my "heaviest" result, the top rows all had the exact same weight, which looks like a cap or a placeholder value. Question outliers before you trust a top or bottom answer.

**Things that tripped me up**

- The file picker said "No buckets found". It only lists buckets in your own project, and the file is in Google's. Type the path into the field instead of browsing.
- The Cloud Shell setup in the written steps is not needed. Everything happens in the console.

## Load and query a table with `bq` commands

**What you build**

The same idea as the console exercise, typed as `bq` commands in Cloud Shell. This time the file is downloaded from a public website instead of read from a bucket.

**What it taught me**

- Domain 1 and 2: the `bq` tool and how each command maps to the console.

  | Command | What it does | Console equivalent |
  |---|---|---|
  | `bq show project:dataset.table` | Schema, row count, size (free) | Details tab |
  | `bq head -n 5 dataset.table` | First rows (free) | Preview tab |
  | `bq query --use_legacy_sql=false '...'` | Runs SQL | Run button |
  | `bq ls` | Lists datasets | Explorer pane |
  | `bq mk my_dataset` | Creates a dataset | Create dataset |
  | `bq load my_dataset.my_table file.txt name:string,count:integer` | Creates a table and loads it in one step | Create table form |
  | `bq rm -r my_dataset` | Deletes a dataset and its tables | Delete |

- Colon versus dot. `bq` commands write `project:dataset.table`. Inside SQL it is all dots.
- `--use_legacy_sql=false` is a flag, not SQL. BigQuery has two dialects. Legacy SQL is the old one and writes table names in square brackets with a colon. GoogleSQL is the current one and uses backticks and dots. The console uses GoogleSQL, and adding the flag makes `bq` do the same.
- String matching is case-sensitive. The same word with and without a capital letter comes back as two rows. Wrap the column in `LOWER()` to merge them.
- A bucket is not a dataset. A bucket holds files in Cloud Storage. A dataset holds tables in BigQuery.
- Cloud Shell is a small free temporary machine with 5 GB of home disk. Treat it as a scratch desk, not as storage.
- Two routes to the same table. Loading from a bucket keeps the file, has no practical size limit, and can be scheduled. Loading from the Cloud Shell disk is for quick one-off tests.
- Before loading any file, look at its first lines and read its documentation. Does it have a header? What is the delimiter? What does each position mean? This file had no header, so I named the columns in the load command.
- Tables in the sandbox expire after 60 days.
- `bq rm` refuses to delete a dataset that still has tables unless you add `-r`. It is a safety catch.

**Things that tripped me up**

- My row count did not match the number in the written steps. The public source revises past years, so a fresh download can differ.
- The download from the public website worked for me, but a site can block it. If that happens the file needs another route, such as a copy in a bucket.
- The downloaded zip contains more yearly files than the written steps show.

## Find duplicates and answer business questions on web store data

**What you build**

Nothing permanent. You explore web analytics data from an online store, where one row is roughly one page hit by one visitor. First you find duplicate rows in the raw table. Then you answer business questions on the clean table: visitors, channels, most viewed products, orders.

**What it taught me**

- Domain 2: reading a table before querying it. The Schema tab shows columns and types, Details shows row count and size, and Preview shows sample rows for free.
- Finding duplicates. Group by the columns that should be unique and keep groups with more than one row. Zero rows back means the check passed.

  ```sql
  SELECT order_id, COUNT(*) AS row_count
  FROM shop.orders
  GROUP BY order_id
  HAVING row_count > 1
  ```

- Checking on the business key beats checking on all columns. If the key columns have no duplicates, the full rows cannot have any either. The reverse is not true: two rows with the same key and one different value pass an all-columns check, and that is the inconsistency you want to catch.
- To keep only the latest row per key, use `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY loaded_at DESC) = 1`. In a pipeline, automate the check, for example with a uniqueness assertion in Dataform.
- Why duplicates matter: this data stores money multiplied by one million, and a duplicated transaction row doubles revenue in a plain `SUM`.
- `WHERE` filters rows before grouping. `HAVING` filters groups after, so it can use `COUNT(*)`.
- `COUNT(*)` counts every row. `COUNT(column)` skips nulls. `COUNT(DISTINCT column)` counts each value once. In this data the quantity column is null on rows with no purchase, so counting it gives orders and summing it gives units. Average per order must divide by orders, not by all rows.
- Unique visitors per channel add up to more than total unique visitors, because one person can arrive through two channels.
- A column alias is only a label. Naming a count "product_views" does not make it count only views. The filter does that.
- `ORDER BY` on a name column sorts alphabetically. To get the biggest first, sort by the number.
- `GROUP BY 1, 2, 3` means the first three columns of the `SELECT`.
- A `WITH` clause (a CTE) is a named temporary result inside one query. Nothing is stored. It pays off when the logic has several steps, like `VAR` and `RETURN` in DAX.
- What to group by: the words after "per" or "each" in the question. Check yourself by finishing the sentence "one row in my result is one ...".

**Things that tripped me up**

- I ran out of time on my first try, at the last step, because I spent the time on concepts. I had to start the exercise again. When time is limited, run the main queries first and explore afterward.
- The new BigQuery Studio screen had no "Star a project by name" option, and searching for the shared project in the Explorer found nothing. What worked: type the full table name in backticks in a query tab and Ctrl+click it. The table opens with its Schema, Details and Preview tabs, and the project then shows in the Explorer.

## Fix broken queries one error at a time

**What you build**

Nothing new. You get a set of broken queries on the store's transactions table and fix them one error at a time until each one answers its business question.

**What it taught me**

- Domain 2: reading error messages and knowing the usual causes. The exercise covers an empty `SELECT`, a typo in a project name, a legacy table name, a missing comma, an aggregate with no `GROUP BY`, a missing `DISTINCT`, a filter on an aggregate in the wrong place, and null values in a name column.
- Legacy versus GoogleSQL, from the table name alone. Square brackets and a colon mean legacy. Backticks and dots mean GoogleSQL. A query marked as standard SQL with a legacy table name fails.
- A query that runs is not always useful. Selecting one ID column with no aggregation, order or limit returns every row with repeats.
- A missing comma creates a silent alias. `AS` is optional in SQL, so two column names in a row are read as "first column, renamed to the second". You get one column with the wrong label and no error.

  ```sql
  # Missing comma: this returns ONE column, customer_id, labeled order_total
  SELECT customer_id order_total
  FROM shop.orders
  ```

- A plain column next to an aggregate must be in the `GROUP BY` or wrapped in an aggregate itself. After grouping by city there is one row per city, and a plain column would have several values for one cell.
- "How many different X" means `COUNT(DISTINCT X)`. Rows of pen, pen, pen, notebook give a `COUNT` of 4 and a `COUNT(DISTINCT)` of 2.
- `WHERE` versus `HAVING` by run order. SQL runs FROM, WHERE, GROUP BY, HAVING, SELECT, ORDER BY, not the written order. `WHERE` sees single rows before any sum or alias exists. BigQuery lets `HAVING` and `ORDER BY` use an alias from the `SELECT`. `WHERE` never can.
- You query tables, not datasets. A dataset is only a folder. Using several tables in one query means a `JOIN`, and that works across datasets and projects.

**Things that tripped me up**

- The query with the missing comma runs without an error. It is easy to move on without noticing the wrong label.

## A report page on a BigQuery table in Data Studio

**What you build**

A Data Studio (used to be Looker Studio) report connected to a BigQuery table with one row per product. You add a table chart, format a ratio as a percent, and lay out a small inventory watchlist page.

**What it taught me**

- Domain 2: Data Studio is Google's free dashboard tool, close to Power BI in purpose. The name went from Data Studio to Looker Studio in 2022 and back to Data Studio in 2026. It is not the same product as Looker.

  | | Data Studio | Data Studio Pro | Looker |
  |---|---|---|---|
  | What it is | Self-service dashboards | The same app plus team workspaces, project ownership and support | Enterprise BI platform |
  | Cost | Free | Paid per user | Paid license |
  | Who builds | Anyone, drag and drop | Same | A data team defines a LookML model first |

- The BigQuery connector can read a table in a shared project, not only in your own.
- Dimension versus metric. A dimension is what you group by: text, dates, IDs. A metric is a number that gets aggregated. If you group by product name instead of SKU, a product with three sizes becomes one row and its stock levels are summed.
- Changing a field to Percent is display formatting in the report. The BigQuery column does not change.
- Who does what. Building the report is analyst work. Building the table and the pipeline that refreshes it is data engineering work.

**Things that tripped me up**

- Searching for "Data Studio" in the Cloud console opens the Data Studio Pro product page. The practice account has no permission to create it. Do not start the trial. Open the Data Studio site directly.
- My incognito window still held the account from the previous exercise. The sign-in page asked me to verify the old account and the new password failed. The fix was to close every incognito window, open a fresh one, and sign in again from the start.
- Check the avatar before connecting data. A photo means a personal account. The practice account shows a plain icon. Connect BigQuery only as the practice account.
- Actions outside the plan can get a practice account blocked. Stay on the path.
- One part of the written steps is titled as if you build an interactive filter. It only adds a dimension and a metric. No filter control gets built.
- The app shows a banner about the rename, so expect both names on screen.

## Train a purchase prediction model in SQL

**What you build**

A logistic regression model, written in SQL inside BigQuery, that predicts whether a website visitor will buy. You train it on web analytics session data, evaluate it, and then rank visitors and countries by predicted purchase probability.

**What it taught me**

- Domain 2: BigQuery ML lets you train and use a model with SQL, on data that is already in BigQuery.

  | Statement | What it answers |
  |---|---|
  | `CREATE MODEL` | Build the model. The options name the model type and the label column |
  | `ML.EVALUATE` | How good is it? Precision, recall, accuracy, F1, log loss, AUC |
  | `ML.PREDICT` | Run it on new rows. Returns the predicted class and the probability behind it |
  | `ML.CONFUSION_MATRIX` | The four-cell table of right and wrong predictions |

  ```sql
  CREATE OR REPLACE MODEL my_dataset.my_model
  OPTIONS(model_type = 'logistic_reg', input_label_cols = ['label']) AS
  SELECT * FROM my_dataset.training_data
  ```

- Label versus feature. The label is the column you want to predict. Features are the columns used to predict it. Here the label is built with `IF(condition, value_if_true, value_if_false)`: 1 if the session had a transaction, 0 if not.
- AUC versus recall. My model scored an AUC of 0.98 and a recall of about 10 percent. Both are true.
  - AUC measures ranking: given one buyer and one non-buyer, how often does the model score the buyer higher? It ignores where you draw the line.
  - Recall measures catching: of all real buyers, how many were flagged? That depends on the cut-off, which defaults to 0.5.
  - Only about 1.5 percent of visitors buy. With a base rate that low, the model rarely gives anyone a probability above 0.5, so almost nobody is flagged.
  - The low recall is a property of the threshold, not of the model. Lower the cut-off and recall jumps.
- Ranking tool versus decision tool. "Who are the 1,000 visitors most likely to buy" only needs a good order, so a high AUC is enough. "Approve this claim, yes or no" needs a hard cut-off, and the cut-off is a business choice about the cost of each kind of mistake. Ask which one you are building before asking if the model is good enough.

**Things that tripped me up**

- The training view uses `LIMIT` with no `ORDER BY`. Tables have no built-in order, so each time the view is read BigQuery may return different rows. My evaluation and my confusion matrix disagreed on how many real buyers there were, although both read "the same" view. The rule I kept: `LIMIT` without `ORDER BY` is a sample, not a fixed subset. If something must be repeatable, order it or save it as a table.
- A high AUC next to a very low recall looked like a mistake. It is not. See the threshold point above.

## Prototype a gen AI agent without code

**What you build**

A prototype gen AI agent in Agent Studio, the low-code visual tool in Agent Platform (used to be Vertex AI). You write system instructions, add a few examples, set model parameters, and try the agent in the built-in test panel.

**What it taught me**

- Domain 2, lightly. The exam guide only touches this area, but the vocabulary is worth knowing.
- System instructions are the standing rules for the agent: its role, tone and limits.
- Few-shot examples are a handful of sample inputs with the outputs you want. They show the model the pattern instead of describing it.
- Model parameters change how the model answers. You can prototype all of this without writing code.

**Things that tripped me up**

Assume any written steps are stale for this tool. The screen has moved on, so look for the modern equivalent instead of hunting for the exact control the steps name.

- The System instructions box was hidden until I expanded the right-hand panel with the double-chevron button.
- The few-shot steps tell you to fill an Input field. That field no longer existed. The example attached silently and I only had to type the new case.
- The written steps say to set Temperature and Top-P. On the Gemini model I was given, those settings were not there. It showed a "Thinking level" setting instead.
- A related BigQuery step pointed to a Gemini sparkle menu with an "Explain this query" option. It was gone. The Gemini Cloud Assist panel and its suggested prompts did the same job.

## Analyze text with a ready-made model

**What you build**

An API key, and then a series of calls to the Cloud Natural Language API from a terminal on a VM. Each call sends a sentence in a small JSON file and gets structured data back.

**What it taught me**

- Domain 1: turning unstructured text into structured data you can group and chart.
- Domain 2: using a pre-trained model. You did not train it and it never runs on your machine. Your text goes to Google's server, the model runs there, and JSON comes back. The pattern is always key, request, call, parse.
- The API key is a ticket. It proves you are allowed in and says who gets billed. It does no analysis.
- One call shape covers every kind of analysis. Only the sentence and the method name in the address change.

  ```
  curl "<API endpoint>/documents:analyzeEntities?key=${API_KEY}" \
    -s -X POST -H "Content-Type: application/json" \
    --data-binary @request.json > result.json
  ```

  `-X POST` sends data. The header says the body is JSON. `@request.json` reads the body from a file. `>` saves the reply to a file.
- The five methods:

  | Method | What it breaks the text into |
  |---|---|
  | `analyzeEntities` | Things: people, places, organizations, works of art |
  | `analyzeSentiment` | Sentences, each with a score |
  | `analyzeEntitySentiment` | Things, each with its own score |
  | `analyzeSyntax` | Single words (tokens) and their grammar |
  | `classifyText` | One topic for the whole document |

- Salience is a number from 0 to 1 for how central an entity is to the text. It separates what a text mentions from what it is about.
- Sentiment has two numbers. Score runs from negative to positive. Magnitude is the total amount of emotion in either direction. A review that is half glowing and half furious has a score near zero, the same as a flat one. Magnitude tells them apart.
- Entity sentiment is the one that earns its keep. "I liked the food but the service was terrible" averages out to neutral as a whole. Scored per entity, it says the food is fine and the service is the problem.
- An entity is a thing, not a string. The same country named in two languages comes back with the same Knowledge Graph ID, so reviews in several languages can be grouped without translating first. The language is detected automatically if you do not state it.
- When to choose this. Pick a pre-trained API when you need the same structured numbers every time, cheaply and at volume, and you have no labeled data. Do not pick it when the labels are specific to your business, such as "is this support ticket a false alarm". That is a job for AutoML or BigQuery ML.
- To analyze a file in Cloud Storage instead of inline text, the request points to the file with `gcsContentUri` in place of `content`.

**Things that tripped me up**

- SSH in the browser failed with a "Cloud Identity-Aware Proxy" not authorized error (code 4033). The error dialog has a button to retry without the proxy, and that connected. A fallback is `gcloud compute ssh` from Cloud Shell. Written steps assume SSH just works. It often does not.
- `export API_KEY=...` prints nothing. Check it with `echo $API_KEY`.
- The sentence scores described in the written steps did not match what the API returned for me. Trust the output.
- The model is not always right. In my results a nationality came back as a location and a job title came back as a person.
- The syntax output is full of fields set to UNKNOWN. That does not mean the model failed. The response format is shared across all languages, and some grammar features do not exist in English.

## Train a loan default model with no code

**What you build**

A tabular dataset in Agent Platform (used to be Vertex AI) and an AutoML classification model that predicts whether a borrower will default on a loan, with no code. Then you send one prediction request to a model that is already deployed.

**What it taught me**

- Domain 2: the full machine learning workflow, once, end to end: prepare data, train, evaluate, deploy, predict. The shape matters more than the loan model.
- Column roles. One column is the target (the thing to predict). An ID column is excluded, because an arbitrary label cannot cause anything and the model may find fake patterns in it. The rest are features.
- AutoML for tabular data needs at least 1,000 rows.
- Budget is measured in node hours. One node hour is one machine for one hour. The budget is a spending cap on the search, not a duration. Early stopping ends training when the scores stop improving, so leave it on.
- The confusion matrix is built from the test set, the rows held back during training. Each row of the matrix is one true label and adds up to 100 percent.
- A prediction returns raw probabilities for each class, not a yes or no. Applying a threshold to turn that into approve or decline is your job.
- Model versus deployed model. The model is the artifact in the registry. A deployed model is that artifact running behind an endpoint. One model can be deployed more than once, which is how traffic is split between versions.
- The machine type behind an endpoint is what costs money. It runs from deploy until undeploy, whether or not anyone calls it.
- AutoML is a black box. You get metrics, feature importance and a deployable model. You do not get the architecture or the settings. Where you must explain each decision, as in regulated lending, a simple model in BigQuery ML or custom training is easier to defend.
- The deployed model I called carried a training date several years old in its name. That is a live example of why data drift matters: the world has changed and the model has not.

**Things that tripped me up**

- Training took about an hour and did not finish in the time I had. The evaluate and deploy steps can be read through without your own model, and the prediction runs against a model that is already deployed. Do not wait for your own model.
- The written steps say to tick a checkbox to exclude the ID column. It is now a minus-circle icon at the far right of the row, and it flips to a plus to add the column back. The "total feature columns" counter includes the target, so ignore it.
- Creating the empty dataset seemed to hang for about two minutes with the Create button spinning. The dataset had in fact been created. Check the Datasets list before submitting again.
- The written steps describe their own confusion matrix backwards. Read the matrix, not the sentence.
- Everything runs in Cloud Shell. There is no VM in this exercise.

## Lessons that apply to every exercise

- Assume the written steps are a little stale. When a button is missing, look for the modern equivalent. A Cloud Shell command often does the same job.
- When time is limited, do the main steps first and explore afterward. I lost one exercise to the clock by doing it the other way around.
- Start slow resources first. A database instance can take 5 to 10 minutes to build.
- Close every incognito window between exercises. A leftover account from the last one breaks the next sign-in. Check the avatar before you connect any data.
- When a Create button seems stuck, check the list before you submit again. The resource may already exist.
- "Created" does not mean "working". Look for the proof: a running worker, a row count, a marker on the table.
- No error does not mean a correct load. Check row counts after every load.
- Searching for a product name in the console's main search bar can land on the wrong product page. I hit this twice. Search inside the tool you are working in.
- When the written steps and the real output disagree, trust the output. Sample files get updated, regions change between runs, and the text sometimes misreads its own example.
- Stay inside the plan. Bigger machines or extra products can get a practice project shut down or the practice account blocked.
- Stop what bills. A streaming job never ends by itself, and a deployed model runs until you undeploy it.
- The optional steps are often the real lesson. Do them.
- Watch for the service account pattern: create a service account, give it roles, and it does the work. I saw it in three exercises in a row, and it is worth recognizing on sight.
