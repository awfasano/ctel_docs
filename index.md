---
layout: default
title: CTel RAG System Documentation
---

# CTel RAG System Documentation

Technical documentation for the CTel Telehealth RAG (Retrieval-Augmented Generation) pipeline.

## Documents

### [RAG Architecture Deep Dive](rag-architecture)
How the CTEL retrieval pipeline works, how it differs from standard RAG, and the design decisions behind each stage:
- Legal-aware chunking with abbreviation protection
- Document summary chunks (template-based, no LLM)
- Contextual retrieval (LLM-generated chunk prefixes)
- Structured state metadata for instant geographic filtering
- 8-stage query pipeline: decompose, multi-query, vector + BM25, diverse pool, LLM rerank, source/state penalties, diversify
- Semantic cache, factual grounding, query audit trail

### [System Overview](system-overview)
Comprehensive technical documentation of the full RAG pipeline -- architecture, APIs, strategies, configuration, and deployment for all 5 services:
- **ctel_ingest** -- Document ingestion (20 connectors, 8 pipelines)
- **ctel_chunk** -- Token-aware document chunking
- **ctel_embed** -- OpenAI embedding generation + Firestore storage
- **ctel_query** -- Semantic search API (Firebase auth, cosine similarity)
- **ctel_eval** -- Evaluation harness (49 golden questions, LLM-as-judge)

### [API & Connector Query Reference](api-reference)
Detailed reference for every API call made by the ingestion layer -- exact endpoints, search terms, query parameters, pagination strategies, rate limits, and authentication for all 20 connectors pulling from federal, state, and commercial data sources.

---

*Last updated: February 2026*
