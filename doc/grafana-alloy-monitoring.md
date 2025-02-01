# Alloy monitoring

[Grafana Alloy](https://grafana.com/docs/alloy/latest/[]) is a vendor-neutral distribution of the OpenTelemetry (OTel) Collector. 
It resuses many OTEL components, so it generates metrics similar to OpenTelemetry collector.
You can transform metrics for partial OpenTelemetry collector format compatibility and then
you can use the same metric dashboard for Alloy and OpenTelemetry collector deployments.

This is an example, not copy&paste setup, which will work for all use cases - customize it for your needs:

## Environment variables

Add environment variables, which you use in the config. Example for K8S helm chart (tested/used with [k8s-monitoring-helm 2.0.4](https://github.com/grafana/k8s-monitoring-helm)):

```
    extraEnv:
      - name: OTLP_ENDPOINT
        value: "https://my-otlp-grpc-collector.com:4317"
      - name: OTLP_USERNAME
        value: "my-username"
      - name: OTLP_PASSWORD
        value: "my-password"
      - name: POD_UID
        valueFrom:
          fieldRef:
            fieldPath: metadata.uid
```


## Alloy configuration
```
    // selfmetrics pipeline transforms the Alloy metrics to be partially compatible 
    # with the OpenTelemetry Collector metrics: https://grafana.com/grafana/dashboards/15983-opentelemetry-collector/
    
    // Exposes Alloy's internal metrics for scraping
    prometheus.exporter.self "selfmetrics" {
    }

    // Relabels the targets to add service.instance.id label from POD_UID environment variable
    discovery.relabel "selfmetrics" {
      targets = prometheus.exporter.self.selfmetrics.targets

      rule {
        action = "replace"
        target_label = "service_instance_id"
        replacement = sys.env("POD_UID")
      }

      rule {
        action = "replace"
        target_label = "job"
        replacement = "integrations/self/alloy"
      }
    }

    // Scrapes the exposed metrics from the self exporter
    prometheus.scrape "selfmetrics" {
      targets         = discovery.relabel.selfmetrics.output
      forward_to      = [otelcol.receiver.prometheus.selfmetrics.receiver]
      scrape_interval = "10s"  // Scrape metrics every 10 seconds
      scrape_timeout  = "9s"   // Timeout for each scrape after 9 seconds
    }

    // Receives the scraped Prometheus metrics and converts them to OpenTelemetry format
    otelcol.receiver.prometheus "selfmetrics" {
      output {
        metrics = [otelcol.processor.transform.selfmetrics.input]
      }
    }

    // Processes and transforms the received metrics
    otelcol.processor.transform "selfmetrics" {
      metric_statements {
        context = "datapoint"
        statements = [
          // Rename the up metric to otelcol_process_uptime - not correct, but required for correct suffix detection
          `set(metric.name, "otelcol_process_uptime_total") where metric.name == "up"`,
          // Rename the alloy_resources_process_resident_memory_bytes metric to otelcol_process_memory_rss
          `set(metric.name, "otelcol_process_memory_rss") where metric.name == "alloy_resources_process_resident_memory_bytes"`,
          // Rename the alloy_resources_process_cpu_seconds_total metric to otelcol_process_cpu_seconds_total
          `set(metric.name, "otelcol_process_cpu_seconds_total") where metric.name == "alloy_resources_process_cpu_seconds_total"`,
          // set otel collector receiver label
          `merge_maps(attributes, ExtractPatterns(attributes["component_id"], ".+\\.(?P<receiver>\\w+\\.\\w+)"), "upsert") where attributes["component_id"] != nil and IsMatch(metric.name, "otelcol_receiver_.*")`,
          // set otel collector processor label
          `merge_maps(attributes, ExtractPatterns(attributes["component_id"], ".+\\.(?P<processor>\\w+\\.\\w+)"), "upsert") where attributes["component_id"] != nil and IsMatch(metric.name, "otelcol_processor_.*")`,
          // set otel collector exporter label
          `merge_maps(attributes, ExtractPatterns(attributes["component_id"], ".+\\.(?P<exporter>\\w+\\.\\w+)"), "upsert") where attributes["component_id"] != nil and IsMatch(metric.name, "otelcol_exporter_.*")`,
        ]
      }

      output {
        metrics = [otelcol.processor.batch.selfmetrics.input]
      }
    }

    otelcol.processor.batch "selfmetrics" {
      send_batch_size = 1000
      send_batch_max_size = 1000
      timeout = "200ms"

      output {
        metrics = [otelcol.exporter.otlp.selfmetrics.input]
      }
    }

    otelcol.exporter.otlp "selfmetrics" {
      client {
        endpoint = sys.env("OTLP_ENDPOINT")
        auth     = otelcol.auth.basic.otlp_auth.handler
      }

      sending_queue {
        num_consumers = 30
        queue_size = 1000
      }
    }

    otelcol.auth.basic "otlp_auth" {
      username = sys.env("OTLP_USERNAME")
      password = sys.env("OTLP_PASSWORD")
    }
  ```
