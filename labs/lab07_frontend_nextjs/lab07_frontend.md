

---

# **Lab 07 — Multi-Document RAG Assistant with Query Routing & Metadata Filtering**

### **Learning Objectives**

In this lab, you will learn:

✅ How to ingest multiple document types into LlamaIndex
✅ How to use metadata extraction & chunk-level metadata
✅ How to build **query routing**, so the LLM decides which document index to use
✅ How to build a **router query engine** and evaluate its performance
✅ How to expose your multi-document RAG pipeline as a single unified interface

---

# **1. Lab Setup**

### **1.1 Install Dependencies**

```bash
pip install llama-index llama-index-llms-openai llama-index-embeddings-openai python-dotenv
```

---

# **2. Create the Lab Structure**

```
lab-07/
│── data/
│     ├── finance.pdf
│     ├── hr-handbook.docx
│     ├── engineering-guidelines.pdf
│── multi_rag.py
│── router_config.json
│── QueryRouterDemo.ipynb
│── .env
```

Your `.env` should contain:

```
OPENAI_API_KEY=your_key
```

---

# **3. Writing the Loader + Metadata Extractor**

### **3.1 Load Multi-Format Documents**

```python
from llama_index.core import SimpleDirectoryReader

docs = SimpleDirectoryReader("./data").load_data()
len(docs)
```

---

# **4. Attach Metadata Automatically**

### **4.1 Metadata Extraction Example**

```python
for d in docs:
    d.metadata["source_file"] = d.metadata.get("file_name")
    d.metadata["department"] = (
        "Finance" if "finance" in d.metadata["file_name"].lower()
        else "HR" if "hr" in d.metadata["file_name"].lower()
        else "Engineering"
    )
```

### **4.2 Verify Metadata**

```python
docs[0].metadata
```

---

# **5. Build Separate Indexes per Department**

```python
from llama_index.core import VectorStoreIndex

finance_docs = [d for d in docs if d.metadata["department"] == "Finance"]
hr_docs = [d for d in docs if d.metadata["department"] == "HR"]
eng_docs = [d for d in docs if d.metadata["department"] == "Engineering"]

finance_index = VectorStoreIndex.from_documents(finance_docs)
hr_index = VectorStoreIndex.from_documents(hr_docs)
eng_index = VectorStoreIndex.from_documents(eng_docs)
```

---

# **6. Create Query Engines**

```python
finance_engine = finance_index.as_query_engine()
hr_engine = hr_index.as_query_engine()
eng_engine = eng_index.as_query_engine()
```

---

# **7. Build the Query Router**

### **7.1 Router Definition JSON**

Create a `router_config.json`:

```json
{
  "finance": "Financial policies, budgets, procurement, invoices",
  "hr": "HR policies, hiring, employee benefits, leave, payroll",
  "engineering": "Engineering processes, architecture, infrastructure"
}
```

---

# **8. Load Router & Build the Routing LLM**

```python
import json
from llama_index.core.selectors import LLMSingleSelector
from llama_index.llms.openai import OpenAI

llm = OpenAI(model="gpt-4.1-mini")

router_map = json.load(open("router_config.json"))
selector = LLMSingleSelector.from_components(llm=llm, choices=list(router_map.keys()))
```

---

# **9. Build a Router Query Engine**

```python
from llama_index.core.query_engine import RouterQueryEngine

router_engine = RouterQueryEngine(
    selector=selector,
    query_engine_tools={
        "finance": finance_engine,
        "hr": hr_engine,
        "engineering": eng_engine
    }
)
```

---

# **10. Run Queries**

### **10.1 Finance Query**

```python
router_engine.query("What is the procurement spending approval limit?")
```

### **10.2 HR Query**

```python
router_engine.query("How many annual leave days do employees get?")
```

### **10.3 Engineering Query**

```python
router_engine.query("Explain the deployment requirements for internal applications.")
```

---

# **11. Add Metadata Filtering**

### **11.1 Build an Index with Metadata Filters**

```python
finance_query_with_filters = finance_index.as_query_engine(
    filters={"department": "Finance"}
)
```

### **11.2 Test Filtering**

```python
finance_query_with_filters.query("List financial compliance rules")
```

---

# **12. Unified RAG Interface**

Create a helper function:

```python
def ask_company(question: str):
    print("Question:", question)
    answer = router_engine.query(question)
    print("Answer:", answer)
```

Usage:

```python
ask_company("Describe the engineering architecture review process.")
```

---

# **13. Exercises**

## **Exercise 1 — Add Another Department**

Add a “Legal” folder and:

1. Load docs
2. Classify metadata
3. Build a Legal index
4. Add to router
5. Demonstrate queries

---

## **Exercise 2 — Improve Metadata Extraction**

Enhance metadata to include:

* Year
* Document owner
* Document category
* Sensitivity level

---

## **Exercise 3 — Create a Confidence Score**

Add code that prints:

* Selected engine
* Router confidence
* Reasoning

Tip: `selector.select(choice_descriptions, query)` returns detailed reasoning.

---

## **Exercise 4 — Build a Hybrid Retrieval Engine**

Enhance each department index with:

* Vector search
* Keyword BM25 search
* Reranking

Bonus: create a combined hybrid engine.

---

# **14. Exercise Answers (Short)**

### **Exercise 1 (Legal Dept) Example**

```python
legal_docs = [d for d in docs if "legal" in d.metadata["file_name"].lower()]
legal_index = VectorStoreIndex.from_documents(legal_docs)
legal_engine = legal_index.as_query_engine()
router_engine.query_engine_tools["legal"] = legal_engine
```

---

### **Exercise 2 (Better Metadata)**

```python
for d in docs:
    d.metadata["year"] = 2024
    d.metadata["owner"] = "Department Head"
    d.metadata["category"] = "Policy"
    d.metadata["sensitivity"] = "Internal Only"
```

---

### **Exercise 3 (Confidence Score)**

```python
sel = selector.select(list(router_map.keys()), "sample query")
print(sel.reasoning)
print(sel.score)
```

---

### **Exercise 4 (Hybrid Engine)**

```python
from llama_index.core.retrievers import BM25Retriever
from llama_index.core.query_engine import RetrieverQueryEngine

bm25 = BM25Retriever.from_defaults(documents=finance_docs)
vec = finance_index.as_retriever()

hybrid = RetrieverQueryEngine.from_retrievers(
    [bm25, vec], weights=[0.3, 0.7]
)
```

---

# **Lab 07 Summary**

By completing this lab, you now understand:

💡 How to manage **multiple indexes**
💡 How to build **router query engines**
💡 How to integrate **metadata filtering**
💡 How to unify your RAG system into a single assistant
💡 How to create **organization-scale RAG workflows**

---

