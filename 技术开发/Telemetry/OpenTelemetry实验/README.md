# 环境准备

## Docker

- [https://www.docker.com/](https://www.docker.com/)

## Python

- otel相关package安装

```
python -m pip install opentelemetry-api
python -m pip install opentelemetry-sdk

# exporter安装 
# https://github.com/open-telemetry/opentelemetry-python/tree/main/exporter
# https://github.com/open-telemetry/opentelemetry-python-contrib/tree/main/exporter
pip install opentelemetry-exporter-{exporter}

# 自动注入安装
# pip install opentelemetry-instrumentation-{instrumentation}
```

# 数据生产

## 手动埋点

```
from random import randint

from flask import Flask
from opentelemetry import trace
from opentelemetry import metrics

from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.sdk.trace import ReadableSpan, TracerProvider
from opentelemetry.sdk.trace.export import (
BatchSpanProcessor,
ConsoleSpanExporter,
SimpleSpanProcessor,
SpanExportResult,
)
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter


from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
ConsoleMetricExporter,
PeriodicExportingMetricReader,
)
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# Service name is required for most backends
resource = Resource(attributes={
    SERVICE_NAME: "your-service-name"
})

# 输出到控制台
# processor = SimpleSpanProcessor(ConsoleSpanExporter())
# 输出到collector
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="127.0.0.1:4317",insecure=True))
trace_provider = TracerProvider(
    resource=resource,
    active_span_processor=processor)

trace.set_tracer_provider(trace_provider)

# 输出到控制台
# reader = PeriodicExportingMetricReader(ConsoleMetricExporter())
# 输出到collector
reader = PeriodicExportingMetricReader(
    OTLPMetricExporter(endpoint="127.0.0.1:4317", insecure=True)
)
metric_provider = MeterProvider(metric_readers=[reader], resource=resource)
metrics.set_meter_provider(metric_provider)

# Acquire a tracer
tracer = trace.get_tracer("diceroller.tracer")
meter = metrics.get_meter("diceroller.meter")

roll_counter = meter.create_counter(
    "roll_counter",
    description="The number of rolls by roll value",
)
roll_counter2 = meter.create_counter(
    "roll_counter2",
    description="The number of rolls by roll value",
)

app = Flask(__name__)
@app.route("/rolldice")
def roll_dice():
    return str(do_roll())


def do_roll():
    with tracer.start_as_current_span("do_roll") as roll_span:
        res = randint(1, 6)
        roll_span.set_attribute("roll.value", res)
        roll_counter.add(1, {"roll.value": res})
        roll_counter2.add(2, {"roll.value": res})
        return res


if __name__ == "__main__":
    app.run(debug=True, port=8080)
```

## 启动

```
python3 -m app
```

## 查看

- [http://127.0.0.1:8080/rolldice](http://127.0.0.1:8080/rolldice)

# 数据采集 - Otel Collector

## 配置

```
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch:
exporters:
  alibabacloud_logservice/traces:
    endpoint: "cn-test-ant-eu95-share.log.aliyuncs.com"
    project: "ant-tnt-infradata-dev"
    logstore: "ods_tnt_infra_metric"
    access_key_id: "xxx"
    access_key_secret: "xxx"
  alibabacloud_logservice/metrics:
    endpoint: "cn-test-ant-eu95-share.log.aliyuncs.com"
    project: "ant-tnt-infradata-dev"
    logstore: "ods_tnt_infra_metric"
    access_key_id: "xxx"
    access_key_secret: "xxx"
  logging:
    loglevel: debug

    # Data sources: metrics
    prometheus:
    # 拉模式，启动时要把port映射到主机
    endpoint: 0.0.0.0:8889
    namespace: "default"
    send_timestamps: true
  
  jaeger:
    # 推模式，配置jaeger暴露ip和port
    endpoint: 30.230.56.186:14250
    tls:
      insecure: true

  prometheusremotewrite:
    endpoint: 'http://0.0.0.0:9090/api/v1/write'
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [alibabacloud_logservice/traces, logging]
      
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus,logging]
      
```

## 启动

```
docker run \
    -p 4317:4317 \
    -p 8889:8889 \
    -v $(pwd)/otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml \
    otel/opentelemetry-collector-contrib:latest
```

注意：collector这里使用是 -contrib版

## 查看

如果exporter中使用了 logging，可以在docker页面查看 trace/metric

# Metric存储与查询 - Prometheus

## 配置

```
scrape_configs:
  - job_name: 'otel-python-demo'
    scrape_interval: 5s
    scheme: http
    static_configs:
      - targets: ["30.230.56.186:8889"]

    tls_config:
      insecure_skip_verify: true
  
  #prometheus自身metric: 0.0.0.0:9090/metrics
  - job_name: 'prometheus'
    # 覆盖全局默认的参数，并将采样时间间隔设置为 5s
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

## 启动

```
docker run -p 9090:9090 \
    -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```

注意：

- 默认端口为9090

## 查看

- 控制台：[http://0.0.0.0:9090](http://0.0.0.0:9090)

# Trace存储与查询 - Jaeger

## 配置

## 启动

```
docker run \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 14250:14250 \
  jaegertracing/all-in-one:latest
```

## 查看

- 控制台： [http://0.0.0.0:16686/](http://0.0.0.0:16686/)