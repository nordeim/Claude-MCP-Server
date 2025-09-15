# Model context protocol workflow overview

MCP connects an AI client (such as Claude Desktop) to real tools via standardized “servers” that expose capabilities (APIs, local files, databases, etc.). In practice, you run MCP servers locally with Docker, then point your client to those servers so the AI can invoke their tools directly and perform grounded, auditable actions. The video demonstrates this end-to-end: running MCP servers in containers, using Claude Desktop as the client, and leveraging Docker’s MCP Catalog to discover and run servers quickly.

Docker Desktop streamlines local container orchestration, networking, and updates, making it a practical base for running multiple MCP servers side-by-side. Using Docker’s curated MCP Catalog, you can pull vetted servers (e.g., GitHub, databases, search tools) and compose them into powerful, safe workflows your AI client can call on demand.

---

# Prerequisites and installation

## Claude Desktop app

- **Download:** Go to the official download page and choose macOS, Windows (x64), or Windows (arm64) installers. Install with default options; the app runs in your tray/menu bar for quick access without tab switching.
- **First launch:** Sign in to your Anthropic account. Keep the app running so it can connect to MCP servers later.

## Docker Desktop

- **Install:** Download and install Docker Desktop for your OS. It provides a full container environment, GUI, Kubernetes option, and seamless local networking and volume mounts.
- **Verify:** After installation, open Docker Desktop and ensure it shows “Running.” This confirms the Docker Engine and networking are healthy.

## Docker MCP catalog

- **Browse catalog:** Explore MCP servers on Docker Hub’s MCP catalog to find official and community servers (e.g., GitHub, Notion, MongoDB, DuckDuckGo, Fetch). Each listing documents available tools and configuration variables.
- **Select servers:** Identify the MCP servers relevant to your workflow (for example, GitHub for repo automation, “Fetch” for URL retrieval, or a database server for read-only queries).

---

# Run MCP servers with Docker

Below are generic, production-friendly patterns you can adapt to any MCP server from the Docker MCP Catalog. Replace image names and environment variables with those specified on each server’s page.

### 1. Create a dedicated Docker network

- **Command:**
  ```
  docker network create mcp-net
  ```
- **Why:** Keeps MCP servers isolated yet discoverable by name from the client side (when proxied or referenced locally).

### 2. Start one or more MCP servers

- **Example GitHub server (illustrative):**
  ```
  docker run -d --name mcp-github --network mcp-net \
    -e GITHUB_TOKEN=ghp_your_token_here \
    -p 4010:4010 \
    someorg/mcp-github:latest
  ```
- **Example Fetch server (illustrative):**
  ```
  docker run -d --name mcp-fetch --network mcp-net \
    -p 4020:4020 \
    someorg/mcp-fetch:latest
  ```
- **Notes:**
  - **Ports:** Map each server to a unique host port.
  - **Env vars:** Supply required tokens, read-only credentials, or API keys as documented by the server.
  - **Persistence:** Use volumes if a server supports local state.

These patterns mirror the video’s approach of running multiple MCP servers locally with Docker and exposing them to a client like Claude Desktop.

### 3. Health-check containers

- **List containers:**
  ```
  docker ps
  ```
- **Logs (per container):**
  ```
  docker logs -f mcp-github
  ```
- **Connectivity:** Ensure each container is Listening on its port and reports “ready” in logs.

---

# Configure Claude Desktop to use MCP servers

Claude Desktop can act as an MCP client. You’ll configure it to point at your locally running MCP servers so Claude can call their tools during a conversation.

## Steps

1. **Open settings in Claude Desktop:** Access preferences for tool or MCP configuration. The video demonstrates Claude Desktop connecting to MCP servers as a client alongside other clients like LM Studio and Cursor.
2. **Add MCP server entries:** For each server you run, add:
   - **Name:** A friendly label (e.g., “GitHub MCP”).
   - **Endpoint/Port:** The local URL (e.g., http://localhost:4010).
   - **Auth headers (if applicable):** Some MCP servers rely on environment variables inside the container; others may require tokens at the client layer. Follow the server’s docs in the Docker MCP Catalog.
3. **Test connection:** Use any built-in “Test” or simply invoke a trivial tool call in a chat. If logs show incoming requests, the wiring is correct.

Tip: The video also discusses the MCP Gateway concept for routing and composition; you can later introduce a gateway if you need advanced routing or orchestration, but it’s optional for a basic local setup.

---

# Executing tasks with Claude and MCP tools

Once Claude Desktop is connected, you can prompt Claude to use specific tools. The video shows how the client lists MCP tools and how you can steer it to call them for grounded actions.

## Prompting patterns

- **Direct tool request:**
  - “Use the GitHub MCP tool to list open issues in repo org/project, then summarize the top three by severity.”
- **Chained actions:**
  - “Call the Fetch MCP tool to retrieve this URL, extract key points, then draft a 200-word summary and propose two follow-up actions.”
- **Multi-tool flow:**
  - “Search with DuckDuckGo MCP for the latest docs on X, Fetch the most authoritative source, and produce a concise implementation plan with references.”

If a tool requires arguments (like a repo name or query), provide them explicitly. The Docker MCP Catalog pages document tool names and parameters so you can reference them precisely in your prompts.

---

https://copilot.microsoft.com/shares/iqGJbizjQXxPHs54nF1EN
