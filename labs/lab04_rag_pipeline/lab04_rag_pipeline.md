

# Lab 04 — Building a RAG Query Engine

**Course:** A Practical Guide to Building a GenAI Application
**Duration:** 2 hours

**Objectives:**

* Implement a complete Retrieval-Augmented Generation (RAG) query engine
* Combine embeddings + vector search + re-ranking + prompt building
* Wire a simple LLM call (mocked or real)
* Evaluate retrieval + answer quality

---

## 1 — Overview

A RAG engine typically performs these steps:

1. User query comes in
2. Optional query rewriting / expansion
3. Query embedding
4. Vector database retrieval
5. Optional re-ranking (e.g., BM25 or cross-encoder)
6. Context assembly into a prompt
7. LLM generates an answer using the context

In this lab we'll implement a minimal but practical RAG pipeline using:

* SentenceTransformers embeddings
* FAISS as the vector store
* A TF-IDF-based re-ranker (scikit-learn)
* Prompt builder and optional OpenAI call

---

## 2 — Install Dependencies

Run this in a notebook cell:

```bash
!pip install sentence-transformers faiss-cpu scikit-learn openai python-dotenv
```

---

## 3 — Imports & Model Load

```python
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
from typing import List, Tuple
import os
from dotenv import load_dotenv
load_dotenv()

EMBED_MODEL_NAME = 'all-MiniLM-L6-v2'
embed_model = SentenceTransformer(EMBED_MODEL_NAME)
print('Loaded embedding model:', EMBED_MODEL_NAME)
```

---

## 4 — Prepare Documents (Demo Corpus)

```python
documents = [
    {'id':'doc1','text':'The Central Bank of Nigeria (CBN) sets monetary policy and regulates banks.'},
    {'id':'doc2','text':'Python is widely used for data science and machine learning tasks.'},
    {'id':'doc3','text':'Weaviate and Pinecone are managed vector databases for embeddings.'},
    {'id':'doc4','text':'Nigeria has diverse ecosystems: Lagos is a commercial hub, while Abuja is the capital.'},
    {'id':'doc5','text':'The Nigerian Stock Exchange lists equities for trading and sets market rules.'}
]

texts = [d['text'] for d in documents]
texts[:3]
```

---

## 5 — Chunking (Optional) and Embeddings

For brevity we treat each document as one chunk. Generate embeddings for each chunk and build a FAISS index.

```python
# Create embeddings
embs = embed_model.encode(texts, convert_to_numpy=True)
print('Embeddings shape:', embs.shape)

# Build FAISS index
d = embs.shape[1]
index = faiss.IndexFlatIP(d)  # use inner product for cosine if vectors are normalized
# Normalize embeddings to unit length
faiss.normalize_L2(embs)
index.add(embs)
print('FAISS index size:', index.ntotal)
```

---

## 6 — Simple Retrieval Function

A function to take a query, embed it, and return top-k candidate chunks from FAISS.

```python
def retrieve(query: str, k: int = 3) -> List[Tuple[float, dict]]:
    q_emb = embed_model.encode([query], convert_to_numpy=True)
    faiss.normalize_L2(q_emb)
    D, I = index.search(q_emb, k)
    results = []
    for score, idx in zip(D[0], I[0]):
        results.append((float(score), documents[idx]))
    return results

# test
print(retrieve('What does the Central Bank of Nigeria do?', k=3))
```

---

## 7 — Re-ranking with TF-IDF

FAISS gives semantic neighbors. A lightweight re-ranker using TF-IDF can improve precision for short queries.

```python
tfidf = TfidfVectorizer()
# Fit on all texts (in production, you'd fit on the corpus once)
tfidf.fit(texts)

from sklearn.metrics.pairwise import cosine_similarity

def rerank_tfidf(query: str, candidates: List[dict]) -> List[Tuple[float, dict]]:
    cand_texts = [c['text'] for c in candidates]
    if not cand_texts:
        return []
    cand_vecs = tfidf.transform(cand_texts)
    q_vec = tfidf.transform([query])
    sims = cosine_similarity(q_vec, cand_vecs)[0]
    scored = sorted(list(zip(sims, candidates)), key=lambda x: x[0], reverse=True)
    return [(float(s), c) for s,c in scored]

# demo
cands = [d for _,d in retrieve('What does the Central Bank do?', k=4)]
print('Candidates before rerank:', [c['id'] for c in cands])
print('Reranked:', rerank_tfidf('What does the Central Bank do?', cands))
```

---

## 8 — Prompt Builder

Assemble retrieved contexts into a prompt. Keep token budget in mind in real systems; here we concatenate.

```python
def build_prompt(query: str, contexts: List[dict]) -> str:
    ctx_block = "\n\n---\n\n".join([f"Source: {c['id']}\n{c['text']}" for c in contexts])
    prompt = f"You are a helpful assistant. Use the context to answer the question.\n\nContext:\n{ctx_block}\n\nQuestion: {query}\n\nAnswer concisely and cite sources by doc id."
    return prompt

# demo
candidates = [c for _,c in retrieve('What does the Central Bank of Nigeria do?', k=3)]
prompt = build_prompt('What does the Central Bank of Nigeria do?', candidates)
print(prompt)
```

---

## 9 — LLM Call (Mock or Real)

You can hook this prompt to an LLM. For cost safety the notebook shows a mocked LLM response function. If you want to call OpenAI, uncomment and provide `OPENAI_API_KEY` in your environment.

```python
def call_llm_mock(prompt: str) -> str:
    # Very simple mock that echoes prompt summary
    return 'MockAnswer: Based on context, the Central Bank sets monetary policy. [Sources: doc1]'

# Optional real call (commented out)
# import openai
# openai.api_key = os.getenv('OPENAI_API_KEY')
# def call_llm_openai(prompt: str):
#     resp = openai.ChatCompletion.create(model='gpt-4o-mini', messages=[{'role':'user','content':prompt}], max_tokens=300)
#     return resp.choices[0].message['content']

# demo
print(call_llm_mock(prompt))
```

---

## 10 — End-to-end RAG function

Combine retrieval → rerank → prompt → LLM call into a single function.

```python
def rag_answer(query: str, k:int=5, rerank_top_n:int=5, use_rerank:bool=True) -> dict:
    # 1. retrieve
    retrieved = retrieve(query, k=k)
    candidates = [d for _,d in retrieved]
    # 2. rerank
    if use_rerank:
        reranked = rerank_tfidf(query, candidates)
        contexts = [c for _,c in reranked[:rerank_top_n]]
    else:
        contexts = candidates[:rerank_top_n]
    # 3. build prompt
    prompt = build_prompt(query, contexts)
    # 4. call LLM (mocked)
    answer = call_llm_mock(prompt)
    return {'query':query,'answer':answer,'contexts':[c['id'] for c in contexts],'prompt':prompt}

# test
print(rag_answer('What is the role of the Central Bank of Nigeria?', k=5, rerank_top_n=3))
```

---

## 11 — Exercise 1: Try different queries

Run `rag_answer` with these queries and inspect contexts and answers:

* `What do banks in Nigeria regulate?`
* `Tell me about Python in AI.`
* `Which cities are major economic hubs in Nigeria?`

Write your observations: which contexts were returned and were they relevant?

```python
# Try the queries
queries = [
    'What do banks in Nigeria regulate?',
    'Tell me about Python in AI.',
    'Which cities are major economic hubs in Nigeria?'
]
for q in queries:
    res = rag_answer(q, k=4, rerank_top_n=3)
    print('\nQUERY:', q)
    print('ANSWER:', res['answer'])
    print('CONTEXTS:', res['contexts'])
```

**Notes for students:** Capture which doc ids appear and whether they support the answer. Think about false positives and missing docs.

---

## 12 — Exercise 2: Improve Reranking

Replace the TF-IDF reranker with a semantic cross-encoder (if available) or implement a heuristic re-scoring combining FAISS score and TF-IDF score:

`score = alpha * faiss_score + (1-alpha) * tfidf_score`.

Implement a function `hybrid_rerank(query, faiss_results, alpha=0.6)` and test it.

```python
def hybrid_rerank(query: str, faiss_results: List[Tuple[float, dict]], alpha: float = 0.6):
    # faiss_results: list of (faiss_score, doc)
    cand_docs = [d for _,d in faiss_results]
    # TF-IDF sims
    cand_texts = [d['text'] for d in cand_docs]
    cand_vecs = tfidf.transform(cand_texts)
    q_vec = tfidf.transform([query])
    tfidf_sims = cosine_similarity(q_vec, cand_vecs)[0]
    hybrid = []
    for (faiss_score, doc), tfidf_score in zip(faiss_results, tfidf_sims):
        combined = alpha * faiss_score + (1-alpha) * float(tfidf_score)
        hybrid.append((combined, doc))
    hybrid.sort(key=lambda x: x[0], reverse=True)
    return hybrid

# test hybrid rerank
fr = retrieve('What does the Central Bank do?', k=4)
print('FAISS results:', [(s,d['id']) for s,d in fr])
print('Hybrid:', [(round(s,3),d['id']) for s,d in hybrid_rerank('What does the Central Bank do?', fr)])
```

**Instructor note:** For production, prefer a learned cross-encoder re-ranker (e.g., small transformer fine-tuned for pairwise relevance). Hybrid heuristics are useful when compute is limited.

---

## 13 — Evaluation: Precision@k

Create a tiny ground-truth mapping of queries → expected doc ids and compute precision@k for your `rag_answer` with `k=3`.

```python
# ground truth (toy)
ground_truth = {
    'What does the Central Bank of Nigeria do?': ['doc1'],
    'Tell me about Python in AI': ['doc2'],
    'Which are major economic hubs in Nigeria?': ['doc4']
}

def precision_at_k(query, k=3):
    res = rag_answer(query, k=k, rerank_top_n=k)
    preds = res['contexts']
    gt = ground_truth.get(query, [])
    if not gt:
        return None
    correct = sum(1 for p in preds if p in gt)
    return correct / k

for q in ground_truth:
    print(q, 'precision@3 =', precision_at_k(q, k=3))
```

---

## 14 — Instructor Key (Hints & Answers)

* `hybrid_rerank` should combine FAISS similarity (inner product) and TF-IDF cosine in a weighted manner.
* Precision@k in this tiny corpus will be imperfect but should be >0 for clear queries.
* For production, prefer a learned cross-encoder re-ranker or a small transformer fine-tuned for semantic reranking.

**Next steps:** integrate this RAG engine into FastAPI (Lab 05), add caching (Lab 11), and replace the mock LLM with real calls and streaming.

---

## Optional Extensions (advanced students)

* Replace FAISS local index with Pinecone/Weaviate/Chroma and add metadata filtering.
* Implement query rewrite/expansion (e.g., add synonyms or context).
* Add chunk-level source highlighting in the frontend.
* Add a cross-encoder re-ranker (e.g., `sentence-transformers` cross-encoder models).
* Measure latency and token usage when calling the real LLM.

---

## Troubleshooting & Tips

* If FAISS import fails on some platforms, install the wheel that matches your Python version or use `pip install faiss-cpu`.
* Keep your API keys in `.env` and never commit them.
* Normalize embeddings before using inner-product index for cosine-similarity-like behavior.
* When building prompts, always bound the total context size by tokens to avoid truncation.

---

