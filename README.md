# Prometheus

Prometheus is a software application used for event monitoring and alerting. It is designed for reliability and scalability, making it a popular choice for monitoring cloud-native applications and infrastructure.

## Overview

- **Metrics Collection**: It records real-time `metrics` and stores them in a `time series database`.
- **Pull Model**: Prometheus is built using an `HTTP pull` model, where it periodically scrapes metrics from `instrumented targets`.
- **Flexible Querying**: It supports flexible queries using its powerful query language, PromQL.
- **Real-Time Alerting**: Prometheus enables real-time alerting based on the data it collects.
- **Key-Value Format**: Metrics are collected in a `key-value` format, stored as time series data.
- **Scraping Intervals**: At regular intervals, Prometheus scrapes `targets` to collect `metrics`, `aggregate data`, and trigger alerts if thresholds are met.
- **User Interface**: Prometheus includes a basic GUI for querying and visualizing data, although it is often complemented with Grafana for advanced visualization.

## Why Use Prometheus?

- **DevOps Monitoring**: It excels at monitoring cloud-native applications, infrastructure (physical or virtual machines), and hundreds of microservices.
- **Time-Series Data**: Prometheus is optimized for recording purely time-series data.
- **Versatility**: It fits both machine-centric (servers, VMs) monitoring and highly dynamic, service-oriented architectures (modern microservices).

## Key featers:

- **Service Discovery**: Prometheus supports automatic service discovery to dynamically detect and scrap metrics from services as they scale up or down(eg., K8s)

- **Label based filtering**: Prometheus uses `labels` to provide `context` to `metrics`, allowing detailed and flexible quering in dynamic environment.

- **Standalone**: Each Prometheus server is `Standalone`. Not depending of network storage.

-  **Alertmanager Integration**: Integrates with Alertmanager to handle alerts, including silencing, inhibition, and routin.

- **Exporters**: A wide range of exporters are available to expose metrics from various systems and third-party services.

- **Custom Instrumentation**: Client libraries for multiple languages allow custom instrumentation of your applications.

### Standalone feature:

- **Each Prometheus Server operates independently**

    - **Independent Data Store**: Each Prometheus server has it's own local time-series data base where it stores the metrics data it collected. There is no built in mechanism for sharing data between instances.
    - **Isolation**: Prometheus servers do not inheritly communicate with each other or share their data. This isolation ensures simplicity and reliability, as each instance is responsible only for its own data.

## Basic Concepts

- **Metrics**: Data points that represent a particular characteristic of a system (e.g., CPU or memory usage, request duration).

- **Targets**: The endpoints from which Prometheus scrapes metrics (e.g., applications, services).

- **Jobs**: Groups of targets with the same scrape configuration.

- **Time Series**: A series of data points indexed in time order, typically comprising metric data and labels.

## Pull based tool:

It means Prometheus actively `scraps(pulls)` the metrics from target at regular interval of time.

## Pull based Model:

1. **Scraping Targets**: Prometheus periodically makes `HTTP` requests to a set of configured endpoints(targets) to fetch the current state of metrics
2. **Endpoint Exposure**: `Targets` exposes their `metrics` via a specific `HTTP endpoint(/metrics)`.

    - Applications and services need to setup[instrumented] to expose their metrics data in a specific format.

    - **Metric Endpoints**: Applicatins must expose their metrics on a specific `HTTP endpoint(/metrics)`. This endpoint provides the current state of metrics in a format that Prometheus can read.
    - **Prometheus Client Library**: With respect to above point, Pormetheus provides client libraries for various programming languages[Go,Java,Python,etc.,]. These libraries facilitate the process of collecting and exposing metrics.
    - **Metrics formate**: Metrics exposed on `/metrics` endpoint needs to be in a `simple text-based exposition format` that prometheus understands. The formate usually consists of: 
        - **Metrics Name**: Eg., `http_request_total`
        - **Labels**: Key-value pair providing additional context to metrics. Eg., `{method="GET", handler=/home}`
        - Metrics value: The actual numerical value of the metrics. Eg., 31 which is no of http request.
    - Once the application is `instrumented` and metrics endpoint is exposed, Prometheus can be configured to scrap this endpoint at regular interval to collect the metrics data.

3. Service Discovery: Prometheus supports dynamic service discovery, allowing it to automatically detect and scrap new targets as they come online
    - This is especially used in k8s, where services are frequently added or removed.

## Advantages of Pull based Model:

1. **Controle over Scraping**: Prometheus has control over when and how often it scraps the target.
2. **Fault isolation**  If the target fails to provide metrics, it does not affect the prometheus ability to scrap other targets/endpoints.
3. **Flexibility**: User can easily adjust the scraping frequency for different targets. Eg., More ciritical servers ca be scraped for more frequently.
4. **Security**: By default, only prometheus server initiates connection to the targets. This can simplifiy the firewall and security configuration because only prometheus server needs to be allowed to make outbound requests to the targets.

