# PR to PO — Engineering: AWS (Synchronous Part)

This document describes what the **engineering team should start building on AWS** for the **synchronous (App Chat User)** PR validation flow, so they can begin while product/timeline work continues. It aligns with the PR to PO Agent Architecture (FastAPI on ECS Fargate, PR Validation Engine on Step Functions + Lambda, Shared Data Layer).

**Tech stack:** LangChain/LangGraph, AWS Bedrock, OpenSearch, RDS, S3.

---

## 1. FastAPI on ECS Fargate (Entry Point)

| Task | Details |
|------|--------|
| **ECS cluster + Fargate service** | Create an ECS cluster (if one doesn’t exist) and a Fargate service that runs the FastAPI app as the PR validation API. |
| **Task definition** | Single task definition: FastAPI container image, 0.5 vCPU / 1 GB RAM to start; env vars for environment (e.g. stage), and placeholders for `client_id` resolution later. |
| **ALB** | Put an Application Load Balancer in front of the ECS service (HTTPS listener, target group pointing at the Fargate service). |
| **S3 for PR attachments** | Dedicated bucket or prefix for synchronous PR uploads: e.g. `s3://bucket/{client_id}/pr_attachments/{pr_id}/`. Define retention/lifecycle as needed. |
| **IAM** | Task role for the FastAPI container: write to the PR-attachments S3 path, permission to start Step Functions executions. Add OpenSearch/RDS/Bedrock only if the API calls them directly; otherwise limit to S3 + Step Functions for now. |

**Goal:** User can hit the FastAPI API to upload a PR and attachments; the API writes files to S3 and kicks off the PR Validation Engine. No need to implement “Document Processor” logic inside FastAPI yet—just upload + trigger.

---

## 2. PR Validation Engine — Step Functions + Lambdas (Skeleton)

| Task | Details |
|------|--------|
| **Step Functions state machine** | One state machine “PR Validation” that implements the flow: **Classify Attachments → Extract Fields → Load Reference Data from VectorDB → Run 12 Compliance/Validation Checks → Score & Aggregate → Provide Explanations**. Start with **placeholder Lambdas** that log input and return a minimal output so the chain runs end-to-end. |
| **Six Lambda functions (stubs)** | One Lambda per step. Each receives the Step Functions payload, logs it, returns a minimal JSON (e.g. `{"step": "classify", "status": "ok"}`). Suggested names: `pr-classify-attachments`, `pr-extract-fields`, `pr-load-reference-data`, `pr-run-compliance-checks`, `pr-score-aggregate`, `pr-provide-explanations`. |
| **Invocation from FastAPI** | After upload, FastAPI calls Step Functions `StartExecution` with input containing at least `client_id`, `pr_id`, and the S3 paths of the uploaded files. Use a naming convention for the execution name (e.g. include `pr_id`) for traceability. |
| **IAM** | Step Functions execution role with permission to invoke each Lambda. Each Lambda’s execution role with the minimum it will need (S3, Bedrock, OpenSearch, RDS—add as you implement). |

**Goal:** The synchronous flow is “visible” in AWS: upload triggers Step Functions, and all six steps execute (as stubs). No real logic yet.

---

## 3. Shared Data Layer — Connectivity and Structure

| Task | Details |
|------|--------|
| **OpenSearch** | Provision (or reuse) an OpenSearch domain. Ensure it’s in the same VPC as the Lambdas (or that Lambdas can reach it via VPC peering / security groups). Create the index used for vector search; include `client_id` in every document so retrieval can be filtered by client. Embedding dimension must match your embedding model (e.g. 1536 for Titan). The Lambdas don’t need to index real data yet—only “index exists and Lambdas can query it.” |
| **RDS (PostgreSQL)** | Provision (or reuse) an RDS instance in the same VPC. Create the **tables** the architecture expects: e.g. `documents_metadata`, `parsed_contracts`, `parsed_sows`, `parsed_service_agreements`, `supplier_master`, `contract_master`, `category_cards`, `catalog_data`, `semantic_layer_config`, `compliance_check_config`, `threshold_config`, `hil_config`, and any table for document chunks if you store them in RDS. Use the project’s synthetic_data as reference. No need to load real data yet—just schema and `client_id` on every table. |
| **S3** | Besides the PR-attachments bucket/prefix, confirm paths for raw onboarding documents and (if applicable) chat logs / audit trail so they’re ready when those features are added. |
| **Bedrock** | Enable the required models (e.g. Claude 3.5 Haiku, Titan Embeddings) in the account/region. Grant the Lambda roles that will call Bedrock the necessary `bedrock:InvokeModel` (and optionally `bedrock:InvokeModelWithResponseStream`) permissions. |

**Goal:** All services the synchronous flow will call exist and are reachable from the Lambdas; RDS has the right schema; OpenSearch has an index with `client_id`.

---

## 4. Lambda–Service Wiring (No Business Logic Yet)

| Task | Details |
|------|--------|
| **Classify / Extract** | These Lambdas should read the S3 paths from the Step Functions input and, at minimum, do a head/get of the object (or a small byte range) so they’re “S3-aware.” No need to implement real classification or extraction yet. |
| **Load Reference Data** | Lambda connects to OpenSearch and RDS; for the `client_id` in the input, run a trivial query (e.g. “return one dummy row” or “return 0 chunks”) so connectivity and IAM are validated. |
| **Run 12 Checks** | Lambda receives the payload from the previous step and returns a fixed list of 12 “placeholder” check results (e.g. all PASS or all SKIP) so the next step has a consistent structure to consume. |
| **Score & Aggregate** | Lambda takes the placeholder check results and returns a minimal summary (e.g. `overall_status`, `recommended_path`). |
| **Provide Explanations** | Lambda calls Bedrock once with a trivial prompt and returns a short string so the “Lambda + LLM” path is tested. |

**Goal:** The synchronous pipeline is end-to-end on AWS with real S3/OpenSearch/RDS/Bedrock calls and placeholder logic. You can then replace placeholders with real document processing, RAG, 12 checks, and LLM integration as per the timeline.

---

## 5. Observability and Safety

| Task | Details |
|------|--------|
| **Logging** | Ensure each Lambda logs to CloudWatch (and that log groups exist). Optionally add a small amount of structured logging (e.g. `client_id`, `pr_id`, `step`) for easier debugging. |
| **Step Functions** | Use execution history and (optional) CloudWatch alarms on failed executions so the team sees when the chain breaks. |
| **Timeouts** | Set Lambda timeouts and Step Functions timeout so long-running or stuck runs don’t run forever (e.g. 5 min per Lambda, 15 min for the state machine). |

---

## Suggested Order of Work

| Phase | Focus |
|-------|--------|
| **Week 1 (foundation)** | ECS Fargate + FastAPI (upload + S3 + trigger Step Functions). Step Functions state machine + six stub Lambdas. S3 bucket/prefix for PR attachments and IAM for FastAPI + Step Functions + Lambdas. |
| **Week 2 (data layer)** | RDS: create schema (tables above) with `client_id` on every table. OpenSearch: create domain + index with `client_id` and correct embedding dimension. Bedrock: enable models and add permissions to Lambda roles. |
| **Week 3 (wiring)** | Each Lambda: minimal S3/OpenSearch/RDS/Bedrock usage as above; Step Functions input/output passed through. One end-to-end test: upload a file → Step Functions runs → all steps succeed and “Provide Explanations” returns a short Bedrock-generated text. |

After that, engineering can swap stubs for real logic (document processing, RAG, 12 semantic checks, semantic layers) in line with the main **DEVELOPMENT_TIMELINE.md**.

---

## Reference: Synchronous Flow (Architecture)

```
PR Validation Documents Upload (Quote, SOW, Invoice, etc.)
         │
         ▼
FastAPI Service (ECS Fargate)
  • Upload & Ingest  →  S3 ({client_id}/pr_attachments/{pr_id}/)
  • Document Processor  →  triggers PR Validation Engine
  • Results & Actions   (later: show results to user)
  • System Status      (later: health/progress)
         │
         ▼
PR Validation Engine (Step Functions)
  1. Classify Attachments (Lambda)
  2. Extract Fields (Lambda)
  3. Load Reference Data from VectorDB (Lambda)  →  OpenSearch + RDS
  4. Run 12 Compliance/Validation Checks (Lambda)  →  RDS config, Semantic Layer
  5. Score & Aggregate (Lambda)
  6. Provide Explanations (Lambda + LLM)  →  Bedrock
         │
         ▼
Shared Data Layer: OpenSearch, Bedrock, RDS, S3
```

---

**Document version:** 1.0  
**Last updated:** January 2026  
**Related:** `DEVELOPMENT_TIMELINE.md`
