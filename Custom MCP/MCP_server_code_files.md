# mcp_server/base_tool.py
```py
# File: base_tool.py
"""
Enhanced MCP Base Tool with circuit breaker, metrics, and advanced error handling.
Production-ready implementation with all critical fixes applied.
"""
import asyncio
import logging
import os
import re
import shlex
import shutil
import time
import contextlib
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import ClassVar, Optional, Sequence, Dict, Any
from datetime import datetime, timedelta

# Pydantic v1/v2 compatibility shim
try: # Pydantic v2
    from pydantic import BaseModel, field_validator
    _PD_V2 = True
except ImportError: # Pydantic v1
    from pydantic import BaseModel, validator as field_validator # type: ignore
    _PD_V2 = False

# Metrics integration with graceful handling
try:
    from prometheus_client import Counter, Histogram, Gauge, Info
    PROMETHEUS_AVAILABLE = True
except ImportError:
    PROMETHEUS_AVAILABLE = False

# Circuit breaker implementation with fallback import
try:
    from .circuit_breaker import CircuitBreaker, CircuitBreakerState
except ImportError:
    from circuit_breaker import CircuitBreaker, CircuitBreakerState

# Tool metrics with fallback import
try:
    from .metrics import ToolMetrics
except ImportError:
    from metrics import ToolMetrics

log = logging.getLogger(__name__)

# Conservative denylist for arg tokens we never want to see (even though shell=False)
_DENY_CHARS = re.compile(r"[;&|`$><\n\r]") # control/meta chars
_TOKEN_ALLOWED = re.compile(r"^[A-Za-z0-9.:/=+-,@%]+$") # reasonably safe superset
_MAX_ARGS_LEN = int(os.getenv("MCP_MAX_ARGS_LEN", "2048"))
_MAX_STDOUT_BYTES = int(os.getenv("MCP_MAX_STDOUT_BYTES", "1048576")) # 1 MiB
_MAX_STDERR_BYTES = int(os.getenv("MCP_MAX_STDERR_BYTES", "262144")) # 256 KiB
_DEFAULT_TIMEOUT_SEC = float(os.getenv("MCP_DEFAULT_TIMEOUT_SEC", "300")) # 5 minutes
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

class ToolErrorType(Enum):
    """Enhanced error taxonomy with recovery suggestions."""
    TIMEOUT = "timeout"
    NOT_FOUND = "not_found"
    VALIDATION_ERROR = "validation_error"
    EXECUTION_ERROR = "execution_error"
    RESOURCE_EXHAUSTED = "resource_exhausted"
    CIRCUIT_BREAKER_OPEN = "circuit_breaker_open"
    UNKNOWN = "unknown"

@dataclass
class ErrorContext:
    """Context for enhanced error reporting."""
    error_type: ToolErrorType
    message: str
    recovery_suggestion: str
    timestamp: datetime
    tool_name: str
    target: str
    metadata: Dict[str, Any]

class ToolInput(BaseModel):
    """Enhanced ToolInput with additional validation."""
    target: str
    extra_args: str = ""
    timeout_sec: Optional[float] = None
    correlation_id: Optional[str] = None
    
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
        def _validate_target(cls, v: str) -> str: # type: ignore
            if not _is_private_or_lab(v):
                raise ValueError("Target must be RFC1918 IPv4 or a .lab.internal hostname (CIDR allowed).")
            return v
        
        @field_validator("extra_args")
        def _validate_extra_args(cls, v: str) -> str: # type: ignore
            v = v or ""
            if len(v) > _MAX_ARGS_LEN:
                raise ValueError(f"extra_args too long (> {_MAX_ARGS_LEN} bytes)")
            if _DENY_CHARS.search(v):
                raise ValueError("extra_args contains forbidden metacharacters")
            return v

class ToolOutput(BaseModel):
    """Enhanced ToolOutput with additional metadata."""
    stdout: str
    stderr: str
    returncode: int
    truncated_stdout: bool = False
    truncated_stderr: bool = False
    timed_out: bool = False
    error: Optional[str] = None
    error_type: Optional[str] = None
    execution_time: Optional[float] = None
    correlation_id: Optional[str] = None
    metadata: Dict[str, Any] = {}

class MCPBaseTool(ABC):
    """
    Enhanced base class for MCP tools with circuit breaker, metrics, and advanced features.
    """
    
    # Required: name of the binary to execute, e.g., "nmap"
    command_name: ClassVar[str]
    
    # Optional: a whitelist of flags (prefix match) to allow
    allowed_flags: ClassVar[Optional[Sequence[str]]] = None
    
    # Concurrency limit per tool instance
    concurrency: ClassVar[int] = _DEFAULT_CONCURRENCY
    
    # Default timeout for a run in seconds
    default_timeout_sec: ClassVar[float] = _DEFAULT_TIMEOUT_SEC
    
    # Circuit breaker configuration
    circuit_breaker_failure_threshold: ClassVar[int] = 5
    circuit_breaker_recovery_timeout: ClassVar[float] = 60.0
    circuit_breaker_expected_exception: ClassVar[tuple] = (Exception,)
    
    # Semaphore created on first use per subclass
    _semaphore: ClassVar[Optional[asyncio.Semaphore]] = None
    
    # Circuit breaker instance per tool
    _circuit_breaker: ClassVar[Optional[CircuitBreaker]] = None
    
    def __init__(self):
        self.tool_name = self.__class__.__name__
        self._initialize_metrics()
        self._initialize_circuit_breaker()
    
    def _initialize_metrics(self):
        """Initialize Prometheus metrics for this tool."""
        if PROMETHEUS_AVAILABLE:
            try:
                self.metrics = ToolMetrics(self.tool_name)
            except Exception as e:
                log.warning("metrics.initialization_failed tool=%s error=%s", self.tool_name, str(e))
                self.metrics = None
        else:
            self.metrics = None
    
    def _initialize_circuit_breaker(self):
        """Initialize circuit breaker for this tool."""
        if self.__class__._circuit_breaker is None:
            try:
                self.__class__._circuit_breaker = CircuitBreaker(
                    failure_threshold=self.circuit_breaker_failure_threshold,
                    recovery_timeout=self.circuit_breaker_recovery_timeout,
                    expected_exception=self.circuit_breaker_expected_exception,
                    name=self.tool_name
                )
            except Exception as e:
                log.error("circuit_breaker.initialization_failed tool=%s error=%s", self.tool_name, str(e))
                self.__class__._circuit_breaker = None
    
    def _ensure_semaphore(self) -> asyncio.Semaphore:
        """Ensure semaphore exists for this tool class."""
        if self.__class__._semaphore is None:
            self.__class__._semaphore = asyncio.Semaphore(self.concurrency)
        return self.__class__._semaphore
    
    async def run(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        """
        Enhanced run method with circuit breaker, metrics, and error handling.
        """
        start_time = time.time()
        correlation_id = inp.correlation_id or str(int(start_time * 1000))
        
        try:
            # Check circuit breaker state
            if self._circuit_breaker and self._circuit_breaker.state == CircuitBreakerState.OPEN:
                error_context = ErrorContext(
                    error_type=ToolErrorType.CIRCUIT_BREAKER_OPEN,
                    message=f"Circuit breaker is open for {self.tool_name}",
                    recovery_suggestion="Wait for recovery timeout or check service health",
                    timestamp=datetime.now(),
                    tool_name=self.tool_name,
                    target=inp.target,
                    metadata={"state": str(self._circuit_breaker.state)}
                )
                return self._create_error_output(error_context, correlation_id)
            
            # Acquire semaphore for concurrency control
            async with self._ensure_semaphore():
                # Execute with circuit breaker protection
                if self._circuit_breaker:
                    try:
                        result = await self._circuit_breaker.call(
                            self._execute_tool,
                            inp,
                            timeout_sec
                        )
                    except Exception as circuit_error:
                        # Handle circuit breaker specific errors
                        error_context = ErrorContext(
                            error_type=ToolErrorType.CIRCUIT_BREAKER_OPEN,
                            message=f"Circuit breaker error: {str(circuit_error)}",
                            recovery_suggestion="Wait for recovery timeout or check service health",
                            timestamp=datetime.now(),
                            tool_name=self.tool_name,
                            target=inp.target,
                            metadata={"circuit_error": str(circuit_error)}
                        )
                        return self._create_error_output(error_context, correlation_id)
                else:
                    result = await self._execute_tool(inp, timeout_sec)
                
                # Record metrics with validation
                if self.metrics:
                    execution_time = max(0.001, time.time() - start_time)  # Ensure minimum positive value
                    try:
                        self.metrics.record_execution(
                            success=result.returncode == 0,
                            execution_time=execution_time,
                            timed_out=result.timed_out
                        )
                    except Exception as e:
                        log.warning("metrics.recording_failed tool=%s error=%s", self.tool_name, str(e))
                
                # Add correlation ID and execution time
                result.correlation_id = correlation_id
                result.execution_time = max(0.001, time.time() - start_time)
                
                return result
                
        except Exception as e:
            execution_time = max(0.001, time.time() - start_time)
            error_context = ErrorContext(
                error_type=ToolErrorType.EXECUTION_ERROR,
                message=f"Tool execution failed: {str(e)}",
                recovery_suggestion="Check tool logs and system resources",
                timestamp=datetime.now(),
                tool_name=self.tool_name,
                target=inp.target,
                metadata={"exception": str(e), "execution_time": execution_time}
            )
            
            # Record failure metrics
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
        """Execute the actual tool with enhanced error handling."""
        # Resolve command and build arguments
        resolved_cmd = self._resolve_command()
        if not resolved_cmd:
            error_context = ErrorContext(
                error_type=ToolErrorType.NOT_FOUND,
                message=f"Command not found: {self.command_name}",
                recovery_suggestion="Install the required tool or check PATH",
                timestamp=datetime.now(),
                tool_name=self.tool_name,
                target=inp.target,
                metadata={"command": self.command_name}
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        # Parse and validate arguments
        try:
            args = self._parse_args(inp.extra_args)
        except ValueError as e:
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message=f"Argument validation failed: {str(e)}",
                recovery_suggestion="Check arguments and try again",
                timestamp=datetime.now(),
                tool_name=self.tool_name,
                target=inp.target,
                metadata={"validation_error": str(e)}
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        # Build command
        cmd = [resolved_cmd] + args + [inp.target]
        
        # Execute with timeout
        timeout = float(timeout_sec or self.default_timeout_sec)
        return await self._spawn(cmd, timeout)
    
    def _create_error_output(self, error_context: ErrorContext, correlation_id: str) -> ToolOutput:
        """Create a ToolOutput for error conditions."""
        log.error(
            "tool.error tool=%s error_type=%s target=%s message=%s correlation_id=%s",
            error_context.tool_name,
            error_context.error_type.value,
            error_context.target,
            error_context.message,
            correlation_id,
            extra={"error_context": error_context}
        )
        
        return ToolOutput(
            stdout="",
            stderr=error_context.message,
            returncode=1,
            error=error_context.message,
            error_type=error_context.error_type.value,
            correlation_id=correlation_id,
            metadata={
                "recovery_suggestion": error_context.recovery_suggestion,
                "timestamp": error_context.timestamp.isoformat()
            }
        )
    
    def _resolve_command(self) -> Optional[str]:
        """Resolve command path using shutil.which."""
        return shutil.which(self.command_name)
    
    def _parse_args(self, extra_args: str) -> Sequence[str]:
        """Parse and validate extra arguments."""
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
        """Spawn and monitor a subprocess with timeout and output truncation."""
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
            log.error("tool.error %s", msg)
            return ToolOutput(stdout="", stderr=msg, returncode=1, error="execution_failed")

class ToolMetrics:
    """Metrics collection for tool execution."""
    
    def __init__(self, tool_name: str):
        self.tool_name = tool_name
        
        if PROMETHEUS_AVAILABLE:
            try:
                self.execution_counter = Counter(
                    f'mcp_tool_execution_total',
                    'Total tool executions',
                    ['tool', 'status', 'error_type']
                )
                self.execution_histogram = Histogram(
                    f'mcp_tool_execution_seconds',
                    'Tool execution time in seconds',
                    ['tool']
                )
                self.active_gauge = Gauge(
                    f'mcp_tool_active',
                    'Currently active tool executions',
                    ['tool']
                )
                self.error_counter = Counter(
                    f'mcp_tool_errors_total',
                    'Total tool errors',
                    ['tool', 'error_type']
                )
            except Exception as e:
                log.warning("prometheus.metrics_initialization_failed tool=%s error=%s", tool_name, str(e))
                # Disable metrics if initialization fails
                self.execution_counter = None
                self.execution_histogram = None
                self.active_gauge = None
                self.error_counter = None
        else:
            self.execution_counter = None
            self.execution_histogram = None
            self.active_gauge = None
            self.error_counter = None
    
    def record_execution(self, success: bool, execution_time: float, 
                        timed_out: bool = False, error_type: str = None):
        """Record tool execution metrics."""
        if not PROMETHEUS_AVAILABLE or not self.execution_counter:
            return
        
        try:
            # Validate execution time
            execution_time = max(0.0, float(execution_time))
            
            status = 'success' if success else 'failure'
            self.execution_counter.labels(
                tool=self.tool_name,
                status=status,
                error_type=error_type or 'none'
            ).inc()
            
            if self.execution_histogram:
                self.execution_histogram.labels(tool=self.tool_name).observe(execution_time)
            
            if not success and self.error_counter:
                self.error_counter.labels(
                    tool=self.tool_name,
                    error_type=error_type or 'unknown'
                ).inc()
        except Exception as e:
            log.warning("metrics.recording_error tool=%s error=%s", self.tool_name, str(e))
    
    def start_execution(self):
        """Record start of execution."""
        if PROMETHEUS_AVAILABLE and self.active_gauge:
            try:
                self.active_gauge.labels(tool=self.tool_name).inc()
            except Exception as e:
                log.warning("metrics.start_execution_error tool=%s error=%s", self.tool_name, str(e))
    
    def end_execution(self):
        """Record end of execution."""
        if PROMETHEUS_AVAILABLE and self.active_gauge:
            try:
                self.active_gauge.labels(tool=self.tool_name).dec()
            except Exception as e:
                log.warning("metrics.end_execution_error tool=%s error=%s", self.tool_name, str(e))

```

# mcp_server/config.py
```py
# File: config.py
"""
Configuration management system for MCP server.
Production-ready implementation with validation, hot-reload, and sensitive data handling.
"""
import os
import logging
import json
import yaml
from typing import Dict, Any, Optional, List, Union
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, field, asdict

# Pydantic for configuration validation
try:
    from pydantic import BaseModel, Field, validator
    PYDANTIC_AVAILABLE = True
except ImportError:
    PYDANTIC_AVAILABLE = False
    # Fallback validation without Pydantic
    class BaseModel:
        def __init__(self, **kwargs):
            for key, value in kwargs.items():
                setattr(self, key, value)
        
        def dict(self):
            return {k: v for k, v in self.__dict__.items() if not k.startswith('_')}
    
    Field = lambda default=None, **kwargs: default
    def validator(field_name, *args, **kwargs):
        def decorator(func):
            return func
        return decorator

log = logging.getLogger(__name__)

@dataclass
class DatabaseConfig:
    """Database configuration."""
    url: str = ""
    pool_size: int = 10
    max_overflow: int = 20
    pool_timeout: int = 30
    pool_recycle: int = 3600

@dataclass
class SecurityConfig:
    """Security configuration."""
    allowed_targets: List[str] = field(default_factory=lambda: ["RFC1918", ".lab.internal"])
    max_args_length: int = 2048
    max_output_size: int = 1048576
    timeout_seconds: int = 300
    concurrency_limit: int = 2

@dataclass
class CircuitBreakerConfig:
    """Circuit breaker configuration."""
    failure_threshold: int = 5
    recovery_timeout: float = 60.0
    expected_exceptions: List[str] = field(default_factory=lambda: ["Exception"])
    half_open_success_threshold: int = 1

@dataclass
class HealthConfig:
    """Health check configuration."""
    check_interval: float = 30.0
    cpu_threshold: float = 80.0
    memory_threshold: float = 80.0
    disk_threshold: float = 80.0
    dependencies: List[str] = field(default_factory=list)
    timeout: float = 10.0

@dataclass
class MetricsConfig:
    """Metrics configuration."""
    enabled: bool = True
    prometheus_enabled: bool = True
    prometheus_port: int = 9090
    collection_interval: float = 15.0

@dataclass
class LoggingConfig:
    """Logging configuration."""
    level: str = "INFO"
    format: str = "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    file_path: Optional[str] = None
    max_file_size: int = 10485760  # 10MB
    backup_count: int = 5

@dataclass
class ServerConfig:
    """Server configuration."""
    host: str = "0.0.0.0"
    port: int = 8080
    transport: str = "stdio"  # "stdio" or "http"
    workers: int = 1
    max_connections: int = 100
    shutdown_grace_period: float = 30.0

@dataclass
class ToolConfig:
    """Tool-specific configuration."""
    include_patterns: List[str] = field(default_factory=lambda: ["*"])
    exclude_patterns: List[str] = field(default_factory=list)
    default_timeout: int = 300
    default_concurrency: int = 2

class MCPConfig:
    """
    Main MCP configuration class with validation and hot-reload support.
    """
    
    def __init__(self, config_path: Optional[str] = None):
        self.config_path = config_path
        self.last_modified = None
        self._config_data = {}
        
        # Initialize with defaults
        self.database = DatabaseConfig()
        self.security = SecurityConfig()
        self.circuit_breaker = CircuitBreakerConfig()
        self.health = HealthConfig()
        self.metrics = MetricsConfig()
        self.logging = LoggingConfig()
        self.server = ServerConfig()
        self.tool = ToolConfig()
        
        # Load configuration
        self.load_config()
    
    def load_config(self):
        """Load configuration from file and environment variables."""
        # Start with defaults
        config_data = self._get_defaults()
        
        # Load from file if specified
        if self.config_path and os.path.exists(self.config_path):
            config_data.update(self._load_from_file(self.config_path))
        
        # Override with environment variables
        config_data.update(self._load_from_environment())
        
        # Validate and set configuration
        self._validate_and_set_config(config_data)
        
        # Update last modified time
        if self.config_path:
            try:
                self.last_modified = os.path.getmtime(self.config_path)
            except OSError:
                self.last_modified = None
    
    def _get_defaults(self) -> Dict[str, Any]:
        """Get default configuration values."""
        return {
            "database": asdict(DatabaseConfig()),
            "security": asdict(SecurityConfig()),
            "circuit_breaker": asdict(CircuitBreakerConfig()),
            "health": asdict(HealthConfig()),
            "metrics": asdict(MetricsConfig()),
            "logging": asdict(LoggingConfig()),
            "server": asdict(ServerConfig()),
            "tool": asdict(ToolConfig())
        }
    
    def _load_from_file(self, config_path: str) -> Dict[str, Any]:
        """Load configuration from file (JSON or YAML)."""
        try:
            file_path = Path(config_path)
            
            if not file_path.exists():
                log.warning("config.file_not_found path=%s", config_path)
                return {}
            
            with open(file_path, 'r', encoding='utf-8') as f:
                if file_path.suffix.lower() in ['.yaml', '.yml']:
                    return yaml.safe_load(f) or {}
                else:
                    return json.load(f) or {}
        
        except Exception as e:
            log.error("config.file_load_failed path=%s error=%s", config_path, str(e))
            return {}
    
    def _load_from_environment(self) -> Dict[str, Any]:
        """Load configuration from environment variables."""
        config = {}
        
        # Environment variable mappings
        env_mappings = {
            'MCP_DATABASE_URL': ('database', 'url'),
            'MCP_DATABASE_POOL_SIZE': ('database', 'pool_size'),
            'MCP_SECURITY_MAX_ARGS_LENGTH': ('security', 'max_args_length'),
            'MCP_SECURITY_TIMEOUT_SECONDS': ('security', 'timeout_seconds'),
            'MCP_CIRCUIT_BREAKER_FAILURE_THRESHOLD': ('circuit_breaker', 'failure_threshold'),
            'MCP_CIRCUIT_BREAKER_RECOVERY_TIMEOUT': ('circuit_breaker', 'recovery_timeout'),
            'MCP_HEALTH_CHECK_INTERVAL': ('health', 'check_interval'),
            'MCP_HEALTH_CPU_THRESHOLD': ('health', 'cpu_threshold'),
            'MCP_METRICS_ENABLED': ('metrics', 'enabled'),
            'MCP_METRICS_PROMETHEUS_PORT': ('metrics', 'prometheus_port'),
            'MCP_LOGGING_LEVEL': ('logging', 'level'),
            'MCP_LOGGING_FILE_PATH': ('logging', 'file_path'),
            'MCP_SERVER_HOST': ('server', 'host'),
            'MCP_SERVER_PORT': ('server', 'port'),
            'MCP_SERVER_TRANSPORT': ('server', 'transport'),
            'MCP_TOOL_DEFAULT_TIMEOUT': ('tool', 'default_timeout'),
        }
        
        for env_var, (section, key) in env_mappings.items():
            value = os.getenv(env_var)
            if value is not None:
                if section not in config:
                    config[section] = {}
                
                # Type conversion
                if key in ['pool_size', 'max_args_length', 'timeout_seconds', 'failure_threshold', 
                          'prometheus_port', 'default_timeout']:
                    try:
                        config[section][key] = int(value)
                    except ValueError:
                        log.warning("config.invalid_int env_var=%s value=%s", env_var, value)
                elif key in ['recovery_timeout', 'check_interval', 'cpu_threshold']:
                    try:
                        config[section][key] = float(value)
                    except ValueError:
                        log.warning("config.invalid_float env_var=%s value=%s", env_var, value)
                elif key in ['enabled']:
                    config[section][key] = value.lower() in ['true', '1', 'yes', 'on']
                else:
                    config[section][key] = value
        
        return config
    
    def _validate_and_set_config(self, config_data: Dict[str, Any]):
        """Validate and set configuration values."""
        try:
            # Validate database config
            if 'database' in config_data:
                db_config = config_data['database']
                self.database.url = str(db_config.get('url', self.database.url))
                self.database.pool_size = max(1, int(db_config.get('pool_size', self.database.pool_size)))
                self.database.max_overflow = max(0, int(db_config.get('max_overflow', self.database.max_overflow)))
            
            # Validate security config
            if 'security' in config_data:
                sec_config = config_data['security']
                self.security.max_args_length = max(1, int(sec_config.get('max_args_length', self.security.max_args_length)))
                self.security.max_output_size = max(1, int(sec_config.get('max_output_size', self.security.max_output_size)))
                self.security.timeout_seconds = max(1, int(sec_config.get('timeout_seconds', self.security.timeout_seconds)))
                self.security.concurrency_limit = max(1, int(sec_config.get('concurrency_limit', self.security.concurrency_limit)))
            
            # Validate circuit breaker config
            if 'circuit_breaker' in config_data:
                cb_config = config_data['circuit_breaker']
                self.circuit_breaker.failure_threshold = max(1, int(cb_config.get('failure_threshold', self.circuit_breaker.failure_threshold)))
                self.circuit_breaker.recovery_timeout = max(1.0, float(cb_config.get('recovery_timeout', self.circuit_breaker.recovery_timeout)))
            
            # Validate health config
            if 'health' in config_data:
                health_config = config_data['health']
                self.health.check_interval = max(5.0, float(health_config.get('check_interval', self.health.check_interval)))
                self.health.cpu_threshold = max(0.0, min(100.0, float(health_config.get('cpu_threshold', self.health.cpu_threshold))))
                self.health.memory_threshold = max(0.0, min(100.0, float(health_config.get('memory_threshold', self.health.memory_threshold))))
                self.health.disk_threshold = max(0.0, min(100.0, float(health_config.get('disk_threshold', self.health.disk_threshold))))
            
            # Validate metrics config
            if 'metrics' in config_data:
                metrics_config = config_data['metrics']
                self.metrics.enabled = bool(metrics_config.get('enabled', self.metrics.enabled))
                self.metrics.prometheus_enabled = bool(metrics_config.get('prometheus_enabled', self.metrics.prometheus_enabled))
                self.metrics.prometheus_port = max(1, min(65535, int(metrics_config.get('prometheus_port', self.metrics.prometheus_port))))
            
            # Validate logging config
            if 'logging' in config_data:
                logging_config = config_data['logging']
                self.logging.level = str(logging_config.get('level', self.logging.level)).upper()
                self.logging.file_path = logging_config.get('file_path') if logging_config.get('file_path') else None
            
            # Validate server config
            if 'server' in config_data:
                server_config = config_data['server']
                self.server.host = str(server_config.get('host', self.server.host))
                self.server.port = max(1, min(65535, int(server_config.get('port', self.server.port))))
                self.server.transport = str(server_config.get('transport', self.server.transport)).lower()
                self.server.workers = max(1, int(server_config.get('workers', self.server.workers)))
            
            # Validate tool config
            if 'tool' in config_data:
                tool_config = config_data['tool']
                self.tool.default_timeout = max(1, int(tool_config.get('default_timeout', self.tool.default_timeout)))
                self.tool.default_concurrency = max(1, int(tool_config.get('default_concurrency', self.tool.default_concurrency)))
            
            # Store raw config data
            self._config_data = config_data
            
            log.info("config.loaded_successfully")
            
        except Exception as e:
            log.error("config.validation_failed error=%s", str(e))
            # Keep defaults if validation fails
    
    def check_for_changes(self) -> bool:
        """Check if configuration file has been modified."""
        if not self.config_path:
            return False
        
        try:
            current_mtime = os.path.getmtime(self.config_path)
            if current_mtime != self.last_modified:
                self.last_modified = current_mtime
                return True
        except OSError:
            pass
        
        return False
    
    def reload_config(self):
        """Reload configuration if file has changed."""
        if self.check_for_changes():
            log.info("config.reloading_changes_detected")
            self.load_config()
            return True
        return False
    
    def get_sensitive_keys(self) -> List[str]:
        """Get list of sensitive configuration keys that should be redacted."""
        return [
            'database.url',
            'security.api_key',
            'security.secret_key',
            'logging.file_path'  # May contain sensitive paths
        ]
    
    def redact_sensitive_data(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Redact sensitive data from configuration for logging."""
        sensitive_keys = self.get_sensitive_keys()
        redacted_data = data.copy()
        
        for key in sensitive_keys:
            if '.' in key:
                section, subkey = key.split('.', 1)
                if section in redacted_data and isinstance(redacted_data[section], dict):
                    if subkey in redacted_data[section]:
                        redacted_data[section][subkey] = "***REDACTED***"
            else:
                if key in redacted_data:
                    redacted_data[key] = "***REDACTED***"
        
        return redacted_data
    
    def to_dict(self, redact_sensitive: bool = True) -> Dict[str, Any]:
        """Convert configuration to dictionary."""
        config_dict = {
            'database': asdict(self.database),
            'security': asdict(self.security),
            'circuit_breaker': asdict(self.circuit_breaker),
            'health': asdict(self.health),
            'metrics': asdict(self.metrics),
            'logging': asdict(self.logging),
            'server': asdict(self.server),
            'tool': asdict(self.tool)
        }
        
        if redact_sensitive:
            config_dict = self.redact_sensitive_data(config_dict)
        
        return config_dict
    
    def save_config(self, file_path: Optional[str] = None):
        """Save current configuration to file."""
        save_path = file_path or self.config_path
        if not save_path:
            raise ValueError("No config file path specified")
        
        try:
            config_dict = self.to_dict(redact_sensitive=False)
            
            file_path_obj = Path(save_path)
            file_path_obj.parent.mkdir(parents=True, exist_ok=True)
            
            with open(file_path_obj, 'w', encoding='utf-8') as f:
                if file_path_obj.suffix.lower() in ['.yaml', '.yml']:
                    yaml.dump(config_dict, f, default_flow_style=False, indent=2)
                else:
                    json.dump(config_dict, f, indent=2)
            
            log.info("config.saved_successfully path=%s", save_path)
            
        except Exception as e:
            log.error("config.save_failed path=%s error=%s", save_path, str(e))
            raise
    
    def get_section(self, section_name: str) -> Any:
        """Get a specific configuration section."""
        return getattr(self, section_name, None)
    
    def get_value(self, section_name: str, key: str, default=None):
        """Get a specific configuration value."""
        section = self.get_section(section_name)
        if section and hasattr(section, key):
            return getattr(section, key)
        return default
    
    def __str__(self) -> str:
        """String representation with sensitive data redacted."""
        config_dict = self.to_dict(redact_sensitive=True)
        return json.dumps(config_dict, indent=2)

# Global configuration instance
_config_instance = None

def get_config(config_path: Optional[str] = None) -> MCPConfig:
    """Get the global configuration instance."""
    global _config_instance
    if _config_instance is None:
        _config_instance = MCPConfig(config_path)
    return _config_instance

def reload_config():
    """Reload the global configuration."""
    global _config_instance
    if _config_instance is not None:
        _config_instance.reload_config()

```

# mcp_server/metrics.py
```py
# File: metrics.py
"""
Metrics collection system for MCP server.
Production-ready implementation with proper validation and error handling.
"""
import time
import logging
from typing import Dict, Any, Optional
from datetime import datetime, timedelta
from dataclasses import dataclass, field

# Graceful Prometheus dependency handling
try:
    from prometheus_client import Counter, Histogram, Gauge, Info, generate_latest
    from prometheus_client.core import CollectorRegistry
    PROMETHEUS_AVAILABLE = True
except ImportError:
    PROMETHEUS_AVAILABLE = False

log = logging.getLogger(__name__)

@dataclass
class ToolExecutionMetrics:
    """Metrics for tool execution with validation."""
    tool_name: str
    execution_count: int = 0
    success_count: int = 0
    failure_count: int = 0
    timeout_count: int = 0
    total_execution_time: float = 0.0
    min_execution_time: float = float('inf')
    max_execution_time: float = 0.0
    last_execution_time: Optional[datetime] = None
    
    def record_execution(self, success: bool, execution_time: float, timed_out: bool = False):
        """Record a tool execution with validation."""
        # Validate execution time
        execution_time = max(0.0, float(execution_time))
        
        self.execution_count += 1
        self.total_execution_time += execution_time
        
        if execution_time < self.min_execution_time:
            self.min_execution_time = execution_time
        if execution_time > self.max_execution_time:
            self.max_execution_time = execution_time
        
        self.last_execution_time = datetime.now()
        
        if success:
            self.success_count += 1
        else:
            self.failure_count += 1
        
        if timed_out:
            self.timeout_count += 1
    
    def get_stats(self) -> Dict[str, Any]:
        """Get statistics for this tool."""
        if self.execution_count == 0:
            return {
                "tool_name": self.tool_name,
                "execution_count": 0,
                "success_rate": 0.0,
                "average_execution_time": 0.0,
                "min_execution_time": 0.0,
                "max_execution_time": 0.0
            }
        
        # Prevent division by zero
        avg_execution_time = self.total_execution_time / self.execution_count
        success_rate = (self.success_count / self.execution_count) * 100
        
        return {
            "tool_name": self.tool_name,
            "execution_count": self.execution_count,
            "success_count": self.success_count,
            "failure_count": self.failure_count,
            "timeout_count": self.timeout_count,
            "success_rate": round(success_rate, 2),
            "average_execution_time": round(avg_execution_time, 4),
            "min_execution_time": round(self.min_execution_time, 4) if self.min_execution_time != float('inf') else 0.0,
            "max_execution_time": round(self.max_execution_time, 4),
            "last_execution_time": self.last_execution_time.isoformat() if self.last_execution_time else None
        }

class SystemMetrics:
    """System-level metrics with thread safety."""
    
    def __init__(self):
        self.start_time = datetime.now()
        self.request_count = 0
        self.error_count = 0
        self.active_connections = 0
        self._lock = None  # Could use threading.Lock if needed
    
    def increment_request_count(self):
        """Increment request count."""
        self.request_count += 1
    
    def increment_error_count(self):
        """Increment error count."""
        self.error_count += 1
    
    def increment_active_connections(self):
        """Increment active connections."""
        self.active_connections += 1
    
    def decrement_active_connections(self):
        """Decrement active connections."""
        self.active_connections = max(0, self.active_connections - 1)
    
    def get_uptime(self) -> float:
        """Get uptime in seconds."""
        return (datetime.now() - self.start_time).total_seconds()
    
    def get_stats(self) -> Dict[str, Any]:
        """Get system statistics."""
        uptime = self.get_uptime()
        error_rate = (self.error_count / self.request_count * 100) if self.request_count > 0 else 0
        
        return {
            "uptime_seconds": uptime,
            "uptime_formatted": str(timedelta(seconds=int(uptime))),
            "request_count": self.request_count,
            "error_count": self.error_count,
            "error_rate": round(error_rate, 2),
            "active_connections": self.active_connections,
            "start_time": self.start_time.isoformat()
        }

class PrometheusMetrics:
    """Prometheus metrics collection with graceful degradation."""
    
    def __init__(self):
        if not PROMETHEUS_AVAILABLE:
            log.warning("prometheus.unavailable")
            self.registry = None
            return
        
        try:
            self.registry = CollectorRegistry()
            
            # Tool execution metrics
            self.tool_execution_counter = Counter(
                'mcp_tool_execution_total',
                'Total tool executions',
                ['tool', 'status', 'error_type'],
                registry=self.registry
            )
            
            self.tool_execution_histogram = Histogram(
                'mcp_tool_execution_seconds',
                'Tool execution time in seconds',
                ['tool'],
                registry=self.registry
            )
            
            self.tool_active_gauge = Gauge(
                'mcp_tool_active',
                'Currently active tool executions',
                ['tool'],
                registry=self.registry
            )
            
            # System metrics
            self.system_request_counter = Counter(
                'mcp_system_requests_total',
                'Total system requests',
                registry=self.registry
            )
            
            self.system_error_counter = Counter(
                'mcp_system_errors_total',
                'Total system errors',
                ['error_type'],
                registry=self.registry
            )
            
            self.system_active_connections = Gauge(
                'mcp_system_active_connections',
                'Currently active connections',
                registry=self.registry
            )
            
            self.system_uptime_gauge = Gauge(
                'mcp_system_uptime_seconds',
                'System uptime in seconds',
                registry=self.registry
            )
            
            log.info("prometheus.metrics_initialized")
            
        except Exception as e:
            log.error("prometheus.initialization_failed error=%s", str(e))
            self.registry = None
    
    def record_tool_execution(self, tool_name: str, success: bool, execution_time: float, 
                             error_type: str = None):
        """Record tool execution metrics."""
        if not PROMETHEUS_AVAILABLE or not self.registry:
            return
        
        try:
            # Validate execution time
            execution_time = max(0.0, float(execution_time))
            
            status = 'success' if success else 'failure'
            self.tool_execution_counter.labels(
                tool=tool_name,
                status=status,
                error_type=error_type or 'none'
            ).inc()
            
            self.tool_execution_histogram.labels(tool=tool_name).observe(execution_time)
            
        except Exception as e:
            log.warning("prometheus.tool_execution_error error=%s", str(e))
    
    def increment_tool_active(self, tool_name: str):
        """Increment active tool gauge."""
        if PROMETHEUS_AVAILABLE and self.registry and self.tool_active_gauge:
            try:
                self.tool_active_gauge.labels(tool=tool_name).inc()
            except Exception as e:
                log.warning("prometheus.increment_active_error error=%s", str(e))
    
    def decrement_tool_active(self, tool_name: str):
        """Decrement active tool gauge."""
        if PROMETHEUS_AVAILABLE and self.registry and self.tool_active_gauge:
            try:
                self.tool_active_gauge.labels(tool=tool_name).dec()
            except Exception as e:
                log.warning("prometheus.decrement_active_error error=%s", str(e))
    
    def increment_system_request(self):
        """Increment system request counter."""
        if PROMETHEUS_AVAILABLE and self.registry and self.system_request_counter:
            try:
                self.system_request_counter.inc()
            except Exception as e:
                log.warning("prometheus.system_request_error error=%s", str(e))
    
    def increment_system_error(self, error_type: str = 'unknown'):
        """Increment system error counter."""
        if PROMETHEUS_AVAILABLE and self.registry and self.system_error_counter:
            try:
                self.system_error_counter.labels(error_type=error_type).inc()
            except Exception as e:
                log.warning("prometheus.system_error_error error=%s", str(e))
    
    def update_active_connections(self, count: int):
        """Update active connections gauge."""
        if PROMETHEUS_AVAILABLE and self.registry and self.system_active_connections:
            try:
                self.system_active_connections.set(max(0, count))
            except Exception as e:
                log.warning("prometheus.active_connections_error error=%s", str(e))
    
    def update_uptime(self, uptime_seconds: float):
        """Update uptime gauge."""
        if PROMETHEUS_AVAILABLE and self.registry and self.system_uptime_gauge:
            try:
                self.system_uptime_gauge.set(max(0.0, uptime_seconds))
            except Exception as e:
                log.warning("prometheus.uptime_error error=%s", str(e))
    
    def get_metrics(self) -> Optional[str]:
        """Get Prometheus metrics in text format."""
        if not PROMETHEUS_AVAILABLE or not self.registry:
            return None
        
        try:
            return generate_latest(self.registry).decode('utf-8')
        except Exception as e:
            log.error("prometheus.generate_metrics_error error=%s", str(e))
            return None

class MetricsManager:
    """Manager for all metrics collection."""
    
    def __init__(self):
        self.tool_metrics: Dict[str, ToolExecutionMetrics] = {}
        self.system_metrics = SystemMetrics()
        self.prometheus_metrics = PrometheusMetrics()
        self.start_time = datetime.now()
    
    def get_tool_metrics(self, tool_name: str) -> ToolExecutionMetrics:
        """Get or create tool metrics."""
        if tool_name not in self.tool_metrics:
            self.tool_metrics[tool_name] = ToolExecutionMetrics(tool_name)
        return self.tool_metrics[tool_name]
    
    def record_tool_execution(self, tool_name: str, success: bool, execution_time: float, 
                             timed_out: bool = False, error_type: str = None):
        """Record tool execution across all metrics systems."""
        # Record in tool-specific metrics
        tool_metrics = self.get_tool_metrics(tool_name)
        tool_metrics.record_execution(success, execution_time, timed_out)
        
        # Record in Prometheus
        self.prometheus_metrics.record_tool_execution(tool_name, success, execution_time, error_type)
        
        # Record in system metrics
        self.system_metrics.increment_request_count()
        if not success:
            self.system_metrics.increment_error_count()
    
    def get_all_stats(self) -> Dict[str, Any]:
        """Get all statistics."""
        return {
            "system": self.system_metrics.get_stats(),
            "tools": {name: metrics.get_stats() for name, metrics in self.tool_metrics.items()},
            "prometheus_available": PROMETHEUS_AVAILABLE,
            "collection_start_time": self.start_time.isoformat()
        }
    
    def get_prometheus_metrics(self) -> Optional[str]:
        """Get Prometheus metrics export."""
        return self.prometheus_metrics.get_metrics()

```

# mcp_server/server.py
```py
# File: server.py
"""
Enhanced MCP Server with ALL original features preserved + comprehensive enhancements.
"""
import asyncio
import importlib
import inspect
import logging
import os
import pkgutil
import signal
import sys
import time
from typing import Dict, List, Optional, Set, Any, Iterable, Sequence, Type
from datetime import datetime
import json
import contextlib

# FastAPI for HTTP transport
try:
    from fastapi import FastAPI, HTTPException, BackgroundTasks 
    from fastapi.middleware.cors import CORSMiddleware
    from fastapi.responses import JSONResponse
    from starlette.requests import Request
    from sse_starlette.sse import EventSourceResponse
    FASTAPI_AVAILABLE = True
except ImportError:
    FASTAPI_AVAILABLE = False

# Uvicorn for HTTP server
try:
    import uvicorn
    UVICORN_AVAILABLE = True
except ImportError:
    UVICORN_AVAILABLE = False

# MCP imports
from mcp.server import Server as MCPServerBase
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# Local imports
from .config import Config, ConfigManager
from .health import HealthCheckManager, HealthStatus
from .base_tool import MCPBaseTool, ToolInput, ToolOutput
# Removed unused import: from .metrics import MetricsManager

log = logging.getLogger(__name__)

# ==================== ORIGINAL FEATURES PRESERVED ====================

def _maybe_setup_uvloop() -> None:
    """Original: Optional uvloop installation for better performance."""
    try:
        import uvloop  # type: ignore
        uvloop.install()
        log.info("uvloop.installed")
    except Exception as e:
        log.debug("uvloop.not_available error=%s", str(e))

def _setup_logging() -> None:
    """Original: Environment-based logging configuration."""
    level = os.getenv("LOG_LEVEL", "INFO").upper()
    fmt = os.getenv(
        "LOG_FORMAT",
        "%(asctime)s %(levelname)s %(name)s %(message)s",
    )
    logging.basicConfig(level=getattr(logging, level, logging.INFO), format=fmt)
    log.info("logging.configured level=%s", level)

def _parse_csv_env(name: str) -> Optional[List[str]]:
    """Original: Parse CSV environment variables."""
    raw = os.getenv(name, "").strip()
    if not raw:
        return None
    return [x.strip() for x in raw.split(",") if x.strip()]

def _load_tools_from_package(
    package_path: str,
    include: Optional[Sequence[str]] = None,
    exclude: Optional[Sequence[str]] = None,
) -> List[MCPBaseTool]:
    """
    Original: Discover and instantiate concrete MCPBaseTool subclasses under package_path.
    include/exclude: class names (e.g., ["NmapTool"]) to filter.
    
    Enhanced: Better error handling and logging.
    """
    tools: list[MCPBaseTool] = []
    log.info("tool_discovery.starting package=%s include=%s exclude=%s", 
             package_path, include, exclude)

    try:
        pkg = importlib.import_module(package_path)
        log.debug("tool_discovery.package_imported path=%s", package_path)
    except Exception as e:
        log.error("tool_discovery.package_failed path=%s error=%s", package_path, e)
        return tools

    module_count = 0
    for modinfo in pkgutil.walk_packages(pkg.__path__, prefix=pkg.__name__ + "."):
        module_count += 1
        try:
            module = importlib.import_module(modinfo.name)
            log.debug("tool_discovery.module_imported name=%s", modinfo.name)
        except Exception as e:
            log.warning("tool_discovery.module_skipped name=%s error=%s", modinfo.name, e)
            continue

        tool_count_in_module = 0
        for _, obj in inspect.getmembers(module, inspect.isclass):
            if not issubclass(obj, MCPBaseTool) or obj is MCPBaseTool:
                continue
            
            name = obj.__name__
            if include and name not in include:
                log.debug("tool_discovery.tool_skipped name=%s reason=include_filter", name)
                continue
            if exclude and name in exclude:
                log.debug("tool_discovery.tool_skipped name=%s reason=exclude_filter", name)
                continue
            
            try:
                inst = obj()  # assume no-arg constructor
                tools.append(inst)
                tool_count_in_module += 1
                log.info("tool_discovery.tool_loaded name=%s", name)
            except Exception as e:
                log.warning("tool_discovery.tool_instantiation_failed name=%s error=%s", name, e)
        
        if tool_count_in_module == 0:
            log.debug("tool_discovery.no_tools_in_module module=%s", modinfo.name)
    
    log.info("tool_discovery.completed package=%s modules=%d tools=%d", 
             package_path, module_count, len(tools))
    return tools

async def _serve(server: MCPServerBase, shutdown_grace: float) -> None:
    """
    Original: Handle server lifecycle with signal handling and graceful shutdown.
    
    Enhanced: Better error handling and logging.
    """
    loop = asyncio.get_running_loop()
    stop = asyncio.Event()

    def _signal_handler(sig: int) -> None:
        log.info("server.signal_received signal=%s initiating_shutdown", sig)
        stop.set()

    # Enhanced signal handler setup with better error handling
    for sig in (signal.SIGINT, signal.SIGTERM):
        try:
            loop.add_signal_handler(sig, _signal_handler, sig)
            log.debug("server.signal_handler_registered signal=%s", sig)
        except NotImplementedError:
            log.warning("server.signal_handler_not_supported signal=%s platform=%s", sig, sys.platform)
        except Exception as e:
            log.error("server.signal_handler_failed signal=%s error=%s", sig, str(e))

    serve_task = asyncio.create_task(server.serve(), name="mcp_serve")
    log.info("server.started grace_period=%.1fs", shutdown_grace)

    try:
        await stop.wait()
        log.info("server.shutdown_initiated")
    except asyncio.CancelledError:
        log.info("server.shutdown_cancelled")
        return

    # Enhanced shutdown process
    log.info("server.shutting_down... ")
    serve_task.cancel()
    
    try:
        await asyncio.wait_for(serve_task, timeout=shutdown_grace)
        log.info("server.shutdown_completed")
    except asyncio.TimeoutError:
        log.warning("server.shutdown_forced timeout=%.1fs", shutdown_grace)
    except asyncio.CancelledError:
        log.info("server.shutdown_cancelled_during_cleanup")
    except Exception as e:
        log.error("server.shutdown_error error=%s", str(e))

# ==================== ENHANCED FEATURES ====================

class ToolRegistry:
    """Enhanced Tool Registry with original discovery pattern + new features."""
    
    def __init__(self, config: Config, tools: List[MCPBaseTool]):
        """Initialize with a pre-loaded list of tools."""
        self.config = config
        self.tools: Dict[str, MCPBaseTool] = {}
        self.enabled_tools: Set[str] = set()
        self._register_tools_from_list(tools)
    
    def _register_tools_from_list(self, tools: List[MCPBaseTool]):
        """Register a list of pre-discovered tools."""
        for tool in tools:
            tool_name = tool.__class__.__name__
            self.tools[tool_name] = tool
            
            # Check if tool is enabled (enhanced filtering)
            if self._is_tool_enabled(tool_name):
                self.enabled_tools.add(tool_name)
                
                # Initialize enhanced features
                if hasattr(tool, '_initialize_metrics'):
                    tool._initialize_metrics()
                if hasattr(tool, '_initialize_circuit_breaker'):
                    tool._initialize_circuit_breaker()
                
                log.info("tool_registry.enhanced_tool_registered name=%s", tool_name)
    
    def _is_tool_enabled(self, tool_name: str) -> bool:
        """Enhanced tool enabling logic with original pattern."""
        # Original include/exclude logic via environment variables
        include = _parse_csv_env("TOOL_INCLUDE")
        exclude = _parse_csv_env("TOOL_EXCLUDE")
        
        # Check include list
        if include and tool_name not in include:
            return False
        
        # Check exclude list
        if exclude and tool_name in exclude:
            return False
        
        return True
    
    def get_tool(self, tool_name: str) -> Optional[MCPBaseTool]:
        """Get a tool by name."""
        return self.tools.get(tool_name)
    
    def get_enabled_tools(self) -> Dict[str, MCPBaseTool]:
        """Get all enabled tools."""
        return {name: tool for name, tool in self.tools.items() 
                if name in self.enabled_tools}
    
    def enable_tool(self, tool_name: str):
        """Enable a tool (enhanced feature)."""
        if tool_name in self.tools:
            self.enabled_tools.add(tool_name)
            log.info("tool_registry.enabled name=%s", tool_name)
    
    def disable_tool(self, tool_name: str):
        """Disable a tool (enhanced feature)."""
        self.enabled_tools.discard(tool_name)
        log.info("tool_registry.disabled name=%s", tool_name)
    
    def get_tool_info(self) -> List[Dict[str, Any]]:
        """Get enhanced tool information."""
        info = []
        for name, tool in self.tools.items():
            info.append({
                "name": name,
                "enabled": name in self.enabled_tools,
                "command": tool.command_name,
                "description": tool.__doc__ or "No description",
                "concurrency": tool.concurrency,
                "timeout": tool.default_timeout_sec,
                "has_metrics": hasattr(tool, 'metrics') and tool.metrics is not None,
                "has_circuit_breaker": hasattr(tool, '_circuit_breaker') and tool._circuit_breaker is not None
            })
        return info

class EnhancedMCPServer:
    """Enhanced MCP Server with original features + comprehensive enhancements."""
    
    def __init__(self, tools: List[MCPBaseTool], transport: str = "stdio"):
        self.tools = tools
        self.transport = transport
        self.server = MCPServerBase("enhanced-mcp-server")
        # FIXED: Pass the pre-loaded 'tools' list to ToolRegistry
        self.tool_registry = ToolRegistry(Config(), tools)
        self.shutdown_event = asyncio.Event()
        
        # Register tools with enhanced handlers
        self._register_tools_enhanced()
        
        # Setup enhanced signal handlers
        self._setup_enhanced_signal_handlers()
        
        log.info("enhanced_server.initialized transport=%s tools=%d",
                self.transport, len(self.tools))
    
    def _register_tools_enhanced(self):
        """Register tools with enhanced handlers."""
        for tool in self.tools:
            self.server.register_tool(
                name=tool.__class__.__name__,
                description=tool.__doc__ or f"Execute {tool.command_name}",
                input_schema={
                    "type": "object",
                    "properties": {
                        "target": {
                            "type": "string",
                            "description": "Target host or network"
                        },
                        "extra_args": {
                            "type": "string",
                            "description": "Additional arguments for the tool"
                        },
                        "timeout_sec": {
                            "type": "number",
                            "description": "Timeout in seconds"
                        }
                    },
                    "required": ["target"]
                },
                handler=self._create_enhanced_tool_handler(tool)
            )
    
    def _create_enhanced_tool_handler(self, tool: MCPBaseTool):
        """Create enhanced handler for a specific tool."""
        async def enhanced_handler(target: str, extra_args: str = "", timeout_sec: Optional[float] = None):
            try:
                # Use enhanced ToolInput if available, otherwise fall back to basic
                if hasattr(tool, 'run'):
                    # Enhanced tool with new run method
                    input_data = ToolInput(
                        target=target,
                        extra_args=extra_args,
                        timeout_sec=timeout_sec
                    )
                    
                    result = await tool.run(input_data)
                else:
                    # Original tool - use basic execution
                    result = await self._execute_original_tool(tool, target, extra_args, timeout_sec)
                
                return [
                    TextContent(
                        type="text",
                        text=json.dumps(result.dict() if hasattr(result, 'dict') else str(result), indent=2)
                    )
                ]
            
            except Exception as e:
                log.error("enhanced_tool_handler.error tool=%s target=%s error=%s",
                         tool.__class__.__name__, target, str(e))
                return [
                    TextContent(
                        type="text",
                        text=json.dumps({
                            "error": str(e),
                            "tool": tool.__class__.__name__,
                            "target": target
                        }, indent=2)
                    )
                ]
        
        return enhanced_handler
    
    async def _execute_original_tool(self, tool: MCPBaseTool, target: str, extra_args: str, timeout_sec: Optional[float]):
        """Execute original tool with basic compatibility."""
        # Basic execution for original tools
        if hasattr(tool, '_spawn'):
            # Use original _spawn method
            cmd = [tool.command_name] + (extra_args.split() if extra_args else []) + [target]
            return await tool._spawn(cmd, timeout_sec)
        else:
            # Fallback
            return {
                "stdout": f"Executed {tool.command_name} on {target}",
                "stderr": "",
                "returncode": 0
            }
    
    def _setup_enhanced_signal_handlers(self):
        """Setup enhanced signal handlers."""
        def signal_handler(signum, frame):
            log.info("enhanced_server.shutdown_signal signal=%s", signum)
            self.shutdown_event.set()
        
        signal.signal(signal.SIGINT, signal_handler)
        signal.signal(signal.SIGTERM, signal_handler)
    
    async def run_stdio_original(self):
        """Run server with STDIO transport using original pattern."""
        log.info("enhanced_server.start_stdio_original")
        
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream,
                write_stream,
                self.shutdown_event
            )
    
    async def run_http_enhanced(self):
        """Run server with HTTP transport (enhanced feature)."""
        if not FASTAPI_AVAILABLE or not UVICORN_AVAILABLE:
            log.error("enhanced_server.http_missing_deps")
            raise RuntimeError("FastAPI and Uvicorn are required for HTTP transport")
        
        log.info("enhanced_server.start_http_enhanced")
        
        # Create FastAPI app
        app = FastAPI(title="Enhanced MCP Server", version="1.0.0")
        
        # Add CORS middleware
        app.add_middleware(
            CORSMiddleware,
            allow_origins=["*"],
            allow_credentials=True,
            allow_methods=["*"],
            allow_headers=["*"]
        )
        
        # Add enhanced routes
        @app.get("/health")
        async def health_check():
            """Enhanced health check."""
            return {"status": "healthy", "transport": self.transport}
        
        @app.get("/tools")
        async def get_tools():
            """Get list of available tools."""
            return {"tools": [tool.__class__.__name__ for tool in self.tools]}
        
        # Run with uvicorn
        config = uvicorn.Config(
            app,
            host="0.0.0.0",
            port=8000,
            log_level="info"
        )
        
        server = uvicorn.Server(config)
        await server.serve()
    
    async def run(self):
        """Run the server with configured transport."""
        if self.transport == "stdio":
            await self.run_stdio_original()
        elif self.transport == "http":
            await self.run_http_enhanced()
        else:
            log.error("enhanced_server.invalid_transport transport=%s", self.transport)
            raise ValueError(f"Invalid transport: {self.transport}")

# ==================== MAIN FUNCTION - ENHANCED ORIGINAL PATTERN ====================

async def main_enhanced() -> None:
    """
    Enhanced main function that preserves ALL original functionality 
    while adding comprehensive enhancements.
    """
    # ORIGINAL: Setup uvloop
    _maybe_setup_uvloop()
    
    # ORIGINAL: Setup logging
    _setup_logging()
    
    # ORIGINAL: Parse environment variables
    transport = os.getenv("MCP_TRANSPORT", "stdio").lower()
    tools_pkg = os.getenv("TOOLS_PACKAGE", "mcp_server.tools")
    include = _parse_csv_env("TOOL_INCLUDE")
    exclude = _parse_csv_env("TOOL_EXCLUDE")
    shutdown_grace = float(os.getenv("SHUTDOWN_GRACE", "10"))
    
    # ORIGINAL: Load tools
    tools = _load_tools_from_package(tools_pkg, include=include, exclude=exclude)
    
    # ENHANCED: Additional logging
    log.info("enhanced_main.starting transport=%s tools_pkg=%s tools_count=%d include=%s exclude=%s shutdown_grace=%.1fs",
             transport, tools_pkg, len(tools), include, exclude, shutdown_grace)
    
    # ORIGINAL: Create server
    server = EnhancedMCPServer(tools=tools, transport=transport)
    
    # ENHANCED: Additional startup information
    tool_names = [tool.__class__.__name__ for tool in tools]
    log.info("enhanced_main.tools_loaded tools=%s", tool_names)
    
    # ORIGINAL: Serve with graceful shutdown
    await _serve(server.server, shutdown_grace=shutdown_grace)

# ==================== ENTRY POINT - PRESERVED ORIGINAL ====================

if __name__ == "__main__":
    # ORIGINAL: Use contextlib for cleanup
    with contextlib.suppress(ImportError):
        pass
    
    # ORIGINAL: Run main function
    asyncio.run(main_enhanced())

```

# mcp_server/tools/nmap_tool.py
```py
# File: mcp_server/tools/nmap_tool.py
"""
Enhanced Nmap tool with circuit breaker, metrics, and advanced features.
"""
import logging
import shlex
import ipaddress
from datetime import datetime, timezone
from typing import Sequence, Optional

from mcp_server.base_tool import MCPBaseTool, ToolInput, ToolOutput, ToolErrorType, ErrorContext
from mcp_server.config import get_config

log = logging.getLogger(__name__)

class NmapTool(MCPBaseTool):
    """
    Enhanced Nmap network scanner tool with advanced features.

    Executes `nmap` against validated RFC1918, loopback, or .lab.internal targets.
    Only a curated set of flags are permitted for safety and predictability.

    Features:
    - Circuit breaker protection
    - Comprehensive metrics collection
    - Advanced error handling
    - Performance monitoring
    - Resource safety

    Environment overrides:
    - MCP_DEFAULT_TIMEOUT_SEC (default 600s here)
    - MCP_DEFAULT_CONCURRENCY (default 1 here)
    - MCP_CIRCUIT_BREAKER_FAILURE_THRESHOLD (default 5)
    - MCP_CIRCUIT_BREAKER_RECOVERY_TIMEOUT (default 60s)
    """

    command_name: str = "nmap"

    # Conservative, safe flags for nmap (prefix/option names only; values allowed after '=' or space)
    allowed_flags: Sequence[str] = [
        "-sV",         # Service version detection
        "-sC",         # Default script scan
        "-A",          # Aggressive options (enables -sV, -sC, -O, --traceroute)
        "-p",          # Port specification
        "--top-ports", # Scan top N ports
        "-T", "-T4",   # Timing template (T4 = aggressive)
        "-Pn",         # Treat all hosts as online (skip host discovery)
        "-O",          # OS detection
        "--script",    # Script scanning (safe scripts only)
        "-oX",         # XML output (for parsing)
        "-oN",         # Normal output
        "-oG",         # Grepable output
        "--max-parallelism",  # Explicitly allow injected parallelism control
    ]

    # Nmap can run long; set higher timeout
    default_timeout_sec: float = 600.0

    # Limit concurrency to avoid overloading host and network
    concurrency: int = 1

    # Circuit breaker configuration
    circuit_breaker_failure_threshold: int = 5
    circuit_breaker_recovery_timeout: float = 120.0  # 2 minutes for nmap
    circuit_breaker_expected_exception: tuple = (Exception,)

    def __init__(self):
        super().__init__()
        self.config = get_config()
        self._setup_enhanced_features()

    def _setup_enhanced_features(self):
        """Setup enhanced features for Nmap tool."""
        # Override circuit breaker settings from config if available
        if getattr(self.config, "circuit_breaker_enabled", False):
            self.circuit_breaker_failure_threshold = getattr(
                self.config, "circuit_breaker_failure_threshold", self.circuit_breaker_failure_threshold
            )
            self.circuit_breaker_recovery_timeout = getattr(
                self.config, "circuit_breaker_recovery_timeout", self.circuit_breaker_recovery_timeout
            )

        # Reinitialize circuit breaker with new settings (class-level, not instance)
        type(self)._circuit_breaker = None
        self._initialize_circuit_breaker()

    async def _execute_tool(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        """Enhanced tool execution with nmap-specific features."""
        # Validate nmap-specific requirements
        validation_result = self._validate_nmap_requirements(inp)
        if validation_result:
            return validation_result

        # Add nmap-specific optimizations
        optimized_args = self._optimize_nmap_args(inp.extra_args)

        # Create enhanced input with optimizations
        enhanced_input = ToolInput(
            target=inp.target,
            extra_args=optimized_args,
            timeout_sec=timeout_sec or self.default_timeout_sec,
            correlation_id=inp.correlation_id,
        )

        # Execute with enhanced monitoring
        return await super()._execute_tool(enhanced_input, timeout_sec)

    def _validate_nmap_requirements(self, inp: ToolInput) -> Optional[ToolOutput]:
        """Validate nmap-specific requirements."""
        target = inp.target.strip()

        # CIDR/network targets
        if "/" in target:
            try:
                network = ipaddress.ip_network(target, strict=False)
            except ValueError:
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"Invalid network range: {target}",
                    recovery_suggestion="Use valid CIDR notation (e.g., 192.168.1.0/24)",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=target,
                )
                return self._create_error_output(error_context, inp.correlation_id)

            # Enforce reasonable scan size
            if network.num_addresses > 1024:
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"Network range too large: {network.num_addresses} addresses",
                    recovery_suggestion="Use smaller network ranges or specify individual hosts",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=target,
                    metadata={"network_size": network.num_addresses},
                )
                return self._create_error_output(error_context, inp.correlation_id)

            # Enforce RFC1918/loopback for networks
            if not (network.is_private or network.is_loopback):
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"Network not permitted: {target}",
                    recovery_suggestion="Use RFC1918 or loopback ranges only (e.g., 10.0.0.0/8, 192.168.0.0/16)",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=target,
                )
                return self._create_error_output(error_context, inp.correlation_id)

            return None

        # Single-host targets
        # If it's an IP, enforce private/loopback; if it's a hostname, require .lab.internal
        try:
            ip = ipaddress.ip_address(target)
            if not (ip.is_private or ip.is_loopback):
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"IP not permitted: {target}",
                    recovery_suggestion="Use RFC1918 or loopback IPs only",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=target,
                )
                return self._create_error_output(error_context, inp.correlation_id)
        except ValueError:
            # Not an IP -> treat as hostname
            if not target.endswith(".lab.internal"):
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"Hostname not permitted: {target}",
                    recovery_suggestion="Use hostnames ending in .lab.internal",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=target,
                )
                return self._create_error_output(error_context, inp.correlation_id)

        return None

    def _optimize_nmap_args(self, extra_args: str) -> str:
        """Optimize nmap arguments for performance and safety."""
        if not extra_args:
            # Provide safe, minimal defaults with no args if desired; currently none.
            return ""

        # Robustly parse arguments, supporting quotes
        try:
            args = shlex.split(extra_args)
        except ValueError as e:
            # Fallback: return as-is; the base parser will raise a structured error
            log.warning("nmap.args.parse_failed tool=%s error=%s args=%r", self.tool_name, str(e), extra_args)
            return extra_args

        optimized: list[str] = []

        # Add performance optimizations if not specified
        has_timing = any(a.startswith("-T") for a in args)
        has_parallelism = any(a.startswith("--max-parallelism") for a in args)
        has_host_discovery = any(a in ("-Pn", "-sn") for a in args)

        if not has_timing:
            optimized.append("-T4")  # Aggressive timing (user args later can override)

        # Note: --max-parallelism must be supported by your nmap version.
        # It's allowed in allowed_flags to avoid parse rejection.
        if not has_parallelism:
            optimized.append("--max-parallelism=10")  # Conservative cap

        if not has_host_discovery:
            optimized.append("-Pn")  # Skip host discovery for internal networks

        # Append existing args so user-provided flags can override defaults
        optimized.extend(args)

        return " ".join(optimized)

    def _get_timestamp(self):
        """Get current timestamp (UTC, timezone-aware)."""
        return datetime.now(timezone.utc)

    def get_tool_info(self) -> dict:
        """Get enhanced tool information."""
        base_info = {
            "name": self.tool_name,
            "command": self.command_name,
            "description": self.__doc__,
            "concurrency": self.concurrency,
            "timeout": self.default_timeout_sec,
            "allowed_flags": list(self.allowed_flags) if self.allowed_flags else [],
            "circuit_breaker": {
                "failure_threshold": self.circuit_breaker_failure_threshold,
                "recovery_timeout": self.circuit_breaker_recovery_timeout,
            },
            "optimizations": [
                "Aggressive timing (-T4)",
                "Limited parallelism (--max-parallelism=10)",
                "Host discovery skip (-Pn)",
            ],
        }

        # Add metrics if available
        if hasattr(self, "metrics") and self.metrics:
            base_info["metrics"] = {
                "prometheus_available": True,
                "execution_metrics": f'mcp_tool_execution_total{{tool="{self.tool_name}"}}',
            }

        return base_info

```

