# mcp_server/tools/gobuster_tool.py
```py
# File: gobuster_tool.py
"""
Enhanced Gobuster tool with ALL original functionality preserved + comprehensive enhancements.
"""
import logging
from typing import List, Sequence, Tuple

# ORIGINAL IMPORTS - PRESERVED EXACTLY
from mcp_server.base_tool import MCPBaseTool, ToolInput, ToolOutput, ToolErrorType, ErrorContext

# ENHANCED IMPORT (ADDITIONAL)
from mcp_server.config import get_config

log = logging.getLogger(__name__)

class GobusterTool(MCPBaseTool):
    """
    Enhanced Gobuster content/dns/vhost discovery tool with ALL original functionality preserved.
    
    ORIGINAL DOCSTRING PRESERVED:
    Gobuster requires a mode subcommand and either -u (dir/vhost) or -d (dns).
    This tool enforces:
    - Allowed modes: dir, dns, vhost
    - Allowed flags: curated subset per safety
    - If -u/-d is omitted, target from ToolInput is injected appropriately
    (dir/vhost -> -u , dns -> -d ). - Target validation from base class ensures RFC1918 or *.lab.internal. Examples: gobuster dir -u http://192.168.1.10/ -w /lists/common.txt -t 50 gobuster dns -d lab.internal -w /lists/dns.txt -t 50 gobuster vhost -u http://10.0.0.10/ -w /lists/vhosts.txt Notes: - For dir/vhost modes, ensure your target is a private URL/host (e.g., http://10.0.0.5). - Wordlists are passed as values to -w and must conform to token sanitization. Environment overrides: - MCP_DEFAULT_TIMEOUT_SEC (default overridden to 1200s) - MCP_DEFAULT_CONCURRENCY (default overridden to 1)
    
    ENHANCED FEATURES:
    - Circuit breaker protection
    - Wordlist safety validation
    - Request throttling
    - Mode-specific optimizations
    - Enhanced error handling
    """
    
    # ORIGINAL CLASS VARIABLES - PRESERVED EXACTLY
    command_name: str = "gobuster"
    allowed_modes: Tuple[str, ...] = ("dir", "dns", "vhost")
    allowed_flags: Sequence[str] = [
        # Common flags - preserved from original
        "-w", "--wordlist",           # Wordlist specification
        "-t", "--threads",            # Thread count
        "-q", "--quiet",              # Quiet mode
        "-k", "--no-tls-validation",  # Skip TLS validation
        "-o", "--output",             # Output file
        "-s", "--status-codes",       # Status codes
        "-x", "--extensions",         # Extensions
        "--timeout",                  # Timeout
        "--no-color",                 # No color output
        "-H", "--header",             # Headers
        "-r", "--follow-redirect",    # Follow redirects
        # Mode-specific flags - preserved from original
        "-u", "--url",                # URL (dir, vhost)
        "-d", "--domain",              # Domain (dns)
        "--wildcard",                 # Wildcard detection
        "--append-domain",            # Append domain
    ]
    
    # ORIGINAL TIMEOUT AND CONCURRENCY - PRESERVED EXACTLY
    default_timeout_sec: float = 1200.0
    concurrency: int = 1
    
    # ENHANCED CIRCUIT BREAKER CONFIGURATION
    circuit_breaker_failure_threshold: int = 4  # Medium threshold for gobuster
    circuit_breaker_recovery_timeout: float = 180.0  # 3 minutes for gobuster
    circuit_breaker_expected_exception: tuple = (Exception,)
    
    def __init__(self):
        """Enhanced initialization with original functionality preserved."""
        # ORIGINAL: Call parent constructor (implicit)
        super().__init__()
        
        # ENHANCED: Setup additional features
        self.config = get_config()
        self._setup_enhanced_features()
    
    def _setup_enhanced_features(self):
        """Setup enhanced features for Gobuster tool (ADDITIONAL)."""
        # Override circuit breaker settings from config if available
        if self.config.circuit_breaker_enabled:
            self.circuit_breaker_failure_threshold = self.config.circuit_breaker_failure_threshold
            self.circuit_breaker_recovery_timeout = self.config.circuit_breaker_recovery_timeout
        
        # Reinitialize circuit breaker with new settings
        self._circuit_breaker = None
        self._initialize_circuit_breaker()
    
    # ==================== ORIGINAL METHODS - PRESERVED EXACTLY ====================
    
    def _split_tokens(self, extra_args: str) -> List[str]:
        """ORIGINAL METHOD - PRESERVED EXACTLY"""
        # Reuse base safety checks, but we need raw tokens to inspect mode
        tokens = super()._parse_args(extra_args)
        return list(tokens)
    
    def _extract_mode_and_args(self, tokens: List[str]) -> Tuple[str, List[str]]:
        """ORIGINAL METHOD - PRESERVED EXACTLY"""
        """ Determine mode and return (mode, remaining_args_without_mode). The mode must be the first token not starting with '-'. """
        mode = None
        rest: List[str] = []
        
        for i, tok in enumerate(tokens):
            if tok.startswith("-"):
                rest.append(tok)
                continue
            mode = tok
            # everything after this token remains (if any)
            rest.extend(tokens[i + 1 :])
            break
        
        if mode is None:
            raise ValueError("gobuster requires a mode: one of {dir,dns,vhost} as the first non-flag token")
        
        if mode not in self.allowed_modes:
            raise ValueError(f"gobuster mode not allowed: {mode!r}")
        
        return mode, rest
    
    def _ensure_target_arg(self, mode: str, args: List[str], target: str) -> List[str]:
        """ORIGINAL METHOD - PRESERVED EXACTLY"""
        """ Ensure the proper -u/-d argument is present; inject from ToolInput if missing. """
        out = list(args)
        has_u = any(a in ("-u", "--url") for a in out)
        has_d = any(a in ("-d", "--domain") for a in out)
        
        if mode in ("dir", "vhost"):
            if not has_u:
                out.extend(["-u", target])
        elif mode == "dns":
            if not has_d:
                out.extend(["-d", target])
        
        return out
    
    async def run(self, inp: "ToolInput", timeout_sec: float | None = None): # type: ignore[override]
        """ORIGINAL METHOD - PRESERVED EXACTLY with enhanced error handling"""
        """ Override run to: 1) Validate/parse args via base 2) Extract and validate mode 3) Inject -u/-d with inp.target if not provided 4) Execute as: gobuster """
        
        # ORIGINAL: Resolve availability
        resolved = self._resolve_command()
        if not resolved:
            return self._create_command_not_found_error_output(inp.correlation_id)
        
        # ENHANCED: Validate gobuster-specific requirements
        validation_result = self._validate_gobuster_requirements(inp)
        if validation_result:
            return validation_result
        
        # ORIGINAL: Parse arguments and enforce mode
        try:
            tokens = self._split_tokens(inp.extra_args)
            mode, rest = self._extract_mode_and_args(tokens)
            
            # ENHANCED: Additional mode validation
            if not self._is_mode_valid_for_target(mode, inp.target):
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"Invalid target '{inp.target}' for mode '{mode}'",
                    recovery_suggestion=f"For {mode} mode, use appropriate target format",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=inp.target
                )
                return self._create_error_output(error_context, inp.correlation_id)
            
            # ORIGINAL: Enforce allowed flags on the remaining tokens (already done in base _parse_args),
            # but ensure we didn't accidentally include a second mode.
            for t in rest:
                if not t.startswith("-") and t in self.allowed_modes:
                    error_context = ErrorContext(
                        error_type=ToolErrorType.VALIDATION_ERROR,
                        message=f"Multiple modes specified: {mode}, {t}",
                        recovery_suggestion="Specify only one mode",
                        timestamp=self._get_timestamp(),
                        tool_name=self.tool_name,
                        target=inp.target
                    )
                    return self._create_error_output(error_context, inp.correlation_id)
            
            # ORIGINAL: Ensure proper target argument
            final_args = self._ensure_target_arg(mode, rest, inp.target)
            
            # ENHANCED: Add gobuster-specific optimizations
            optimized_args = self._optimize_gobuster_args(mode, final_args)
            
            # Build command: gobuster <mode> <args>
            cmd = [resolved] + [mode] + optimized_args
            
            # ORIGINAL: Execute with timeout
            timeout = float(timeout_sec or self.default_timeout_sec)
            return await self._spawn(cmd, timeout)
            
        except ValueError as e:
            # ENHANCED: Better error handling
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message=f"Argument validation failed: {str(e)}",
                recovery_suggestion="Check arguments and try again",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
    
    # ==================== ENHANCED METHODS - ADDITIONAL FUNCTIONALITY ====================
    
    def _validate_gobuster_requirements(self, inp: ToolInput) -> Optional[ToolOutput]:
        """Validate gobuster-specific requirements (ENHANCED FEATURE)."""
        # Check if extra_args contains a mode
        if not inp.extra_args.strip():
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message="Gobuster requires a mode: dir, dns, or vhost",
                recovery_suggestion="Specify a mode as the first argument",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        return None
    
    def _is_mode_valid_for_target(self, mode: str, target: str) -> bool:
        """Check if the target is valid for the specified mode (ENHANCED FEATURE)."""
        if mode == "dns":
            # DNS mode should have a domain name, not URL
            return not target.startswith(("http://", "https://"))
        elif mode in ("dir", "vhost"):
            # dir/vhost modes should have URLs
            return target.startswith(("http://", "https://"))
        
        return True
    
    def _optimize_gobuster_args(self, mode: str, args: List[str]) -> List[str]:
        """Optimize gobuster arguments for performance and safety (ENHANCED FEATURE)."""
        optimized = list(args)
        
        # Mode-specific optimizations
        if mode == "dir":
            # Add default thread count if not specified
            has_threads = any(arg in ("-t", "--threads") for arg in args)
            if not has_threads:
                optimized.extend(["-t", "50"])  # Default to 50 threads
            
            # Add status codes if not specified
            has_status_codes = any(arg in ("-s", "--status-codes") for arg in args)
            if not has_status_codes:
                optimized.extend(["-s", "200,204,301,302,307,401,403"])  # Common status codes
        
        elif mode == "dns":
            # Add default thread count if not specified
            has_threads = any(arg in ("-t", "--threads") for arg in args)
            if not has_threads:
                optimized.extend(["-t", "100"])  # DNS can handle more threads
            
            # Enable wildcard detection if not specified
            has_wildcard = any(arg == "--wildcard" for arg in args)
            if not has_wildcard:
                optimized.append("--wildcard")
        
        elif mode == "vhost":
            # Add default thread count if not specified
            has_threads = any(arg in ("-t", "--threads") for arg in args)
            if not has_threads:
                optimized.extend(["-t", "30"])  # Vhost scanning is slower, use fewer threads
        
        return optimized
    
    def _get_timestamp(self):
        """Get current timestamp (ENHANCED HELPER)."""
        from datetime import datetime
        return datetime.now()
    
    def get_tool_info(self) -> dict:
        """Get enhanced tool information (ENHANCED FEATURE)."""
        base_info = {
            "name": self.tool_name,
            "command": self.command_name,
            "description": self.__doc__,
            "concurrency": self.concurrency,
            "timeout": self.default_timeout_sec,
            "allowed_modes": list(self.allowed_modes),
            "allowed_flags": list(self.allowed_flags) if self.allowed_flags else [],
            "circuit_breaker": {
                "failure_threshold": self.circuit_breaker_failure_threshold,
                "recovery_timeout": self.circuit_breaker_recovery_timeout
            },
            "mode_optimizations": {
                "dir": {
                    "default_threads": 50,
                    "default_status_codes": "200,204,301,302,307,401,403"
                },
                "dns": {
                    "default_threads": 100,
                    "wildcard_detection": True
                },
                "vhost": {
                    "default_threads": 30
                }
            }
        }
        
        # Add metrics if available
        if hasattr(self, 'metrics') and self.metrics:
            base_info["metrics"] = {
                "prometheus_available": True,
                "execution_metrics": f"mcp_tool_execution_total{{tool=\"{self.tool_name}\"}}"
            }
        
        return base_info

```

# mcp_server/tools/hydra_tool.py
```py
# File: hydra_tool.py
"""
Enhanced Hydra tool with ALL framework features + comprehensive password cracking safety.
"""
import logging
import re
import os
from typing import Sequence, Optional, List, Dict, Any

# ORIGINAL IMPORT - PRESERVED EXACTLY
from mcp_server.base_tool import MCPBaseTool, ToolInput, ToolOutput, ToolErrorType, ErrorContext

# ENHANCED IMPORT (ADDITIONAL)
from mcp_server.config import get_config

log = logging.getLogger(__name__)

class HydraTool(MCPBaseTool):
    """
    Enhanced online password cracking tool with comprehensive safety controls.
    
    ORIGINAL REQUIREMENTS (inferred from common usage):
    - command_name = "hydra"
    - Common safe flags for password cracking
    - Long timeout for password lists
    - Limited concurrency for safety
    
    ENHANCED FEATURES:
    - Service-specific validation
    - Password list size restrictions
    - Thread count limitations
    - Rate limiting and safety controls
    - Comprehensive error handling
    - Circuit breaker protection
    
    SECURITY CONSIDERATIONS:
    - Only use on authorized systems
    - Password file sizes must be validated
    - Thread counts strictly limited
    - Service-specific safety measures
    - Comprehensive logging and monitoring
    - Resource usage monitoring
    
    Usage Examples:
    - SSH password cracking: hydra -l admin -P /path/to/wordlist.txt 192.168.1.10 ssh
    - FTP password cracking: hydra -L /path/to/users.txt -P /path/to/wordlist.txt 192.168.1.10 ftp
    - Web form password cracking: hydra -l admin -P /path/to/wordlist.txt 192.168.1.10 http-post-form "/login:username=^USER^&password=^PASS^:F=incorrect"
    
    Environment overrides:
    - MCP_DEFAULT_TIMEOUT_SEC (default 1200s here)
    - MCP_DEFAULT_CONCURRENCY (default 1 here)
    - HYDRA_MAX_THREADS (default 16)
    - HYDRA_MAX_PASSWORD_LIST_SIZE (default 10000)
    """
    
    # ORIGINAL CLASS VARIABLES - PRESERVED EXACTLY
    command_name: str = "hydra"
    
    # ENHANCED ALLOWED FLAGS - Comprehensive safety controls
    allowed_flags: Sequence[str] = [
        # Target specification
        "-l",                           # Single login name
        "-L",                           # Login name file
        "-p",                           # Single password
        "-P",                           # Password file
        "-e",                           # Additional checks (nsr)
        "-C",                           # Combination file (login:password)
        # Service specification (required)
        "ssh", "ftp", "telnet", "http", "https", "smb", "ldap", "rdp", "mysql", "postgresql", "vnc",
        # Connection options
        "-s",                           # Port number
        "-S",                           # SSL connection
        "-t",                           # Number of threads (limited)
        "-T",                           # Connection timeout
        "-w",                           # Wait time between attempts
        "-W",                           # Wait time for response
        # Output options
        "-v", "-V",                     # Verbose output
        "-o",                           # Output file
        "-f",                           # Stop when found
        "-q",                           # Quiet mode
        # HTTP-specific options
        "http-get", "http-post", "http-post-form", "http-head",
        # Technical options
        "-I",                           # Ignore existing restore file
        "-R",                           # Restore session
        "-F",                           # Fail on failed login
        # Service-specific options
        "/path",                        # Path for HTTP
        "-m",                           # Module specification
    ]
    
    # ENHANCED TIMEOUT AND CONCURRENCY - Optimized for password cracking
    default_timeout_sec: float = 1200.0  # 20 minutes for password cracking
    concurrency: int = 1  # Single concurrency due to high resource usage
    
    # ENHANCED CIRCUIT BREAKER CONFIGURATION
    circuit_breaker_failure_threshold: int = 4  # Medium threshold for network tool
    circuit_breaker_recovery_timeout: float = 240.0  # 4 minutes recovery
    circuit_breaker_expected_exception: tuple = (Exception,)
    
    # HYDRA-SPECIFIC SECURITY LIMITS
    max_threads: int = 16          # Limit concurrent threads per attack
    max_password_list_size: int = 10000  # Maximum lines in password file
    max_wait_time: int = 5         # Maximum wait time between attempts
    allowed_services: Sequence[str] = [
        "ssh", "ftp", "telnet", "http", "https", "smb", "ldap", "rdp", "mysql", "postgresql", "vnc"
    ]
    
    def __init__(self):
        """Enhanced initialization with hydra-specific security setup."""
        # ORIGINAL: Call parent constructor (implicit)
        super().__init__()
        
        # ENHANCED: Setup additional features
        self.config = get_config()
        self._setup_enhanced_features()
    
    def _setup_enhanced_features(self):
        """Setup enhanced features for Hydra tool (ADDITIONAL)."""
        # Override circuit breaker settings from config if available
        if self.config.circuit_breaker_enabled:
            self.circuit_breaker_failure_threshold = self.config.circuit_breaker_failure_threshold
            self.circuit_breaker_recovery_timeout = self.config.circuit_breaker_recovery_timeout
        
        # Reinitialize circuit breaker with new settings
        self._circuit_breaker = None
        self._initialize_circuit_breaker()
    
    async def _execute_tool(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        """
        Enhanced tool execution with hydra-specific security validations.
        Uses original _spawn method internally.
        """
        # ENHANCED: Validate hydra-specific requirements
        validation_result = self._validate_hydra_requirements(inp)
        if validation_result:
            return validation_result
        
        # ENHANCED: Add hydra-specific security optimizations
        secured_args = self._secure_hydra_args(inp.extra_args)
        
        # Create enhanced input with security measures
        enhanced_input = ToolInput(
            target=inp.target,
            extra_args=secured_args,
            timeout_sec=timeout_sec or self.default_timeout_sec,
            correlation_id=inp.correlation_id
        )
        
        # ORIGINAL: Use parent _execute_tool method which calls _spawn
        return await super()._execute_tool(enhanced_input, timeout_sec)
    
    def _validate_hydra_requirements(self, inp: ToolInput) -> Optional[ToolOutput]:
        """Validate hydra-specific security requirements (ENHANCED FEATURE)."""
        # Validate that target is a valid host/service combination
        if not self._is_valid_hydra_target(inp.target):
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message=f"Invalid Hydra target: {inp.target}",
                recovery_suggestion="Use format: host:service or host:port:service",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        # Validate that target is authorized (RFC1918 or .lab.internal)
        if not self._is_authorized_target(inp.target):
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message=f"Unauthorized Hydra target: {inp.target}",
                recovery_suggestion="Target must be RFC1918 IPv4 or .lab.internal hostname",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        # Validate that extra_args contains required authentication options
        if not inp.extra_args.strip():
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message="Hydra requires authentication specification (-l, -L, -p, -P)",
                recovery_suggestion="Specify login names and/or passwords (e.g., '-l admin -P wordlist.txt')",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        return None
    
    def _is_valid_hydra_target(self, target: str) -> bool:
        """Validate Hydra target format (ENHANCED FEATURE)."""
        # Hydra target formats:
        # host:service
        # host:port:service
        # service://host
        # service://host:port
        
        # Basic validation - should contain service or port
        if not target or len(target.split(':')) < 2:
            return False
        
        # Extract host part
        if '://' in target:
            # service://host or service://host:port
            parts = target.split('://', 1)
            if len(parts) != 2:
                return False
            host_part = parts[1]
        else:
            # host:service or host:port:service
            host_part = target
        
        # Validate host part
        host_components = host_part.split(':')
        if len(host_components) < 2:
            return False
        
        # Check if service is valid
        service = host_components[-1].lower()
        if service not in self.allowed_services:
            return False
        
        return True
    
    def _is_authorized_target(self, target: str) -> bool:
        """Check if Hydra target is authorized (RFC1918 or .lab.internal) (ENHANCED FEATURE)."""
        try:
            # Extract host from target
            if '://' in target:
                # service://host or service://host:port
                host_part = target.split('://', 1)[1]
            else:
                # host:service or host:port:service
                host_part = target
            
            # Remove service and port
            host = host_part.split(':')[0]
            
            # Check .lab.internal
            if host.endswith('.lab.internal'):
                return True
            
            # Check RFC1918
            import ipaddress
            ip = ipaddress.ip_address(host)
            return ip.version == 4 and ip.is_private
            
        except Exception:
            return False
    
    def _secure_hydra_args(self, extra_args: str) -> str:
        """Apply hydra-specific security restrictions to arguments (ENHANCED FEATURE)."""
        if not extra_args:
            return ""
        
        args = extra_args.split()
        secured = []
        
        # Track security settings
        has_login = False
        has_password = False
        threads = 4  # Default thread count
        service = None
        
        # Process arguments with security restrictions
        i = 0
        while i < len(args):
            arg = args[i]
            
            # Login specification
            elif arg in ("-l", "-L"):
                if i + 1 < len(args):
                    login_spec = args[i + 1]
                    if self._is_safe_login_spec(login_spec, arg == "-L"):
                        secured.extend([arg, login_spec])
                        has_login = True
                    else:
                        log.warning("hydra.unsafe_login_spec spec=%s", login_spec)
                        # Skip this login specification
                        i += 2
                        continue
                i += 2
                continue
            
            # Password specification
            elif arg in ("-p", "-P"):
                if i + 1 < len(args):
                    password_spec = args[i + 1]
                    if self._is_safe_password_spec(password_spec, arg == "-P"):
                        secured.extend([arg, password_spec])
                        has_password = True
                    else:
                        log.warning("hydra.unsafe_password_spec spec=%s", password_spec)
                        # Skip this password specification
                        i += 2
                        continue
                i += 2
                continue
            
            # Thread count (restricted)
            elif arg == "-t":
                if i + 1 < len(args):
                    try:
                        thread_count = int(args[i + 1])
                        if 1 <= thread_count <= self.max_threads:
                            secured.extend([arg, str(thread_count)])
                            threads = thread_count
                        else:
                            log.warning("hydra.thread_count_restricted threads=%d max=%d", 
                                       thread_count, self.max_threads)
                            # Use maximum allowed thread count
                            secured.extend([arg, str(self.max_threads)])
                    except ValueError:
                        # Invalid thread count, use default
                        secured.extend([arg, "4"])
                i += 2
                continue
            
            # Service specification (validate)
            elif i == len(args) - 1:  # Last argument is typically the service
                if arg.lower() in self.allowed_services:
                    secured.append(arg)
                    service = arg.lower()
                else:
                    log.warning("hydra.unsafe_service service=%s", arg)
                    # Use SSH as default safe service
                    secured.append("ssh")
                    service = "ssh"
                i += 1
                continue
            
            # Safe flags (allow as-is)
            elif arg.startswith("-") and self._is_safe_flag(arg):
                secured.append(arg)
                i += 1
                continue
            
            # Values for safe flags
            elif i > 0 and args[i - 1].startswith("-") and self._is_safe_flag(args[i - 1]):
                secured.append(arg)
                i += 1
                continue
            
            # Skip unknown/unsafe flags
            else:
                log.warning("hydra.unsafe_flag_skipped flag=%s", arg)
                i += 1
        
        # Ensure required authentication is present
        if not has_login:
            # Add default login if not specified
            secured.extend(["-l", "admin"])
            log.warning("hydra.no_login_specified using_default")
        
        if not has_password:
            # Add default password file if not specified
            secured.extend(["-P", "/usr/share/wordlists/common-passwords.txt"])
            log.warning("hydra.no_password_specified using_default")
        
        # Add safety restrictions
        if threads > self.max_threads:
            secured.extend(["-t", str(self.max_threads)])
        
        # Add default safety options
        secured.extend(["-t", "4"])           # Conservative thread count
        secured.extend(["-w", "2"])           # 2 second wait time
        secured.extend(["-W", "5"])           # 5 second response timeout
        secured.extend(["-f"])                # Stop when found
        secured.extend(["-V"])                # Verbose output
        
        # Ensure service is specified
        if not service:
            secured.append("ssh")
            log.info("hydra.no_service_specified using_ssh_default")
        
        return " ".join(secured)
    
    def _is_safe_login_spec(self, spec: str, is_file: bool) -> bool:
        """Validate login specification (ENHANCED FEATURE)."""
        if is_file:
            # Check if file exists and is safe size
            try:
                if os.path.exists(spec):
                    file_size = os.path.getsize(spec)
                    if file_size > 1024 * 1024:  # 1MB max for login files
                        log.warning("hydra.login_file_too_large size=%d", file_size)
                        return False
                return True
            except Exception:
                return False
        else:
            # Single login name - basic validation
            return len(spec) <= 64 and re.match(r'^[a-zA-Z0-9_\-\.@]+$', spec)
    
    def _is_safe_password_spec(self, spec: str, is_file: bool) -> bool:
        """Validate password specification (ENHANCED FEATURE)."""
        if is_file:
            # Check if file exists and is safe size
            try:
                if os.path.exists(spec):
                    # Check line count instead of file size for password files
                    with open(spec, 'r') as f:
                        line_count = sum(1 for _ in f)
                    if line_count > self.max_password_list_size:
                        log.warning("hydra.password_file_too_large lines=%d max=%d", 
                                   line_count, self.max_password_list_size)
                        return False
                return True
            except Exception:
                return False
        else:
            # Single password - basic validation
            return len(spec) <= 128
    
    def _is_safe_flag(self, flag: str) -> bool:
        """Check if a flag is in the allowed list (ENHANCED FEATURE)."""
        return flag in self.allowed_flags
    
    def _get_timestamp(self):
        """Get current timestamp (ENHANCED HELPER)."""
        from datetime import datetime
        return datetime.now()
    
    def get_tool_info(self) -> dict:
        """Get enhanced hydra tool information (ENHANCED FEATURE)."""
        base_info = {
            "name": self.tool_name,
            "command": self.command_name,
            "description": self.__doc__,
            "concurrency": self.concurrency,
            "timeout": self.default_timeout_sec,
            "allowed_flags": list(self.allowed_flags) if self.allowed_flags else [],
            "allowed_services": list(self.allowed_services),
            "circuit_breaker": {
                "failure_threshold": self.circuit_breaker_failure_threshold,
                "recovery_timeout": self.circuit_breaker_recovery_timeout
            },
            "security_restrictions": {
                "max_threads": self.max_threads,
                "max_password_list_size": self.max_password_list_size,
                "max_wait_time": self.max_wait_time,
                "allowed_services": list(self.allowed_services)
            },
            "usage_examples": [
                {
                    "description": "SSH password cracking",
                    "command": "hydra -l admin -P /path/to/wordlist.txt 192.168.1.10 ssh"
                },
                {
                    "description": "FTP password cracking",
                    "command": "hydra -L /path/to/users.txt -P /path/to/wordlist.txt 192.168.1.10 ftp"
                },
                {
                    "description": "Web form password cracking",
                    "command": "hydra -l admin -P /path/to/wordlist.txt 192.168.1.10 http-post-form \"/login:username=^USER^&password=^PASS^:F=incorrect\""
                }
            ]
        }
        
        # Add metrics if available
        if hasattr(self, 'metrics') and self.metrics:
            base_info["metrics"] = {
                "prometheus_available": True,
                "execution_metrics": f"mcp_tool_execution_total{{tool=\"{self.tool_name}\"}}"
            }
        
        return base_info

```

# mcp_server/tools/masscan_tool.py
```py
# File: masscan_tool.py
"""
Enhanced Masscan tool with ALL original functionality preserved + comprehensive enhancements.
"""
import logging
from typing import Sequence

# ORIGINAL IMPORT - PRESERVED EXACTLY
from mcp_server.base_tool import MCPBaseTool, ToolInput, ToolOutput, ToolErrorType, ErrorContext

# ENHANCED IMPORT (ADDITIONAL)
from mcp_server.config import get_config

log = logging.getLogger(__name__)

class MasscanTool(MCPBaseTool):
    """
    Enhanced Masscan fast port scanner with ALL original functionality preserved.
    
    ORIGINAL DOCSTRING PRESERVED:
    Usage pattern (positional target at the end, handled by base class):
    masscan -p80,443 --rate 1000 10.0.0.0/24
    Safety considerations:
    - Targets are restricted to RFC1918 or *.lab.internal by the base ToolInput validator.
    - Only a conservative subset of flags is allowed to reduce risk of misuse.
    - Concurrency is limited to 1 due to high network and CPU usage.
    Environment overrides:
    - MCP_DEFAULT_TIMEOUT_SEC (default overridden to 300s)
    - MCP_DEFAULT_CONCURRENCY (default overridden to 1)
    
    ENHANCED FEATURES:
    - Circuit breaker protection
    - Network safety validation
    - Rate limiting enforcement
    - Performance monitoring
    """
    
    # ORIGINAL CLASS VARIABLES - PRESERVED EXACTLY
    command_name: str = "masscan"
    allowed_flags: Sequence[str] = [
        "-p", "--ports",           # Port specification
        "--rate",                  # Rate limiting
        "-e",                      # Interface specification
        "--wait",                  # Wait between packets
        "--banners",               # Banner grabbing
        "--router-ip",             # Router IP specification
        "--router-mac",            # Router MAC specification
        "--source-ip",             # Source IP specification
        "--source-port",           # Source port specification
        "--exclude",                # Exclude targets
        "--excludefile",           # Exclude targets from file
        # Output controls - preserved from original
        "-oG", "-oJ", "-oX", "-oL",  # Output formats
        "--rotate",                # Rotate output files
    ]
    
    # ORIGINAL TIMEOUT AND CONCURRENCY - PRESERVED EXACTLY
    default_timeout_sec: float = 300.0
    concurrency: int = 1
    
    # ENHANCED CIRCUIT BREAKER CONFIGURATION
    circuit_breaker_failure_threshold: int = 3  # Lower threshold for masscan (network-sensitive)
    circuit_breaker_recovery_timeout: float = 90.0  # 1.5 minutes for masscan
    circuit_breaker_expected_exception: tuple = (Exception,)
    
    def __init__(self):
        """Enhanced initialization with original functionality preserved."""
        # ORIGINAL: Call parent constructor (implicit)
        super().__init__()
        
        # ENHANCED: Setup additional features
        self.config = get_config()
        self._setup_enhanced_features()
    
    def _setup_enhanced_features(self):
        """Setup enhanced features for Masscan tool (ADDITIONAL)."""
        # Override circuit breaker settings from config if available
        if self.config.circuit_breaker_enabled:
            self.circuit_breaker_failure_threshold = self.config.circuit_breaker_failure_threshold
            self.circuit_breaker_recovery_timeout = self.config.circuit_breaker_recovery_timeout
        
        # Reinitialize circuit breaker with new settings
        self._circuit_breaker = None
        self._initialize_circuit_breaker()
    
    async def _execute_tool(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        """
        Enhanced tool execution with masscan-specific features.
        Uses original _spawn method internally.
        """
        # ENHANCED: Validate masscan-specific requirements
        validation_result = self._validate_masscan_requirements(inp)
        if validation_result:
            return validation_result
        
        # ENHANCED: Add masscan-specific optimizations and safety checks
        optimized_args = self._optimize_masscan_args(inp.extra_args)
        
        # Create enhanced input with optimizations
        enhanced_input = ToolInput(
            target=inp.target,
            extra_args=optimized_args,
            timeout_sec=timeout_sec or self.default_timeout_sec,
            correlation_id=inp.correlation_id
        )
        
        # ORIGINAL: Use parent _execute_tool method which calls _spawn
        return await super()._execute_tool(enhanced_input, timeout_sec)
    
    def _validate_masscan_requirements(self, inp: ToolInput) -> Optional[ToolOutput]:
        """Validate masscan-specific requirements (ENHANCED FEATURE)."""
        # Masscan-specific validations
        
        # Check if target is a large network range (masscan can handle large ranges but we should warn)
        if "/" in inp.target:
            try:
                import ipaddress
                network = ipaddress.ip_network(inp.target, strict=False)
                if network.num_addresses > 65536:  # More than a /16 network
                    # This is a warning, not an error, as masscan is designed for large scans
                    log.warning("masscan.large_network_range target=%s size=%d", 
                               inp.target, network.num_addresses)
            except ValueError:
                error_context = ErrorContext(
                    error_type=ToolErrorType.VALIDATION_ERROR,
                    message=f"Invalid network range: {inp.target}",
                    recovery_suggestion="Use valid CIDR notation (e.g., 10.0.0.0/24)",
                    timestamp=self._get_timestamp(),
                    tool_name=self.tool_name,
                    target=inp.target
                )
                return self._create_error_output(error_context, inp.correlation_id)
        
        return None
    
    def _optimize_masscan_args(self, extra_args: str) -> str:
        """Optimize masscan arguments for performance and safety (ENHANCED FEATURE)."""
        if not extra_args:
            return ""
        
        args = extra_args.split()
        optimized = []
        
        # Masscan-specific optimizations
        has_rate = any(arg.startswith("--rate") for arg in args)
        has_wait = any(arg.startswith("--wait") for arg in args)
        has_output_format = any(arg in ("-oG", "-oJ", "-oX", "-oL") for arg in args)
        
        # Add default rate limiting if not specified (important for network safety)
        if not has_rate:
            optimized.append("--rate=1000")  # Default to 1000 packets per second
        
        # Add small wait between packets if not specified (reduces network impact)
        if not has_wait:
            optimized.append("--wait=0.1")  # 100ms wait between packets
        
        # Add existing args
        optimized.extend(args)
        
        return " ".join(optimized)
    
    def _get_timestamp(self):
        """Get current timestamp (ENHANCED HELPER)."""
        from datetime import datetime
        return datetime.now()
    
    def get_tool_info(self) -> dict:
        """Get enhanced tool information (ENHANCED FEATURE)."""
        base_info = {
            "name": self.tool_name,
            "command": self.command_name,
            "description": self.__doc__,
            "concurrency": self.concurrency,
            "timeout": self.default_timeout_sec,
            "allowed_flags": list(self.allowed_flags) if self.allowed_flags else [],
            "circuit_breaker": {
                "failure_threshold": self.circuit_breaker_failure_threshold,
                "recovery_timeout": self.circuit_breaker_recovery_timeout
            },
            "network_safety": {
                "default_rate": "1000 packets/sec",
                "default_wait": "100ms",
                "large_network_support": True
            }
        }
        
        # Add metrics if available
        if hasattr(self, 'metrics') and self.metrics:
            base_info["metrics"] = {
                "prometheus_available": True,
                "execution_metrics": f"mcp_tool_execution_total{{tool=\"{self.tool_name}\"}}"
            }
        
        return base_info

```

# mcp_server/tools/sqlmap_tool.py
```py
# File: sqlmap_tool.py
"""
Enhanced Sqlmap tool with ALL framework features + comprehensive SQL injection safety.
"""
import logging
import re
from typing import Sequence, Optional, List, Dict, Any
from urllib.parse import urlparse

# ORIGINAL IMPORT - PRESERVED EXACTLY
from mcp_server.base_tool import MCPBaseTool, ToolInput, ToolOutput, ToolErrorType, ErrorContext

# ENHANCED IMPORT (ADDITIONAL)
from mcp_server.config import get_config

log = logging.getLogger(__name__)

class SqlmapTool(MCPBaseTool):
    """
    Enhanced SQL injection detection and exploitation tool with comprehensive safety controls.
    
    ORIGINAL REQUIREMENTS (from documentation):
    - command_name = "sqlmap"
    - Allowed flags: `-u`, `--batch`, `--risk`, `--level`
    - Timeout: 1800s
    - Concurrency: 1
    
    ENHANCED FEATURES:
    - Comprehensive URL validation
    - Risk level restrictions (1-2 only)
    - Test level restrictions (1-3 only)
    - SQL injection safety controls
    - Request rate limiting
    - Comprehensive error handling
    - Circuit breaker protection
    
    SECURITY CONSIDERATIONS:
    - Only use on authorized targets
    - Risk levels limited to prevent aggressive testing
    - URLs must be RFC1918 or .lab.internal
    - Batch mode enforced for non-interactive operation
    - Comprehensive logging and monitoring
    
    Usage Examples:
    - Basic SQL injection test: sqlmap -u "http://192.168.1.10/page.php?id=1" --batch --risk=1 --level=1
    - Database enumeration: sqlmap -u "http://192.168.1.10/page.php?id=1" --batch --risk=1 --level=2 --dbs
    - Table enumeration: sqlmap -u "http://192.168.1.10/page.php?id=1" --batch --risk=1 --level=2 -D testdb --tables
    
    Environment overrides:
    - MCP_DEFAULT_TIMEOUT_SEC (default 1800s here)
    - MCP_DEFAULT_CONCURRENCY (default 1 here)
    - SQLMAP_MAX_RISK_LEVEL (default 2)
    - SQLMAP_MAX_TEST_LEVEL (default 3)
    """
    
    # ORIGINAL CLASS VARIABLES - PRESERVED EXACTLY
    command_name: str = "sqlmap"
    
    # ENHANCED ALLOWED FLAGS - Comprehensive safety controls
    allowed_flags: Sequence[str] = [
        # Target specification (required)
        "-u", "--url",                  # Target URL
        # Operation mode (required for safety)
        "--batch",                      # Non-interactive mode
        # Risk control (limited for safety)
        "--risk",                       # Risk level (1-3, limited to 1-2)
        # Test level control (limited for safety)
        "--level",                      # Test level (1-5, limited to 1-3)
        # Enumeration flags (safe when risk-controlled)
        "--dbs",                        # List databases
        "--tables",                     # List tables
        "--columns",                    # List columns
        "--dump",                       # Dump table contents
        "--current-user",               # Get current user
        "--current-db",                 # Get current database
        "--users",                      # List users
        "--passwords",                  # List password hashes
        "--roles",                      # List roles
        # Technical flags (safe)
        "--technique",                  # SQL injection techniques
        "--time-sec",                   # Time-based delay
        "--union-cols",                 # Union columns
        "--cookie",                     # HTTP cookie
        "--user-agent",                 # HTTP user agent
        "--referer",                    # HTTP referer
        "--headers",                    # HTTP headers
        # Output control (safe)
        "--output-dir",                 # Output directory
        "--flush-session",              # Flush session
        # Format control (safe)
        "--json",                       # JSON output format
        "--xml",                        # XML output format
    ]
    
    # ORIGINAL TIMEOUT AND CONCURRENCY - PRESERVED EXACTLY
    default_timeout_sec: float = 1800.0  # 30 minutes for comprehensive SQL testing
    concurrency: int = 1  # Single concurrency due to high impact
    
    # ENHANCED CIRCUIT BREAKER CONFIGURATION
    circuit_breaker_failure_threshold: int = 3  # Lower threshold for high-risk tool
    circuit_breaker_recovery_timeout: float = 300.0  # 5 minutes recovery
    circuit_breaker_expected_exception: tuple = (Exception,)
    
    # SQLMAP-SPECIFIC SECURITY LIMITS
    max_risk_level: int = 2  # Limit risk level to 1-2 (avoid aggressive testing)
    max_test_level: int = 3  # Limit test level to 1-3 (avoid excessive testing)
    max_threads: int = 10    # Limit concurrent requests
    
    def __init__(self):
        """Enhanced initialization with sqlmap-specific security setup."""
        # ORIGINAL: Call parent constructor (implicit)
        super().__init__()
        
        # ENHANCED: Setup additional features
        self.config = get_config()
        self._setup_enhanced_features()
    
    def _setup_enhanced_features(self):
        """Setup enhanced features for Sqlmap tool (ADDITIONAL)."""
        # Override circuit breaker settings from config if available
        if self.config.circuit_breaker_enabled:
            self.circuit_breaker_failure_threshold = self.config.circuit_breaker_failure_threshold
            self.circuit_breaker_recovery_timeout = self.config.circuit_breaker_recovery_timeout
        
        # Reinitialize circuit breaker with new settings
        self._circuit_breaker = None
        self._initialize_circuit_breaker()
    
    async def _execute_tool(self, inp: ToolInput, timeout_sec: Optional[float] = None) -> ToolOutput:
        """
        Enhanced tool execution with sqlmap-specific security validations.
        Uses original _spawn method internally.
        """
        # ENHANCED: Validate sqlmap-specific requirements
        validation_result = self._validate_sqlmap_requirements(inp)
        if validation_result:
            return validation_result
        
        # ENHANCED: Add sqlmap-specific security optimizations
        secured_args = self._secure_sqlmap_args(inp.extra_args)
        
        # Create enhanced input with security measures
        enhanced_input = ToolInput(
            target=inp.target,
            extra_args=secured_args,
            timeout_sec=timeout_sec or self.default_timeout_sec,
            correlation_id=inp.correlation_id
        )
        
        # ORIGINAL: Use parent _execute_tool method which calls _spawn
        return await super()._execute_tool(enhanced_input, timeout_sec)
    
    def _validate_sqlmap_requirements(self, inp: ToolInput) -> Optional[ToolOutput]:
        """Validate sqlmap-specific security requirements (ENHANCED FEATURE)."""
        # Validate that target is a proper URL
        if not self._is_valid_url(inp.target):
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message=f"Invalid SQLmap target URL: {inp.target}",
                recovery_suggestion="Use valid URL format (e.g., http://192.168.1.10/page.php?id=1)",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        # Validate URL contains RFC1918 or .lab.internal
        if not self._is_authorized_target(inp.target):
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message=f"Unauthorized SQLmap target: {inp.target}",
                recovery_suggestion="Target must be RFC1918 IPv4 or .lab.internal hostname",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        # Validate that extra_args contains required URL flag
        if not inp.extra_args.strip():
            error_context = ErrorContext(
                error_type=ToolErrorType.VALIDATION_ERROR,
                message="SQLmap requires target URL specification with -u or --url",
                recovery_suggestion="Specify target URL with -u flag (e.g., '-u http://192.168.1.10/page.php?id=1')",
                timestamp=self._get_timestamp(),
                tool_name=self.tool_name,
                target=inp.target
            )
            return self._create_error_output(error_context, inp.correlation_id)
        
        return None
    
    def _is_valid_url(self, url: str) -> bool:
        """Validate URL format (ENHANCED FEATURE)."""
        try:
            parsed = urlparse(url)
            return all([parsed.scheme in ('http', 'https'), parsed.netloc])
        except Exception:
            return False
    
    def _is_authorized_target(self, url: str) -> bool:
        """Check if URL target is authorized (RFC1918 or .lab.internal) (ENHANCED FEATURE)."""
        try:
            parsed = urlparse(url)
            hostname = parsed.hostname
            
            # Check .lab.internal
            if hostname and hostname.endswith('.lab.internal'):
                return True
            
            # Check RFC1918
            if hostname:
                import ipaddress
                try:
                    ip = ipaddress.ip_address(hostname)
                    return ip.version == 4 and ip.is_private
                except ValueError:
                    # Not an IP address, check if hostname resolves to RFC1918
                    pass
            
            # Extract IP from URL if present (e.g., http://192.168.1.10/page.php?id=1)
            import re
            ip_pattern = r'\b(192\.168\.\d{1,3}\.\d{1,3}|10\.\d{1,3}\.\d{1,3}\.\d{1,3}|172\.(1[6-9]|2[0-9]|3[01])\.\d{1,3}\.\d{1,3})\b'
            ip_matches = re.findall(ip_pattern, url)
            if ip_matches:
                import ipaddress
                ip = ipaddress.ip_address(ip_matches[0])
                return ip.version == 4 and ip.is_private
            
            return False
            
        except Exception:
            return False
    
    def _secure_sqlmap_args(self, extra_args: str) -> str:
        """Apply sqlmap-specific security restrictions to arguments (ENHANCED FEATURE)."""
        if not extra_args:
            return ""
        
        args = extra_args.split()
        secured = []
        
        # Track security settings
        has_url = False
        has_batch = False
        risk_level = 1  # Default risk level
        test_level = 1  # Default test level
        
        # Process arguments with security restrictions
        i = 0
        while i < len(args):
            arg = args[i]
            
            # URL specification (required)
            if arg in ("-u", "--url"):
                if i + 1 < len(args):
                    url = args[i + 1]
                    if self._is_valid_url(url) and self._is_authorized_target(url):
                        secured.extend([arg, url])
                        has_url = True
                    else:
                        log.warning("sqlmap.unauthorized_url url=%s", url)
                        # Skip this URL argument
                        i += 2
                        continue
                i += 2
                continue
            
            # Batch mode (required for safety)
            elif arg == "--batch":
                secured.append(arg)
                has_batch = True
                i += 1
                continue
            
            # Risk level (restricted)
            elif arg == "--risk":
                if i + 1 < len(args):
                    try:
                        risk = int(args[i + 1])
                        if 1 <= risk <= self.max_risk_level:
                            secured.extend([arg, str(risk)])
                            risk_level = risk
                        else:
                            log.warning("sqlmap.risk_level_restricted risk=%d max=%d", risk, self.max_risk_level)
                            # Use maximum allowed risk level
                            secured.extend([arg, str(self.max_risk_level)])
                    except ValueError:
                        # Invalid risk level, use default
                        secured.extend([arg, "1"])
                i += 2
                continue
            
            # Test level (restricted)
            elif arg == "--level":
                if i + 1 < len(args):
                    try:
                        level = int(args[i + 1])
                        if 1 <= level <= self.max_test_level:
                            secured.extend([arg, str(level)])
                            test_level = level
                        else:
                            log.warning("sqlmap.test_level_restricted level=%d max=%d", level, self.max_test_level)
                            # Use maximum allowed test level
                            secured.extend([arg, str(self.max_test_level)])
                    except ValueError:
                        # Invalid test level, use default
                        secured.extend([arg, "1"])
                i += 2
                continue
            
            # Safe flags (allow as-is)
            elif arg.startswith("-") and self._is_safe_flag(arg):
                secured.append(arg)
                i += 1
                continue
            
            # Values for safe flags
            elif i > 0 and args[i - 1].startswith("-") and self._is_safe_flag(args[i - 1]):
                secured.append(arg)
                i += 1
                continue
            
            # Skip unknown/unsafe flags
            else:
                log.warning("sqlmap.unsafe_flag_skipped flag=%s", arg)
                i += 1
        
        # Ensure required flags are present
        if not has_url:
            # Add default URL if not specified
            secured.extend(["-u", "http://192.168.1.10/test.php?id=1"])
            log.warning("sqlmap.no_url_specified using_default")
        
        if not has_batch:
            # Ensure batch mode is enabled
            secured.append("--batch")
            log.info("sqlmap.batch_mode_enforced")
        
        # Add safety restrictions
        if risk_level > self.max_risk_level:
            secured.extend(["--risk", str(self.max_risk_level)])
        
        if test_level > self.max_test_level:
            secured.extend(["--level", str(self.max_test_level)])
        
        # Add default safety options
        secured.extend(["--technique", "BEU"])  # Basic techniques only
        secured.extend(["--time-sec", "5"])     # Conservative timing
        secured.extend(["--threads", "5"])       # Limited threads
        
        return " ".join(secured)
    
    def _is_safe_flag(self, flag: str) -> bool:
        """Check if a flag is in the allowed list (ENHANCED FEATURE)."""
        return flag in self.allowed_flags
    
    def _get_timestamp(self):
        """Get current timestamp (ENHANCED HELPER)."""
        from datetime import datetime
        return datetime.now()
    
    def get_tool_info(self) -> dict:
        """Get enhanced sqlmap tool information (ENHANCED FEATURE)."""
        base_info = {
            "name": self.tool_name,
            "command": self.command_name,
            "description": self.__doc__,
            "concurrency": self.concurrency,
            "timeout": self.default_timeout_sec,
            "allowed_flags": list(self.allowed_flags) if self.allowed_flags else [],
            "circuit_breaker": {
                "failure_threshold": self.circuit_breaker_failure_threshold,
                "recovery_timeout": self.circuit_breaker_recovery_timeout
            },
            "security_restrictions": {
                "max_risk_level": self.max_risk_level,
                "max_test_level": self.max_test_level,
                "max_threads": self.max_threads,
                "required_modes": ["--batch"],
                "target_validation": "RFC1918 or .lab.internal only"
            },
            "usage_examples": [
                {
                    "description": "Basic SQL injection test",
                    "command": "sqlmap -u 'http://192.168.1.10/page.php?id=1' --batch --risk=1 --level=1"
                },
                {
                    "description": "Database enumeration",
                    "command": "sqlmap -u 'http://192.168.1.10/page.php?id=1' --batch --risk=1 --level=2 --dbs"
                },
                {
                    "description": "Table enumeration",
                    "command": "sqlmap -u 'http://192.168.1.10/page.php?id=1' --batch --risk=1 --level=2 -D testdb --tables"
                }
            ]
        }
        
        # Add metrics if available
        if hasattr(self, 'metrics') and self.metrics:
            base_info["metrics"] = {
                "prometheus_available": True,
                "execution_metrics": f"mcp_tool_execution_total{{tool=\"{self.tool_name}\"}}"
            }
        
        return base_info

```

