# Chapter 4: Grafana

## Grafana Observability

Open Source (Vendor Neutral)

### LGTM

- Loki: Log Aggregation
- Grafana: Dashboard
- Tempo: Tracing
- Mimir: Metrics

### Application for Observability

- Consul
  - Open Source Service Registry
  - Similar to istio, in the point of service discovery. (MSA)
  - Key-value CMDB : Configuration Management Database

## Loki : Log Management System

### Architecture

![Loki Architecture](https://grafana.com/docs/loki/latest/get-started/loki_architecture_components.svg)

- Ingester
- Distributor
- Query Frontend
- Query Scheduler
- Querier
- index Gateway
- Ruler
- Compactor

### Data Flow

1. Write data flow
   - Distributor -> Ingester -> Storage
2. Read data flow
   - Query Frontend -> Query Scheduler -> Querier -> Storage

### Promtail

- Agent for Loki
- Pull log from log file and push to Loki

## Mimir : Metrics Management System

Distributed, Scalable, and Highly Available Metrics Store System

### Prometheus's disadvantages

- Scalability & High Availability
  - Prometheus is designed to be a single node system.
- Long-term storage
  - Prometheus is designed to store data in local disk.
  - Automatically delete old data.

### Mimir's advantages

- 100% Compatibility with Prometheus
- Horizontal Scalability
- Support object storage (S3, GCS, etc)
- Sharded query engine for query performance

### Mimir's Architecture

#### Write Path

![Write Path](https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/write-path.svg)

#### Read Path

![Read Path](https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/read-path.svg) |
