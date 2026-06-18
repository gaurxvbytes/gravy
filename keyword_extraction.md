# Keyword Extraction Process

This document provides a high level explanation of how search keywords are extracted from brand crawled content for domain verticals in the Gravton Console platform.

## Objective

The keyword extraction step parses crawled brand content and extracts a set of highly relevant, search-like keywords tailored to specific active product verticals. These keywords serve as inputs for downstream tasks like clustering, search volume estimation, and prompt generation.

The extraction pipeline is designed to prevent context window overflows (e.g. HTTP 413/400 errors) on massive crawls and eliminate "Lost in the Middle" LLM hallucinations by using a localized Retrieval-Augmented Generation (RAG) approach.

---

## Architecture: In-Memory Gemini Embedding RAG

Instead of feeding the entire crawl payload to the LLM in a single request, the system uses an in-memory RAG pipeline:
1. **Document Chunking**: The crawl dataset is split into smaller, overlapping text chunks.
2. **Text Chunk Embedding**: Chunks are embedded using a high-density embedding model.
3. **Query Embedding & Matching**: For each active product vertical, a semantic query vector is generated and compared against all chunk vectors in memory to identify the most relevant context.
4. **Parallel Target Generation**: The LLM processes each vertical in parallel (multi-threaded), receiving only the top semantically relevant chunks for that vertical.

---

## Detailed Implementation Steps

The process is orchestrated in the Airflow task `extract_keywords` within the file [keyword_dag.py](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/airflow/dags/keyword_dag.py#L525).

### 1. Run Context and Data Loading
- **Database Resolution**: The task fetches the domain settings and lists all active product verticals associated with it (`ProductVertical` models).
- **Competitor Deduplication**: Vertical names are deduplicated to avoid redundant generation calls.
- **Crawl Data Fetching**: The latest crawl payload (`enriched_payload`) is retrieved from S3.

### 2. Text Chunking
The helper function [_extract_text_chunks](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/airflow/dags/keyword_dag.py#L284) performs the text parsing:
- It processes pages from the crawled dataset.
- For each page, it prioritizes structured section contents (`sections[].content` with its heading).
- If sections are unavailable, it falls back to parsing page fields (`text`, `markdown`, `readableText`, or `content`).
- Chunks are extracted using a sliding window method [_slide_window](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/airflow/dags/keyword_dag.py#L273) with a size of 3,000 characters and a 300 character overlap.

### 3. In-Memory Vector Search
If verticals and text chunks are both present, the task executes the RAG retrieval flow:
- **Chunk Matrix Embedding**: All text chunks are embedded as documents using the helper function [_gemini_embed](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/airflow/dags/keyword_dag.py#L340) and the embedding model defined by `KEYWORD_EMBED_MODEL` (defaulting to `"google/gemini-embedding-2"`). The output vectors are normalized to L2 norm.
- **Vertical Query Construction**: For each unique product vertical, a search query is built from its name and businesses/services descriptions (`vertical.product_businesses_services`).
- **Semantic Score Calculation**: The query is embedded as a retrieval query. Cosine similarity is computed via matrix multiplication against all document chunks:
  ```python
  scores = (q_emb @ chunk_matrix.T)[0]
  ```
- **Context Selection**: The top-8 chunks with the highest scores are extracted to serve as the context window for that vertical.

### 4. Parallel LLM Extraction
- **Concurrency**: A Python `ThreadPoolExecutor` runs parallel extractions for all verticals (up to a maximum of 6 parallel threads).
- **LLM Call**: Each thread runs `_extract_for_vertical`, invoking the generation model defined by `KEYWORD_EXTRACTION_MODEL` (defaulting to `"openai/gpt-4.1-mini"` via OpenRouter).
- **Prompting**: The request uses the system prompt [KEYWORD_EXTRACTION_SYSTEM_PROMPT](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/airflow/dags/llm/prompts/system_prompts.py#L2082), which guides the LLM to output a JSON array of natural, search-like keywords (avoiding blog titles or command prompts). The minimum keyword target count (`VERTICAL_MIN`) is adjusted dynamically based on the total target and the number of verticals.

### 5. Fallback Flow
If no product verticals are defined or no text chunks are extracted, the task falls back to a non-RAG path:
- The entire S3 crawl dataset is serialized as JSON and passed directly to the LLM in a single request.
- The system prompt is adjusted to request keywords across all available verticals.

### 6. Post-Processing & Filtering
- **Aggregation & Case-Insensitive Deduplication**: Raw keywords from all verticals are gathered, deduplicated case-insensitively, and order is preserved.
- **Hard Filtering**: The list is cleaned by [apply_hard_filter](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/backend_src/apps/keywords/services.py#L24). This function applies a series of regexes and checks to strip out junk keywords:
  - Empty or null inputs.
  - Queries shorter than 3 characters.
  - Purely numeric strings.
  - Single character keywords.
  - Strings containing HTTP/HTTPS URLs.
  - Symbol soup (pure punctuation).
  - Null bytes.
- **Database Persistence**: The remaining keywords are saved in bulk into the `KeywordLibrary` table with their source set to `"brand"`.

---

## Downstream Pipeline Lifecycle

Once extracted, keywords pass through the following phases in the DAG:
1. **Branching**: The DAG checks the volume of extracted keywords via `branch_on_cluster_count` to decide the clustering pathway.
2. **Clustering**: Keywords are clustered using mathematical models (HDBSCAN / Agglomerative Clustering) or LLM-based grouping to form cohesive topics.
3. **Vertical Mapping**: Resulting keyword clusters and topics are mapped back to their corresponding product verticals to ensure strategic alignment.

---

## Configuration Reference

The following settings are located in [config.py](file:///Users/gauravbhardwaj/iCloud%20Drive%20%28Archive%29%20-%201/Desktop/Project/gravton-console/airflow/dags/llm/config.py):

- **KEYWORD_EMBED_MODEL** (default: `"google/gemini-embedding-2"`): Used to embed document chunks and queries for RAG context selection.
- **KEYWORD_EXTRACTION_MODEL** (default: `"openai/gpt-4.1-mini"`): The model used to generate vertical keywords.
- **KEYWORD_EXTRACTION_MIN_COUNT** (default: `100`): The target minimum number of keywords to extract.
