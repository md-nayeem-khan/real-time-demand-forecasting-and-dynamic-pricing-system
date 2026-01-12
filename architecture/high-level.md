# High-Level System Architecture

## Overview

This document describes the high-level architecture of the Real-Time Demand Forecasting & Dynamic Pricing System. The system follows a **stream-first, event-driven architecture** with clear separation of concerns across data ingestion, feature computation, ML inference, pricing logic, and API serving.

---

## Architectural Principles

### 1. **Event-Driven Architecture**
- All state changes represented as immutable events
- Services communicate via message queues (Kafka)
- Enables replay, debugging, and audit trails

### 2. **Stateless Services**
- API servers maintain no local state
- Enables horizontal scaling and fault tolerance
- State stored in external systems (Redis, PostgreSQL)

### 3. **Separation of Concerns**
- **Data Layer**: Event ingestion and storage
- **Processing Layer**: Feature computation and stream processing
- **ML Layer**: Model training and inference
- **Business Logic Layer**: Pricing rules and optimization
- **API Layer**: Request handling and response formatting

### 4. **Polyglot Persistence**
- Different data storage technologies optimized for specific use cases
- Redis for hot cache, PostgreSQL for transactional, ClickHouse for analytics

### 5. **Observability First**
- Metrics, logs, and traces instrumented from day one
- Distributed tracing across all services
- Real-time dashboards for business and technical metrics

---

## Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT APPLICATIONS                            │
│  (Mobile Apps, Web Portals, Partner APIs, Internal Dashboards)          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ HTTPS/TLS
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY / LOAD BALANCER                     │
│            (AWS ALB / NGINX / Cloudflare) + Rate Limiting               │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
┌───────────────────────────┐   ┌────────────────────────────────┐
│    PRICING API SERVICE    │   │   INTERNAL ADMIN API           │
│      (FastAPI/Python)     │   │  (Management, Monitoring)      │
│  - Compute Price          │   │  - Model Management            │
│  - Get Forecast           │   │  - Pricing Rules Config        │
│  - Explain Decision       │   │  - Feature Store Admin         │
│  - Health Checks          │   │  - Audit Log Queries           │
└──────────┬────────────────┘   └────────────┬───────────────────┘
           │                                  │
           │                                  │
           ├──────────────┬───────────────────┴──────────┬────────────────┐
           │              │                              │                │
           ▼              ▼                              ▼                ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌─────────────────┐
│ PRICING ENGINE   │ │ FORECAST SERVICE │ │ FEATURE STORE    │ │ RULE ENGINE     │
│  (gRPC Service)  │ │  (gRPC Service)  │ │  (Redis Cluster) │ │  (PostgreSQL)   │
│ - Apply Rules    │ │ - Get Forecast   │ │ - Online Store   │ │ - Min/Max Caps  │
│ - Surge Calc     │ │ - Multi-Horizon  │ │ - Feature Fetch  │ │ - Region Rules  │
│ - Explainability │ │ - Confidence     │ │ - TTL Management │ │ - A/B Config    │
└──────────┬───────┘ └──────────┬───────┘ └──────────┬───────┘ └─────────┬───────┘
           │                    │                    │                   │
           │                    │                    │                   │
           │                    ▼                    │                   │
           │         ┌──────────────────┐            │                   │
           │         │ ML INFERENCE     │            │                   │
           │         │  (ONNX Runtime)  │◄───────────┘                   │
           │         │ - XGBoost Model  │ Feature Retrieval              │
           │         │ - Prophet Model  │                                │
           │         │ - Model Registry │                                │
           │         └──────────┬───────┘                                │
           │                    │                                        │
           │                    │ Read Models                            │
           │                    ▼                                        │
           │         ┌──────────────────────────────────────────────┐    │
           │         │      MODEL REGISTRY (MLflow / S3)            │    │
           │         │  - Model Versioning                          │    │
           │         │  - A/B Test Config                           │    │
           │         │  - Metadata & Lineage                        │    │
           │         └──────────────────────────────────────────────┘    │
           │                                                             │
           ├─────────────────────────────────────────────────────────────┘
           │ Write Audit Logs
           ▼
┌──────────────────────────────────────────────────────────────────────── ──┐
│                        AUDIT & METADATA STORE                             │
│                           (PostgreSQL)                                    │
│  - Pricing Decisions Log                                                  │
│  - API Request History                                                    │
│  - Model Inference Metadata                                               │
│  - Configuration Changes                                                  │
└───────────────────────────────────────────────────────────────────────────┘


════════════════════════════════════════════════════════════════════════════
                           STREAMING DATA PIPELINE
════════════════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────────────┐
│                          EVENT SOURCES                                   │
│  User Actions | Bookings | Cancellations | Supply Updates | External APIs│
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             │ HTTP / SDK
                             ▼
┌─────────────────────────────────────────────────────────────────────── ───┐
│                    EVENT INGESTION SERVICE                                │
│                    (FastAPI / Kafka Producer)                             │
│  - Event Validation                                                       │
│  - Schema Enforcement (Avro/Protobuf)                                     │
│  - Deduplication (Idempotency Keys)                                       │
│  - Rate Limiting                                                          │
└────────────────────────────┬────────────────────────────────────────── ───┘
                             │
                             │ Produce Events
                             ▼
┌──────────────────────────────────────────────────────────────────── ──────┐
│                      KAFKA / REDPANDA CLUSTER                             │
│  Topics:                                                                  │
│  - user-events (booking, search, cancel)                                  │
│  - supply-events (driver location, availability)                          │
│  - external-events (weather, traffic, events)                             │
│  - demand-signals (aggregated demand metrics)                             │
│  - pricing-decisions (audit trail)                                        │
│                                                                           │
│  Partitions: By region_id for parallel processing                         │
│  Replication: Factor 3 for durability                                     │
└─────────────┬──────────────────────────────┬──────────────────────── ─────┘
              │                              │
              │ Consume                      │ Consume
              ▼                              ▼
┌────────────────────────────┐  ┌───────────────────────────────────────┐
│  FEATURE COMPUTATION       │  │   ANALYTICS PIPELINE                  │
│   (Apache Flink / Spark)   │  │   (Kafka Connect → ClickHouse)        │
│ - Real-time Aggregation    │  │ - Historical Data Warehousing         │
│ - Windowing (1min, 5min)   │  │ - Business Intelligence               │
│ - Feature Engineering      │  │ - Data Science Exploration            │
│ - State Management         │  │ - Regulatory Reporting                │
└──────────┬─────────────────┘  └───────────────────────────────────────┘
           │
           │ Write Features
           ▼
┌───────────────────────────────────────────────────────────────────── ─────┐
│                         FEATURE STORE                                     │
│                    Online: Redis Cluster                                  │
│                    Offline: S3 + Parquet / Feast                          │
│  - Time-based features (hour, day_of_week, season)                        │
│  - Demand features (rolling_avg_7d, trend, seasonality)                   │
│  - Supply features (available_drivers, utilization)                       │
│  - External features (weather, traffic_index, events)                     │
└────────────────────────────────┬──────────────────────────────────── ─────┘
                                 │
                                 │ Training Pipeline
                                 ▼
┌───────────────────────────────────────────────────────────────────── ─────┐
│                       MODEL TRAINING PIPELINE                             │
│                      (Offline / Batch - Airflow)                          │
│  1. Data Extraction (S3/ClickHouse → Training Dataset)                    │
│  2. Feature Engineering (Offline Feature Store)                           │
│  3. Model Training (XGBoost, Prophet, LightGBM)                           │
│  4. Model Evaluation (Validation Set, Metrics)                            │
│  5. Model Registration (MLflow → Model Registry)                          │
│  6. Canary Deployment (10% traffic → gradual rollout)                     │
│                                                                           │
│  Schedule: Weekly (Sunday 2am UTC)                                        │
│  Infrastructure: Spot instances (cost optimization)                       │
└───────────────────────────────────────────────────────────────────────────┘


════════════════════════════════════════════════════════════════════════════
                     OBSERVABILITY & MONITORING STACK
════════════════════════════════════════════════════════════════════════════

┌───────────────────────────────────────────────────────────────────── ─────┐
│                         PROMETHEUS + GRAFANA                              │
│  - RED Metrics (Rate, Errors, Duration)                                   │
│  - Business Metrics (Revenue, Conversion, Surge %)                        │
│  - Infrastructure Metrics (CPU, Memory, Network)                          │
│  - Kafka Metrics (Consumer Lag, Throughput)                               │
│  - Model Metrics (Inference Latency, Accuracy)                            │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────── ─────┐
│                    ELK STACK / LOKI (LOGGING)                             │
│  - Structured Logs (JSON format)                                          │
│  - Correlation IDs (trace requests across services)                       │
│  - Error Logs (with stack traces)                                         │
│  - Audit Logs (pricing decisions, config changes)                         │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────── ─────┐
│                  JAEGER / OPENTELEMETRY (TRACING)                         │
│  - Distributed Tracing (end-to-end request flow)                          │
│  - Latency Breakdown (per service hop)                                    │
│  - Error Propagation Tracking                                             │
│  - Sampling (1% of successful, 100% of errors)                            │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────── ─────┐
│                       ALERTING (PagerDuty)                                │
│  - SLA Violations (P99 latency > 50ms)                                    │
│  - Error Rate Spikes (> 1%)                                               │
│  - Model Accuracy Degradation (MAPE > 20%)                                │
│  - Infrastructure Issues (pod crashes, OOM)                               │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Request Flow: Price Computation

### Standard Request Flow

```
1. User initiates booking request
   ↓
2. Mobile/Web App → API Gateway (rate limit check)
   ↓
3. API Gateway → Pricing API Service
   ↓
4. Pricing API extracts request context:
   - region_id: "sg-central"
   - timestamp: 2026-01-11T10:30:00Z
   - product_type: "standard_ride"
   ↓
5. [PARALLEL] Pricing API calls:
   a Forecast Service (gRPC) → Get demand forecast
      - Forecast Service → ML Inference → XGBoost Model
      - ML Inference → Feature Store (Redis) → Fetch features
      - Return: {demand_score: 0.87, confidence: 0.94}
   
   b Rule Engine (PostgreSQL) → Get pricing rules
      - Return: {base_price: 12.0, min_multiplier: 1.0, max_multiplier: 3.0}
   ↓
6. Pricing Engine computes optimal price:
   - surge_multiplier = f(demand_score, supply_score, rules)
   - final_price = base_price * surge_multiplier
   - factor_breakdown = {demand: 0.45, supply: -0.12, ...}
   ↓
7. Write audit log (async, non-blocking):
   - PostgreSQL: pricing_decisions table
   - Kafka: pricing-decisions topic
   ↓
8. Cache result (async, non-blocking):
   - Redis: SET price:sg-central:standard EX 60 {price_data}
   ↓
9. Return response to client:
   {
     "price": 15.75,
     "surge_multiplier": 1.31,
     "valid_until": "2026-01-11T10:32:00Z",
     "factors": {...},
     "trace_id": "abc123"
   }
```

**Latency Budget Breakdown:**
- API Gateway: 2ms
- Pricing API processing: 5ms
- Forecast Service (gRPC call): 8ms
  - Redis feature fetch: 2ms
  - Model inference: 4ms
  - Network + serialization: 2ms
- Rule Engine query: 3ms
- Pricing computation: 2ms
- Response serialization: 2ms
- **Total P50: ~22ms, P99: ~45ms** ✅

---

## Data Flow: Event Ingestion to Feature Store

```
1. User performs action (e.g., booking, search, cancel)
   ↓
2. Mobile/Web App → Event Ingestion API
   POST /api/v1/events
   {
     "event_type": "booking",
     "region_id": "sg-central",
     "timestamp": "2026-01-11T10:30:00Z",
     "user_id": "hashed_abc123",
     "metadata": {...}
   }
   ↓
3. Event Ingestion Service validates:
   - Schema validation (Avro/Protobuf)
   - Deduplication (idempotency_key)
   - Rate limiting (per user)
   ↓
4. Produce to Kafka:
   Topic: user-events
   Partition: hash(region_id) % num_partitions
   Key: region_id
   Value: {event_data}
   ↓
5. Flink Stream Processor consumes:
   - Event-time windowing (tumbling 1-minute windows)
   - Aggregate by region_id:
     * booking_count_1min
     * search_count_1min
     * cancel_rate_1min
   - Join with supply-events stream:
     * available_drivers
     * driver_utilization
   ↓
6. Flink computes features:
   - rolling_avg_7d (from state store)
   - demand_trend (current vs. historical)
   - seasonality_score (hourly patterns)
   ↓
7. Write to Feature Store:
   - Redis: HMSET features:sg-central {...} EX 120
   - S3 (offline): Parquet files for training
   ↓
8. Feature Store serves:
   - ML Inference Service (online predictions)
   - Training Pipeline (offline batch)
```

---

## Data Storage Strategy

### Hot Data (Redis)
- **Use Case**: Real-time feature serving, pricing cache
- **TTL**: 30-120 seconds
- **Size**: ~50GB (top 20% most-accessed data)
- **Eviction**: LRU policy

### Warm Data (PostgreSQL)
- **Use Case**: Pricing rules, metadata, audit logs (recent 30 days)
- **Backup**: Daily snapshots, point-in-time recovery
- **Size**: ~500GB
- **Queries**: Indexed on region_id, timestamp

### Cold Data (ClickHouse / BigQuery)
- **Use Case**: Historical analytics, model training data
- **Retention**: 2 years
- **Size**: ~10TB
- **Compression**: Columnar storage (10:1 ratio)

### Archive (S3 Glacier)
- **Use Case**: Regulatory compliance (7 years)
- **Access**: Rare (audit requests)

---

## Security Architecture

### Network Security
- **Perimeter**: VPC with private subnets
- **Ingress**: Only API Gateway publicly accessible
- **Inter-service**: gRPC with mTLS (mutual TLS)
- **Egress**: Restricted to approved external APIs

### Authentication & Authorization
- **External API**: OAuth2 + JWT tokens
- **Internal Services**: Workload identity (SPIFFE/SPIRE)
- **Database**: Connection pooling with IAM roles
- **Kafka**: SASL/SCRAM authentication

### Data Protection
- **In Transit**: TLS 1.3 for all external, mTLS internal
- **At Rest**: AES-256 encryption (KMS-managed keys)
- **PII**: Hashed/anonymized user IDs, no raw PII in logs

### Secrets Management
- **HashiCorp Vault** or AWS Secrets Manager
- Automatic secret rotation (90 days)
- Audit logs for secret access

---

## Deployment Architecture

### Kubernetes Clusters
- **Production**: 3 clusters (us-east-1, eu-west-1, ap-southeast-1)
- **Staging**: 1 cluster (cost optimization)
- **Development**: Local (Docker Compose)

### Namespaces
- `pricing-api`: API services
- `ml-inference`: Model serving
- `streaming`: Flink, Kafka consumers
- `data`: Redis, PostgreSQL operators
- `monitoring`: Prometheus, Grafana

### Scaling Strategy
```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: pricing-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: pricing-api
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

---

## Capacity Planning

### Current Scale (as of Q1 2026)

| Component | Current | Peak | Headroom |
|-----------|---------|------|----------|
| **API Requests** | 5K/s | 12K/s | 2.4x |
| **Kafka Events** | 20K/s | 50K/s | 2.5x |
| **Redis QPS** | 50K/s | 120K/s | 2.4x |
| **PostgreSQL QPS** | 5K/s | 8K/s | 1.6x |
| **ML Inference** | 4K/s | 10K/s | 2.5x |

### Infrastructure Cost (Monthly)

| Component | Cost | Percentage |
|-----------|------|------------|
| Compute (K8s) | $12K | 40% |
| Data Storage | $5K | 17% |
| Kafka/Streaming | $6K | 20% |
| Redis | $4K | 13% |
| Monitoring | $3K | 10% |
| **Total** | **$30K** | **100%** |

---

## Disaster Recovery

### Backup Strategy
- **Database**: Daily full backup, 5-minute PITR
- **Redis**: Daily RDB snapshot
- **Kafka**: Topic replication (3x), retention 7 days
- **Models**: Versioned in S3 with immutability

### Recovery Objectives
- **RTO (Recovery Time Objective)**: 15 minutes
- **RPO (Recovery Point Objective)**: 5 minutes (database), 1 minute (events)

### Failover Procedure
1. DNS failover to healthy region (Route53 health checks)
2. Promote read replica to primary (database)
3. Scale up pods in target region
4. Verify end-to-end smoke tests
5. Monitor for anomalies

---

## Related Documentation

- [Sequence Diagrams](sequence-diagrams.md) - Detailed interaction flows
- [Data Flow](data-flow.md) - Deep dive into data pipelines
- [Trade-offs](../docs/trade-offs.md) - Design decisions
- [Failure Scenarios](../docs/failure-scenarios.md) - Fault tolerance
