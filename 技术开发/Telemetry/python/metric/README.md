````python
#!/usr/bin/env python
# coding: utf-8

import os
import socket
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

from opentelemetry import context
from opentelemetry.exporter.otlp.proto.http.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
    MetricExportResult,
    MetricsData,
    PeriodicExportingMetricReader,
)
from opentelemetry.sdk.resources import HOST_NAME, PROCESS_PID, SERVICE_NAME, Resource

from src.config import CONF, ENV

from .log import logger
from .trace import observe

conf = CONF["telemetry"]["metric"]


def new_meter_provider(
    endpoint: str = None,
    interval: int = 60000,
    timeout: int = 60000,
    parallel_exporter_enabled: bool = False,
) -> MeterProvider:
    resource = Resource(
        attributes={
            SERVICE_NAME: CONF["app_name"],
            HOST_NAME: socket.gethostname(),
            PROCESS_PID: os.getpid(),
            "env": ENV,
        }
    )
    endpoint = endpoint or conf["endpoint"]
    if not parallel_exporter_enabled:
        exporter = OTLPMetricExporter(endpoint=endpoint)
    else:
        exporter = ParallelOTLPMetricExporter(endpoint=endpoint)
    reader = PeriodicExportingMetricReader(exporter, export_interval_millis=interval, export_timeout_millis=timeout)
    provider = MeterProvider(resource=resource, metric_readers=[reader])
    return provider


class ParallelOTLPMetricExporter(OTLPMetricExporter):
    def __init__(self, endpoint: str):
        super().__init__(endpoint, headers={"Content-Type": "application/x-protobuf"})

    def export(self, metrics_data: MetricsData, timeout_millis: float = 10_000, **kwargs) -> MetricExportResult:
        err = self.export_parallel(metrics_data, timeout_millis, **kwargs)
        return MetricExportResult.FAILURE if err else MetricExportResult.SUCCESS

    @observe
    def export_parallel(
        self, metrics_data: MetricsData, timeout_millis: float = 10_000, **kwargs
    ) -> MetricExportResult:
        start = time.perf_counter()
        logger.info("export start")
        ctx = context.get_current()

        try:
            rm = metrics_data.resource_metrics[0]
            sm = rm.scope_metrics[0]
        except Exception as e:
            logger.error(f"no metrics data to export: {repr(e)}")
            return MetricExportResult.FAILURE

        def export(self, ctx, sm_metric, timeout_millis: float, **kwargs) -> MetricExportResult:
            start = time.perf_counter()
            try:
                ctx and context.attach(ctx)
            except Exception as e:
                logger.warning(repr(e))

            name = sm_metric.name
            count = len(sm_metric.data.data_points)

            child_metrics_data = MetricsData(
                resource_metrics=[
                    rm.__class__(
                        resource=rm.resource,
                        scope_metrics=[sm.__class__(scope=sm.scope, metrics=[sm_metric], schema_url=sm.schema_url)],
                        schema_url=rm.schema_url,
                    )
                ]
            )
            ret = super().export(child_metrics_data, timeout_millis, **kwargs)
            duration = time.perf_counter() - start

            if ret == MetricExportResult.FAILURE:
                logger.error(f"export metric failed name:{name} count:{count} duration:{duration}")
                return "export error"
            else:
                if duration > timeout_millis / 2000:
                    logger.warning(f"export metric timeout name:{name} count:{count} duration:{duration}")
                else:
                    logger.info(f"export metric succeeded name:{name} count:{count} duration:{duration}")

            return None

        futures = []

        with ThreadPoolExecutor(thread_name_prefix="MetricParallelExporter", max_workers=os.cpu_count() + 1) as pool:
            for sm_metric in sm.metrics:
                future = pool.submit(export, self, ctx, sm_metric, timeout_millis, **kwargs)
                futures.append(future)

        errs = []

        try:
            for future in as_completed(futures, timeout=timeout_millis / 1000):
                err = future.result()
                if err:
                    errs.append(err)
                    continue
        except TimeoutError as e:
            errs.append(repr(e))

        if errs:
            logger.error(f"export failed with {errs}")
        else:
            duration = time.perf_counter() - start
            if duration > timeout_millis / 1000:
                logger.warning(f"export succeeded with timeout duration:{duration}")
            else:
                logger.info(f"export succeeded duration:{duration}")

        return f"{errs}" if errs else None


meter = new_meter_provider().get_meter(CONF["app_name"])
```