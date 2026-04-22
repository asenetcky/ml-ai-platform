# Changelog

All notable changes to this project will be documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions follow [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

---

## [0.2.0] - 2026-04-22

### Added
- `CLAUDE.md` — repository guide for Claude Code with architecture overview, service anatomy, and shared env var reference
- `services/mlflow/README.md` — setup guide covering prerequisites, volumes, MagicDNS, DNS rebinding protection, and client usage
- `services/mlflow/.env-example` — standardized template matching the ScaleTail pattern used by Ollama and Open WebUI
- `services/ollama/README.md` — setup guide covering prerequisites, volumes, MagicDNS, port exposure, and first-run model pull
- `services/ollama/.env-example` — env template with inline documentation
- `services/open-webui/README.md` — setup guide covering prerequisites, Ollama connection options, and first-launch gotchas
- `services/open-webui/.env-example` — env template with Ollama URL examples

### Changed
- **README.md** — complete rewrite; now leads with the problem (locally-running-but-not-accessible AI), explains the Tailscale sidecar model, includes architecture diagram, and covers getting-started steps for all three services
- **`services/mlflow/compose.yaml`** — major refactor:
  - Dropped PostgreSQL dependency; switched to SQLite backend (`sqlite:////mlflow/mlflow.db`) — no extra service to run
  - Removed custom `Dockerfile` (no longer needed without psycopg2)
  - Switched to upstream `ghcr.io/mlflow/mlflow:latest` image via `${IMAGE_URL}`
  - Standardized to ScaleTail sidecar pattern: `tailscale` + `application` containers, named `tailscale-${SERVICE}` / `app-${SERVICE}`
  - Added inline `configs:` block for Tailscale Serve (proxying HTTPS → port 5000)
  - Replaced named volumes with bind mounts (`./config`, `./ts/state`, `./mlflow-data`)
  - Added proper healthchecks on both containers; `application` now waits for `tailscale` to pass `service_healthy`
  - Added `ALLOWED_HOSTS` and `CORS_ALLOWED_ORIGINS` flags to fix DNS rebinding 403s when accessing via Tailnet FQDN
- **`services/ollama/compose.yaml`** — refactored to ScaleTail sidecar pattern (matching structure above)
- **`services/open-webui/compose.yaml`** — refactored to ScaleTail sidecar pattern
- **`.gitignore`** — updated to cover all bind-mount data directories and runtime config

### Removed
- `services/mlflow/Dockerfile` — no longer needed after dropping PostgreSQL

---

## [0.1.0] - 2025-12-12

### Added
- Initial project structure with three services: Ollama, Open WebUI, MLflow
- MIT License
- Basic `README.md`
- MLflow compose with PostgreSQL backend and custom Dockerfile
- Ollama and Open WebUI compose stacks with basic Tailscale integration

[Unreleased]: https://github.com/asenetcky/ml-ai-platform/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/asenetcky/ml-ai-platform/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/asenetcky/ml-ai-platform/releases/tag/v0.1.0
