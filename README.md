# Personal AI Docker

A local AI environment powered by Docker, Ollama, OpenClaw, and optional VS Code integration with Continue.

## Overview

This project runs local AI models through Ollama inside Docker and connects them to OpenClaw for AI agent workflows.

Continue can optionally connect VS Code to Ollama for AI-assisted coding.

### Main Components

* **Docker Desktop:** Runs and manages containers on Windows.
* **Ollama:** Runs and manages local AI models inside Docker.
* **OpenClaw:** Provides AI agent functionality using configured Ollama models.
* **Continue:** Optional VS Code extension for AI-assisted development.

### Available Models

The current Ollama environment contains:

| Model              |   Size |
| ------------------ | -----: |
| `qwen3:8b`         | 5.2 GB |
| `qwen3:8b-32k`     | 5.2 GB |
| `qwen2.5-coder:7b` | 4.7 GB |

The sizes shown are reported by Ollama and do not represent actual runtime memory usage.

## Architecture

The current environment uses two Docker containers. Ollama manages all three AI models inside its container, while OpenClaw runs separately and connects to Ollama through Docker networking.

### System Overview

```text
Windows Host
└── Docker Desktop
    ├── Ollama Container
    │   ├── qwen3:8b
    │   ├── qwen3:8b-32k
    │   └── qwen2.5-coder:7b
    │
    └── OpenClaw Container
        └── Ollama API
            └── http://ollama:11434

Optional Windows Development Tools
├── Visual Studio Code
└── Continue Extension
    └── http://127.0.0.1:11435
```

### Network Connections

* **OpenClaw → Ollama:** Uses the internal Docker endpoint `http://ollama:11434`.
* **Windows → Ollama:** Uses `http://127.0.0.1:11435`.
* **Port mapping:** Docker maps host port `11435` to container port `11434`.
* **OpenClaw gateway:** Configured to listen on port `18789` inside its container.

The current Compose configuration does not publish the OpenClaw gateway port to the Windows host.

## Requirements

### Required for the Current Setup

* Windows.
* Docker Desktop.
* Docker Compose.
* Sufficient disk space for container images, AI models, and persistent data.
* Sufficient system memory for the selected models and services.

### Optional Development Tools

* Visual Studio Code.
* Continue extension.
* PowerShell for running Docker commands.

### GPU Support

The current Compose configuration includes an NVIDIA GPU device reservation for Ollama.

GPU availability depends on compatible hardware, drivers, and Docker Desktop configuration. CPU-only operation has not been verified for this configuration.

## Configuration

### Docker Compose

The `docker-compose.yml` file defines the Ollama and OpenClaw services, their container settings, port mapping, and persistent volumes.

### Ollama

* Image: `ollama/ollama:latest`
* Container: `personal-ai-ollama`
* Internal API port: `11434`
* Windows host port: `11435`
* Model storage: `personal-ai-models`
* Restart policy: `unless-stopped`

### OpenClaw

* Image: `openclaw:local`
* Container: `personal-ai-openclaw`
* Verified version: `2026.9.6`
* Gateway port: `18789` inside the container
* Restart policy: `unless-stopped`
* Base image recorded in image metadata: `node:24-bookworm-slim`

The image metadata identifies the upstream project as [openclaw/openclaw](https://github.com/openclaw/openclaw).

The local image is available in the current environment. Rebuilding it from scratch requires build instructions that are not yet included in this repository.

### OpenClaw Model Configuration

The current OpenClaw configuration registers these models through the Ollama provider:

* `ollama/qwen3:8b`
* `ollama/qwen3:8b-32k`
* `ollama/qwen2.5-coder:7b`

The default agent model is:

`ollama/qwen2.5-coder:7b`

The provider configuration specifies a context setting of `32768` tokens.

### Persistent Volumes

Docker Compose defines these named volumes:

| Volume                  | Container Path                   | Purpose                         |
| ----------------------- | -------------------------------- | ------------------------------- |
| `personal-ai-models`    | `/root/.ollama`                  | Ollama models and data          |
| `personal-ai-openclaw`  | `/home/node/.openclaw`           | OpenClaw configuration and data |
| `personal-ai-workspace` | `/home/node/.openclaw/workspace` | OpenClaw workspace              |

These volumes preserve data independently of the lifecycle of individual containers. They should be backed up before destructive maintenance.

### Environment Variables and Secrets

The OpenClaw service reads `OPENCLAW_GATEWAY_TOKEN` from the local environment.

* Keep `.env` local when it contains secrets.
* Never publish gateway tokens, passwords, or other credentials.
* Review `.gitignore` before committing changes.
* Do not publish private configuration files or sensitive backups.

## Run

The following commands are intended for the existing configured environment.

### 1. Start the Services

Open PowerShell in the directory containing `docker-compose.yml`:

```powershell
docker compose up -d
```

This starts the services defined in the Compose file. The local image `openclaw:local` must already exist because the current Compose configuration does not define a build context.

### 2. Check Container Status

```powershell
docker ps
```

Confirm that these containers are running:

* `personal-ai-ollama`
* `personal-ai-openclaw`

### 3. List Available Models

```powershell
docker exec personal-ai-ollama ollama list
```

The current environment contains:

* `qwen3:8b`
* `qwen3:8b-32k`
* `qwen2.5-coder:7b`

### 4. Check the Ollama API

Run this command from the OpenClaw container:

```powershell
docker exec personal-ai-openclaw node -e "fetch('http://ollama:11434/api/tags').then(r => r.json()).then(console.log)"
```

A successful response should include information about the available models.

### 5. Check OpenClaw

```powershell
docker exec personal-ai-openclaw node dist/index.js --version
```

The verified version in the current environment is `2026.9.6`.

Check the container state:

```powershell
docker inspect --format='{{.State.Status}}' personal-ai-openclaw
```

The expected running state is `running`.

If a health check is configured, inspect it separately:

```powershell
docker inspect --format='{{if .State.Health}}{{.State.Health.Status}}{{else}}no healthcheck{{end}}' personal-ai-openclaw
```

## Testing

### Test Model Inference

```powershell
docker exec personal-ai-ollama ollama run qwen2.5-coder:7b "Write a Python function that adds two numbers."
```

A generated response indicates that the model can perform inference.

### Test the Ollama API

```powershell
docker exec personal-ai-openclaw node -e "fetch('http://ollama:11434/api/tags').then(r => r.json()).then(console.log)"
```

Confirm that the response contains the expected models.

### Test the Development Workflow

If VS Code and Continue are configured:

1. Open a project in VS Code.
2. Send a coding request through Continue.
3. Verify that the selected model responds.
4. Test file creation or editing when the required Agent tools and permissions are enabled.

A running container or a successful API response does not prove that every integration works correctly.

## Security

* Never publish API tokens, passwords, or other secrets.
* Keep private configuration files out of version control.
* Review `.gitignore` before committing changes.
* Restrict access to OpenClaw tools and workspace operations.
* Back up important configuration and workspace data securely.
* Rotate any credential that has been exposed.

Removing a secret from a current file does not remove it from Git history.

## References

* [Docker Documentation](https://docs.docker.com/)
* [Docker Compose Documentation](https://docs.docker.com/compose/)
* [Ollama Documentation](https://docs.ollama.com/)
* [Ollama Model Library](https://ollama.com/library)
* [OpenClaw Repository](https://github.com/openclaw/openclaw)
* [OpenClaw Docker Installation](https://docs.openclaw.ai/install/docker)
* [OpenClaw Documentation](https://docs.openclaw.ai/)
* [Continue Documentation](https://docs.continue.dev/)
* [Qwen3 Models](https://ollama.com/library/qwen3)
* [Qwen2.5 Coder Models](https://ollama.com/library/qwen2.5-coder)

## Project Status

The current environment has been configured with:

* Ollama running in Docker.
* Three AI models available through Ollama.
* OpenClaw running in a separate Docker container.
* Persistent Docker volumes for models, configuration, and workspace data.
* Optional VS Code and Continue integration on Windows.

The current environment has been inspected, but a complete from-scratch installation procedure has not yet been verified. The build process for the local `openclaw:local` image remains to be documented.
