# Personal AI Docker — Installation Guide

A step-by-step guide to setting up a local AI environment on Windows using Docker, Ollama, and OpenClaw.

## Prerequisites

Before starting, make sure you have:

* Windows 10 or Windows 11
* Docker Desktop installed and running
* PowerShell
* Git (optional, for cloning the repository)
* NVIDIA GPU and compatible drivers (optional, for GPU acceleration)

## 1. Open the Project Directory

Open PowerShell and navigate to the project folder:

```powershell
cd C:\personal-ai-docker
```

Make sure the directory contains `docker-compose.yml` and the required configuration files.

## 2. Configure Environment Variables

Check whether the project contains a `.env` file.

If required, configure the `OPENCLAW_GATEWAY_TOKEN` variable according to the project's configuration.

**Security:** Never commit your `.env` file, API keys, or gateway tokens to GitHub.

## 3. Check the Docker Image

The OpenClaw service requires the Docker image `openclaw:local`.

Verify that the image exists:

```powershell
docker image inspect openclaw:local
```

If the image does not exist, follow the project's OpenClaw image build instructions before proceeding.

## 4. Start Ollama

Start the Ollama service:

```powershell
docker compose up -d ollama
```

Check the running containers:

```powershell
docker ps
```

## 5. Download the AI Models

Download the required models:

```powershell
docker exec personal-ai-ollama ollama pull qwen3:8b
```

```powershell
docker exec personal-ai-ollama ollama pull qwen3:8b-32k
```

```powershell
docker exec personal-ai-ollama ollama pull qwen2.5-coder:7b
```

Verify the downloaded models:

```powershell
docker exec personal-ai-ollama ollama list
```

Note: Model availability and names depend on the Ollama version and configured model registry.

## 6. Start OpenClaw

After confirming that the required Docker image and environment variables are configured, start OpenClaw:

```powershell
docker compose up -d openclaw
```

Check the container status:

```powershell
docker ps
```

## 7. Verify the Ollama Connection

Test whether OpenClaw can reach the Ollama API:

```powershell
docker exec personal-ai-openclaw node -e "fetch('http://ollama:11434/api/tags').then(r => r.json()).then(console.log).catch(console.error)"
```

If the request succeeds, the Ollama API is reachable from the OpenClaw container.

## 8. Test Model Inference

Run a test prompt:

```powershell
docker exec personal-ai-ollama ollama run qwen2.5-coder:7b "Write a Python function that adds two numbers."
```

If the model returns a response, model inference is working.

## 9. Verify OpenClaw

Check the installed version:

```powershell
docker exec personal-ai-openclaw node dist/index.js --version
```

Check the container status:

```powershell
docker inspect --format='{{.State.Status}}' personal-ai-openclaw
```

The expected status is:

```text
running
```

## 10. Optional: Visual Studio Code Integration

To use local AI models for coding:

1. Install Visual Studio Code.
2. Install the Continue extension.
3. Configure Continue to use the Ollama API.
4. Set the API endpoint to `http://127.0.0.1:11435` if this port mapping exists in your Docker Compose configuration.
5. Select an available model and test a prompt.

Verify the port mapping in `docker-compose.yml` before using this endpoint.

## Troubleshooting

### Ollama container is not running

```powershell
docker compose logs ollama
```

### OpenClaw container is not running

```powershell
docker compose logs openclaw
```

### Check all service logs

```powershell
docker compose logs
```

### Check container status

```powershell
docker compose ps
```

## Important Notes

* Docker Compose must define the expected service names.
* The `openclaw:local` image must exist before starting OpenClaw.
* Container names used in the commands must match your Docker Compose configuration.
* Persistent volumes may contain downloaded models and application data.
* Do not delete Docker volumes unless you understand that doing so may remove persistent data.
* GPU acceleration requires compatible hardware, drivers, and Docker configuration.
* The OpenClaw gateway uses container port `18789`. Publish this port in Docker Compose if you need to access the gateway from Windows.
* Keep credentials and secrets out of source control.

---

**Project:** Personal AI Docker
**Platform:** Windows + Docker
**AI Runtime:** Ollama
**AI Agent:** OpenClaw
