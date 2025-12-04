
---

# **Lab 08 — Deployment of RAG System Using Docker**

### **Learning Objectives**

In this lab, you will:

✅ Package your RAG pipeline (indexes + query routing) into a FastAPI service
✅ Build a production-ready Dockerfile
✅ Use Docker Compose to run the API + Caddy reverse proxy (optional)
✅ Add environment variables and `.env` files
✅ Test the deployed API locally and via a browser/postman
✅ Understand best practices for deploying RAG workloads in containers

---

# **1. Lab Folder Structure**

Use this recommended structure:

```
lab-08/
│── app/
│    ├── main.py
│    ├── rag_router.py
│    ├── build_indexes.py
│    ├── data/
│    └── indexes/
│── Dockerfile
│── docker-compose.yml
│── requirements.txt
│── .env
```

---

# **2. Create Requirements File**

`requirements.txt`

```
fastapi
uvicorn
llama-index
llama-index-llms-openai
llama-index-embeddings-openai
python-dotenv
pydantic
```

---

# **3. Create Your Environment File**

`.env`

```
OPENAI_API_KEY=your-api-key
LLM_MODEL=gpt-4o-mini
```

> **Note:** We will load this inside the container.

---

# **4. Build Indexes Before Deployment**

Create `app/build_indexes.py`.

This script loads documents, creates department indexes, and saves them locally so the Docker image can preload them.

```python
import os
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex

DATA_DIR = "./app/data"
INDEX_DIR = "./app/indexes"

os.makedirs(INDEX_DIR, exist_ok=True)

docs = SimpleDirectoryReader(DATA_DIR).load_data()

finance_docs = [d for d in docs if "finance" in d.metadata["file_name"].lower()]
hr_docs = [d for d in docs if "hr" in d.metadata["file_name"].lower()]
eng_docs = [d for d in docs if "eng" in d.metadata["file_name"].lower()]

finance_index = VectorStoreIndex.from_documents(finance_docs)
finance_index.storage_context.persist(f"{INDEX_DIR}/finance")

hr_index = VectorStoreIndex.from_documents(hr_docs)
hr_index.storage_context.persist(f"{INDEX_DIR}/hr")

eng_index = VectorStoreIndex.from_documents(eng_docs)
eng_index.storage_context.persist(f"{INDEX_DIR}/eng")

print("Indexes built and saved.")
```

Run the script locally to generate your indexes:

```bash
python app/build_indexes.py
```

---

# **5. Build the Router + RAG Engine**

Create `app/rag_router.py`.

```python
import os, json
from llama_index.core import VectorStoreIndex, StorageContext, load_index_from_storage
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector
from llama_index.llms.openai import OpenAI

MODEL = os.getenv("LLM_MODEL", "gpt-4o-mini")

llm = OpenAI(model=MODEL)

INDEX_DIR = "./app/indexes"

def load_saved_index(path):
    storage_ctx = StorageContext.from_defaults(persist_dir=path)
    return load_index_from_storage(storage_ctx)

finance_index = load_saved_index(f"{INDEX_DIR}/finance")
hr_index = load_saved_index(f"{INDEX_DIR}/hr")
eng_index = load_saved_index(f"{INDEX_DIR}/eng")

finance_engine = finance_index.as_query_engine()
hr_engine = hr_index.as_query_engine()
eng_engine = eng_index.as_query_engine()

router_config = {
  "finance": "Financial policies, budgets, procurement",
  "hr": "Hiring, benefits, leave",
  "engineering": "Architecture, infrastructure"
}

selector = LLMSingleSelector.from_components(
    llm=llm,
    choices=list(router_config.keys())
)

router_engine = RouterQueryEngine(
    selector=selector,
    query_engine_tools={
        "finance": finance_engine,
        "hr": hr_engine,
        "engineering": eng_engine
    }
)

def answer_question(query: str):
    response = router_engine.query(query)
    return str(response)
```

---

# **6. Create FastAPI Application**

Create `app/main.py`.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from rag_router import answer_question

class Query(BaseModel):
    question: str

app = FastAPI(title="Multi-RAG API", version="1.0")

@app.post("/query")
def ask(q: Query):
    return {"answer": answer_question(q.question)}

@app.get("/")
def home():
    return {"status": "ok", "message": "RAG API is running"}
```

---

# **7. Write the Dockerfile**

`Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# env file
ENV PYTHONUNBUFFERED=1

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# **8. Create Docker Compose**

`docker-compose.yml`

```yaml
version: "3.8"

services:
  rag-api:
    build: .
    container_name: rag_api
    env_file: .env
    ports:
      - "8000:8000"
    volumes:
      - ./app/indexes:/app/app/indexes
```

---

# **9. Build and Run the Container**

### **9.1 Build**

```bash
docker compose build
```

### **9.2 Run**

```bash
docker compose up -d
```

---

# **10. Test the API**

### **In Browser**

Open:

```
http://localhost:8000
```

### **Swagger UI**

```
http://localhost:8000/docs
```

### **POST Query**

```bash
curl -X POST http://localhost:8000/query \
-H "Content-Type: application/json" \
-d '{"question":"What is our procurement limit?"}'
```

---

# **11. Optional — Add Caddy Reverse Proxy**

Add to `docker-compose.yml`:

```yaml
  caddy:
    image: caddy
    container_name: rag_caddy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - rag-api
```

`Caddyfile`:

```
yourdomain.com {
    reverse_proxy rag-api:8000
}
```

---

# **12. Deployment Best Practices**

| Category                 | Best Practice                                           |
| ------------------------ | ------------------------------------------------------- |
| **Model Calls**          | Use API key from env, never hardcoded                   |
| **Indexes**              | Prebuild indexes to reduce container startup time       |
| **Docker layer caching** | Copy requirements first, then install                   |
| **Security**             | Use Docker secrets for production API keys              |
| **Autoscaling**          | Stateless FastAPI + persistent vector store recommended |
| **Logging**              | Add structured JSON logs                                |

---

# **13. Exercises**

### **Exercise 1 — Add Logging Middleware**

Implement request + response logging inside FastAPI.

### **Exercise 2 — Add a `/healthz` Endpoint**

Check router, indexes, and LLM availability.

### **Exercise 3 — Push to Docker Hub**

Build, tag & push:

```bash
docker build -t remitpro/openledger-rag:latest .
docker push remitpro/openledger-rag:latest
```

### **Exercise 4 — Add a Redis Cache Layer**

Cache repeated queries.

---

# **Lab 08 Summary**

You learned how to:

💡 Prebuild RAG indexes
💡 Load indexes inside a FastAPI backend
💡 Containerize the entire system using Docker
💡 Run and test using Docker Compose
💡 Add a production-ready reverse proxy with Caddy

---

