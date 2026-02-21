---
layout: default
title: CTel RAG System Documentation
---

# CTel RAG System Documentation

Technical documentation for the CTel Telehealth RAG (Retrieval-Augmented Generation) pipeline.

## Documents

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
