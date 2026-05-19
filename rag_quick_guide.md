# RAG Quick Guide

---

## Table of Contents

1. [The Core Task](#the-core-task)
2. [Fundamentals (First Principles)](#fundamentals-first-principles)
3. [Day 1 — Build Understanding](#day-1--build-understanding)
4. [Day 2 — Hands-on Build](#day-2--hands-on-build)
5. [Day 3 — Polish & Practice](#day-3--polish--practice)
6. [How to Tackle Any Question](#how-to-tackle-any-question)
7. [Common Questions + Answer Patterns](#common-questions--answer-patterns)
8. [Minimum Viable Path (4–5 hours)](#minimum-viable-path-45-hours)

---

## The Core Task

Build a FastAPI service with two endpoints:

- `POST /ingest` — accept document(s), chunk, embed, store in vector DB
- `POST /search` — accept a query, embed it, retrieve top-k chunks, return them


---

## Fundamentals (First Principles)

### Why RAG exists

LLMs have two limits: knowledge cutoffs, and no access to your private data. Fine-tuning is expensive and slow to update. RAG sidesteps this by fetching relevant text at query time and stuffing it into the prompt. The model becomes a reasoning engine over retrieved context.

**Key insight:** retrieval quality is the bottleneck, not the LLM. Garbage in → garbage out.

### Embeddings

An embedding model maps text to a vector (e.g., 384 or 1536 floats) so semantically similar texts land near each other in vector space. Similarity is measured by **cosine similarity** (angle between vectors).

Why it works: the model was trained to predict which sentences appear in similar contexts, and "similar context" is a strong proxy for "similar meaning."

**Failure modes:** negation ("I love it" vs "I don't love it" can be too close), rare jargon, multi-hop reasoning.

### Vector databases

A vector DB stores `(vector, document, metadata)` tuples and answers: "give me the k vectors most similar to this query." Brute-force is too slow at scale, so they use **Approximate Nearest Neighbor (ANN)** algorithms. **HNSW** (Hierarchical Navigable Small Worlds) is the most common — a graph where similar vectors are connected; you trade a tiny bit of recall for huge speed gains.

### Chunking

LLMs have context windows; embedding models have input limits (~512 tokens for sentence-transformers); retrieval works best when each chunk is one coherent idea.

**The tension:** small chunks → precise retrieval, lost context. Large chunks → more context, noisy retrieval.

**Sweet spot:** 300–800 tokens, 50–100 token overlap. Overlap matters because a key sentence might sit on a boundary; duplicating edges keeps it findable from either side.

**Strategy by document type:**

- Markdown with headings → split by heading
- PDFs with tables → extract tables separately
- Code → split by function/class
- Long unstructured prose → recursive character splitting (try `\n\n`, then `\n`, then `. `, then chars)

### The full RAG pipeline

**Ingestion:** load → parse → chunk → embed each chunk → store `(vector, text, metadata, id)`.

**Query:** embed query → vector DB returns top-k chunks → optionally pass to LLM with prompt like "answer using only this context" → return answer + citations.

### Failure modes

| Failure | Solution |
|---|---|
| Duplicate documents bloating the index | Hash content (SHA-256), use as ID |
| "Lost in the middle" — LLMs attend to start/end | Re-rank so most relevant is first |
| Semantic search missing exact matches | Hybrid search (BM25 + vector) or metadata filtering |
| Stale data | Re-ingestion strategy, version metadata |
| Chunk boundaries cut key info | Overlap, semantic chunking |
| Hallucinations even with context | Prompt for citations, return source chunks for verification |

---

## Day 1 — Build Understanding

### Watch (in this order)

1. **[IBM: What is Retrieval-Augmented Generation? (6 min)](https://www.youtube.com/watch?v=T-D1OfcDW1M)** — executive-summary mental model.
2. **[Umar Jamil: RAG Explained — Embedding, Sentence BERT, Vector DB (HNSW) (~48 min)](https://www.youtube.com/watch?v=rhZgXNdhWDY)** — the must-watch. Explains *why* embeddings work and *how* HNSW searches. If asked "how does a vector DB actually find similar vectors?" — this is the answer.
3. **[Cole Medin: Every RAG Strategy Explained in 13 Minutes](https://www.youtube.com/watch?v=tLMViADvSNE)** — fast tour of variations (hybrid search, re-ranking, query expansion). Gives you senior-sounding vocabulary.

### Read

- **[Weaviate: Chunking Strategies to Improve RAG](https://weaviate.io/blog/chunking-strategies-for-rag)** — canonical chunking reference. Read carefully.
- **[Unstructured: Chunking for RAG Best Practices](https://unstructured.io/blog/chunking-for-rag-best-practices)** — complements Weaviate. Together they cover everything chunking-related.

---

## Day 2 — Hands-on Build

### Read first, then build

- **[DataCamp: ChromaDB Step-by-Step Guide](https://www.datacamp.com/tutorial/chromadb-tutorial-step-by-step-guide)** — cleanest Chroma walkthrough. Collections, embedding functions, querying, metadata filters.
- **[ChromaDB Cookbook](https://cookbook.chromadb.dev/)** — bookmark this. Skim the [Filters](https://cookbook.chromadb.dev/core/filters/), [Collections](https://cookbook.chromadb.dev/core/collections/), and [Embedding Models](https://cookbook.chromadb.dev/embeddings/embedding-models/) sections.

### Build it yourself

Stack: FastAPI + ChromaDB + sentence-transformers (`all-MiniLM-L6-v2`) + LangChain's `RecursiveCharacterTextSplitter`.

Minimum requirements:
- `POST /ingest` accepts text/file, chunks, embeds, stores with metadata (filename, chunk_index, content_hash)
- `POST /search` accepts query + optional k, returns top-k chunks with similarity scores
- Handle duplicates (hash content for IDs)
- Add a `DELETE /documents/{id}` for bonus points

**Then break it on purpose:**
- Upload the same document twice — see what happens
- Query for something not in the corpus — does it return garbage?
- Upload a very short doc and a very long doc — observe chunk counts
- Watch the failure modes so you can talk about them

---

## Day 3 — Polish & Practice

### One advanced read 

- **[Firecrawl: Best Chunking Strategies for RAG 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)** — real benchmark numbers. If you can casually mention "recursive chunking hits ~85-90% recall at 400 tokens," you sound credible.

### Mock yourself

Pull up your code. For every choice — embedding model, chunk size, overlap, vector DB, ID strategy, metadata schema — answer **out loud** in this pattern:

> **State** your choice → **Justify** with reasoning → **Acknowledge** a tradeoff → **Name** when you'd switch.

Time yourself. Aim for 30–60 seconds per answer.

---

## How to Tackle Any Question

### Seven tactics

1. **Narrate as you code.** Talk through choices *before* you make them. This single habit raises perceived seniority more than anything else.

2. **Ask clarifying questions first.** What document types? PDFs with tables? Markdown? Scale — hundreds or millions? Single tenant or multi? Latency targets? These questions earn points and shape every downstream decision. If they say "your call," state your assumption out loud and proceed.

3. **Default "why" answer pattern:** choice → reason → tradeoff → switch trigger.

   *Example for "why this chunk size?":* "I picked 500 tokens with 100 overlap because our docs are technical prose where ideas span paragraphs. Smaller would fragment concepts; larger would dilute retrieval precision. If we were chunking code or structured data I'd use structure-based splitting instead."

4. **For "what if" questions, think in failure modes.** Duplicate uploads → exact dupes (hash), near-dupes (embedding similarity at ingest), versioned dupes (metadata tracking, soft delete). Search returns nothing relevant → similarity threshold, return empty rather than misleading, fall back to keyword search.

5. **Know the scale-up path.** Separate ingestion from query service, async embedding via queue, batch API calls, switch local Chroma → managed vector DB, add caching, monitoring on retrieval latency + quality, auth, tenant isolation. You don't have to build it — just show you've thought about it.

6. **When stuck, think out loud.** Silence is the enemy. "Let me check the docs for the exact API" is fine. "I'm weighing X vs Y because..." is great. Freezing is bad.

7. **Have evaluation ready.** If asked "how would you know if it works?" — retrieval metrics (recall@k, MRR), end-to-end via RAGAS (faithfulness, answer relevance, context precision), and the honest one: have humans rate 50 queries.

---

## Common Questions + Answer Patterns

### "Why this chunking method?"

Tie it to document type. Long prose → recursive with overlap. Structured docs → by section. Code → by function. Mention you'd benchmark on real queries because chunking config affects retrieval quality as much as embedding model choice.

### "What happens if a user uploads duplicate documents?"

Three layers:
- **Exact duplicates:** SHA-256 hash of content as part of the ID; skip insert if exists.
- **Near duplicates:** embed the new doc, check similarity against existing; flag if above threshold.
- **Versioned duplicates:** treat as updates — keep metadata `version`, `created_at`; soft-delete old version or keep both with version filter.

### "Why this vector DB?"

- **Chroma:** easy local dev, persistent, good metadata filtering, perfect for prototypes (this scenario).
- **Pinecone:** managed, scales horizontally — switch when you hit millions of vectors.
- **Weaviate:** hybrid search built-in.
- **pgvector:** if you already have Postgres — single source of truth, transactional.
- **FAISS:** fastest in-memory — when latency is critical and you don't need persistence.

### "Why this embedding model?"

- **`all-MiniLM-L6-v2`:** fast, 384-dim, runs locally, free, great baseline.
- **OpenAI `text-embedding-3-small`:** better quality, API cost, 1536-dim. Switch when quality matters more than cost/latency.
- **BGE-M3 / E5-large:** open source, strong on benchmarks. Switch when you need top quality but self-hosted.

### "How would you handle updates/deletes?"

Document IDs structured as `{doc_hash}_{chunk_index}` so all chunks of a doc share a prefix. Delete by filtering `where={"source_doc_id": doc_id}`. For updates, delete then re-ingest, or version metadata.

### "What if the query has no good matches?"

Set a similarity threshold (e.g., distance > 0.7). Return "no relevant results found" rather than confident-sounding garbage. Optionally fall back to keyword/BM25 search.

### "How would you scale this to production?"

- Separate ingestion service (async, queue-based) from query service
- Batch embedding API calls
- Move from local Chroma → managed vector DB
- Cache common queries
- Monitor retrieval latency and quality metrics (recall@k on a labeled set)
- Auth and per-tenant collection isolation
- Re-ranking with a cross-encoder for top results

### "How would you evaluate it?"

- **Retrieval-level:** recall@k, MRR (mean reciprocal rank) against a labeled set of 50–100 queries.
- **End-to-end:** RAGAS framework — faithfulness, answer relevance, context precision.
- **Human eval:** the unglamorous but essential one — have someone rate 50 queries' answers.

### "Chunk size tradeoff?"

Small chunks → precise retrieval but lose context. Large chunks → more context but noisy retrieval and may exceed LLM context. Start at 512 tokens with 50–100 overlap; tune on real queries.

### "How does the vector DB actually find similar vectors?"

Approximate Nearest Neighbor algorithms. Most use **HNSW** — a multi-layer graph where each layer is sparser than the one below. Search starts at the top, greedily walks to the closest neighbor, drops to the next layer, repeats. Logarithmic complexity vs O(n) for brute force. Trade a few % recall for huge speed gains.

---

## Minimum Viable Path (4–5 hours)

If short on time:

1. [IBM video](https://www.youtube.com/watch?v=T-D1OfcDW1M) — 6 min
2. [Umar Jamil RAG video](https://www.youtube.com/watch?v=rhZgXNdhWDY) — 48 min
3. [Weaviate chunking blog](https://weaviate.io/blog/chunking-strategies-for-rag) — 20 min
4. [DataCamp ChromaDB guide](https://www.datacamp.com/tutorial/chromadb-tutorial-step-by-step-guide) — 30 min + try in a notebook
5. **Build the FastAPI + Chroma service yourself** — 2–3 hours

That alone covers ~90% of what they'll test.

---

## Quick Reference Links

| Resource | Type | Cost |
|---|---|---|
| [IBM: What is RAG?](https://www.youtube.com/watch?v=T-D1OfcDW1M) | Video, 6 min | Free |
| [Umar Jamil: RAG + HNSW Explained](https://www.youtube.com/watch?v=rhZgXNdhWDY) | Video, 48 min | Free |
| [Cole Medin: Every RAG Strategy in 13 Min](https://www.youtube.com/watch?v=tLMViADvSNE) | Video, 13 min | Free |
| [Weaviate: Chunking Strategies](https://weaviate.io/blog/chunking-strategies-for-rag) | Blog | Free |
| [Unstructured: Chunking Best Practices](https://unstructured.io/blog/chunking-for-rag-best-practices) | Blog | Free |
| [Firecrawl: Best Chunking Strategies 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag) | Blog | Free |
| [DataCamp: ChromaDB Step-by-Step](https://www.datacamp.com/tutorial/chromadb-tutorial-step-by-step-guide) | Tutorial | Free (article in browser) |
| [ChromaDB Cookbook](https://cookbook.chromadb.dev/) | Docs | Free |
| [ChromaDB Cookbook: Filters](https://cookbook.chromadb.dev/core/filters/) | Docs | Free |
| [ChromaDB Cookbook: Collections](https://cookbook.chromadb.dev/core/collections/) | Docs | Free |
| [ChromaDB Cookbook: Embedding Models](https://cookbook.chromadb.dev/embeddings/embedding-models/) | Docs | Free |

---
