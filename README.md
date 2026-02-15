# DGX Spark + JetBrains AI Assistant: Two Local LLMs over SSH

Use an NVIDIA DGX Spark as a **remote AI workstation** for JetBrains IDEs:

- Two NVIDIA NIM LLMs (fast coder + general chat) run on the Spark.
- A small OpenAI‑compatible router exposes both under one `/v1` API.
- JetBrains Gateway connects from your Mac to the DGX, and AI Assistant sees both models as local choices.

---

## 1. Architecture

- **DGX Spark (remote)**  
  - Runs Docker + NVIDIA Container Toolkit. [web:31][web:32]  
  - `docker-compose` stack:
    - `fast_python_coder` (small NIM LLM for low‑latency Python/code assist). [web:118][web:120]
    - `general_chat_llm` (larger NIM LLM for reasoning/chat). [web:42][web:116]
    - `openai_router` (OpenAI‑compatible proxy exposing both models via `/v1`). [web:107][web:121]

- **Mac (local)**  
  - Runs JetBrains Gateway + JetBrains IDE frontend. [web:138][web:141]  
  - Opens the project directly on DGX via Gateway remote development. [web:137]  
  - AI Assistant uses `http://localhost:8000/v1` on the DGX backend and lists both models separately. [web:27][web:95][web:88]

---

## 2. Prerequisites

### On DGX Spark

- DGX OS with NVIDIA drivers and CUDA working:
  - `nvidia-smi` should succeed. [web:31][web:32]
- Docker + NVIDIA Container Toolkit installed:
  - `docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu24.04 nvidia-smi` should work. [web:31]
- NGC credentials:
  - An **NGC API key** to pull NIM images. [web:55][web:118]

### On Mac

- JetBrains Gateway installed. [web:138][web:141]
- A recent JetBrains IDE (IntelliJ IDEA, PyCharm, etc.) with **AI Assistant** available. [web:94][web:101]
- SSH access to the DGX Spark (key or password). [web:140]

---

## 3. Repo Layout

Suggested layout for this repo:

```text
spark-jetbrains-llm/
  docker-compose.yml
  router-config.json
  .env.example
  plan.md
  README.md
  http/
    list-models.http
    test-fast-python-coder.http
    test-general-chat-llm.http
```

- `README.md`: high‑level overview (good for Medium readers).
- `plan.md`: detailed operational steps.
- `http/*.http`: JetBrains HTTP client requests for testing the stack.

---

## 4. Clone & Configure on DGX Spark

SSH into the DGX Spark:

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

- `FAST_MODEL_URI`: small, fast model for coding (good for Python code assist). [web:57][web:67]
- `GENERAL_MODEL_URI`: 7–8B instruct model from NIM for general chat. [web:42][web:63][web:116]

> Users can adjust these URIs to any NIM‑supported models later.

---

## 5. docker-compose Stack

### 5.1. `docker-compose.yml`

```yaml
version: "3.9"

services:
  fast_python_coder:
    image: nvcr.io/nim/nvidia/llm-nim:latest
    container_name: fast_python_coder
    restart: unless-stopped
    environment:
      NGC_API_KEY: "${NGC_API_KEY}"
      NIM_MODEL_NAME: "${FAST_MODEL_URI}"
      NIM_MANIFEST_ALLOW_UNSAFE: "1"
      NIM_SERVED_MODEL_NAME: "fast-python-coder"
      NIM_SERVER_PORT: "8000"
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    ports:
      - "8101:8000"

  general_chat_llm:
    image: nvcr.io/nim/nvidia/llm-nim:latest
    container_name: general_chat_llm
    restart: unless-stopped
    environment:
      NGC_API_KEY: "${NGC_API_KEY}"
      NIM_MODEL_NAME: "${GENERAL_MODEL_URI}"
      NIM_MANIFEST_ALLOW_UNSAFE: "1"
      NIM_SERVED_MODEL_NAME: "general-chat-llm"
      NIM_SERVER_PORT: "8000"
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: ["gpu"]
    ports:
      - "8102:8000"

  openai_router:
    image: ghcr.io/xenova/openai-router:latest
    container_name: spark_openai_router
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      OPENAI_ROUTER_CONFIG: /app/config/router-config.json
    volumes:
      - ./router-config.json:/app/config/router-config.json:ro
```

Key NIM configuration points: [web:118][web:120][web:116]

- `NIM_MODEL_NAME`: model location (NGC or HF URI).
- `NIM_SERVED_MODEL_NAME`: model name exposed through the API.
- `NIM_SERVER_PORT`: HTTP port inside the container (default 8000).

### 5.2. `router-config.json`

```json
{
  "default_provider": "fast_python_coder",
  "providers": {
    "fast_python_coder": {
      "type": "openai",
      "base_url": "http://fast_python_coder:8000/v1",
      "api_key": "dummy-key"
    },
    "general_chat_llm": {
      "type": "openai",
      "base_url": "http://general_chat_llm:8000/v1",
      "api_key": "dummy-key"
    }
  },
  "model_map": {
    "fast-python-coder": "fast_python_coder",
    "general-chat-llm": "general_chat_llm"
  }
}
```

This router:

- Listens on `:8000` and speaks OpenAI‑style API (`/v1/models`, `/v1/chat/completions`). [web:107][web:121]
- Routes by `"model"`:
  - `fast-python-coder` → `fast_python_coder` NIM container.
  - `general-chat-llm` → `general_chat_llm` NIM container.

---

## 6. Smoke Test using IntelliJ .http Files

This repo includes HTTP client files under `http/` that you can run directly from the JetBrains IDE (Gateway session) instead of using `curl`.

```text
spark-jetbrains-llm/
  http/
    list-models.http
    test-fast-python-coder.http
    test-general-chat-llm.http
```

### 6.1. HTTP files

`http/list-models.http`:

```http
### List models from DGX Spark router
GET http://localhost:8000/v1/models
Authorization: Bearer test-key
```

`http/test-fast-python-coder.http`:

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

`http/test-general-chat-llm.http`:

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

### 6.2. How to run them

Once `docker compose up -d` is running on the DGX:

1. Connect via JetBrains Gateway to the DGX and open this repo. [web:137][web:141]
2. In the IDE, open `http/list-models.http` and click **Run**.
   - You should see both `fast-python-coder` and `general-chat-llm` in the JSON response.
3. Open `http/test-fast-python-coder.http` and run it.
   - Expect a short Python function that reverses a list.
4. Open `http/test-general-chat-llm.http` and run it.
   - Expect a brief explanation of Python list comprehensions in bullet form.

These `.http` files are part of the repo, so anyone cloning it can validate the stack entirely from the IDE.

---

## 7. Start / Stop the Stack on DGX

From the DGX (in repo root):

```bash
docker compose up -d
docker compose ps
```

To stop:

```bash
docker compose down
```

This controls both LLMs and the router in one command.

---

## 8. Connect from Mac via JetBrains Gateway

On your Mac:

1. Install **JetBrains Gateway**. [web:138]
2. Launch Gateway → **Start New Session** → choose your IDE (IntelliJ IDEA / PyCharm).
3. Add an SSH host:
   - Host: DGX Spark hostname or IP.
   - User: your Linux username on DGX.
   - Auth: SSH key or password. [web:140][web:137]
4. Gateway will:
   - Connect to the DGX.
   - Install the IDE backend there if needed.
   - Let you choose the project directory on DGX; select `~/spark-jetbrains-llm`. [web:137][web:141]

Now your IDE UI runs on the Mac, but all files, terminals, and HTTP requests are executed on the DGX.

---

## 9. Configure JetBrains AI Assistant (inside remote IDE)

Inside the remote IDE session:

1. Make sure **AI Assistant** is installed/enabled. [web:94][web:101]
2. Open **Settings** → `Tools | AI Assistant | Models & API keys`. [web:27][web:95]

Add your DGX Spark as an **OpenAI‑compatible** provider:

- Provider type: **OpenAI‑compatible**. [web:27]
- Endpoint URL: `http://localhost:8000/v1`
  - From the remote IDE’s perspective, `localhost` is the DGX, and port 8000 is the router.
- API key: `test-key` (or any token; must match the `Authorization` header you send).
- Click **Test Connection** → it should succeed and list both models. [web:27][web:95]
- Click **Apply**.

AI Assistant now knows about both NIM‑backed models via your DGX Spark.

---

## 10. Use the Two DGX Models in AI Assistant

In your JetBrains IDE (remote session):

- Open **AI Chat**. [web:88]
- Use the **model dropdown** in the chat panel. You should see:
  - `fast-python-coder` (low‑latency inline coding help).
  - `general-chat-llm` (heavier reasoning / chat). [web:27][web:88]

In **Models & API keys → Models assignment**:

- Map `fast-python-coder` → **Code completion / core**.
- Map `general-chat-llm` → **Chat / Refactoring / Docs**. [web:27][web:95]

From this point, switching models is just a dropdown change; both are served from your DGX Spark.

---

## 11. Daily Workflow

On DGX:

```bash
cd ~/spark-jetbrains-llm
docker compose up -d
```

On Mac:

1. Open JetBrains Gateway.
2. Connect to DGX → open the `spark-jetbrains-llm` project.
3. Use AI Chat / inline completion and select either `fast-python-coder` or `general-chat-llm` in the model dropdown.

Everything—source code, Docker, and LLM inference—runs on the DGX; you only see the UI.

---

## 12. Customization Ideas

- Swap models via `.env` without touching YAML. [web:118][web:120]
- Add more LLM containers + update `router-config.json` to expose more model IDs.
- Replace NIM with vLLM or Ollama as long as they expose an OpenAI‑style `/v1` API and list multiple models in `/v1/models`. [web:106][web:111][web:27]
- Extend the repo with:
  - Example JetBrains AI Assistant settings screenshots.
  - A `Medium.md` outline that mirrors this README for your article.
