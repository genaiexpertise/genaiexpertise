

---

# **Lab 09 — Monitoring & Observability for RAG Systems**

### **Learning Objectives**

In this lab, you will:

✅ Add structured logging to your RAG API
✅ Log every query, routing decision, model response time
✅ Instrument your RAG pipeline with **Prometheus metrics**
✅ Add **OpenTelemetry tracing**
✅ Deploy **Grafana + Prometheus** with Docker Compose
✅ Build dashboards to monitor your LLM-based system
✅ Configure alerting for slow responses / high error rates

---

# **1. Why Monitoring Matters in GenAI Apps**

RAG/LLM apps require monitoring because:

| Problem                 | Why Monitoring Helps                    |
| ----------------------- | --------------------------------------- |
| Slow LLM responses      | Detect latency spikes                   |
| hallucinations          | Track confidence + token usage          |
| wrong routing decisions | Log final + fallback routes             |
| expensive tokens        | Track cost estimates                    |
| 500 errors              | See error cause + stacktrace            |
| degraded vector search  | Monitor DB load or missing index chunks |

**Observability = Logs + Metrics + Traces**

---

# **2. Lab Folder Structure**

```
lab-09/
│── app/
│    ├── main.py
│    ├── rag_router.py
│    ├── monitor.py
│── Dockerfile
│── docker-compose.yml
│── requirements.txt
│── .env
│── grafana/
│── prometheus/
```

---

# **3. requirements.txt**

```
fastapi
uvicorn
llama-index
llama-index-llms-openai
llama-index-embeddings-openai
prometheus-fastapi-instrumentator
opentelemetry-sdk
opentelemetry-exporter-otlp
opentelemetry-instrumentation-fastapi
python-dotenv
```

---

# **4. Add Monitoring Utilities**

Create `app/monitor.py`

```python
import time
import logging
from prometheus_fastapi_instrumentator import Instrumentator

logger = logging.getLogger("rag_monitor")
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)

def start_timer():
    return time.time()

def stop_timer(start):
    return round((time.time() - start) * 1000, 2)

def log_query(query, route, latency):
    logger.info({
        "event": "rag_query",
        "query": query,
        "routed_to": route,
        "latency_ms": latency
    })

def init_metrics(app):
    Instrumentator().instrument(app).expose(app)
```

---

# **5. Update the RAG Router to Emit Logs + Metrics**

Modify `rag_router.py`

```python
from monitor import start_timer, stop_timer, log_query

def answer_question(query: str):
    timer = start_timer()
    result = router_engine.query(query)

    route = router_engine.selector._latest_reasoning  # department selected
    latency = stop_timer(timer)

    log_query(query, route, latency)

    return {
        "route": route,
        "latency_ms": latency,
        "answer": str(result)
    }
```

---

# **6. Integrate Monitoring into FastAPI**

`app/main.py`

```python
from fastapi import FastAPI
from pydantic import BaseModel
from rag_router import answer_question
from monitor import init_metrics

class Query(BaseModel):
    question: str

app = FastAPI(title="RAG Monitoring API")

init_metrics(app)

@app.post("/query")
def ask(q: Query):
    return answer_question(q.question)

@app.get("/healthz")
def health():
    return {"status": "ok"}
```

---

# **7. Observability via Docker Compose**

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  rag-api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      - prometheus
      - grafana

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

---

# **8. Prometheus Configuration**

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "rag_api"
    static_configs:
      - targets: ["rag-api:8000"]
```

Prometheus will scrape:

```
http://rag-api:8000/metrics
```

---

# **9. Start the Monitoring Stack**

```bash
docker compose up -d
```

Services available:

| Service    | URL                                            |
| ---------- | ---------------------------------------------- |
| RAG API    | [http://localhost:8000](http://localhost:8000) |
| Prometheus | [http://localhost:9090](http://localhost:9090) |
| Grafana    | [http://localhost:3000](http://localhost:3000) |

---

# **10. Build Grafana Dashboards**

### **Recommended dashboards:**

🟦 **RAG Query Performance Dashboard**

* Average model latency
* Route distribution (finance/hr/eng)
* Vector DB latency
* LLM token usage
* Error rate (%)
* Request throughput

🟩 **LLM Cost Monitoring Dashboard**

* Tokens per query
* Estimated cost per hour
* Queries grouped by department

🟥 **System Health Dashboard**

* CPU/memory usage
* Container restarts
* API 4xx/5xx responses

### Step-by-Step Setup in Grafana:

1. Open Grafana → Login
   **username:** admin
   **password:** admin
2. Add data source → Prometheus
3. Set URL: `http://prometheus:9090`
4. Import dashboard ID:

   ```
   1860   (Grafana Node Exporter Full)
   3662   (API Latency Overview)
   12486  (FastAPI metrics)
   ```
5. Customize to add RAG-specific metrics.

---

# **11. Add OpenTelemetry Tracing (Optional Advanced)**

Add tracing to FastAPI:

```bash
pip install opentelemetry-instrumentation-fastapi opentelemetry-exporter-otlp
```

`main.py`

```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

FastAPIInstrumentor().instrument_app(app)
```

You can export traces to:

* Jaeger
* Grafana Tempo
* OpenTelemetry Collector

---

# **12. Add Alerts (Prometheus Alertmanager)**

Example alert rule: **RAG latency > 2000 ms**

```yaml
groups:
  - name: rag_alerts
    rules:
      - alert: HighLatency
        expr: http_server_request_duration_seconds_mean > 2
        for: 1m
        labels:
          severity: warning
        annotations:
          description: "RAG response latency above 2s"
```

---

# **13. Exercises**

### **Exercise 1 — Add Token Usage Metric**

Add:

```
rag_token_usage_total
```

### **Exercise 2 — Add Route Distribution Metric**

Counter per department:

```
rag_queries_finance_total
rag_queries_hr_total
rag_queries_engineering_total
```

### **Exercise 3 — Add Error Monitoring**

Track:

```
rag_error_total
```

### **Exercise 4 — Trigger High Latency Alert**

Simulate a query with artificial delay.

---

# **Lab 09 Summary**

You now know how to:

✅ Log every RAG query (structured logging)
✅ Add metrics using Prometheus
✅ Create dashboards using Grafana
✅ Add tracing with OpenTelemetry
✅ Deploy a full monitoring stack using Docker
✅ Set up alerting for slow/failed LLM calls

---
