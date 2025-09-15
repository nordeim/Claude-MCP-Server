## Custom Kali Linux MCP Build Guide

### 0. One-Time Repo Bootstrap

```bash
git init custom-mcp-server && cd $_
cat > README.md <<'EOF'
# custom-mcp-server
Polyglot security-tool wrapper for Model Context Protocol (MCP).
EOF
git add . && git commit -m "chore: repo bootstrap"
```

---

### 1. Project Layout

```
custom-mcp-server/
├── src/
│   ├── mcp_server/
│   │   ├── __init__.py
│   │   ├── server.py          # asyncio entry-point
│   │   ├── base_tool.py       # abstract MCPBaseTool
│   │   └── tools/
│   │       ├── __init__.py
│   │       ├── nmap.py
│   │       ├── masscan.py
│   │       ├── gobuster.py
│   │       └── ...
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/robot/
├── Dockerfile
├── Dockerfile.distroless
├── docker-compose.dev.yml
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── .env.example
├── helm/
│   └── custom-mcp-server/
├── .github/workflows/ci.yml
├── scripts/
│   ├── generate_claude_config.py
│   └── dev_container_cli.sh
└── docs/
    ├── ARCHITECTURE.md
    └── RUNBOOK.md
```

---

### 2. Core Framework (`base_tool.py`)

```python
import asyncio, shlex, logging
from abc import ABC, abstractmethod
from pydantic import BaseModel, validator
from mcp.tool import tool

log = logging.getLogger(__name__)

class ToolInput(BaseModel):
    target: str
    extra_args: str = ""

    @validator('target')
    def must_be_rfc1918_or_lab(cls, v):
        import ipaddress, re
        if re.fullmatch(r"10\.\d+\.\d+\.\d+(/\d+)?", v): return v
        if re.fullmatch(r"192\.168\.\d+\.\d+(/\d+)?", v): return v
        if re.fullmatch(r"172\.(1[6-9]|2[0-9]|3[01])\.\d+\.\d+(/\d+)?", v): return v
        if v.endswith(".lab.internal"): return v
        raise ValueError("Target must be RFC-1918 or .lab.internal")

class ToolOutput(BaseModel):
    stdout: str
    stderr: str
    returncode: int

class MCPBaseTool(ABC):
    command_name: str

    @tool
    async def run(self, inp: ToolInput) -> ToolOutput:
        log.info("Running %s on %s", self.command_name, inp.target)
        cmd = [self.command_name, *shlex.split(inp.extra_args), inp.target]
        proc = await asyncio.create_subprocess_exec(
            *cmd, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE
        )
        out, err = await proc.communicate()
        return ToolOutput(stdout=out.decode(), stderr=err.decode(), returncode=proc.returncode)
```

---

### 3. Example Tool (`nmap.py`)

```python
from .base_tool import MCPBaseTool, ToolInput, ToolOutput

class NmapTool(MCPBaseTool):
    command_name = "nmap"
```

That’s it—schema, validation, and logging are inherited.

---

### 4. Server Entry-Point (`server.py`)

```python
import asyncio, logging, os
from mcp.server import MCPServer
from mcp.tools import NmapTool, MasscanTool, GobusterTool, NiktoTool, SqlmapTool, WfuzzTool, HydraTool, JohnTool, HashcatTool

logging.basicConfig(level=logging.INFO)

TOOLS = [
    NmapTool(), MasscanTool(), GobusterTool(), NiktoTool(),
    SqlmapTool(), WfuzzTool(), HydraTool(), JohnTool(), HashcatTool()
]

async def main():
    server = MCPServer(tools=TOOLS, transport=os.getenv("MCP_TRANSPORT", "stdio"))
    await server.serve()

if __name__ == "__main__":
    asyncio.run(main())
```

---

### 5. Multi-Stage Dockerfile (distroless)

```dockerfile
# ----- build -----
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ----- runtime -----
FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY src/ ./src
ENV PATH=/root/.local/bin:$PATH
ENV PYTHONUNBUFFERED=1
ENTRYPOINT ["python3", "-m", "mcp_server.server"]
```

Build & scan:

```bash
docker build -t ghcr.io/your-org/custom-mcp-server:$(git rev-parse --short HEAD) -f Dockerfile.distroless .
trivy image --exit-code 1 --severity HIGH,CRITICAL ghcr.io/your-org/custom-mcp-server:$(git rev-parse --short HEAD)
```

---

### 6. docker-compose.dev.yml (hot-reload)

```yaml
version: "3.9"
services:
  mcp:
    build:
      context: .
      dockerfile: Dockerfile.distroless
    volumes:
      - ./src:/app/src:ro
    environment:
      - MCP_TRANSPORT=stdio
    stdin_open: true
    tty: true
```

---

### 7. Generate Claude Desktop Config

```bash
python scripts/generate_claude_config.py \
  --transport stdio \
  --image ghcr.io/your-org/custom-mcp-server:$(git rev-parse --short HEAD) \
  --out ~/.config/Claude/claude_desktop_config.json
```

The script appends:

```json
"mcpServers": {
  "custom_security": {
    "command": "docker",
    "args": ["run", "--rm", "-i", "ghcr.io/your-org/custom-mcp-server:abc123"]
  }
}
```

---

### 8. End-to-End Robot Test (excerpt)

```robot
*** Settings ***
Library    Process
Library    OperatingSystem
Suite Setup    Start DVWA Container
Suite Teardown    Remove DVWA Container

*** Test Cases ***
Scan Discovers SQLi
    ${result}=    Run MCP Tool    nmap_scan    target=172.20.0.2    extra_args=-p 80
    Should Contain    ${result.stdout}    80/tcp open
    ${sql}=    Run MCP Tool    sqlmap_scan    target=http://172.20.0.2/login.php
    Should Contain    ${sql.stdout}    vulnerable
```

Run:  
```bash
robot -d reports/ tests/e2e/robot/
```

---

### 9. CI/CD (GitHub Actions) – condensed

```yaml
name: ci
on: [push, pull_request]
jobs:
  lint-test-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements-dev.txt
      - run: ruff check src/
      - run: mypy src/
      - run: pytest --cov=src --cov-report=xml
      - uses: docker/build-push-action@v5
        with:
          file: Dockerfile.distroless
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          push: true
          provenance: true
          sbom: true
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          exit-code: 1
          severity: HIGH,CRITICAL
```

---

### 10. Helm Deployment (lab only)

```bash
helm upgrade --install mcp-lab ./helm/custom-mcp-server \
  --set image.tag=$(git rev-parse --short HEAD) \
  --set networkPolicy.enabled=true \
  --set allowedCidrs="{10.0.0.0/8,192.168.0.0/16,172.16.0.0/12}"
```

---

### 11. Daily Usage Cheat-Sheet

| Intent | Prompt to Claude |
|--------|------------------|
| Quick ping-sweep | “Use nmap_scan on 192.168.56.0/24 with extra_args ‘-sn’ and show live hosts.” |
| Directory brute | “Gobuster target http://target.lab.internal with extra_args ‘-w /usr/share/wordlists/dirb/common.txt -x php’.” |
| Crack 100 hashes | “John_tool target /tmp/hashes.txt with extra_args ‘--wordlist=/opt/rockyou.txt’.” |
| Full chain | “Masscan 10.0.0.0/24 port 0-65535, then nmap-service-scan only open ports, then nikto any HTTP, finally give me a markdown report.” |

---

### 12. Maintenance & Extensibility

- Add a new tool?  
  1. Create `src/mcp_server/tools/newtool.py` inheriting `MCPBaseTool`.  
  2. Add class to `TOOLS` list in `server.py`.  
  3. Write unit test → CI auto-builds new image.  
- Update wordlists? Mount config-map; no rebuild needed.  
- Need GUI? The same image exposes `/streamable_http` transport—point Cursor, ChatGPT-Desktop, or n8n at it.

---

### 13. Final Check-Out

```bash
pre-commit run --all-files
helm lint helm/custom-mcp-server
robot tests/e2e/robot/
echo "Done—tag and release."
```

https://www.kimi.com/share/d34a13uf5ku9ms640640

You now own a **production-grade, audit-ready, CISO-friendly MCP server** that turns any MCP-enabled LLM into a conversational security analyst.  
Use wisely, scan only what you own, and keep extending.
