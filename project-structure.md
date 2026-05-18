# Project Structure

## 1) Purpose

This document defines the recommended folder and module layout for Knowledge Support Copilot.  
The structure is designed for:

- quick MVP delivery
- clean ownership boundaries
- easy migration from local testing to production deployment
- governance, observability, and quality evaluation from day one

---

## 2) Recommended Repository Layout

```text
simple-rag/
  docs/
    problem-framing.md
    system-design.md
    retrieval-and-indexing-design.md
    project-structure.md
    implementation-plan.md

  app/
    api/
      routes/
        health.py
        query.py
        feedback.py
      schemas/
        request_response.py
      dependencies/
        auth.py
        rate_limit.py

    ingestion/
      loaders/
        markdown_loader.py
        pdf_loader.py
        text_loader.py
      parser/
        normalize.py
        sanitize.py
      chunking/
        chunker.py
      queue/
        job_queue.py
        dlq.py
      pipeline/
        ingest_job.py
        retry_policy.py

    indexing/
      embeddings/
        embedder.py
        embedding_version.py
      vector_store/
        chroma_store.py
      keyword_index/
        bm25_index.py
      sync/
        index_sync.py

    retrieval/
      retrievers/
        vector_retriever.py
        keyword_retriever.py
        hybrid_retriever.py
      reranker/
        rerank.py
      fusion/
        rank_fusion.py
      filters/
        access_filter.py

    generation/
      llm/
        llm_client.py
        hf_client.py
        ollama_client.py
        provider_router.py
      prompts/
        answer_prompt.txt
      answering/
        generate_answer.py
      citations/
        citation_builder.py

    safety/
      confidence/
        scorer.py
        thresholds.py
      guardrails/
        policy_check.py
      fallback/
        fallback_handler.py
      escalation/
        escalation_router.py

    eval/
      datasets/
        eval_questions.json
        README.md
      labels/
        human_labels.json
      offline/
        retrieval_metrics.py
        answer_scorer.py
        grounding_checks.py
        citation_checks.py
        confidence_calibration.py
        eval_report.py
      online/
        feedback_analytics.py
        calibration_drift.py
      drift/
        drift_monitor.py

    observability/
      logging/
        logger.py
        audit_log.py
      metrics/
        metrics.py
      tracing/
        tracing.py
      alerts/
        alert_rules.md

    governance/
      config_registry/
        model_versions.py
        retrieval_config.py
      policy/
        data_access_policy.md
      change_control/
        release_checklist.md

    shared/
      config/
        settings.py
      models/
        entities.py
      utils/
        text_utils.py
        ids.py
        time_utils.py

    main.py

  scripts/
    run_ingestion.py
    run_eval_offline.py
    run_eval_calibration.py
    run_eval_online_report.py
    reindex.py

  tests/
    unit/
    integration/
    e2e/

  data/
    raw/
    processed/
    index/
    eval/
      eval_questions.json
      eval_runs/
      calibration_reports/
    dlq/

  .env.example
  requirements.txt
  README.md
```

---

## 3) Module Responsibilities

### API

- expose endpoints for query, feedback, and health
- apply request validation, auth checks, and rate limits

### Ingestion

- read approved sources
- parse and clean text
- chunk content with metadata
- manage retries and DLQ for failed jobs

### Indexing

- generate embeddings
- write to vector store and keyword index
- keep index consistency across updates

### Retrieval

- fetch vector and keyword candidates
- fuse/rerank candidates
- apply role/access filters

### Generation

- route LLM calls to provider abstraction
- support Hugging Face API primary and optional Ollama fallback
- enforce prompt and citation format rules

### Safety

- confidence checks
- policy guardrails
- fallback and escalation routing

### Evaluation

- labeled benchmark schema (`question`, `gold_chunk_ids`, `reference_answer`, `must_cite_chunk_ids`, `expected_fallback`)
- offline retrieval metrics (Recall@k, Precision@k, MRR)
- offline answer metrics (groundedness, citation precision, answer relevance)
- offline confidence calibration (accuracy by band, high-confidence error rate, missed fallback)
- online quality, feedback analytics, and calibration drift tracking
- eval report artifact per run under `data/eval/eval_runs/`

### Observability

- logs, metrics, traces
- alert thresholds and incident signal flow

### Governance

- model/version registry
- retrieval and threshold versioning
- release and change control documentation

---

## 4) Environment Strategy

### Local (MVP)

- local app runtime
- local vector and keyword stores
- Hugging Face API keys for generation
- optional local Ollama fallback

### Staging

- production-like configs
- controlled data subset
- alert rules and evaluation checks active

### Production

- managed infrastructure
- strict access and audit policies
- automated alerts and release gates

---

## 5) Naming and Ownership Conventions

- keep one responsibility per module
- avoid business logic inside route handlers
- separate provider-specific logic from shared interfaces
- include model and config versions in audit logs
- assign owner team per major module (ingestion, retrieval, generation, ops)

---

## 6) Why This Structure Works

- easy for new contributors to navigate
- supports quick MVP without blocking future scale
- enables hybrid LLM provider strategy cleanly
- embeds governance, evaluation, and observability into core design
