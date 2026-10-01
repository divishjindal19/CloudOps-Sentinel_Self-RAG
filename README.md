# 🚨 CloudOps Sentinel — Enterprise Incident Response Self-RAG Copilot

CloudOps Sentinel is a production-style **Self-RAG (Self-Reflective Retrieval-Augmented Generation)** copilot designed for cloud operations, DevOps, and incident-response teams.

It helps engineers troubleshoot production incidents by first searching the organization's **private operational knowledge base**, evaluating the quality of retrieved evidence, correcting weak retrieval when necessary, and using **internet search only when internal knowledge is insufficient**.

The system also maintains **persistent incident context** using LangGraph and SQLite, allowing follow-up questions to reuse previous conversation context through a stable `thread_id`.

---

## 🎯 Project Objective

Traditional AI chatbots may generate technically plausible answers without considering an organization's internal procedures.

CloudOps Sentinel addresses this problem by combining:

- 🔐 Private enterprise knowledge
- 🔎 Vector-based semantic retrieval
- 🧠 Self-RAG evaluation
- 🔄 Automatic query rewriting
- 🌐 Controlled web-search fallback
- 💾 Persistent conversational memory
- 📄 Live knowledge-base document ingestion
- ⚙️ Production-style API architecture

### Core Principle

> **Private Knowledge First → Evaluate → Correct → Retry → Web Fallback → Verify → Answer**

---

# 🏗️ Architecture

```text
                    User Incident / Follow-up
                              │
                              ▼
                    LangGraph SQLite Memory
                              │
                              ▼
                    Contextualize Follow-up
                              │
                              ▼
                       Retrieval Decision
                              │
                              ▼
                  ┌─────────────────────────┐
                  │   Private Pinecone KB   │
                  │ Runbooks / SOPs / Docs  │
                  └────────────┬────────────┘
                               │
                               ▼
                     Grade Retrieved Evidence
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
              Relevant                  Weak / Missing
                 │                           │
                 ▼                           ▼
             Generate                  Rewrite Query
                 │                           │
                 ▼                           ▼
              IsSUP?                   Retry Private KB
                 │                           │
          ┌──────┴──────┐                    │
          │             │                    │
        Pass          Fail                   │
          │             │                    │
          │          Revise                  │
          │             │                    │
          └──────┬──────┘                    │
                 │                           │
                 │                    Still insufficient?
                 │                           │
                 │                           ▼
                 │                     Tavily Web Search
                 │                           │
                 │                           ▼
                 │                  Grade Web Evidence
                 │                           │
                 └──────────────┬────────────┘
                                ▼
                           Final Answer
                                │
                                ▼
                     SQLite Checkpoint / Memory
