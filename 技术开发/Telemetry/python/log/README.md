```python

#!/usr/bin/env python
# coding: utf-8

import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path

from opentelemetry import trace

from src.config import CONF

conf = CONF["telemetry"]["log"]

LEVEL_MAP = {
    "debug": logging.DEBUG,
    "info": logging.INFO,
    "warning": logging.WARNING,
    "error": logging.ERROR,
    "critical": logging.CRITICAL,
}
level = LEVEL_MAP.get(conf["level"].lower(), logging.INFO)

Path(conf["filename"]).parent.mkdir(parents=True, exist_ok=True)
file_handler = RotatingFileHandler(
    conf["filename"],
    maxBytes=10 * 1024 * 1024,
    backupCount=5,
)
file_handler.setLevel(level)


class TraceFormatter(logging.Formatter):
    def format(self, record):
        ctx = trace.get_current_span().get_span_context()
        if ctx.is_valid:
            record.trace_id = trace.format_trace_id(ctx.trace_id)
            record.span_id = trace.format_span_id(ctx.span_id)
        else:
            record.trace_id = "0" * 32
            record.span_id = "0" * 16
        return super().format(record)


FORMAT = "%(asctime)s %(levelname)s [%(filename)s %(lineno)d] [%(trace_id)s %(span_id)s] %(message)s"
file_handler.setFormatter(TraceFormatter(FORMAT))

logger = logging.getLogger(CONF["app_name"])
logger.addHandler(file_handler)
logger.propagate = False

```