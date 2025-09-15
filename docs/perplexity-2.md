To set up MCP (Model Context Protocol) servers for Kali Linux security tools—and connect them to AI clients like Claude Desktop, Cursor, or others—follow these comprehensive steps. This approach gives secure, sandboxed, customizable access to the vast toolbox of Kali Linux for ethical hacking, penetration testing, and AI-assisted security tasks.

***

### Prerequisites

- **Kali Linux** (or another Linux system for MCP server hosting, physical or virtual)
- **Docker Desktop** (for containerized options, on any OS)[1][2]
- **Python 3.8+** (for some MCP server implementations)[3][4]
- **An MCP-enabled AI client** (Claude Desktop, Cursor, etc.)

***

### Step 1: Obtain or Build a Kali Linux MCP Server

There are two common deployment types:
#### Option A: Dockerized MCP Kali Server

- Clone a reputable Kali MCP server repository, e.g.:
  ```bash
  git clone https://github.com/dev-lu/PentestMCP.git
  cd PentestMCP
  ```
- Build and start the Docker container:
  ```bash
  docker build -t kali-mcp-server .
  docker run -d -p 8000:8000 kali-mcp-server
  ```
  Or use a prebuilt image if available (replace image name as appropriate):
  ```bash
  docker pull lobehub/kali-mcp-server:latest
  docker run -d -p 8000:8000 lobehub/kali-mcp-server
  ```
  This exposes the MCP server on port 8000 for local or remote AI agents.[5][6][2][1]

#### Option B: Native Python MCP Server

- Clone another MCP Kali server repository if running natively (no Docker):
  ```bash
  git clone https://github.com/Wh0am123/MCP-Kali-Server.git
  cd MCP-Kali-Server
  python3 kali_server.py
  ```
  - This typically starts the API on port 5000 by default.[4][7][8]

***

### Step 2: Expose and Test the MCP API

- The container or script exposes HTTP or SSE endpoints (e.g., `http://localhost:8000/sse` or `http://localhost:5000/`).
- Check running containers with:
  ```bash
  docker ps
  ```
- Test the endpoint in your browser or with curl:
  ```bash
  curl http://localhost:8000/sse
  ```

***

### Step 3: Configure the AI MCP Client (e.g., Claude Desktop)

- Modify the configuration file (example for Claude Desktop on Windows):
  ```json
  {
    "mcpServers": {
      "kali_mcp": {
        "command": "docker",
        "args": [
          "run", "-p", "8000:8000", "kali-mcp-server"
        ],
        "url": "http://localhost:8000/sse",
        "transport": "sse"
      }
    }
  }
  ```
  - On Linux, point to the local script/APIs: `"command": "python3", "args": ["/path/to/mcp_server.py", "--server", "http://LINUX_IP:5000/"]`.[6][7][1][4]

- Restart the AI client to enable and verify connection to the MCP server.
- Use the client’s UI/settings to add the MCP server, if available (refer to app-specific documentation).[9][10][6]

***

### Step 4: Use Security Tools via MCP

- Once integrated, the AI client should detect the exposed Kali tools available via the MCP server (like nmap, ffuf, wpscan, nikto, gobuster, etc.).[7][1][4]
- Example: Ask the AI client to run a network scan:
  ```
  /run nmap -F target.domain.com
  ```
- The AI will orchestrate the command and return results in the chat.

***

### Security & Management Tips

- Always limit the tools and scope for the MCP server using allowlists or yaml config as warranted.[2][1]
- Limit inbound connections to the MCP server API (firewall or Docker network config).
- For production, consider HTTPS endpoints and authentication.[1][5]
- Consult the official/project-specific READMEs for advanced configuration, updating, or building custom MCP servers easily with templated prompt builders.[11][9][6]

***

### Official and Open-Source Resources

- Docker MCP toolkit: https://hub.docker.com/mcp[12][1]
- MCP server for Kali: https://github.com/Wh0am123/MCP-Kali-Server, https://github.com/dev-lu/PentestMCP[4][7][2]
- Community catalogs: https://mcpmarket.com/server/kali-tools, https://playbooks.com/mcp/vermaxvr-kali-linux-security-tools[13][14][6][1]

This workflow lets you safely, reproducibly, and programmatically use the full range of Kali Linux tools via standard MCP protocol in your AI-driven security workflow.[6][7][2][1][4]

[1](https://lobehub.com/mcp/k3nn3dy-ai-kali-mcp)
[2](https://github.com/dev-lu/PentestMCP)
[3](https://github.com/whit3rabbit0/project_astro)
[4](https://github.com/Wh0am123/MCP-Kali-Server)
[5](https://www.mcpevals.io/blog/setup-local-mcp-servers-with-docker)
[6](https://www.magicslides.app/mcps/ccq1-kali-security-tools)
[7](https://playbooks.com/mcp/wh0am123-kali-penetration-testing)
[8](https://www.kali.org/tools/mcp-kali-server/)
[9](https://github.com/theNetworkChuck/docker-mcp-tutorial)
[10](https://en.bioerrorlog.work/entry/connect-claude-desktop-to-mcp-server)
[11](https://www.docker.com/blog/build-to-prod-mcp-servers-with-docker/)
[12](https://www.docker.com/blog/introducing-docker-hub-mcp-server/)
[13](https://playbooks.com/mcp/vermaxvr-kali-linux-security-tools)
[14](https://mcpmarket.com/server/kali-tools)
[15](https://mcp-hunt.com/mcp/mcp-kali-server)
[16](https://gitlab.com/kalilinux/packages/python-mcp)
[17](https://dev.to/pradumnasaraf/run-mcp-servers-in-seconds-with-docker-1ik5)
[18](https://snyk.io/articles/10-mcp-servers-for-cybersecurity-professionals-and-elite-hackers/)
[19](https://snyk.io/articles/how-to-run-mcp-servers-with-docker/)
[20](https://dev.to/hax/how-to-install-all-the-tools-you-need-and-want-in-kali-linux-with-one-command-from-top-10-to-default-to-everything-5221)
