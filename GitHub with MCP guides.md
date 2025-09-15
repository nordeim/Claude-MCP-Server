Perfect — I’ll break this down into **three standalone deep-dive howto guides**, one for each of the requested tracks (A, B, C). Each guide is self-contained, detailed, and production-ready.

---

# **Guide A — Running GitHub MCP + Git/Shell MCP with Docker Compose**

### 1. Directory layout

```
mcp-setup/
  ├─ docker-compose.yml
  ├─ .env
  └─ secrets/
       └─ github_pat.txt
```

### 2. `.env` file (safe variable injection)

```dotenv
# .env
GITHUB_MCP_IMAGE=ghcr.io/mcp/github-mcp:latest
GIT_MCP_IMAGE=ghcr.io/mcp/git-mcp:latest

# host paths
WORKSPACE_PATH=./workspace

# secrets paths
GITHUB_PAT_FILE=./secrets/github_pat.txt
```

> 🔐 *Tip:* never commit `.env` or `secrets/` to git. Add to `.gitignore`.

### 3. Docker secrets file

Create `secrets/github_pat.txt` containing only your GitHub PAT with repo/issue/PR scopes.

### 4. `docker-compose.yml`

```yaml
version: "3.9"

services:
  github-mcp:
    image: ${GITHUB_MCP_IMAGE}
    container_name: github-mcp
    ports:
      - "8787:8787"   # MCP endpoint
    secrets:
      - github_pat
    environment:
      MCP_BIND: "0.0.0.0:8787"
      GITHUB_PAT_FILE: "/run/secrets/github_pat"
    restart: unless-stopped

  git-mcp:
    image: ${GIT_MCP_IMAGE}
    container_name: git-mcp
    volumes:
      - ${WORKSPACE_PATH}:/workspace
    ports:
      - "8788:8788"
    environment:
      MCP_BIND: "0.0.0.0:8788"
    restart: unless-stopped

secrets:
  github_pat:
    file: ${GITHUB_PAT_FILE}
```

### 5. Start stack

```bash
docker compose up -d
docker ps
```

### 6. Configure Claude Desktop

* Open **Claude Desktop → Settings → MCP Servers → Add server**
* Add `http://127.0.0.1:8787` for GitHub MCP
* Add `http://127.0.0.1:8788` for Git MCP
* Restart Claude Desktop

Claude will now discover tool names like:

* `github.create_issue`, `github.create_ref`, `github.create_pull_request`
* `git.clone`, `git.checkout_new`, `git.commit_and_push`

---

# **Guide B — Ready-to-paste Claude Desktop Prompts (End-to-End Flow)**

Use these prompts in sequence. Replace placeholders (e.g., `<ISSUE_NUMBER>`, `<SHA>`).

### 1. Create GitHub Issue

```
[TOOL: github.create_issue]
repo: "nordeim/Frontend"
title: "Bug: pagination shows same products on all pages"
body: "Steps:\n1. Go to /products\n2. Switch to page 2\nExpected: new results\nActual: duplicate results"
labels: ["bug"]
[/TOOL]
```

### 2. Create Branch (via refs API)

```
[TOOL: github.get_ref]
repo: "nordeim/Frontend"
ref: "heads/main"
[/TOOL]

[TOOL: github.create_ref]
repo: "nordeim/Frontend"
ref: "refs/heads/issue-<ISSUE_NUMBER>-fix-pagination"
sha: "<SHA_FROM_PREVIOUS>"
[/TOOL]
```

### 3. Clone Repo Locally

```
[TOOL: git.clone]
repo_url: "https://github.com/nordeim/Frontend.git"
target_dir: "/workspace/Frontend"
[/TOOL]
```

### 4. Checkout Branch

```
[TOOL: git.checkout_new]
path: "/workspace/Frontend"
branch: "issue-<ISSUE_NUMBER>-fix-pagination"
base: "main"
[/TOOL]
```

### 5. Commit & Push Fix

```
[TOOL: git.commit_and_push]
path: "/workspace/Frontend"
branch: "issue-<ISSUE_NUMBER>-fix-pagination"
commit_message: "fix(pagination): correct offset calculation (fixes #<ISSUE_NUMBER>)"
remote: "origin"
[/TOOL]
```

### 6. Open Pull Request

```
[TOOL: github.create_pull_request]
repo: "nordeim/Frontend"
title: "Fix: pagination shows duplicate items (closes #<ISSUE_NUMBER>)"
body: "Corrects offset calculation. Verified locally with npm test."
head: "issue-<ISSUE_NUMBER>-fix-pagination"
base: "main"
[/TOOL]
```

---

# **Guide C — GitHub Actions CI Template for MCP-driven PRs**

Save as `.github/workflows/ci.yml` in repo.

```yaml
name: CI

on:
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm install

      - name: Run linter
        run: npm run lint --if-present

      - name: Run tests
        run: npm test -- --ci --coverage

      - name: Build project
        run: npm run build --if-present

      - name: Upload test coverage
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage
```

### Features:

* Runs on every PR targeting `main`.
* Installs Node.js v20, runs `npm install`, lint, test, build.
* Uploads coverage as artifact for inspection.
* Extensible: add security scan (e.g., `npm audit`, `trivy`), or deploy jobs if needed.

---

✅ So now you’ve got:

* **Guide A**: secure, real-world Docker Compose deployment for MCP servers.
* **Guide B**: copy-paste Claude Desktop prompts for the full GitHub issue → branch → commit → PR flow.
* **Guide C**: GitHub Actions CI pipeline to validate every MCP-created PR automatically.

---

Would you like me to now *merge these three guides into a single “master playbook”* (with cross-references, prerequisites, and flow diagrams), so your team can use it as a one-stop **Claude + MCP GitHub Contribution Manual**?
