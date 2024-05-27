# Prometheus Architecture

Prometheus is a robust monitoring and alerting toolkit designed for reliability and performance.

### High level overview of Prometheus Architecture

![prometheus-server-full-fledged](https://github.com/Shreyank031/prometheus/assets/115367978/2684bb84-b03e-4613-bd53-bc82751a2355)

## Prometheus primarily consists of the following.

- Prometheus Server
- Service Discovery
- Time-Series Database (TSDB)
- Targets
- Exporters
- Push Gateway
- Alert Manager
- Client Libraries
- PromQL

## Prometheus Server:

The Prometheus server is the central component that scrapes and stores time-series data.[Brain of metric-based monittoring system]
The main job of the server is to collect the metrics from various targets using pull model.

![prom-server](https://github.com/Shreyank031/prometheus/assets/115367978/282591cf-0b9d-4f75-8d44-c8ec5b2dc713)

Prometheus periodically scrapes the metrics, based on the `scrape interval`. 

```yaml
global:
  scrape_interval: 15s  # The default frequency at which metrics are scraped from targets.
  evaluation_interval: 15s  # The default frequency at which rules are evaluated.
  scrape_timeout: 10s  # The default timeout for scraping metrics from targets.

rule_files:
  - "rules/*.rules"  # Specifies the location of rule files for recording and alerting rules.

scrape_configs:
  - job_name: 'prometheus'  # Defines a scrape job named 'prometheus'.
    static_configs:
      - targets: ['localhost:9090']  # Targets to scrape metrics from; here, Prometheus itself.

  - job_name: 'node-exporter'  # Defines a scrape job named 'node-exporter'.
    static_configs:
      - targets: ['node-exporter:9100']  # Targets to scrape metrics from; here, the Node Exporter.

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']  # Alertmanager instances to send alerts to; here, Alertmanager.
```

## Time-Series Database (TSDB)

1. The metric data which prometheus receives changes over time (CPU, memory, network IO etc..). It is called time-series data. So Prometheus uses a Time Series Database (TSDB) to store all its data.

2. By default Prometheus stores all its data in an `efficient format (chunks)` in the local disk. Overtime, it `compacts all the old` data to save space. It also has retention policy to get rid of old data.

3. The TSDB has inbuilt mechanisms to manage data kept for long time. You can choose any of the following data retention policies.
  - `Time based retention`: Data will be kept for the specified days. The default retention is 15 days.

  - `Size-based retention`: You can specify the maximum data TSDB can hold. Once this limit it reached, prometheus will free up the space to accommodate new data. 

4. Prometheus also offers remote storage options. This is primarily required for storage scalability, long-term storage, backup & disaster recovery etc.

## Scraping Targets

These are endpoints that expose metrics, which Prometheus scrapes.

![targt](https://github.com/Shreyank031/prometheus/assets/115367978/b75b97dc-03c5-4fb6-9482-2e9fee4cc130)

By default prometheus looks for metrics under `/metrics` path of the target. The default path can be changed in the target configuration. This means, if you dont specify a custom metric path, Prometheus looks for the metrics under `/metrics`.

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'node-exporter1:9100'  # The first Node Exporter instance.
          - 'node-exporter2:9100'  # The second Node Exporter instance.
```

- From the target endpoints, prometheus expects data in certain text format. Each metric has to be on a new line.

- Usually these metrics are exposed on target nodes using prometheus exporters running on the targets. 

## Service Discovery

Prometheus can automatically discover targets using two methods to scrape metrics

- **Static configs**: When the targets have a static IP or DNS endpoint, we can use those endpoints as targets.

- **Sevice Discovery**: In most autoscaling systems and distributed systems like Kubernetes, the target will not have a static endpoint. In this case, that target endpoints are discovered using prometheus service discovery and targets are added automatically to the prometheus configuration.

Kubernetes is the perfect example for dynamic targets. Here, you cannot use the static targets method, because targets (pods) in a Kubernetes cluster is ephemeral in nature and could be short lived.

## Prometheus Exporters

- Prometheus exporters are components that translate metrics from various systems, applications, or services into a format that Prometheus can scrape.

- It could be system metrics like CPU, memory etc, or Java JMX metrics, MySQL metrics etc.

![exporters](https://github.com/Shreyank031/prometheus/assets/115367978/6045996e-12ab-4946-bf4a-7ca97537dfd1)

- By default these converted metrics are exposed by the exporter on /metrics path(HTTPS endpoint) of the target.

- For example, if you want to monitor a servers CPU and memory, you need to install a node exporter on that server and the node exporter exposes the CPU and memory metrics in the prometheus metrics format on /metrics.

- Once the Prometheus pulls the metrics, it will then combine the metric name, labels, value, and timestamp to give a structure to that data.

-  Here are some common Prometheus exporters and what they are used for:
  1. **Node Exporter**: Collects hardware and OS metrics[ CPU, memory, disk usage, network statistics, etc.] from the system.
  2. **cAdvisor**: Collects `container` metrics[Resource usage statistics for containers, including CPU, memory, and network usage.].
  3. **Pushgateway**: Used for metrics that cannot be scraped (e.g., batch jobs). Metrics are pushed to Pushgateway, which Prometheus scrapes.

```yaml
scrape_configs:
  - job_name: 'node-exporter'  # Defines a scrape job named 'node-exporter'.
    static_configs:
      - targets:  # List of targets to scrape metrics from.
          - 'node-exporter1:9100'  # The first Node Exporter instance running on node-exporter1 at port 9100.
          - 'node-exporter2:9100'  # The second Node Exporter instance running on node-exporter2 at port 9100.

  - job_name: 'blackbox-exporter'  # Defines a scrape job named 'blackbox-exporter'.
    static_configs:
      - targets:  # List of targets to scrape metrics from.
          - 'blackbox-exporter1:9115'  # The first Blackbox Exporter instance running on blackbox-exporter1 at port 9115.
          - 'blackbox-exporter2:9115'  # The second Blackbox Exporter instance running on blackbox-exporter2 at port 9115.
    metrics_path: /probe  # The HTTP path to query for metrics; Blackbox Exporter uses /probe.

  - job_name: 'snmp-exporter'  # Defines a scrape job named 'snmp-exporter'.
    static_configs:
      - targets:  # List of targets to scrape metrics from.
          - 'snmp-exporter1:9116'  # The first SNMP Exporter instance running on snmp-exporter1 at port 9116.
          - 'snmp-exporter2:9116'  # The second SNMP Exporter instance running on snmp-exporter2 at port 9116.
    metrics_path: /snmp  # The HTTP path to query for metrics; SNMP Exporter uses /snmp.

  - job_name: 'pushgateway'
    honor_labels: true  # Keeps the original labels from the pushed metrics.
    static_configs:
      - targets: ['pushgateway:9091']
```

## Prometheus Pushgateway

- Prometheus by default uses pull mechanism to scrap the metrics.

- However, there are scenarios where metrics need to be pushed to prometheus.

- Lets take an example of a batch job running on a Kubernetes cronjob that runs daily for 5 mins based on certain events. In this scenario, Prometheus will not be able to scrape the service level metrics properly using pull mechanism.

- So instead for waiting for prometheus to pull the metrics, we need to push the metrics to prometheus. To push metrics, prometheus offers a solution called Pushgateway. It is kind of a intermediate gateway.

- Pushgateway needs to be run as a standalone component. The batch jobs can push the metrics to the pushgateway using HTTP API. Then Pushgateway exposes those metrics on /metrics endpoint. Then prometheus scrapes those metrics from the Pushgateway. 

![pushGateway](https://github.com/Shreyank031/prometheus/assets/115367978/bbaad5db-bb36-4e60-a8cb-0edceb6b8d66)

> Pushgateway stores the metrics data temporarily in in-memory storage. It’s more of a temporary cache. 

```yaml
scrape_configs:
  - job_name: "pushgateway"
        honor_labels: true
        static_configs:
        - targets: [pushgateway.monitoring.svc:9091]
```

## Prometheus Client Libraries

- To send metrics to the Pushgateways, you need to use the prometheus Client Libraries and instrument the application or script to expose the required metrics.

- Prometheus Client Libraries are software libraries that can be used to instrument application code to expose metrics in the way Prometheus understands. 

- A very good use case is batch jobs that need to push metrics to the Pushgateway. The batch job needs to be instrumented with client libraries to expose requirement metrics in prometheus format

- When using client libraries, prometheus_client HTTP server exposes the metrics in /metrics endpoint.

- Prometheus has `client libraries` for almost every programming languages.

## Prometheus Alert Manager

- Alertmanager is the key part of Prometheus monitoring system. It handles alerts generated by Prometheus based on alerting rules.

- The alert get triggered by Prometheus and sent to Alertmanager. It in turn sends the alerts to the respective notification systems/receivers (email, slack etc) configured in the alert manager configurations.

- Alert manager takes care of the following.

  1. **Alert Deduplicating**: Process of silencing duplicated alerts.
  2. **Grouping**: Process of grouping related alerts togther.
  3. **Silencing**: Silence alerts for maintenance or false positives.
  4. **Routing**: Routing alerts to appropriate receivers based on severities.
  5. **Inhibition**: Process of stopping low severity alert when there is a medium of high severity alert.

![alert-manager](https://github.com/Shreyank031/prometheus/assets/115367978/e3df5f08-3ccd-4fd2-b83f-4210016838da)

## PromQL (Prometheus Query Language)

- A powerful query language for retrieving and manipulating time-series data.
- We can directly used the queries from the Prometheus user interface or we can use curl command to make a query over the command line interface

