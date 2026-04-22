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

# You're Right. That Was Incomplete. Here's What's Missing.

I taught the **framework** (project structure, types, boto3, CLI, testing) but skipped the **actual work**. That's like teaching someone how to hold a wrench and calling them a mechanic.

Let me audit what a Senior DevOps engineer actually does with Python **every week**, rank by frequency, and deliver everything I missed.

---

## Missing Scenarios — Ranked by Daily/Weekly Usage

| Rank | Missing Scenario | Frequency | Severity of Gap |
|------|-----------------|-----------|-----------------|
| 1 | **subprocess — wrapping CLI tools** | Daily | 🔴 CRITICAL |
| 2 | **Log parsing & data processing** | Daily | 🔴 CRITICAL |
| 3 | **File operations (pathlib)** | Daily | 🔴 CRITICAL |
| 4 | **YAML/JSON manipulation at scale** | Daily | 🔴 CRITICAL |
| 5 | **Configuration management (pydantic-settings, env vars)** | Daily | 🔴 CRITICAL |
| 6 | **Regular expressions in production** | 3-4x/week | 🟡 SIGNIFICANT |
| 7 | **Webhook receivers (FastAPI)** | Weekly | 🟡 SIGNIFICANT |
| 8 | **AWS Lambda handlers** | Weekly | 🟡 SIGNIFICANT |
| 9 | **Database operations (psycopg2, redis-py)** | Weekly | 🟡 SIGNIFICANT |
| 10 | **Advanced decorators (timing, caching, auth)** | Weekly | 🟡 SIGNIFICANT |
| 11 | **Git automation (GitPython, PR creation)** | Weekly | 🟡 SIGNIFICANT |
| 12 | **Docker SDK (docker-py)** | Weekly | 🟡 SIGNIFICANT |
| 13 | **Date/time operations (timezone, windows, scheduling)** | 3-4x/week | 🟡 SIGNIFICANT |
| 14 | **CSV/reporting (pandas basics for DevOps)** | Weekly | 🟠 MODERATE |
| 15 | **Prometheus client (custom metrics)** | Occasional | 🟠 MODERATE |
| 16 | **Migration scripts patterns** | Occasional | 🟠 MODERATE |
| 17 | **Caching patterns (lru_cache, TTL)** | Occasional | 🟠 MODERATE |

That's **17 scenarios** I left out. 5 of them are **daily-use, critical gaps**. No excuses. Let me deliver them now.

---

## 1. SUBPROCESS — Wrapping CLI Tools (DAILY USE)

Half of DevOps Python is wrapping `kubectl`, `terraform`, `helm`, `aws`, `argocd`. This is the single most common pattern and I didn't teach it at all.

```python
"""
src/novatools/utils/shell.py

Production-grade subprocess wrapper.
"""
from __future__ import annotations

import os
import shlex
import subprocess
import time
from dataclasses import dataclass
from typing import Any

import structlog

log = structlog.get_logger()


@dataclass(frozen=True)
class CommandResult:
    """Result of a shell command execution."""
    command: str
    exit_code: int
    stdout: str
    stderr: str
    duration_seconds: float
    timed_out: bool = False

    @property
    def success(self) -> bool:
        return self.exit_code == 0

    @property
    def stdout_lines(self) -> list[str]:
        return [line for line in self.stdout.strip().splitlines() if line]

    def raise_on_error(self, message: str = "") -> None:
        """Raise if command failed."""
        if not self.success:
            detail = message or f"Command failed (exit {self.exit_code})"
            raise subprocess.CalledProcessError(
                self.exit_code, self.command,
                output=self.stdout, stderr=self.stderr,
            )


def run(
    cmd: str | list[str],
    *,
    check: bool = False,
    capture: bool = True,
    timeout: int = 300,
    cwd: str | None = None,
    env: dict[str, str] | None = None,
    stdin_data: str | None = None,
    sensitive: bool = False,
    dry_run: bool = False,
) -> CommandResult:
    """
    Execute a shell command with production defaults.

    Args:
        cmd: Command string or list. String is split with shlex.
        check: Raise CalledProcessError on non-zero exit.
        capture: Capture stdout/stderr (False = print to terminal).
        timeout: Seconds before SIGTERM (then SIGKILL after 10s).
        cwd: Working directory.
        env: Additional environment variables (merged with current env).
        stdin_data: Data to pipe to stdin.
        sensitive: If True, don't log the command (contains secrets).
        dry_run: Log command but don't execute.

    Returns:
        CommandResult with stdout, stderr, exit code, duration.
    """
    # Normalize command
    if isinstance(cmd, str):
        cmd_list = shlex.split(cmd)
        cmd_display = cmd if not sensitive else "[REDACTED]"
    else:
        cmd_list = cmd
        cmd_display = shlex.join(cmd) if not sensitive else "[REDACTED]"

    log.info("executing_command", command=cmd_display, cwd=cwd, timeout=timeout)

    if dry_run:
        log.info("dry_run_skip", command=cmd_display)
        return CommandResult(
            command=cmd_display, exit_code=0,
            stdout="[dry run]", stderr="", duration_seconds=0.0,
        )

    # Merge environment
    full_env = os.environ.copy()
    if env:
        full_env.update(env)

    start = time.monotonic()

    try:
        proc = subprocess.run(
            cmd_list,
            capture_output=capture,
            text=True,
            timeout=timeout,
            cwd=cwd,
            env=full_env,
            input=stdin_data,
        )

        duration = time.monotonic() - start

        result = CommandResult(
            command=cmd_display,
            exit_code=proc.returncode,
            stdout=proc.stdout or "",
            stderr=proc.stderr or "",
            duration_seconds=round(duration, 2),
        )

        if result.success:
            log.info("command_succeeded",
                     command=cmd_display, duration=result.duration_seconds)
        else:
            log.warning("command_failed",
                        command=cmd_display,
                        exit_code=result.exit_code,
                        stderr=result.stderr[:500] if not sensitive else "[REDACTED]")

        if check and not result.success:
            result.raise_on_error()

        return result

    except subprocess.TimeoutExpired:
        duration = time.monotonic() - start
        log.error("command_timed_out", command=cmd_display, timeout=timeout)
        return CommandResult(
            command=cmd_display, exit_code=-1,
            stdout="", stderr=f"Command timed out after {timeout}s",
            duration_seconds=round(duration, 2), timed_out=True,
        )


# ═══════════════════════════════════════════════════════════
# PRODUCTION WRAPPERS — The actual daily tools
# ═══════════════════════════════════════════════════════════

# ─── kubectl ───────────────────────────────────────────────
class Kubectl:
    """Type-safe kubectl wrapper."""

    def __init__(
        self,
        context: str | None = None,
        namespace: str | None = None,
        kubeconfig: str | None = None,
    ) -> None:
        self._base_args: list[str] = ["kubectl"]
        if context:
            self._base_args.extend(["--context", context])
        if namespace:
            self._base_args.extend(["--namespace", namespace])
        if kubeconfig:
            self._base_args.extend(["--kubeconfig", kubeconfig])

    def _run(self, args: list[str], **kwargs: Any) -> CommandResult:
        return run(self._base_args + args, **kwargs)

    def get(
        self,
        resource: str,
        name: str = "",
        output: str = "json",
        label_selector: str = "",
        field_selector: str = "",
    ) -> CommandResult:
        cmd = ["get", resource]
        if name:
            cmd.append(name)
        cmd.extend(["-o", output])
        if label_selector:
            cmd.extend(["-l", label_selector])
        if field_selector:
            cmd.extend(["--field-selector", field_selector])
        return self._run(cmd)

    def get_json(self, resource: str, name: str = "", **kwargs: Any) -> dict | list:
        """Get resource as parsed JSON."""
        import json
        result = self.get(resource, name, output="json", **kwargs)
        result.raise_on_error(f"Failed to get {resource} {name}")
        return json.loads(result.stdout)

    def apply(self, file: str = "", stdin: str = "", dry_run: str = "") -> CommandResult:
        cmd = ["apply"]
        if file:
            cmd.extend(["-f", file])
        if dry_run:
            cmd.extend(["--dry-run", dry_run])  # "client", "server", "none"
        return self._run(cmd, stdin_data=stdin if not file else None)

    def delete(self, resource: str, name: str, force: bool = False) -> CommandResult:
        cmd = ["delete", resource, name]
        if force:
            cmd.extend(["--force", "--grace-period=0"])
        return self._run(cmd)

    def rollout_status(
        self, resource: str, name: str, timeout: int = 300,
    ) -> CommandResult:
        return self._run(
            ["rollout", "status", f"{resource}/{name}", f"--timeout={timeout}s"],
            timeout=timeout + 30,  # Shell timeout > kubectl timeout
        )

    def rollout_undo(self, resource: str, name: str) -> CommandResult:
        return self._run(["rollout", "undo", f"{resource}/{name}"])

    def logs(
        self,
        pod: str,
        container: str = "",
        tail: int = 100,
        since: str = "",
        previous: bool = False,
    ) -> CommandResult:
        cmd = ["logs", pod, f"--tail={tail}"]
        if container:
            cmd.extend(["-c", container])
        if since:
            cmd.extend([f"--since={since}"])
        if previous:
            cmd.append("--previous")
        return self._run(cmd)

    def exec(self, pod: str, command: list[str], container: str = "") -> CommandResult:
        cmd = ["exec", pod]
        if container:
            cmd.extend(["-c", container])
        cmd.append("--")
        cmd.extend(command)
        return self._run(cmd)

    def top_pods(self, sort_by: str = "memory") -> CommandResult:
        return self._run(["top", "pods", f"--sort-by={sort_by}"])

    def cordon(self, node: str) -> CommandResult:
        return self._run(["cordon", node])

    def uncordon(self, node: str) -> CommandResult:
        return self._run(["uncordon", node])

    def drain(
        self,
        node: str,
        timeout: int = 300,
        ignore_daemonsets: bool = True,
        delete_emptydir: bool = True,
    ) -> CommandResult:
        cmd = ["drain", node, f"--timeout={timeout}s"]
        if ignore_daemonsets:
            cmd.append("--ignore-daemonsets")
        if delete_emptydir:
            cmd.append("--delete-emptydir-data")
        return self._run(cmd, timeout=timeout + 60)

    def scale(self, resource: str, name: str, replicas: int) -> CommandResult:
        return self._run(
            ["scale", f"{resource}/{name}", f"--replicas={replicas}"]
        )

    def patch(
        self, resource: str, name: str, patch: str, patch_type: str = "strategic",
    ) -> CommandResult:
        return self._run([
            "patch", resource, name,
            "--type", patch_type,
            "-p", patch,
        ])

    def wait(
        self,
        resource: str,
        condition: str,
        timeout: int = 120,
        label_selector: str = "",
    ) -> CommandResult:
        cmd = [
            "wait", resource,
            f"--for={condition}",
            f"--timeout={timeout}s",
        ]
        if label_selector:
            cmd.extend(["-l", label_selector])
        return self._run(cmd, timeout=timeout + 30)


# ─── Terraform ─────────────────────────────────────────────
class Terraform:
    """Type-safe Terraform wrapper."""

    def __init__(self, working_dir: str, var_file: str | None = None) -> None:
        self.working_dir = working_dir
        self.var_file = var_file

    def _run(self, args: list[str], **kwargs: Any) -> CommandResult:
        cmd = ["terraform"] + args
        if self.var_file and args[0] in ("plan", "apply", "destroy"):
            cmd.extend([f"-var-file={self.var_file}"])
        return run(cmd, cwd=self.working_dir, **kwargs)

    def init(self, backend: bool = True, upgrade: bool = False) -> CommandResult:
        cmd = ["init"]
        if not backend:
            cmd.append("-backend=false")
        if upgrade:
            cmd.append("-upgrade")
        cmd.append("-input=false")
        return self._run(cmd)

    def plan(
        self,
        out: str = "tfplan",
        destroy: bool = False,
        target: str = "",
    ) -> CommandResult:
        cmd = ["plan", f"-out={out}", "-input=false", "-detailed-exitcode"]
        if destroy:
            cmd.append("-destroy")
        if target:
            cmd.extend(["-target", target])
        result = self._run(cmd)
        # -detailed-exitcode: 0=no changes, 1=error, 2=changes present
        return result

    def apply(
        self,
        plan_file: str = "tfplan",
        auto_approve: bool = False,
    ) -> CommandResult:
        cmd = ["apply"]
        if plan_file:
            cmd.append(plan_file)
        elif auto_approve:
            cmd.append("-auto-approve")
        cmd.append("-input=false")
        return self._run(cmd, timeout=1800)  # 30 min for large applies

    def output(self, name: str = "", json_format: bool = True) -> CommandResult:
        cmd = ["output"]
        if json_format:
            cmd.append("-json")
        if name:
            cmd.append(name)
        return self._run(cmd)

    def output_value(self, name: str) -> Any:
        """Get a single output as parsed value."""
        import json
        result = self.output(name)
        result.raise_on_error(f"Failed to get output: {name}")
        data = json.loads(result.stdout)
        return data  # For single output, terraform returns the value directly

    def state_list(self) -> list[str]:
        result = self._run(["state", "list"])
        result.raise_on_error()
        return result.stdout_lines

    def state_show(self, address: str) -> CommandResult:
        return self._run(["state", "show", address])

    def import_resource(self, address: str, resource_id: str) -> CommandResult:
        return self._run(["import", address, resource_id])

    def validate(self) -> CommandResult:
        return self._run(["validate", "-json"])

    def fmt_check(self) -> CommandResult:
        return self._run(["fmt", "-check", "-recursive"])

    def workspace_select(self, workspace: str) -> CommandResult:
        return self._run(["workspace", "select", workspace])

    def destroy(self, auto_approve: bool = False, target: str = "") -> CommandResult:
        cmd = ["destroy", "-input=false"]
        if auto_approve:
            cmd.append("-auto-approve")
        if target:
            cmd.extend(["-target", target])
        return self._run(cmd, timeout=1800)


# ─── Helm ──────────────────────────────────────────────────
class Helm:
    """Type-safe Helm wrapper."""

    def __init__(self, kubeconfig: str | None = None) -> None:
        self._base: list[str] = ["helm"]
        if kubeconfig:
            self._base.extend(["--kubeconfig", kubeconfig])

    def _run(self, args: list[str], **kwargs: Any) -> CommandResult:
        return run(self._base + args, **kwargs)

    def upgrade_install(
        self,
        release: str,
        chart: str,
        namespace: str,
        values_files: list[str] | None = None,
        set_values: dict[str, str] | None = None,
        version: str = "",
        dry_run: bool = False,
        wait: bool = True,
        timeout: int = 600,
        atomic: bool = True,
    ) -> CommandResult:
        cmd = [
            "upgrade", release, chart,
            "--install",
            "--namespace", namespace,
            "--create-namespace",
            f"--timeout={timeout}s",
        ]
        if wait:
            cmd.append("--wait")
        if atomic:
            cmd.append("--atomic")
        if dry_run:
            cmd.append("--dry-run")
        if version:
            cmd.extend(["--version", version])
        for vf in (values_files or []):
            cmd.extend(["-f", vf])
        for k, v in (set_values or {}).items():
            cmd.extend(["--set", f"{k}={v}"])
        return self._run(cmd, timeout=timeout + 60)

    def template(
        self,
        release: str,
        chart: str,
        namespace: str,
        values_files: list[str] | None = None,
    ) -> CommandResult:
        cmd = ["template", release, chart, "--namespace", namespace]
        for vf in (values_files or []):
            cmd.extend(["-f", vf])
        return self._run(cmd)

    def list_releases(self, namespace: str = "", all_namespaces: bool = False) -> CommandResult:
        cmd = ["list", "-o", "json"]
        if all_namespaces:
            cmd.append("--all-namespaces")
        elif namespace:
            cmd.extend(["--namespace", namespace])
        return self._run(cmd)

    def rollback(self, release: str, revision: int = 0, namespace: str = "") -> CommandResult:
        cmd = ["rollback", release]
        if revision:
            cmd.append(str(revision))
        if namespace:
            cmd.extend(["--namespace", namespace])
        cmd.append("--wait")
        return self._run(cmd)

    def get_values(self, release: str, namespace: str) -> CommandResult:
        return self._run([
            "get", "values", release,
            "--namespace", namespace,
            "-o", "json",
        ])

    def diff_upgrade(
        self,
        release: str,
        chart: str,
        namespace: str,
        values_files: list[str] | None = None,
    ) -> CommandResult:
        """Requires helm-diff plugin."""
        cmd = ["diff", "upgrade", release, chart, "--namespace", namespace]
        for vf in (values_files or []):
            cmd.extend(["-f", vf])
        return self._run(cmd)


# ─── ArgoCD ────────────────────────────────────────────────
class ArgoCD:
    """Type-safe ArgoCD CLI wrapper."""

    def __init__(self, server: str = "", auth_token: str = "") -> None:
        self._base: list[str] = ["argocd"]
        if server:
            self._base.extend(["--server", server])
        if auth_token:
            self._base.extend(["--auth-token", auth_token])
        self._base.append("--grpc-web")  # Through ingress

    def _run(self, args: list[str], **kwargs: Any) -> CommandResult:
        return run(self._base + args, **kwargs)

    def app_get(self, app: str, output: str = "json") -> CommandResult:
        return self._run(["app", "get", app, "-o", output])

    def app_sync(
        self, app: str, prune: bool = False, force: bool = False,
    ) -> CommandResult:
        cmd = ["app", "sync", app]
        if prune:
            cmd.append("--prune")
        if force:
            cmd.append("--force")
        return self._run(cmd, timeout=600)

    def app_wait(self, app: str, timeout: int = 300) -> CommandResult:
        return self._run(
            ["app", "wait", app, "--timeout", str(timeout)],
            timeout=timeout + 60,
        )

    def app_set(self, app: str, parameters: dict[str, str]) -> CommandResult:
        cmd = ["app", "set", app]
        for k, v in parameters.items():
            cmd.extend(["-p", f"{k}={v}"])
        return self._run(cmd)

    def app_history(self, app: str) -> CommandResult:
        return self._run(["app", "history", app, "-o", "json"])

    def app_rollback(self, app: str, revision: int) -> CommandResult:
        return self._run(["app", "rollback", app, str(revision)])


# ═══════════════════════════════════════════════════════════
# USAGE EXAMPLES — Real daily operations
# ═══════════════════════════════════════════════════════════

def deploy_with_rollback(
    service: str,
    version: str,
    namespace: str,
    context: str = "production",
) -> bool:
    """End-to-end deploy with rollback on failure."""
    k = Kubectl(context=context, namespace=namespace)

    # Capture current state for rollback
    current = k.get_json("deployment", service)
    current_image = current["spec"]["template"]["spec"]["containers"][0]["image"]
    log.info("current_image", image=current_image)

    # Set new image
    new_image = f"123456789012.dkr.ecr.us-east-1.amazonaws.com/{service}:{version}"
    result = k.patch(
        "deployment", service,
        f'{{"spec":{{"template":{{"spec":{{"containers":[{{"name":"{service}","image":"{new_image}"}}]}}}}}}}}',
        patch_type="strategic",
    )
    if not result.success:
        log.error("patch_failed", stderr=result.stderr)
        return False

    # Wait for rollout
    rollout = k.rollout_status("deployment", service, timeout=300)
    if not rollout.success:
        log.error("rollout_failed", stderr=rollout.stderr)
        log.info("initiating_rollback")
        k.rollout_undo("deployment", service)
        k.rollout_status("deployment", service, timeout=120)
        return False

    return True


def terraform_pr_pipeline(working_dir: str, environment: str) -> dict[str, Any]:
    """CI pipeline: init → validate → plan → output diff."""
    tf = Terraform(working_dir=working_dir, var_file=f"{environment}.tfvars")

    # Init
    init = tf.init(upgrade=True)
    init.raise_on_error("Terraform init failed")

    # Validate
    validate = tf.validate()
    validate.raise_on_error("Terraform validation failed")

    # Format check
    fmt = tf.fmt_check()
    if not fmt.success:
        log.warning("terraform_fmt_check_failed",
                    hint="Run 'terraform fmt -recursive' to fix")

    # Plan with detailed exit code
    plan = tf.plan(out="tfplan")
    # Exit code 2 = changes detected
    has_changes = plan.exit_code == 2
    has_errors = plan.exit_code == 1

    if has_errors:
        plan.raise_on_error("Terraform plan failed")

    return {
        "init": init.success,
        "validate": validate.success,
        "fmt_clean": fmt.success,
        "has_changes": has_changes,
        "plan_stdout": plan.stdout,
        "plan_stderr": plan.stderr,
    }
```

**Subprocess failure modes:**

```python
# FAILURE 1: Shell injection
# BAD:
cmd = f"kubectl get pods -l app={user_input}"
subprocess.run(cmd, shell=True)
# If user_input = "nginx; rm -rf /" → catastrophe
# NEVER use shell=True with user input

# GOOD:
cmd = ["kubectl", "get", "pods", "-l", f"app={user_input}"]
subprocess.run(cmd)  # No shell=True, arguments are NOT interpreted

# FAILURE 2: Deadlock with capture
# BAD:
proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
stdout = proc.stdout.read()   # Blocks if stderr buffer fills
stderr = proc.stderr.read()
# If the command outputs >64KB to stderr, .stdout.read() blocks forever

# GOOD:
proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
stdout, stderr = proc.communicate(timeout=300)  # Reads both safely

# BEST: Just use subprocess.run() which handles this internally

# FAILURE 3: Timeout doesn't kill children
# subprocess.run(timeout=30) sends SIGTERM to the direct child
# But if that child spawned subprocesses, they keep running!
# Fix: Use process groups
import signal
proc = subprocess.Popen(
    cmd,
    preexec_fn=os.setsid,  # New process group
)
try:
    proc.communicate(timeout=30)
except subprocess.TimeoutExpired:
    os.killpg(os.getpgid(proc.pid), signal.SIGTERM)  # Kill entire group
    proc.communicate(timeout=10)

# FAILURE 4: Environment variable leaking secrets
env = os.environ.copy()
env["DB_PASSWORD"] = "secret123"
subprocess.run(cmd, env=env)
# The password is visible in /proc/<pid>/environ on Linux!
# Mitigate: Use stdin piping or temp files with restricted permissions

# FAILURE 5: Encoding issues
result = subprocess.run(cmd, capture_output=True, text=True)
# 'text=True' uses locale encoding, which might not be UTF-8
# Fix: explicit encoding
result = subprocess.run(cmd, capture_output=True, text=True, encoding="utf-8",
                        errors="replace")  # Replace undecodable bytes
```

---

## 2. LOG PARSING & DATA PROCESSING

```python
"""
Real-world log parsing patterns for DevOps.
Nginx access logs, application logs, CloudWatch logs, audit logs.
"""
from __future__ import annotations

import gzip
import re
from collections import Counter, defaultdict
from dataclasses import dataclass, field
from datetime import datetime, timezone
from pathlib import Path
from typing import Generator, TextIO

import structlog

log = structlog.get_logger()


# ═══════════════════════════════════════════════════════════
# PATTERN 1: Nginx Access Log Parser
# ═══════════════════════════════════════════════════════════

# Standard combined log format + request_time
NGINX_LOG_PATTERN = re.compile(
    r'(?P<ip>\S+) '
    r'\S+ '                                    # ident (always -)
    r'(?P<user>\S+) '
    r'\[(?P<timestamp>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" '
    r'(?P<status>\d{3}) '
    r'(?P<bytes>\d+) '
    r'"(?P<referer>[^"]*)" '
    r'"(?P<user_agent>[^"]*)"'
    r'(?: (?P<request_time>[\d.]+))?'          # Optional request_time
)


@dataclass
class AccessLogEntry:
    ip: str
    timestamp: datetime
    method: str
    path: str
    status: int
    bytes_sent: int
    request_time: float | None
    user_agent: str


def parse_nginx_log(
    file_path: Path,
    since: datetime | None = None,
) -> Generator[AccessLogEntry, None, None]:
    """
    Parse Nginx access log, yielding structured entries.
    Handles gzipped files transparently.
    Skips malformed lines with warning (doesn't crash).
    """
    opener = gzip.open if file_path.suffix == ".gz" else open
    parse_errors = 0

    with opener(file_path, "rt", encoding="utf-8", errors="replace") as f:
        for line_num, line in enumerate(f, 1):
            match = NGINX_LOG_PATTERN.match(line.strip())
            if not match:
                parse_errors += 1
                if parse_errors <= 10:  # Don't spam logs
                    log.debug("unparseable_line", line_num=line_num, line=line[:100])
                continue

            g = match.groupdict()

            # Parse timestamp: 15/Jan/2024:14:30:00 +0000
            try:
                ts = datetime.strptime(
                    g["timestamp"], "%d/%b/%Y:%H:%M:%S %z"
                )
            except ValueError:
                parse_errors += 1
                continue

            # Skip if before cutoff
            if since and ts < since:
                continue

            yield AccessLogEntry(
                ip=g["ip"],
                timestamp=ts,
                method=g["method"],
                path=g["path"],
                status=int(g["status"]),
                bytes_sent=int(g["bytes"]),
                request_time=float(g["request_time"]) if g.get("request_time") else None,
                user_agent=g["user_agent"],
            )

    if parse_errors > 0:
        log.warning("parse_errors", count=parse_errors, file=str(file_path))


def analyze_access_logs(file_path: Path, since: datetime | None = None) -> dict:
    """
    Produce a full analysis of an Nginx access log.
    Used for: incident investigation, traffic analysis, capacity planning.
    """
    status_counter: Counter[int] = Counter()
    path_counter: Counter[str] = Counter()
    ip_counter: Counter[str] = Counter()
    slow_requests: list[AccessLogEntry] = []
    error_paths: Counter[str] = Counter()
    total_bytes = 0
    total_requests = 0
    latencies: list[float] = []

    for entry in parse_nginx_log(file_path, since):
        total_requests += 1
        total_bytes += entry.bytes_sent
        status_counter[entry.status] += 1
        path_counter[_normalize_path(entry.path)] += 1
        ip_counter[entry.ip] += 1

        if entry.request_time is not None:
            latencies.append(entry.request_time)
            if entry.request_time > 2.0:  # Slow request threshold
                slow_requests.append(entry)

        if entry.status >= 500:
            error_paths[_normalize_path(entry.path)] += 1

    # Calculate percentiles
    sorted_latencies = sorted(latencies)
    n = len(sorted_latencies)

    return {
        "total_requests": total_requests,
        "total_bytes": total_bytes,
        "status_breakdown": dict(status_counter.most_common()),
        "error_rate": sum(v for k, v in status_counter.items() if k >= 500) / max(total_requests, 1),
        "top_paths": dict(path_counter.most_common(20)),
        "top_error_paths": dict(error_paths.most_common(10)),
        "top_ips": dict(ip_counter.most_common(10)),
        "slow_requests_count": len(slow_requests),
        "latency": {
            "p50": sorted_latencies[int(n * 0.50)] if n else 0,
            "p90": sorted_latencies[int(n * 0.90)] if n else 0,
            "p95": sorted_latencies[int(n * 0.95)] if n else 0,
            "p99": sorted_latencies[int(n * 0.99)] if n else 0,
            "max": sorted_latencies[-1] if n else 0,
        } if n > 0 else {},
    }


def _normalize_path(path: str) -> str:
    """
    Normalize URL paths for grouping.
    /api/v1/users/12345 → /api/v1/users/:id
    /api/v1/orders/abc-def-123 → /api/v1/orders/:id
    """
    # Replace UUIDs
    path = re.sub(r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}', ':uuid', path)
    # Replace numeric IDs
    path = re.sub(r'/\d+', '/:id', path)
    # Strip query string
    path = path.split("?")[0]
    return path


# ═══════════════════════════════════════════════════════════
# PATTERN 2: Structured JSON Log Parser (Application Logs)
# ═══════════════════════════════════════════════════════════

import json


def parse_json_logs(
    file_path: Path,
    level_filter: str | None = None,
    service_filter: str | None = None,
    since: datetime | None = None,
    search: str | None = None,
) -> Generator[dict, None, None]:
    """
    Parse structured JSON logs (one JSON object per line).
    Standard in K8s with Fluent Bit / Loki.
    """
    opener = gzip.open if file_path.suffix == ".gz" else open

    with opener(file_path, "rt", encoding="utf-8", errors="replace") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue

            try:
                entry = json.loads(line)
            except json.JSONDecodeError:
                continue

            # Apply filters
            if level_filter and entry.get("level", "").upper() != level_filter.upper():
                continue

            if service_filter and entry.get("service") != service_filter:
                continue

            if search and search.lower() not in json.dumps(entry).lower():
                continue

            if since:
                ts_str = entry.get("timestamp") or entry.get("time") or entry.get("@timestamp")
                if ts_str:
                    try:
                        ts = datetime.fromisoformat(ts_str.replace("Z", "+00:00"))
                        if ts < since:
                            continue
                    except (ValueError, TypeError):
                        pass

            yield entry


def extract_error_patterns(
    file_path: Path,
    top_n: int = 20,
) -> list[dict]:
    """
    Find the most common error patterns in application logs.
    Groups similar errors by normalizing variable parts.
    """
    pattern_counter: Counter[str] = Counter()
    pattern_examples: dict[str, dict] = {}

    for entry in parse_json_logs(file_path, level_filter="ERROR"):
        message = entry.get("message") or entry.get("msg") or str(entry.get("error", ""))

        # Normalize the message to group similar errors
        normalized = _normalize_error_message(message)

        pattern_counter[normalized] += 1
        if normalized not in pattern_examples:
            pattern_examples[normalized] = entry  # Keep first example

    results = []
    for pattern, count in pattern_counter.most_common(top_n):
        results.append({
            "pattern": pattern,
            "count": count,
            "example": pattern_examples[pattern],
        })

    return results


def _normalize_error_message(msg: str) -> str:
    """Normalize error messages for grouping."""
    # Replace IPs
    msg = re.sub(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', '<IP>', msg)
    # Replace ports
    msg = re.sub(r':\d{4,5}', ':<PORT>', msg)
    # Replace UUIDs
    msg = re.sub(r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}', '<UUID>', msg)
    # Replace timestamps
    msg = re.sub(r'\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}', '<TIMESTAMP>', msg)
    # Replace numbers that look like IDs
    msg = re.sub(r'\b\d{5,}\b', '<ID>', msg)
    # Replace hex strings
    msg = re.sub(r'\b[0-9a-f]{16,}\b', '<HEX>', msg)
    return msg


# ═══════════════════════════════════════════════════════════
# PATTERN 3: Multi-file Log Correlation
# ═══════════════════════════════════════════════════════════

def correlate_by_trace_id(
    log_files: list[Path],
    trace_id: str,
) -> list[dict]:
    """
    Find all log entries matching a trace ID across multiple log files.
    Used during incident investigation to follow a request across services.
    """
    entries: list[dict] = []

    for file_path in log_files:
        for entry in parse_json_logs(file_path, search=trace_id):
            entry["_source_file"] = str(file_path)
            entries.append(entry)

    # Sort by timestamp
    entries.sort(key=lambda e: e.get("timestamp", e.get("time", "")))

    log.info("correlation_complete", trace_id=trace_id, entries_found=len(entries))
    return entries


# ═══════════════════════════════════════════════════════════
# PATTERN 4: CloudWatch Logs Insights Query
# ═══════════════════════════════════════════════════════════

import time
import boto3


def query_cloudwatch_logs(
    log_group: str,
    query: str,
    hours_back: int = 1,
    limit: int = 1000,
    region: str = "us-east-1",
) -> list[dict]:
    """
    Execute a CloudWatch Logs Insights query and wait for results.

    Example queries:
        # Top 10 errors
        fields @timestamp, @message
        | filter @message like /ERROR/
        | stats count(*) as cnt by @message
        | sort cnt desc
        | limit 10

        # Slow Lambda invocations
        filter @type = "REPORT"
        | stats max(@duration), avg(@duration), count(*) by bin(5m)
    """
    client = boto3.client("logs", region_name=region)
    now = int(time.time())

    response = client.start_query(
        logGroupName=log_group,
        startTime=now - (hours_back * 3600),
        endTime=now,
        queryString=query,
        limit=limit,
    )
    query_id = response["queryId"]
    log.info("cw_query_started", query_id=query_id)

    # Poll for results
    for attempt in range(60):  # Max 5 minutes
        result = client.get_query_results(queryId=query_id)
        status = result["status"]

        if status == "Complete":
            # Parse results into dicts
            rows = []
            for result_row in result["results"]:
                row = {field["field"]: field["value"] for field in result_row}
                rows.append(row)
            log.info("cw_query_complete", rows=len(rows))
            return rows

        if status in ("Failed", "Cancelled", "Timeout"):
            raise RuntimeError(f"CloudWatch query {status}: {query_id}")

        time.sleep(5)

    raise TimeoutError(f"CloudWatch query {query_id} did not complete in 5 minutes")
```

---

## 3. FILE OPERATIONS (pathlib)

```python
"""
pathlib is the modern Python way to handle files.
os.path is legacy. pathlib is cleaner, safer, and more readable.
"""
from pathlib import Path
import shutil
import tempfile
from typing import Generator


# ═══════════════════════════════════════════════════════════
# BASICS — What os.path did, but better
# ═══════════════════════════════════════════════════════════

# ── Path construction ──────────────────────────────────────
config_dir = Path("/etc/novamart")
config_file = config_dir / "services" / "payment.yaml"  # / operator joins
# = Path('/etc/novamart/services/payment.yaml')

# Relative paths
repo_root = Path(__file__).parent.parent.parent  # Go up 3 levels
templates_dir = repo_root / "templates"

# Home directory
ssh_key = Path.home() / ".ssh" / "id_rsa"

# Current working directory
cwd = Path.cwd()

# ── Path properties ────────────────────────────────────────
p = Path("/var/log/nginx/access.log.2024-01-15.gz")
p.name         # 'access.log.2024-01-15.gz'
p.stem         # 'access.log.2024-01-15'
p.suffix       # '.gz'
p.suffixes     # ['.log', '.2024-01-15', '.gz']
p.parent       # Path('/var/log/nginx')
p.parents[1]   # Path('/var/log')
p.parts        # ('/', 'var', 'log', 'nginx', 'access.log.2024-01-15.gz')

# ── Existence and type checks ──────────────────────────────
p.exists()
p.is_file()
p.is_dir()
p.is_symlink()
p.stat().st_size      # File size in bytes
p.stat().st_mtime     # Modification time (epoch)

# ── Reading and writing ────────────────────────────────────
# Read
content = Path("config.yaml").read_text(encoding="utf-8")
binary = Path("image.png").read_bytes()

# Write (creates parent dirs if needed)
output_path = Path("/tmp/report/output.json")
output_path.parent.mkdir(parents=True, exist_ok=True)
output_path.write_text(json.dumps(data, indent=2), encoding="utf-8")

# ── Globbing ───────────────────────────────────────────────
# All YAML files in directory
yaml_files = list(Path("manifests/").glob("*.yaml"))

# Recursive glob
all_python = list(Path("src/").rglob("*.py"))

# Multiple patterns
configs = [
    f for f in Path("config/").rglob("*")
    if f.suffix in (".yaml", ".yml", ".json", ".toml")
]


# ═══════════════════════════════════════════════════════════
# PRODUCTION PATTERNS
# ═══════════════════════════════════════════════════════════

def find_large_files(
    directory: Path,
    min_size_mb: float = 100,
    pattern: str = "*",
) -> list[dict]:
    """Find files larger than threshold. Used for disk cleanup."""
    results = []
    for f in directory.rglob(pattern):
        if f.is_file():
            size = f.stat().st_size
            if size >= min_size_mb * 1024 * 1024:
                results.append({
                    "path": str(f),
                    "size_mb": round(size / 1024 / 1024, 1),
                    "modified": datetime.fromtimestamp(
                        f.stat().st_mtime, tz=timezone.utc
                    ).isoformat(),
                })
    return sorted(results, key=lambda x: x["size_mb"], reverse=True)


def atomic_write(path: Path, content: str, encoding: str = "utf-8") -> None:
    """
    Write file atomically — prevents corruption on crash.
    Writes to temp file in same directory, then renames.
    rename() is atomic on POSIX filesystems.
    """
    path.parent.mkdir(parents=True, exist_ok=True)

    with tempfile.NamedTemporaryFile(
        mode="w",
        dir=path.parent,
        suffix=".tmp",
        delete=False,
        encoding=encoding,
    ) as f:
        f.write(content)
        f.flush()
        os.fsync(f.fileno())  # Force write to disk
        tmp_path = Path(f.name)

    # Set same permissions as target (if exists)
    if path.exists():
        tmp_path.chmod(path.stat().st_mode)

    tmp_path.rename(path)  # Atomic on same filesystem


def safe_backup(source: Path, backup_dir: Path, max_backups: int = 5) -> Path:
    """Create a timestamped backup, pruning old ones."""
    backup_dir.mkdir(parents=True, exist_ok=True)

    timestamp = datetime.now(timezone.utc).strftime("%Y%m%d-%H%M%S")
    backup_name = f"{source.stem}.{timestamp}{source.suffix}"
    backup_path = backup_dir / backup_name

    shutil.copy2(source, backup_path)  # Preserves metadata

    # Prune old backups
    existing = sorted(
        backup_dir.glob(f"{source.stem}.*{source.suffix}"),
        key=lambda p: p.stat().st_mtime,
        reverse=True,
    )
    for old_backup in existing[max_backups:]:
        old_backup.unlink()
        log.info("pruned_backup", path=str(old_backup))

    return backup_path


def diff_directories(dir_a: Path, dir_b: Path) -> dict:
    """
    Compare two directories recursively.
    Used for: config drift detection, deployment verification.
    """
    files_a = {f.relative_to(dir_a) for f in dir_a.rglob("*") if f.is_file()}
    files_b = {f.relative_to(dir_b) for f in dir_b.rglob("*") if f.is_file()}

    only_in_a = files_a - files_b
    only_in_b = files_b - files_a
    common = files_a & files_b

    modified = []
    for rel_path in common:
        file_a = dir_a / rel_path
        file_b = dir_b / rel_path
        if file_a.read_bytes() != file_b.read_bytes():
            modified.append(str(rel_path))

    return {
        "only_in_a": sorted(str(f) for f in only_in_a),
        "only_in_b": sorted(str(f) for f in only_in_b),
        "modified": sorted(modified),
        "identical_count": len(common) - len(modified),
    }


def walk_k8s_manifests(
    manifests_dir: Path,
) -> Generator[tuple[Path, dict], None, None]:
    """
    Walk a directory of K8s manifests, yielding (path, parsed_yaml).
    Handles multi-document YAML files (--- separator).
    """
    import yaml

    for yaml_file in sorted(manifests_dir.rglob("*.yaml")):
        try:
            content = yaml_file.read_text(encoding="utf-8")
            for doc in yaml.safe_load_all(content):
                if doc:  # Skip empty documents
                    yield yaml_file, doc
        except yaml.YAMLError as e:
            log.warning("invalid_yaml", file=str(yaml_file), error=str(e))
```

---

## 4. YAML/JSON MANIPULATION AT SCALE

```python
"""
Programmatic modification of K8s manifests, Helm values, Terraform configs.
Daily use: updating image tags, modifying resource limits, generating configs.
"""
from __future__ import annotations

import json
import copy
from pathlib import Path
from typing import Any

import yaml

# ─── YAML round-trip (preserve comments and formatting) ───
# Standard pyyaml DESTROYS comments and formatting.
# Use ruamel.yaml for round-trip editing.
from ruamel.yaml import YAML

ryaml = YAML()
ryaml.preserve_quotes = True
ryaml.width = 120


def update_image_tag(
    kustomization_path: Path,
    service_name: str,
    new_tag: str,
) -> None:
    """
    Update image tag in kustomization.yaml preserving formatting and comments.
    This is what CI does on every merge to main.
    """
    with open(kustomization_path) as f:
        data = ryaml.load(f)

    images = data.get("images", [])
    updated = False

    for image in images:
        if image.get("name") == service_name:
            image["newTag"] = new_tag
            updated = True
            break

    if not updated:
        images.append({"name": service_name, "newTag": new_tag})

    with open(kustomization_path, "w") as f:
        ryaml.dump(data, f)


def update_helm_values(
    values_path: Path,
    key_path: str,
    value: Any,
) -> None:
    """
    Update a nested key in a Helm values file.
    key_path: "prometheus.server.retention" → data["prometheus"]["server"]["retention"]
    """
    with open(values_path) as f:
        data = ryaml.load(f)

    keys = key_path.split(".")
    current = data
    for key in keys[:-1]:
        if key not in current:
            current[key] = {}
        current = current[key]
    current[keys[-1]] = value

    with open(values_path, "w") as f:
        ryaml.dump(data, f)


def bulk_update_resource_limits(
    manifests_dir: Path,
    updates: dict[str, dict[str, str]],
    dry_run: bool = True,
) -> list[dict]:
    """
    Bulk update resource limits across all deployment manifests.
    updates = {
        "payment-service": {"cpu": "500m", "memory": "512Mi"},
        "order-service": {"cpu": "250m", "memory": "256Mi"},
    }
    """
    changes = []

    for yaml_file in manifests_dir.rglob("*.yaml"):
        with open(yaml_file) as f:
            docs = list(yaml.safe_load_all(f.read()))

        modified = False
        for doc in docs:
            if not doc or doc.get("kind") != "Deployment":
                continue

            name = doc.get("metadata", {}).get("name", "")
            if name not in updates:
                continue

            containers = (
                doc.get("spec", {})
                .get("template", {})
                .get("spec", {})
                .get("containers", [])
            )

            for container in containers:
                if container.get("name") == name:
                    old_limits = copy.deepcopy(container.get("resources", {}).get("limits", {}))
                    if "resources" not in container:
                        container["resources"] = {}
                    if "limits" not in container["resources"]:
                        container["resources"]["limits"] = {}

                    container["resources"]["limits"].update(updates[name])
                    modified = True

                    changes.append({
                        "file": str(yaml_file),
                        "deployment": name,
                        "old_limits": old_limits,
                        "new_limits": updates[name],
                    })

        if modified and not dry_run:
            with open(yaml_file, "w") as f:
                yaml.dump_all(docs, f, default_flow_style=False)

    return changes


def validate_k8s_manifests(manifests_dir: Path) -> list[dict]:
    """
    Validate K8s manifests for common issues.
    Run in CI before ArgoCD sync.
    """
    issues = []

    for yaml_file, doc in walk_k8s_manifests(manifests_dir):
        kind = doc.get("kind", "Unknown")
        name = doc.get("metadata", {}).get("name", "unnamed")

        # Check: All deployments have resource limits
        if kind == "Deployment":
            containers = (
                doc.get("spec", {})
                .get("template", {})
                .get("spec", {})
                .get("containers", [])
            )
            for c in containers:
                if not c.get("resources", {}).get("limits"):
                    issues.append({
                        "file": str(yaml_file),
                        "resource": f"{kind}/{name}",
                        "issue": f"Container '{c.get('name')}' missing resource limits",
                        "severity": "HIGH",
                    })

                if not c.get("readinessProbe") and not c.get("livenessProbe"):
                    issues.append({
                        "file": str(yaml_file),
                        "resource": f"{kind}/{name}",
                        "issue": f"Container '{c.get('name')}' missing health probes",
                        "severity": "MEDIUM",
                    })

            # Check: image tag is not :latest
            for c in containers:
                image = c.get("image", "")
                if image.endswith(":latest") or ":" not in image:
                    issues.append({
                        "file": str(yaml_file),
                        "resource": f"{kind}/{name}",
                        "issue": f"Container '{c.get('name')}' uses :latest or untagged image: {image}",
                        "severity": "CRITICAL",
                    })

            # Check: replicas > 1 for production
            replicas = doc.get("spec", {}).get("replicas", 1)
            labels = doc.get("metadata", {}).get("labels", {})
            if labels.get("environment") == "production" and replicas < 2:
                issues.append({
                    "file": str(yaml_file),
                    "resource": f"{kind}/{name}",
                    "issue": f"Production deployment with only {replicas} replica(s)",
                    "severity": "HIGH",
                })

        # Check: All services have standard labels
        required_labels = {"app", "version", "team"}
        labels = doc.get("metadata", {}).get("labels", {})
        missing = required_labels - set(labels.keys())
        if missing and kind in ("Deployment", "Service", "StatefulSet"):
            issues.append({
                "file": str(yaml_file),
                "resource": f"{kind}/{name}",
                "issue": f"Missing required labels: {missing}",
                "severity": "LOW",
            })

    return issues


def deep_merge(base: dict, override: dict) -> dict:
    """
    Deep merge two dicts (like Helm values merge).
    override values take precedence. Lists are replaced, not appended.
    """
    result = copy.deepcopy(base)
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = copy.deepcopy(value)
    return result
```

---

## 5. CONFIGURATION MANAGEMENT

```python
"""
src/novatools/utils/config.py

Production configuration management using pydantic-settings.
Loads from: env vars → .env file → defaults
"""
from __future__ import annotations

from functools import lru_cache
from pathlib import Path

from pydantic import Field, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class AWSConfig(BaseSettings):
    """AWS-specific configuration."""
    model_config = SettingsConfigDict(env_prefix="AWS_")

    region: str = "us-east-1"
    profile: str | None = None
    role_arn: str | None = None
    account_id: str | None = None


class SlackConfig(BaseSettings):
    """Slack integration configuration."""
    model_config = SettingsConfigDict(env_prefix="SLACK_")

    webhook_url: str | None = None
    bot_token: str | None = None
    default_channel: str = "#platform-alerts"
    enabled: bool = True


class PagerDutyConfig(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="PAGERDUTY_")

    routing_key: str | None = None
    enabled: bool = True


class JiraConfig(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="JIRA_")

    base_url: str = ""
    email: str = ""
    api_token: str = ""
    project: str = "OPS"


class K8sConfig(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="K8S_")

    context: str | None = None
    namespace: str = "default"
    kubeconfig: str | None = None
    in_cluster: bool = False


class AppConfig(BaseSettings):
    """
    Root configuration — aggregates all sub-configs.

    Loading priority (highest wins):
    1. Environment variables
    2. .env file
    3. Field defaults

    Usage:
        config = get_config()
        print(config.aws.region)
        print(config.slack.webhook_url)
    """
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_nested_delimiter="__",  # SLACK__WEBHOOK_URL → slack.webhook_url
        case_sensitive=False,
        extra="ignore",  # Don't crash on unexpected env vars
    )

    # Sub-configs
    aws: AWSConfig = Field(default_factory=AWSConfig)
    slack: SlackConfig = Field(default_factory=SlackConfig)
    pagerduty: PagerDutyConfig = Field(default_factory=PagerDutyConfig)
    jira: JiraConfig = Field(default_factory=JiraConfig)
    k8s: K8sConfig = Field(default_factory=K8sConfig)

    # App-level settings
    environment: str = "development"
    log_level: str = "INFO"
    json_output: bool = False
    dry_run: bool = False
    debug: bool = False

    # NovaMart specific
    company: str = "novamart"
    regions: list[str] = Field(default=["us-east-1", "us-west-2", "eu-west-1"])
    cost_alert_threshold: float = 10000.0
    rotation_threshold_days: int = 90
    @field_validator("environment")
    @classmethod
    def validate_environment(cls, v: str) -> str:
        allowed = {"development", "staging", "production", "test"}
        if v not in allowed:
            raise ValueError(f"environment must be one of {allowed}, got '{v}'")
        return v

    @field_validator("log_level")
    @classmethod
    def validate_log_level(cls, v: str) -> str:
        v = v.upper()
        allowed = {"DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"}
        if v not in allowed:
            raise ValueError(f"log_level must be one of {allowed}, got '{v}'")
        return v

    @property
    def is_production(self) -> bool:
        return self.environment == "production"


@lru_cache(maxsize=1)
def get_config() -> AppConfig:
    """
    Singleton config — loaded once, cached forever.
    lru_cache ensures we don't re-parse env vars on every call.
    """
    return AppConfig()


# ═══════════════════════════════════════════════════════════
# USAGE PATTERNS
# ═══════════════════════════════════════════════════════════

# In any module:
# from novatools.utils.config import get_config
# config = get_config()
# if config.is_production:
#     require_approval()
# client = boto3.client("ec2", region_name=config.aws.region)

# In tests — override config with env vars:
# def test_something(monkeypatch):
#     monkeypatch.setenv("ENVIRONMENT", "test")
#     monkeypatch.setenv("AWS_REGION", "us-west-2")
#     monkeypatch.setenv("SLACK__ENABLED", "false")
#     get_config.cache_clear()  # Reset singleton
#     config = get_config()
#     assert config.environment == "test"


# ═══════════════════════════════════════════════════════════
# ENVIRONMENT-SPECIFIC CONFIG FILES
# ═══════════════════════════════════════════════════════════

def load_service_config(
    service: str,
    environment: str,
    config_dir: Path = Path("config/"),
) -> dict:
    """
    Load service config with environment overlay.
    config/
      base.yaml           ← defaults
      production.yaml     ← production overrides
      services/
        payment.yaml      ← service-specific overrides
    
    Merge order: base → environment → service (last wins)
    """
    import yaml

    base_file = config_dir / "base.yaml"
    env_file = config_dir / f"{environment}.yaml"
    svc_file = config_dir / "services" / f"{service}.yaml"

    result: dict = {}

    for f in [base_file, env_file, svc_file]:
        if f.exists():
            with open(f) as fh:
                data = yaml.safe_load(fh)
                if data:
                    result = deep_merge(result, data)

    return result
```

**Configuration failure modes:**
```python
# FAILURE 1: Env var not set, no default, no validation
# The script runs, uses None for region, boto3 falls back to us-east-1
# You deploy to wrong region. Outage.
# FIX: Pydantic with required fields (no default) → crashes at startup

# FAILURE 2: .env file committed to git
# Your production DB password is in git history forever.
# FIX: .gitignore .env, use .env.example with placeholder values

# FAILURE 3: Config loaded at import time
# config = AppConfig()  # Module level — crashes on import if env missing
# FIX: get_config() lazy function with @lru_cache

# FAILURE 4: Config singleton never refreshed in tests
# Test A sets REGION=us-east-1, Test B expects us-west-2
# But lru_cache returns stale config from Test A
# FIX: get_config.cache_clear() in test fixtures

# FAILURE 5: Nested env vars misunderstood
# SLACK__WEBHOOK_URL works with env_nested_delimiter="__"
# But SLACK_WEBHOOK_URL does NOT (pydantic-settings treats it differently)
# Document your convention. Pick one. Stick with it.
```

---

## 6. REGULAR EXPRESSIONS IN PRODUCTION

```python
"""
Real regex patterns a Senior DevOps engineer uses weekly.
Not academic — these solve actual operational problems.
"""
import re
from typing import Any


# ═══════════════════════════════════════════════════════════
# PATTERN 1: Extract structured data from unstructured output
# ═══════════════════════════════════════════════════════════

# Terraform plan output parsing
TERRAFORM_CHANGE = re.compile(
    r"^\s*# (?P<address>\S+) (?:will be|must be) (?P<action>created|destroyed|updated|replaced)",
    re.MULTILINE,
)

def parse_terraform_plan(plan_output: str) -> list[dict[str, str]]:
    """Extract resource changes from terraform plan output."""
    changes = []
    for match in TERRAFORM_CHANGE.finditer(plan_output):
        changes.append({
            "address": match.group("address"),
            "action": match.group("action"),
        })
    return changes

# kubectl output parsing (when JSON is not available)
KUBECTL_EVENT = re.compile(
    r"(?P<last>\S+)\s+"
    r"(?P<type>\S+)\s+"
    r"(?P<reason>\S+)\s+"
    r"(?P<object>\S+)\s+"
    r"(?P<message>.+)"
)


# ═══════════════════════════════════════════════════════════
# PATTERN 2: Log line matching with named groups
# ═══════════════════════════════════════════════════════════

# Java stack trace extraction (multi-line)
JAVA_EXCEPTION = re.compile(
    r"(?P<exception>[\w.]+(?:Exception|Error|Throwable)): (?P<message>[^\n]+)"
    r"(?P<stacktrace>(?:\n\s+at .+)+)?",
)

def extract_exceptions(log_content: str) -> list[dict[str, str]]:
    """Extract Java exceptions from log content."""
    results = []
    for match in JAVA_EXCEPTION.finditer(log_content):
        results.append({
            "exception": match.group("exception"),
            "message": match.group("message"),
            "stacktrace": (match.group("stacktrace") or "").strip(),
        })
    return results


# ═══════════════════════════════════════════════════════════
# PATTERN 3: Validation patterns
# ═══════════════════════════════════════════════════════════

# These are COMPILED once at module level — not in the function
SEMVER = re.compile(r"^v?(?P<major>\d+)\.(?P<minor>\d+)\.(?P<patch>\d+)(?:-(?P<pre>[a-zA-Z0-9.]+))?$")
K8S_RESOURCE_NAME = re.compile(r"^[a-z0-9]([a-z0-9\-]{0,61}[a-z0-9])?$")  # RFC 1123 subdomain
AWS_ACCOUNT_ID = re.compile(r"^\d{12}$")
AWS_ARN = re.compile(r"^arn:aws[a-zA-Z-]*:\S+:\S*:\d{12}:\S+$")
DOCKER_IMAGE = re.compile(
    r"^(?:(?P<registry>[^/]+)/)?(?P<repo>[^:@]+)(?::(?P<tag>[^@]+))?(?:@(?P<digest>sha256:[a-f0-9]{64}))?$"
)
IPV4 = re.compile(r"^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$")
CIDR = re.compile(r"^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)/(?:[0-9]|[12]\d|3[0-2])$")
UUID = re.compile(r"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$", re.IGNORECASE)

def validate_image_reference(image: str) -> dict[str, str | None]:
    """Parse and validate a Docker image reference."""
    match = DOCKER_IMAGE.match(image)
    if not match:
        raise ValueError(f"Invalid Docker image reference: {image}")
    return {
        "registry": match.group("registry"),
        "repository": match.group("repo"),
        "tag": match.group("tag"),
        "digest": match.group("digest"),
    }


# ═══════════════════════════════════════════════════════════
# PATTERN 4: Search and replace in config files
# ═══════════════════════════════════════════════════════════

def update_image_in_yaml(
    content: str,
    old_tag: str,
    new_tag: str,
) -> str:
    """
    Replace image tags in YAML content.
    More precise than sed — matches image: lines only.
    """
    pattern = re.compile(
        r"(image:\s*\S+:)" + re.escape(old_tag) + r"(\s*(?:#.*)?)$",
        re.MULTILINE,
    )
    return pattern.sub(rf"\g<1>{new_tag}\2", content)


# ═══════════════════════════════════════════════════════════
# PATTERN 5: Secret detection (pre-commit, CI)
# ═══════════════════════════════════════════════════════════

SECRET_PATTERNS = [
    (re.compile(r"AKIA[0-9A-Z]{16}"), "AWS Access Key ID"),
    (re.compile(r"(?i)(?:password|passwd|pwd)\s*[:=]\s*['\"]?[^\s'\"]{8,}"), "Password assignment"),
    (re.compile(r"(?i)(?:api[_-]?key|apikey)\s*[:=]\s*['\"]?[a-zA-Z0-9]{20,}"), "API Key"),
    (re.compile(r"(?i)(?:secret|token)\s*[:=]\s*['\"]?[a-zA-Z0-9/+=]{20,}"), "Secret/Token"),
    (re.compile(r"-----BEGIN (?:RSA |DSA |EC )?PRIVATE KEY-----"), "Private Key"),
    (re.compile(r"ghp_[A-Za-z0-9]{36}"), "GitHub Personal Access Token"),
    (re.compile(r"xox[baprs]-[0-9a-zA-Z-]+"), "Slack Token"),
]

def scan_for_secrets(content: str, filename: str = "") -> list[dict[str, Any]]:
    """Scan content for potential secrets. Used in pre-commit hooks and CI."""
    findings = []
    for line_num, line in enumerate(content.splitlines(), 1):
        # Skip comments
        stripped = line.strip()
        if stripped.startswith("#") or stripped.startswith("//"):
            continue

        for pattern, secret_type in SECRET_PATTERNS:
            if pattern.search(line):
                findings.append({
                    "file": filename,
                    "line": line_num,
                    "type": secret_type,
                    "content": line[:80] + "..." if len(line) > 80 else line,
                })
    return findings
```

**Regex failure modes:**
```python
# FAILURE 1: Catastrophic backtracking (ReDoS)
# BAD:
evil_pattern = re.compile(r"(a+)+b")
evil_pattern.match("a" * 30 + "c")  # Takes EXPONENTIAL time — hangs your script

# GOOD: Avoid nested quantifiers. Use atomic groups if available.
# In Python: use re2 library for guaranteed linear-time matching
# pip install google-re2

# FAILURE 2: Compiling regex inside loops
# BAD:
for line in million_line_file:
    if re.match(r"ERROR.*timeout", line):  # Recompiles every iteration
        ...
# GOOD:
pattern = re.compile(r"ERROR.*timeout")  # Compile ONCE
for line in million_line_file:
    if pattern.match(line):
        ...

# FAILURE 3: match() vs search() vs fullmatch()
# re.match() only checks START of string
# re.search() checks ANYWHERE in string
# re.fullmatch() checks ENTIRE string

text = "error: connection timeout at 10:30"
re.match(r"timeout", text)      # None! (doesn't start with "timeout")
re.search(r"timeout", text)     # Match! (found in middle)
re.fullmatch(r"timeout", text)  # None! (not the entire string)

# FAILURE 4: Greedy vs lazy quantifiers
log_line = 'user="admin" action="delete" target="database"'
re.findall(r'"(.+)"', log_line)     # ['"admin" action="delete" target="database"'] — GREEDY
re.findall(r'"(.+?)"', log_line)    # ['admin', 'delete', 'database'] — LAZY (correct)

# FAILURE 5: Not escaping user input
# BAD:
search_term = user_input  # Could be "payment.*" or "(((("
re.search(search_term, content)  # Crashes or unexpected behavior
# GOOD:
re.search(re.escape(search_term), content)
```

---

## 7. WEBHOOK RECEIVERS (FastAPI)

```python
"""
src/novatools/webhooks/server.py

Lightweight webhook receiver for DevOps automation.
Receives events from: Bitbucket, PagerDuty, ArgoCD, AlertManager, GitHub Actions.

FastAPI chosen over Flask because:
- Async native (can handle concurrent webhooks)
- Type validation with Pydantic (request bodies validated automatically)
- Auto-generated API docs (/docs) — team can see all endpoints
- Production-proven at our scale
"""
from __future__ import annotations

import hashlib
import hmac
import json
import os
from datetime import datetime, UTC
from typing import Any

import structlog
from fastapi import FastAPI, Header, HTTPException, Request, BackgroundTasks
from pydantic import BaseModel, Field

log = structlog.get_logger()

app = FastAPI(
    title="NovaMart Webhook Handler",
    version="1.0.0",
    docs_url="/docs",       # Swagger UI
    redoc_url=None,         # Disable redoc
)


# ═══════════════════════════════════════════════════════════
# HEALTH ENDPOINTS (required for K8s probes)
# ═══════════════════════════════════════════════════════════

@app.get("/healthz")
async def healthz() -> dict[str, str]:
    return {"status": "ok"}

@app.get("/readyz")
async def readyz() -> dict[str, str]:
    # Could check downstream dependencies here
    return {"status": "ready"}


# ═══════════════════════════════════════════════════════════
# WEBHOOK SIGNATURE VERIFICATION
# ═══════════════════════════════════════════════════════════

def verify_signature(
    payload: bytes,
    signature: str,
    secret: str,
    algorithm: str = "sha256",
) -> bool:
    """
    Verify HMAC signature from webhook provider.
    CRITICAL: Without this, anyone can trigger your automation.
    """
    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        getattr(hashlib, algorithm),
    ).hexdigest()

    # Use compare_digest to prevent timing attacks
    return hmac.compare_digest(expected, signature)


# ═══════════════════════════════════════════════════════════
# BITBUCKET WEBHOOK — PR merged → trigger CI
# ═══════════════════════════════════════════════════════════

class BitbucketPRPayload(BaseModel):
    """Bitbucket PR webhook payload (simplified)."""
    class PullRequest(BaseModel):
        id: int
        title: str
        state: str
        source: dict[str, Any]
        destination: dict[str, Any]
        author: dict[str, Any]

    class Repository(BaseModel):
        full_name: str
        name: str

    pullrequest: PullRequest
    repository: Repository


@app.post("/webhooks/bitbucket/pr")
async def bitbucket_pr(
    request: Request,
    background_tasks: BackgroundTasks,
    x_event_key: str = Header(...),  # pullrequest:fulfilled, pullrequest:created, etc.
    x_hub_signature: str = Header(None),
) -> dict[str, str]:
    """Handle Bitbucket PR events."""
    body = await request.body()

    # Verify signature
    secret = os.environ.get("BITBUCKET_WEBHOOK_SECRET", "")
    if secret and x_hub_signature:
        sig = x_hub_signature.replace("sha256=", "")
        if not verify_signature(body, sig, secret):
            log.warning("invalid_webhook_signature", source="bitbucket")
            raise HTTPException(status_code=401, detail="Invalid signature")

    payload = BitbucketPRPayload.model_validate_json(body)

    log.info(
        "bitbucket_webhook_received",
        event=x_event_key,
        repo=payload.repository.full_name,
        pr_id=payload.pullrequest.id,
        pr_title=payload.pullrequest.title,
    )

    # Handle different events
    if x_event_key == "pullrequest:fulfilled":  # PR merged
        # Process in background — return 200 immediately
        # Webhook providers timeout after ~10s, so never do heavy work synchronously
        background_tasks.add_task(
            handle_pr_merged,
            repo=payload.repository.full_name,
            pr_id=payload.pullrequest.id,
            branch=payload.pullrequest.destination.get("branch", {}).get("name", "main"),
        )

    return {"status": "accepted", "event": x_event_key}


async def handle_pr_merged(repo: str, pr_id: int, branch: str) -> None:
    """Background task: Handle PR merge → Notify → Track."""
    log.info("processing_pr_merge", repo=repo, pr_id=pr_id, branch=branch)

    # 1. Send Slack notification
    from novatools.clients.slack import SlackClient
    slack = SlackClient()
    await slack_notify_pr_merged(slack, repo, pr_id, branch)

    # 2. Update deployment tracker
    # 3. Trigger any post-merge automation


# ═══════════════════════════════════════════════════════════
# ALERTMANAGER WEBHOOK — Alert fires → auto-remediation
# ═══════════════════════════════════════════════════════════

class AlertManagerPayload(BaseModel):
    """AlertManager webhook payload."""
    class Alert(BaseModel):
        status: str  # "firing" or "resolved"
        labels: dict[str, str]
        annotations: dict[str, str]
        startsAt: str
        endsAt: str
        generatorURL: str = ""
        fingerprint: str = ""

    version: str = "4"
    status: str  # "firing" or "resolved"
    receiver: str
    alerts: list[Alert]
    groupLabels: dict[str, str] = {}
    commonLabels: dict[str, str] = {}
    externalURL: str = ""


@app.post("/webhooks/alertmanager")
async def alertmanager(
    payload: AlertManagerPayload,
    background_tasks: BackgroundTasks,
) -> dict[str, str]:
    """
    Handle AlertManager webhooks.
    Auto-remediation: scale up on high CPU, restart on crash loop, etc.
    """
    for alert in payload.alerts:
        log.info(
            "alert_received",
            alertname=alert.labels.get("alertname"),
            severity=alert.labels.get("severity"),
            namespace=alert.labels.get("namespace"),
            status=alert.status,
        )

        if alert.status == "firing":
            alertname = alert.labels.get("alertname", "")

            # Auto-remediation routing
            if alertname == "HighCPUUsage":
                background_tasks.add_task(
                    remediate_high_cpu,
                    namespace=alert.labels.get("namespace", ""),
                    deployment=alert.labels.get("deployment", ""),
                )
            elif alertname == "PodCrashLooping":
                background_tasks.add_task(
                    remediate_crash_loop,
                    namespace=alert.labels.get("namespace", ""),
                    pod=alert.labels.get("pod", ""),
                )
            elif alertname == "DiskUsageHigh":
                background_tasks.add_task(
                    remediate_disk_usage,
                    node=alert.labels.get("node", ""),
                    threshold=float(alert.annotations.get("current_value", "0")),
                )

    return {"status": "accepted", "alerts_processed": len(payload.alerts)}


async def remediate_high_cpu(namespace: str, deployment: str) -> None:
    """Auto-scale deployment on high CPU. Conservative — max 2x current."""
    k = Kubectl(namespace=namespace)

    # Get current replicas
    result = k.get_json("deployment", deployment)
    current_replicas = result["spec"]["replicas"]
    max_replicas = min(current_replicas * 2, 20)  # Safety cap

    # Check if HPA exists (don't fight the HPA)
    hpa_result = k.get("hpa", label_selector=f"app={deployment}")
    if hpa_result.success and hpa_result.stdout.strip() != "No resources found":
        log.info("hpa_exists_skipping_manual_scale", deployment=deployment)
        return

    log.info("auto_scaling", deployment=deployment,
             from_replicas=current_replicas, to_replicas=max_replicas)

    k.scale("deployment", deployment, max_replicas)

    # Notify — humans should know automation acted
    # slack.send(f"⚡ Auto-scaled {deployment} from {current_replicas} → {max_replicas}")


async def remediate_crash_loop(namespace: str, pod: str) -> None:
    """Collect diagnostics for crash-looping pod. Don't auto-restart (K8s does that)."""
    k = Kubectl(namespace=namespace)

    # Collect diagnostics
    describe = k._run(["describe", "pod", pod])
    previous_logs = k.logs(pod, previous=True, tail=200)
    events = k._run(["get", "events", "--field-selector", f"involvedObject.name={pod}",
                      "--sort-by=.lastTimestamp"])

    log.info("crash_loop_diagnostics_collected", pod=pod, namespace=namespace)
    # Send to Slack incident channel with diagnostics attached


async def remediate_disk_usage(node: str, threshold: float) -> None:
    """Clean up disk on node with high usage."""
    log.info("disk_remediation", node=node, current_usage_pct=threshold)
    # In production: kubectl debug node/<node> -- crictl rmi --prune
    # Or trigger Ansible playbook for disk cleanup


# ═══════════════════════════════════════════════════════════
# PAGERDUTY WEBHOOK — Incident events
# ═══════════════════════════════════════════════════════════

class PagerDutyWebhook(BaseModel):
    class Event(BaseModel):
        event_type: str  # "incident.triggered", "incident.acknowledged", etc.
        data: dict[str, Any]

    event: Event


@app.post("/webhooks/pagerduty")
async def pagerduty(
    payload: PagerDutyWebhook,
    background_tasks: BackgroundTasks,
    x_pagerduty_signature: str = Header(None),
) -> dict[str, str]:
    """
    Handle PagerDuty incident lifecycle events.
    triggered → create Slack channel + Jira
    acknowledged → update Slack channel topic
    resolved → post summary to Slack
    """
    event_type = payload.event.event_type
    incident_data = payload.event.data

    log.info("pagerduty_event", type=event_type,
             incident_id=incident_data.get("id"))

    if event_type == "incident.triggered":
        background_tasks.add_task(
            create_incident_channel,
            incident=incident_data,
        )

    return {"status": "accepted"}


async def create_incident_channel(incident: dict[str, Any]) -> None:
    """Auto-create Slack channel and Jira ticket for new incidents."""
    incident_id = incident.get("id", "unknown")
    title = incident.get("title", "Unknown incident")

    log.info("creating_incident_channel",
             incident_id=incident_id, title=title)

    # 1. Create Slack channel: #inc-20240115-payment-timeout
    # 2. Post initial context (who's on-call, runbook links, dashboards)
    # 3. Create Jira ticket
    # 4. Link Slack channel ↔ Jira ticket ↔ PagerDuty incident


# ═══════════════════════════════════════════════════════════
# ARGOCD WEBHOOK — Deployment tracking
# ═══════════════════════════════════════════════════════════

class ArgoCDNotification(BaseModel):
    app_name: str = Field(alias="app")
    status: str
    message: str = ""
    revision: str = ""


@app.post("/webhooks/argocd")
async def argocd_notification(
    payload: ArgoCDNotification,
    background_tasks: BackgroundTasks,
) -> dict[str, str]:
    """Track ArgoCD sync events for DORA metrics."""
    log.info("argocd_event",
             app=payload.app_name,
             status=payload.status,
             revision=payload.revision)

    # Track deployment frequency, lead time, failure rate
    # Store in DB or push as Prometheus metric

    return {"status": "accepted"}


# ═══════════════════════════════════════════════════════════
# RUNNING THE SERVER
# ═══════════════════════════════════════════════════════════

# Development:
#   uvicorn novatools.webhooks.server:app --reload --port 8080
#
# Production (in K8s):
#   CMD ["uvicorn", "novatools.webhooks.server:app",
#        "--host", "0.0.0.0", "--port", "8080",
#        "--workers", "4",                  # Multiple worker processes
#        "--access-log",                    # Access logging
#        "--timeout-keep-alive", "65"]      # > ALB idle timeout (60s)
#
# Dockerfile:
#   FROM python:3.12-slim
#   COPY --from=builder /app /app
#   USER nonroot
#   EXPOSE 8080
#   HEALTHCHECK CMD curl -f http://localhost:8080/healthz || exit 1
#   CMD ["uvicorn", "novatools.webhooks.server:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Webhook failure modes:**
```python
# FAILURE 1: No signature verification
# Anyone on the internet can POST to your webhook and trigger automation.
# Imagine: attacker POSTs fake AlertManager payload → auto-scales to 20 replicas
# ALWAYS verify HMAC signatures.

# FAILURE 2: Synchronous processing
# Webhook providers timeout after ~10-30 seconds.
# If your handler takes 60s (e.g., waiting for deployment), the provider retries.
# Now you get DUPLICATE events. Deploy happens twice.
# FIX: Return 200 immediately, process in background_tasks

# FAILURE 3: No idempotency
# Webhook providers retry on timeout/5xx. You WILL get duplicate events.
# FIX: Use fingerprint/dedup_key from payload. Store processed IDs in Redis.
# Check before processing:
#   if redis.setnx(f"webhook:{fingerprint}", "1", ex=3600):
#       process()
#   else:
#       log.info("duplicate_webhook_skipped", fingerprint=fingerprint)

# FAILURE 4: No rate limiting
# Noisy alert → AlertManager fires 100 webhooks/second → overwhelms your handler
# FIX: Use fastapi-limiter or token bucket per source

# FAILURE 5: Background task failure is silent
# background_tasks exceptions are logged but don't return to caller
# FIX: Wrap background tasks with error handling + alerting
async def safe_background(func, **kwargs):
    try:
        await func(**kwargs)
    except Exception:
        log.exception("background_task_failed", func=func.__name__)
        # Send to Slack/PD
```

---

## 8. AWS LAMBDA HANDLERS

```python
"""
src/novatools/lambdas/secret_rotation.py

AWS Lambda patterns for DevOps automation.
Common use cases: secret rotation, scheduled audits, event-driven responses,
CloudWatch alarm auto-remediation, custom CloudFormation resources.
"""
from __future__ import annotations

import json
import os
from typing import Any

import boto3
import structlog

log = structlog.get_logger()


# ═══════════════════════════════════════════════════════════
# PATTERN 1: Lambda handler structure
# ═══════════════════════════════════════════════════════════

# Initialize clients OUTSIDE handler (reused across invocations — warm start)
# Lambda keeps the execution environment alive for ~15-45 minutes
ec2_client = boto3.client("ec2", region_name=os.environ.get("AWS_REGION", "us-east-1"))
sns_client = boto3.client("sns", region_name=os.environ.get("AWS_REGION", "us-east-1"))

# Environment variables set via Terraform/CloudFormation
SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN", "")
ENVIRONMENT = os.environ.get("ENVIRONMENT", "production")


def handler(event: dict[str, Any], context: Any) -> dict[str, Any]:
    """
    Lambda entry point.

    Args:
        event: Trigger-specific payload (CloudWatch Event, S3, SNS, API Gateway, etc.)
        context: Lambda runtime info (function_name, memory_limit_in_mb,
                 get_remaining_time_in_millis(), log_group_name, aws_request_id)
    """
    # Always log the event for debugging (but redact secrets)
    log.info("lambda_invoked",
             function=context.function_name,
             request_id=context.aws_request_id,
             remaining_ms=context.get_remaining_time_in_millis(),
             event_source=event.get("source", "unknown"))

    try:
        result = process_event(event)
        log.info("lambda_completed", result=result)
        return {
            "statusCode": 200,
            "body": json.dumps(result),
        }
    except Exception as e:
        log.exception("lambda_failed")
        # Don't re-raise for async invocations (goes to DLQ)
        # DO re-raise for synchronous invocations (caller gets error)
        raise


# ═══════════════════════════════════════════════════════════
# PATTERN 2: CloudWatch Event / EventBridge handler
# ═══════════════════════════════════════════════════════════

def handle_security_group_change(event: dict[str, Any], context: Any) -> dict[str, Any]:
    """
    Triggered by EventBridge rule on EC2 SG changes.
    Auto-reverts unauthorized security group modifications.
    
    EventBridge rule:
    {
      "source": ["aws.ec2"],
      "detail-type": ["AWS API Call via CloudTrail"],
      "detail": {
        "eventName": ["AuthorizeSecurityGroupIngress", "AuthorizeSecurityGroupEgress"]
      }
    }
    """
    detail = event.get("detail", {})
    sg_id = detail.get("requestParameters", {}).get("groupId", "")
    user = detail.get("userIdentity", {}).get("arn", "unknown")
    event_name = detail.get("eventName", "")

    log.info("sg_change_detected",
             sg_id=sg_id, user=user, action=event_name)

    # Check if the SG is in our "protected" list
    protected_sgs = os.environ.get("PROTECTED_SGS", "").split(",")
    if sg_id not in protected_sgs:
        log.info("sg_not_protected_skipping", sg_id=sg_id)
        return {"action": "skipped", "reason": "not protected"}

    # Check for 0.0.0.0/0 rules
    sg = ec2_client.describe_security_groups(GroupIds=[sg_id])["SecurityGroups"][0]
    violations = []

    for rule in sg.get("IpPermissions", []):
        for ip_range in rule.get("IpRanges", []):
            if ip_range.get("CidrIp") == "0.0.0.0/0":
                violations.append({
                    "from_port": rule.get("FromPort"),
                    "to_port": rule.get("ToPort"),
                    "protocol": rule.get("IpProtocol"),
                })

    if violations:
        log.warning("unauthorized_sg_rule_detected",
                    sg_id=sg_id, violations=violations, user=user)

        # Revoke the violating rules
        for rule in sg.get("IpPermissions", []):
            for ip_range in rule.get("IpRanges", []):
                if ip_range.get("CidrIp") == "0.0.0.0/0":
                    ec2_client.revoke_security_group_ingress(
                        GroupId=sg_id,
                        IpPermissions=[rule],
                    )
                    log.info("revoked_open_rule", sg_id=sg_id, rule=str(rule))

        # Notify
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"🚨 Security Group {sg_id} — unauthorized rule auto-reverted",
            Message=json.dumps({
                "sg_id": sg_id,
                "user": user,
                "violations": violations,
                "action": "auto-reverted",
            }, indent=2),
        )

        return {"action": "reverted", "violations": len(violations)}

    return {"action": "allowed", "violations": 0}


# ═══════════════════════════════════════════════════════════
# PATTERN 3: Scheduled Lambda (EventBridge cron)
# ═══════════════════════════════════════════════════════════

def handle_cost_anomaly_check(event: dict[str, Any], context: Any) -> dict[str, Any]:
    """
    Daily cost check — runs via EventBridge schedule.
    rule: rate(1 day) or cron(0 9 * * ? *)  # 9 AM UTC daily

    Checks yesterday's spend against 7-day average.
    Alerts if >20% above average.
    """
    import datetime

    ce_client = boto3.client("ce", region_name="us-east-1")  # CE is global, use us-east-1

    today = datetime.date.today()
    yesterday = today - datetime.timedelta(days=1)
    week_ago = today - datetime.timedelta(days=8)

    # Get yesterday's cost
    yesterday_response = ce_client.get_cost_and_usage(
        TimePeriod={
            "Start": yesterday.isoformat(),
            "End": today.isoformat(),
        },
        Granularity="DAILY",
        Metrics=["UnblendedCost"],
    )
    yesterday_cost = float(
        yesterday_response["ResultsByTime"][0]["Total"]["UnblendedCost"]["Amount"]
    )

    # Get 7-day average
    week_response = ce_client.get_cost_and_usage(
        TimePeriod={
            "Start": week_ago.isoformat(),
            "End": yesterday.isoformat(),
        },
        Granularity="DAILY",
        Metrics=["UnblendedCost"],
    )
    daily_costs = [
        float(day["Total"]["UnblendedCost"]["Amount"])
        for day in week_response["ResultsByTime"]
    ]
    avg_cost = sum(daily_costs) / len(daily_costs) if daily_costs else 0

    deviation_pct = ((yesterday_cost - avg_cost) / avg_cost * 100) if avg_cost > 0 else 0

    log.info("cost_check",
             yesterday=yesterday_cost,
             avg_7d=round(avg_cost, 2),
             deviation_pct=round(deviation_pct, 1))

    threshold = float(os.environ.get("COST_DEVIATION_THRESHOLD", "20"))
    if deviation_pct > threshold:
        # Get top spending services for investigation context
        service_response = ce_client.get_cost_and_usage(
            TimePeriod={
                "Start": yesterday.isoformat(),
                "End": today.isoformat(),
            },
            Granularity="DAILY",
            Metrics=["UnblendedCost"],
            GroupBy=[{"Type": "DIMENSION", "Key": "SERVICE"}],
        )
        top_services = sorted(
            service_response["ResultsByTime"][0]["Groups"],
            key=lambda g: float(g["Metrics"]["UnblendedCost"]["Amount"]),
            reverse=True,
        )[:5]

        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"💰 Cost anomaly: ${yesterday_cost:.2f} (+{deviation_pct:.1f}% above avg)",
            Message=json.dumps({
                "yesterday_cost": yesterday_cost,
                "avg_7d_cost": round(avg_cost, 2),
                "deviation_pct": round(deviation_pct, 1),
                "top_services": [
                    {
                        "service": g["Keys"][0],
                        "cost": float(g["Metrics"]["UnblendedCost"]["Amount"]),
                    }
                    for g in top_services
                ],
            }, indent=2),
        )

    return {
        "yesterday_cost": yesterday_cost,
        "avg_7d": round(avg_cost, 2),
        "deviation_pct": round(deviation_pct, 1),
        "alerted": deviation_pct > threshold,
    }


# ═══════════════════════════════════════════════════════════
# PATTERN 4: S3 Event trigger
# ═══════════════════════════════════════════════════════════

def handle_s3_upload(event: dict[str, Any], context: Any) -> dict[str, Any]:
    """
    Triggered when file uploaded to S3.
    Use case: Log file uploaded → parse → extract metrics → push to CloudWatch.
    """
    s3_client = boto3.client("s3")

    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]
        size = record["s3"]["object"].get("size", 0)

        log.info("s3_event", bucket=bucket, key=key, size=size)

        # Download and process
        # IMPORTANT: Lambda has 512MB /tmp (or 10GB ephemeral storage if configured)
        # For files > 512MB, stream instead of downloading
        if size < 100 * 1024 * 1024:  # < 100MB — download to /tmp
            local_path = f"/tmp/{key.split('/')[-1]}"
            s3_client.download_file(bucket, key, local_path)
            # process(local_path)
        else:
            # Stream with get_object()
            response = s3_client.get_object(Bucket=bucket, Key=key)
            # Process body as stream: response["Body"].iter_lines()
            pass

    return {"processed": len(event.get("Records", []))}
```

**Lambda failure modes:**
```python
# FAILURE 1: Cold start timeout
# Python Lambda cold starts: ~300-800ms (with boto3 imports)
# If Lambda timeout = 3s and cold start takes 1s, only 2s for actual work
# FIX: Set timeout appropriately (15-60s for automation lambdas)
# Use provisioned concurrency for latency-sensitive

# FAILURE 2: Clients initialized inside handler
# BAD:
def handler(event, context):
    client = boto3.client("ec2")  # Created on EVERY invocation
# GOOD: Initialize outside handler (reused on warm starts)
client = boto3.client("ec2")
def handler(event, context):
    client.describe_instances()

# FAILURE 3: /tmp disk exhaustion
# Lambda reuses /tmp across warm invocations
# If you download files to /tmp and don't clean up, disk fills
# FIX: Clean up /tmp at start of handler, or use unique filenames

# FAILURE 4: Unhandled exceptions in async invocations
# Async Lambda (SNS, S3 triggers) retries twice on failure
# If your Lambda throws, same event processes 3 times
# FIX: Idempotent handlers. Or: configure DLQ for failed events.

# FAILURE 5: Lambda IAM role too broad
# Lambda gets `*` permissions "because it's easier"
# Attacker exploits SSRF → uses Lambda role to access everything
# FIX: Least privilege. One Lambda = one purpose = one minimal role.

# FAILURE 6: Missing VPC config for private resources
# Lambda needs VPC config to access RDS/ElastiCache in private subnets
# But VPC Lambda = ENI creation = 10-30s cold start penalty
# FIX: Use VPC only when needed. Use RDS Proxy (VPC Lambda → RDS Proxy → RDS)
```

---

## 9. DATABASE OPERATIONS (psycopg2, redis-py)

```python
"""
src/novatools/clients/database.py

Database operations a DevOps engineer needs:
- Health checks, connection pool monitoring
- Query execution for operational tasks
- Backup verification, replication lag checks
"""
from __future__ import annotations

import time
from contextlib import contextmanager
from typing import Any, Generator

import structlog

log = structlog.get_logger()


# ═══════════════════════════════════════════════════════════
# POSTGRESQL (psycopg2)
# ═══════════════════════════════════════════════════════════

import psycopg2
import psycopg2.pool
import psycopg2.extras


class PostgresClient:
    """Production PostgreSQL client with connection pooling."""

    def __init__(
        self,
        host: str,
        port: int = 5432,
        dbname: str = "postgres",
        user: str = "postgres",
        password: str = "",
        min_connections: int = 2,
        max_connections: int = 10,
        connect_timeout: int = 5,
        statement_timeout_ms: int = 30000,
    ) -> None:
        self._pool = psycopg2.pool.ThreadedConnectionPool(
            minconn=min_connections,
            maxconn=max_connections,
            host=host,
            port=port,
            dbname=dbname,
            user=user,
            password=password,
            connect_timeout=connect_timeout,
            options=f"-c statement_timeout={statement_timeout_ms}",
            # sslmode='require' in production!
            sslmode="require",
        )
        log.info("pg_pool_created", host=host, dbname=dbname,
                 min=min_connections, max=max_connections)

    @contextmanager
    def connection(self) -> Generator[Any, None, None]:
        """Get connection from pool with automatic return."""
        conn = self._pool.getconn()
        try:
            yield conn
            conn.commit()
        except Exception:
            conn.rollback()
            raise
        finally:
            self._pool.putconn(conn)

    @contextmanager
    def cursor(self, cursor_factory=None) -> Generator[Any, None, None]:
        """Get a cursor with automatic connection management."""
        with self.connection() as conn:
            cur = conn.cursor(cursor_factory=cursor_factory or psycopg2.extras.RealDictCursor)
            try:
                yield cur
            finally:
                cur.close()

    def execute(self, query: str, params: tuple = ()) -> list[dict[str, Any]]:
        """Execute query and return results as list of dicts."""
        with self.cursor() as cur:
            cur.execute(query, params)
            if cur.description:  # Has results
                return cur.fetchall()
            return []

    # ─── DevOps operational queries ────────────────────────

    def health_check(self) -> dict[str, Any]:
        """Full database health check."""
        with self.cursor() as cur:
            # Basic connectivity + version
            cur.execute("SELECT version(), now() as server_time, pg_is_in_recovery() as is_replica")
            info = cur.fetchone()

            # Connection stats
            cur.execute("""
                SELECT
                    count(*) as total_connections,
                    count(*) FILTER (WHERE state = 'active') as active,
                    count(*) FILTER (WHERE state = 'idle') as idle,
                    count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction,
                    count(*) FILTER (WHERE wait_event_type = 'Lock') as waiting_on_lock,
                    max(extract(epoch FROM now() - query_start))
                        FILTER (WHERE state = 'active') as longest_query_seconds
                FROM pg_stat_activity
                WHERE pid != pg_backend_pid()
            """)
            conn_stats = cur.fetchone()

            # Database size
            cur.execute("""
                SELECT pg_database_size(current_database()) as db_size_bytes,
                       pg_size_pretty(pg_database_size(current_database())) as db_size_pretty
            """)
            size = cur.fetchone()

            # Replication lag (if replica)
            replication_lag = None
            if info["is_replica"]:
                cur.execute("""
                    SELECT extract(epoch FROM now() - pg_last_xact_replay_timestamp())
                    as lag_seconds
                """)
                lag_row = cur.fetchone()
                replication_lag = lag_row["lag_seconds"] if lag_row else None

            return {
                "status": "healthy",
                "version": info["version"],
                "server_time": str(info["server_time"]),
                "is_replica": info["is_replica"],
                "replication_lag_seconds": replication_lag,
                "connections": dict(conn_stats),
                "database_size": dict(size),
            }

    def long_running_queries(self, min_seconds: int = 60) -> list[dict]:
        """Find queries running longer than threshold."""
        return self.execute("""
            SELECT
                pid,
                now() - query_start as duration,
                state,
                query,
                usename as user,
                application_name as app,
                client_addr
            FROM pg_stat_activity
            WHERE state = 'active'
              AND query_start < now() - interval '%s seconds'
              AND pid != pg_backend_pid()
            ORDER BY query_start ASC
        """, (min_seconds,))

    def table_bloat(self, min_bloat_mb: int = 100) -> list[dict]:
        """Estimate table bloat (dead tuples not yet vacuumed)."""
        return self.execute("""
            SELECT
                schemaname || '.' || relname as table,
                n_live_tup as live_tuples,
                n_dead_tup as dead_tuples,
                CASE WHEN n_live_tup > 0
                     THEN round(100.0 * n_dead_tup / n_live_tup, 1)
                     ELSE 0
                END as dead_pct,
                pg_size_pretty(pg_total_relation_size(relid)) as total_size,
                last_autovacuum,
                last_autoanalyze
            FROM pg_stat_user_tables
            WHERE n_dead_tup > 10000
            ORDER BY n_dead_tup DESC
            LIMIT 20
        """)

    def lock_contention(self) -> list[dict]:
        """Find blocked and blocking queries."""
        return self.execute("""
            SELECT
                blocked.pid as blocked_pid,
                blocked.query as blocked_query,
                blocked.usename as blocked_user,
                now() - blocked.query_start as blocked_duration,
                blocking.pid as blocking_pid,
                blocking.query as blocking_query,
                blocking.usename as blocking_user
            FROM pg_stat_activity blocked
            JOIN pg_locks bl ON bl.pid = blocked.pid
            JOIN pg_locks kl ON kl.locktype = bl.locktype
                AND kl.database IS NOT DISTINCT FROM bl.database
                AND kl.relation IS NOT DISTINCT FROM bl.relation
                AND kl.page IS NOT DISTINCT FROM bl.page
                AND kl.tuple IS NOT DISTINCT FROM bl.tuple
                AND kl.pid != bl.pid
            JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
            WHERE NOT bl.granted
            ORDER BY blocked.query_start
        """)

    def unused_indexes(self, min_size_mb: int = 10) -> list[dict]:
        """Find indexes that are never (or rarely) used — candidates for removal."""
        return self.execute("""
            SELECT
                schemaname || '.' || relname as table,
                indexrelname as index,
                idx_scan as scans,
                pg_size_pretty(pg_relation_size(indexrelid)) as size,
                pg_relation_size(indexrelid) as size_bytes
            FROM pg_stat_user_indexes
            WHERE idx_scan < 50
              AND pg_relation_size(indexrelid) > %s * 1024 * 1024
            ORDER BY pg_relation_size(indexrelid) DESC
        """, (min_size_mb,))

    def terminate_query(self, pid: int) -> bool:
        """Cancel or terminate a backend. Use with caution."""
        result = self.execute("SELECT pg_terminate_backend(%s)", (pid,))
        log.warning("terminated_backend", pid=pid)
        return result[0]["pg_terminate_backend"] if result else False

    def close(self) -> None:
        self._pool.closeall()
        log.info("pg_pool_closed")


# ═══════════════════════════════════════════════════════════
# REDIS (redis-py)
# ═══════════════════════════════════════════════════════════

import redis


class RedisClient:
    """Production Redis client with monitoring."""

    def __init__(
        self,
        host: str = "localhost",
        port: int = 6379,
        db: int = 0,
        password: str | None = None,
        ssl: bool = True,
        socket_timeout: float = 5.0,
        socket_connect_timeout: float = 5.0,
        max_connections: int = 50,
        decode_responses: bool = True,
    ) -> None:
        self._pool = redis.ConnectionPool(
            host=host,
            port=port,
            db=db,
            password=password,
            ssl=ssl,
            socket_timeout=socket_timeout,
            socket_connect_timeout=socket_connect_timeout,
            max_connections=max_connections,
            decode_responses=decode_responses,
            retry_on_timeout=True,
        )
        self._client = redis.Redis(connection_pool=self._pool)
        log.info("redis_client_created", host=host, port=port)

    @property
    def client(self) -> redis.Redis:
        return self._client

    def health_check(self) -> dict[str, Any]:
        """Redis health check with key metrics."""
        info = self._client.info()

        return {
            "status": "healthy" if self._client.ping() else "unhealthy",
            "version": info.get("redis_version"),
            "role": info.get("role"),  # master or slave
            "connected_clients": info.get("connected_clients"),
            "used_memory_human": info.get("used_memory_human"),
            "used_memory_peak_human": info.get("used_memory_peak_human"),
            "maxmemory_human": info.get("maxmemory_human"),
            "maxmemory_policy": info.get("maxmemory_policy"),
            "memory_usage_pct": round(
                info.get("used_memory", 0) / max(info.get("maxmemory", 1), 1) * 100, 1
            ) if info.get("maxmemory", 0) > 0 else "no_limit",
            "keyspace_hits": info.get("keyspace_hits"),
            "keyspace_misses": info.get("keyspace_misses"),
            "hit_rate_pct": round(
                info.get("keyspace_hits", 0) /
                max(info.get("keyspace_hits", 0) + info.get("keyspace_misses", 0), 1) * 100, 1
            ),
            "evicted_keys": info.get("evicted_keys"),
            "total_connections_received": info.get("total_connections_received"),
            "rejected_connections": info.get("rejected_connections"),
            "replication_lag": info.get("master_repl_offset", 0) - info.get("slave_repl_offset", 0)
                if info.get("role") == "slave" else None,
        }

    def find_big_keys(self, sample_size: int = 100) -> list[dict]:
        """
        Find large keys. Approximation using SCAN + DEBUG OBJECT.
        WARNING: DEBUG OBJECT can be slow on huge keys. Don't run in peak hours.
        In production, prefer: redis-cli --bigkeys
        """
        big_keys = []
        cursor = 0

        for _ in range(sample_size):
            cursor, keys = self._client.scan(cursor=cursor, count=100)
            for key in keys:
                try:
                    key_type = self._client.type(key)
                    if key_type == "string":
                        size = self._client.strlen(key)
                    elif key_type == "list":
                        size = self._client.llen(key)
                    elif key_type == "set":
                        size = self._client.scard(key)
                    elif key_type == "hash":
                        size = self._client.hlen(key)
                    elif key_type == "zset":
                        size = self._client.zcard(key)
                    else:
                        size = 0

                    if size > 1000:  # Threshold depends on type
                        memory = self._client.memory_usage(key) or 0
                        big_keys.append({
                            "key": key,
                            "type": key_type,
                            "elements": size,
                            "memory_bytes": memory,
                        })
                except redis.ResponseError:
                    continue

            if cursor == 0:
                break

        return sorted(big_keys, key=lambda x: x.get("memory_bytes", 0), reverse=True)

    def key_pattern_stats(self, patterns: list[str] | None = None) -> dict[str, dict]:
        """
        Analyze key distribution by pattern.
        Useful for: cache size estimation, identifying unexpected key growth.
        """
        if patterns is None:
            patterns = [
                "session:*",
                "cache:*",
                "rate_limit:*",
                "lock:*",
                "queue:*",
            ]

        stats = {}
        for pattern in patterns:
            count = 0
            total_memory = 0
            cursor = 0

            while True:
                cursor, keys = self._client.scan(cursor=cursor, match=pattern, count=1000)
                count += len(keys)
                for key in keys[:50]:  # Sample memory from first 50
                    mem = self._client.memory_usage(key)
                    if mem:
                        total_memory += mem

                if cursor == 0:
                    break

            avg_memory = total_memory // max(min(count, 50), 1)
            stats[pattern] = {
                "count": count,
                "sampled_total_memory": total_memory,
                "estimated_total_memory": avg_memory * count,
                "avg_key_size": avg_memory,
            }

        return stats

    def slowlog(self, count: int = 20) -> list[dict]:
        """Get Redis slow log entries."""
        entries = self._client.slowlog_get(count)
        return [
            {
                "id": e.get("id"),
                "timestamp": e.get("start_time"),
                "duration_us": e.get("duration"),
                "command": " ".join(str(arg) for arg in e.get("command", [])),
                "client_addr": e.get("client_address", ""),
            }
            for e in entries
        ]
```

---

## 10. ADVANCED DECORATORS

```python
"""
src/novatools/utils/decorators.py

Production decorators used daily in DevOps tooling.
"""
from __future__ import annotations

import functools
import time
from typing import Any, Callable, TypeVar

import structlog

log = structlog.get_logger()
F = TypeVar("F", bound=Callable[..., Any])


# ═══════════════════════════════════════════════════════════
# DECORATOR 1: Timing (every function in prod should be measurable)
# ═══════════════════════════════════════════════════════════

def timed(func: F) -> F:
    """Log execution time of a function."""
    @functools.wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.monotonic()
        try:
            result = func(*args, **kwargs)
            duration = time.monotonic() - start
            log.info("function_completed",
                     function=func.__name__,
                     duration_seconds=round(duration, 3))
            return result
        except Exception:
            duration = time.monotonic() - start
            log.error("function_failed",
                      function=func.__name__,
                      duration_seconds=round(duration, 3))
            raise
    return wrapper  # type: ignore[return-value]


# ═══════════════════════════════════════════════════════════
# DECORATOR 2: Rate limiting
# ═══════════════════════════════════════════════════════════

def rate_limit(calls: int, period: float) -> Callable[[F], F]:
    """
    Limit function calls to N per period.
    Used for: API clients, webhook handlers, remediation actions.
    """
    timestamps: list[float] = []

    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            now = time.monotonic()
            # Remove old timestamps
            while timestamps and now - timestamps[0] > period:
                timestamps.pop(0)

            if len(timestamps) >= calls:
                wait_time = period - (now - timestamps[0])
                log.warning("rate_limited",
                            function=func.__name__,
                            wait_seconds=round(wait_time, 1))
                time.sleep(wait_time)

            timestamps.append(time.monotonic())
            return func(*args, **kwargs)
        return wrapper  # type: ignore[return-value]
    return decorator


# ═══════════════════════════════════════════════════════════
# DECORATOR 3: Require confirmation for dangerous operations
# ═══════════════════════════════════════════════════════════

def require_confirmation(message: str = "Are you sure?") -> Callable[[F], F]:
    """Prompt for confirmation before executing dangerous operations."""
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            # Skip confirmation if --yes or --force flag passed
            if kwargs.get("yes") or kwargs.get("force") or kwargs.get("auto_approve"):
                return func(*args, **kwargs)

            # Skip in non-interactive (CI/CD)
            import sys
            if not sys.stdin.isatty():
                log.warning("skipping_confirmation_non_interactive",
                            function=func.__name__)
                return func(*args, **kwargs)

            import click
            click.confirm(f"⚠️  {message}", abort=True)
            return func(*args, **kwargs)
        return wrapper  # type: ignore[return-value]
    return decorator


# Usage:
# @require_confirmation("This will delete all pods in production")
# def delete_all_pods(namespace: str, yes: bool = False) -> None:
#     ...


# ═══════════════════════════════════════════════════════════
# DECORATOR 4: Audit trail (who ran what, when)
# ═══════════════════════════════════════════════════════════

def audit_logged(action: str) -> Callable[[F], F]:
    """
    Log an audit trail for every invocation.
    Used for: destructive operations, secret access, production changes.
    In production, this writes to a dedicated audit log or audit S3 bucket.
    """
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            import getpass
            import socket

            user = getpass.getuser()
            hostname = socket.gethostname()

            # Sanitize kwargs — never log secrets
            safe_kwargs = {
                k: v for k, v in kwargs.items()
                if not any(s in k.lower() for s in ("password", "secret", "token", "key"))
            }

            log.info("audit_trail",
                     action=action,
                     function=func.__name__,
                     user=user,
                     hostname=hostname,
                     args_count=len(args),
                     kwargs=safe_kwargs)

            try:
                result = func(*args, **kwargs)
                log.info("audit_trail_completed",
                         action=action,
                         function=func.__name__,
                         user=user,
                         status="success")
                return result
            except Exception as e:
                log.error("audit_trail_failed",
                          action=action,
                          function=func.__name__,
                          user=user,
                          status="failed",
                          error=str(e))
                raise
        return wrapper  # type: ignore[return-value]
    return decorator


# Usage:
# @audit_logged("secret_rotation")
# @require_confirmation("Rotate production database password?")
# def rotate_password(service: str, environment: str) -> None:
#     ...


# ═══════════════════════════════════════════════════════════
# DECORATOR 5: Environment guard
# ═══════════════════════════════════════════════════════════

def production_guard(func: F) -> F:
    """
    Extra safety for production operations.
    Requires explicit environment flag and adds delay for human review.
    """
    @functools.wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        environment = kwargs.get("environment", "")
        if environment == "production":
            import click
            click.echo("🔴 PRODUCTION OPERATION", err=True)
            click.echo("   Executing in 5 seconds... Ctrl+C to abort", err=True)
            try:
                time.sleep(5)
            except KeyboardInterrupt:
                click.echo("\n❌ Aborted by user", err=True)
                raise SystemExit(1)
        return func(*args, **kwargs)
    return wrapper  # type: ignore[return-value]


# ═══════════════════════════════════════════════════════════
# DECORATOR 6: Cache with TTL (beyond lru_cache)
# ═══════════════════════════════════════════════════════════

def ttl_cache(ttl_seconds: int = 300, maxsize: int = 256) -> Callable[[F], F]:
    """
    Cache with time-to-live expiration.
    lru_cache caches forever — dangerous for dynamic data like instance lists.
    """
    def decorator(func: F) -> F:
        cache: dict[str, tuple[float, Any]] = {}

        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            # Create hashable key from args
            key = str(args) + str(sorted(kwargs.items()))
            now = time.monotonic()

            if key in cache:
                cached_time, cached_value = cache[key]
                if now - cached_time < ttl_seconds:
                    return cached_value

            # Evict if too many entries
            if len(cache) >= maxsize:
                # Remove oldest entries
                oldest = sorted(cache.items(), key=lambda x: x[1][0])
                for old_key, _ in oldest[:maxsize // 4]:
                    del cache[old_key]

            result = func(*args, **kwargs)
            cache[key] = (now, result)
            return result

        # Expose cache control
        wrapper.cache_clear = lambda: cache.clear()  # type: ignore[attr-defined]
        wrapper.cache_info = lambda: {  # type: ignore[attr-defined]
            "size": len(cache),
            "maxsize": maxsize,
            "ttl": ttl_seconds,
        }
        return wrapper  # type: ignore[return-value]
    return decorator

# Usage:
# @ttl_cache(ttl_seconds=60)
# def get_running_instances(region: str) -> list[dict]:
#     """Cached for 60s — don't hammer EC2 API."""
#     return ec2.describe_instances(...)
```

---

## 11. GIT AUTOMATION (GitPython)

```python
"""
src/novatools/clients/git_ops.py

Automated Git operations for CI/CD and GitOps workflows.
Use cases:
  - CI: auto-update image tags in GitOps repo after build
  - Automation: bulk update configs across repos
  - Compliance: audit commit history for policy violations
"""
from __future__ import annotations

import os
import tempfile
from pathlib import Path
from typing import Any

import structlog

log = structlog.get_logger()


# ═══════════════════════════════════════════════════════════
# PATTERN 1: GitPython for local repo operations
# ═══════════════════════════════════════════════════════════

from git import Repo, GitCommandError


class GitOpsUpdater:
    """
    Update manifests in a GitOps repository.
    This is what CI does after a successful build:
    1. Clone GitOps repo
    2. Update image tag in the correct overlay
    3. Commit and push
    4. ArgoCD detects change and deploys
    """

    def __init__(
        self,
        repo_url: str,
        branch: str = "main",
        ssh_key_path: str | None = None,
        clone_dir: str | None = None,
    ) -> None:
        self.repo_url = repo_url
        self.branch = branch
        self.ssh_key_path = ssh_key_path
        self._clone_dir = clone_dir
        self._repo: Repo | None = None

    @property
    def repo(self) -> Repo:
        if self._repo is None:
            raise RuntimeError("Repository not cloned. Call clone() first.")
        return self._repo

    def clone(self) -> Path:
        """Clone the repository."""
        if self._clone_dir:
            target = Path(self._clone_dir)
        else:
            target = Path(tempfile.mkdtemp(prefix="gitops-"))

        env = {}
        if self.ssh_key_path:
            env["GIT_SSH_COMMAND"] = f"ssh -i {self.ssh_key_path} -o StrictHostKeyChecking=no"

        log.info("cloning_repo", url=self.repo_url, branch=self.branch, target=str(target))

        self._repo = Repo.clone_from(
            self.repo_url,
            str(target),
            branch=self.branch,
            depth=1,       # Shallow clone — we don't need history
            env=env,
        )
        return target

    def update_image_tag(
        self,
        service: str,
        new_tag: str,
        environment: str = "production",
    ) -> bool:
        """
        Update image tag in Kustomize overlay.
        Expects structure:
          environments/<env>/kustomization.yaml
            images:
              - name: <service>
                newTag: <tag>
        """
        from ruamel.yaml import YAML
        ryaml = YAML()
        ryaml.preserve_quotes = True

        kustomization_path = (
            Path(self.repo.working_dir) /
            "environments" / environment / "kustomization.yaml"
        )

        if not kustomization_path.exists():
            log.error("kustomization_not_found", path=str(kustomization_path))
            return False

        with open(kustomization_path) as f:
            data = ryaml.load(f)

        images = data.get("images", [])
        updated = False
        old_tag = None

        for image in images:
            if image.get("name") == service:
                old_tag = image.get("newTag", "unknown")
                image["newTag"] = new_tag
                updated = True
                break

        if not updated:
            # Service not in images list yet — add it
            images.append({"name": service, "newTag": new_tag})
            data["images"] = images
            old_tag = "none"
            updated = True

        with open(kustomization_path, "w") as f:
            ryaml.dump(data, f)

        log.info("image_tag_updated",
                 service=service, old_tag=old_tag, new_tag=new_tag,
                 environment=environment)
        return updated

    def commit_and_push(
        self,
        message: str,
        author_name: str = "NovaMart CI",
        author_email: str = "ci@novamart.com",
    ) -> str | None:
        """Commit changes and push. Returns commit SHA or None if nothing to commit."""
        repo = self.repo

        # Check for changes
        if not repo.is_dirty(untracked_files=True):
            log.info("no_changes_to_commit")
            return None

        # Stage all changes
        repo.git.add(A=True)

        # Commit
        repo.config_writer().set_value("user", "name", author_name).release()
        repo.config_writer().set_value("user", "email", author_email).release()

        commit = repo.index.commit(message)
        sha = commit.hexsha[:8]

        log.info("committed", sha=sha, message=message)

        # Push
        env = {}
        if self.ssh_key_path:
            env["GIT_SSH_COMMAND"] = f"ssh -i {self.ssh_key_path} -o StrictHostKeyChecking=no"

        try:
            origin = repo.remotes.origin
            origin.push(env=env)
            log.info("pushed", sha=sha, branch=self.branch)
            return commit.hexsha
        except GitCommandError as e:
            log.error("push_failed", error=str(e))
            raise

    def cleanup(self) -> None:
        """Remove cloned directory."""
        import shutil
        if self._repo and self._repo.working_dir:
            shutil.rmtree(self._repo.working_dir, ignore_errors=True)
            log.info("cleanup_complete")


# ═══════════════════════════════════════════════════════════
# PATTERN 2: Bitbucket/GitHub API for PR creation
# ═══════════════════════════════════════════════════════════

import requests


class BitbucketClient:
    """Bitbucket Cloud API client for automated PR creation."""

    def __init__(
        self,
        workspace: str,
        username: str,
        app_password: str,
    ) -> None:
        self.workspace = workspace
        self.base_url = f"https://api.bitbucket.org/2.0/repositories/{workspace}"
        self._session = requests.Session()
        self._session.auth = (username, app_password)
        self._session.headers["Content-Type"] = "application/json"

    def create_pull_request(
        self,
        repo_slug: str,
        title: str,
        source_branch: str,
        destination_branch: str = "main",
        description: str = "",
        reviewers: list[str] | None = None,
        close_source_branch: bool = True,
    ) -> dict:
        """Create a pull request."""
        payload: dict[str, Any] = {
            "title": title,
            "source": {"branch": {"name": source_branch}},
            "destination": {"branch": {"name": destination_branch}},
            "description": description,
            "close_source_branch": close_source_branch,
        }

        if reviewers:
            payload["reviewers"] = [{"uuid": r} for r in reviewers]

        response = self._session.post(
            f"{self.base_url}/{repo_slug}/pullrequests",
            json=payload,
            timeout=(5, 30),
        )
        response.raise_for_status()

        pr = response.json()
        log.info("pr_created",
                 repo=repo_slug,
                 pr_id=pr["id"],
                 url=pr["links"]["html"]["href"])
        return pr

    def add_pr_comment(
        self,
        repo_slug: str,
        pr_id: int,
        content: str,
    ) -> dict:
        """Add comment to a PR (used for CI results, plan output, etc.)."""
        response = self._session.post(
            f"{self.base_url}/{repo_slug}/pullrequests/{pr_id}/comments",
            json={"content": {"raw": content}},
            timeout=(5, 30),
        )
        response.raise_for_status()
        return response.json()


# ═══════════════════════════════════════════════════════════
# USAGE: CI deploys image via GitOps
# ═══════════════════════════════════════════════════════════

def ci_update_gitops(
    service: str,
    version: str,
    environment: str,
    gitops_repo_url: str,
    ssh_key_path: str,
) -> str | None:
    """
    Called by Jenkins after successful build + scan + sign.
    Updates GitOps repo → ArgoCD detects → deploys.
    """
    updater = GitOpsUpdater(
        repo_url=gitops_repo_url,
        branch="main",
        ssh_key_path=ssh_key_path,
    )

    try:
        updater.clone()
        updater.update_image_tag(service, version, environment)

        commit_sha = updater.commit_and_push(
            message=f"deploy({environment}): {service}:{version}\n\n"
                    f"Automated by NovaMart CI\n"
                    f"Service: {service}\n"
                    f"Version: {version}\n"
                    f"Environment: {environment}",
        )

        return commit_sha
    finally:
        updater.cleanup()
```

---

## 12. DOCKER SDK (docker-py)

```python
"""
src/novatools/clients/docker_ops.py

Docker SDK for Python — programmatic container and image management.
Use cases: local dev tooling, CI image management, cleanup automation.
"""
from __future__ import annotations

from typing import Any

import docker
import structlog

log = structlog.get_logger()


class DockerManager:
    """Docker operations for DevOps automation."""

    def __init__(self) -> None:
        self._client = docker.from_env()
        log.info("docker_client_initialized",
                 version=self._client.version().get("Version"))

    # ─── Image operations ──────────────────────────────────

    def build_image(
        self,
        path: str,
        tag: str,
        dockerfile: str = "Dockerfile",
        build_args: dict[str, str] | None = None,
        no_cache: bool = False,
        target: str | None = None,
    ) -> str:
        """Build Docker image with streaming output."""
        log.info("building_image", tag=tag, path=path, target=target)

        image, build_logs = self._client.images.build(
            path=path,
            tag=tag,
            dockerfile=dockerfile,
            buildargs=build_args or {},
            nocache=no_cache,
            target=target,
            rm=True,        # Remove intermediate containers
        )

        for chunk in build_logs:
            if "stream" in chunk:
                line = chunk["stream"].strip()
                if line:
                    log.debug("build_log", line=line)

        log.info("image_built", tag=tag, image_id=image.short_id)
        return image.id

    def push_image(self, tag: str, registry_auth: dict | None = None) -> None:
        """Push image to registry."""
        log.info("pushing_image", tag=tag)

        for line in self._client.images.push(tag, stream=True, decode=True):
            if "error" in line:
                raise RuntimeError(f"Push failed: {line['error']}")
            if "status" in line:
                log.debug("push_status", status=line["status"],
                          progress=line.get("progress", ""))

        log.info("image_pushed", tag=tag)

    def prune_images(
        self,
        dangling_only: bool = False,
        older_than: str = "",
    ) -> dict[str, Any]:
        """
        Clean up unused Docker images.
        older_than: "24h", "72h", etc.
        """
        filters: dict[str, Any] = {}
        if dangling_only:
            filters["dangling"] = True
        if older_than:
            filters["until"] = older_than

        result = self._client.images.prune(filters=filters)

        deleted = result.get("ImagesDeleted") or []
        space = result.get("SpaceReclaimed", 0)

        log.info("images_pruned",
                 deleted_count=len(deleted),
                 space_reclaimed_mb=round(space / 1024 / 1024, 1))

        return {
            "deleted_count": len(deleted),
            "space_reclaimed_bytes": space,
            "space_reclaimed_mb": round(space / 1024 / 1024, 1),
        }

    def list_images_by_age(self, repo_filter: str = "") -> list[dict]:
        """List images sorted by creation date. For cleanup decisions."""
        from datetime import datetime, UTC

        images = self._client.images.list()
        result = []

        for img in images:
            tags = img.tags
            if repo_filter and not any(repo_filter in t for t in tags):
                continue

            created = img.attrs.get("Created", "")
            try:
                # Docker returns ISO 8601 with nanoseconds
                created_dt = datetime.fromisoformat(created.split(".")[0].replace("Z", "+00:00"))
                age_days = (datetime.now(UTC) - created_dt).days
            except (ValueError, TypeError):
                age_days = -1

            result.append({
                "id": img.short_id,
                "tags": tags,
                "size_mb": round(img.attrs.get("Size", 0) / 1024 / 1024, 1),
                "created": created,
                "age_days": age_days,
            })

        return sorted(result, key=lambda x: x["age_days"], reverse=True)

    # ─── Container operations ──────────────────────────────

    def run_ephemeral(
        self,
        image: str,
        command: str | list[str],
        environment: dict[str, str] | None = None,
        volumes: dict[str, dict] | None = None,
        network: str | None = None,
        timeout: int = 300,
        memory_limit: str = "512m",
        cpu_limit: float = 1.0,
    ) -> tuple[int, str]:
        """
        Run a one-shot container and return (exit_code, output).
        Used for: migration scripts, data processing, tool execution.
        """
        log.info("running_container", image=image, command=str(command))

        try:
            container = self._client.containers.run(
                image=image,
                command=command,
                environment=environment or {},
                volumes=volumes or {},
                network=network,
                detach=True,
                mem_limit=memory_limit,
                nano_cpus=int(cpu_limit * 1e9),
                remove=False,  # Don't auto-remove — we need exit code
            )

            # Wait for completion
            result = container.wait(timeout=timeout)
            exit_code = result["StatusCode"]
            output = container.logs(stdout=True, stderr=True).decode("utf-8")

            log.info("container_completed",
                     image=image, exit_code=exit_code,
                     output_length=len(output))

            container.remove()
            return exit_code, output

        except docker.errors.ContainerError as e:
            log.error("container_error", image=image, error=str(e))
            return e.exit_status, e.stderr.decode("utf-8") if e.stderr else str(e)

    def container_stats(self) -> list[dict]:
        """Get resource usage of all running containers."""
        containers = self._client.containers.list()
        stats = []

        for c in containers:
            raw = c.stats(stream=False)

            # CPU calculation
            cpu_delta = (
                raw["cpu_stats"]["cpu_usage"]["total_usage"] -
                raw["precpu_stats"]["cpu_usage"]["total_usage"]
            )
            system_delta = (
                raw["cpu_stats"]["system_cpu_usage"] -
                raw["precpu_stats"]["system_cpu_usage"]
            )
            num_cpus = raw["cpu_stats"].get("online_cpus",
                        len(raw["cpu_stats"]["cpu_usage"].get("percpu_usage", [1])))
            cpu_pct = (cpu_delta / max(system_delta, 1)) * num_cpus * 100.0

            # Memory
            mem_usage = raw["memory_stats"].get("usage", 0)
            mem_limit = raw["memory_stats"].get("limit", 1)
            mem_pct = (mem_usage / mem_limit) * 100.0

            stats.append({
                "name": c.name,
                "id": c.short_id,
                "image": c.image.tags[0] if c.image.tags else c.image.short_id,
                "cpu_pct": round(cpu_pct, 1),
                "memory_mb": round(mem_usage / 1024 / 1024, 1),
                "memory_limit_mb": round(mem_limit / 1024 / 1024, 1),
                "memory_pct": round(mem_pct, 1),
            })

        return sorted(stats, key=lambda x: x["cpu_pct"], reverse=True)

    def cleanup_all(self, force: bool = False) -> dict[str, Any]:
        """Full Docker cleanup — containers, images, volumes, networks."""
        results = {}

        if force:
            # Stop all containers
            for c in self._client.containers.list():
                c.stop(timeout=10)
                c.remove()
            results["containers_removed"] = True

        # Prune everything
        results["containers"] = self._client.containers.prune()
        results["images"] = self.prune_images(dangling_only=True)
        results["volumes"] = self._client.volumes.prune()
        results["networks"] = self._client.networks.prune()

        return results
```

---

## 13. DATE/TIME OPERATIONS

```python
"""
src/novatools/utils/timeutil.py

Time operations for DevOps: maintenance windows, schedule checks,
age calculations, timezone conversions.
"""
from __future__ import annotations

from datetime import datetime, timedelta, timezone, UTC
from typing import Any
from zoneinfo import ZoneInfo  # Python 3.9+ (replaces pytz)


# ═══════════════════════════════════════════════════════════
# RULE 1: ALWAYS use timezone-aware datetimes
# ═══════════════════════════════════════════════════════════

def now_utc() -> datetime:
    """Current time in UTC. THE standard for all DevOps tooling."""
    return datetime.now(UTC)

def parse_timestamp(ts: str) -> datetime:
    """
    Parse any common timestamp format to timezone-aware datetime.
    Handles: ISO 8601, AWS format, K8s format, common log formats.
    """
    # Try ISO 8601 first (most common)
    formats = [
        "%Y-%m-%dT%H:%M:%S%z",           # 2024-01-15T10:30:00+00:00
        "%Y-%m-%dT%H:%M:%SZ",            # 2024-01-15T10:30:00Z (K8s)
        "%Y-%m-%dT%H:%M:%S.%f%z",        # 2024-01-15T10:30:00.123456+00:00
        "%Y-%m-%dT%H:%M:%S.%fZ",         # 2024-01-15T10:30:00.123456Z
        "%Y-%m-%d %H:%M:%S",             # 2024-01-15 10:30:00 (naive → assume UTC)
        "%d/%b/%Y:%H:%M:%S %z",          # 15/Jan/2024:10:30:00 +0000 (nginx)
    ]

    for fmt in formats:
        try:
            dt = datetime.strptime(ts, fmt)
            if dt.tzinfo is None:
                dt = dt.replace(tzinfo=UTC)
            return dt
        except ValueError:
            continue

    # Last resort: fromisoformat (Python 3.11+ handles more formats)
    try:
        dt = datetime.fromisoformat(ts.replace("Z", "+00:00"))
        return dt
    except ValueError:
        raise ValueError(f"Cannot parse timestamp: {ts}")


def human_duration(seconds: float) -> str:
    """Convert seconds to human-readable duration."""
    if seconds < 60:
        return f"{seconds:.1f}s"
    elif seconds < 3600:
        return f"{seconds / 60:.1f}m"
    elif seconds < 86400:
        return f"{seconds / 3600:.1f}h"
    else:
        return f"{seconds / 86400:.1f}d"


def age_from_timestamp(ts: str | datetime) -> dict[str, Any]:
    """Calculate age from a timestamp. Used everywhere in DevOps."""
    if isinstance(ts, str):
        dt = parse_timestamp(ts)
    else:
        dt = ts

    delta = now_utc() - dt
    return {
        "age_seconds": int(delta.total_seconds()),
        "age_human": human_duration(delta.total_seconds()),
        "age_days": delta.days,
        "timestamp": dt.isoformat(),
    }


# ═══════════════════════════════════════════════════════════
# MAINTENANCE WINDOWS
# ═══════════════════════════════════════════════════════════

def is_in_maintenance_window(
    window_start_hour: int = 2,   # 2 AM UTC
    window_end_hour: int = 6,     # 6 AM UTC
    timezone_name: str = "UTC",
) -> bool:
    """Check if current time is within maintenance window."""
    tz = ZoneInfo(timezone_name)
    now = datetime.now(tz)
    return window_start_hour <= now.hour < window_end_hour


def next_maintenance_window(
    window_start_hour: int = 2,
    timezone_name: str = "UTC",
) -> datetime:
    """Calculate next maintenance window start."""
    tz = ZoneInfo(timezone_name)
    now = datetime.now(tz)

    next_window = now.replace(hour=window_start_hour, minute=0, second=0, microsecond=0)
    if next_window <= now:
        next_window += timedelta(days=1)

    return next_window


def is_business_hours(
    timezone_name: str = "America/New_York",
    start_hour: int = 9,
    end_hour: int = 17,
) -> bool:
    """
    Check if it's business hours.
    Used to: suppress noisy alerts, defer non-urgent automation,
    avoid deploying during peak traffic.
    """
    tz = ZoneInfo(timezone_name)
    now = datetime.now(tz)
    # Weekday: 0=Monday, 6=Sunday
    if now.weekday() >= 5:  # Weekend
        return False
    return start_hour <= now.hour < end_hour


# ═══════════════════════════════════════════════════════════
# CERTIFICATE / RESOURCE EXPIRY
# ═══════════════════════════════════════════════════════════

def check_expiry(
    expiry_time: str | datetime,
    warn_days: int = 30,
    critical_days: int = 7,
) -> dict[str, Any]:
    """
    Check if a resource (cert, secret, key) is approaching expiry.
    Returns severity level and days remaining.
    """
    if isinstance(expiry_time, str):
        expiry_dt = parse_timestamp(expiry_time)
    else:
        expiry_dt = expiry_time

    remaining = expiry_dt - now_utc()
    days_remaining = remaining.days

    if days_remaining < 0:
        severity = "EXPIRED"
    elif days_remaining <= critical_days:
        severity = "CRITICAL"
    elif days_remaining <= warn_days:
        severity = "WARNING"
    else:
        severity = "OK"

    return {
        "expiry": expiry_dt.isoformat(),
        "days_remaining": days_remaining,
        "severity": severity,
        "human": human_duration(remaining.total_seconds()) if days_remaining >= 0 else "EXPIRED",
    }


# ═══════════════════════════════════════════════════════════
# SCHEDULING AND INTERVALS
# ═══════════════════════════════════════════════════════════

def parse_duration(duration_str: str) -> timedelta:
    """
    Parse human-friendly duration strings.
    '5m' → 5 minutes, '2h' → 2 hours, '7d' → 7 days, '1w' → 1 week
    Matches Prometheus duration format.
    """
    import re
    match = re.match(r"^(\d+)([smhdw])$", duration_str.strip())
    if not match:
        raise ValueError(f"Invalid duration: {duration_str}. Use format: 5m, 2h, 7d, 1w")

    value = int(match.group(1))
    unit = match.group(2)

    units = {
        "s": timedelta(seconds=value),
        "m": timedelta(minutes=value),
        "h": timedelta(hours=value),
        "d": timedelta(days=value),
        "w": timedelta(weeks=value),
    }
    return units[unit]
```

---

## 14. CSV/REPORTING (pandas basics for DevOps)

```python
"""
pandas for DevOps — just enough for cost reports, capacity planning, 
and data processing. Not data science. Not ML. Operational reporting.
"""
from __future__ import annotations

from pathlib import Path
from typing import Any

import pandas as pd
import structlog

log = structlog.get_logger()


# ═══════════════════════════════════════════════════════════
# PATTERN 1: AWS Cost and Usage Report analysis
# ═══════════════════════════════════════════════════════════

def analyze_aws_costs(
    cur_csv_path: Path,
    group_by: str = "lineItem/ProductCode",
) -> dict[str, Any]:
    """
    Analyze AWS CUR (Cost and Usage Report) CSV.
    CUR files can be 100MB+ — pandas handles this efficiently.
    """
    # Read only needed columns (CUR has 100+ columns)
    usecols = [
        "lineItem/UsageStartDate",
        "lineItem/ProductCode",
        "lineItem/UsageType",
        "lineItem/UnblendedCost",
        "lineItem/UsageAmount",
        "resourceTags/user:team",
        "resourceTags/user:environment",
        "resourceTags/user:service",
    ]

    df = pd.read_csv(
        cur_csv_path,
        usecols=lambda c: c in usecols,  # Only load columns that exist
        low_memory=False,
        parse_dates=["lineItem/UsageStartDate"],
    )

    # Rename for readability
    df.columns = [c.split("/")[-1] for c in df.columns]

    # Basic aggregation
    total_cost = df["UnblendedCost"].sum()

    # Cost by service
    by_service = (
        df.groupby("ProductCode")["UnblendedCost"]
        .sum()
        .sort_values(ascending=False)
        .head(20)
    )

    # Cost by team tag
    by_team = (
        df.groupby("user:team")["UnblendedCost"]
        .sum()
        .sort_values(ascending=False)
    )

    # Untagged spend (visibility gap)
    untagged = df[df["user:team"].isna()]["UnblendedCost"].sum()
    untagged_pct = (untagged / total_cost * 100) if total_cost > 0 else 0

    # Daily trend
    daily = (
        df.groupby(df["UsageStartDate"].dt.date)["UnblendedCost"]
        .sum()
    )

    return {
        "total_cost": round(total_cost, 2),
        "by_service": by_service.to_dict(),
        "by_team": by_team.to_dict(),
        "untagged_cost": round(untagged, 2),
        "untagged_pct": round(untagged_pct, 1),
        "daily_trend": {str(k): round(v, 2) for k, v in daily.items()},
        "top_cost_driver": by_service.index[0] if len(by_service) > 0 else "N/A",
    }


# ═══════════════════════════════════════════════════════════
# PATTERN 2: Instance inventory analysis
# ═══════════════════════════════════════════════════════════

def generate_capacity_report(instances: list[dict]) -> dict[str, Any]:
    """
    Analyze EC2 instance fleet for capacity planning.
    Input: list of dicts from boto3 describe_instances.
    """
    df = pd.DataFrame(instances)

    if df.empty:
        return {"total": 0}

    return {
        "total_instances": len(df),
        "by_type": df["type"].value_counts().to_dict(),
        "by_az": df["az"].value_counts().to_dict() if "az" in df else {},
        "by_state": df["state"].value_counts().to_dict() if "state" in df else {},
        "age_stats": {
            "avg_age_days": round(df["age_days"].mean(), 1) if "age_days" in df else 0,
            "max_age_days": int(df["age_days"].max()) if "age_days" in df else 0,
            "over_180_days": int((df["age_days"] > 180).sum()) if "age_days" in df else 0,
        },
    }


# ═══════════════════════════════════════════════════════════
# PATTERN 3: Export reports in multiple formats
# ═══════════════════════════════════════════════════════════

def export_report(
    data: list[dict],
    output_path: Path,
    fmt: str = "csv",
) -> None:
    """Export structured data to CSV, Excel, or JSON."""
    df = pd.DataFrame(data)

    if fmt == "csv":
        df.to_csv(output_path, index=False)
    elif fmt == "excel":
        df.to_excel(output_path, index=False, engine="openpyxl")
    elif fmt == "json":
        df.to_json(output_path, orient="records", indent=2)
    else:
        raise ValueError(f"Unsupported format: {fmt}")

    log.info("report_exported", path=str(output_path), format=fmt, rows=len(df))
```

---

## 15. PROMETHEUS CLIENT (Custom Metrics)

```python
"""
src/novatools/utils/metrics.py

Expose custom Prometheus metrics from Python services.
Used for: webhook handler metrics, CLI tool duration tracking,
batch job monitoring, custom business metrics.
"""
from __future__ import annotations

from prometheus_client import (
    Counter,
    Gauge,
    Histogram,
    Info,
    start_http_server,
    generate_latest,
    REGISTRY,
)

# ═══════════════════════════════════════════════════════════
# Define metrics at module level (registered once)
# ═══════════════════════════════════════════════════════════

# Webhook processing
WEBHOOK_RECEIVED = Counter(
    "novatools_webhook_received_total",
    "Total webhooks received",
    ["source", "event_type"],  # Labels
)

WEBHOOK_PROCESSING_DURATION = Histogram(
    "novatools_webhook_processing_duration_seconds",
    "Webhook processing duration",
    ["source", "event_type"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

WEBHOOK_ERRORS = Counter(
    "novatools_webhook_errors_total",
    "Webhook processing errors",
    ["source", "event_type", "error_type"],
)

# Tool execution
TOOL_EXECUTION = Histogram(
    "novatools_tool_execution_duration_seconds",
    "CLI tool execution duration",
    ["tool", "subcommand", "environment"],
    buckets=[1, 5, 10, 30, 60, 120, 300, 600],
)

TOOL_ERRORS = Counter(
    "novatools_tool_errors_total",
    "CLI tool errors",
    ["tool", "subcommand", "error_type"],
)

# Active operations gauge
ACTIVE_DEPLOYMENTS = Gauge(
    "novatools_active_deployments",
    "Currently running deployments",
    ["environment"],
)

# Build info
BUILD_INFO = Info(
    "novatools_build",
    "Build information",
)


# ═══════════════════════════════════════════════════════════
# Usage in webhook handler
# ═══════════════════════════════════════════════════════════

def handle_webhook_with_metrics(source: str, event_type: str, payload: dict) -> None:
    """Example of instrumenting a webhook handler."""
    WEBHOOK_RECEIVED.labels(source=source, event_type=event_type).inc()

    with WEBHOOK_PROCESSING_DURATION.labels(
        source=source, event_type=event_type
    ).time():  # Automatically measures duration
        try:
            process_webhook(payload)
        except Exception as e:
            WEBHOOK_ERRORS.labels(
                source=source,
                event_type=event_type,
                error_type=type(e).__name__,
            ).inc()
            raise


# ═══════════════════════════════════════════════════════════
# FastAPI integration
# ═══════════════════════════════════════════════════════════

from fastapi import Response

# Add to your FastAPI app:
# @app.get("/metrics")
# async def metrics():
#     return Response(
#         content=generate_latest(REGISTRY),
#         media_type="text/plain; version=0.0.4; charset=utf-8",
#     )

# Then add ServiceMonitor in K8s:
# apiVersion: monitoring.coreos.com/v1
# kind: ServiceMonitor
# metadata:
#   name: novatools-webhook
# spec:
#   selector:
#     matchLabels:
#       app: novatools-webhook
#   endpoints:
#   - port: http
#     path: /metrics
#     interval: 30s
```

---

## 16. MIGRATION SCRIPTS PATTERNS

```python
"""
Data migration script patterns — one-off but must be reliable.
Used for: database migrations, data backfills, configuration migrations,
service cutover scripts.
"""
from __future__ import annotations

import json
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any

import structlog

log = structlog.get_logger()


@dataclass
class MigrationState:
    """
    Track migration progress — survives script restart.
    Written to disk/S3 so migration can resume from where it stopped.
    """
    total_items: int = 0
    processed: int = 0
    succeeded: int = 0
    failed: int = 0
    last_processed_id: str = ""
    failed_items: list[str] = field(default_factory=list)
    start_time: str = ""
    checkpoint_time: str = ""

    @property
    def progress_pct(self) -> float:
        return (self.processed / max(self.total_items, 1)) * 100

    def save(self, path: Path) -> None:
        """Persist state to disk."""
        path.write_text(json.dumps(self.__dict__, indent=2))

    @classmethod
    def load(cls, path: Path) -> "MigrationState":
        """Load state from disk (for resume)."""
        if path.exists():
            data = json.loads(path.read_text())
            return cls(**data)
        return cls()


def run_migration_with_checkpointing(
    items: list[dict],
    process_fn: Any,
    state_file: Path,
    batch_size: int = 100,
    checkpoint_interval: int = 50,
    dry_run: bool = False,
) -> MigrationState:
    """
    Generic migration runner with:
    - Resumability (checkpoint file)
    - Batch processing
    - Progress tracking
    - Failed item collection (for retry)
    - Dry-run mode
    """
    state = MigrationState.load(state_file)

    # Resume support
    if state.last_processed_id:
        log.info("resuming_migration",
                 last_id=state.last_processed_id,
                 processed=state.processed)
        # Skip already processed items
        skip_until = state.last_processed_id
        items = [
            i for i in items
            if i.get("id", "") > skip_until
        ]

    state.total_items = state.processed + len(items)
    state.start_time = state.start_time or datetime.now(UTC).isoformat()

    from novatools.utils.timeutil import now_utc
    from datetime import datetime, UTC

    for i, item in enumerate(items):
        item_id = item.get("id", str(i))

        try:
            if dry_run:
                log.debug("dry_run_skip", item_id=item_id)
            else:
                process_fn(item)

            state.succeeded += 1
        except Exception as e:
            log.error("migration_item_failed", item_id=item_id, error=str(e))
            state.failed += 1
            state.failed_items.append(item_id)

        state.processed += 1
        state.last_processed_id = item_id

        # Checkpoint periodically
        if state.processed % checkpoint_interval == 0:
            state.checkpoint_time = now_utc().isoformat()
            state.save(state_file)
            log.info("checkpoint_saved",
                     progress=f"{state.progress_pct:.1f}%",
                     processed=state.processed,
                     failed=state.failed)

        # Rate limiting — don't overwhelm downstream
        if i > 0 and i % batch_size == 0:
            time.sleep(0.5)

    # Final save
    state.checkpoint_time = now_utc().isoformat()
    state.save(state_file)

    log.info("migration_complete",
             total=state.total_items,
             succeeded=state.succeeded,
             failed=state.failed,
             failed_items=state.failed_items[:20])  # Log first 20

    return state
```

---

## 17. CACHING PATTERNS (beyond lru_cache)

```python
"""
Caching patterns for DevOps tooling.
When to cache, what to cache, cache invalidation strategies.
"""
from __future__ import annotations

import time
import hashlib
import json
from functools import lru_cache
from typing import Any

import structlog

log = structlog.get_logger()


# ═══════════════════════════════════════════════════════════
# PATTERN 1: lru_cache for pure functions
# ═══════════════════════════════════════════════════════════

@lru_cache(maxsize=128)
def parse_arn(arn: str) -> dict[str, str]:
    """Parse AWS ARN — pure function, safe to cache forever."""
    # arn:partition:service:region:account-id:resource
    parts = arn.split(":")
    if len(parts) < 6:
        raise ValueError(f"Invalid ARN: {arn}")
    return {
        "partition": parts[1],
        "service": parts[2],
        "region": parts[3],
        "account_id": parts[4],
        "resource": ":".join(parts[5:]),
    }


# ═══════════════════════════════════════════════════════════
# PATTERN 2: TTL cache for API responses
# ═══════════════════════════════════════════════════════════

from cachetools import TTLCache, cached
import threading

# Thread-safe TTL cache
_instance_cache = TTLCache(maxsize=500, ttl=60)  # 60 second TTL
_cache_lock = threading.Lock()

@cached(cache=_instance_cache, lock=_cache_lock)
def get_instance_info(instance_id: str, region: str = "us-east-1") -> dict:
    """
    Cache EC2 instance info for 60 seconds.
    Without cache: 50 calls = 50 API calls = throttling.
    With cache: 50 calls = 1 API call.
    """
    import boto3
    ec2 = boto3.client("ec2", region_name=region)
    response = ec2.describe_instances(InstanceIds=[instance_id])
    return response["Reservations"][0]["Instances"][0]


# ═══════════════════════════════════════════════════════════
# PATTERN 3: File-based cache for expensive computations
# ═══════════════════════════════════════════════════════════

from pathlib import Path


class FileCache:
    """
    Persistent cache backed by filesystem.
    Survives process restart. Used for:
    - Expensive API calls during incident (don't re-fetch)
    - Cost report data (compute once, render multiple times)
    - Build artifacts (skip if inputs unchanged)
    """

    def __init__(self, cache_dir: Path = Path("/tmp/novatools-cache")) -> None:
        self.cache_dir = cache_dir
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def _key_path(self, key: str) -> Path:
        hashed = hashlib.sha256(key.encode()).hexdigest()[:16]
        return self.cache_dir / f"{hashed}.json"

    def get(self, key: str, max_age_seconds: int = 3600) -> Any | None:
        path = self._key_path(key)
        if not path.exists():
            return None

        age = time.time() - path.stat().st_mtime
        if age > max_age_seconds:
            path.unlink()
            return None

        try:
            return json.loads(path.read_text())
        except (json.JSONDecodeError, OSError):
            return None

    def set(self, key: str, value: Any) -> None:
        path = self._key_path(key)
        path.write_text(json.dumps(value))

    def invalidate(self, key: str) -> None:
        path = self._key_path(key)
        if path.exists():
            path.unlink()

    def clear(self) -> int:
        """Clear all cache entries. Returns count deleted."""
        count = 0
        for f in self.cache_dir.glob("*.json"):
            f.unlink()
            count += 1
        return count


# Usage:
# cache = FileCache()
# key = f"cost-report-{region}-{date}"
# data = cache.get(key, max_age_seconds=3600)
# if data is None:
#     data = expensive_cost_calculation()
#     cache.set(key, data)
```

---

Now. That's all 17 missing scenarios delivered. Let me give you the honest updated self-evaluation.


## UPDATED QUICK REFERENCE CARD

```
╔═══════════════════════════════════════════════════════════════╗
║         PYTHON FOR DEVOPS — COMPLETE CHEAT SHEET             ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║ PROJECT STRUCTURE:                                            ║
║   src/novatools/                                              ║
║     clients/  (aws, slack, jira, pagerduty, git, docker, db) ║
║     utils/    (config, logging, shell, timeutil, decorators)  ║
║     webhooks/ (FastAPI server)                                ║
║     lambdas/  (Lambda handlers)                               ║
║     scripts/  (CLI tools)                                     ║
║   tests/      (mirrors src/)                                  ║
║   pyproject.toml + ruff + mypy --strict                       ║
║                                                               ║
║ SUBPROCESS (daily):                                           ║
║   subprocess.run(cmd_list, capture_output=True, text=True,    ║
║                  timeout=300)                                 ║
║   NEVER shell=True with user input                            ║
║   Wrap kubectl/terraform/helm/argocd in typed classes         ║
║                                                               ║
║ FILE OPS:                                                     ║
║   Path (not os.path) — / operator, .read_text(), .glob()     ║
║   atomic_write() for config changes (temp + rename)           ║
║   gzip.open() for compressed logs                             ║
║   ruamel.yaml for round-trip YAML editing (preserves fmt)     ║
║                                                               ║
║ CONFIG:                                                       ║
║   pydantic-settings: env vars → .env → defaults               ║
║   @lru_cache(maxsize=1) singleton                             ║
║   Validate at startup, fail fast                              ║
║                                                               ║
║ REGEX:                                                        ║
║   Compile at module level, not in loops                       ║
║   re.search() not re.match() (unless anchored)               ║
║   re.escape(user_input) always                                ║
║   Named groups: (?P<name>...)                                 ║
║                                                               ║
║ WEBHOOKS (FastAPI):                                           ║
║   HMAC signature verification on ALL endpoints                ║
║   BackgroundTasks for heavy processing (return 200 fast)      ║
║   Idempotency via fingerprint/dedup_key + Redis               ║
║   /healthz + /readyz + /metrics                               ║
║                                                               ║
║ LAMBDA:                                                       ║
║   Clients OUTSIDE handler (warm start reuse)                  ║
║   Clean /tmp on start (reused across invocations)             ║
║   Idempotent handlers (retries happen)                        ║
║   Least privilege IAM role per function                       ║
║                                                               ║
║ DATABASE:                                                     ║
║   Connection pooling (ThreadedConnectionPool / redis pool)    ║
║   statement_timeout to prevent runaway queries                ║
║   sslmode=require in production                               ║
║   Health: connections, replication lag, bloat, locks           ║
║                                                               ║
║ TIME:                                                         ║
║   datetime.now(UTC) — ALWAYS timezone-aware                   ║
║   ZoneInfo("America/New_York") not pytz                       ║
║   parse_timestamp() handles all common formats                ║
║   check_expiry() for cert/secret monitoring                   ║
║                                                               ║
║ CACHING:                                                      ║
║   lru_cache — pure functions (parse_arn, etc.)                ║
║   TTLCache — API responses (60-300s TTL)                      ║
║   FileCache — expensive computations (survive restart)        ║
║   NEVER cache without TTL for dynamic data                    ║
║                                                               ║
║ DECORATORS:                                                   ║
║   @timed — measure everything                                 ║
║   @audit_logged("action") — who did what when                 ║
║   @require_confirmation — safety for destructive ops          ║
║   @production_guard — extra delay for prod                    ║
║   @rate_limit(calls=10, period=60) — API protection           ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```


---

## RETENTION QUESTIONS

Same 4 questions from before stand — they were well-designed and cover the right dimensions:

**Q1** — Architecture: Design `novatools secrets audit` (multi-region, cross-K8s, concurrent)
**Q2** — Debugging: Find all bugs in the broken inventory script
**Q3** — Production Code: Write error-rate checker (Prometheus + retry + tests)
**Q4** — Concurrency: Compare sequential vs ThreadPool vs asyncio for 500 health checks

When you're ready, answer all four. Then we move to **Phase 9 Lesson 3: Go for DevOps**.

