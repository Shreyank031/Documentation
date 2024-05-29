# Grafana Mimir

Grafana Mimir is an open source software project that provides a scalable long-term storage for Prometheus.
It's a distributed Time Series Database(TSDB) 

**Prometheus and Grafana are de-factor standard form metrics monitoring of modern cloud native applications and Infrastructures.**

> Note: Mimir **does not replace** `Prometheus`. It **extends** `Prometheus`

- When using `Mimir`, you keep running `Prometheus` configured to `scrape` the `metrics` form your applications/infrastructure. Then you configure prometheus to `remote write` these metrics to centralized Mimir Cluster[TSDB]

    - You can also use `Grafana Agent` instead of Prometheus to scrape metrics from applicationsa and `remote write` tocentrilized Mimir cluster.
    > **Grafana Agent**: Light-weight version of Prometheus.

- And to `query` back these metrics, you configure `Grafana`.

- Then `Mimir` duplicates received from each prometheus HA(Hight Availability) pair / Grafana Agents. And exposes `100%` compatible prometheus API to query metrics.

## Overview

1. **High Availability**
2. **Horizontal Scalability**
3. **Native multi-tenency**
4. **Durable Storage**
5. **Fast Query Perormance**
6. **Production proven dashboards, Alerts and Runbook**

## Typical Setup

<img width="736" alt="Typical-Setup" src="https://github.com/Shreyank031/Documentation/assets/115367978/6f329a01-cf9c-4bd7-bb02-923b221db27e">

**Note**: Mimir requires a `Object Storage` where we store all the metrics data.
> Object Storage: AWS S3 or S3 compatibe[Minio], GCS, Azure Blob storage, OpenStack Swift.


<img width="654" alt="Object Storage" src="https://github.com/Shreyank031/Documentation/assets/115367978/5afdde9c-f182-4433-a108-c0b785fd0fe9">


## Grafana Mimir architecture

Grafana Mimir has a microservices-based architecture. The system has multiple horizontally scalable microservices that can run separately and in parallel. Grafana Mimir microservices are called components.

- **Grafana Mimir components**:
    1. Compactor
    2. Distributor
    3. Ingester
    4. Querier
    5. Query-frontend
    6. Store-gateway
    7. (Optional) Alertmanager
    8. (Optional) Overrides-exporter
    9. (Optional) Query-scheduler
    10. (Optional) Ruler

### The Write Path

<img width="679" alt="Screenshot 2024-05-29 at 07 32 28" src="https://github.com/Shreyank031/Documentation/assets/115367978/29d1cfbe-a43b-47fe-9a84-379ea75ca62d">

1. **Distributor**:
    - The distributor is a stateless component that receives time-series data from Prometheus or the Grafana agent.

    - The distributor validates the data for correctness and ensures that it is within the configured limits for a given tenant. 

    - The distributor then divides the data into batches and sends it to multiple ingesters in parallel, shards the series among ingesters, and replicates each series by the configured replication factor. By default, the configured replication factor is three.
    
    1. **Validation**:
        - The distributor cleans and validates data that it receives before writing the data to the ingesters. Because a single request can contain valid and invalid metrics, samples, metadata, and exemplars, the distributor only passes valid data to the ingesters.

        - The distributor does not include invalid data in its requests to the ingesters. If the request contains invalid data, the distributor returns a 400 HTTP status code and the details appear in the response body.

        - The details about the first invalid data are typically logged by the sender, be it Prometheus or Grafana Agent.

    2. **Rate limiting**: The distributor includes two different types of rate limiters that apply to each tenant.

        - **Request rate**: The maximum number of requests per second that can be served across Grafana Mimir cluster for each tenant.

        - **Ingestion rate**: The maximum samples per second that can be ingested across Grafana Mimir cluster for each tenant.

        > If any of these rates is exceeded, the distributor drops the request and returns an HTTP 429 response code.

    3. **High-availability tracker**:

        - Remote write senders, such as Prometheus, can be configured in pairs, which means that metrics continue to be scraped and written to Grafana Mimir even when one of the remote write senders is down for maintenance or is unavailable due to a failure. We refer to this configuration as high-availability (HA) pairs.

        - The distributor includes an HA tracker. When the HA tracker is enabled, the distributor deduplicates incoming series from Prometheus HA pairs. 
        - This enables you to have multiple HA replicas of the same Prometheus servers that write the same series to Mimir and then deduplicates the series in the Mimir distributor.

    4. **Sharding and Replication**
        - The distributor shards and replicates incoming series across ingesters. The default replcation count is 3 by. 
        - Distributors use consistent hashing, in conjunction with a configurable replication factor, to determine which ingesters receive a given series.

        - `Sharding` and `replication` uses the ingesters’ hash ring. For each incoming series, the distributor computes a hash using the metric name, labels, and tenant ID.

        - The computed hash is called a token. The distributor looks up the token in the hash ring to determine which ingesters to write a series to.

