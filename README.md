# CloudOps Sentinel — Enterprise Incident Response Self-RAG Copilot

A production-style **Self-RAG** demo for cloud operations and incident response. It searches private operational knowledge in **Pinecone** first, self-grades the retrieved evidence, corrects weak retrieval or generation, and falls back to **internet search only when the private knowledge base is insufficient**.

Conversation state is persisted with **LangGraph + SQLite**, so follow-up questions reuse the same incident context through a stable `thread_id`.

---

## Table of Contents

1. [Key Features](#key-features)
2. [Architecture](#architecture)
3. [Self-RAG Decision Points](#self-rag-decision-points)
4. [Tech Stack](#tech-stack)
5. [Project Structure](#project-structure)
6. [Setup](#1-setup)
7. [Build the Knowledge Base](#2-build-the-knowledge-base-with-one-command)
8. [Run the Application](#3-run-the-application)
9. [Run with Docker](#4-run-with-docker)
10. [Add Documents from the UI](#add-documents-from-the-ui)
11. [Persistent Memory](#langgraph-sqlite-persistence-memory)
12. [Demo Flow](#demo-flow)
13. [Configuration Consistency](#important-configuration-rule)
14. [Troubleshooting](#troubleshooting)
15. [Production Notes](#production-notes)

---

## Key Features

| Feature | Description |
|---|---|
| **Private-first retrieval** | Runbooks, SOPs, and postmortems in Pinecone are always searched before anything else |
| **Self-grading** | Retrieved documents are graded for relevance before they reach generation |
| **Self-correction** | Weak retrieval triggers query rewriting and a retry; unsupported answers trigger revision |
| **Controlled web fallback** | Tavily internet search is used only after the private KB fails |
| **Persistent memory** | SQLite-backed LangGraph checkpoints keep incident context per `thread_id` |
| **Live knowledge updates** | Upload PDF / TXT / MD / DOCX from the UI straight into the existing Pinecone namespace |
| **Incident command UI** | HTML/CSS/JS front end served by FastAPI |
| **Containerized** | Dockerfile and docker-compose included |

---

## Architecture

```text
User Incident / Follow-up
        ↓
LangGraph SQLite Memory
        ↓
Contextualize Follow-up
        ↓
Decide Retrieval
        ↓
Private Pinecone Knowledge Base
        ↓
Grade Retrieved Documents
   ┌────┴───────────────┐
Relevant             Weak / Missing
   ↓                      ↓
Generate              Rewrite Query
   ↓                      ↓
IsSUP              Retry Private KB
   ↓                      ↓
Revise if needed   Internet Search Fallback
   ↓                      ↓
IsUSE              Grade Web Evidence
   ↓                      ↓
Final Answer ← Generate → IsSUP → IsUSE
        ↓
SQLite Checkpoint / Memory
```

### Source routing at a glance

| Situation | Route |
|---|---|
| Private documents are relevant | Private Runbooks (Pinecone) |
| Private documents are weak → rewritten query succeeds | Private Runbooks (Pinecone) |
| Private documents remain insufficient after retry | Internet Search (Tavily) |

---

## Self-RAG Decision Points

| Step | Purpose | Outcome |
|---|---|---|
| **Contextualize** | Turns a follow-up into a standalone operational question using persisted thread context | Standalone query |
| **Decide Retrieval** | Determines whether retrieval is needed | Retrieve / skip |
| **Relevance grading** | Scores each retrieved chunk against the question | Keep / discard |
| **Rewrite Query** | Reformulates the question when evidence is weak | Retry private KB |
| **IsSUP** | Checks that the answer is supported by the evidence | Pass / revise |
| **IsUSE** | Checks that the answer actually helps the user | Pass / regenerate |

---

## Tech Stack

| Layer | Technology | Role |
|---|---|---|
| Orchestration | **LangGraph** | Self-RAG workflow, conditional routing, persistent thread state |
| LLM | **OpenAI `gpt-5-mini`** | Routing, query rewriting, generation, relevance grading, IsSUP, IsUSE |
| Embeddings | **OpenAI `text-embedding-3-large`** | Embeddings for private operational documents |
| Vector DB | **Pinecone** | Private runbook / SOP / postmortem knowledge base |
| Web search | **Tavily** | Controlled internet-search fallback |
| Persistence | **SQLite** | LangGraph checkpoints + simple audit database |
| Backend | **FastAPI** | Application and API layer |
| Frontend | **HTML / CSS / JavaScript** | Incident command center with document upload |
| Packaging | **Docker** | Deployment |

---

## Project Structure

```text
CloudOps-Sentinel-Enterprise-Incident-Response-Self-RAG-Copilot/
├── app.py                     # FastAPI app entry point
├── data_ingestion.py          # ONE file to build the Pinecone KB
├── src/
│   ├── config.py              # Settings loaded from .env
│   ├── db.py                  # SQLite audit database
│   ├── ingestion.py           # Load → chunk → embed → upsert
│   ├── models.py              # Request / response models
│   ├── self_rag.py            # LangGraph Self-RAG workflow
│   └── vectorstore.py         # Shared embeddings + Pinecone config
├── documents/                 # Initial private knowledge documents
│   ├── checkout-api-runbook.md
│   ├── payments-high-cpu-runbook.md
│   └── deployment-rollback-sop.md
├── templates/index.html
├── static/styles.css
├── static/app.js
├── uploads/                   # Documents uploaded through the UI
├── data/                      # SQLite persistence files
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

---

## 1. Setup

### Prerequisites

- Python 3.10+
- OpenAI, Pinecone, and Tavily API keys
- Docker (optional, for containerized runs)

### Create a virtual environment

```bash
python -m venv venv
```

Activate it:

```bash
# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate
```

### Install dependencies

```bash
pip install -r requirements.txt
```

### Configure environment variables

Copy `.env.example` to `.env` and add your keys:

```env
OPENAI_API_KEY=...
PINECONE_API_KEY=...
TAVILY_API_KEY=...
```

| Variable | Default | Required | Description |
|---|---|---|---|
| `OPENAI_API_KEY` | — | Yes | OpenAI access for LLM + embeddings |
| `PINECONE_API_KEY` | — | Yes | Pinecone access |
| `TAVILY_API_KEY` | — | Yes | Internet-search fallback |
| `OPENAI_MODEL` | `gpt-5-mini` | No | Model used for all LLM steps |
| `EMBEDDING_MODEL` | `text-embedding-3-large` | No | Embedding model |
| `EMBEDDING_DIMENSION` | `3072` | No | Must match the Pinecone index dimension |
| `PINECONE_INDEX_NAME` | `cloudops-sentinel-openai-self-rag` | No | Pinecone index |
| `PINECONE_NAMESPACE` | `incident-runbooks` | No | Namespace for runbook vectors |

> **Note:** The project intentionally uses a new default Pinecone index name. The earlier local-embedding version used a different vector dimension, and Pinecone index dimensions cannot be mixed.

---

## 2. Build the Knowledge Base with ONE Command

Place your initial PDF / TXT / MD / DOCX files inside `documents/`, then run:

```bash
python data_ingestion.py
```

This single script runs the complete ingestion pipeline:

```text
Read .env
  ↓
Create / validate Pinecone index
  ↓
Load ./documents
  ↓
Split documents into chunks
  ↓
OpenAI text-embedding-3-large
  ↓
Upsert to Pinecone
  ↓
Configured namespace is ready
```

Ingestion uses **stable chunk IDs**, so re-running the script updates matching vectors instead of creating a new random ID for every chunk.

---

## 3. Run the Application

```bash
python app.py
```

or, with auto-reload for development:

```bash
uvicorn app:app --reload
```

Open the UI:

```text
http://127.0.0.1:8000
```

---

## 4. Run with Docker

```bash
docker compose up --build
```

Make sure `.env` is configured first, and that the knowledge base has been built (step 2) so the Pinecone namespace is populated before you start asking questions.

---

## Add Documents from the UI

The left-side **Runbook Vault** accepts:

| Format | Supported |
|---|---|
| PDF | ✅ |
| TXT | ✅ |
| Markdown | ✅ |
| DOCX | ✅ |

When you click **Add to knowledge base**, the backend uses the exact same ingestion layer and OpenAI embedding model as `data_ingestion.py`:

```text
New UI Document
      ↓
FastAPI /api/upload
      ↓
Document Loader + Chunking
      ↓
OpenAI Embeddings
      ↓
EXISTING Pinecone Index
      ↓
EXISTING Namespace
```

You can prepare the initial knowledge base ahead of time, upload another runbook live, and immediately ask questions against the expanded knowledge base.

---

## LangGraph SQLite Persistence Memory

Every browser incident session has a persistent `thread_id`. The LangGraph workflow is compiled with a SQLite checkpointer that stores checkpoints in:

```text
data/langgraph_memory.sqlite
```

### Example

| Turn | Message |
|---|---|
| 1 | *Our checkout API is returning 502 errors after deployment. What should I check first?* |
| 2 | *What should I check next if that doesn't work?* |

For the second message, the `contextualize` node uses the persisted incident context and rewrites the follow-up into a standalone operational question before continuing through Self-RAG.

> SQLite is intentionally used for the local demonstration. In production, this persistence layer can be swapped for PostgreSQL or another production-grade LangGraph checkpointer.

---

## Demo Flow

| Demo | Goal | Prompt / Action | Expected Route |
|---|---|---|---|
| **A — Existing Pinecone knowledge** | Show private-first retrieval | *Our checkout API is returning 502 errors after deployment. What should the on-call engineer check first?* | Private Runbooks |
| **B — Persistent memory** | Show follow-up context reuse | *What should I check next if that does not work?* (same `thread_id`) | Private Runbooks, with prior incident context |
| **C — Upload new knowledge** | Show live KB expansion | Upload a PDF / MD / DOCX, then ask a question answered only by that document | Private Runbooks (new chunks) |
| **D — Internet-search fallback** | Show corrective routing | Ask about a technical issue not covered by any private runbook | Private KB → rewrite → Internet Search |

For Demo C, the UI status reports how many chunks were added to the existing namespace.

---

## Important Configuration Rule

The application and the ingestion script both import `src/vectorstore.py`, so they always share the same:

| Setting | Shared by app and ingestion |
|---|---|
| OpenAI embedding model | ✅ |
| Embedding dimension | ✅ |
| Pinecone index | ✅ |
| Pinecone namespace | ✅ |

This prevents a common RAG mistake where offline ingestion and runtime retrieval use different embeddings or different namespaces.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Pinecone dimension mismatch error | Index was created with a different embedding dimension | Use a new `PINECONE_INDEX_NAME`, or set `EMBEDDING_DIMENSION=3072` to match `text-embedding-3-large` |
| Answers always come from the web | Namespace is empty or ingestion was never run | Run `python data_ingestion.py` and confirm `PINECONE_NAMESPACE` |
| Uploaded document not found in answers | App and ingestion use different index / namespace | Verify both read the same `.env` values |
| Follow-up questions lose context | `thread_id` changed (e.g. new browser session) | Reuse the same session / `thread_id` |
| Authentication errors | Missing or invalid API keys | Re-check `OPENAI_API_KEY`, `PINECONE_API_KEY`, `TAVILY_API_KEY` in `.env` |

---

## Production Notes

| Area | Demo Setup | Production Suggestion |
|---|---|---|
| Checkpointer | SQLite | PostgreSQL or another production-grade LangGraph checkpointer |
| Audit log | SQLite | Managed relational database |
| Secrets | `.env` file | Secret manager (e.g. a cloud key vault) |
| Web fallback | Tavily, open | Allow-listed domains and rate limits |
| Observability | Basic | Tracing, metrics, and grading-outcome logging |
