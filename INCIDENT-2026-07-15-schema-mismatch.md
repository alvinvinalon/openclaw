# Incident Report: Gateway Schema Version Mismatch
**Date:** 2026-07-15  
**Severity:** High — gateway fully down, required data loss to recover  
**Duration:** Several hours across session  

---

## What Happened

During a session to add Azure CLI support to the Docker image, the OpenClaw gateway was taken down and entered a crash loop. The gateway could not start and logged:

```
OpenClaw state database /home/node/.openclaw/state/openclaw.sqlite
uses newer schema version 2; this OpenClaw build supports 1.
```

---

## Root Cause

**The direct cause was a change to `Dockerfile.local` that swapped the base image tag from `:latest` to `:2026.7.1`.**

Timeline:
1. The original working `openclaw-local` image was built from `Dockerfile.local` using `FROM ghcr.io/openclaw/openclaw:latest`. That specific pull of `:latest` supported OpenClaw state database **schema v2**, and the state database at `C:\Users\alvin\.openclaw\state\openclaw.sqlite` was created at schema v2.
2. During this session, `Dockerfile.local` was modified to pin the base image to `FROM ghcr.io/openclaw/openclaw:2026.7.1` (intended for build reproducibility).
3. A rebuild was triggered. The `2026.7.1` image only supports **schema v1**.
4. When the gateway restarted with the rebuilt image, it could not read the existing schema v2 database → crash loop.

**Secondary finding:** The current `ghcr.io/openclaw/openclaw:latest` tag on the registry also only supports schema v1 as of this date — the tag has moved since the original setup. This means the original schema v2 image no longer exists anywhere: not in the registry, not in local Docker storage (it was overwritten by the rebuild).

---

## What Was NOT the Cause

- The Azure CLI addition (`RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash` in `Dockerfile.local`) did not cause the schema mismatch.
- `docker compose build` was not the wrong command — `docker-compose.override.yml` already redirects it to `Dockerfile.local`. Using `docker compose build` is safe in this repo.
- The root `Dockerfile` (TypeScript source compilation) was not used.

---

## Impact

- Gateway was down for several hours.
- `openclaw doctor --fix` could not resolve a schema downgrade (it only handles upgrades).
- **The state database had to be renamed** (`openclaw.sqlite` → `openclaw.sqlite.bak`), causing permanent loss of all session/conversation history. A fresh schema v1 database was created on next start.
- All configuration files were unaffected: `openclaw.json`, `exec-approvals.json`, `.env`, skills, credentials.

---

## Resolution Steps

1. Reverted `Dockerfile.local` back to `FROM ghcr.io/openclaw/openclaw:latest`
2. Rebuilt: `docker build -f Dockerfile.local -t openclaw-local .` (all layers were cached, completed instantly)
3. Found that current `:latest` also only supports schema v1 → rebuild alone was not enough
4. Renamed state database: `C:\Users\alvin\.openclaw\state\openclaw.sqlite` → `openclaw.sqlite.bak`
5. Also renamed orphaned WAL/SHM files left from the old database
6. Restarted gateway → fresh schema v1 database created → gateway healthy
7. Verified all configuration, models, skills, exec allowlist, and Azure integration intact

---

## Do You Need to Remove Dockerfile or Dockerfile.local?

**No. Both files should stay. Here is what each is for:**

| File | Purpose | Used by this deployment? |
|---|---|---|
| `Dockerfile` | Full TypeScript source compilation — builds OpenClaw from scratch. Part of the upstream OpenClaw repo. | **No** |
| `Dockerfile.local` | Extends a pre-built `ghcr.io` image with gog + azure-cli. Designed for this deployment. | **Yes** |

`docker-compose.override.yml` already overrides the build target for both services to use `Dockerfile.local`. So `docker compose build` is safe and will always use the right file.

The root `Dockerfile` would only be used if someone ran `docker build .` directly (no `-f` flag) or ran `setup.sh` (which triggers its own build). Neither should be done during normal operations.

---

## Prevention

1. **Never change the base image tag in `Dockerfile.local` without first checking schema compatibility.** Before pinning to a specific release tag, verify the schema version it supports matches or exceeds the current state database.

2. **Tag a backup before any rebuild:**
   ```
   docker tag openclaw-local openclaw-local:backup
   ```
   If anything goes wrong, restore immediately with:
   ```
   docker tag openclaw-local:backup openclaw-local
   docker compose restart openclaw-gateway
   ```

3. **Keep `Dockerfile.local` on `:latest` unless there is a specific reason to pin**, and verify schema compatibility before pinning.

4. **Never run `setup.sh` to trigger a rebuild** — it is a full bootstrapper that rewrites `.env` and pulls potentially incompatible images.

---

## Current State (Post-Recovery)

- Gateway: healthy, schema v1
- Image: `openclaw-local` built from `Dockerfile.local` with `FROM ghcr.io/openclaw/openclaw:latest`
- State database: fresh v1 (session history from before 2026-07-15 is in `openclaw.sqlite.bak`)
- All skills, models, exec approvals, Azure integration: fully restored
