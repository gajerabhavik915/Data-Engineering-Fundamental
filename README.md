# Data-Engineering-Fundamental

<img width="492" height="130" alt="image" src="https://github.com/user-attachments/assets/e3bd7f53-48b0-4136-afd8-23a7127acd12" />

## Phase 1 — The three storage paradigms
Before we dive in, here's the mental model a senior DE carries at all times: these three aren't just "storage options" — they represent three different answers to the question "what do we actually need data infrastructure to do?" Each one was invented because the previous one failed at something important.

<img width="697" height="439" alt="image" src="https://github.com/user-attachments/assets/c2537ff3-c892-4494-9ef6-85e154de0570" />

A data warehouse is a central repository of integrated, historical, structured data — optimized entirely for reading and analytics, not for writing transactions.
The key mental model: a warehouse enforces a contract upfront. Before any data enters, it must conform to a defined schema. This is called "schema-on-write." The benefit is that every query is fast and clean. The cost is that loading is expensive and rigid — if a source changes its format, your pipeline breaks.
Real-life scenario: you're a retailer. Your ERP has daily sales. Your CRM has customer records. Your POS has store transactions. None of these speak the same language. The warehouse is where you hire a translator (ETL), reconcile all three into one consistent model, and let Finance run reports on it at 9am every morning. This is exactly what your LCBO reconciliation work does — you're building the "translator" part.
When senior DEs choose a warehouse: when the data is well-understood, the schema is stable, users are analysts running known reports, and query performance on structured data is the top priority.
Azure tools: Synapse Analytics (Dedicated SQL Pool), Azure SQL Data Warehouse (legacy).
