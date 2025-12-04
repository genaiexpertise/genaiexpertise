
---

# **Lab 12 — Capstone Project: End-to-End GenAI Application**

### **🚀 Build & Deploy a Production-Ready RAG Application From Scratch**

This capstone guides you to build **a complete, production-grade Generative AI application**, combining all concepts from Labs 1–11:

| Layer             | Component                                   |
| ----------------- | ------------------------------------------- |
| Data Layer        | Document ingestion, preprocessing, chunking |
| Embeddings        | Batch workers, async processing             |
| Retrieval         | Vector search, hybrid ranking               |
| LLM               | Prompting, templates, safety                |
| Application Layer | FastAPI server, task queue                  |
| Orchestration     | Worker pipelines, caching                   |
| Deployment        | Docker, orchestration, monitoring           |
| Scaling           | Autoscaling & load balancing                |
| Observability     | Logs, metrics, tracing                      |

You will deliver a **fully working system**, published via:

✔ GitHub Repository
✔ Docker Compose / K8s deployment
✔ API documentation
✔ README + architecture diagram
✔ Demo notebook + sample queries

---

# ------------------------------------------

# **🎯 Capstone Goals**

# ------------------------------------------

By completing this project, you will:

* Build a complete production-grade **Retrieval-Augmented Generation (RAG)** pipeline
* Deploy a scalable **FastAPI + Worker + Weaviate** system
* Serve a working GenAI inference API
* Implement observability, security, and scaling components
* Produce a professional portfolio project suitable for job interviews

---

# ------------------------------------------

# **🧱 Project Overview**

# ------------------------------------------

You will build:

```
GenAI Capstone System
│
├── 1) Document Ingestion Pipeline
├── 2) Embedding & Storage Pipeline
├── 3) RAG Query Engine
├── 4) FastAPI Application Layer
├── 5) Caching & Async Workers
├── 6) Deployment (Docker Compose or Kubernetes)
└── 7) Monitoring & Metrics
```

Your final deliverables:

* `notebooks/capstone_demo.ipynb`
* `backend/fastapi_app/`
* `backend/workers/`
* `vectorstore/weaviate/`
* `docker-compose.yaml` (or K8s manifests)
* Architecture diagram (PNG)
* README.md (Professional Documentation)

---

# ------------------------------------------

# **📁 Project Folder Structure**

# ------------------------------------------

```
capstone/
├── backend/
│   ├── fastapi_app/
│   │   ├── main.py
│   │   ├── routers/
│   │   ├── services/
│   │   ├── models/
│   │   └── utils/
│   ├── workers/
│   │   ├── celery_app.py
│   │   ├── tasks.py
│   │   └── embeddings.py
│   └── shared/
│       └── config.py
│
├── data/
│   └── raw_docs/
│
├── vectorstore/
│   └── weaviate/
│
├── notebooks/
│   └── capstone_demo.ipynb
│
├── docker-compose.yaml
└── README.md
```

---

# ------------------------------------------

# **1️⃣ Step 1 — Document Ingestion Notebook**

# ------------------------------------------

### **Notebook: `capstone_demo.ipynb`**

```python
import os
from langchain.text_splitter import RecursiveCharacterTextSplitter

docs_dir = "../data/raw_docs/"

documents = []
for file in os.listdir(docs_dir):
    if file.endswith(".pdf") or file.endswith(".txt"):
        with open(os.path.join(docs_dir, file), "rb") as f:
            text = f.read().decode("utf-8", errors="ignore")
            documents.append(text)

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=200,
)

chunks = splitter.split_text("\n".join(documents))
len(chunks)
```

---

# ------------------------------------------

# **2️⃣ Step 2 — Embedding Pipeline (Workers)**

# ------------------------------------------

### `workers/tasks.py`

```python
from celery import Celery
from embeddings import embed_text
from vectorstore import store_vector

app = Celery("tasks", broker="redis://redis:6379/0")

@app.task
def embed_and_store(chunk, chunk_id):
    vector = embed_text(chunk)
    store_vector(chunk_id, vector)
    return {"id": chunk_id, "status": "stored"}
```

### Running workers

```bash
celery -A tasks worker --concurrency=4
```

---

# ------------------------------------------

# **3️⃣ Step 3 — Vector DB Storage (Weaviate)**

# ------------------------------------------

### Example ingestion call

```python
from workers.tasks import embed_and_store

for i, chunk in enumerate(chunks):
    embed_and_store.delay(chunk, f"chunk-{i}")
```

---

# ------------------------------------------

# **4️⃣ Step 4 — Build the RAG Engine**

# ------------------------------------------

### `services/rag.py`

```python
import weaviate
from llm import call_llm

client = weaviate.Client("http://vector-db:8080")

def rag_query(question):
    query_vector = embed(question)

    results = client.query.get("Document", ["text"]) \
        .with_near_vector({"vector": query_vector}) \
        .with_limit(5) \
        .do()

    context = "\n".join([r["text"] for r in results["data"]["Get"]["Document"]])

    prompt = f"""
    Use ONLY the context below to answer the question.

    Context:
    {context}

    Question: {question}
    """

    return call_llm(prompt)
```

---

# ------------------------------------------

# **5️⃣ Step 5 — Expose the RAG API (FastAPI)**

# ------------------------------------------

### `fastapi_app/main.py`

```python
from fastapi import FastAPI
from services.rag import rag_query

app = FastAPI()

@app.post("/query")
def query(payload: dict):
    answer = rag_query(payload["question"])
    return {"answer": answer}
```

Run server:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

---

# ------------------------------------------

# **6️⃣ Step 6 — Add Caching (Critical)**

# ------------------------------------------

### `utils/cache.py`

```python
from cachetools import TTLCache

cache = TTLCache(maxsize=1000, ttl=3600)

def cached_query(question, fn):
    if question in cache:
        return cache[question]

    result = fn(question)
    cache[question] = result
    return result
```

Integrate:

```python
@app.post("/query")
def query(payload):
    return cached_query(payload["question"], rag_query)
```

---

# ------------------------------------------

# **7️⃣ Step 7 — Deploy With Docker Compose**

# ------------------------------------------

### **docker-compose.yaml**

```yaml
services:
  api:
    build: ./backend/fastapi_app
    ports:
      - "8000:8000"
    depends_on:
      - redis
      - vector-db

  worker:
    build: ./backend/workers
    depends_on:
      - redis
      - vector-db
    deploy:
      replicas: 4

  redis:
    image: redis:7

  vector-db:
    image: semitechnologies/weaviate
    ports:
      - "8080:8080"
    environment:
      QUERY_DEFAULTS_LIMIT: 20
      ENABLE_MODULES: text2vec-transformers
```

---

# ------------------------------------------

# **8️⃣ Step 8 — Monitoring**

# ------------------------------------------

Add `/metrics`:

```python
from starlette_exporter import PrometheusMiddleware, handle_metrics

app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", handle_metrics)
```

Deploy **Prometheus + Grafana**.

---

# ------------------------------------------

# **9️⃣ Step 9 — Scaling**

# ------------------------------------------

Scale workers:

```bash
docker compose up --scale worker=10
```

Scale FastAPI:

```bash
docker compose up --scale api=5
```

---

# ------------------------------------------

# **🔟 Step 10 — Capstone Deliverables**

# ------------------------------------------

You must submit:

### ✔ **Working code repository**

### ✔ **Working Docker deployment**

### ✔ **API documentation (Swagger)**

### ✔ **Architecture diagram**

### ✔ **Demo notebook**

### ✔ **5 example RAG queries**

### ✔ **README + Setup Instructions**

---

# ------------------------------------------

# **📝 Final Capstone Exercises**

# ------------------------------------------

### **Exercise 1 — Add re-ranking**

Use Cohere or Jina reranker to improve answer quality.

### **Exercise 2 — Add embeddings batching**

Improve throughput.

### **Exercise 3 — Add OpenAI O1 or GPT-4o mini**

Include as an optional model.

### **Exercise 4 — Add metadata filters**

Filter retrieval by:

* date
* source
* topic

### **Exercise 5 — Add auth tokens to API**

Secure your app.

### **Exercise 6 — Deploy to cloud**

Choose:

* Fly.io
* AWS ECS
* GCP Cloud Run
* Modal

---

# ------------------------------------------

# 🎉 **Congratulations — You completed the entire GenAI Engineering Capstone!**

# ------------------------------------------


