# DocuVault Core 🏛️
### Enterprise Multi-Tenant Document Intelligence & RAG API

> 🚧 **Notice:** **This project is currently under active development. Core implementation, benchmarks, and production codebase will be pushed by the end of October 2026.**

---

[![Python Version](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)
[![AWS S3](https://img.shields.io/badge/AWS_S3-Presigned_Uploads-FF9900?style=for-the-badge&logo=amazons3&logoColor=white)](https://aws.amazon.com/s3/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

---

## 📌 Executive Overview

**DocuVault** is an enterprise-grade, multi-tenant document intelligence backend designed to solve the critical security, memory, and compliance bottlenecks of production RAG (*Retrieval-Augmented Generation*) systems.

Most open-source RAG implementations operate as fragile prototypes: they load multi-megabyte PDFs directly into API server memory, aggregate embeddings into unpartitioned global vector stores (causing severe cross-company data leakage risks), and lack measurable retrieval benchmarks.

DocuVault delivers a **production-hardened B2B architecture**:
- **Zero-RAM Cloud Ingestion:** Clients stream uploads directly to **AWS S3** via presigned URLs, eliminating API server memory spikes.
- **Cryptographic Multi-Tenancy:** Hard row-level data isolation at the PostgreSQL layer via `tenant_id` scopes on all vector operations.
- **Real-Time Token Streaming:** Asynchronous Server-Sent Events (SSE) via FastAPI's `StreamingResponse`.
- **Empirical Evaluation Harness:** Quantitative benchmarking verifying retrieval hit rates, citation accuracy, and deterministic fallback behavior on unanswerable queries.

---

## 🏗️ System Architecture

```
[ Web / Mobile Client ]
         │
         ├── 1. POST /api/v1/documents/upload-url (JWT with tenant_id)
         │         │
         │         ▼
         │   [ FastAPI Backend Gateway ]
         │         │ (Validates JWT claims, MIME type, file size bounds)
         │         ├── Generates AWS S3 Presigned PUT URL (15-min expiry)
         │         └── Records document record in PostgreSQL (status: 'pending_upload')
         │
         ├── 2. Direct PUT Binary Stream (0 MB API Server RAM utilized)
         │         │
         │         ▼
         │   [ AWS S3 Object Storage ] (Encrypted at rest with SSE-S3)
         │
         ├── 3. POST /api/v1/documents/{doc_id}/process
         │         │
         │         ▼
         │   [ Ingestion Worker (BackgroundTasks) ]
         │         ├── Streams byte stream directly from S3 -> PDF Text Extractor
         │         ├── Chunks text with sliding window (500 tokens, 50-token overlap)
         │         ├── Generates dense vector embeddings via Gemini text-embedding-004
         │         └── Batch inserts vector chunks into PostgreSQL with tenant_id
         │                 │
         │                 ▼
         │         [ PostgreSQL 16 + pgvector ]
         │         Stores: (id, tenant_id, doc_id, chunk_index, page_num, text, embedding)
         │         Indexed via HNSW (Cosine metric) + B-Tree on (tenant_id, doc_id)
         │
         └── 4. POST /api/v1/chat/ask (Natural language prompt)
                   │
                   ▼
             [ RAG Retrieval & Inference Engine ]
                   ├── Generates query embedding vector
                   ├── Executes tenant-isolated Cosine similarity query:
                   │     SELECT chunk_text, page_num, doc_id 
                   │     FROM document_chunks 
                   │     WHERE tenant_id = :jwt_tenant_id 
                   │     ORDER BY embedding <=> :query_vector LIMIT 4;
                   ├── Injects retrieved chunks into strict anti-hallucination prompt
                   ├── Calls Gemini 2.5 Flash with stream=True
                   └── Streams SSE tokens to client with document & page citations
```

---

## ⚡ The 4 Engineering Pillars

### 1. Zero-RAM Direct S3 Ingestion
* **The Failure Mode:** Standard `UploadFile` endpoints buffer incoming files in server memory or disk. Under 10 concurrent 50MB PDF uploads, memory consumption exceeds container limits, triggering OS `OOMKilled` termination.
* **The DocuVault Solution:** The API generates cryptographically signed AWS S3 presigned PUT URLs with short TTLs. The client streams the file payload directly to AWS S3. The backend manages only metadata and authentication, maintaining baseline memory usage under heavy concurrent load.

### 2. Multi-Tenant Cryptographic Isolation
* **The Failure Mode:** Monolithic vector databases lacking hardware or namespace partitioning risk retrieving competitor records under prompt injection or vector proximity collisions.
* **The DocuVault Solution:** Every vector query is bound by a strict database-level filter `WHERE tenant_id = :authenticated_tenant_id`, sourced exclusively from verified JWT cryptographic claims. Compound B-Tree indexes on `(tenant_id, document_id)` ensure index scans remain partitioned and high-speed.

### 3. Sub-35ms Semantic Search via `pgvector` HNSW
* Vectors are indexed using Hierarchical Navigable Small World (**HNSW**) graphs (`vector_cosine_ops`) configured with `m = 16` and `ef_construction = 64`.
* Combines transactional ACID integrity (users, documents, audit trails) with high-dimensional vector search inside a single unified PostgreSQL cluster, eliminating vector database sync drift.

### 4. Automated Evaluation & Guardrails (`eval.py`)
* Includes an automated 30-question evaluation suite measuring:
  - **Retrieval Hit Rate:** Target $\ge 94\%$ accuracy across semantic test queries.
  - **Faithfulness:** Verifies responses cite only retrieved document context.
  - **Deterministic Fallback:** Verifies 100% adherence to standard refusal (*"Information not found in available documents"*) when questions target absent or out-of-tenant content.

---

## 📊 Database Schema Design

```sql
-- Enable vector extension
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Multi-Tenant Organizations
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    plan_tier VARCHAR(50) DEFAULT 'enterprise',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Users & RBAC
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    role VARCHAR(50) DEFAULT 'member', -- 'admin', 'member', 'auditor'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_users_tenant ON users(tenant_id);

-- Document Metadata Registry
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    filename VARCHAR(255) NOT NULL,
    s3_key VARCHAR(512) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending_upload', -- 'pending_upload', 'processing', 'ready', 'failed'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX idx_docs_tenant_status ON documents(tenant_id, status);

-- Chunk Registry with High-Dimensional Vectors
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INT NOT NULL,
    page_number INT NOT NULL,
    chunk_text TEXT NOT NULL,
    embedding VECTOR(768) NOT NULL, -- Gemini text-embedding-004
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Compound index for tenant-scoped vector search
CREATE INDEX idx_chunks_tenant ON document_chunks(tenant_id);
CREATE INDEX idx_chunks_hnsw ON document_chunks 
USING hnsw (embedding vector_cosine_ops) 
WITH (m = 16, ef_construction = 64);

-- Compliance & Audit Trail
CREATE TABLE query_audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    query_text TEXT NOT NULL,
    retrieved_chunk_ids UUID[] DEFAULT '{}',
    latency_ms INT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 🔌 API Endpoint Specifications

| HTTP Method | Route | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/register` | Public | Registers tenant organization and initial Admin account. |
| `POST` | `/api/v1/auth/login` | Public | Authenticates user; returns signed JWT with `tenant_id` and `role`. |
| `POST` | `/api/v1/documents/upload-url` | Authenticated (`member`+) | Generates AWS S3 presigned PUT URL and creates pending document entry. |
| `POST` | `/api/v1/documents/{id}/process`| Authenticated (`member`+) | Triggers background chunking, embedding generation, and vector indexing. |
| `GET` | `/api/v1/documents` | Authenticated (`member`+) | Lists all indexed documents belonging strictly to the caller's tenant. |
| `POST` | `/api/v1/chat/ask` | Authenticated (`member`+) | Semantic vector search + SSE token streaming with source citations. |
| `GET` | `/api/v1/audit/logs` | Admin Only (`role: admin`) | Retrieves compliance logs, user query records, and latency metrics. |
| `GET` | `/health` | Public | Liveness probe verifying PostgreSQL, `pgvector`, and AWS S3 connectivity. |

---

## 📈 Target Performance Benchmarks

| Metric | Target Specification | Architectural Enabler |
| :--- | :---: | :--- |
| **Vector Retrieval Latency** | **$< 35\text{ ms}$** | PostgreSQL `pgvector` HNSW index with tenant-isolated filtering |
| **Server RAM During Uploads** | **$0\text{ MB}$ overhead** | AWS S3 Presigned direct-to-cloud upload URLs |
| **Retrieval Hit Rate** | **$\ge 94\%$** | 500-token chunks with 50-token overlapping semantic window |
| **Zero-Context Hallucinations** | **$0\%$** | Strict prompt grounding with deterministic refuse instructions |
| **Time-to-First-Token (TTFT)** | **$< 450\text{ ms}$** | Real-time Server-Sent Events (SSE) streaming pipeline |

---

## 🗂️ Repository Structure

```text
DocuVault/
├── app/
│   ├── api/
│   │   ├── v1/
│   │   │   ├── endpoints/
│   │   │   │   ├── auth.py         # JWT generation & RBAC verification
│   │   │   │   ├── documents.py    # S3 presigned URLs & ingestion triggers
│   │   │   │   ├── chat.py         # Vector similarity search & SSE streaming
│   │   │   │   └── audit.py        # Compliance query logs
│   │   │   └── router.py           # V1 route aggregator
│   ├── core/
│   │   ├── config.py               # Pydantic v2 BaseSettings (.env loading)
│   │   ├── security.py             # Bcrypt password hashing & JWT handling
│   │   └── database.py             # Async SQLAlchemy 2.0 engine + asyncpg
│   ├── models/
│   │   ├── tenant.py               # Multi-tenant organization model
│   │   ├── user.py                 # User & RBAC schema
│   │   └── document.py             # Document & DocumentChunk (pgvector)
│   ├── schemas/
│   │   ├── auth.py                 # Request/Response validation models
│   │   ├── document.py             # Presigned URL contract models
│   │   └── chat.py                 # Query payload & Citation models
│   ├── services/
│   │   ├── s3.py                   # Boto3 client for presigned URL generation
│   │   ├── embedder.py             # Gemini text-embedding-004 async client
│   │   ├── chunker.py              # Recursive token/character text splitter
│   │   └── rag.py                  # Grounded context injection & SSE generator
│   └── main.py                     # FastAPI application factory & middleware
├── tests/
│   ├── test_auth.py                # Authentication & RBAC security unit tests
│   ├── test_documents.py           # Presigned URL generation & isolation tests
│   └── eval_rag.py                 # 30-question empirical evaluation benchmark
├── Dockerfile                      # Multi-stage production container definition
├── docker-compose.yml              # Local PostgreSQL 16 (pgvector) + API service
├── requirements.txt                # Pinned production dependencies
├── .env.example                    # Environment variable template
└── README.md                       # System architecture & documentation
```

---

## 🚀 Quickstart & Local Setup (Preview)

### Prerequisites
- Docker & Docker Compose
- Python 3.11+
- AWS Account (S3 bucket configured with CORS)
- Google AI Studio API Key (`GEMINI_API_KEY`)

```bash
# 1. Clone repository
git clone https://github.com/your-username/DocuVault.git
cd DocuVault

# 2. Configure environment
cp .env.example .env

# 3. Spin up PostgreSQL with pgvector
docker-compose up -d db

# 4. Initialize virtual environment & install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 5. Launch development server
uvicorn app.main:app --reload --port 8000
```

Interactive OpenAPI Swagger UI will be accessible at: `http://localhost:8000/docs`

---

## 👨‍💻 Engineering Author
Built with a relentless focus on clean backend architecture, cloud scalability, and production-grade AI systems engineering.

* **Tech Stack:** Python • FastAPI • PostgreSQL (`pgvector`) • AWS S3 • Docker • Gemini API
* **License:** MIT License
