# directory structure
```
/mcp-deployment/
  ├── docker-compose.yml
  └── .env
```

# .env
```env
# GITHUB CONFIGURATION
GH_PAT="ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
GH_REPO_OWNER="YourOrg"
GH_REPO_NAME="YourRepo"

# MCP CONFIGURATION
GIT_MCP_PORT=8081
LINTER_MCP_PORT=8082
```

# docker-compose.yml
```yml
version: '3.8'

services:
  # ----------------------------------------------------
  # 1. GIT MCP Server (Handles branch, commit, PR creation)
  # ----------------------------------------------------
  git-mcp:
    image: mcp/git-server:latest
    container_name: git_mcp_server
    ports:
      - "${GIT_MCP_PORT}:8080"
    environment:
      # Inject secrets securely
      - GITHUB_PAT=${GH_PAT}
      - REPO_OWNER=${GH_REPO_OWNER}
      - REPO_NAME=${GH_REPO_NAME}
    volumes:
      # Mount the local host directory for file access by Git operations
      - ./repo_data:/app/repo_data
    restart: always

  # ----------------------------------------------------
  # 2. LINTER MCP Server (Handles static analysis)
  # ----------------------------------------------------
  linter-mcp:
    image: mcp/linter-server:latest
    container_name: linter_mcp_server
    ports:
      - "${LINTER_MCP_PORT}:8080"
    volumes:
      # Access to the code files being analyzed
      - ./repo_data:/app/code
    restart: always
```

```json
{
  "mcpServers": {
    "git_mcp": {
      "transport": "streamable_http",
      "url": "http://localhost:8081/mcp"
    },
    "linter_mcp": {
      "transport": "streamable_http",
      "url": "http://localhost:8082/mcp"
    }
  }
}
```

```yml
name: CI Verification of Claude PR

on:
  pull_request:
    branches: [ main ]

jobs:
  static_analysis:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4
      with:
        # Ensure we checkout the exact commit created by the MCP server
        ref: ${{ github.event.pull_request.head.sha }}

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.x'

    - name: Install dependencies and linter
      run: |
        pip install pylint

    - name: Run Static Code Analysis (Verifies Linter MCP)
      # This command must match the check run by the Linter MCP server
      run: |
        pylint src/service.py
```

```
graph TD
    A[Developer Prompt] -->|1. Request Fix| B(Claude Desktop Client);

    subgraph Local Execution (MCP Servers)
        B -->|2. Invoke Tool| C{git_mcp: create_branch};
        B -->|3. Invoke Tool| D{linter_mcp: check_file};
        B -->|4. Apply Fix| E{git_mcp: commit_changes};
        B -->|5. Open PR| F{git_mcp: create_pr};
        C & D & E & F --> B;
    end

    B -->|6. Results & Decision| G[Human Approval];

    G -->|7. Confirmed Action| F;

    F -->|8. Push PR| H[GitHub Repository];

    H -->|9. PR Created| I(GitHub Actions CI);

    I -->|10. Verify Changes| J[CI Success/Failure Report];

    J -->|11. Feedback| B;
    H -->|12. Final Merge| K[Main Branch];
```
