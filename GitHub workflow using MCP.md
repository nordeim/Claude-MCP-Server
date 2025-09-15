Nice — I’ll give you a **clear, repeatable workflow** and a **detailed step-by-step guide** for opening an issue, creating a bug-fix branch, and submitting a pull request against the public repo `https://github.com/nordeim/Frontend` **using Claude Desktop + MCP servers**. I’ll include:

* the required MCP servers and permissions
* secure token handling
* exact, copy-pasteable Claude prompts (showing how to target MCP tools)
* fallback CLI / API commands (manual alternative)
* verification, logging and troubleshooting tips

I’ve validated the repo and the key GitHub REST endpoints and PAT guidance I reference below. Citations for the load-bearing facts are included inline.

---

# Short workflow overview (the 3-step goal)

1. *Create an issue on GitHub* describing the bug (this documents the problem & becomes the canonical task).
2. *Create a branch* off `main` for the fix (branch naming includes the issue number).
3. *Make the fix, push the branch, and open a pull request (PR)* targeting `main`.

We’ll orchestrate this primarily by instructing **Claude Desktop** to call two MCP services:

* **GitHub MCP** — an MCP server that calls the GitHub REST API (create issues, refs, PRs).
* **Git MCP / Shell MCP** — an MCP server that can run local `git` commands (clone, commit, push) or a `git`-aware MCP that manipulates the repo on disk.

(If you prefer a single path, you can do everything through the GitHub MCP using the GitHub REST API — I’ll show both options.) MCP background & spec: MCP is an open protocol for connecting LLM clients to tool servers. ([Anthropic][1])

Repository confirmed public: `nordeim/Frontend` (default branch `main`). ([GitHub][2])

GitHub REST endpoints we’ll use (create issue, create git ref, create pull request) — official docs. ([GitHub Docs][3])

---

# Pre-requisites (what to prepare)

*Account & permissions*

* A GitHub account with push/PR permissions on the repo (fork+PR flow works for external contributors).
* A **Personal Access Token (PAT)** or equivalent with repo permissions (create issues, push branches, create PRs). See GitHub PAT docs for scopes and creation. *Use fine-grained tokens where possible.* ([GitHub Docs][4])

*Local environment & MCP*

* Claude Desktop installed and signed in.
* Docker Desktop (or another runtime) to run MCP server containers locally.
* MCP server images you’ll use:

  * **GitHub MCP**: MCP server that issues authenticated calls to GitHub REST API.
  * **Git / Shell MCP**: MCP server that can execute `git` commands in a workspace (or you can run `git` locally yourself).
  * Use only trusted MCP images or build your own. (MCP is an open standard; Docker Hub has MCP images and the MCP spec is public.) ([Docker Hub][5])

*Secrets handling*

* Store PATs securely (OS keychain, Docker secrets, or a secrets manager). Avoid pasting tokens into chat or into the Claude history. Configure MCP servers to read tokens from environment variables or mounted secrets.

---

# High-level decision: two supported execution models

A. **MCP-driven remote (GitHub API) flow** — Claude + *GitHub MCP* will call GitHub REST API endpoints to create the issue, create the branch (via Git refs API) and create the PR. This requires the GitHub MCP be configured with a PAT that has repo access. (No local code changes required — all operations happen through GitHub APIs.) ([GitHub Docs][3])

B. **Local repo flow (git + GitHub MCP for PR)** — Claude uses a *Git/Shell MCP* to run local `git` commands (clone repo, create branch, patch files, commit, push). Then either the GitHub MCP or the GitHub web UI is used to open the PR. This is useful when the fix needs local build/test before PR.

I’ll provide a step-by-step for both. Pick the one you want to run; both include explicit Claude prompts.

---

# Detailed, step-by-step guide — Model A: API-only (GitHub MCP)

> Summary: Claude will call GitHub API endpoints using the MCP that wraps authenticated API calls. This is fast and works for small edits (e.g., file edits via the Git data API) or repo operations (create issue, create ref, create PR).

### 0) Configure the GitHub MCP

1. Run the GitHub MCP container locally (or use your hosted MCP). Configure it with the PAT (from a secrets store). **Important:** bind to `127.0.0.1` or a private network and require authentication tokens (don’t expose publicly).
2. Confirm the MCP exposes a discovery endpoint that Claude Desktop can read so Claude sees the tools: e.g., `http://127.0.0.1:8787` and lists tools: `github.create_issue`, `github.create_ref`, `github.create_pull_request`, etc. (Tool naming will depend on the MCP server implementation.)

> Verification: From the MCP host, curl the capability endpoint or check container logs to ensure it advertises those capabilities to MCP clients. (MCP spec / servers advertise capabilities per the protocol.) ([Anthropic][1])

---

### 1) Create the issue (via Claude prompt)

**Goal:** create an issue on `nordeim/Frontend` that documents the bug.

**Example Claude Desktop prompt** (tell Claude to use the GitHub MCP tool named `github.create_issue` — replace the tool name with whatever your MCP advertises):

```
[TOOL: github.create_issue]
repo: "nordeim/Frontend"
title: "Bug: pagination shows same products on all pages"
body: "Steps to reproduce:\n1) Open /products page\n2) Click page 2\nObserved: same products appear on every page. Expected: different product set per page.\nEnvironment: local dev, commit hash <insert if known>.\nPlease create the issue and return the new issue number and URL only."
[/TOOL]
```

*What happens:* Claude will call the MCP tool `github.create_issue` (which issues a POST to `POST /repos/:owner/:repo/issues`). If successful, the MCP returns the issue URL and number, which Claude will show. (GitHub issues API docs.) ([GitHub Docs][3])

*Manual fallback (curl):* if you want to verify manually, this is the equivalent API call (replace `TOKEN`):

```bash
curl -X POST -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/nordeim/Frontend/issues \
  -d '{"title":"Bug: pagination shows same products on all pages","body":"Steps to reproduce:\n1) ..."}'
```

---

### 2) Create a bug-fix branch (via Git refs API)

**Approach A1 (API):** Create a new git ref using the Git data / refs API. This requires the SHA of the base commit (usually the `main` branch head). Steps:

1. Use GitHub MCP to *get* the `main` ref to obtain the commit SHA (`GET /repos/:owner/:repo/git/ref/heads/main`).
2. Use GitHub MCP to *create* a new ref (`POST /repos/:owner/:repo/git/refs`) with body like `{"ref":"refs/heads/issue-123-fix-pagination","sha":"<main-sha>"}`. (Git refs API docs.) ([GitHub Docs][6])

**Example Claude prompt** (tool calls chained or separate — show both steps):

```
[TOOL: github.get_ref]
repo: "nordeim/Frontend"
ref: "heads/main"
[/TOOL]

[TOOL: github.create_ref]
repo: "nordeim/Frontend"
ref: "refs/heads/issue-<ISSUE_NUMBER>-fix-pagination"
sha: "<PASTE_SHA_FROM_PREVIOUS>"
[/TOOL]
```

*Manual fallback (curl)*:

```bash
# get SHA of main
curl -H "Authorization: token $TOKEN" \
  https://api.github.com/repos/nordeim/Frontend/git/ref/heads/main

# create new branch ref (replace <SHA>)
curl -X POST -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/nordeim/Frontend/git/refs \
  -d '{"ref":"refs/heads/issue-123-fix-pagination","sha":"<SHA>"}'
```

---

### 3) Create the PR (via GitHub PR API)

**Once branch exists**, request a PR:

**Claude prompt**:

```
[TOOL: github.create_pull_request]
repo: "nordeim/Frontend"
title: "Fix: pagination shows duplicate items (issue #<ISSUE_NUMBER>)"
body: "This PR fixes the pagination by ... (summary of changes). Related: #<ISSUE_NUMBER>"
head: "issue-<ISSUE_NUMBER>-fix-pagination"
base: "main"
[/TOOL]
```

*What happens:* MCP performs `POST /repos/:owner/:repo/pulls`. On success the PR URL is returned. ([GitHub Docs][7])

*Manual fallback (curl):*

```bash
curl -X POST -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/nordeim/Frontend/pulls \
  -d '{"title":"Fix: pagination shows duplicate items (issue #123)","head":"issue-123-fix-pagination","base":"main","body":"Fix details ..."}'
```

---

# Detailed, step-by-step guide — Model B: Local repo + Git MCP (recommended when you need to run/tests locally)

> Summary: clone repo locally, create branch, change code, run tests, commit & push. Use `git` either directly or via a *Git/Shell MCP* that can run commands in a workspace. Finally use GitHub MCP to create the PR.

### 0) Prepare workspace & MCP

* Create a workspace path (e.g., `/home/user/workspaces/Frontend`).
* Ensure a Git/Shell MCP can read and write to that workspace (or run `git` yourself locally). The `git` MCP should advertise tools like: `git.clone`, `git.checkout`, `git.commit`, `git.push`. (MCP pattern: each tool is one capability.)

---

### 1) Clone repo (local or via MCP)

**Claude prompt** (to Git MCP):

```
[TOOL: git.clone]
repo_url: "https://github.com/nordeim/Frontend.git"
target_dir: "/workspaces/Frontend"
depth: 1
[/TOOL]
```

*Or run locally:*

```bash
git clone https://github.com/nordeim/Frontend.git /workspaces/Frontend
cd /workspaces/Frontend
```

---

### 2) Create branch locally

**Claude prompt** (git MCP):

```
[TOOL: git.checkout_new]
path: "/workspaces/Frontend"
branch: "issue-<ISSUE_NUMBER>-fix-pagination"
base: "main"
[/TOOL]
```

*Or local commands:*

```bash
git fetch origin
git checkout -b issue-123-fix-pagination origin/main
```

---

### 3) Make the fix & run tests

* Use Claude to propose a code change (for small edits Claude can generate a diff). But **human review is mandatory** — never allow automated commits without inspection.
* If you want Claude to produce the patch, instruct it to output a unified diff for the file(s); then apply the patch locally (`git apply`) after review.

**Example Claude prompt (generate patch only):**

```
Use the file at src/components/ProductList.jsx. Based on the repo structure, produce a minimal unified diff that fixes the pagination bug by ensuring offset is computed as (page-1)*pageSize rather than page*pageSize. Do NOT run any commands; output only a patch (diff) that can be applied with git apply.
```

*Human step:* review the patch, run tests:

```bash
# apply patch (after careful review)
git apply fix-pagination.patch

# run tests / build
npm install
npm run test
npm run dev   # or build / lint checks
```

---

### 4) Commit & push

**Claude prompt (git MCP) to commit and push:**

```
[TOOL: git.commit_and_push]
path: "/workspaces/Frontend"
branch: "issue-123-fix-pagination"
commit_message: "fix(pagination): correct offset calculation (fixes #123)"
remote: "origin"
[/TOOL]
```

*Local commands:*

```bash
git add path/to/changed/file
git commit -m "fix(pagination): correct offset calculation (fixes #123)"
git push -u origin issue-123-fix-pagination
```

---

### 5) Create PR (via GitHub MCP)

Use the same `github.create_pull_request` tool as Model A (head is your pushed branch). Claude can write a good PR description from the issue text and the commit summary.

**Claude prompt:**

```
[TOOL: github.create_pull_request]
repo: "nordeim/Frontend"
title: "Fix: pagination duplicate items — fixes #123"
body: "This PR corrects the offset calculation so each page shows the intended slice of items. Tests: ran unit tests and validated locally. See commit: <commit-sha>.\nCloses #123."
head: "issue-123-fix-pagination"
base: "main"
[/TOOL]
```

---

# Example Claude instruction patterns (practical templates)

Use these templates in Claude Desktop as plain text. Replace bracket fields. They assume MCP tools named `github.*`, `git.*`. Adjust to the exact names your MCP advertises.

**Create Issue template**

```
[TOOL: github.create_issue]
repo: "nordeim/Frontend"
title: "<SHORT_TITLE>"
body: "<DETAILED STEPS TO REPRODUCE + EXPECTED BEHAVIOR + ENVIRONMENT>"
labels: ["bug"]
assignees: []
[/TOOL]
```

**Create branch (API) template**

```
[TOOL: github.get_ref]
repo: "nordeim/Frontend"
ref: "heads/main"
[/TOOL]

[TOOL: github.create_ref]
repo: "nordeim/Frontend"
ref: "refs/heads/issue-<ISSUE>-<short>"
sha: "<SHA_FROM_PREV_TOOL>"
[/TOOL]
```

**Local branch (git MCP) template**

```
[TOOL: git.checkout_new]
path: "/workspaces/Frontend"
branch: "issue-<ISSUE>-fix-<short>"
base: "main"
[/TOOL]
```

**Create PR template**

```
[TOOL: github.create_pull_request]
repo: "nordeim/Frontend"
title: "<PR_TITLE>"
body: "<PR_BODY referencing issue #...>"
head: "issue-<ISSUE>-fix-<short>"
base: "main"
[/TOOL]
```

---

# Security, audit & governance notes (must-read)

* **Token safety:** never paste PATs into Claude chat. Give tokens to MCP servers through environment variables, Docker secrets, or a secrets manager. Rotate tokens regularly. See GitHub PAT docs. ([GitHub Docs][4])
* **Human in the loop:** require manual approval for any commit PR generated by an LLM. Use Claude to *draft* patches / PR text, but a human must review and run tests.
* **Least privilege:** PAT scope — grant only what’s necessary (repo contents + pull/issue creation). Use fine-grained tokens where possible. ([GitHub Docs][8])
* **Logging:** keep MCP request/response logs (redacting sensitive tokens) to trace actions.
* **Rate limits:** be aware of API rate limits when automating (GitHub enforces rate limits). See GitHub API docs. ([GitHub Docs][9])

---

# Verification & smoke tests (after each step)

* After issue creation: should receive issue number and URL from the MCP tool. Manually open in browser to confirm.
* After branch creation: verify branch exists on GitHub (web UI or `GET /repos/:owner/:repo/branches`).
* After push: `git ls-remote --heads origin` or check GitHub UI for branch.
* After PR creation: check PR URL, run CI checks (if present). Ensure PR links to issue with `Closes #<n>`.

---

# Troubleshooting (common problems)

* *MCP tool not found in Claude:* confirm Claude Desktop’s MCP server list includes your GitHub/Git MCP endpoint and that capabilities are advertised. Restart the Claude Desktop after MCP changes. ([Anthropic][1])
* *Auth errors (401/403):* check PAT scopes and that the MCP is passing the token correctly. Use a fine-grained token with *Contents* and *Issues* permissions as needed. ([GitHub Docs][4])
* *Branch creation 422 / bad request:* ensure the `sha` is valid and branch name doesn’t already exist. If using the Git API, you must POST a `ref` in the form `refs/heads/branch-name`. ([GitHub Docs][6])

---

# Quick manual / CLI reference (useful fallback)

* Create issue (curl): `POST /repos/:owner/:repo/issues`. ([GitHub Docs][3])
* Create branch (refs): `POST /repos/:owner/:repo/git/refs` with `{"ref":"refs/heads/BRANCH","sha":"<SHA>"}`. ([GitHub Docs][6])
* Create PR: `POST /repos/:owner/:repo/pulls`. ([GitHub Docs][7])

---

# Example end-to-end minimal script (manual, for reference)

*(only a reference; don’t run without your PAT in `$TOKEN` and after careful review)*

```bash
# create issue
curl -s -X POST -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/nordeim/Frontend/issues \
  -d '{"title":"Bug: pagination shows same products on all pages","body":"Steps..."}'

# get main sha
MAIN_SHA=$(curl -s -H "Authorization: token $TOKEN" \
  https://api.github.com/repos/nordeim/Frontend/git/ref/heads/main | jq -r .object.sha)

# create branch
curl -s -X POST -H "Authorization: token $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/nordeim/Frontend/git/refs \
  -d "{\"ref\":\"refs/heads/issue-123-fix-pagination\",\"sha\":\"$MAIN_SHA\"}"

# create PR
curl -s -X POST -H "Authorization: token $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/nordeim/Frontend/pulls \
  -d '{"title":"Fix pagination bug","head":"issue-123-fix-pagination","base":"main","body":"Fixes #123"}'
```

(These curl snippets map directly to the MCP GitHub tool behavior — MCP just wraps these REST calls safely and exposes them as tools to Claude Desktop.) ([GitHub Docs][9])

---

# Final checklist you can run now (copy & tick)

1. [ ] Create PAT with appropriate scopes and store in secrets manager. ([GitHub Docs][4])
2. [ ] Start GitHub MCP server and bind to localhost; configure it to read PAT from secret. ([Docker Hub][5])
3. [ ] Start Git/Shell MCP (if using local repo flow) and mount workspace.
4. [ ] Register MCP servers in Claude Desktop (Settings → MCP servers). Restart Claude. ([Anthropic][1])
5. [ ] Use the *Create Issue* Claude prompt to open the issue (capture returned issue number).
6. [ ] Use *Create Branch* (API or git) flow with the issue number in the branch name.
7. [ ] Edit, test, commit, and push the branch (local or via MCP).
8. [ ] Use *Create PR* Claude prompt to open a PR referencing the issue.
9. [ ] Verify PR in GitHub UI and run CI/tests.
10. [ ] Close or link PR to issue (`Closes #N`) in PR body.

---

If you want I can now:

* *A)* Produce an exact `docker-compose.yml` snippet and environment file example showing how to run a **GitHub MCP** and **Git/Shell MCP** locally (with safe token injection via Docker secrets), or
* *B)* Produce a set of **ready-to-paste Claude Desktop prompts** (one per step) tuned for the exact MCP tool names — I’ll assume common names (`github.create_issue`, `github.create_ref`, `github.create_pull_request`, `git.clone`, `git.commit_and_push`), or
* *C)* Produce a **CI/GitHub Actions template** you can use to automatically run tests on PRs created by MCP workflows.

Pick A, B, or C and I’ll generate it next (I’ll include any necessary file content and instructions).

[1]: https://docs.anthropic.com/en/docs/mcp?utm_source=chatgpt.com "Model Context Protocol (MCP)"
[2]: https://github.com/nordeim/Frontend.git "GitHub - nordeim/Frontend"
[3]: https://docs.github.com/rest/reference/issues?utm_source=chatgpt.com "REST API endpoints for issues"
[4]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens?utm_source=chatgpt.com "Managing your personal access tokens"
[5]: https://hub.docker.com/u/mcp?utm_source=chatgpt.com "mcp"
[6]: https://docs.github.com/en/rest/git/refs?utm_source=chatgpt.com "REST API endpoints for Git references"
[7]: https://docs.github.com/en/rest/pulls?utm_source=chatgpt.com "REST API endpoints for pull requests"
[8]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens?utm_source=chatgpt.com "Permissions required for fine-grained personal access tokens"
[9]: https://docs.github.com/rest/guides/getting-started-with-the-rest-api?utm_source=chatgpt.com "Getting started with the REST API"

https://chatgpt.com/share/68c81f1b-44a0-8000-8f76-a1786a5d13ef
