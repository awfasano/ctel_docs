# CTEL RAG: Architecture Deep Dive

How the CTEL retrieval pipeline works, how it differs from standard RAG, and the design decisions behind each stage.

---

## Table of Contents

1. [What This System Is](#what-this-system-is)
2. [How It Differs From Standard RAG](#how-it-differs-from-standard-rag)
3. [Ingestion: Content-Addressed Document IDs](#ingestion-content-addressed-document-ids)
4. [Chunking: Legal-Aware Splitting](#chunking-legal-aware-splitting)
5. [Document Summary Chunks](#document-summary-chunks)
6. [Contextual Retrieval (LLM-Generated Chunk Prefixes)](#contextual-retrieval-llm-generated-chunk-prefixes)
7. [Structured State Metadata](#structured-state-metadata)
8. [Query Pipeline: 8-Stage Ranking](#query-pipeline-8-stage-ranking)
9. [Semantic Query Cache](#semantic-query-cache)
10. [Factual Grounding Check](#factual-grounding-check)
11. [Query Audit Trail](#query-audit-trail)
12. [Comparison Table](#comparison-table)
13. [Late Chunking Research Notes](#late-chunking-research-notes)

---

## What This System Is

CTEL is a retrieval-augmented generation pipeline built for telehealth regulatory intelligence. It ingests documents from 20+ federal and state sources -- legislative bills, federal regulations, CDC survey data, PubMed research, CMS billing rules, OIG exclusion lists -- and makes them searchable through a multi-stage ranking pipeline optimized for legal/regulatory content.

The system is split into four services: **ingest**, **chunk**, **embed**, **query**. Each runs as a Google Cloud Function triggered by Pub/Sub events, forming an asynchronous pipeline.

```
Connectors (20+) -> GCS raw files
  -> Pub/Sub: rag.file.ingested
    -> Chunk CF: parse, split, summarize, upload to GCS
      -> Pub/Sub: rag.chunks.ready
        -> Embed CF: contextualize (LLM), embed (OpenAI), upsert to Firestore
          -> Query CF (HTTP): 8-stage retrieval pipeline + audit trail
```

---

## How It Differs From Standard RAG

A standard RAG tutorial looks like this:

```
Split document into chunks -> Embed chunks -> Store in vector DB
-> Query: embed question -> cosine similarity -> top K -> feed to LLM
```

That works for simple use cases. It does not work when you have 78K+ vectors from 20+ source types, where a query about "Ohio telehealth laws" pulls up CDC survey data that mentions Ohio, New Jersey bills that are semantically similar to "telehealth laws," and only one actual Ohio bill -- but just its 89-token "Recent Actions" fragment.

What follows is how we solve each of these problems.

---

## Ingestion: Content-Addressed Document IDs

Standard RAG uses UUIDs or auto-incrementing IDs for documents. We use content-addressed file IDs:

```
file_id = "{source}_{safe_key}_{sha256[:12]}"
```

The hash is deterministic from content, not from timestamp or insertion order. The prefix (`source_key = file_id[:-13]`) stays stable across content changes. This means:

- Re-ingesting the same document creates a new file_id (new hash) but the same source_key
- Before upserting new vectors, we query Firestore for all vectors with the same source_key and delete them
- No orphaned vectors. No version conflicts. No manual cleanup.

Standard RAG doesn't have this problem because most tutorials don't re-ingest documents. We do -- state bills get amended, federal rules get updated, CDC data refreshes quarterly.

**Connector metadata enrichment**: Each of the 20+ connectors (Open States, LegiScan, eCFR, Federal Register, CDC, PubMed, SAM.gov, Congress.gov, etc.) writes structured metadata to GCS blob metadata at ingest time. Fields like `state`, `identifier`, `session`, `abstract`, `document_number` are set by the connector that understands the source's data model, then propagated through the entire pipeline without any downstream service needing source-specific logic.

---

## Chunking: Legal-Aware Splitting

Standard RAG splits on character count or token count with a fixed overlap window. This breaks legal documents in the wrong places.

### Legal Abbreviation Protection

80+ hardcoded abbreviations (U.S.C., C.F.R., Sec., e.g., et al., vs., Dept., Inc.) are protected during sentence splitting. A standard sentence tokenizer treats "42 U.S.C." as a sentence boundary -- we don't.

### Legal Heading Detection

Regex patterns detect structural headings -- `SECTION`, `PART`, `TITLE`, `CHAPTER`, subsection markers like `(a)`, numbered sections like `1.2.3`, and all-caps lines. Chunks never split across these boundaries. A standard character splitter doesn't know what a heading is.

### Priority Separator Hierarchy

Instead of one split strategy, we use a cascade:

1. Paragraph breaks (`\n\n`)
2. Legal headings
3. Sentence boundaries (abbreviation-aware)
4. Single newlines
5. Word boundaries
6. Hard token limit (fallback)

This preserves document structure. An eCFR regulation with subsections `(a)`, `(b)`, `(c)` gets split at subsection boundaries, not mid-sentence.

### Semantic XLSX Parsing

Spreadsheet rows are rendered as `Header: Value | Header: Value` pairs, not raw cell dumps. This means an embedding can capture "State: Ohio | Telehealth Utilization: 24.3%" instead of "Ohio 24.3" with no column context.

### BM25 Keyword Enrichment

Each chunk gets a `bm25_keywords` field generated from 29 bidirectional synonym groups. If a chunk contains "telehealth," the keywords include "telemedicine, virtual care, remote patient monitoring." If it contains "CPT," the keywords include "Current Procedural Terminology." Standard RAG relies entirely on vector similarity. We augment with lexical matching on expanded terminology so that a user searching "telemedicine" finds chunks that say "telehealth."

---

## Document Summary Chunks

This is the most significant departure from standard RAG. Every document gets a synthetic summary chunk -- a template-based prose description of what the document is, where it's from, and what it's about. No LLM calls; pure string formatting from connector metadata.

For an Open States bill:

> "This is a state legislative bill from Ohio. Bill SB352 (136th GA). Title: Regards behavioral health screenings via telehealth. Source: Open States."

For a Federal Register rule:

> "This is a Final Rule published in the Federal Register by CMS. Document: 2024-12345. Citation: 89 FR 12345. Title: Medicare and Medicaid Programs; CY 2025 Payment Policies."

9 source-type-specific templates (`open_states`, `legiscan`, `federal_register`, `ecfr`, `cdc_telehealth`, `pubmed_telehealth`, `sam_gov`, `congress_gov`) plus a default fallback. Each template skips empty fields gracefully -- no "Title: . Citation: ." when metadata is sparse.

### Why This Matters

In standard RAG, each chunk floats in isolation. A 200-token chunk from an Ohio bill that says "Section 4. The Director shall establish standards for behavioral health screenings" has no context about what document it belongs to, what state it's from, or what the bill is called. Its embedding captures the semantic meaning of that sentence, but not the document-level context. When a user asks "Ohio telehealth laws," this chunk might not rank high because there's no "Ohio" or "telehealth" in it -- it's just about "behavioral health screenings."

The summary chunk's embedding captures "Ohio state bill telehealth behavioral health" all at once. It ranks high for the query. And because it shares a `file_id` with the content chunks, the downstream system knows they belong together.

### Summary Chunk Format

```json
{
  "chunk_id": "{file_id}_chunk_summary",
  "chunk_index": -1,
  "section": "Document Summary",
  "chunk_type": "summary",
  "state": "Ohio",
  "identifier": "SB352",
  "session": "136th General Assembly"
}
```

The summary chunk uses `chunk_type: "summary"`, `chunk_index: -1` (sorts before content), and `section: "Document Summary"`. It carries the same structured metadata as content chunks, so it participates in all the same filtering and scoring logic.

---

## Contextual Retrieval (LLM-Generated Chunk Prefixes)

Standard RAG embeds chunks as-is. We optionally prepend an LLM-generated context sentence to each chunk before embedding.

Using GPT-4o-mini in batches of 10 chunks, the contextualizer generates 1-2 sentence prefixes like:

> "This chunk from the 2024 CMS Physician Fee Schedule discusses telehealth payment rates under Section III.D."

This is prepended to the chunk content before embedding:

```
{context_prefix}

{original_chunk_content}
```

The original content is preserved in `content_original` so downstream systems have both. Token counts are updated to reflect the new length.

### Token Budget Management

The contextualizer tracks token usage per batch. If a batch exceeds the 128K model limit (minus 500 for prompt overhead, minus 2000 for response reserve), it proportionally truncates chunk content to fit -- with a floor of 200 characters per chunk.

### Summary Chunks Skip Contextualization

Summary chunks are already structured descriptions -- they don't need an LLM context prefix. This saves one API call per document.

This is Anthropic's contextual retrieval pattern, adapted for batch processing and domain-specific content.

---

## Structured State Metadata

Standard RAG has no concept of geographic relevance. If a user asks about Ohio, the only way to filter is to search for "Ohio" in the content text -- which also matches CDC data that mentions Ohio in a table row, or federal laws that list Ohio alongside 49 other states.

We propagate a structured `state` field from connector metadata through the entire pipeline:

```
Connector sets state="Ohio" on GCS blob
  -> Chunk service reads it, attaches to every chunk
    -> Embed service persists to Firestore
      -> Query service reads it for instant classification
```

In the query service's state classification function, the structured field is checked **first**:

```python
chunk_state = meta.get("state", "")
if chunk_state:
    if matches target -> "target_state"     # 1.0x (no penalty)
    if source is legislation -> "other_state"  # 0.05x (near-zero)
```

Only if the field is empty does it fall back to URL parsing and content scanning. This makes state classification instant and deterministic for new documents, while remaining backward-compatible with existing vectors that don't have the field.

---

## Query Pipeline: 8-Stage Ranking

This is where the system diverges most from standard RAG. Instead of "embed query -> cosine similarity -> top K," we run an 8-stage pipeline:

### Stage 1: Query Decomposition

Complex queries are broken into sub-queries by GPT-4o-mini.

"How do federal and state telehealth laws interact?" becomes:
- "federal telehealth regulations"
- "state telehealth parity laws"
- "federal preemption of state telehealth laws"

Each sub-query runs through the rest of the pipeline independently, with results merged at the end.

**Standard RAG:** Single query, single embedding, single search.

### Stage 2: Multi-Query RAG-Fusion

For each query/sub-query, GPT-4o-mini generates 3 variant phrasings at temperature 0.7. "Medicare billing codes for telehealth" might also become "CPT codes for telehealth services" and "CMS reimbursement rates for virtual visits." Each variant gets its own embedding and search, with results merged by keeping the highest score per chunk_id.

**Standard RAG:** One embedding per query.

### Stage 3: Vector Search with Source-Targeted Supplementary Search

The primary search fetches `10 * top_k` candidates (100 by default) using Firestore's native vector search with cosine distance.

Then, based on detected query intent (legislation, billing, clinical, etc.), a **second search** runs filtered to boosted source types. If the query matches "legislation" intent, we search specifically within `open_states`, `legiscan`, and `congress_gov` and merge up to 20 additional candidates. This guarantees legislative sources make it into the pipeline even if CDC data dominates raw vector similarity.

**Standard RAG:** One search, one vector index, no source awareness.

### Stage 4: BM25 Hybrid Scoring with Reciprocal Rank Fusion

Candidates are re-scored using BM25 (via `rank_bm25.BM25Okapi`) on expanded content (chunk content + synonym-expanded keywords). Scores are fused with vector scores using RRF:

```
score = 1/(60 + vector_rank) + 1/(60 + bm25_rank)
```

This helps surface exact term matches -- CFR section numbers, CPT codes, bill identifiers -- that vector similarity misses because they're low-frequency tokens.

**Standard RAG:** Vector similarity only, or BM25 only. Rarely both.

### Stage 5: Source-Diverse Candidate Reservation

Before reranking, we reserve 5 candidates per boosted source type in the candidate pool (50 total). This prevents the reranker from only seeing the top 50 by score -- which could all be from one source type.

**Standard RAG:** Top N by score, whatever source they're from.

### Stage 6: LLM Reranking

Top 25 candidates are reranked by Cohere Rerank 3.5 (or GPT-4o-mini fallback). The LLM scores each chunk's relevance to the query, reordering based on semantic understanding rather than just embedding distance.

**Standard RAG:** No reranking, or reranking without source diversity guarantees.

### Stage 7: Source-Type and State Penalties

After reranking, intent-specific penalties are applied:

**Source-type penalties (7 intent rules):**

| Intent | Example Patterns | Boosted Sources | Penalized Sources |
|--------|-----------------|-----------------|-------------------|
| legislation | law, bill, statute, SB123 | open_states, legiscan, congress_gov | cdc (0.5x), pubmed (0.5x) |
| regulation | regulate, CFR, Federal Register | federal_register, ecfr | cdc (0.6x), pubmed (0.6x) |
| billing_coverage | billing, CPT, reimburse, CMS | cms_telehealth, ecfr | cdc (0.6x), pubmed (0.6x) |
| clinical_research | study, research, meta-analysis | pubmed, cdc | open_states (0.6x), legiscan (0.6x) |
| utilization_data | utilization, statistics, survey | cdc, pubmed | open_states (0.6x), legiscan (0.6x) |
| compliance_fraud | fraud, compliance, OIG, LEIE | oig_leie, federal_register | cdc (0.5x), pubmed (0.5x) |
| licensure | license, compact, interstate | licensure_compacts, open_states | cdc (0.6x), pubmed (0.6x) |

**State-relevance penalties:**

| Classification | Multiplier | Example |
|---------------|------------|---------|
| target_state | 1.0x | Ohio bill for "Ohio telehealth laws" query |
| federal | 0.6x | Federal rules apply everywhere |
| national_data | 0.15x | CDC generic data |
| other_state | 0.05x | New Jersey bill for Ohio query |
| unknown | 0.4x | Can't determine |

**Standard RAG:** No concept of source type relevance or geographic filtering.

### Stage 8: Progressive Source Diversification

Final results are selected using round-robin across source types -- cap of 1 per source first, then 2, then 3, until top_k is filled. Prevents any single large source collection from sweeping all result slots.

**Standard RAG:** Top K by score, period.

---

## Semantic Query Cache

An in-memory LRU cache stores recent query results keyed by embedding vectors. If a new query's embedding has cosine similarity >= 0.95 with a cached query, the cached results are returned immediately -- skipping all 8 pipeline stages.

The threshold is deliberately high (0.95). Lower thresholds risk false positives where "Ohio telehealth laws" matches "Texas telehealth laws." At 0.95, only near-duplicate queries hit the cache.

- Max entries: 500
- TTL: 1 hour
- Thread-safe with LRU eviction

---

## Factual Grounding Check

After generating an answer from retrieved chunks, GPT-4o-mini extracts individual claims from the answer and checks whether each is supported by at least one retrieved chunk. Returns a grounding score (0.0-1.0) and identifies unsupported claims.

This is a post-hoc verification step. Standard RAG assumes the LLM's answer is grounded because it was given relevant context. We verify.

---

## Query Audit Trail

Every query -- cache hit or fresh search -- writes an audit record to a `rag_query_audit` Firestore collection. Each record includes:

| Field | Purpose |
|-------|---------|
| `user_id` | Firebase UID -- who asked |
| `query` | Original query text |
| `timestamp` | Firestore server timestamp (clock-skew safe) |
| `search_time_ms` | Wall-clock latency |
| `cache_hit` | Whether the result came from semantic cache |
| `result_count` | How many chunks were returned |
| `results[].rank` | Final position (0-indexed) |
| `results[].chunk_id` | Which chunk |
| `results[].document_id` | Which parent document |
| `results[].source_type` | Where it came from (open_states, ecfr, etc.) |
| `results[].chunk_type` | summary vs content |
| `results[].state` | Structured state field |
| `results[].relevance_score` | Final score after all penalties |
| `filters` | Any filters the user applied |
| `sub_queries` | Decomposed sub-queries (if decomposition ran) |
| `total_embedding_tokens` | Token consumption for cost tracking |

### Design Decisions

- **Fire-and-forget**: Writes happen in a daemon thread. The query response is never delayed by an audit write. If the write fails, it's logged and the query still succeeds.
- **Separate collection**: No impact on vector search performance.
- **Compact entries**: Stores chunk_id and metadata per result, not full content. Enough to reconstruct ranking decisions without bloating storage.
- **Both code paths**: Audit runs on both cache hits and fresh searches.

This lets you answer "what did we show user X for query Y on a given date, and why did chunk Z rank at position 3" -- which matters when you're serving legal/regulatory content.

---

## Comparison Table

| Capability | Standard RAG | CTEL |
|-----------|-------------|------|
| Chunking | Fixed-size character/token split | Legal-aware hierarchical splitting with abbreviation protection |
| Document context | None -- chunks float in isolation | Summary chunks + LLM context prefixes |
| State/geographic filtering | Text search for state name | Structured metadata field, instant classification |
| Query processing | Embed -> search -> return | Decompose -> multi-query -> vector + BM25 -> diverse pool -> LLM rerank -> source/state penalties -> diversify |
| Source type awareness | None | Intent detection drives boosting, penalties, and supplementary searches |
| Keyword matching | Vector-only or BM25-only | Synonym-expanded BM25 fused with vectors via RRF |
| Reranking | None or single-pass | Multi-backend (Cohere + OpenAI fallback) on source-diverse candidate pool |
| Caching | None or exact match | Semantic cache (cosine similarity >= 0.95, LRU, TTL) |
| Answer verification | Trust the LLM | Claim-level grounding check against retrieved chunks |
| Audit trail | None | Fire-and-forget Firestore audit: who, what, when, why, how scored |
| Document ID strategy | UUID or auto-increment | Content-addressed hash with stable source key prefix |
| Re-ingestion | Manual or append-only | Automatic stale vector cleanup via source key |

---

## Late Chunking Research Notes

Late chunking (arXiv 2409.04701, Jina AI) is an alternative approach to preserving cross-chunk context. Instead of splitting first and embedding each chunk independently (naive chunking), it feeds the entire document through a long-context embedding model first, gets token-level embeddings where self-attention sees the whole document, then applies mean pooling to each chunk's token span afterward.

### Performance

Consistently outperforms naive chunking (2-28% nDCG@10 improvement depending on document length). The gain scales with document length -- short documents see minimal benefit.

### Why We Don't Use It (Yet)

Late chunking requires a long-context embedding model that exposes token-level outputs (Jina v2/v3, or a self-hosted model). Our pipeline uses OpenAI's embedding API (`text-embedding-3-large`), which only returns the final pooled vector -- you can't intercept token-level embeddings to do per-chunk pooling.

### Current Equivalent

Our stack (naive chunking + contextual retrieval + document summary chunks) achieves similar goals through different means:

- **Summary chunks** capture document-level context ("this is an Ohio bill about telehealth")
- **Contextual retrieval** adds chunk-level context ("this chunk discusses payment rates under Section III.D")
- **BM25 keyword enrichment** adds lexical bridge terms across synonyms

Worth revisiting if the pipeline moves to self-hosted embeddings.

### References

- [Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models](https://arxiv.org/abs/2409.04701)
- [Weaviate: Late Chunking -- Balancing Precision and Cost in Long Context Retrieval](https://weaviate.io/blog/late-chunking)
- [Jina AI: Late Chunking in Long-Context Embedding Models](https://jina.ai/news/late-chunking-in-long-context-embedding-models/)

---

*Last updated: February 2026*
