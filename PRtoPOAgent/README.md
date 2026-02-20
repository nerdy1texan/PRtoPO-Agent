# PR to PO Agent — Purchase Requisition Validation

**Goal:** Build a PR validation AI product that ingests a Purchase Requisition (PR) plus attachments (quotes, MSAs, SOWs), classifies and parses documents into structured JSON, persists them, and runs **12 compliance checks** to determine if the PR is compliant before conversion to a Purchase Order (PO).

This repo contains **schemas**, **compliance check specs**, **synthetic data**, **architecture docs**, and a **master notebook** that runs the full pipeline locally or against AWS (Bedrock, RDS, Titan).

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Architecture](#architecture)
3. [Tech Stacks: Production vs Local](#tech-stacks-production-vs-local)
4. [What Else Is Needed? (MCP, Vector Store, etc.)](#what-else-is-needed)
5. [Schemas & Examples](#schemas--examples)
6. [Compliance Checks](#compliance-checks)
7. [System Prompts](#system-prompts)
8. [Environment Variables](#environment-variables)
9. [Repository Structure](#repository-structure)
10. [References](#references)

---

## Getting Started

Follow **Local setup** to run the notebook on your machine with Groq and **SQLite** (no database server install). Use **[DBeaver](https://dbeaver.io/)** to view and manage the local database. Follow **AWS setup** to run against Bedrock (and optional RDS) in the cloud.

---

### Local setup (conda, API keys, SQLite + DBeaver, run)

#### 1. Conda environment

- Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html) or Anaconda if you don’t have it.
- Open a terminal in the repo root (e.g. `PR to PO Agent`).

```bash
# Create and activate a dedicated environment (Python 3.10+)
conda create -n pr2po python=3.10 -y
conda activate pr2po

# Go to the folder with the notebook and requirements
cd document_processing_rag
```

#### 2. Dependencies

```bash
pip install -r requirements.txt
# For Word document parsing (optional)
# pip install python-docx
# For PostgreSQL (only if you set DATABASE_URL to Postgres)
# pip install psycopg2-binary
```

#### 3. API key (Groq, for local LLM)

- Sign up at [Groq](https://console.groq.com/).
- In the console, create an API key (e.g. **API Keys** → **Create API Key**).
- Copy the key (starts with `gsk_...`). You’ll set it as `GROQ_API_KEY` in the next step.

#### 4. Input and output folders

- The notebook uses **input** (documents in) and **output** (extracted JSONs out) under `document_processing_rag/`.
- They are created automatically when you run **Section 1** of the notebook. You can also create them manually:

```bash
mkdir -p input output
```

- Put the PDFs (or Word/Excel) you want to process into **input** before running **Section 3**.

#### 5. Local database: SQLite (default) + DBeaver

The notebook uses **SQLite** by default — no database server to install. A single file `pr_validation.db` is created under `document_processing_rag/` when you run **Section 6**. Use **[DBeaver](https://dbeaver.io/)** (free, open-source) to view and query the data.

**5a. Install DBeaver**

- Download and install [DBeaver Community](https://dbeaver.io/download/) (Windows, macOS, or Linux).
- DBeaver supports SQLite out of the box; no extra drivers needed for the default setup.

**5b. Connect DBeaver to the SQLite database**

1. After running the notebook **Section 6** at least once, the file `document_processing_rag/pr_validation.db` will exist.
2. In DBeaver: **Database** → **New Database Connection** → select **SQLite** → **Next**.
3. **Path:** Browse to or enter the full path to `pr_validation.db`, e.g.  
   `C:\...\PRtoPOAgent\document_processing_rag\pr_validation.db` (Windows) or  
   `/path/to/PRtoPOAgent/document_processing_rag/pr_validation.db` (macOS/Linux).
4. **Test Connection** → **Finish**. You can now browse tables: `pr_header`, `pr_line_items`, `pr_attachments`, `parsed_quote`, `parsed_msa` (or a single `pr` table if the pipeline stores the combined PR).

**5c. Optional: PostgreSQL instead of SQLite**

If you prefer PostgreSQL locally, install PostgreSQL, create a database (e.g. `pr_validation`), set `DATABASE_URL` in `.env` (see step 6), and run the notebook. Use DBeaver to create a **PostgreSQL** connection (host `localhost`, port `5432`, database `pr_validation`) to view the same tables. Install the driver when DBeaver prompts. You will need `pip install psycopg2-binary` for the notebook to connect.

#### 6. Environment variables (.env)

In `document_processing_rag/`, create a `.env` file (do not commit it):

```bash
cd document_processing_rag
# Create .env (edit with your values)
```

Contents (minimum for local):

```env
# Required for local LLM
GROQ_API_KEY=gsk_your_groq_api_key_here

# Optional: use PostgreSQL instead of SQLite (then connect with DBeaver to localhost)
# DATABASE_URL=postgresql://pr2po_user:your_secure_password@localhost:5432/pr_validation
# If you omit DATABASE_URL, the notebook uses SQLite (pr_validation.db); open it with DBeaver.
```

#### 7. Run the notebook

1. From the repo root or `document_processing_rag/`, open Jupyter:
   ```bash
   jupyter notebook PR_to_PO_Master.ipynb
   # or: jupyter lab
   ```
2. Run cells in order: **Section 0** (env detection) → **1** (paths, LLM) → **2** (schemas) → **3** (list from input) → **4** (classify) → **5** (parse) → **6** (DB) → **7** (Check 1) → **8** (Check 2) → **9** (summary) → **10** (write outputs).
3. Results:
   - **output/** — `pr.json` (single PR template: header + attachments + line_items), `parsed_quotes.json`, `parsed_msas.json`, `check1_result.json`, `check2_result.json`.
   - **Database** — Same data in SQLite (`pr_validation.db`) or PostgreSQL if `DATABASE_URL` is set. Open with [DBeaver](https://dbeaver.io/) to browse tables.

**Quick start checklist (local)**

1. Install Miniconda/Anaconda → `conda create -n pr2po python=3.10 -y` → `conda activate pr2po`
2. `cd PRtoPOAgent/document_processing_rag` → `pip install -r requirements.txt`
3. Create `.env` with `GROQ_API_KEY=gsk_...` (get key from [Groq](https://console.groq.com/))
4. (Optional) Install [DBeaver](https://dbeaver.io/) to view the database later
5. `jupyter notebook PR_to_PO_Master.ipynb` → run all cells in order (Section 0 through 10)
6. Check **output/** for JSONs; open **pr_validation.db** in DBeaver (New Connection → SQLite → path to `pr_validation.db`) to browse tables

---

### AWS setup (Bedrock, optional RDS)

Use this when you want to run the notebook against **Amazon Bedrock** (and optionally **RDS**) instead of Groq and local Postgres.

#### 1. AWS account and CLI

- Have an [AWS account](https://aws.amazon.com/) and [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured:
  ```bash
  aws configure
  # Enter Access Key ID, Secret Access Key, default region (e.g. us-east-1)
  ```

#### 2. Bedrock model access

- In **AWS Console** → **Amazon Bedrock** → **Model access** (or **Manage model access**), enable the model you plan to use (e.g. **Claude 3.5 Haiku** or **Sonnet**).
- Note the **model ID** (e.g. `anthropic.claude-3-5-haiku-20241022-v2:0`). You’ll set it as `BEDROCK_MODEL_ID`.

#### 3. IAM permissions

- The IAM user or role used by the notebook needs at least:
  - **bedrock:InvokeModel** (and optionally **bedrock:InvokeModelWithResponseStream**) for the Bedrock model.
  - If you use RDS: **rds** (connect to the DB; usually you connect from the notebook using a connection string and a password, not IAM auth, unless you set that up).
- Example policy (Bedrock only; restrict resource as needed):
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
        "Resource": "arn:aws:bedrock:us-east-1::foundation-model/*"
      }
    ]
  }
  ```

#### 4. RDS (optional, for production DB)

- Create a **PostgreSQL** DB in RDS (e.g. **Create database** → PostgreSQL 15, pick instance class and storage).
- Note **endpoint**, **port**, **master username**, and **master password**.
- Ensure the notebook’s network can reach RDS (security group allows inbound on the DB port from your IP or VPC).
- Connection string format: `postgresql://USER:PASSWORD@RDS_ENDPOINT:5432/DB_NAME`

#### 5. Environment variables for AWS

Set these so the notebook uses **AWS** instead of local Groq (and optional RDS):

```env
# AWS region (e.g. us-east-1)
AWS_REGION=us-east-1
# Or use a named profile: AWS_PROFILE=your_profile

# Bedrock model (use the ID from Bedrock model access)
BEDROCK_MODEL_ID=anthropic.claude-3-5-haiku-20241022-v2:0

# Optional: RDS PostgreSQL
# DATABASE_URL=postgresql://admin:password@your-rds-endpoint.region.rds.amazonaws.com:5432/pr_validation
```

- Do **not** set `GROQ_API_KEY` when you want to use Bedrock; the notebook detects AWS when `AWS_REGION` (or `AWS_PROFILE`) and `BEDROCK_MODEL_ID` are set.

#### 6. Dependencies for AWS

```bash
conda activate pr2po
pip install langchain-aws boto3
```

#### 7. Run the notebook on AWS

1. Ensure **input** has your documents (or use synthetic data).
2. Open `PR_to_PO_Master.ipynb` and run **Section 0**. It should print something like `Detected mode: aws` and use Bedrock.
3. Run Sections **1** through **10** in order. DB writes go to RDS if `DATABASE_URL` is set; otherwise the notebook still uses SQLite locally (or you can set a local Postgres URL).

---

### Quick reference: Local vs AWS

| Step              | Local                               | AWS (production)                          |
|-------------------|-------------------------------------|-------------------------------------------|
| Env               | `conda activate pr2po`              | Same + `langchain-aws`, `boto3`           |
| LLM               | `GROQ_API_KEY`                      | `AWS_REGION` + `BEDROCK_MODEL_ID`         |
| DB                | SQLite (default); view with [DBeaver](https://dbeaver.io/). Optional: PostgreSQL | RDS PostgreSQL (set `DATABASE_URL`)       |
| Input/Output      | `document_processing_rag/input` and `output` | Same paths (or S3 if you extend the notebook) |

---

## Architecture

End-to-end flow:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  USER / SYSTEM                                                                   │
│  • PR form (header, line items) + attachments (PDF, Word, Excel)               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  DOCUMENT UPLOAD & INGEST                                                        │
│  • Accept files (or read from folder / S3)                                       │
│  • Store paths / bytes for downstream                                             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  DOCUMENT CATEGORIZATION (Check 1 prerequisite)                                  │
│  • LLM classifies each attachment: Quotation | Contract | SOW | SA | Invoice | …  │
│  • Output: document_type, confidence, reason per file                            │
│  • Uses: document markers (keywords, structure), no deep extraction               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  DOCUMENT PARSING (Extraction)                                                    │
│  • Per document type, extract into JSON using schema templates                    │
│  • Quote → quote.json schema   • MSA/Contract → msa.json schema                  │
│  • PR Header/Line Items/Attachments from form or system (or simulated)            │
│  • Output: pr (header + attachments + line_items), quote(s), msa(s)               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  DATABASE STRUCTURED ENTRY                                                       │
│  • Persist parsed PR header, line items, attachments metadata                     │
│  • Persist parsed quote(s), MSA(s) for audit and validation                      │
│  • Production: Amazon RDS (PostgreSQL)  • Local: SQLite (default); view with DBeaver  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PR VALIDATION ENGINE (12 Compliance Checks)                                     │
│  • Check 1:  Attachment Existence & Classification (gatekeeper)                  │
│  • Check 2:  Document Validity — Dates & Timings (quote/contract dates only)     │
│  • Check 3:  Supplier Information Validation                                     │
│  • Check 4:  Item Description Accuracy                                           │
│  • Check 5:  Quantity & Unit of Measure                                          │
│  • Check 6:  Amount / Price Validation                                            │
│  • Check 7:  Currency Validation                                                  │
│  • Check 8:  Payment Terms                                                        │
│  • Check 9:  Category / Commodity Classification                                 │
│  • Check 10: Buying Channel Validation                                            │
│  • Check 11: After-the-Fact (ATF) Detection                                       │
│  • Check 12: Delivery Date Lead Time                                              │
│  • Score & Aggregate → Explain (LLM) → Dashboard / API response                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  OUTPUT                                                                          │
│  • Per-check results (PASS | FAIL | NEEDS_REVIEW), evidence, policy refs        │
│  • Overall status, recommended path, human-in-the-loop flags                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Shared data layer (production):** RDS (PR/contract/supplier master data), S3 (raw documents), OpenSearch (vector search for RAG if needed), Bedrock (LLM + Titan embeddings).  
**Local:** SQLite (default; no server install); use [DBeaver](https://dbeaver.io/) to view data. Optional: local PostgreSQL; Groq or local LLM.

---

## Tech Stacks: Production vs Local

| Component        | Production (AWS)                    | Local (run & test)                          |
|-----------------|-------------------------------------|---------------------------------------------|
| **Orchestration** | LangChain, LangGraph                | LangChain, LangGraph                        |
| **LLM**          | Amazon Bedrock (e.g. Claude 3.5 Haiku) | Groq (free tier, e.g. llama-3)           |
| **Embeddings**   | Amazon Titan Embeddings             | sentence-transformers (e.g. all-MiniLM-L6-v2) or PostgresML |
| **Structured DB**| Amazon RDS (PostgreSQL)             | SQLite (default); view with DBeaver. Optional: PostgreSQL |
| **Vector DB**    | OpenSearch (pgvector or dedicated)  | PostgresML (pgvector) or in-memory / Chroma |
| **Document Store** | S3                                | Local folder (e.g. `synthetic_data/`)      |
| **Optional**     | Step Functions, Lambda (see ENGINEERING_AWS.md) | Not required for notebook |

The **master notebook** detects which environment is available (AWS env vars vs `GROQ_API_KEY`, etc.) and selects the appropriate LLM and embedding backend so the same code path runs in both modes.

---

## What Else Is Needed?

- **MCP server for intelligent parsing / checks**  
  An MCP (Model Context Protocol) server can expose tools such as “classify_document”, “extract_quote”, “run_compliance_check”. The notebook currently uses LangChain/LangGraph with direct LLM and Python logic; an MCP layer is **optional** and useful when you want:
  - A single parsing/validation service callable from multiple clients (IDE, API, CLI).
  - Standardized tools (e.g. “run_check_1”) that other agents or systems can invoke.
  - Clear separation between “tool definitions” and “orchestration”.  
  **Recommendation:** Start with the notebook; add an MCP server later if you need multi-client or agentic tool use.

- **Vector store / RAG**  
  For “load reference data” (e.g. policy, category cards, contract summaries), you need a vector store. Production: OpenSearch or RDS + pgvector. Local: PostgresML (pgvector) or Chroma/FAISS. The first two checks do not require RAG; later checks (e.g. policy lookup, semantic layer) can consume embeddings + retrieval.

- **Threshold / policy config**  
  Check 1 is policy-driven (e.g. value bands for “1 quote” vs “3 quotes”). Store threshold config in RDS or a JSON/YAML file and pass it into the check.

- **Supplier Master & Contract Master**  
  Check 2 (and 3, 8, 10) need reference data: supplier list (with status, aliases), contract list (effective/expiry, ceiling). In production these live in RDS; locally the notebook can use stub data or a small SQLite/Postgres schema.

---

## Schemas & Examples

All schemas are **JSON Schema** with an **`example`** block for one-shot prompting and validation. Paths are under `document_processing_rag/schemas/`.

### Extraction schemas (document parsing → structured output)

| Schema | Path | Purpose |
|--------|------|--------|
| **PR** | `schemas/pr.json` | Single PR template: **header** (pr_number, dates, requestor, company, cost center, total, currency, payment terms, need_by_date, buying_channel, contract_reference, category L1–L4), **attachments** (file_name, file_type, document_classification, upload_date, document_date), **line_items** (description, quantity, UOM, unit_price_usd, extended_amount_usd, supplier, category, delivery). Feeds checks 1–12. |
| **Quote** | `schemas/quote.json` | Parsed quotation: quote_number, quote_date, valid_through, supplier_name, currency, payment_terms, contract_ref, line_items (description, qty, UOM, unit_price, extended_price), total. Feeds checks 2, 3, 4, 5, 6, 7, 8. |
| **MSA** | `schemas/msa.json` | Parsed Master Service Agreement: agreement_id, parties, effective_date, end_date, payment_terms_standard, maximum_contract_value, scope, lead times, warranties, returns, termination. Feeds checks 8, 10. |

Use these as **templates + one-shot**: send the schema (and optionally the `example`) to the LLM so it returns JSON conforming to the schema.

### Compliance check result schemas

| Check | Path | Purpose |
|-------|------|--------|
| **Check 1** | `schemas/compliance_checks/check_01_attachment_existence_classification/check_01_result.json` | Output: check_1_status, attachments_found, classified_documents[], policy_requirements_met, missing_requirements[], invoice_detected, plus UI (sub_checks, field_level_assessment, evidence). |
| **Check 2** | `schemas/compliance_checks/check_02_document_validity_applicability/check_02_result.json` | **Dates & timings only.** Output: check_2_status, quotation_validity (quote date sequence, not expired, staleness), contract_validity (effective, not expired, covers delivery, near expiry), sub_checks, field_level_assessment, evidence, policy_reference. |

Checks 3–12 will follow the same pattern: each in its own folder with a result schema and example.

---

## Compliance Checks

| # | Name | Brief description |
|---|------|-------------------|
| 1 | Attachment Existence & Classification | Gatekeeper: required doc types present per policy (value/category); no full extraction. |
| 2 | Document Validity — Dates & Timings | Quote/contract dates and date ranges only: quote date ≤ PR date, quote not expired, contract effective/not expired, delivery period. Supplier/currency/total match are in other checks. |
| 3 | Supplier Information Validation | PR supplier matches document supplier; supplier in master and active. |
| 4 | Item Description Accuracy | PR line description semantically matches quote line items. |
| 5 | Quantity & Unit of Measure | PR qty/UOM match quote (with normalization). |
| 6 | Amount / Price Validation | PR unit price and total match quote within tolerance. |
| 7 | Currency Validation | PR currency matches document currency. |
| 8 | Payment Terms | PR payment terms comply with policy/contract. |
| 9 | Category / Commodity Classification | PR category matches agent-inferred/category card. |
| 10 | Buying Channel Validation | Catalog vs contract vs spot; contract reference valid. |
| 11 | After-the-Fact (ATF) Detection | No invoice attached for new PR; no backdated documents. |
| 12 | Delivery Date Lead Time | Delivery date meets minimum lead time for category. |

Detailed field-level rules for **Check 1** and **Check 2** are in:  
`synthetic_data/compliance_check_writeups/PR_Compliance_Check1_Check2_Fields.md`.

---

## System Prompts

All system prompts in the notebook **use the JSON schemas and their `example`** from `schemas/*.json`: the classification prompt includes an example output JSON; the Quote and MSA extraction prompts embed the full schema example so the LLM outputs structure that matches the template. **Compliance checks (Check 1, Check 2, and any future check)** use a **system prompt** and a **result template** from `document_processing_rag/schemas/compliance_checks/`: each `check_0X_result.json` provides the schema and an **example** object; the notebook builds the prompt with **build_check_system_prompt(schema, policy_rules)** and runs **run_check_with_llm(llm, check_num, context)** (with one retry on validation failure). Adding Check 3+ means adding a new folder and result schema with an **example** and calling the same helper.

### Document classification (Check 1 input)

Uses an example JSON output in the system prompt (e.g. `{"document_type": "Quotation", "confidence": 0.94, "reason": "..."}`).

```
You are a procurement document classifier. Given the file name and, if provided, a short excerpt or summary of the document content, classify the document into exactly one of these types: Quotation, Contract, SOW, Service Agreement, Invoice, BidSummary, Justification, Spec, Other.

Consider:
- Quotation/Quote: keywords like "Quote", "Quotation", "Valid Until"; line items with prices; supplier letterhead; reference number.
- Contract/MSA: "Agreement", "Contract", "Master Service Agreement"; parties, effective/expiry dates, signatures.
- SOW: "Statement of Work", "Scope of Work"; scope section, deliverables, milestones.
- Invoice: "Invoice", "Bill To", "Payment Due"; invoice number and date (red flag if date after PR date).
- BidSummary: multiple suppliers, comparison table, selection justification.
- Justification: "Single Source", "Sole Source", justification reason.
- Spec: technical requirements, no pricing/commercial terms.

Respond with JSON only: {"document_type": "<type>", "confidence": <0-1>, "reason": "<brief explanation>"}.
```

### Quote / MSA extraction (for parsing)

- **System prompt** embeds the full JSON **example** from `schemas/quote.json` or `schemas/msa.json` (the `example` key). The LLM is instructed to match that structure and types.
- **Production flow**: retry loop (e.g. 3 attempts), **validation tool** (`validate_parsed_output`), and **merge** on retry. Optional **LangGraph** graph: extract → validate → conditional retry or end.

### Check 1 (policy evaluation)

**Notebook:** System prompt and template come from **check_01_result.json** (description + **example**). **build_check_system_prompt(check1_schema, CHECK_1_POLICY)** → **CHECK_1_SYSTEM**; **run_check_with_llm(llm, 1, context)** returns the result JSON or `None` (then deterministic **run_check_1** fallback).

```
Given: (1) List of classified attachments [filename, document_type, confidence], (2) PR total value, (3) PR category L1, (4) Policy: value < $5K no quote required; $5K–$25K min 1 quote; $25K–$50K min 2 quotes; >$50K 3 quotes or active contract; Services require SOW/SA.

Determine: policy_requirements_met (boolean), missing_requirements (list of strings), check_1_status (PASS|FAIL|NEEDS_REVIEW). If any confidence < 0.8, consider NEEDS_REVIEW. If any document_type is Invoice, set invoice_detected true.

Output JSON matching the Check 1 result schema.
```

### Check 2 (dates & timings only)

**Notebook:** System prompt and template from **check_02_result.json** (description + **example**). **build_check_system_prompt(check2_schema, CHECK_2_POLICY)** → **CHECK_2_SYSTEM**; **run_check_with_llm(llm, 2, context)** with fallback to **run_check_2** when no parsed quote or LLM fails.

Check 2 validates **dates and date ranges only** (supplier/currency/total match are in other checks):

```
Given: (1) PR header (pr_created_date, need_by_date), (2) Parsed quote (quote_date, valid_through), (3) Parsed contract if any (effective_date, end_date).

Validate:
- Quote date <= PR date (quote_date_sequence_pass).
- Quote validity/expiry >= PR date (quote_not_expired_pass).
- Quote staleness per policy (quote_staleness_warning).
- Contract effective date <= PR date (contract_effective_pass).
- Contract expiration >= PR date (contract_not_expired_pass).
- Contract covers delivery period / near expiry (covers_delivery_period_warning, near_expiry_warning).

Set check_2_status: PASS if all date/timing rules pass; FAIL if expired or wrong sequence; NEEDS_REVIEW if e.g. expiration within 30 days. Output JSON matching the Check 2 result schema (dates & timings only).
```

---

## Environment Variables

### Production (AWS)

| Variable | Purpose |
|----------|--------|
| `AWS_REGION` | Bedrock, RDS region |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (or profile) | Bedrock, S3, RDS access |
| `BEDROCK_MODEL_ID` | e.g. `anthropic.claude-3-5-haiku-20241022-v2:0` |
| `RDS_HOST`, `RDS_PORT`, `RDS_DATABASE`, `RDS_USER`, `RDS_PASSWORD` | PostgreSQL connection |
| `OPENSEARCH_ENDPOINT` (optional) | Vector search for RAG |
| `S3_BUCKET`, `S3_PREFIX` (optional) | Document store |

### Local

| Variable | Purpose |
|----------|--------|
| `GROQ_API_KEY` | Groq API (free tier LLM) |
| `LOCAL_LLM_MODEL` | e.g. `llama-3-8b-8192` (Groq) |
| `LOCAL_EMBEDDING_MODEL` | e.g. `all-MiniLM-L6-v2` (sentence-transformers) |
| `DATABASE_URL` or `LOCAL_DB_PATH` | Optional. Omit for default SQLite (`pr_validation.db`); use DBeaver to open. Or PostgreSQL URL. |

**Detection logic in notebook:** If `GROQ_API_KEY` is set and AWS Bedrock env is not configured, use Groq + local embeddings. If AWS credentials and `BEDROCK_MODEL_ID` are set, use Bedrock + Titan (or Bedrock embeddings).

---

## Repository Structure

```
PR to PO Agent/
├── README.md                          ← This file
├── document_processing_rag/
│   ├── DEVELOPMENT_TIMELINE.md        ← Product/timeline
│   ├── ENGINEERING_AWS.md             ← AWS build guide (ECS, Step Functions, RDS, etc.)
│   ├── PR_to_PO_Test.ipynb            ← Demo notebook (upload → classify; uses pr.json)
│   ├── _PR_to_PO_Master.ipynb         ← (Optional) Full pipeline (parse → DB → Check 1 & 2); uses pr.json, Check 2 dates-only
│   └── schemas/
│       ├── pr.json
│       ├── quote.json
│       ├── msa.json
│       └── compliance_checks/
│           ├── check_01_attachment_existence_classification/
│           │   └── check_01_result.json
│           ├── check_02_document_validity_applicability/
│           │   └── check_02_result.json
│           └── (check_03 … check_12 to be added)
└── synthetic_data/
    ├── compliance_check_writeups/
    │   └── PR_Compliance_Check1_Check2_Fields.md
    └── multi item example/
        └── live/
            ├── 02_CDW_Quote_Q25-0847.pdf
            └── 02b_MSA_CDW_2024_0156_Executed.pdf
```

---

## References

- **Check 1 & 2 field spec:** `synthetic_data/compliance_check_writeups/PR_Compliance_Check1_Check2_Fields.md`
- **Timeline:** `document_processing_rag/DEVELOPMENT_TIMELINE.md`
- **AWS build:** `document_processing_rag/ENGINEERING_AWS.md`

---

*Version 1.0 | January 2026*
