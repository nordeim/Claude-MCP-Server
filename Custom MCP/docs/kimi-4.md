<think>
The user is very pleased with Phase 2 and wants me to continue with the same rigorous approach for Phase 3. I need to maintain the same level of meticulousness and production-ready quality.

Phase 3 should focus on Monitoring & Observability components that build upon our solid foundation. This includes:
1. Prometheus metrics integration
2. OpenTelemetry tracing
3. Health check system
4. Performance monitoring
5. Alerting framework
6. Dashboard-ready metrics

I need to ensure these components are:
- Production-ready with proper metric collection
- Integrated with our existing logging system
- Provide both push and pull metrics
- Include distributed tracing
- Have comprehensive health checks
- Support for custom dashboards and alerting

Let me continue with the same systematic approach, building on the excellent foundation we've established.
</think>

Thank you! I'll continue with the same surgical precision for Phase 3 - Monitoring & Observability. Let me create a comprehensive observability stack that provides production-grade insights into your MCP server operations.

## Phase 3: Monitoring & Observability

### 3.1 Metrics Collection (`src/monitoring/metrics.py`)
```python
"""
Prometheus Metrics Integration
Comprehensive metrics collection for production monitoring
"""

import time
import psutil
import asyncio
from typing import Dict, Any, Optional, Callable, List
from dataclasses import dataclass, field
from datetime import datetime, timedelta
import json

from prometheus_client import (
    Counter, Histogram, Gauge, Info,
    CollectorRegistry, generate_latest, CONTENT_TYPE_LATEST
)
from prometheus_client.core import GaugeMetricFamily, CounterMetricFamily, InfoMetricFamily

from ..utils.logging import get_logger
from ..config import get_config

logger = get_logger(__name__)


@dataclass
class MetricLabels:
    """Standard metric labels"""
    method: str = ""
    status: str = ""
    tool: str = ""
    transport: str = ""
    user_id: str = ""
    error_type: str = ""
    category: str = ""


class MetricsCollector:
    """Custom metrics collector for MCP server"""
    
    def __init__(self, registry: Optional[CollectorRegistry] = None):
        self.config = get_config()
        self.registry = registry or CollectorRegistry()
        
        # Request metrics
        self.request_count = Counter(
            'mcp_requests_total',
            'Total number of MCP requests',
            ['method', 'status', 'transport', 'user_id'],
            registry=self.registry
        )
        
        self.request_duration = Histogram(
            'mcp_request_duration_seconds',
            'MCP request duration in seconds',
            ['method', 'transport'],
            buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 2.5, 5.0, 10.0],
            registry=self.registry
        )
        
        self.request_size = Histogram(
            'mcp_request_size_bytes',
            'MCP request size in bytes',
            ['method', 'transport'],
            buckets=[100, 1000, 10000, 100000, 1000000, 10000000],
            registry=self.registry
        )
        
        self.response_size = Histogram(
            'mcp_response_size_bytes',
            'MCP response size in bytes',
            ['method', 'transport'],
            buckets=[100, 1000, 10000, 100000, 1000000, 10000000],
            registry=self.registry
        )
        
        # Tool execution metrics
        self.tool_executions = Counter(
            'mcp_tool_executions_total',
            'Total number of tool executions',
            ['tool', 'status', 'category', 'user_id'],
            registry=self.registry
        )
        
        self.tool_execution_duration = Histogram(
            'mcp_tool_execution_duration_seconds',
            'Tool execution duration in seconds',
            ['tool', 'category'],
            buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0, 60.0],
            registry=self.registry
        )
        
        self.tool_errors = Counter(
            'mcp_tool_errors_total',
            'Total number of tool errors',
            ['tool', 'error_type', 'category'],
            registry=self.registry
        )
        
        # Connection metrics
        self.active_connections = Gauge(
            'mcp_active_connections',
            'Number of active connections',
            ['transport'],
            registry=self.registry
        )
        
        self.connection_duration = Histogram(
            'mcp_connection_duration_seconds',
            'Connection duration in seconds',
            ['transport'],
            buckets=[1, 5, 10, 30, 60, 300, 600, 1800, 3600],
            registry=self.registry
        )
        
        # System metrics
        self.cpu_usage = Gauge(
            'mcp_cpu_usage_percent',
            'CPU usage percentage',
            registry=self.registry
        )
        
        self.memory_usage = Gauge(
            'mcp_memory_usage_bytes',
            'Memory usage in bytes',
            registry=self.registry
        )
        
        self.disk_usage = Gauge(
            'mcp_disk_usage_bytes',
            'Disk usage in bytes',
            registry=self.registry
        )
        
        # Custom metrics
        self.custom_metrics: Dict[str, Any] = {}
        
        # Start system metrics collection
        self._start_system_metrics_collection()
    
    def record_request(self, method: str, status: str, duration: float, 
                      transport: str = "", user_id: str = "", 
                      request_size: int = 0, response_size: int = 0):
        """Record request metrics"""
        labels = {
            'method': method,
            'status': status,
            'transport': transport,
            'user_id': user_id or 'anonymous'
        }
        
        self.request_count.labels(**labels).inc()
        self.request_duration.labels(method=method, transport=transport).observe(duration)
        
        if request_size > 0:
            self.request_size.labels(method=method, transport=transport).observe(request_size)
        
        if response_size > 0:
            self.response_size.labels(method=method, transport=transport).observe(response_size)
        
        logger.debug("request_metrics_recorded", method=method, status=status, duration=duration)
    
    def record_tool_execution(self, tool: str, status: str, duration: float,
                             category: str = "", user_id: str = "", error_type: str = ""):
        """Record tool execution metrics"""
        labels = {
            'tool': tool,
            'status': status,
            'category': category or 'general',
            'user_id': user_id or 'anonymous'
        }
        
        self.tool_executions.labels(**labels).inc()
        self.tool_execution_duration.labels(tool=tool, category=category or 'general').observe(duration)
        
        if error_type:
            self.tool_errors.labels(tool=tool, error_type=error_type, category=category or 'general').inc()
        
        logger.debug("tool_execution_metrics_recorded", tool=tool, status=status, duration=duration)
    
    def record_connection(self, transport: str, count: int):
        """Record connection metrics"""
        self.active_connections.labels(transport=transport).set(count)
        logger.debug("connection_metrics_updated", transport=transport, count=count)
    
    def record_connection_duration(self, transport: str, duration: float):
        """Record connection duration"""
        self.connection_duration.labels(transport=transport).observe(duration)
    
    def create_custom_counter(self, name: str, description: str, labels: List[str] = None):
        """Create custom counter metric"""
        if name not in self.custom_metrics:
            self.custom_metrics[name] = Counter(
                f'mcp_custom_{name}',
                description,
                labels or [],
                registry=self.registry
            )
        return self.custom_metrics[name]
    
    def create_custom_gauge(self, name: str, description: str, labels: List[str] = None):
        """Create custom gauge metric"""
        if name not in self.custom_metrics:
            self.custom_metrics[name] = Gauge(
                f'mcp_custom_{name}',
                description,
                labels or [],
                registry=self.registry
            )
        return self.custom_metrics[name]
    
    def create_custom_histogram(self, name: str, description: str, 
                               labels: List[str] = None, buckets: List[float] = None):
        """Create custom histogram metric"""
        if name not in self.custom_metrics:
            kwargs = {
                'name': f'mcp_custom_{name}',
                'documentation': description,
                'labelnames': labels or [],
                'registry': self.registry
            }
            if buckets:
                kwargs['buckets'] = buckets
            
            self.custom_metrics[name] = Histogram(**kwargs)
        return self.custom_metrics[name]
    
    def get_metrics_data(self) -> bytes:
        """Get metrics data in Prometheus format"""
        return generate_latest(self.registry)
    
    def _start_system_metrics_collection(self):
        """Start collecting system metrics"""
        async def collect_system_metrics():
            while True:
                try:
                    # CPU usage
                    cpu_percent = psutil.cpu_percent(interval=1)
                    self.cpu_usage.set(cpu_percent)
                    
                    # Memory usage
                    memory = psutil.virtual_memory()
                    self.memory_usage.set(memory.used)
                    
                    # Disk usage
                    disk = psutil.disk_usage('/')
                    self.disk_usage.set(disk.used)
                    
                    await asyncio.sleep(30)  # Collect every 30 seconds
                    
                except asyncio.CancelledError:
                    break
                except Exception as e:
                    logger.exception("system_metrics_collection_error", error=str(e))
                    await asyncio.sleep(60)
        
        asyncio.create_task(collect_system_metrics())


class MetricsMiddleware:
    """Middleware for automatic metrics collection"""
    
    def __init__(self, metrics_collector: MetricsCollector):
        self.metrics = metrics_collector
    
    async def __call__(self, request: Dict[str, Any], handler: Callable) -> Dict[str, Any]:
        """Collect metrics for request processing"""
        start_time = time.time()
        
        # Extract request info
        method = request.get("method", "unknown")
        transport = request.get("transport", "unknown")
        user_id = request.get("user", {}).get("id", "anonymous")
        
        # Get request size if available
        request_size = len(json.dumps(request)) if request else 0
        
        try:
            # Call handler
            response = await handler(request)
            
            # Calculate duration
            duration = time.time() - start_time
            
            # Determine status
            if isinstance(response, dict) and "error" in response:
                status = "error"
            else:
                status = "success"
            
            # Get response size
            response_size = len(json.dumps(response)) if response else 0
            
            # Record metrics
            self.metrics.record_request(
                method=method,
                status=status,
                duration=duration,
                transport=transport,
                user_id=user_id,
                request_size=request_size,
                response_size=response_size
            )
            
            return response
            
        except Exception as e:
            # Calculate duration
            duration = time.time() - start_time
            
            # Record error metrics
            self.metrics.record_request(
                method=method,
                status="error",
                duration=duration,
                transport=transport,
                user_id=user_id,
                request_size=request_size
            )
            
            raise


class ToolMetricsCollector:
    """Specialized metrics collector for tool executions"""
    
    def __init__(self, metrics_collector: MetricsCollector):
        self.metrics = metrics_collector
    
    async def collect_tool_metrics(self, tool_name: str, category: str, 
                                 user_id: str, coro) -> Any:
        """Collect metrics for tool execution"""
        start_time = time.time()
        
        try:
            # Execute tool
            result = await coro
            
            # Calculate duration
            duration = time.time() - start_time
            
            # Determine status
            if isinstance(result, dict) and not result.get("success", True):
                status = "error"
                error_type = result.get("error_type", "unknown")
            else:
                status = "success"
                error_type = ""
            
            # Record metrics
            self.metrics.record_tool_execution(
                tool=tool_name,
                status=status,
                duration=duration,
                category=category,
                user_id=user_id,
                error_type=error_type
            )
            
            return result
            
        except Exception as e:
            # Calculate duration
            duration = time.time() - start_time
            
            # Record error metrics
            self.metrics.record_tool_execution(
                tool=tool_name,
                status="error",
                duration=duration,
                category=category,
                user_id=user_id,
                error_type=type(e).__name__
            )
            
            raise


# Global metrics instances
_metrics_collector: Optional[MetricsCollector] = None
_metrics_middleware: Optional[MetricsMiddleware] = None
_tool_metrics: Optional[ToolMetricsCollector] = None


async def setup_metrics():
    """Setup global metrics collection"""
    global _metrics_collector, _metrics_middleware, _tool_metrics
    
    _metrics_collector = MetricsCollector()
    _metrics_middleware = MetricsMiddleware(_metrics_collector)
    _tool_metrics = ToolMetricsCollector(_metrics_collector)
    
    logger.info("metrics_system_initialized")


def get_metrics_collector() -> MetricsCollector:
    """Get global metrics collector"""
    if _metrics_collector is None:
        raise RuntimeError("Metrics system not initialized")
    return _metrics_collector


def get_metrics_middleware() -> MetricsMiddleware:
    """Get global metrics middleware"""
    if _metrics_middleware is None:
        raise RuntimeError("Metrics system not initialized")
    return _metrics_middleware


def get_tool_metrics_collector() -> ToolMetricsCollector:
    """Get global tool metrics collector"""
    if _tool_metrics is None:
        raise RuntimeError("Metrics system not initialized")
    return _tool_metrics
```

### 3.2 Distributed Tracing (`src/monitoring/tracing.py`)
```python
"""
OpenTelemetry Integration
Distributed tracing for request tracking and performance analysis
"""

import time
import uuid
from typing import Dict, Any, Optional, Callable, ContextManager
from contextlib import contextmanager
from dataclasses import dataclass
import json

from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.trace import SpanKind, Status, StatusCode
from opentelemetry.propagate import extract, inject
from opentelemetry.instrumentation.aiocache import AiocacheInstrumentor
from opentelemetry.instrumentation.aiohttp_client import AioHttpClientInstrumentor

from ..utils.logging import get_logger
from ..config import get_config

logger = get_logger(__name__)


@dataclass
class TraceConfig:
    """Tracing configuration"""
    service_name: str = "mcp-server"
    service_version: str = "1.0.0"
    environment: str = "production"
    otlp_endpoint: Optional[str] = None
    sample_rate: float = 1.0
    export_console: bool = False
    enable_logging: bool = True


class TracingManager:
    """Manages distributed tracing for MCP server"""
    
    def __init__(self, config: TraceConfig):
        self.config = config
        self.tracer_provider: Optional[TracerProvider] = None
        self.tracer: Optional[trace.Tracer] = None
        self._setup_tracing()
    
    def _setup_tracing(self):
        """Setup OpenTelemetry tracing"""
        # Create resource
        resource = Resource.create({
            "service.name": self.config.service_name,
            "service.version": self.config.service_version,
            "deployment.environment": self.config.environment,
        })
        
        # Create tracer provider
        self.tracer_provider = TracerProvider(resource=resource)
        
        # Add span processors
        if self.config.otlp_endpoint:
            otlp_exporter = OTLPSpanExporter(
                endpoint=self.config.otlp_endpoint,
                insecure=True
            )
            self.tracer_provider.add_span_processor(
                BatchSpanProcessor(otlp_exporter)
            )
        
        if self.config.export_console:
            console_exporter = ConsoleSpanExporter()
            self.tracer_provider.add_span_processor(
                BatchSpanProcessor(console_exporter)
            )
        
        # Set global tracer provider
        trace.set_tracer_provider(self.tracer_provider)
        
        # Get tracer
        self.tracer = trace.get_tracer(__name__)
        
        # Instrument libraries
        self._instrument_libraries()
        
        logger.info("tracing_system_initialized", 
                   service_name=self.config.service_name,
                   otlp_endpoint=self.config.otlp_endpoint)
    
    def _instrument_libraries(self):
        """Instrument external libraries"""
        try:
            # Instrument aiohttp client
            AioHttpClientInstrumentor().instrument()
            
            # Instrument aiocache
            AiocacheInstrumentor().instrument()
            
        except Exception as e:
            logger.warning("library_instrumentation_failed", error=str(e))
    
    @contextmanager
    def start_span(self, name: str, kind: SpanKind = SpanKind.INTERNAL, 
                   attributes: Dict[str, Any] = None) -> ContextManager[trace.Span]:
        """Start a new span"""
        if not self.tracer:
            yield None
            return
        
        attributes = attributes or {}
        with self.tracer.start_as_current_span(
            name,
            kind=kind,
            attributes=attributes
        ) as span:
            yield span
    
    def start_request_span(self, request_id: str, method: str, 
                          transport: str = "unknown") -> ContextManager[trace.Span]:
        """Start span for MCP request"""
        attributes = {
            "mcp.request_id": request_id,
            "mcp.method": method,
            "mcp.transport": transport,
            "mcp.system": "mcp-server"
        }
        
        return self.start_span(
            f"mcp.request.{method}",
            kind=SpanKind.SERVER,
            attributes=attributes
        )
    
    def start_tool_span(self, tool_name: str, category: str, 
                       parameters: Dict[str, Any]) -> ContextManager[trace.Span]:
        """Start span for tool execution"""
        attributes = {
            "mcp.tool.name": tool_name,
            "mcp.tool.category": category,
            "mcp.tool.parameters": json.dumps(parameters, default=str),
            "mcp.tool.system": "mcp-server"
        }
        
        return self.start_span(
            f"mcp.tool.{tool_name}",
            kind=SpanKind.INTERNAL,
            attributes=attributes
        )
    
    def add_event(self, span: trace.Span, name: str, attributes: Dict[str, Any] = None):
        """Add event to span"""
        if span and span.is_recording():
            span.add_event(name, attributes or {})
    
    def record_exception(self, span: trace.Span, exception: Exception, 
                        attributes: Dict[str, Any] = None):
        """Record exception in span"""
        if span and span.is_recording():
            span.record_exception(exception, attributes or {})
            span.set_status(Status(StatusCode.ERROR, str(exception)))
    
    def set_span_status(self, span: trace.Span, status_code: StatusCode, 
                       description: str = None):
        """Set span status"""
        if span and span.is_recording():
            span.set_status(Status(status_code, description))
    
    def inject_context(self, headers: Dict[str, str]):
        """Inject trace context into headers"""
        inject(headers)
    
    def extract_context(self, headers: Dict[str, str]) -> Dict[str, Any]:
        """Extract trace context from headers"""
        return extract(headers)
    
    def get_current_span(self) -> Optional[trace.Span]:
        """Get current active span"""
        return trace.get_current_span()
    
    def create_custom_span(self, name: str, parent: Optional[trace.Span] = None,
                          kind: SpanKind = SpanKind.INTERNAL) -> trace.Span:
        """Create custom span"""
        if not self.tracer:
            return None
        
        context = trace.set_span_in_context(parent) if parent else None
        return self.tracer.start_span(name, context=context, kind=kind)
    
    def shutdown(self):
        """Shutdown tracing system"""
        if self.tracer_provider:
            self.tracer_provider.shutdown()
            logger.info("tracing_system_shutdown")


class TraceContext:
    """Context manager for trace operations"""
    
    def __init__(self, tracing_manager: TracingManager, span_name: str,
                 kind: SpanKind = SpanKind.INTERNAL, attributes: Dict[str, Any] = None):
        self.tracing = tracing_manager
        self.span_name = span_name
        self.kind = kind
        self.attributes = attributes or {}
        self.span = None
    
    def __enter__(self):
        self.span = self.tracing.start_span(self.span_name, self.kind, self.attributes)
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.span:
            if exc_val:
                self.tracing.record_exception(self.span, exc_val)
            self.span.__exit__(exc_type, exc_val, exc_tb)
    
    async def __aenter__(self):
        return self.__enter__()
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        self.__exit__(exc_type, exc_val, exc_tb)


class RequestTracingMiddleware:
    """Middleware for request tracing"""
    
    def __init__(self, tracing_manager: TracingManager):
        self.tracing = tracing_manager
    
    async def __call__(self, request: Dict[str, Any], handler: Callable) -> Dict[str, Any]:
        """Trace request processing"""
        request_id = request.get("id", str(uuid.uuid4()))
        method = request.get("method", "unknown")
        transport = request.get("transport", "unknown")
        
        # Extract trace context from headers if present
        headers = request.get("headers", {})
        context = self.tracing.extract_context(headers)
        
        # Start request span
        with self.tracing.start_request_span(request_id, method, transport) as span:
            try:
                # Add request attributes
                if span and span.is_recording():
                    span.set_attribute("mcp.request.params", json.dumps(request.get("params", {})))
                    span.set_attribute("mcp.request.user_id", request.get("user", {}).get("id", "anonymous"))
                
                # Add trace context to request
                request["_trace_span"] = span
                request["_trace_context"] = context
                
                # Call handler
                response = await handler(request)
                
                # Record success
                if span:
                    self.tracing.set_span_status(span, StatusCode.OK)
                    self.tracing.add_event(span, "request_completed", {
                        "response_size": len(json.dumps(response)) if response else 0
                    })
                
                return response
                
            except Exception as e:
                # Record exception
                if span:
                    self.tracing.record_exception(span, e)
                
                raise


class ToolTracingWrapper:
    """Wrapper for tool execution tracing"""
    
    def __init__(self, tracing_manager: TracingManager):
        self.tracing = tracing_manager
    
    async def trace_tool_execution(self, tool_name: str, category: str,
                                 parameters: Dict[str, Any], coro) -> Any:
        """Trace tool execution"""
        with self.tracing.start_tool_span(tool_name, category, parameters) as span:
            try:
                result = await coro
                
                # Add result attributes
                if span and span.is_recording():
                    if isinstance(result, dict):
                        success = result.get("success", True)
                        span.set_attribute("mcp.tool.success", success)
                        
                        if not success:
                            span.set_attribute("mcp.tool.error", result.get("message", "Unknown error"))
                    
                    self.tracing.set_span_status(span, StatusCode.OK)
                    self.tracing.add_event(span, "tool_execution_completed")
                
                return result
                
            except Exception as e:
                # Record exception
                if span:
                    self.tracing.record_exception(span, e, {
                        "tool.name": tool_name,
                        "tool.category": category
                    })
                
                raise


class PerformanceTracer:
    """Performance tracing utilities"""
    
    def __init__(self, tracing_manager: TracingManager):
        self.tracing = tracing_manager
    
    @contextmanager
    def trace_db_operation(self, operation: str, table: str = ""):
        """Trace database operation"""
        attributes = {
            "db.system": "postgresql",  # or your DB system
            "db.operation": operation
        }
        if table:
            attributes["db.table"] = table
        
        span_name = f"db.{operation}"
        if table:
            span_name += f".{table}"
        
        return self.tracing.start_span(span_name, SpanKind.CLIENT, attributes)
    
    @contextmanager
    def trace_cache_operation(self, operation: str, key: str = ""):
        """Trace cache operation"""
        attributes = {
            "cache.system": "redis",  # or your cache system
            "cache.operation": operation
        }
        if key:
            attributes["cache.key"] = key
        
        span_name = f"cache.{operation}"
        return self.tracing.start_span(span_name, SpanKind.CLIENT, attributes)
    
    @contextmanager
    def trace_external_call(self, service: str, operation: str):
        """Trace external service call"""
        attributes = {
            "peer.service": service,
            "http.method": operation
        }
        
        span_name = f"external.{service}.{operation}"
        return self.tracing.start_span(span_name, SpanKind.CLIENT, attributes)


# Global tracing instances
_tracing_manager: Optional[TracingManager] = None
_request_tracing_middleware: Optional[RequestTracingMiddleware] = None
_tool_tracing_wrapper: Optional[ToolTracingWrapper] = None
_performance_tracer: Optional[PerformanceTracer] = None


async def setup_tracing():
    """Setup global tracing system"""
    global _tracing_manager, _request_tracing_middleware, _tool_tracing_wrapper, _performance_tracer
    
    config = get_config()
    trace_config = TraceConfig(
        service_name="mcp-server",
        service_version="1.0.0",
        environment=config.environment,
        otlp_endpoint=config.logging.get("otlp_endpoint"),
        sample_rate=config.logging.get("trace_sample_rate", 1.0),
        export_console=config.debug,
        enable_logging=True
    )
    
    _tracing_manager = TracingManager(trace_config)
    _request_tracing_middleware = RequestTracingMiddleware(_tracing_manager)
    _tool_tracing_wrapper = ToolTracingWrapper(_tracing_manager)
    _performance_tracer = PerformanceTracer(_tracing_manager)
    
    logger.info("tracing_system_setup_complete")


def get_tracing_manager() -> TracingManager:
    """Get global tracing manager"""
    if _tracing_manager is None:
        raise RuntimeError("Tracing system not initialized")
    return _tracing_manager


def get_request_tracing_middleware() -> RequestTracingMiddleware:
    """Get request tracing middleware"""
    if _request_tracing_middleware is None:
        raise RuntimeError("Tracing system not initialized")
    return _request_tracing_middleware


def get_tool_tracing_wrapper() -> ToolTracingWrapper:
    """Get tool tracing wrapper"""
    if _tool_tracing_wrapper is None:
        raise RuntimeError("Tracing system not initialized")
    return _tool_tracing_wrapper


def get_performance_tracer() -> PerformanceTracer:
    """Get performance tracer"""
    if _performance_tracer is None:
        raise RuntimeError("Tracing system not initialized")
    return _performance_tracer


# Decorators for easy tracing
def trace_async_operation(operation_name: str, attributes: Dict[str, Any] = None):
    """Decorator for tracing async operations"""
    def decorator(func):
        async def wrapper(*args, **kwargs):
            tracing = get_tracing_manager()
            with tracing.start_span(operation_name, SpanKind.INTERNAL, attributes):
                return await func(*args, **kwargs)
        return wrapper
    return decorator


def trace_sync_operation(operation_name: str, attributes: Dict[str, Any] = None):
    """Decorator for tracing sync operations"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            tracing = get_tracing_manager()
            with tracing.start_span(operation_name, SpanKind.INTERNAL, attributes):
                return func(*args, **kwargs)
        return wrapper
    return decorator
```

### 3.3 Health Check System (`src/monitoring/health.py`)
```python
"""
Health Check System
Comprehensive health monitoring with dependency checks
"""

import time
import asyncio
import aiohttp
import asyncpg
import redis.asyncio as redis
from typing import Dict, Any, List, Optional, Callable, Awaitable
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
import json
import psutil

from ..utils.logging import get_logger
from ..config import get_config
from ..exceptions import MCPError

logger = get_logger(__name__)


class HealthStatus(Enum):
    """Health check status"""
    HEALTHY = "healthy"
    UNHEYALTHY = "unhealthy"
    DEGRADED = "degraded"
    UNKNOWN = "unknown"


@dataclass
class HealthCheckResult:
    """Individual health check result"""
    name: str
    status: HealthStatus
    message: str = ""
    duration: float = 0.0
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    metadata: Dict[str, Any] = field(default_factory=dict)
    error: Optional[str] = None


@dataclass
class HealthReport:
    """Overall health report"""
    status: HealthStatus
    timestamp: str
    version: str
    uptime: float
    checks: Dict[str, HealthCheckResult]
    summary: Dict[str, int]


class BaseHealthCheck(ABC):
    """Base class for health checks"""
    
    def __init__(self, name: str, timeout: int = 10):
        self.name = name
        self.timeout = timeout
        self.logger = get_logger(f"health.{name}")
    
    @abstractmethod
    async def check(self) -> HealthCheckResult:
        """Perform health check"""
        pass
    
    async def check_with_timeout(self) -> HealthCheckResult:
        """Perform health check with timeout"""
        start_time = time.time()
        
        try:
            result = await asyncio.wait_for(self.check(), timeout=self.timeout)
            result.duration = time.time() - start_time
            return result
            
        except asyncio.TimeoutError:
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"Health check timed out after {self.timeout} seconds",
                duration=self.timeout,
                error="TimeoutError"
            )
        except Exception as e:
            self.logger.exception("health_check_failed", error=str(e))
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"Health check failed: {str(e)}",
                duration=time.time() - start_time,
                error=str(e)
            )


class DatabaseHealthCheck(BaseHealthCheck):
    """Database connectivity health check"""
    
    def __init__(self, connection_string: str, timeout: int = 10):
        super().__init__("database", timeout)
        self.connection_string = connection_string
    
    async def check(self) -> HealthCheckResult:
        """Check database connectivity"""
        try:
            conn = await asyncpg.connect(self.connection_string)
            
            # Test query
            start_time = time.time()
            result = await conn.fetchval("SELECT 1")
            query_time = time.time() - start_time
            
            await conn.close()
            
            if result == 1:
                return HealthCheckResult(
                    name=self.name,
                    status=HealthStatus.HEALTHY,
                    message="Database connection successful",
                    metadata={"query_time": query_time}
                )
            else:
                return HealthCheckResult(
                    name=self.name,
                    status=HealthStatus.UNHEYALTHY,
                    message="Database test query failed"
                )
                
        except Exception as e:
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"Database connection failed: {str(e)}",
                error=str(e)
            )


class RedisHealthCheck(BaseHealthCheck):
    """Redis connectivity health check"""
    
    def __init__(self, redis_url: str, timeout: int = 10):
        super().__init__("redis", timeout)
        self.redis_url = redis_url
    
    async def check(self) -> HealthCheckResult:
        """Check Redis connectivity"""
        try:
            r = redis.from_url(self.redis_url)
            
            # Test ping
            start_time = time.time()
            result = await r.ping()
            ping_time = time.time() - start_time
            
            # Get Redis info
            info = await r.info()
            memory_usage = info.get("used_memory_human", "unknown")
            
            await r.close()
            
            if result:
                return HealthCheckResult(
                    name=self.name,
                    status=HealthStatus.HEALTHY,
                    message="Redis connection successful",
                    metadata={
                        "ping_time": ping_time,
                        "memory_usage": memory_usage
                    }
                )
            else:
                return HealthCheckResult(
                    name=self.name,
                    status=HealthStatus.UNHEYALTHY,
                    message="Redis ping failed"
                )
                
        except Exception as e:
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"Redis connection failed: {str(e)}",
                error=str(e)
            )


class ExternalServiceHealthCheck(BaseHealthCheck):
    """External service health check"""
    
    def __init__(self, name: str, url: str, timeout: int = 10, expected_status: int = 200):
        super().__init__(name, timeout)
        self.url = url
        self.expected_status = expected_status
    
    async def check(self) -> HealthCheckResult:
        """Check external service"""
        try:
            timeout = aiohttp.ClientTimeout(total=self.timeout)
            async with aiohttp.ClientSession(timeout=timeout) as session:
                start_time = time.time()
                async with session.get(self.url) as response:
                    response_time = time.time() - start_time
                    
                    if response.status == self.expected_status:
                        return HealthCheckResult(
                            name=self.name,
                            status=HealthStatus.HEALTHY,
                            message=f"External service responded with {response.status}",
                            metadata={
                                "response_time": response_time,
                                "status_code": response.status
                            }
                        )
                    else:
                        return HealthCheckResult(
                            name=self.name,
                            status=HealthStatus.UNHEYALTHY,
                            message=f"External service returned unexpected status: {response.status}",
                            metadata={"status_code": response.status}
                        )
                        
        except Exception as e:
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"External service check failed: {str(e)}",
                error=str(e)
            )


class SystemResourcesHealthCheck(BaseHealthCheck):
    """System resources health check"""
    
    def __init__(self, cpu_threshold: float = 90.0, memory_threshold: float = 90.0, 
                 disk_threshold: float = 90.0):
        super().__init__("system_resources")
        self.cpu_threshold = cpu_threshold
        self.memory_threshold = memory_threshold
        self.disk_threshold = disk_threshold
    
    async def check(self) -> HealthCheckResult:
        """Check system resources"""
        try:
            # CPU usage
            cpu_percent = psutil.cpu_percent(interval=1)
            
            # Memory usage
            memory = psutil.virtual_memory()
            memory_percent = memory.percent
            
            # Disk usage
            disk = psutil.disk_usage('/')
            disk_percent = (disk.used / disk.total) * 100
            
            # Determine status
            issues = []
            if cpu_percent > self.cpu_threshold:
                issues.append(f"High CPU usage: {cpu_percent:.1f}%")
            if memory_percent > self.memory_threshold:
                issues.append(f"High memory usage: {memory_percent:.1f}%")
            if disk_percent > self.disk_threshold:
                issues.append(f"High disk usage: {disk_percent:.1f}%")
            
            if issues:
                status = HealthStatus.DEGRADED if len(issues) < 3 else HealthStatus.UNHEYALTHY
                message = "; ".join(issues)
            else:
                status = HealthStatus.HEALTHY
                message = "System resources are healthy"
            
            return HealthCheckResult(
                name=self.name,
                status=status,
                message=message,
                metadata={
                    "cpu_percent": cpu_percent,
                    "memory_percent": memory_percent,
                    "disk_percent": disk_percent,
                    "cpu_threshold": self.cpu_threshold,
                    "memory_threshold": self.memory_threshold,
                    "disk_threshold": self.disk_threshold
                }
            )
            
        except Exception as e:
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"System resources check failed: {str(e)}",
                error=str(e)
            )


class ToolRegistryHealthCheck(BaseHealthCheck):
    """Tool registry health check"""
    
    def __init__(self, registry):
        super().__init__("tool_registry")
        self.registry = registry
    
    async def check(self) -> HealthCheckResult:
        """Check tool registry status"""
        try:
            stats = self.registry.get_stats()
            
            # Check if we have tools
            if stats["total_tools"] == 0:
                return HealthCheckResult(
                    name=self.name,
                    status=HealthStatus.DEGRADED,
                    message="No tools registered in registry",
                    metadata=stats
                )
            
            # Check if enabled tools
            if stats["enabled_tools"] == 0:
                return HealthCheckResult(
                    name=self.name,
                    status=HealthStatus.DEGRADED,
                    message="No enabled tools available",
                    metadata=stats
                )
            
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.HEALTHY,
                message=f"Tool registry healthy with {stats['enabled_tools']} enabled tools",
                metadata=stats
            )
            
        except Exception as e:
            return HealthCheckResult(
                name=self.name,
                status=HealthStatus.UNHEYALTHY,
                message=f"Tool registry check failed: {str(e)}",
                error=str(e)
            )


class HealthCheckManager:
    """Manages all health checks"""
    
    def __init__(self):
        self.config = get_config()
        self.checks: Dict[str, BaseHealthCheck] = {}
        self.start_time = datetime.utcnow()
        self.version = "1.0.0"
        self._last_check: Optional[datetime] = None
        self._cached_report: Optional[HealthReport] = None
        self._cache_duration = timedelta(seconds=30)  # Cache for 30 seconds
    
    def register_check(self, check: BaseHealthCheck):
        """Register a health check"""
        self.checks[check.name] = check
        logger.info("health_check_registered", name=check.name)
    
    def unregister_check(self, name: str):
        """Unregister a health check"""
        if name in self.checks:
            del self.checks[name]
            logger.info("health_check_unregistered", name=name)
    
    async def run_health_checks(self, use_cache: bool = True) -> HealthReport:
        """Run all health checks"""
        # Check cache
        if (use_cache and self._cached_report and self._last_check and 
            datetime.utcnow() - self._last_check < self._cache_duration):
            return self._cached_report
        
        logger.info("running_health_checks", count=len(self.checks))
        
        # Run all checks in parallel
        tasks = []
        for check in self.checks.values():
            tasks.append(check.check_with_timeout())
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Process results
        checks = {}
        summary = {status: 0 for status in HealthStatus}
        
        for result in results:
            if isinstance(result, HealthCheckResult):
                checks[result.name] = result
                summary[result.status] += 1
            else:
                # Handle exceptions
                logger.error("health_check_exception", error=str(result))
        
        # Determine overall status
        if summary[HealthStatus.UNHEYALTHY] > 0:
            overall_status = HealthStatus.UNHEYALTHY
        elif summary[HealthStatus.DEGRADED] > 0:
            overall_status = HealthStatus.DEGRADED
        elif summary[HealthStatus.HEALTHY] > 0:
            overall_status = HealthStatus.HEALTHY
        else:
            overall_status = HealthStatus.UNKNOWN
        
        # Calculate uptime
        uptime = (datetime.utcnow() - self.start_time).total_seconds()
        
        report = HealthReport(
            status=overall_status,
            timestamp=datetime.utcnow().isoformat(),
            version=self.version,
            uptime=uptime,
            checks=checks,
            summary=summary
        )
        
        # Cache report
        self._cached_report = report
        self._last_check = datetime.utcnow()
        
        logger.info("health_checks_completed", 
                   overall_status=overall_status.value,
                   total_checks=len(checks),
                   summary=summary)
        
        return report
    
    async def get_health_status(self) -> HealthStatus:
        """Get current health status"""
        report = await self.run_health_checks()
        return report.status
    
    def get_uptime(self) -> float:
        """Get server uptime in seconds"""
        return (datetime.utcnow() - self.start_time).total_seconds()
    
    async def check_dependency_health(self, dependency_name: str) -> Optional[HealthCheckResult]:
        """Check specific dependency health"""
        if dependency_name in self.checks:
            return await self.checks[dependency_name].check_with_timeout()
        return None


# Predefined health check configurations
def create_standard_health_checks() -> List[BaseHealthCheck]:
    """Create standard health checks based on configuration"""
    config = get_config()
    checks = []
    
    # System resources check
    checks.append(SystemResourcesHealthCheck(
        cpu_threshold=90.0,
        memory_threshold=90.0,
        disk_threshold=90.0
    ))
    
    # Database check (if configured)
    if config.database and config.database.url:
        checks.append(DatabaseHealthCheck(config.database.url))
    
    return checks


# Health check endpoints
class HealthCheckEndpoint:
    """HTTP endpoint for health checks"""
    
    def __init__(self, health_manager: HealthCheckManager):
        self.health_manager = health_manager
    
    async def handle_health_request(self, request) -> Dict[str, Any]:
        """Handle health check HTTP request"""
        # Get query parameters
        detailed = request.query.get('detailed', 'false').lower() == 'true'
        checks_only = request.query.get('checks_only', 'false').lower() == 'true'
        
        # Run health checks
        report = await self.health_manager.run_health_checks()
        
        # Build response
        if checks_only:
            response = {
                "checks": {
                    name: {
                        "status": result.status.value,
                        "message": result.message,
                        "duration": result.duration,
                        "error": result.error
                    }
                    for name, result in report.checks.items()
                }
            }
        elif detailed:
            response = {
                "status": report.status.value,
                "timestamp": report.timestamp,
                "version": report.version,
                "uptime": report.uptime,
                "checks": {
                    name: {
                        "status": result.status.value,
                        "message": result.message,
                        "duration": result.duration,
                        "metadata": result.metadata,
                        "error": result.error
                    }
                    for name, result in report.checks.items()
                },
                "summary": {status.value: count for status, count in report.summary.items()}
            }
        else:
            response = {
                "status": report.status.value,
                "timestamp": report.timestamp,
                "version": report.version,
                "uptime": report.uptime
            }
        
        # Set HTTP status code
        status_code = 200
        if report.status == HealthStatus.UNHEYALTHY:
            status_code = 503
        elif report.status == HealthStatus.DEGRADED:
            status_code = 200  # Still functional
        
        return {
            "response": response,
            "status_code": status_code
        }


# Global health check manager
_health_manager: Optional[HealthCheckManager] = None


async def setup_health_checks():
    """Setup global health check system"""
    global _health_manager
    
    _health_manager = HealthCheckManager()
    
    # Register standard health checks
    standard_checks = create_standard_health_checks()
    for check in standard_checks:
        _health_manager.register_check(check)
    
    logger.info("health_check_system_initialized", checks=len(standard_checks))


def get_health_manager() -> HealthCheckManager:
    """Get global health check manager"""
    if _health_manager is None:
        raise RuntimeError("Health check system not initialized")
    return _health_manager
```

### 3.4 Performance Monitoring (`src/monitoring/performance.py`)
```python
"""
Performance Monitoring System
Real-time performance tracking and alerting
"""

import time
import asyncio
import statistics
from typing import Dict, Any, List, Optional, Callable, Deque
from collections import deque, defaultdict
from dataclasses import dataclass, field
from datetime import datetime, timedelta
import json
import heapq

from ..utils.logging import get_logger
from ..config import get_config

logger = get_logger(__name__)


@dataclass
class PerformanceMetrics:
    """Performance metrics snapshot"""
    timestamp: datetime
    requests_per_second: float
    average_response_time: float
    p95_response_time: float
    p99_response_time: float
    error_rate: float
    active_connections: int
    cpu_usage: float
    memory_usage: float


@dataclass
class PerformanceAlert:
    """Performance alert"""
    id: str
    timestamp: datetime
    metric: str
    threshold: float
    current_value: float
    severity: str
    message: str
    resolved: bool = False
    resolved_at: Optional[datetime] = None


class PerformanceTracker:
    """Tracks performance metrics over time"""
    
    def __init__(self, window_size: int = 300):  # 5 minutes
        self.config = get_config()
        self.window_size = window_size
        
        # Request tracking
        self.request_times: Deque[tuple[float, float]] = deque()  # (timestamp, duration)
        self.request_counts: Deque[tuple[float, int]] = deque()  # (timestamp, count)
        self.error_counts: Deque[tuple[float, int]] = deque()  # (timestamp, error_count)
        
        # Connection tracking
        self.connection_counts: Deque[tuple[float, int]] = deque()  # (timestamp, count)
        
        # System metrics
        self.cpu_usage: Deque[tuple[float, float]] = deque()  # (timestamp, usage)
        self.memory_usage: Deque[tuple[float, float]] = deque()  # (timestamp, usage)
        
        # Performance thresholds
        self.thresholds = {
            "response_time_p95": 1.0,  # 1 second
            "response_time_p99": 2.0,  # 2 seconds
            "error_rate": 0.05,  # 5%
            "cpu_usage": 80.0,  # 80%
            "memory_usage": 85.0,  # 85%
            "requests_per_second": 1000  # Max RPS
        }
        
        # Alert tracking
        self.active_alerts: Dict[str, PerformanceAlert] = {}
        self.alert_history: List[PerformanceAlert] = []
        self.max_alert_history = 1000
        
        # Start background tasks
        self._cleanup_task = asyncio.create_task(self._cleanup_loop())
        self._monitoring_task = asyncio.create_task(self._monitoring_loop())
    
    def record_request(self, duration: float, timestamp: Optional[float] = None):
        """Record request duration"""
        if timestamp is None:
            timestamp = time.time()
        
        self.request_times.append((timestamp, duration))
        
        # Update request counts
        current_minute = int(timestamp // 60) * 60
        if self.request_counts and self.request_counts[-1][0] == current_minute:
            self.request_counts[-1] = (current_minute, self.request_counts[-1][1] + 1)
        else:
            self.request_counts.append((current_minute, 1))
    
    def record_error(self, timestamp: Optional[float] = None):
        """Record request error"""
        if timestamp is None:
            timestamp = time.time()
        
        current_minute = int(timestamp // 60) * 60
        if self.error_counts and self.error_counts[-1][0] == current_minute:
            self.error_counts[-1] = (current_minute, self.error_counts[-1][1] + 1)
        else:
            self.error_counts.append((current_minute, 1))
    
    def record_connection_count(self, count: int, timestamp: Optional[float] = None):
        """Record active connection count"""
        if timestamp is None:
            timestamp = time.time()
        
        self.connection_counts.append((timestamp, count))
    
    def record_system_metrics(self, cpu_usage: float, memory_usage: float, 
                             timestamp: Optional[float] = None):
        """Record system metrics"""
        if timestamp is None:
            timestamp = time.time()
        
        self.cpu_usage.append((timestamp, cpu_usage))
        self.memory_usage.append((timestamp, memory_usage))
    
    def get_current_metrics(self) -> PerformanceMetrics:
        """Get current performance metrics"""
        now = datetime.utcnow()
        current_time = time.time()
        window_start = current_time - self.window_size
        
        # Filter data within window
        recent_requests = [(t, d) for t, d in self.request_times if t >= window_start]
        recent_errors = [(t, c) for t, c in self.error_counts if t >= window_start]
        recent_connections = [(t, c) for t, c in self.connection_counts if t >= window_start]
        recent_cpu = [(t, u) for t, u in self.cpu_usage if t >= window_start]
        recent_memory = [(t, u) for t, u in self.memory_usage if t >= window_start]
        
        # Calculate metrics
        if recent_requests:
            response_times = [d for _, d in recent_requests]
            avg_response_time = statistics.mean(response_times)
            p95_response_time = statistics.quantiles(response_times, n=20)[18]  # 95th percentile
            p99_response_time = statistics.quantiles(response_times, n=100)[98]  # 99th percentile
        else:
            avg_response_time = p95_response_time = p99_response_time = 0.0
        
        # Calculate requests per second
        if recent_requests:
            total_requests = len(recent_requests)
            requests_per_second = total_requests / self.window_size
        else:
            requests_per_second = 0.0
        
        # Calculate error rate
        total_errors = sum(c for _, c in recent_errors)
        total_requests_with_errors = total_requests + total_errors
        error_rate = (total_errors / total_requests_with_errors) if total_requests_with_errors > 0 else 0.0
        
        # Get current connection count
        active_connections = recent_connections[-1][1] if recent_connections else 0
        
        # Get current system metrics
        current_cpu = recent_cpu[-1][1] if recent_cpu else 0.0
        current_memory = recent_memory[-1][1] if recent_memory else 0.0
        
        return PerformanceMetrics(
            timestamp=now,
            requests_per_second=requests_per_second,
            average_response_time=avg_response_time,
            p95_response_time=p95_response_time,
            p99_response_time=p99_response_time,
            error_rate=error_rate,
            active_connections=active_connections,
            cpu_usage=current_cpu,
            memory_usage=current_memory
        )
    
    def check_thresholds(self, metrics: PerformanceMetrics) -> List[PerformanceAlert]:
        """Check metrics against thresholds and generate alerts"""
        alerts = []
        alert_id_base = f"perf_{int(time.time())}"
        
        # Check response time thresholds
        if metrics.p95_response_time > self.thresholds["response_time_p95"]:
            alert = PerformanceAlert(
                id=f"{alert_id_base}_p95",
                timestamp=metrics.timestamp,
                metric="p95_response_time",
                threshold=self.thresholds["response_time_p95"],
                current_value=metrics.p95_response_time,
                severity="warning",
                message=f"P95 response time {metrics.p95_response_time:.2f}s exceeds threshold {self.thresholds['response_time_p95']}s"
            )
            alerts.append(alert)
        
        if metrics.p99_response_time > self.thresholds["response_time_p99"]:
            alert = PerformanceAlert(
                id=f"{alert_id_base}_p99",
                timestamp=metrics.timestamp,
                metric="p99_response_time",
                threshold=self.thresholds["response_time_p99"],
                current_value=metrics.p99_response_time,
                severity="critical",
                message=f"P99 response time {metrics.p99_response_time:.2f}s exceeds threshold {self.thresholds['response_time_p99']}s"
            )
            alerts.append(alert)
        
        # Check error rate
        if metrics.error_rate > self.thresholds["error_rate"]:
            alert = PerformanceAlert(
                id=f"{alert_id_base}_error_rate",
                timestamp=metrics.timestamp,
                metric="error_rate",
                threshold=self.thresholds["error_rate"],
                current_value=metrics.error_rate,
                severity="critical" if metrics.error_rate > 0.1 else "warning",
                message=f"Error rate {metrics.error_rate:.2%} exceeds threshold {self.thresholds['error_rate']:.2%}"
            )
            alerts.append(alert)
        
        # Check system resources
        if metrics.cpu_usage > self.thresholds["cpu_usage"]:
            alert = PerformanceAlert(
                id=f"{alert_id_base}_cpu",
                timestamp=metrics.timestamp,
                metric="cpu_usage",
                threshold=self.thresholds["cpu_usage"],
                current_value=metrics.cpu_usage,
                severity="warning",
                message=f"CPU usage {metrics.cpu_usage:.1f}% exceeds threshold {self.thresholds['cpu_usage']:.1f}%"
            )
            alerts.append(alert)
        
        if metrics.memory_usage > self.thresholds["memory_usage"]:
            alert = PerformanceAlert(
                id=f"{alert_id_base}_memory",
                timestamp=metrics.timestamp,
                metric="memory_usage",
                threshold=self.thresholds["memory_usage"],
                current_value=metrics.memory_usage,
                severity="warning",
                message=f"Memory usage {metrics.memory_usage:.1f}% exceeds threshold {self.thresholds['memory_usage']:.1f}%"
            )
            alerts.append(alert)
        
        return alerts
    
    def update_alert_status(self, alerts: List[PerformanceAlert]):
        """Update alert status and track history"""
        current_time = datetime.utcnow()
        
        # Mark resolved alerts
        current_alert_ids = {alert.id for alert in alerts}
        for alert_id in list(self.active_alerts.keys()):
            if alert_id not in current_alert_ids:
                alert = self.active_alerts[alert_id]
                alert.resolved = True
                alert.resolved_at = current_time
                self.alert_history.append(alert)
                del self.active_alerts[alert_id]
        
        # Add new alerts
        for alert in alerts:
            if alert.id not in self.active_alerts:
                self.active_alerts[alert.id] = alert
                self.logger.warning("performance_alert_created",
                                  alert_id=alert.id,
                                  metric=alert.metric,
                                  severity=alert.severity,
                                  message=alert.message)
        
        # Trim alert history
        if len(self.alert_history) > self.max_alert_history:
            self.alert_history = self.alert_history[-self.max_alert_history:]
    
    async def _cleanup_loop(self):
        """Periodic cleanup of old data"""
        while True:
            try:
                await asyncio.sleep(300)  # Cleanup every 5 minutes
                
                current_time = time.time()
                cutoff_time = current_time - (self.window_size * 2)  # Keep 2x window size
                
                # Clean up old data
                for deque_obj in [self.request_times, self.request_counts, self.error_counts,
                                self.connection_counts, self.cpu_usage, self.memory_usage]:
                    while deque_obj and deque_obj[0][0] < cutoff_time:
                        deque_obj.popleft()
                        
            except asyncio.CancelledError:
                break
            except Exception as e:
                self.logger.exception("cleanup_error", error=str(e))
    
    async def _monitoring_loop(self):
        """Periodic performance monitoring and alerting"""
        while True:
            try:
                await asyncio.sleep(60)  # Check every minute
                
                # Get current metrics
                metrics = self.get_current_metrics()
                
                # Check thresholds
                alerts = self.check_thresholds(metrics)
                
                # Update alert status
                self.update_alert_status(alerts)
                
                # Log performance summary
                self.logger.info("performance_summary",
                               requests_per_second=metrics.requests_per_second,
                               p95_response_time=metrics.p95_response_time,
                               error_rate=metrics.error_rate,
                               active_connections=metrics.active_connections,
                               cpu_usage=metrics.cpu_usage,
                               memory_usage=metrics.memory_usage,
                               active_alerts=len(self.active_alerts))
                
            except asyncio.CancelledError:
                break
            except Exception as e:
                self.logger.exception("monitoring_loop_error", error=str(e))
    
    def get_active_alerts(self) -> List[PerformanceAlert]:
        """Get active performance alerts"""
        return list(self.active_alerts.values())
    
    def get_alert_history(self, limit: int = 100) -> List[PerformanceAlert]:
        """Get alert history"""
        return self.alert_history[-limit:]
    
    def update_threshold(self, metric: str, threshold: float):
        """Update performance threshold"""
        if metric in self.thresholds:
            old_threshold = self.thresholds[metric]
            self.thresholds[metric] = threshold
            self.logger.info("threshold_updated",
                           metric=metric,
                           old_threshold=old_threshold,
                           new_threshold=threshold)
    
    def get_performance_summary(self, hours: int = 24) -> Dict[str, Any]:
        """Get performance summary for the last N hours"""
        current_time = time.time()
        window_start = current_time - (hours * 3600)
        
        # Filter data
        recent_requests = [(t, d) for t, d in self.request_times if t >= window_start]
        recent_errors = [(t, c) for t, c in self.error_counts if t >= window_start]
        
        # Calculate summary
        if recent_requests:
            response_times = [d for _, d in recent_requests]
            total_requests = len(recent_requests)
            avg_response_time = statistics.mean(response_times)
            
            if len(response_times) > 1:
                p95_response_time = statistics.quantiles(response_times, n=20)[18]
                p99_response_time = statistics.quantiles(response_times, n=100)[98]
            else:
                p95_response_time = p99_response_time = avg_response_time
        else:
            total_requests = 0
            avg_response_time = p95_response_time = p99_response_time = 0.0
        
        total_errors = sum(c for _, c in recent_errors)
        error_rate = (total_errors / (total_requests + total_errors)) if (total_requests + total_errors) > 0 else 0.0
        
        # Calculate requests per hour
        requests_per_hour = total_requests / hours if hours > 0 else 0
        
        return {
            "period_hours": hours,
            "total_requests": total_requests,
            "requests_per_hour": requests_per_hour,
            "avg_response_time": avg_response_time,
            "p95_response_time": p95_response_time,
            "p99_response_time": p99_response_time,
            "total_errors": total_errors,
            "error_rate": error_rate,
            "alert_count": len([a for a in self.alert_history 
                              if a.timestamp >= datetime.fromtimestamp(window_start)])
        }


# Global performance tracker
_performance_tracker: Optional[PerformanceTracker] = None


async def setup_performance_monitoring():
    """Setup global performance monitoring"""
    global _performance_tracker
    
    _performance_tracker = PerformanceTracker(window_size=300)  # 5-minute window
    
    logger.info("performance_monitoring_initialized")


def get_performance_tracker() -> PerformanceTracker:
    """Get global performance tracker"""
    if _performance_tracker is None:
        raise RuntimeError("Performance monitoring not initialized")
    return _performance_tracker


# Performance decorators
def track_performance(metric_name: str):
    """Decorator for tracking function performance"""
    def decorator(func):
        async def wrapper(*args, **kwargs):
            tracker = get_performance_tracker()
            start_time = time.time()
            
            try:
                result = await func(*args, **kwargs)
                duration = time.time() - start_time
                tracker.record_request(duration)
                return result
            except Exception as e:
                duration = time.time() - start_time
                tracker.record_request(duration)
                tracker.record_error()
                raise
                
        return wrapper
    return decorator
```

### 3.5 Alerting System (`src/monitoring/alerts.py`)
```python
"""
Alerting System
Multi-channel alerting with escalation and suppression
"""

import asyncio
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from typing import Dict, Any, List, Optional, Callable, Awaitable
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
import json
import aiohttp
import ssl

from ..utils.logging import get_logger
from ..config import get_config

logger = get_logger(__name__)


class AlertSeverity(Enum):
    """Alert severity levels"""
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"
    EMERGENCY = "emergency"


class AlertStatus(Enum):
    """Alert status"""
    ACTIVE = "active"
    SUPPRESSED = "suppressed"
    RESOLVED = "resolved"
    ESCALATED = "escalated"


@dataclass
class AlertRule:
    """Alert rule definition"""
    name: str
    condition: str
    severity: AlertSeverity
    message_template: str
    channels: List[str]
    cooldown_minutes: int = 60
    escalation_minutes: int = 30
    auto_resolve: bool = True
    enabled: bool = True
    metadata: Dict[str, Any] = field(default_factory=dict)


@dataclass
class Alert:
    """Alert instance"""
    id: str
    rule_name: str
    severity: AlertSeverity
    message: str
    status: AlertStatus
    created_at: datetime
    updated_at: datetime
    metadata: Dict[str, Any] = field(default_factory=dict)
    resolved_at: Optional[datetime] = None
    escalated_at: Optional[datetime] = None
    suppressed_until: Optional[datetime] = None
    notification_count: int = 0
    last_notification_at: Optional[datetime] = None


class NotificationChannel(ABC):
    """Base class for notification channels"""
    
    def __init__(self, name: str, config: Dict[str, Any]):
        self.name = name
        self.config = config
        self.enabled = config.get("enabled", True)
        self.rate_limit_minutes = config.get("rate_limit_minutes", 5)
        self._last_notification: Optional[datetime] = None
    
    @abstractmethod
    async def send_notification(self, alert: Alert, context: Dict[str, Any]) -> bool:
        """Send notification for alert"""
        pass
    
    def can_send_notification(self) -> bool:
        """Check if channel can send notification (rate limiting)"""
        if not self.enabled:
            return False
        
        if self._last_notification is None:
            return True
        
        time_since_last = datetime.utcnow() - self._last_notification
        return time_since_last.total_seconds() >= (self.rate_limit_minutes * 60)
    
    def record_notification(self):
        """Record that notification was sent"""
        self._last_notification = datetime.utcnow()


class EmailNotificationChannel(NotificationChannel):
    """Email notification channel"""
    
    async def send_notification(self, alert: Alert, context: Dict[str, Any]) -> bool:
        """Send email notification"""
        if not self.can_send_notification():
            return False
        
        try:
            smtp_config = self.config.get("smtp", {})
            smtp_host = smtp_config.get("host", "localhost")
            smtp_port = smtp_config.get("port", 587)
            smtp_username = smtp_config.get("username")
            smtp_password = smtp_config.get("password")
            use_tls = smtp_config.get("use_tls", True)
            
            # Create message
            msg = MIMEMultipart("alternative")
            msg["Subject"] = f"[{alert.severity.value.upper()}] MCP Server Alert: {alert.rule_name}"
            msg["From"] = self.config.get("from_address", "alerts@mcp-server.local")
            msg["To"] = ", ".join(self.config.get("to_addresses", []))
            
            # Create email body
            text_body = self._create_text_body(alert, context)
            html_body = self._create_html_body(alert, context)
            
            msg.attach(MIMEText(text_body, "plain"))
            msg.attach(MIMEText(html_body, "html"))
            
            # Send email
            context = ssl.create_default_context() if use_tls else None
            
            with smtplib.SMTP(smtp_host, smtp_port) as server:
                if use_tls:
                    server.starttls(context=context)
                
                if smtp_username and smtp_password:
                    server.login(smtp_username, smtp_password)
                
                server.send_message(msg)
            
            self.record_notification()
            logger.info("email_notification_sent", alert_id=alert.id, channel=self.name)
            return True
            
        except Exception as e:
            logger.error("email_notification_failed", alert_id=alert.id, error=str(e))
            return False
    
    def _create_text_body(self, alert: Alert, context: Dict[str, Any]) -> str:
        """Create plain text email body"""
        body = f"""
MCP Server Alert Notification
============================

Alert ID: {alert.id}
Rule: {alert.rule_name}
Severity: {alert.severity.value.upper()}
Status: {alert.status.value}
Created: {alert.created_at.strftime('%Y-%m-%d %H:%M:%S UTC')}

Message:
{alert.message}

Context:
{json.dumps(context, indent=2)}

Server Information:
- Environment: {context.get('environment', 'unknown')}
- Version: {context.get('version', 'unknown')}
- Uptime: {context.get('uptime', 'unknown')}

This alert was generated automatically by the MCP Server monitoring system.
"""
        return body.strip()
    
    def _create_html_body(self, alert: Alert, context: Dict[str, Any]) -> str:
        """Create HTML email body"""
        severity_colors = {
            "info": "#17a2b8",
            "warning": "#ffc107",
            "critical": "#dc3545",
            "emergency": "#721c24"
        }
        
        color = severity_colors.get(alert.severity.value, "#6c757d")
        
        html = f"""
<!DOCTYPE html>
<html>
<head>
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        .alert-header {{ background-color: {color}; color: white; padding: 15px; border-radius: 5px; }}
        .alert-content {{ margin: 20px 0; }}
        .context {{ background-color: #f8f9fa; padding: 15px; border-radius: 5px; font-family: monospace; }}
        .footer {{ margin-top: 30px; font-size: 12px; color: #6c757d; }}
    </style>
</head>
<body>
    <div class="alert-header">
        <h2>MCP Server Alert</h2>
        <p><strong>{alert.rule_name}</strong> - {alert.severity.value.upper()}</p>
    </div>
    
    <div class="alert-content">
        <p><strong>Alert ID:</strong> {alert.id}</p>
        <p><strong>Status:</strong> {alert.status.value}</p>
        <p><strong>Created:</strong> {alert.created_at.strftime('%Y-%m-%d %H:%M:%S UTC')}</p>
        
        <h3>Message</h3>
        <p>{alert.message}</p>
        
        <h3>Context</h3>
        <div class="context">
            <pre>{json.dumps(context, indent=2)}</pre>
        </div>
    </div>
    
    <div class="footer">
        <p>This alert was generated automatically by the MCP Server monitoring system.</p>
        <p>Environment: {context.get('environment', 'unknown')} | Version: {context.get('version', 'unknown')}</p>
    </div>
</body>
</html>
"""
        return html


class SlackNotificationChannel(NotificationChannel):
    """Slack notification channel"""
    
    async def send_notification(self, alert: Alert, context: Dict[str, Any]) -> bool:
        """Send Slack notification"""
        if not self.can_send_notification():
            return False
        
        try:
            webhook_url = self.config.get("webhook_url")
            if not webhook_url:
                logger.error("slack_webhook_url_not_configured")
                return False
            
            # Create Slack message
            color_map = {
                "info": "good",
                "warning": "warning",
                "critical": "danger",
                "emergency": "danger"
            }
            
            payload = {
                "attachments": [
                    {
                        "color": color_map.get(alert.severity.value, "warning"),
                        "title": f"MCP Server Alert: {alert.rule_name}",
                        "text": alert.message,
                        "fields": [
                            {
                                "title": "Severity",
                                "value": alert.severity.value.upper(),
                                "short": True
                            },
                            {
                                "title": "Status",
                                "value": alert.status.value,
                                "short": True
                            },
                            {
                                "title": "Alert ID",
                                "value": alert.id,
                                "short": True
                            },
                            {
                                "title": "Created",
                                "value": alert.created_at.strftime('%Y-%m-%d %H:%M:%S UTC'),
                                "short": True
                            }
                        ],
                        "footer": "MCP Server Monitoring",
                        "ts": int(alert.created_at.timestamp())
                    }
                ]
            }
            
            # Send to Slack
            async with aiohttp.ClientSession() as session:
                async with session.post(webhook_url, json=payload) as response:
                    if response.status == 200:
                        self.record_notification()
                        logger.info("slack_notification_sent", alert_id=alert.id)
                        return True
                    else:
                        logger.error("slack_notification_failed", alert_id=alert.id, status=response.status)
                        return False
                        
        except Exception as e:
            logger.error("slack_notification_error", alert_id=alert.id, error=str(e))
            return False


class WebhookNotificationChannel(NotificationChannel):
    """Generic webhook notification channel"""
    
    async def send_notification(self, alert: Alert, context: Dict[str, Any]) -> bool:
        """Send webhook notification"""
        if not self.can_send_notification():
            return False
        
        try:
            webhook_url = self.config.get("url")
            method = self.config.get("method", "POST")
            headers = self.config.get("headers", {})
            timeout = self.config.get("timeout", 30)
            
            payload = {
                "alert": {
                    "id": alert.id,
                    "rule_name": alert.rule_name,
                    "severity": alert.severity.value,
                    "message": alert.message,
                    "status": alert.status.value,
                    "created_at": alert.created_at.isoformat(),
                    "metadata": alert.metadata
                },
                "context": context,
                "timestamp": datetime.utcnow().isoformat()
            }
            
            async with aiohttp.ClientSession() as session:
                async with session.request(method, webhook_url, 
                                         json=payload, headers=headers, timeout=timeout) as response:
                    if 200 <= response.status < 300:
                        self.record_notification()
                        logger.info("webhook_notification_sent", alert_id=alert.id, url=webhook_url)
                        return True
                    else:
                        logger.error("webhook_notification_failed", 
                                   alert_id=alert.id, 
                                   status=response.status, 
                                   url=webhook_url)
                        return False
                        
        except Exception as e:
            logger.error("webhook_notification_error", alert_id=alert.id, error=str(e))
            return False


class AlertingManager:
    """Manages alerting system"""
    
    def __init__(self):
        self.config = get_config()
        self.channels: Dict[str, NotificationChannel] = {}
        self.rules: Dict[str, AlertRule] = {}
        self.active_alerts: Dict[str, Alert] = {}
        self.alert_history: List[Alert] = []
        self.max_history_size = 10000
        
        # Rate limiting
        self.global_rate_limit_minutes = 1
        self._last_global_notification: Optional[datetime] = None
        
        # Start background tasks
        self._escalation_task = asyncio.create_task(self._escalation_loop())
        self._cleanup_task = asyncio.create_task(self._cleanup_loop())
    
    def register_channel(self, channel: NotificationChannel):
        """Register notification channel"""
        self.channels[channel.name] = channel
        logger.info("notification_channel_registered", name=channel.name)
    
    def unregister_channel(self, name: str):
        """Unregister notification channel"""
        if name in self.channels:
            del self.channels[name]
            logger.info("notification_channel_unregistered", name=name)
    
    def add_rule(self, rule: AlertRule):
        """Add alert rule"""
        self.rules[rule.name] = rule
        logger.info("alert_rule_added", name=rule.name)
    
    def remove_rule(self, name: str):
        """Remove alert rule"""
        if name in self.rules:
            del self.rules[name]
            logger.info("alert_rule_removed", name=name)
    
    async def trigger_alert(self, rule_name: str, message: str, context: Dict[str, Any]):
        """Trigger alert based on rule"""
        if rule_name not in self.rules:
            logger.warning("alert_rule_not_found", rule_name=rule_name)
            return
        
        rule = self.rules[rule_name]
        
        # Check if alert is suppressed
        alert_id = f"{rule_name}_{context.get('alert_id', '')}"
        if alert_id in self.active_alerts:
            alert = self.active_alerts[alert_id]
            if alert.suppressed_until and alert.suppressed_until > datetime.utcnow():
                logger.info("alert_suppressed", alert_id=alert_id)
                return
        
        # Create new alert
        alert = Alert(
            id=alert_id,
            rule_name=rule.name,
            severity=rule.severity,
            message=message,
            status=AlertStatus.ACTIVE,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow(),
            metadata=context,
            resolved_at=None,
            escalated_at=None,
            suppressed_until=None,
            notification_count=0,
            last_notification_at=None
        )
        
        self.active_alerts[alert_id] = alert
        logger.info("alert_triggered", alert_id=alert_id, rule_name=rule_name)
        
        # Send notifications
        await self.send_alert_notifications(alert, context)
    
    async def send_alert_notifications(self, alert: Alert, context: Dict[str, Any]):
        """Send notifications for alert"""
        if not self.can_send_global_notification():
            logger.info("global_rate_limit_skipped", alert_id=alert.id)
            return
        
        rule = self.rules.get(alert.rule_name)
        if not rule:
            logger.error("alert_rule_missing", alert_id=alert.id)
            return
        
        channels = [self.channels.get(channel) for channel in rule.channels]
        channels = [channel for channel in channels if channel and channel.enabled]
        
        if not channels:
            logger.warning("no_channels_for_alert", alert_id=alert.id)
            return
        
        tasks = [channel.send_notification(alert, context) for channel in channels]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        for channel, result in zip(channels, results):
            if isinstance(result, bool) and result:
                alert.notification_count += 1
                alert.last_notification_at = datetime.utcnow()
                logger.info("notification_sent", alert_id=alert.id, channel=channel.name)
            else:
                logger.error("notification_failed", alert_id=alert.id, channel=channel.name, error=str(result))
        
        # Update alert status
        alert.updated_at = datetime.utcnow()
        self.active_alerts[alert.id] = alert
    
    def can_send_global_notification(self) -> bool:
        """Check global rate limit for notifications"""
        if self._last_global_notification is None:
            return True
        
        time_since_last = datetime.utcnow() - self._last_global_notification
        return time_since_last.total_seconds() >= (self.global_rate_limit_minutes * 60)
    
    def record_global_notification(self):
        """Record global notification"""
        self._last_global_notification = datetime.utcnow()
    
    async def _escalation_loop(self):
        """Periodic escalation of unresolved alerts"""
        while True:
            try:
                await asyncio.sleep(300)  # Check every 5 minutes
                
                now = datetime.utcnow()
                for alert in list(self.active_alerts.values()):
                    rule = self.rules.get(alert.rule_name)
                    if not rule:
                        continue
                    
                    if alert.status == AlertStatus.RESOLVED:
                        continue
                    
                    if alert.escalated_at and (now - alert.escalated_at).total_seconds() >= (rule.escalation_minutes * 60):
                        alert.status = AlertStatus.ESCALATED
                        alert.escalated_at = now
                        logger.info("alert_escalated", alert_id=alert.id)
                        
                        # Re-send notifications
                        await self.send_alert_notifications(alert, alert.metadata)
                        
            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.exception("escalation_loop_error", error=str(e))
    
    async def _cleanup_loop(self):
        """Periodic cleanup of old alerts"""
        while True:
            try:
                await asyncio.sleep(3600)  # Cleanup every hour
                
                now = datetime.utcnow()
                cutoff = now - timedelta(days=30)  # Keep alerts for 30 days
                
                # Remove old alerts from history
                self.alert_history = [alert for alert in self.alert_history if alert.created_at >= cutoff]
                
                # Remove resolved alerts from active list
                for alert_id, alert in list(self.active_alerts.items()):
                    if alert.status == AlertStatus.RESOLVED and alert.resolved_at and alert.resolved_at < cutoff:
                        del self.active_alerts[alert_id]
                        logger.info("resolved_alert_removed", alert_id=alert_id)
                        
            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.exception("cleanup_loop_error", error=str(e))
    
    def get_active_alerts(self) -> List[Alert]:
        """Get active alerts"""
        return list(self.active_alerts.values())
    
    def get_alert_history(self, limit: int = 1000) -> List[Alert]:
        """Get alert history"""
        return self.alert_history[-limit:]
    
    def resolve_alert(self, alert_id: str):
        """Resolve alert"""
        if alert_id in self.active_alerts:
            alert = self.active_alerts[alert_id]
            alert.status = AlertStatus.RESOLVED
            alert.resolved_at = datetime.utcnow()
            logger.info("alert_resolved", alert_id=alert_id)
            self.alert_history.append(alert)
            del self.active_alerts[alert_id]
    
    def suppress_alert(self, alert_id: str, duration_minutes: int):
        """Suppress alert for specified duration"""
        if alert_id in self.active_alerts:
            alert = self.active_alerts[alert_id]
            alert.suppressed_until = datetime.utcnow() + timedelta(minutes=duration_minutes)
            logger.info("alert_suppressed", alert_id=alert_id, duration_minutes=duration_minutes)


# Predefined alert rules
def create_standard_alert_rules() -> List[AlertRule]:
    """Create standard alert rules"""
    return [
        AlertRule(
            name="high_error_rate",
            condition="error_rate > 0.05",
            severity=AlertSeverity.CRITICAL,
            message_template="Error rate exceeded 5%",
            channels=["email", "slack"],
            cooldown_minutes=30,
            escalation_minutes=60,
            auto_resolve=True,
            enabled=True
        ),
        AlertRule(
            name="high_response_time",
            condition="p95_response_time > 1.0",
            severity=AlertSeverity.WARNING,
            message_template="P95 response time exceeded 1 second",
            channels=["email"],
            cooldown_minutes=60,
            escalation_minutes=120,
            auto_resolve=True,
            enabled=True
        ),
        AlertRule(
            name="cpu_usage_high",
            condition="cpu_usage > 80.0",
            severity=AlertSeverity.WARNING,
            message_template="CPU usage exceeded 80%",
            channels=["email", "webhook"],
            cooldown_minutes=15,
            escalation_minutes=30,
            auto_resolve=True,
            enabled=True
        ),
        AlertRule(
            name="memory_usage_high",
            condition="memory_usage > 85.0",
            severity=AlertSeverity.WARNING,
            message_template="Memory usage exceeded 85%",
            channels=["email", "webhook"],
            cooldown_minutes=15,
            escalation_minutes=30,
            auto_resolve=True,
            enabled=True
        )
    ]


# Global alerting manager
_alerting_manager: Optional[AlertingManager] = None


async def setup_alerting():
    """Setup global alerting system"""
    global _alerting_manager
    
    _alerting_manager = AlertingManager()
    
    # Register channels
    email_channel = EmailNotificationChannel("email", {
        "enabled": True,
        "smtp": {
            "host": "smtp.example.com",
            "port": 587,
            "username": "alert@example.com",
            "password": "password",
            "use_tls": True
        },
        "from_address": "alerts@mcp-server.local",
        "to_addresses": ["admin@example.com"]
    })
    _alerting_manager.register_channel(email_channel)
    
    slack_channel = SlackNotificationChannel("slack", {
        "enabled": True,
        "webhook_url": "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
    })
    _alerting_manager.register_channel(slack_channel)
    
    webhook_channel = WebhookNotificationChannel("webhook", {
        "enabled": True,
        "url": "https://example.com/alerts",
        "method": "POST",
        "headers": {"Content-Type": "application/json"},
        "timeout": 30
    })
    _alerting_manager.register_channel(webhook_channel)
    
    # Add standard rules
    for rule in create_standard_alert_rules():
        _alerting_manager.add_rule(rule)
    
    logger.info("alerting_system_initialized")


def get_alerting_manager()
