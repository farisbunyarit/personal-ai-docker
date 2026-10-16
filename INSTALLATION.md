# Installation Guide — Personal AI Docker

This guide explains how to build a local AI environment similar to the one documented in Personal AI Docker.

The reference setup runs Ollama and OpenClaw in separate Docker containers on Windows using Docker Desktop. Ollama manages the local models, while OpenClaw connects to Ollama through Docker networking.

**Important:** This guide describes how to create your own environment. It does not require access to the author's private configuration, Docker volumes, or local files.

## 1. Reference Environment

The original environment documented by this project contains:

| Component             | Reference configuration      |
| --------------------- | ---------------------------- |
| Host OS               | Windows                      |
| Container runtime     | Docker Desktop               |
| Ollama image          | `ollama/ollama:latest`       |
| OpenClaw image        | `openclaw:local`             |
| OpenClaw version      | `2026.9.6`                   |
| Ollama host port      | `11435`                      |
| Ollama container port | `11434`                      |
| OpenClaw gateway port | `18789` inside the container |

### Models in the Reference Environment

| Model              | Reported size |
| ------------------ | ------------: |
| `qwen3:8b`         |        5.2 GB |
| `qwen3:8b-32k`     |        5.2 GB |
| `qwen2.5-coder:7b` |        4.7 GB |

These models were observed in the original environment. Their availability and exact reproducibility on another machine must be verified separately.

## 2. Prerequisites

Install the following:

* Windows.
* Docker Desktop with Docker Compose.
* Git.
* PowerShell.
* Sufficient RAM and disk space for the selected models and containers.

An NVIDIA GPU is used as a GPU reservation in the reference Compose configuration. GPU operation requires compatible hardware, drivers, and Docker Desktop support.

Verify the installation:

```powershell
docker --version
docker compose version
docker info
```

## 3. Prepare a New Environment

Create a separate directory for your own environment. Do not use an existing source-code directory or overwrite another project.

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\personal-ai"
Set-Location "$HOME\personal-ai"
```

Download the official Ollama image:

```powershell
docker pull ollama/ollama:latest
```

Create persistent storage for the models:

```powershell
docker volume create personal-ai-models
```

## 4. Start Ollama

Start Ollama in Docker and expose its API on Windows port `11435`:

```powershell
docker run -d `
  --name personal-ai-ollama `
  --restart unless-stopped `
  -p 11435:11434 `
  -v personal-ai-models:/root/.ollama `
  ollama/ollama:latest
```

Check the container:

```powershell
docker ps
docker logs --tail 50 personal-ai-ollama
```

## 5. Download and Verify Models

Download the standard model tags used by the reference setup:

```powershell
docker exec personal-ai-ollama ollama pull qwen3:8b
docker exec personal-ai-ollama ollama pull qwen2.5-coder:7b
```

List the installed models:

```powershell
docker exec personal-ai-ollama ollama list
```

Test inference:

```powershell
docker exec personal-ai-ollama ollama run qwen3:8b "Explain what Docker is."
```

### About `qwen3:8b-32k`

The reference environment contains a model registered under this identifier. Do not assume that the same tag can be pulled directly on a new installation.

Verify the model's origin, exact identifier, and reproducible installation method before attempting to recreate it.

## 6. Verify the Ollama API

From Windows PowerShell:

```powershell
Invoke-RestMethod http://127.0.0.1:11435/api/tags
```

A successful response should include the models available in Ollama.

Check the port mapping:

```powershell
docker port personal-ai-ollama
```

The expected host-to-container mapping is `11435` to `11434`.

## 7. Build OpenClaw

The reference environment uses a locally built image named `openclaw:local`. Its image metadata identifies the upstream source repository:

https://github.com/openclaw/openclaw

Official Docker installation instructions:

https://docs.openclaw.ai/install/docker

Obtain the upstream source in a separate directory:

```powershell
git clone https://github.com/openclaw/openclaw.git "$HOME\openclaw-source"
Set-Location "$HOME\openclaw-source"
```

Inspect the source checkout and its Docker instructions:

```powershell
Get-ChildItem -Force
Get-Content Dockerfile
```

The original reference image was built from source revision:

`eb377ac59e6c9fd6c7705028034812becf00271b`

For closer reproducibility, check out the intended revision in this separate source directory:

```powershell
git checkout eb377ac59e6c9fd6c7705028034812becf00271b
```

Build the image using the Dockerfile and documented build options for that revision:

```powershell
docker build -t openclaw:local .
```

**Reproducibility note:** The source revision alone does not guarantee a byte-for-byte identical image. Build arguments, dependencies, custom patches, and configuration may affect the result. The command above is a general build example, not a verified reproduction of the original image.

## 8. Configure OpenClaw to Use Ollama

OpenClaw and Ollama must be able to communicate over a shared Docker network.

The reference configuration uses this internal Ollama endpoint:

```text
http://ollama:11434
```

The hostname `ollama` must resolve to the Ollama service on the shared network.

Do not use `127.0.0.1:11435` as the Ollama endpoint inside the OpenClaw container. That address refers to the container itself.

Configure the Ollama provider, model identifiers, and default agent model using the official OpenClaw configuration format for the selected version.

The reference environment uses:

* `ollama/qwen3:8b`
* `ollama/qwen3:8b-32k`
* `ollama/qwen2.5-coder:7b`

Its default agent model is `ollama/qwen2.5-coder:7b`, with a configured context setting of `32768` tokens.

## 9. Docker Compose and Persistent Storage

The reference Compose configuration defines two services:

* `ollama`
* `openclaw`

It also defines three named volumes:

| Volume                  | Purpose                         |
| ----------------------- | ------------------------------- |
| `personal-ai-models`    | Ollama models and data          |
| `personal-ai-openclaw`  | OpenClaw configuration and data |
| `personal-ai-workspace` | OpenClaw workspace              |

OpenClaw's gateway listens on port `18789` inside the container. The reference Compose file does not publish this port to Windows.

Before starting a Compose project, validate its configuration:

```powershell
docker compose config
```

Start the services only after preparing a valid Compose file, the required image, and the necessary environment variables:

```powershell
docker compose up -d
```

The reference Compose file uses `openclaw:local` without a `build:` section. Therefore, that image must exist locally before the OpenClaw service can start.

## 10. Verify the Environment

### Check Running Containers

```powershell
docker ps
```

### Check All Containers

```powershell
docker ps -a
```

### List Ollama Models

```powershell
docker exec personal-ai-ollama ollama list
```

### Test the Ollama API

```powershell
Invoke-RestMethod http://127.0.0.1:11435/api/tags
```

### Check the OpenClaw Version

```powershell
docker exec personal-ai-openclaw node dist/index.js --version
```

### Check OpenClaw Container State

```powershell
docker inspect --format='{{.State.Status}}' personal-ai-openclaw
```

The expected container state is `running`.

### Inspect Container Logs

```powershell
docker logs --tail 100 personal-ai-ollama
docker logs --tail 100 personal-ai-openclaw
```

### Inspect Persistent Volumes

```powershell
docker volume ls
docker volume inspect personal-ai-models
```

A running container alone does not prove that the model provider, agent workflow, or every integration is working. Test each component separately.

## 11. Security and Maintenance

* Never commit `.env` files containing secrets.
* Never publish gateway tokens or private credentials.
* Review `.gitignore` before committing changes.
* Restrict access to agent tools and workspace operations.
* Back up important Docker volumes before destructive maintenance.
* Do not remove named volumes as a routine troubleshooting step.
* Rotate credentials that may have been exposed.
* Review the official documentation before upgrading OpenClaw or changing its image.

## 12. Official References

* Docker: https://docs.docker.com/
* Docker Compose: https://docs.docker.com/compose/
* Ollama: https://docs.ollama.com/
* Ollama model library: https://ollama.com/library
* OpenClaw source: https://github.com/openclaw/openclaw
* OpenClaw Docker installation: https://docs.openclaw.ai/install/docker
* OpenClaw documentation: https://docs.openclaw.ai
* Qwen3 models: https://ollama.com/library/qwen3
* Qwen2.5 Coder models: https://ollama.com/library/qwen2.5-coder
