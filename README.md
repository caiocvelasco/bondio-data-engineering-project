# Table of Contents

0. [Executive Summary](#section-0-executive-summary)  

1. [Section 1: Ingestion](#section-1-ingestion)  
   - [1.1 Partitioning (and folder structure)](#11-partitioning-and-folder-structure)  
   - [1.2 Descriptive File Names](#12-descriptive-file-names)  
   - [1.3 Idempotency](#13-idempotency)  
   - [1.4 Wrap-Up](#14-wrap-up)  

2. [Section 2: Validation](#section-2-validation)  
   - [2.1 Definition](#21-definition)  
   - [2.2 Why validation matters](#22-why-validation-matters)  
   - [2.3 Types of validation checks](#23-types-of-validation-checks)  
   - [2.4 Tools to implement validation](#24-tools-to-implement-validation)  
   - [2.5 Bondio example](#25-bondio-example)  
   - [2.6 Wrap-Up](#26-wrap-up)  

3. [Section 3: Loading (Raw → Staging)](#section-3-loading-raw--staging)  
   - [3.1 Loader definition and options](#31-loader-definition-and-options)  
   - [3.2 Raw layer](#32-raw-layer)  
   - [3.3 Staging layer](#33-staging-layer)  
   - [3.4 Bondio example](#34-bondio-example)  
   - [3.5 Wrap-Up](#35-wrap-up)  

4. [Section 4: Transformations (Staging → Analytics)](#section-4-transformations-staging--analytics)  
   - [4A. Staging](#4a-staging)  
     - [Definition](#definition)  
     - [Why we need it](#why-we-need-it)  
     - [How we do it](#how-we-do-it)  
     - [Bondio example (staging)](#bondio-example-staging)  
     - [Wrap-Up (Staging)](#wrap-up-staging)  
   - [4B. Analytics](#4b-analytics)  
     - [Definition](#definition-1)  
     - [Why we need it](#why-we-need-it-1)  
     - [How we do it](#how-we-do-it-1)  
     - [Bondio example (analytics)](#bondio-example-analytics)  
     - [Wrap-Up (Analytics)](#wrap-up-analytics)  

5. [Section 5: Analytics and Reporting](#section-5-analytics-and-reporting)  
   - [5.1 Common use cases](#51-common-use-cases)  
   - [5.2 Example dbt marts](#52-example-dbt-marts)  
   - [5.3 Bondio example](#53-bondio-example)  
   - [5.4 Wrap-Up](#54-wrap-up)  

6. [Section 6: Monitoring and Observability](#section-6-monitoring-and-observability)  
   - [6.1 Data quality tests](#61-data-quality-tests)  
   - [6.2 File arrival checks](#62-file-arrival-checks)  
   - [6.3 Logging and manifest audits](#63-logging-and-manifest-audits)  
   - [6.4 Bondio example](#64-bondio-example)  
   - [6.5 Wrap-Up](#65-wrap-up)  

7. [Section 7: Wrap-Up and Next Steps](#section-7-wrap-up-and-next-steps)  
   - [7.1 Key principles](#71-key-principles)  
   - [7.2 Future improvements](#72-future-improvements)  
   - [7.3 Why this matters for Bondio](#73-why-this-matters-for-bondio)  
   - [7.4 Next step: Data Science & ML idea](#74-next-step-data-science--ml-idea)  

# Section 0: Executive Summary

This document outlines a simple but robust **end-to-end data pipeline** for Bondio’s pricing and invoicing data.  
It follows the **medallion architecture (Bronze → Silver → Gold)** and ensures data is:  
- **Organized** (partitioning, naming conventions).  
- **Validated** (schema and business checks).  
- **Reliable** (idempotency, manifest tracking).  
- **Usable** (facts/dimensions for finance & business teams).  
- **Monitored** (tests, alerts, observability).  

---

### Key Flow
1. **Ingestion (S3)** → CSVs land in structured folders (`landing/`, `archive/`, `quarantine/`).  
2. **Validation** → lightweight checks before loading (schema, row count, encoding).  
3. **Loading** → RAW tables mirror files; STAGING tables clean and deduplicate.  
4. **Transformations** → dbt incremental models apply:  
   - **CDC (upserts)** for invoices/payments.  
   - **SCD2 (history)** for price lists/discounts.  
   - **Append-only** for usage logs.  
5. **Analytics** → facts & dimensions (invoices, usage, products, customers).  
6. **Reporting** → marts for revenue, ARPU, discounts, FX impact.  
7. **Monitoring** → data quality tests, file arrival checks, manifest audits.  
8. **Next Steps** → extend into ML (e.g., churn prediction for SIM resellers).  

---

### Bondio Business Value
- **Finance**: clean, auditable revenue reporting.  
- **Operations**: reliable usage tracking across 200+ countries.  
- **Growth**: ML models for churn prediction and discount optimization.  

This pipeline makes Bondio’s data **trustworthy, scalable, and business-ready** — setting the foundation for analytics and machine learning.

# Section 1: Ingestion

When building an end-to-end data pipeline, the first step is **Ingestion**.  
This is where data first arrives in our system: raw CSVs from Bondio’s pricing and invoicing sources land in our storage (S3), and we need to make sure they’re organized, traceable, and safe to load.  

**Issues we want to avoid:**

If we just dump everything into a single folder like this:

```
s3://bondio-data/
  invoices.csv
  invoices2.csv
  usage_eSIM.csv
  file(1).csv
```

…we quickly hit problems:
- **Hard to find**: Which file belongs to which day?  
- **Risk of overwrites**: Someone re-uploads `invoices.csv` and overwrites history.  
- **Impossible to trace**: We don’t know which file generated which numbers.  

To avoid these pitfalls, we design the ingestion layer around **three** principles: 
- Partitioning
- Descriptive file names
- Idempotency


---

## 1. Partitioning (and folder structure)

**Definition:** Partitioning means organizing files into subfolders by a key. Here, we use **date**.  

We also group files first by **dataset name** (e.g., `invoices`, `invoice_lines`, `usage`) so that each dataset has its own *“home”* in the bucket.

Instead of one messy bucket, we structure S3 like this:

```
s3://bondio-data/pricing_invoicing/
  invoices/
    landing/YYYY=2025/MM=10/DD=01/invoices_2025-10-01.csv.gz
    landing/YYYY=2025/MM=10/DD=02/invoices_2025-10-02.csv.gz
    archive/...
    quarantine/...
  invoice_lines/
    landing/YYYY=2025/MM=10/DD=01/invoice_lines_2025-10-01.csv.gz
    ...
```

### Why this structure?
- **Dataset folder first (`invoices/`, `invoice_lines/`)**: keeps different datasets separated. Without this, invoice headers, usage, and payments could mix together, making management and validation messy.  
- **Landing**: the “front door.” Files arrive here directly from Bondio’s billing exports or partner systems.  
- **Archive**: once a file is **validated and successfully loaded into Snowflake**, it is moved here. Archiving ensures we keep an immutable history of what actually fed our reporting.  
- **Quarantine**: if a file **fails validation** (e.g., missing required columns, corrupt encoding, or empty rows), it is moved here instead. Quarantine isolates the bad file so it cannot pollute downstream data, while still keeping a copy for troubleshooting.  

**What makes these movements happen?**  
- A lightweight **validator** (Python script, AWS Lambda, or GitHub Action) checks new files in `landing/` for schema, encoding, row counts, and duplicates.  
  - If validation **fails** → the file is moved to `quarantine/` and logged in a manifest table with the error reason.  
  - If validation **passes** → the file is handed to the **loader**.  

**What is the loader?**  
The loader is the process that **ingests trusted files into Snowflake**.  
- Simple options:  
  1. **dbt + external tables** (common, easy to manage).  
  2. **Snowpipe** (auto-ingest).  
  3. **Python script with `COPY INTO`** (lightweight alternative).  

After a successful load, the loader:  
- Moves the file from `landing/` → `archive/`.  
- Updates the manifest to mark it as `loaded` with row counts.  

This ensures only files that are **both valid and successfully loaded** make it into `archive/`, which becomes the audit trail.  

This way, every file has a **clear lifecycle**:  
1. Land in `landing/`.  
2. Validated automatically.  
3. Either archived permanently (good) or quarantined for review (bad).  

### Benefits
- Data is tidy and **easy to navigate** (like shelves in a library).  
- Queries in Snowflake can use **partition pruning** (scan just `DD=01` if you need Oct 1).  
- **Clear lifecycle**: files always start in `landing`, then either move to `archive` (good) or `quarantine` (bad).  
- Quarantine ensures “bad data” is caught early, not silently loaded into reports.  

### Bondio example
Every night, Bondio’s billing system drops an invoice file into `invoices/landing/`.  
- If the file looks correct, the validator marks it as accepted. The loader ingests it into Snowflake, then moves it into `archive/`.  
- If the file is malformed (for example, missing the `invoice_id` column), the validator moves it into `quarantine/` and sends an alert. The partner team is then asked to resend a corrected file.

---

## 2. Descriptive File Names

**Definition:** File names should encode metadata directly: dataset name, business date, source, and a hash.

**Example filename:**
invoices_2025-09-30_src=billing_v1_hash=4f8a9c.csv.gz

**Why we do it:**
- **Self-explanatory:** You know what’s inside without opening the file.  
- **Traceability:** We can point to the exact file that fed a given report.  
- **Error detection:** If two files for the same day have different hashes, we know the data changed upstream.

**Bondio example:**  
An IoT client resends September usage after correcting a mistake.  
By naming the file:  
usage_agg_2025-09_src=rating_v2_hash=aa72bd.csv.gz  
we can clearly tell it’s the same dataset and period, but with new content.

---

## 3. Idempotency

**Definition:**  
Idempotency means the pipeline can process the same file multiple times and always end up with the same result — no duplicates, no double counting, no corruption.

---

**Why we do it:**  
- Finance data **cannot** afford duplicates: loading the same invoices twice would double revenue reports.  
- Pipelines must be safe to **rerun** after failures or bug fixes.  
- It builds **trust**: once ingested, data is consistent and stable.

---

**How we achieve it:**  
1. **Manifest table (in Snowflake)**  
   - A control table that logs every file processed (`dataset, s3_key, hash, date, status, row_count`).  
   - On arrival, the validator writes a row (`accepted` or `rejected`).  
   - After load, the loader updates it to `loaded`.  
   - If the same hash + date shows up again, the loader skips it.  

   Example table structure:  
   ```sql
   create table if not exists control.file_manifest (
     dataset        string,
     s3_key         string,
     file_hash      string,
     business_date  date,
     status         string,   -- accepted | loaded | rejected
     row_count      number,
     reason         string,
     first_seen_at  timestamp
   );
   ```

2. **Row hashes**  
   - Each row in RAW tables carries a `_row_hash` (e.g., `md5(invoice_id + line_id + amount)`).  
   - Staging keeps only one version per hash, protecting against duplicates inside files.  

3. **Overwrite-safe partitioning**  
   - Reprocessing a day replaces that partition instead of stacking more rows, ensuring only the correct version remains.  

---

**Bondio example:**  
On 2025-09-30, the billing system drops `invoices_2025-09-30_src=billing_v1_hash=4f8a9c.csv.gz`.  
- Validator records it in the manifest with `status=accepted`.  
- Loader ingests it, updates manifest to `loaded` with `row_count=1245`.  
- Later, the same file is uploaded again. The loader checks the manifest, sees the hash already loaded, and skips it.  
- September revenue stays correct.  

**Manifest table after the first successful load (1 row):**
| dataset  | s3_key                                                                                  | file_hash | business_date | status  | row_count | reason | first_seen_at        |
|----------|------------------------------------------------------------------------------------------|-----------|---------------|---------|-----------|--------|----------------------|
| invoices | s3://bondio-data/pricing_invoicing/invoices/landing/YYYY=2025/MM=09/DD=30/invoices_2025-09-30_src=billing_v1_hash=4f8a9c.csv.gz | 4f8a9c    | 2025-09-30    | loaded  | 1245      | null   | 2025-09-30 02:05:00  |

---

## Wrap-Up

Ingestion is not just about “getting the files in.” It’s about **building trust from the very first step**:  
- **Partitioning** keeps files organized by dataset and date, making navigation and queries efficient.  
- **Descriptive file names** make each file transparent and traceable back to its source.  
- **Validator and loader** enforce the lifecycle: files start in `landing/`, then move to `archive/` (good) or `quarantine/` (bad).  
- **Idempotency** with a manifest table, row hashes, and partition overwrites guarantees correctness even when jobs are rerun or files are resent.  

Together, these principles turn the messy world of CSV drops into a **stable, auditable foundation** for the rest of the pipeline.

---

# Section 2: Validation

After ingestion (files arriving in S3), the next step is **Validation**.  
This is the gatekeeper step: before a file is trusted and ingested into Snowflake, we need to confirm it follows the expected structure and quality rules.

---

## 2.1 Definition
Validation is the process of checking that each file arriving in `landing/` is:
- **Well-formed** (correct format, schema, encoding).  
- **Complete** (not empty, has required columns).  
- **Consistent** (matches the rules we expect for this dataset).  

Files that pass validation move forward to the loader.  
Files that fail validation are sent to `quarantine/` with a logged error.

---

## 2.2 Why validation matters
Without validation, we risk:  
- **Silent errors**: a missing column (`invoice_id`) could break joins later, producing wrong revenue.  
- **Corrupt data**: a bad delimiter or non-UTF-8 character could block Snowflake loads.  
- **Wrong totals**: an empty file might overwrite an entire day with nothing, losing data.  
- **Trust issues**: analysts won’t believe reports if numbers change unexpectedly.  

Validation ensures only **good, trusted data** moves forward.

---

## 2.3 Types of validation checks
Typical checks at the ingestion stage include:

1. **File-level checks**  
   - Name matches expected pattern (dataset + date + source + hash).  
   - File is compressed (`.csv.gz`).  
   - Encoding = UTF-8.  

2. **Schema checks**  
   - All required columns are present.  
   - Column order matches contract (optional).  
   - No unexpected/renamed critical columns.  

3. **Content checks**  
   - File is not empty (`row_count > 0`).  
   - Dates and numbers parse correctly for a sample of rows.  
   - Required fields (like `invoice_id`) are not null.  

4. **Business sanity checks** (lightweight)  
   - Totals are not negative (unless allowed).  
   - Currencies match ISO codes.  
   - Status is within allowed values (e.g., `paid`, `pending`, `cancelled`).  

---

## 2.4 Tools to implement validation
Keep it simple:
- **Python script** (most flexible, can run anywhere).  
- **AWS Lambda** (triggered by S3 upload, serverless and automatic).  
- **GitHub Actions** (cron job that scans `landing/` periodically).  
- **dbt tests** (can be run once the data is inside RAW tables, as a second layer of validation).  

Best practice: do **fast, cheap checks before Snowflake** (Python/Lambda), and **deeper checks inside Snowflake** (dbt tests).

---

## 2.5 Bondio example
Every night, Bondio’s billing system drops an `invoices` file into `landing/`.  

1. Validator runs:  
   - ✅ Name matches: `invoices_2025-09-30_src=billing_v1_hash=4f8a9c.csv.gz`.  
   - ✅ Encoding is UTF-8, compressed as `.gz`.  
   - ✅ Columns: `invoice_id, customer_id, invoice_date, due_date, subtotal, total, status`.  
   - ✅ Row count = 1245.  

   → File passes validation → marked as *accepted* in the **manifest**.  

2. Next day, a travel app uploads a malformed file:  
   - ❌ Missing `invoice_id` column.  
   - Validator fails schema check → moves file to `quarantine/`.  
   - Manifest updated: `status=rejected, reason='missing invoice_id'`.  
   - **An alert is sent to the Bondio team, who request a corrected file from the partner**.  

---

## Wrap-Up
Validation is the **gatekeeper**:  
- It ensures data is structurally correct before it enters Snowflake.  
- Files are either **accepted** (go to loader → archive) or **rejected** (go to quarantine).  
- Combining **lightweight checks (before load)** and **dbt tests (after load)** ensures Bondio’s revenue and usage data remain consistent and trustworthy.

---

# Section 3: Loading (Raw → Staging)

Once files have passed validation, they are ready to be **loaded into Snowflake**.  
This step creates the **RAW layer** (a faithful copy of the source files) and then the **Staging layer** (cleaned, typed, deduplicated, business-ready).  

---

## 3.1 Loader definition and options
The **loader** is the process that moves trusted data from S3 into Snowflake.  
Options include:  
1. **dbt + external tables** (simple and transparent).  
2. **Snowpipe** (auto-ingest with notifications).  
3. **Python script with `COPY INTO`** (lightweight and flexible).  

At Bondio’s scale, **dbt + external tables** is a good default:  
- Create **external tables** pointing to `landing/`.  
- Use **dbt incremental models** to materialize RAW tables inside Snowflake.  
- After successful load, move files to `archive/`.  

---

## 3.2 Raw layer
**Definition:** RAW tables are a *1:1 replica* of the ingested files, stored in Snowflake.  

- No transformations, no casting — keep the data exactly as it arrived.  
- Add only metadata: load timestamp, filename, file hash.  
- Purpose: an auditable “source of truth” that analysts can always trace back to.  

**Example (RAW.invoices):**
| invoice_id | customer_id | invoice_date | due_date | subtotal | total | status | _filename | _loaded_at |
|------------|-------------|--------------|----------|----------|-------|--------|-----------|------------|
| INV-1001   | CUST-001    | 2025-09-30   | 2025-10-15 | 100.00   | 110.00 | paid    | invoices_2025-09-30_src=billing_v1_hash=4f8a9c.csv.gz | 2025-10-01 02:05 |

---

## 3.3 Staging layer
**Definition:** Staging tables are **cleaned versions** of RAW, with consistent types and deduplicated rows.  

- Cast fields into correct types (`date`, `number`, `boolean`).  
- Deduplicate using `_row_hash` or `invoice_id`.  
- Apply lightweight renaming (snake_case, standard column names).  
- Keep them close to the source — avoid heavy business logic here.  

**Example (STG.invoices):**
| invoice_id | customer_id | invoice_date | due_date   | subtotal | total  | status   | source_file |
|------------|-------------|--------------|------------|----------|--------|----------|-------------|
| INV-1001   | CUST-001    | 2025-09-30   | 2025-10-15 | 100.00   | 110.00 | paid     | billing_v1  |

---

## 3.4 Bondio example
- The validator accepts `invoices_2025-09-30_src=billing_v1_hash=4f8a9c.csv.gz`.  
- Loader (dbt + external table) ingests it into **RAW.invoices**.  
- A staging model (`stg_invoices.sql`) cleans the data:  
  - Casts `invoice_date` and `due_date` as `date`.  
  - Standardizes `status` values (`paid`, `pending`, `cancelled`).  
  - Removes duplicates by `invoice_id`.  
- The file is marked as **loaded** in the manifest and moved to `archive/`.  

---

## Wrap-Up
Loading bridges the gap between raw files and usable data:  
- **RAW** = exact replica (for audit and traceability).  
- **STAGING** = cleaned, deduplicated, and standardized (ready for business logic).  
- dbt + external tables give Bondio a **transparent, modular, and maintainable** approach to ingest data consistently into Snowflake.

---

# Section 4: Transformations (Staging → Analytics)

Transformations are where Bondio’s pipeline moves from **raw files** to **business insights**.  
We do this in two steps:  
1. **Staging (A)** → cleaning, deduplication, handling updates and history (CDC/SCD).  
2. **Analytics (B)** → building facts and dimensions that power reporting.  

---

## 4A. Staging

### Definition
The **Staging layer** is the bridge between RAW and Analytics.  
It contains cleaned, standardized, and deduplicated data, but still close to the source.  

### Why we need it
- **Auditability**: keep RAW untouched, but create a trusted version for downstream use.  
- **Consistency**: cast correct data types, standardize field names, normalize values.  
- **Correctness**: apply CDC logic so transactional updates are reflected properly.  
- **History**: apply SCD logic for reference data that changes over time.  

Without staging, analytics tables would mix messy data handling with business logic — making them harder to debug and trust.

---

### How we do it
We use **dbt incremental models** in staging, with different strategies depending on the dataset type:

1. **Transactional data (invoices, payments, invoice_lines)**  
   - Changes happen (status updates, corrections).  
   - Use **dbt incremental with `unique_key`** → implements **CDC (upserts)**.  
   - Ensures the warehouse matches the latest truth of the source.  

   Example (dbt):
   ```sql
   {{ config(materialized='incremental', unique_key='invoice_id') }}

   select
       invoice_id,
       customer_id,
       cast(invoice_date as date) as invoice_date,
       cast(due_date as date) as due_date,
       cast(subtotal as numeric(18,2)) as subtotal,
       cast(total as numeric(18,2)) as total,
       lower(status) as status,
       md5(invoice_id || customer_id || total) as _row_hash,
       current_timestamp as loaded_at
   from {{ source('raw', 'invoices') }}
   ```

   → dbt compares `unique_key=invoice_id`. If a row exists but the `_row_hash` changed, it updates it.

---

2. **Reference data (price_list, discounts)**  
   - Slowly changing by nature.  
   - Use **dbt incremental with SCD Type 2 logic**.  
   - Add `valid_from` and `valid_to` to track historical changes.  

   Example (SCD2 logic):  
   ```
   product_id | price | valid_from | valid_to
   ---------- | ----- | ---------- | --------
   SKU-1001   | 10.00 | 2025-09-01 | 2025-09-30
   SKU-1001   | 12.00 | 2025-10-01 | null
   ```

   → Analysts can query the correct price for any point in time.

---

3. **Immutable logs (usage_agg)**  
   - Append-only by nature.  
   - Use **simple incremental (insert-only)**.  
   - No updates, just keep adding new rows.

---

### Bondio example (staging)
- **Invoice**: A travel app invoice changes from `pending` → `paid`.  
  - dbt incremental (CDC) updates the row instead of duplicating it.  
- **Price list**: A SKU changes from €10 to €12 on Oct 1.  
  - dbt SCD2 model closes the old row (`valid_to=2025-09-30`) and opens a new row (`valid_from=2025-10-01`).  
- **Usage**: New daily usage file arrives.  
  - dbt simply appends it to `stg_usage`.  

---

### Wrap-Up (Staging)
- **dbt incremental** = performance (don’t reprocess everything).  
- **CDC with unique_key** = correctness for transactional tables.  
- **SCD2** = history for reference tables.  
- **Append-only** = simple insert for immutable logs.  

Together, these ensure Bondio’s staging layer is **fast, correct, and historically accurate**.

---

## 4B. Analytics

### Definition
The **Analytics layer** is where we build **facts and dimensions** that power business dashboards and reporting.

### Why we need it
- Business users don’t want raw invoices or price lists.  
- They need aggregated, well-modeled data: revenue, ARPU, usage, discounts.  
- Facts & dimensions provide a **semantic layer**: consistent numbers, one source of truth.

---

### How we do it
We follow a **Medallion architecture**:
- **Bronze (RAW):** exact replica of source files.  
- **Silver (STAGING):** cleaned, deduplicated, CDC/SCD applied.  
- **Gold (ANALYTICS):** fact & dimension tables for business.  

**Dimensions:**  
- `dim_customers` (stable attributes from invoices).  
- `dim_products` (price_list with SCD2 for historical prices).  
- `dim_dates` (calendar).  

**Facts:**  
- `fct_invoices` (1 row per invoice).  
- `fct_invoice_lines` (granular items per SKU).  
- `fct_usage` (per SIM card, aggregated).  
- `fct_payments` (transactions linked to invoices).  

---

### Bondio example (analytics)
- **Revenue report:** `fct_invoices` + `dim_products` → revenue by SKU and country.  
- **ARPU (average revenue per user):** `fct_usage` + `dim_customers`.  
- **Discount impact:** `fct_invoice_lines` + `discounts` → how much revenue was reduced.  
- **FX effects:** `fct_invoices` + `fx_rates` → normalize revenue into EUR.  

---

### Wrap-Up (Analytics)
- Staging gives us **trusted, corrected, historical data**.  
- Analytics turns it into **facts and dimensions** for reporting.  
- This layered approach makes Bondio’s pipeline **auditable, performant, and business-ready**.

---

# Section 5: Analytics and Reporting

Once the pipeline delivers clean fact and dimension tables, the last step is **making data usable for the business**.  

---

## 5.1 Common use cases
- **Revenue reporting:** total invoiced, paid vs pending, discounts applied.  
- **Customer metrics:** ARPU (average revenue per user), churn, active SIMs.  
- **Product metrics:** SKU-level performance, price change effects, usage by network/country.  
- **Finance support:** FX impact, outstanding balances, credit notes.  

---

## 5.2 Example dbt marts
Typical marts are built in dbt from facts + dimensions:  
- `mrt_revenue_summary` → daily/weekly/monthly revenue.  
- `mrt_customer_lifecycle` → active users, churn, ARPU.  
- `mrt_discount_effects` → revenue loss from discounts.  
- `mrt_fx_normalized` → all invoices converted to EUR.  

---

## 5.3 Bondio example
Bondio’s finance team wants to know:  
- **September revenue by country and SKU.**  
- **Impact of discounts on IoT clients vs travel apps.**  
- **How FX fluctuations affect reported revenue.**  

Analysts query the marts directly (or through Looker/PowerBI) instead of stitching raw tables together.

---

## Wrap-Up
Analytics is where Bondio’s business users get value:  
- Facts/dimensions → stable building blocks.  
- Marts → curated answers to recurring questions.  
- Dashboards → the final interface for decision-making.  

---

# Section 6: Monitoring and Observability

Pipelines are only useful if we can **trust they run correctly every day**.  
Monitoring and observability ensure Bondio knows when data is missing, broken, or inconsistent.  

---

## 6.1 Data quality tests
- **Row count checks:** today’s invoices > 0.  
- **Not null checks:** `invoice_id`, `customer_id` must exist.  
- **Unique checks:** no duplicate `invoice_id`.  
- **Accepted values:** invoice `status` ∈ {paid, pending, cancelled}.  

(dbtl tests or Great Expectations can automate these.)  

---

## 6.2 File arrival checks
- Confirm each dataset delivers on schedule (e.g., daily invoices by 02:00 UTC).  
- Alert if expected files are missing or delayed.  
- Implement with AWS Lambda, Airflow sensors, or dbt freshness tests.  

---

## 6.3 Logging and manifest audits
- Manifest table records **every file** processed (accepted, loaded, rejected).  
- Simple dashboards show load status by day.  
- Audits ensure no gaps in history and no duplicate loads.  

---

## 6.4 Bondio example
- On Oct 1, the usage file is missing → alert is sent to the data team.  
- A malformed invoice file goes to quarantine → marked as rejected in manifest.  
- Daily dbt tests confirm all invoices have `invoice_id` and no duplicates.  

---

## Wrap-Up
Monitoring provides **early warnings**, observability provides **root-cause traceability**.  
Together, they ensure Bondio’s revenue and usage pipeline remains **reliable and auditable**.

---

# Section 7: Wrap-Up and Next Steps

---

## 7.1 Key principles
Bondio’s pipeline is designed around three core principles:  
- **Trust** → validation, idempotency, and manifest tracking.  
- **Structure** → raw → staging → analytics layering.  
- **Business value** → marts, facts, and dimensions that power decisions.  

---

## 7.2 Future improvements
- Automate schema evolution (handle new columns gracefully).  
- Add anomaly detection in monitoring (spot revenue outliers).  
- Expand marts with customer segmentation and churn metrics.  
- Introduce data contracts with upstream systems for stability.  

---

## 7.3 Why this matters for Bondio
Connectivity resale is high-volume and global. Even small errors in invoicing or usage reporting can misstate revenue.  
This pipeline ensures:  
- **Accuracy** → correct invoices, FX handling, discounts.  
- **Scalability** → daily ingestion of thousands of files without manual effort.  
- **Auditability** → regulators, partners, and finance can all trace numbers back to source files.  

---

## 7.4 Next step: Data Science & ML idea
With a clean foundation, Bondio can go beyond reporting into **predictive insights**:  

**Churn prediction for SIM resellers**  
- Train an ML model using `fct_usage` + `fct_invoices` + discounts.  
- Predict which customers (e.g., IoT clients, travel apps) are likely to churn.  
- Proactively offer targeted discounts or new connectivity packages.  

This moves Bondio from **reactive reporting** to **proactive revenue growth**.

---

## Wrap-Up
The pipeline is not just an engineering exercise — it’s the foundation for Bondio’s growth.  
With reliable ingestion, validation, staging, and analytics, Bondio is positioned to:  
- Report revenue confidently.  
- Scale operations globally.  
- Explore data science and ML opportunities to **optimize pricing and retain customers**.

