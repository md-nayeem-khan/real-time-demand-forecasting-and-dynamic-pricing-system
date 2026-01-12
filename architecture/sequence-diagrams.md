# Sequence Diagrams

## Overview

This document provides detailed sequence diagrams for key interaction flows within the Real-Time Demand Forecasting & Dynamic Pricing System. Each diagram illustrates the request flow, component interactions, and decision points.

---

## 1. Price Computation Request Flow

### Standard Success Path

```
┌────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Client │    │   API    │    │ Pricing  │    │ Forecast │    │    ML    │    │ Feature  │
│        │    │ Gateway  │    │  Engine  │    │ Service  │    │Inference │    │  Store   │
└───┬────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
    │              │               │               │               │               │
    │ POST /price  │               │               │               │               │
    ├─────────────>│               │               │               │               │
    │              │               │               │               │               │
    │              │ Rate Limit    │               │               │               │
    │              │ Check         │               │               │               │
    │              │               │               │               │               │
    │              │ compute_price │               │               │               │
    │              ├──────────────>│               │               │               │
    │              │               │               │               │               │
    │              │               │ get_forecast  │               │               │
    │              │               ├──────────────>│               │               │
    │              │               │               │               │               │
    │              │               │               │ get_features  │               │
    │              │               │               ├──────────────>│               │
    │              │               │               │               │               │
    │              │               │               │               │ fetch (Redis) │
    │              │               │               │               ├──────────────>│
    │              │               │               │               │               │
    │              │               │               │               │ {features}    │
    │              │               │               │               │<──────────────┤
    │              │               │               │               │               │
    │              │               │               │  predict()    │               │
    │              │               │               │  [XGBoost]    │               │
    │              │               │               │               │               │
    │              │               │               │ {forecast}    │               │
    │              │               │               │<──────────────┤               │
    │              │               │               │               │               │
    │              │               │ {demand_score}│               │               │
    │              │               │<──────────────┤               │               │
    │              │               │               │               │               │
    │              │               │ [Parallel]    │               │               │
    │              │               │ get_rules()   │               │               │
    │              │               │ [PostgreSQL]  │               │               │
    │              │               │               │               │               │
    │              │               │ compute:      │               │               │
    │              │               │ - surge_mult  │               │               │
    │              │               │ - final_price │               │               │
    │              │               │ - factors     │               │               │
    │              │               │               │               │               │
    │              │               │ [Async]       │               │               │
    │              │               │ write_audit() │               │               │
    │              │               │ cache_price() │               │               │
    │              │               │               │               │               │
    │              │ {price_response}              │               │               │
    │              │<──────────────┤               │               │               │
    │              │               │               │               │               │
    │ 200 OK       │               │               │               │               │
    │ {price_data} │               │               │               │               │
    │<─────────────┤               │               │               │               │
    │              │               │               │               │               │
    └──────────────┴───────────────┴───────────────┴───────────────┴───────────────┴────────

Latency Breakdown:
├─ API Gateway: 2ms
├─ Pricing Engine: 5ms
│  ├─ Forecast Service (gRPC): 8ms
│  │  ├─ Feature Store (Redis): 2ms
│  │  └─ Model Inference: 4ms
│  └─ Rule Engine (PostgreSQL): 3ms
└─ Total: ~22ms (P50), ~45ms (P99)
```

---

### Failure Path: Inference Timeout → Fallback

```
┌────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Client │    │   API    │    │ Pricing  │    │ Forecast │    │  Redis   │
│        │    │ Gateway  │    │  Engine  │    │ Service  │    │  Cache   │
└───┬────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
    │              │               │               │               │
    │ POST /price  │               │               │               │
    ├─────────────>│               │               │               │
    │              │               │               │               │
    │              │ compute_price │               │               │
    │              ├──────────────>│               │               │
    │              │               │               │               │
    │              │               │ get_forecast  │               │
    │              │               ├──────────────>│               │
    │              │               │               │               │
    │              │               │  TIMEOUT      │               │
    │              │               │   (50ms)      │               │
    │              │               │               │               │
    │              │               │ [Fallback L1] │               │
    │              │               │ get_cached    │               │
    │              │               ├──────────────────────────────>│
    │              │               │               │               │
    │              │               │               │ {cached_fcst} │
    │              │               │<──────────────────────────────┤
    │              │               │               │               │
    │              │               │ compute_price │               │
    │              │               │ (cached data) │               │
    │              │               │               │               │
    │              │               │  Log:         │               │
    │              │               │ "degraded"    │               │
    │              │               │               │               │
    │              │ {price_resp}  │               │               │
    │              │ degraded=true │               │               │
    │              │<──────────────┤               │               │
    │              │               │               │               │
    │ 200 OK       │               │               │               │
    │ {price_data} │               │               │               │
    │ + warning    │               │               │               │
    │<─────────────┤               │               │               │
    │              │               │               │               │
    └──────────────┴───────────────┴───────────────┴───────────────┴────────

Fallback Hierarchy:
1. Real-time ML inference (failed )
2. Cached forecast (success )
3. Historical average (not needed)
4. Static base price (not needed)
```

---

## 2. Event Ingestion to Feature Computation

```
┌────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Mobile │    │  Event   │    │  Kafka   │    │  Flink   │    │ Feature  │
│  App   │    │Ingestion │    │  Broker  │    │ Stream   │    │  Store   │
└───┬────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
    │              │               │               │               │
    │ User books   │               │               │               │
    │ ride         │               │               │               │
    │              │               │               │               │
    │ POST /events │               │               │               │
    ├─────────────>│               │               │               │
    │              │               │               │               │
    │              │ Validate:     │               │               │
    │              │ - Schema      │               │               │
    │              │ - Dedup       │               │               │
    │              │ - Rate        │               │               │
    │              │               │               │               │
    │              │ produce()     │               │               │
    │              ├──────────────>│               │               │
    │              │  Topic:       │               │               │
    │              │  user-events  │               │               │
    │              │  Key: region  │               │               │
    │              │               │               │               │
    │              │               │ ack (ISR=2)   │               │
    │              │               │<──────────────┤               │
    │              │               │               │               │
    │ 202 Accepted │               │               │               │
    │<─────────────┤               │               │               │
    │              │               │               │               │
    │              │               │               │ poll()        │
    │              │               │               │<──────────────┤
    │              │               │               │               │
    │              │               │ {batch}       │               │
    │              │               ├──────────────>│               │
    │              │               │               │               │
    │              │               │               │ Window:       │
    │              │               │               │ Tumbling 1min │
    │              │               │               │               │
    │              │               │               │ Aggregate:    │
    │              │               │               │ - bookings    │
    │              │               │               │ - searches    │
    │              │               │               │ - cancels     │
    │              │               │               │               │
    │              │               │               │ Compute:      │
    │              │               │               │ - rolling_avg │
    │              │               │               │ - trend       │
    │              │               │               │ - seasonality │
    │              │               │               │               │
    │              │               │               │ write_features│
    │              │               │               ├──────────────>│
    │              │               │               │               │
    │              │               │               │               │ Redis:
    │              │               │               │               │ HMSET
    │              │               │               │               │ EX 120
    │              │               │               │               │
    │              │               │               │ ack           │
    │              │               │               │<──────────────┤
    │              │               │               │               │
    │              │               │               │ commit offset │
    │              │               │               ├──────────────>│
    │              │               │               │               │ Kafka
    │              │               │               │               │
    └──────────────┴───────────────┴───────────────┴───────────────┴────────

Timeline:
├─ Event production: <100ms
├─ Kafka ingestion: <10ms
├─ Flink processing: ~30s (windowing)
└─ Feature available: ~45s total
```

---

## 3. Model Training & Deployment Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Airflow  │    │ Feature  │    │ Training │    │  Model   │    │    ML    │
│Scheduler │    │  Store   │    │ Pipeline │    │ Registry │    │Inference │
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │               │               │
     │ Trigger:      │               │               │               │
     │ Weekly        │               │               │               │
     │ (Sunday 2am)  │               │               │               │
     │               │               │               │               │
     │ start_job     │               │               │               │
     ├──────────────────────────────>│               │               │
     │               │               │               │               │
     │               │               │ extract_data  │               │
     │               │               │ (2yr history) │               │
     │               │               │               │               │
     │               │               │ fetch_offline │               │
     │               │  features     │<──────────────┤               │
     │               │<──────────────┤               │               │
     │               │               │               │               │
     │               │ {parquet}     │               │               │
     │               ├──────────────>│               │               │
     │               │               │               │               │
     │               │               │ Train:        │               │
     │               │               │ - XGBoost     │               │
     │               │               │ - Prophet     │               │
     │               │               │ - LightGBM    │               │
     │               │               │               │               │
     │               │               │ Validate:     │               │
     │               │               │ - MAPE < 15%  │               │
     │               │               │ - Backtest    │               │
     │               │               │ - A/B setup   │               │
     │               │               │               │               │
     │               │               │ register_model│               │
     │               │               ├──────────────>│               │
     │               │               │  version: v42 │               │
     │               │               │  metrics: ... │               │
     │               │               │  artifacts:   │               │
     │               │               │  - model.onnx │               │
     │               │               │               │               │
     │               │               │               │ stored        │
     │               │               │               │ (S3 + MLflow) │
     │               │               │               │               │
     │               │               │ ack           │               │
     │               │               │<──────────────┤               │
     │               │               │               │               │
     │               │               │ notify_deploy │               │
     │               │               │──────────────────────────────>│
     │               │               │  strategy:    │               │
     │               │               │  canary 10%   │               │
     │               │               │               │               │
     │               │               │               │               │ download
     │               │               │               │               │ v42.onnx
     │               │               │               │               │
     │               │               │               │               │ Phase 1:
     │               │               │               │               │ 10% traffic
     │               │               │               │               │ (1 hour)
     │               │               │               │               │
     │               │               │               │               │ Monitor:
     │               │               │               │               │ - Latency
     │               │               │               │               │ - Accuracy
     │               │               │               │               │ - Errors
     │               │               │               │               │
     │               │               │               │               │ Phase 2:
     │               │               │               │               │ 50% traffic
     │               │               │               │               │ (2 hours)
     │               │               │               │               │
     │               │               │               │               │ Phase 3:
     │               │               │               │               │ 100% traffic
     │               │               │               │               │
     │ job_complete  │               │               │               │
     │<──────────────────────────────┤               │               │
     │               │               │               │               │
     └───────────────┴───────────────┴───────────────┴───────────────┴────────

Timeline:
├─ Data extraction: 30min
├─ Model training: 2-4 hours
├─ Validation: 30min
├─ Canary rollout: 3-4 hours
└─ Total: ~8 hours (automated)
```

---

## 4. A/B Test Experiment Flow

```
┌────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Client │    │   API    │    │ Pricing  │    │ Exp      │    │ Metrics  │
│        │    │ Gateway  │    │  Engine  │    │ Service  │    │ Pipeline │
└───┬────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
    │              │               │               │               │
    │ POST /price  │               │               │               │
    ├─────────────>│               │               │               │
    │  user_id:    │               │               │               │
    │  abc123      │               │               │               │
    │              │               │               │               │
    │              │ compute_price │               │               │
    │              ├──────────────>│               │               │
    │              │               │               │               │
    │              │               │ get_variant   │               │
    │              │               ├──────────────>│               │
    │              │               │ user: abc123  │               │
    │              │               │ exp: surge_v2 │               │
    │              │               │               │               │
    │              │               │               │ Hash(user_id) │
    │              │               │               │ % 100         │
    │              │               │               │               │
    │              │               │               │ if < 50:      │
    │              │               │               │   variant = A │
    │              │               │               │ else:         │
    │              │               │               │   variant = B │
    │              │               │               │               │
    │              │               │ variant: B    │               │
    │              │               │<──────────────┤               │
    │              │               │               │               │
    │              │               │ compute_price │               │
    │              │               │ (strategy B)  │               │
    │              │               │               │               │
    │              │               │ log_exposure  │               │
    │              │               ├──────────────>│               │
    │              │               │               │               │
    │              │               │               │ write_kafka   │
    │              │               │               ├──────────────>│
    │              │               │               │ Topic:        │
    │              │               │               │ experiments   │
    │              │               │               │               │
    │              │ {price}       │               │               │
    │              │ variant: B    │               │               │
    │              │<──────────────┤               │               │
    │              │               │               │               │
    │ 200 OK       │               │               │               │
    │<─────────────┤               │               │               │
    │              │               │               │               │
    │              │               │               │               │
    │ [User books] │               │               │               │
    │              │               │               │               │
    │ POST /book   │               │               │               │
    ├─────────────>│               │               │               │
    │              │               │               │               │
    │              │ log_conversion│               │               │
    │              ├──────────────────────────────────────────────>│
    │              │               │               │               │
    │              │               │               │               │ Aggregate:
    │              │               │               │               │ - Conv rate
    │              │               │               │               │ - Revenue
    │              │               │               │               │ - By variant
    │              │               │               │               │
    │              │               │               │               │ Dashboard:
    │              │               │               │               │ Variant A:
    │              │               │               │               │   Conv: 12%
    │              │               │               │               │   Rev: $50K
    │              │               │               │               │ Variant B:
    │              │               │               │               │   Conv: 14%
    │              │               │               │               │   Rev: $55K
    │              │               │               │               │ Winner: B ✓
    │              │               │               │               │
    └──────────────┴───────────────┴───────────────┴───────────────┴────────

Experiment Lifecycle:
1. Define hypothesis & variants
2. Randomize user assignment (hash-based)
3. Log exposures & conversions
4. Collect metrics (min 7 days)
5. Statistical significance test
6. Rollout winner / rollback loser
```

---

## 5. Circuit Breaker Activation Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Pricing  │    │ Forecast │    │ Circuit  │    │ Fallback │
│  Engine  │    │ Service  │    │ Breaker  │    │ Handler  │
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │               │
     │ get_forecast  │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │               │  Timeout      │               │
     │               │               │               │
     │               │<──────────────┤ failure_count │
     │               │               │ = 1           │
     │               │               │               │
     │ get_forecast  │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │               │  Timeout      │               │
     │               │               │               │
     │               │<──────────────┤ failure_count │
     │               │               │ = 2           │
     │               │               │               │
     │ ...           │               │               │
     │               │               │               │
     │ get_forecast  │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │               │  Timeout      │               │
     │               │               │               │
     │               │<──────────────┤ failure_count │
     │               │               │ = 5           │
     │               │               │               │
     │               │               │ OPEN CIRCUIT  │
     │               │               │               │
     │               │               │               │
     │               │               │ start_timer   │
     │               │               │ (30s)         │
     │               │               │               │
     │ get_forecast  │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │               │               │ Circuit OPEN  │
     │               │               │ Fail Fast     │
     │               │<──────────────┤               │
     │               │               │               │
     │  CircuitOpen  │               │               │
     │<──────────────┤               │               │
     │               │               │               │
     │               │               │               │ fallback
     ├──────────────────────────────────────────────>│
     │               │               │               │
     │               │               │               │ get_cached
     │               │               │               │ forecast
     │               │               │               │
     │ {cached_fcst} │               │               │
     │<──────────────────────────────────────────────┤
     │               │               │               │
     │ [30s later]   │               │               │
     │               │               │               │
     │               │               │ HALF-OPEN     │
     │               │               │ (test 1 req)  │
     │               │               │               │
     │ get_forecast  │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │               │  Success      │               │
     │<──────────────┤               │               │
     │               │               │               │
     │               │               │ CLOSE CIRCUIT │
     │               │               │  Recovered    │
     │               │               │               │
     │               │               │ reset_counter │
     │               │               │               │
     └───────────────┴───────────────┴───────────────┴────────

Circuit Breaker States:
- CLOSED: Normal operation
- OPEN: Fail fast for 30s
- HALF-OPEN: Test 1 request
- CLOSED: Recovery confirmed
```

---

## 6. Health Check & Auto-Restart Flow

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│Kubernetes│    │    ML    │    │  Metrics │    │ PagerDuty│
│  Kubelet │    │Inference │    │ Monitor  │    │  Alert   │
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │               │
     │ [Every 10s]   │               │               │
     │               │               │               │
     │ GET /health   │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │               │ check:        │               │
     │               │ - DB conn     │               │
     │               │ - Redis       │               │
     │               │ - Model       │               │
     │               │               │               │
     │ 200 OK        │               │               │
     │<──────────────┤               │               │
     │               │               │               │
     │ ...           │               │               │
     │               │               │               │
     │ [Pod OOM]     │               │               │
     │               │  Crash        │               │
     │               │               │               │
     │ GET /health   │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │  Timeout      │               │               │
     │               │               │               │
     │ GET /health   │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │  Timeout      │               │               │
     │               │               │               │
     │ GET /health   │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │  Timeout      │               │               │
     │  (3 failures) │               │               │
     │               │               │               │
     │ RESTART POD   │               │               │
     │               │               │               │
     │               │               │               │
     │               │ [Restarting]  │               │
     │               │ Load model    │               │
     │               │ Init Redis    │               │
     │               │               │               │
     │               │               │               │ pod_restart
     │               │               │               │ increment
     │               │               │<──────────────┤
     │               │               │               │
     │               │               │ if restarts   │
     │               │               │ > 3 in 5min:  │
     │               │               │               │
     │               │               │ send_alert    │
     │               │               ├──────────────>│
     │               │               │  Severity:    │
     │               │               │  HIGH         │
     │               │               │               │
     │               │               │               │  Page
     │               │               │               │ On-Call
     │               │               │               │
     │ GET /health   │               │               │
     ├──────────────>│               │               │
     │               │               │               │
     │ 200 OK        │               │               │
     │<──────────────┤               │               │
     │  [Recovered]  │               │               │
     │               │               │               │
     │ Mark Ready    │               │               │
     │ Route Traffic │               │               │
     │               │               │               │
     └───────────────┴───────────────┴───────────────┴────────

Kubernetes Probe Config:
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
  
readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

---

## Related Documentation

- [High-Level Architecture](high-level.md) - Component overview
- [Data Flow](data-flow.md) - Data pipeline details
- [Failure Scenarios](../docs/failure-scenarios.md) - Error handling strategies
