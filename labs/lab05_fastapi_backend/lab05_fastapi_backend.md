

---

# 📘 **Lab 05 — Integrating Your RAG Engine with FastAPI**

*A Practical Guide to Production-Ready GenAI APIs*

---

## **Markdown Cell**

# **Lab 05 — Integrating Your RAG Engine with FastAPI**

In this lab, you will transform your standalone RAG pipeline into a production-ready **FastAPI** service.

By the end of the lab, you will build:

✅ A FastAPI backend with endpoints for:

* **`/embed`** — create embeddings
* **`/ingest`** — upload text and store embeddings
* **`/query`** — perform retrieval-augmented generation

✅ A clean modular architecture
✅ Automatic API docs via Swagger
--------------------------------

---

## **Markdown Cell**

## **Learning Objectives**

After completing this lab, you will be able to:

* Build modern API backends using **FastAPI**
* Organize RAG pipelines into structured services and routers
* Expose endpoints for embedding, ingestion, and querying
* Test endpoints using Swagger UI and Python scripts

---

## **Markdown Cell**

## **1. Environment Setup**

Install dependencies:

---

## **Code Cell**

```python
!pip install fastapi uvicorn openai tiktoken python-dotenv weaviate-client
```

---

## **Markdown Cell**

## **2. Recommended Project Structure**

We follow a modular API architecture:

```
rag-fastapi/
│── main.py
│── routers/
│     ├── ingest.py
│     ├── query.py
│── services/
│     ├── embedding_service.py
│     ├── rag_service.py
│── vectorstore/
│     ├── weaviate_client.py
│── data/
│── .env
│── requirements.txt
```

---

## **Markdown Cell**

## **3. Create `.env` file**

Add this content:

---

## **Code Cell**

```python
OPENAI_API_KEY=your_api_key_here
WEAVIATE_URL=http://localhost:8080
```

---

## **Markdown Cell**

## **4. FastAPI Application — `main.py`**

This file initializes the API and mounts routers.

---

## **Code Cell**

```python
from fastapi import FastAPI
from routers import ingest, query

app = FastAPI(
    title="RAG API Service",
    description="API for embedding, ingestion, and retrieval-augmented generation.",
    version="1.0.0",
)

app.include_router(ingest.router, prefix="/ingest", tags=["Ingestion"])
app.include_router(query.router, prefix="/query", tags=["RAG Query"])

@app.get("/")
def root():
    return {"message": "RAG API is running!"}
```

---

## **Markdown Cell**

## **5. Embedding Service — `embedding_service.py`**

This class creates OpenAI embeddings.

---

## **Code Cell**

```python
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

class EmbeddingService:
    def embed(self, text: str):
        response = client.embeddings.create(
            model="text-embedding-3-large",
            input=text
        )
        return response.data[0].embedding
```

---

## **Markdown Cell**

## **6. Vector Store (Weaviate Client)**

Wraps Weaviate operations.

---

## **Code Cell**

```python
import os
import weaviate

class VectorDB:
    def __init__(self):
        self.client = weaviate.Client(os.getenv("WEAVIATE_URL"))

    def add_document(self, text, embedding):
        self.client.data_object.create({
            "text": text,
            "embedding": embedding
        }, "Document")

    def search(self, embedding, k=3):
        result = (
            self.client.query.get("Document", ["text"])
            .with_near_vector({"vector": embedding})
            .with_limit(k)
            .do()
        )
        return [d["text"] for d in result["data"]["Get"]["Document"]]
```

---

## **Markdown Cell**

## **7. RAG Service**

Combines embedding, retrieval, and generation.

---

## **Code Cell**

```python
class RAGService:
    def __init__(self, embedder, vectordb):
        self.embedder = embedder
        self.vectordb = vectordb

    def query(self, question):
        q_embed = self.embedder.embed(question)
        docs = self.vectordb.search(q_embed)

        context = "\n".join(docs)

        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a RAG assistant."},
                {"role": "user", "content": f"Context: {context}\nQuestion: {question}"}
            ]
        )

        return {
            "answer": response.choices[0].message["content"],
            "retrieved_docs": docs
        }
```

---

## **Markdown Cell**

## **8. Ingestion Endpoint — `routers/ingest.py`**

---

## **Code Cell**

```python
from fastapi import APIRouter
from pydantic import BaseModel

from services.embedding_service import EmbeddingService
from vectorstore.weaviate_client import VectorDB

router = APIRouter()
embedder = EmbeddingService()
vectordb = VectorDB()

class IngestRequest(BaseModel):
    text: str

@router.post("/")
def ingest_text(req: IngestRequest):
    embedding = embedder.embed(req.text)
    vectordb.add_document(req.text, embedding)
    return {"status": "success", "message": "Document added!"}
```

---

## **Markdown Cell**

## **9. Query Endpoint — `routers/query.py`**

---

## **Code Cell**

```python
from fastapi import APIRouter
from pydantic import BaseModel

from services.rag_service import RAGService
from services.embedding_service import EmbeddingService
from vectorstore.weaviate_client import VectorDB

router = APIRouter()

rag = RAGService(EmbeddingService(), VectorDB())

class QueryRequest(BaseModel):
    question: str

@router.post("/")
def query_rag(req: QueryRequest):
    return rag.query(req.question)
```

---

## **Markdown Cell**

## **10. Run the FastAPI Application**

---

## **Code Cell**

```python
!uvicorn main:app --reload --port 8000
```

---

## **Markdown Cell**

# ✅ **11. Exercises**

### **Exercise 1 — Add Batch Ingestion**

Extend `/ingest` to accept a list of documents instead of a single string.

---

### **Exercise 2 — Add Metadata Support**

Modify your VectorDB to store:

* title
* URL
* tags
* upload timestamp

---

### **Exercise 3 — Add Embedding Inspection Endpoint**

Create a new router:

```
/embed  
```

Which returns embeddings for debugging.

---

### **Exercise 4 — Add Hybrid Search**

Implement:

* Keyword Search (BM25)
* Vector Search
* Rerank using OpenAI

---

### **Exercise 5 — Add Streaming LLM Responses**

Modify the `/query` endpoint to stream output tokens back to the client.

---

## **Markdown Cell**

# ⭐ **End of Lab 05**

You now have a production-ready RAG backend with FastAPI.

---


