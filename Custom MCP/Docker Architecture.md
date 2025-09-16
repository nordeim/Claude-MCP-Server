# Review of the current setup

Your multi-stage distroless build is a strong start. Here’s what’s solid and what needs attention.

## Strengths

- **Slim, non-root runtime:** Distroless + nonroot reduces attack surface.  
- **Multi-stage build:** Dependencies isolated from runtime.  
- **Explicit entrypoint:** Running module by name is clean and portable.  
- **Security scanning:** Trivy integration is already in place.

## Gaps and improvements

- **Pip install location mismatch:** You install with `--user` into `/root/.local`, then switch to a nonroot user. That path may be unreadable or not on `PATH` for `nonroot`. Prefer a venv in `/opt/venv` with proper permissions and `PATH`.  
- **Reproducibility and supply chain:** Use a locked requirements file with hashes, build wheels offline, and verify hashes. Consider SBOM generation.  
- **Hot reload mismatch:** Distroless images don’t carry dev tooling. Separate Dockerfile.dev for fast reload and tooling is more maintainable.  
- **Health checks:** No runtime health checks; add JSON `HEALTHCHECK` for HTTP transport, and readiness checks strategy for stdio transport.  
- **Security hardening:** Add read-only root FS, tmpfs for `/tmp`, drop caps, seccomp/apparmor profiles, `no-new-privileges`, and run as a non-root UID with owned files.  
- **Signals and graceful shutdown:** Ensure `ENTRYPOINT` and process model forward signals cleanly for asyncio/uvicorn.  
- **Observability:** Provide basic structured logging and log level env vars.  
- **Transport profiles:** Separate stdio vs HTTP/SSE profiles for clear network/scalability options.  
- **Build caching and size:** Enable wheels cache stage, avoid copying whole `src/` tree when not needed, use `.dockerignore`.  
- **Configuration:** Centralize settings via env vars with sane defaults; avoid code rewrites.

---

# Redesigned docker architecture

Two build profiles and two runtime profiles to cover dev and production cleanly.

- **Build profiles**
  - **Production distroless:** Minimal, non-root, venv at `/opt/venv`, wheels built in builder with hash verification.
  - **Developer image:** Full Python base, `uv`, `pip-tools` or `pip` with `watchfiles`/`uvicorn[standard]` and hot-reload.

- **Runtime transports**
  - **STDIO (default):** For MCP stdio integration. Container typically runs with `--network=none`, but offer config toggles.
  - **HTTP/SSE (optional):** Uvicorn/FastAPI server exposure, health endpoints, and K8s readiness/liveness.

- **Security posture**
  - Non-root UID/GID, read-only root filesystem, minimal mounts, tmpfs `/tmp`, `cap_drop: [ALL]`, `no-new-privileges`, seccomp/apparmor profiles.
  - Build-time: verify hashes, vendor wheels, generate SBOM, scan image.

---

# Execution plan with integrated checklists

## 1) Project files

- **Files to add/update**
  - Dockerfile.distroless
  - Dockerfile.dev
  - docker-compose.dev.yml
  - docker-compose.http.yml
  - .dockerignore
  - requirements.in (optional) and requirements.txt (locked with hashes)
  - Makefile (quality-of-life targets)
  - src/mcp_server/server.py (ensure graceful shutdown, configurable transports)
  - src/mcp_server/health.py (for HTTP health)

### Checklist

- **Requirements lock**
  - Generate or update `requirements.txt` with exact versions and hashes.
  - Validate lock file in CI before builds.

- **Source layout**
  - Ensure `mcp_server.server` remains the module entrypoint.
  - Confirm imports work without manipulating `PYTHONPATH`.

- **Health endpoints**
  - Provide `/healthz` and `/readyz` when HTTP mode is enabled.

- **Config**
  - Read env vars: MCP_TRANSPORT, HOST, PORT, LOG_LEVEL, WORKERS (if HTTP), UVICORN_*, etc.
  - Default to stdio.

## 2) Build pipeline

- **Builder stage**
  - Build wheels via `pip wheel`.
  - Install into `/opt/venv` using wheels only.
  - Enable `--require-hashes`.

- **Runtime stage**
  - Copy `/opt/venv`, copy `src/`.
  - Set `PATH=/opt/venv/bin:$PATH`.
  - Non-root user, correct ownership.

### Checklist

- **Reproducibility**
  - Use `--require-hashes`.
  - Pin Python minor version to avoid surprise ABI changes.

- **Size**
  - Avoid cache bloat.
  - Don’t copy build caches into runtime.

## 3) Dev workflow

- **Compose dev**
  - Mount `./src` as read-only or read-write as needed.
  - Use `Dockerfile.dev`.
  - Add `stdin_open: true`, `tty: true` only if using stdio interactive sessions.

- **Hot reload**
  - For HTTP: `uvicorn --reload`.
  - For stdio: `watchfiles` runner to restart on changes.

### Checklist

- **Speed**
  - Cache pip in builder if needed (optional).
  - Keep inner loop fast; avoid distroless for dev.

- **Parity**
  - Match Python version and dependency versions with production.

## 4) Runtime security and health

- **Security options**
  - Read-only root FS and tmpfs `/tmp`.
  - `cap_drop: [ALL]`, `no-new-privileges: true`.
  - Disable networking for stdio runs when feasible.

- **Health**
  - HTTP: Docker/K8s HEALTHCHECK hitting `/healthz`.
  - STDIO: readiness proven by starting successfully, or optional sidecar health endpoint.

### Checklist

- **Signals**
  - Ensure graceful shutdown on SIGTERM/SIGINT.
  - Uvicorn lifespan hooks if HTTP enabled.

## 5) CI/CD and scanning

- **Build**
  - Tag with commit SHA and semver.
- **Scan**
  - Trivy image scan fails on HIGH/CRITICAL.
- **SBOM**
  - Generate SBOM (syft/grype or equivalent).
- **Push**
  - Push to GHCR with provenance (optional).

### Checklist

- **Policies**
  - Ensure base images are updated regularly.
  - Renovate/Dependabot to bump images and dependencies.

---

# Validated plan and risk review

- **Distroless path and permissions:** Solved by `/opt/venv`, owned by non-root, PATH exported.  
- **STDIO health checks limitations:** Acknowledged; prefer HTTP health or external orchestrator-level checks.  
- **Hot reload in distroless:** Avoided by separate dev image.  
- **Hash-locked installs:** Requires discipline to maintain `requirements.txt`. Worth it for integrity and reproducibility.  
- **Network-less stdio:** Recommend `network_mode: none` in prod when practical; ensure no libs need network at runtime.

---

# Final deliverables

## Production Dockerfile (distroless)

```dockerfile
# syntax=docker/dockerfile:1.7
# ----- builder -----
FROM python:3.12-slim-trixie AS builder
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app

# Install build essentials only if needed for scientific deps; keep minimal
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc build-essential \
 && rm -rf /var/lib/apt/lists/*

# Copy requirements with hashes
COPY requirements.txt .

# Build wheels offline for deterministic installs
RUN python -m venv /opt/venv && . /opt/venv/bin/activate \
 && pip install --upgrade pip wheel \
 && pip wheel --no-deps --wheel-dir /wheels --require-hashes -r requirements.txt

# Install from wheels into clean venv
RUN rm -rf /opt/venv && python -m venv /opt/venv && . /opt/venv/bin/activate \
 && pip install --no-index --find-links=/wheels --require-hashes -r requirements.txt \
 && find /opt/venv -type d -name '__pycache__' -prune -exec rm -rf {} +

# ----- runtime -----
FROM gcr.io/distroless/python3-debian12:nonroot
ENV PATH=/opt/venv/bin:/usr/local/bin:/usr/bin:/bin \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UVICORN_WORKERS=1 \
    LOG_LEVEL=info \
    MCP_TRANSPORT=stdio

# Create app directory owned by nonroot user (UID 65532)
WORKDIR /app
USER nonroot:nonroot

# Copy venv and app source
COPY --from=builder --chown=nonroot:nonroot /opt/venv /opt/venv
COPY --chown=nonroot:nonroot src/ ./src

# Optional: HTTP mode healthcheck; harmless if HTTP isn’t used
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD [ "python3", "-c", "import os,sys,urllib.request; \
      t=os.getenv('MCP_TRANSPORT','stdio'); \
      (sys.exit(0) if t!='http' else urllib.request.urlopen(f\"http://127.0.0.1:{os.getenv('PORT','8000')}/healthz\").getcode()==200) \
      or sys.exit(1)" ]

# Run the server module; server should read MCP_TRANSPORT
ENTRYPOINT [ "python3", "-m", "mcp_server.server" ]
```

## Developer Dockerfile (hot reload and tooling)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim-trixie AS dev
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app

# Optional OS deps for local builds of scientific libs
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl build-essential \
 && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN python -m venv /opt/venv && . /opt/venv/bin/activate \
 && pip install --upgrade pip wheel \
 && pip install --require-hashes -r requirements.txt \
 && pip install watchfiles uvicorn[standard]

ENV PATH=/opt/venv/bin:$PATH PYTHONUNBUFFERED=1 LOG_LEVEL=debug MCP_TRANSPORT=stdio
WORKDIR /app
COPY src/ ./src

# Defaults for HTTP dev runs can be overridden in compose
ENTRYPOINT [ "python", "-m", "mcp_server.server" ]
```

## docker-compose.dev.yml (stdio hot-reload and debugging)

```yaml
version: "3.9"
services:
  mcp-dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: mcp-dev
    working_dir: /app
    environment:
      - MCP_TRANSPORT=stdio
      - LOG_LEVEL=debug
    volumes:
      - ./src:/app/src:rw
    stdin_open: true
    tty: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    read_only: false
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    command: >
      python -m watchfiles
      "python -m mcp_server.server"
```

## docker-compose.http.yml (HTTP/SSE mode with health and hardening)

```yaml
version: "3.9"
services:
  mcp-http:
    build:
      context: .
      dockerfile: Dockerfile.distroless
    image: ghcr.io/your-org/custom-mcp-server:${GIT_SHA:-dev}
    container_name: mcp-http
    environment:
      - MCP_TRANSPORT=http
      - HOST=0.0.0.0
      - PORT=8000
      - LOG_LEVEL=info
      - UVICORN_WORKERS=1
    ports:
      - "8000:8000"
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    healthcheck:
      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/healthz').read()"]
      interval: 30s
      timeout: 3s
      retries: 3
    restart: on-failure
```

## .dockerignore

```dockerignore
__pycache__/
*.pyc
*.pyo
*.pyd
*.log
.env
.env.*
.vscode/
.idea/
.git/
.gitignore
dist/
build/
.wheels/
.venv/
.pytest_cache/
.coverage
htmlcov/
```

## requirements.txt (locked with hashes, example)

```text
# Pin exact versions; include hashes for integrity
fastapi==0.115.2 \
  --hash=sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa \
  --hash=sha256:bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
uvicorn[standard]==0.30.6 \
  --hash=sha256:cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc \
  --hash=sha256:dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
# ... plus your MCP libraries and tools, each with hashes
```

Tip: Use pip-compile to generate this with `--generate-hashes`.

## Example server structure adjustments

- Ensure `mcp_server.server` reads env and starts correctly in both transports.
- Add health endpoints only if HTTP mode is enabled.

```python
# src/mcp_server/server.py
import os
import asyncio
import logging

LOG_LEVEL = os.getenv("LOG_LEVEL", "info").upper()
logging.basicConfig(level=getattr(logging, LOG_LEVEL, logging.INFO),
                    format="%(asctime)s %(levelname)s %(name)s %(message)s")

async def run_stdio():
    # Initialize MCP stdio transport
    # Your existing asyncio loop + handlers
    ...

def run_http():
    import uvicorn
    from .health import app as health_app  # optionally mount into your FastAPI app
    host = os.getenv("HOST", "0.0.0.0")
    port = int(os.getenv("PORT", "8000"))
    workers = int(os.getenv("UVICORN_WORKERS", "1"))
    # If you have your own FastAPI app, mount health routes there
    uvicorn.run("mcp_server.health:app", host=host, port=port, workers=workers, log_level=LOG_LEVEL.lower())

def main():
    transport = os.getenv("MCP_TRANSPORT", "stdio").lower()
    if transport == "http":
        run_http()
    else:
        try:
            asyncio.run(run_stdio())
        except KeyboardInterrupt:
            pass

if __name__ == "__main__":
    main()
```

```python
# src/mcp_server/health.py
from fastapi import FastAPI

app = FastAPI()
@app.get("/healthz")
def healthz():
    return {"status": "ok"}
@app.get("/readyz")
def readyz():
    # Optionally check readiness of tools/resources
    return {"ready": True}
```

## Makefile (quality-of-life)

```makefile
IMAGE=ghcr.io/your-org/custom-mcp-server
SHA=$(shell git rev-parse --short HEAD)

.PHONY: build scan sbom push dev up-http

build:
	docker build -t $(IMAGE):$(SHA) -f Dockerfile.distroless .

scan:
	trivy image --exit-code 1 --severity HIGH,CRITICAL $(IMAGE):$(SHA)

sbom:
	syft packages $(IMAGE):$(SHA) -o spdx-json > sbom-$(SHA).json

push:
	docker push $(IMAGE):$(SHA)

dev:
	docker compose -f docker-compose.dev.yml up --build

up-http:
	docker compose -f docker-compose.http.yml up --build
```

---

# Usage guide

## Local development (stdio with hot-reload)

- Build and run:
  - make dev
- Behavior:
  - Restarts on source changes.
  - Interactive stdio available with TTY.
- Notes:
  - For stdio-only debugging, you can run with `--network=none` by adding `network_mode: none` in compose.

## Local HTTP testing

- Start:
  - make up-http
- Verify:
  - curl http://localhost:8000/healthz
- Logs:
  - Controlled via LOG_LEVEL env var.

## Production build and scan

- Build:
  - make build
- Scan:
  - make scan
- SBOM:
  - make sbom
- Push:
  - make push

---

# Security hardening recommendations

- **Runtime**
  - Read-only filesystem with tmpfs `/tmp`.
  - Drop all Linux capabilities and enforce `no-new-privileges`.
  - Consider `network_mode: none` for stdio deployments.
  - Resource limits (CPU/memory) in orchestrator manifests.
- **Supply chain**
  - Use pinned base images, track with Dependabot/Renovate.
  - Require hashes in `requirements.txt`.
  - Maintain SBOMs and continuous image scanning.
- **Secrets**
  - Pass secrets via orchestrator secret managers; avoid env for sensitive values.

---

# Final validation

- The redesigned Dockerfiles address the non-root path and interpreter issues by using `/opt/venv` with PATH exported.
- Distinct developer and production images ensure fast iteration without compromising slim, secure prod images.
- Transport-agnostic entrypoint cleanly supports stdio and HTTP/SSE.
- Health checks are practical for HTTP and acknowledged as limited for stdio.
- Security posture is significantly improved with minimal privileges and immutable root FS.
- The plan is modular, verifiable, and ready to integrate into CI/CD with scanning and SBOM.

---
https://copilot.microsoft.com/shares/hmkpULfWgMMhxA8PT4RNL
