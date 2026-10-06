# Open Questions

Questions I asked while learning data work on Google Cloud, and the answers reached. Mostly resolved. Anything still open is marked **OPEN**.

---

## Why does Google have so many database products? Why divide at all?

Three axes force the split, no single engine wins on all of them:

1. **Data shape**: rows/columns (SQL) vs documents (Firestore) vs wide-column key-value (Bigtable).
2. **Workload pattern**: transactional (many small fast reads/writes) vs analytical (few huge scans/aggregations). An engine tuned for millisecond single-row writes (Bigtable) is bad at scanning petabytes; one tuned for scanning petabytes (BigQuery) is bad at millisecond writes.
3. **Scale/consistency tradeoff**: Cloud SQL is simple but regional; Spanner adds global scale with strong consistency at higher complexity.

The decision tree isn't product padding, each characteristic eliminates options until one fits.

---

## For someone with a BI background, is knowing BigQuery enough?

No. BigQuery covers the store + query end only. Real data work covers the whole pipeline. Gaps a pure-BI background usually has:

- **Ingest**: Pub/Sub (streaming), Cloud Storage (batch landing).
- **Transform tooling**: Dataflow, Dataproc, Cloud Data Fusion (not just BigQuery SQL).
- **Orchestration**: Cloud Composer (Airflow).
- **Storage choice judgment**: when Bigtable beats BigQuery, when Spanner beats Cloud SQL.
- **IAM**: dataset/table/column-level access, which BI tools usually abstract away.

BigQuery SQL transfers directly and is the biggest chunk, but real work needs pipeline-level reasoning.

---

## Is SQL enough for a data warehouse but not a data lake?

Yes, that read is correct.

- **Warehouse (BigQuery)**: schema already applied, someone else did the transform. SQL is sufficient.
- **Lake (Cloud Storage)**: raw native formats (JSON blobs, images, audio, PDFs, unschema'd CSVs). SQL can't touch most of it. Feature engineering, model training, and exploration on raw data need Python/ML tooling (pandas, TensorFlow, Vertex AI, Dataproc/Spark).

This is also why the lake's "Dependencies" row lists discovery/governance/metadata tooling, with no schema, you need extra tooling just to know what's in there.

Nuance: BigQuery isn't purely structured, object tables blur the line (see below).

---

## What are BigQuery object tables, concretely?

**Example, e-commerce vendor photo quality control.** Vendors upload product photos to a Cloud Storage bucket, named `sku_48213_1.jpg`. Too many to review by hand.

1. **Cloud Storage** holds the raw images (unstructured, lake).
2. **BigQuery object table** points at the bucket; BigQuery **auto-generates one row per image** (uri, size, content type, metadata, not pixels). You don't create the keys.
3. **BigQuery ML / Vertex AI remote model** runs image classification over those rows via `ML.PREDICT`, outputting labels (`blurry: 0.92`, `watermark_detected: true`) as structured columns.
4. **Join to the structured `products` table** on a SKU extracted from the filename.
5. **Result**: a queryable "photo quality" column; QA filters for bad SKUs and emails vendors before listings go live.

BigQuery becomes the catalog/pointer layer for lake files, not an image processor. Heavy unstructured work still happens elsewhere.

---

## A ticketing system API field that should never be null comes back null. Feasible? How would a data engineer fix it?

Feasible, but "buggy API" is usually the wrong diagnosis. Two very different problems look identical downstream:

**Source data quality problem**: the field really is empty in the ticketing system:
- The "required" rule was added later; old records never backfilled.
- The field is mandatory only on certain forms/workflow states, not globally.
- The record was created via import/script/integration that bypassed UI validation (mandatory rules often fire only on the form, not on backend writes).

**Integration/transport problem**: value exists but the API returns null:
- Field-level ACL blocks that field for the API service account.
- It's a reference field that has to be followed to the linked record; you're reading the raw internal id.
- An async business rule populates it after you read the record (race condition).

**How to tell them apart:** open the record in the ticketing system's UI. Populated in UI but null in the pull = transport bug. Empty in UI too = real data quality issue.

**What the engineer does, per case:**
- **Transport bug** → engineer fixes it: grant field read access, follow the reference, or delay/re-trigger the pull after the async rule completes.
- **Real source issue** → *not* patched in the pipeline. Silently defaulting or dropping hides the problem. Route those rows to a quarantine/exception output and report back to the source system's process owner.

This is exactly what the **Raw zone** is for, validation catches it before anything reaches the trusted Curated zone. A null reaching a Curated reporting table is the failure mode.

---

## For dashboards at massive scale (YouTube-sized), create tables or query on demand?

**Create tables.** The cost math decides it, BigQuery bills by bytes scanned:

- **Query on demand:** dashboard scans 1 PB. 50 analysts × 10 loads/day = 500 PB scanned daily. Financially unviable, minutes per load.
- **Pre-aggregated:** one scheduled job scans 1 PB once at 2am, writes a 200 MB summary. Dashboards scan 200 MB, near-free, instant.

Shape:
```
raw watch events (trillions of rows)
   ↓ nightly scheduled job
daily_video_metrics (one row per video per day)
   ↓
dashboards
```

Plus **incremental processing**: append yesterday's partition only. Re-aggregating five years of history nightly is the rookie mistake.

---

## A website's live clicks and logs into BigQuery, is that Datastream?

**No, that's Pub/Sub.** The trigger is the distinction:

- **A row changed in a database** → Datastream.
- **An app emitted an event** → Pub/Sub.

Clicks, page views and app logs never touch a database; the web server just emits them. Path: `site → Pub/Sub → Dataflow → BigQuery`.

Same website, both tools, different data:

| Data | Lives where | Tool |
|---|---|---|
| Someone clicked "Add to cart" | Nowhere, emitted | Pub/Sub |
| The order row now says `shipped` | Postgres | Datastream |

**A real Datastream example on that same site:** orders live in Cloud SQL Postgres. Finance wants a near-live revenue dashboard, but nobody will let a BI tool run `SELECT *` against production every 15 minutes. Datastream tails the Postgres log and mirrors `orders` + `order_lines` into BigQuery. Checkout stays fast, BigQuery is ~10 seconds behind.

**Is Datastream a database?** No, a managed *service*, a pipe. It stores nothing.

**Four source databases?** One **stream per source**, all landing in the same BigQuery dataset. Unified types are what make the four feeds compatible on arrival. Format control is limited to *what* to replicate and the landing format; real reshaping is Dataflow's job.

---

## Why would four RDBMS ever land in one BigQuery?

**Acquisitions.** A company buys three companies over five years; each already runs its own ERP, SQL Server, Oracle, MySQL, PostgreSQL. Nobody is migrating all four onto one system (three-year project, each business keeps running on what it has). But the CFO wants **one revenue number for the group**, daily.

4 streams → 1 BigQuery dataset → one query unions the four `orders` tables → one dashboard. The alternative was four reports finance adds up in Excel.

---

## Clean each source first, or append first then clean?

**Clean first, then append.** Raw sources can't be appended, different column names, different currencies; the union would fail or lie.

But the per-source code is **thin**: it only conforms to a common shape (rename, cast, convert). All actual business logic (margin, YoY, dedupe) is written **once**, after the union.

**The rule: per-source code does only what's different about that source. Anything common gets written once, downstream.** Same instinct as Power Query, conform in the queries, model once.

You do *not* run four pipelines. Four **pipes**, then one process:

```
SQL Server ─┐
Oracle     ─┤→ raw.orders_hq / _a / _b / _c ─→ stg.* (conform) ─→ curated.orders_all ─→ dashboard
MySQL      ─┤                                        ↑
PostgreSQL ─┘                                   ref.fx_rates
```

**Worked example (2 sources, 5 tables)**: `raw.orders_hq` (SQL Server, CAD) and `raw.orders_a` (Oracle, EUR, different column names), plus `ref.fx_rates` from a bank feed. Staging views rename both to a common shape and convert EUR using **that order date's** rate (join on `rate_date = order_dt`, not today's rate, otherwise last year's revenue changes every morning). Then `UNION ALL` into `curated.orders_all` with a `company` column added, since without it you lose track of which source a row came from.

FX rates do **not** live in an orders table, they're a separate reference/lookup dataset, effectively a fifth source.

---

## What is Apache Spark, and what does a data scientist do with it?

**Spark** = an engine for processing huge datasets with **code** (Python or Scala) instead of SQL, splitting the work across many machines. **PySpark** is the Python interface. **Dataproc** is Google running Spark for you.

The split: SQL answers questions you can express as SQL; Spark handles what SQL can't, ML, parsing messy text, custom row logic, images/audio. Analysts write SQL, data scientists write Spark.

**Why it comes up with BigLake:** the same Parquet files in Cloud Storage get read by *both* engines, sharing the same metadata cache.

```
Parquet files in Cloud Storage
   ├── BigQuery  ← analysts, SQL
   └── Spark     ← data scientists, Python
```

One copy, two tools. (A spark-bigquery connector also lets Spark read BigQuery tables directly.)

---

## How does a model actually predict? (logistic regression, worked by hand)

**Training data, 5 customers whose outcome you already know:**

| customer | orders_count | days_since_last | avg_amount | **churned** |
|---|---|---|---|---|
| A | 12 | 5 | 340 | 0 |
| B | 2 | 200 | 60 | **1** |
| C | 9 | 20 | 280 | 0 |
| D | 1 | 310 | 45 | **1** |
| E | 15 | 8 | 410 | 0 |

**`churned` is the answer key**, and it comes from your own history, look back a year, mark who left. No history, no model. In code that's `labelCol="churned"`.

**`VectorAssembler`** just packs the input columns into one column of numbers (`A → [12, 5, 340]`). Plumbing, not cleverness, Spark's ML functions want one column, not three.

**`.fit()`** searches for weights that make this match the known answers:

```
z = w0 + w1×orders + w2×days_since_last + w3×avg_amount
```

Say it lands on `z = -2.0 + (-0.15 × orders) + (0.03 × days) + (-0.004 × avg_amount)`. **The signs are the learned knowledge:** more orders → less churn; longer quiet → more churn; bigger spender → less churn.

Then squash into a probability: **`p = 1 / (1 + e^-z)`**

Check against B (who did churn): `z = -2.0 - 0.3 + 6.0 - 0.24 = 3.46` → `p = 0.97` OK
Check against A (who stayed): `z = -2.0 - 1.8 + 0.15 - 1.36 = -5.01` → `p = 0.007` OK
New customer F (3 orders, 150 days quiet, $80 avg): `z = 1.73` → `p = 0.85`: 85% likely to churn.

Nothing in the data says F will churn. The formula, built from A to E, says F *looks like* B and D.

**Output columns:** `probability` = `[stay, churn]`, `prediction` = the 0/1 call at a 0.5 cutoff.

**In one line:** SQL tells you *who left*; the model turns that history into a formula and applies it to people who haven't left yet.

### The sigmoid, and why it exists

`z` can be any number (`-5.01`, `3.46`, `+900`); a probability must sit in 0 to 1. `p = 1/(1+e^-z)` squashes it:

| z | -5 | -1 | **0** | 1 | 3.46 | 900 |
|---|---|---|---|---|---|---|
| p | 0.007 | 0.27 | **0.5** | 0.73 | 0.97 | ~1.0 |

Never quite reaches 0 or 1, it flattens. **Names:** the `1/(1+e^-z)` part is the **sigmoid** (or logistic function, S-shaped); `z` is the **linear combination**; the whole thing is **logistic regression**.

### Is it always linear? No, the model families

The `w1×x1 + w2×x2 + ...` shape is what makes it a *linear* model: each input pushes the answer by a fixed amount, independently. It cannot learn "high spend matters only for new customers."

| Family | Examples | Shape it learns |
|---|---|---|
| **Linear** | linear regression, logistic regression | Straight-line effects |
| **Trees** | decision tree, random forest, **gradient boosting / XGBoost** | Nested if/then splits |
| **Neural networks** | deep learning | Arbitrary curves |
| **Instance-based** | k-nearest neighbours | "Who does this resemble?" |
| **Kernel** | SVM | Curved boundaries |

**On tabular business data, gradient boosted trees usually win.** Neural networks dominate images, audio and text, not spreadsheets. Logistic regression stays popular because you can *read* it, those weights say "more orders → less churn" in plain sight. XGBoost is more accurate and much harder to defend to a CFO.

---

## Gradient boosted trees, worked by hand (support ticket example)

**Problem:** predict how many hours a new ticket will take to resolve, at the moment it arrives, to staff the week and flag likely SLA breaches.

| ticket | priority | reassignments | attachments | **actual_hours** |
|---|---|---|---|---|
| T1 | 1 | 3 | 2 | 40 |
| T2 | 4 | 0 | 0 | 4 |
| T3 | 2 | 1 | 1 | 20 |
| T4 | 4 | 1 | 0 | 8 |
| T5 | 1 | 2 | 3 | 32 |

```python
from pyspark.ml.regression import GBTRegressor
model = GBTRegressor(labelCol="actual_hours", maxIter=100, stepSize=0.1).fit(features)
```

**The mechanism: each tree fixes the mistakes of the ones before it.**

**Step 0, start dumb.** Predict the average, 20.8, for everyone.
Errors (`actual − prediction`): T1 +19.2, T2 −16.8, T3 −0.8, T4 −12.8, T5 +11.2. Total 60.8.

**Step 1, build a tree that predicts the *error*, not the hours.**
```
priority <= 2 ?
├── yes → T1, T3, T5   errors +19.2, −0.8, +11.2  →  average +9.9
└── no  → T2, T4       errors −16.8, −12.8        →  average −14.8
```
Add the leaf value scaled by the learning rate (0.5 here so the numbers move visibly; real jobs use 0.1):
T1/T3/T5 → `20.8 + 0.5×9.9` = **25.7**; T2/T4 → `20.8 + 0.5×(−14.8)` = **13.4**

**Recompute the errors**: this is just `actual − new prediction`:

| ticket | actual | prediction | error | (was) |
|---|---|---|---|---|
| T1 | 40 | 25.7 | +14.3 | +19.2 |
| T2 | 4 | 13.4 | −9.4 | −16.8 |
| T3 | 20 | 25.7 | −5.7 | −0.8 |
| T4 | 8 | 13.4 | −5.4 | −12.8 |
| T5 | 32 | 25.7 | +6.3 | +11.2 |

Positive = predicted too low, negative = too high. Four improved; **T3 got worse**: it was lumped in with T1/T5 by the priority split. Fixing that is the next tree's job.

**Step 2, a tree on those new errors.**
```
reassignments >= 2 ?
├── yes → T1, T5      (14.3 + 6.3) ÷ 2 = +10.3    → ×0.5 = +5.2
└── no  → T2, T3, T4  (−9.4 −5.7 −5.4) ÷ 3 = −6.8 → ×0.5 = −3.4
```

| ticket | current | leaf | new prediction | actual | error |
|---|---|---|---|---|---|
| T1 | 25.7 | +5.2 | **30.9** | 40 | +9.1 |
| T2 | 13.4 | −3.4 | **10.0** | 4 | −6.0 |
| T3 | 25.7 | −3.4 | **22.3** | 20 | −2.3 |
| T4 | 13.4 | −3.4 | **10.0** | 8 | −2.0 |
| T5 | 25.7 | +5.2 | **30.9** | 32 | +1.1 |

Total error: **60.8 → 41.1 → 20.5**. T3 recovered (`−0.8 → −5.7 → −2.3`), tree 1 hurt it, tree 2 caught it. That is the self-correcting part.

**The loop, in three lines:**
1. Look at how wrong you are (`actual − prediction`)
2. Build a small tree that predicts *that wrongness*
3. Add a fraction of it to your prediction, then go back to 1

**"Gradient boosting" decoded:** *boosting* = many weak models stacked; *gradient* = each aims at the leftover error. **The learning rate matters:** at 1.0 each tree overcorrects and the model memorizes; small steps × many trees generalizes.

**Why it beats logistic regression here:** step 2's split happens *within* step 1's split, it learned "reassignments matter, especially on low-priority tickets." A linear model can't express that; every input acts independently.

**Output**: predicted hours per open ticket, plus a feature-importance ranking (not readable weights):
```
reassignments  0.51
priority       0.34
attachments    0.15
```

**What it means on a Monday:** *Triage*, a ticket predicted at 44h against a 24h SLA gets escalated at 9am, not at hour 23. *Capacity*, 60 open tickets summing to 380 predicted hours against 150 available is a number you can put in front of a manager before anything breaches. *The unasked-for finding*, `reassignments` outranking `priority` says tickets are slow because they bounce between teams, not because they're hard. That's a routing problem, and the dashboard would never have surfaced it.

**The catch:** you cannot explain *why* one ticket got 44.2, it's 100 stacked trees. Permanent trade: accuracy vs defending the number in a meeting.

---

## As a PM / product owner, how do I tell if a model is any good?

**Real column counts:** hundreds to thousands, not 3. **Nobody reads the rows**: not even the data scientist. What gets looked at is summary statistics per column, distribution plots, missing-value counts, correlation to the target. Twenty numbers per column instead of 40 million rows.

You don't audit the code. **You ask five questions.**

### 1. "What did you hold out?"
The model must be scored on rows it never saw in training (train on 80%, test on 20%). Ask for **both** numbers:

| | Training | Test | Verdict |
|---|---|---|---|
| Healthy | 82% | 79% | Fine |
| **Overfit** | **99%** | **61%** | It memorized |

**The gap between train and test is the overfitting tell.** A single impressive number quoted alone is usually the training score, which is meaningless. (The 5-ticket example above with 100 trees would be catastrophically overfit.)

### 2. "What's the baseline?"
The dumbest rule, scored the same way, "always predict the average", or "always predict the majority class". If the model barely beats it, it's a science project.

**The trap this catches:** fraud detection at 99.5% accuracy sounds excellent until you notice 99.5% of transactions aren't fraud, so "always say no fraud" scores identically. Almost nobody asks this question.

### 3. "Would we actually have this column at prediction time?"
This is **data leakage**: the most common way a model looks brilliant in testing and dies in production. Concrete: someone adds `resolution_notes_length` to the ticket model and accuracy jumps to 97%, but resolution notes only exist *after* resolution, so at 9am the column is empty. The model was reading the answer.
**Ask it as:** *"Walk me through a brand-new ticket at 9am. Which of these columns is actually populated?"*

### 4. "How did you split, randomly, or by date?"
With time-based data, a random split trains on December and tests on November: predicting the past using the future. The honest version trains on Jan to Sep and tests on Oct to Dec. Scores drop, and that drop is real.

### 5. "Does the feature importance list make sense to you?"
Read what the model thinks matters, not the model. `reassignments` at the top matching your process instinct = good. `ticket_id` at the top = something is badly wrong. **This is where domain knowledge beats technical knowledge**: a PM is better placed to catch it than the data scientist.

### Vocabulary

| Term | Means |
|---|---|
| **Overfitting** | Memorized the training data, fails on new data |
| **Holdout / test set** | Rows kept aside to score honestly |
| **Cross-validation** | Do the split five different ways and average, a more robust #1 |
| **Baseline** | The dumb rule you must beat |
| **Leakage** | A column that secretly contains the answer |
| **Feature** | An input column |
| **Label / target** | The answer column |

**The one-line version:** *"What's the test score, what's the baseline, and would we have every one of those columns at 9am on a new ticket?"* Three questions, no code.

---

## ETL vs ELT, is there a real difference?

Yes, and **Power Query is ETL**: it transforms before the data lands in the model.

**Same task, two shapes.** 500M help desk incidents → a department summary:

```
ETL:  source → [a machine that cleans/aggregates] → warehouse gets 340 clean rows
ELT:  source → warehouse gets all 500M raw rows → SQL cleans → 340 rows
```

| | ETL | ELT |
|---|---|---|
| Who does the compute | A machine you own and size | The warehouse |
| What lands | Only clean data | Everything, raw |
| Bug in your logic | **Re-extract from source** | Re-run SQL on data you already have |
| Scale limit | That machine | The warehouse (huge) |

**The replay point is the practical one.** Discover the department mapping was wrong six months ago: with ELT the raw rows are still there, so fix the SQL and re-run. With ETL you must pull two years of history from the source again, if it still exists. Same argument as the **landing zone** rule: never clean data on the way in.

**Why ELT won:** ETL made sense when warehouse compute was expensive and fixed-size, so you kept junk out. BigQuery is serverless and elastic, so it is cheaper to dump everything in and let it do the work.

**Neither is dead. ELT is the default; ETL is the exception.** Use ETL when:
- **Compliance**: sensitive fields must be masked or dropped *before* they land
- **The target has a size limit**: the Power BI Pro 1 GB model cap
- **Volume is not worth storing raw**: billions of events where only the daily rollup matters
- **The target is not a warehouse**: feeding an app or an API

**Personal split:** anything going to BigQuery → ELT. Anything going straight into a Power BI model → ETL, because the cap forces it.

**When to move a transform upstream out of Power Query**: row count is *not* the only trigger:
- refresh creeping toward an hour (this can happen at far below 500M rows)
- the same transform copied into a second report
- someone outside Power BI needs the same data

Shape when it moves: `source → warehouse (raw) → SQL transform → small curated table → Power BI`. Power BI reads the small table, refresh in seconds.

## Dataprep vs Dataform: related?

**No. Two separate products that only share the "Data" prefix.** Neither is used inside the other.

| | **Dataprep** | **Dataform** |
|---|---|---|
| What you do | Click through a messy dataset in a visual grid, fix it step by step | Write `.sqlx` files in a git repo |
| Saved as | A **recipe** (list of wrangling steps) | SQL code with `${ref()}` dependencies |
| Runs where | On **Dataflow**, behind the scenes | Inside **BigQuery** |
| Pattern | ETL (cleans before landing) | ELT (transforms after landing) |
| Velocity | **Batch only** | Batch (scheduled SQL) |
| Who it is for | Analyst, no code | Data engineer or analyst who writes SQL |
| Power BI analogy | **Power Query editor**: click "remove nulls", "split column", each click becomes an Applied Step | A chain of SQL views that reference each other |

**Scenario:** a vendor sends a CSV with dates in three formats and a "Name" column that mixes "Smith, John" and "John Smith". Someone who does not write code opens it in Dataprep, fixes it by clicking, saves the recipe, and runs it on each new file → **Dataprep**. The clean data is already in BigQuery and you need a daily summary table built from it, plus a table built from that summary → **Dataform**.

**Trigger words:** "recipes", "wrangling", "no-code", "visual" + one dataset → Dataprep. "SQL workflow", "dependencies", "assertions", "inside BigQuery" → Dataform.

## What is Cloud Data Fusion?

A **visual, drag-and-drop pipeline builder**. You place boxes on a canvas: sources (databases, SaaS apps, files, other clouds), transforms, sinks (BigQuery, Cloud Storage). Many connectors come ready-made as plugins.

- **Built on CDAP** (open source): **Cask Data Application Platform**, made by Cask Data, which Google acquired in 2018. It sat on top of Hadoop so developers didn't have to write raw Hadoop code.
- **Runs the pipeline on Dataproc** (Spark) clusters it spins up for the run. You draw, it generates the Spark job.
- **Not serverless**: you create and pay for a Data Fusion *instance* that hosts the designer.
- Strength: **hybrid and multicloud** integration, many systems wired together.

**Analogy:** closest to Azure Data Factory or SSIS. Power Automate's canvas is the right *feel*, but for bulk data rather than one record at a time.

**vs Dataprep:** both visual. Dataprep cleans **one dataset** with a recipe. Data Fusion **connects many systems** in a pipeline.

## Which tools are managed open source?

None are *just* a rename. The pattern is: **the open source project is the engine, Google runs the servers, patching and scaling for you.**

| Google service | Open source underneath | Relationship |
|---|---|---|
| **Dataproc** | Apache Hadoop, Spark | Hosted clusters running the real thing. Closest to "same software, managed" |
| **Cloud Composer** | Apache Airflow | Hosted Airflow. Your DAG files are plain Airflow Python |
| **Cloud Data Fusion** | CDAP | Managed CDAP with a Google UI and plugins |
| **Dataflow** | Apache Beam | Reverse direction: Google built it, then open sourced the code model as Beam. Beam is the SDK you write; Dataflow is one place to run it (Spark and Flink can also run Beam) |
| **Dataform** | Dataform core | Google acquired the company; the SQLX compiler is open source, the managed service is Google's |
| **Dataprep** | none | A partner product (Trifacta, now Alteryx), not open source |
| **BigQuery, Pub/Sub, Bigtable, Datastream** | none | Google proprietary |

**Why it matters: the "existing code" situations.**
- "Existing Spark / Hadoop jobs, move with minimal changes" → **Dataproc**
- "Existing Airflow DAGs" → **Cloud Composer**
- "Portable pipeline, avoid lock-in, batch + streaming" → **Dataflow** (Beam)
- "Existing CDAP pipelines", hybrid → **Data Fusion**

## When do you use Cloud SQL, and when BigQuery?

**Cloud SQL = the cash register. BigQuery = the accountant.**

- **Cloud SQL:** an app saves or changes one record at a time. A customer places an order (insert a row, stock −1); an employee submits leave and the manager approves (update one row); a user unlocks a bike (save one trip).
- **BigQuery:** people ask questions across many rows. "Total sales by region last year", "which department takes the most sick days", "busiest station across 83M trips".
- **Real systems have both:** app writes to Cloud SQL → copied over (for example with Datastream) → BigQuery → dashboard.
- Power BI / Power Platform framing: Cloud SQL is the SQL Server or SharePoint list behind a Power App; BigQuery is the Power BI dataset. Cloud SQL is like Azure SQL: you pay for the instance while it runs, even idle.

## The bike trips exercise counted trips per station. What would a real company do with that?

The exercise has no business goal, it is a SQL tour. A real bike-share operator would use it for:

1. **Rebalancing** (the big one). More starts than ends at a station → it runs out of bikes; more ends → docks fill up. Trucks move bikes daily. A train station empties every morning and overflows every evening.
2. **Where to add docks:** busiest stations get bigger ones.
3. **Maintenance:** busiest stations wear out first.

That is why starts and ends belong **side by side** (JOIN: `station | starts | ends | gap`), not stacked (the exercise's UNION, which mixes both with no label).

## Two ways to get a website file into BigQuery. Same outcome?

Yes, same table. Different route:

| | Via bucket | Via Cloud Shell disk |
|---|---|---|
| Steps | download → upload to bucket → create table from bucket | `wget` + `unzip` on Cloud Shell → `bq load` |
| File afterwards | kept | gone when the VM / practice project ends |
| Size | practically unlimited | 5 GB |
| Automate on a schedule | yes | no, typed by hand |
| Use for | real pipelines | quick one-off tests |

In real work files land in a **bucket first** (data lake) and load from there into BigQuery (warehouse). Cloud Shell = a borrowed library laptop: fine for doing work, never for keeping files.

## Is "GROUP BY all 32 columns HAVING COUNT > 1" how duplicates are found in real life?

The idea yes, the form usually not. **Still a bit fuzzy, revisit.**

- **Group on the business key, not every column.** Real duplicates often differ in an unimportant column (a `loaded_at` timestamp) and slip past an all-columns check. Decide what should be unique (`order_id`, or visitor + visit + time + product + action) and group on that.
- **Key check is stronger than all-columns.** If the key has no duplicates, all columns can't either (more columns only split piles). Not the reverse: same click with a different city = 0 duplicates by all columns, 1 by the key.
- **Shortcuts:** any exact duplicates? compare `COUNT(*)` with `COUNT(DISTINCT TO_JSON_STRING(t))`. Remove: `SELECT DISTINCT *`. Keep latest per key: `QUALIFY ROW_NUMBER() OVER (PARTITION BY key ORDER BY loaded_at DESC) = 1`.
- **Automate it:** pipelines run the check on every load, e.g. a Dataform `uniqueKey` assertion that fails the run.

One line to keep: **group by the columns that should be unique; HAVING > 1 shows what isn't.**

## Interview: "How do you know this data isn't duplicated, and that it's real?" (PM voice)

### The framework: 5 questions, in order

1. **What is one row?** (the grain) One click? One order? One order *line*? Nothing else makes sense until this is answered. In the ecommerce exercise one row was one hit, so 21.5M rows ≠ 21.5M visits.
2. **What should make a row unique?** (the key) Ask the owner or the docs. Orders → `order_id`. Order lines → `order_id + line_number`. GA hits → visitor + visit + time + product + action. Watch for columns that are *not* identity: `loaded_at`, `city`, descriptions.
3. **Is the key actually unique?** Group by the key, `HAVING COUNT(*) > 1`. Sort the result by count descending (worst offenders first). Then pull a few offenders and sort **by key, then timestamp**, so copies sit next to each other and you can see *how* they differ.
4. **Why are they duplicated?** The cause decides the fix. A pipeline retried and loaded twice (drop exact copies). The record was updated and each version was kept (keep the latest per key). A join fanned out (one order × three shipments = three rows; fix the join, don't delete). Genuinely repeated events (two identical mug purchases; not a duplicate at all).
5. **Is it real?** (plausibility) Duplicates are one failure; wrong-but-unique rows are the other.

### "Is it real?" checks, with examples from the exercises

| Check | Question | Example |
|---|---|---|
| Ranges | Are values physically possible? | Negative quantity, a date in the future, a 90-hour bike ride |
| Placeholders | Is one exact value suspiciously common? | Top 10 natality babies all 18.0007436923 lb = a cap, not ten real babies |
| Test data | Rows nobody real created? | "test destination" inserted in the bike trips exercise; users named test@, qa_ |
| Outliers | One entity doing far too much? | One visitor with 50,000 hits in a day = a bot, not a customer |
| Nulls | Blank where it shouldn't be? | GA "not available in demo dataset"; state null on births |
| Row counts | Did the load keep everything, and nothing extra? | 955 rows for 954 stations = header loaded as data |
| Reconciliation | Does it match the system of record? | Warehouse revenue vs the finance report for the same month |
| Units | Is the scale what you think? | GA revenue stored × 1,000,000 (304,320,000 = $304.32) |

**Reconciliation is the one that convinces stakeholders.** "Our March revenue in BigQuery is within 0.5% of Finance's close" beats any SQL.

### Worked example: online store orders

The dashboard says March revenue is $1.2M. Finance says $1.0M.

1. **Grain:** the `sales` table is one row per *order line*.
2. **Key:** `order_id + line_number`.
3. **Check:** 4,000 key groups have 2 rows. Sorted by key then `loaded_at`: identical lines, loaded 10 minutes apart on March 14.
4. **Cause:** the nightly load failed halfway, was re-run, and appended instead of replacing. Fix: dedupe to the latest load per key, and make the load replace that day's data (idempotent).
5. **Real:** after dedupe, revenue is $1.02M. The remaining gap: 30 orders to `@company.com` test accounts ($15K) and refunds recorded a month late. Exclude test accounts, agree the refund timing rule with Finance. Now it reconciles.

### How to say it in an interview (PM who understands data, not a data engineer)

> "Before I trust a number I ask three things. First, what's one row, and what should make it unique? That's usually a conversation with whoever owns the source. Second, does the data actually respect that? The team checks the key for duplicates, and if there are some, we find out why, because a retried load, a kept update history and a bad join each need a different fix. Third, is it plausible and does it reconcile? Ranges, test accounts, bots, and above all: does our total match the system of record, like Finance's revenue? And I want those checks automated in the pipeline, so we hear about a problem before a stakeholder does."

**What makes this sound experienced, not scripted:** you name the *grain* first, you ask *why* duplicates exist instead of just deleting them, and you anchor trust in *reconciliation with the business*, not in the query. You don't need to recite SQL; mentioning "group by the key, HAVING count above one" once is enough.

## What is this exercise setup called in real life, if I were a data engineer?

All of it together is a **cloud data platform**. Building and running one on Google Cloud is the GCP data engineer job.

| Exercise piece | Real-world name |
|---|---|
| Temporary practice project (auto-generated project id) | a sandbox / dev environment, safe to break |
| `data-to-insights` project | the data warehouse, often a separate prod project the team reads from |
| BigQuery Studio query editor | the SQL workspace / IDE |
| Fixing the analyst's queries | query review / debugging |
| Dedup and row counts | data profiling / data quality checks |
| One-off questions like "top cities" | ad hoc analysis |

A company typically runs **dev → test → prod** projects: write and fix in dev, read prod data, promote finished pipelines to prod.

The layers, with the Microsoft stack I already know:

| Layer | Job | GCP | Microsoft |
|---|---|---|---|
| Access | who can do what | IAM | Entra ID + RBAC |
| Storage (raw files) | data lake | Cloud Storage | Data Lake / Blob |
| Ingestion | bring data in | Pub/Sub, Datastream, Data Transfer Service | Event Hubs, Data Factory |
| Processing | clean, join, transform | Dataflow, Dataproc (Spark), Dataform | Data Factory, Databricks, dataflows |
| Warehouse | tables + SQL | BigQuery | Synapse / Fabric Warehouse |
| Orchestration | schedule and chain jobs | Cloud Composer (Airflow) | Data Factory pipelines, Power Automate |
| BI | dashboards | Data Studio, Looker | Power BI |
| Tooling | command line | Cloud Shell + `bq`, `gcloud` | Azure CLI, PowerShell, `pac` |

Real-life shape: data lands in Cloud Storage, Dataflow cleans it into BigQuery, Composer runs that on a schedule, Data Studio reads the final tables. The SQL exercises sit in the warehouse layer, the reporting exercise in BI.

## How do I know which columns to GROUP BY?

Say the question out loud. The nouns after **"per"** or **"each"** are the GROUP BY columns.

| Question | GROUP BY |
|---|---|
| Visitors per channel | `channelGrouping` |
| Views per product | `v2ProductName` |
| Each person once per product | `fullVisitorId, v2ProductName` |
| Sales per country per month | `country, month` |

Check: finish the sentence "one row in my result = one ___". Then every other column in SELECT must be wrapped in an aggregate.

---

## What is PI and PII in security?

**PII**: Personally Identifiable Information: identifies a specific person, alone or combined. SIN/SSN, passport, email, phone, employee ID, address, IP, a photo.
**PI**: Personal Information: the broader term, *any* data relating to an identifiable person even if it doesn't identify them alone. Job title, salary, door-swipe times. All PII is PI; not all PI is PII.

Which word you use is a legal accident: GDPR and PIPEDA say *personal data / personal information*, US/NIST says *PII*. Used interchangeably in practice.

**The part that matters:** PII is not a fixed list of columns, it's whether the *combination* identifies someone. A row of `department | office | login_hour | events` has no name, but if one small office has one IT person who swipes in at 08:00, that row **is** a named individual. Those are **quasi-identifiers**. "We dropped the name column" is not anonymization.

Related: **PHI** (health, HIPAA), **SPI / sensitive PII** (health, biometrics, race, religion, union membership, sexuality). On Google Cloud, **Sensitive Data Protection** (formerly Cloud DLP) scans and classifies columns by infoType; Knowledge Catalog is where you tag the asset and attach policy.

---

## What is a write-ahead log (WAL)?

The database writes down what it is *about to do*, before doing it.

`UPDATE employees SET salary = 95000 WHERE id = 42` →
(1) write to the log *"about to change id 42 from 88000 to 95000"*, flushed to disk;
(2) tell the client "done";
(3) update the real data file later, lazily, in a batch.

Power dies between 2 and 3 → on restart the DB replays the log. Nothing lost. Without the log, step 2 would have to wait for a slow random disk write on every statement. One fast sequential append makes the database **faster and safer at once**.

**Why data engineers care:** that log is accidentally a perfect, ordered record of every change with before/after values. CDC tools just read it. The database was writing it anyway, which is why Datastream costs the source almost nothing.

---

## The four log mechanisms, are they different SQL languages?

No. They're four competing **database products**, like Chrome vs Firefox vs Safari. All store tables and speak SQL. Each vendor named its change log something different.

| Product | Its log | What you actually read |
|---|---|---|
| Oracle | **LogMiner** | a queryable view giving reconstructed SQL text (`SQL_REDO` / `SQL_UNDO`) |
| MySQL | **binary log** | a binary file via `mysqlbinlog`; before/after images, positional columns (`@1`, `@3`) |
| PostgreSQL | **logical decoding** | a decoder plugin's output stream, the WAL is *physical* ("page 17 bytes changed") and useless raw |
| SQL Server | **transaction logs** | shadow tables (`cdc.hr_employees_CT`) you query with plain SQL |

Setup burden: Postgres highest (`wal_level=logical` + a replication slot), SQL Server medium (enable per table), the other two low. The part worth knowing is the name-to-engine mapping.

---

## Why did the exercise need `cloudsql.logical_decoding=on`?

Because it isn't free, it makes Postgres **write more**. Default WAL is physical, sized for crash recovery only. Turning logical decoding on adds enough detail to reconstruct row-level changes, costing disk, write throughput, and a retention obligation (slots hold log entries until a reader collects them, a stopped stream can fill the disk). Most databases never need it, so it ships off.

Turn it on for: replication to a warehouse, Postgres→Postgres version upgrades with no downtime, event-driven triggers, audit trails. Leave it off for a plain app database nobody reports on. Same call as enabling change tracking on a source for incremental refresh: you only pay for it when something downstream consumes the changes.

---

## Publication vs replication slot

Think Netflix. **Publication** = which shows you're allowed to watch. **Slot** = where you paused.

| Thing | Answers | Named in the exercise |
|---|---|---|
| Publication | which tables may be copied | `test_publication` |
| Slot | how far the reader got | `test_replication` |

The names are deliberately confusing, `test_replication` is the *slot*. "Replication" on its own isn't an object you create, it's the general word for the whole activity.

---

## User vs superuser

**Superuser** bypasses all permission checks, drop any table, read any data, create users, change server settings. A regular user can only do what was explicitly granted. Same as `root` / Administrator vs a normal account.

In the Datastream exercise you are `postgres`, the superuser, which is why `ALTER USER POSTGRES WITH REPLICATION` worked with nobody granting you anything. In production you'd create a dedicated non-super user with only replication rights, so a leaked Datastream password reads the change log and nothing else.

---

## Difference between a bucket and a database

**Bucket = files. Database = rows.**

| | Bucket | Database |
|---|---|---|
| Holds | files: CSV, photos, PDFs, anything | tables with columns and rows |
| You ask it | "give me `invoice.csv`" | "which invoices are over $500?" |
| Can it filter? | no, you get the whole file | yes, that's the point |

A bucket doesn't know what's inside the files, it's a cloud hard drive. The Lakehouse exercise's whole trick is making bucket files *behave* like database rows, which is why it needs the connection robot.

---

## Is Datastream a storage thing?

No, **Datastream stores nothing.** It's a pipe. `Oracle → Datastream → BigQuery`: it reads a change, passes it along, forgets it. You always give it a destination (BigQuery or Cloud Storage) and that's where data actually sits. Turn it off and there's nothing in it to lose.

| Holds data | Moves data |
|---|---|
| Cloud Storage, BigQuery, Bigtable, Cloud SQL | Datastream, Dataflow, Storage Transfer Service, `gcloud storage` |

The name misleads, read it as the verb.

---

## Couldn't Dataflow do Datastream's job?

Not the important half. Dataflow **can** read a database, but only by **running a query** (`SELECT * FROM tickets WHERE modified > yesterday`), which is the nightly-export problem again: real load on the source, a snapshot only, and it misses anything that changed and changed back. Dataflow **cannot tap the change log**; that means speaking each vendor's private protocol, which is Datastream's entire reason to exist.

| | Datastream | Dataflow |
|---|---|---|
| How it gets data | reads the change log | asks the DB a question |
| Load on source | ~none | a real query |
| Sees every change? | yes | only current state at query time |
| Transforms? | no | yes, anything |

Common shape: `Oracle → Datastream → Dataflow (mask/enrich) → BigQuery`. Datastream gets the changes out, Dataflow decides what they look like.

---

## Is masking PII all Dataflow is for?

No, that's just the easiest use case to explain. Dataflow is managed Apache Beam: the same pipeline code runs over a bounded batch or an unbounded stream, autoscaling.

Plain-English list, all from one story, a coffee chain where every card reader sends a message on each sale:

1. **Streaming**: the dashboard updates as sales happen, instead of overnight.
2. **Windowing on time**: "cups sold in the last 5 minutes, refreshed every 30s". The stream never ends, so there's no table to query; Dataflow chops it into buckets.
3. **Enrich in flight**: the reader only sends `shop_id: 17`; Dataflow adds `city: Toronto` as it passes, so nobody joins a lookup table forever after.
4. **Masking / filtering**: strip the card number mid-flight; the stored copy never had it.
5. **Non-SQL logic**: call a fraud model per sale and add `fraud_risk: high`. SQL can't.
6. **Fan-out**: read the message once, send it to the warehouse, the points app, and the alert system.
7. **Templates**: "Pub/Sub → BigQuery" already written; fill in a form, no code.
8. **Late / out-of-order data**: a shop's internet drops 09:00 to 09:07 and seven minutes of sales arrive at once. Dataflow reads each sale's own timestamp and files it under 09:00; a simple loader would dump them all into 09:07 and the 9am number is wrong forever.

**Rule of thumb:** if the answer could be a SQL query in BigQuery, it's *not* Dataflow, that's ELT and it's cheaper. Reach for Dataflow when it's streaming, when the logic isn't expressible in SQL, or when data must change before it lands.

---

## Is Dataflow a form? Where does the code live?

Dataflow is a **processing engine**; the form is only the easy front door. Three ways in: a **template** (fill in a form, Google wrote the logic), a **template + your function**, or **your own pipeline** in Java/Python with Apache Beam.

You never type code into the form, the form holds a **pointer**. For a template UDF you write a small JavaScript function, save it as `transform.js`, **upload it to a Cloud Storage bucket**, and give the form two fields: the `gs://` path and the function name. Dataflow fetches it at runtime. For a full pipeline, the code lives in your repo; running it stages a package to a bucket and hands it to Dataflow, because the workers can't read your laptop.

**The bucket is always where code goes to be picked up**: a recurring Google Cloud idiom.

Naming trap: Google's **Dataflow** is unrelated to Power BI's **dataflow**.

---

## Is Dataform a form?

No, it's **code**. The name is just a company Google acquired. You write `.sqlx` files in a git repo (`definitions/`, `includes/`, `workflow_settings.yaml`), edit them in a web IDE, and Dataform works out the run order and executes it in BigQuery.

Three unrelated products sharing letters:

| | |
|---|---|
| **Dataform** | SQL files in git, transforms inside BigQuery |
| **Dataflow** | Beam engine, batch and streaming, real code |
| **Datastream** | CDC pipe, database → BigQuery |

---

## What is a `.sqlx` file, in plain terms?

A SQL file with a small label on top. File 1 says *"make a view"* then lists four fruits. File 2 says *"make a table"* then adds up the fruits from file 1. The label (`config`) decides view or table, you never write `CREATE OR REPLACE VIEW`, the label does it.

The one special bit: `${ref("quickstart-source")}` means *"use file 1's output"*. Writing it also tells Dataform file 1 must run first. **You never write the order.** With 40 files instead of 2, that is the entire value of the tool.

Full block list (the exercise only uses two): `config`, `js`, `pre_operations`, the body, `post_operations`.

---

## Workspace vs branch

A workspace **contains** a branch, plus what git calls the working directory.

| | |
|---|---|
| **Branch** | the saved commit history |
| **Workspace** | that branch + your uncommitted edits + the editor |

Local equivalent: `git checkout -b my-work` is the branch, the folder you type in is the working directory. Dataform merged both into one browser thing. In the Dataform exercise you never commit, so nothing reaches the branch, execution runs straight from the workspace.

---

## Why can't I just edit Dataform files in my own editor, outside the browser?

You can, in real life. A Dataform repo can be **linked to an external git repo** (GitHub, GitLab, Bitbucket, Azure DevOps): edit `.sqlx` locally, push, Dataform runs it, with proper PRs and review. What stays in the service is **compiling** (resolving every `${ref()}` into the DAG) and **executing**, which you can trigger from Cloud Scheduler or Airflow.

The exercise's built-in repo is Google-hosted with no external clone, which is why it's browser-only.

---

## Is an AI agent just "foundation model + some instructions + MCP"?

Two-thirds right.

- **Foundation model**: yes, that is the brain.
- **Instructions**: half-right. The system prompt and goal are what make it goal-oriented, but they are the *input* that steers the model, not a separate component of the architecture.
- **MCP**: half-right. **MCP (Model Context Protocol)** is one standard format for handing tools to a model, so any model can talk to any tool without custom wiring per tool. It is plumbing inside the **tools** component. An agent does not need MCP; plenty call APIs directly or use a framework's own tool format.

**What the formula misses: the orchestration layer.** The loop is
model decides → tool runs → result comes *back* to the model → model decides the next step.
Without that feedback cycle you have a model that makes one tool call and stops. With it,
the agent can check the claim history, notice the claim looks invalid, go read a different
policy document, and send a different email. That is the difference between an agent and a
chatbot with a plugin.

**Honest version:** agent = foundation model (plus its goal and instructions) + tools + a
loop that feeds tool results back to the model.

---

## What is Model Garden, with an example?

A **catalogue** inside Vertex AI listing every model you can deploy, each with a card
showing what it does, how to call it, and a Deploy button. It does not run anything itself,
it is the shelf you pick from.

Three kinds of model sit on it:

- **Google's own**: Gemini 3 Pro, Gemini Flash, Imagen and Nano Banana (images), Veo (video), Chirp (speech).
- **Third-party, hosted by Google**: Claude, Meta's Llama, Mistral.
- **Open models you can fine-tune and host yourself**: Gemma, plus task-specific ones like a BERT text classifier.

**Worked example.** You want an agent that reads support incident tickets and drafts
a summary. In Model Garden you filter to text generation and see Gemini 3 Flash (fast, cheap)
next to Gemini 3 Pro (slower, better reasoning). You deploy Flash, which gives you an
**endpoint**: a URL your agent calls. A week later the summaries read too shallow, so you go
back, deploy Pro, and change one model name in the agent's config. **The agent code does not
change.** That swap-without-rewriting is the entire point of having a catalogue.

---

## What is ADK?

**Agent Development Kit**: the pro-code path. A Python library you install and write agents
in, instead of clicking them together in Agent Studio.

What it gives you:

- **Define the agent in code**: its instructions, which tools it may call, which model it uses.
- **Turn your own functions into tools**: write a plain Python function like `get_claim_history(customer_id)` and ADK exposes it to the model as a callable tool.
- **Multi-agent structure**: a manager agent that delegates to sub-agents (one for underwriting, one for claims), with explicit rules for who hands off to whom. This is graph-based logic: you draw the routes rather than hoping the model works them out.
- **Runs anywhere**: locally for testing, then deployed to Agent Runtime for production.

**The trade-off in one line:** Agent Studio gives you a visual box where you type
instructions and pick tools from a list, fast, but you inherit the reasoning loop Google
built. ADK makes you write that structure yourself, which is why it is the answer whenever
the requirement is "must talk to our legacy internal system" or "must follow this exact
approval sequence".

Power Platform mapping: **Agent Studio is Copilot Studio; ADK is writing the bot in code.**

---

## What is the equivalent of BigQuery ML in Microsoft, AWS, Databricks?

BigQuery ML's idea: train the model **inside the warehouse, in SQL**, with no data movement.

| Platform | Equivalent | How close |
|---|---|---|
| **AWS** | **Redshift ML**: `CREATE MODEL ... FROM (SELECT ...)` in SQL, delegates to SageMaker behind the scenes | near-exact copy |
| **Microsoft** | no true equivalent. **Synapse / Fabric `PREDICT`** scores a model in SQL but cannot *train* one; training lives in **Azure ML** (that is the Vertex AI equivalent, not the BQML one) | partial, scoring only |
| **Databricks** | **AI functions** in Databricks SQL (`ai_classify`, `ai_analyze_sentiment`, `ai_query`) call a model from SQL; classic ML training is AutoML + MLflow in a notebook, not SQL | partial, LLM-flavoured |

**The gap worth noticing:** Microsoft is the odd one out. In the Microsoft stack you move data
to the ML tool; in BigQuery ML and Redshift ML the model comes to the data. That is the whole
selling point, no export, no copy, no second service.

---

## When does a job actually deserve an agent? (the word→PDF question)

Headless is fine, an agent behind an API endpoint with no chat UI is the *normal* production
shape. Chat is just one front end.

**But a Word→PDF converter is not an agent use case.** One input, one fixed operation, one
output, no decisions. That is a library call in three lines; an agent adds latency, cost and a
model that can get it wrong.

**The rule: if you can draw the flowchart, write the code.** An agent earns its place only
when the *steps vary by input* and cannot be known in advance.

Three that pass the test:

- **Inbound RFP intake**: a PDF lands; the agent decides which of your 12 product sheets are relevant, pulls those, checks the deadline against capacity, and either drafts a response or routes it to a human with the reason. Every RFP needs a different set of steps.
- **Incident triage**: new security ticket; the agent decides whether to query the SIEM, look up the asset owner, or check for a duplicate, does whichever it picked, then re-decides based on what came back.
- **Invoice exception handling**: invoice does not match the PO; the agent works out *why* (partial shipment? price change? wrong PO number?) and each cause needs a different system checked.

**Common thread: branching a human would otherwise do.** Not "run this pipeline".

---

## What does agent code look like, and is it all code instead of prompt?

**Both, the prompt lives inside the code as a string.** Code wires up which tools exist and
who calls whom; the prompt still does the reasoning.

```python
from google.adk.agents import Agent

def get_ticket(ticket_id: str) -> dict:
    """Fetch a support ticket by id."""  # the model reads this line to decide when to call it
    return ticketing_api.get(ticket_id)

triage_agent = Agent(
    model="gemini-3-flash",
    instruction="Triage the ticket. Check for duplicates before summarising.",
    tools=[get_ticket, find_duplicates],
)
```

Three things to see: **`instruction=`** is the prompt, same words you would type into Agent
Studio. **`tools=[...]`** is a list of your own functions, and the triple-quoted docstring
under each is not a comment, it is how the model knows what the tool does. **You never write
the loop**; ADK runs the decide-call-observe cycle, you only supply the pieces.

**Languages:** ADK is Python-first with a Java version. Other frameworks cover other
languages, such as LangChain (Python/TypeScript) and the model vendors' own SDKs. Python is the default because that is where the ML ecosystem lives.

---

## What is LangChain, and do I need it?

A third-party **library** (free, open source, `pip install langchain`) for building agents and
LLM apps, the vendor-neutral equivalent of ADK. Nothing hosted, nobody to pay. The company
behind it sells **LangSmith**, hosted observability that traces every model call. Give away
the library, charge for the dashboard.

Same code shape as ADK, with two differences: a `@tool` decorator above each function, and
**the model as a separate object** (`ChatGoogleGenerativeAI(...)`), that line is the whole
selling point, swap it for `ChatAnthropic(...)` and nothing else changes.

**Do you need it?** Roughly: one vendor → that vendor's SDK or ADK. Several vendors or
self-hosted models → a thin router like LiteLLM. A framework only when you actually want its
prebuilt pieces, knowing the cost is a layer between you and the model.

**Two things that weakened its case:** it was criticised for over-abstraction, the wrappers
hid what was actually sent to the model, so debugging meant debugging LangChain instead of
your prompt. And portability got cheaper anyway: the SDKs converged and most providers expose
an OpenAI-compatible endpoint.

Recognise the name in job posts and architecture diagrams.

---

## What does "portability" actually mean?

**How much you would have to rewrite if you switched vendors.** High portability = change one
line. Low portability = rewrite the project.

*Locked in:* your code says `model="gemini-3-flash"` and uses Google's library throughout.
"We're moving to another vendor's model" → rewrite the file: different library, function names, response shape.

*Portable:* one line says `ChatGoogleGenerativeAI(...)` and everything else is generic. Same
request → change that line.

**From the BI world:** a Power BI report with one ticketing system's field names hardcoded into 40 M
queries is not portable, switch to another ticketing system and you rebuild it. Read everything from a staging
table with generic column names and you swap one query.

**Portability is insurance against a vendor change.** It costs an extra layer now to save a
rewrite later, and if you never switch you paid for nothing. That is exactly the argument
people had about LangChain.

---

## Is agentic workflow about swapping models per step to save tokens?

**Right mechanism, wrong motive.**

Model routing per step is real, cheap model for easy steps (classify, extract), expensive
model for the one hard decision. Flash vs Pro is roughly an order of magnitude apart in price,
so at volume it is a genuine lever.

**But that is not why agentic workflows exist.** You would have the same routing in a plain
pipeline with no agent at all. The reason to build an agent is that *the steps are not known
in advance*. Model routing is an optimisation applied afterwards. Getting this backwards leads
to building an agent for a job that was a function, the most common wasted project in this
space.

**Two corrections inside that reasoning:**

- You do not need LangChain to mix models. **Model Garden hosts Claude, Llama and Mistral on Vertex**: you can route across vendors without leaving Google Cloud. LangChain's pitch is portability across *clouds*, not across models.
- Token cost is mostly **not** about which model. It is about how much context you resend on every loop. An agent that carries the full conversation plus five tool results into every step burns more than model choice ever saves. The levers: trim what goes back in, cache the stable prefix, make fewer tool calls.

And eyeballing a few outputs in Agent Studio is prototyping, not evaluation. Teams doing this
seriously run a fixed set of test cases through both models and score them, same reason you
do not ship a Power BI measure after checking one row.

---

## Can my Power BI measures become ML features?

Yes, same logic, different shape. Two changes, and the second is the trap.

**1. Grain.** A dashboard measure aggregates *up*: "12% of tickets breached this month." A
feature points *down* to one row: "this ticket's caller has a 23% breach rate." Same
calculation, opposite direction.

**2. Point-in-time correctness.** A dashboard measure uses everything known today. A feature
may only use what was known **when that row's event happened**. Get it wrong and you get
leakage.

**The rule:** a feature can only use data that existed *before* the thing you are predicting.
So dashboard logic is a fine starting point, but every measure needs the question asked of
it: *would I have known this at prediction time?* Anything touching the outcome, resolution
date, closure code, final state, is out.

---

## Can all the feature engineering be done in SQL, in BigQuery?

**Yes, for tabular data, and it is the preferred path, not just a possible one.** Joins,
aggregates, date maths, `CASE` binning, window functions for running totals and rankings are
all native. Write one query, `CREATE TABLE ... AS SELECT`, point AutoML or BigQuery ML at the
result.

**Why preferred:** the data does not move. No CSV export, no pandas on a laptop that dies at 10
million rows, no second copy drifting from the first.

**Where SQL runs out:** images, audio, video (not a SQL job at all), some library-specific
scikit-learn transforms, and turning free text into **embeddings**: although BigQuery ML can
now call an embedding model, so even that is reachable.

For help desk ticket and vulnerability data, all tabular and all in a ticketing system, none of those limits apply.

### Reading the self-join feature query

Three aliases, **all the same table**: `incidents i`, `incidents p`, `incidents q`. The letter
after a table name is an **alias**, a short label for *which copy you mean*. You need copies
because you are on one row and asking about other rows in the same table (a self-join, or here
a correlated subquery that re-runs per row).

With 4 tickets, building the training row for **INC004** (jdoe, opened Jan 20):

| number | caller | group | opened | closed | breached |
|---|---|---|---|---|---|
| INC001 | jdoe | L2 | Jan 05 | Jan 06 | TRUE |
| INC002 | jdoe | L2 | Jan 10 | Jan 11 | FALSE |
| INC003 | asmith | L2 | Jan 18 | Jan 25 | TRUE |
| INC004 | jdoe | L2 | **Jan 20** | Jan 22 | TRUE |

- **`i`** = the ticket being described → INC004.
- **`p`** = that caller's *earlier* tickets. `p.caller = i.caller AND p.closed_at < i.opened_at` leaves INC001 (TRUE) and INC002 (FALSE). INC004 itself is excluded (closed Jan 22, after Jan 20); INC003 is excluded (wrong caller). **Average = 0.5.**
- **`q`** = tickets open *at that moment*: `q.opened_at < i.opened_at AND q.closed_at > i.opened_at`. Only INC003 spans Jan 20. **Count = 1.**

Result row: `priority_rank, category, group, opened_hour, 0.5, 1, breached_sla = TRUE`.

`WHERE i.closed_at IS NOT NULL` at the end, only finished tickets can be training rows,
because only they have a known answer to learn from.

**Power BI parallel:** a DAX measure using `CALCULATE(..., ALL(Table))`: inside one row's
context but reaching across the whole table.

---

## Is hardcoding a correction into SQL good practice?

**Business logic in SQL is fine**: that is what ELT is, and why Dataform exists (`.sqlx` in
git, reviewed, versioned). **Hardcoding `id=2` is the problem.** It is a magic value buried in
a query: nobody knows why 500, who decided, or when. Next month there are three more and the
`CASE` grows a tail; six months later someone deletes it because it looks like test code.

**The fix, corrections in a table, not in the query:**

```sql
-- sales_corrections (id, corrected_amount, reason, approved_by, applied_on)
SELECT s.* REPLACE(COALESCE(c.corrected_amount, s.amount) AS amount)
FROM their_dataset.sales s
LEFT JOIN my_dataset.sales_corrections c USING (id)
```

- **`LEFT JOIN`** keeps every sales row even with no matching correction (those get NULLs).
- **`COALESCE(a, b)`** takes the first non-NULL, the correction if one exists, otherwise the original.
- **`s.* REPLACE(... AS amount)`** takes every column but swaps out `amount`, so you need not list the others by hand.

Now the *logic* is "apply approved corrections" and never changes again; the *data*, which
rows, what values, who approved, lives in an auditable table.

**The rule: logic belongs in SQL, exceptions belong in data. If you are editing a query to
change a number, that number should have been a row.**

**And the question above all of it:** why is the row wrong at source? A correction layer is
right only when you genuinely cannot fix upstream (a vendor feed, a frozen system). Every
correction layer is permanent debt you will maintain for years.

---

## Does that correction pattern still work at 50 billion rows?

**The join is fine; the rewrite is not.** A tiny corrections table gets broadcast to every
worker, so the join costs almost nothing. The problem is that `CREATE TABLE AS SELECT` scans
50B rows and writes 50B rows *every time you add one correction*, a full scan plus doubled
storage to change 40 values.

In order of preference:

1. **A view, not a table.** No copy, no storage, but every query pays the join. Fine if the table is partitioned and users filter to a date range.
2. **Partition, then rewrite only what is affected.** `MERGE INTO ... WHEN MATCHED THEN UPDATE`: with partition pruning this touches gigabytes, not petabytes. **The right answer if you own the table.**
3. **Materialized view**: Google keeps it incrementally refreshed; good when the same corrected result is queried constantly.

**General rule at that scale: never rewrite the whole thing to change part of it**: which is
exactly Iceberg's argument.

**And the scale question behind the scale question:** 40 hand-approved fixes on 50B rows is
statistically invisible and nobody downstream will notice. That effort belongs upstream.
Manual corrections make sense at thousands or millions of rows, where one wrong invoice moves
a total. At 50B, chasing individual rows is usually the wrong fight.

---

## Making permanent changes to old Iceberg files

**If you can read but not write** (someone else's bucket): `CREATE TABLE ... AS SELECT` into
your own project. The originals stay. That is not a failure, you never get to delete data you
do not control.

**If it is your own Iceberg table**, two *separate* operations people confuse:

1. **Compaction**: merges many small files into fewer big ones. Housekeeping, improves query speed. **Old files still exist and time travel still works.**
2. **`expire_snapshots`**: the one that actually deletes. Drops snapshots older than a cut-off and removes data files nothing points at any more.

**The gotcha:** expiring snapshots permanently kills time travel for that period and removes
the ability to undo a bad write from before the cut-off. Often exactly what compliance wants
("delete data older than 7 years") and exactly what an auditor does not.

People compact and are then surprised storage costs do not drop. Compaction is housekeeping;
expiry is deletion.

---

## Text columns → embeddings → what for?

**An embedding converts text into numbers representing meaning.** Nothing is looked up or
fetched. One column holding an array (typically 768 numbers), one row, one vector.

| number | description | desc_vector (4 of 768 shown) |
|---|---|---|
| INC001 | "Server won't boot" | `[0.91, 0.12, 0.88, 0.04]` |
| INC002 | "Machine fails to start" | `[0.89, 0.15, 0.85, 0.06]` |
| INC003 | "Printer out of toner" | `[0.07, 0.94, 0.11, 0.81]` |

INC001 and INC002 share **zero words** but sit almost on top of each other. INC003 is far away.
**Close numbers = close meaning**, and a model can compare lists of numbers where it cannot
compare sentences.

**Four use cases, all just "close numbers":**

1. **Duplicate detection**: compare a new ticket's vector to open tickets. Catches "Server won't boot" vs "Machine fails to start"; keyword matching cannot.
2. **Clustering into themes**: k-means over 40,000 ticket vectors produces groups nobody defined: VPN, printers, password resets. Unsupervised learning on text.
3. **Semantic search**: "can't log in remotely" finds VPN tickets that never use the word login.
4. **As features**: those 768 numbers become 768 columns, so a model can learn that *certain kinds of problem description* breach more often. Signal `description_length` would never capture.

**And it is the engine behind RAG.** "Retrieve the relevant documents" = vectorise the
question, find the nearest document vectors, paste those into the prompt.

**All of it in BigQuery ML, no data movement:**

```sql
CREATE TABLE incidents_vec AS
SELECT *, ML.GENERATE_EMBEDDING(MODEL `my_embed`, description) AS desc_vector FROM incidents;

SELECT * FROM VECTOR_SEARCH(TABLE incidents_vec, 'desc_vector',
  (SELECT desc_vector FROM incidents_vec WHERE number='INC0500'), top_k => 5);

CREATE MODEL ticket_themes OPTIONS(model_type='kmeans', num_clusters=8) AS
SELECT desc_vector FROM incidents_vec;
```

`MODEL my_embed` is a Vertex AI embedding model registered once and then called from SQL like
a function, BigQuery makes the API call. `VECTOR_SEARCH` is built in. `num_clusters=8` is the
k-means hyperparameter.

---

## BigQuery ML equivalents, updated for embeddings and vectors

| Platform | Equivalent | How close |
|---|---|---|
| **Snowflake** | **Cortex**: `SNOWFLAKE.CORTEX.EMBED_TEXT_768()`, `.SENTIMENT()`, `.COMPLETE()`, a native `VECTOR` type and `VECTOR_COSINE_SIMILARITY()` | closest of all, arguably ahead |
| **Databricks** | `ai_query()`, `ai_classify()`, `ai_analyze_sentiment()` in SQL, plus Mosaic **Vector Search** | very close |
| **AWS** | **Redshift ML** (`CREATE MODEL` in SQL) for classic ML; embeddings route through Bedrock or SageMaker | close on classic, clunkier on embeddings |
| **Microsoft** | still weakest. Fabric has notebooks and Azure AI integration, Azure SQL added vector support, but no `CREATE MODEL` in SQL, training lives in Azure ML | partial |

**The pattern:** everyone converged on *call a model as a SQL function, keep the data where it
is*. Snowflake Cortex and BigQuery ML are the two purest versions. **Microsoft remains the
outlier, in that stack you still move data to the ML tool.**

---

## Can the platform draw a cost-weighted threshold chart for me?

**Not in Vertex. There is no "enter your costs" box, and that is a genuine gap.** Google gives
you precision, recall and the curves, never money.

**So you build it yourself, and it is small.** Export the test-set predictions, then:

```sql
SELECT t AS threshold,
       COUNTIF(prob >= t AND actual = 0) * 1500    -- FP cost
     + COUNTIF(prob <  t AND actual = 1) * 10000   -- FN cost
       AS total_cost
FROM predictions, UNNEST(GENERATE_ARRAY(0.05, 0.95, 0.05)) AS t
GROUP BY t ORDER BY total_cost
```

Twenty rows, sorted, lowest cost first. Drops straight into Power BI: threshold on x, dollars
on y, look for the bottom of the U.

**Where the platform helps a little:** the **optimization objective** under Advanced options at
training time (precision-at-recall, recall-at-precision rather than plain AUC). Not
cost-aware, but it nudges training in the right direction.

**Why no vendor ships the cost chart:** the two cost numbers are not data, they are a contested
business claim. Finance says a default costs $10,000; growth says a refused customer costs far
more than $1,500 in lifetime value. The tool cannot resolve that argument so it stays out of
it. **Getting those two numbers agreed, in writing, is the product decision, and it is ten
minutes of SQL once you have them.**


## Side questions from practice

- **ACL** = Access Control List, a permission list on ONE object (file). IAM = one grant on bucket/project covering everything. Fine-grained bucket = IAM + ACLs; uniform = IAM only.
- **Dataplex / Knowledge Catalog** is one product (nav menu > Analytics), not a page per table. Jobs: catalog/search, glossary, profiling, data quality scans, lineage, lakes/zones. Surfaces inside BigQuery as the table's Lineage / Data profile / Data quality tabs. Not for moving data (Dataflow/Dataform) or external sharing (Analytics Hub).
- **Looker Studio**: Share button = who can open the report (per report). Data credentials = whose login runs the query (per data source). Owner's credentials never hands out your login; like a stored API login in a Power BI semantic model. Share as Can view, not Can edit (an editor can chart any column on your access).
- **BQML LOGISTIC_REG is multiclass** (up to 50 labels). It still loses when the job is classifying images, because file metadata says nothing about pixels. Answer in the pixels → image model; in the columns → BQML.
- **KMEANS** = unsupervised, finds its own groups; you choose only how many (num_clusters). Known categories + labels → classify instead. Also usable for anomaly detection.
- **ARIMA_PLUS** is forecasting (learns from the past values of the same column), not unsupervised. Labels → classify/regress; no labels → cluster; future over time → forecast.
- **Bigtable** = NoSQL wide-column, massive time-series writes (constant stream of new timestamped rows) + millisecond lookup by row key. Open-source equivalents HBase / Cassandra; Azure = Cosmos DB for Apache Cassandra.
- **Cloud SQL vs AlloyDB**: Cloud SQL by default; AlloyDB (PostgreSQL only) when CPU is maxed, reporting slows the app, or you need analytics on live data. Spanner = global, multi-region writes, 99.999%.
- **Workflows** = Google-only serverless step-runner (≈ AWS Step Functions / Azure Logic Apps / Power Automate in YAML). Composer when dozens of interdependent pipelines, backfills, Python, Airflow operators, cross-cloud.
- **BigQuery long-term storage**: a table or partition unmodified 90 days bills ~50% cheaper automatically; reads don't reset the clock, writes do.
- **Clustering** = rows kept sorted in blocks by up to 4 columns so filters skip blocks. Partition by date + cluster by customer_id is the usual combo. Both cut query cost, not storage cost.
- **Expiration scopes**: BigQuery table expiration, dataset default table expiration (new tables only), partition expiration (drops days, table stays). GCS: bucket lifecycle rules with filters (age, prefix, class). Nothing at project level, no per-file expiration.
- **Cloud SQL HA vs cross-region replica**: HA = zone failure, automatic, no data loss. Cross-region replica = region failure, manual promote, a few seconds of loss. Read replica = read-only copy so reports don't slow the writer.
- **Policy tags**: one reusable lock defined once in a taxonomy, attached to many columns across tables, access granted once (Fine-Grained Reader). Column only (rows = row-level policies). Works on native and BigLake tables, not on plain external tables or raw bucket files (those = IAM only). XML must be converted before loading.
