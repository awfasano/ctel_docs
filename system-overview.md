# CTel RAG System - Comprehensive Technical Documentation

## Table of Contents

1. [System Overview](#system-overview)
2. [Pipeline Architecture](#pipeline-architecture)
3. [ctel_ingest - Document Ingestion Service](#ctel_ingest---document-ingestion-service)
4. [ctel_chunk - Document Chunking Service](#ctel_chunk---document-chunking-service)
5. [ctel_embed - Embedding Generation Service](#ctel_embed---embedding-generation-service)
6. [ctel_query - Semantic Search Service](#ctel_query---semantic-search-service)
7. [ctel_eval - Evaluation Harness](#ctel_eval---evaluation-harness)
8. [Cross-Cutting Concerns](#cross-cutting-concerns)

---

## System Overview

The CTel RAG (Retrieval-Augmented Generation) system is a multi-service pipeline built on Google Cloud Platform that ingests telehealth regulatory, legislative, and policy documents from 20+ federal and state sources, processes them into searchable vector embeddings, and serves semantic search queries from an iOS application. The system is purpose-built for the telehealth compliance domain.

**Technology Stack:**
- **Runtime:** Google Cloud Functions Gen 2 (Python 3.11/3.12)
- **Vector Database:** Google Cloud Firestore (native vector search)
- **Object Storage:** Google Cloud Storage (`ctel-telehealth` bucket)
- **Messaging:** Google Cloud Pub/Sub (event-driven pipeline)
- **Embeddings:** OpenAI `text-embedding-3-small` (1536 dimensions)
- **Authentication:** Firebase Authentication (ID token verification)
- **Secrets:** Google Cloud Secret Manager

---

## Pipeline Architecture

```
                         Cloud Scheduler (cron)
                                |
                                v
                     Pub/Sub: rag.ingest.trigger
                                |
                                v
+----------------------------------------------------------------------+
|                        ctel_ingest                                    |
|  20 connectors fetching from federal, state, and commercial APIs     |
|  SHA-256 dedup -> GCS upload -> Firestore state tracking             |
+----------------------------------------------------------------------+
                                |
                     Pub/Sub: rag.file.ingested
                                |
                                v
+----------------------------------------------------------------------+
|                        ctel_chunk                                     |
|  PDF/DOCX/HTML/TXT parsing -> tiktoken-based sliding window          |
|  512-token chunks with 50-token overlap -> GCS upload                |
+----------------------------------------------------------------------+
                                |
                     Pub/Sub: rag.chunks.ready
                                |
                                v
+----------------------------------------------------------------------+
|                        ctel_embed                                     |
|  OpenAI text-embedding-3-small -> Firestore rag_vectors collection   |
|  Batch processing (100 per call) -> 1536-dim vectors                 |
+----------------------------------------------------------------------+
                                |
                     Pub/Sub: rag.embeddings.stored
                                |
                                v
+----------------------------------------------------------------------+
|                        ctel_query                                     |
|  Firebase auth -> OpenAI query embedding -> Firestore find_nearest   |
|  Cosine similarity -> ranked results with metadata                   |
+----------------------------------------------------------------------+
                                |
                                v
                     iOS App / ctel_chat
                                |
                                v
+----------------------------------------------------------------------+
|                        ctel_eval                                      |
|  49 golden questions -> retrieval + faithfulness + correctness        |
|  LLM-as-judge scoring -> composite metrics                           |
+----------------------------------------------------------------------+
```

---

## ctel_ingest - Document Ingestion Service

### Purpose

Fetches raw documents from 20+ external data sources covering telehealth regulation, legislation, billing, licensing, compliance, policy, clinical evidence, and utilization data. Implements content-hash deduplication and publishes events for downstream processing.

### APIs Used

#### Federal Regulatory Sources (4 APIs)

| API | Auth | Rate Limit | Data Format |
|-----|------|------------|-------------|
| **Federal Register API** (`federalregister.gov/api/v1`) | None | Unrestricted | JSON -> HTML |
| **eCFR Renderer API** (`ecfr.gov/api/renderer/v1`) | None | Unrestricted | HTML |
| **Regulations.gov API** (`api.regulations.gov/v4`) | `api.data.gov` key | 900 req/hr | JSON |
| **Congress.gov API** (`api.congress.gov/v3`) | `api.data.gov` key | Unrestricted | JSON |

- **Federal Register:** Searches for proposed/final rules, notices, and presidential documents across 9 federal agencies (HHS, CMS, FDA, DEA, FTC, FCC, OCR, ONC, SAMHSA) using 7 telehealth-related search terms.
- **eCFR:** Pulls the current text of 19 specific CFR parts related to telehealth (e.g., Title 42 Part 410, Title 21 Part 1300).
- **Regulations.gov:** Retrieves public rulemaking dockets and documents for 7 agencies with 5 search terms.
- **Congress.gov:** Tracks federal bills from the 118th and 119th Congress related to telehealth.

#### State Legislative Sources (2 APIs)

| API | Auth | Rate Limit | Data Format |
|-----|------|------------|-------------|
| **Open States API v3** (`v3.openstates.org`) | API key | 500 req/hr | JSON |
| **LegiScan Pull API** (`api.legiscan.com`) | API key | Default tier | JSON (base64 content) |

- **Open States:** Monitors state bills from all 50 states + DC + Puerto Rico using 7 telehealth search terms.
- **LegiScan:** Supplements state and federal bill tracking with full bill text (base64-decoded).

#### Billing & Coverage Sources (4 connectors)

| Source | Auth | Method | Data Format |
|--------|------|--------|-------------|
| **telehealth.hhs.gov / cms.gov / medicare.gov** | None | Web scraping | HTML + PDF |
| **CMS Telehealth Services List** (`data.cms.gov` + ZIP download) | None | REST + ZIP | XLSX + JSON + PDF |
| **NLM Clinical Tables API** (`clinicaltables.nlm.nih.gov`) | None | REST (20 req/s) | JSON |
| **Payer Policies** (UHC, Aetna, Cigna, Humana, Anthem) | None | Web scraping | HTML + PDF |

- **CMS Telehealth:** Scrapes 13+ billing and policy pages, downloads linked PDFs.
- **CMS Telehealth List:** Downloads the official Medicare Telehealth Services List (ZIP extraction of XLSX), plus PDF billing guides.
- **HCPCS:** Queries the NLM Clinical Tables API for 2025+ 98000-series HCPCS billing codes.
- **Payer Policies:** Scrapes telehealth reimbursement policies from the 5 largest commercial payers.

#### Provider Licensing Sources (2 connectors)

| Source | Auth | Method | Data Format |
|--------|------|--------|-------------|
| **CMS NPPES NPI Registry API** (`npiregistry.cms.hhs.gov`) | None | REST | JSON |
| **Interstate Licensure Compacts** (IMLC, NLC, PSYPACT, etc.) | None | Web scraping + static | HTML |

- **NPPES:** Queries provider demographics across 20 telehealth-relevant specialties, per state.
- **Licensure Compacts:** Tracks 6 interstate compacts (IMLC, NLC, PSYPACT, ASWB, COUN, PT) with member state data.

#### Compliance Sources (2 APIs)

| API | Auth | Method | Data Format |
|-----|------|--------|-------------|
| **OIG LEIE** (`oig.hhs.gov`) | None | CSV download | CSV |
| **SAM.gov Entity API** (`api.sam.gov`) | API key | REST | JSON |

- **OIG LEIE:** Downloads the full Excluded Individuals/Entities List as CSV, builds national/state/NPI summaries.
- **SAM.gov:** Queries federal exclusions and debarments for 10 HHS-related agency codes.

#### Telehealth Policy Sources (2 scrapers)

| Source | Auth | Method | Data Format |
|--------|------|--------|-------------|
| **CCHP** (`cchpca.org`) | None | Web scraping | HTML + PDF |
| **NCSL** (`ncsl.org`) | None | Web scraping | HTML |

- **CCHP (Center for Connected Health Policy):** Scrapes state-level telehealth policy pages for all 50 states + DC, extracts linked PDFs.
- **NCSL (National Conference of State Legislatures):** Scrapes legislative tracking pages and policy briefs.

#### Clinical Evidence Sources (2 APIs)

| API | Auth | Method | Data Format |
|-----|------|--------|-------------|
| **PubMed E-utilities** (`eutils.ncbi.nlm.nih.gov`) | None (optional key) | REST (ESearch + EFetch) | XML -> HTML |
| **QPP Measures** (CMSgov GitHub) | None | GitHub raw JSON | JSON -> HTML |

- **PubMed:** Searches for systematic reviews, meta-analyses, and clinical trials using 5 telehealth queries within a 2-year rolling window.
- **QPP Measures:** Fetches CMS Quality Payment Program MIPS measures data from the official GitHub repo, filters for telehealth relevance.

#### Utilization Data (1 API)

| API | Auth | Method | Data Format |
|-----|------|--------|-------------|
| **CDC Household Pulse Survey** (Socrata API at `data.cdc.gov`) | None | REST JSON | JSON -> HTML |

- Queries 3 Household Pulse Survey datasets for telemedicine utilization statistics with demographic breakdowns.

#### News (1 scraper)

| Source | Auth | Method | Data Format |
|--------|------|--------|-------------|
| **Telehealth News** | None | Web scraping | HTML |

### Ingestion Strategy

**Pipeline Flow:**
```
Cloud Scheduler (per-connector cron) -> Pub/Sub: rag.ingest.trigger
    -> Cloud Function: ingest() -> Connector Registry Lookup
    -> connector.fetch_documents(since=last_fetched)
    -> SHA-256 content hash vs. Firestore stored hash
    -> Changed? Upload to GCS + publish rag.file.ingested
    -> Update Firestore state (hash + timestamp)
```

**Deduplication Mechanism:**
1. Compute SHA-256 hash of the raw document bytes.
2. Query Firestore collection `ingest_state` for document key `{connector_name}__{safe_doc_key}`.
3. If the stored hash matches the computed hash, skip the document (no change detected).
4. If different or new, proceed with upload.

**File ID Generation:**
```
{source}_{safe_key}_{hash[:12]}
Example: federal_register_2026-01234_a1b2c3d4e5f6
```

**GCS Storage Layout:**
```
gs://ctel-telehealth/raw/
  federal_regulatory/federal_register/2026-01234.html
  federal_regulatory/ecfr/title42-part410.html
  state_legislative/open_states/TX-HB1234.html
  billing_coverage/cms_telehealth/telehealth-services.html
  provider_licensing/nppes/cardiology-CA.json
  compliance/oig_leie/national-summary.html
  telehealth_policy/cchp/california.html
  clinical_evidence/pubmed_telehealth/39012345.html
  utilization_data/cdc_telehealth/pulse-survey-2024.html
```

**HTTP Client Strategy:**
- Async HTTP via `httpx` with token-bucket rate limiting per connector.
- Exponential backoff on failure: 2^attempt seconds (2s, 4s, 8s max).
- Automatic retry on HTTP 429, 5xx, and connection errors.
- Default 30-second timeout per request, configurable follow-redirects.

**Scheduling (Cloud Scheduler):**
| Frequency | Connectors |
|-----------|------------|
| Daily | federal_register, regulations_gov, congress_gov |
| Weekly | ecfr, open_states, legiscan, cms_telehealth, payer_policies, sam_gov, cchp, ncsl, pubmed_telehealth, cdc_telehealth |
| Monthly | cms_telehealth_list, hcpcs, nppes, licensure_compacts, oig_leie, qpp_measures |

### Supported Document Formats

| Format | Extensions | MIME Type |
|--------|-----------|-----------|
| HTML | `.html` | `text/html` |
| PDF | `.pdf` | `application/pdf` |
| CSV | `.csv` | `text/csv` |
| XLSX | `.xlsx` | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| XML | (parsed internally) | `application/xml` |

### Configuration

**Required Environment Variables:**
```
GCS_BUCKET=ctel-telehealth
PUBSUB_TOPIC_ID=rag.file.ingested
GOOGLE_CLOUD_PROJECT=ctelrag
FIRESTORE_COLLECTION=ingest_state
```

**API Keys (Google Secret Manager):**
```
REGULATIONS_GOV_API_KEY    # api.data.gov
CONGRESS_GOV_API_KEY       # api.data.gov
OPEN_STATES_API_KEY        # openstates.org
LEGISCAN_API_KEY           # legiscan.com
SAM_GOV_API_KEY            # sam.gov
```

**Deployment:**
- Runtime: Python 3.12
- Memory: 512 MiB
- Timeout: 540s (9 minutes)
- Trigger: Pub/Sub (`rag.ingest.trigger`)

### Key Architecture Patterns

**BaseConnector (Abstract Base Class):**
```python
class BaseConnector(ABC):
    @property
    def name(self) -> str: ...       # e.g., "federal_register"
    @property
    def pipeline(self) -> str: ...   # e.g., "federal_regulatory"
    async def fetch_documents(self, since: datetime | None) -> list[Document]: ...
    async def run(self, settings, *, since=None, dry_run=False) -> int: ...
```

**Document Dataclass:**
```python
@dataclass
class Document:
    unique_key: str       # Stable source identifier
    title: str            # Human-readable name
    content: bytes        # Raw file bytes
    content_type: str     # MIME type
    extension: str        # File extension
    metadata: dict        # Source-specific key-value pairs
```

**StateTracker (Firestore):**
- Collection: `ingest_state`
- Per-document: `{connector}__{doc_key}` -> `{content_hash, updated_at, metadata}`
- Per-connector: `{connector}__meta` -> `{last_fetched}`

**Local Development CLI:**
```bash
python local_runner.py --connector federal_register --dry-run
python local_runner.py --all --since 2026-02-01
python local_runner.py --list
```

---

## ctel_chunk - Document Chunking Service

### Purpose

Receives `rag.file.ingested` events, downloads raw documents from GCS, parses them into plain text, splits text into token-aware overlapping chunks, and stores chunk JSON files back in GCS for downstream embedding.

### APIs Used

| Library/Service | Version | Purpose |
|-----------------|---------|---------|
| **Google Cloud Storage** | `>=2.16.0,<3` | Download raw files, upload chunk JSONs |
| **Google Cloud Pub/Sub** | `>=2.22.0,<3` | Subscribe to ingestion events, publish completion events |
| **Cloud Functions Framework** | `>=3.5.0,<4` | Cloud Function Gen 2 runtime |
| **PyMuPDF (fitz)** | `>=1.24.0,<2` | PDF text extraction with page boundary tracking |
| **python-docx** | `>=1.1.0,<2` | DOCX (Word) document parsing |
| **BeautifulSoup4** | `>=4.12.0,<5` | HTML parsing and tag filtering |
| **tiktoken** | `>=0.7.0,<1` | Token-aware text encoding using `cl100k_base` |
| **cloudevents** | `>=1.11.0,<2` | CloudEvent deserialization for Pub/Sub messages |

### Chunking Strategy

**Algorithm: Token-Aware Sliding Window with Overlap**

The chunking service uses a tokenizer-aligned sliding window approach rather than naive character splitting, ensuring chunks align with the token boundaries used by OpenAI's embedding models.

1. **Tokenization:** Full document text is encoded using `cl100k_base` (tiktoken), the same encoding used by OpenAI's `text-embedding-3-small`, `text-embedding-3-large`, and GPT-4.

2. **Window Parameters:**
   - **Chunk Size:** 512 tokens (default, configurable via `CHUNK_SIZE`)
   - **Chunk Overlap:** 50 tokens (default, configurable via `CHUNK_OVERLAP`)
   - **Step Size:** `chunk_size - chunk_overlap` = 462 tokens per advance

3. **Sliding Window Process:**
   ```
   Full text -> encode to tokens [t0, t1, t2, ..., tN]

   Chunk 0: tokens[0:512]     -> decode to text
   Chunk 1: tokens[462:974]   -> decode to text
   Chunk 2: tokens[924:1436]  -> decode to text
   ...until all tokens covered
   ```

4. **Overlap Rationale:** The 50-token overlap (~10% of chunk size) ensures that sentences and concepts spanning chunk boundaries appear in at least two chunks, improving retrieval recall for queries that target boundary content.

**Structural Metadata Tracking:**
- **Page Numbers (PDF):** Page breaks are tracked as character offsets during parsing. When building chunks, `bisect` performs O(log n) lookups to assign each chunk to its source page (1-indexed).
- **Section Headings (HTML):** `<h1>` through `<h4>` tags are extracted with their character offsets. Each chunk is annotated with the most recent heading that precedes it.

### Document Parsing

| Format | Library | Method |
|--------|---------|--------|
| PDF (`.pdf`) | PyMuPDF (fitz) | Page-by-page text extraction; pages joined with `\n\n`; page boundaries tracked |
| DOCX (`.docx`) | python-docx | Paragraph extraction; non-empty paragraphs joined with `\n\n` |
| HTML (`.html`, `.htm`) | BeautifulSoup4 | Removes `<script>` and `<style>` tags; extracts visible text; captures heading offsets |
| Plain Text (`.txt`, `.md`) | Built-in | UTF-8 decoding with Latin-1 fallback |

All parsers return a `ParseResult` dataclass containing:
```python
@dataclass
class ParseResult:
    text: str                                    # Extracted plain text
    page_breaks: list[int]                       # Character offsets of page starts (PDF only)
    section_headings: list[tuple[int, str]]      # (offset, heading_text) pairs (HTML only)
```

### Pub/Sub Message Contracts

**Input: `rag.file.ingested`**
```json
{
  "file_id": "federal_register_2026-01234_a1b2c3d4e5f6",
  "raw_path": "gs://ctel-telehealth/raw/federal_regulatory/federal_register/2026-01234.html"
}
```

**Output: `rag.chunks.ready`**
```json
{
  "file_id": "federal_register_2026-01234_a1b2c3d4e5f6",
  "chunk_count": 15,
  "gcs_prefix": "gs://ctel-telehealth/chunks/federal_register_2026-01234_a1b2c3d4e5f6/"
}
```

### Chunk JSON Storage Format

Each chunk is stored as an individual JSON file in GCS:

**Path:** `gs://<bucket>/chunks/<file_id>/<file_id>_chunk_<index>.json`

```json
{
  "chunk_id": "federal_register_2026-01234_a1b2c3d4e5f6_chunk_0",
  "file_id": "federal_register_2026-01234_a1b2c3d4e5f6",
  "content": "The committee voted unanimously to advance the bill...",
  "token_count": 498,
  "metadata": {
    "page_number": 1,
    "section": "Introduction",
    "source_file": "2026-01234.html",
    "source_url": "https://federalregister.gov/documents/2026-01234",
    "document_title": "Telehealth Prescribing Rule",
    "source_type": "federal_register",
    "document_type": "rule",
    "agencies": "DEA, HHS",
    "publication_date": "2026-01-15",
    "citation": "91 FR 12345",
    "chunk_index": 0,
    "total_chunks": 15
  }
}
```

**Metadata Propagation:** The chunk service reads GCS blob metadata set by `ctel_ingest` (source_url, document_title, source_type, agencies, publication_date, citation) and flows it into every chunk. Source URL resolution checks multiple metadata keys in order: `source_url`, `url`, `html_url`, `legiscan_url`, `openstates_url`.

### Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GCS_BUCKET` | Yes | -- | GCS bucket for raw files and chunks |
| `PUBSUB_TOPIC` or `PUBSUB_TOPIC_ID` | Yes | -- | Pub/Sub topic for `rag.chunks.ready` |
| `CHUNK_SIZE` | No | `512` | Max tokens per chunk |
| `CHUNK_OVERLAP` | No | `50` | Token overlap between chunks |

**Deployment:**
- Runtime: Python 3.11
- Memory: 1 GB
- Timeout: 300s (5 minutes)
- Trigger: Pub/Sub (`rag.file.ingested`)

---

## ctel_embed - Embedding Generation Service

### Purpose

Receives `rag.chunks.ready` events, loads chunk JSON files from GCS, generates vector embeddings via OpenAI, and upserts the vectors with full metadata into Firestore's `rag_vectors` collection for semantic search.

### APIs Used

| Library/Service | Version | Purpose |
|-----------------|---------|---------|
| **OpenAI Python SDK** | `>=1.30.0,<2` | Vector embedding generation |
| **Google Cloud Storage** | `>=2.16.0,<3` | Read chunk JSON files |
| **Google Cloud Pub/Sub** | `>=2.22.0,<3` | Subscribe to chunk events, publish completion events |
| **Google Cloud Firestore** | `>=2.16.0,<3` | Vector storage with native Vector type |
| **Cloud Functions Framework** | `>=3.5.0,<4` | Cloud Function Gen 2 runtime |
| **cloudevents** | `>=1.11.0,<2` | CloudEvent parsing |

### Embedding Strategy

**Model:** OpenAI `text-embedding-3-small`
- **Dimensions:** 1536
- **Encoding:** Optimized for semantic similarity (cosine distance)
- **Configurable via:** `EMBEDDING_MODEL` environment variable

**Batch Processing:**
1. All chunk content texts are collected into a flat list.
2. Texts are grouped into batches of 100 (configurable via `BATCH_SIZE`).
3. Each batch makes a single OpenAI embeddings API call.
4. Responses are sorted by index within each batch to maintain deterministic ordering.
5. Results are concatenated into a flat list of 1536-dimensional float vectors.

**Why Batch?** Reduces API call overhead. A 1500-chunk document requires only 15 API calls instead of 1500, staying well within OpenAI rate limits while minimizing latency.

### Vector Database: Firestore

**Collection:** `rag_vectors` (configurable)

**Document Schema (per vector):**
```python
{
    "file_id": str,                    # Parent file identifier
    "chunk_id": str,                   # Unique chunk ID (used as Firestore doc ID)
    "content": str,                    # Full chunk text
    "embedding": Vector([...]),        # Firestore native Vector type, 1536 dims
    "source_file": str,                # Original filename
    "source_url": str,                 # Source URL for attribution
    "page_number": int | None,         # PDF page number (1-indexed)
    "section": str | None,             # Section heading
    "chunk_index": int,                # Zero-based position in parent file
    "total_chunks": int,               # Total chunks for parent file
    "token_count": int,                # Token count of content
    "document_title": str,             # Human-readable title
    "source_type": str,                # e.g., "federal_register", "ecfr"
    "document_type": str,              # e.g., "rule", "bill", "policy"
    "agencies": str,                   # Comma-separated agency names
    "publication_date": str,           # ISO date string
    "citation": str,                   # Citation identifier
}
```

**Write Strategy:**
- Firestore batch writes with a 400-document cap per batch (safely under Firestore's 500-write limit).
- Each chunk becomes a Firestore document with its `chunk_id` as the document ID (upsert semantics).
- Embedding stored using Firestore's native `Vector` type, enabling server-side `find_nearest()` queries.
- Remaining vectors after the main loop are committed in a final partial batch.

### Alternative: Pinecone (Implemented but Inactive)

A fully implemented Pinecone storage module exists (`pinecone_store.py`) but is not currently wired into the handler. It supports:
- Configurable index and namespace
- Batch upserts matching the configurable batch size
- Metadata including content, source_file, page_number, section, etc.

### Pub/Sub Message Contracts

**Input: `rag.chunks.ready`**
```json
{
  "file_id": "federal_register_2026-01234_a1b2c3d4e5f6",
  "chunk_count": 15,
  "gcs_prefix": "gs://ctel-telehealth/chunks/federal_register_2026-01234_a1b2c3d4e5f6/"
}
```

**Output: `rag.embeddings.stored`**
```json
{
  "file_id": "federal_register_2026-01234_a1b2c3d4e5f6",
  "vector_count": 15
}
```

### Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | -- | OpenAI API key (from Secret Manager) |
| `GCS_BUCKET` | Yes | -- | GCS bucket containing chunk files |
| `VECTOR_COLLECTION` | No | `rag_vectors` | Firestore collection name |
| `EMBEDDING_MODEL` | No | `text-embedding-3-small` | OpenAI embedding model |
| `BATCH_SIZE` | No | `100` | Batch size for embeddings and Firestore writes |
| `PUBSUB_TOPIC` or `PUBSUB_TOPIC_ID` | No | -- | Output topic (disabled if unset) |

**Deployment:**
- Runtime: Python 3.11
- Memory: 1 GB
- Timeout: 540s (9 minutes)
- Trigger: Pub/Sub (`rag.chunks.ready`)

### Error Handling

The embed service uses a defensive "log and return" strategy to prevent infinite Pub/Sub retries on non-transient errors:
- Missing environment variables: logged, function returns without raising
- Missing `file_id` or `gcs_prefix` in payload: warning logged, event skipped
- No chunk files found in GCS: warning logged, returns
- Embedding count mismatch (OpenAI returned fewer vectors than chunks): error logged, no upsert
- GCS/OpenAI/Firestore failures: logged with stack trace, function returns
- Pub/Sub publish failure: logged but non-blocking (vectors already stored)

---

## ctel_query - Semantic Search Service

### Purpose

Serves as the search API for the iOS application. Accepts natural-language queries, generates query embeddings via OpenAI, performs cosine similarity search against Firestore's `rag_vectors` collection, and returns ranked document chunks with relevance scores and rich metadata.

**Endpoint:** `POST https://ctel-query-sqrfzvc37a-uc.a.run.app`

### APIs Used

| Library/Service | Version | Purpose |
|-----------------|---------|---------|
| **OpenAI Python SDK** | `>=1.30.0,<2` | Query embedding generation (`text-embedding-3-small`) |
| **Firebase Admin SDK** | `>=6.5.0,<7` | Firebase ID token verification |
| **Google Cloud Firestore** | `>=2.16.0,<3` | Native vector search (`find_nearest`) |
| **Cloud Functions Framework** | `>=3.5.0,<4` | HTTP Cloud Function runtime |

### Querying Strategy

The query pipeline is a 5-stage process:

```
[1] Authentication
    Bearer token from Authorization header
    -> firebase_admin.auth.verify_id_token()
    -> Extract uid for logging
                |
[2] Request Parsing
    Parse JSON body: query (required), top_k (default 5), filters (optional)
    Validate non-empty query text
                |
[3] Embedding Generation
    query_text -> OpenAI text-embedding-3-small -> 1536-dim vector
    Track token count for billing visibility
                |
[4] Vector Similarity Search (Firestore)
    collection("rag_vectors")
      .where("section", "in", categories)        # optional pre-filter
      .find_nearest(
          vector_field="embedding",
          query_vector=Vector(embedding),
          distance_measure=COSINE,
          limit=top_k (or top_k*3 for date filtering)
      )
    Score = 1.0 - cosine_distance
                |
[5] Post-Filtering & Response
    Apply min_score threshold
    Apply publication_date_after filter
    Format chunks with metadata
    Return ranked results
```

**Pre-Filtering (Category-Based):**
If `filters.categories` is provided, Firestore applies a `where("section", "in", categories)` clause before the vector search, narrowing the candidate set and reducing computational cost.

**Post-Filtering:**
- **Min Score:** If `filters.min_score` > 0, results below the threshold are excluded client-side after retrieval.
- **Date Filtering:** If `filters.publication_date_after` is set, the service over-fetches `top_k * 3` results from Firestore and filters by `publication_date` client-side, ensuring enough results survive the date filter to fill the requested `top_k`.

**Ranking:** Results are ranked by cosine similarity score (1.0 - cosine_distance), already sorted by Firestore's `find_nearest`. Higher scores = more relevant.

### API Specification

**Request:**
```
POST /query
Authorization: Bearer <firebase_id_token>
Content-Type: application/json

{
  "query": "What are the latest DEA telehealth prescribing rules?",
  "top_k": 5,
  "filters": {
    "categories": ["federal_regulatory"],
    "min_score": 0.7,
    "publication_date_after": "2025-01-01"
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `query` | string | Yes | -- | Natural-language search query |
| `top_k` | integer | No | 5 | Maximum results to return |
| `filters.categories` | string[] | No | -- | Section/category filter (Firestore pre-filter) |
| `filters.min_score` | float | No | 0.0 | Minimum cosine similarity threshold (0.0-1.0) |
| `filters.publication_date_after` | string | No | -- | ISO date string for recency filter |

**Response (200 OK):**
```json
{
  "chunks": [
    {
      "chunk_id": "federal_register_2026-01234_chunk_0",
      "document_id": "federal_register_2026-01234_a1b2c3d4e5f6",
      "document_title": "DEA Telehealth Prescribing Rule",
      "source_url": "https://federalregister.gov/documents/2026-01234",
      "content": "The Drug Enforcement Administration is extending...",
      "relevance_score": 0.92,
      "metadata": {
        "page_number": 1,
        "section": "Introduction",
        "token_count": 498,
        "source_type": "federal_register",
        "document_type": "rule",
        "agencies": "DEA, HHS",
        "publication_date": "2026-01-15",
        "citation": "91 FR 12345"
      }
    }
  ],
  "query_embedding_tokens": 12,
  "search_time_ms": 145
}
```

**Error Responses:**

| Status | Cause |
|--------|-------|
| 400 | Invalid JSON body or missing `query` field |
| 401 | Missing/invalid Authorization header or expired Firebase token |
| 405 | Non-POST method |
| 500 | Configuration error, OpenAI API failure, or Firestore search failure |

**CORS:** Allows all origins (`*`), POST and OPTIONS methods, Authorization and Content-Type headers, 1-hour preflight cache.

### Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | -- | OpenAI API key (Secret Manager) |
| `VECTOR_COLLECTION` | No | `rag_vectors` | Firestore collection |
| `EMBEDDING_MODEL` | No | `text-embedding-3-small` | OpenAI embedding model |
| `DEFAULT_TOP_K` | No | `5` | Default results limit |
| `FIREBASE_PROJECT_ID` | No | `GOOGLE_CLOUD_PROJECT` | Firebase project override |

**Deployment:**
- Runtime: Python 3.11
- Memory: 512 MB
- Timeout: 60s
- Trigger: HTTP (unauthenticated at GCP IAM, authenticated at app layer via Firebase)

### Legacy: Pinecone Search

A `pinecone_search.py` module exists as a reference/fallback from a previous architecture. It is not currently wired into the handler. The system migrated to Firestore native vector search.

---

## ctel_eval - Evaluation Harness

### Purpose

Evaluates the quality of the CTel RAG system end-to-end using 49 golden questions across 10 telehealth domain pipelines. Measures retrieval quality, answer faithfulness, and answer correctness using a combination of deterministic metrics and LLM-as-judge scoring.

### APIs Used

| Library/Service | Version | Purpose |
|-----------------|---------|---------|
| **OpenAI Python SDK** | `>=1.30` | LLM-as-judge scoring (faithfulness + correctness) and local-mode embeddings |
| **httpx** | `>=0.27` | HTTP client for calling `ctel_query` and `ctel_chat` endpoints |
| **Google Cloud Firestore** | (transitive) | Direct vector search in local mode |
| **Google Secret Manager** | (transitive) | Retrieve `OPENAI_API_KEY` when not in env |

### Evaluation Strategy

The evaluation framework implements a **3-layer scoring approach** plus a composite metric:

#### Layer 1: Retrieval Quality (Deterministic)

Measures whether the correct source documents were retrieved.

| Metric | Formula | Description |
|--------|---------|-------------|
| `source_recall` | found / expected | Fraction of expected sources actually retrieved |
| `source_precision` | from_expected / total_retrieved | Fraction of retrieved chunks from expected sources |

Comparison is done by matching retrieved chunk source identifiers against the `expected_sources` list in each golden question.

#### Layer 2: Faithfulness (LLM-as-Judge)

Evaluates whether the generated answer is grounded in the retrieved context (no hallucination).

- **Judge Model:** `gpt-4o-mini` (configurable via `JUDGE_MODEL`)
- **System Prompt:** "You are a strict evaluator of RAG system faithfulness"
- **Scale:** 1-5 Likert
  - 1 = Mostly fabricated, not supported by sources
  - 2 = Significant unsupported claims
  - 3 = Partially supported, some claims lack backing
  - 4 = Mostly grounded, minor unsupported details
  - 5 = Fully grounded in retrieved documents
- **Output:** `faithfulness_score` (1-5) + `reasoning` (free text)
- **Method:** OpenAI chat completion with `response_format={"type": "json_object"}`

#### Layer 3: Correctness (LLM-as-Judge)

Evaluates whether the generated answer matches the reference answer from the golden question set.

- **Judge Model:** `gpt-4o-mini` (configurable)
- **Scale:** 1-5 Likert
  - 1 = Completely wrong or contradicts reference
  - 2 = Mostly incorrect, misses key points
  - 3 = Partially correct, captures some key points
  - 4 = Mostly correct, captures main points with minor gaps
  - 5 = Fully correct, captures all key points
- **Output:** `correctness_score` (1-5) + `reasoning` (free text)

#### Keyword Coverage (Deterministic)

Simple substring matching of expected keywords in the answer text (case-insensitive).

| Metric | Formula | Description |
|--------|---------|-------------|
| `keyword_coverage` | found / expected | Fraction of expected keywords present in answer |

#### Composite Score

Weighted average of all metrics, normalized to 0-1:

```
If LLM judge enabled:
  composite = 0.25 * retrieval_recall
            + 0.25 * keyword_coverage
            + 0.25 * (faithfulness - 1) / 4
            + 0.25 * (correctness - 1) / 4

If LLM judge skipped:
  composite = 0.50 * retrieval_recall
            + 0.50 * keyword_coverage
```

### Golden Questions Dataset

**49 questions across 10 pipelines:**

| Pipeline | Questions | Topics |
|----------|-----------|--------|
| `federal_regulatory` | 6 | DEA prescribing, CMS reimbursement, FTC oversight |
| `state_legislative` | 5 | Parity laws, audio-only, interstate compacts |
| `telehealth_policy` | 4 | CCHP resources, NTTRC |
| `provider_licensing` | 5 | State licensing, IMLC, NLC, PSYPACT |
| `billing_coverage` | 6 | Reimbursement, parity, CPT codes |
| `clinical_evidence` | 5 | Clinical guidelines, outcomes research |
| `compliance` | 5 | HIPAA, security requirements |
| `quality_measures` | 3 | Quality measurement frameworks |
| `utilization_data` | 4 | Telehealth adoption statistics |
| `cross_cutting` | 6 | General telehealth topics |

Each question includes:
- `id` - Unique identifier (e.g., `fed_reg_01`)
- `pipeline` - Domain category
- `question` - Natural-language query
- `expected_sources` - List of source identifiers that should be retrieved
- `expected_keywords` - List of keywords that should appear in the answer
- `reference_answer` - Gold-standard answer for correctness comparison
- `difficulty` - Rating (easy/medium/hard)

### Execution Modes

**Remote Mode (default):** Calls `ctel_query` and `ctel_chat` Cloud Function endpoints with Firebase authentication.
```bash
export CTEL_QUERY_URL=https://ctel-query-sqrfzvc37a-uc.a.run.app
export CTEL_CHAT_URL=https://ctel-chat-sqrfzvc37a-uc.a.run.app
export CTEL_AUTH_TOKEN=<firebase_id_token>
export OPENAI_API_KEY=sk-...
python run_eval.py
```

**Local Mode (`--local`):** Bypasses Cloud Functions entirely. Queries Firestore directly for vector search and calls OpenAI directly for answer generation. No Firebase auth needed.
```bash
export OPENAI_API_KEY=sk-...
python run_eval.py --local
```

### CLI Options

```bash
python run_eval.py                                    # All 49 questions
python run_eval.py --ids fed_reg_01,billing_01       # Specific questions
python run_eval.py --pipeline billing_coverage       # Single pipeline
python run_eval.py --skip-llm-judge                  # Skip LLM scoring (faster/cheaper)
python run_eval.py --output results.json             # Custom output file
python run_eval.py --local                           # Direct Firestore + OpenAI
```

### Output Format

Results are written to a timestamped JSON file (e.g., `eval_results_20260215_062959.json`):

```json
{
  "run_timestamp": "2026-02-15T06:20:02.847939",
  "elapsed_seconds": 250.1,
  "question_count": 49,
  "config": {
    "mode": "local",
    "query_url": "firestore://ctelrag/rag_vectors",
    "chat_url": "openai://gpt-4o-mini",
    "judge_model": "gpt-4o-mini",
    "top_k": 5,
    "skip_llm_judge": false
  },
  "results": [
    {
      "id": "fed_reg_01",
      "pipeline": "federal_regulatory",
      "question": "...",
      "difficulty": "medium",
      "retrieval_scores": {
        "source_recall": 0.5,
        "source_precision": 0.4,
        "sources_found": ["ecfr"],
        "sources_missing": ["federal_register"]
      },
      "keyword_scores": {
        "keyword_coverage": 0.8,
        "keywords_found": ["prescribing", "telehealth"],
        "keywords_missing": ["schedule II"]
      },
      "faithfulness_scores": {
        "faithfulness_score": 5,
        "reasoning": "All claims are directly supported by the retrieved documents."
      },
      "correctness_scores": {
        "correctness_score": 4,
        "reasoning": "Captures main points but misses the specific effective date."
      },
      "composite": {
        "retrieval_recall": 0.5,
        "keyword_coverage": 0.8,
        "faithfulness": 1.0,
        "correctness": 0.75,
        "composite": 0.763
      },
      "answer": { "text": "...", "citations": [...], "confidence": "high" }
    }
  ]
}
```

### Sample Results

From evaluation runs:
- **Without LLM judge:** Average composite 0.329 (retrieval + keyword only)
- **With LLM judge:** Average composite 0.583
  - Best pipeline: `utilization_data` (0.753)
  - Lowest pipeline: `compliance` (0.513)

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `CTEL_QUERY_URL` | `http://localhost:8082` | Query endpoint URL |
| `CTEL_CHAT_URL` | `http://localhost:8083` | Chat endpoint URL |
| `CTEL_AUTH_TOKEN` | `""` | Firebase Bearer token |
| `OPENAI_API_KEY` | `""` (or from Secret Manager) | OpenAI API key |
| `JUDGE_MODEL` | `gpt-4o-mini` | LLM judge model |
| `EVAL_TOP_K` | `5` | Number of chunks to retrieve |

---

## Cross-Cutting Concerns

### Authentication Flow

The entire system uses **Google Cloud IAM** for service-to-service authentication and **Firebase Authentication** for client-to-service authentication:

- **Service-to-service:** Cloud Functions use default service accounts with appropriate IAM roles for GCS, Firestore, Pub/Sub, and Secret Manager access.
- **Client-to-service:** The iOS app obtains Firebase ID tokens and passes them as `Authorization: Bearer <token>` headers to `ctel_query`. The query service verifies tokens using `firebase_admin.auth.verify_id_token()`.

### Secrets Management

All API keys are stored in **Google Cloud Secret Manager** and injected at deployment time via `--set-secrets`:
- `openai-api-key` -> `OPENAI_API_KEY`
- `regulations-gov-api-key` -> `REGULATIONS_GOV_API_KEY`
- `congress-gov-api-key` -> `CONGRESS_GOV_API_KEY`
- `open-states-api-key` -> `OPEN_STATES_API_KEY`
- `legiscan-api-key` -> `LEGISCAN_API_KEY`
- `sam-gov-api-key` -> `SAM_GOV_API_KEY`

### Client Initialization Pattern

All five services use the same **lazy singleton** pattern for external clients:
```python
_client: SomeClient | None = None

def _get_client(settings: Settings) -> SomeClient:
    global _client
    if _client is None:
        _client = SomeClient(api_key=settings.api_key)
    return _client
```
This amortizes initialization cost across warm Cloud Function invocations while keeping the first cold start fast.

### Metadata Flow Through the Pipeline

Document metadata established at ingestion propagates through every stage:

```
ctel_ingest                              ctel_chunk                     ctel_embed                    ctel_query
 Document.metadata         ->        GCS blob metadata    ->     Firestore document fields   ->  Response metadata
  source_url                           source_url                    source_url                     source_url
  title                                document_title                document_title                 document_title
  source_type                          source_type                   source_type                    source_type
  document_type                        document_type                 document_type                  document_type
  agencies                             agencies                      agencies                       agencies
  publication_date                     publication_date              publication_date               publication_date
  citation                             citation                      citation                       citation
```

This ensures that every search result returned to the iOS app carries full provenance for source attribution and citation.

### GCS Bucket Organization

```
gs://ctel-telehealth/
├── raw/                              # ctel_ingest writes here
│   ├── federal_regulatory/
│   │   ├── federal_register/
│   │   ├── ecfr/
│   │   └── regulations_gov/
│   ├── state_legislative/
│   │   ├── congress_gov/
│   │   ├── open_states/
│   │   └── legiscan/
│   ├── billing_coverage/
│   ├── provider_licensing/
│   ├── compliance/
│   ├── telehealth_policy/
│   ├── clinical_evidence/
│   └── utilization_data/
└── chunks/                           # ctel_chunk writes here
    ├── federal_register_2026-01234_a1b2c3d4e5f6/
    │   ├── ..._chunk_0.json
    │   ├── ..._chunk_1.json
    │   └── ..._chunk_N.json
    └── ecfr_title42-part410_b2c3d4e5f6a1/
        └── ...
```

### Firestore Collections

| Collection | Service | Purpose |
|------------|---------|---------|
| `ingest_state` | ctel_ingest | Content hash dedup and last-fetched timestamps |
| `rag_vectors` | ctel_embed (write), ctel_query (read) | Vector embeddings with metadata for semantic search |

### Pub/Sub Topics

| Topic | Publisher | Subscriber |
|-------|-----------|------------|
| `rag.ingest.trigger` | Cloud Scheduler | ctel_ingest |
| `rag.file.ingested` | ctel_ingest | ctel_chunk |
| `rag.chunks.ready` | ctel_chunk | ctel_embed |
| `rag.embeddings.stored` | ctel_embed | (downstream consumers) |
