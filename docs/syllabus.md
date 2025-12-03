
---

# **Course Syllabus**

## **Course Overview**

This course teaches the end-to-end process of building production-ready Generative AI (GenAI) applications. Participants will learn how to design, build, deploy, and maintain modern AI systems using LLMs, embeddings, vector databases, RAG pipelines, and scalable backend/frontend architectures.

The course follows a **hands-on, project-based approach**, culminating in a fully functional GenAI application deployed to the cloud.

---

## **Course Format**

* **Duration:** 6–8 weeks
* **Structure:** Weekly lectures + hands-on labs
* **Delivery:** Online or in-person
* **Tools:** Python, FastAPI, LangChain/LlamaIndex, Weaviate/Pinecone, Next.js, Docker

---

# **Learning Outcomes**

By the end of this course, participants will be able to:

1. Understand core GenAI concepts and architectures.
2. Ingest and process documents, create embeddings, and build vector indexes.
3. Implement Retrieval-Augmented Generation (RAG).
4. Build and deploy GenAI APIs using FastAPI.
5. Build interactive frontends using Next.js or Streamlit.
6. Deploy systems using Docker, Fly.io, AWS, or Railway.
7. Understand observability, monitoring, and model evaluation.
8. Apply governance, security, and compliance best practices.
9. Build real-world GenAI apps tailored to finance, legal, and enterprise use cases.

---

# **Target Audience**

* Software Engineers
* Data Scientists / AI Engineers
* Product Teams
* IT Professionals
* Students and tech enthusiasts interested in modern AI development

---

# **Prerequisites**

* Basic Python programming
* Basic understanding of APIs
* Familiarity with Docker (optional but helpful)
* Basic knowledge of machine learning concepts

---

# **Course Modules**

---

## **Module 1: Introduction to Generative AI**

### Topics:

* What is Generative AI?
* Types of GenAI applications
* LLM architecture & fundamentals
* RAG overview

### Lab:

* Run your first LLM locally (Llama / Mistral)
* Make your first API call to an LLM provider (OpenAI, Anthropic)

---

## **Module 2: Understanding the Problem & Designing the System**

### Topics:

* Identifying business use cases
* Problem-to-solution mapping
* Data requirements & structure
* Functional and non-functional requirements
* Architecture blueprint of a GenAI system

### Lab:

* Document your project concept
* Design an architecture diagram

---

## **Module 3: Embeddings, Vector Databases & Indexing**

### Topics:

* What are embeddings?
* Choosing an embedding model
* Vector similarity search (cosine, dot-product, etc.)
* Chunking strategies
* Metadata design
* Choosing a vector DB (Weaviate, Pinecone, Chroma, Milvus)

### Lab:

* Build an ingestion pipeline
* Chunk documents and generate embeddings
* Create & query a vector database

---

## **Module 4: Building a RAG Pipeline**

### Topics:

* Anatomy of a RAG system
* Query → embeddings → retrieval → prompt building → LLM
* Hybrid retrieval (BM25 + vector search)
* Advanced prompts engineering
* Managing hallucinations

### Lab:

* Build your first RAG engine
* Evaluate retrieval quality
* Implement contextual prompts

---

## **Module 5: Backend Engineering with FastAPI**

### Topics:

* API design principles
* FastAPI structure & endpoints
* Integration with RAG pipeline
* Streaming responses
* Error handling & logging

### Lab:

* Build `/ingest`, `/query`, `/search`, and `/feedback` endpoints
* Integrate RAG engine into backend
* Implement streaming chat responses

---

## **Module 6: Orchestration Frameworks**

### Topics:

* LangChain basics
* LlamaIndex basics
* Agents & tools
* Assessing when to use or avoid orchestration frameworks

### Lab:

* Build a simple Agent
* Combine tools + RAG inside an agent workflow

---

## **Module 7: Frontend Engineering**

### Topics:

* Next.js for GenAI frontends
* Chat UI components
* Integrating streaming from FastAPI
* File upload UI
* Authentication (optional)

### Lab:

* Build a working chat interface
* Connect frontend to backend
* Implement file upload → ingestion

---

## **Module 8: Deployment & CI/CD**

### Topics:

* Docker for GenAI applications
* Building GPU and CPU images
* Deploying on Fly.io, Railway, AWS, GCP
* Managing environment variables
* GitHub Actions CI/CD pipelines

### Lab:

* Containerize your application
* Deploy the full stack
* Set up simple CI/CD for automatic deployment

---

## **Module 9: Observability, Monitoring & Evaluation**

### Topics:

* Token usage monitoring
* Cost optimization
* Model quality evaluation (automated + manual)
* Logging user feedback
* Groundedness scoring

### Lab:

* Implement logging & analytics
* Build an evaluation dataset
* Create a monitoring dashboard (Grafana or simple UI)

---

## **Module 10: Security, Governance & Compliance**

### Topics:

* Access control
* Role-based access for documents
* Redaction & filtering
* Data privacy (PII handling)
* Model versioning & audit logs

### Lab:

* Implement document-level access control
* Add API authentication (JWT)

---

## **Module 11: Scaling GenAI Systems**

### Topics:

* Scaling vector DB
* Caching strategies for LLMs
* When to use fine-tuning
* Load balancing & distributed workloads

### Lab:

* Implement caching for responses
* Run load tests

---

## **Module 12: Capstone Project**

Participants build a complete GenAI application from scratch.
Options include:

* **Legal Document Intelligence Assistant**
* **Financial Market Insights Bot**
* **Business Data Search Assistant**
* **Enterprise Knowledge Management Tool**
* **PDF Smart Q&A Application**

### Deliverables:

* Working full-stack GenAI app
* Deployment link
* Architecture document
* README & documentation
* Demo presentation

---

# **Assessment Strategy**

* **Weekly Labs (40%)**
* **Quizzes (10%)**
* **Final Project (50%)**

---

# **Course Materials**

* Lecture slides
* Code templates / GitHub repos
* Sample data (PDFs, CSVs, webpages)
* Recommended tools:

  * Python, FastAPI
  * LangChain, LlamaIndex
  * Weaviate, Pinecone
  * Docker
  * Next.js

---

# **Instructor**

Tajudeen Abdulazeez
Generative AI Engineer
Specialist in DataOps, MLOps, LLMOps
Founder, Toraaglobal LLC / Remitpro Ltd

---

