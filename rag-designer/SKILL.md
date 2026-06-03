---
name: rag-designer
description: Expert knowledge for RAG (Retrieval-Augmented Generation) systems — chunking strategies, embedding models, vector databases, dense/sparse/hybrid retrieval, reranking, and RAG evaluation. Use when building RAG pipelines, choosing retrieval strategies, optimizing search quality, or diagnosing RAG failures.
triggers:
  - RAG
  - retrieval augmented generation
  - vector database
  - embedding
  - chunking
  - semantic search
  - hybrid search
  - reranking
  - retrieval pipeline
  - knowledge base
---

# RAG Designer Expert

You are now loaded with deep knowledge about Retrieval-Augmented Generation (RAG) systems. Apply these patterns when helping design or optimize RAG pipelines.

## RAG Architecture Overview

```
Documents → Chunking → Embedding → Vector Store
                                       ↓
Query → Query Processing → Retrieval → Reranking → Generation
```

**Why RAG?** Grounds LLM responses in specific, up-to-date knowledge. Reduces hallucination for factual queries. Cheaper than fine-tuning.

## Chunking Strategies

Chunking is the most impactful decision in RAG — poor chunking causes most retrieval failures.

### Fixed-Size Chunking
```python
def chunk_fixed(text: str, chunk_size: int = 512, overlap: int = 50) -> list[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks
```
**Use when**: Simple documents, uniform content, baseline approach.
**Problem**: Cuts sentences mid-thought.

### Sentence/Paragraph Chunking
```python
import nltk
def chunk_by_sentence(text: str, sentences_per_chunk: int = 5) -> list[str]:
    sentences = nltk.sent_tokenize(text)
    return [' '.join(sentences[i:i+sentences_per_chunk])
            for i in range(0, len(sentences), sentences_per_chunk)]
```
**Use when**: Prose, articles, documentation.

### Semantic Chunking
Split on semantic boundaries (topic changes):
```python
# Use embedding similarity to detect topic shifts
# Split where similarity between adjacent sentences drops below threshold
# Libraries: langchain SemanticChunker, llama-index SemanticSplitterNodeParser
```
**Use when**: Long-form documents with distinct topics. Higher quality, slower.

### Document-Aware Chunking (Recommended)
Respect the document structure:
- **Markdown**: chunk by headers (`#`, `##`, `###`)
- **Code**: chunk by function/class definitions
- **PDFs**: chunk by pages or sections
- **HTML**: chunk by semantic sections (`<article>`, `<section>`)

```python
# Markdown chunking example
import re

def chunk_markdown(text: str) -> list[dict]:
    sections = re.split(r'\n#{1,3} ', text)
    return [{'content': s, 'type': 'section'} for s in sections if s.strip()]
```

### Chunk Size Guidelines
| Content Type | Recommended Size |
|---|---|
| Dense technical docs | 256–512 tokens |
| General prose | 512–1024 tokens |
| Code | Function/class level |
| QA pairs | Keep as-is |

**Rule**: Chunk size should match what the LLM needs to answer a question. Too small = no context. Too large = dilutes relevance.

### Parent-Child Chunking (HyDE alternative)
```
Parent chunk (2048 tokens) — stored in DB, returned to LLM
└── Child chunks (256 tokens each) — used for retrieval (more precise)
```
Index child chunks for retrieval, return parent chunk to LLM for richer context.

## Embedding Models

| Model | Dimensions | Use Case |
|---|---|---|
| `text-embedding-3-small` (OpenAI) | 1536 | Balanced cost/quality |
| `text-embedding-3-large` (OpenAI) | 3072 | High quality |
| `voyage-3` (Anthropic) | 1024 | Best for Claude workflows |
| `nomic-embed-text` | 768 | Open source, local |
| `bge-m3` | 1024 | Multilingual |

**Matching rule**: Use the same model to embed queries and documents.

## Retrieval Strategies

### 1. Dense Retrieval (Vector Similarity)
- Cosine similarity between query embedding and chunk embeddings
- Best for: semantic meaning, paraphrases, conceptual matches
- Fails for: exact keywords, names, codes, IDs

### 2. Sparse Retrieval (BM25 / keyword)
```python
from rank_bm25 import BM25Okapi

corpus = [chunk.split() for chunk in chunks]
bm25 = BM25Okapi(corpus)
scores = bm25.get_scores(query.split())
```
- Best for: exact terms, product codes, names, technical identifiers
- Fails for: semantic similarity, synonyms

### 3. Hybrid Retrieval (Recommended Default)
Combine dense + sparse with Reciprocal Rank Fusion (RRF):
```python
def reciprocal_rank_fusion(rankings: list[list[str]], k: int = 60) -> list[str]:
    scores = defaultdict(float)
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] += 1 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

### 4. HyDE (Hypothetical Document Embedding)
```python
# Generate a hypothetical answer, embed it, search with that embedding
hypothetical = llm.generate(f"Write a document that answers: {query}")
embedding = embed(hypothetical)
results = vector_store.search(embedding, top_k=10)
```
Good for: when queries are short/vague and documents are long.

## Reranking

Reranking improves precision — take top-K from retrieval, rerank for relevance:

```python
from cohere import Client

co = Client(api_key)
results = co.rerank(
    query=user_query,
    documents=[chunk.content for chunk in retrieved_chunks],
    model='rerank-english-v3.0',
    top_n=5
)
```

**Pipeline**: Dense+Sparse → top-20 → Reranker → top-5 → LLM

**Reranker options**: Cohere Rerank, BGE-Reranker, FlashRank (local), Cross-Encoder models.

## Vector Databases

| DB | Strength | Use Case |
|---|---|---|
| Pinecone | Managed, scalable | Production, large scale |
| Weaviate | Hybrid search built-in | Default hybrid choice |
| Qdrant | Fast, local or cloud | Performance-sensitive |
| pgvector | PostgreSQL extension | Already using Postgres |
| Chroma | Local, simple | Prototyping |
| LanceDB | Embedded, multimodal | Lightweight |

**Metadata filtering**: Always store metadata with chunks (source, date, section, type) for filtered retrieval:
```python
results = vector_store.query(
    embedding=query_embedding,
    filter={"source": "api-docs", "version": "v2"},
    top_k=10
)
```

## RAG Failure Modes & Fixes

| Failure | Cause | Fix |
|---|---|---|
| Wrong chunks retrieved | Poor chunking or low relevance | Better chunking + hybrid retrieval |
| Right chunks retrieved, wrong answer | LLM ignores context | Stronger grounding instructions |
| Missing context | Chunk too small | Use parent-child chunking |
| Irrelevant context | Top-K too large | Add reranker, reduce top-K |
| Stale answers | Old documents in index | Metadata filtering by date |
| Hallucination | LLM goes beyond context | Constrain with "only use provided context" |

## Query Processing

### Query Rewriting
```python
# Expand vague queries before retrieval
rewritten = llm.generate(f"""
Rewrite this query to be more specific for document retrieval.
Original: {query}
Rewritten:""")
```

### Multi-Query Retrieval
```python
# Generate multiple query variants, retrieve for each, deduplicate
variants = llm.generate(f"Write 3 different versions of: {query}")
all_results = [retrieve(q) for q in variants]
unique_results = deduplicate(all_results)
```

### Query Routing
```python
# Route to different indexes based on query type
if is_code_query(query):
    results = code_index.search(query)
elif is_legal_query(query):
    results = legal_index.search(query)
else:
    results = general_index.search(query)
```

## RAG Evaluation

### RAGAS Metrics
```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall

# faithfulness: answer grounded in context (0-1)
# answer_relevancy: answer relevant to question (0-1)
# context_precision: retrieved context is relevant (0-1)
# context_recall: all relevant context was retrieved (0-1)
```

### Minimum Eval Dataset
- 50–100 question-answer-context triplets
- Include: simple lookups, multi-hop questions, unanswerable questions
- Measure all 4 RAGAS metrics

## Checklist
- [ ] Chunking strategy matches document structure?
- [ ] Chunk size appropriate for content type?
- [ ] Same embedding model used for indexing and querying?
- [ ] Hybrid retrieval (dense + sparse) implemented?
- [ ] Reranker applied before passing to LLM?
- [ ] Metadata stored for filtering?
- [ ] top-K tuned (start at 10–20, rerank to 3–5)?
- [ ] System prompt instructs LLM to only use provided context?
- [ ] Evaluated with RAGAS on 50+ questions?
- [ ] Failure modes analyzed and addressed?
