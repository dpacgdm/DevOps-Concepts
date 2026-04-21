# Phase 9, Lesson 2: Python for DevOps

**Challenge accepted. No gaps. No excuses.**

---

## 1. PROJECT STRUCTURE — THE Foundation

Every Python tool you write at NovaMart follows this structure. Not optional. Not "when the project gets big enough." **From line one.**

```
novamart-tools/
├── pyproject.toml              # Modern packaging (replaces setup.py, setup.cfg)
├── Makefile                    # Developer workflow
├── README.md
├── .python-version             # Pin Python version (pyenv)
├── .pre-commit-config.yaml
├── .gitignore
│
├── src/                        # src layout (prevents accidental local imports)
│   └── novatools/
│       ├── __init__.py         # Package marker + version
│       ├── py.typed            # PEP 561: "this package has type hints"
│       │
│       ├── cli/                # CLI entry points
│       │   ├── __init__.py
│       │   ├── main.py         # Root CLI group
│       │   ├── cost.py         # novatools cost report
│       │   ├── incidents.py    # novatools incident create
│       │   └── secrets.py      # novatools secrets rotate
│       │
│       ├── aws/                # AWS service wrappers
│       │   ├── __init__.py
│       │   ├── ec2.py
│       │   ├── eks.py
│       │   ├── rds.py
│       │   ├── s3.py
│       │   └── session.py      # Centralized session/client factory
│       │
│       ├── integrations/       # Third-party API clients
│       │   ├── __init__.py
│       │   ├── slack.py
│       │   ├── pagerduty.py
│       │   ├── jira_client.py  # Don't name it jira.py — shadows the jira library
│       │   └── argocd.py
│       │
│       ├── k8s/                # Kubernetes operations
│       │   ├── __init__.py
│       │   ├── client.py
│       │   ├── pods.py
│       │   └── deployments.py
│       │
│       ├── models/             # Data classes / Pydantic models
│       │   ├── __init__.py
│       │   ├── incident.py
│       │   └── cost.py
│       │
│       ├── utils/              # Shared utilities
│       │   ├── __init__.py
│       │   ├── logging.py
│       │   ├── retry.py
│       │   ├── config.py
│       │   └── formatting.py
│       │
│       └── templates/          # Jinja2 templates
│           ├── incident_report.md.j2
│           ├── cost_report.html.j2
│           └── alert_config.yaml.j2
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # Shared fixtures
│   ├── unit/
│   │   ├── __init__.py
│   │   ├── test_cost.py
│   │   ├── test_incidents.py
│   │   └── aws/
│   │       ├── __init__.py
│   │       └── test_ec2.py
│   │
│   └── integration/
│       ├── __init__.py
│       └── test_slack.py
│
└── scripts/                    # Standalone scripts (not part of package)
    ├── migrate_secrets.py
    └── one_time_cleanup.py
```

**Why `src/` layout?**
```
# WITHOUT src/:
novatools/
  __init__.py
  cli.py
tests/
  test_cli.py

# Running: python -m pytest
# Python adds CWD to sys.path → imports from LOCAL novatools/ (source)
# NOT the installed package. Tests pass locally, fail in CI after install.
# You're testing DIFFERENT CODE than what ships.

# WITH src/:
src/novatools/
  __init__.py
  cli.py
tests/
  test_cli.py

# Python adds CWD to sys.path → no novatools/ in CWD
# Must install package (pip install -e .) to import
# Tests ALWAYS test the installed code. No surprises.
```

**`pyproject.toml` — The single source of truth:**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "novatools"
version = "1.4.0"
description = "NovaMart Platform Engineering CLI Tools"
readme = "README.md"
license = { text = "Proprietary" }
requires-python = ">=3.11"
authors = [{ name = "Platform Engineering", email = "platform@novamart.com" }]

dependencies = [
    "boto3>=1.34,<2",
    "click>=8.1,<9",
    "rich>=13.7,<14",
    "pydantic>=2.5,<3",
    "requests>=2.31,<3",
    "jinja2>=3.1,<4",
    "kubernetes>=28.1,<30",
    "pyyaml>=6.0,<7",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4",
    "pytest-cov>=4.1",
    "pytest-mock>=3.12",
    "moto[all]>=5.0",
    "mypy>=1.8",
    "ruff>=0.2",
    "responses>=0.25",
    "types-requests",
    "types-pyyaml",
    "boto3-stubs[essential]",
]

[project.scripts]
novatools = "novatools.cli.main:cli"

[tool.hatch.build.targets.wheel]
packages = ["src/novatools"]

# ─── Ruff (replaces flake8 + isort + black in one tool) ───
[tool.ruff]
target-version = "py311"
line-length = 120
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "T20",  # flake8-print (no print() in production code)
    "RUF",  # ruff-specific rules
    "S",    # flake8-bandit (security)
    "PTH",  # flake8-use-pathlib
]
ignore = [
    "S101",  # Allow assert in tests
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "T20"]    # Allow assert and print in tests
"scripts/**" = ["T20"]          # Allow print in scripts

# ─── Mypy ──────────────────────────────────────────────────
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = ["kubernetes.*", "moto.*"]
ignore_missing_imports = true

# ─── Pytest ────────────────────────────────────────────────
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "--strict-markers",
    "--tb=short",
    "-v",
    "--cov=novatools",
    "--cov-report=term-missing",
    "--cov-fail-under=80",
]
markers = [
    "integration: marks tests requiring external services",
    "slow: marks tests that take > 5 seconds",
]
```

**Key decisions:**
- `hatchling` build backend (modern, fast, PEP 621 native)
- `ruff` replaces flake8 + isort + black (10-100x faster, single tool)
- Strict mypy (catches bugs before they reach production)
- `[project.scripts]` creates the CLI entry point on `pip install`
- Version bounds: `>=X,<Y` (compatible range, not exact pins — pins go in lock files)

**Makefile for the project:**

```makefile
.PHONY: help install dev lint type-check test test-unit test-integration ci clean

help: ## Show help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

install: ## Install package
	pip install -e .

dev: ## Install with dev dependencies
	pip install -e ".[dev]"
	pre-commit install

lint: ## Run linter
	ruff check src/ tests/
	ruff format --check src/ tests/

format: ## Auto-format code
	ruff format src/ tests/
	ruff check --fix src/ tests/

type-check: ## Run mypy
	mypy src/

test: ## Run all tests
	pytest

test-unit: ## Run unit tests only
	pytest tests/unit/

test-integration: ## Run integration tests
	pytest tests/integration/ -m integration

ci: lint type-check test ## Run all CI checks

clean: ## Clean artifacts
	rm -rf dist/ build/ *.egg-info .pytest_cache .mypy_cache .ruff_cache htmlcov/
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
```

---

## 2. TYPE HINTS — Non-Negotiable at Senior Level

```python
"""Type hints are not optional in production Python. Period.

They catch bugs at lint time, serve as documentation, and enable IDE 
autocompletion. Senior engineers write typed Python. Junior engineers 
argue about whether types are "Pythonic."
"""

from __future__ import annotations  # PEP 604: Enable X | Y syntax in older Python

import datetime
from collections.abc import Callable, Generator, Sequence
from pathlib import Path
from typing import Any, TypeAlias, TypedDict, TypeVar

# ── Basic types ──────────────────────────────────────────
def get_pod_name(namespace: str, label: str) -> str: ...
def count_pods(namespace: str) -> int: ...
def is_healthy(pod_name: str) -> bool: ...
def get_tags(resource_arn: str) -> dict[str, str]: ...
def list_namespaces() -> list[str]: ...
def get_config() -> dict[str, Any]: ...  # Any when truly dynamic

# ── Optional (value or None) ────────────────────────────
def find_pod(name: str) -> str | None:  # Python 3.10+ syntax
    """Returns pod name or None if not found."""
    ...

# ── Collections ─────────────────────────────────────────
def process_services(services: list[str]) -> dict[str, bool]:
    """Process services, return {name: success}."""
    results: dict[str, bool] = {}
    for svc in services:
        results[svc] = deploy(svc)
    return results

# Use Sequence for read-only (accepts list, tuple, etc.)
def validate_images(images: Sequence[str]) -> list[str]:
    """Accept any sequence, return list of invalid images."""
    ...

# ── TypedDict (structured dicts, common for API responses) ──
class PodStatus(TypedDict):
    name: str
    namespace: str
    phase: str
    restarts: int
    ready: bool
    node: str | None

class PodStatusPartial(TypedDict, total=False):
    """All fields optional — for partial updates."""
    phase: str
    restarts: int

def get_pod_status(name: str, namespace: str) -> PodStatus:
    ...

# ── Callables (functions as arguments) ──────────────────
def retry(
    func: Callable[..., Any],
    max_attempts: int = 3,
    delay: float = 1.0,
) -> Any:
    ...

# More specific callable:
HealthCheck = Callable[[str, str], bool]  # (pod_name, namespace) -> bool

def run_checks(checks: list[HealthCheck]) -> list[bool]:
    ...

# ── Generics (TypeVar) ──────────────────────────────────
T = TypeVar("T")

def first_or_default(items: list[T], default: T) -> T:
    """Return first item or default. Type-safe."""
    return items[0] if items else default

# ── Type aliases for readability ────────────────────────
ServiceName: TypeAlias = str
Namespace: TypeAlias = str
ImageTag: TypeAlias = str
AWSRegion: TypeAlias = str
PodMap: TypeAlias = dict[ServiceName, list[PodStatus]]

def get_all_pods(region: AWSRegion, cluster: str) -> PodMap:
    ...

# ── Generator types ─────────────────────────────────────
def stream_logs(pod: str, namespace: str) -> Generator[str, None, None]:
    """Yields log lines."""
    ...

# ── Pydantic models (runtime validation + types) ────────
from pydantic import BaseModel, Field, field_validator

class DeploymentRequest(BaseModel):
    """Validated deployment configuration."""
    service: str = Field(min_length=1, max_length=63, pattern=r"^[a-z][a-z0-9-]*$")
    version: str = Field(pattern=r"^sha-[a-f0-9]{7,40}$")
    environment: str = Field(pattern=r"^(dev|staging|production)$")
    replicas: int = Field(ge=1, le=100, default=3)
    canary_steps: list[int] = Field(default=[5, 25, 50, 100])
    timeout_seconds: int = Field(ge=30, le=3600, default=300)
    notify_slack: bool = True

    @field_validator("canary_steps")
    @classmethod
    def validate_canary_steps(cls, v: list[int]) -> list[int]:
        if v[-1] != 100:
            raise ValueError("Last canary step must be 100")
        if v != sorted(v):
            raise ValueError("Canary steps must be ascending")
        return v

# Usage:
req = DeploymentRequest(
    service="payment-service",
    version="sha-abc1234",
    environment="production",
)
# Pydantic validates ALL fields at construction time.
# Invalid data → ValidationError with precise field-level messages.
# This replaces hundreds of lines of manual if/else validation.

# ── Why Pydantic > TypedDict for external data ──────────
# TypedDict: checked by mypy at lint time only. No runtime validation.
# Pydantic: validated at RUNTIME. Use for API responses, configs, user input.
# Rule: TypedDict for internal data shapes. Pydantic for external boundaries.
```

**How types break (and why `strict = true` in mypy matters):**

```python
# WITHOUT type checking:
def get_instances(region):
    client = boto3.client("ec2", region_name=region)
    response = client.describe_instances()
    return response["Reservations"]  # Returns list[dict]... of what shape?

# Caller:
instances = get_instances("us-east-1")
for i in instances:
    print(i["InstanceId"])  # RUNTIME KeyError — it's i["Instances"][0]["InstanceId"]
    # You find this at 2 AM, not at code review.

# WITH type checking:
def get_instances(region: str) -> list[dict[str, Any]]:
    client = boto3.client("ec2", region_name=region)
    response = client.describe_instances()
    return response["Reservations"]

# Better with boto3-stubs:
# pip install boto3-stubs[ec2]
from mypy_boto3_ec2.type_defs import ReservationTypeDef
def get_instances(region: str) -> list[ReservationTypeDef]:
    ...
# Now mypy knows the exact shape. Typos caught at lint time.
```

---

## 3. LOGGING — Production Python Logging

```python
"""
src/novatools/utils/logging.py

Python's logging module is powerful but has traps.
This module configures it correctly for CLI tools and services.
"""
import logging
import sys
from typing import Any

import structlog  # pip install structlog — structured logging

# ─── Standard Library Logging (when you can't add deps) ───
def setup_stdlib_logging(
    level: str = "INFO",
    json_output: bool = False,
) -> logging.Logger:
    """Configure stdlib logging for CLI tools."""
    logger = logging.getLogger("novatools")
    logger.setLevel(getattr(logging, level.upper()))

    # Don't add handlers if they already exist (double-logging trap)
    if logger.handlers:
        return logger

    handler = logging.StreamHandler(sys.stderr)

    if json_output:
        # JSON format for machine consumption (log aggregators)
        formatter = logging.Formatter(
            '{"timestamp":"%(asctime)s","level":"%(levelname)s",'
            '"logger":"%(name)s","message":"%(message)s"}'
        )
    else:
        # Human-readable for terminal
        formatter = logging.Formatter(
            "%(asctime)s [%(levelname)-5s] %(name)s: %(message)s",
            datefmt="%Y-%m-%dT%H:%M:%SZ",
        )

    handler.setFormatter(formatter)
    logger.addHandler(handler)

    # Silence noisy libraries
    logging.getLogger("boto3").setLevel(logging.WARNING)
    logging.getLogger("botocore").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)
    logging.getLogger("kubernetes").setLevel(logging.WARNING)

    return logger


# ─── Structlog (production-grade — use this) ──────────────
def setup_structlog(level: str = "INFO", json_output: bool = False) -> None:
    """
    Configure structlog for structured, context-rich logging.
    
    Why structlog > stdlib logging:
    - Automatic context binding (request_id, service, environment)
    - Native JSON output (no string formatting)
    - Immutable log context (thread-safe)
    - Processors pipeline (add timestamp, level, caller, etc.)
    """
    shared_processors: list[structlog.types.Processor] = [
        structlog.contextvars.merge_contextvars,  # Merge async context
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.UnicodeDecoder(),
    ]

    if json_output:
        # Production: JSON to stdout for Fluent Bit / Loki
        renderer = structlog.processors.JSONRenderer()
    else:
        # Development: colored, human-readable
        renderer = structlog.dev.ConsoleRenderer(colors=True)

    structlog.configure(
        processors=[
            *shared_processors,
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            renderer,
        ],
    )

    handler = logging.StreamHandler(sys.stderr)
    handler.setFormatter(formatter)

    root_logger = logging.getLogger()
    root_logger.addHandler(handler)
    root_logger.setLevel(getattr(logging, level.upper()))

    # Silence noisy libraries
    for noisy in ["boto3", "botocore", "urllib3", "kubernetes", "httpcore"]:
        logging.getLogger(noisy).setLevel(logging.WARNING)


# ─── Usage ─────────────────────────────────────────────────
# In your tool:
import structlog

log = structlog.get_logger()

def deploy_service(service: str, version: str, env: str) -> None:
    # Bind context that appears in ALL subsequent log lines
    log_ctx = log.bind(service=service, version=version, environment=env)

    log_ctx.info("starting_deployment")
    # {"timestamp":"2024-01-15T14:30:00Z","level":"info","service":"payment-service",
    #  "version":"sha-abc123","environment":"production","event":"starting_deployment"}

    try:
        update_image(service, version)
        log_ctx.info("image_updated")
        wait_for_rollout(service, env)
        log_ctx.info("deployment_complete", duration_seconds=45.2)
    except Exception:
        log_ctx.exception("deployment_failed")  # Includes full traceback
        raise
```

**Logging failure modes:**

```python
# FAILURE 1: Double logging (most common)
logger = logging.getLogger("myapp")
logger.addHandler(logging.StreamHandler())  # Called twice → two handlers
# Every log line prints TWICE. Check: if logger.handlers before adding.

# FAILURE 2: Root logger pollution
logging.basicConfig(level=logging.DEBUG)  # Configures ROOT logger
# Now EVERY library's debug logs appear. boto3 alone generates thousands.
# FIX: Configure specific loggers, silence libraries.

# FAILURE 3: Credential leak in logs
log.info("Connecting to DB", password=db_password)  # Password in plain text!
# FIX: Never log secrets. Use structlog processors to redact:
def redact_secrets(logger, method, event_dict):
    for key in list(event_dict.keys()):
        if any(s in key.lower() for s in ["password", "secret", "token", "key"]):
            event_dict[key] = "***REDACTED***"
    return event_dict

# FAILURE 4: f-string in log call (performance waste)
log.debug(f"Processing {expensive_function()}")
# expensive_function() runs even if debug logging is disabled!
# FIX: Use lazy formatting:
log.debug("Processing %s", expensive_function())  # stdlib
log.debug("processing", result=expensive_function())  # structlog — same issue
# For structlog, wrap expensive computations in a check:
if log.isEnabledFor(logging.DEBUG):
    log.debug("processing", result=expensive_function())

# FAILURE 5: Missing exception context
try:
    risky_operation()
except Exception as e:
    log.error(f"Failed: {e}")  # LOST THE TRACEBACK
    # FIX:
    log.exception("Failed")    # stdlib: logs traceback automatically
    log.exception("failed")    # structlog: same
    # OR:
    log.error("failed", exc_info=True)  # Explicit traceback
```

---

## 4. ERROR HANDLING AND RETRY PATTERNS

```python
"""
src/novatools/utils/retry.py

Production retry with exponential backoff, jitter, and selective retries.
"""
import functools
import random
import time
from collections.abc import Callable
from typing import Any, TypeVar

import structlog

log = structlog.get_logger()

F = TypeVar("F", bound=Callable[..., Any])


class RetryExhausted(Exception):
    """All retry attempts failed."""

    def __init__(self, attempts: int, last_exception: Exception) -> None:
        self.attempts = attempts
        self.last_exception = last_exception
        super().__init__(f"Failed after {attempts} attempts: {last_exception}")


def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True,
    retryable_exceptions: tuple[type[Exception], ...] = (Exception,),
    non_retryable_exceptions: tuple[type[Exception], ...] = (),
    on_retry: Callable[[int, Exception, float], None] | None = None,
) -> Callable[[F], F]:
    """
    Decorator: retry with exponential backoff + jitter.

    Usage:
        @retry(max_attempts=5, retryable_exceptions=(ConnectionError, TimeoutError))
        def call_api(url: str) -> dict:
            ...
    """
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_exception: Exception | None = None

            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except non_retryable_exceptions:
                    raise  # Never retry these
                except retryable_exceptions as e:
                    last_exception = e

                    if attempt == max_attempts:
                        raise RetryExhausted(attempt, e) from e

                    # Calculate delay
                    delay = min(
                        base_delay * (exponential_base ** (attempt - 1)),
                        max_delay,
                    )
                    if jitter:
                        delay = delay * (0.5 + random.random())  # 50%-150% of delay

                    log.warning(
                        "retry_attempt",
                        function=func.__name__,
                        attempt=attempt,
                        max_attempts=max_attempts,
                        delay_seconds=round(delay, 2),
                        error=str(e),
                        error_type=type(e).__name__,
                    )

                    if on_retry:
                        on_retry(attempt, e, delay)

                    time.sleep(delay)

            # Should never reach here, but safety net
            assert last_exception is not None
            raise RetryExhausted(max_attempts, last_exception)

        return wrapper  # type: ignore[return-value]
    return decorator


# ─── Retry helper for non-decorated calls ─────────────────
def retry_call(
    func: Callable[..., Any],
    args: tuple[Any, ...] = (),
    kwargs: dict[str, Any] | None = None,
    max_attempts: int = 3,
    base_delay: float = 1.0,
    retryable_exceptions: tuple[type[Exception], ...] = (Exception,),
) -> Any:
    """Retry a function call without decorator."""
    kwargs = kwargs or {}

    @retry(
        max_attempts=max_attempts,
        base_delay=base_delay,
        retryable_exceptions=retryable_exceptions,
    )
    def _inner() -> Any:
        return func(*args, **kwargs)

    return _inner()


# ─── Custom exception hierarchy ───────────────────────────
class NovaToolsError(Exception):
    """Base exception for all NovaMart tools."""

class AWSError(NovaToolsError):
    """AWS API call failed."""
    def __init__(self, service: str, operation: str, message: str) -> None:
        self.service = service
        self.operation = operation
        super().__init__(f"AWS {service}.{operation}: {message}")

class DeploymentError(NovaToolsError):
    """Deployment operation failed."""

class ValidationError(NovaToolsError):
    """Input validation failed."""

class TimeoutError(NovaToolsError):
    """Operation timed out."""


# ─── Context manager for operation tracking ───────────────
import contextlib
from typing import Iterator

@contextlib.contextmanager
def operation_context(
    name: str,
    **context: Any,
) -> Iterator[None]:
    """
    Context manager that logs start/end/failure of an operation.
    
    Usage:
        with operation_context("deploy", service="payment", version="sha-abc"):
            do_deployment()
    """
    op_log = log.bind(operation=name, **context)
    op_log.info("operation_started")
    start = time.monotonic()

    try:
        yield
        duration = time.monotonic() - start
        op_log.info("operation_completed", duration_seconds=round(duration, 2))
    except Exception:
        duration = time.monotonic() - start
        op_log.exception("operation_failed", duration_seconds=round(duration, 2))
        raise
```

**Error handling patterns — the right way:**

```python
# ─── PATTERN 1: Catch specific, not broad ──────────────
# BAD:
try:
    response = requests.get(url)
    data = response.json()
    return data["results"]
except Exception:
    return []
# Swallows EVERYTHING: network errors, JSON parse errors, KeyError, 
# even KeyboardInterrupt in Python < 3.11. You'll never know it broke.

# GOOD:
try:
    response = requests.get(url, timeout=10)
    response.raise_for_status()
except requests.ConnectionError:
    log.error("connection_failed", url=url)
    raise AWSError("http", "get", f"Cannot connect to {url}")
except requests.Timeout:
    log.error("request_timeout", url=url)
    raise TimeoutError(f"Request to {url} timed out")
except requests.HTTPError as e:
    log.error("http_error", url=url, status=e.response.status_code)
    raise

try:
    data = response.json()
except ValueError:
    log.error("invalid_json", url=url, body=response.text[:200])
    raise ValidationError(f"Non-JSON response from {url}")

return data.get("results", [])  # .get() with default — no KeyError

# ─── PATTERN 2: Re-raise with context ──────────────────
try:
    db_connection = create_connection(host, port)
except ConnectionRefusedError as e:
    raise DeploymentError(
        f"Cannot connect to database at {host}:{port}. "
        f"Is RDS instance running? Check Security Groups."
    ) from e  # 'from e' preserves the original traceback chain

# ─── PATTERN 3: Multiple exception types ───────────────
from botocore.exceptions import ClientError, BotoCoreError

try:
    client.describe_instances()
except ClientError as e:
    error_code = e.response["Error"]["Code"]
    if error_code == "UnauthorizedAccess":
        raise AWSError("ec2", "describe_instances", "Check IAM permissions") from e
    elif error_code == "Throttling":
        raise  # Let retry decorator handle this
    else:
        raise AWSError("ec2", "describe_instances", str(e)) from e
except BotoCoreError as e:
    # SDK-level errors (config, credentials, network)
    raise AWSError("ec2", "describe_instances", f"SDK error: {e}") from e

# ─── PATTERN 4: Ensure cleanup (finally / contextmanager) ──
# GOOD — contextmanager pattern:
@contextlib.contextmanager
def temporary_security_group(ec2_client, vpc_id: str, name: str):
    """Create a temp SG, yield it, delete on exit."""
    sg = ec2_client.create_security_group(
        GroupName=name, Description="Temporary", VpcId=vpc_id
    )
    sg_id = sg["GroupId"]
    log.info("created_temp_sg", sg_id=sg_id)
    try:
        yield sg_id
    finally:
        ec2_client.delete_security_group(GroupId=sg_id)
        log.info("deleted_temp_sg", sg_id=sg_id)

# Usage:
with temporary_security_group(ec2, vpc_id, "temp-migration") as sg_id:
    # Do work with sg_id
    # Guaranteed cleanup even on exception
    pass
```

---

## 5. BOTO3 DEEP DIVE

This is the most-used Python library in DevOps. Let me cover it thoroughly.

```python
"""
src/novatools/aws/session.py

Centralized AWS session and client management.
"""
import functools
from typing import Any

import boto3
import botocore.config
import structlog
from botocore.exceptions import ClientError, NoCredentialsError

log = structlog.get_logger()

# ─── Session Factory ──────────────────────────────────────
# RULE: One session per thread. Sessions are NOT thread-safe.
# Clients created FROM a session are thread-safe for API calls.

_default_config = botocore.config.Config(
    retries={
        "max_attempts": 5,
        "mode": "adaptive",  # adaptive > standard > legacy
        # adaptive: dynamic rate limiting based on throttling
        # Backs off automatically on Throttling/TooManyRequests
    },
    connect_timeout=5,
    read_timeout=30,
    max_pool_connections=25,  # Connection pool per client
    # Default is 10 — too low for concurrent operations
)


def get_session(
    profile: str | None = None,
    region: str | None = None,
    role_arn: str | None = None,
) -> boto3.Session:
    """
    Create a boto3 session with optional role assumption.
    
    Priority: role_arn > profile > default chain
    Default chain: env vars > ~/.aws/credentials > instance profile/IRSA
    """
    if role_arn:
        # Cross-account or role assumption
        sts = boto3.client("sts")
        try:
            credentials = sts.assume_role(
                RoleArn=role_arn,
                RoleSessionName="novatools-session",
                DurationSeconds=3600,
            )["Credentials"]
        except ClientError as e:
            raise AWSError("sts", "assume_role", f"Cannot assume {role_arn}: {e}") from e

        return boto3.Session(
            aws_access_key_id=credentials["AccessKeyId"],
            aws_secret_access_key=credentials["SecretAccessKey"],
            aws_session_token=credentials["SessionToken"],
            region_name=region or "us-east-1",
        )

    return boto3.Session(
        profile_name=profile,
        region_name=region or "us-east-1",
    )


def get_client(
    service: str,
    session: boto3.Session | None = None,
    region: str | None = None,
    config: botocore.config.Config | None = None,
) -> Any:
    """Create a boto3 client with sensible defaults."""
    session = session or boto3.Session(region_name=region)
    return session.client(
        service,
        config=config or _default_config,
    )


# ─── Paginator (NEVER do manual pagination) ──────────────
def paginate(
    client: Any,
    method: str,
    result_key: str,
    **kwargs: Any,
) -> list[Any]:
    """
    Paginate ANY AWS API call automatically.
    
    Usage:
        instances = paginate(ec2, "describe_instances", "Reservations",
                           Filters=[{"Name": "tag:Environment", "Values": ["production"]}])
    """
    paginator = client.get_paginator(method)
    results: list[Any] = []

    for page in paginator.paginate(**kwargs):
        results.extend(page.get(result_key, []))

    return results

# Why this matters:
# BAD (silent data loss):
response = ec2.describe_instances()
instances = response["Reservations"]
# If you have > 1000 instances, this returns ONLY THE FIRST PAGE.
# You silently miss hundreds of instances. No error. No warning.
# This is one of the most common AWS SDK bugs in production.

# GOOD:
instances = paginate(ec2, "describe_instances", "Reservations")
# Gets ALL instances across ALL pages.


# ─── Waiters ──────────────────────────────────────────────
def wait_for(
    client: Any,
    waiter_name: str,
    delay: int = 15,
    max_attempts: int = 40,
    **kwargs: Any,
) -> None:
    """
    Wait for AWS resource state with configurable timeouts.
    
    Usage:
        wait_for(rds, "db_instance_available", 
                 DBInstanceIdentifier="novamart-payments")
    """
    waiter = client.get_waiter(waiter_name)
    log.info("waiting_for_state", waiter=waiter_name, max_wait=f"{delay * max_attempts}s")

    waiter.wait(
        WaiterConfig={"Delay": delay, "MaxAttempts": max_attempts},
        **kwargs,
    )
    log.info("state_reached", waiter=waiter_name)

# Common waiters:
# EC2: instance_running, instance_stopped, instance_terminated
# RDS: db_instance_available, db_instance_deleted, db_snapshot_available
# EKS: cluster_active, nodegroup_active
# ELB: load_balancer_available, target_in_service
# S3:  bucket_exists, object_exists


# ─── Error Handling Patterns ──────────────────────────────
def handle_client_error(
    service: str,
    operation: str,
    error: ClientError,
) -> None:
    """
    Standardized AWS error handling with actionable messages.
    """
    code = error.response["Error"]["Code"]
    message = error.response["Error"]["Message"]
    request_id = error.response["ResponseMetadata"]["RequestId"]

    error_map: dict[str, str] = {
        "AccessDeniedException": f"IAM permission denied. Check role policies for {service}:{operation}",
        "UnauthorizedAccess": f"IAM permission denied. Check role policies for {service}:{operation}",
        "Throttling": f"API rate limit hit. Increase retry config or reduce concurrency",
        "ResourceNotFoundException": f"Resource not found. Verify resource exists in target region",
        "ValidationException": f"Invalid parameters. Check API documentation for {service}.{operation}",
        "InvalidParameterValue": f"Invalid parameter value for {service}.{operation}",
        "ResourceInUseException": f"Resource is in use. Wait for current operation to complete",
    }

    actionable = error_map.get(code, f"Unhandled error code: {code}")

    log.error(
        "aws_api_error",
        service=service,
        operation=operation,
        error_code=code,
        message=message,
        request_id=request_id,
        action=actionable,
    )
```

### EC2 Operations — Real Production Code

```python
"""
src/novatools/aws/ec2.py
"""
from typing import Any

import boto3
import structlog
from botocore.exceptions import ClientError

from novatools.aws.session import get_client, paginate
from novatools.utils.retry import retry

log = structlog.get_logger()


class EC2Service:
    """EC2 operations for NovaMart infrastructure."""

    def __init__(self, session: boto3.Session | None = None, region: str = "us-east-1") -> None:
        self.client = get_client("ec2", session=session, region=region)
        self.region = region

    def get_instances(
        self,
        environment: str | None = None,
        service: str | None = None,
        state: str = "running",
    ) -> list[dict[str, Any]]:
        """
        Get EC2 instances with optional filters.
        Always paginates. Returns flattened instance list.
        """
        filters: list[dict[str, Any]] = [
            {"Name": "instance-state-name", "Values": [state]},
        ]
        if environment:
            filters.append({"Name": "tag:Environment", "Values": [environment]})
        if service:
            filters.append({"Name": "tag:Service", "Values": [service]})

        reservations = paginate(
            self.client, "describe_instances", "Reservations",
            Filters=filters,
        )

        # Flatten: Reservations → Instances
        instances: list[dict[str, Any]] = []
        for reservation in reservations:
            instances.extend(reservation.get("Instances", []))

        log.info("fetched_instances", count=len(instances), filters=filters)
        return instances

    def get_instance_tags(self, instance: dict[str, Any]) -> dict[str, str]:
        """Extract tags as a simple dict."""
        return {
            tag["Key"]: tag["Value"]
            for tag in instance.get("Tags", [])
        }

    def find_unattached_volumes(self) -> list[dict[str, Any]]:
        """Find EBS volumes not attached to any instance (cost waste)."""
        volumes = paginate(
            self.client, "describe_volumes", "Volumes",
            Filters=[{"Name": "status", "Values": ["available"]}],
        )

        results = []
        for vol in volumes:
            tags = {t["Key"]: t["Value"] for t in vol.get("Tags", [])}
            results.append({
                "volume_id": vol["VolumeId"],
                "size_gb": vol["Size"],
                "volume_type": vol["VolumeType"],
                "created": vol["CreateTime"].isoformat(),
                "name": tags.get("Name", "unnamed"),
                "monthly_cost_estimate": self._estimate_volume_cost(vol),
            })

        log.info("found_unattached_volumes", count=len(results))
        return results

    @staticmethod
    def _estimate_volume_cost(volume: dict[str, Any]) -> float:
        """Rough monthly cost estimate for an EBS volume."""
        prices = {"gp3": 0.08, "gp2": 0.10, "io1": 0.125, "io2": 0.125, "st1": 0.045, "sc1": 0.015}
        price_per_gb = prices.get(volume["VolumeType"], 0.10)
        return round(volume["Size"] * price_per_gb, 2)

    def find_old_snapshots(self, days: int = 90) -> list[dict[str, Any]]:
        """Find EBS snapshots older than N days owned by this account."""
        import datetime
        cutoff = datetime.datetime.now(datetime.UTC) - datetime.timedelta(days=days)

        # get_caller_identity for owner filter
        sts = boto3.client("sts")
        account_id = sts.get_caller_identity()["Account"]

        snapshots = paginate(
            self.client, "describe_snapshots", "Snapshots",
            OwnerIds=[account_id],
        )

        old_snapshots = [
            {
                "snapshot_id": snap["SnapshotId"],
                "volume_id": snap.get("VolumeId", "deleted-volume"),
                "size_gb": snap["VolumeSize"],
                "start_time": snap["StartTime"].isoformat(),
                "description": snap.get("Description", "")[:80],
                "age_days": (datetime.datetime.now(datetime.UTC) - snap["StartTime"].replace(tzinfo=datetime.UTC)).days,
            }
            for snap in snapshots
            if snap["StartTime"].replace(tzinfo=datetime.UTC) < cutoff
        ]

        log.info("found_old_snapshots", count=len(old_snapshots), cutoff_days=days)
        return old_snapshots

    @retry(max_attempts=3, retryable_exceptions=(ClientError,))
    def tag_resource(self, resource_id: str, tags: dict[str, str]) -> None:
        """Tag an EC2 resource with retry."""
        self.client.create_tags(
            Resources=[resource_id],
            Tags=[{"Key": k, "Value": v} for k, v in tags.items()],
        )
```

### S3 Operations

```python
"""
src/novatools/aws/s3.py
"""
import hashlib
from pathlib import Path
from typing import Any, BinaryIO

import boto3
import structlog
from botocore.exceptions import ClientError

from novatools.aws.session import get_client

log = structlog.get_logger()


class S3Service:
    """S3 operations for NovaMart."""

    def __init__(self, session: boto3.Session | None = None, region: str = "us-east-1") -> None:
        self.client = get_client("s3", session=session, region=region)

    def upload_file(
        self,
        bucket: str,
        key: str,
        file_path: Path,
        metadata: dict[str, str] | None = None,
        content_type: str | None = None,
    ) -> str:
        """
        Upload file to S3 with integrity verification.
        Returns the ETag (MD5 of object).
        """
        extra_args: dict[str, Any] = {}
        if metadata:
            extra_args["Metadata"] = metadata
        if content_type:
            extra_args["ContentType"] = content_type

        # Calculate local MD5 for verification
        md5 = hashlib.md5()  # noqa: S324 — not for security, for S3 integrity
        with open(file_path, "rb") as f:
            for chunk in iter(lambda: f.read(8192), b""):
                md5.update(chunk)
        local_md5 = md5.hexdigest()

        file_size = file_path.stat().st_size
        log.info("uploading_to_s3", bucket=bucket, key=key, size_bytes=file_size)

        # Use multipart for large files (>100MB)
        config = boto3.s3.transfer.TransferConfig(
            multipart_threshold=100 * 1024 * 1024,  # 100MB
            multipart_chunksize=50 * 1024 * 1024,   # 50MB chunks
            max_concurrency=10,
            use_threads=True,
        )

        self.client.upload_file(
            str(file_path), bucket, key,
            ExtraArgs=extra_args if extra_args else None,
            Config=config,
        )

        # Verify upload
        head = self.client.head_object(Bucket=bucket, Key=key)
        remote_etag = head["ETag"].strip('"')

        # For non-multipart uploads, ETag == MD5
        if "-" not in remote_etag and remote_etag != local_md5:
            log.error("upload_integrity_mismatch",
                      local_md5=local_md5, remote_etag=remote_etag)
            raise ValueError(f"Upload integrity check failed for s3://{bucket}/{key}")

        log.info("upload_complete", bucket=bucket, key=key, etag=remote_etag)
        return remote_etag

    def download_file(
        self,
        bucket: str,
        key: str,
        dest_path: Path,
    ) -> Path:
        """Download file from S3 with directory creation."""
        dest_path.parent.mkdir(parents=True, exist_ok=True)
        log.info("downloading_from_s3", bucket=bucket, key=key)
        self.client.download_file(bucket, key, str(dest_path))
        return dest_path

    def list_objects(
        self,
        bucket: str,
        prefix: str = "",
        max_keys: int | None = None,
    ) -> list[dict[str, Any]]:
        """List objects with automatic pagination."""
        paginator = self.client.get_paginator("list_objects_v2")
        objects: list[dict[str, Any]] = []

        for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
            for obj in page.get("Contents", []):
                objects.append(obj)
                if max_keys and len(objects) >= max_keys:
                    return objects

        return objects

    def object_exists(self, bucket: str, key: str) -> bool:
        """Check if S3 object exists (HEAD request)."""
        try:
            self.client.head_object(Bucket=bucket, Key=key)
            return True
        except ClientError as e:
            if e.response["Error"]["Code"] == "404":
                return False
            raise

    def generate_presigned_url(
        self,
        bucket: str,
        key: str,
        expiration: int = 3600,
    ) -> str:
        """Generate a pre-signed URL for temporary access."""
        return self.client.generate_presigned_url(
            "get_object",
            Params={"Bucket": bucket, "Key": key},
            ExpiresIn=expiration,
        )
```

### Secrets Manager

```python
"""
src/novatools/aws/secrets.py
"""
import json
from typing import Any

import boto3
import structlog
from botocore.exceptions import ClientError

from novatools.aws.session import get_client
from novatools.utils.retry import retry

log = structlog.get_logger()


class SecretsService:
    """AWS Secrets Manager operations."""

    def __init__(self, session: boto3.Session | None = None, region: str = "us-east-1") -> None:
        self.client = get_client("secretsmanager", session=session, region=region)

    @retry(max_attempts=3, retryable_exceptions=(ClientError,))
    def get_secret(self, secret_id: str) -> dict[str, Any]:
        """
        Get secret value. Parses JSON automatically.
        
        SECURITY: Never log the secret value.
        """
        try:
            response = self.client.get_secret_value(SecretId=secret_id)
        except ClientError as e:
            code = e.response["Error"]["Code"]
            if code == "ResourceNotFoundException":
                raise ValueError(f"Secret not found: {secret_id}") from e
            if code == "DecryptionFailure":
                raise PermissionError(f"Cannot decrypt {secret_id}. Check KMS permissions.") from e
            raise

        # Parse JSON if possible, otherwise return raw string
        secret_string = response["SecretString"]
        try:
            return json.loads(secret_string)
        except json.JSONDecodeError:
            return {"value": secret_string}

    def rotate_secret(
        self,
        secret_id: str,
        new_value: dict[str, Any],
        stage: str = "AWSCURRENT",
    ) -> str:
        """
        Update secret value. Returns new version ID.
        
        Pattern: put_secret → verify get returns new value → done
        """
        log.info("rotating_secret", secret_id=secret_id)  # Never log the value!

        response = self.client.put_secret_value(
            SecretId=secret_id,
            SecretString=json.dumps(new_value),
            VersionStages=[stage],
        )

        version_id = response["VersionId"]
        log.info("secret_rotated", secret_id=secret_id, version_id=version_id)

        # Verify
        verify = self.get_secret(secret_id)
        if verify != new_value:
            raise RuntimeError(f"Secret verification failed for {secret_id}")

        return version_id
```

---

## 6. HTTP/API AUTOMATION

```python
"""
src/novatools/utils/http.py

Production HTTP client with retries, timeouts, rate limiting.
"""
import time
from typing import Any

import requests
import structlog
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

log = structlog.get_logger()


def create_session(
    max_retries: int = 3,
    backoff_factor: float = 0.5,
    status_forcelist: tuple[int, ...] = (429, 500, 502, 503, 504),
    timeout: tuple[float, float] = (5.0, 30.0),  # (connect, read)
    pool_connections: int = 10,
    pool_maxsize: int = 20,
) -> requests.Session:
    """
    Create a requests Session with production defaults.
    
    WHY a session and not raw requests.get():
    - Connection pooling (reuses TCP connections)
    - Automatic retries with backoff
    - Shared headers, auth, cookies
    - 3-5x faster for multiple calls to same host
    """
    session = requests.Session()

    retry_strategy = Retry(
        total=max_retries,
        backoff_factor=backoff_factor,
        status_forcelist=status_forcelist,
        allowed_methods=["GET", "HEAD", "OPTIONS", "POST", "PUT", "DELETE"],
        raise_on_status=False,  # We handle status codes ourselves
    )

    adapter = HTTPAdapter(
        max_retries=retry_strategy,
        pool_connections=pool_connections,
        pool_maxsize=pool_maxsize,
    )

    session.mount("https://", adapter)
    session.mount("http://", adapter)

    # Default timeout for all requests (can be overridden per-call)
    session.timeout = timeout

    return session


# ─── Rate Limiter ──────────────────────────────────────────
class RateLimiter:
    """
    Token bucket rate limiter for API calls.
    
    Usage:
        limiter = RateLimiter(calls_per_second=10)
        for item in items:
            limiter.wait()
            api_call(item)
    """

    def __init__(self, calls_per_second: float) -> None:
        self.min_interval = 1.0 / calls_per_second
        self._last_call = 0.0

    def wait(self) -> None:
        now = time.monotonic()
        elapsed = now - self._last_call
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        self._last_call = time.monotonic()


# ─── Slack Client ──────────────────────────────────────────
class SlackClient:
    """Slack API client for notifications."""

    def __init__(self, webhook_url: str | None = None, token: str | None = None) -> None:
        self.webhook_url = webhook_url
        self.token = token
        self.session = create_session(max_retries=2, timeout=(5.0, 10.0))

        if token:
            self.session.headers["Authorization"] = f"Bearer {token}"
            self.api_base = "https://slack.com/api"

    def send_webhook(
        self,
        text: str,
        color: str = "#36a64f",
        fields: list[dict[str, str]] | None = None,
        channel: str | None = None,
    ) -> bool:
        """Send message via incoming webhook."""
        if not self.webhook_url:
            log.warning("slack_webhook_not_configured")
            return False

        payload: dict[str, Any] = {
            "attachments": [{
                "color": color,
                "text": text,
                "fields": fields or [],
                "footer": f"novatools | {time.strftime('%H:%M UTC')}",
            }]
        }
        if channel:
            payload["channel"] = channel

        try:
            response = self.session.post(self.webhook_url, json=payload, timeout=5)
            if response.status_code != 200 or response.text != "ok":
                log.error("slack_webhook_failed",
                          status=response.status_code, body=response.text[:200])
                return False
            return True
        except requests.RequestException as e:
            log.error("slack_webhook_error", error=str(e))
            return False

    def send_message(
        self,
        channel: str,
        text: str,
        blocks: list[dict[str, Any]] | None = None,
        thread_ts: str | None = None,
    ) -> dict[str, Any] | None:
        """Send message via Slack API (richer formatting)."""
        if not self.token:
            log.warning("slack_token_not_configured")
            return None

        payload: dict[str, Any] = {
            "channel": channel,
            "text": text,
        }
        if blocks:
            payload["blocks"] = blocks
        if thread_ts:
            payload["thread_ts"] = thread_ts

        response = self.session.post(
            f"{self.api_base}/chat.postMessage",
            json=payload,
        )
        data = response.json()

        if not data.get("ok"):
            log.error("slack_api_error", error=data.get("error"))
            return None

        return data


# ─── PagerDuty Client ─────────────────────────────────────
class PagerDutyClient:
    """PagerDuty Events API v2 client."""

    EVENTS_URL = "https://events.pagerduty.com/v2/enqueue"

    def __init__(self, routing_key: str) -> None:
        self.routing_key = routing_key
        self.session = create_session(max_retries=3)

    def trigger(
        self,
        summary: str,
        severity: str = "error",  # critical, error, warning, info
        source: str = "novatools",
        component: str | None = None,
        group: str | None = None,
        custom_details: dict[str, Any] | None = None,
        dedup_key: str | None = None,
    ) -> str | None:
        """Trigger a PagerDuty incident. Returns dedup_key."""
        payload = {
            "routing_key": self.routing_key,
            "event_action": "trigger",
            "dedup_key": dedup_key,
            "payload": {
                "summary": summary[:1024],  # PD limit
                "severity": severity,
                "source": source,
                "component": component,
                "group": group,
                "custom_details": custom_details or {},
                "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S.000Z"),
            },
        }

        response = self.session.post(self.EVENTS_URL, json=payload)
        if response.status_code == 202:
            data = response.json()
            return data.get("dedup_key")
        else:
            log.error("pagerduty_trigger_failed",
                      status=response.status_code, body=response.text[:200])
            return None

    def resolve(self, dedup_key: str) -> bool:
        """Resolve a PagerDuty incident."""
        payload = {
            "routing_key": self.routing_key,
            "event_action": "resolve",
            "dedup_key": dedup_key,
        }
        response = self.session.post(self.EVENTS_URL, json=payload)
        return response.status_code == 202


# ─── Jira Client ──────────────────────────────────────────
class JiraClient:
    """Jira REST API client for incident tracking."""

    def __init__(self, base_url: str, email: str, api_token: str) -> None:
        self.base_url = base_url.rstrip("/")
        self.session = create_session()
        self.session.auth = (email, api_token)
        self.session.headers["Content-Type"] = "application/json"

    def create_issue(
        self,
        project: str,
        summary: str,
        description: str,
        issue_type: str = "Bug",
        priority: str = "High",
        labels: list[str] | None = None,
        assignee: str | None = None,
    ) -> str:
        """Create a Jira issue. Returns issue key (e.g., 'OPS-1234')."""
        payload: dict[str, Any] = {
            "fields": {
                "project": {"key": project},
                "summary": summary,
                "description": {
                    "type": "doc",
                    "version": 1,
                    "content": [{
                        "type": "paragraph",
                        "content": [{"type": "text", "text": description}],
                    }],
                },
                "issuetype": {"name": issue_type},
                "priority": {"name": priority},
                "labels": labels or [],
            }
        }
        if assignee:
            payload["fields"]["assignee"] = {"accountId": assignee}

        response = self.session.post(
            f"{self.base_url}/rest/api/3/issue",
            json=payload,
        )
        response.raise_for_status()
        data = response.json()
        issue_key = data["key"]

        log.info("jira_issue_created", key=issue_key, summary=summary)
        return issue_key

    def add_comment(self, issue_key: str, body: str) -> None:
        """Add a comment to an existing Jira issue."""
        payload = {
            "body": {
                "type": "doc",
                "version": 1,
                "content": [{
                    "type": "paragraph",
                    "content": [{"type": "text", "text": body}],
                }],
            }
        }
        response = self.session.post(
            f"{self.base_url}/rest/api/3/issue/{issue_key}/comment",
            json=payload,
        )
        response.raise_for_status()

    def transition_issue(self, issue_key: str, transition_name: str) -> None:
        """Transition issue (e.g., 'In Progress', 'Done')."""
        # First, get available transitions
        response = self.session.get(
            f"{self.base_url}/rest/api/3/issue/{issue_key}/transitions"
        )
        response.raise_for_status()
        transitions = response.json()["transitions"]

        transition_id = None
        for t in transitions:
            if t["name"].lower() == transition_name.lower():
                transition_id = t["id"]
                break

        if not transition_id:
            available = [t["name"] for t in transitions]
            raise ValueError(
                f"Transition '{transition_name}' not found for {issue_key}. "
                f"Available: {available}"
            )

        self.session.post(
            f"{self.base_url}/rest/api/3/issue/{issue_key}/transitions",
            json={"transition": {"id": transition_id}},
        ).raise_for_status()

        log.info("jira_issue_transitioned", key=issue_key, transition=transition_name)
```

---

## 7. CLI FRAMEWORKS — Click + Rich

```python
"""
src/novatools/cli/main.py

Root CLI group. All subcommands registered here.
"""
import sys

import click
import structlog

from novatools.utils.logging import setup_structlog

# ─── Root Group ────────────────────────────────────────────
@click.group()
@click.version_option(package_name="novatools")
@click.option(
    "--log-level",
    type=click.Choice(["DEBUG", "INFO", "WARNING", "ERROR"], case_sensitive=False),
    default="INFO",
    envvar="LOG_LEVEL",
    help="Logging verbosity",
)
@click.option(
    "--json-output",
    is_flag=True,
    default=False,
    envvar="JSON_OUTPUT",
    help="Output logs in JSON format",
)
@click.option(
    "--region",
    default="us-east-1",
    envvar="AWS_REGION",
    help="AWS region",
)
@click.option(
    "--profile",
    default=None,
    envvar="AWS_PROFILE",
    help="AWS profile name",
)
@click.pass_context
def cli(
    ctx: click.Context,
    log_level: str,
    json_output: bool,
    region: str,
    profile: str | None,
) -> None:
    """NovaMart Platform Engineering CLI Tools."""
    # Ensure context object exists
    ctx.ensure_object(dict)

    # Store shared config in context
    ctx.obj["region"] = region
    ctx.obj["profile"] = profile

    # Initialize logging
    setup_structlog(level=log_level, json_output=json_output)


# ─── Register subcommand groups ───────────────────────────
from novatools.cli.cost import cost_group
from novatools.cli.incidents import incident_group
from novatools.cli.secrets import secrets_group

cli.add_command(cost_group, "cost")
cli.add_command(incident_group, "incident")
cli.add_command(secrets_group, "secrets")


# ─── Entry point ──────────────────────────────────────────
def main() -> None:
    try:
        cli()
    except Exception as e:
        log = structlog.get_logger()
        log.exception("unhandled_error")
        click.echo(f"Error: {e}", err=True)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### Cost Reporting Subcommand

```python
"""
src/novatools/cli/cost.py

Usage:
    novatools cost report --days 30 --group-by service
    novatools cost waste --type volumes
    novatools cost savings
"""
import datetime
from typing import Any

import click
import structlog
from rich.console import Console
from rich.table import Table
from rich.panel import Panel
from rich.text import Text

from novatools.aws.ec2 import EC2Service
from novatools.aws.session import get_client

log = structlog.get_logger()
console = Console(stderr=True)  # Rich output to stderr so stdout stays pipeable


@click.group("cost")
def cost_group() -> None:
    """Cost management and optimization."""
    pass


@cost_group.command("waste")
@click.option("--type", "waste_type",
              type=click.Choice(["volumes", "snapshots", "all"]),
              default="all", help="Type of waste to find")
@click.option("--format", "output_format",
              type=click.Choice(["table", "json", "csv"]),
              default="table")
@click.option("--min-age-days", default=30, help="Minimum age in days")
@click.pass_context
def find_waste(
    ctx: click.Context,
    waste_type: str,
    output_format: str,
    min_age_days: int,
) -> None:
    """Find wasted AWS resources (unattached volumes, old snapshots)."""
    region = ctx.obj["region"]
    ec2 = EC2Service(region=region)

    total_monthly_waste = 0.0

    if waste_type in ("volumes", "all"):
        volumes = ec2.find_unattached_volumes()
        if volumes:
            _render_volumes(volumes, output_format)
            total_monthly_waste += sum(v["monthly_cost_estimate"] for v in volumes)
        else:
            console.print("[green]No unattached volumes found[/green]")

    if waste_type in ("snapshots", "all"):
        snapshots = ec2.find_old_snapshots(days=min_age_days)
        if snapshots:
            _render_snapshots(snapshots, output_format)
            # Rough estimate: $0.05/GB/month for snapshots
            snapshot_cost = sum(s["size_gb"] * 0.05 for s in snapshots)
            total_monthly_waste += snapshot_cost
        else:
            console.print("[green]No old snapshots found[/green]")

    if output_format == "table":
        console.print(Panel(
            f"[bold red]Estimated monthly waste: ${total_monthly_waste:,.2f}[/bold red]\n"
            f"Estimated annual waste: ${total_monthly_waste * 12:,.2f}",
            title="💰 Cost Summary",
        ))


def _render_volumes(volumes: list[dict[str, Any]], fmt: str) -> None:
    """Render volume data in requested format."""
    if fmt == "json":
        import json
        click.echo(json.dumps(volumes, indent=2, default=str))
        return

    if fmt == "csv":
        click.echo("volume_id,size_gb,type,name,monthly_cost")
        for v in volumes:
            click.echo(f"{v['volume_id']},{v['size_gb']},{v['volume_type']},"
                       f"{v['name']},{v['monthly_cost_estimate']}")
        return

    # Rich table
    table = Table(title="Unattached EBS Volumes", show_lines=False)
    table.add_column("Volume ID", style="cyan")
    table.add_column("Size (GB)", justify="right")
    table.add_column("Type")
    table.add_column("Name")
    table.add_column("Created")
    table.add_column("$/month", justify="right", style="red")

    for v in sorted(volumes, key=lambda x: x["monthly_cost_estimate"], reverse=True):
        table.add_row(
            v["volume_id"],
            str(v["size_gb"]),
            v["volume_type"],
            v["name"],
            v["created"][:10],
            f"${v['monthly_cost_estimate']:.2f}",
        )

    console.print(table)


def _render_snapshots(snapshots: list[dict[str, Any]], fmt: str) -> None:
    """Render snapshot data in requested format."""
    if fmt == "json":
        import json
        click.echo(json.dumps(snapshots, indent=2, default=str))
        return

    table = Table(title="Old EBS Snapshots", show_lines=False)
    table.add_column("Snapshot ID", style="cyan")
    table.add_column("Volume", style="dim")
    table.add_column("Size (GB)", justify="right")
    table.add_column("Age (days)", justify="right")
    table.add_column("Description")

    for s in sorted(snapshots, key=lambda x: x["age_days"], reverse=True):
        age_style = "bold red" if s["age_days"] > 180 else "yellow"
        table.add_row(
            s["snapshot_id"],
            s["volume_id"],
            str(s["size_gb"]),
            Text(str(s["age_days"]), style=age_style),
            s["description"],
        )

    console.print(table)


@cost_group.command("report")
@click.option("--days", default=30, help="Lookback period in days")
@click.option("--group-by",
              type=click.Choice(["service", "environment", "team"]),
              default="service")
@click.option("--format", "output_format",
              type=click.Choice(["table", "json", "csv"]),
              default="table")
@click.pass_context
def cost_report(
    ctx: click.Context,
    days: int,
    group_by: str,
    output_format: str,
) -> None:
    """Generate AWS cost report grouped by tag."""
    region = ctx.obj["region"]
    ce_client = get_client("ce", region="us-east-1")  # Cost Explorer is us-east-1 only

    end_date = datetime.date.today()
    start_date = end_date - datetime.timedelta(days=days)

    tag_key_map = {
        "service": "Service",
        "environment": "Environment",
        "team": "Team",
    }
    tag_key = tag_key_map[group_by]

    try:
        response = ce_client.get_cost_and_usage(
            TimePeriod={
                "Start": start_date.isoformat(),
                "End": end_date.isoformat(),
            },
            Granularity="MONTHLY",
            Metrics=["UnblendedCost", "UsageQuantity"],
            GroupBy=[
                {"Type": "TAG", "Key": tag_key},
            ],
        )
    except Exception as e:
        log.error("cost_explorer_failed", error=str(e))
        raise click.ClickException(f"Cost Explorer query failed: {e}")

    # Parse results
    cost_data: dict[str, float] = {}
    for result in response["ResultsByTime"]:
        for group in result["Groups"]:
            tag_value = group["Keys"][0].replace(f"{tag_key}$", "") or "(untagged)"
            amount = float(group["Metrics"]["UnblendedCost"]["Amount"])
            cost_data[tag_value] = cost_data.get(tag_value, 0.0) + amount

    # Sort by cost descending
    sorted_costs = sorted(cost_data.items(), key=lambda x: x[1], reverse=True)
    total = sum(v for _, v in sorted_costs)

    if output_format == "json":
        import json
        click.echo(json.dumps(
            {"period_days": days, "group_by": group_by, "total": total,
             "items": [{"name": k, "cost": round(v, 2)} for k, v in sorted_costs]},
            indent=2,
        ))
        return

    if output_format == "csv":
        click.echo(f"{group_by},cost,percentage")
        for name, cost in sorted_costs:
            click.echo(f"{name},{cost:.2f},{cost / total * 100:.1f}")
        return

    # Rich table
    table = Table(title=f"AWS Cost Report ({days} days, by {group_by})")
    table.add_column(group_by.title(), style="cyan")
    table.add_column("Cost", justify="right")
    table.add_column("% of Total", justify="right")
    table.add_column("Bar", min_width=30)

    for name, cost in sorted_costs:
        pct = cost / total * 100 if total > 0 else 0
        bar_width = int(pct / 100 * 30)
        bar = "█" * bar_width + "░" * (30 - bar_width)

        cost_style = "bold red" if cost > 10000 else "yellow" if cost > 1000 else "green"
        table.add_row(
            name,
            Text(f"${cost:,.2f}", style=cost_style),
            f"{pct:.1f}%",
            Text(bar, style="blue"),
        )

    table.add_section()
    table.add_row("TOTAL", Text(f"${total:,.2f}", style="bold"), "100%", "")

    console.print(table)
```

### Incident Management Subcommand

```python
"""
src/novatools/cli/incidents.py

Usage:
    novatools incident create --severity SEV1 --service payment-service --summary "5xx spike"
    novatools incident collect --id INC-2024-0142 --namespace production
"""
import datetime
import json
import time
from typing import Any

import click
import structlog
from rich.console import Console
from rich.panel import Panel
from rich.progress import Progress, SpinnerColumn, TextColumn

from novatools.integrations.slack import SlackClient
from novatools.integrations.jira_client import JiraClient
from novatools.integrations.pagerduty import PagerDutyClient
from novatools.aws.secrets import SecretsService

log = structlog.get_logger()
console = Console(stderr=True)


@click.group("incident")
def incident_group() -> None:
    """Incident management automation."""
    pass


@incident_group.command("create")
@click.option("--severity", "-s",
              type=click.Choice(["SEV1", "SEV2", "SEV3", "SEV4"]),
              required=True)
@click.option("--service", required=True, help="Affected service name")
@click.option("--summary", required=True, help="Brief incident description")
@click.option("--commander", default=None, help="Incident commander (Slack user ID)")
@click.option("--dry-run", is_flag=True, help="Show what would be created without executing")
@click.pass_context
def create_incident(
    ctx: click.Context,
    severity: str,
    service: str,
    summary: str,
    commander: str | None,
    dry_run: bool,
) -> None:
    """
    Create an incident: Slack channel + Jira ticket + PagerDuty alert.

    Automates the first 5 minutes of incident response.
    """
    region = ctx.obj["region"]
    secrets = SecretsService(region=region)

    # Generate incident ID
    incident_id = f"INC-{datetime.datetime.now(datetime.UTC).strftime('%Y%m%d-%H%M%S')}"
    timestamp = datetime.datetime.now(datetime.UTC).isoformat()

    incident_data = {
        "id": incident_id,
        "severity": severity,
        "service": service,
        "summary": summary,
        "commander": commander,
        "created_at": timestamp,
        "status": "investigating",
    }

    if dry_run:
        console.print(Panel(
            json.dumps(incident_data, indent=2),
            title=f"[yellow]DRY RUN: Would create {incident_id}[/yellow]",
        ))
        console.print("[yellow]Would create: Slack channel, Jira ticket, PagerDuty alert[/yellow]")
        return

    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        console=console,
    ) as progress:
        # Step 1: Create Jira ticket
        task = progress.add_task("Creating Jira ticket...", total=None)
        try:
            jira_creds = secrets.get_secret("novamart/platform/jira")
            jira = JiraClient(
                base_url=jira_creds["url"],
                email=jira_creds["email"],
                api_token=jira_creds["api_token"],
            )

            severity_priority_map = {
                "SEV1": "Highest",
                "SEV2": "High",
                "SEV3": "Medium",
                "SEV4": "Low",
            }

            jira_key = jira.create_issue(
                project="OPS",
                summary=f"[{severity}] [{service}] {summary}",
                description=(
                    f"Incident: {incident_id}\n"
                    f"Severity: {severity}\n"
                    f"Service: {service}\n"
                    f"Summary: {summary}\n"
                    f"Created: {timestamp}\n"
                    f"Commander: {commander or 'TBD'}\n"
                ),
                issue_type="Bug",
                priority=severity_priority_map[severity],
                labels=["incident", severity.lower(), service],
            )
            progress.update(task, description=f"✅ Jira ticket: {jira_key}")
        except Exception as e:
            progress.update(task, description=f"❌ Jira failed: {e}")
            jira_key = "FAILED"

        # Step 2: Create Slack channel
        task2 = progress.add_task("Creating Slack channel...", total=None)
        try:
            slack_creds = secrets.get_secret("novamart/platform/slack")
            slack = SlackClient(token=slack_creds["bot_token"])

            channel_name = f"inc-{severity.lower()}-{service}-{datetime.date.today():%m%d}"

            # Create channel via Slack API
            resp = slack.session.post(
                "https://slack.com/api/conversations.create",
                json={"name": channel_name, "is_private": False},
            ).json()

            if resp.get("ok"):
                channel_id = resp["channel"]["id"]

                # Set topic
                slack.session.post(
                    "https://slack.com/api/conversations.setTopic",
                    json={
                        "channel": channel_id,
                        "topic": f"{severity} | {service} | {summary} | Jira: {jira_key}",
                    },
                )

                # Post initial message
                slack.send_message(
                    channel=channel_id,
                    text=f"🚨 *Incident Created: {incident_id}*",
                    blocks=[
                        {
                            "type": "header",
                            "text": {"type": "plain_text", "text": f"🚨 {incident_id} — {severity}"},
                        },
                        {
                            "type": "section",
                            "fields": [
                                {"type": "mrkdwn", "text": f"*Service:*\n{service}"},
                                {"type": "mrkdwn", "text": f"*Severity:*\n{severity}"},
                                {"type": "mrkdwn", "text": f"*Summary:*\n{summary}"},
                                {"type": "mrkdwn", "text": f"*Jira:*\n{jira_key}"},
                                {"type": "mrkdwn", "text": f"*Commander:*\n{commander or 'TBD'}"},
                                {"type": "mrkdwn", "text": f"*Status:*\nInvestigating"},
                            ],
                        },
                        {"type": "divider"},
                        {
                            "type": "section",
                            "text": {
                                "type": "mrkdwn",
                                "text": (
                                    "*Runbook:* <https://wiki.novamart.com/runbooks/"
                                    f"{service}|{service} runbook>\n"
                                    "*Dashboard:* <https://grafana.novamart.com/d/"
                                    f"{service}|{service} dashboard>\n"
                                    "*Logs:* <https://grafana.novamart.com/explore?"
                                    f"query={{app=\"{service}\"}}|Loki logs>"
                                ),
                            },
                        },
                    ],
                )

                # Invite commander if specified
                if commander:
                    slack.session.post(
                        "https://slack.com/api/conversations.invite",
                        json={"channel": channel_id, "users": commander},
                    )

                progress.update(task2, description=f"✅ Slack channel: #{channel_name}")
            else:
                progress.update(task2, description=f"❌ Slack failed: {resp.get('error')}")
                channel_name = "FAILED"

        except Exception as e:
            progress.update(task2, description=f"❌ Slack failed: {e}")
            channel_name = "FAILED"

        # Step 3: PagerDuty alert (SEV1/SEV2 only)
        if severity in ("SEV1", "SEV2"):
            task3 = progress.add_task("Triggering PagerDuty...", total=None)
            try:
                pd_creds = secrets.get_secret("novamart/platform/pagerduty")
                pd = PagerDutyClient(routing_key=pd_creds["routing_key"])

                dedup_key = pd.trigger(
                    summary=f"[{severity}] {service}: {summary}",
                    severity="critical" if severity == "SEV1" else "error",
                    source="novatools",
                    component=service,
                    group="platform",
                    custom_details={
                        "incident_id": incident_id,
                        "jira": jira_key,
                        "slack_channel": channel_name,
                    },
                )
                progress.update(task3, description=f"✅ PagerDuty alert: {dedup_key}")
            except Exception as e:
                progress.update(task3, description=f"❌ PagerDuty failed: {e}")

    # Final summary
    console.print(Panel(
        f"ID: {incident_id}\n"
        f"Jira: {jira_key}\n"
        f"Slack: #{channel_name}\n"
        f"PagerDuty: {'Triggered' if severity in ('SEV1', 'SEV2') else 'N/A (SEV3/4)'}",
        title=f"[bold green]✅ Incident {incident_id} Created[/bold green]",
    ))

    # Output JSON to stdout (for piping to other tools)
    click.echo(json.dumps({
        **incident_data,
        "jira_key": jira_key,
        "slack_channel": channel_name,
    }))
```

---

## 8. JINJA2 TEMPLATING

```python
"""
src/novatools/utils/templates.py

Jinja2 for generating configs, reports, runbooks.
"""
from pathlib import Path
from typing import Any

import jinja2
import structlog

log = structlog.get_logger()

# ─── Template Environment ─────────────────────────────────
_template_dir = Path(__file__).parent.parent / "templates"

_env = jinja2.Environment(
    loader=jinja2.FileSystemLoader(str(_template_dir)),
    autoescape=jinja2.select_autoescape(["html"]),  # HTML auto-escaping
    undefined=jinja2.StrictUndefined,  # Error on undefined variables (like set -u)
    trim_blocks=True,       # Remove first newline after block tag
    lstrip_blocks=True,     # Strip leading whitespace before block tags
    keep_trailing_newline=True,
)

# ─── Custom Filters ───────────────────────────────────────
def bytes_human(value: int) -> str:
    """Convert bytes to human-readable."""
    for unit in ["B", "KB", "MB", "GB", "TB"]:
        if abs(value) < 1024:
            return f"{value:.1f}{unit}"
        value /= 1024
    return f"{value:.1f}PB"

def duration_human(seconds: float) -> str:
    """Convert seconds to human-readable duration."""
    if seconds < 60:
        return f"{seconds:.1f}s"
    if seconds < 3600:
        return f"{seconds / 60:.1f}m"
    if seconds < 86400:
        return f"{seconds / 3600:.1f}h"
    return f"{seconds / 86400:.1f}d"

_env.filters["bytes_human"] = bytes_human
_env.filters["duration_human"] = duration_human


def render_template(template_name: str, **context: Any) -> str:
    """Render a Jinja2 template from the templates directory."""
    try:
        template = _env.get_template(template_name)
        return template.render(**context)
    except jinja2.UndefinedError as e:
        raise ValueError(f"Template variable not provided: {e}") from e
    except jinja2.TemplateNotFound:
        raise FileNotFoundError(f"Template not found: {template_name}") from None


def render_string(template_str: str, **context: Any) -> str:
    """Render a Jinja2 template from a string."""
    template = _env.from_string(template_str)
    return template.render(**context)
```

**Template examples:**

```jinja2
{# templates/incident_report.md.j2 #}
# Incident Report: {{ incident_id }}

## Summary
| Field | Value |
|-------|-------|
| **Incident ID** | {{ incident_id }} |
| **Severity** | {{ severity }} |
| **Service** | {{ service }} |
| **Duration** | {{ duration | duration_human }} |
| **Impact** | {{ impact }} |
| **Commander** | {{ commander }} |
| **Started** | {{ started_at }} |
| **Resolved** | {{ resolved_at }} |

## Timeline
{% for event in timeline %}
- **{{ event.time }}** — {{ event.description }}
{% endfor %}

## Root Cause
{{ root_cause }}

## Contributing Factors
{% for factor in contributing_factors %}
1. {{ factor }}
{% endfor %}

## Action Items
| # | Action | Owner | Priority | Due Date | Status |
|---|--------|-------|----------|----------|--------|
{% for item in action_items %}
| {{ loop.index }} | {{ item.description }} | {{ item.owner }} | {{ item.priority }} | {{ item.due_date }} | {{ item.status }} |
{% endfor %}

## Metrics Impact
- Error rate peak: {{ error_rate_peak }}%
- Latency P99 peak: {{ latency_p99_peak | duration_human }}
- Affected users: ~{{ affected_users | int }}
- Estimated revenue impact: ${{ revenue_impact | int }}
```

```jinja2
{# templates/alert_config.yaml.j2 #}
{# Generates Prometheus alerting rules from service definitions #}
{% for service in services %}
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: {{ service.name }}-alerts
  namespace: monitoring
  labels:
    app.kubernetes.io/part-of: novamart
    team: {{ service.team }}
spec:
  groups:
    - name: {{ service.name }}.availability
      rules:
        - alert: {{ service.name | title | replace('-', '') }}HighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{service="{{ service.name }}", code=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{service="{{ service.name }}"}[5m]))
            ) > {{ service.error_threshold | default(0.01) }}
          for: 5m
          labels:
            severity: {{ "critical" if service.tier == 1 else "warning" }}
            service: {{ service.name }}
            team: {{ service.team }}
          annotations:
            summary: "High error rate on {{ service.name }}"
            runbook_url: "https://wiki.novamart.com/runbooks/{{ service.name }}"

{% if service.latency_slo_ms is defined %}
        - alert: {{ service.name | title | replace('-', '') }}HighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="{{ service.name }}"}[5m])) by (le)
            ) > {{ service.latency_slo_ms / 1000 }}
          for: 5m
          labels:
            severity: warning
            service: {{ service.name }}
          annotations:
            summary: "P99 latency above {{ service.latency_slo_ms }}ms on {{ service.name }}"
{% endif %}
{% endfor %}
```

```python
# Usage — generate alert configs from a service catalog:
services = [
    {"name": "payment-service", "team": "payments", "tier": 1,
     "error_threshold": 0.001, "latency_slo_ms": 500},
    {"name": "order-service", "team": "orders", "tier": 1,
     "error_threshold": 0.005, "latency_slo_ms": 800},
    {"name": "notification-service", "team": "notifications", "tier": 2,
     "error_threshold": 0.01},
]

output = render_template("alert_config.yaml.j2", services=services)
Path("generated/alerting-rules.yaml").write_text(output)
```

---

## 9. TESTING — pytest + moto + responses

```python
"""
tests/conftest.py

Shared fixtures for all tests.
"""
import os
from typing import Any, Generator
from unittest.mock import MagicMock

import boto3
import pytest
from moto import mock_aws

# Prevent real AWS calls during tests
os.environ["AWS_ACCESS_KEY_ID"] = "testing"
os.environ["AWS_SECRET_ACCESS_KEY"] = "testing"
os.environ["AWS_SECURITY_TOKEN"] = "testing"
os.environ["AWS_SESSION_TOKEN"] = "testing"
os.environ["AWS_DEFAULT_REGION"] = "us-east-1"


@pytest.fixture
def aws_credentials() -> None:
    """Mocked AWS credentials for moto."""
    pass  # env vars set above


@pytest.fixture
def ec2_client(aws_credentials: None) -> Generator[Any, None, None]:
    """Mocked EC2 client."""
    with mock_aws():
        client = boto3.client("ec2", region_name="us-east-1")
        yield client


@pytest.fixture
def s3_client(aws_credentials: None) -> Generator[Any, None, None]:
    """Mocked S3 client."""
    with mock_aws():
        client = boto3.client("s3", region_name="us-east-1")
        yield client


@pytest.fixture
def rds_client(aws_credentials: None) -> Generator[Any, None, None]:
    """Mocked RDS client."""
    with mock_aws():
        client = boto3.client("rds", region_name="us-east-1")
        yield client


@pytest.fixture
def secretsmanager_client(aws_credentials: None) -> Generator[Any, None, None]:
    """Mocked Secrets Manager client."""
    with mock_aws():
        client = boto3.client("secretsmanager", region_name="us-east-1")
        yield client


@pytest.fixture
def mock_slack() -> MagicMock:
    """Mock Slack client."""
    slack = MagicMock()
    slack.send_webhook.return_value = True
    slack.send_message.return_value = {"ok": True, "ts": "1234567890.123456"}
    return slack
```

### Unit Tests

```python
"""
tests/unit/aws/test_ec2.py
"""
import boto3
import pytest
from moto import mock_aws

from novatools.aws.ec2 import EC2Service


@mock_aws
class TestEC2Service:
    """Test EC2 operations with moto mocks."""

    @pytest.fixture(autouse=True)
    def setup(self) -> None:
        """Create test infrastructure."""
        self.ec2_client = boto3.client("ec2", region_name="us-east-1")
        self.ec2_resource = boto3.resource("ec2", region_name="us-east-1")

        # Create VPC
        vpc = self.ec2_resource.create_vpc(CidrBlock="10.0.0.0/16")
        self.vpc_id = vpc.id

        # Create instances
        for i, env in enumerate(["production", "production", "staging"]):
            instances = self.ec2_resource.create_instances(
                ImageId="ami-12345678",
                MinCount=1,
                MaxCount=1,
                InstanceType="t3.medium",
                TagSpecifications=[{
                    "ResourceType": "instance",
                    "Tags": [
                        {"Key": "Environment", "Value": env},
                        {"Key": "Service", "Value": "payment-service"},
                        {"Key": "Name", "Value": f"test-instance-{i}"},
                    ],
                }],
            )

        # Create unattached volume
        self.ec2_client.create_volume(
            AvailabilityZone="us-east-1a",
            Size=100,
            VolumeType="gp3",
            TagSpecifications=[{
                "ResourceType": "volume",
                "Tags": [{"Key": "Name", "Value": "orphaned-volume"}],
            }],
        )

        self.service = EC2Service(region="us-east-1")

    def test_get_instances_all(self) -> None:
        """Should return all running instances."""
        instances = self.service.get_instances()
        assert len(instances) == 3

    def test_get_instances_by_environment(self) -> None:
        """Should filter by environment tag."""
        instances = self.service.get_instances(environment="production")
        assert len(instances) == 2

    def test_get_instances_by_service(self) -> None:
        """Should filter by service tag."""
        instances = self.service.get_instances(service="payment-service")
        assert len(instances) == 3

    def test_get_instances_by_environment_and_service(self) -> None:
        """Should apply multiple filters."""
        instances = self.service.get_instances(
            environment="staging",
            service="payment-service",
        )
        assert len(instances) == 1

    def test_get_instance_tags(self) -> None:
        """Should extract tags as dict."""
        instances = self.service.get_instances(environment="production")
        tags = self.service.get_instance_tags(instances[0])
        assert tags["Environment"] == "production"
        assert tags["Service"] == "payment-service"

    def test_find_unattached_volumes(self) -> None:
        """Should find volumes with status 'available'."""
        volumes = self.service.find_unattached_volumes()
        assert len(volumes) >= 1
        assert volumes[0]["name"] == "orphaned-volume"
        assert volumes[0]["size_gb"] == 100
        assert volumes[0]["volume_type"] == "gp3"
        assert volumes[0]["monthly_cost_estimate"] == 8.0  # 100 * 0.08


@mock_aws
class TestEC2ServiceEdgeCases:
    """Edge cases and error handling."""

    def test_no_instances_returns_empty_list(self) -> None:
        """Should handle zero results gracefully."""
        service = EC2Service(region="us-east-1")
        instances = service.get_instances(environment="nonexistent")
        assert instances == []

    def test_no_unattached_volumes(self) -> None:
        """Should handle zero unattached volumes."""
        service = EC2Service(region="us-east-1")
        volumes = service.find_unattached_volumes()
        assert volumes == []

    def test_instance_without_tags(self) -> None:
        """Should handle instances with no tags."""
        ec2 = boto3.resource("ec2", region_name="us-east-1")
        ec2.create_instances(
            ImageId="ami-12345678", MinCount=1, MaxCount=1,
            InstanceType="t3.micro",
        )

        service = EC2Service(region="us-east-1")
        instances = service.get_instances()
        tags = service.get_instance_tags(instances[0])
        assert tags == {}
```

### Testing HTTP Integrations

```python
"""
tests/unit/test_slack.py
"""
import pytest
import responses

from novatools.integrations.slack import SlackClient


class TestSlackWebhook:
    """Test Slack webhook integration."""

    @responses.activate
    def test_send_webhook_success(self) -> None:
        """Should send message and return True."""
        responses.add(
            responses.POST,
            "https://hooks.slack.com/services/test",
            body="ok",
            status=200,
        )

        client = SlackClient(webhook_url="https://hooks.slack.com/services/test")
        result = client.send_webhook("Test message", color="#36a64f")

        assert result is True
        assert len(responses.calls) == 1
        # Verify payload structure
        import json
        payload = json.loads(responses.calls[0].request.body)
        assert payload["attachments"][0]["text"] == "Test message"
        assert payload["attachments"][0]["color"] == "#36a64f"

    @responses.activate
    def test_send_webhook_failure(self) -> None:
        """Should return False on non-200 response."""
        responses.add(
            responses.POST,
            "https://hooks.slack.com/services/test",
            body="invalid_token",
            status=403,
        )

        client = SlackClient(webhook_url="https://hooks.slack.com/services/test")
        result = client.send_webhook("Test message")

        assert result is False

    def test_send_webhook_no_url(self) -> None:
        """Should return False if no webhook URL configured."""
        client = SlackClient()
        result = client.send_webhook("Test message")
        assert result is False

    @responses.activate
    def test_send_webhook_timeout(self) -> None:
        """Should return False on network timeout."""
        responses.add(
            responses.POST,
            "https://hooks.slack.com/services/test",
            body=responses.ConnectionError("Connection timed out"),
        )

        client = SlackClient(webhook_url="https://hooks.slack.com/services/test")
        result = client.send_webhook("Test message")
        assert result is False


class TestSlackAPI:
    """Test Slack API client."""

    @responses.activate
    def test_send_message_success(self) -> None:
        """Should send message via API."""
        responses.add(
            responses.POST,
            "https://slack.com/api/chat.postMessage",
            json={"ok": True, "ts": "1234567890.123456", "channel": "C123"},
            status=200,
        )

        client = SlackClient(token="xoxb-test-token")
        result = client.send_message(channel="C123", text="Hello")

        assert result is not None
        assert result["ok"] is True
        # Verify auth header
        assert responses.calls[0].request.headers["Authorization"] == "Bearer xoxb-test-token"

    @responses.activate
    def test_send_message_thread(self) -> None:
        """Should send threaded reply."""
        responses.add(
            responses.POST,
            "https://slack.com/api/chat.postMessage",
            json={"ok": True, "ts": "1234567890.999"},
            status=200,
        )

        client = SlackClient(token="xoxb-test-token")
        result = client.send_message(
            channel="C123",
            text="Thread reply",
            thread_ts="1234567890.123456",
        )

        import json
        payload = json.loads(responses.calls[0].request.body)
        assert payload["thread_ts"] == "1234567890.123456"
```

### Testing the Retry Decorator

```python
"""
tests/unit/test_retry.py
"""
import pytest

from novatools.utils.retry import retry, RetryExhausted


class TestRetryDecorator:
    """Test retry decorator behavior."""

    def test_succeeds_first_attempt(self) -> None:
        """Should not retry if first call succeeds."""
        call_count = 0

        @retry(max_attempts=3)
        def always_succeeds() -> str:
            nonlocal call_count
            call_count += 1
            return "ok"

        result = always_succeeds()
        assert result == "ok"
        assert call_count == 1

    def test_succeeds_after_retries(self) -> None:
        """Should retry and eventually succeed."""
        call_count = 0

        @retry(max_attempts=3, base_delay=0.01)  # Fast for tests
        def fails_twice() -> str:
            nonlocal call_count
            call_count += 1
            if call_count < 3:
                raise ConnectionError("Temporary failure")
            return "recovered"

        result = fails_twice()
        assert result == "recovered"
        assert call_count == 3

    def test_exhausted_raises(self) -> None:
        """Should raise RetryExhausted after all attempts fail."""
        @retry(max_attempts=3, base_delay=0.01)
        def always_fails() -> None:
            raise ConnectionError("Permanent failure")

        with pytest.raises(RetryExhausted) as exc_info:
            always_fails()

        assert exc_info.value.attempts == 3
        assert isinstance(exc_info.value.last_exception, ConnectionError)

    def test_non_retryable_exception(self) -> None:
        """Should NOT retry non-retryable exceptions."""
        call_count = 0

        @retry(
            max_attempts=5,
            base_delay=0.01,
            retryable_exceptions=(ConnectionError,),
            non_retryable_exceptions=(ValueError,),
        )
        def raises_value_error() -> None:
            nonlocal call_count
            call_count += 1
            raise ValueError("Bad input")

        with pytest.raises(ValueError):
            raises_value_error()

        assert call_count == 1  # No retries — ValueError is non-retryable

    def test_preserves_function_metadata(self) -> None:
        """Decorator should preserve function name and docstring."""
        @retry(max_attempts=3)
        def my_function() -> None:
            """My docstring."""
            pass

        assert my_function.__name__ == "my_function"
        assert my_function.__doc__ == "My docstring."

    def test_on_retry_callback(self) -> None:
        """Should call on_retry callback between attempts."""
        retry_log: list[tuple[int, str, float]] = []

        def on_retry_cb(attempt: int, error: Exception, delay: float) -> None:
            retry_log.append((attempt, str(error), delay))

        @retry(max_attempts=3, base_delay=0.01, on_retry=on_retry_cb)
        def fails_once() -> str:
            if len(retry_log) == 0:
                raise ConnectionError("First try fails")
            return "ok"

        result = fails_once()
        assert result == "ok"
        assert len(retry_log) == 1
        assert retry_log[0][0] == 1  # First attempt
        assert "First try fails" in retry_log[0][1]
```

### Testing CLI Commands

```python
"""
tests/unit/test_cli_cost.py
"""
import json
from unittest.mock import MagicMock, patch

import pytest
from click.testing import CliRunner

from novatools.cli.main import cli


class TestCostWaste:
    """Test cost waste CLI command."""

    @pytest.fixture
    def runner(self) -> CliRunner:
        return CliRunner()

    @patch("novatools.cli.cost.EC2Service")
    def test_waste_volumes_table(self, mock_ec2_cls: MagicMock, runner: CliRunner) -> None:
        """Should display unattached volumes in table format."""
        mock_ec2 = mock_ec2_cls.return_value
        mock_ec2.find_unattached_volumes.return_value = [
            {
                "volume_id": "vol-abc123",
                "size_gb": 500,
                "volume_type": "gp3",
                "name": "old-data",
                "created": "2023-06-15T00:00:00",
                "monthly_cost_estimate": 40.0,
            }
        ]
        mock_ec2.find_old_snapshots.return_value = []

        result = runner.invoke(cli, ["cost", "waste", "--type", "all"])
        assert result.exit_code == 0
        assert "vol-abc123" in result.output or "vol-abc123" in result.stderr  # Rich renders to stderr

    @patch("novatools.cli.cost.EC2Service")
    def test_waste_volumes_json(self, mock_ec2_cls: MagicMock, runner: CliRunner) -> None:
        """Should output valid JSON in json format."""
        mock_ec2 = mock_ec2_cls.return_value
        mock_ec2.find_unattached_volumes.return_value = [
            {
                "volume_id": "vol-abc123",
                "size_gb": 100,
                "volume_type": "gp3",
                "name": "test",
                "created": "2023-01-01T00:00:00",
                "monthly_cost_estimate": 8.0,
            }
        ]

        result = runner.invoke(cli, ["cost", "waste", "--type", "volumes", "--format", "json"])
        assert result.exit_code == 0
        data = json.loads(result.output)
        assert len(data) == 1
        assert data[0]["volume_id"] == "vol-abc123"

    @patch("novatools.cli.cost.EC2Service")
    def test_waste_no_results(self, mock_ec2_cls: MagicMock, runner: CliRunner) -> None:
        """Should handle zero waste gracefully."""
        mock_ec2 = mock_ec2_cls.return_value
        mock_ec2.find_unattached_volumes.return_value = []
        mock_ec2.find_old_snapshots.return_value = []

        result = runner.invoke(cli, ["cost", "waste"])
        assert result.exit_code == 0
```

---

## 10. ASYNC PATTERNS

```python
"""
src/novatools/utils/async_utils.py

Async patterns for concurrent API operations.
"""
import asyncio
from collections.abc import Coroutine
from typing import Any, TypeVar

import aiohttp
import structlog

log = structlog.get_logger()

T = TypeVar("T")


async def gather_with_concurrency(
    limit: int,
    *tasks: Coroutine[Any, Any, T],
) -> list[T]:
    """
    Run tasks concurrently with a bounded concurrency limit.
    
    Why: asyncio.gather() runs ALL tasks at once.
    With 1000 API calls, you DOS the API.
    Semaphore bounds it to N concurrent calls.
    """
    semaphore = asyncio.Semaphore(limit)

    async def sem_task(task: Coroutine[Any, Any, T]) -> T:
        async with semaphore:
            return await task

    return await asyncio.gather(*(sem_task(t) for t in tasks))


async def check_endpoints(
    urls: list[str],
    timeout: float = 10.0,
    concurrency: int = 20,
) -> list[dict[str, Any]]:
    """
    Check multiple HTTP endpoints concurrently.
    Returns health status for each endpoint.
    """
    results: list[dict[str, Any]] = []

    async def check_one(session: aiohttp.ClientSession, url: str) -> dict[str, Any]:
        try:
            start = asyncio.get_event_loop().time()
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=timeout)) as resp:
                elapsed = asyncio.get_event_loop().time() - start
                body = await resp.text()
                return {
                    "url": url,
                    "status": resp.status,
                    "latency_ms": round(elapsed * 1000, 1),
                    "healthy": 200 <= resp.status < 300,
                    "body_preview": body[:200],
                }
        except asyncio.TimeoutError:
            return {"url": url, "status": 0, "healthy": False, "error": "timeout"}
        except aiohttp.ClientError as e:
            return {"url": url, "status": 0, "healthy": False, "error": str(e)}

    connector = aiohttp.TCPConnector(limit=concurrency, ttl_dns_cache=300)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [check_one(session, url) for url in urls]
        results = await asyncio.gather(*tasks)

    return results


# ─── Sync wrapper for CLI tools ───────────────────────────
def run_async(coro: Coroutine[Any, Any, T]) -> T:
    """
    Run async code from sync context.
    
    Why: Click commands are sync. boto3 is sync. But sometimes you need 
    async (concurrent HTTP checks, parallel API calls).
    This bridges the gap.
    """
    try:
        loop = asyncio.get_running_loop()
        # Already in an async context — shouldn't happen in CLI, but safety
        raise RuntimeError("Cannot use run_async inside an existing event loop")
    except RuntimeError:
        pass

    return asyncio.run(coro)


# ─── Usage in CLI ─────────────────────────────────────────
# @cost_group.command("health-check")
# def health_check():
#     urls = [
#         "https://api.novamart.com/health",
#         "https://payments.novamart.com/health",
#         ...
#     ]
#     results = run_async(check_endpoints(urls))
#     for r in results:
#         status = "✅" if r["healthy"] else "❌"
#         print(f"{status} {r['url']} — {r.get('latency_ms', 'N/A')}ms")


# ─── When NOT to use async ───────────────────────────────
# 1. boto3 is NOT async. Don't mix async with boto3.
#    Use ThreadPoolExecutor or aioboto3 (third-party, less mature).
# 2. If you're calling ONE API: async is overhead with zero benefit.
# 3. If you're CPU-bound: use multiprocessing, not asyncio.
# 4. If your team doesn't know async: use threading instead.
#    Wrong async code is worse than correct sync code.
```

**Async with boto3 — the right way:**

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
from typing import Any

import boto3

# boto3 is sync-only. Use ThreadPoolExecutor to parallelize.
async def get_all_instances_multi_region(
    regions: list[str],
) -> dict[str, list[dict[str, Any]]]:
    """Get instances from multiple regions concurrently."""
    loop = asyncio.get_event_loop()
    executor = ThreadPoolExecutor(max_workers=len(regions))

    def _get_instances(region: str) -> tuple[str, list[dict[str, Any]]]:
        client = boto3.client("ec2", region_name=region)
        paginator = client.get_paginator("describe_instances")
        instances = []
        for page in paginator.paginate():
            for reservation in page["Reservations"]:
                instances.extend(reservation["Instances"])
        return region, instances

    tasks = [
        loop.run_in_executor(executor, _get_instances, region)
        for region in regions
    ]
    results = await asyncio.gather(*tasks)
    return dict(results)
```

---

## 11. KUBERNETES CLIENT

```python
"""
src/novatools/k8s/client.py

Kubernetes Python client for NovaMart operations.
"""
from typing import Any

import structlog
from kubernetes import client, config
from kubernetes.client.rest import ApiException

log = structlog.get_logger()


def get_k8s_clients(
    context: str | None = None,
    in_cluster: bool = False,
) -> tuple[client.CoreV1Api, client.AppsV1Api, client.CustomObjectsApi]:
    """
    Initialize K8s API clients.
    
    in_cluster=True: Uses service account mounted at /var/run/secrets/kubernetes.io
    in_cluster=False: Uses ~/.kube/config (or KUBECONFIG env var)
    """
    if in_cluster:
        config.load_incluster_config()
    else:
        config.load_kube_config(context=context)

    core_v1 = client.CoreV1Api()
    apps_v1 = client.AppsV1Api()
    custom = client.CustomObjectsApi()

    return core_v1, apps_v1, custom


class K8sService:
    """Kubernetes operations for NovaMart."""

    def __init__(self, context: str | None = None, in_cluster: bool = False) -> None:
        self.core, self.apps, self.custom = get_k8s_clients(context, in_cluster)

    def get_pods(
        self,
        namespace: str,
        label_selector: str | None = None,
        field_selector: str | None = None,
    ) -> list[dict[str, Any]]:
        """Get pods with optional filters. Returns simplified dicts."""
        try:
            pods = self.core.list_namespaced_pod(
                namespace=namespace,
                label_selector=label_selector or "",
                field_selector=field_selector or "",
            )
        except ApiException as e:
            if e.status == 403:
                raise PermissionError(
                    f"No RBAC access to list pods in {namespace}. "
                    f"Check ServiceAccount/ClusterRoleBinding."
                ) from e
            raise

        results: list[dict[str, Any]] = []
        for pod in pods.items:
            container_statuses = pod.status.container_statuses or []
            results.append({
                "name": pod.metadata.name,
                "namespace": pod.metadata.namespace,
                "phase": pod.status.phase,
                "node": pod.spec.node_name,
                "ip": pod.status.pod_ip,
                "start_time": pod.status.start_time.isoformat() if pod.status.start_time else None,
                "containers": [
                    {
                        "name": cs.name,
                        "ready": cs.ready,
                        "restart_count": cs.restart_count,
                        "state": self._get_container_state(cs),
                        "last_termination_reason": (
                            cs.last_state.terminated.reason
                            if cs.last_state and cs.last_state.terminated
                            else None
                        ),
                    }
                    for cs in container_statuses
                ],
                "labels": dict(pod.metadata.labels or {}),
            })

        return results

    @staticmethod
    def _get_container_state(cs: Any) -> str:
        """Extract container state as string."""
        if cs.state.running:
            return "running"
        if cs.state.waiting:
            return f"waiting:{cs.state.waiting.reason or 'unknown'}"
        if cs.state.terminated:
            return f"terminated:{cs.state.terminated.reason or 'unknown'}"
        return "unknown"

    def get_deployment(self, name: str, namespace: str) -> dict[str, Any] | None:
        """Get deployment details."""
        try:
            dep = self.apps.read_namespaced_deployment(name, namespace)
        except ApiException as e:
            if e.status == 404:
                return None
            raise

        return {
            "name": dep.metadata.name,
            "namespace": dep.metadata.namespace,
            "replicas": dep.spec.replicas,
            "ready_replicas": dep.status.ready_replicas or 0,
            "updated_replicas": dep.status.updated_replicas or 0,
            "available_replicas": dep.status.available_replicas or 0,
            "image": dep.spec.template.spec.containers[0].image,
            "strategy": dep.spec.strategy.type,
            "conditions": [
                {
                    "type": c.type,
                    "status": c.status,
                    "reason": c.reason,
                    "message": c.message,
                }
                for c in (dep.status.conditions or [])
            ],
        }

    def scale_deployment(
        self,
        name: str,
        namespace: str,
        replicas: int,
    ) -> None:
        """Scale a deployment."""
        log.info("scaling_deployment", name=name, namespace=namespace, replicas=replicas)
        self.apps.patch_namespaced_deployment_scale(
            name,
            namespace,
            body={"spec": {"replicas": replicas}},
        )

    def get_events(
        self,
        namespace: str,
        field_selector: str | None = None,
        limit: int = 50,
    ) -> list[dict[str, Any]]:
        """Get recent events in a namespace."""
        events = self.core.list_namespaced_event(
            namespace=namespace,
            field_selector=field_selector or "",
            limit=limit,
        )

        results = []
        for event in sorted(events.items, key=lambda e: e.last_timestamp or e.event_time or "", reverse=True):
            results.append({
                "type": event.type,  # Normal, Warning
                "reason": event.reason,
                "message": event.message,
                "object": f"{event.involved_object.kind}/{event.involved_object.name}",
                "count": event.count,
                "last_seen": (event.last_timestamp or event.event_time or "").isoformat() 
                             if hasattr(event.last_timestamp or event.event_time or "", 'isoformat') 
                             else str(event.last_timestamp or event.event_time or ""),
            })

        return results[:limit]

    def get_resource_usage(self, namespace: str) -> dict[str, Any]:
        """Get resource requests/limits summary for a namespace."""
        pods = self.core.list_namespaced_pod(namespace=namespace)

        total_cpu_request = 0  # millicores
        total_cpu_limit = 0
        total_mem_request = 0  # bytes
        total_mem_limit = 0
        pod_count = 0

        for pod in pods.items:
            if pod.status.phase not in ("Running", "Pending"):
                continue
            pod_count += 1
            for container in pod.spec.containers:
                requests = container.resources.requests or {}
                limits = container.resources.limits or {}

                total_cpu_request += self._parse_cpu(requests.get("cpu", "0"))
                total_cpu_limit += self._parse_cpu(limits.get("cpu", "0"))
                total_mem_request += self._parse_memory(requests.get("memory", "0"))
                total_mem_limit += self._parse_memory(limits.get("memory", "0"))

        return {
            "namespace": namespace,
            "pod_count": pod_count,
            "cpu_request_millicores": total_cpu_request,
            "cpu_limit_millicores": total_cpu_limit,
            "memory_request_bytes": total_mem_request,
            "memory_limit_bytes": total_mem_limit,
        }

    @staticmethod
    def _parse_cpu(value: str) -> int:
        """Parse CPU value to millicores."""
        if value.endswith("m"):
            return int(value[:-1])
        try:
            return int(float(value) * 1000)
        except ValueError:
            return 0

    @staticmethod
    def _parse_memory(value: str) -> int:
        """Parse memory value to bytes."""
        units = {"Ki": 1024, "Mi": 1024**2, "Gi": 1024**3, "Ti": 1024**4}
        for suffix, multiplier in units.items():
            if value.endswith(suffix):
                return int(value[: -len(suffix)]) * multiplier
        try:
            return int(value)
        except ValueError:
            return 0
```

---

## 12. REAL NOVAMART TOOLS — Complete Working Scripts

### Tool 1: Secret Rotation Script

```python
"""
scripts/rotate_db_passwords.py

Rotate database passwords across all environments.

Steps:
1. Generate new password
2. Update in AWS Secrets Manager
3. Update RDS master password
4. Wait for RDS to apply
5. Verify application connectivity
6. Update External Secrets operator to sync

Usage:
    python scripts/rotate_db_passwords.py --environment production --service payments
    python scripts/rotate_db_passwords.py --environment production --service payments --dry-run
"""
import secrets
import string
import time
from typing import Any

import boto3
import click
import structlog
from botocore.exceptions import ClientError

log = structlog.get_logger()


def generate_password(length: int = 32) -> str:
    """Generate a cryptographically secure password."""
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*"
    # Exclude characters that cause issues in connection strings
    exclude = set("'\"\\`/;")
    safe_alphabet = "".join(c for c in alphabet if c not in exclude)

    while True:
        password = "".join(secrets.choice(safe_alphabet) for _ in range(length))
        # Ensure complexity requirements
        if (
            any(c.isupper() for c in password)
            and any(c.islower() for c in password)
            and any(c.isdigit() for c in password)
            and any(c in "!@#$%^&*" for c in password)
        ):
            return password


def rotate_rds_password(
    rds_client: Any,
    sm_client: Any,
    db_identifier: str,
    secret_id: str,
    dry_run: bool = False,
) -> None:
    """Rotate a single RDS instance password."""
    log_ctx = log.bind(db=db_identifier, secret=secret_id)

    # Step 1: Get current secret for backup
    log_ctx.info("fetching_current_secret")
    try:
        current = sm_client.get_secret_value(SecretId=secret_id)
        current_version = current["VersionId"]
    except ClientError as e:
        raise RuntimeError(f"Cannot read current secret {secret_id}: {e}") from e

    # Step 2: Generate new password
    new_password = generate_password()
    log_ctx.info("new_password_generated", length=len(new_password))
    # NEVER log the actual password

    if dry_run:
        log_ctx.info("dry_run_skip", action="would rotate password")
        return

    # Step 3: Update Secrets Manager FIRST (dual-write safe)
    # Why SM first? If RDS update fails, secret still has old password (matches RDS).
    # If RDS succeeds but SM fails, RDS has new password but nothing knows it = outage.
    # Solution: Write new password to SM as AWSPENDING, then update RDS, then promote.
    log_ctx.info("updating_secrets_manager_pending")
    try:
        sm_client.put_secret_value(
            SecretId=secret_id,
            SecretString=new_password,
            VersionStages=["AWSPENDING"],
        )
    except ClientError as e:
        raise RuntimeError(f"Failed to update secret {secret_id}: {e}") from e

    # Step 4: Update RDS master password
    log_ctx.info("updating_rds_password")
    try:
        rds_client.modify_db_instance(
            DBInstanceIdentifier=db_identifier,
            MasterUserPassword=new_password,
            ApplyImmediately=True,
        )
    except ClientError as e:
        # Rollback: Remove pending version
        log_ctx.error("rds_update_failed_rolling_back", error=str(e))
        try:
            sm_client.update_secret_version_stage(
                SecretId=secret_id,
                VersionStage="AWSPENDING",
                RemoveFromVersionId=current_version,
            )
        except ClientError:
            log_ctx.error("rollback_failed_manual_intervention_required")
        raise RuntimeError(f"Failed to update RDS password: {e}") from e

    # Step 5: Wait for RDS to apply the change
    log_ctx.info("waiting_for_rds_modification")
    waiter = rds_client.get_waiter("db_instance_available")
    try:
        waiter.wait(
            DBInstanceIdentifier=db_identifier,
            WaiterConfig={"Delay": 15, "MaxAttempts": 40},  # Up to 10 minutes
        )
    except Exception as e:
        log_ctx.error("rds_waiter_failed", error=str(e))
        raise

    # Step 6: Promote AWSPENDING to AWSCURRENT
    log_ctx.info("promoting_secret_to_current")
    try:
        # Get the version ID of the pending secret
        metadata = sm_client.describe_secret(SecretId=secret_id)
        pending_version = None
        for version_id, stages in metadata.get("VersionIdsToStages", {}).items():
            if "AWSPENDING" in stages:
                pending_version = version_id
                break

        if pending_version:
            sm_client.update_secret_version_stage(
                SecretId=secret_id,
                VersionStage="AWSCURRENT",
                MoveToVersionId=pending_version,
                RemoveFromVersionId=current_version,
            )
    except ClientError as e:
        raise RuntimeError(
            f"CRITICAL: RDS password changed but secret promotion failed for {secret_id}. "
            f"Manual intervention required! Error: {e}"
        ) from e

    log_ctx.info("rotation_complete")


def verify_connectivity(
    db_identifier: str,
    secret_id: str,
    sm_client: Any,
    region: str,
) -> bool:
    """Verify the application can connect with the new password."""
    import json

    log.info("verifying_connectivity", db=db_identifier)

    try:
        secret = sm_client.get_secret_value(SecretId=secret_id)
        # In production, this would attempt an actual DB connection:
        # import psycopg2
        # conn = psycopg2.connect(
        #     host=endpoint, port=port, dbname=dbname,
        #     user=username, password=secret["SecretString"],
        #     connect_timeout=5,
        # )
        # conn.close()
        log.info("connectivity_verified", db=db_identifier)
        return True
    except Exception as e:
        log.error("connectivity_check_failed", db=db_identifier, error=str(e))
        return False


@click.command()
@click.option("--environment", "-e", required=True,
              type=click.Choice(["dev", "staging", "production"]))
@click.option("--service", "-s", required=True,
              type=click.Choice(["payments", "orders", "users"]))
@click.option("--dry-run", is_flag=True, help="Show what would happen")
@click.option("--region", default="us-east-1")
@click.option("--skip-verify", is_flag=True, help="Skip connectivity verification")
def main(
    environment: str,
    service: str,
    dry_run: bool,
    region: str,
    skip_verify: bool,
) -> None:
    """Rotate database password for a NovaMart service."""
    from novatools.utils.logging import setup_structlog
    setup_structlog(level="INFO")

    # Service → RDS mapping
    db_map = {
        "payments": {
            "db_identifier": f"novamart-{environment}-payments",
            "secret_id": f"novamart/{environment}/rds/payments/master",
        },
        "orders": {
            "db_identifier": f"novamart-{environment}-orders",
            "secret_id": f"novamart/{environment}/rds/orders/master",
        },
        "users": {
            "db_identifier": f"novamart-{environment}-users",
            "secret_id": f"novamart/{environment}/rds/users/master",
        },
    }

    config = db_map[service]
    rds_client = boto3.client("rds", region_name=region)
    sm_client = boto3.client("secretsmanager", region_name=region)

    log.info(
        "starting_rotation",
        environment=environment,
        service=service,
        db=config["db_identifier"],
        dry_run=dry_run,
    )

    if environment == "production" and not dry_run:
        click.confirm(
            f"⚠️  You are about to rotate the {service} database password in PRODUCTION. Continue?",
            abort=True,
        )

    try:
        rotate_rds_password(
            rds_client=rds_client,
            sm_client=sm_client,
            db_identifier=config["db_identifier"],
            secret_id=config["secret_id"],
            dry_run=dry_run,
        )

        if not dry_run and not skip_verify:
            if not verify_connectivity(
                config["db_identifier"], config["secret_id"], sm_client, region
            ):
                log.error("post_rotation_verification_failed")
                click.echo("❌ Connectivity verification FAILED after rotation!", err=True)
                click.echo("Manual investigation required. The new password is in Secrets Manager.", err=True)
                raise SystemExit(1)

        click.echo(f"✅ Password rotation {'simulated' if dry_run else 'completed'} for {service} in {environment}")

    except Exception as e:
        log.exception("rotation_failed")
        click.echo(f"❌ Rotation failed: {e}", err=True)
        raise SystemExit(1)


if __name__ == "__main__":
    main()
```

### Tool 2: Compliance Audit Script

```python
"""
scripts/compliance_audit.py

Automated compliance checks for NovaMart infrastructure.
Checks IAM, Security Groups, Encryption, and tagging compliance.

Usage:
    novatools compliance audit --checks all
    novatools compliance audit --checks iam,encryption --format json
"""
import datetime
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

import boto3
import click
import structlog
from rich.console import Console
from rich.table import Table
from rich.panel import Panel

log = structlog.get_logger()
console = Console(stderr=True)


class Severity(str, Enum):
    CRITICAL = "CRITICAL"
    HIGH = "HIGH"
    MEDIUM = "MEDIUM"
    LOW = "LOW"


class Status(str, Enum):
    PASS = "PASS"
    FAIL = "FAIL"
    WARN = "WARN"


@dataclass
class Finding:
    check_id: str
    title: str
    severity: Severity
    status: Status
    resource: str
    detail: str
    remediation: str


@dataclass
class AuditReport:
    timestamp: str = field(default_factory=lambda: datetime.datetime.now(datetime.UTC).isoformat())
    findings: list[Finding] = field(default_factory=list)

    @property
    def critical_count(self) -> int:
        return sum(1 for f in self.findings if f.severity == Severity.CRITICAL and f.status == Status.FAIL)

    @property
    def high_count(self) -> int:
        return sum(1 for f in self.findings if f.severity == Severity.HIGH and f.status == Status.FAIL)

    @property
    def pass_count(self) -> int:
        return sum(1 for f in self.findings if f.status == Status.PASS)

    @property
    def fail_count(self) -> int:
        return sum(1 for f in self.findings if f.status == Status.FAIL)


class IAMAuditor:
    """IAM compliance checks."""

    def __init__(self, session: boto3.Session) -> None:
        self.iam = session.client("iam")

    def check_root_mfa(self, report: AuditReport) -> None:
        """CIS 1.5: Ensure MFA is enabled for root account."""
        summary = self.iam.get_account_summary()
        mfa_enabled = summary["SummaryMap"].get("AccountMFAEnabled", 0)

        report.findings.append(Finding(
            check_id="IAM-001",
            title="Root account MFA",
            severity=Severity.CRITICAL,
            status=Status.PASS if mfa_enabled else Status.FAIL,
            resource="arn:aws:iam::root",
            detail=f"Root MFA enabled: {bool(mfa_enabled)}",
            remediation="Enable MFA on root account via IAM console",
        ))

    def check_unused_credentials(self, report: AuditReport, max_age_days: int = 90) -> None:
        """CIS 1.3: Ensure credentials unused for 90 days are disabled."""
        credential_report = self._get_credential_report()
        cutoff = datetime.datetime.now(datetime.UTC) - datetime.timedelta(days=max_age_days)

        for user in credential_report:
            username = user["user"]
            if username == "<root_account>":
                continue

            # Check access key age
            for key_num in ["1", "2"]:
                active = user.get(f"access_key_{key_num}_active", "false") == "true"
                last_used = user.get(f"access_key_{key_num}_last_used_date", "N/A")

                if active and last_used != "N/A":
                    try:
                        last_used_dt = datetime.datetime.fromisoformat(
                            last_used.replace("Z", "+00:00")
                        )
                        if last_used_dt < cutoff:
                            report.findings.append(Finding(
                                check_id="IAM-002",
                                title=f"Unused access key ({username})",
                                severity=Severity.HIGH,
                                status=Status.FAIL,
                                resource=f"arn:aws:iam::user/{username}",
                                detail=f"Access key {key_num} last used {last_used}",
                                remediation=f"Disable or delete access key {key_num} for {username}",
                            ))
                    except (ValueError, TypeError):
                        pass

            # Check password age
            password_last_used = user.get("password_last_used", "N/A")
            if password_last_used not in ("N/A", "no_information", "not_supported"):
                try:
                    pwd_dt = datetime.datetime.fromisoformat(
                        password_last_used.replace("Z", "+00:00")
                    )
                    if pwd_dt < cutoff:
                        report.findings.append(Finding(
                            check_id="IAM-003",
                            title=f"Inactive console user ({username})",
                            severity=Severity.MEDIUM,
                            status=Status.FAIL,
                            resource=f"arn:aws:iam::user/{username}",
                            detail=f"Password last used {password_last_used}",
                            remediation=f"Review and potentially disable console access for {username}",
                        ))
                except (ValueError, TypeError):
                    pass

    def check_admin_policies(self, report: AuditReport) -> None:
        """Check for users/roles with full admin access."""
        # Check IAM users with AdministratorAccess
        paginator = self.iam.get_paginator("list_users")
        for page in paginator.paginate():
            for user in page["Users"]:
                username = user["UserName"]
                attached = self.iam.list_attached_user_policies(UserName=username)

                for policy in attached["AttachedPolicies"]:
                    if policy["PolicyArn"] == "arn:aws:iam::aws:policy/AdministratorAccess":
                        report.findings.append(Finding(
                            check_id="IAM-004",
                            title=f"User with AdministratorAccess ({username})",
                            severity=Severity.HIGH,
                            status=Status.FAIL,
                            resource=f"arn:aws:iam::user/{username}",
                            detail="User has full admin access via managed policy",
                            remediation=f"Apply least-privilege policy to {username}",
                        ))

    def _get_credential_report(self) -> list[dict[str, str]]:
        """Generate and parse IAM credential report."""
        import csv
        import io
        import time

        # Generate report
        self.iam.generate_credential_report()
        time.sleep(2)  # Report generation takes a moment

        for _ in range(10):
            try:
                response = self.iam.get_credential_report()
                content = response["Content"].decode("utf-8")
                reader = csv.DictReader(io.StringIO(content))
                return list(reader)
            except self.iam.exceptions.CredentialReportNotReadyException:
                time.sleep(2)

        return []


class SecurityGroupAuditor:
    """Security Group compliance checks."""

    def __init__(self, session: boto3.Session) -> None:
        self.ec2 = session.client("ec2")

    def check_open_ingress(self, report: AuditReport) -> None:
        """Check for security groups with 0.0.0.0/0 ingress on dangerous ports."""
        dangerous_ports = {22, 3389, 3306, 5432, 6379, 27017, 9200}

        paginator = self.ec2.get_paginator("describe_security_groups")
        for page in paginator.paginate():
            for sg in page["SecurityGroups"]:
                sg_id = sg["GroupId"]
                sg_name = sg["GroupName"]

                for rule in sg.get("IpPermissions", []):
                    from_port = rule.get("FromPort", 0)
                    to_port = rule.get("ToPort", 65535)

                    for ip_range in rule.get("IpRanges", []):
                        cidr = ip_range.get("CidrIp", "")
                        if cidr in ("0.0.0.0/0", "::/0"):
                            # Check if any dangerous port is in range
                            exposed_dangerous = [
                                p for p in dangerous_ports
                                if from_port <= p <= to_port
                            ]
                            if exposed_dangerous:
                                report.findings.append(Finding(
                                    check_id="SG-001",
                                    title=f"Open ingress on dangerous port(s)",
                                    severity=Severity.CRITICAL,
                                    status=Status.FAIL,
                                    resource=f"{sg_id} ({sg_name})",
                                    detail=f"Ports {exposed_dangerous} open to {cidr}",
                                    remediation=f"Restrict {sg_id} ingress to specific CIDRs",
                                ))
                            elif from_port == 0 and to_port == 65535:
                                report.findings.append(Finding(
                                    check_id="SG-002",
                                    title="All ports open to internet",
                                    severity=Severity.CRITICAL,
                                    status=Status.FAIL,
                                    resource=f"{sg_id} ({sg_name})",
                                    detail=f"All ports open to {cidr}",
                                    remediation=f"Restrict {sg_id} to specific ports and CIDRs",
                                ))


class EncryptionAuditor:
    """Encryption compliance checks."""

    def __init__(self, session: boto3.Session) -> None:
        self.ec2 = session.client("ec2")
        self.rds = session.client("rds")
        self.s3 = session.client("s3")

    def check_ebs_encryption(self, report: AuditReport) -> None:
        """Check that EBS encryption by default is enabled."""
        response = self.ec2.get_ebs_encryption_by_default()
        enabled = response["EbsEncryptionByDefault"]

        report.findings.append(Finding(
            check_id="ENC-001",
            title="EBS encryption by default",
            severity=Severity.HIGH,
            status=Status.PASS if enabled else Status.FAIL,
            resource="account-level",
            detail=f"EBS encryption by default: {enabled}",
            remediation="Enable EBS encryption by default via EC2 settings",
        ))

    def check_rds_encryption(self, report: AuditReport) -> None:
        """Check all RDS instances are encrypted."""
        paginator = self.rds.get_paginator("describe_db_instances")
        for page in paginator.paginate():
            for db in page["DBInstances"]:
                db_id = db["DBInstanceIdentifier"]
                encrypted = db.get("StorageEncrypted", False)

                report.findings.append(Finding(
                    check_id="ENC-002",
                    title=f"RDS encryption ({db_id})",
                    severity=Severity.CRITICAL,
                    status=Status.PASS if encrypted else Status.FAIL,
                    resource=db_id,
                    detail=f"Storage encrypted: {encrypted}",
                    remediation=f"Enable encryption for {db_id} (requires snapshot + restore)",
                ))

    def check_s3_encryption(self, report: AuditReport) -> None:
        """Check S3 buckets have default encryption."""
        buckets = self.s3.list_buckets()["Buckets"]

        for bucket in buckets:
            name = bucket["Name"]
            try:
                enc = self.s3.get_bucket_encryption(Bucket=name)
                rules = enc["ServerSideEncryptionConfiguration"]["Rules"]
                algorithm = rules[0]["ApplyServerSideEncryptionByDefault"]["SSEAlgorithm"]
                status = Status.PASS
                detail = f"Encryption: {algorithm}"
            except self.s3.exceptions.ClientError:
                status = Status.FAIL
                detail = "No default encryption configured"
                algorithm = "none"

            report.findings.append(Finding(
                check_id="ENC-003",
                title=f"S3 bucket encryption ({name})",
                severity=Severity.HIGH,
                status=status,
                resource=name,
                detail=detail,
                remediation=f"Enable default encryption (AES256 or aws:kms) for {name}",
            ))


def render_report(report: AuditReport, fmt: str) -> None:
    """Render audit report in requested format."""
    if fmt == "json":
        import json
        data = {
            "timestamp": report.timestamp,
            "summary": {
                "total": len(report.findings),
                "pass": report.pass_count,
                "fail": report.fail_count,
                "critical": report.critical_count,
                "high": report.high_count,
            },
            "findings": [
                {
                    "check_id": f.check_id,
                    "title": f.title,
                    "severity": f.severity.value,
                    "status": f.status.value,
                    "resource": f.resource,
                    "detail": f.detail,
                    "remediation": f.remediation,
                }
                for f in report.findings
            ],
        }
        click.echo(json.dumps(data, indent=2))
        return

    # Rich table output
    severity_colors = {
        Severity.CRITICAL: "bold red",
        Severity.HIGH: "red",
        Severity.MEDIUM: "yellow",
        Severity.LOW: "blue",
    }

    status_icons = {
        Status.PASS: "[green]✅ PASS[/green]",
        Status.FAIL: "[red]❌ FAIL[/red]",
        Status.WARN: "[yellow]⚠️  WARN[/yellow]",
    }

    # Summary panel
    summary = (
        f"Total checks: {len(report.findings)}\n"
        f"[green]Pass: {report.pass_count}[/green]  "
        f"[red]Fail: {report.fail_count}[/red]  "
        f"[bold red]Critical: {report.critical_count}[/bold red]  "
        f"[red]High: {report.high_count}[/red]"
    )
    console.print(Panel(summary, title="📊 Audit Summary"))

    # Failures only table
    failures = [f for f in report.findings if f.status != Status.PASS]
    if failures:
        table = Table(title="❌ Findings Requiring Action", show_lines=True)
        table.add_column("ID", style="dim", width=10)
        table.add_column("Severity", width=10)
        table.add_column("Title", width=35)
        table.add_column("Resource", width=30)
        table.add_column("Detail")
        table.add_column("Remediation", style="green")

        for f in sorted(failures, key=lambda x: list(Severity).index(x.severity)):
            sev_style = severity_colors.get(f.severity, "white")
            table.add_row(
                f.check_id,
                f"[{sev_style}]{f.severity.value}[/{sev_style}]",
                f.title,
                f.resource,
                f.detail,
                f.remediation,
            )

        console.print(table)
    else:
        console.print("[bold green]🎉 All checks passed![/bold green]")


@click.command("audit")
@click.option("--checks", default="all",
              help="Comma-separated: iam,sg,encryption,all")
@click.option("--format", "fmt", type=click.Choice(["table", "json"]), default="table")
@click.option("--region", default="us-east-1")
@click.option("--profile", default=None)
def audit(checks: str, fmt: str, region: str, profile: str | None) -> None:
    """Run compliance audit against AWS account."""
    from novatools.utils.logging import setup_structlog
    setup_structlog(level="INFO")

    session = boto3.Session(profile_name=profile, region_name=region)
    report = AuditReport()

    check_list = checks.split(",") if checks != "all" else ["iam", "sg", "encryption"]

    if "iam" in check_list:
        console.print("[bold]Running IAM checks...[/bold]")
        auditor = IAMAuditor(session)
        auditor.check_root_mfa(report)
        auditor.check_unused_credentials(report)
        auditor.check_admin_policies(report)

    if "sg" in check_list:
        console.print("[bold]Running Security Group checks...[/bold]")
        auditor = SecurityGroupAuditor(session)
        auditor.check_open_ingress(report)

    if "encryption" in check_list:
        console.print("[bold]Running Encryption checks...[/bold]")
        auditor = EncryptionAuditor(session)
        auditor.check_ebs_encryption(report)
        auditor.check_rds_encryption(report)
        auditor.check_s3_encryption(report)

    render_report(report, fmt)

    # Exit code based on findings
    if report.critical_count > 0:
        raise SystemExit(2)
    if report.high_count > 0:
        raise SystemExit(1)


if __name__ == "__main__":
    audit()
```

---

## 13. PYTHON FAILURE MODES AND ANTI-PATTERNS

```python
# ═══════════════════════════════════════════════════════════
# FAILURE MODE 1: Mutable Default Arguments
# ═══════════════════════════════════════════════════════════
# BAD:
def add_tag(resource: str, tags: list[str] = []) -> list[str]:
    tags.append(resource)
    return tags

add_tag("a")  # ["a"]
add_tag("b")  # ["a", "b"] ← WHAT?! The default list is SHARED across calls!

# GOOD:
def add_tag(resource: str, tags: list[str] | None = None) -> list[str]:
    tags = tags or []  # New list every time if None
    # OR: tags = list(tags) if tags else []  # Defensive copy
    tags.append(resource)
    return tags


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 2: Bare except
# ═══════════════════════════════════════════════════════════
# BAD:
try:
    deploy()
except:  # Catches EVERYTHING including KeyboardInterrupt, SystemExit
    log.error("failed")  # Ctrl+C won't stop this script!

# ALSO BAD:
try:
    deploy()
except Exception:
    pass  # Silently swallows ALL errors. You'll never know it broke.

# GOOD:
try:
    deploy()
except (ConnectionError, TimeoutError) as e:
    log.error("deploy_failed", error=str(e))
    raise DeploymentError(f"Deploy failed: {e}") from e


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 3: Not closing resources
# ═══════════════════════════════════════════════════════════
# BAD:
f = open("data.json")
data = json.load(f)
# If json.load() raises, file handle leaks. Do this 10000 times = fd exhaustion.

# GOOD:
with open("data.json") as f:
    data = json.load(f)
# File closed automatically even on exception.

# Same pattern for DB connections, HTTP sessions, temp files:
import tempfile
with tempfile.NamedTemporaryFile(suffix=".json", delete=False) as tf:
    tf.write(data)
    temp_path = tf.name  # Use after context manager closes the file


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 4: String concatenation in loops
# ═══════════════════════════════════════════════════════════
# BAD (O(n²) — creates new string every iteration):
result = ""
for line in huge_log_file:
    result += line  # 1M lines = 1M string allocations

# GOOD (O(n)):
parts: list[str] = []
for line in huge_log_file:
    parts.append(line)
result = "".join(parts)

# ALSO GOOD (generator):
result = "".join(line for line in huge_log_file)


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 5: Import-time side effects
# ═══════════════════════════════════════════════════════════
# BAD (in a module):
import boto3
client = boto3.client("ec2")  # Runs at IMPORT TIME
# If AWS creds aren't configured, importing this module crashes.
# Tests can't mock it. Other modules that import this fail.

# GOOD:
import boto3

def get_client():
    return boto3.client("ec2")
# OR: use class with lazy initialization (see EC2Service above)


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 6: Not handling pagination
# ═══════════════════════════════════════════════════════════
# BAD (silent data loss):
instances = ec2.describe_instances()["Reservations"]
# Returns ONLY first page. If you have 1500 instances, you see 1000.
# No error. No warning. Silent data loss.

# GOOD:
instances = paginate(ec2, "describe_instances", "Reservations")


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 7: datetime timezone confusion
# ═══════════════════════════════════════════════════════════
# BAD:
from datetime import datetime
now = datetime.now()  # Naive datetime — no timezone
aws_time = instance["LaunchTime"]  # Aware datetime (has UTC)
diff = now - aws_time  # TypeError: can't subtract offset-naive and offset-aware

# GOOD:
from datetime import datetime, UTC
now = datetime.now(UTC)  # Always use timezone-aware
diff = now - aws_time  # Works correctly

# ALSO: Never use datetime.utcnow() — it's naive despite the name!
# datetime.utcnow() is DEPRECATED in Python 3.12.


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 8: requests without timeout
# ═══════════════════════════════════════════════════════════
# BAD:
response = requests.get("https://api.example.com/data")
# No timeout! If server hangs, this blocks FOREVER.
# Your CronJob never completes. On-call gets paged for a stuck job.

# GOOD:
response = requests.get("https://api.example.com/data", timeout=(5, 30))
# (connect_timeout=5s, read_timeout=30s)


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 9: Credential leak in logs or exceptions
# ═══════════════════════════════════════════════════════════
# BAD:
log.info(f"Connecting to DB with password={password}")
log.error(f"Request failed: {response.text}")  # Response may contain tokens

# BAD:
raise Exception(f"Auth failed with token {api_token}")  # Token in stack trace!

# GOOD:
log.info("connecting_to_db", host=host, port=port)  # No password
log.error("request_failed", status=response.status_code)  # No body
raise Exception("Auth failed — check API token validity")  # No token value


# ═══════════════════════════════════════════════════════════
# FAILURE MODE 10: Global state and thread safety
# ═══════════════════════════════════════════════════════════
# BAD:
_cache = {}  # Module-level mutable state
def get_instance(instance_id):
    if instance_id not in _cache:
        _cache[instance_id] = ec2.describe_instances(...)
    return _cache[instance_id]
# Race condition with threads. Cache never invalidates.

# GOOD:
from functools import lru_cache

@lru_cache(maxsize=256)
def get_instance(instance_id: str) -> dict:
    return ec2.describe_instances(...)

# Or with TTL:
from cachetools import TTLCache
_cache = TTLCache(maxsize=256, ttl=300)  # 5 minute TTL
```

---

## QUICK REFERENCE CARD

```
╔══════════════════════════════════════════════════════════════╗
║             PYTHON FOR DEVOPS — CHEAT SHEET                  ║
╠══════════════════════════════════════════════════════════════╣
║ PROJECT:                                                     ║
║   src/ layout + pyproject.toml + ruff + mypy --strict        ║
║   pip install -e ".[dev]"                                    ║
║                                                              ║
║ TYPES (non-negotiable):                                      ║
║   def func(x: str) -> int: ...                               ║
║   x: str | None          Optional                            ║
║   x: list[str]           Generic collections                 ║
║   TypedDict              Structured dicts (lint-time)        ║
║   Pydantic BaseModel     Validated models (runtime)          ║
║                                                              ║
║ BOTO3:                                                       ║
║   Config(retries={"mode":"adaptive"})  Always set            ║
║   paginator.paginate()    NEVER manual next_token            ║
║   waiter.wait()           Don't poll manually                ║
║   ClientError → .response["Error"]["Code"]                   ║
║   Session per thread, Client is thread-safe                  ║
║                                                              ║
║ HTTP:                                                        ║
║   requests.Session() + HTTPAdapter(max_retries=Retry(...))   ║
║   ALWAYS set timeout=(connect, read)                         ║
║   session.post(url, json=payload)   Not data=                ║
║                                                              ║
║ CLI:                                                         ║
║   Click: @click.group/@click.command/@click.option           ║
║   Rich:  Console, Table, Panel, Progress                     ║
║   Multiple output: --format table|json|csv                   ║
║   click.echo() for stdout (pipeable)                         ║
║   console.print() for stderr (human-readable)                ║
║                                                              ║
║ TESTING:                                                     ║
║   @mock_aws           moto for AWS                           ║
║   @responses.activate responses for HTTP                     ║
║   CliRunner()         Click CLI testing                      ║
║   pytest -x --cov     First failure + coverage               ║
║                                                              ║
║ ERROR HANDLING:                                              ║
║   Catch specific exceptions, not Exception                   ║
║   raise NewError("msg") from original_error                  ║
║   Use context managers for cleanup                           ║
║   Log errors, never log secrets                              ║
║                                                              ║
║ ANTI-PATTERNS:                                               ║
║   ✗ def f(x=[]):       → def f(x=None): x = x or []          ║
║   ✗ except:            → except SpecificError:               ║
║   ✗ except Exception: pass → at minimum log it               ║
║   ✗ requests.get(url)  → requests.get(url, timeout=(5,30))   ║
║   ✗ datetime.now()     → datetime.now(UTC)                   ║
║   ✗ datetime.utcnow()  → datetime.now(UTC) (utcnow naive!)   ║
║   ✗ f"token={secret}"  → never log/raise with secrets        ║
║   ✗ describe_instances()["R"] → paginate()                   ║
╚══════════════════════════════════════════════════════════════╝
```

---

## RETENTION QUESTIONS

**Q1 (Architecture — Design a Tool):**

You're asked to build `novatools secrets audit` — a CLI command that:
1. Scans all secrets in AWS Secrets Manager across 3 regions
2. For each secret: checks last rotation date, last accessed date, and whether it's referenced by any running EKS pod (via External Secrets Operator)
3. Flags secrets not rotated in >90 days, secrets never accessed in >180 days, and orphaned secrets (not referenced by any ExternalSecret CR)
4. Outputs a table (Rich) or JSON, and sends a Slack summary of findings

Design the complete solution. Show: module structure (which files), the main function signature, the data model (Pydantic or dataclass), the cross-region concurrency approach, how you'd test it (what you mock), and the Click command definition. You don't need to write every line of implementation, but the architecture must be production-complete.

---

**Q2 (Debugging — What's Wrong?):**

Your teammate wrote this boto3 code and it's been running in production for 3 months. Last week, the team added 200 new EC2 instances (total now: 1,400). Suddenly the nightly inventory report shows only ~1,000 instances. No errors in logs. Find every bug:

```python
import boto3
import json
from datetime import datetime

def get_all_instances(region="us-east-1"):
    ec2 = boto3.client("ec2", region_name=region)
    response = ec2.describe_instances(
        Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
    )
    
    instances = []
    for reservation in response["Reservations"]:
        for instance in reservation["Instances"]:
            instances.append({
                "id": instance["InstanceId"],
                "type": instance["InstanceType"],
                "launched": str(instance["LaunchTime"]),
                "name": [t["Value"] for t in instance["Tags"] if t["Key"] == "Name"][0],
                "age_days": (datetime.now() - instance["LaunchTime"]).days,
            })
    
    return instances

def save_report(instances):
    with open("/tmp/inventory.json", "w") as f:
        json.dump(instances, f, indent=2)
    
    print(f"Saved {len(instances)} instances")

if __name__ == "__main__":
    regions = ["us-east-1", "us-west-2", "eu-west-1"]
    all_instances = []
    for region in regions:
        all_instances.extend(get_all_instances(region))
    save_report(all_instances)
```

---

**Q3 (Production Code — Write It):**

Write a complete, testable Python function (with type hints, error handling, retry logic, and logging) that:

1. Takes a list of K8s namespaces and a Prometheus URL
2. For each namespace, queries Prometheus for the current error rate: `sum(rate(http_requests_total{namespace="X",code=~"5.."}[5m])) / sum(rate(http_requests_total{namespace="X"}[5m]))`
3. Returns results sorted by error rate (highest first)
4. Handles: Prometheus unreachable, empty results, division by zero, partial failures (some namespaces fail, others succeed)
5. Include the moto/responses-based test for this function

---

**Q4 (Concurrency — Design Decision):**

You need to check the health of 500 internal service endpoints. Each check is an HTTP GET to `/health` with a 5-second timeout. You need results in under 30 seconds.

Compare three approaches:
1. Sequential `requests.get()` in a for loop
2. `ThreadPoolExecutor` with `requests`
3. `asyncio` + `aiohttp`

For each: calculate the expected wall-clock time, explain memory/CPU implications, identify failure modes specific to that approach, and state which you'd choose for NovaMart and why.

# Q1 — Architecture: `novatools secrets audit`

## Module Structure

```
novatools/
├── cli/
│   ├── __init__.py
│   └── secrets_cmd.py              # Click command group + `audit` subcommand
├── secrets/
│   ├── __init__.py
│   ├── models.py                   # Pydantic data models
│   ├── scanner.py                  # AWS Secrets Manager scanning (per-region)
│   ├── k8s_resolver.py            # ExternalSecret CR cross-referencing
│   ├── analyzer.py                 # Policy engine (rotation, access, orphan checks)
│   ├── reporter.py                 # Rich table + JSON output formatting
│   └── notifier.py                 # Slack summary dispatch
├── common/
│   ├── aws.py                      # Shared boto3 session factory, retry config
│   ├── concurrency.py              # Cross-region concurrent execution wrapper
│   └── config.py                   # Settings via pydantic-settings
tests/
├── secrets/
│   ├── test_scanner.py
│   ├── test_k8s_resolver.py
│   ├── test_analyzer.py
│   ├── test_reporter.py
│   ├── test_notifier.py
│   └── test_integration.py         # End-to-end with all mocks wired together
├── conftest.py                     # Shared fixtures (moto, fake k8s, etc.)
└── fixtures/
    ├── secrets_manager_response.json
    └── external_secrets_crs.yaml
```

## Data Models (`models.py`)

```python
from __future__ import annotations

import enum
from datetime import datetime
from pydantic import BaseModel, Field


class FindingType(str, enum.Enum):
    ROTATION_OVERDUE = "rotation_overdue"
    NEVER_ACCESSED = "never_accessed"
    STALE_ACCESS = "stale_access"
    ORPHANED = "orphaned"


class Finding(BaseModel):
    finding_type: FindingType
    message: str
    severity: str = "warning"  # warning | critical


class SecretReference(BaseModel):
    """A Kubernetes ExternalSecret that references this AWS secret."""
    namespace: str
    external_secret_name: str
    cluster: str
    target_secret_name: str


class SecretAuditRecord(BaseModel):
    """Core domain object — one per AWS secret, per region."""
    arn: str
    name: str
    region: str
    created_date: datetime
    last_rotated_date: datetime | None = None
    last_accessed_date: datetime | None = None
    rotation_enabled: bool = False
    rotation_lambda_arn: str | None = None
    days_since_rotation: int | None = None
    days_since_access: int | None = None
    tags: dict[str, str] = Field(default_factory=dict)

    # Cross-reference results (populated by k8s_resolver)
    k8s_references: list[SecretReference] = Field(default_factory=list)
    is_orphaned: bool = False

    # Findings (populated by analyzer)
    findings: list[Finding] = Field(default_factory=list)

    @property
    def has_findings(self) -> bool:
        return len(self.findings) > 0


class AuditSummary(BaseModel):
    """Aggregate summary for the Slack notification."""
    total_secrets: int = 0
    total_findings: int = 0
    rotation_overdue: int = 0
    stale_access: int = 0
    orphaned: int = 0
    regions_scanned: list[str] = Field(default_factory=list)
    scan_duration_seconds: float = 0.0
    errors: list[str] = Field(default_factory=list)

    # Per-region breakdown
    by_region: dict[str, int] = Field(default_factory=dict)


class AuditResult(BaseModel):
    """Top-level result object returned by the audit pipeline."""
    records: list[SecretAuditRecord] = Field(default_factory=list)
    summary: AuditSummary = Field(default_factory=AuditSummary)
    generated_at: datetime = Field(default_factory=datetime.utcnow)
```

## Concurrency Approach (`concurrency.py`)

```python
"""
Cross-region concurrent execution using asyncio + ThreadPoolExecutor.

Why this hybrid:
- boto3 is synchronous (not natively async). Running it in threads is the
  pragmatic choice (vs. adopting aioboto3 and its maturity gaps).
- asyncio as the orchestrator gives us structured concurrency: fan-out across
  regions, fan-in results, first-class timeout/cancellation.
- K8s API calls (via kubernetes client) are also sync → same thread pool.
- We bound the thread pool to avoid socket/connection exhaustion against AWS.

Alternative considered: multiprocessing — rejected because we're I/O-bound
(API calls), not CPU-bound. Threads + asyncio is lighter and shares memory
for the result aggregation.
"""

from __future__ import annotations

import asyncio
from concurrent.futures import ThreadPoolExecutor
from typing import TypeVar, Callable, Sequence
import logging

T = TypeVar("T")
logger = logging.getLogger(__name__)

DEFAULT_MAX_WORKERS = 10  # 3 regions × ~3 concurrent API calls each


async def run_across_regions(
    regions: Sequence[str],
    fn: Callable[[str], T],
    max_workers: int = DEFAULT_MAX_WORKERS,
    timeout: float = 300.0,
) -> dict[str, T | Exception]:
    """
    Execute `fn(region)` concurrently across all regions.
    Returns {region: result_or_exception}.
    Partial failures are captured, never raised — the caller decides policy.
    """
    loop = asyncio.get_running_loop()
    results: dict[str, T | Exception] = {}

    with ThreadPoolExecutor(max_workers=max_workers) as pool:
        tasks = {
            region: loop.run_in_executor(pool, fn, region)
            for region in regions
        }

        for region, task in tasks.items():
            try:
                results[region] = await asyncio.wait_for(task, timeout=timeout)
                logger.info("Region %s completed successfully", region)
            except asyncio.TimeoutError:
                results[region] = TimeoutError(
                    f"Region {region} timed out after {timeout}s"
                )
                logger.error("Region %s timed out", region)
            except Exception as exc:
                results[region] = exc
                logger.error("Region %s failed: %s", region, exc, exc_info=True)

    return results
```

## Scanner (`scanner.py`)

```python
"""Scans AWS Secrets Manager for a single region. Called per-region by the concurrency layer."""

from __future__ import annotations

import logging
from datetime import datetime, timezone

import boto3
from botocore.config import Config as BotoConfig

from novatools.secrets.models import SecretAuditRecord

logger = logging.getLogger(__name__)

BOTO_RETRY_CONFIG = BotoConfig(
    retries={"max_attempts": 5, "mode": "adaptive"},
    connect_timeout=10,
    read_timeout=30,
)


def scan_region(region: str) -> list[SecretAuditRecord]:
    """
    List all secrets in a region, then describe each to get rotation/access metadata.
    Uses paginator to handle >100 secrets.
    """
    client = boto3.client(
        "secretsmanager", region_name=region, config=BOTO_RETRY_CONFIG
    )
    records: list[SecretAuditRecord] = []
    now = datetime.now(timezone.utc)

    paginator = client.get_paginator("list_secrets")
    for page in paginator.paginate(
        PaginationConfig={"PageSize": 100},
        IncludePlannedDeletion=False,
    ):
        for entry in page.get("SecretList", []):
            last_rotated = entry.get("LastRotatedDate")
            last_accessed = entry.get("LastAccessedDate")

            record = SecretAuditRecord(
                arn=entry["ARN"],
                name=entry["Name"],
                region=region,
                created_date=entry["CreatedDate"],
                last_rotated_date=last_rotated,
                last_accessed_date=last_accessed,
                rotation_enabled=entry.get("RotationEnabled", False),
                rotation_lambda_arn=entry.get("RotationLambdaARN"),
                days_since_rotation=(
                    (now - last_rotated).days if last_rotated else None
                ),
                days_since_access=(
                    (now - last_accessed).days if last_accessed else None
                ),
                tags={
                    t["Key"]: t["Value"]
                    for t in entry.get("Tags", [])
                },
            )
            records.append(record)

    logger.info("Region %s: found %d secrets", region, len(records))
    return records
```

## K8s Cross-Reference (`k8s_resolver.py`)

```python
"""
Resolves which AWS secrets are referenced by ExternalSecret CRs across EKS clusters.
"""

from __future__ import annotations

import logging
from kubernetes import client as k8s_client, config as k8s_config

from novatools.secrets.models import SecretAuditRecord, SecretReference

logger = logging.getLogger(__name__)


def load_external_secret_refs(
    kubeconfig_contexts: list[str] | None = None,
) -> dict[str, list[SecretReference]]:
    """
    Returns a mapping: aws_secret_name -> [SecretReference, ...]
    Scans all namespaces for ExternalSecret custom resources.
    """
    ref_map: dict[str, list[SecretReference]] = {}

    contexts = kubeconfig_contexts or _discover_eks_contexts()

    for ctx in contexts:
        try:
            k8s_config.load_kube_config(context=ctx)
            api = k8s_client.CustomObjectsApi()

            # ExternalSecret CRD from external-secrets.io
            items = api.list_cluster_custom_object(
                group="external-secrets.io",
                version="v1beta1",
                plural="externalsecrets",
            ).get("items", [])

            for es in items:
                _extract_refs(es, ctx, ref_map)

        except Exception:
            logger.exception("Failed to scan context %s", ctx)

    return ref_map


def enrich_records(
    records: list[SecretAuditRecord],
    ref_map: dict[str, list[SecretReference]],
) -> None:
    """Mutates records in-place with k8s reference data + orphan flag."""
    for record in records:
        refs = ref_map.get(record.name, [])
        record.k8s_references = refs
        record.is_orphaned = len(refs) == 0


def _extract_refs(
    es: dict, cluster: str, ref_map: dict[str, list[SecretReference]]
) -> None:
    """Parse an ExternalSecret manifest and populate ref_map."""
    metadata = es.get("metadata", {})
    spec = es.get("spec", {})
    ns = metadata.get("namespace", "default")
    name = metadata.get("name", "unknown")
    target = spec.get("target", {}).get("name", name)

    for data_entry in spec.get("data", []):
        remote_ref = data_entry.get("remoteRef", {})
        secret_key = remote_ref.get("key", "")
        if secret_key:
            ref_map.setdefault(secret_key, []).append(
                SecretReference(
                    namespace=ns,
                    external_secret_name=name,
                    cluster=cluster,
                    target_secret_name=target,
                )
            )


def _discover_eks_contexts() -> list[str]:
    """Auto-discover EKS contexts from kubeconfig."""
    _, contexts = k8s_config.list_kube_config_contexts()
    return [
        c["name"] for c in (contexts or [])
        if "eks" in c.get("context", {}).get("cluster", "").lower()
    ]
```

## Analyzer (`analyzer.py`)

```python
"""Policy engine — evaluates each SecretAuditRecord against audit rules."""

from __future__ import annotations

from novatools.secrets.models import (
    SecretAuditRecord,
    AuditSummary,
    AuditResult,
    Finding,
    FindingType,
)

DEFAULT_ROTATION_THRESHOLD_DAYS = 90
DEFAULT_ACCESS_THRESHOLD_DAYS = 180


def analyze(
    records: list[SecretAuditRecord],
    rotation_days: int = DEFAULT_ROTATION_THRESHOLD_DAYS,
    access_days: int = DEFAULT_ACCESS_THRESHOLD_DAYS,
) -> AuditResult:
    summary = AuditSummary(
        total_secrets=len(records),
        regions_scanned=list({r.region for r in records}),
    )

    for record in records:
        # Rule 1: Rotation overdue
        if record.days_since_rotation is not None and record.days_since_rotation > rotation_days:
            record.findings.append(Finding(
                finding_type=FindingType.ROTATION_OVERDUE,
                message=f"Last rotated {record.days_since_rotation} days ago (threshold: {rotation_days}d)",
                severity="critical" if record.days_since_rotation > rotation_days * 2 else "warning",
            ))
            summary.rotation_overdue += 1
        elif record.last_rotated_date is None and record.rotation_enabled:
            record.findings.append(Finding(
                finding_type=FindingType.ROTATION_OVERDUE,
                message="Rotation enabled but secret has never been rotated",
                severity="critical",
            ))
            summary.rotation_overdue += 1

        # Rule 2: Stale access
        if record.days_since_access is not None and record.days_since_access > access_days:
            record.findings.append(Finding(
                finding_type=FindingType.STALE_ACCESS,
                message=f"Last accessed {record.days_since_access} days ago (threshold: {access_days}d)",
                severity="warning",
            ))
            summary.stale_access += 1
        elif record.last_accessed_date is None:
            record.findings.append(Finding(
                finding_type=FindingType.NEVER_ACCESSED,
                message="Secret has never been accessed",
                severity="warning",
            ))
            summary.stale_access += 1

        # Rule 3: Orphaned
        if record.is_orphaned:
            record.findings.append(Finding(
                finding_type=FindingType.ORPHANED,
                message="Not referenced by any ExternalSecret CR in any cluster",
                severity="warning",
            ))
            summary.orphaned += 1

    summary.total_findings = sum(len(r.findings) for r in records)
    summary.by_region = _count_by_region(records)

    return AuditResult(records=records, summary=summary)


def _count_by_region(records: list[SecretAuditRecord]) -> dict[str, int]:
    counts: dict[str, int] = {}
    for r in records:
        if r.has_findings:
            counts[r.region] = counts.get(r.region, 0) + 1
    return counts
```

## Click Command (`secrets_cmd.py`)

```python
"""CLI entry point for `novatools secrets audit`."""

from __future__ import annotations

import asyncio
import json
import logging
import sys
import time
from typing import Literal

import click
from rich.console import Console
from rich.table import Table

from novatools.secrets.scanner import scan_region
from novatools.secrets.k8s_resolver import load_external_secret_refs, enrich_records
from novatools.secrets.analyzer import analyze
from novatools.secrets.notifier import send_slack_summary
from novatools.common.concurrency import run_across_regions
from novatools.secrets.models import AuditResult

logger = logging.getLogger(__name__)
console = Console(stderr=True)

DEFAULT_REGIONS = ["us-east-1", "us-west-2", "eu-west-1"]


@click.group()
def secrets():
    """Manage and audit secrets."""
    pass


@secrets.command()
@click.option(
    "--regions", "-r",
    multiple=True,
    default=DEFAULT_REGIONS,
    help="AWS regions to scan",
)
@click.option(
    "--rotation-days",
    type=int,
    default=90,
    show_default=True,
    help="Threshold for rotation overdue (days)",
)
@click.option(
    "--access-days",
    type=int,
    default=180,
    show_default=True,
    help="Threshold for stale access (days)",
)
@click.option(
    "--output-format", "-o",
    type=click.Choice(["table", "json"]),
    default="table",
    show_default=True,
)
@click.option(
    "--slack/--no-slack",
    default=True,
    help="Send Slack summary",
)
@click.option(
    "--kubeconfig-contexts",
    multiple=True,
    default=None,
    help="Explicit kubeconfig contexts (default: auto-discover EKS)",
)
@click.option(
    "--findings-only",
    is_flag=True,
    default=False,
    help="Only show secrets with findings",
)
def audit(
    regions: tuple[str, ...],
    rotation_days: int,
    access_days: int,
    output_format: Literal["table", "json"],
    slack: bool,
    kubeconfig_contexts: tuple[str, ...] | None,
    findings_only: bool,
) -> None:
    """Audit secrets across AWS regions and EKS clusters."""
    start = time.monotonic()

    # ── Step 1: Scan AWS Secrets Manager across regions (concurrent) ─────
    console.print(f"[bold]Scanning {len(regions)} regions...[/bold]")
    region_results = asyncio.run(
        run_across_regions(list(regions), scan_region)
    )

    all_records = []
    errors = []
    for region, result in region_results.items():
        if isinstance(result, Exception):
            errors.append(f"{region}: {result}")
            console.print(f"  [red]✗[/red] {region}: {result}")
        else:
            all_records.extend(result)
            console.print(f"  [green]✓[/green] {region}: {len(result)} secrets")

    if not all_records and errors:
        console.print("[bold red]All regions failed. Aborting.[/bold red]")
        sys.exit(1)

    # ── Step 2: Cross-reference with K8s ExternalSecret CRs ─────────────
    console.print("[bold]Scanning EKS clusters for ExternalSecret references...[/bold]")
    ctx_list = list(kubeconfig_contexts) if kubeconfig_contexts else None
    ref_map = load_external_secret_refs(ctx_list)
    enrich_records(all_records, ref_map)
    console.print(f"  Found {sum(len(v) for v in ref_map.values())} references across {len(ref_map)} secrets")

    # ── Step 3: Analyze ──────────────────────────────────────────────────
    result = analyze(all_records, rotation_days, access_days)
    result.summary.scan_duration_seconds = round(time.monotonic() - start, 2)
    result.summary.errors = errors

    # ── Step 4: Output ───────────────────────────────────────────────────
    if output_format == "json":
        _output_json(result, findings_only)
    else:
        _output_table(result, findings_only)

    # ── Step 5: Slack ────────────────────────────────────────────────────
    if slack:
        try:
            send_slack_summary(result.summary)
            console.print("[green]Slack notification sent.[/green]")
        except Exception as exc:
            console.print(f"[yellow]Slack notification failed: {exc}[/yellow]")

    # Exit code reflects findings
    sys.exit(1 if result.summary.total_findings > 0 else 0)


def _output_table(result: AuditResult, findings_only: bool) -> None:
    table = Table(title="Secrets Audit Results", show_lines=True)
    table.add_column("Secret", style="bold")
    table.add_column("Region")
    table.add_column("Rotated (days)", justify="right")
    table.add_column("Accessed (days)", justify="right")
    table.add_column("K8s Refs", justify="right")
    table.add_column("Findings", style="red")

    for r in result.records:
        if findings_only and not r.has_findings:
            continue
        findings_str = "\n".join(f.message for f in r.findings) if r.findings else "—"
        style = "red" if r.has_findings else None
        table.add_row(
            r.name,
            r.region,
            str(r.days_since_rotation) if r.days_since_rotation is not None else "never",
            str(r.days_since_access) if r.days_since_access is not None else "never",
            str(len(r.k8s_references)),
            findings_str,
            style=style,
        )

    console.print(table)

    # Summary footer
    s = result.summary
    console.print(
        f"\n[bold]Summary:[/bold] {s.total_secrets} secrets, "
        f"[red]{s.total_findings} findings[/red] "
        f"(rotation: {s.rotation_overdue}, stale: {s.stale_access}, orphaned: {s.orphaned}) "
        f"in {s.scan_duration_seconds}s"
    )


def _output_json(result: AuditResult, findings_only: bool) -> None:
    data = result.model_dump(mode="json")
    if findings_only:
        data["records"] = [r for r in data["records"] if r.get("findings")]
    click.echo(json.dumps(data, indent=2, default=str))
```

## Test Strategy

```python
# tests/conftest.py
import pytest
from unittest.mock import MagicMock, patch
from moto import mock_secretsmanager
from datetime import datetime, timezone, timedelta
import boto3


@pytest.fixture
def aws_secrets():
    """Provision moto Secrets Manager with known test data."""
    with mock_secretsmanager():
        regions = ["us-east-1", "us-west-2"]
        now = datetime.now(timezone.utc)
        for region in regions:
            client = boto3.client("secretsmanager", region_name=region)
            # Secret rotated recently
            client.create_secret(Name="db/prod/password", SecretString="s3cret")
            # Old secret (will trigger rotation_overdue)
            client.create_secret(Name="api/legacy-key", SecretString="old")
        yield regions


@pytest.fixture
def mock_k8s_refs():
    """Fake ExternalSecret reference map."""
    return {
        "db/prod/password": [
            MagicMock(namespace="production", external_secret_name="db-creds", cluster="prod-1")
        ],
        # api/legacy-key is intentionally absent → orphaned
    }


# tests/secrets/test_scanner.py
from moto import mock_secretsmanager
from novatools.secrets.scanner import scan_region


@mock_secretsmanager
def test_scan_region_paginates(aws_secrets):
    records = scan_region("us-east-1")
    assert len(records) == 2
    assert all(r.region == "us-east-1" for r in records)


@mock_secretsmanager
def test_scan_region_captures_rotation_metadata(aws_secrets):
    records = scan_region("us-east-1")
    db_secret = next(r for r in records if r.name == "db/prod/password")
    assert db_secret.arn.startswith("arn:aws:secretsmanager")


# tests/secrets/test_analyzer.py
from novatools.secrets.analyzer import analyze
from novatools.secrets.models import SecretAuditRecord, FindingType
from datetime import datetime, timezone, timedelta


def _make_record(**overrides):
    now = datetime.now(timezone.utc)
    defaults = dict(
        arn="arn:aws:secretsmanager:us-east-1:123:secret:test",
        name="test-secret", region="us-east-1",
        created_date=now - timedelta(days=365),
        last_rotated_date=now - timedelta(days=100),
        days_since_rotation=100,
        last_accessed_date=now - timedelta(days=10),
        days_since_access=10,
        is_orphaned=False,
    )
    defaults.update(overrides)
    return SecretAuditRecord(**defaults)


def test_rotation_overdue_flagged():
    record = _make_record(days_since_rotation=91)
    result = analyze([record], rotation_days=90)
    assert result.summary.rotation_overdue == 1
    assert result.records[0].findings[0].finding_type == FindingType.ROTATION_OVERDUE


def test_orphaned_secret_flagged():
    record = _make_record(is_orphaned=True, days_since_rotation=10)
    result = analyze([record])
    assert result.summary.orphaned == 1


def test_clean_secret_no_findings():
    record = _make_record(
        days_since_rotation=10, days_since_access=5, is_orphaned=False
    )
    result = analyze([record])
    assert result.summary.total_findings == 0


# tests/secrets/test_concurrency.py
import asyncio
import pytest
from novatools.common.concurrency import run_across_regions


def test_partial_failure_captured():
    def scan(region):
        if region == "us-west-2":
            raise ConnectionError("timeout")
        return [f"secret-in-{region}"]

    results = asyncio.run(run_across_regions(
        ["us-east-1", "us-west-2"], scan, timeout=5.0
    ))

    assert isinstance(results["us-east-1"], list)
    assert isinstance(results["us-west-2"], ConnectionError)


# tests/secrets/test_notifier.py
import responses
from novatools.secrets.notifier import send_slack_summary
from novatools.secrets.models import AuditSummary


@responses.activate
def test_slack_notification_sent():
    responses.add(responses.POST, "https://hooks.slack.com/test", status=200)
    summary = AuditSummary(
        total_secrets=50, total_findings=3,
        rotation_overdue=1, stale_access=1, orphaned=1,
        regions_scanned=["us-east-1"]
    )
    send_slack_summary(summary, webhook_url="https://hooks.slack.com/test")
    assert len(responses.calls) == 1
    assert "3" in responses.calls[0].request.body  # total findings in payload
```

### Architecture Diagram (Execution Flow)

```
CLI (Click)
    │
    ├── asyncio.run(run_across_regions)
    │       │
    │       ├── Thread-1 → scan_region("us-east-1") → [SecretAuditRecord, ...]
    │       ├── Thread-2 → scan_region("us-west-2") → [SecretAuditRecord, ...]
    │       └── Thread-3 → scan_region("eu-west-1") → [SecretAuditRecord, ...]
    │       │
    │       └── {region: records | Exception}
    │
    ├── load_external_secret_refs()  ←── K8s API (per cluster context)
    │       └── {aws_secret_name: [SecretReference, ...]}
    │
    ├── enrich_records()  ←── Joins AWS data + K8s references
    │
    ├── analyze()  ←── Policy evaluation → AuditResult
    │
    ├── _output_table() / _output_json()  ←── stdout
    │
    └── send_slack_summary()  ←── Slack webhook POST
```

---

# Q2 — Debugging: The Silent 1,000-Instance Ceiling

## Bug 1: **Missing Pagination** — THE ROOT CAUSE OF THE OUTAGE

```python
response = ec2.describe_instances(
    Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
)
```

**Problem:** `describe_instances` returns a **maximum of 1,000 instances per page** by default. The response includes a `NextToken` field when there are more results. This code reads **only the first page** and ignores the rest.

**Why it "worked" for 3 months:** With ~1,200 instances, the first page returned ~1,000. When 200 more were added (total 1,400), still only ~1,000 were returned. The remaining ~400 are silently discarded.

**Production consequence:** The inventory report is missing ~400 instances. Capacity planning, cost reporting, and compliance auditing are all wrong. If security relies on this report for patching coverage, hundreds of instances are invisible.

**Fix:**
```python
def get_all_instances(region="us-east-1"):
    ec2 = boto3.client("ec2", region_name=region)
    paginator = ec2.get_paginator("describe_instances")
    
    instances = []
    for page in paginator.paginate(
        Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
    ):
        for reservation in page["Reservations"]:
            for instance in reservation["Instances"]:
                # ... process instance
```

---

## Bug 2: **Unguarded `Tags` access — KeyError crash**

```python
"name": [t["Value"] for t in instance["Tags"] if t["Key"] == "Name"][0],
```

**Problem (a):** `instance["Tags"]` raises `KeyError` if the instance has **no tags at all**. Not every EC2 instance has tags — instances launched via certain auto-scaling configs, spot fleets, or third-party tools may have zero tags.

**Problem (b):** Even if tags exist, the `Name` tag might not be present. The list comprehension produces `[]`, and `[0]` raises `IndexError`.

**Production consequence:** A single untagged instance crashes the entire script. Every instance after it in the iteration is lost from the report. With 1,400 instances, the probability of hitting one without a Name tag is near-certain.

**Fix:**
```python
tags = {t["Key"]: t["Value"] for t in instance.get("Tags", [])}
"name": tags.get("Name", "unnamed"),
```

---

## Bug 3: **Timezone-naive vs timezone-aware datetime comparison**

```python
"age_days": (datetime.now() - instance["LaunchTime"]).days,
```

**Problem:** `instance["LaunchTime"]` is a timezone-aware `datetime` (UTC, from boto3). `datetime.now()` returns a **timezone-naive** datetime. Subtracting them raises:

```
TypeError: can't subtract offset-naive and offset-aware datetimes
```

**Production consequence:** Same as Bug 2 — every instance hits this exception, crashing the entire function. The fact that this "worked for 3 months" suggests either: (a) this field was added recently, (b) the exception was silently swallowed somewhere upstream, or (c) the Python version/environment handled it differently.

**Fix:**
```python
from datetime import datetime, timezone

"age_days": (datetime.now(timezone.utc) - instance["LaunchTime"]).days,
```

---

## Bug 4: **`LaunchTime` serialization — `str()` loses timezone info**

```python
"launched": str(instance["LaunchTime"]),
```

**Problem:** `str()` on a datetime produces a format like `2024-01-15 08:30:00+00:00`. This is not ISO 8601, it's not consistently parseable, and it **loses timezone information** when later read back (many parsers ignore the `+00:00` suffix from this format).

**Production consequence:** Downstream consumers of `inventory.json` that parse this field may compute incorrect ages, or fail to parse it entirely.

**Fix:**
```python
"launched": instance["LaunchTime"].isoformat(),
```

---

## Bug 5: **No error handling on any region — one failure kills all data**

```python
for region in regions:
    all_instances.extend(get_all_instances(region))
```

**Problem:** If `get_all_instances("us-west-2")` throws (API error, throttling, temporary network issue), the **entire script crashes** and `save_report` is never called. Instances from `us-east-1` (which succeeded) are lost.

**Production consequence:** A transient issue in one region means zero inventory data across all regions.

**Fix:**
```python
for region in regions:
    try:
        all_instances.extend(get_all_instances(region))
    except Exception as e:
        logging.error("Failed to scan %s: %s", region, e)
```

---

## Bug 6: **No retry logic / throttling handling**

**Problem:** boto3's default retry mode handles some throttling, but `describe_instances` across 1,400 instances (paginated) can easily hit EC2 API rate limits. There's no explicit retry configuration and no backoff.

**Fix:**
```python
from botocore.config import Config
ec2 = boto3.client("ec2", region_name=region, config=Config(
    retries={"max_attempts": 5, "mode": "adaptive"}
))
```

---

## Bug 7: **`/tmp/inventory.json` — non-atomic write, hardcoded path**

```python
with open("/tmp/inventory.json", "w") as f:
    json.dump(instances, f, indent=2)
```

**Problem:** If the script crashes mid-write, the file is truncated/corrupted. Downstream consumers reading this file may get partial JSON. Also, `/tmp` is world-writable — any user on the system can replace this file (symlink attack vector).

**Fix:** Write to a temp file, then atomically rename:
```python
import tempfile, os

with tempfile.NamedTemporaryFile(mode="w", dir="/tmp", suffix=".json", delete=False) as f:
    json.dump(instances, f, indent=2)
    tmp_path = f.name
os.replace(tmp_path, "/tmp/inventory.json")
```

---

## Bug 8: **`json.dump` will fail on datetime objects**

```python
json.dump(instances, f, indent=2)
```

**Problem:** If `str(instance["LaunchTime"])` somehow isn't called (or a code path is added later that passes raw datetime objects), `json.dump` raises `TypeError: Object of type datetime is not JSON serializable`. Even with the current code, this is fragile — one missed `str()` wrapper breaks serialization.

**Fix:** Use a default serializer:
```python
json.dump(instances, f, indent=2, default=str)
```

---

### Corrected Code

```python
#!/usr/bin/env python3
"""EC2 inventory report — all running instances across regions."""

import json
import logging
import os
import tempfile
from datetime import datetime, timezone

import boto3
from botocore.config import Config

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)

BOTO_CONFIG = Config(retries={"max_attempts": 5, "mode": "adaptive"})


def get_all_instances(region: str) -> list[dict]:
    ec2 = boto3.client("ec2", region_name=region, config=BOTO_CONFIG)
    paginator = ec2.get_paginator("describe_instances")

    now = datetime.now(timezone.utc)
    instances = []

    for page in paginator.paginate(
        Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
    ):
        for reservation in page["Reservations"]:
            for instance in reservation["Instances"]:
                tags = {t["Key"]: t["Value"] for t in instance.get("Tags", [])}
                launch_time = instance["LaunchTime"]

                instances.append({
                    "id": instance["InstanceId"],
                    "type": instance["InstanceType"],
                    "launched": launch_time.isoformat(),
                    "name": tags.get("Name", "unnamed"),
                    "region": region,
                    "age_days": (now - launch_time).days,
                })

    logger.info("Region %s: %d running instances", region, len(instances))
    return instances


def save_report(instances: list[dict], path: str = "/tmp/inventory.json") -> None:
    dir_name = os.path.dirname(path)
    with tempfile.NamedTemporaryFile(
        mode="w", dir=dir_name, suffix=".json", delete=False
    ) as f:
        json.dump(instances, f, indent=2, default=str)
        tmp_path = f.name
    os.replace(tmp_path, path)
    logger.info("Saved %d instances to %s", len(instances), path)


if __name__ == "__main__":
    regions = ["us-east-1", "us-west-2", "eu-west-1"]
    all_instances: list[dict] = []

    for region in regions:
        try:
            all_instances.extend(get_all_instances(region))
        except Exception:
            logger.exception("Failed to scan region %s", region)

    save_report(all_instances)
```

---

# Q3 — Production Code: Prometheus Error Rate Checker

## Implementation

```python
"""
prometheus_error_rates.py — Query per-namespace HTTP error rates from Prometheus.

Production-grade: typed, retried, partial-failure-tolerant, fully tested.
"""

from __future__ import annotations

import logging
from dataclasses import dataclass, field
from enum import Enum
from typing import Sequence

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

logger = logging.getLogger(__name__)


# ── Data Models ──────────────────────────────────────────────────────────────


class QueryStatus(str, Enum):
    SUCCESS = "success"
    ERROR = "error"
    NO_DATA = "no_data"


@dataclass(frozen=True, slots=True)
class NamespaceErrorRate:
    namespace: str
    error_rate: float | None  # 0.0–1.0, None if query failed
    total_rps: float | None  # requests/sec, None if unavailable
    error_rps: float | None  # error requests/sec
    status: QueryStatus
    error_message: str | None = None


@dataclass
class ErrorRateReport:
    results: list[NamespaceErrorRate] = field(default_factory=list)
    failed_namespaces: list[str] = field(default_factory=list)
    prometheus_url: str = ""

    @property
    def successful(self) -> list[NamespaceErrorRate]:
        return [r for r in self.results if r.status == QueryStatus.SUCCESS]

    @property
    def sorted_by_error_rate(self) -> list[NamespaceErrorRate]:
        """Successful results sorted by error rate (highest first), then failures."""
        successes = sorted(
            [r for r in self.results if r.error_rate is not None],
            key=lambda r: r.error_rate,  # type: ignore[arg-type]
            reverse=True,
        )
        failures = [r for r in self.results if r.error_rate is None]
        return successes + failures


# ── HTTP Session Factory ─────────────────────────────────────────────────────


def _build_session(
    max_retries: int = 3,
    backoff_factor: float = 0.5,
    timeout: float = 10.0,
) -> tuple[requests.Session, float]:
    """Build a requests.Session with retry + backoff on transient errors."""
    session = requests.Session()
    retry_strategy = Retry(
        total=max_retries,
        backoff_factor=backoff_factor,
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["GET", "POST"],
        raise_on_status=False,
    )
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    return session, timeout


# ── Core Query Logic ─────────────────────────────────────────────────────────

_ERROR_RATE_QUERY = (
    'sum(rate(http_requests_total{{namespace="{ns}",code=~"5.."}}[5m]))'
    " / "
    'sum(rate(http_requests_total{{namespace="{ns}"}}[5m]))'
)

_ERROR_RPS_QUERY = (
    'sum(rate(http_requests_total{{namespace="{ns}",code=~"5.."}}[5m]))'
)

_TOTAL_RPS_QUERY = (
    'sum(rate(http_requests_total{{namespace="{ns}"}}[5m]))'
)


def _query_prometheus(
    session: requests.Session,
    prometheus_url: str,
    query: str,
    timeout: float,
) -> float | None:
    """
    Execute an instant query against Prometheus.
    Returns the scalar float result, or None if empty/error.
    """
    url = f"{prometheus_url.rstrip('/')}/api/v1/query"

    response = session.get(url, params={"query": query}, timeout=timeout)
    response.raise_for_status()

    payload = response.json()

    if payload.get("status") != "success":
        raise ValueError(f"Prometheus query failed: {payload.get('error', 'unknown')}")

    results = payload.get("data", {}).get("result", [])

    if not results:
        return None

    # Instant query returns vector; we expect a single aggregated value
    value_str = results[0]["value"][1]

    # Prometheus returns "NaN" for 0/0, "+Inf" for n/0
    if value_str in ("NaN", "+Inf", "-Inf"):
        return None

    return float(value_str)


def _query_namespace(
    session: requests.Session,
    prometheus_url: str,
    namespace: str,
    timeout: float,
) -> NamespaceErrorRate:
    """Query all metrics for a single namespace. Never raises."""
    try:
        # Query total RPS first to detect "no traffic" case
        total_rps = _query_prometheus(
            session, prometheus_url,
            _TOTAL_RPS_QUERY.format(ns=namespace), timeout,
        )

        if total_rps is None or total_rps == 0.0:
            return NamespaceErrorRate(
                namespace=namespace,
                error_rate=None,
                total_rps=total_rps,
                error_rps=None,
                status=QueryStatus.NO_DATA,
                error_message="No traffic or no metrics for namespace",
            )

        error_rps = _query_prometheus(
            session, prometheus_url,
            _ERROR_RPS_QUERY.format(ns=namespace), timeout,
        )

        # Compute rate safely (total_rps is non-zero here)
        error_rps_val = error_rps if error_rps is not None else 0.0
        error_rate = error_rps_val / total_rps

        # Clamp to [0.0, 1.0] — floating point edge cases
        error_rate = max(0.0, min(1.0, error_rate))

        return NamespaceErrorRate(
            namespace=namespace,
            error_rate=error_rate,
            total_rps=round(total_rps, 4),
            error_rps=round(error_rps_val, 4),
            status=QueryStatus.SUCCESS,
        )

    except requests.exceptions.ConnectionError as exc:
        logger.error("Prometheus unreachable for namespace %s: %s", namespace, exc)
        return NamespaceErrorRate(
            namespace=namespace, error_rate=None, total_rps=None, error_rps=None,
            status=QueryStatus.ERROR,
            error_message=f"Connection error: {exc}",
        )
    except requests.exceptions.Timeout as exc:
        logger.error("Prometheus timeout for namespace %s: %s", namespace, exc)
        return NamespaceErrorRate(
            namespace=namespace, error_rate=None, total_rps=None, error_rps=None,
            status=QueryStatus.ERROR,
            error_message=f"Timeout: {exc}",
        )
    except Exception as exc:
        logger.exception("Unexpected error querying namespace %s", namespace)
        return NamespaceErrorRate(
            namespace=namespace, error_rate=None, total_rps=None, error_rps=None,
            status=QueryStatus.ERROR,
            error_message=f"Unexpected: {type(exc).__name__}: {exc}",
        )


# ── Public API ───────────────────────────────────────────────────────────────


def get_namespace_error_rates(
    namespaces: Sequence[str],
    prometheus_url: str,
    max_retries: int = 3,
    backoff_factor: float = 0.5,
    timeout: float = 10.0,
) -> ErrorRateReport:
    """
    Query Prometheus for per-namespace HTTP 5xx error rates.

    Args:
        namespaces: List of Kubernetes namespace names to check.
        prometheus_url: Base URL of Prometheus (e.g., "http://prometheus:9090").
        max_retries: Number of retries on transient HTTP errors.
        backoff_factor: Exponential backoff factor between retries.
        timeout: Per-request timeout in seconds.

    Returns:
        ErrorRateReport with results sorted by error rate (highest first).
        Partial failures are captured per-namespace; the function never raises
        for individual namespace failures.

    Raises:
        ValueError: If namespaces is empty or prometheus_url is empty.
    """
    if not namespaces:
        raise ValueError("namespaces must not be empty")
    if not prometheus_url:
        raise ValueError("prometheus_url must not be empty")

    session, req_timeout = _build_session(max_retries, backoff_factor, timeout)
    report = ErrorRateReport(prometheus_url=prometheus_url)

    for namespace in namespaces:
        logger.info("Querying error rate for namespace: %s", namespace)
        result = _query_namespace(session, prometheus_url, namespace, req_timeout)
        report.results.append(result)

        if result.status == QueryStatus.ERROR:
            report.failed_namespaces.append(namespace)
            logger.warning(
                "Namespace %s failed: %s", namespace, result.error_message
            )

    # Sort by error rate descending (primary output contract)
    report.results = report.sorted_by_error_rate

    logger.info(
        "Error rate check complete: %d/%d namespaces succeeded",
        len(report.successful),
        len(namespaces),
    )

    return report
```

## Complete Test Suite

```python
"""
test_prometheus_error_rates.py — Full test coverage using responses library.
"""

import pytest
import json
import responses
from responses import matchers

from prometheus_error_rates import (
    get_namespace_error_rates,
    ErrorRateReport,
    NamespaceErrorRate,
    QueryStatus,
)

PROM_URL = "http://prometheus.monitoring:9090"
QUERY_ENDPOINT = f"{PROM_URL}/api/v1/query"


# ── Helpers ──────────────────────────────────────────────────────────────────

def _prom_response(value: str | float | None, status: str = "success"):
    """Build a Prometheus API response body."""
    if value is None:
        return {"status": status, "data": {"resultType": "vector", "result": []}}
    return {
        "status": status,
        "data": {
            "resultType": "vector",
            "result": [{"metric": {}, "value": [1700000000, str(value)]}],
        },
    }


def _register_namespace_responses(
    namespace: str,
    total_rps: float | str | None,
    error_rps: float | str | None,
):
    """Register mocked Prometheus responses for total_rps then error_rps queries."""
    # First call: total RPS query
    responses.add(
        responses.GET,
        QUERY_ENDPOINT,
        json=_prom_response(total_rps),
        status=200,
    )
    # Second call: error RPS query (only if total_rps is non-None and non-zero)
    if total_rps is not None and total_rps != 0 and total_rps != "NaN":
        responses.add(
            responses.GET,
            QUERY_ENDPOINT,
            json=_prom_response(error_rps),
            status=200,
        )


# ── Tests: Happy Path ───────────────────────────────────────────────────────


class TestHappyPath:

    @responses.activate
    def test_single_namespace_returns_correct_error_rate(self):
        _register_namespace_responses("payments", total_rps=100.0, error_rps=5.0)

        report = get_namespace_error_rates(["payments"], PROM_URL)

        assert len(report.results) == 1
        result = report.results[0]
        assert result.namespace == "payments"
        assert result.status == QueryStatus.SUCCESS
        assert result.error_rate == pytest.approx(0.05, abs=1e-6)
        assert result.total_rps == pytest.approx(100.0, abs=0.01)
        assert result.error_rps == pytest.approx(5.0, abs=0.01)

    @responses.activate
    def test_multiple_namespaces_sorted_by_error_rate_descending(self):
        # "orders" has higher error rate (10%) than "payments" (2%)
        _register_namespace_responses("orders", total_rps=200.0, error_rps=20.0)
        _register_namespace_responses("payments", total_rps=500.0, error_rps=10.0)

        report = get_namespace_error_rates(["orders", "payments"], PROM_URL)

        assert len(report.results) == 2
        assert report.results[0].namespace == "orders"
        assert report.results[0].error_rate == pytest.approx(0.10)
        assert report.results[1].namespace == "payments"
        assert report.results[1].error_rate == pytest.approx(0.02)

    @responses.activate
    def test_zero_errors_returns_zero_rate(self):
        _register_namespace_responses("clean-ns", total_rps=1000.0, error_rps=0.0)

        report = get_namespace_error_rates(["clean-ns"], PROM_URL)

        assert report.results[0].error_rate == pytest.approx(0.0)
        assert report.results[0].status == QueryStatus.SUCCESS


# ── Tests: Edge Cases ────────────────────────────────────────────────────────


class TestEdgeCases:

    @responses.activate
    def test_no_traffic_namespace_returns_no_data(self):
        """Namespace exists but has zero traffic."""
        _register_namespace_responses("idle-ns", total_rps=None, error_rps=None)

        report = get_namespace_error_rates(["idle-ns"], PROM_URL)

        assert report.results[0].status == QueryStatus.NO_DATA
        assert report.results[0].error_rate is None

    @responses.activate
    def test_nan_from_prometheus_handled(self):
        """Prometheus returns NaN for 0/0 division."""
        _register_namespace_responses("nan-ns", total_rps="NaN", error_rps=None)

        report = get_namespace_error_rates(["nan-ns"], PROM_URL)

        assert report.results[0].status == QueryStatus.NO_DATA
        assert report.results[0].error_rate is None

    @responses.activate
    def test_zero_total_rps_avoids_division_by_zero(self):
        _register_namespace_responses("zero-ns", total_rps=0.0, error_rps=None)

        report = get_namespace_error_rates(["zero-ns"], PROM_URL)

        assert report.results[0].status == QueryStatus.NO_DATA
        assert report.results[0].error_rate is None

    def test_empty_namespaces_raises_value_error(self):
        with pytest.raises(ValueError, match="namespaces must not be empty"):
            get_namespace_error_rates([], PROM_URL)

    def test_empty_url_raises_value_error(self):
        with pytest.raises(ValueError, match="prometheus_url must not be empty"):
            get_namespace_error_rates(["ns"], "")


# ── Tests: Failure Modes ─────────────────────────────────────────────────────


class TestFailureModes:

    @responses.activate
    def test_prometheus_unreachable_returns_error_status(self):
        """Connection refused — no mocked response registered."""
        responses.add(
            responses.GET,
            QUERY_ENDPOINT,
            body=ConnectionError("Connection refused"),
        )

        report = get_namespace_error_rates(
            ["payments"], PROM_URL, max_retries=0
        )

        assert report.results[0].status == QueryStatus.ERROR
        assert "Connection" in (report.results[0].error_message or "")
        assert "payments" in report.failed_namespaces

    @responses.activate
    def test_prometheus_500_retries_then_fails(self):
        """Server errors exhaust retries."""
        for _ in range(4):  # 1 initial + 3 retries
            responses.add(responses.GET, QUERY_ENDPOINT, status=500)

        report = get_namespace_error_rates(
            ["payments"], PROM_URL, max_retries=3
        )

        assert report.results[0].status == QueryStatus.ERROR

    @responses.activate
    def test_partial_failure_returns_mixed_results(self):
        """First namespace succeeds, second fails."""
        # "orders" succeeds
        _register_namespace_responses("orders", total_rps=100.0, error_rps=3.0)

        # "payments" fails
        responses.add(
            responses.GET,
            QUERY_ENDPOINT,
            body=ConnectionError("timeout"),
        )

        report = get_namespace_error_rates(
            ["orders", "payments"], PROM_URL, max_retries=0
        )

        assert len(report.results) == 2
        assert len(report.successful) == 1
        assert report.successful[0].namespace == "orders"
        assert "payments" in report.failed_namespaces

        # Sorted: successes first, then failures
        assert report.results[0].namespace == "orders"
        assert report.results[1].namespace == "payments"

    @responses.activate
    def test_prometheus_returns_error_status_in_body(self):
        """Prometheus returns 200 but status=error in JSON body."""
        responses.add(
            responses.GET,
            QUERY_ENDPOINT,
            json={"status": "error", "error": "query parse error"},
            status=200,
        )

        report = get_namespace_error_rates(
            ["bad-query-ns"], PROM_URL, max_retries=0
        )

        assert report.results[0].status == QueryStatus.ERROR


# ── Tests: Report Structure ──────────────────────────────────────────────────


class TestReportStructure:

    @responses.activate
    def test_report_metadata(self):
        _register_namespace_responses("ns1", total_rps=50.0, error_rps=1.0)

        report = get_namespace_error_rates(["ns1"], PROM_URL)

        assert report.prometheus_url == PROM_URL
        assert report.failed_namespaces == []

    @responses.activate
    def test_sorted_by_error_rate_property(self):
        """Verify the sort is descending by error rate, failures last."""
        # Three namespaces with known error rates
        _register_namespace_responses("low", total_rps=1000.0, error_rps=1.0)    # 0.1%
        _register_namespace_responses("high", total_rps=1000.0, error_rps=100.0)  # 10%
        _register_namespace_responses("mid", total_rps=1000.0, error_rps=50.0)    # 5%

        report = get_namespace_error_rates(["low", "high", "mid"], PROM_URL)

        rates = [r.error_rate for r in report.results if r.error_rate is not None]
        assert rates == sorted(rates, reverse=True)
        assert report.results[0].namespace == "high"
        assert report.results[1].namespace == "mid"
        assert report.results[2].namespace == "low"
```

Run with:
```bash
pytest test_prometheus_error_rates.py -v --tb=short
```

---

# Q4 — Concurrency Analysis: 500 Health Checks in <30s

## Approach 1: Sequential `requests.get()`

### Expected Wall-Clock Time

```
Best case (all healthy):  500 × ~0.05s avg = ~25 seconds
Worst case (timeouts):    500 × 5s timeout  = 2,500 seconds (41+ minutes)
Typical (mixed):          500 × ~0.2s avg   = ~100 seconds
```

**Verdict:** Fails the 30-second requirement in all realistic scenarios. A single 5-second timeout blocks everything behind it.

### Memory/CPU
- ~50 MB RSS. One socket at a time. Minimal CPU — mostly sleeping in `recv()`.
- **Most resource-efficient** of all three approaches.

### Failure Modes
| Failure Mode | Impact |
|---|---|
| **One slow endpoint** | Blocks all subsequent checks. 1 timeout = 5s delay cascading. |
| **DNS resolution hang** | Can exceed the 5s timeout (socket-level timeout doesn't cover DNS in all cases). |
| **Connection pool exhaustion** | N/A — one connection at a time. |
| **Process killed (OOM/signal)** | Zero results. No partial output unless you flush incrementally. |

---

## Approach 2: `ThreadPoolExecutor` with `requests`

### Expected Wall-Clock Time

```python
# With 50 workers (sweet spot for I/O-bound HTTP):
# Batches: ceil(500 / 50) = 10 batches
# Per batch: ~5 seconds worst case (timeout-dominated)
# Total: 10 × 5s = ~50s worst case

# With 100 workers:
# Batches: ceil(500 / 100) = 5 batches
# Total: 5 × 5s = ~25s worst case
# Typical: 5 × 0.2s = ~1 second best case

# With 200 workers:
# Batches: ceil(500 / 200) = 3 batches
# Total: 3 × 5s = 15s worst case ✅
```

**Need ~100-200 threads to guarantee <30s** even with timeout-heavy scenarios.

### Memory/CPU
```
Per thread stack: ~8 MB (default Linux), reducible to ~512 KB with threading.stack_size()
200 threads × 512 KB = ~100 MB stack memory
+ requests session overhead per thread
+ GIL contention (minimal for I/O-bound; GIL is released during socket operations)

Total: ~200-400 MB practical memory usage
CPU: Near zero — threads sleep in I/O syscalls
```

### Failure Modes
| Failure Mode | Impact |
|---|---|
| **Thread starvation** | If pool is too small, slow endpoints block workers, starving fast endpoints. Workers return once the 5s timeout fires, but throughput degrades. |
| **File descriptor exhaustion** | 200 concurrent connections = 200 FDs. Default `ulimit -n` is 1024 — safe. But in a container with low limits, `OSError: [Errno 24] Too many open files`. |
| **DNS thundering herd** | 200 threads all hitting DNS simultaneously on startup. System resolver may queue/drop. Mitigate with `cachetools` or a local DNS cache. |
| **GIL + CPU-bound processing** | If health check responses are large and need heavy JSON parsing, GIL serializes CPU work. Not a real concern for simple `/health` 200 OK responses. |
| **Uncaught exception in thread** | `ThreadPoolExecutor` captures exceptions in `Future` objects — safe if you call `.result()`. But if you use `map()` and don't iterate all results, exceptions are silently swallowed. |
| **Graceful shutdown** | `Ctrl+C` during execution: Python's signal handling in threads is messy. `KeyboardInterrupt` only fires in main thread; worker threads continue until completion. |

### Implementation Sketch
```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def check_health(endpoint: str) -> dict:
    try:
        r = requests.get(f"http://{endpoint}/health", timeout=5)
        return {"endpoint": endpoint, "status": r.status_code, "healthy": r.ok}
    except requests.RequestException as e:
        return {"endpoint": endpoint, "status": None, "healthy": False, "error": str(e)}

with ThreadPoolExecutor(max_workers=150) as pool:
    futures = {pool.submit(check_health, ep): ep for ep in endpoints}
    for future in as_completed(futures, timeout=30):
        result = future.result()
```

---

## Approach 3: `asyncio` + `aiohttp`

### Expected Wall-Clock Time

```python
# All 500 requests launched nearly simultaneously
# Bounded by: max(individual_response_times) ≈ 5 seconds (timeout)
# Plus overhead: event loop scheduling, connection setup ≈ ~0.5s
# Total: ~5.5 seconds worst case ✅✅✅

# Even with a semaphore limiting to 100 concurrent:
# 5 waves × 5s timeout = ~25s worst case ✅
```

**Fastest approach by a wide margin.** All 500 requests are in-flight concurrently within a single thread.

### Memory/CPU
```
No thread stacks — coroutines are ~2-10 KB each
500 coroutines × 10 KB = ~5 MB
aiohttp connection pool overhead: ~20 MB
Total: ~30-50 MB  ← 4-8× less than threading

CPU: Single thread, single core. Event loop does

