# Plan: Two DGX Spark LLMs for JetBrains AI Assistant (docker-compose + Gateway)

Goal: Run **two** NVIDIA NIM LLMs on DGX Spark (fast coder + general chat), expose them via one OpenAI‑compatible `/v1` endpoint, and select each model separately in JetBrains AI Assistant running via JetBrains Gateway from your Mac.

This file is the detailed implementation plan for the `spark-jetbrains-llm` repo.

---

## 1. High-Level Steps

1. Prepare DGX Spark: drivers, Docker, NGC API key.
2. Clone the repo and configure `.env`.
3. Bring up two NIM LLM services + OpenAI router with `docker-compose`.
4. Validate via `.http` files inside JetBrains IDE on the DGX.
5. Connect Mac → DGX using JetBrains Gateway.
6. Configure JetBrains AI Assistant to use the DGX endpoint `http://localhost:8000/v1`.
7. Use `fast-python-coder` and `general-chat-llm` as separate models inside the IDE.

---

## 2. Prepare DGX Spark

### 2.1. Verify GPU & Docker

SSH into DGX Spark:

```bash
nvidia-smi
docker --version
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu24.04 nvidia-smi
```

- If `nvidia-smi` or the GPU inside the container fails, fix DGX drivers / NVIDIA Container Toolkit before continuing. [web:31][web:32]

### 2.2. Configure NGC Access

NIM LLM images are pulled from NVIDIA NGC and configured via environment variables. [web:55][web:118][web:120]

1. Get your **NGC API key** from your NVIDIA account.
2. On DGX:

```bash
export NGC_API_KEY="<your-ngc-api-key>"
docker login nvcr.io   # username: $oauthtoken, password: NGC_API_KEY
```

---

## 3. Clone Repo & Configure `.env`

From DGX:

```bash
cd ~
git clone https://github.com/<you>/spark-jetbrains-llm.git
cd spark-jetbrains-llm
cp .env.example .env
```

Edit `.env`:

```env
NGC_API_KEY=YOUR_NGC_API_KEY_HERE

FAST_MODEL_URI=hf://Qwen/Qwen2.5-0.5B
GENERAL_MODEL_URI=ngc://nim/meta/llama3-8b-instruct:hf
```

- `FAST_MODEL_URI`: small, fast model for Python/code assist (low latency). [web:57][web:67]
- `GENERAL_MODEL_URI`: mid‑size instruct model for general chat / reasoning. [web:42][web:63][web:116]

You can later swap these URIs to experiment with different models.

---

## 4. docker-compose Stack (Two NIM LLMs + Router)

The compose stack is already in `docker-compose.yml`.

Key parts:

- `fast_python_coder` service: small NIM LLM.
- `general_chat_llm` service: larger NIM LLM.
- `openai_router` service: OpenAI‑compatible router that exposes both models on `:8000`. [web:107][web:121]

Important NIM env vars per service: [web:118][web:120][web:116]

- `NIM_MODEL_NAME`: where to load the model from (NGC or HF URI).
- `NIM_SERVED_MODEL_NAME`: model ID exposed via the API and `/v1/models`.
- `NIM_SERVER_PORT`: HTTP port inside the container, default 8000.

---

## 5. Bring Up / Down the Stack

From `~/spark-jetbrains-llm`:

### 5.1. Start

```bash
docker compose up -d
docker compose ps
```

You should see three containers: `fast_python_coder`, `general_chat_llm`, `spark_openai_router`.

### 5.2. Stop

```bash
docker compose down
```

This stops and removes all containers in this stack.

---

## 6. Smoke Test via .http Files (Recommended)

These steps are run from inside the JetBrains IDE backend on DGX (via Gateway).

### 6.1. Connect via JetBrains Gateway

On your Mac:

1. Open **JetBrains Gateway**. [web:138]
2. **Start New Session** → choose your IDE (IntelliJ IDEA / PyCharm).
3. Add an SSH host:
   - Host: DGX Spark hostname/IP.
   - User: your DGX username.
   - Auth: SSH key or password. [web:140][web:137]
4. When prompted, choose the project directory: `~/spark-jetbrains-llm`. [web:137][web:141]

Now the IDE UI is on your Mac, but code and terminals are on the DGX.

### 6.2. Validate Models with HTTP Requests

In the remote IDE session, open the `http/` folder.

#### 6.2.1. List models

Open `http/list-models.http`:

```http
### List models from DGX Spark router
GET http://localhost:8000/v1/models
Authorization: Bearer test-key
```

Click **Run**.

- Expected: response JSON with `data` array containing IDs:
  - `fast-python-coder`
  - `general-chat-llm`

#### 6.2.2. Test fast coder model

Open `http/test-fast-python-coder.http`:

```http
### Test fast-python-coder model
POST http://localhost:8000/v1/chat/completions
Content-Type: application/json
Authorization: Bearer test-key

{
  "model": "fast-python-coder",
  "messages": [
    {
      "role": "user",
      "content": "Write a small Python function that reverses a list."
    }
  ]
}
```

Click **Run**.

- Expected: a short Python function in the response `choices[0].message.content`.

#### 6.2.3. Test general chat model

Open `http/test-general-chat-llm.http`:

```http
### Test general-chat-llm model
POST http://localhost:8000/v1/chat/completions
Content-Type: application/json
Authorization: Bearer test-key

{
  "model": "general-chat-llm",
  "messages": [
    {
      "role": "user",
      "content": "Explain Python list comprehensions in 3 bullet points."
    }
  ]
}
```

Click **Run**.

- Expected: bullet‑style explanation of list comprehensions.

### 6.3. Optional: CLI curl Tests

If you prefer CLI, from a DGX shell:

```bash
curl http://localhost:8000/v1/models \
  -H "Authorization: Bearer test-key"
```

And replicate the `.http` POST bodies with `curl`. This is equivalent to the `.http` files.

---

## 7. Configure JetBrains AI Assistant

All of the following is done **inside the remote IDE session** on the DGX.

### 7.1. Enable AI Assistant

Make sure **AI Assistant** is installed/enabled:

- Check `Settings | Plugins` and search for “AI Assistant”. [web:94][web:101]

### 7.2. Add DGX as an OpenAI-Compatible Provider

1. Open `Settings | Tools | AI Assistant | Models & API keys`. [web:27][web:95]
2. Under **Third‑party AI providers**, click **Add**.
3. Set:
   - Provider type: **OpenAI‑compatible**. [web:27]
   - Endpoint URL: `http://localhost:8000/v1`
   - API key: `test-key` (or any token; must match what you send to the router).
4. Click **Test connection**; it should succeed and list both models. [web:27][web:95]
5. Click **Apply**.

---

## 8. Use Both Models in AI Assistant

### 8.1. In AI Chat

1. Open **AI Chat** in the IDE. [web:88]
2. In the **model dropdown**:
   - Select `fast-python-coder` for snappy coding and refactors.
   - Select `general-chat-llm` for heavier reasoning / design discussions. [web:27][web:88]

### 8.2. Models Assignment

In `Settings | Tools | AI Assistant | Models & API keys`:

- Use **Models assignment** (or similar section) to map features: [web:27][web:95]
  - Assign `fast-python-coder` to **code completion / core**.
  - Assign `general-chat-llm` to **chat / refactoring / documentation**.

Now JetBrains AI Assistant is effectively backed by two DGX Spark LLMs, each selectable per feature and in the chat dropdown.

---

## 9. Daily Operational Flow

On DGX:

```bash
cd ~/spark-jetbrains-llm
docker compose up -d
```

On Mac:

1. Open JetBrains Gateway.
2. Connect to the DGX host.
3. Open the `spark-jetbrains-llm` project.
4. Code with AI Assistant:
   - Use the model dropdown to pick `fast-python-coder` or `general-chat-llm`.
   - Use `.http` files in `http/` to troubleshoot or validate models if needed.

Stopping stack at the end of the day (DGX):

```bash
cd ~/spark-jetbrains-llm
docker compose down
```

---

## 10. Extensions and Variants

- **More models**:  
  - Add more NIM services in `docker-compose.yml`.
  - Add corresponding entries in `router-config.json` (`providers` and `model_map`).
  - They will show up as additional models in AI Assistant. [web:107][web:121]

- **Different backends**:
  - Swap NIM for vLLM, Ollama, or LM Studio as long as:
    - They expose OpenAI‑style `/v1` endpoints.
    - `/v1/models` lists all model IDs you want available. [web:106][web:111][web:27]

- **Medium article structure**:
  - Use `README.md` sections as article headings:
    - “Turning DGX Spark into a Personal AI Workstation”
    - “Two Local LLMs via Docker Compose”
    - “Remote Development with JetBrains Gateway”
    - “Using Local Models from JetBrains AI Assistant”
  - Include screenshots of:
    - Gateway connection screen. [web:137][web:138]
    - AI Assistant models screen showing `http://localhost:8000/v1` provider. [web:27][web:95]
    - AI Chat dropdown listing `fast-python-coder` and `general-chat-llm`. [web:88]

This plan, plus the repo files, gives you a complete, reproducible story for DGX Spark + JetBrains AI Assistant with two local LLMs.
