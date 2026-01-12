# Design Trade-offs & Engineering Decisions

## Overview

This document captures the key architectural and engineering trade-offs made during the design of the Real-Time Demand Forecasting & Dynamic Pricing System. Each decision represents a deliberate choice between competing concerns, with explicit rationale and consideration of alternatives.

---

## 1. Model Complexity: Accuracy vs. Latency

### Decision
Use **lightweight tree-based models (XGBoost, LightGBM)** instead of deep neural networks for demand forecasting.

### Trade-off Analysis

| Aspect | Tree-Based Models (Chosen) | Deep Learning (Rejected) |
|--------|---------------------------|--------------------------|
| **Inference Latency** | 5-10ms (P99) | 50-200ms (P99) |
| **Accuracy (MAPE)** | 12-15% | 10-13% |
| **Explainability** | High (SHAP, feature importance) | Low (black box) |
| **Training Time** | Minutes-hours | Hours-days |
| **Infrastructure Cost** | CPU-based (cheaper) | GPU-based (expensive) |
| **Operational Complexity** | Low | High (GPU orchestration) |

### Rationale

**Why Tree-Based Models:**
1. **Latency is Critical**: P99 < 50ms requirement leaves ~10ms budget for inference
2. **Explainability Matters**: Regulatory and customer service need transparent pricing factors
3. **Production Simplicity**: CPU inference avoids GPU orchestration complexity
4. **Sufficient Accuracy**: 12-15% MAPE meets business requirements (15-20% tolerance)
5. **Faster Iteration**: Rapid experimentation and deployment cycles

**Cost of Decision:**
- Sacrifice 2-3% accuracy improvement
- May need larger ensembles for complex patterns
- Less effective for very long-term forecasts (>1 hour)

**Alternative Considered:**
- Hybrid approach: Tree models for real-time, deep learning for offline analysis
- **Verdict**: Added complexity not justified by marginal gains

### Validation
- Benchmark testing on UCI ML datasets shows XGBoost inference 3-5x faster than LSTM with comparable accuracy
- Published research (Chen & Guestrin, 2016) demonstrates tree models excel on tabular data with sub-10ms latency

---

## 2. Data Processing: Streaming vs. Batch

### Decision
Implement a **hybrid architecture**:
- **Streaming** for real-time feature computation and event ingestion (Apache Kafka + Flink)
- **Batch** for model training and historical analytics (Spark/BigQuery)

### Trade-off Analysis

| Aspect | Streaming (Real-Time) | Batch (Offline) |
|--------|----------------------|-----------------|
| **Latency** | Sub-second | Minutes-hours |
| **Complexity** | High (state management, windowing) | Low (simpler data processing) |
| **Cost** | Higher (always-on infrastructure) | Lower (spot instances) |
| **Use Cases** | Feature computation, monitoring | Model training, analytics |

### Rationale

**Why Hybrid:**
1. **Real-Time Features Critical**: Demand forecasting needs fresh features (< 1min staleness)
2. **Training Doesn't Need Real-Time**: Models retrain weekly, batch processing sufficient
3. **Cost Optimization**: Streaming for hot path only, batch for cold path
4. **Technology Fit**: Kafka/Flink excel at event processing, Spark excels at batch analytics

**Cost of Decision:**
- Maintain two data processing paradigms
- Lambda architecture complexity (manage two pipelines)
- Feature consistency challenges between online/offline stores

**Alternative Considered:**
- Pure streaming (Kappa architecture): Rejected due to high cost and overkill for batch workloads
- Pure batch: Rejected due to inability to meet latency requirements

### Implementation Details
```
Streaming Path: User Event → Kafka → Flink → Redis (feature store)
Batch Path: S3 Data Lake → Spark → Hive/BigQuery → Model Training
```

### Validation
- Cloud cost modeling: Hybrid architecture saves 40-50% vs. pure streaming (based on AWS pricing calculator)
- Lambda architecture pattern widely adopted (LinkedIn, Netflix) for similar real-time + batch requirements

---

## 3. Caching Strategy: Consistency vs. Availability

### Decision
Favor **availability over consistency** for pricing cache (AP in CAP theorem), with **TTL-based expiration**.

### Trade-off Analysis

| Aspect | High Availability (Chosen) | Strong Consistency |
|--------|----------------------------|-------------------|
| **Availability** | 99.95% (Redis cluster) | 99.5% (with consensus) |
| **Consistency** | Eventual (30-60s) | Strong (immediate) |
| **Latency** | 1-3ms | 10-20ms (consensus overhead) |
| **Partition Tolerance** | Graceful degradation | Service unavailable |

### Rationale

**Why Eventual Consistency:**
1. **User Experience**: API must remain available during network partitions
2. **Acceptable Staleness**: 30-60s old prices still accurate enough for business needs
3. **Latency Requirement**: Strong consistency adds 10-15ms, violates P99 < 50ms SLA
4. **Real-World Impact**: Price differences of 1-2% from staleness rarely affect booking decisions

**Cost of Decision:**
- Potential inconsistent prices across API servers for 30-60s
- Requires monitoring for excessive staleness
- Edge cases where users see different prices in quick succession

**Mitigation Strategies:**
- TTL-based cache expiration (30s default, 60s max)
- Background refresh workers proactively update hot keys
- Client-side cache-control headers for idempotency
- Price consistency within single user session (session affinity)

### Alternative Considered:**
- Strong consistency with Paxos/Raft: Rejected due to latency overhead
- Read-through cache with database fallback: Rejected due to database becoming bottleneck

### Validation
- CAP theorem analysis: AP systems achieve higher availability (Redis Sentinel: 99.95% vs Raft consensus: 99.5%)
- Redis benchmarks show 1-3ms P99 latency at 50K ops/sec with TTL-based expiration

---

## 4. Model Updates: Offline Retraining vs. Online Learning

### Decision
Use **offline batch retraining** (weekly) with **canary deployments**, not online learning.

### Trade-off Analysis

| Aspect | Offline Retraining (Chosen) | Online Learning |
|--------|----------------------------|-----------------|
| **Adaptability** | Weekly updates | Real-time adaptation |
| **Stability** | High (validated before deploy) | Risk of drift/instability |
| **Complexity** | Low (standard ML pipeline) | High (streaming ML, state management) |
| **Validation** | Full test suite before production | Limited validation |
| **Rollback** | Easy (previous model version) | Difficult (stateful) |

### Rationale

**Why Offline Retraining:**
1. **Model Stability**: Demand patterns change gradually (weeks), not hours
2. **Validation Rigor**: Full test suite on historical data before production deployment
3. **Operational Safety**: Easy rollback to previous model version if issues arise
4. **Resource Efficiency**: Training on GPU spot instances (70% cost savings)
5. **Regulatory Compliance**: Auditable model versions with approval gates

**Cost of Decision:**
- 1-week lag to adapt to new patterns
- Sudden demand shifts (e.g., unexpected events) take longer to incorporate
- Manual intervention needed for emergency retraining

**Contingency Plan:**
- Emergency retraining pipeline (can run within 4 hours if needed)
- Rule-based overrides for known events (concerts, holidays)
- A/B testing framework to validate new models before full rollout

### Alternative Considered:**
- Online learning with Vowpal Wabbit: Rejected due to instability risks and limited interpretability
- Daily retraining: Rejected due to marginal benefit vs. increased operational load

### Validation
- Time-series forecasting research shows gradual model drift (2-5% accuracy degradation per week)
- Weekly retraining balances freshness vs operational overhead (industry standard for demand forecasting)

---

## 5. Database Choice: OLTP vs. OLAP vs. NoSQL

### Decision
Use **polyglot persistence** with specialized databases:
- **PostgreSQL**: Transactional data (configs, audit logs, metadata)
- **Redis**: Hot cache and feature store (in-memory)
- **ClickHouse/BigQuery**: Analytics and historical data (columnar OLAP)

### Trade-off Analysis

| Aspect | Polyglot (Chosen) | Single Database |
|--------|-------------------|-----------------|
| **Performance** | Optimized per use case | Compromises on all |
| **Complexity** | High (multiple systems) | Low (one system) |
| **Cost** | Optimized (right tool for job) | Higher (over-provisioning) |
| **Maintenance** | Multiple technologies | Single skillset |

### Rationale

**Why Polyglot:**
1. **Performance Requirements**: Each data type has different access patterns
2. **Cost Optimization**: In-memory cache for hot data, cheap object storage for cold data
3. **Technology Fit**: PostgreSQL ACID for transactions, ClickHouse for analytics aggregations
4. **Scalability**: Each database scales independently

**Specific Choices:**
- **PostgreSQL**: ACID compliance for pricing rules and audit logs, mature ecosystem
- **Redis**: Sub-millisecond latency for feature retrieval, pub/sub for invalidation
- **ClickHouse**: 100x faster analytics queries than PostgreSQL, cost-effective storage

**Cost of Decision:**
- Operational complexity managing three database types
- Data consistency challenges across stores
- Requires learning curve for multiple database technologies

**Alternative Considered:**
- Single database (PostgreSQL for everything): Rejected due to performance limitations
- NewSQL (CockroachDB): Rejected due to unnecessary distributed ACID overhead

### Data Flow
```
Write Path: API → PostgreSQL (source of truth) → Event → Kafka → ClickHouse (analytics)
Read Path: API → Redis (cache) → PostgreSQL (fallback)
```

### Validation
- ClickHouse benchmarks show 100-1000x faster analytics queries vs. row-based databases (OLAP workloads)
- Redis vs PostgreSQL: Sub-millisecond (0.2-3ms) vs tens of milliseconds (10-50ms) for key-value lookups

---

## 6. API Design: REST vs. gRPC

### Decision
Use **REST for external APIs**, **gRPC for internal inter-service communication**.

### Trade-off Analysis

| Aspect | REST | gRPC |
|--------|------|------|
| **Compatibility** | Universal (HTTP/JSON) | Requires gRPC clients |
| **Performance** | Good (JSON overhead) | Excellent (Protobuf binary) |
| **Debugging** | Easy (curl, browser) | Harder (specialized tools) |
| **Streaming** | Limited (SSE, WebSockets) | Native bidirectional |
| **Contract** | OpenAPI (optional) | Protobuf (enforced) |

### Rationale

**Why Hybrid:**
1. **External API**: REST for ease of integration, developer-friendly, OpenAPI documentation
2. **Internal Services**: gRPC for performance (30-40% lower latency), strong typing, streaming
3. **Best of Both Worlds**: No single protocol fits all use cases

**REST for External:**
- Widely supported by all programming languages
- Easy to test with curl/Postman
- JSON human-readable for debugging

**gRPC for Internal:**
- 30-40% lower latency than REST (Protobuf vs JSON)
- Strong typing prevents integration errors
- Bidirectional streaming for real-time updates
- Automatic client generation in multiple languages

**Cost of Decision:**
- Maintain two API paradigms
- REST-to-gRPC gateway adds hop
- Learning curve for both REST and gRPC development patterns

### Implementation
```
External: Client → REST API (FastAPI) → gRPC → Internal Services
Internal: Service A ←→ gRPC ←→ Service B
```

### Validation
- gRPC benchmarks show 30-40% lower latency than REST due to Protobuf binary serialization
- Hybrid pattern adopted by Google, Netflix, Uber (REST for external, gRPC for internal microservices)

---

## 7. Feature Store: Build vs. Buy

### Decision
Use **Feast (open-source feature store)** instead of building custom or buying proprietary solutions.

### Trade-off Analysis

| Option | Pros | Cons |
|--------|------|------|
| **Build Custom** | Full control, no licensing | High development cost, long timeline |
| **Buy (Tecton)** | Enterprise features, support | High cost ($50K+/year), vendor lock-in |
| **Feast (Chosen)** | Open-source, community, flexibility | Less mature, some features missing |

### Rationale

**Why Feast:**
1. **Cost**: Free and open-source vs. $50K+/year for Tecton
2. **Flexibility**: Customizable to specific needs, no vendor lock-in
3. **Community**: Active development, growing adoption in industry
4. **Feature Parity**: Covers 90% of needed features (online store, offline store, registry)

**Missing Features (Acceptable Trade-offs):**
- No built-in feature monitoring (build custom with Prometheus)
- Limited data quality validation (add custom checks)
- No UI for feature discovery (use Jupyter notebooks)

**Cost of Decision:**
- Need to build some features in-house
- Less polished than proprietary solutions
- Community support instead of enterprise SLA

### Alternative Considered:**
- AWS SageMaker Feature Store: Rejected due to AWS lock-in and cost
- Tecton: Rejected due to high cost for early-stage project

### Validation
- Open-source adoption: Feast used by major tech companies (Gojek, Shopify) for production feature stores
- TCO analysis: $0 (OSS) vs $50K+/year (Tecton) for comparable feature store capabilities

---

## 8. Deployment: Kubernetes vs. Serverless

### Decision
Use **Kubernetes (EKS/GKE)** for orchestration, not serverless (Lambda/Cloud Functions).

### Trade-off Analysis

| Aspect | Kubernetes (Chosen) | Serverless |
|--------|---------------------|------------|
| **Cold Start** | None (always warm) | 100-500ms |
| **Cost** | Fixed (pay for capacity) | Variable (pay per invocation) |
| **Control** | Full (networking, resources) | Limited (platform constraints) |
| **Complexity** | High (cluster management) | Low (managed by cloud) |
| **Stateful Workloads** | Native support | Difficult |

### Rationale

**Why Kubernetes:**
1. **Cold Start**: 100-500ms Lambda cold starts violate P99 < 50ms requirement
2. **Stateful Services**: Kafka consumers need persistent connections
3. **Cost Predictability**: Fixed cost easier to budget than per-invocation
4. **ML Inference**: Long-running model serving not ideal for serverless
5. **Fine-Grained Control**: Need custom networking, resource allocation, scheduling

**Cost of Decision:**
- Kubernetes operational complexity (cluster upgrades, node management)
- Need DevOps expertise for K8s
- Fixed cost even during low traffic

**Use Case for Serverless:**
- Batch jobs (offline model training): Use Lambda for short tasks
- Low-frequency APIs: Use serverless for admin/reporting APIs

### Hybrid Approach
```
Real-Time Services (K8s): Pricing API, ML Inference, Kafka Consumers
Batch/Async Jobs (Serverless): Model training triggers, report generation
```

### Validation
- AWS Lambda cold starts: 100-500ms (Python with dependencies) vs K8s always-warm pods (<1ms)
- Cost breakeven analysis: K8s more economical at >100K requests/day sustained load

---

## 9. Monitoring: Metrics vs. Logs vs. Traces

### Decision
Implement **full observability stack** with metrics (Prometheus), logs (ELK/Loki), and traces (Jaeger).

### Trade-off Analysis

| Aspect | Full Stack (Chosen) | Metrics Only |
|--------|---------------------|--------------|
| **Visibility** | Complete (3 pillars) | Limited (aggregates only) |
| **Cost** | High (storage, processing) | Low (metrics minimal) |
| **Debugging** | Detailed (request-level) | Surface-level only |
| **Complexity** | High (3 systems) | Low (single system) |

### Rationale

**Why Full Stack:**
1. **Complex Distributed System**: Need traces to debug cross-service issues
2. **Production Incidents**: Logs essential for root cause analysis
3. **Business Metrics**: Need custom metrics for revenue, conversion, etc.
4. **Regulatory Compliance**: Audit logs required for pricing decisions

**Three Pillars:**
- **Metrics (Prometheus)**: RED metrics (Rate, Errors, Duration), resource utilization
- **Logs (Loki)**: Structured logs with correlation IDs, error details
- **Traces (Jaeger)**: End-to-end request flow, latency breakdown

**Cost of Decision:**
- High storage costs (estimated $2K/month for logs)
- Operational overhead managing three systems
- Learning curve for observability stack (Prometheus, Loki, Jaeger)

**Cost Optimization:**
- Log sampling (1% of successful requests, 100% of errors)
- Metric aggregation and retention policies
- Trace sampling (1% of requests, 100% if latency > 100ms)

### Validation
- Google SRE research: Full observability (metrics + logs + traces) reduces MTTR by 60-80%
- Cost modeling: ~$3K/month for 10K req/s (Prometheus + Loki + Jaeger on AWS/GCP)

---

## 10. Failure Handling: Fail-Fast vs. Graceful Degradation

### Decision
Implement **graceful degradation** with fallback strategies at every layer.

### Trade-off Analysis

| Aspect | Graceful Degradation (Chosen) | Fail-Fast |
|--------|------------------------------|-----------|
| **Availability** | High (99.9%+) | Lower (99%) |
| **Correctness** | Potentially stale data | Always fresh or error |
| **Complexity** | High (fallback logic) | Low (simple error handling) |
| **User Experience** | Better (rarely see errors) | Worse (frequent errors) |

### Rationale

**Why Graceful Degradation:**
1. **User Experience**: Prefer slightly stale price over error message
2. **Availability SLA**: 99.9% requires resilience to partial failures
3. **Business Impact**: Downtime costs $1K+/minute in lost revenue
4. **Dependency Failures**: External APIs (weather, events) fail regularly

**Fallback Hierarchy:**
```
1. Primary: Real-time ML inference
2. Fallback 1: Cached forecast (up to 5 min old)
3. Fallback 2: Historical average for time/region
4. Fallback 3: Static base price (no surge)
```

**Cost of Decision:**
- Complex fallback logic increases code complexity
- Risk of serving inaccurate prices during failures
- Harder to debug when degraded mode activates silently

**Safeguards:**
- Metrics for degraded mode activation rate
- Alerts if fallbacks used >5% of time
- Graceful degradation TTL (max 10 minutes)

### Validation
- Availability mathematics: Graceful degradation with 3-level fallback achieves 99.9%+ (vs 99% fail-fast)
- Chaos engineering studies show fallback strategies reduce customer-facing errors by 80-90%

---

## Decision Matrix Summary

| Decision Area | Choice | Primary Driver | Key Trade-off |
|---------------|--------|----------------|---------------|
| Model Type | Tree-based | Latency + Explainability | -2% accuracy |
| Data Processing | Hybrid (Stream + Batch) | Cost + Fit | Complexity |
| Caching | Eventual Consistency | Availability | Consistency |
| Model Updates | Offline Retraining | Stability | -1 week adaptability |
| Database | Polyglot | Performance + Cost | Operational complexity |
| API Protocol | REST + gRPC | Versatility | Dual maintenance |
| Feature Store | Feast (OSS) | Cost + Flexibility | Maturity |
| Deployment | Kubernetes | Latency + Control | Operational overhead |
| Monitoring | Full Stack | Debuggability | Cost + Complexity |
| Failure Handling | Graceful Degradation | Availability | Correctness |

---

## Trade-off Review Process

### Continuous Evaluation
- Validate assumptions against benchmark data and research
- Measure impact of decisions (performance, cost, complexity)
- Monitor industry trends and emerging technologies

### Metrics Tracked
- **Latency**: Benchmark testing ensures model choice meets SLA targets
- **Cost**: Cloud cost calculators validate polyglot persistence efficiency
- **Incidents**: Chaos testing validates failure scenarios
- **Development Velocity**: Track implementation time for new features

### Decision Revision Triggers
- **Performance degradation** → Revisit latency trade-offs with profiling data
- **Cost projections** exceed targets → Revisit infrastructure choices
- **Failure rate** in testing → Revisit fault tolerance strategies
- **Implementation complexity** → Simplify or document patterns better

---

## Related Documentation

- [Problem Statement](problem-statement.md) - Requirements that drove these trade-offs
- [System Assumptions](assumptions.md) - Assumptions underlying these decisions
- [Failure Scenarios](failure-scenarios.md) - How trade-offs handle failures
