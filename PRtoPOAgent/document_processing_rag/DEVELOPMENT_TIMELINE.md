# PR to PO Compliance Agent — Development Timeline

**Start date:** February 17, 2025  
**Duration:** 6 weeks  
**Tech stack:** LangChain / LangGraph, AWS Bedrock, OpenSearch, RDS, S3

---

## Overview

| Stage | Dates | Main outcome |
|-------|--------|------------------|
| **Document processing** | Feb 17 – Mar 2, 2025 | W1: Loaders + validation. W2: S3 + RDS persistence, Bedrock extraction. |
| **RAG** | Feb 24 – Mar 9, 2025 | Chunk → Bedrock embed → OpenSearch; RAG query with client_id filter. |
| **12 semantic checks** | Mar 3 – Mar 23, 2025 | Check 1 & 2, then 3–12; config from RDS; OpenSearch for RAG where needed. |
| **LLM integration** | Mar 17 – Mar 30, 2025 | Bedrock for classification, fuzzy match, category, explanations; E2E wiring. |

---

## Stage 1: Document Processing (Weeks 1–2)

### Week 1 — Feb 17 – Feb 23, 2025

**Focus:** Ingestion & loaders only.

| Deliverable | Description |
|-------------|-------------|
| LangChain document loaders | PDF, Word, Excel loaders (from S3 or local paths). |
| Normalized output | One document object per file with metadata: `client_id`, `source_path`, `file_type`. |
| Basic validation | Encoding, empty files, max size checks. |
| Scope | No DB or S3 writes yet — focus on loaders and in-memory document objects. |

### Week 2 — Feb 24 – Mar 2, 2025

**Focus:** Storage, extraction & RDS.

| Deliverable | Description |
|-------------|-------------|
| RDS persistence | File metadata in `documents_metadata` with `client_id`. |
| S3 storage | Raw files in S3 with client separation: `s3://bucket/{client_id}/onboarding/`. |
| Bedrock extraction | Structured extraction for Contracts, SOWs, SAs (see field list below). |
| RDS parsed tables | `parsed_contracts`, `parsed_sows`, `parsed_service_agreements` with `client_id` on every row. |
| Pipeline run | Full pipeline on 5–10 sample docs; validate RDS + S3. |

**Exit criteria:** Supported files (PDF/DOCX/XLSX) load via LangChain, land in S3, and structured data is in RDS with correct `client_id`.

---

### Detailed tech stack — Document processing

| Component | Technology | Use in this stage |
|-----------|------------|-------------------|
| **Document loaders** | LangChain (`langchain_community.document_loaders`) | `PyPDFLoader`, `UnstructuredWordDocumentLoader`, `UnstructuredExcelLoader` (or S3-backed equivalents) for PDF, DOCX, XLSX. |
| **Document schema** | LangChain `Document` | Normalized object with `page_content` and `metadata` (client_id, source_path, file_type, etc.). |
| **Object store** | **Amazon S3** | Raw onboarding documents; key pattern `{client_id}/onboarding/{filename}`; optional versioning. |
| **Structured DB** | **Amazon RDS (PostgreSQL)** | `documents_metadata` (doc_id, client_id, source_path, file_name, file_type, doc_type); `parsed_contracts`, `parsed_sows`, `parsed_service_agreements` with client_id on every row. |
| **LLM for extraction** | **AWS Bedrock** | Claude (e.g. Claude 3.5 Sonnet) for document-type classification and structured field extraction (Contracts: ID, supplier, dates, entities, payment terms, price table; SOWs: scope, deliverables, milestones, rates; SAs: validity, terms, coverage). |
| **SDK / runtime** | `boto3` (Bedrock), `psycopg2` or SQLAlchemy (RDS) | Invoke Bedrock `InvokeModel` / `InvokeModelWithResponseStream`; run SQL for inserts. |
| **Orchestration (optional)** | LangGraph | Optional graph: load → validate → [classify → extract] → write RDS/S3. |

---

## Stage 2: RAG (Weeks 2–3)

### Week 2 (overlap) — Feb 24 – Mar 2, 2025

**Focus:** Chunking & embedding.

| Deliverable | Description |
|-------------|-------------|
| Fixed chunking | e.g. 512 chars, 50 overlap; metadata on every chunk: `client_id`, `doc_id`, `doc_type`, `source_file`. |
| Embedding | Bedrock Titan Embeddings (or chosen Bedrock embedding model). |
| OpenSearch index | Index vectors into OpenSearch with `client_id` in document/metadata for filtered search. |
| Pipeline | Load from S3 or RDS → chunk → embed via Bedrock → index into OpenSearch (client-separated). |

### Week 3 — Mar 3 – Mar 9, 2025

**Focus:** Retrieval & integration.

| Deliverable | Description |
|-------------|-------------|
| RAG query | Given `(client_id, query, top_k)`, query OpenSearch with `client_id` filter; return top-k chunks (+ doc_id, doc_type, score). |
| Reference-data use | Use RAG in “Load reference data” step: policy/SOP or category-card retrieval for checks and explanations. |
| Optional | LangGraph node or LangChain tool wrapping OpenSearch RAG. |

**Exit criteria:** For a given `client_id`, semantic search returns only that client’s chunks from OpenSearch; pipeline can call this for reference data.

---

### Detailed tech stack — RAG

| Component | Technology | Use in this stage |
|-----------|------------|-------------------|
| **Text splitting** | LangChain `RecursiveCharacterTextSplitter` (or custom) | Fixed-size chunking (e.g. chunk_size=512, chunk_overlap=50); attach client_id, doc_id, doc_type to each chunk. |
| **Embeddings** | **AWS Bedrock** | Titan Embeddings (e.g. `amazon.titan-embed-text-v2` or v1) for vector generation; 1024/1536 dims depending on model. |
| **Vector store** | **Amazon OpenSearch Service** | Index embeddings with KNN; every document/chunk includes `client_id` in metadata for filter; use same embedding dimension as Bedrock model. |
| **Search** | OpenSearch KNN / vector search | Query: embed query via Bedrock → KNN search with filter `client_id = :client_id`; return top_k chunks, scores, doc_id, doc_type. |
| **Orchestration** | LangChain / LangGraph | Optional: “RAG” node/tool that takes (client_id, query, top_k), calls Bedrock for query embedding, runs OpenSearch, returns chunks. |
| **SDK** | `boto3` (Bedrock), `opensearch-py` | Bedrock for embed; OpenSearch client for index and search. |

---

## Stage 3: 12 Semantic Checks (Weeks 3–5)

### Week 3 (overlap) — Mar 3 – Mar 9, 2025

**Focus:** Check 1 & 2.

| Deliverable | Description |
|-------------|-------------|
| Check 1 | Attachment existence (≥1), document type classification (Quote/SOW/Contract/Invoice/…), threshold-based doc requirements; sub-results 1.1, 1.2, 1.3. |
| Check 2 | Document validity (effective ≤ PR date, expiration ≥ PR date), supplier applicability, entity applicability; config from RDS. |

### Week 4 — Mar 10 – Mar 16, 2025

**Focus:** Checks 3–7.

| Deliverable | Description |
|-------------|-------------|
| Checks 3–7 | **3** Supplier (PR vs doc, PR vs master, status, contact). **4** Item description. **5** Quantity/UOM. **6** Amount/price. **7** Currency. |
| Contract | Each check: input (PR + extracted docs + reference data from RDS/OpenSearch), output (PASS/FAIL/WARN/SKIP + evidence). Semantic layers and thresholds from RDS. |

### Week 5 — Mar 17 – Mar 23, 2025

**Focus:** Checks 8–12.

| Deliverable | Description |
|-------------|-------------|
| Checks 8–12 | **8** Payment terms. **9** Category/commodity (optional OpenSearch RAG). **10** Buying channel. **11** ATF. **12** Delivery date. |
| Check runner | Single runner: run all 12, collect results, aggregate score and outcome path (A/B/C). |

**Exit criteria:** All 12 checks run in sequence; semantics config-driven via RDS; OpenSearch used where RAG is needed.

---

### Detailed tech stack — 12 semantic checks

| Component | Technology | Use in this stage |
|-----------|------------|-------------------|
| **Config & reference data** | **Amazon RDS (PostgreSQL)** | Semantic layer definitions, compliance_check_config, threshold_config, hil_config; supplier_master, contract_master, category_cards, catalog_data. All tables have `client_id`; checks read config at runtime. |
| **Reference retrieval** | **Amazon OpenSearch Service** | RAG for policy/SOP and category-card text where checks need “search by meaning” (e.g. Check 9 category, Check 10 buying channel); query filtered by client_id. |
| **Check logic** | Python (optionally LangGraph nodes) | Deterministic rules (dates, amounts, exact match) in code; thresholds and options from RDS config. |
| **LLM for checks** | **AWS Bedrock** | Claude for: document classification (Check 1), fuzzy supplier/item match (Checks 2–4), category classification (Check 9), and any semantic comparison; invoked per check or batched as needed. |
| **Input/output** | RDS + S3 | PR header/line items/attachments (from API or S3); extracted docs from RDS or re-read from S3; check results written to RDS/DynamoDB or returned in API response. |
| **Orchestration** | LangGraph (optional) | Graph: load PR + ref data → run check 1 → … → run check 12 → aggregate; each check can be a node with RDS/OpenSearch/Bedrock calls. |

---

## Stage 4: LLM integration & E2E (Weeks 5–6)

### Week 5 (overlap) — Mar 17 – Mar 23, 2025

**Focus:** Bedrock for checks.

| Deliverable | Description |
|-------------|-------------|
| Bedrock usage | Document classification, fuzzy supplier/item match, category classification, per-check and overall explanations. |
| Interface | Single LLM interface (Bedrock only); retries and token limits. |
| Context | Minimal context per call to control cost and latency. |

### Week 6 — Mar 24 – Mar 30, 2025

**Focus:** E2E & hardening.

| Deliverable | Description |
|-------------|-------------|
| E2E flow | Load PR + attachments from S3 → document processing (or load from RDS) → run 12 checks (RDS + OpenSearch RAG + Bedrock) → score → outcome path → Bedrock explanations. |
| Config | Semantic layers, thresholds, check toggles in RDS; optional backup in S3. |
| Testing | Smoke tests: 2–3 PR scenarios (compliant, non-compliant, ATF). |

**Exit criteria:** Single pipeline on LangChain/LangGraph, Bedrock, OpenSearch, RDS, S3; 12 checks and explanations working; config in RDS.

---

### Detailed tech stack — LLM integration & E2E

| Component | Technology | Use in this stage |
|-----------|------------|-------------------|
| **LLM** | **AWS Bedrock** | Claude (e.g. Claude 3.5 Sonnet or Claude 3 Haiku) for all generation: classification, fuzzy match, category, per-check explanation, overall summary. Single provider. |
| **Embeddings** | **AWS Bedrock** | Titan Embeddings for query embedding in RAG when generating explanations (e.g. retrieve policy snippets then generate text). |
| **RAG for explanations** | **OpenSearch** | Retrieve policy/SOP chunks by client_id + query; pass to Claude as context for explanation generation. |
| **Orchestration** | LangChain / LangGraph | End-to-end graph or chain: load PR → load ref data (RDS + OpenSearch) → run 12 checks (each may call Bedrock) → aggregate → Bedrock explanation node → return result. |
| **Config** | **RDS** | Semantic layers, check definitions, thresholds, HIL config; no LLM logic hardcoded; config versioning optional. |
| **Storage** | **S3** | PR attachments, onboarding docs; optional audit logs or explanation cache. |
| **API (if applicable)** | FastAPI / Lambda | Optional REST or async trigger that runs the LangGraph pipeline and returns compliance result. |

---

## Tech stack summary by stage

| Stage | LangChain/LangGraph | AWS Bedrock | OpenSearch | RDS | S3 |
|-------|---------------------|-------------|------------|-----|-----|
| **Document processing** | Loaders, Document schema | Extraction (Claude) | — | documents_metadata, parsed_* | Raw docs |
| **RAG** | Text splitter, optional RAG tool | Titan Embeddings | Vector index, KNN search | — | — |
| **12 checks** | Optional graph nodes | Classification, fuzzy match, category | RAG for policy/category | Config, reference tables | — |
| **LLM & E2E** | Orchestration (graph/chain) | All generation + embeddings | RAG for explanations | Config, results | PR attachments, optional audit |

---

## Document types & extracted fields (reference)

- **Contracts / Supplier Content Forms:** Contract ID, supplier, effective/expiration dates, covered entities, payment terms, price tables.
- **SOWs:** Scope, deliverables, milestones, rates, supplier.
- **Service Agreements:** Validity, terms, coverage.

---

**Document version:** 1.0  
**Last updated:** February 17, 2025
