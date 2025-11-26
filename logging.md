Example of `app/core/logging.py` file:

```
import logging
import sys
import structlog
from app.core.config import settings


def setup_logging():
    """Configure structured logging for the application."""

    # Configure standard logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=settings.LOG_LEVEL.upper(),
    )

    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.dev.set_exc_info,
        structlog.processors.TimeStamper(fmt="iso"),
    ]

    if sys.stderr.isatty():
        # Pretty printing for development
        processors = shared_processors + [
            structlog.dev.ConsoleRenderer(),
        ]
    else:
        # JSON for production
        processors = shared_processors + [
            structlog.processors.JSONRenderer(),
        ]

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(logging.getLogger().level),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )


logger = structlog.get_logger()
```
This file configures Structured Logging for your application using the structlog library.

1. **Sets up Standard Logging:** It initializes Python's built-in 
logging module to output to `stdout` with the log level defined in your settings (e.g., INFO, DEBUG).
2. **Configures Structlog:** It sets up a processing pipeline for your logs:
Timestamps: Adds an ISO timestamp to every log.
  **Log Level:** Adds the severity (INFO, ERROR, etc.).
  **Context:** Allows you to bind variables (like `user_id` or `request_id`) to all subsequent logs.
  Smart Formatting:
    - Development: If you are running locally (detected via `sys.stderr.isatty()`), it prints colorful, readable logs.
    - Production: If running on a server, it outputs **JSON**, which is perfect for log aggregators (Datadog, CloudWatch, etc.).
**How to Use It**
You don't need to call `setup_logging()` manually; it's already called in `app/main.py`
 when the app starts.

To use the logger in your code:

1. Import it:
```python
import structlog
logger = structlog.get_logger()
```
2. Log messages:
```python
logger.info("User logged in")
logger.error("Database connection failed")
```
3. Add Context (The Power of Structured Logging): Instead of formatting strings like f"User {user_id} failed", pass them as keyword arguments:
```python
# BAD
logger.info(f"Processing order {order_id} for user {user_id}")

# GOOD
logger.info("processing_order", order_id=order_id, user_id=user_id, status="pending")
```
Why is this better? In production, this outputs JSON:
```json
{"event": "processing_order", "order_id": 123, "user_id": 456, "status": "pending", "timestamp": "..."}
```
You can then search your logs for order_id=123 instantly.
