# Model Context Protocol (MCP) Workflow for AI-Driven Ethical Hacking with Kali Linux

> **Status:** September 2025  
> **Audience:** Practitioners who want to combine Claude, Docker containers and Kali Linux tools into a single, conversational penetration-testing assistant.

---

## 1. Executive Summary

The **Model Context Protocol (MCP)** is an open specification created by Anthropic that turns large-language-model (LLM) clients into *agentic systems* capable of calling real tools.  
By containerising Kali Linux security tools as **MCP servers**, you can ask Claude (or any MCP-enabled client) in plain English to:

- “Nmap-scan 192.168.1.0/24 and give me a table of live hosts.”  
- “SQL-map the login form at http://testphp.vulnweb.com and report injection flaws.”  
- “Crack this MD5 hash with john-the-ripper using the rockyou wordlist.”

All commands run inside **sandboxed Docker containers**; no tool is installed on the host, and secrets never leave the container runtime.  
The workflow is 100 % **local-first**, GDPR-friendly and CI/CD-ready.

---

## 2. Conceptual Architecture

```
┌-----------------┐
│ Claude Desktop  │  ←— 5) natural-language prompt
└--------┬--------┘
         │ JSON-RPC over stdio
┌--------┴--------┐
│  MCP Gateway    │  ←— Docker Desktop “MCP Toolkit” plug-in
│  (docker/mcp-gateway) │
└--------┬--------┘
         │ discovers & routes
┌--------┴--------┐     ┌-------------------------┐
│ Kali-MCP-Server │ … │ GitHub-MCP-Server, etc. │
│ (kali-linux)    │   │ (optional extra tools)  │
└-----------------┘     └-------------------------┘
```

Key design wins  
- *Dynamic tool catalogue* – add/remove containers without touching Claude’s config.  
- *Secrets isolation* – PATs, API keys and SSH creds live in **Docker secrets**, invisible to the LLM.  
- *Reproducibility* – every tool ships as a versioned image; same scan = same result everywhere.

---

## 3. Prerequisites & System Requirements

| Component       | Minimum Version | Download link |
|-----------------|-------------|-------------|
| OS              | Windows 10 22H2 / macOS 13 / any modern Linux | — |
| Docker Desktop  | 4.35 (includes Docker Engine 27) | [https://dockr.ly/3HgjIGj](https://dockr.ly/3HgjIGj) |
| Claude Desktop  | 0.8.0+ (MCP tier) | [https://claude.ai/download](https://claude.ai/download) |
| Git             | 2.40+ | [https://git-scm.com](https://git-scm.com) |

Hardware  
- 4 CPU cores, 8 GB RAM, 20 GB free disk (for Kali + images).  
- Apple-Silicon Macs: use `--platform linux/arm64` builds (official images already multi-arch).

---

## 4. Step-by-Step Installation Guide

### 4.1 Install Docker Desktop

**Windows** (PowerShell **as Administrator**)  
```powershell
# 1. Download
Invoke-WebRequest -Uri https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe -OutFile "$env:TEMP\DockerDesktop.exe"
# 2. Install with WSL2 backend
Start-Process "$env:TEMP\DockerDesktop.exe" -ArgumentList "install","--quiet","--accept-license" -Wait
# 3. Add your user to the docker-users group (omit if already admin)
net localgroup docker-users $env:USERNAME /add
# 4. Reboot
Restart-Computer
```

**macOS** (Homebrew)  
```bash
brew install --cask docker
# Launch from /Applications once to complete setup
```

**Linux (Ubuntu)**  
```bash
curl -fsSL https://get.docker.com | sudo bash
sudo usermod -aG docker $USER && newgrp docker
```

Verify  
```bash
docker version          # Client & server ≥ 27
docker compose version  ≥ v2.29
```

---

### 4.2 Install Claude Desktop

1. Download the pkg/msi from [https://claude.ai/download](https://claude.ai/download)  
2. Sign in with an account that has **Claude Pro** (MCP functionality is Pro-only).  
3. **Completely quit** the app (tray icon → Quit) – important later when we reload config.

---

### 4.3 Enable Docker “MCP Toolkit” (Gateway)

Docker Desktop ≥ 4.35 ships with the **MCP Toolkit** extension. Enable it:

1. Open Docker Desktop → **Settings → Extensions → Allow system extensions**.  
2. **Extensions Marketplace** → search “MCP Toolkit” → Install.  
3. You will now see a new sidebar item **“MCP Toolkit”** (icon: 🛠️).

---

### 4.4 Pull or Build the Kali-MCP-Server

**Option A – fastest (pre-built)**  
```bash
docker pull ghcr.io/dev-lu/pentestmcp:latest
```

**Option B – build locally (customise tools)**  
```bash
git clone https://github.com/dev-lu/PentestMCP.git
cd PentestMCP
docker build -t pentestmcp:latest .
```

The container embeds:  
- nmap, gobuster, sqlmap, hydra, john, nikto, aircrack-ng, metasploit-framework  
- Python 3.12 + FastMCP library  
- Non-root user `mcp` (UID 1000) for safety

---

### 4.5 Register the Server with the Gateway

1. Open **Docker Desktop → MCP Toolkit → Catalog tab**  
2. Click **“Add custom server”** → fill:
   - Name: `kali-sec`  
   - Image: `ghcr.io/dev-lu/pentestmcp:latest` (or your local tag)  
   - Command: `python -m mcp_server` (default ENTRYPOINT)  
   - Environment variables (optional):  
     `EXTRA_TOOLS=dirb,whatweb`  
3. **Save & Start**. The gateway will pull/run the container and expose its tools.

---

### 4.6 Connect Claude Desktop to the Gateway

1. **MCP Toolkit → Clients → “Claude Desktop” → Connect**  
   This auto-writes the JSON snippet below into Claude’s config file:

**Windows**  
`%APPDATA%\Claude\claude_desktop_config.json`

**macOS / Linux**  
`~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "docker-gateway": {
      "command": "docker",
      "args": ["mcp", "gateway", "run"],
      "env": {
        "DOCKER_GATEWAY_PORT": "8811"
      }
    }
  }
}
```

2. **Quit and restart** Claude Desktop (not just close window).  
3. In the chat box you should now see a **🔨 tools icon** – click it to list ≈ 30 tools including `run_nmap`, `run_sqlmap`, `run_gobuster`, etc.

---

## 5. Ethical Hacking Walk-through Example

**Scenario**  
You have **written authorisation** to assess `10.0.0.0/24` in a lab.  
Goal: discover live hosts, enumerate web servers, test for SQL-i, capture evidence.

| Step | Human prompt to Claude | MCP tool(s) invoked | Typical container stdout returned |
|------|------------------------|---------------------|------------------------------------|
| 1 | “Use nmap to ping-scan 10.0.0.0/24 and show only live IPs.” | `run_nmap: "-sn 10.0.0.0/24"` | `10.0.0.7`, `10.0.0.23` |
| 2 | “Run a service/version scan on the live targets.” | `run_nmap: "-sV -O 10.0.0.7,10.0.0.23"` | 7: ssh,http; 23: mysql,http |
| 3 | “Gobuster common directories on 10.0.0.7 port 80.” | `run_gobuster: "-u http://10.0.0.7 -w /usr/share/wordlists/dirb/common.txt"` | `/admin`, `/backup` |
| 4 | “Check the /admin page with nikto.” | `run_nikto: "-h http://10.0.0.7/admin"` | outdated Apache, XSS possible |
| 5 | “Test parameter id in http://10.0.0.7/profile.php?id=1 for SQL injection.” | `run_sqlmap: "-u 'http://10.0.0.7/profile.php?id=1' --batch --level=1"` | `Parameter id is vulnerable (union-based)` |
| 6 | “Generate a markdown report with all findings.” | (Claude composes) | Report saved to `~/Reports/10.0.0-assess-2025-09-16.md` |

All commands execute **inside the Kali container**; only JSON results reach Claude.  
No traffic originates from your host OS, keeping your workstation safe.

---

## 6. Advanced Usage Patterns

### 6.1 Chained tool calls (Claude can auto-pipe)

> “For every open port found by nmap, run nikto and then sqlmap if HTTP is detected.”

Claude will:  
1. Call `run_nmap` → parse JSON → loop over services.  
2. Invoke `run_nikto` when service=HTTP.  
3. Invoke `run_sqlmap` if nikto flags a PHP parameter.

### 6.2 Custom wordlists & scripts

Mount a local volume into the container:

```bash
docker run -d --name kali-sec \
  -v $HOME/wordlists:/opt/wordlists:ro \
  -p 8000 ghcr.io/dev-lu/pentestmcp
```

Reference inside prompt:  
“Use gobuster with my wordlist `/opt/wordlists/raft-small-words.txt`…”

### 6.3 Secrets management (API tokens, SSH keys)

1. **Docker Desktop → Settings → Secrets → Create**  
   Name: `github_pat` Value: `ghp_xxx`  
2. In MCP Toolkit tick **“Use Docker secrets”** → select `github_pat`.  
3. Inside container the secret appears as file `/run/secrets/github_pat` (0600) – no env var needed.

### 6.4 CI / nightly scans

A one-liner in GitHub Actions:

```yaml
- name: Run MCP gateway scan
  run: |
    docker run --rm -v $PWD:/data ghcr.io/dev-lu/pentestmcp \
      python -m mcp_server client --tool run_nmap --args "-oX /data/nmap.xml 10.0.0.0/24"
```

---

## 7. Security & Compliance Considerations

- **Scope validation** – the demo server uses an **allow-list regex** (`^([0-9./,\-]+)$`) for IP arguments; adapt to your ranges.  
- **Runtime sandbox** – container runs as non-root, `--cap-drop ALL`, `--network=bridge` only.  
- **Log retention** – gateway stores JSON-RPC logs under `~/.docker/mcp/logs` – rotate or forward to SIEM.  
- **Responsible disclosure** – only target assets you **own** or have **written permission** to test.  
- **ISO-27001 alignment** – container images pinned by SHA-256; config stored in Git; pipeline gates on `ramparts` security scan (see below).

---

## 8. Troubleshooting Cheat-Sheet

| Symptom | Fix |
|---------|-----|
| Claude shows no tools | Quit app completely → restart; verify `claude_desktop_config.json` valid JSON |
| `docker: command not found` in Claude | Ensure Docker Desktop is **running** and `docker` is in system PATH |
| `permission denied` on socket | Linux: `sudo usermod -aG docker $USER` and re-login |
| Tool returns empty | Check container logs: `docker logs <container-id>` – often missing wordlist path |
| Scan hangs | Add timeout argument: `run_nmap: "--max-scan-delay 20ms --host-timeout 5m"` |

---

## 9. Hardening & Blue-Team Add-ons

- **Ramparts** – scan your MCP servers for misconfig:  
  `docker run --rm getjavelin/ramparts scan-config --report`  
- **OpenPolicyAgent** – inject OPA side-car to gate each JSON-RPC call.  
- **Falco** – runtime sensor detects suspicious syscalls inside Kali container.  
- **Docker Bench** – weekly compliance check:  
  `docker run --rm --pid host -v /var/run/docker.sock:/var/run/docker.sock docker/docker-bench-security`

---

## 10. Future Road-map & Community Repos

Upstream projects to watch:  
- [https://github.com/docker/mcp-gateway](https://github.com/docker/mcp-gateway) – gateway itself  
- [https://github.com/dev-lu/PentestMCP](https://github.com/dev-lu/PentestMCP) – Kali server used here  
- [https://gitlab.com/kalilinux/documentation/kali-tools](https://gitlab.com/kalilinux/documentation/kali-tools) – official tool docs

Planned enhancements (open PRs welcome):  
- OWASP-ZAP and Metasploit RPC integration  
- MITRE-ATT&CK tagging of tool outputs  
- JSON schema validation for every tool argument  
- Parallel scan orchestration via `docker compose scale`

---

## 11. TL;DR

1. Install **Docker Desktop** → enable **MCP Toolkit** extension.  
2. `docker pull ghcr.io/dev-lu/pentestmcp`  
3. **MCP Toolkit → Add server → Start**.  
4. Connect **Claude Desktop** (one JSON stanza).  
5. Chat: *“Nmap-scan my lab, SQL-map what you find, and write me a report.”* – done.

You now have a **conversational, audit-ready, containerised ethical-hacking assistant** running on your laptop.  
Happy (responsible) hacking!

https://www.kimi.com/share/d34980pe3tpvqb13frhg
