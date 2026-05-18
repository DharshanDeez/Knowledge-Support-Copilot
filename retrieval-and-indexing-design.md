# Retrieval and Indexing Design

## 1) Purpose

This document defines how Knowledge Support Copilot ingests documents, creates searchable indexes, retrieves evidence, ranks relevance, and returns grounded answers with citations.

It is the detailed technical companion to `system-design.md`.

---

## 2) Design Principles

- Retrieval quality is more important than model size for factual accuracy.
- Every answer must be grounded in retrieved evidence.
- If evidence is weak, the system must not guess.
- MVP must be simple; scale controls are added as load grows.
- Retrieval and generation should be measurable end-to-end.

---

## 3) End-to-End Pipeline

1. Ingest approved documents
2. Parse and normalize content
3. Chunk content with overlap
4. Generate embeddings for each chunk
5. Store chunks in vector index and keyword index
6. Process user query
7. Run hybrid retrieval (vector + keyword)
8. Re-rank candidate chunks
9. Build context window
10. Generate answer from context
11. Validate confidence and safety
12. Return answer with citations or fallback

---

## 4) Document Ingestion and Parsing

### 4.1 Supported Source Types (MVP)

- Markdown (`.md`)
- Text (`.txt`)
- PDF (text-extractable)
- HTML exports or help pages (sanitized)

### 4.2 Parsing Rules

- Remove boilerplate noise (headers/footers/navigation)
- Preserve section structure and headings
- Keep tables as text blocks where possible
- Normalize whitespace and unicode
- Preserve source references for citation mapping

### 4.3 Required Metadata

Per document:

- `doc_id`
- `source_name`
- `source_type`
- `owner_team`
- `access_scope`
- `version`
- `updated_at`

Per chunk:

- `chunk_id`
- `doc_id`
- `chunk_text`
- `section_path`
- `char_start`
- `char_end`
- `token_count`
- `embedding_model`

---

## 5) Chunking Strategy

### 5.1 MVP Defaults

- Chunk size: `500` tokens
- Overlap: `80` tokens
- Split priority:
  1. heading boundaries
  2. paragraph boundaries
  3. sentence boundaries (fallback)

### 5.2 Chunk Quality Rules

- Avoid cutting lists/tables mid-meaning where possible
- Keep chunk semantically coherent (single topic if possible)
- Keep citations traceable to section/title
- Reject empty/near-empty chunks

### 5.3 Future Tuning

- Domain-specific chunk profiles by source type
- Adaptive chunking by heading depth and content density
- Table-aware chunking for richer document types

---

## 6) Embedding Strategy

### 6.1 Embedding Model Requirements

- Strong semantic retrieval on support and policy language
- Good performance on short and medium chunks
- Stable output dimension and versioning support

### 6.2 Embedding Operations

- Generate embeddings for all chunks at indexing time
- Use same embedding family for query vectors
- Store embedding model version for compatibility control

### 6.3 Re-Index Triggers

- Source content updated
- Chunking policy changed
- Embedding model upgraded

---

## 7) Index Design

### 7.1 Hybrid Indexing (Required)

Use two parallel indexes:

- **Vector index** for semantic similarity
- **Keyword index (BM25 or equivalent)** for exact term matching

Hybrid retrieval improves precision and recall versus single-method retrieval.

### 7.2 MVP Storage Pattern

- Single logical vector collection
- Single keyword index
- Metadata filters enforced at query time

### 7.3 Future-Scale Pattern (10k to 100k docs)

- Partition/shard strategy by source or tenant
- Parallel indexing workers via queue
- Incremental indexing for changed documents only
- Background compaction and index health checks

---

## 8) Query Processing

### 8.1 Input Normalization

- Trim whitespace and normalize punctuation
- Detect empty or unsupported queries
- Basic language detection (optional in MVP)

### 8.2 Query Rewrite (Optional, Controlled)

Use rewrite only when needed for retrieval quality:

- expand abbreviations
- clarify ambiguous terms
- preserve original intent

Always log rewritten query for observability.

### 8.3 Access and Policy Filters

Apply filters before retrieval:

- user role and access scope
- source allowlist/denylist
- compliance policy constraints

---

## 9) Retrieval Strategy

### 9.1 Candidate Retrieval

- Vector top-k: `k_vector = 20`
- Keyword top-k: `k_keyword = 20`
- Merge candidates into a shared pool

### 9.2 Fusion

Use weighted rank fusion (or reciprocal rank fusion):

- semantic score weight: `0.6`
- keyword score weight: `0.4`

These are MVP starting values and should be tuned via evaluation.

### 9.3 Re-ranking

- Run re-ranker on merged top candidates
- Keep final `k_final = 6` chunks for generation context
- Prefer diversity across sources when scores are close

---

## 10) Context Building for Generation

### 10.1 Context Rules

- Include top-ranked chunks with source metadata
- Keep within model token budget
- Deduplicate near-identical chunks
- Keep section/title references for citation rendering

### 10.2 Prompt Constraints

- answer only from provided context
- if evidence is weak, return uncertainty
- cite sources for key claims

---

## 11) Confidence and Fallback

### 11.1 Confidence Signals

Combine multiple signals:

- retrieval relevance score distribution
- re-ranker confidence
- coverage of query intent in context
- generation self-check (optional, controlled)

### 11.2 Fallback Rules (MVP)

Trigger fallback if:

- no relevant chunks above threshold
- top chunk scores are low/flat
- answer contains unsupported claims
- calibrated band is `low` (or `medium` when policy requires conservative routing)

Fallback response should:

- clearly say confidence is low
- offer best available cited fragments if safe
- recommend human escalation path

### 11.3 Confidence Calibration (Required)

Heuristic signals are not sufficient alone. Before production pilot:

1. Run the full pipeline on the labeled eval set (`data/eval/`).
2. Measure answer correctness vs `confidence_score` and band (see `system-design.md` §12.4).
3. Tune fusion weights and thresholds until `high` band meets target precision (≥90% correct on eval set; ≤5% high-confidence errors).
4. Version scorer weights and thresholds in the governance registry.

Log per request: raw signal values, fused `confidence_score`, band, and fallback decision for offline replay and calibration.

---

## 12) Citation Design

Each response citation should include:

- source title
- section or heading
- short supporting snippet
- internal reference id (`doc_id/chunk_id`)

This supports trust, auditability, and debugging.

---

## 13) Evaluation Framework

Canonical detail: `system-design.md` §12. This section covers retrieval-specific evaluation requirements.

### 13.1 Labeled Benchmark Set (Required)

Each eval row must include `question`, `gold_chunk_ids`, `reference_answer` or rubric, `must_cite_chunk_ids`, and `expected_fallback` where the system should not answer confidently.

### 13.2 Offline Evaluation (before release)

Use benchmark set from real user questions.

**Retrieval metrics** (against `gold_chunk_ids`):

- Recall@k
- Precision@k
- MRR (Mean Reciprocal Rank)

**Answer metrics** (full RAG output on eval set):

- Groundedness (claims supported by retrieved/cited chunks)
- Citation precision (cited chunks match required evidence)
- Answer relevance (vs reference answer or rubric)
- Grounded answer rate

**Confidence metrics** (join pipeline output with labels):

- Accuracy by confidence band
- High-confidence error rate
- Missed-fallback rate (`expected_fallback=true` but system answered `high`)

### 13.3 Online Evaluation (after release)

- helpfulness feedback rate
- escalation rate by topic
- unresolved query rate
- fallback rate trend
- high-confidence complaints (user flags answer as wrong after `high` band)
- latency percentiles (p50, p95)

### 13.4 Evaluation Cadence

- full offline re-run before every prompt, model, retrieval, or threshold change
- weekly metric review during pilot (retrieval + answer + calibration slices)
- monthly threshold and scorer-weight tuning after stabilization

---

## 14) Observability and Alerts

### 14.1 What to Log

- request id, user role, query text hash
- rewritten query (if used)
- candidate chunk ids and scores
- final selected chunk ids
- confidence value and fallback decision
- end-to-end latency and stage latencies

### 14.2 Critical Alerts

- retrieval failure rate above threshold
- sudden fallback spike
- citation-missing responses > threshold
- p95 latency breach
- ingestion backlog growth

---

## 15) Failure Modes and Mitigations

- **No retrieval hit** -> fallback + escalate
- **Low relevance retrieval** -> tighten filters, retune chunking/reranker
- **Outdated content** -> enforce refresh SLA and source ownership
- **Latency spikes** -> caching, parallel retrieval tuning, index optimization
- **Model instability** -> circuit breaker + rollback policy
- **Ingestion failures** -> retries + dead-letter queue

---

## 16) Scalability Plan (Future, Not MVP)

Target future readiness for approximately `10,000` to `100,000` documents.

Key upgrades:

- queue-based ingestion
- parallel parser/embedding workers
- sharded vector and keyword indexes
- incremental indexing by change events
- retrieval gateway and cache layer
- dedicated evaluation service pipeline

---

## 17) Suggested Initial Configuration

- Chunk size: `500`
- Overlap: `80`
- `k_vector`: `20`
- `k_keyword`: `20`
- `k_final`: `6`
- Fusion weights: semantic `0.6`, keyword `0.4`
- Fallback policy: conservative (prefer escalation over guess)

These are baseline defaults and should be tuned with offline and online evaluation.

---

## 18) Open Decisions

- Final embedding model choice for local and production
- Re-ranker model choice and latency budget
- Exact confidence threshold by domain (from calibration, not defaults)
- Answer judge implementation (NLI vs LLM-as-judge) and human review sample size
- Citation UI format and length constraints
- Query rewrite policy by intent class
- Multi-language strategy timeline

---

## 19) Final Technical Statement

This design ensures that answer quality is driven by robust retrieval, not only generation. By combining hybrid indexing, controlled reranking, confidence gating, and measurable evaluation loops, the system can deliver trustworthy responses in MVP and scale safely to larger document volumes in future production phases.
