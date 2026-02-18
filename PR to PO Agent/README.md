# PR to PO Agent — Purchase Requisition Validation

**Goal:** Build a PR validation AI product that ingests a Purchase Requisition (PR) plus attachments (quotes, MSAs, SOWs), classifies and parses documents into structured JSON, persists them, and runs **12 compliance checks** to determine if the PR is compliant before conversion to a Purchase Order (PO).

This repo contains **schemas**, **compliance check specs**, **synthetic data**, **architecture docs**, and a **master notebook** that runs the full pipeline locally or against AWS (Bedrock, RDS, Titan).

---

## Table of Contents

1. [Architecture](#architecture)
2. [Tech Stacks: Production vs Local](#tech-stacks-production-vs-local)
3. [What Else Is Needed? (MCP, Vector Store, etc.)](#what-else-is-needed)
4. [Schemas & Examples](#schemas--examples)
5. [Compliance Checks](#compliance-checks)
6. [System Prompts](#system-prompts)
7. [Environment Variables](#environment-variables)
8. [Repository Structure](#repository-structure)
9. [How to Run](#how-to-run)
10. [References](#references)

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
│  • Output: pr_header, pr_line_items, pr_attachments, quote(s), msa(s)             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  DATABASE STRUCTURED ENTRY                                                       │
│  • Persist parsed PR header, line items, attachments metadata                     │
│  • Persist parsed quote(s), MSA(s) for audit and validation                      │
│  • Production: Amazon RDS (PostgreSQL)  • Local: PostgreSQL or SQLite             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PR VALIDATION ENGINE (12 Compliance Checks)                                     │
│  • Check 1:  Attachment Existence & Classification (gatekeeper)                  │
│  • Check 2:  Document Validity & Applicability (dates, supplier, entity)         │
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
**Local:** SQLite or local PostgreSQL for structured data; local embedding model; Groq or local LLM.

---

## Tech Stacks: Production vs Local

| Component        | Production (AWS)                    | Local (run & test)                          |
|-----------------|-------------------------------------|---------------------------------------------|
| **Orchestration** | LangChain, LangGraph                | LangChain, LangGraph                        |
| **LLM**          | Amazon Bedrock (e.g. Claude 3.5 Haiku) | Groq (free tier, e.g. llama-3)           |
| **Embeddings**   | Amazon Titan Embeddings             | sentence-transformers (e.g. all-MiniLM-L6-v2) or PostgresML |
| **Structured DB**| Amazon RDS (PostgreSQL)             | PostgreSQL (local) or SQLite                |
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
| **PR Header** | `schemas/pr_header.json` | PR form header: pr_number, dates, requestor, company, cost center, total, currency, payment terms, need_by_date, attachments count, buying_channel, contract_reference, category L1–L4. Feeds checks 1–12. |
| **PR Line Items** | `schemas/pr_line_items.json` | Per-line: description, quantity, UOM, unit_price_usd, extended_amount_usd, supplier, category, delivery. Feeds checks 4, 5, 6, 9, 12. |
| **PR Attachments** | `schemas/pr_attachments.json` | List of attachments: file_name, file_type, document_classification, upload_date, document_date. Feeds checks 1, 11. |
| **Quote** | `schemas/quote.json` | Parsed quotation: quote_number, quote_date, valid_through, supplier_name, currency, payment_terms, contract_ref, line_items (description, qty, UOM, unit_price, extended_price), total. Feeds checks 2, 3, 4, 5, 6, 7, 8. |
| **MSA** | `schemas/msa.json` | Parsed Master Service Agreement: agreement_id, parties, effective_date, end_date, payment_terms_standard, maximum_contract_value, scope, lead times, warranties, returns, termination. Feeds checks 8, 10. |

Use these as **templates + one-shot**: send the schema (and optionally the `example`) to the LLM so it returns JSON conforming to the schema.

### Compliance check result schemas

| Check | Path | Purpose |
|-------|------|--------|
| **Check 1** | `schemas/compliance_checks/check_01_attachment_existence_classification/check_01_result.json` | Output: check_1_status, attachments_found, classified_documents[], policy_requirements_met, missing_requirements[], invoice_detected, plus UI (sub_checks, field_level_assessment, evidence). |
| **Check 2** | `schemas/compliance_checks/check_02_document_validity_applicability/check_02_result.json` | Output: check_2_status, quotation_validity, contract_validity, applicability, sub_checks, field_level_assessment, evidence, policy_reference. |

Checks 3–12 will follow the same pattern: each in its own folder with a result schema and example.

---

## Compliance Checks

| # | Name | Brief description |
|---|------|-------------------|
| 1 | Attachment Existence & Classification | Gatekeeper: required doc types present per policy (value/category); no full extraction. |
| 2 | Document Validity & Applicability | Quote/contract valid (dates, status) and applicable (supplier, entity, category). |
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

### Check 2 (validity & applicability)

**Notebook:** System prompt and template from **check_02_result.json** (description + **example**). **build_check_system_prompt(check2_schema, CHECK_2_POLICY)** → **CHECK_2_SYSTEM**; **run_check_with_llm(llm, 2, context)** with fallback to **run_check_2** when no parsed quote or LLM fails.

```
Given: (1) PR header (pr_created_date, company_code, currency, total_estimated_value, need_by_date), (2) PR suggested supplier, (3) Parsed quote (quote_date, valid_through, supplier_name, currency, total, customer_name/bill_to).

Validate:
- Quote date <= PR date (quote_date_sequence_pass).
- Quote validity/expiry >= PR date (quote_not_expired_pass).
- Quote addressee matches PR company (quote_addressee_match_pass).
- Quote currency matches PR currency (quote_currency_match_pass).
- Quote total within ±1% or ±$100 of PR total (quote_total_match_pass).
- Quote supplier fuzzy matches PR supplier >= 85% (quote_supplier_match_pass).

Set check_2_status: PASS if all pass; FAIL if any critical fail; NEEDS_REVIEW if e.g. expiration within 30 days or fuzzy match 80–85%. Output JSON matching the Check 2 result schema, including sub_checks, field_level_assessment, and evidence.
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
| `DATABASE_URL` or `LOCAL_DB_PATH` | PostgreSQL URL or SQLite path (e.g. `sqlite:///pr_validation.db`) |

**Detection logic in notebook:** If `GROQ_API_KEY` is set and AWS Bedrock env is not configured, use Groq + local embeddings. If AWS credentials and `BEDROCK_MODEL_ID` are set, use Bedrock + Titan (or Bedrock embeddings).

---

## Repository Structure

```
PR to PO Agent/
├── README.md                          ← This file
├── document_processing_rag/
│   ├── DEVELOPMENT_TIMELINE.md        ← Product/timeline
│   ├── ENGINEERING_AWS.md             ← AWS build guide (ECS, Step Functions, RDS, etc.)
│   ├── PR_to_PO_Master.ipynb          ← Master notebook (upload → classify → parse → DB → Check 1 & 2)
│   └── schemas/
│       ├── pr_header.json
│       ├── pr_line_items.json
│       ├── pr_attachments.json
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

## How to Run

1. **Clone / open** the repo; ensure Python 3.10+.
2. **Install dependencies** (see notebook or below):
   - `langchain`, `langchain-community`, `langchain-groq` (and optionally `langchain-aws` for Bedrock)
   - `pypdf` or `pdfplumber` for PDF text
   - `python-dotenv` for env
   - For local DB: `sqlalchemy`; for Postgres: `psycopg2-binary`
3. **Set environment:**
   - Local: set `GROQ_API_KEY` and optionally `DATABASE_URL` or `LOCAL_DB_PATH`.
   - Production: set AWS credentials and `BEDROCK_MODEL_ID`, RDS vars.
4. **Open** `document_processing_rag/PR_to_PO_Master.ipynb` and run cells in order.  
   The notebook will detect env and run document upload → categorization → parsing → DB → Check 1 → Check 2.

---

## References

- **Check 1 & 2 field spec:** `synthetic_data/compliance_check_writeups/PR_Compliance_Check1_Check2_Fields.md`
- **Timeline:** `document_processing_rag/DEVELOPMENT_TIMELINE.md`
- **AWS build:** `document_processing_rag/ENGINEERING_AWS.md`

---

*Version 1.0 | January 2026*
