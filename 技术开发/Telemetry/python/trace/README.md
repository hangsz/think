```python
#!/usr/bin/env python
# coding: utf-8

import asyncio
import inspect
import os
import socket
import traceback
from functools import wraps

from opentelemetry import context, trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import HOST_NAME, PROCESS_PID, SERVICE_NAME, Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.trace import StatusCode

from src.config import CONF, ENV

from .log import logger

conf = CONF["telemetry"]["trace"]


def new_trace_provider(endpoint: str = None) -> TracerProvider:
    resource = Resource(
        attributes={
            SERVICE_NAME: CONF["app_name"],
            HOST_NAME: socket.gethostname(),
            PROCESS_PID: os.getpid(),
            "env": ENV,
        }
    )
    endpoint = endpoint or conf["endpoint"]
    exporter = OTLPSpanExporter(endpoint=endpoint, insecure=True)
    processor = BatchSpanProcessor(exporter)
    provider = TracerProvider(resource=resource, active_span_processor=processor)
    return provider


tracer = new_trace_provider().get_tracer(CONF["app_name"])


def get_trace_id() -> str:
    span = trace.get_current_span()
    trace_id = trace.format_trace_id(span.get_span_context().trace_id)
    return trace_id


def get_span_id() -> str:
    span = trace.get_current_span()
    span_id = trace.format_span_id(span.get_span_context().span_id)
    return span_id


def as_span(fn):
    @wraps(fn)
    def sync_wrapper(*args, **kwargs):
        ctx = context.get_current()
        fn_name = f"{fn.__module__}.{fn.__name__}"
        with tracer.start_as_current_span(fn_name, context=ctx) as span:
            span.set_attributes({"fn": fn_name})
            logger.info(f"{fn_name} start...")
            err = None
            try:
                return fn(*args, **kwargs)
            except Exception as e:
                logger.exception(traceback.format_exc())
                span.set_status(StatusCode.ERROR, repr(err))
                raise RuntimeError from e
            finally:
                logger.info(f"{fn_name} end.")

    @wraps(fn)
    async def async_wrapper(*args, **kwargs):
        ctx = context.get_current()
        fn_name = f"{fn.__module__}.{fn.__name__}"
        with tracer.start_as_current_span(fn_name, context=ctx) as span:
            span.set_attributes({"fn": fn_name})
            logger.info(f"{fn_name} start...")
            try:
                return await fn(*args, **kwargs)
            except Exception as e:
                logger.exception(traceback.format_exc())
                span.set_status(StatusCode.ERROR, repr(e))
                raise RuntimeError from e
            finally:
                logger.info(f"{fn_name} end.")

    if not inspect.iscoroutinefunction(fn):
        return sync_wrapper
    else:
        return async_wrapper

```