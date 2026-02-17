# Atlas AI RAG Financial Showcase

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&height=220&text=Atlas%20AI%20RAG%20Financial%20Showcase&fontAlign=50&fontSize=36&color=0:0f172a,100:1d4ed8&fontColor=ffffff" alt="Atlas AI RAG Financial Showcase banner" />
</p>

This tutorial walks through a practical, production-oriented financial RAG system.  
The focus is learning by building: document ingestion, chunking, embeddings, retrieval, LLM answering, guardrails, and cost control.

> This repository is optimized for learning and portfolio demonstration.  
> It is not a one-click production template, but a transparent architecture you can inspect end-to-end.

## Target Audience

Engineers and builders who want a real-world example of:
- financial document ingestion
- RAG query workflows
- metrics extraction and earnings analysis
- FinOps-aware AI operations

## Project Details

- Stack: FastAPI, React, PostgreSQL + pgvector, Anthropic/OpenAI/Ollama
- Focus: RAG on financial filings (10-Q/10-K style documents)
- Objective: grounded answers with traceability, low hallucination risk, and cost visibility

## Navigation

- [Query Flow](#query-flow)
- [Query Cache and Cost Saving](#query-cache-and-cost-saving)
- [Documents](#documents)
- [Data Ingestion](#data-ingestion)
- [RAG Query Execution](#rag-query-execution)
- [Earnings Signals](#earnings-signals)
- [Cost and Usage](#cost-and-usage)
- [Audit](#audit)
- [Cost Trade-offs](#cost-trade-offs)
- [Multi-Layer Protection](#multi-layer-protection-cost-and-reliability)
- [Selected Architecture](#selected-architecture)
- [ADR-002: API Gateway + VPC Link](#adr-002-api-gateway--vpc-link)
- [Data Sources](#data-sources)

## Query Flow

This is the main entry point of the system.  
The user submits a question, the backend retrieves relevant chunks from the vector database, and the model returns a grounded answer with evidence.

Flow:
1. user writes a question
2. backend identifies intent/ticker and retrieves top chunks
3. LLM generates the answer from retrieved context
4. UI shows answer and source snippets

![Query Flow](docs/images/query.png)

## Query Cache and Cost Saving

When the same normalized query is repeated with the same parameters, the system serves the response from internal cache instead of triggering a new provider call.

Why it matters:
- lower latency for repeated requests
- lower token usage and lower cost
- consistent answer for identical query inputs

Cache behavior:
1. check query signature (query + ticker + model + top-k)
2. cache hit -> return cached answer
3. cache miss -> run retrieval + LLM, then persist

![Query Cache](docs/images/query_cache.png)

## Documents

This stage is where uploaded files are listed, reviewed, and selected before running deeper processing workflows.

What this view enables:
- inspect available documents in the local corpus
- verify filenames and uploaded assets
- prepare the document set that will be ingested and queried

![Documents](docs/images/ingest_document.png)

## Data Ingestion

This step shows how documents enter the RAG pipeline.

Ingestion flow:
1. upload PDF or UTF-8 text document
2. extract and normalize text
3. split text into chunks
4. generate embeddings for each chunk
5. store documents/chunks/vectors in PostgreSQL + pgvector

Result:
- searchable knowledge base ready for semantic retrieval
- traceability from answer to source chunks

![Data Ingestion](docs/images/ingestion.png)

## RAG Query Execution

This view represents runtime query execution after ingestion.

Execution path:
1. user asks a financial question
2. backend retrieves top relevant chunks
3. model composes a grounded answer
4. UI shows score, token usage, and evidence chunks

![RAG Query](docs/images/rag_query.png)

## Earnings Signals

This section focuses on extracted financial signals from filings.

What it highlights:
- key earnings indicators for selected tickers
- normalized values for analysis
- fast review panel for portfolio-style questions

![Earnings Signals](docs/images/earnings.png)

### PDF Extract Example

Total current liabilities is shown through an extracted snippet from a source filing PDF.  
It demonstrates that values can be traced back to original document evidence.

![Total Current Liabilities Extract](docs/images/total_current_liabilities.png)

This extract is shown as evidence from an original public filing document used in the showcase.

## Cost and Usage

This dashboard tracks operational cost behavior of the RAG platform.

Main purpose:
- monitor volume and spend trend
- verify cache impact vs fresh calls
- keep usage visible for FinOps decisions

![Cost and Usage](docs/images/cost_usage.png)

## Audit

The Audit view is the operational trace layer.

What it is used for:
- inspect recent query executions
- verify provider, retrieval depth, and token behavior
- detect anomalies or repeated failures
- support post-incident analysis with concrete evidence

![Audit](docs/images/audit.png)

## Cost Trade-offs

This view explains architecture trade-offs behind infrastructure and AI spending.

Trade-off dimensions:
- security boundary vs fixed monthly cost
- performance/latency vs token and compute cost
- richer retrieval context vs higher inference spend
- operational simplicity vs fine-grained control

![Cost Trade-offs](docs/images/cost_tradeoff.png)

## Multi-Layer Protection (Cost and Reliability)

This project applies 8 protection layers to reduce runaway behavior, control spend, and improve reliability.

| # | Layer | Protection | Prevents |
|---|-------|------------|----------|
| 1 | Max Retries | Hard retry limit | Infinite retry loops |
| 2 | Circuit Breaker | Stop after repeated failures | Cascading failures |
| 3 | Cost Budget | Daily and hourly budget controls | Budget drain |
| 4 | Timeout | Maximum wait between retries | Hanging requests |
| 5 | Request Deduplication | Reuse identical requests when possible | Duplicate queries and duplicate cost |
| 6 | Loading State Control | Block repeated actions while processing | Double-click execution |
| 7 | Client Rate Limit | Sliding-window request cap | Spam bursts |
| 8 | Abort Control | Cancel superseded or obsolete runs | Orphan requests |

## Selected Architecture

We selected `CloudFront + S3 + ALB + ECS Fargate + PostgreSQL (pgvector)` because it gives the best balance between reliability, security, and cost for a portfolio-grade production pattern.

Why this choice:
- frontend delivery stays fast and low-cost with CloudFront + S3
- API traffic is controlled through ALB instead of exposing app containers directly
- backend compute runs isolated on Fargate
- `PostgreSQL` stores structured operational data (documents, metadata, audit, cache, usage)
- `pgvector` enables semantic search over embeddings for accurate RAG retrieval
- this keeps the system maintainable while avoiding over-complex enterprise-only patterns

### Why `top-k = 10` and not `3`

Financial answers usually need evidence spread across multiple pages and sections.

With `top-k=3`, recall often drops; with `top-k=10`, context coverage is safer for grounded answers.

- **`k=3`: cheaper, but higher risk of missing key evidence.**
- **`k=10`: better recall with an acceptable token increase.**
- **Start with `10`, then tune using real evaluation data (`quality`, `latency`, `cost/query`).**

### Python Backend Outbound Calls (Chronological Flow)

This is the practical runtime sequence for external service calls from backend startup to live inference.

| Step | Source | Destination | Purpose | Exposure |
|---|---|---|---|---|
| 1 | ECS task startup | ECR (Interface Endpoint) | Discover image metadata before pull | Private |
| 1.5 | ECR | ECS task startup | Return layer locations and pull tokens | Private |
| 2 | ECS task startup | S3 (Gateway Endpoint) | Download image layers required for container boot | Private |
| 3 | ECS/Fargate runtime | Task ephemeral disk | Materialize layers and start Python container | Internal |
| 4 | Python backend | Secrets Manager (Interface Endpoint) | Load runtime secrets and credentials | Private |
| 5 | Python backend | Anthropic API (`api.anthropic.com`) | Run live LLM inference for query answering | Controlled public egress |
| 6 | ECS + Python backend | CloudWatch Logs (Interface Endpoint) | Persist operational logs and diagnostics | Private |
| 7 | ECS + Python backend | STS (Interface Endpoint) | Obtain temporary AWS credentials | Private |
| 8 | Python backend | Bedrock Runtime (Interface Endpoint) | Optional embedding generation path (`EMBEDDING_MODE=bedrock`) | Private |

**Data boundary recap:**
- **S3 stores source filing documents (PDFs).**
- **PostgreSQL + pgvector stores extracted text, chunks, embeddings, and retrieval metadata.**

![Selected Architecture - PostgreSQL and pgvector](docs/images/postgre_pgvector.png)

## ADR-002: API Gateway + VPC Link

![Ingress: ALB](https://img.shields.io/badge/Ingress-ALB-FF9900?logo=amazonaws&logoColor=white)
![Compute: ECS Fargate](https://img.shields.io/badge/Compute-ECS%20Fargate-FF9900?logo=amazonaws&logoColor=white)
![Data: PostgreSQL + pgvector](https://img.shields.io/badge/Data-PostgreSQL%20%2B%20pgvector-336791?logo=postgresql&logoColor=white)
![Deferred: API Gateway + VPC Link](https://img.shields.io/badge/Deferred-API%20Gateway%20%2B%20VPC%20Link-6f42c1?logo=amazonaws&logoColor=white)

**Decision:** API Gateway + VPC Link is intentionally deferred in this phase.  
**Current baseline:** ALB + private ECS already meets security, delivery speed, and cost discipline targets.

| Decision Area | Current Choice (Now) | API Gateway + VPC Link (Later) | Why Deferred |
|---|---|---|---|
| Product shape | Single internal dashboard | External API product for partners/clients | We do not need public developer APIs yet |
| Access model | Cognito + backend auth validation | Per-tenant API keys, usage plans, API products | Tenant API governance is not required yet |
| Rate and quotas | App-level controls + backend protections | Central API throttling and quota plans | Existing controls are sufficient for current load |
| Ops complexity | One ingress layer (ALB) | Additional routing, policies, contracts | Extra moving parts with limited immediate value |
| Cost profile | Lean baseline for fast iteration | Higher fixed and operational overhead | Current priority is cost discipline and shipping speed |

### When We Will Introduce API Gateway

- external client/partner APIs become a core product surface
- per-tenant API keys and usage plans become mandatory
- strict per-route governance and monetization controls are needed
- multi-team API lifecycle management requires centralized enforcement

## Data Sources

The showcase uses public financial filing documents that can be downloaded freely online.  
Those documents are ingested, chunked, embedded, and used as evidence for answers shown in this portfolio.
