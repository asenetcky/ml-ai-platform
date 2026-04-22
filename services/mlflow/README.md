# MLflow with Tailscale Sidecar Configuration

This Docker Compose configuration sets up [MLflow](https://mlflow.org) with Tailscale as a sidecar container to keep the tracking UI and API reachable securely over your Tailnet.

## MLflow

[MLflow](https://mlflow.org) is an open-source platform for managing the ML lifecycle: experiment tracking, model registry, and reproducible runs. Pairing it with Tailscale means the tracking server is reachable from any device on your Tailnet (laptop, workstation, training box) without exposing it to the public internet.

This setup uses MLflow's default **SQLite** backend store — the experiment metadata DB and artifacts both live on disk under `./mlflow-data`. No external database is required.

## Configuration Overview

The `tailscale-mlflow` service runs Tailscale, which manages secure networking for MLflow. The `app-mlflow` service uses Docker's `network_mode: service:tailscale` so all traffic is routed through the Tailscale network stack. The MLflow UI/API remains Tailnet-only by default unless you explicitly expose the port to your LAN.

## Prerequisites

- The host user must be in the `docker` group.
- The `/dev/net/tun` device must be available on the host (standard on most Linux systems).
- Pre-create the bind-mount directories before starting the stack to avoid Docker creating root-owned folders:

```bash
mkdir -p config ts/state mlflow-data
```

## Volumes

| Path            | Purpose                                                     |
| --------------- | ----------------------------------------------------------- |
| `./config`      | Tailscale serve config (`serve.json`)                       |
| `./ts/state`    | Tailscale persistent state                                  |
| `./mlflow-data` | MLflow SQLite metadata DB (`mlflow.db`) and `artifacts/`    |

## MagicDNS and HTTPS

Tailscale Serve is pre-configured to proxy HTTPS on port 443 to MLflow's internal port 5000. To enable it:

1. Uncomment `TS_ACCEPT_DNS=true` in the `tailscale` service environment.
2. Ensure your Tailnet has MagicDNS and HTTPS certificates enabled in the [Tailscale admin console](https://login.tailscale.com/admin/dns).
3. The `serve.json` config in `compose.yaml` uses `$TS_CERT_DOMAIN` automatically — no manual editing needed.

You can then reach MLflow at `https://mlflow.<your-tailnet-name>.ts.net`.

## Port Exposure (LAN access)

By default, the `ports:` section is commented out — MLflow is only accessible over your Tailnet. If you also want LAN access (e.g. from devices not on Tailscale), uncomment it in `compose.yaml`:

```yaml
ports:
  - 0.0.0.0:5000:5000
```

This is optional and not required for Tailnet-only usage.

## Using MLflow from a Client

Point your training scripts at the tracking server:

```python
import mlflow

# Over Tailnet
mlflow.set_tracking_uri("http://<tailscale-ip>:5000")
# Or via HTTPS/MagicDNS
mlflow.set_tracking_uri("https://mlflow.<your-tailnet-name>.ts.net")

mlflow.set_experiment("my-experiment")
with mlflow.start_run():
    mlflow.log_param("lr", 0.01)
    mlflow.log_metric("accuracy", 0.92)
```

## Switching to a Different Backend Store

SQLite is fine for a single user and small-to-medium experiment volumes. If you outgrow it, swap the `--backend-store-uri` flag in `compose.yaml` for a Postgres/MySQL connection string and wire in a database service.

## DNS Rebinding Protection

MLflow 3.x enables a Host-header allowlist by default and returns `403 Forbidden` with *"Invalid Host header - possible DNS rebinding attack detected"* for any host not on the list. Set `ALLOWED_HOSTS` (and `CORS_ALLOWED_ORIGINS` for browser calls) in `.env` to include your Tailnet FQDN:

```
ALLOWED_HOSTS=mlflow.<your-tailnet-name>.ts.net,localhost,127.0.0.1
CORS_ALLOWED_ORIGINS=https://mlflow.<your-tailnet-name>.ts.net
```

Using `*` disables the protection entirely — fine if you're strictly on a private Tailnet, but an explicit allowlist is preferred.

## Files to check

Please check the following contents for validity as some variables need to be defined upfront.

- `.env` — Set `TS_AUTHKEY` (required), `ALLOWED_HOSTS` and `CORS_ALLOWED_ORIGINS` (required — must include your Tailnet FQDN).

## Useful Links

- [MLflow official site](https://mlflow.org)
- [MLflow documentation](https://mlflow.org/docs/latest/index.html)
- [MLflow GitHub](https://github.com/mlflow/mlflow)
- [Tailscale auth keys](https://tailscale.com/kb/1085/auth-keys)
- [Tailscale Serve docs](https://tailscale.com/kb/1312/serve)
