# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal Tailscale-connected ML/AI homelab platform. Each service lives in `services/<name>/` and runs as an independent Docker Compose stack — there is no top-level compose file that orchestrates everything together.

## Starting, stopping, and updating services

Each service is managed independently from its own directory:

```bash
# Start a service
cd services/<name>
cp .env-example .env   # first time only — fill in TS_AUTHKEY and any required vars
mkdir -p config ts/state <name>-data   # pre-create bind mounts (prevents root-owned dirs)
docker compose up -d

# Restart a single container without recreating everything
docker compose up -d --force-recreate application

# View logs
docker compose logs -f
docker logs app-<name>

# Pull updated images and redeploy
docker compose pull && docker compose up -d
```

## Service anatomy

Every service follows the same pattern:

| File / Dir        | Purpose |
|-------------------|---------|
| `compose.yaml`    | Two services: `tailscale` (sidecar) + `application` |
| `.env-example`    | Template — copy to `.env` and fill in secrets |
| `config/`         | Tailscale `serve.json` (written by Docker at runtime, gitignored) |
| `ts/state/`       | Tailscale persistent state (gitignored) |
| `<name>-data/`    | Application persistent data (gitignored) |
| `README.md`       | Service-specific setup notes |

**Networking model:** `application` runs with `network_mode: service:tailscale`, so it shares the Tailscale container's network namespace. All inbound traffic arrives through Tailscale; the application binds to `0.0.0.0` on its port and Tailscale proxies HTTPS (port 443) to it via `serve.json`.

**Health dependency:** `application` only starts after `tailscale` passes its healthcheck (`wget /healthz` on `127.0.0.1:41234`).

## Environment variables shared across services

| Variable | Purpose |
|----------|---------|
| `SERVICE` | Hostname in Tailscale; also used in container names (`app-${SERVICE}`, `tailscale-${SERVICE}`) |
| `IMAGE_URL` | Full Docker image reference |
| `TS_AUTHKEY` | Tailscale auth key (required, from tailscale.com/admin/authkeys) |
| `SERVICEPORT` | Port to expose to LAN (used only if the `ports:` block is uncommented) |
| `TZ` | Container timezone |

## Services

| Service | Port | Notes |
|---------|------|-------|
| `ollama` | 11434 | LLM inference backend; models stored in `ollama-data/` |
| `open-webui` | 8080 | Chat UI; connects to Ollama via `OLLAMA_BASE_URL` |
| `mlflow` | 5000 | Experiment tracking; SQLite at `mlflow-data/mlflow.db`, artifacts at `mlflow-data/artifacts/`; requires `ALLOWED_HOSTS` and `CORS_ALLOWED_ORIGINS` set to the Tailnet FQDN to avoid DNS-rebinding 403s |

## Adding a new service

Copy the structure of an existing service. The `configs:` block at the top of `compose.yaml` writes the Tailscale serve config inline — update the proxy port to match the new application. Add the service data directory to `.gitignore` at the repo root.

## Tailscale HTTPS / MagicDNS

Each `compose.yaml` has `TS_ACCEPT_DNS=true` commented out. Uncomment it and enable MagicDNS + HTTPS certificates in the Tailscale admin console to reach services at `https://<SERVICE>.<tailnet-name>.ts.net`.
