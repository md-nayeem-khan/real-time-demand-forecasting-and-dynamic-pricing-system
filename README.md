# Real-Time Demand Forecasting & Dynamic Pricing System

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

## Overview

This project implements a **production-grade, distributed backend system** for **real-time demand forecasting and dynamic pricing**. The system processes millions of events daily, executes ML inference at scale, and serves pricing decisions with sub-50ms P99 latency.

Designed with **distributed systems best practices**, **event-driven architecture**, **stream processing**, and **ML operations at scale**, this system demonstrates expertise in building mission-critical infrastructure for marketplace platforms.

---

## Key Capabilities

### Real-Time Processing
- **Sub-second demand forecasting** with minute-level granularity
- **Event-driven architecture** processing xxxK+ events/second
- **Streaming feature computation** with exactly-once semantics
- **Low-latency ML inference** (P99 < xxxms)

### Dynamic Pricing Engine
- **Rule-based pricing optimization** with configurable constraints
- **Multi-factor price computation** (demand, supply, seasonality, events)
- **A/B testing framework** for pricing strategies
- **Explainable pricing decisions** with audit trails

### Enterprise-Grade Reliability
- **99.9% availability** with graceful degradation
- **Circuit breakers** and fallback mechanisms
- **Multi-region deployment** support
- **Comprehensive observability** (metrics, logs, traces)

### ML Operations
- **Online/offline feature stores** with consistency guarantees
- **Model versioning** and registry integration
- **Canary deployments** for model rollouts
- **Automated model monitoring** and drift detection

---

## Business Use Cases

| Industry | Use Case | Key Benefit |
|----------|----------|-------------|
| **Ride-Hailing** | Surge pricing during peak hours | xxx% revenue optimization |
| **Hospitality** | Hotel room dynamic pricing | xxx% occupancy improvement |
| **E-Commerce** | Flash sale & inventory pricing | xxx% margin improvement |
| **Food Delivery** | Demand-based delivery fees | Balanced supply-demand |

---

## System Architecture

### High-Level Components

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Event Sources  │───▶│  Stream Pipeline │────▶│ Feature Store   │
│ (Kafka/Kinesis) │     │  (Flink/Spark)   │     │ (Redis/Feast)   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                           │
                                                           ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Client APIs   │◀───│  Pricing Engine  │◀────│ ML Inference    │
│  (REST/gRPC)    │     │   (FastAPI)      │     │ (XGBoost/ONNX)  │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

### Technology Stack

**Backend & APIs**
- **Python 3.9+** with FastAPI for async request handling
- **gRPC** for inter-service communication
- **REST APIs** with OpenAPI documentation
- **Pydantic** for data validation and serialization

**Streaming & Messaging**
- **Apache Kafka** / Redpanda for event streaming
- **Apache Flink** for stream processing (alternative: Spark Streaming)
- **Kafka Connect** for data integration

**Data Storage**
- **Redis** - Hot cache for features and pricing (in-memory)
- **PostgreSQL** - Metadata, configs, audit logs (OLTP)
- **ClickHouse** / BigQuery - Analytics and historical data (OLAP)
- **S3** / MinIO - Model artifacts and backups

**ML & Forecasting**
- **XGBoost / LightGBM** - Primary forecasting models
- **Prophet** - Time series decomposition
- **ONNX Runtime** - Model inference optimization
- **MLflow** - Experiment tracking and model registry
- **Feast** - Feature store (online + offline)

**Infrastructure & DevOps**
- **Docker** & Docker Compose for containerization
- **Kubernetes** (K8s) with Helm charts
- **Horizontal Pod Autoscaler** (HPA) for auto-scaling
- **Prometheus + Grafana** for monitoring
- **Jaeger** / OpenTelemetry for distributed tracing
- **GitHub Actions** / GitLab CI for CI/CD

**Testing**
- **pytest** with async support
- **locust** for load testing
- **testcontainers** for integration tests

---

## Non-Functional Requirements

| Metric | Target | Rationale |
|--------|--------|-----------|
| **API Latency (P99)** | < 50ms | Real-time pricing SLA |
| **Throughput** | 10K requests/sec | Peak load handling |
| **Availability** | 99.9% | Mission-critical service |
| **Data Freshness** | < 1 minute | Accurate demand signals |
| **Model Inference** | < 10ms | Low-latency ML pipeline |
| **Event Processing** | Exactly-once | Financial accuracy |

---

## Quick Start

### Prerequisites

- Python 3.9+
- Docker & Docker Compose
- Kafka (or use Docker setup)
- Redis

### Local Development Setup

```bash
# Clone the repository
git clone https://github.com/md-nayeem-khan/real-time-demand-forecasting-and-dynamic-pricing-system.git
cd real-time-demand-forecasting-and-dynamic-pricing-system

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Start infrastructure services
docker-compose up -d

# Run database migrations
python scripts/migrate.py

# Start the API server
uvicorn src.api.main:app --reload --port 8000

# In another terminal, start the stream processor
python src/streaming/consumer.py
```

### Running Tests

```bash
# Unit tests
pytest tests/unit -v

# Integration tests
pytest tests/integration -v

# Load tests
locust -f tests/load/pricing_load_test.py
```

---

## Key Design Decisions

### 1. **Stream-First Architecture**
- Event-driven design for real-time responsiveness
- Decoupled services via message queues
- Enables horizontal scaling and fault isolation

### 2. **Hybrid Feature Store**
- Online store (Redis) for low-latency serving
- Offline store (BigQuery) for model training
- Feast for feature consistency

### 3. **Lightweight ML Models**
- Tree-based models (XGBoost) over deep learning
- Predictable latency and explainability
- ONNX format for cross-platform inference

### 4. **Graceful Degradation**
- Circuit breakers for external dependencies
- Fallback pricing strategies
- Stale data tolerance with TTL policies

### 5. **Multi-Layer Caching**
- L1: Application memory cache (LRU)
- L2: Redis distributed cache
- L3: Database with read replicas

---

### Get Demand Forecast

```bash
curl http://localhost:8000/api/v1/forecast/demand?region_id=sg-central&horizon=60
```

---

## Performance Benchmarks

| Operation | P50 | P95 | P99 | Throughput |
|-----------|-----|-----|-----|------------|
| Price Compute | xxxms | xxxms | xxxms | xxxK req/s |
| Forecast Fetch | xxxms | xxxms | xxxms | xxxK req/s |
| Feature Retrieval | xxxms | xxxms | xxxms | xxxK req/s |

*Tested on: AWS xxx instances*

---

## Observability

### Metrics
- Request latency histograms
- Error rates and status codes
- Model inference latency
- Cache hit rates
- Kafka consumer lag

### Dashboards
- Grafana dashboards for real-time monitoring
- Business metrics (revenue, conversion)
- System health and resource utilization

### Alerting
- PagerDuty integration for critical alerts
- SLA breach notifications
- Anomaly detection on key metrics

---

## Testing Strategy

### Unit Tests (xxx% coverage target)
- Business logic validation
- Model inference correctness
- Feature computation accuracy

### Integration Tests
- End-to-end API workflows
- Database interactions
- Kafka producer/consumer pipelines

### Load Tests
- Sustained load: xxxK req/s for 1 hour
- Burst traffic: xxxK req/s for 5 minutes
- Chaos engineering with failure injection

---

## Deployment

### Docker Deployment

```bash
docker-compose -f deployments/docker/docker-compose.yml up -d
```

### Kubernetes Deployment

```bash
# Deploy with Helm
helm install pricing-system deployments/kubernetes/helm/pricing-system \
  --namespace production \
  --values deployments/kubernetes/helm/values-prod.yaml

# Check deployment status
kubectl get pods -n production
```

---

## Security Considerations

- **API Authentication**: JWT-based with OAuth2
- **Rate Limiting**: Token bucket algorithm (1000 req/min per user)
- **Data Encryption**: TLS 1.3 for transport, AES-256 at rest
- **PII Handling**: Anonymization and data retention policies
- **Secret Management**: HashiCorp Vault integration

---

## Documentation

- [Problem Statement](docs/problem-statement.md) - Business context and requirements
- [System Assumptions](docs/assumptions.md) - Design constraints and assumptions
- [Trade-offs Analysis](docs/trade-offs.md) - Key engineering decisions
- [Failure Scenarios](docs/failure-scenarios.md) - Fault tolerance strategies
- [Future Improvements](docs/future-improvements.md) - Enhancement roadmap
- [Architecture Overview](architecture/high-level.md) - System design
- [Sequence Diagrams](architecture/sequence-diagrams.md) - Interaction flows
- [Data Flow](architecture/data-flow.md) - Data pipeline architecture

---

## Contributing

This is a portfolio project. For questions or collaboration:
- Open an issue for bug reports
- Submit PRs for enhancements
- Follow conventional commits

---

## License

MIT License - see LICENSE file for details

---

## Author

**[Md. Nayeem Khan]**  
Senior Software Engineer

- LinkedIn: [linkedin.com/in/mdnayeemkhan/](https://linkedin.com/in/mdnayeemkhan/)
- Email: nayeem1505@gmail.com
- Portfolio: [nayeemkhan.dev](https://www.nayeemkhan.dev/)

---
