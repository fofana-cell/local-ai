# My Local AI Assistant

A personal AI that runs **on my own computer**, inside **Docker**, with small open-source models. It works in **VS Code**, on a **simple website**, and through the **Hermes Agent**. After the first download it works **without internet**.

## What is inside

```
VS Code (Continue)  ─┐
Website (chat.html) ─┼──►  Ollama (in Docker)  ──►  models
Hermes Agent (Docker)┘
```

| Part | What it does |
|---|---|
| **Docker** | Keeps everything in its own boxes, easy to start and stop |
| **Ollama** | The program that runs the AI models |
| **Continue** (VS Code extension) | The AI assistant inside VS Code |
| **chat.html** | A simple website where I ask questions and the AI answers |
| **Hermes Agent** (Nous Research) | An autonomous agent that runs in Docker and talks to my Ollama |
| **Open WebUI** (optional) | A ready-made chat dashboard |

## The models

| Model | Job | Size |
|---|---|---|
| `qwen2.5-coder:3b` | Chat and code help in VS Code | 1.9 GB |
| `qwen2.5-coder:1.5b` | Fast auto-complete while typing | 986 MB |
| `nomic-embed-text` | Helps the AI search my project files | 274 MB |
| `llama3.2:3b` | Used by Hermes (it needs a 64K context window and tool support) | 2 GB |

I chose small models because my laptop has 15 GB of RAM and no free GPU memory (4 GB).

## What you need

- Windows with **8 GB RAM** or more (more is better)
- About **12 GB** of free disk space
- **Docker Desktop**
- **VS Code**

## How to set it up

### 1. Make the Docker file

Make a folder called `my-ai`. Inside it, make a file called `docker-compose.yml`:

```yaml
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    ports:
      - "11434:11434"
    environment:
      - OLLAMA_ORIGINS=*
      - OLLAMA_CONTEXT_LENGTH=65536
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_KV_CACHE_TYPE=q8_0
      - OLLAMA_KEEP_ALIVE=2h
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open_webui_data:/app/backend/data
    depends_on:
      - ollama
    restart: unless-stopped

  hermes:
    image: nousresearch/hermes-agent:latest
    profiles: ["agent"]
    stdin_open: true
    tty: true
    volumes:
      - hermes_data:/opt/data
    depends_on:
      - ollama

volumes:
  ollama_data:
  open_webui_data:
  hermes_data:
```

### 2. Start it

```powershell
cd $HOME\my-ai
docker compose up -d
```

Open http://localhost:11434 in a browser. It should say **"Ollama is running"**.

### 3. Download the models

```powershell
docker exec -it ollama ollama pull qwen2.5-coder:3b
docker exec -it ollama ollama pull qwen2.5-coder:1.5b
docker exec -it ollama ollama pull nomic-embed-text
docker exec -it ollama ollama pull llama3.2:3b
docker exec -it ollama ollama list
```

### 4. Connect VS Code

1. In VS Code, install the **Continue** extension.
2. Open Continue, click the gear icon, and open `config.yaml`.
3. Replace everything with this and save:

```yaml
name: Local Assistant
version: 1.0.0
schema: v1

models:
  - name: Qwen Coder 3B
    provider: ollama
    model: qwen2.5-coder:3b
    apiBase: http://localhost:11434
    roles:
      - chat
      - edit
      - apply

  - name: Qwen Coder 1.5B
    provider: ollama
    model: qwen2.5-coder:1.5b
    apiBase: http://localhost:11434
    roles:
      - autocomplete

  - name: Nomic Embed
    provider: ollama
    model: nomic-embed-text
    apiBase: http://localhost:11434
    roles:
      - embed
```

### 5. Open the website

```powershell
start $HOME\my-ai\chat.html
```

Pick a model, type a question, and press Enter.

### 6. Hermes Agent (in Docker)

First time only, set it up:

```powershell
docker compose run --rm hermes
```

In the setup menu choose:

- Provider: **Custom endpoint (enter URL manually)**
- URL: `http://ollama:11434/v1` (inside Docker the name is `ollama`, not `localhost`)
- Model: `llama3.2:3b`
- API mode: chat completions
- Terminal backend: local
- Telemetry: **No**

Ask one question and exit:

```powershell
docker compose run --rm hermes chat --oneshot -Q --max-turns 1 --ignore-rules -q "say hi"
```

To prove Hermes talks to my local model, look at the Ollama log. It shows the requests from the Hermes container:

```powershell
docker logs ollama --tail 8
```

## How to use it

- **Chat in VS Code:** click the Continue icon, use **Chat** mode, and ask a short, clear question.
- **Explain or fix code:** select code, press `Ctrl + L`, and ask.
- **Edit code:** select code, press `Ctrl + I`, and say what to change.
- **Auto-complete:** just type, then press `Tab` to accept the grey suggestion.
- **Website:** open `chat.html` and ask any question.

## Test that it works without internet

1. Ask a question on the website.
2. Turn off Wi-Fi.
3. Ask another question. It still answers.

## Start and stop

```powershell
cd $HOME\my-ai
docker compose stop      # stop the AI
docker compose start     # start it again
```

Do not delete the Docker volumes. The models are saved there, and you would need to download them again.

## Problems I found and how I fixed them

- **Agent mode in Continue did not work** with the small model, so I use **Chat** mode.
- **Open WebUI sometimes printed a tool request** (like `{"name": ...}`) instead of an answer. Small models are weak at tools, so I made my own simple page, `chat.html`, that talks straight to Ollama.
- **Hermes refused `qwen2.5-coder:3b`**, because that model has a 32K context window and Hermes needs at least 64K. I switched Hermes to `llama3.2:3b` and set `OLLAMA_CONTEXT_LENGTH=65536`.
- **Inside Docker, `localhost` does not work for Hermes.** The Hermes container must use `http://ollama:11434/v1`.

## Limits (honest notes)

- **Hermes is slow on my laptop.** It runs in Docker and reaches my local Ollama (the Ollama log shows the requests). But Ollama is using the **CPU only**, and the Hermes prompt is about 6,000 tokens. Reading it takes several minutes, and Hermes closes the connection before the first answer arrives. A GPU, more RAM, or a faster computer would fix this. The model itself answers fine when called directly.
- Small models make more mistakes than big cloud AI.
- The first answer after starting can take 10-30 seconds, because the model has to load.
- Internet is needed once, for the downloads.

## Future work

- Use my NVIDIA GPU (RTX 3050, 4 GB) with Docker to make answers much faster.
- Try a stronger model on a computer with more RAM.
- Try OpenClaw as another agent on top of Ollama.

## Built with

I did not build the AI. I installed and connected open-source tools: **Docker**, **Ollama**, the **Qwen2.5-Coder**, **Nomic Embed** and **Llama 3.2** models, the **Continue** extension, and **Hermes Agent**. My work was setting them up, connecting them, fixing the problems above, and making the website.
