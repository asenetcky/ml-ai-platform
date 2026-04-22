# ML/AI Tailnet Platform

A self-hosted AI and machine learning platform that runs entirely on your own hardware, secured and connected by Tailscale. No cloud subscriptions. No data leaving your network. No open ports.

## The Problem

Running AI tools locally is easy. Running them *accessibly* — from your laptop at a coffee shop, your phone, a remote workstation, a second machine at home — is where things fall apart fast.

The naive solution is port forwarding: punch a hole in your firewall, expose your services to the internet, hope for the best. That's a bad trade. You're exposing a GPU-backed inference server or an experiment tracking UI to every scanner on the internet, with nothing but a port number between you and them.

The slightly-less-naive solution is a VPN. But traditional VPNs require a central server, constant maintenance, certificate management, and enough networking knowledge to make a mess of your firewall rules.

This platform takes a third path.

## How Tailscale Changes Everything

[Tailscale](https://tailscale.com) builds a private mesh network — a *Tailnet* — between any devices you authorize. Traffic flows peer-to-peer where possible, encrypted end-to-end with WireGuard. There's no central server routing your data. Each machine on your Tailnet gets a stable private IP (`100.x.x.x`) and, with MagicDNS, a human-readable hostname.

Every service in this platform uses Tailscale as a **sidecar container**. The AI service (Ollama, Open WebUI, MLflow) binds only to localhost inside the container's network namespace. Tailscale owns the network interface. Nothing is reachable unless you're on the Tailnet — no firewall rules to write, no ports to forward, no certificates to manage manually.

When you enable Tailscale Serve, each service gets a valid HTTPS endpoint (`https://<service>.<tailnet>.ts.net`) with a certificate Tailscale manages automatically. You open a browser on any of your devices, anywhere in the world, and it just works.

## What's Running

| Service | What it does |
|---------|-------------|
| **[Ollama](https://ollama.com)** | Pulls and serves local LLMs (Llama, Gemma, Mistral, and others). Exposes an OpenAI-compatible API so any client that speaks OpenAI can use it. |
| **[Open WebUI](https://openwebui.com)** | A polished, feature-complete chat interface for your local models. Supports conversation history, RAG, image generation, and more — all pointed at your Ollama instance. |
| **[MLflow](https://mlflow.org)** | Experiment tracking for ML work. Log parameters, metrics, and artifacts from training runs; compare experiments; store models. Backed by SQLite, so there's nothing else to run. |

## Architecture

Each service is an independent Docker Compose stack:

```
services/
├── ollama/        → LLM backend          → https://ollama.<tailnet>.ts.net
├── open-webui/    → Chat UI              → https://open-webui.<tailnet>.ts.net
└── mlflow/        → Experiment tracking  → https://mlflow.<tailnet>.ts.net
```

Every stack follows the same pattern: a **Tailscale sidecar** container and an **application** container that shares the sidecar's network namespace (`network_mode: service:tailscale`). The application never touches the host network directly. Tailscale handles inbound traffic, TLS termination, and routing.

```
  Your device (on Tailnet)
        │
        │  WireGuard / HTTPS
        ▼
  ┌─────────────────────┐
  │  Tailscale sidecar  │  ← owns the network interface
  │                     │
  │  app (localhost)    │  ← binds only to 127.0.0.1
  └─────────────────────┘
        │
     host machine (no exposed ports)
```

Persistent data (model weights, chat history, experiment databases) is stored in bind-mounted directories on the host so it survives container restarts and image updates.

## Getting Started

### Prerequisites

- Docker and Docker Compose
- A [Tailscale account](https://tailscale.com) (free tier covers personal use)
- MagicDNS and HTTPS certificates enabled in your [Tailscale admin console](https://login.tailscale.com/admin/dns)
- An auth key from [tailscale.com/admin/authkeys](https://tailscale.com/admin/authkeys)

### Deploying a Service

Each service is self-contained. Deploy them independently in any order.

```bash
cd services/ollama          # or open-webui, mlflow

cp .env-example .env        # copy the template
$EDITOR .env                # set TS_AUTHKEY and any required vars

mkdir -p config ts/state ollama-data   # pre-create bind mounts

docker compose up -d
```

The Tailscale container will register with your Tailnet on first start. Within a few seconds the service is reachable at `https://<SERVICE>.<tailnet>.ts.net` from any device on your network.

See the `README.md` in each service directory for service-specific configuration details.

### Pointing Open WebUI at Ollama

Set `OLLAMA_BASE_URL` in `services/open-webui/.env`. If both are running on the same host, `http://host.docker.internal:11434` works out of the box. If Ollama is on a different machine on your Tailnet, use its Tailscale IP (`http://100.x.x.x:11434`).

### Logging ML Experiments

Point your training scripts at the MLflow server:

```python
import mlflow
mlflow.set_tracking_uri("https://mlflow.<tailnet>.ts.net")
```

Runs, parameters, metrics, and artifacts are tracked automatically from any machine on your Tailnet.

## Credits

Inspired by the [ScaleTail](https://github.com/tailscale-dev/ScaleTail) project and Tailscale's container sidecar pattern.
