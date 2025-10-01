# Table of Contents

1. **Ingestion**
   - [1.1 Partitioning (and folder structure)](#11-partitioning-and-folder-structure)
   - [1.2 Descriptive file names](#12-descriptive-file-names)
   - [1.3 Idempotency](#13-idempotency)
   - [1.4 Wrap-Up](#wrap-up)

2. **Validation**
   - 2.1 Definition  
   - 2.2 Why validation matters  
   - 2.3 Types of validation checks  
   - 2.4 Tools to implement validation  
   - 2.5 Bondio example  

3. **Loading (Raw → Staging)**
   - 3.1 Loader definition and options  
   - 3.2 Raw layer  
   - 3.3 Staging layer  
   - 3.4 Bondio example  

4. **Transformations (Staging → Analytics)**
   - 4.1 Medallion architecture  
   - 4.2 Facts and dimensions  
   - 4.3 Handling slowly changing dimensions  
   - 4.4 Bondio example  

5. **Analytics and Reporting**
   - 5.1 Common use cases  
   - 5.2 Example dbt marts  
   - 5.3 Bondio example  

6. **Monitoring and Observability**
   - 6.1 Data quality tests  
   - 6.2 File arrival checks  
   - 6.3 Logging and manifest audits  
   - 6.4 Bondio example  

7. **Wrap-Up and Next Steps**
   - 7.1 Key principles  
   - 7.2 Future improvements  
   - 7.3 Why this matters for Bondio


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

