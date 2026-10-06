# Self-check questions

These are questions I use to check myself, each with the right answer. Where I slipped, I also wrote down the trap I fell for and why it is wrong.

## Data engineering basics

1. **Which pipeline stage reshapes and prepares data so it fits what the next consumer needs?**
   Answer: **Transform**

2. **What really separates a data lake from a data warehouse?**
   Answer: **A lake keeps data raw and unprocessed. A warehouse keeps data that is already processed and organized.**
   Missed: on the first pass I picked the option that said lakes are for real-time analytics and warehouses are for long-term storage. That option describes *when you query*, not *what is stored*. The real split is raw vs pre-processed or aggregated.

3. **What is Analytics Hub for?**
   Answer: **Sharing data in a secure, controlled way, both inside the organization and with outside parties.**

4. **Where do unstructured files such as images and videos belong?**
   Answer: **Cloud Storage**

5. **What is the core job of a data engineer?**
   Answer: **Building data pipelines and keeping them running.**

## Data replication and migration

1. **Which service moves big datasets into Cloud Storage from on-premises systems, other clouds' file systems and object stores, or HDFS, and can run on a schedule?**
   Answer: **Storage Transfer Service**
   Signature words: "scheduled" plus "S3 / Azure / HDFS". Distractor: Vertex AI (unrelated, that is ML).

2. **A Datastream event message has several sections. Which one holds the changed data itself, as key-value pairs?**
   Answer: **Payload**
   The three sections are generic metadata, source-specific metadata, and payload. Metadata = context; payload = the values.
   Trap: **"Change log" is a distractor.** The change log (WAL) is what Datastream *reads from* on the Postgres side. It is not a section of the event message. Sounds right, is not.
   Anchor from my hands-on work: in BigQuery, `id / text_col / int_col / date_col` were the payload; `datastream_metadata` was the metadata column beside them.

3. **Which option fits a very large dataset that has to move offline?**
   Answer: **Transfer Appliance**
   "Offline" only ever means the appliance.

4. **Which tool does one-off copies into Cloud Storage with a `cp` command?**
   Answer: **The `gcloud storage` command**
   `cp` and "ad-hoc" both point at the CLI.

5. **Which two things decide most how easy a migration will be?**
   Answer: **Data size and network bandwidth**
   The 1 TB math: at 100 Gbps it takes about 2 minutes, at 100 Mbps about 30 hours.

## Extract and load

1. **Why choose a BigLake table over a plain external table?**
   Answer: **BigLake gives better performance, stronger security, and more flexibility.**
   The other three options describe *external* tables or are false. "Simpler to set up" is the external table's advantage. Needing separate permissions on the table and on the data source is the external table's *weakness*. "Tight limits on formats and storage locations" is backwards, BigLake supports more.

2. **How stale can BigLake's metadata cache be allowed to get, and how is it refreshed?**
   Answer: **Anywhere from 30 minutes to 7 days, with the refresh either automatic or manual.**
   The distractors each flip one half: "15 minutes to 1 day", "manual only", "not configurable".

3. **When is the `LOAD DATA` statement the right choice?**
   Answer: **When loading into BigQuery tables has to be automated from a script or an application.**
   Loading a local CSV by clicking through a graphical interface is the *UI* option. `LOAD DATA` exists precisely because the UI cannot be automated.

4. **What does the BigQuery Data Transfer Service do?**
   Answer: **It is a fully managed service that schedules and automates moving data into BigQuery from many sources.**
   The distractors are the other tools: querying across Cloud Storage and other clouds' object stores = BigLake; querying in place in Cloud Storage = external tables; a command-line tool for loading = `bq load`.

5. **Which feature lets you query files sitting in Cloud Storage without loading them first?**
   Answer: **External tables**
   BigLake would also be correct in principle, but it was not among the options. `bq load` and `bq mk` both involve loading or creating, and the Data Transfer Service copies data in.

## ELT

1. **What does BigQuery's procedural language mainly give you?**
   Answer: **Running several SQL statements one after another while they share state.**
   "Shared state" is the phrase to remember: variables persist across statements. Distractors: custom transformations in Python = remote functions; external libraries = JavaScript UDFs; making a single query faster = not what procedural means.

2. **What are assertions in Dataform for?**
   Answer: **Data quality tests that check the data is consistent and accurate.**
   The distractors are the other Dataform pieces: dependencies = `ref()` / the `dependencies` array; compiling SQLX = Dataform's compiler; scheduling = internal or external triggers.

3. **What is the main benefit of stored procedures in BigQuery?**
   Answer: **Code that is easier to reuse and to maintain.**
   Each distractor is a different tool. "Only Python" is false (stored procedures are SQL; Spark stored procedures take Python, Java, or Scala). "Integrate with external APIs" = remote functions plus Cloud Run. "Ad-hoc SQL execution" is backwards, stored procedures exist for *repeatable* logic.

4. **What is the core idea of ELT?**
   Answer: **Load the data into the warehouse first, then transform it inside the warehouse.**
   Trap: **I nearly got this wrong.** The tempting option extracts, loads into a *staging area*, and then transforms. That describes **ETL**, where the staging area sits *before* the warehouse. The term "BigQuery staging tables" does exist, but those are **inside** BigQuery.
   **The test:** *where does the compute happen?* Inside the warehouse = ELT. Somewhere else first = ETL.

5. **How does Dataform make SQL workflows easier to manage?**
   Answer: **It brings transformation, assertions, and automation together inside BigQuery, so data operations run in one place.**
   This follows Google's own wording. Distractors: "replaces SQL entirely with Python" (Dataform is SQL plus JavaScript, no Python); automating migration from on-prem = Database Migration Service; a visual interface for ad-hoc queries = BigQuery Studio.

## ETL

1. **Why use Dataflow templates?**
   Answer: **The pipeline becomes reusable, and you configure it with parameters.**
   Same pattern as the streaming taxi exercise: Google's pipeline code is fixed, you only supply parameters (source, target, schema, UDF). Distractors: on-prem migration = Database Migration Service; external APIs = not a template feature; "replaces SQL" = templates have nothing to do with SQL.

2. **Which service transforms data serverless and with no code, using recipes?**
   Answer: **Dataprep**
   The trigger word is **"recipes"**: Dataprep's name for a saved list of wrangling steps. Two clues rule out the rest. **Serverless** kills Data Fusion and Dataproc (of the four, only Dataprep and Dataflow are serverless). **No-code** kills Dataflow (Beam code or templates). Data Fusion is also visual, but it calls its jobs *pipelines*, not recipes.

3. **Which service suits complex pipelines built on a visual, drag-and-drop canvas?**
   Answer: **Data Fusion**
   Missed: **I picked Dataflow.** Dataflow is code (Beam) or templates filled in by form; it has no drag-and-drop canvas. The two visual tools are Dataprep and Data Fusion, and the tiebreaker is **"complex pipelines"**: Dataprep cleans one dataset with a recipe, Data Fusion wires many sources to transforms to sinks on a canvas (built on CDAP, hybrid and multicloud).
   **The test:** visual + *cleaning one file* = Dataprep. Visual + *connecting many systems* = Data Fusion.

4. **The same point as question 2, asked again with the options in a different order.**
   Answer: **Dataprep**

5. **What makes Dataproc Serverless for Spark a good fit for interactive development and exploration?**
   Answer: **JupyterLab integration**
   The trigger word is **"interactive"**: a notebook where you run one cell, look at the result, adjust. Distractors: BigQuery external procedures = running Spark *from* BigQuery as a stored procedure (production, not exploration); workflow templates = scheduling multi-step jobs (automation); custom containers = packaging your own dependencies (environment control).

## Automation

1. **Which service ties loosely coupled services together in one event-driven architecture?**
   Answer: **Eventarc**
   Tell: "architecture" + "loosely coupled" = the router between services. Functions are the endpoint that runs code, Scheduler is a clock, Composer is fixed-order pipelines (tightly coupled).

2. **Which service runs your code when a Google Cloud event fires?**
   Answer: **Cloud Run functions**
   Tell: "execute code". Eventarc delivers the event, the function does the work.

3. **Which service kicks off workloads on a fixed, repeating timetable?**
   Answer: **Cloud Scheduler**
   Missed: **I picked Composer first.** "Automate tasks" is the distracting phrase, because every option automates. "Recurring intervals" = a clock = cron. Composer can run on a schedule but is built for multi-step pipelines.

4. **Which service is the central orchestrator that connects pipelines across many different systems?**
   Answer: **Cloud Composer**
   Missed: **I guessed Eventarc first.** Eventarc *routes* one event from A to B and does not care what happens next. An orchestrator *conducts*: runs step 1, waits, runs step 2, retries, tracks state. Example: a nightly run that pulls from an API, loads BigQuery, transforms, then refreshes a report.

5. **In Cloud Composer, what does DAG stand for?**
   Answer: **Directed acyclic graph**
   Directed = arrows one way; acyclic = no loops back; graph = tasks (boxes) + dependencies (arrows). Tasks run in dependency order; independent ones can run in parallel. Built on Apache Airflow.

**Quick reference:** Eventarc = router ("event-driven architecture", "loosely coupled"). Cloud Run functions = doer ("execute code in response to"). Cloud Scheduler = clock ("recurring intervals", cron). Cloud Composer = conductor ("orchestrate pipelines", DAG, Airflow).

## BigQuery basics and query debugging

Questions from my hands-on BigQuery work that were worth keeping.

1. **Which product is the fully managed, petabyte-scale data warehouse on Google Cloud?**
   Answer: **BigQuery**
   Missed: **I wondered if it was a Cloud Storage bucket.** Tell: "data warehouse" = tables you query with SQL. A bucket keeps files (the data lake), not a warehouse.

2. **True or false: tables sit inside datasets, and datasets sit inside projects.**
   Answer: **True.** Read the table name left to right: `bigquery-public-data` . `london_bicycles` . `cycle_hire`.

3. **True or false: BigQuery lets you query public datasets that live in other Google Cloud projects.**
   Answer: **True.** I queried Google's `bigquery-public-data` from my own practice project.

4. **In which ways can you reach BigQuery? (more than one)**
   Answer: **Command line tool, web UI, BigQuery REST API.** GLib and GStreamer are unrelated Linux libraries, pure distractors.

5. **Which command-line tool talks to BigQuery?**
   Answer: **bq**

6. **How many rows does `all_sessions_raw` hold?**
   Answer: **Over 21 million.** 5.63 GB is a size, not a row count.

7. **In the de-duplication query, which clause removes the duplicate records?**
   Answer: **GROUP BY**

8. **In the results, how does `orders` differ from `quantity_product_ordered`?**
   Answer: **`orders` is the number of orders, the quantity is the number of items ordered.** `COUNT(productQuantity)` counts non-null rows (purchases); `SUM(productQuantity)` adds the units. Sara 1 + Tom 5 + Ali 4 = 3 orders, 10 units.

9. **True or false: the product with the most views was also the one with the most orders.**
   Answer: **True.**
   Missed: **my earlier notes said False, from a guess made before the query was run.** Confirmed True when I ran the query again. Tell: do not answer a results question before running the query.

10. **What is wrong with `SELECT FROM data-to-inghts.ecommerce.rev_transactions LIMIT 1000`?**
    Answer: **No columns in SELECT**, and the **typo** (`inghts`). The answer option calls the misspelled part the dataset name. It is really the project name.

11. **What is wrong with `SELECT * FROM [data-to-insights:ecommerce.rev_transactions] LIMIT 1000`?**
    Answer: **It is legacy SQL.** Tell: square brackets + colon in the table name. Standard SQL = backticks + dots.

12. **What is wrong with `SELECT FROM data-to-insights.ecommerce.rev_transactions` (name fixed, still no columns)?**
    Answer: **Still no columns defined in SELECT.**

13. **What is wrong with `SELECT fullVisitorId FROM ...rev_transactions`? (more than one)**
    Answer: **With no aggregation, limit, or sorting the query gives no insight**, and **the page title is missing from the columns in SELECT.**
    Missed: **I ticked only the "no insight" option and missed the page title.** The question is who reached *checkout*, and only `hits_page_pageTitle` says which page a visitor was on. The column alias and LIMIT options are wrong: an alias is optional, and LIMIT is already covered by the "no insight" option.

14. **How many columns does `SELECT fullVisitorId hits_page_pageTitle FROM ... LIMIT 1000` return?**
    Answer: **1, a column named hits_page_pageTitle.**
    Missed: **I picked "0, the query returns an error".** I spotted the missing comma correctly, but no comma = alias (`AS` is optional), so it runs and returns visitor IDs under the wrong label. That silence is the danger.

15. **What is wrong with `SELECT COUNT(fullVisitorId) AS visitor_count, hits_page_pageTitle FROM ...`? (more than one)**
    Answer: **The GROUP BY is missing**, and **COUNT() does not de-duplicate fullVisitorId.** Got both.

16. **The same aggregates, now filtered with `WHERE avg_products_ordered > 20`. What is wrong? (more than one)**
    Answer: **WHERE cannot filter on an aliased field**, and **WHERE cannot filter on an aggregated field (use HAVING).** WHERE runs before grouping, so neither the SUM nor its alias exists yet. Dividing a SUM by a COUNT is fine.

17. **What is wrong with `COUNT(hits_product_v2ProductName) ... GROUP BY hits_product_v2ProductCategory`?**
    Answer: **COUNT() here is not the distinct number of products in each category.**
    Missed: **I also ticked the option saying the GROUP BY column is wrong.** "Per category" means `GROUP BY` category is exactly right; only the count is off. Pen, Pen, Pen, Notebook = COUNT 4, COUNT(DISTINCT) 2.

## AI foundations

1. **Which family of models sorts unlabelled photos into sets?**
   Answer: **Clustering**
   Missed: I picked **dimensionality reduction**.
   **The tell is the word *group*.** Clustering groups *rows* (each photo goes into a set). Dimensionality reduction does not group anything; it reduces *columns*, squeezing many features into fewer while keeping the signal. Both are unsupervised, which is why the distractor works. Ask "is it sorting rows into piles, or shrinking the number of columns?"

The other four were correct on the first pass and I did not record them.

## AI agent components

Scenario: an agent that handles insurance claims. It has to (a) fetch the claim history from an outside database, (b) check the claim against the company's own policy documents, and (c) send a confirmation email.

Match each description to its component:

| Description | Component |
|---|---|
| Understands the language and does the reasoning | **Model** (the brain) |
| Reaches internal and external resources and services | **Tools** (hands, feet, senses) |
| Controls the order of actions and decisions | **Orchestration layer** (the nervous system) |

Got all three. The wording to watch: **"sequence" and "decisions" always mean orchestration**, never the model. The model decides *what* the next step is; orchestration decides *when and in what order* steps run and carries results back.

## AI development workflow

1. **What is the right order of the ML workflow?**
   Answer: **Data preparation, model development, model serving.** Correct.

2. **A farm screens apples for defects. It wants to flag only the apples that are truly bad, so that no good apple is thrown away. Which metric matters?**
   Answer: **Precision**
   Missed: I answered **Recall**.
   Positive = defective apple. A good apple flagged bad gets thrown out = **false positive**. Minimising FP is precision.

3. **Which stage covers training the model and evaluating it?**
   Answer: **Model development.** Correct.

4. **Which product is the toolkit that automates, monitors, and governs ML systems by orchestrating the workflow in a serverless way?**
   Answer: **Agent Platform Pipelines**
   Missed: I answered **Explainable AI**. Explainable AI tells you *why* a prediction happened; it orchestrates nothing. The keywords **automate / orchestrate / serverless** always point at Pipelines.

5. **A hospital uses past patient data to pre-screen for cancer. It wants to catch as many possible cases as it can. Which metric matters?**
   Answer: **Recall**
   Missed: I answered **Precision**.
   "As many as possible" means the fear is missing a case = **false negative**. Minimising FN is recall.

### The miss that matters: questions 2 and 5 were exactly swapped

I understood the concepts; I had the labels reversed. The hook to keep:

| Phrase in the scenario | Metric | Afraid of |
|---|---|---|
| "catch them ALL" | **Recall** | missing one (FN) |
| "only flag the REAL" | **Precision** | crying wolf (FP) |

- **Recall** = did we catch everything? Denominator = all actual positives.
- **Precision** = when we shouted, were we right? Denominator = everything we flagged.

Read the cost of each error in the scenario, not the wording of the goal:

| Scenario | A miss (FN) costs | A false alarm (FP) costs | Metric |
|---|---|---|---|
| Cancer screening | a death | one unnecessary biopsy | **Recall** |
| Defective apples | one bad apple ships | good stock destroyed | **Precision** |

And the loan exercise is the same trade: FN cost 10,000 dollars, FP cost 1,500 dollars, so the threshold dropped to 0.3. **Lowering the threshold buys recall and spends precision.**
