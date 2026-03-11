# Transfer Plan: Pinecone Vector RAG → Neo4j Graph RAG

## 1. Current Architecture Overview

The project is a **Competitor Intelligence Tool** that compares two companies by crawling their websites, embedding the content, storing it in a vector database, and generating LLM-powered comparisons.

### Current Pipeline

```
User Query
  │
  ▼
keyword_extractor.py ── Groq (Llama-3) extracts company names
  │
  ▼
crawler.py ── Firecrawl scrapes company websites (async/parallel)
  │
  ▼
embeddings.py ── Splits markdown → token-based chunks → SentenceTransformer embeddings
  │
  ▼
pinecone_index.py ── Upserts vectors to Pinecone (namespaced by company)
  │
  ▼
main.py / app.py ── Queries Pinecone → feeds context to Groq → comparison response
```

### Current Files & Responsibilities

| File | Role | What Changes |
|---|---|---|
| `keyword_extractor.py` | Extracts company names via Groq | **No change** |
| `crawler.py` | Scrapes websites via Firecrawl | **No change** |
| `embeddings.py` | Chunking + SentenceTransformer embeddings | **Modify** — add entity/relationship extraction |
| `pinecone_index.py` | Pinecone CRUD (create, upsert, query, delete) | **Replace** → `neo4j_graph.py` |
| `main.py` | CLI pipeline orchestrator | **Modify** — swap Pinecone calls for Neo4j |
| `app.py` | Streamlit UI pipeline | **Modify** — swap Pinecone calls for Neo4j |
| `requirements.txt` | Dependencies | **Modify** — remove `pinecone`, add `neo4j`, `langchain-neo4j` |

---

## 2. Why Graph RAG?

The current vector-only RAG retrieves chunks based on **semantic similarity** alone. Graph RAG adds:

| Capability | Vector RAG (Current) | Graph RAG (Target) |
|---|---|---|
| Semantic search | ✅ Cosine similarity | ✅ Vector index in Neo4j |
| Relationship traversal | ❌ | ✅ `Company ─HAS_FEATURE→ Feature` |
| Multi-hop reasoning | ❌ | ✅ Traverse pricing → plan → feature chains |
| Structured metadata | Flat key-value on vectors | First-class nodes & relationships |
| Cross-company linking | Separate namespaces | Shared graph, queryable across entities |

---

## 3. Target Architecture

### New Pipeline

```
User Query
  │
  ▼
keyword_extractor.py ── (unchanged)
  │
  ▼
crawler.py ── (unchanged)
  │
  ▼
embeddings.py ── Chunking + embeddings (unchanged)
  │
  ▼
entity_extractor.py [NEW] ── LLM extracts entities & relationships from chunks
  │
  ▼
neo4j_graph.py [NEW] ── Stores nodes, relationships, and vector embeddings in Neo4j
  │
  ▼
graph_retriever.py [NEW] ── Hybrid retrieval: vector search + graph traversal
  │
  ▼
main.py / app.py ── Feeds enriched context to Groq → comparison response
```

### Neo4j Graph Schema

```
(:Company {name, source_url})
    ─[:HAS_CHUNK]→ (:Chunk {text, embedding, section, category})
    ─[:HAS_FEATURE]→ (:Feature {name, description})
    ─[:HAS_PLAN]→ (:Plan {name, price, billing_cycle})
        ─[:INCLUDES_FEATURE]→ (:Feature)
    ─[:HAS_SECURITY]→ (:SecurityCert {name, description})

(:Chunk)
    ─[:MENTIONS_ENTITY]→ (:Feature | :Plan | :SecurityCert)
    ─[:NEXT_CHUNK]→ (:Chunk)  // preserves document ordering
```

---

## 4. New & Modified Files

### 4.1 `entity_extractor.py` — **[NEW]**

Extracts structured entities and relationships from text chunks using Groq.

```python
# Responsibilities:
# - Takes a chunk of text + company name
# - Calls Groq with a structured output prompt to extract:
#     - Features (name, description)
#     - Pricing plans (name, price, billing_cycle)
#     - Security certifications (name, description)
#     - Relationships between them
# - Returns a list of structured entities + relationships

# Key function:
def extract_entities(chunk_text: str, company: str) -> dict:
    """
    Returns:
    {
        "entities": [
            {"type": "Feature", "name": "...", "description": "..."},
            {"type": "Plan", "name": "Basic", "price": "$20/mo", ...},
        ],
        "relationships": [
            {"from": "Basic Plan", "to": "Global Payments", "type": "INCLUDES_FEATURE"},
        ]
    }
    """
```

### 4.2 `neo4j_graph.py` — **[NEW]** (replaces `pinecone_index.py`)

All Neo4j CRUD operations, including vector index management.

```python
# Responsibilities:
# - Connect to Neo4j instance (Aura or local Docker)
# - Create vector index on Chunk.embedding property
# - Ingest company nodes, chunk nodes with embeddings, entity nodes, and relationships
# - Provide vector similarity search via Neo4j's built-in vector index
# - Provide Cypher-based graph traversal queries
# - Cleanup: delete all nodes for a company

# Key functions:
def connect(uri, user, password) -> Driver
def ensure_vector_index(driver, index_name, dimension=384)
def ingest_company_graph(driver, company, chunks, embeddings, entities)
def delete_company(driver, company)
```

### 4.3 `graph_retriever.py` — **[NEW]**

Hybrid retrieval combining vector search with graph context enrichment.

```python
# Responsibilities:
# - Vector search: Query Neo4j vector index for top-k similar chunks
# - Graph enrichment: For each retrieved chunk, traverse the graph to pull in:
#     - The parent Company node
#     - Connected Feature / Plan / SecurityCert nodes
#     - Neighboring chunks (via NEXT_CHUNK) for broader context
# - Return enriched context strings for the LLM

# Key functions:
def hybrid_retrieve(driver, query_embedding, company: str, top_k=5) -> list[dict]:
    """
    1. Vector search → top-k Chunk nodes
    2. For each chunk, run Cypher to expand:
       MATCH (c:Chunk)-[:MENTIONS_ENTITY]->(e)
       MATCH (c)<-[:HAS_CHUNK]-(company:Company)
       RETURN c, e, company
    3. Return list of {chunk_text, related_entities, company_name, source_url}
    """
```

### 4.4 `embeddings.py` — **[MODIFY]**

Minimal changes — the chunking and embedding logic stays the same. Add a utility to attach chunk ordering metadata.

```diff
 def build_chunks(markdown: str, company: str, source_url: str) -> list[dict]:
     sections = split_markdown_sections(markdown)
     final_chunks = []
+    chunk_index = 0

     for section in sections:
         ...
         for chunk in token_chunks:
             if not is_valid_chunk(chunk):
                 continue
             final_chunks.append({
                 "text": chunk,
                 "metadata": {
                     "company": company,
                     "category": detect_category(chunk),
                     "section": section_title,
-                    "source_url": source_url
+                    "source_url": source_url,
+                    "chunk_index": chunk_index
                 }
             })
+            chunk_index += 1

     return final_chunks
```

### 4.5 `main.py` — **[MODIFY]**

Replace Pinecone imports and calls with Neo4j equivalents.

```diff
-from pinecone_index import create_index, upsert_chunks, query_index, delete_namespace
+from neo4j_graph import connect, ensure_vector_index, ingest_company_graph, delete_company
+from graph_retriever import hybrid_retrieve
+from entity_extractor import extract_entities

 async def run_comparison(user_prompt: str) -> str:
-    index = create_index()
+    driver = connect(os.getenv("NEO4J_URI"), os.getenv("NEO4J_USER"), os.getenv("NEO4J_PASSWORD"))
+    ensure_vector_index(driver, "chunk_embeddings", dimension=384)

     for keyword, data in crawl_results.items():
         chunks = build_chunks(content, company_name, source_url)
         embeddings, _ = embed_chunks(chunks)
-        upsert_chunks(index, chunks, embeddings, namespace=company_name)
+        entities = [extract_entities(c["text"], company_name) for c in chunks]
+        ingest_company_graph(driver, company_name, chunks, embeddings, entities)

-    context_a = query_index(index, query_embedding, namespace=company_a_name, top_k=5)
-    context_b = query_index(index, query_embedding, namespace=company_b_name, top_k=5)
+    context_a = hybrid_retrieve(driver, query_embedding, company_a_name, top_k=5)
+    context_b = hybrid_retrieve(driver, query_embedding, company_b_name, top_k=5)

-    delete_namespace(index, company_a_name)
-    delete_namespace(index, company_b_name)
+    delete_company(driver, company_a_name)
+    delete_company(driver, company_b_name)
+    driver.close()
```

### 4.6 `app.py` — **[MODIFY]**

Same changes as `main.py` — swap Pinecone calls for Neo4j + graph retriever. Additionally:

```diff
-if not os.getenv("PINECONE_API"):
-    st.error("PINECONE_API not found in environment variables.")
+if not os.getenv("NEO4J_URI"):
+    st.error("NEO4J_URI not found in environment variables.")
+if not os.getenv("NEO4J_PASSWORD"):
+    st.error("NEO4J_PASSWORD not found in environment variables.")
```

### 4.7 `requirements.txt` — **[MODIFY]**

```diff
-pinecone
+neo4j
+langchain-neo4j   # optional: for LangChain GraphRAG integrations
```

### 4.8 `pinecone_index.py` — **[DELETE]**

No longer needed — fully replaced by `neo4j_graph.py` and `graph_retriever.py`.

---

## 5. Environment Setup

### Neo4j Options

| Option | Setup | Best For |
|---|---|---|
| **Neo4j Aura Free** | Cloud-hosted, no install | Quick start, prototyping |
| **Docker (local)** | `docker run neo4j` | Development, full control |
| **Neo4j Desktop** | GUI app | Visual exploration |

### Required Environment Variables

```env
# Remove
PINECONE_API=...

# Add
NEO4J_URI=bolt://localhost:7687       # or neo4j+s://xxxx.databases.neo4j.io
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password
```

### Docker Quick Start

```bash
docker run \
  --name neo4j-graphrag \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your_password \
  -e NEO4J_PLUGINS='["apoc"]' \
  -v neo4j_data:/data \
  neo4j:5
```

Access the Neo4j Browser UI at `http://localhost:7474`.

---

## 6. Migration Steps (Execution Order)

### Phase 1: Foundation
- [ ] Set up Neo4j instance (Docker or Aura)
- [ ] Add `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` to `.env`
- [ ] Update `requirements.txt` — add `neo4j`, remove `pinecone`
- [ ] Install: `pip install neo4j`

### Phase 2: New Modules
- [ ] Create `neo4j_graph.py` — connection, schema creation, vector index, CRUD
- [ ] Create `entity_extractor.py` — LLM-based entity + relationship extraction
- [ ] Create `graph_retriever.py` — hybrid vector + graph retrieval

### Phase 3: Integration
- [ ] Modify `embeddings.py` — add `chunk_index` to metadata
- [ ] Modify `main.py` — replace Pinecone pipeline with Neo4j graph pipeline
- [ ] Modify `app.py` — same swap + update env var checks + Streamlit status messages

### Phase 4: Cleanup
- [ ] Delete `pinecone_index.py`
- [ ] Remove `PINECONE_API` from `.env`
- [ ] Update `README.md` with new architecture, setup instructions, and graph schema

### Phase 5: Verification
- [ ] Run `entity_extractor.py` standalone on sample text — verify structured output
- [ ] Run `neo4j_graph.py` standalone — verify nodes and relationships in Neo4j Browser
- [ ] Run full CLI pipeline via `main.py` — verify end-to-end comparison
- [ ] Run Streamlit app via `app.py` — verify UI works with Neo4j backend
- [ ] Verify cleanup — after pipeline run, no orphaned nodes remain in Neo4j

---

## 7. Key Technical Decisions

### 7.1 Vector Search in Neo4j

Neo4j 5+ has **native vector index** support. No need for a separate vector DB:

```cypher
-- Create index
CREATE VECTOR INDEX chunk_embeddings IF NOT EXISTS
FOR (c:Chunk) ON (c.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 384,
  `vector.similarity_function`: 'cosine'
}}

-- Query
CALL db.index.vector.queryNodes('chunk_embeddings', 5, $queryVector)
YIELD node, score
RETURN node.text, score
```

### 7.2 Entity Extraction Strategy

Use Groq with structured JSON output (same pattern as `keyword_extractor.py`) to extract entities from each chunk. This keeps the extraction fast and cost-effective.

**Trade-off**: LLM extraction may hallucinate entities. Mitigate by:
- Using low temperature (0.2)
- Validating extracted entities against the chunk text
- Merging duplicate entities by normalized name

### 7.3 Hybrid Retrieval

The retriever does a **two-stage** retrieval:

1. **Vector search** — find semantically similar chunks (same as current)
2. **Graph expansion** — for each chunk, traverse 1–2 hops to gather related entities

This enriches the LLM context with structured information that pure vector search misses.

### 7.4 Why Not LangChain's GraphRAG?

LangChain has `Neo4jGraph` and `GraphCypherQAChain`, but using them adds heavy abstractions. This project benefits from **direct Neo4j driver + custom Cypher** for:
- Fine-grained control over the graph schema
- Custom hybrid retrieval logic
- Minimal dependency footprint

If you later want LangChain integration, `langchain-neo4j` is included in requirements as an optional dependency.

---

## 8. Risk & Rollback

| Risk | Mitigation |
|---|---|
| Neo4j vector index performance | Benchmark against Pinecone on same dataset; fallback to Pinecone if degraded |
| Entity extraction quality | Log extracted entities; add a review/validation step if needed |
| Neo4j Aura free tier limits | Use local Docker for development; Aura for production |
| Breaking existing CLI/UI | Keep `pinecone_index.py` in repo until Neo4j pipeline is fully verified |

**Rollback**: The `pinecone_index.py` file is only deleted in Phase 4 (after verification). If issues arise, revert imports in `main.py` and `app.py` to restore the Pinecone pipeline.

---

## 9. Summary of File Changes

| Action | File | Description |
|---|---|---|
| **NEW** | `entity_extractor.py` | LLM-based entity/relationship extraction |
| **NEW** | `neo4j_graph.py` | Neo4j connection, schema, vector index, CRUD |
| **NEW** | `graph_retriever.py` | Hybrid vector + graph traversal retrieval |
| **MODIFY** | `embeddings.py` | Add `chunk_index` to metadata |
| **MODIFY** | `main.py` | Swap Pinecone → Neo4j pipeline |
| **MODIFY** | `app.py` | Swap Pinecone → Neo4j pipeline + env checks |
| **MODIFY** | `requirements.txt` | Remove `pinecone`, add `neo4j` |
| **DELETE** | `pinecone_index.py` | Replaced by `neo4j_graph.py` |
