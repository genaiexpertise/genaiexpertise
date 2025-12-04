
---

# **Lab 11 — Scaling & High-Availability for GenAI Applications**

*From local prototype → enterprise-grade scalable GenAI system*

---

## **🎯 Learning Objectives**

By the end of this lab, you will be able to:

* Understand horizontal vs. vertical scaling for GenAI apps
* Implement autoscaling strategies for:

  * **FastAPI inference servers**
  * **Vector database nodes**
  * **Embedding workers**
* Configure **Load Balancers** (K8s / Docker Swarm / Nginx / AWS ALB)
* Implement **Async Task Queues** (Celery + Redis / RabbitMQ)
* Evaluate **Throughput, Concurrency & Latency**
* Perform a **Scaling Load Test** using Locust
* Apply **Caching strategies** for high throughput
* Implement **Multi-model routing**
* Understand **GPU & cost optimization strategies**

---

# **Notebook Structure**

### 1️⃣ Introduction to Scaling GenAI Systems

### 2️⃣ Core Scaling Challenges

### 3️⃣ Scaling RAG Architecture

### 4️⃣ Horizontal vs. Vertical Scaling

### 5️⃣ Load Balancing Your FastAPI App

### 6️⃣ Autoscaling Workers (Embeddings + Retrieval)

### 7️⃣ Scaling Vector Databases

### 8️⃣ Caching Strategies

### 9️⃣ Locust Load Testing

### 🔟 Metrics & Observability

### 1️⃣1️⃣ Exercises + Answers

---

# ------------------------------------------

# **1️⃣ Introduction to Scaling GenAI Systems**

# ------------------------------------------

GenAI systems demand significantly more resources than standard web apps:

| Component            | Scaling Need                     |
| -------------------- | -------------------------------- |
| LLM API              | High QPS, low latency            |
| Embeddings generator | Batch processing jobs            |
| Vector DB            | RAM-heavy index, sharding        |
| RAG Pipeline         | Parallelism across retrieval/LLM |
| Frontend API         | Autoscaling based on concurrency |

Large-scale infrastructure often uses:

* Kubernetes (K8s)
* Kubernetes + KEDA autoscaler
* ECS + Fargate
* GCP Cloud Run
* Fly.io
* Modal / BentoML

---

# --------------------------------------------------

# **2️⃣ Core Scaling Challenges in GenAI Pipelines**

# --------------------------------------------------

**Challenge 1 — Long running GPU inference**
**Challenge 2 — Low latency requirements (sub-2 sec)**
**Challenge 3 — Large memory consumption**
**Challenge 4 — High concurrency on retrieval**
**Challenge 5 — Expensive computation → aggressive caching**

---

# ----------------------------------------

# **3️⃣ Scaling a RAG Architecture Diagram**

# ----------------------------------------

```
+--------------------------+
|   Client / Frontend UI   |
+-----------+--------------+
            |
            v
+-------------------------------+
|        API Gateway / LB       |
+---------------+---------------+
                |
  +-------------+--------------+
  |                            |
  v                            v
FastAPI Server A         FastAPI Server B
  |                          |
  +-----------+--------------+
              |
              v
   +-------------------------+
   |  Task Queue (Redis)     |
   +-----------+-------------+
               |
               v
      +-------------------+
      | Embedding Workers |
      +---------+---------+
                |
                v
        +---------------+
        | Vector DB     |
        | (Weaviate)    |
        +-------+-------+
                |
                v
         +-------------+
         | LLM API     |
         +-------------+
```

---

# -------------------------------------

# **4️⃣ Horizontal vs Vertical Scaling**

# -------------------------------------

### **Vertical Scaling (Scale Up)**

Increase CPU/RAM/GPU on a single machine
→ *Easy but expensive; limited ceiling*

### **Horizontal Scaling (Scale Out)**

Add more nodes & distribute load
→ *Preferred for GenAI systems*

---

# -------------------------------------------------

# **5️⃣ Load Balancing Your FastAPI Inference API**

# -------------------------------------------------

Below is a simple **Nginx load balancer** for FastAPI:

```nginx
upstream fastapi_backend {
    server app1:8000;
    server app2:8000;
    server app3:8000;
}

server {
    listen 80;

    location / {
        proxy_pass http://fastapi_backend;
    }
}
```

### **FastAPI running behind Gunicorn + Uvicorn workers**

```bash
gunicorn app:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000
```

---

# -----------------------------------------------------------

# **6️⃣ Autoscaling Workers (Embeddings + Retrieval Workers)**

# -----------------------------------------------------------

### Example **Celery worker for embeddings**

```python
from celery import Celery
from embeddings import embed_document

app = Celery("worker", broker="redis://redis:6379/0")

@app.task
def async_embed(text):
    return embed_document(text)
```

### Run multiple workers to scale:

```bash
celery -A worker worker --concurrency=4
```

In production, you run dozens:

```bash
docker-compose up --scale worker=20
```

---

# --------------------------------------

# **7️⃣ Scaling Vector Databases (Weaviate)**

# --------------------------------------

Weaviate supports:

* **Sharding**
* **Replication**
* **Query parallelism**
* **Memory-mapped storage**

### Basic multi-node config:

```yaml
cluster:
  nodeCount: 3
  replicationFactor: 2
  sharding:
    virtualShards: 128
    physicalShards: 3
```

---

# -------------------------------------------

# **8️⃣ Caching Strategies (Critical in GenAI)**

# -------------------------------------------

### Types of caching:

| Layer      | Cache Strategy              |
| ---------- | --------------------------- |
| LLM        | Cache identical prompts     |
| RAG        | Cache retrieved chunks      |
| Embeddings | Cache hash → vector mapping |
| API        | HTTP cache headers          |
| Frontend   | Client-side caching         |

### **In-memory TTL cache example**

```python
from cachetools import TTLCache

response_cache = TTLCache(maxsize=5000, ttl=3600)

def cached_llm(prompt):
    if prompt in response_cache:
        return response_cache[prompt]

    answer = call_llm(prompt)
    response_cache[prompt] = answer
    return answer
```

---

# -------------------------------------

# **9️⃣ Load Testing With Locust**

# -------------------------------------

### Install:

```bash
pip install locust
```

### locustfile.py:

```python
from locust import HttpUser, task, between

class RAGUser(HttpUser):
    wait_time = between(1, 5)

    @task
    def ask_question(self):
        self.client.post(
            "/query",
            json={"question": "What is the policy?"}
        )
```

### Run:

```bash
locust -f locustfile.py
```

Then visit `http://localhost:8089`.

---

# -------------------------------------

# **🔟 Metrics & Observability (Critical)**

# -------------------------------------

Track:

### **Latency**

* P50, P90, P95, P99

### **Throughput**

* Requests per second
* Concurrent users

### **Resource Usage**

* CPU, RAM, GPU
* Disk IO
* Network IO

### Tools:

* Prometheus
* Grafana Dashboards
* OpenTelemetry
* FastAPI Instrumentation

Example FastAPI metrics middleware:

```python
from starlette_exporter import PrometheusMiddleware, handle_metrics

app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", handle_metrics)
```

---

# -------------------------------

# **1️⃣1️⃣ Exercises**

# -------------------------------

### **Exercise 1 — Scaling Strategy Design**

Design a scaling plan for a RAG system receiving **1 million requests per day**.
Break it into:

* FastAPI layer
* Workers
* Vector DB
* Caching

### **Exercise 2 — Write a Locust Test**

Modify the Locust test so each user:

* Sends 3 queries
* Uses random questions
* Logs latency

### **Exercise 3 — Add LLM Cache**

Implement a prompt hash → answer cache.

### **Exercise 4 — Vector DB Scaling**

Explain when to:

* increase shards
* increase replicas
* increase RAM
* relocate nodes

---

# -------------------------------

# **Answers**

# -------------------------------

### **Answer 1 — Scaling Strategy**

* 8–20 FastAPI instances
* 20–50 worker nodes
* Vector DB cluster: 3 nodes, RF=2
* LLM cache reduces 40–70% duplicate calls
* CDN for static content

---

### **Answer 2 — Locust test**

```python
import random
from locust import HttpUser, task

questions = [
    "What is the KYC policy?",
    "Explain loan requirements.",
    "Give me a summary of the AML rule."
]

class RAGUser(HttpUser):
    @task
    def query(self):
        q = random.choice(questions)
        with self.client.post("/query", json={"question": q}, catch_response=True) as resp:
            resp.success()
```

---

### **Answer 3 — Prompt Cache Implementation**

```python
import hashlib

def hash_prompt(p):
    return hashlib.sha256(p.encode()).hexdigest()
```

---

### **Answer 4 — Vector DB scaling guidance**

* More shards → better parallelism
* More replicas → higher availability
* More RAM → faster ANN query performance
* Node relocation → reduce network latency

---

# ✅ Lab 11 Completed!
