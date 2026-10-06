# Learning notes 3: AI and machine learning

## The three-layer AI stack (the spine of everything below)

Every product in these notes sits on one of three shelves. If a question asks "which
product", it is usually really asking "which layer".

| Layer | What lives there | Who uses it |
|---|---|---|
| **AI applications / solutions** | Gemini Enterprise app, Gemini Enterprise for CX, Gemini Notebook | business users, analysts, no code |
| **AI development tools** | Agent Platform (Agent Studio, ADK, Agent Garden, Agent Runtime), Vertex AI training/serving, BigQuery ML | developers, data scientists |
| **Foundation models / infrastructure** | Model Garden, Gemini family, Nano Banana, Veo, TPUs/GPUs | the raw engine everything above calls |

The trade-off axis across the layers is **ease of use vs flexibility**: you buy one with
the other. Top layer: minimal setup, minimal customisation. Bottom layer: full control,
full effort.

## AI foundations

### The nesting, worth knowing

**AI** (any machine mimicking human intelligence) contains **ML** (learns patterns from data
instead of being explicitly programmed), which contains **deep learning** (ML using
multi-layer neural networks), which contains **generative AI** (deep learning models that
*produce* new content rather than only scoring existing content).

Generative AI is a subset of deep learning, not a sibling of ML. Drawing it as a separate
box is the classic wrong answer.

### Supervised vs unsupervised, the split that drives every "which model" question

| | Supervised | Unsupervised |
|---|---|---|
| Data | **labelled**: you already know the right answer for past rows | **unlabelled**: no known answer |
| Question | "predict this value" | "find structure in here" |
| Model families | **classification** (which category, will this visitor buy? yes/no), **regression** (what number, what will revenue be?) | **clustering** (which rows belong together), **dimensionality reduction** (squeeze many columns into few without losing the signal), **association** (which things co-occur) |

**The tell for clustering: the word *group*.** "Group unlabelled photos into sets" is
clustering. Dimensionality reduction does not group *rows*, it reduces *columns*. Got this
one wrong the first time.

### Google Cloud's ML layers, bottom to top

- **TPUs**: Google's own chips built specifically for the matrix maths in neural networks. The reason "train it on Google Cloud" is a real argument and not just marketing.
- **Vertex AI**: the managed platform for the whole ML lifecycle: data, train, evaluate, deploy, monitor. AutoML (point at data, Google picks the model) and custom training (bring your own code) both live here.
- **BigQuery ML**: train models with `CREATE MODEL` in SQL, on data already in BigQuery. No data movement, no Python, no separate service.

### The four rungs of AI development, easiest to hardest

1. **Pre-built APIs**: call someone else's finished model (Vision, Speech-to-Text, Translation). No training, no data.
2. **BigQuery ML**: SQL-trained models on your own warehouse data.
3. **AutoML on Vertex AI**: your data, Google picks and tunes the model.
4. **Custom training**: your data, your code, your model architecture.

Same ease-vs-flexibility ladder as the agent tools. Recognise the shape; it repeats.

## Hands-on: a BigQuery ML model that predicts which visitors will buy

**The job:** Google Analytics ecommerce session data already sitting in BigQuery, train a
**logistic regression** classifier that predicts whether a visitor will buy, evaluate it,
then rank the top visitors and top countries by predicted purchase probability.

Flow: `bigquery-public-data GA sessions -> a training view in my dataset -> CREATE MODEL -> ML.EVALUATE / ML.PREDICT`.

### The four functions, and what each is actually for

| Function | Answers |
|---|---|
| `CREATE MODEL ... OPTIONS(model_type='logistic_reg', input_label_cols=['label'])` | build it. `input_label_cols` names the column holding the known right answer |
| `ML.EVALUATE(MODEL x, TABLE y)` | how good is it, precision, recall, accuracy, F1, log loss, **AUC** |
| `ML.PREDICT(MODEL x, TABLE y)` | run it on new rows; returns the predicted class **and** the probability behind it |
| `ML.CONFUSION_MATRIX(MODEL x, TABLE y)` | the four-cell truth table: true/false by positive/negative |

**Label** = the column you are trying to predict. **Feature** = every column used to predict
it. In this exercise the label was built from the transactions column with
`IF(totals.transactions IS NULL, 0, 1)`. `IF()` takes three arguments: the condition, the
value when true, the value when false. Slot 2 is the true-case.

### The real gotcha, a view over an unordered LIMIT is not reproducible

The exercise's `training_data` view is defined with `LIMIT 10000` and **no `ORDER BY`**. SQL
tables have no inherent order, so every time the view is referenced BigQuery is free to
return a *different* 10,000 rows. `ML.EVALUATE` and `ML.CONFUSION_MATRIX` therefore
disagreed with each other, 85 actual buyers in one, 146 in the other, from "the same" view.

Rule to keep: **`LIMIT` without `ORDER BY` is a sample, not a subset.** If a view feeds
anything that must be reproducible (training data, a reconciliation, a published number),
either order it deterministically or materialise it as a table.

### Why AUC 0.98 still gave about 10% recall

The model scored an AUC of 0.98 and a recall around 10%. Both true, no contradiction.

- **AUC** measures *ranking* quality: given one buyer and one non-buyer, how often does the model score the buyer higher? 0.98 = almost always. AUC ignores where you draw the cut-off.
- **Recall** measures *catching*: of all real buyers, how many did we flag? That depends entirely on the cut-off, which defaults to **0.5**.
- The base rate here is about 1.5% buyers. A well-calibrated model on a 1.5% base rate rarely outputs a probability above 0.5 for anyone, so almost nobody gets flagged and recall collapses.

**The 10% is a property of the threshold, not of the model.** Drop the cut-off to 0.02 and
recall jumps. This is the single most transferable idea in the exercise.

### Ranking tool vs decision tool, the framing for whether a model is usable

- **Ranking tool:** "who are the 1,000 visitors most likely to buy?" Only the *order* matters. AUC 0.98 means this works beautifully. Retargeting, lead scoring, triage queues.
- **Decision tool:** "should we auto-approve this claim, yes or no?" A hard cut-off is required, so precision and recall at that specific threshold are what matter, and the threshold is a **business** choice about the relative cost of a false positive vs a false negative, not a modelling choice.

Ask which one you are building *before* asking whether the model is good enough.

## Generative AI

### Foundation models

A **foundation model** is a very large model pre-trained on broad general data, then
adapted to many uses. Pre-trained on general data is exactly why it does not know your
company's policies, the gap that grounding, RAG, tuning and agents all exist to close.

- **Gemini family**: general purpose, **multimodal**: takes text, images, audio, video, PDF as input in the same prompt. Pro (deeper reasoning) vs Flash (faster, cheaper) is the usual choice.
- **Specialised models**: **Nano Banana** (image generation and editing), **Veo** (video), Chirp (speech), Imagen (images).
- **Model Garden** is the catalogue you pick any of these from, Google's own, third-party (Claude, Llama, Mistral) and open models (Gemma) you can host and tune yourself. It is a shelf, not a runtime; deploying from it gives you an endpoint your app calls.

### Prompt engineering, the first half of the prompt-to-production lifecycle

A good prompt has three components:

1. **Task**: what to do. The verb.
2. **Context**: the background, constraints, role, tone, output format.
3. **Examples**: sample input/output pairs. Zero-shot (none), one-shot (one), few-shot (several).

Examples are the strongest lever on *format* and *style*. They do not add knowledge.

**Model parameters** exposed for tuning the output:

| Parameter | What it controls |
|---|---|
| **Temperature** | randomness. 0 = always the most likely next token (deterministic, good for extraction and classification). High = more varied, good for creative work |
| **Top-K** | only consider the K most likely next tokens |
| **Top-P** (nucleus) | only consider the most likely tokens whose probabilities sum to P |

Agent Studio supports prompt templates and **side-by-side prompt comparison** for evaluating
a change instead of eyeballing it.

**In the exercise's Agent Studio screen, the Gemini 3.x models showed no Temperature or Top-P**: only a "Thinking level".
If the point of an exercise is to watch those parameters move, switch to a 2.5-era model.

### Deployment and tuning, the second half

Agent Studio generates the application code and integrates with **Cloud Run** (serverless
container hosting) and **Cloud Shell**, so a prototype becomes a deployed app without
leaving the console.

**Grounding** = forcing the model to base its answer on a specified source (Google Search,
or your own data) and cite it, instead of answering from pre-trained memory.

**RAG (Retrieval-Augmented Generation)** = the mechanism behind grounding on your own data:
before answering, retrieve the relevant documents from your corpus, paste them into the
prompt, then let the model answer from them. The model is unchanged; the *prompt* got richer.

**The tuning ladder, cheapest to most expensive:**

| Rung | What changes | Cost |
|---|---|---|
| **Prompt design** | nothing in the model, just the instructions | free |
| **Parameter-efficient tuning (PEFT)** | a small set of added weights; the base model stays frozen | moderate |
| **Full fine-tuning** | every weight in the model is retrained | high, needs lots of data and compute |

### Fine-tuning vs RAG, the distinction worth memorising

**Fine-tuning teaches format and behaviour. RAG supplies facts.**

- Fine-tune when the model needs to consistently *sound* or *structure* a certain way, always output this JSON shape, always use this claims-report tone, always follow this house style.
- Use RAG when the model needs to *know* something it was never trained on, current policy documents, this quarter's numbers, a specific customer's history.

**Fine-tuning does not stop hallucination.** A fine-tuned model produces confidently wrong
answers in your house style. Only grounding and RAG put real facts in front of it, and even
then only for what is in the retrieved corpus. Facts change daily; re-training daily is
absurd, re-indexing daily is routine, which is the practical argument on its own.

## Hands-on: prototyping an agent in the low-code Agent Studio interface

Prototype agent built in the low-code visual interface: system instructions, few-shot
examples, model parameters, test in the chat panel.

### Console drift, assume the written steps are stale

The UI has moved away from the written steps of both exercises. Look for the modern equivalent rather than
hunting for the named control:

| Written steps say | Reality |
|---|---|
| BigQuery Gemini sparkle menu, "Explain this query" | gone. Use the **Gemini Cloud Assist** panel and its suggested chips |
| Agent Studio, System instructions | hidden until the right-hand panel is expanded via the **double-chevron** |
| Few-shot steps, fill the Input field | that field no longer exists; the example attaches silently, just type the new case |
| Set Temperature / Top-P | not shown for Gemini 3.x in the exercise screen, only "Thinking level" |

## AI agents, the next evolution

### The definition

An **AI agent** is an application combining AI models for reasoning, tools for external
interaction, and coordination, to achieve a goal. It is goal-oriented, uses a model as its
brain, employs tools to act, and can reason and decide autonomously.

The problem it solves: a foundation model can only *talk*. Its knowledge is pre-trained,
generally not field-specific, and it cannot reach your applications. An agent connects to
information and applications outside the model, **takes action**, and **observes feedback
from the environment** to improve over time.

### The three components, know these cold

| Component | Body analogy | Job |
|---|---|---|
| **Model** | the brain | reasoning centre. Thinks, plans, decides the steps needed to reach the goal. Usually general-purpose, optionally refined with agent-specific examples |
| **Tools** | hands, feet and senses | connectors to the outside world, usually **APIs** (GET / POST / PATCH / DELETE = read / create / update / delete). Hands and feet *act* (send an email); senses *gather* (fetch weather data) |
| **Orchestration layer** | the nervous system | the central **cyclical** process. Carries the brain's decision out to a tool, then carries the result back to the brain to inform the next step |

**The loop is the thing.** Model, then tool, then result back to model, then next decision.
Without it you have a chatbot that can make one call and stop. With it, the agent adapts to
what each step returned.

**Agentic AI** = a more autonomous system coordinating **multiple agents** through complex,
multi-step reasoning, beyond what a single agent does. Progression:
**chatbot, then AI agent, then agentic AI**: conversational, then actionable, then proactive.

Reference: Google's **"Agents" whitepaper**.

### Is an agent just "model + instructions + MCP"?

Nearly. Instructions are the *input* that makes it goal-oriented, not a fourth component.
**MCP (Model Context Protocol)** is one standard way of handing tools to a model, plumbing
that sits inside the **tools** component. An agent does not require MCP; many call APIs
directly or use a framework's own tool format. And the formula misses the orchestration
loop, which is the part that makes it an agent rather than a chatbot with a plugin.

## Agent building on Google Cloud, the product map

### The tools, by layer

- **Model Garden** (bottom), the brain supply.
- **Agent Platform** (middle, for developers), build end-to-end, design to deployment:
  - **Agent Studio**: low-code visual interface. Design the reasoning loop and test prompts in natural language.
  - **ADK (Agent Development Kit)**: pro-code. Graph-based logic defining how networks of **sub-agents** collaborate. Write your own Python functions and expose them as tools.
  - **Agent Garden**: pre-built templates.
  - **Agent-to-Agent orchestration**: agents delegate work to one another, generative or deterministic patterns.
  - **Secure workspaces**: hardened, sandboxed environments for agents to handle files and run commands.
  - **Agent Runtime**: deploy, manage and scale in production. Sub-second cold starts, long-running autonomous workflows, integrated observability.
- **Gemini Enterprise app** (top, for business users), no-code. Unites AI, Google-quality **search** and your proprietary data in one secure space, breaking data silos. Multimodal search returns **cited** answers across documents, images and video. Hosts pre-built Google agents plus a no-code **Agent Designer**. **Gemini Enterprise for CX** is the customer-facing variant (retail and similar). **Gemini Notebook** (free to the public) is the research-assistant agent in this layer.

### The decision tree, the part worth knowing

| Need | Reach for | Who |
|---|---|---|
| Out-of-the-box, no code, minimal setup | **Gemini Enterprise app** / Gemini Enterprise for CX | business users |
| Quick idea to prototype, low code, a starting template | **Agent Studio** / **Agent Garden** on Agent Platform | data scientists, analysts |
| Highly tailored, complex logic, deep integration with legacy systems and internal databases | **ADK** or build your own | developers |

**The trigger for leaving the no-code layer** (a typical scenario):
Gemini Enterprise app could not integrate with the company's legacy
applications and internal databases, and could not enforce specific response behaviours
(industry policy for claim reports, warm empathetic tone for customer emails). Custom
integration plus custom behaviour equals Agent Platform.

Power Platform analogy: **Agent Studio is Copilot Studio** (click, configure, ship);
**ADK is writing the bot in code**: more work, nothing off-limits.

---

# Predictive AI: building an ML model

## The four development options (the ladder, filled in)

| Option | You bring | You need | Data types | Hyperparameters |
|---|---|---|---|---|
| **Pre-trained APIs** | nothing | nothing | tabular, image, text, video, **audio** | no |
| **BigQuery ML** | lots of data, already in BigQuery | SQL | tabular + semi-structured (JSON) only | yes |
| **AutoML** | your own data | clicks | tabular + image | no |
| **Custom training** | your own data | code | tabular, image, text, video | yes |

Training time: pre-trained APIs zero, the other three depend on the project, custom
training longest because it starts from nothing.

**The pick-one rules:** no ML experience and no intent to train → pre-trained
APIs. SQL people with data already in BigQuery → BigQuery ML. Own data, minimal coding →
AutoML. Full control of the workflow → custom training.

**One example across all four, "is this customer email angry?"**

- Pre-trained API: call the Natural Language API. One line, no data.
- BigQuery ML: 500k past emails tagged angry/not, already in BigQuery → `CREATE MODEL` in SQL.
- AutoML: same 500k rows → upload, click train, Google picks the architecture.
- Custom training: you need *industry-specific sarcastic* anger → write the model yourself.

Go down the list only when the rung above cannot do the job.

### Parameters vs hyperparameters vs inference parameters

The three get confused constantly, and Google's own UI labels make it worse.

| Term | When it applies | Examples |
|---|---|---|
| **Parameters** | learned *during* training | the millions of weights inside the model. You never touch them |
| **Hyperparameters** | set *before* training. Change one → must retrain | `learn_rate`, `max_iterations`, `l2_reg`, `num_trees`, `max_depth`, `num_clusters`, `batch_size` |
| **Inference / decoding parameters** | set at *call* time, per request | **temperature, top-K, top-P**, max output tokens |

**The line is training vs calling.** Temperature changes nothing about the model, only how
the finished model samples its next token on this one request. Same model, two calls, two
temperatures, two answers. They are also **generative-only**: a logistic regression has no
temperature, it outputs a probability and there is nothing to sample.

⚠️ Agent Studio labels that panel "model parameters" and people say "tune the
hyperparameters" loosely. **If a question ties temperature to training, that is wrong**:
temperature is inference.

**Hyperparameters are per-algorithm.** `num_trees` is meaningless to a regression,
`num_clusters` is meaningless to anything supervised. The near-universal two are `learn_rate`
and `max_iterations`, because anything that trains iteratively needs a step size and a
stopping point. In BigQuery ML the `OPTIONS` block only accepts the dials valid for the
`model_type` picked; the wrong one errors.

## Agent Platform (= Vertex AI), the unified platform

**The problem it exists for:** per Gartner, only about half of enterprise ML projects get
past pilot. Models die between "works in a notebook" and "runs in production":
scalability, monitoring, CI/CD/CT. Plus the ease-of-use problem: too many disconnected tools.

**Unified means two things:**

1. **The whole lifecycle in one place**: data readiness (load from Cloud Storage, BigQuery or a local machine) → feature readiness (build **features**, the processed columns fed to the model; share them via the **Feature Store**) → training and hyperparameter tuning → deployment and model monitoring.
2. **Both kinds of AI**: generative (Agent Studio, ADK) and predictive (AutoML, custom training).

**Feature Store, why it matters:** you build a feature `avg_ticket_age_30d` for a churn
model. Six months later a staffing model needs the same number and pulls it from the store
instead of rewriting the SQL and getting a slightly different answer. That is reuse of a
*definition*, not just data.

**Tools for predictive AI:** AutoML (no-code UI) or custom training in **Workbench** or
**Colab Enterprise**: both hosted Jupyter notebooks. Workbench can run SQL straight against
BigQuery.

**The four Ss** (marketing, but worth knowing): seamless, scalable, sustainable, speedy.

## AutoML, how the automation actually works

Announced January 2018, folded into Vertex AI in 2021. Four phases:

1. **Data processing**: automatically converts numbers, datetime, text, categories, arrays of categories and nested fields into the format a model can eat.
2. **Search and tune**, powered by two technologies:
   - **Neural architecture search (NAS)**: brute force with taste. Tries many architectures and hyperparameter combinations, compares scores, keeps the winners. The thing a data scientist does by hand over weeks.
   - **Transfer learning**: do not start from zero. Reuse a model already trained on a huge dataset and adapt it to your small one. This is why AutoML works with less data and less compute, and reaches higher accuracy faster.
3. **Ensemble**: AutoML does **not** ship one model. It takes the top N from the search (typically ~10, depending on training budget) and averages their predictions. Multiple models voting beats the single best model.
4. **Prediction.**

**Transfer learning, the sticky example:** 800 photos of damaged equipment, far too few to
train an image model from scratch. Transfer learning starts from a model that already knows
edges, metal, rust and cracks, typically trained on **ImageNet**, 14 million labelled
everyday photos. Your 800 only have to teach it *your* categories. You are not teaching it to
see; you are teaching it your vocabulary. Same reason an LLM fine-tunes on a small company
dataset.

**Where does that base model come from?** In AutoML you never see or choose it, that is the
no-code trade. The generic term for a collection of pre-trained models is a **model zoo**;
the set of architectures NAS searches is the **search space**. The browsable catalogue you
*can* pick from yourself is **Model Garden** (or Hugging Face / TensorFlow Hub outside Google).

⚠️ **AutoML is not "one level better than a generative model", wrong axis.** AutoML builds
*predictive* models: output is a number or a label with a confidence score you can threshold,
average and chart. Generative models produce content. Ask Gemini "is this rust?" and you get
prose with no score; ask AutoML to write the maintenance report and it cannot. AutoML is
**narrower and sharper**, not higher.

### AutoML use case, with concrete names

Predict which support tickets will breach SLA.

| | |
|---|---|
| Data | 40,000 closed help desk tickets exported from a ticketing system to BigQuery |
| Table | `support.tickets_closed` |
| Label | `breached_sla` (TRUE/FALSE, known for every past ticket) |
| Features | `priority`, `category`, `assignment_group`, `caller_company`, `opened_hour`, `reopen_count`, `description_length` |
| Model | AutoML Tabular → Classification |
| Output | per new ticket, a probability: `0.87 likely to breach` |

Clicks: Vertex AI → Datasets → Create → Tabular → point at the BigQuery table → Train new
model → AutoML → target column `breached_sla` → training budget (e.g. 2 node-hours) → Train.
Google handles encoding, architecture search, tuning and ensembling; you get an evaluation
page (AUC, precision, recall, confusion matrix) and an endpoint.

**Worth:** a queue sorted by breach risk at 8am so the support team works the dangerous ones first. Ranking
tool, not decision tool, nothing auto-closes.

## Pre-trained APIs

**"Pre-trained API" is sloppy shorthand.** An API is a doorway, not a model, you cannot
train a doorway. Read it as **pre-trained model, reachable by API**. The training happened at
Google, on Google's data, once. You call the door.

**The analogy:** an API is a wall outlet. Know which plug fits and what voltage; ignore how
the grid works. Same for an API, know which method and what parameters, ignore the model
training and deployment behind it.

**The four-line shape** (Google AI for Python SDK; **SDK** = a library that wraps the raw HTTP
calls so they look like normal functions):

```python
genai.configure(api_key="YOUR_API_KEY")            # who are you
model = genai.GenerativeModel('gemini-2.5-flash')  # which model
response = model.generate_content("your prompt")   # send
print(response.text)                               # read
```

Same authenticate → call → read pattern as the agent code, minus the tools and the loop.

**The families:** generative AI APIs (Gemini, multimodal), ML APIs (train/monitor/tune), and
the older single-purpose ones, Speech, Vision, Document, Natural Language.

### Why the old single-purpose APIs still exist

Before LLMs, each job needed its own model, so Google shipped one API per job. Gemini is
multimodal and general, so it can do all of them in one call, and many
**can be replaced by** the Gemini APIs.

Same input, both paths. *"Your service has been terrible all week."*

- **Natural Language API** → `sentiment: { score: -0.8, magnitude: 0.9 }`
- **Gemini** → *"The customer is expressing strong dissatisfaction with the service..."*

That `-0.8` is a **number**. You can store it in a column, average it by month, threshold it
at -0.5 to auto-escalate, and chart it. You cannot `AVG()` a paragraph.

Three reasons the specialised APIs survive:

- **Structured output**: guaranteed shape every call, no parsing or prompt-wrangling.
- **Cost**: a purpose-built model is far cheaper per call. At a million emails that gap is the budget.
- **Speed**: smaller model, lower latency.

**The rule:** one fixed question at high volume → specialised API. Open-ended or mixed
questions → Gemini. Same ease-vs-flexibility trade, fourth appearance in these notes.

## Custom training

For when complete control over model architecture, frameworks and training logic is
essential, beyond what AutoML's automation can express.

**Decision 1: the container.** A container is a packaged environment, OS, Python, every
library, frozen together so it runs identically anywhere.

- **Pre-built container**: a furnished kitchen. Python, TensorFlow, PyTorch already installed. Take it and cook.
- **Custom container**: an empty room. You define environment, machine type and disks yourself.

**Decision 2: where you write it**: **Workbench** (a hosted Jupyter notebook covering the
whole data science workflow, can run SQL against BigQuery) or **Colab Enterprise**.

**ML libraries**: pre-written code so you do not start from scratch: **TensorFlow**,
**scikit-learn**, **PyTorch**, all open source.

**TensorFlow's layers, bottom to top:** hardware (CPU / GPU / TPU) → low-level TF APIs (write
your own ops in C++, call core numeric functions from Python) → model libraries (neural
network layers, evaluation metrics) → **high-level APIs like Keras**, which hide the building
details and deploy training automatically. Keras is the layer you actually touch. Agent
Platform hosts TensorFlow at every level as a managed service.

### The three-step Keras pattern, memorise this shape

```python
model = tf.keras.Sequential([                       # 1. BUILD: stack the layers
    tf.keras.layers.Dense(64, activation='relu'),   #    64 neurons
    tf.keras.layers.Dense(32, activation='relu'),   #    32 neurons
    tf.keras.layers.Dense(1)                        #    output: 1 number
])
model.compile(loss=..., optimizer=...)              # 2. COMPILE: how to judge / how to improve
model.fit(x_train, y_train, epochs=10)              # 3. FIT: train it
```

- **`Sequential`** = layers in a straight line, each feeding the next, no branching.
- **Neuron** = takes incoming numbers, multiplies each by a weight, sums them, passes the result on. The weights are what training adjusts.
- **loss function** = how wrong an answer is; the number being minimised.
- **optimizer** = the strategy for adjusting weights to reduce that loss.
- **epochs** = how many full passes over *all* the training data.

**Epoch vs iteration:** an **epoch** is one full pass over the whole dataset; an
**iteration/step** is one batch within it. 10,000 rows with `batch_size=100` → 100 iterations
per epoch. Too few epochs = undertrained; too many = memorising the rows instead of the
pattern (overfitting). It is a hyperparameter.

### How do you know which architecture to use?

**You mostly do not, which is the entire argument for AutoML.**

1. **The output layer is forced** by the question: predicting a number → 1 neuron, no activation; yes/no → 1 neuron, sigmoid; 5 categories → 5 neurons, softmax.
2. **The middle is convention plus trial**: start 2-3 layers, powers of two, narrowing (64 → 32). Nobody derived that; it is what tends to work.
3. **Mostly you copy** whatever won on a similar problem, from a paper or a previous project. Original architectures are research, not the day job.

The real workflow is guess → train → evaluate → adjust → repeat, which is what a data
scientist spends weeks on. **Neural architecture search automates exactly that loop.** Reach
for custom training only when you have reason to believe *your* structure beats what a search
would find, not because you want to feel in control.

**JAX** gets a name-drop: newer high-performance numerical computation library, flexible,
research-heavy. Recognise the name.

## Hands-on: calling the Natural Language API for entities and sentiment

**The goal:** turn free text into structured data using a model you did not train. The thing
being learned is the *shape* of calling a pre-trained API in production, key, request, call,
parse, not the linguistics.

**The loop, repeated five times:** `edit request.json → curl sends it to Google → read the answer`.
Only two things change between steps: the sentence, and the method name in the URL.

### Setup

1. Console → APIs & Services → Credentials → Create credentials → API key, restricted to **Cloud Natural Language API**, Application restrictions **None**.
2. Compute Engine → VM instances → **SSH** on `linux-instance`.
3. In the SSH window: `export API_KEY=<key>` (prints nothing; check with `echo $API_KEY`).

⚠️ **Gotcha not in the written steps: SSH-in-browser failed with "Connection via Cloud
Identity-Aware Proxy Failed, code 4033, not authorized".** The fix is the dialog's own
**"Retry without Cloud Identity-Aware Proxy"** button, which connects to the VM's external IP
instead. Fallback is `gcloud compute ssh linux-instance --zone=us-central1-b` from Cloud
Shell. Written steps assume SSH just works. It often does not.

### The five methods, the endpoint name is the only thing that changes

| Method | What it breaks the text into |
|---|---|
| `analyzeEntities` | things, people, places, works of art |
| `analyzeSentiment` | sentences, each scored |
| `analyzeEntitySentiment` | things, each scored |
| `analyzeSyntax` | individual words (tokens) |
| `classifyText` | one topic tag for the whole document |

You are not telling the model *how* to chop the text. You are picking which question to ask,
and each question has its own fixed answer shape. Everything else in the curl line, key,
`-X POST`, the header, `@request.json`: is plumbing and never changes.

### The curl command, decoded once

```bash
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > result.json
```

| Piece | Job |
|---|---|
| the URL | which method, plus the key |
| `-X POST` | sending data, not requesting a page |
| `-H "Content-Type: application/json"` | telling the server the body is JSON |
| `--data-binary @request.json` | **`@` = read the body from this file** |
| `-s` | silent, hides the progress bar |
| `> result.json` | save the reply instead of printing it |
| `\` | one command split across lines |

### Tooling concepts this exercise actually teaches

- **`nano`** is a text editor that runs inside the terminal, because a Linux VM has no Notepad and no desktop. `nano request.json` opens that file (creating it if new); bare `nano` opens an unnamed buffer and asks for the filename on save. `Ctrl+O` save, `Ctrl+X` exit, `Ctrl+K` cut a line. Alternatives: `vim` (harder, `:q!` to escape), or `cat > file` + paste + `Ctrl+D`.
- **Why a file at all?** Because curl's `--data-binary @request.json` reads the body from disk. Cleaner than cramming quoted JSON onto a command line, and you reopen the same file five times changing only the `content`.
- **`curl` is a program, not a language and not bash syntax.** It lives at `/usr/bin/curl`; bash just launches it, same as it launches `nano`. Bash syntax is `export`, `if`, `|`, `>`, `$VAR`: built into the shell. Closest familiar thing: `pac`. You run it with flags; there is no such thing as "curl code".
- **The split on screen:** bash gets you to the file, JSON is what is inside it, curl sends it, `cat` reads the reply.
- **Nothing ran on the VM.** No model downloaded, no AI on that machine. curl opened a connection to `language.googleapis.com`, Google's server checked the key, ran the model, sent JSON back. The same curl would work from a laptop. **That is what "pre-trained API" means physically**: the model never comes to you, your text goes to it.
- **The API key is a ticket, not a worker.** It proves you are allowed in and says who gets billed. It does no analysis.

### Step 3, entities, and what the response teaches

Input: *"Joanne Rowling, who writes under the pen names J. K. Rowling and Robert Galbraith,
is a British novelist and screenwriter who wrote the Harry Potter fantasy series."*

| Field | Means |
|---|---|
| `name` | the thing found |
| `type` | PERSON, LOCATION, WORK_OF_ART, ORGANIZATION, CONSUMER_GOOD, OTHER |
| `salience` | 0-1, how central it is to the text |
| `wikipedia_url` / `mid` | the real-world entity in Google's Knowledge Graph; `mid` is its permanent ID |
| `beginOffset` | character position in the input |
| `mentions` | every place that entity appears |

**The impressive bit:** "Joanne Rowling", "Rowling", **"Robert Galbraith"** and "novelist"
all appear in the `mentions` list of *one* entity. The model resolved a pen name and a job
description to the same person, **coreference resolution**, which string matching cannot do.

**The useful bit:** salience ranked Rowling **0.798** against Harry Potter **0.015**. The
sentence *mentions* Harry Potter but is *about* Rowling, and the number says so. Run that over
10,000 news articles and you can answer "which of these are actually about our company".

**Not infallible:** "British" came back `LOCATION` (it is a nationality) and "screenwriter"
came back `PERSON`.

### Step 4, document sentiment, and why two numbers

Input: *"Harry Potter is the best book. I think everyone should read it."*
Result: `documentSentiment: { score: 0.9, magnitude: 1.9 }`, plus per-sentence scores.

- **`score`** −1.0 to +1.0, how positive or negative.
- **`magnitude`** 0 to infinity, total weight of emotion, regardless of direction.

**Why both:** a long review that is half glowing and half furious averages to `score ≈ 0`,
identical to a flat, indifferent one. `magnitude` separates them: the mixed review scores
high, the indifferent one near zero. **Score without magnitude misleads.**

⚠️ The exercise's prose claims 0.7 and 0.1 for those two sentences; the actual output is 0.9 and
0.9. The written steps do not match their own example. Ignore it.

### Step 5, entity sentiment, the one that earns its keep

Input: *"I liked the sushi but the service was terrible."*
Result: `sushi` → score 0, `service` → score −0.7, each with its own `salience` and `mentions`.

Run Step 4's method on that sentence and the positive and negative cancel to roughly 0, and
you would conclude the customer was indifferent. They were not. Across 500 reviews this is
the difference between "our rating is 3.2" and "the food is fine, fix the service".

### Step 6, syntax, and the parse tree

Input: *"Joanne Rowling is a British novelist, screenwriter and film producer."*
Returns one block **per token** (word), not per sentence.

- **`partOfSpeech.tag`**: NOUN, VERB, ADJ, DET, CONJ, PUNCT. A closed list of ~13 values Google published; the model cannot invent a new one.
- **`dependencyEdge`**: `headTokenIndex` (which token this one attaches to) plus a `label`. Chain them and you get the grammar tree.
- **`lemma`**: the base form. run / runs / ran / running all lemma to `run`. Useful for counting mentions over time: searching raw text for "run" misses "running".

**How it knows:** trained on huge corpora that linguists had already tagged by hand, millions
of examples of "in this position, this word is a past-tense verb".

**Why so many `UNKNOWN` fields** (`GENDER_UNKNOWN`, `VOICE_UNKNOWN`): the response schema is
shared across every supported language. English verbs carry no grammatical gender, so the
field comes back UNKNOWN. **It does not mean the model could not tell, it means the concept
does not apply to this language.** Run Japanese and fields blank here come back populated.

**Reading the dependency parse tree** (the drawing is just `headTokenIndex` + `label`
rendered as arrows; the model outputs the data, not the picture):

| Label | Meaning in the example |
|---|---|
| `root` | **is**: the main verb everything hangs off |
| `nsubj` | **Rowling**: the subject |
| `nn` | **Joanne**: noun compound modifying "Rowling" |
| `amod` | **British**: adjective modifying "novelist" |
| `attr` | **novelist**: what Rowling *is* |
| `conj` / `cc` | **screenwriter**, **and**, **producer**: the joined list |
| `p` | punctuation |

**Why anyone cares:** the tree says *what goes with what*. "The battery is good but the screen
is terrible" has two adjectives and two nouns floating free until the tree pins `good→battery`
and `terrible→screen`. **That is how entity sentiment in Step 5 knew which opinion belonged
to the sushi.**

### Step 7, multilingual

Input: `日本のグーグルのオフィスは、東京の六本木ヒルズにあります`, with **no `language` field
in the request**. Response comes back `"language": "ja"`.

Two things demonstrated:

1. **Auto-detection.** Mostly from the characters: `の`, `は`, `に`, `ます` are **hiragana**, used only by Japanese, one of those settles it. Where script is ambiguous (kanji is shared with Chinese, or any Latin-alphabet language), a small statistical language-detection model trained on ~100 languages scores which one the sequences most resemble, and runs *before* the main model. Declaring `"language"` yourself skips it, slightly faster and safer on very short text.
2. **Entities resolve across languages.** `日本` → `LOCATION` with `mid: /m/03_3d` and the Wikipedia link for **Japan**; `グーグル` → `ORGANIZATION` → **Google**. The `mid` is the same ID the English word "Japan" returns.

**The point: an entity is not a string, it is a thing.** "Japan", "日本" and "Japon" map to one
Knowledge Graph ID, so reviews in eight languages group by entity without translating first.

### Three closing questions

1. **What problem did this solve, and what without it?** Free text is unqueryable. Without it you either read 10,000 reviews by hand or build and train your own NLP model. With it, prose becomes rows you can `GROUP BY` and chart.
2. **What would I change for a different job?** Swap the method name in the URL, that is the whole change. For a file instead of inline text, replace `content` with `gcsContentUri` pointing at Cloud Storage. For another language, nothing: it detects.
3. **When would I choose this over something simpler?** Over Gemini: when you need a *number* every time, structured, cheap, fast, at volume. Over training your own: when you have no labelled data. Against it: when the classification is domain-specific (is this support ticket a false alarm?), because a general sentiment model knows nothing about your domain, that is AutoML or BigQuery ML territory.

---

# The ML workflow on Agent Platform, end to end

## The three stages

Restaurant analogy, used throughout this part:

| Stage | Steps | Cooking |
|---|---|---|
| **Data preparation** | upload data, feature engineering | prep the ingredients |
| **Model development** | train ↔ evaluate, iteratively | experiment with recipes |
| **Model serving** | deploy, monitor | serve the meal |

**Model management** runs across all three, handling the underlying infrastructure so data
scientists work on *what* rather than *how*.

**It is not linear, it is a loop.** Training reveals you need better features, so you go back
to the data. Monitoring shows accuracy sliding, so you retrain.

**Data drift** is why the loop never ends: the world changes, the model does not. A model
trained on 2025 tickets is quietly predicting on a 2026 world after a team reorg, accuracy
decays with nobody touching anything. Monitoring catches it; MLOps automates the response.

**Two ways to run the workflow:** AutoML through the UI (no code), or **Pipelines** written in
Workbench / Colab Enterprise. Pipelines is a toolkit of pre-built SDKs, the building blocks.

## Stage 1, Data preparation

**Upload** from Cloud Storage, BigQuery, or a local machine. AutoML is mainly tabular now,
and you choose an **objective**: regression (a number), classification (a category),
forecasting (a number over time).

**A feature** is one factor contributing to the prediction, an independent variable in
statistics, a column in a table. **Feature engineering** is processing raw data into
something the model can use.

### Feature engineering, worked

Raw ticketing-system column `opened_at = 2026-03-14 02:17:00` is useless as-is; a timestamp is just
a big number. Engineered:

| Raw | Engineered feature | Why the model needs it |
|---|---|---|
| `opened_at` | `opened_hour = 2`, `is_weekend`, `is_after_hours` | now it can learn "2am tickets breach more" |
| `caller_email` | `caller_domain = "acme.com"` | the person is noise, the company is the pattern |
| `description` (prose) | `description_length`, `contains_error_code` | a model cannot read prose, it can read numbers about prose |
| `opened_at` + `resolved_at` | `resolution_hours` | the gap is the signal, not either timestamp |
| last 10 tickets from this caller | `caller_breach_rate_90d` | history the current row does not contain |

**The last kind is where the predictive power usually lives**: the feature does not exist in
any column, you compute it by looking across *other* rows.

**Two kinds of feature engineering:** *selection* (deciding which existing columns matter) and
*creation* (columns that did not exist until you wrote them). Both count; the second is the work.

### Data leakage, the trap

**Leakage = using a fact you would only know afterwards.**

Predicting "will this ticket breach its 24h SLA?" and including `resolution_hours` as a
feature: training looks 99% accurate, because if resolution took 50 hours and the SLA is 24,
the answer is sitting right there. The model learned `50 > 24` and nothing else. In production
the ticket just opened, `resolution_hours` is blank, and the model falls over.

**Dumbest version:** predicting who wins a football match using the final score. Perfect in
testing, useless before kickoff.

**The check, on every feature:** *at the moment I press predict, do I actually have this
number?* If it only exists after the event, drop it. Anything touching the outcome,
resolution date, closure code, final state, is out.

**Feature importance will flag it**: one bar dwarfing everything else usually means leakage.

### Feature Store, and the online/offline split

A central repository to manage, serve and share features. It aggregates from BigQuery sources
and serves them two ways:

- **Offline** = batch, millions of rows, nobody waiting. **Training is always offline**, even for a model that will serve online. Also batch scoring, backfills, reports.
- **Online** = real-time, one row, low latency, something is blocked waiting.

**The deciding question: is anything blocked waiting for this answer?** No → offline, run it
nightly. Yes, within a second → online.

**Same feature, both paths, worked on `caller_breach_rate_90d`:**

| | Offline | Online |
|---|---|---|
| When | training, or the 02:00 batch job | the instant a ticket is submitted |
| How | BigQuery scans 40,000 rows | key-value lookup by caller, ~5 ms |
| Budget | 20 minutes | under 50 ms |
| Needs | a scheduled query | an online store (Bigtable), a streaming job keeping it fresh, an always-on endpoint |

**Training-serving skew**: the reason the Feature Store exists. Those two paths are built by
different engines. If the training SQL says `closed_at < opened_at` (strictly before) and the
streaming job includes the same minute, the two definitions drift. **No error appears
anywhere**, offline evaluation metrics still look great, and live predictions are subtly
wrong. One definition feeding both paths is the fix.

**Power BI parallel:** a measure defined once in `_Measures` and used by 10 visuals, versus
the same calculation retyped into 10 visuals. One stays consistent; the other slowly stops
agreeing with itself.

**Four stated benefits:** shareable (training *and* serving, the skew fix), reusable (across
models and teams), scalable (auto-scales for low latency), easy to use. It stores **both** the
definition (feature groups and features, pointing at a BigQuery source) and the values (a
feature view copies current values into the online store).

**Also ready for gen AI**: it can manage and serve **embeddings** and retrieve similar items
in real time.

### The honest recommendation for ticket-queue work

You do not need the online path. A risk-ranked queue at 8am is as useful as one computed in
70 ms, and it is a scheduled query instead of a streaming pipeline plus an always-on service.
**Online earns its cost only when a decision is genuinely blocked**: declining a card,
gating a login. Not when a human reads the result over coffee.

## Stage 2, Model development

Setup: dataset → **objective** → AutoML or custom → **target column** + which features
participate → budget → Start training.

### The confusion matrix

**Step 0, always: declare which class is "positive."** TP/FP/TN/FN mean nothing until you do,
and mixing framings mid-read is exactly what tangles people up.

**The naming rule, read the name backwards:**

- **Second word** (Positive / Negative) = **what the model SAID**
- **First word** (True / False) = **was the model RIGHT?**

So "False Positive" = said positive, was wrong. **A miss is always a False Negative.**

With **positive = default**, the exercise's matrix:

| | actually default | actually repay |
|---|---|---|
| **predicted default** | TP **87%** | FP **0%** |
| **predicted repay** | FN **13%** | TN **100%** |

Diagonal = correct. **The two cells starting with False are your errors.**

**Where the numbers come from:** the **test set**: rows held back during training (AutoML
splits roughly 80/10/10 train/validate/test). Feed each held-back row through the finished
model, compare to the known answer, tally the four boxes, convert to percentages **within
each true-label column** (which is why each column sums to 100%). Scoring on rows it trained on
would be grading a student on questions they already saw.

⚠️ **The exercise's prose describes its own 13% backwards** ("13% of good borrowers identified as
defaulters"). The matrix says the opposite: 13% of actual defaulters were predicted as
repayers. Read the matrix, not the sentence.

### Precision and recall

- **Recall** = of all real positives, how many did I catch? `TP / (TP + FN)`
- **Precision** = of everything I flagged, how many were right? `TP / (TP + FP)`

**The spam example:** maximise recall → catch every spam, accept real mail in the junk folder.
Maximise precision → only flag certain spam, accept some spam reaching the inbox. You cannot
have both.

### The threshold, the most important idea here

**The model does not output yes/no. It outputs a probability.** The threshold is the line you
draw on that 0 to 1 ruler to turn a probability into a decision.

| Person | Model says | Truth |
|---|---|---|
| A | 0.05 | repaid |
| B | 0.22 | repaid |
| C | 0.35 | defaulted |
| D | 0.60 | defaulted |
| E | 0.90 | defaulted |

- **Threshold 0.5** → flag D, E. C is a **FN** (missed).
- **Threshold 0.3** → flag C, D, E. **Zero FN.** C crossed the line.
- **Threshold 0.2** → flag B, C, D, E. B is now a **FP**: you bought that last defaulter by refusing a good customer.

**Nothing about the model changed. 0.35 is still 0.35.** The counts move because people cross
a line you moved.

**Changing the threshold does NOT create a new model.** The model outputs the same probability
regardless. A new model means retraining, different features, data, budget. Model → a
probability (hours to change). Threshold → a decision (seconds to change). **The threshold is
the cheapest lever you have.**

### Choosing the threshold, cost arithmetic, not intuition

**Total cost = (FN count × cost of an FN) + (FP count × cost of an FP)**, where each count =
rate × the size of that true class. Only the two error cells appear; TP and TN cost nothing.

**The loan case.** FN = lent to someone who defaults ≈ $10,000. FP = refused someone who would
have repaid ≈ $1,500 of lost profit. Ratio ~6.7:1, so lower the threshold, trade a cheap
error for an expensive one.

Per 100 applicants (80 repay, 20 default), illustrative:

| Threshold | FN | FP | FN cost | FP cost | **Total** |
|---|---|---|---|---|---|
| 0.5 | 13% | 0% | $26,000 | $0 | **$26,000** |
| 0.3 | 5% | 4% | $10,000 | $4,800 | **$14,800** |
| 0.2 | 2% | 12% | $4,000 | $14,400 | **$18,400** |

**0.3 wins.** The U-shape is what you are always looking for: cost falls, bottoms out, rises.

**Comparing two models:** each at *its own* optimal threshold, compared on total cost, not
both at 0.5, and not on accuracy. A model can make *more* errors overall and still win by
making cheaper ones. **Counting errors and counting money give different answers, and only one
of them is the business's.**

**The step people skip:** nobody makes the call, so the model ships at the default 0.5.
**0.5 is not a neutral choice, it is an unexamined one**: and it is exactly what produced 10%
recall on the excellent BigQuery ML model back in the first hands-on exercise.

**Whose call:**

| Decision | Owner |
|---|---|
| What are we predicting, and what happens to the prediction? | **Product / PO** |
| What does each error type cost the business? | **Product / PO**, with ops and finance |
| Is the model good enough to ship? | **Product / PO** |
| Build the model, produce the precision-recall curve | ML engineer |
| Recommend a threshold, flag where the model is unreliable | ML engineer |

An ML engineer can say "at 0.2 you catch 80% and waste 300 escalations." They cannot say
whether 300 wasted escalations is acceptable. **This is the product owner's job**,
not building the model, but framing the cost of being wrong and owning the threshold.

### Reading the three evaluation charts

| Chart | Axes | Answers |
|---|---|---|
| **Precision-recall curve** | precision vs recall | how good is the trade-off overall |
| **ROC curve** | true positive rate vs **false positive rate** | is the model good at *ranking* |
| **Precision-recall by threshold** | threshold on x, precision and recall as two lines | **which threshold do I pick** |

**The ROC curve's blue line is a trail of dots, one per threshold**, joined up:

| Threshold | TPR | FPR | Lands |
|---|---|---|---|
| 1.0 | 0% | 0% | bottom-left (flag nobody) |
| 0.5 | 87% | 0% | far left, near top |
| 0.3 | 95% | 4% | slightly right, higher |
| 0.2 | 98% | 12% | further right |
| 0.0 | 100% | 100% | top-right (flag everyone) |

A curve hugging the **top-left** is excellent. The **diagonal** is coin-flipping, every
defaulter caught costs one wrongly flagged good customer. **ROC AUC** is the area under it:
0.5 random, 1.0 perfect. The exercise model scored **0.986**.

**Why the third chart is the practical one:** ROC and the PR curve plot the metrics *against
each other*, so the threshold is invisible, you cannot tell which point is 0.3. The third
puts threshold on the x-axis, so you can read a number off it.

**But it still cannot pick for you.** It narrows the range (flat band ≈ 0.1 to 0.85 here means
almost any threshold works; past ~0.9 recall falls off a cliff) and rules out the bad regions.
Choosing *within* the band needs the cost numbers, which are not in the chart, the data, or
the model. **Chart narrows, cost decides.**

A wide flat band means the model separates the classes so well that the threshold barely
matters. On a weaker model the lines cross steeply and every 0.05 swings the numbers hard.

### Feature importance and Explainable AI

A bar chart showing how much each feature contributed. Longer bar = more important.

**How it is computed:** shuffle one feature's values randomly, re-score the model, and measure
how much accuracy drops. Big drop = the model was leaning on it. (Vertex uses Shapley values,
a refined version of the same idea.)

**Two questions to ask of the chart, and only two:**

1. **Anything suspiciously dominant?** Usually leakage.
2. **Anything near zero?** Drop it next time, cost with no benefit.

**Caveat:** importance is not causation. Two correlated features split the credit and both
look moderate, when either alone would look strong.

**Explainable AI is the umbrella; feature importance is one tool in it:**

| Tool | Scope | Answers |
|---|---|---|
| **Feature importance** | global | what the *model* relies on overall |
| **Feature attributions** | local, per prediction | why *this one decision* came out this way |
| **Example-based explanations** | local | which training rows are most similar to this case |
| **What-If tool** | interactive | change an input, watch the prediction move |

**Global tells you about the model; local tells you about a decision.** "Age matters most in
our lending model" is global. "You were declined because your loan-to-income ratio contributed
−0.4" is local, and that is what goes in a letter to a declined applicant.

**Feature attribution, what it actually returns** (ticked when creating the endpoint):

```json
"baselineOutputValue": 0.15,     // the average applicant
"instanceOutputValue": 0.28,     // this applicant
"featureAttributions": { "loan": 0.24, "income": -0.09, "age": -0.02 }
```

The attributions reconcile to the gap: 0.24 − 0.09 − 0.02 ≈ +0.13, which is 0.15 → 0.28.
`loan` pushed risk up, `income` pulled it back. Costs latency and money on **every** request.

**And the compliance flag worth raising out loud:** `age` being the top predictor of
creditworthiness is a problem in real lending, age is a protected characteristic in most
jurisdictions. Explainable AI exists so you catch that before a regulator does.

## Stage 3, Model serving

**Three ways to use a trained model:**

| | What happens | When |
|---|---|---|
| **Endpoint (online)** | the model sits running on a server, waiting. One row in, answer in ms | someone is waiting |
| **Batch** | hand it a whole table, it writes answers to another table, then stops | nobody is waiting |
| **Edge** | the model is copied onto a device and runs there, no internet | cannot reach the internet, or cannot afford the round trip |

**The detail worth calling out: batch prediction needs no endpoint.** People deploy one
out of habit and pay for an idle server nothing calls. An endpoint bills per *hour running*,
not per prediction.

**Edge, the standard example:** object detection on a camera in a manufacturing plant. 30
parts a second, a cloud round trip would back the line up. Three reasons for edge: latency,
privacy (data never leaves the building), offline capability.

**Security tools, mapped to the same split:**

- **An endpoint protection sensor, genuinely edge.** The sensor carries a model on each laptop, scoring processes as they execute, and must work on a plane with no connection. Also runs heavier cloud-side analytics, both, not either.
- **A vulnerability scanner, cloud.** The agent collects inventory, prioritisation is computed server-side. Batch-shaped, nothing time-critical.
- **A SIEM, server-side by definition.** Correlation needs events from many sources; an edge device only sees itself, so "this user logged in from two countries in an hour" can never be an edge decision.

**The pattern: edge when the decision is about *this one device, right now*. Central when the
decision needs context from everywhere else.**

**Model monitoring** compares live request distributions against the training baseline, which
is why you must hand it the training dataset when deploying. It produces per-feature
distribution charts, a drift score, and email alerts.

⚠️ **It can only watch the inputs.** It never learns whether the loan was repaid, so it cannot
tell you accuracy dropped, only that conditions changed. Those are different things and
people conflate them.

## MLOps and workflow automation

**MLOps = DevOps applied to ML.** The extra problem: in normal software only the *code*
changes; in ML the **data** changes too, so a correct system becomes wrong with nobody
touching it. Hence **CI / CD / CT**: **continuous training** is the ML-only addition.

**Vertex AI Pipelines** is the backbone. Supports **KFP** (Kubeflow Pipelines) and **TFX**
(TensorFlow Extended). Already on TensorFlow with terabytes of structured data → TFX.
Otherwise → KFP.

**A component** is one step, packaged so it can run on its own. Three things inside:

1. **The code**: what it does.
2. **Inputs and outputs**: declared explicitly.
3. **Its environment**: the container it needs.

**Component vs script:**

| | Script | Component |
|---|---|---|
| Interface | reads and writes whatever it likes | inputs and outputs declared up front |
| Environment | assumes your machine has the libraries | ships its own container |
| Who runs it | you, by typing `python thing.py` | the orchestrator |
| Failure | dies, you notice later | retried, logged, recorded |
| Reuse | copy-paste | imported by name |

**Why the declaration matters:** because inputs and outputs are declared, the orchestrator can
*reason* about the step, run two in parallel when neither depends on the other, skip one
whose inputs have not changed (caching), retry a failure, and draw the dependency graph. **A
script is a black box; the orchestrator can only run it and hope.**

A script is a note to yourself; a component is a form with labelled fields. They usually start
as the same code, phase 1 is literally wrapping working scripts.

### The bean-classifier example

```python
@component(packages_to_install=["google-cloud-aiplatform"])
def classification_model_eval_metrics(
    model: Input[Model],
    threshold: float,
) -> NamedTuple("out", [("deploy", str)]):
    metrics = model.get_evaluation()
    return ("true" if metrics.auc > threshold else "false",)

@pipeline()
def pipeline():
    dataset = gcc_aip.TabularDatasetCreateOp(bq_source="...")
    model   = gcc_aip.AutoMLTabularTrainingJobRunOp(dataset=dataset.output)
    check   = classification_model_eval_metrics(model=model.output, threshold=0.85)
    with dsl.If(check.outputs["deploy"] == "true"):
        endpoint = gcc_aip.EndpointCreateOp()
        gcc_aip.ModelDeployOp(model=model.output, endpoint=endpoint.output)
```

- `@component` / `@pipeline`: **decorators**, tags on the function below.
- `gcc_aip`: alias for `google_cloud_pipeline_components`. Anything with that prefix is prebuilt; `classification_model_eval_metrics` has no prefix because it is custom.
- **`.output` is the wiring.** `dataset.output` feeding the next step is how components chain, and how Google works out what can run in parallel.
- **Running this Python trains nothing.** It builds a definition, a graph. You compile it to JSON and submit that to Vertex. (Power Platform parallel: writing the flow definition, not clicking Run.)

**The custom component is the point.** `classification_model_eval_metrics` compares the fresh
model's AUC to a threshold, **above it deploy, below it retrain.** That single if-statement is
what makes it MLOps rather than a script: the pipeline decides whether the model is good
enough, with no human looking.

**Prebuilt components used:** `TabularDatasetCreateOp`, `AutoMLTabularTrainingJobRunOp`,
`EndpointCreateOp`, `ModelDeployOp`. **Check the prebuilt list before writing a custom one.**

**Single responsibility matters:** a component that only creates a dataset is reusable by every
pipeline you write. One that creates *and* trains *and* deploys is reusable by nothing.

### Reading the pipeline graph in the console

Vertex draws the DAG automatically from the `.output` chaining. **Two kinds of box:** a blue
cube is a **component** (a step that runs, with its container listed beneath), everything else
is an **artifact** (the thing produced, dataset, model, metrics, endpoint). It alternates:
step → thing it made → next step.

**The dashed box is the conditional.** Everything inside runs only if the check passed; a
model below threshold means that whole section is skipped.

**What the view is for:** after a run each box is green or red, click a failed one for logs.
And because artifacts are tracked, you can click any model and ask which dataset and which run
produced it, **lineage**, which is why auditors like pipelines and dislike notebooks.

**"Model" in that graph means your own trained artifact, a file of learned numbers**: not
Gemini. Three things get called "model": a foundation model (Google's, shared), your trained
model (weights learned from your data), and an architecture (a design, not a file). The
concrete check is *what does it output*: Gemini outputs text; the bean model outputs one of 7
bean types with a confidence score.

### The three phases of ML automation

- **Phase 0**: no MLOps. GUI workflow, AutoML by hand. **Critical, not a failure**: you build the end-to-end workflow manually before automating it.
- **Phase 1**: build individual components with the Pipelines SDK.
- **Phase 2**: integrate components into one workflow, achieving CI / CT / CD.

## Hands-on: a no-code AutoML model that scores loan default risk

**Goal:** walk the full ML workflow once, end to end, with zero code. Not the loan model,
the *shape*.

**Dataset:** `LoanRisk.csv` (a public sample file in Cloud Storage), 2,050 rows, 5 columns. **AutoML requires at
least 1,000 rows.**

| Column | Role |
|---|---|
| `Default` | **target**: 0 repaid, 1 defaulted |
| `ClientID` | **excluded**: an arbitrary label cannot cause a default; leave it in and the model may find fake patterns |
| `age`, `income`, `loan` | the 3 features actually trained on |

Three features, three bars on the feature importance chart. 97.9% precision from three columns,
because the relationship between income, loan size and default is genuinely strong.

**The five steps:**

| Step | What happens |
|---|---|
| 1 | create dataset, import CSV |
| 2 | train (Classification, target `Default`, exclude `ClientID`, budget 1, early stopping on), ~1 hour |
| 3 | evaluate |
| 4 | deploy |
| 5 | get predictions, against a model that was already deployed |

### Setup notes

- **Budget = node hours.** One node hour = one machine for one hour. It is a **spending cap on the architecture search**, not a duration. 1 is right for 2,050 rows and 3 features. **Early stopping** quits when scores stop improving and **refunds the unused budget**: always leave it on.
- **Transformation = Automatic** is AutoML deciding how to encode each column. That is the data-processing phase, exposed as a dropdown.
- UI drift: the written steps say "click the checkbox on the ClientID row to exclude it", it is now a **⊖ minus-circle icon** at the far right of the row, which flips to ⊕ to re-add. The "Total N feature columns" counter includes the target; ignore it.
- Creating an empty dataset is a metadata operation and should take seconds. Mine appeared to hang ~2 minutes with the Create button stuck spinning, the dataset had in fact been created. **Check the Datasets list before resubmitting.**
- **Machine type is what costs money.** `e2-standard-8` runs 24/7 from deploy until undeploy, whether or not anyone calls it. The exercise picks it for speed; in real life start smaller.

### Step 5, the prediction call

```bash
gcloud storage cp gs://<sample-bucket>/INPUT-JSON .
export INPUT_DATA_FILE="INPUT-JSON"
export PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format="value(projectNumber)")
export AUTOML_SERVICE="https://automl-proxy-$PROJECT_NUMBER.us-central1.run.app/v1"
curl -X POST -H "Content-Type: application/json" $AUTOML_SERVICE -d "@${INPUT_DATA_FILE}" -s | jq
```

All **Cloud Shell**: there is no VM in this exercise. **`| jq`** pipes the reply through a JSON
formatter so it prints indented. Same POST / header / `@file` shape as the Natural Language API
call, except this time the model is a trained classifier rather than Google's.

Input: age 40.77, income $44,964, loan $3,944. Output:

```json
"scores":  [0.9999980926513672, 0.000001897001311590429],
"classes": ["0", "1"]
```

99.9998% repay. **That is the raw model output, two probabilities, no yes/no.** Applying a
threshold to turn it into approve/decline is your job.

**Two details in the response worth noticing:**

- **`"modelDisplayName": "credit_risk_20211119212817"`**: the timestamp is the training date. **November 2021.** A five-year-old model, trained before two years of rate rises, still answering. **Data drift, sitting right there in the output.**
- **`deployedModelId` vs `model`**: two different IDs. The model is the artifact in the registry; the deployed model is that artifact running behind an endpoint. One model can be deployed several times, which is how traffic splitting between versions works.

### What AutoML does not tell you

No architecture name, no list of which ~10 models it ensembled, no hyperparameters. You get the
metrics, feature importance, a deployable artifact, and the ability to export it. **You cannot
reproduce it, tweak one layer, or explain a decision beyond "age contributed most."**

**Where that becomes a real problem: regulated lending.** "Explain why you declined this
applicant" is a legal requirement in many places, and "AutoML picked something" is not an
answer. That is a genuine reason to drop to BigQuery ML or custom training, where a plain
logistic regression gives you a defensible coefficient per feature.

### Honest assessment of the exercise

Thin. **The threshold-and-cost reasoning is the part worth keeping; the clicking was the
easy half.**

---

# How a machine learns

Worth doing: it is where the vocabulary already in use, hyperparameters,
epochs, learning rate, actually comes from. Everything is demonstrated on the most basic
network, the **ANN (Artificial Neural Network)**, also called a shallow neural network. DNN,
CNN, RNN and LLMs are all this loop with different wiring and vastly more weights.

Three layers: **input, hidden, output**. Each node is a neuron; the lines between them carry
**weights**.

## The worked example, with real numbers

**Question:** will this ticket breach its SLA?

| Number | What it is | Where it came from |
|---|---|---|
| **x1 = 0.75** | priority | the ticketing system said "2 - High", a CASE table maps it to 0.75 |
| **x2 = 0.50** | caller breach rate | 5 breaches / 10 past tickets, from the self-join SQL |
| **w1 = 0.4** | x1 to hidden | **random** at startup |
| **w2 = 0.7** | x2 to hidden | **random** at startup |
| **w3 = 0.6** | hidden to output | **random** at startup |
| **y = 1** | the truth | the ticket did breach |

**The distinction that matters most: inputs are data and never change. Weights start random
and are the only thing training touches.** When people say "the model" they mean the weights,
nothing else.

### Where the inputs come from, two kinds of feature

**A lookup, no arithmetic.** Priority is text in the ticketing system, so a human writes a translation
table. The numbers are a judgement, not a measurement, mapping P1 to 1.0 and P3 to 0.5
asserts P1 is exactly twice as urgent.

```sql
CASE priority
  WHEN '1 - Critical' THEN 1.0
  WHEN '2 - High'     THEN 0.75
  WHEN '3 - Moderate' THEN 0.5
  WHEN '4 - Low'      THEN 0.25
END AS priority_rank
```

**Min-max scaling, real arithmetic.** Subtract the floor, divide by the range. The minimum
lands on 0, the maximum on 1.

```
this ticket waited 45 min, fastest ever 5, slowest ever 245
scaled = (45 - 5) / (245 - 5) = 40 / 240 = 0.167
```

**Why force everything into 0 to 1, normalization.** Leave `ticket_age_minutes` raw at 4,320
next to a breach rate of 0.5 and the big feature drowns the other out; gradient descent then
takes wild steps on one and crawls on the other.

**Both count as feature engineering, but they are different jobs:** x1 was **encoded** (the
information existed, in the wrong form), x2 was **created** (no column in the ticketing system holds it).
x2 is where the predictive power lives, and its point-in-time guard
`p.closed_at < i.opened_at` is what keeps it out of leakage territory.

## Step 1, weighted sum at the hidden neuron

```
formula:   h_sum = (x1 * w1) + (x2 * w2)
numbers:   h_sum = (0.75 * 0.4) + (0.50 * 0.7) = 0.30 + 0.35
result:    h_sum = 0.65
```

Two facts about the ticket squashed into one number. The weights are the neuron's **opinion
about what matters**: right now w2 (0.7) beats w1 (0.4), so the network believes caller
history matters more than priority. That belief is random. Training finds out whether it holds.

*(A bias term `b` is normally added here. Ignored throughout.)*

## Step 2, activation at the hidden neuron (ReLU)

```
formula:   h = ReLU(h_sum) = max(0, h_sum)
numbers:   h = max(0, 0.65)
result:    h = 0.65        (already positive, unchanged)
```

**The whole ReLU rule: negative goes to 0, positive is left alone.** When it does fire, a
neuron **switches off** for that row and contributes nothing to the layer above:

```
h_sum = (0.25 * -0.6) + (0.1 * -0.3) = -0.18  ->  ReLU -> 0
```

### Why an activation function exists at all

Without one, the network is multiplication and addition all the way through, and any chain of
those collapses into a single straight line. With one input and two weights:

```
NO activation:   x = 2  -> h = 0.8  -> y = 0.48
                 x = 10 -> h = 4.0  -> y = 2.40
                 x = -2 -> h = -0.8 -> y = -0.48       every pair is just x * 0.24
```

Multiplying by 0.4 then by 0.6 is the same as multiplying by 0.24. **The hidden layer bought
nothing**: delete it, write one weight of 0.24, get identical answers forever. Twenty layers
would give exactly what zero layers give.

```
WITH ReLU:       x = 2  -> 0.8  -> 0.48
                 x = 10 -> 4.0  -> 2.40
                 x = -2 -> 0    -> 0        <- breaks the pattern
```

No single multiplier produces this, because the rule **changes** at zero. That is a **kink**.

**Why a kink is worth anything:** group backlog vs breach risk is flat from 0 to 20 open
tickets, then climbs hard past 20. A straight line has one slope for all values, so it either
over-predicts the quiet range or under-predicts the busy one. One ReLU neuron gives flat then
climbing. A hundred neurons kinking at different points trace any shape at all.
**That is what depth buys, and it only works because of the activation function.**

## Step 3, weighted sum at the output neuron

```
formula:   out_sum = h * w3
numbers:   out_sum = 0.65 * 0.6
result:    out_sum = 0.39
```

**The output neuron never sees the ticket.** It has no idea about priority or caller history,
it only sees 0.65, a summary the hidden layer invented. Each layer hands the next one its own
compressed version of reality.

## Step 4, activation at the output neuron (sigmoid)

```
formula:   yhat = 1 / (1 + e^(-out_sum))
numbers:   yhat = 1 / (1 + e^(-0.39)) = 1 / (1 + 0.677) = 1 / 1.677
result:    yhat = 0.60
```

ReLU would have handed back 0.39 unchanged, but it could just as easily hand back 4.2, and
4.2 is not an answer to a yes/no question. **Sigmoid squeezes any number into 0 to 1**: feed
it -50 and get ~0.00, feed it +50 and get ~1.00. The output is always readable as a probability.

**yhat = 0.60 means "60 percent chance this ticket breaches", and this is the number from the
loan exercise**, the one compared against 0.5, then 0.3, then 0.2. Moving the threshold never
touched it. That is why changing a threshold is not a new model.

**Sigmoid vs softmax:** sigmoid answers one yes/no. Four outcomes instead (breach / close
early / reassign / cancel) needs **softmax**, which outputs four numbers summing to 1.00.
Binary to sigmoid, multi-class to softmax. Different layers can use different functions:
ReLU hidden, sigmoid or softmax output is the standard pairing.

## Step 5, the cost: how wrong were we?

Predicted 0.60, actual 1. The naive answer is `1 - 0.60 = 0.40`, and nobody uses that for
classification. **Cross-entropy** is used instead:

```
formula:   loss = -ln(yhat)        when the true answer is 1
numbers:   loss = -ln(0.60)
result:    loss = 0.51
```

**Why not plain subtraction, look at the shape:**

| yhat | loss |
|---|---|
| 0.90 | **0.11**: confident and right, barely punished |
| 0.60 | 0.51 |
| 0.30 | 1.20 |
| 0.10 | 2.30 |
| 0.01 | **4.61**: confident and wrong, savaged |

Plain subtraction (1 minus the prediction) scores that last row 0.99, about ten times worse
than the 0.90 row's 0.10. Cross-entropy scores it about 40 times worse (4.61 against 0.11). **It punishes confident mistakes far harder than hesitant ones, so the network
learns to shut up when it does not know.**

**Loss vs cost, the distinction worth knowing:** loss is **one row**, cost is the **average across the
whole training set**. Cost is what training minimises, weights are never adjusted to fix one
ticket.

**If this were regression** (predicting hours to resolve rather than yes/no) the cost would be
**MSE, Mean Squared Error**: the average of (yhat - y) squared. Squared so being off by 10
hurts 100 times more than being off by 1, and so over- and under-shooting do not cancel out.

### Regression vs classification

| | Predicting | Example | Output | Cost function |
|---|---|---|---|---|
| **Regression** | a number | "resolves in 14.3 hours" | any value | **MSE** |
| **Classification** | a category | "breach / no breach" | a probability 0 to 1 | **cross-entropy** |

**The trap: logistic regression is classification, not regression.** Bad name, stuck since the
1950s. When you see "logistic regression", read "classification".

## Step 6, backpropagation: fix the weights

Two questions in order: **which direction**, then **how big a step**.

**1. The error**

```
error = yhat - y = 0.60 - 1 = -0.40          negative means we predicted too low
```

**2. Blame for w3**: it carried h into the output:

```
blame(w3) = error * h = -0.40 * 0.65 = -0.26
```

**3. Push the error back to the hidden layer**: it crosses w3, so it is scaled by w3:

```
error at hidden = error * w3 = -0.40 * 0.6 = -0.24
```

**4. Blame for w1 and w2**: each carried its own input:

```
blame(w1) = -0.24 * x1 = -0.24 * 0.75 = -0.18
blame(w2) = -0.24 * x2 = -0.24 * 0.50 = -0.12
```

**The whole algorithm in one line: a weight's blame is the error arriving at that weight times
the value that flowed through it.** Going backwards, the error is scaled by each weight it
crosses. Everything else is bookkeeping.

**w1 got more blame than w2 (0.18 vs 0.12) because its input was bigger**: 0.75 vs 0.50, so
it did more to cause the wrong answer. **Blame is proportional to contribution.**

**5. Update, using learning rate 0.1**

```
formula:   new = old - (learning rate * blame)

w1 = 0.4 - (0.1 * -0.18) = 0.418
w2 = 0.7 - (0.1 * -0.12) = 0.712
w3 = 0.6 - (0.1 * -0.26) = 0.626
```

All three nudged **up**, as the negative error predicted. Tiny moves, deliberately, this is
one ticket, and you do not rebuild a worldview over one ticket.

**The learning rate is the hill-walking step size**, and it is a **hyperparameter**: nobody
computes it, you try values:

- 0.001, correct direction, takes forever
- 0.1, steady
- 5.0, leap across the valley, overshoot, leap back, never land

**Gradient descent in two questions:** the **derivative** tells you which direction is downhill
(negative slope means go right, positive means go left), the **learning rate** tells you how
far to step.

## Step 7, iterate

Re-run the forward pass with the updated weights, same ticket:

```
h_sum   = (0.75 * 0.418) + (0.50 * 0.712) = 0.314 + 0.356 = 0.670
h       = ReLU(0.670) = 0.670
out_sum = 0.670 * 0.626 = 0.419
yhat    = 1 / (1 + e^(-0.419)) = 0.603
loss    = -ln(0.603) = 0.506
```

```
                   OLD weights 0.4/0.7/0.6     NEW weights 0.418/0.712/0.626
h_sum              0.650                       0.670
yhat               0.596                       0.603     <- closer to 1
loss               0.518                       0.506     <- smaller
```

**A full round trip moved the prediction 0.007.** It is supposed to be that small, thousands
of tickets each nudging the same weights by a hair, the noise cancels, the real signal
accumulates.

**What an epoch is:** one row through steps 1 to 6 is one update. **ALL 50,000 rows through
1 to 6 is one epoch.** Setting epochs = 100 means every ticket is seen 100 times, not that
100 tickets were seen. Epochs is a hyperparameter.

## When to stop, and what you actually end up with

**The goal is the final weights. Nothing else.** Everything in the seven steps existed only to
find them. The file saved to Model Registry is essentially those numbers.

**You do not stop where training cost bottoms out.** Drive it to zero and the model has
memorised 50,000 tickets and is useless on the 50,001st.

| epoch | training cost | validation cost | |
|---|---|---|---|
| 10 | 0.41 | 0.43 | both falling, learning |
| 50 | 0.22 | 0.24 | still good |
| 80 | 0.19 | **0.19** | best point |
| 100 | 0.14 | **0.26** | training still falling, validation **rising** |
| 120 | 0.09 | 0.35 | worse |

**Epoch 100 is the tell.** By training cost the model looks better; on rows it cannot see it is
worse. It has stopped learning "P1 tickets from repeat offenders breach" and started
memorising "ticket #0472 breached". That gap opening is the definition of **overfitting**.

**Validation cost is the same formula on different rows.** The 80/10/10 split in the loan exercise:

```
80%  training     weights get updated from these
10%  validation   scored every epoch, NEVER updates weights
10%  test         untouched until the very end
```

**Training cost only tells you how well it memorised. Validation cost tells you whether it
learned anything.** And the third split exists because validation was used to *choose* when to
stop, so that number is now slightly flattered, the test set has influenced nothing, so it is
the one unbiased score. Look at it once.

### What happens after the weights are frozen

1. **Freeze them.** Training stops, weights stop changing, artifact goes to Model Registry.
2. **Throw away the answers.** A new ticket has no y. Run steps 1 to 4 only, forward pass,
   no cost, no backprop.
3. **Apply the threshold.** yhat = 0.87 against a threshold of 0.3 means "this will breach".
4. **Hand it to something that acts**: a red flag in the support queue, a jumped sort order, a
   Teams alert, a Power BI card reading "23 tickets at breach risk today".

**The model outputs a probability. The threshold is a business decision. The action is
somebody's job.** Three separate things, and only the first is machine learning.

## Vocabulary recap

**Things in the network**

| Term | What it is | In the example |
|---|---|---|
| Input / feature | one fact about the row, as a number | x1=0.75, x2=0.50 |
| Neuron / node | one place a weighted sum happens | 2 in, 1 hidden, 1 out |
| Weight | how much one connection matters | w1, w2, w3 |
| Bias | constant added to the sum, shifts the threshold | ignored here |
| **Parameter** | **weights and biases, learned by the machine** | 0.4 became 0.418 |
| **Hyperparameter** | **set by a human before training** | learning rate, epochs, layers, neurons, activation choice |
| yhat vs y | predicted vs actual | 0.603 vs 1 |

**Activation functions**

| Function | Rule | Used where |
|---|---|---|
| ReLU | negative to 0, positive unchanged | hidden layers, the default |
| Sigmoid | squeezes anything into 0 to 1 | output, **binary** classification |
| Softmax | outputs summing to 1.00 | output, **multi-class** |
| tanh | sigmoid shifted to -1 to +1 | occasionally hidden layers |

**Learning machinery**

| Term | What it does |
|---|---|
| Loss function | error on **one row**: cross-entropy or MSE |
| Cost function | **average loss across the set**, what training minimises |
| Backpropagation | pushes the error backwards, assigns blame to each weight |
| Gradient descent | walking downhill on the cost surface to find the bottom |
| Derivative | which direction is downhill |
| Learning rate | how big a step to take |
| Epoch | one complete pass through **all** training rows |
| Convergence | cost stopped decreasing, stop training |
| Overfitting | kept going too long, memorised the training set |

**The three that get mixed up**

```
parameters          learned by the machine          weights, biases
hyperparameters     set by you before training      learning rate, epochs, layers
inference params    set at call time, gen-AI only   temperature, top-K, top-P
```

**Two one-liners worth keeping:**
**Loss is one row, cost is the batch.**
**Derivative = which way. Learning rate = how far.**

**And the callback: AutoML's expensive search is a machine trying hyperparameter
combinations**: layers, neurons, learning rate, epochs, training a model for each and
keeping the best. That is why it took an hour and why the recipe was never shown.

---

# Summary: the ML workflow

Nothing new. Two things to lock in.

**The restaurant analogy**, which Google reuses:

| Stage | Kitchen |
|---|---|
| Data preparation | gather ingredients, chop and prep |
| Model development | experiment with recipes, taste it |
| Model serving | serve the meal, adjust the menu as reviews come in |

"Adjust the menu as more people try it" is **monitoring and data drift** in costume.

**Two ways to build end-to-end:** UI (the AutoML exercise) or code (Agent Platform Pipelines with
prebuilt SDKs). **The stated reason to pick code is automation, CI, CT and CD.** If the
question is why write a pipeline instead of clicking, the answer is continuous training, not
convenience.

## Nested and repeated data: RECORD, REPEATED, UNNEST (added later, after practice questions showed the gap)

JSON often has things inside things. BigQuery keeps that shape instead of flattening it.

- **RECORD** (also called STRUCT) = **one** thing with sub-fields. A column that has its own columns. Query as `customer.city`.
- **REPEATED** = a **list**, many values in one cell. A list of records = REPEATED RECORD.
- Tell: a single `{…}` in the JSON = RECORD. A list `[…]` = REPEATED.

One order row, as stored:

| order_id | customer (RECORD) | line_items (REPEATED RECORD) |
|---|---|---|
| 7 | name: Sara, city: Toronto | [{sku: chair, qty: 2}, {sku: table, qty: 1}] |

**UNNEST** unpacks a list into one row per item at query time (Power Query analogy: Expand on a list column):

```sql
SELECT order_id, item.sku, item.qty
FROM orders, UNNEST(line_items) AS item
```

| order_id | sku | qty |
|---|---|---|
| 7 | chair | 2 |
| 7 | table | 1 |

- Loading: newline-delimited JSON (and Avro, Parquet, ORC) loads nested and repeated fields natively with a plain load job. No pipeline needed.
- Why keep it nested: no duplicated order data per item, and one row per order until you choose to unpack.
- The tell: "nested JSON" + "query each item" + "avoid a pipeline" → load job with RECORD / REPEATED columns, query with UNNEST.
- Rival to reject: one JSON-type column works too, but values stay untyped and every query must extract and cast.
