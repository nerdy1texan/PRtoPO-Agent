# PR to PO Agent — Purchase Requisition Validation

**Goal:** Build a PR validation AI product that ingests a Purchase Requisition (PR) plus attachments (quotes, MSAs, SOWs), classifies and parses documents into structured JSON, persists them, and runs **12 compliance checks** to determine if the PR is compliant before conversion to a Purchase Order (PO).

This repo contains **schemas**, **compliance check specs**, **synthetic data**, **architecture docs**, and a **master notebook** that runs the full pipeline locally or against AWS (Bedrock, RDS, Titan).

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Install via Conda environment file (reproducible)](#install-via-conda-environment-file-reproducible)
3. [Command-line setup (CMD and Git Bash)](#command-line-setup-windows-cmd-and-git-bash)
4. [Venv environment setup (alternative to conda)](#venv-environment-setup-alternative-to-conda)
5. [Architecture](#architecture)
6. [Tech Stacks: Production vs Local](#tech-stacks-production-vs-local)
7. [What Else Is Needed? (MCP, Vector Store, etc.)](#what-else-is-needed)
8. [Schemas & Examples](#schemas--examples)
9. [Compliance Checks](#compliance-checks)
10. [System Prompts](#system-prompts)
11. [Environment Variables](#environment-variables)
12. [Repository Structure](#repository-structure)
13. [References](#references)

---

## Getting Started

Choose **ENV_MODE**: **`aws`** (Bedrock + OpenSearch + Titan) or **`local`** (Groq/Gemini + Qdrant Cloud + sentence-transformers). Copy **`document_processing_rag/.env.example`** to **`.env`** and set the required variables for your mode (Section 0 validates them). For a **reproducible environment** with a fixed Python version on any machine, use the [Conda environment file](#install-via-conda-environment-file-reproducible). Otherwise follow **Local setup** for conda/venv + API keys + SQLite. Use **[DBeaver](https://dbeaver.io/)** to view the structured database.

---

### Install via Conda environment file (reproducible)

Use this to get the **same Python version and dependencies** on any system (Windows, macOS, Linux) without manually creating an env and installing from `requirements.txt`.

**Prerequisite:** [Miniconda](https://docs.conda.io/en/latest/miniconda.html) or Anaconda installed.

From the repo root (e.g. `PRtoPOAgent/`):

```bash
# Create the pr2po environment from the yml (Python 3.10 + all pip deps)
conda env create -f document_processing_rag/environment.yml

# Activate it
conda activate pr2po

# Go to the notebook folder
cd document_processing_rag

# Copy env template and set your keys (see .env.example)
copy .env.example .env   # Windows CMD
# cp .env.example .env  # Git Bash / macOS / Linux

# Run the notebook
jupyter notebook PR_to_PO_Master.ipynb
```

- **Python:** Pinned to **3.10** in `environment.yml` so everyone gets the same interpreter.
- **Update env later:** After changing `environment.yml`, run `conda env update -f document_processing_rag/environment.yml --prune` with `pr2po` activated.
- **Export exact pins (optional):** For fully frozen versions, after a working install run `pip freeze > requirements-frozen.txt` and use that list in your own process.

---

### Local setup (conda, API keys, SQLite + DBeaver, run)

#### 1. Conda environment

- Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html) or Anaconda if you don’t have it.
- Open a terminal in the repo root (e.g. `PR to PO Agent`).

```bash
# Create and activate a dedicated environment (Python 3.10+)
conda create -n pr2po python=3.10 -y
# If your company uses conda-forge (e.g. restricted default channels), use:
# conda create -n pr2po -c conda-forge python=3.10 -y
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

In `document_processing_rag/`, copy `.env.example` to `.env` and set values (do not commit `.env`):

```bash
cd document_processing_rag
copy .env.example .env   # Windows
# cp .env.example .env  # Git Bash / macOS / Linux
```

For **local** mode, set at least:

```env
ENV_MODE=local
GROQ_API_KEY=gsk_your_groq_api_key_here
QDRANT_URL=https://your-cluster-id.us-east-1-1.aws.cloud.qdrant.io
QDRANT_API_KEY=your_qdrant_cloud_api_key
```

- Get **Groq** key at [Groq Console](https://console.groq.com/); get **Qdrant** URL and API key from [Qdrant Cloud](https://cloud.qdrant.io/) (create a cluster, then create an API key).
- Optional: `QDRANT_COLLECTION_NAME=pr2po`, `DATABASE_URL=sqlite:///pr_validation.db` (default). See [Environment Variables](#environment-variables) and `.env.example` for the full list.

#### 7. Run the notebook

1. From the repo root or `document_processing_rag/`, open Jupyter:
   ```bash
   jupyter notebook PR_to_PO_Master.ipynb
   # or: jupyter lab
   ```
2. Run cells in order: **Section 0** (env detection & validation) → **1** (paths, LLM) → **2** (schemas) → **3** (list from input) → **4** (classify) → **5** (parse) → **5b** (save parsed outputs to output/) → **6** (structured DB: single `pr` table + structured quote/MSA) → **7** (RAG: chunking, embeddings, vector store — OpenSearch or Qdrant) → **8** (agentic orchestrator: SQL + RAG, test question) → **9** (Check 1) → **10** (Check 2) → **11** (summary) → **12** (write check results).
3. Results:
   - **output/** — `pr.json`, `parsed_quotes.json`, `parsed_msas.json` (saved in 5b); `check1_result.json`, `check2_result.json` (saved in 12).
   - **Database** — Structured tables in SQLite (`pr_validation.db`) or PostgreSQL. Open with [DBeaver](https://dbeaver.io/) to browse **pr**, **parsed_quote**, **parsed_msa**.
   - **Vector store** — Chunks indexed in Qdrant (local) or OpenSearch (AWS); used by the orchestrator in Section 8.

**Quick start checklist (local)**

1. **Environment:** **Conda (reproducible):** `conda env create -f document_processing_rag/environment.yml` → `conda activate pr2po`. **Or manual:** `conda create -n pr2po python=3.10 -y` → `conda activate pr2po`. **Venv (alternative):** See [Venv environment setup](#venv-environment-setup-alternative-to-conda).
2. If you didn’t use the yml: `cd PRtoPOAgent/document_processing_rag` → `pip install -r requirements.txt`
3. Copy `.env.example` to `.env`; set **ENV_MODE=local**, **GROQ_API_KEY** ([Groq](https://console.groq.com/)), **QDRANT_URL** and **QDRANT_API_KEY** ([Qdrant Cloud](https://cloud.qdrant.io/)).
4. (Optional) Install [DBeaver](https://dbeaver.io/) to view the database later.
5. `jupyter notebook PR_to_PO_Master.ipynb` → run all cells in order (Section 0 through 12).
6. Check **output/** for JSONs; open **pr_validation.db** in DBeaver; vector store is in Qdrant Cloud.

---

### Command-line setup (Windows CMD and Git Bash)

Use these steps to set up the **conda** environment, install dependencies, and run the notebook from **Command Prompt (CMD)** or **Git Bash**. Commands are the same in both; paths work with either forward slashes or backslashes on Windows.

#### 1. Open a terminal

- **CMD:** Press `Win + R`, type `cmd`, Enter. Or open **Command Prompt** from the Start menu.
- **Git Bash:** Right-click in the repo folder → **Git Bash Here**, or open **Git Bash** from the Start menu and `cd` to the repo.

#### 2. Go to the repo root

From your project root (adjust the path to match your machine):

```bash
# CMD or Git Bash — use your actual path
cd M:\AI_consulting\2025\Bhavin\JSONs\PRtoPOAgent
# Git Bash also accepts:
# cd /m/AI_consulting/2025/Bhavin/JSONs/PRtoPOAgent
```

#### 3. Create and activate the conda environment

```bash
conda create -n pr2po python=3.10 -y
# On company machines (restricted channels), use conda-forge:
# conda create -n pr2po -c conda-forge python=3.10 -y
conda activate pr2po
```

- **Company / conda-forge:** If your company limits default channels, use `conda create -n pr2po -c conda-forge python=3.10 -y` instead.
- **Git Bash:** If `conda activate` fails, run `conda init bash`, close and reopen Git Bash, then run `conda activate pr2po` again.

#### 4. Go to the notebook folder and install requirements

```bash
cd document_processing_rag
pip install -r requirements.txt
```

#### 5. Create the `.env` file (first time only)

Copy `.env.example` to `.env` in `document_processing_rag/` and set (for local):

```env
ENV_MODE=local
GROQ_API_KEY=gsk_your_actual_key_here
QDRANT_URL=https://your-cluster-id.us-east-1-1.aws.cloud.qdrant.io
QDRANT_API_KEY=your_qdrant_api_key
```

- **CMD:** `copy .env.example .env` then `notepad .env`.
- **Git Bash:** `cp .env.example .env` then edit in VS Code/Cursor.

#### 6. Run the notebook

From `document_processing_rag/` (same folder as `requirements.txt` and `.env`):

```bash
# Start Jupyter and open the master notebook (browser opens)
jupyter notebook PR_to_PO_Master.ipynb
```

Or use Jupyter Lab:

```bash
jupyter lab PR_to_PO_Master.ipynb
```

To run the **test** notebook (upload + classification only):

```bash
jupyter notebook PR_to_PO_Test.ipynb
```

#### 7. Run the notebook from the command line without opening the browser (optional)

To execute all cells and save output from CMD or Git Bash:

```bash
# From document_processing_rag/
jupyter nbconvert --to notebook --execute PR_to_PO_Master.ipynb --output PR_to_PO_Master_executed.ipynb
```

Or run and get a log only (no new file):

```bash
jupyter nbconvert --to notebook --execute PR_to_PO_Master.ipynb --ExecutePreprocessor.timeout=600 --stdout
```

#### Summary: one-time setup vs every session

| Step | When |
|------|------|
| `conda create -n pr2po python=3.10 -y` (or `-c conda-forge` on company machines) | Once |
| `conda activate pr2po` | Every new CMD/Git Bash window |
| `cd document_processing_rag` | When you open a new terminal |
| `pip install -r requirements.txt` | Once (or after changing requirements) |
| Copy `.env.example` to `.env`; set `GROQ_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY` | Once |
| `jupyter notebook PR_to_PO_Master.ipynb` | Whenever you want to run the notebook |

---

### Venv environment setup (alternative to conda)

If you prefer Python’s built-in **venv** instead of conda (e.g. no Miniconda/Anaconda, or company policy), use these steps. You need **Python 3.10+** on your PATH.

#### 1. Open a terminal and go to the repo

- **CMD or Git Bash (Windows):** Navigate to the repo root, then into `document_processing_rag`.
- **macOS/Linux:** Same; use `cd` to your repo path.

```bash
cd M:\AI_consulting\2025\Bhavin\JSONs\PRtoPOAgent\document_processing_rag
# Git Bash (Windows): cd /m/AI_consulting/2025/Bhavin/JSONs/PRtoPOAgent/document_processing_rag
# macOS/Linux: cd /path/to/PRtoPOAgent/document_processing_rag
```

#### 2. Create the virtual environment

From **document_processing_rag/** (same folder as `requirements.txt`):

```bash
python -m venv venv
```

This creates a `venv` folder there. To use a different name (e.g. `.venv`):

```bash
python -m venv .venv
```

#### 3. Activate the virtual environment

Activation is different per shell:

- **Windows CMD:**
  ```cmd
  venv\Scripts\activate.bat
  ```
  If you used `.venv`: `.venv\Scripts\activate.bat`

- **Windows PowerShell:**
  ```powershell
  venv\Scripts\Activate.ps1
  ```
  If execution policy blocks scripts: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` (one-time), then run the command again.

- **Git Bash (Windows):**
  ```bash
  source venv/Scripts/activate
  ```
  If you used `.venv`: `source .venv/Scripts/activate`

- **macOS / Linux:**
  ```bash
  source venv/bin/activate
  ```
  If you used `.venv`: `source .venv/bin/activate`

When active, your prompt usually shows `(venv)` or `(.venv)`.

#### 4. Install dependencies

With the venv **activated** and still in **document_processing_rag/**:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

#### 5. Create the `.env` file (first time only)

In **document_processing_rag/**, copy `.env.example` to `.env` and set (for local): **ENV_MODE=local**, **GROQ_API_KEY**, **QDRANT_URL**, **QDRANT_API_KEY**. See `.env.example` for the full list.

#### 6. Run the notebook

From **document_processing_rag/** with the venv **activated**:

```bash
jupyter notebook PR_to_PO_Master.ipynb
# or: jupyter lab PR_to_PO_Master.ipynb
# Test notebook only: jupyter notebook PR_to_PO_Test.ipynb
```

Select the kernel that points to this venv (e.g. **Python 3 (venv)** or the path containing `venv`).

#### 7. Optional: Run notebook from command line (no UI)

```bash
jupyter nbconvert --to notebook --execute PR_to_PO_Master.ipynb --output PR_to_PO_Master_executed.ipynb
```

#### Summary: venv one-time vs every session

| Step | When |
|------|------|
| `python -m venv venv` | Once (in document_processing_rag/) |
| Activate venv (e.g. `venv\Scripts\activate` on CMD, `source venv/Scripts/activate` on Git Bash) | Every new terminal |
| `pip install -r requirements.txt` | Once (or after changing requirements) |
| Copy `.env.example` to `.env`; set `GROQ_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY` | Once |
| `jupyter notebook PR_to_PO_Master.ipynb` | Whenever you want to run the notebook |

---

### AWS setup (Bedrock, OpenSearch, optional RDS)

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
| **ENV_MODE**      | `local`                             | `aws`                                      |
| **.env**          | Copy `.env.example` → `.env`; set `GROQ_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY` | Set `AWS_REGION` (or `AWS_PROFILE`), `BEDROCK_MODEL_ID`, `OPENSEARCH_ENDPOINT` |
| **LLM**           | Groq                                | Bedrock                                    |
| **Vector store**  | Qdrant Cloud                        | OpenSearch                                 |
| **Embeddings**    | sentence-transformers all-MiniLM-L6-v2 | Titan (Bedrock)                         |
| **DB**            | SQLite (default); [DBeaver](https://dbeaver.io/). Optional: PostgreSQL | RDS PostgreSQL (set `DATABASE_URL`)       |
| **Input/Output** | `document_processing_rag/input` and `output` | Same (or S3 if extended)              |

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
│  SAVE PARSED OUTPUTS (Section 5b)                                                │
│  • Write pr.json, parsed_quotes.json, parsed_msas.json to output/ immediately    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  STRUCTURED DATA STORAGE (Section 6)                                             │
│  • Single pr table (header columns + attachments_json, line_items_json)          │
│  • Structured parsed_quote, parsed_msa tables (scalars + line_items/scope JSON) │
│  • Production: RDS (PostgreSQL)  • Local: SQLite (default); view with DBeaver    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  RAG & VECTOR STORAGE (Section 7)                                                │
│  • Chunking: RecursiveCharacterTextSplitter (500 chars, 50 overlap)              │
│  • Embeddings: Titan (AWS) or sentence-transformers all-MiniLM-L6-v2 (local)     │
│  • Vector store: OpenSearch (AWS) or Qdrant Cloud (local); index chunks           │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  AGENTIC ORCHESTRATOR (Section 8)                                                │
│  • Tools: query_sql (structured DB), rag_search (vector store)                   │
│  • Orchestrator answers test questions by combining SQL + RAG; see diagram below  │
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

**Shared data layer (production):** RDS, S3, OpenSearch (vector store), Bedrock (LLM + Titan embeddings).  
**Local:** SQLite, [Qdrant Cloud](https://cloud.qdrant.io/) (vector store), Groq (LLM), sentence-transformers (embeddings). Use [DBeaver](https://dbeaver.io/) to view the structured DB.

---

### Agentic architecture: Orchestrator and agents

The notebook uses an **orchestrator** that combines **structured** (SQL) and **unstructured** (RAG) data to answer questions before compliance checks run. Diagram:

```mermaid
flowchart TB
    subgraph User
        Q[User question]
    end
    subgraph Orchestrator["Orchestrator (Section 8)"]
        O[orchestrator_answer]
    end
    subgraph Tools
        SQL[query_sql]
        RAG[rag_search]
    end
    subgraph Data
        DB[(Structured DB\npr, parsed_quote, parsed_msa)]
        VS[(Vector store\nOpenSearch or Qdrant)]
    end
    Q --> O
    O --> SQL
    O --> RAG
    SQL --> DB
    RAG --> VS
    SQL --> O
    RAG --> O
    O --> A[Answer]
```

- **Orchestrator:** Receives a question (e.g. “What is the PR number and total value? Which contract applies?”), calls **query_sql** for facts from the structured tables and **rag_search** for relevant chunks from the vector store, then uses the LLM to synthesize an answer.
- **SQL agent / tool:** Runs read-only SQL on **pr**, **parsed_quote**, **parsed_msa** (Section 6).
- **RAG agent / tool:** Runs semantic search via **rag_search(query, k=3)** on the vector store (Section 7: OpenSearch or Qdrant).
- Compliance **Check 1** and **Check 2** (Sections 9–10) run after the orchestrator; they use the same LLM and structured/RAG context as needed.

---

## Tech Stacks: Production vs Local

| Component        | Production (AWS)                    | Local (run & test)                          |
|-----------------|-------------------------------------|---------------------------------------------|
| **ENV_MODE**    | `aws`                               | `local`                                      |
| **LLM**         | Amazon Bedrock (e.g. Claude 3.5 Haiku) | Groq (free tier, e.g. llama-3)           |
| **Embeddings**  | Amazon Titan Embeddings (1536 dim)  | **sentence-transformers** **all-MiniLM-L6-v2** (384 dim) |
| **Chunking**    | RecursiveCharacterTextSplitter (500 chars, 50 overlap) | Same (LangChain) |
| **Structured DB** | Amazon RDS (PostgreSQL)           | SQLite (default); view with DBeaver. Optional: PostgreSQL |
| **Vector DB**   | **AWS OpenSearch** (k-NN index)     | **Qdrant Cloud** (collection)               |
| **Orchestrator**| SQL tool + RAG tool → LLM synthesis | Same                                         |
| **Document Store** | S3                               | Local folder (e.g. `synthetic_data/` or `input/`) |
| **Optional**    | Step Functions, Lambda (see ENGINEERING_AWS.md) | Not required for notebook |

The **master notebook** uses **ENV_MODE** (or auto-detect from env vars) and **validates required variables** in Section 0: for **local** you must set **GROQ_API_KEY**, **QDRANT_URL**, **QDRANT_API_KEY**; for **aws** you must set **AWS_REGION** (or **AWS_PROFILE**), **BEDROCK_MODEL_ID**, **OPENSEARCH_ENDPOINT**.

### Local RAG: chunking and embedding model

For **ENV_MODE=local**, RAG uses:

- **Chunking:** LangChain **RecursiveCharacterTextSplitter** — **500 characters** per chunk, **50 character** overlap. Applied to uploaded file text and parsed PR/quote/MSA summaries.
- **Embedding model:** **sentence-transformers** **all-MiniLM-L6-v2** — 384-dimensional vectors, cosine similarity in Qdrant. No API key; runs locally after `pip install sentence-transformers`. **CPU-only:** If you have no GPU, PyTorch and sentence-transformers automatically use CPU; no code changes needed.

---

## What Else Is Needed?

- **MCP server for intelligent parsing / checks**  
  An MCP (Model Context Protocol) server can expose tools such as “classify_document”, “extract_quote”, “run_compliance_check”. The notebook currently uses LangChain/LangGraph with direct LLM and Python logic; an MCP layer is **optional** and useful when you want:
  - A single parsing/validation service callable from multiple clients (IDE, API, CLI).
  - Standardized tools (e.g. “run_check_1”) that other agents or systems can invoke.
  - Clear separation between “tool definitions” and “orchestration”.  
  **Recommendation:** Start with the notebook; add an MCP server later if you need multi-client or agentic tool use.

- **Vector store / RAG**  
  **Implemented:** Production uses **AWS OpenSearch**; local uses **Qdrant Cloud**. Chunks (500 chars, 50 overlap) are embedded with Titan (AWS) or sentence-transformers **all-MiniLM-L6-v2** (local) and indexed. The **orchestrator** (Section 8) uses **rag_search** plus **query_sql** to answer questions before Check 1 and Check 2.

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

### Checks 3–6 (schemas, category config, and LLM context)

Checks **3–6** follow the same pattern as Checks 1–2:

- A **result schema** (`check_0X_result.json`) that defines the output JSON and includes an **example**.
- An **input reference** (`check_0X_input_reference.json`) that documents which fields we pass into the LLM for that check.
- A **category config** (`check_0X_*_category_config.json`) that tells the check how to behave for different categories (IT Hardware vs MRO vs Services, etc.).
- A **system prompt** built from the result schema + policy text.
- A **context JSON** built from the PR/quote/MSA/contract data plus the category config, passed in the user message.
- A **deterministic fallback** that uses the same inputs and category rules if the LLM JSON is invalid.

In **all cases**, the user message sent to the LLM looks like:

```text
Context for this compliance check:
{ ...context_json_built_from_input_reference_and_category_config... }

Produce the check result JSON matching the required structure. Output only valid JSON.
```

The context JSON keys differ per check (for example `check_3_input`, `check_4_input`, `check_5_input`, `check_6_input`), but the shape and flow are the same.

#### Check 3 — Supplier Existence & Status

- **Input reference**: `check_03_input_reference.json` lists fields like:
  - PR supplier fields (e.g. `suggested_supplier_name`, `company_code`, category).
  - Supplier from quote / contract.
  - Optional Supplier Master fields (status, approved_entities, insurance, certifications).
- **Category config**: `check_03_category_config.json` encodes, per category (§3.3/§3.4), which **insurance** and **certifications** are required and typical minimums.
- **Result schema**: `check_03_result.json` defines:
  - `check_3_status` (PASS / FAIL / WARNING / NEEDS_REVIEW).
  - Fields like `supplier_found`, `match_confidence`, `supplier_status`, `entity_authorized`.
  - `insurance_met` / `insurance_gaps` and `certifications_met` / `certification_gaps`.
  - `document_supplier_match`, `failure_reasons`, `sub_checks` (3.2.1–3.2.6), `field_level_assessment`, `evidence`.
- **Notebook flow**:
  - Builds `context_check3` with a `check_3_input` block (PR supplier, company_code, category, document supplier) and a `category_requirements` block (required insurance/certs for the PR’s category), plus raw `pr_header`, `pr_line_items`, `parsed_quotes`, and `parsed_msas`.
  - `build_check_system_prompt(check3_schema, CHECK_3_POLICY)` → `CHECK_3_SYSTEM`; `run_check_with_llm(llm, 3, context_check3)` runs the LLM path.
  - Deterministic `run_check_3(...)` uses the same inputs and category config as a fallback when the LLM output can’t be validated.

#### Check 4 — Item Description Match

- **Input reference**: `check_04_input_reference.json` identifies:
  - PR line items: `line_number`, short/long `description`, `manufacturer`.
  - Quote line items: `line_number`, `description`, `manufacturer`.
  - SOW fields for services: `scope_summary`, `in_scope[]`, `out_of_scope[]`.
  - Contract catalog items: `catalog_items[]` with `sku`, `description`, `manufacturer`.
- **Category config**: `check_04_category_config.json` captures catalog/SKU applicability (§4.4):
  - IT Hardware / IT: approved product list, **part + manufacturer** critical.
  - MRO: storeroom catalog / VMI.
  - Office Supplies: punch‑out catalog.
  - Services / Professional Services: rate card / SOW scope.
- **Result schema**: `check_04_result.json` defines:
  - `check_4_status` and booleans: `description_semantic_match`, `manufacturer_match`, `line_count_alignment`, `scope_coverage_services`.
  - `failure_reasons`, `sub_checks` (4.1, 4.3, 4.4, 4.6), `field_level_assessment`, `evidence`.
- **Notebook flow**:
  - Builds `check_4_input` with simplified PR/quote line items and any SOW scope, plus `category_catalog_note` from the category config.
  - LLM compares descriptions, manufacturers, and scope for the given category; fallback `run_check_4(...)` implements a conservative rule‑based version of the same logic.

#### Check 5 — Quantity & Unit of Measure Match

- **Input reference**: `check_05_input_reference.json` specifies:
  - PR line `quantity` and `unit_of_measure`.
  - PR category (for category‑specific logic).
  - Quote line `quantity` and `unit_of_measure`.
  - Optional contract MOQ / max qty / increment / units_per_case and item‑master default UOM.
- **Category config**: `check_05_category_config.json` encodes §5.5 rules:
  - MRO / Office Supplies: CS vs EA warnings.
  - Services / Professional Services: **time‑based** UOM for T&M (HRS, DAY, WK, MO).
  - Print / Marketing: valid print increments (e.g. 500, 1000).
  - Plus a shared `uom_equivalences` map (EA/EACH/PC, CS/CASE, BX/BOX, HRS/HOURS, etc.).
- **Result schema**: `check_05_result.json` defines:
  - `check_5_status` and booleans: `quantity_match` (5.1), `partial_order_ok` (5.2), `uom_match` (5.3), `uom_price_alignment` (5.7).
  - `failure_reasons`, `sub_checks`, `field_level_assessment`, `evidence`.
- **Notebook flow**:
  - Builds `check_5_input` with PR/quote quantity + UOM and a `category_specific_note` from the category config.
  - LLM evaluates 5.1–5.3, 5.7 plus category‑specific rules; fallback `run_check_5(...)`:
    - Compares quantities (exact match and PR ≤ quote per line).
    - Uses `_uom_equivalent` + category rules for UOM and print increments.
    - Sets `check_5_status` to PASS / FAIL / WARNING accordingly.

#### Check 6 — Unit Price Match

- **Input reference**: `check_06_input_reference.json` lists:
  - PR line `unit_price` and `extended_amount`, PR `currency`, PR category.
  - Quote `unit_price` and `currency`.
  - Optional contract price/list/discount.
  - Optional rate‑card role + approved rate and validity dates.
- **Category config**: `check_06_category_config.json` maps categories to **pricing models**:
  - IT Hardware / Office Supplies: **catalog** (discount off list; compare to quote).
  - MRO / Facilities / Construction: **contract** (unit price ceiling).
  - Professional Services / IT Services / Staffing / Services: **rate card**.
  - Direct Materials: **negotiated/index‑based**.
- **Result schema**: `check_06_result.json` defines:
  - `check_6_status` and booleans: `quote_price_match`, `contract_ceiling_ok`, `rate_card_ok`, `discount_applied`, `no_markup_vs_quote`, `currency_alignment`.
  - `failure_reasons`, `sub_checks` (6.1–6.6), `field_level_assessment`, `evidence`.
- **Notebook flow**:
  - Builds `check_6_input` with PR/quote prices and currencies plus `category_pricing_note` from the category config.
  - LLM enforces the 6.1–6.6 rules for the appropriate pricing model; fallback `run_check_6(...)`:
    - Enforces 6.1 (±1% or $1 tolerance) and 6.5 (no markup) on PR vs quote.
    - Flags obvious 6.6 currency mismatches.
    - Fills the structured result JSON defined by the schema.

---

## Environment Variables

Use **`document_processing_rag/.env.example`** as the template: copy to **`.env`** and set values for your chosen mode. **Section 0** validates required vars and fails with a clear error if any are missing.

### Mode selection

| Variable | Purpose |
|----------|--------|
| `ENV_MODE` | `aws` or `local`. If unset, auto-detected from which block of vars is present. |

### Required when ENV_MODE=aws

| Variable | Purpose |
|----------|--------|
| `AWS_REGION` or `AWS_PROFILE` | Bedrock and OpenSearch region / profile |
| `BEDROCK_MODEL_ID` | e.g. `anthropic.claude-3-5-haiku-20241022-v2:0` |
| `OPENSEARCH_ENDPOINT` or `OPENSEARCH_HOST` | OpenSearch URL for vector store |

Optional: `OPENSEARCH_INDEX_NAME`, `OPENSEARCH_USER`, `OPENSEARCH_PASSWORD`, `DATABASE_URL` (RDS), `S3_BUCKET`, `S3_PREFIX`.

### Required when ENV_MODE=local

| Variable | Purpose |
|----------|--------|
| `GROQ_API_KEY` | Groq API key (free tier LLM); from [Groq Console](https://console.groq.com/) |
| `QDRANT_URL` | Qdrant Cloud cluster URL (e.g. `https://xxx.us-east-1-1.aws.cloud.qdrant.io`) |
| `QDRANT_API_KEY` | Qdrant Cloud API key; from [Qdrant Cloud](https://cloud.qdrant.io/) (create cluster → API key) |

Optional: `QDRANT_COLLECTION_NAME=pr2po`, `LOCAL_LLM_MODEL`, `DATABASE_URL` or `LOCAL_DB_PATH` (default SQLite).

### Optional (both modes)

| Variable | Purpose |
|----------|--------|
| `DATABASE_URL` | PostgreSQL URL. Omit for default SQLite (`pr_validation.db`); view with [DBeaver](https://dbeaver.io/). |

---

## Repository Structure

```
PRtoPOAgent/
├── README.md                              ← This file
├── document_processing_rag/
│   ├── .env.example                      ← Env template (copy to .env); ENV_MODE, AWS vs local vars
│   ├── PR_to_PO_Master.ipynb             ← Full pipeline: §0–12 (env → parse → save → structured DB → RAG → orchestrator → Check 1 & 2)
│   ├── PR_to_PO_Test.ipynb               ← Demo (upload → classify)
│   ├── requirements.txt                  ← pip dependencies (used by manual setup)
│   ├── environment.yml                   ← Conda env: exact Python 3.10 + pip deps (reproducible across systems)
│   ├── schemas/
│   │   ├── pr.json
│   │   ├── quote.json
│   │   ├── msa.json
│   │   └── compliance_checks/
│   │       ├── check_01_attachment_existence_classification/
│   │       │   └── check_01_result.json
│   │       ├── check_02_document_validity_applicability/
│   │       │   └── check_02_result.json
│   │       └── (check_03 … check_12 to be added)
│   ├── DEVELOPMENT_TIMELINE.md
│   └── ENGINEERING_AWS.md
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
