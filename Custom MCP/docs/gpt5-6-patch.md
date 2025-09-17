## **Plan**

### **Goals**
1. **Centralize timestamp creation** in `MCPBaseTool` so all tools — including `NmapTool` — use the same UTC, timezone‑aware format.
2. **Avoid breaking changes** for any subclasses or call sites.
3. **Make it easy to override** if a specific tool needs a different timestamp format in the future.

---

### **Steps**

1. **Add a `_get_timestamp()` helper** to `MCPBaseTool` that returns `datetime.now(timezone.utc)`.
2. **Replace all direct `datetime.now()` calls** in `MCPBaseTool` with `self._get_timestamp()`.
3. **Ensure subclasses** (like `NmapTool`) can override `_get_timestamp()` if needed — but by default, they’ll inherit the UTC behavior.
4. **Remove redundant `_get_timestamp()`** from subclasses if it’s identical to the base implementation (optional cleanup).
5. **Test**:
   - Run a tool that triggers error handling and confirm timestamps are UTC and timezone‑aware.
   - Ensure JSON serialization still works (timezone‑aware datetimes are serializable if handled correctly).

---

## **Patch for `mcp_server/base_tool.py`**

```python
# At the top of the file
from datetime import datetime, timezone

class MCPBaseTool(ABC):
    """
    Enhanced base class for MCP tools with circuit breaker, metrics, and advanced features.
    """

    # ... existing class variables ...

    def __init__(self):
        self.tool_name = self.__class__.__name__
        self._initialize_metrics()
        self._initialize_circuit_breaker()

    # --- NEW METHOD ---
    def _get_timestamp(self):
        """
        Get the current UTC timestamp (timezone-aware).
        Subclasses may override if they require a different format.
        """
        return datetime.now(timezone.utc)

    # --- UPDATE EXISTING CODE ---
    async def run(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        start_time = time.time()
        correlation_id = inp.correlation_id or str(int(start_time * 1000))

        try:
            if self._circuit_breaker and self._circuit_breaker.state == CircuitBreakerState.OPEN:
                error_context = ErrorContext(
                    error_type=ToolErrorType.CIRCUIT_BREAKER_OPEN,
                    message=f"Circuit breaker is open for {self.tool_name}",
                    recovery_suggestion="Wait for recovery timeout or check service health",
                    timestamp=self._get_timestamp(),  # CHANGED
                    tool_name=self.tool_name,
                    target=inp.target,
                    metadata={"state": str(self._circuit_breaker.state)}
                )
                return self._create_error_output(error_context, correlation_id)

            async with self._ensure_semaphore():
                if self._circuit_breaker:
                    try:
                        result = await self._circuit_breaker.call(
                            self._execute_tool,
                            inp,
                            timeout_sec
                        )
                    except Exception as circuit_error:
                        error_context = ErrorContext(
                            error_type=ToolErrorType.CIRCUIT_BREAKER_OPEN,
                            message=f"Circuit breaker error: {str(circuit_error)}",
                            recovery_suggestion="Wait for recovery timeout or check service health",
                            timestamp=self._get_timestamp(),  # CHANGED
                            tool_name=self.tool_name,
                            target=inp.target,
                            metadata={"circuit_error": str(circuit_error)}
                        )
                        return self._create_error_output(error_context, correlation_id)
                else:
                    result = await self._execute_tool(inp, timeout_sec)

                if self.metrics:
                    execution_time = max(0.001, time.time() - start_time)
                    try:
                        self.metrics.record_execution(
                            success=result.returncode == 0,
                            execution_time=execution_time,
                            timed_out=result.timed_out
                        )
                    except Exception as e:
                        log.warning("metrics.recording_failed tool=%s error=%s", self.tool_name, str(e))

                result.correlation_id = correlation_id
                result.execution_time = max(0.001, time.time() - start_time)

                return result

        except Exception as e:
            execution_time = max(0.001, time.time() - start_time)
            error_context = ErrorContext(
                error_type=ToolErrorType.EXECUTION_ERROR,
                message=f"Tool execution failed: {str(e)}",
                recovery_suggestion="Check tool logs and system resources",
                timestamp=self._get_timestamp(),  # CHANGED
                tool_name=self.tool_name,
                target=inp.target,
                metadata={"exception": str(e), "execution_time": execution_time}
            )

            if self.metrics:
                try:
                    self.metrics.record_execution(
                        success=False,
                        execution_time=execution_time,
                        error_type=ToolErrorType.EXECUTION_ERROR.value
                    )
                except Exception as metrics_error:
                    log.warning("metrics.failure_recording_failed tool=%s error=%s",
                                self.tool_name, str(metrics_error))

            return self._create_error_output(error_context, correlation_id)

    async def _execute_tool(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        resolved_cmd = self._resolve_command()
        if not resolved_cmd:
            error_context = ErrorContext(
                error_type=ToolErrorType.NOT_FOUND,
                message=f"Command not found: {self.command_name}",
                recovery_suggestion="Install the required tool or check PATH",
                timestamp=self._get_timestamp(),  # CHANGED
                tool_name=self.tool_name,
                target=inp.target,
                metadata={"command": self.command_name}
            )
            return self._create_error_output(error_context, inp.correlation_id)

        # ... rest of method unchanged ...
```

---

### **Why this is future‑safe**
- All timestamp creation is now centralized in `_get_timestamp()`.
- Any subclass can override `_get_timestamp()` if it needs a different format (e.g., local time for a specific compliance requirement).
- No need to touch each tool — they’ll automatically inherit UTC timestamps.
- This change is backward‑compatible for all existing tools.

---

### Plan

- Remove the redundant NmapTool._get_timestamp method so it inherits MCPBaseTool._get_timestamp (UTC, timezone-aware).
- Remove unused imports introduced solely for the deleted method (datetime, timezone).
- Keep all calls to self._get_timestamp as-is; they will now resolve to the base-class implementation.
- Ensure no other references require datetime/timezone in this file.

---

### Patch: remove redundant _get_timestamp and unused imports

```diff
*** Begin Patch
*** Update File: mcp_server/tools/nmap_tool.py
@@
 """
 Enhanced Nmap tool with circuit breaker, metrics, and advanced features.
 """
 import logging
 import shlex
 import ipaddress
-from datetime import datetime, timezone
 from typing import Sequence, Optional
 
 from mcp_server.base_tool import MCPBaseTool, ToolInput, ToolOutput, ToolErrorType, ErrorContext
 from mcp_server.config import get_config
 
@@
 
         return " ".join(optimized)
-    
-    def _get_timestamp(self):
-        """Get current timestamp (UTC, timezone-aware)."""
-        return datetime.now(timezone.utc)
 
     def get_tool_info(self) -> dict:
         """Get enhanced tool information."""
         base_info = {
             "name": self.tool_name,
*** End Patch
```

https://copilot.microsoft.com/shares/me8zcukjZeLzY4cvHnRvh
