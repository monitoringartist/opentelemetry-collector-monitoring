# OpenTelemetry (OTEL) collector monitoring

## Metric name/label name conventions

OpenTelemetry metrics and label names can vary significantly. These variations depend on several factors:

- OpenTelemetry Collector version
- Specific receivers/exporters in use
- Receiver/exporter configuration settings
- Metric storage backend configuration

For example uptime metric can be saved as:
```
otelcol_process_uptime{service_instance_id="123456", ...}
otelcol_process_uptime_total{service_instance_id="123456", ...}
otelcol_process_uptime_seconds_total{service_instance_id="123456", ...}
otelcol_process_uptime_seconds_total{service.instance.id="123456", ...}
otelcol_process_uptime_seconds_total_total{service.instance.id="123456", ...}
```

This dashboard attempts to detect all these variations through hidden dashboard variables 
to ensure compatibility across different configurations.

## Metrics

Collector can expose Prometheus metrics locally on port 8888 and path 
`/metrics`. For containerized environments it may be desirable to expose this 
port on a public interface instead of just locally.

```
receivers:
  prometheus:
    config:
      scrape_configs:
      - job_name: otel-collector-metrics
        scrape_interval: 10s
        static_configs:
        - targets: ['127.0.0.1:8888']

service:
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: []
      exporters: [...]
  telemetry:
    resource:
      service.name: grafana-opentelemetry/metrics
    metrics:
      level: detailed
      readers:
        - pull:
            exporter:
              prometheus:
                host: 127.0.0.1
                port: 8888
                with_resource_constant_labels:
                  included:
                      - service.name
```

Collector can scrape own metric via own metric pipeline, so real configuration 
can looks like:
```
extensions:
  sigv4auth/aws:

receivers:
  prometheus:
    config:
      scrape_configs:
      - job_name: otel-collector-metrics
        scrape_interval: 10s
        static_configs:
          - targets: ['127.0.0.1:8888']

exporters:
  prometheusremotewrite/aws:
    endpoint: ${PROMETHEUS_ENDPOINT}
    auth:
      authenticator: sigv4auth/aws
    retry_on_failure:
      enabled: true
      initial_interval: 1s
      max_interval: 10s
      max_elapsed_time: 30s

service:
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: []
      exporters: [prometheusremotewrite/aws]
  telemetry:
    resource:
      service.name: grafana-opentelemetry/metrics
    metrics:
      level: detailed
      readers:
        - pull:
            exporter:
              prometheus:
                host: 127.0.0.1
                port: 8888
                with_resource_constant_labels:
                  included:
                      - service.name
```

## Grafana dashboard for OpenTelemetry collector metrics

[![OpenTelemetry collector dashboard](dashboard/opentelemetry-collector-dashboard.png)](https://github.com/monitoringartist/opentelemetry-collector-monitoring/tree/main/dashboard)

This dashboard can also be used for [Grafana Alloy monitoring](doc/grafana-alloy-monitoring.md).

## Prometheus alerts

Recommended Prometheus alerts for OpenTelemetry collector metrics:
```
# keep in mind that metrics may have "_total|seconds_total|seconds_total_total" suffixes - check your metrics/configuration first
groups:
  - name: opentelemetry-collector
    rules:
      - alert: processor-dropped-spans
        expr: sum(rate(otelcol_processor_dropped_spans{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some spans have been dropped by processor
          description: Maybe collector has received non standard spans or it reached some limits
      - alert: processor-dropped-metrics
        expr: sum(rate(otelcol_processor_dropped_metric_points{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some metric points have been dropped by processor
          description: Maybe collector has received non standard metric points or it reached some limits
      - alert: processor-dropped-logs
        expr: sum(rate(otelcol_processor_dropped_log_records{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some log records have been dropped by processor
          description: Maybe collector has received non standard log records or it reached some limits
      - alert: receiver-refused-spans
        expr: sum(rate(otelcol_receiver_refused_spans{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some spans have been refused by receiver
          description: Maybe collector has received non standard spans or it reached some limits
      - alert: receiver-refused-metrics
        expr: sum(rate(otelcol_receiver_refused_metric_points{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some metric points have been refused by receiver
          description: Maybe collector has received non standard metric points or it reached some limits
      - alert: receiver-refused-logs
        expr: sum(rate(otelcol_receiver_refused_log_records{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some log records have been refused by receiver
          description: Maybe collector has received non standard log records or it reached some limits
      - alert: exporter-enqueued-spans
        expr: sum(rate(otelcol_exporter_enqueue_failed_spans{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some spans have been enqueued by exporter
          description: Maybe used destination has a problem or used payload is not correct
      - alert: exporter-enqueued-metrics
        expr: sum(rate(otelcol_exporter_enqueue_failed_metric_points{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some metric points have been enqueued by exporter
          description: Maybe used destination has a problem or used payload is not correct
      - alert: exporter-enqueued-logs
        expr: sum(rate(otelcol_exporter_enqueue_failed_log_records{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some log records have been enqueued by exporter
          description: Maybe used destination has a problem or used payload is not correct
      - alert: exporter-failed-requests
        expr: sum(rate(otelcol_exporter_send_failed_requests{}[1m])) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Some exporter requests failed
          description: Maybe used destination has a problem or used payload is not correct
      - alert: high-cpu-usage
        expr: max(rate(otelcol_process_cpu_seconds{}[1m])*100) > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: High max CPU usage
          description: Collector needs to scale up
```

Example of alert rules provided as a opentelemetry collector part of helm chart: https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-collector/templates/prometheusrule.yaml

## Documentation

- https://opentelemetry.io/docs/collector/internal-telemetry/
- https://opentelemetry.io/docs/collector/troubleshooting/
