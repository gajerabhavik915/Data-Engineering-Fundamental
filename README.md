# Data-Engineering-Fundamental

<img width="492" height="130" alt="image" src="https://github.com/user-attachments/assets/e3bd7f53-48b0-4136-afd8-23a7127acd12" />

## Phase 1 — The three storage paradigms
Before we dive in, here's the mental model a senior DE carries at all times: these three aren't just "storage options" — they represent three different answers to the question "what do we actually need data infrastructure to do?" Each one was invented because the previous one failed at something important.

### Concept 1 — Data Warehouse
<img width="697" height="439" alt="image" src="https://github.com/user-attachments/assets/c2537ff3-c892-4494-9ef6-85e154de0570" />

A data warehouse is a central repository of integrated, historical, structured data — optimized entirely for reading and analytics, not for writing transactions.

The key mental model: a warehouse enforces a contract upfront. Before any data enters, it must conform to a defined schema. This is called "schema-on-write." The benefit is that every query is fast and clean. The cost is that loading is expensive and rigid — if a source changes its format, your pipeline breaks.

Real-life scenario: you're a retailer. Your ERP has daily sales. Your CRM has customer records. Your POS has store transactions. None of these speak the same language. The warehouse is where you hire a translator (ETL), reconcile all three into one consistent model, and let Finance run reports on it at 9am every morning. This is exactly what your LCBO reconciliation work does — you're building the "translator" part.

When senior DEs choose a warehouse: when the data is well-understood, the schema is stable, users are analysts running known reports, and query performance on structured data is the top priority.

###### Azure tools: Synapse Analytics (Dedicated SQL Pool), Azure SQL Data Warehouse (legacy).

#### Concept 2 — Data Lake
A decade after warehouses, companies ran into a wall: warehouses are expensive, can only hold structured data, and transforming everything upfront meant losing raw data forever.

The Data Lake inverts the warehouse contract entirely — store everything raw first, apply structure only when you read it. This is called "schema-on-read."

<img width="618" height="406" alt="image" src="https://github.com/user-attachments/assets/cd11b8df-1e93-4fd3-a456-0fe562adf72e" />

The data lake's killer feature is that it accepts anything — CSVs, JSON, Parquet, images, audio, logs, sensor streams — without needing to know what you'll do with it yet. You store first, decide the structure later.

The Medallion architecture (Bronze → Silver → Gold) is how mature teams organize a lake. Bronze is the untouched raw landing zone — you never modify or delete from here, it's your insurance policy. Silver is where cleaning and validation happen. Gold is business-ready, aggregated, often star-schema shaped — this is what BI tools read.

Real-life scenario: Netflix. Every time you watch something, hover over a thumbnail, or pause — that's an event log landing in their lake. They don't know at ingest time whether they want it for recommendation models, A/B testing analysis, or billing reconciliation. The lake accepts it all. Different teams apply their own schema when they read it.

When senior DEs choose a lake: when data types are diverse, you need raw data for ML/AI, storage cost matters, or you genuinely don't know the schema at ingestion time.

The dark side: lakes become "data swamps" fast. Without governance, nobody knows what's in there, nothing is documented, and querying raw files is slow. This is exactly the problem that Delta Tables solve.

#### Concept 3 — Delta Tables and the Lakehouse

Delta Tables were invented by Databricks to fix the specific failure mode of data lakes: they're great for storage but terrible for reliable, ACID-compliant querying. The Lakehouse pattern merges the warehouse's reliability and query performance with the lake's flexibility and raw storage.

<img width="583" height="393" alt="image" src="https://github.com/user-attachments/assets/afa1f4a6-af0c-49ec-8c13-ed7bda4235dc" />

The key insight: Delta is not a different place to store data. It's a thin layer you add on top of Parquet files in your existing lake. That layer is the _delta_log folder — a series of JSON commit files that record every single change. This is what makes the magic happen.

Because every write is recorded in the log, you get four things for free: ACID transactions (no partial writes, ever), time travel (query your table as it was 30 days ago with VERSION AS OF 10), schema enforcement (bad data gets rejected, not silently written), and an audit trail for compliance.

Real-life scenario: your LCBO reconciliation pipeline. Imagine you process a settlement file and accidentally load duplicate rows. With a plain Parquet lake, they're there forever. With Delta, you run RESTORE TABLE settlements TO VERSION AS OF 5 and you're back to before the bad load. In production, this is the difference between a 5-minute fix and a 3-hour incident.

Microsoft Fabric is Microsoft's answer to the full Lakehouse — their entire platform is built on OneLake, which is Delta under the hood. Everything — Spark notebooks, SQL endpoints, Power BI semantic models — reads from the same Delta tables. This is where the industry is heading.

<img width="661" height="534" alt="image" src="https://github.com/user-attachments/assets/4c1eb31b-2ad4-49d1-9bb7-a4ae86266434" />

#### The Bronze Zone — the most important layer you'll ever leave alone

The rules senior DEs enforce on Bronze
Rule 1 — Append-only, never modify. Once data lands in Bronze, it is immutable. You never update, delete, or overwrite a Bronze file. If you got bad data, you land the correction as a new record. The bad record stays.

Rule 2 — No business logic, ever. No filters, no joins, no derivations. You are not allowed to say "we only need orders above $100" at this layer. You don't know yet what future use cases will need.

Rule 3 — Preserve source fidelity exactly. If the source sends JSON, store JSON. If it sends pipe-delimited flat files, store those. Don't convert to Parquet here — that's Silver's job. Store the exact bytes you received.

Rule 4 — Always include ingestion metadata. Add three columns the source never sent you: _ingested_at (timestamp), _source_file (what file or API call this came from), and _pipeline_run_id (which pipeline execution loaded this). These become priceless when you're debugging at 2am.

Rule 5 — Partition by ingestion date, not business date. Folder structure should be bronze/source_system/entity/year=2025/month=06/day=13/. Not by the date on the record — by the date it arrived. This matters for reprocessing.

<img width="662" height="607" alt="image" src="https://github.com/user-attachments/assets/f1998591-3858-4c86-b996-69331f1bf906" />



### Phase 2 — Data Modeling

<img width="651" height="170" alt="image" src="https://github.com/user-attachments/assets/a93291ba-e2b5-4285-90ff-2ec31b6cd461" />

#### Pillar 1 — Star Schema vs Snowflake Schema

<img width="707" height="552" alt="image" src="https://github.com/user-attachments/assets/918ad5c4-8760-410a-bc92-824e02043ff4" />

How Phase 2 maps to your actual LCBO work right now
The reconciliation pipeline you're building is a data modeling problem wearing pipeline clothes. Think about it this way:

Your settlement file is a transaction-grain fact table — one row per product sold per period. The item ID is a dimension foreign key into a product dimension. The settlement period is a date dimension key. The dollar values and quantities are additive measures.

The Excel scientific notation bug that keeps breaking your pipelines is a natural key corruption issue — exactly the problem surrogate keys in SCD Type 2 were designed to isolate. If you had a proper product_sk generated in the warehouse, the item ID format from Excel becomes irrelevant — it's just the natural key you use to look up the surrogate, and you handle the format conversion once, cleanly, in the staging model.

The pattern in dbt would be: stg_lcbo__settlements.sql casts item_id as a string explicitly, dim_product assigns a surrogate key, and fact_settlements joins on the surrogate. The format problem is contained to one 3-line staging model instead of scattered across every pipeline.

#### Pillar 2 — Slowly Changing Dimensions (SCDs)

<img width="724" height="609" alt="image" src="https://github.com/user-attachments/assets/bcf18a4d-5010-4414-895c-66c18e159cae" />

SCD Type 2 is the one you'll implement most often in real pipelines. 

The implementation pattern in Python/pandas that you already know maps directly — you're essentially doing a MERGE: compare incoming rows to current dimension rows, close changed rows, insert new rows. dbt has a built-in snapshot feature that handles this automatically.

#### Pillar 3 — dbt (data build tool)

Learn from the dbt repo....



### Phase 3 — Pipelines and Data Movement

This phase is about the journey data takes from source to destination. Phase 1 was where data lives. Phase 2 was how it's shaped. Phase 3 is how it actually gets there — and what can go wrong along the way.

Three pillars: ETL vs ELT, Batch vs Streaming, Change Data Capture (CDC). These aren't just concepts — every pipeline decision you make for the rest of your career maps to one of these three.

#### Pillar 1 — ETL vs ELT

<img width="708" height="329" alt="image" src="https://github.com/user-attachments/assets/9151b81c-1395-45ae-8e16-4b898eab1b55" />

<img width="756" height="348" alt="image" src="https://github.com/user-attachments/assets/2386c204-ccd1-4d30-b72d-c8d6517a95de" />


#### Pillar 2 — Batch vs Streaming

This is one of the most consequential architectural decisions you make on any pipeline. It determines your tools, your infrastructure cost, your latency, and your complexity budget.

<img width="740" height="338" alt="image" src="https://github.com/user-attachments/assets/04601e01-1281-4162-8ca8-6ec12490574e" />


#### Pillar 3 — Change Data Capture (CDC)

CDC is the answer to one of the hardest problems in data engineering: source databases change constantly — rows get inserted, updated, deleted. How do you capture those changes efficiently without hammering the source system with full table scans every hour?

This is directly relevant to your work — PostgreSQL to SQL Server pipelines are a classic CDC use case.

<img width="737" height="187" alt="image" src="https://github.com/user-attachments/assets/94d4adf7-e0da-4576-a7dd-926d54528e37" />

<img width="706" height="609" alt="image" src="https://github.com/user-attachments/assets/1a7646ad-1d52-4dcf-8ae9-59eedf5bdcfa" />

<img width="724" height="268" alt="image" src="https://github.com/user-attachments/assets/09e0117d-1239-4953-b0a2-a6633301d44a" />

<img width="728" height="292" alt="image" src="https://github.com/user-attachments/assets/4da4ff00-de5a-48b1-84d8-118b4ba49e40" />

##### How all three pillars connect in one real pipeline:

Your PostgreSQL source has new orders coming in all day (CDC problem). You use ADF's CDC connector to capture only changed rows from the WAL log (log-based CDC). Those changes land in ADLS Bronze as a micro-batch every 15 minutes (ELT pattern, micro-batch cadence). A dbt incremental model runs a MERGE into the Delta fact table (incremental materialization). Power BI reads the Gold layer. End-to-end latency: 15–20 minutes. Source DB impact: near zero. This is a production-grade pipeline.


##### How Phase 3 connects everything you've learned so far

Phase 1 told you where data lives — Bronze lake, Silver lake, Gold warehouse/lakehouse.

Phase 2 told you what shape it needs to be in — star schema, SCD Type 2 dimensions, dbt models.

Phase 3 tells you how it gets there — ELT pattern, batch or streaming cadence, CDC to capture source changes efficiently.

The senior DE mental model is: every pipeline is a combination of these three decisions. When someone asks you to build a pipeline, you're really answering: What pattern moves the data (ELT)? How often does it need to move (batch/streaming)? How do we capture only what changed (CDC)? What shape does it need to be in at the destination (Phase 2)? Where does it live (Phase 1)?

Your LCBO reconciliation work maps to: ELT + daily batch + timestamp watermark CDC + star schema fact table + Delta storage. Every word of that sentence now means something concrete.


















 
