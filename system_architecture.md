```markdown
# Project Brief: Medical Operations Data Pipeline

### 1. Project Objective
To design and build a scalable system that ingests, processes, and analyzes a high-volume, high-variety (e.g., `.pdf`, `.csv`, `.txt`, `.xlsx`, images) of source files from medical device operations. The goal is to transform this unstructured and semi-structured data into structured, auditable, and actionable insights for reporting and decision-making.

---

### 2. Core Architecture: A Four-Stage Pipeline

Here is the refined 4-stage model, incorporating our key decisions. The stages are coordinated by an event-driven workflow service that records execution metadata (status, retries, operator, timestamps) and enforces automated retries/back-pressure handling.

#### 1. Dispatcher (Ingestion & Parsing)
* **Objective:** To upload, identify, and parse all source files into a standardized raw data format, and to "tag" all data for end-to-end traceability.
* **Input:** Original source files of any type.
* **Key Functions:**
    1.  **Generate Unique ID:** Immediately upon upload, the system generates a **Unique Document ID (`doc_id`)**. This `doc_id` will be passed through every subsequent stage to ensure full data lineage and auditability.
    2.  **Identify Type:** Detects file type (e.g., `.pdf`, `.csv`, `.png`).
    3.  **Extract Data:**
        * **Text/Data Files (`.csv`, `.txt`, `.xlsx`):** Parses text and data directly.
        * **Image/Scanned Files (`.pdf`, `.jpg`):** Uses **AI-powered OCR** (e.g., Google Cloud Vision, Azure Read API) to extract all text, tables, and handwriting.
* **Output:** A **JSON object** (not a CSV) containing the extracted raw data and metadata. This format is flexible and can store paragraphs of text, table data, and file properties.
    * *Example JSON:* `{ "doc_id": "pdf-abc-123", "filename": "batch_report.pdf", "raw_text": "...", "tables": [...] }`
    4.  **Enrich Metadata:** Stamps ingestion timestamp, source system identifier, checksum/hash, schema version, and security classification for lineage and deduplication.
* **Data Storage:** Data Lake (e.g., Google Cloud Storage, Amazon S3) for storing both the original source file and the output JSON object with encryption at rest, lifecycle policies, and RBAC-enforced access controls.

#### 2. Classifier (Enrichment & Clustering)
* **Objective:** To read the raw data and apply business logic to categorize and group the information into distinct, known objects (e.g., Complaint File, Supplier Invoice, Machine Output).
* **Input:** Raw Data JSON objects from the Data Lake.
* **Key Functions:**
    1.  **Classification (AI-driven):** Uses **Text Classification (NLP)** models to read the `raw_text` and assign a `document_type` (e.g., "Invoice", "Complaint_Report").
    2.  **Clustering (AI-driven):** Can optionally use **Unsupervised ML (Clustering)** to identify new or unknown document groupings.
    3.  **Feedback Loop:** Captures model confidence scores, routes low-confidence items to HITL review, and logs labeled outcomes for retraining and drift monitoring.
* **Output:** Updates to a central database table that links the `doc_id` to its new classification, confidence metrics, and review disposition.
    * *Example DB Row:* `doc_id: "pdf-abc-123" | document_type: "Complaint_Report" | status: "Classified"`
* **Data Storage:** Database (e.g., Postgres, SQL Server).

#### 3. Analyzer (Extraction & Analysis)
* **Objective:** To perform specific, pre-determined analyses by extracting the *exact* data points needed from classified documents and running cross-functional logic. Analyzer "recipes" are version-controlled, tested, and mapped to document schema versions for traceable evolution.
* **Input:** A `doc_id` and `document_type` from the Classifier database.
* **Key Functions:** The system runs a specific "recipe" based on the `document_type`.
    * *Example:* If `document_type` is "Complaint_Report," it will fetch the raw JSON and execute logic to find the `Device_ID`, `Date_of_Event`, and `Event_Description`.
    * **Monitoring:** Measures extraction coverage/accuracy and raises alerts when thresholds or schema drift is detected.
* **Output:** New rows in structured "Analysis Tables" alongside recipe version, processing timestamp, and lineage references.
* **Data Storage:** Data Warehouse (e.g., BigQuery, Snowflake) or the analytics database with enforced data quality checks and access controls.

#### 4. Reporter (Presentation)
* **Objective:** To convert the final, structured analysis tables into human-readable reports and dashboards.
* **Input:** Analysis tables from the Data Warehouse or curated semantic layer.
* **Key Functions:** Connects to the data source, validates data quality before refresh, and publishes results per agreed SLAs.
* **Output:** Report files (e.t., Excel exports), dashboards, charts, and optionally automated narratives.
* **Tools:** Power BI, Tableau, Google Data Studio (Looker), etc.
* **AI (Optional):** **Natural Language Generation (NLG)** can be used to write automated text summaries of the dashboard's findings.

---

### 3. Non-Functional Requirements & Compliance
* **Throughput & Latency:** Target sustained ingestion of priority documents at defined volumes (e.g., X docs/hour) with end-to-end latency under Y minutes for regulated reporting commitments.
* **Retention & Lineage:** Retain raw artifacts and derived datasets per regulatory mandates (e.g., 7-year minimum), maintaining immutable audit logs that link `doc_id`, recipe/model version, operator, and timestamps.
* **Security & Privacy:** Enforce encryption in transit/at rest, PHI masking in lower environments, least-privilege RBAC, and quarterly HIPAA/ISO 13485/SOC 2 compliance reviews.
* **Observability:** Standardize logging, metrics, and tracing across stages; provide dashboards for error rates, backlog depth, SLA adherence, and HITL workload.
* **Resilience & DR:** Define RPO/RTO targets, replicate critical stores cross-region, and perform automated backup validation with documented runbooks.

### 4. Key Decision Point: Analyzer Logic

As we discussed, the "Analyzer" (Step 3) is the core of the business logic. We have two primary options for its implementation, which directly impacts the need for a Human-in-the-Loop (HITL) step.

* **Option A: Rule-Based Logic (Non-AI)**
    * **Description:** This approach uses traditional code (e.g., Python scripts, SQL, Regular Expressions) to extract data. For example, "For 'Complaint_Report', find the line that starts with 'Device ID:' and copy the next 10 characters."
    * **Pros:**
        * Highly predictable, testable, and deterministic.
        * 100% accurate for known, structured patterns.
        * **Does not require a HITL validation step,** as the logic is explicit and approved.
    * **Cons:**
        * Very brittle. A small change in a source file's template (e.g., "Device ID" changes to "Device") will break the extractor.
        * Cannot handle unstructured text (like a "notes" field).
        * High maintenance cost.

* **Option B: AI-Driven Logic (LLM)**
    * **Description:** This approach uses **NER** and **LLMs** to extract data. For example, "Read this 'Complaint_Report' and return the Device ID and a summary of the event, no matter where they are in the document."
    * **Pros:**
        * Extremely flexible and resilient to changes in format.
        * Can "understand" and extract meaning from unstructured paragraphs.
    * **Cons:**
        * Non-deterministic (an answer may vary slightly).
        * Requires careful prompt engineering.
        * **May require a HITL review step** for 100% validation, especially in a medical device context.

**Initial Recommendation:**
Start by building the PoC (Proof of Concept) with **Option A (Rule-Based)**. This will prove the 4-stage pipeline is working. We can then introduce **Option B (AI-Driven)** in Phase 2 for more complex documents, at which point we will re-evaluate the need for a HITL system.

### 5. Operational Roadmap
1. **PoC (Rule-Based Analyzer):** Deliver the end-to-end pipeline with deterministic recipes, automated regression tests, baseline observability, and Classifier HITL handling for low-confidence items.
2. **Stabilization:** Implement recipe drift detection, expand monitoring coverage, and tune workflow scaling/back-pressure against measured volumes.
3. **AI Analyzer Pilot:** Run AI-based extraction in shadow mode, compare outputs versus rule-based logic, and formalize approval criteria plus HITL thresholds.
4. **Production Rollout:** Promote approved AI recipes using versioned deployments, documented retraining cadence, and runbooks for incident response.

### 6. Reporting & Semantic Layer
* Decide whether BI tools connect directly to the warehouse or via a curated semantic layer (e.g., dbt, LookML).
* Publish refresh SLAs and data certification tiers so consumers understand freshness and trust levels.
* Provide a data catalog linking metrics back to `doc_id`, recipe/model versions, and source artifacts to streamline audit and self-service analytics.
```