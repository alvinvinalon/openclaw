# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository manages a Dockerized deployment of [OpenClaw](https://docs.openclaw.ai/install/docker) — a containerized AI gateway. It is not an application codebase; it is an infrastructure/ops workspace for installation, configuration, and ongoing maintenance of OpenClaw running in Docker.

## First-Time Setup

```bash
# Option 1: Build locally
./scripts/docker/setup.sh

# Option 2: Use a pre-built image (faster)
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./scripts/docker/setup.sh

# Option 3: Airgapped/offline install
docker load -i openclaw-image.tar
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./scripts/docker/setup.sh --offline
```

The setup script runs interactive onboarding: it prompts for provider API keys, generates a gateway token stored in `.env`, and launches the stack via Docker Compose.

## Rebuilding the Local Image

**CRITICAL:** This deployment uses `Dockerfile.local` (extends the pre-built ghcr.io image), NOT the root `Dockerfile` (full TypeScript source compilation).

`docker-compose.override.yml` already redirects `docker compose build` to use `Dockerfile.local`, so both of the following are equivalent and safe:

```bash
# Option A — explicit (preferred, always unambiguous)
docker build -f Dockerfile.local -t openclaw-local .

# Option B — also safe because override redirects to Dockerfile.local
docker compose build
```

**Never run `docker build .`** (without `-f`) — that uses the root `Dockerfile` and compiles from source.

**Never run `setup.sh` to rebuild** — it is a full bootstrapper that rewrites `.env`.

**Always tag a backup before rebuilding:**
```bash
docker tag openclaw-local openclaw-local:backup
```

If the gateway crashes after a bad build, restore instantly:
```bash
docker tag openclaw-local:backup openclaw-local
docker compose restart openclaw-gateway
```

**Never change the `FROM` tag in `Dockerfile.local` without verifying schema compatibility.** The state database schema version must be ≤ what the new base image supports. A downgrade (e.g. `:latest` → `:2026.7.1` that supports an older schema) will crash the gateway and may require wiping the state database. See `INCIDENT-2026-07-15-schema-mismatch.md` for the full post-mortem.

## Common Operations

```bash
# Get dashboard URL (or re-open it)
docker compose run --rm openclaw-cli dashboard --no-open

# Health checks (unauthenticated — safe for orchestration probes)
curl -fsS http://127.0.0.1:18789/healthz   # liveness
curl -fsS http://127.0.0.1:18789/readyz    # readiness

# Authenticated health snapshot
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"

# Add messaging channels
docker compose run --rm openclaw-cli channels login                                   # WhatsApp
docker compose run --rm openclaw-cli channels add --channel telegram --token "<tok>"  # Telegram
docker compose run --rm openclaw-cli channels add --channel discord  --token "<tok>"  # Discord

# Reset gateway to default local/lan binding
docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
```

## Upgrading

New images run automatic startup-safe migrations. If a migration cannot complete safely, the container exits instead of reporting healthy.

```bash
# For a stuck upgrade, run the doctor manually
docker run --rm -v <state-path>:/home/node/.openclaw <image> openclaw doctor --fix
```

## Key Environment Variables

Set these before running `setup.sh` to change build/runtime behavior:

| Variable | Effect |
|---|---|
| `OPENCLAW_IMAGE` | Pull this image instead of building locally |
| `OPENCLAW_SANDBOX` | `1` enables Docker-backed sandbox execution for agents |
| `OPENCLAW_INSTALL_BROWSER` | Bundle Chromium/Xvfb in the image |
| `OPENCLAW_EXTENSIONS` | Comma/space-separated plugins to bundle |
| `OPENCLAW_IMAGE_APT_PACKAGES` | Additional apt packages baked into the image |
| `OPENCLAW_IMAGE_PIP_PACKAGES` | Python packages baked into the image |
| `OPENCLAW_EXTRA_MOUNTS` | Extra host bind mounts (`source:target[:opts]`) |
| `OPENCLAW_HOME_VOLUME` | Persist `/home/node` in a named Docker volume |
| `OPENCLAW_SKIP_ONBOARDING` | Skip interactive setup prompts |
| `OPENCLAW_DOCKER_SOCKET` | Override Docker socket path |
| `OPENCLAW_DISABLE_BONJOUR` | Control Bonjour/mDNS advertising |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry collector endpoint |

## Persistence

Docker Compose bind-mounts these host directories into the container:

- `OPENCLAW_CONFIG_DIR` → `/home/node/.openclaw` — behavior settings, OAuth credentials, runtime secrets
- `OPENCLAW_WORKSPACE_DIR` → `/home/node/.openclaw/workspace`
- `OPENCLAW_AUTH_PROFILE_SECRET_DIR` → `/home/node/.config/openclaw`

These directories survive container replacement. Ensure they are owned by uid 1000:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config
```

## Connecting Host-Side AI Providers (LM Studio, Ollama, etc.)

Inside the container, `127.0.0.1` refers to the container itself — use `host.docker.internal` to reach the host:

- LM Studio: `http://host.docker.internal:1234`
- Ollama: `http://host.docker.internal:11434`

## Agent Sandbox

To enable Docker-backed isolated execution for agents:

```bash
export OPENCLAW_SANDBOX=1
./scripts/docker/setup.sh
```

Build the sandbox image separately:
```bash
scripts/sandbox-setup.sh
```

`openclaw.json` sandbox options:
```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",  // off | non-main | all
        scope: "agent",    // session | agent | shared
      },
    },
  },
}
```

**Never mount the host Docker socket into agent sandboxes.**

## Security Posture

- Container runs as non-root `node` user (uid 1000)
- Linux capabilities `NET_RAW` and `NET_ADMIN` are dropped; `no-new-privileges` is enabled
- Gateway token is stored in `.env` — do not commit `.env` to version control
- For VPS/public deployments, consult the OpenClaw security hardening docs before exposing the gateway port

## Network Binding

Configured via `gateway.bind` in `openclaw.json`:

- `lan` (default): host browser and CLI can reach the published port
- `loopback`: only processes inside the container network namespace can reach the gateway
- `tailnet` / `custom` / `auto`: for specialized network topologies

## Windows Docker Desktop Notes

This deployment runs on Windows with Docker Desktop (WSL2 backend). Two known platform differences apply:

**File permissions:** NTFS bind mounts appear as mode 777 inside Linux containers; `chmod` on them fails with EPERM. The `security audit` CRITICAL finding `fs.config.perms_writable` is unfixable on NTFS mounts — it is a false positive on Windows. Files are protected by Windows NTFS ACLs, not Linux DAC. If this is a concern, migrate config storage to a Docker named volume (`OPENCLAW_HOME_VOLUME=openclaw-home`).

**Loopback binding and the browser dashboard:** With `OPENCLAW_GATEWAY_BIND=loopback` the gateway listens on `127.0.0.1` inside the container. Docker's port mapping cannot forward to the container's loopback, so `http://127.0.0.1:18789` is unreachable from the host browser. All management must go through the CLI:

```bash
docker compose run --rm openclaw-cli <command>
```

To enable the browser dashboard temporarily, change `OPENCLAW_GATEWAY_BIND=lan` in `.env` and run `./scripts/docker/setup.sh` again, or run:

```bash
docker compose run --rm openclaw-cli config set gateway.bind lan
```

then restart the gateway. Revert to `loopback` when done.

**`sync_gateway_config` path translation:** The `setup.sh` one-off containers write config via bind mounts. On Windows, if the mount path fails to translate (e.g. `/c/Users/...` not mapping correctly for one-off containers), the config write succeeds inside an ephemeral layer and is lost. **Fix:** write `gateway.mode` and other settings directly to `C:\Users\<you>\.openclaw\openclaw.json` on the host, then restart the gateway container.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Gateway blocked: missing gateway.mode | Write `gateway.mode=local` directly to `C:\Users\<you>\.openclaw\openclaw.json` and restart |
| OOM during image build | `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 ./scripts/docker/setup.sh` |
| Permission errors on mounted paths | On Linux: `sudo chown -R 1000:1000 <path>`; on Windows: use a named volume |
| DNS failures in CLI sidecar | Use the provided override Compose file for affected commands |
| Bonjour not working on Docker bridge | mDNS multicast (`224.0.0.251:5353`) may not forward — expected limitation |
| security audit CRITICAL chmod EPERM | Windows NTFS bind mount limitation — not fixable via chmod; use named volume if needed |
