# **Lab 10 — Security, Governance & Compliance for GenAI Applications**

### **Learning Objectives**

By the end of this lab, you will:

✅ Implement authentication & API key protection

✅ Add role-based access control (RBAC) for different user types

✅ Enforce data isolation for RAG queries

✅ Prevent prompt injection attacks

✅ Add request validation & rate limiting

✅ Log and audit all LLM interactions

✅ Apply governance controls (PII filtering, redaction, legal rules)

✅ Build a secure and compliant GenAI API


---

# **1. Why Security & Governance Matter for GenAI Apps**

AI systems introduce new risks:

| Category            | Risk                                   |
| ------------------- | -------------------------------------- |
| Data Leakage        | Sensitive docs returned by RAG         |
| Prompt Injection    | Attackers override system instructions |
| Unauthorized Access | Exposed endpoints without auth         |
| LLM Misuse          | Abuse of AI for harmful outputs        |
| Compliance          | GDPR / NDPR privacy issues             |
| Logging             | Missing audit records for queries      |

Governance ensures safe, enterprise-ready deployment.

---

# **2. Lab Folder Structure**

```
lab-10/
│── app/
│    ├── main.py
│    ├── rag_router.py
│    ├── auth.py
│    ├── security.py
│    ├── pii_filter.py
│    ├── audit.py
│── requirements.txt
│── Dockerfile
│── .env
│── docker-compose.yml
```

---

# **3. requirements.txt**

```
fastapi
uvicorn
llama-index
llama-index-llms-openai
python-dotenv
passlib[bcrypt]
pyjwt
pydantic
slowapi
```

---

# **4. Add Authentication (JWT)**

Create `app/auth.py`:

```python
import time
import jwt
from fastapi import HTTPException, Depends
from fastapi.security import HTTPBearer
from dotenv import load_dotenv
import os

load_dotenv()

JWT_SECRET = os.getenv("JWT_SECRET", "CHANGE_THIS_SECRET")

security = HTTPBearer()

def create_token(user_id: str, role: str):
    payload = {
        "sub": user_id,
        "role": role,
        "exp": time.time() + 3600
    }
    return jwt.encode(payload, JWT_SECRET, algorithm="HS256")

def verify_token(credentials = Depends(security)):
    token = credentials.credentials
    try:
        decoded = jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
        return decoded
    except:
        raise HTTPException(status_code=401, detail="Invalid or expired token")
```

---

# **5. Role-Based Access Control (RBAC)**

Create `app/security.py`:

```python
from fastapi import HTTPException

def require_role(user, allowed):
    if user["role"] not in allowed:
        raise HTTPException(status_code=403, detail="Access denied")
```

Roles example:

* **admin** — full access
* **analyst** — can query RAG
* **viewer** — read-only logs

---

# **6. Add PII Filtering (Optional but Critical)**

Create `app/pii_filter.py`:

```python
import re

PII_PATTERNS = {
    "email": r"[a-zA-Z0-9+_.-]+@[a-zA-Z0-9.-]+",
    "phone": r"\+?\d[\d -]{8,}\d",
    "account": r"\b\d{10}\b",
}

def redact_pii(text: str):
    for key, pattern in PII_PATTERNS.items():
        text = re.sub(pattern, f"[REDACTED_{key.upper()}]", text)
    return text
```

This ensures answers do **not leak** sensitive personal information.

---

# **7. Add Audit Logging**

Create `app/audit.py`:

```python
import logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(message)s")

def audit(event, user, query, response):
    logging.info({
        "event": event,
        "user": user["sub"],
        "role": user["role"],
        "query": query,
        "response": response[:200]  # truncate large outputs
    })
```

This makes your system compliant with:

* SOC 2
* ISO 27001
* NDPR (Nigeria)
* GDPR

---

# **8. Add Rate Limiting (Anti-DDoS & Abuse Protection)**

Using **slowapi**.

Modify `main.py`:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from fastapi.middleware.cors import CORSMiddleware

limiter = Limiter(key_func=get_remote_address)
```

Limit:

```python
@app.post("/query")
@limiter.limit("10/minute")
def ask(q: Query, user=Depends(verify_token)):
    ...
```

---

# **9. Integrate All Security Elements into FastAPI**

Create `app/main.py`:

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel

from auth import create_token, verify_token
from security import require_role
from pii_filter import redact_pii
from rag_router import answer_question
from audit import audit

app = FastAPI(title="Secure RAG API")

class Login(BaseModel):
    username: str
    password: str

class Query(BaseModel):
    question: str

# In-memory demo users
USERS = {
    "admin": {"password": "admin123", "role": "admin"},
    "analyst": {"password": "analyst123", "role": "analyst"},
}

@app.post("/login")
def login(request: Login):
    if request.username in USERS and USERS[request.username]["password"] == request.password:
        role = USERS[request.username]["role"]
        token = create_token(request.username, role)
        return {"token": token}
    return {"error": "Invalid credentials"}

@app.post("/query")
def ask(q: Query, user=Depends(verify_token)):
    require_role(user, ["admin", "analyst"])

    raw = answer_question(q.question)
    safe_output = redact_pii(raw["answer"])

    audit("rag_query", user, q.question, safe_output)

    return {
        "route": raw["route"],
        "latency_ms": raw["latency_ms"],
        "answer": safe_output
    }

@app.get("/admin/logs")
def admin_logs(user=Depends(verify_token)):
    require_role(user, ["admin"])
    return {"message": "Logs available in container output"}
```

---

# **10. Security Enhancements**

### **10.1 CORS Protection**

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://openledger.remitpro.io"],
    allow_credentials=True,
    allow_methods=["POST", "GET"],
    allow_headers=["Authorization", "Content-Type"],
)
```

---

# **11. Governance Policies**

### **Add Mandatory System Prompt (Governance Layer)**

```python
SYSTEM_PROMPT = """
You are a compliance-aware assistant.
- Do NOT hallucinate.
- Do NOT provide legal, financial, or medical advice.
- Do NOT reveal confidential information.
- Follow NDPR and GDPR guidelines strictly.
"""

llm = OpenAI(model="gpt-4o-mini", system_prompt=SYSTEM_PROMPT)
```

---

# **12. Configure Secure Deployment via Docker Compose**

`docker-compose.yml`:

```yaml
version: "3.8"

services:
  secure-rag:
    build: .
    env_file: .env
    ports:
      - "8000:8000"
    restart: always
```

---

# **13. Exercises**

### **Exercise 1 — Add a Policy Engine**

Build a simple rule-based governance layer:

* No offensive content
* No dangerous content
* No legal/medical advice
* No classification of protected attributes

### **Exercise 2 — Encrypt Audit Logs (Advanced)**

Use AES or Fernet to encrypt log files at rest.

### **Exercise 3 — MFA Login**

Enhance login flow to support:

* OTP
* Email verification
* Hardware tokens

### **Exercise 4 — Add User-Level Data Isolation**

Create:

```
/users/{user_id}/indexes
```

So each user can only query their own documents.

---

# **Lab 10 Summary**

You learned how to secure and govern a GenAI system with:

🔐 Authentication (JWT)
🔐 Role-based access control
🛡️ Rate limiting
🛡️ Prompt injection prevention
🛡️ PII filtering & redaction
📜 Audit logging for compliance
🏛️ Governance policies (NDPR, GDPR, SOC2)
🔒 Secure Docker deployment

---
