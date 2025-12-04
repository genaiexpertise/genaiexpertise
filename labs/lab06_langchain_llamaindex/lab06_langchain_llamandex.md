
---

# **Lab 06 — LangChain + LlamaIndex Integration**

### **🎯 Learning Objectives**

By the end of this lab, you will:

* Understand **LangChain** and **LlamaIndex** roles in RAG pipelines
* Load documents into **LlamaIndex**
* Use **LangChain chains** to query LLMs
* Integrate **LlamaIndex retrieval with LangChain prompts**
* Build a **query interface** for your RAG system

---

# **1️⃣ Lab Folder Structure**

```
lab-06/
│── app/
│    ├── main.py
│    ├── rag_index.py
│    ├── llm_chains.py
│── data/
│    └── documents/
│── requirements.txt
```

---

# **2️⃣ requirements.txt**

```
langchain
llama-index
openai
python-dotenv
```

---

# **3️⃣ Create LlamaIndex for Document Storage**

### `app/rag_index.py`

```python
from llama_index import SimpleDirectoryReader, GPTVectorStoreIndex, LLMPredictor, ServiceContext
from langchain.chat_models import ChatOpenAI
import os
from dotenv import load_dotenv

load_dotenv()

os.environ["OPENAI_API_KEY"] = os.getenv("OPENAI_API_KEY")

# Load documents
documents = SimpleDirectoryReader('../data/documents/').load_data()

# Create LLM predictor
llm_predictor = LLMPredictor(llm=ChatOpenAI(model_name="gpt-4o-mini", temperature=0))

# Service context
service_context = ServiceContext.from_defaults(llm_predictor=llm_predictor)

# Build the index
index = GPTVectorStoreIndex.from_documents(documents, service_context=service_context)

# Save index for reuse
index.save_to_disk("app/index.json")
```

✅ **Exercise:** Add **document chunking** with overlap (e.g., 500 tokens with 50 overlap).

---

# **4️⃣ Load Index and Create Query Engine**

### `app/llm_chains.py`

```python
from llama_index import GPTVectorStoreIndex, ServiceContext, LLMPredictor
from langchain.chat_models import ChatOpenAI
from llama_index.prompts.prompts import QuestionAnswerPrompt
from llama_index.query_engine import RetrieverQueryEngine

# Load index
index = GPTVectorStoreIndex.load_from_disk("app/index.json")

# LLM setup
llm_predictor = LLMPredictor(llm=ChatOpenAI(model_name="gpt-4o-mini", temperature=0))
service_context = ServiceContext.from_defaults(llm_predictor=llm_predictor)

# Create retriever
retriever = index.as_retriever(search_kwargs={"k":5})

# Optional: Custom prompt
QA_PROMPT = QuestionAnswerPrompt("Answer the question based ONLY on the following context:\n{context_str}\nQuestion: {query_str}\nAnswer:")

# Create query engine
query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=None,  # Default synthesizer
    prompt=QA_PROMPT
)

def ask_question(query: str):
    response = query_engine.query(query)
    return str(response)
```

---

# **5️⃣ Build FastAPI Endpoint for RAG Queries**

### `app/main.py`

```python
from fastapi import FastAPI
from pydantic import BaseModel
from llm_chains import ask_question

app = FastAPI(title="LangChain + LlamaIndex RAG API")

class Query(BaseModel):
    question: str

@app.post("/query")
def query(payload: Query):
    answer = ask_question(payload.question)
    return {"answer": answer}

@app.get("/healthz")
def health():
    return {"status": "ok"}
```

---

# **6️⃣ Run FastAPI Server**

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Test with:

```json
POST http://localhost:8000/query
{
  "question": "What is the main function of LlamaIndex?"
}
```

---

# **7️⃣ Exercises**

### **Exercise 1 — Add Prompt Templates**

Create a custom LangChain `PromptTemplate` to include metadata in responses (e.g., document source).

### **Exercise 2 — Add Streaming Responses**

Use LangChain streaming to show partial answers in real-time.

### **Exercise 3 — Filter by Metadata**

Filter retrieved documents by a metadata field (e.g., `doc_type="policy"`).

### **Exercise 4 — Chain Multiple LLMs**

Combine a **smaller model for retrieval summaries** and **GPT-4o-mini for final answer synthesis**.

---

# **8️⃣ Answers / Hints**

* **Exercise 1:** Use `from langchain.prompts import PromptTemplate` and pass to LlamaIndex response synthesizer.
* **Exercise 2:** Use `ChatOpenAI(streaming=True)` in LLMPredictor.
* **Exercise 3:** Pass `metadata_filters={"doc_type": "policy"}` to `index.as_retriever()`.
* **Exercise 4:** Use `llm_predictor_small` for initial summary, then feed to `llm_predictor_large` for final answer.

---

# ✅ **Lab 06 Completed**

You now have a working **RAG pipeline** using **LangChain + LlamaIndex** with:

* Document ingestion & embedding
* Vector store index
* Retriever + query engine
* FastAPI API
* Prompt customization + optional chaining

---

