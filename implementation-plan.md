# Implementation Plan

## 1) Purpose

This plan translates the approved architecture into execution phases for MVP delivery and future-scale readiness.

It aligns with:

- `problem-framing.md`
- `system-design.md`
- `retrieval-and-indexing-design.md`
- `project-structure.md`

---

## 2) Scope and Delivery Strategy

### MVP Focus

- deliver reliable query -> retrieval -> answer flow with citations
- run locally with API-key based LLM provider (Hugging Face)
- enforce fallback behavior for low confidence
- capture core logs and evaluation metrics (retrieval, answer quality, confidence calibration)

### Future Focus (Not MVP)

- scale ingestion and retrieval for approximately 10,000 to 100,000 documents
- introduce parallel workers, shard-ready indexes, and advanced alerting

---

## 3) Finalized Tech Stack (Approved)

- **Backend API:** FastAPI
- **Primary LLM provider:** Hugging Face Inference API (API-key based)
- **Fallback provider:** optional local Ollama runtime
- **Embeddings:** sentence-transformers (version pinned)
- **Vector store:** Chroma (MVP local persistent)
- **Keyword retrieval:** BM25 implementation
- **Reranker:** add in Phase 2
- **Testing:** pytest
- **Observability:** structured logs + metrics + traces (MVP baseline)

---

## 4) Phase Plan

## Phase 0: Foundation Setup (1-2 days)

### Deliverables

- create base repository structure
- setup environment config and secret handling
- add health endpoint and base logging
- setup provider abstraction (`llm_client`, `hf_client`, `ollama_client`)

### Exit Criteria

- app boots locally without errors
- API health route works
- config loaded from environment safely

---

## Phase 1: Ingestion and Indexing MVP (3-5 days)

### Deliverables

- loaders for `.md`, `.txt`, and basic `.pdf`
- parser and text normalization
- chunking with overlap and metadata
- embeddings generation and storage
- vector + keyword index creation

### Reliability Controls

- retry policy for ingestion failures
- dead-letter queue for failed ingestion jobs

### Exit Criteria

- sample dataset indexed end-to-end
- chunk metadata and source traceability validated
- failed jobs visible in DLQ

---

## Phase 2: Retrieval and Ranking (3-4 days)

### Deliverables

- vector retriever
- keyword retriever
- hybrid rank fusion
- reranking stage
- access and policy filters

### Quality Targets

- establish baseline Recall@k and Precision@k on labeled `gold_chunk_ids`
- seed labeled eval set (`data/eval/eval_questions.json`) with at least 20 questions before Phase 3
- verify top-k result relevance manually on sample questions

### Exit Criteria

- hybrid retrieval returns stable, relevant candidates
- metrics pipeline captures retrieval performance

---

## Phase 3: Generation, Citations, and Safety (3-4 days)

### Deliverables

- grounded answer generation with provider router
- citation builder and response schema
- confidence scoring and thresholds (log raw signals for calibration)
- fallback response + escalation path

### Reliability Controls

- provider retries with backoff and timeout controls
- safe response when provider or retrieval fails

### Exit Criteria

- each response includes citations or fallback
- low-confidence behavior is enforced

---

## Phase 4: Evaluation, Drift Monitoring, and Observability (2-3 days)

### Deliverables

- labeled eval dataset schema and starter benchmark (`data/eval/`)
- offline evaluation pipeline:
  - retrieval metrics (`Recall@k`, `Precision@k`, `MRR`)
  - answer metrics (`groundedness`, `citation_precision`, `answer_relevance`)
  - confidence calibration report (accuracy by band, high-confidence error rate, missed fallback)
- online feedback capture endpoint
- drift monitoring starter checks (including calibration drift)
- core dashboards for latency, fallback, failures, and confidence-band accuracy
- alert rules for critical thresholds

### Exit Criteria

- evaluation report generated from labeled benchmark set (retrieval + answer + confidence)
- confidence thresholds tuned so `high` band meets target precision on eval set
- alerts trigger on simulated threshold breaches

---

## Phase 5: MVP Hardening and Demo Readiness (2-3 days)

### Deliverables

- integration tests for full query pipeline
- timeout/circuit-breaker tuning
- runbook basics for known incidents
- final MVP documentation and demo checklist

### Exit Criteria

- stable local demo for stakeholder walkthrough
- known risks and next-scale actions documented

---

## 5) Governance and Version Control Plan

Track the following in every run and release:

- LLM provider and model id
- embedding model/version
- prompt version
- retrieval config (`k`, fusion weights, rerank settings)
- confidence threshold version

All changes should go through change notes and release checklist before rollout.

---

## 6) Retry, Timeout, and DLQ Policy

### API and Provider Calls

- retries: max `3`
- backoff: `1s -> 2s -> 4s` + jitter
- strict request timeout per dependency
- circuit breaker for repeated provider failures

### Ingestion Pipeline

- retry transient failures automatically
- move persistent failures to DLQ with error payload:
  - `doc_id`
  - pipeline stage
  - error type/message
  - retry count
  - timestamp

---

## 7) MVP Acceptance Gates

- citation coverage `>= 90%` on eval set
- fallback behavior active and measurable
- baseline retrieval metrics published (Recall@k, Precision@k, MRR)
- answer-quality baselines published (groundedness, citation precision, answer relevance)
- high-confidence error rate `<= 5%` on labeled eval set
- missed-fallback rate `<= 5%` when `expected_fallback=true`
- p95 latency tracked and reported
- critical alerting paths validated

---

## 8) Risks and Mitigations

- **Risk:** weak retrieval quality  
  **Mitigation:** tune chunking, hybrid weights, and reranker

- **Risk:** provider instability or rate limits  
  **Mitigation:** retry policy, timeout, fallback provider route

- **Risk:** stale source content  
  **Mitigation:** scheduled re-indexing and source ownership

- **Risk:** model/config drift  
  **Mitigation:** versioned config registry and periodic eval review

- **Risk:** overconfident wrong answers (confidence says high, answer is wrong)  
  **Mitigation:** labeled offline eval, confidence calibration, high-confidence error rate gate

- **Risk:** answer metrics not tied to retrieval fixes  
  **Mitigation:** full-pipeline eval on same benchmark; failure tagging (`weak_retrieval` vs `hallucination`)

---

## 9) Future-Scale Track (Post-MVP)

Planned after MVP acceptance:

- queue-first ingestion architecture
- parallel parser and embedding workers
- sharded indexes for 10,000 to 100,000 documents
- cache layer for repeated queries
- stronger alerting and auto-remediation patterns
- continuous evaluation service with release gates

---

## 10) Immediate Next Actions

1. create repo skeleton from approved project structure  
2. implement Phase 0 foundation setup  
3. ingest a small real sample corpus  
4. create starter labeled eval set (`data/eval/`)  
5. run first retrieval quality baseline  
6. review results and tune before Phase 3 generation rollout
