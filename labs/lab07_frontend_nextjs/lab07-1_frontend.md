
---

# Lab 07 — Frontend for Your GenAI App (Next.js & Streamlit)

**Course:** A Practical Guide to Building a GenAI Application
**Duration:** 2 hours
**Objectives:**

* Build a production-minded chat UI in Next.js that connects to the FastAPI RAG backend
* Build a quick prototype UI in Streamlit for internal demos
* Support file upload → ingestion flow from the frontend
* Add streaming responses (SSE) support in the UI
* Add basic auth & client-side safety checks

---

# 1 — Overview & Choices

Two frontend tracks in this lab:

* **Next.js (React)** — production-ready, SSR/SSG, ideal for public product UIs.
* **Streamlit** — rapid prototyping, great for demos and internal tooling.

We'll provide full examples for both and exercises to extend them.

---

# 2 — Requirements (install locally)

Next.js (uses Node) and Streamlit (Python). Install tools as needed.

**Node / Next.js** (on dev machine)

```bash
# Node (16+), npm/yarn/pnpm
npx create-next-app@latest openledger-frontend
cd openledger-frontend
# later we will add dependencies
```

**Python / Streamlit**

```bash
python -m venv venv
source venv/bin/activate
pip install streamlit httpx python-dotenv
```

---

# 3 — Design Patterns & UX considerations

* Keep chat message state local and sync to a backend conversation store if needed.
* Support streaming so users see tokens as they generate.
* Provide "cite sources" UI: returned `contexts` with doc ids + clickable highlights.
* File upload should POST to `/ingest` then show ingestion status.
* Limit message length and sanitize before sending to backend.

---

# 4 — Next.js Chat UI (Minimal, production-minded)

Below are key files and components. Put these in `openledger-frontend/`.

## 4.1 `components/ChatBox.jsx`

```jsx
import { useState, useRef, useEffect } from "react";

export default function ChatBox({ backendUrl }) {
  const [input, setInput] = useState("");
  const [messages, setMessages] = useState([]); // {role, text, sources}
  const [loading, setLoading] = useState(false);
  const scrollRef = useRef();

  useEffect(() => {
    scrollRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  async function sendMessage() {
    if (!input.trim()) return;
    const userMsg = { role: "user", text: input };
    setMessages((m) => [...m, userMsg]);
    setLoading(true);

    try {
      // For streaming: use the SSE endpoint /query-stream (if implemented)
      const resp = await fetch("/api/query", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ question: input }),
      });
      const data = await resp.json();
      // Expected shape: { answer: "...", retrieved_docs: [...] }
      const botMsg = { role: "assistant", text: data.answer, sources: data.retrieved_docs || [] };
      setMessages((m) => [...m, botMsg]);
    } catch (err) {
      setMessages((m) => [...m, { role: "assistant", text: "Error: failed to get response." }]);
    } finally {
      setLoading(false);
      setInput("");
    }
  }

  return (
    <div className="max-w-3xl mx-auto p-4">
      <div className="border rounded p-3 h-96 overflow-auto bg-white">
        {messages.map((m, i) => (
          <div key={i} className={"my-2 " + (m.role === "user" ? "text-right" : "text-left")}>
            <div className="inline-block p-2 rounded" style={{ background: m.role==="user" ? "#e6f4ff" : "#f3f4f6" }}>
              <div className="whitespace-pre-wrap">{m.text}</div>
              {m.sources && m.sources.length > 0 && (
                <div className="mt-1 text-xs text-gray-500">Sources: {m.sources.join(", ")}</div>
              )}
            </div>
          </div>
        ))}
        <div ref={scrollRef} />
      </div>

      <div className="flex mt-3 gap-2">
        <input
          className="flex-1 border p-2 rounded"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => { if (e.key === "Enter") sendMessage(); }}
          placeholder="Ask a question..."
        />
        <button onClick={sendMessage} disabled={loading} className="px-4 py-2 bg-blue-600 text-white rounded">
          {loading ? "Thinking..." : "Send"}
        </button>
      </div>
    </div>
  );
}
```

## 4.2 `pages/api/query.js` (Next.js API route proxy)

```js
export default async function handler(req, res) {
  const backend = process.env.BACKEND_URL || "http://localhost:8000";
  try {
    const r = await fetch(`${backend}/query`, {
      method: "POST",
      headers: {"Content-Type":"application/json"},
      body: JSON.stringify({ question: req.body.question })
    });
    const data = await r.json();
    res.status(200).json(data);
  } catch (err) {
    res.status(500).json({ error: "backend error", details: err.message });
  }
}
```

## 4.3 `pages/index.js` (mount ChatBox)

```jsx
import dynamic from "next/dynamic";
const ChatBox = dynamic(() => import("../components/ChatBox"), { ssr: false });

export default function Home() {
  return (
    <main className="p-6">
      <h1 className="text-2xl font-bold">Openledger — Chat</h1>
      <ChatBox />
    </main>
  );
}
```

## 4.4 Styling & Tailwind (optional)

If you want modern UI quickly, use Tailwind. Install and configure per Next.js docs.

---

# 5 — Streaming Responses (SSE) — optional but highly recommended

**Backend:** Provide `/query-stream` SSE endpoint that yields partial tokens as they arrive from your LLM provider (or from the LLM stream).
**Frontend:** Use EventSource or fetch + ReadableStream.

### Frontend streaming example (simplified)

```js
async function streamQuery(question) {
  const resp = await fetch("/api/query-stream", {
    method: "POST",
    headers: {"Content-Type":"application/json"},
    body: JSON.stringify({ question })
  });
  const reader = resp.body.getReader();
  const decoder = new TextDecoder();
  let assistantText = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    assistantText += decoder.decode(value);
    // update UI with partial assistantText
    setPartial(assistantText);
  }
}
```

**Note:** streaming implementation must be paired with backend that flushes partial tokens.

---

# 6 — File Upload & Document Ingestion Flow

**Next.js front-end** file upload (client) POSTs to `/api/upload` (proxy) which forwards to backend `/ingest` endpoint. Example client code:

```jsx
async function uploadFile(file) {
  const fd = new FormData();
  fd.append("file", file);
  const res = await fetch("/api/upload", {
    method: "POST",
    body: fd
  });
  return await res.json();
}
```

**Backend** (FastAPI) should accept file, extract text (e.g., via `textract` or `pypdf`), chunk, embed, and store.

---

# 7 — Streamlit Rapid Prototype

Streamlit is perfect for internal demos. Below is a complete `app.py`.

```python
# app.py
import streamlit as st
import httpx
import os
from dotenv import load_dotenv
load_dotenv()

BACKEND = os.getenv("BACKEND_URL", "http://localhost:8000")

st.title("Openledger — RAG Demo (Streamlit)")

with st.sidebar:
    st.header("Upload")
    uploaded = st.file_uploader("Upload PDF or TXT", type=["pdf","txt"])
    if uploaded:
        files = {"file": (uploaded.name, uploaded.getvalue())}
        r = httpx.post(f"{BACKEND}/ingest", files=files, timeout=60)
        st.write("Ingest result:", r.json())

st.header("Chat with your docs")
q = st.text_input("Question")
if st.button("Ask"):
    if not q.strip():
        st.warning("Enter a question first.")
    else:
        with st.spinner("Querying..."):
            r = httpx.post(f"{BACKEND}/query", json={"question": q}, timeout=60)
            if r.status_code == 200:
                data = r.json()
                st.subheader("Answer")
                st.write(data.get("answer"))
                st.subheader("Sources")
                for s in data.get("retrieved_docs", []):
                    st.write("- ", s)
            else:
                st.error("Backend error")
```

Run:

```bash
streamlit run app.py
```

---

# 8 — Authentication (Basic guidance)

* Use JWT tokens (FastAPI issues tokens on `/auth/login`).
* Protect ingestion endpoints — only authenticated users can ingest.
* On the frontend, store access token in memory or `httpOnly` cookie for security.
* Example flow: Next.js page requests `/api/login`, receives Set-Cookie; subsequent calls include cookie.

---

# 9 — UI Tests & Quality

* Manual test: use Swagger UI for backend endpoints, then test Frontend flows.
* Unit tests: use React Testing Library for Next.js components.
* E2E tests: Playwright or Cypress to test upload → ingest → query flow.

---

# 10 — Exercises

### Exercise 1 — Add streaming UI

Implement the streaming frontend with `fetch` + `ReadableStream` and modify backend to stream tokens.

### Exercise 2 — Add document preview

After ingestion, allow users to view the chunk(s) that matched a query and highlight the exact substring in the text.

### Exercise 3 — Add conversation history persistence

Store conversation history server-side per user and load it when the user returns.

### Exercise 4 — Improve UX with typing indicator & partial answers

Show typing indicator while partial stream arrives and display answer progressively.

### Exercise 5 — Add rate limiting & client-side protection

Prevent excessive queries (e.g., more than 1 per second per client) and show a graceful message.

---

# 11 — Instructor Key / Hints

* For streaming, many LLM APIs provide stream endpoints. If your provider sends chunked tokens, forward them via SSE or chunked responses to the frontend.
* For highlighting sources, store `start/end` offsets with each chunk so you can show exact spans in the UI.
* When building Next.js, prefer API route proxying to avoid exposing backend URLs and to centralize authentication.
* Use optimistic UI updates: show user message immediately; append assistant message as stream arrives.
* For file upload: scan files for PII before ingestion and sanitize.

---

# 12 — Next Steps & Resources

* Connect this frontend to the Lab 05 FastAPI backend you built earlier.
* Add CI to build/deploy the Next.js app (Vercel) and Streamlit (Streamlit Cloud or internal server).
* Consider accessibility (a11y) — keyboard support for chat, color contrast for bubbles.
* For production, add monitoring for frontend errors (Sentry) and performance (Web Vitals).

---

