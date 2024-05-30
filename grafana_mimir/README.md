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

    4. **Sharding and Replication**:
        - The distributor shards and replicates incoming series across ingesters. The default replcation count is 3 by. 
        - Distributors use consistent hashing, in conjunction with a configurable replication factor, to determine which ingesters receive a given series.

        - `Sharding` and `replication` uses the ingesters’ hash ring. For each incoming series, the distributor computes a hash using the metric name, labels, and tenant ID.

        - The computed hash is called a token. The distributor looks up the token in the hash ring to determine which ingesters to write a series to.

    5. **Quorum consistency**:

        - Because distributors `share access` to the **same hash ring**, write requests can be sent to any distributor. 
        - You can also set up a **stateless load balancer** in front of it.

        - To ensure consistent query results, Mimir uses **Dynamo-style quorum consistency** on reads and writes. 
        - The distributor waits for a successful response from `n/2 + 1` ingesters, where n is the configured replication factor, before sending a successful response to the Prometheus write request.

    6. **Load balancing across distributors**:
        - Grafana Team recommend **randomly load balancing write requests** across distributor instances. If you’re running Grafana Mimir in a **Kubernetes cluster**, you can define a **Kubernetes Service** as **ingress** for the distributors.

2. **Ingester**: The ingester is a `stateful component` that writes incoming series to `long-term storage` on the **write path** and returns series samples for `queries` on the **read path**.

    - Ingesters receive incoming samples from the distributors. Each push request belongs to a tenant, and the ingester appends the received samples to the specific per-tenant TSDB that is stored on the local disk. 

    - The samples that are received are both kept in-memory and written to a `write-ahead log (WAL)`. If the ingester abruptly terminates, the WAL can help to recover the **in-memory series**. 

    - The per-tenant TSDB is lazily created in each ingester as soon as the first samples are received for that tenant.

    - The **in-memory** samples are periodically flushed to disk, and the **WAL** is truncated, when a new TSDB block is created. By default, this occurs every two hours. 

    - Each newly created block is uploaded to `long-term storage` and kept in the ingester until the configured `-blocks-storage.tsdb.retention-period` expires. This gives queriers and **store-gateways** enough time to discover the new block on the storage and download its **index-header**.
    
    - To effectively use the WAL, and to be able to `recover` the `in-memory series` if an ingester abruptly **terminates**, store the WAL to a persistent disk that can survive an ingester failure. 
    - For example, when running in the cloud, include an `AWS EBS volume`.  
    - If you are running the Grafana Mimir cluster in `Kubernetes`, you can use a **StatefulSet** with a `persistent volume` claim for the ingesters. 
    - The location on the `filesystem` where the `WAL` is stored is the same location where local TSDB blocks (compacted from head) are stored. The locations of the WAL and the local TSDB blocks cannot be `decoupled`.
    
    1. **Ingesters write de-amplification**:

        - Ingesters store recently received samples in-memory in order to perform write de-amplification. 
        - If the ingesters immediately write received samples to the long-term storage, the system would have difficulty scaling due to the high pressure on the long-term storage. 
        - For this reason, the ingesters batch and compress samples in-memory and periodically upload them to the long-term storage.
        >Note: Write de-amplification is a key factor in reducing Mimir’s total cost of ownership (TCO).

    2. **Ingesters failure and data loss**: If an ingester process crashes or exits abruptly, any in-memory time series data that have not yet been uploaded to long-term storage might be lost. There are the following ways to mitigate this failure mode:

        - Replication
        - Write-ahead log (WAL)
        - Write-behind log (WBL), only used if out-of-order ingestion is enabled.

        1. **Replication and availability**:

            - Writes to the Mimir cluster are successful if a majority of ingesters received the data. With the default replication factor of 3, this means 2 out of 3 writes to ingesters must succeed. 
            - If the Mimir cluster loses a minority of ingesters, the in-memory series samples held by the lost ingesters are available in at least one other ingester, meaning no time series samples are lost.
            - If a majority of ingesters fail, time series might be lost if the failure affects all the ingesters holding the replicas of a specific time series.

        2. **Write-ahead log**: 
            
            - The write-ahead log (WAL) writes all incoming series to a `persistent disk` until the series are uploaded to the `long-term storage`. If an ingester fails, a subsequent process restart replays the WAL and recovers the in-memory series samples.

            - Unlike sole replication, the WAL ensures that in-memory time series data are not lost in the case of multiple ingester failures. Each ingester can recover the data from the WAL after a subsequent restart.

            - Replication is still recommended in order to gracefully handle a single ingester failure.

        3. **Write-behind log**:

            - The write-behind log (WBL) is similar to the WAL, but it only writes incoming `out-of-order` samples to a persistent disk until the series are uploaded to `long-term` storage.

            - There is a different log for this because it is not possible to know if a sample is out-of-order until Mimir tries to append it. First Mimir needs to attempt to append it, the TSDB will detect that it is `out-of-order`, append it anyway if `out-of-order` is enabled and then write it to the log.

            - If the ingesters fail, the same characteristics as in the WAL apply.
