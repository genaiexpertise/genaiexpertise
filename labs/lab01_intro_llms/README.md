
---

# Lab overview & prerequisites (before you start)

* OS: Linux / macOS recommended (Windows WSL works too).
* Disk: 20+ GB free (models can be large).
* RAM & GPU:

  * CPU-only: possible for small/quantized models (slower).
  * GPU: recommended for reasonable latency when running larger models locally (NVIDIA + CUDA for many runtimes).
* Tools: `git`, `curl`/`wget`, `python3` (>=3.8), `pip`, optionally `cmake` and `build-essential`.
* Accounts: (optional) Hugging Face for model downloads that require access, and provider accounts for API keys (OpenAI, Anthropic).
* Security: never paste API keys into public repos. Use environment variables or secret managers.

---

## Part A — Run your first LLM locally

Two practical local approaches (pick one):

### Option A1 — Quick: `llama.cpp` (LLaMA-family / GGML quantized models)

`llama.cpp` runs quantized ggml versions of LLaMA-family models on CPU (fast enough for small/quantized models). This is the easiest path to get local LLM inference.

1. Clone and build `llama.cpp`

```bash
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
make
```

2. Obtain a GGML model file

* Download a compatible `ggml` quantized model (e.g., `ggml-model-q4_0.bin`) — many LLaMA-family models in GGML format are distributed via Hugging Face; follow license/terms. Put it in `~/models/llama/` and name it e.g. `ggml-model.bin`.

3. Run an interactive REPL

```bash
# example: replace path with your model file
./main -m ~/models/llama/ggml-model.bin -p "Hello, what's your name?" -t 8
```

* `-p` supplies a prompt.
* `-t` threads.

4. Try interactive chat

```bash
./main -m ~/models/llama/ggml-model.bin --interactive
# then type queries at the prompt
```

**Expected result:** the model will print generated tokens to your terminal. If slow, reduce model size or use fewer threads.

**Tips & troubleshooting**

* If `make` fails, install build tools: `sudo apt install build-essential cmake`.
* If model download is blocked, ensure you have access rights on the host (Hugging Face token sometimes required).
* Quantized models are faster and much smaller; prefer them for CPU-only runs.

---

### Option A2 — More featureful: `text-generation-webui` or `vLLM` backend (good for Mistral-style models)

`text-generation-webui` provides a web interface and supports many model formats and GPU inference. `vLLM` offers fast GPU inference but needs more setup.

**Example with text-generation-webui (GPU recommended):**

1. Clone repo and create venv

```bash
git clone https://github.com/oobabooga/text-generation-webui.git
cd text-generation-webui
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

2. Download a model (Hugging Face) and put it under `models/<model-name>/`. Example names: `mistral-7b-v0` or `llama-2-7b` (subject to licensing and access).

3. Launch web UI

```bash
python server.py --model <model-folder-name> --gpu-id 0
```

4. Open `http://localhost:7860` in your browser, enter prompts and interact.

**Expected result:** a browser UI where you type prompts and see streaming token output.

**Tips & troubleshooting**

* GPU drivers & CUDA must match the runtime’s expected versions.
* For CPU-only, run with `--no-stream` or using a small quantized model.
* vLLM requires specific versions of CUDA and additional setup.

---

## Part B — Make your first API call to an LLM provider

> **Important:** store API keys in environment variables (never commit them).

### 1) OpenAI (Python example using `openai` library)

1. Install OpenAI client:

```bash
pip install openai
```

2. Export your API key:

```bash
export OPENAI_API_KEY="sk-REPLACE_ME"
```

3. Simple Python script (`openai_test.py`):

```python
import os
import openai

openai.api_key = os.environ["OPENAI_API_KEY"]

resp = openai.ChatCompletion.create(
    model="gpt-4o-mini",  # replace with your available model
    messages=[
        {"role":"system","content":"You are a helpful assistant."},
        {"role":"user","content":"Give me a one-paragraph summary of Retrieval-Augmented Generation."}
    ],
    max_tokens=200,
    temperature=0.2,
)
print(resp['choices'][0]['message']['content'])
```

4. Run:

```bash
python openai_test.py
```

**Expected:** a concise paragraph summarizing RAG.

**Note:** model names / availability vary by account; if `gpt-4o-mini` isn't available, use `gpt-4o`, `gpt-4o-mini`, or `gpt-3.5-turbo` — check your provider dashboard.

---

### 2) Anthropic (Claude) — HTTP / Python example

1. Install requests:

```bash
pip install requests
```

2. Set key:

```bash
export ANTHROPIC_API_KEY="claude-REPLACE_ME"
```

3. Python example:

```python
import os, requests, json

API_KEY = os.environ["ANTHROPIC_API_KEY"]
url = "https://api.anthropic.com/v1/complete"  # endpoint may vary with provider docs

payload = {
    "model": "claude-2.1", 
    "prompt": "You are a helpful assistant. Explain in one paragraph what a transformer is.",
    "max_tokens_to_sample": 200
}

headers = {
    "x-api-key": API_KEY,
    "Content-Type": "application/json"
}

r = requests.post(url, headers=headers, json=payload)
print(r.status_code, r.text)
```

**Expected:** a JSON response containing the generated text.

**Note:** Anthropic’s API shape or headers may differ; consult your account docs. Use environment variables to keep keys secret.

---

## Interactive exercises (do these during the lab)

1. **Local LLM**

   * Run `llama.cpp` interactive mode.
   * Prompt: “Explain Retrieval-Augmented Generation in three bullets.”
   * Modify prompt temperature/seed (if supported) and observe changes.

2. **Web UI**

   * Using text-generation-webui, compare outputs for the same prompt across two models (e.g., one LLaMA-derived and one Mistral-style model). Note differences in tone and factuality.

3. **API call**

   * Call OpenAI / Anthropic with the same RAG explanation prompt. Compare answers to your local model and note hallucinations, length, and correctness.

4. **Streaming**

   * If your local runtime or web UI supports streaming, observe the token-by-token output and measure initial latency (time to first token).

5. **Embedding / RAG mini-test**

   * (Optional) Use an embeddings endpoint from OpenAI to embed two sentences and compute cosine similarity locally to see semantic closeness.

---

## Security, cost & ethical considerations (must-know)

* **Cost:** Provider API calls cost money and tokens; monitor usage. Try low `max_tokens` and low `temperature` for labs.
* **Data privacy:** Don’t send sensitive PII in prompts when using third-party APIs unless you understand the provider’s data usage policy.
* **Licensing:** Some models (LLaMA, Mistral, etc.) require following specific license terms — review them before distribution.
* **Rate limits:** APIs may throttle; handle retries gracefully.
* **Hallucinations:** Local smaller models often hallucinate or be less factual — verify outputs.

---

## Common troubleshooting & tips

* **Slow inference (local):** use quantized models (ggml q4), increase threads, or use GPU runtime.
* **Build fails:** ensure dependencies (`cmake`, `build-essential`, CUDA drivers) are installed.
* **Model not found:** confirm model path and file name; ensure you have access and accepted model license on Hugging Face if required.
* **Auth errors (API):** check environment variables and that key has required permissions.
* **Firewall errors:** ensure ports are open (e.g., for web UI `7860`) and not blocked by local firewall.

---

## Lab wrap-up tasks (deliverables)

1. Screenshot or short screen recording of:

   * Local model interactive session OR text-generation-webui response.
   * Successful OpenAI/Anthropic API response printed in terminal.
2. Short note (~100–200 words) comparing:

   * Latency, quality, and usefulness of local model vs hosted API for the same prompt.
3. Optional: paste the small script you ran and point out the environment variables you used (no API keys).

---

