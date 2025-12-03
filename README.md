![](./docs/Gemini_Generated_Image_pynidvpynidvpyni.png)

***

# **A Practical Guide to Building a GenAI Application**

***

Welcome to the official lab repository for the course **A Practical Guide to Building a GenAI Application**.

This repository contains **all hands-on lab materials**, including code templates, notebooks, sample datasets, backend/frontend scaffolding, deployment scripts, and capstone project guides.

---

## **📚 Course Modules Covered**

Each folder under `/labs` corresponds to a course module:

1. **Intro to LLMs & APIs**
2. **Architecture Design**
3. **Embeddings & Vector Databases**
4. **RAG Pipeline Development**
5. **Backend Engineering (FastAPI)**
6. **LangChain & LlamaIndex Agents**
7. **Frontend Development (Next.js)**
8. **Deployment with Docker & Cloud**
9. **Monitoring, Logging & Evaluation**
10. **Security & Governance**
11. **Scaling GenAI Applications**
12. **Capstone Project**

---

# **🛠 Setup Instructions**

### **1. Clone the repo**

```bash
git clone https://github.com/<your-username>/genaiexpertise.git
cd genaiexpertise
```

### **2. Create a virtual environment**

```bash
python -m venv venv
source venv/bin/activate  # Mac/Linux
venv\Scripts\activate     # Windows
```

### **3. Install dependencies**

```bash
pip install -r requirements.txt
```

### **4. Setup environment variables**

Create `.env` file in project root:

```
OPENAI_API_KEY=your_key
WEAVIATE_URL=your_url
PINECONE_API_KEY=your_key
```

---

# **📂 Lab Highlights**

---

## **Lab 01: Intro to LLMs**

* Call OpenAI / Anthropic APIs
* Run open-source LLM locally (Llama / Mistral)
* Simple chat bot in Python

---

## **Lab 03: Embeddings & Vector Stores**

Includes:

* Chunking script
* Embedding creation
* Weaviate / Pinecone setup
* Vector search examples

---

## **Lab 04: RAG Pipeline**

You will build:

* Retrieval module
* Context builder
* RAG answer generator
* Reranking with Cohere or HuggingFace

---

## **Lab 05: FastAPI Backend**

A ready-made API project with endpoints:

```
POST /ingest
POST /query
POST /feedback
GET /health
```

---

## **Lab 07: Frontend (Next.js)**

Includes:

* Chat UI
* Streaming responses
* File upload → ingestion
* API integration

---

## **Lab 08: Deployment**

Templates for:

* Dockerfile
* docker-compose.yml
* Fly.io deployment guide

---

## **Lab 12: Capstone Project**

Includes:

* Project guide
* High-quality template structure
* Evaluation rubric
* Presentation tips

---

# **📦 Requirements**

Contents of `requirements.txt`:

```
fastapi
uvicorn
langchain
llama-index
weaviate-client
pinecone-client
openai
tiktoken
python-dotenv
pydantic
httpx
transformers
sentence-transformers
numpy
pandas
python-multipart
loguru
locust
redis
```

---

# **🔧 Contribution Guide**

Fork → Branch → Commit → Pull Request.

---

# **📄 License**

MIT License.

---

# **🚀 Author**

**Tajudeen Abdulazeez**
Generative AI Engineer | DataOps | MLOps | LLMOps
Founder, Toraaglobal LLC / Remitpro Ltd

---

