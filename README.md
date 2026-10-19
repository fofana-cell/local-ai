# My Local AI Assistant

A private AI that runs on my own computer, in Docker, with small open-source models.
It works in VS Code (Continue extension), on a simple website (chat.html), and through
the Hermes Agent (runs in Docker and talks to my local Ollama). It works without
internet after the first download.

## Models
- qwen2.5-coder:3b - chat and code help
- qwen2.5-coder:1.5b - fast auto-complete
- nomic-embed-text - file search
- llama3.2:3b - used by Hermes (needs a 64K context window)

## How to run
cd my-ai
docker compose up -d
Then open chat.html in a browser.

## Hermes Agent
docker compose run --rm hermes

## Limits
Hermes reaches my local Ollama, but my laptop uses only the CPU, so Hermes is slow
and can time out before the first answer. A GPU or a faster computer would fix this.
The models answer fine in VS Code and on the website.
