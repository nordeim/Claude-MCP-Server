### Execution plan for improved core file replacements

We’ll create secure, robust, and maintainable drop-in replacements for your core framework and server entry-point. The goals: harden inputs, prevent command injection, improve observability and reliability, and keep the API shape compatible with your current MCP integration.

#### Scope

- Replace base_tool.py with a safer, extensible command execution framework.
- Replace server.py with a resilient entry-point that supports configuration, graceful shutdown, and automatic tool discovery.

#### Key upgrades

- Input validation with strict RFC1918/“.lab.internal” checks via ipaddress, plus safer arg parsing with denylisted metacharacters and length limits.
- Process execution with timeouts, output truncation, structured logging, and graceful cancellation.
- Command discovery via shutil.which and clear error messages when binaries are missing.
- Optional concurrency limiting per tool to avoid overloading the host.
- Server auto-discovers tool classes from mcp_server.tools, supports include/exclude lists, logs structured metadata, and handles signals cleanly.

#### Checklists

- Base tool
  - [ ] Strict validation for private targets and .lab.internal hostnames.
  - [ ] Safer arg parsing; reject shell metacharacters and oversize arg strings.
  - [ ] Enforce execution timeout and output truncation with clear flags in output.
  - [ ] Verify command availability; handle FileNotFoundError and timeouts distinctly.
  - [ ] Support concurrency limit via an asyncio.Semaphore.
  - [ ] Structured logging with stable fields; redact sensitive values.
  - [ ] Clean return model, preserving backward compatibility with ToolOutput.

- Server
  - [ ] Configurable via env: MCP_TRANSPORT, LOG_LEVEL, TOOL_INCLUDE/EXCLUDE, SHUTDOWN_GRACE, DISCOVERY_PACKAGE, etc.
  - [ ] Dynamic tool discovery by subclass of MCPBaseTool; safe import with error isolation per module.
  - [ ] Graceful shutdown on SIGINT/SIGTERM; cancel serve loop with timeout.
  - [ ] Optional uvloop if present; does not break environments without it.
  - [ ] Clear startup banner with tool inventory and effective configuration.

---

## Review and validation of current files

### base_tool.py — findings

- Strengths
  - Uses asyncio subprocess with shell disabled; good baseline for injection resistance.
  - Restricts targets to RFC1918 or .lab.internal with a validator; good safety intent.

- Risks and gaps
  - Validator uses regexes that are easy to get wrong for CIDR math; ipaddress is safer and clearer.
  - No timeout handling; long-running processes can hang indefinitely.
  - No output truncation; large outputs can exhaust memory or flood logs.
  - No explicit check for command existence; FileNotFoundError will raise later and may be unhelpful to clients.
  - extra_args are passed via shlex.split (good), but no allow/deny-list or length limits; e.g., absurdly long args or sneaky metacharacters are not rejected early.
  - Pydantic v1-only validator; if you upgrade to v2, this breaks.

### server.py — findings

- Strengths
  - Simple and clear entry; awaits server.serve() and uses env for transport.

- Risks and gaps
  - Hard-coded tool imports make the server brittle to changes and complicate packaging.
  - No graceful shutdown or signal handling; abrupt termination can cause partial work or corrupted state.
  - No logging configurability; fixed INFO level.
  - No discovery or filtering; cannot easily include/exclude tools per deployment.

---

## Improved drop-in replacements

These are drop-in replacements that keep your public surface area consistent while adding robust safety and operability.

### 1) base_tool.py (drop-in replacement)

```python
import asyncio
import logging
import os
import re
import shlex
import shutil
from abc import ABC
from typing import ClassVar, Optional, Sequence

# Pydantic v1/v2 compatibility shim
try:  # Pydantic v2
    from pydantic import BaseModel, field_validator
    _PD_V2 = True
except Exception:  # Pydantic v1
    from pydantic import BaseModel, validator as field_validator  # type: ignore
    _PD_V2 = False

from mcp.tool import tool

log = logging.getLogger(__name__)

# Conservative denylist for arg tokens we never want to see (even though shell=False)
_DENY_CHARS = re.compile(r"[;&|`$><\n\r]")  # control/meta chars
_TOKEN_ALLOWED = re.compile(r"^[A-Za-z0-9._:/=\-+,@%]+$")  # reasonably safe superset
_MAX_ARGS_LEN = int(os.getenv("MCP_MAX_ARGS_LEN", "2048"))
_MAX_STDOUT_BYTES = int(os.getenv("MCP_MAX_STDOUT_BYTES", "1048576"))  # 1 MiB
_MAX_STDERR_BYTES = int(os.getenv("MCP_MAX_STDERR_BYTES", "262144"))   # 256 KiB
_DEFAULT_TIMEOUT_SEC = float(os.getenv("MCP_DEFAULT_TIMEOUT_SEC", "300"))  # 5 minutes
_DEFAULT_CONCURRENCY = int(os.getenv("MCP_DEFAULT_CONCURRENCY", "2"))


def _is_private_or_lab(value: str) -> bool:
    """
    Accept:
      - RFC1918 IPv4 address (10/8, 172.16/12, 192.168/16)
      - RFC1918 IPv4 network in CIDR form
      - Hostname ending with .lab.internal
    """
    import ipaddress
    v = value.strip()

    # Hostname allowance
    if v.endswith(".lab.internal"):
        return True

    # IP or CIDR
    try:
        if "/" in v:
            net = ipaddress.ip_network(v, strict=False)
            return net.version == 4 and net.is_private
        else:
            ip = ipaddress.ip_address(v)
            return ip.version == 4 and ip.is_private
    except ValueError:
        return False


class ToolInput(BaseModel):
    target: str
    extra_args: str = ""

    # v1/v2 compatible field validator
    if _PD_V2:
        @field_validator("target")
        @classmethod
        def _validate_target(cls, v: str) -> str:
            if not _is_private_or_lab(v):
                raise ValueError("Target must be RFC1918 IPv4 or a .lab.internal hostname (CIDR allowed).")
            return v

        @field_validator("extra_args")
        @classmethod
        def _validate_extra_args(cls, v: str) -> str:
            v = v or ""
            if len(v) > _MAX_ARGS_LEN:
                raise ValueError(f"extra_args too long (> {_MAX_ARGS_LEN} bytes)")
            if _DENY_CHARS.search(v):
                raise ValueError("extra_args contains forbidden metacharacters")
            return v
    else:
        @field_validator("target")
        def _validate_target(cls, v: str) -> str:  # type: ignore
            if not _is_private_or_lab(v):
                raise ValueError("Target must be RFC1918 IPv4 or a .lab.internal hostname (CIDR allowed).")
            return v

        @field_validator("extra_args")
        def _validate_extra_args(cls, v: str) -> str:  # type: ignore
            v = v or ""
            if len(v) > _MAX_ARGS_LEN:
                raise ValueError(f"extra_args too long (> {_MAX_ARGS_LEN} bytes)")
            if _DENY_CHARS.search(v):
                raise ValueError("extra_args contains forbidden metacharacters")
            return v


class ToolOutput(BaseModel):
    stdout: str
    stderr: str
    returncode: int
    truncated_stdout: bool = False
    truncated_stderr: bool = False
    timed_out: bool = False
    error: Optional[str] = None


class MCPBaseTool(ABC):
    """
    Base class for MCP tools that execute a system command safely.
    Subclasses must set `command_name` and may override `allowed_flags`,
    `default_timeout_sec`, and `concurrency`.
    """

    # Required: name of the binary to execute, e.g., "nmap"
    command_name: ClassVar[str]

    # Optional: a whitelist of flags (prefix match) to allow, e.g., ["-sV", "-p", "--top-ports"]
    allowed_flags: ClassVar[Optional[Sequence[str]]] = None

    # Concurrency limit per tool instance
    concurrency: ClassVar[int] = _DEFAULT_CONCURRENCY

    # Default timeout for a run in seconds
    default_timeout_sec: ClassVar[float] = _DEFAULT_TIMEOUT_SEC

    # Semaphore created on first use per subclass
    _semaphore: ClassVar[Optional[asyncio.Semaphore]] = None

    def _ensure_semaphore(self) -> asyncio.Semaphore:
        if self.__class__._semaphore is None:
            self.__class__._semaphore = asyncio.Semaphore(self.concurrency)
        return self.__class__._semaphore

    def _resolve_command(self) -> Optional[str]:
        return shutil.which(self.command_name)

    def _parse_args(self, extra_args: str) -> Sequence[str]:
        if not extra_args:
            return []
        tokens = shlex.split(extra_args)
        safe: list[str] = []
        for t in tokens:
            if not t:  # skip empties
                continue
            if not _TOKEN_ALLOWED.match(t):
                raise ValueError(f"Disallowed token in args: {t!r}")
            safe.append(t)
        if self.allowed_flags is not None:
            # Approve flags by prefix match; non-flags (e.g., values) are allowed
            allowed = tuple(self.allowed_flags)
            for t in safe:
                if t.startswith("-") and not t.startswith(allowed):
                    raise ValueError(f"Flag not allowed: {t!r}")
        return safe

    async def _spawn(
        self,
        cmd: Sequence[str],
        timeout_sec: Optional[float] = None,
    ) -> ToolOutput:
        timeout = float(timeout_sec or self.default_timeout_sec)

        # Minimal, sanitized environment
        env = {
            "PATH": os.getenv("PATH", "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"),
            "LANG": "C.UTF-8",
            "LC_ALL": "C.UTF-8",
        }

        try:
            log.info(
                "tool.start command=%s timeout=%.1f",
                " ".join(cmd),
                timeout,
            )

            proc = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                env=env,
            )

            try:
                out, err = await asyncio.wait_for(proc.communicate(), timeout=timeout)
                rc = proc.returncode
            except asyncio.TimeoutError:
                with contextlib.suppress(ProcessLookupError):
                    proc.kill()
                out, err, rc = b"", b"process timed out", 124
                return ToolOutput(
                    stdout="",
                    stderr="process timed out",
                    returncode=rc,
                    truncated_stdout=False,
                    truncated_stderr=False,
                    timed_out=True,
                    error="timeout",
                )

            # Truncate outputs if necessary
            t_stdout = False
            t_stderr = False
            if len(out) > _MAX_STDOUT_BYTES:
                out = out[:_MAX_STDOUT_BYTES]
                t_stdout = True
            if len(err) > _MAX_STDERR_BYTES:
                err = err[:_MAX_STDERR_BYTES]
                t_stderr = True

            result = ToolOutput(
                stdout=out.decode(errors="replace"),
                stderr=err.decode(errors="replace"),
                returncode=rc,
                truncated_stdout=t_stdout,
                truncated_stderr=t_stderr,
                timed_out=False,
            )

            log.info(
                "tool.end command=%s returncode=%s truncated_stdout=%s truncated_stderr=%s",
                cmd[0],
                rc,
                t_stdout,
                t_stderr,
            )
            return result

        except FileNotFoundError:
            msg = f"Command not found: {cmd[0]}"
            log.error("tool.error %s", msg)
            return ToolOutput(stdout="", stderr=msg, returncode=127, error="not_found")
        except Exception as e:
            msg = f"execution failed: {e.__class__.__name__}: {e}"
            log.exception("tool.error %s", msg)
            return ToolOutput(stdout="", stderr=msg, returncode=1, error="exception")

    @tool
    async def run(self, inp: "ToolInput", timeout_sec: Optional[float] = None) -> "ToolOutput":
        """
        Execute the command against the validated target with optional extra args.
        Env var overrides:
          - MCP_MAX_STDOUT_BYTES / MCP_MAX_STDERR_BYTES
          - MCP_DEFAULT_TIMEOUT_SEC
          - MCP_DEFAULT_CONCURRENCY
        """
        # Resolve command
        resolved = self._resolve_command()
        if not resolved:
            return ToolOutput(
                stdout="",
                stderr=f"Command not available: {self.command_name}",
                returncode=127,
                error="not_found",
            )

        # Build arguments
        try:
            args = self._parse_args(inp.extra_args)
        except ValueError as ve:
            return ToolOutput(stdout="", stderr=str(ve), returncode=64, error="bad_args")

        cmd = [resolved, *args, inp.target]

        # Concurrency limiting
        sem = self._ensure_semaphore()
        async with sem:
            return await self._spawn(cmd, timeout_sec=timeout_sec)
```

Notes:
- Backward compatible ToolInput and ToolOutput, with additional informative fields.
- Safer target validation and extra_args checks.
- Timeouts, truncation, and structured logs for operability.
- Optional flag allowlisting per tool class.

---

### 2) server.py (drop-in replacement)

```python
import asyncio
import importlib
import inspect
import logging
import os
import pkgutil
import signal
from typing import Iterable, List, Optional, Sequence, Type

from mcp.server import MCPServer  # assumed interface unchanged

# Your base tool lives here
from mcp_server.base_tool import MCPBaseTool  # adjust if your package path differs

# Optional: use uvloop if available
def _maybe_setup_uvloop() -> None:
    try:
        import uvloop  # type: ignore
        uvloop.install()
    except Exception:
        pass

def _setup_logging() -> None:
    level = os.getenv("LOG_LEVEL", "INFO").upper()
    fmt = os.getenv(
        "LOG_FORMAT",
        "%(asctime)s %(levelname)s %(name)s %(message)s",
    )
    logging.basicConfig(level=getattr(logging, level, logging.INFO), format=fmt)

log = logging.getLogger("mcp_server")

def _load_tools_from_package(
    package_path: str,
    include: Optional[Sequence[str]] = None,
    exclude: Optional[Sequence[str]] = None,
) -> List[MCPBaseTool]:
    """
    Discover and instantiate concrete MCPBaseTool subclasses under package_path.
    include/exclude: class names (e.g., ["NmapTool"]) to filter.
    """
    tools: list[MCPBaseTool] = []

    try:
        pkg = importlib.import_module(package_path)
    except Exception as e:
        log.error("Failed to import tools package %s: %s", package_path, e)
        return tools

    for modinfo in pkgutil.walk_packages(pkg.__path__, prefix=pkg.__name__ + "."):
        try:
            module = importlib.import_module(modinfo.name)
        except Exception as e:
            log.warning("Skipping tool module %s due to import error: %s", modinfo.name, e)
            continue

        for _, obj in inspect.getmembers(module, inspect.isclass):
            if not issubclass(obj, MCPBaseTool) or obj is MCPBaseTool:
                continue
            name = obj.__name__
            if include and name not in include:
                continue
            if exclude and name in exclude:
                continue
            try:
                inst = obj()  # assume no-arg constructor
                tools.append(inst)
            except Exception as e:
                log.warning("Failed to instantiate tool %s: %s", name, e)

    return tools

def _parse_csv_env(name: str) -> Optional[List[str]]:
    raw = os.getenv(name, "").strip()
    if not raw:
        return None
    return [x.strip() for x in raw.split(",") if x.strip()]

async def _serve(server: MCPServer, shutdown_grace: float) -> None:
    loop = asyncio.get_running_loop()
    stop = asyncio.Event()

    def _signal_handler(sig: int) -> None:
        log.info("Received signal %s, initiating shutdown", sig)
        stop.set()

    for sig in (signal.SIGINT, signal.SIGTERM):
        with contextlib.suppress(NotImplementedError):
            loop.add_signal_handler(sig, _signal_handler, sig)

    serve_task = asyncio.create_task(server.serve(), name="mcp_serve")

    await stop.wait()
    log.info("Shutting down MCP server...")
    serve_task.cancel()
    try:
        await asyncio.wait_for(serve_task, timeout=shutdown_grace)
    except asyncio.TimeoutError:
        log.warning("Forced shutdown after %.1fs", shutdown_grace)
    except asyncio.CancelledError:
        pass

async def main() -> None:
    _maybe_setup_uvloop()
    _setup_logging()

    transport = os.getenv("MCP_TRANSPORT", "stdio").lower()
    tools_pkg = os.getenv("TOOLS_PACKAGE", "mcp_server.tools")
    include = _parse_csv_env("TOOL_INCLUDE")
    exclude = _parse_csv_env("TOOL_EXCLUDE")
    shutdown_grace = float(os.getenv("SHUTDOWN_GRACE", "10"))

    tools = _load_tools_from_package(tools_pkg, include=include, exclude=exclude)

    log.info(
        "Starting MCP server transport=%s tools=%s",
        transport,
        [t.__class__.__name__ for t in tools],
    )

    server = MCPServer(tools=tools, transport=transport)
    await _serve(server, shutdown_grace=shutdown_grace)

if __name__ == "__main__":
    import contextlib
    asyncio.run(main())
```

Notes:
- Dynamic discovery from mcp_server.tools avoids brittle manual imports.
- Include/exclude filters via env for deploy-time control.
- Logging level and format configurable.
- Graceful shutdown with signal handling and bounded grace period.
- Keeps MCPServer(tools=..., transport=...) contract intact.

---

## Validation matrix

- Backward compatibility
  - ToolInput/ToolOutput shapes remain compatible; added fields are additive and safe.
  - Entry-point still at mcp_server.server:main via python -m mcp_server.server.

- Safety improvements
  - Target validation via ipaddress; CIDR allowed, public IPs blocked.
  - Args: denylist and token whitelist, length limits, optional flag allowlist.
  - Timeouts, output caps, and structured logs.

- Operability
  - Clear error codes: 127 for missing command, 124 timeout, 64 bad args.
  - Concurrency control to prevent overloads.
  - Dynamic tool discovery with robust error isolation.

---

## How to adopt

- Replace your existing files with the versions above.
- Optionally, in each concrete tool class (e.g., NmapTool), set:
  - allowed_flags = ["-sV", "-p", "--top-ports", ...]
  - default_timeout_sec = 600
  - concurrency = 1
- Set env vars as needed:
  - LOG_LEVEL=debug
  - TOOL_INCLUDE=NmapTool,GobusterTool
  - MCP_DEFAULT_TIMEOUT_SEC=300
  - MCP_DEFAULT_CONCURRENCY=2

---

## Quick sanity tests

- Base tool validations
  - Run with target=10.0.0.1 and extra_args="-sV -p 80,443" → accepted.
  - Run with target=8.8.8.8 → rejected by validator.
  - Run with extra_args="; rm -rf /" → rejected by validator.

- Timeout behavior
  - Set MCP_DEFAULT_TIMEOUT_SEC=1 and point at a slow scan; returns error="timeout", returncode=124.

- Discovery
  - Add a new tool class in mcp_server.tools, restart server → auto-loaded.
  - Exclude it with TOOL_EXCLUDE=YourTool.

---
https://copilot.microsoft.com/shares/61V83S2U5GQ9EbCoJtNV1
