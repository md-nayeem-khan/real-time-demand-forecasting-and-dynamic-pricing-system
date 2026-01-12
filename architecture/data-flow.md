# Data Flow Architecture

## Overview

This document provides a comprehensive view of data flows within the Real-Time Demand Forecasting & Dynamic Pricing System, covering ingestion, processing, storage, and consumption patterns. It illustrates how data moves through the system from various sources to end-user applications.

---

## End-to-End Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            DATA SOURCES                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │   User      │  │   Supply    │  │  External   │  │  System Events   │    │
│  │  Actions    │  │  Updates    │  │    APIs     │  │  (Internal)      │    │
│  │             │  │             │  │             │  │                  │    │
│  │ • Searches  │  │ • Driver    │  │ • Weather   │  │ • Model Updates  │    │
│  │ • Bookings  │  │   Location  │  │ • Traffic   │  │ • Config Changes │    │
│  │ • Cancels   │  │ • Availability │ • Events    │  │ • Price Changes  │    │
│  │ • Views     │  │ • Status    │  │ • Holidays  │  │ • Alerts         │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘    │
│         │                │                │                  │              │
└─────────┼────────────────┼────────────────┼──────────────────┼───────────── ┘
          │                │                │                  │
          │                │                │                  │
          ▼                ▼                ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INGESTION LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────────────────────────────────────────────────────────────── ┐  │
│   │              Event Ingestion API (FastAPI)                           │  │
│   │  • Schema Validation (Avro/Protobuf)                                 │  │
│   │  • Deduplication (idempotency_key)                                   │  │
│   │  • Rate Limiting (per user, per region)                              │  │
│   │  • Enrichment (timestamp, region_id, metadata)                       │  │
│   └───────────────────────────┬───────────────────────────────────────── ┘  │
│                               │                                             │
│                               │ Produce Events                              │
│                               ▼                                             │
│   ┌──────────────────────────────────────────────────────────────────── ─┐  │
│   │                    Kafka / Redpanda Cluster                          │  │
│   │                                                                      │  │
│   │  Topics:                                                             │  │
│   │  ├─ user-events          (bookings, searches, cancels)               │  │
│   │  ├─ supply-events        (driver updates, availability)              │  │
│   │  ├─ external-events      (weather, traffic, event data)              │  │
│   │  ├─ demand-signals       (aggregated demand metrics)                 │  │
│   │  ├─ pricing-decisions    (audit trail of price computations)         │  │
│   │  └─ system-events        (config changes, model updates)             │  │
│   │                                                                      │  │
│   │  Configuration:                                                      │  │
│   │  • Partitions: 12 per topic (by region_id hash)                      │  │
│   │  • Replication Factor: 3                                             │  │
│   │  • Min In-Sync Replicas: 2                                           │  │
│   │  • Retention: 7 days (compliance + replay capability)                │  │
│   │  • Compression: Snappy (balance speed/ratio)                         │  │
│   └───────────────────────┬─────────────────┬────────────────────────── ─┘  │
│                           │                 │                               │
└───────────────────────────┼─────────────────┼───────────────────────────────┘
                            │                 │
                            │ Consume         │ Consume
                            ▼                 ▼
┌──────────────────────────────────┐  ┌──────────────────────────────────────┐
│   STREAM PROCESSING LAYER        │  │     ANALYTICS PIPELINE               │
├──────────────────────────────────┤  ├──────────────────────────────────────┤
│                                  │  │                                      │
│  Apache Flink / Spark Streaming  │  │  Kafka Connect → ClickHouse          │
│                                  │  │                                      │
│  ┌────────────────────────────┐  │  │  ┌────────────────────────────────┐  │
│  │ Windowing Operations       │  │  │  │ Batch Ingestion                │  │
│  ├────────────────────────────┤  │  │  ├────────────────────────────────┤  │
│  │ • Tumbling: 1min, 5min     │  │  │  │ • Micro-batches (10K events)   │  │
│  │ • Sliding: 15min overlaps  │  │  │  │ • Bulk insert for efficiency   │  │
│  │ • Session: 30min inactivity│  │  │  │ • Schema evolution handling    │  │
│  └────────────────────────────┘  │  │  └────────────────────────────────┘  │
│                                  │  │                                      │
│  ┌────────────────────────────┐  │  │  ┌────────────────────────────────┐  │
│  │ Aggregation Logic          │  │  │  │ Data Warehouse Tables          │  │
│  ├────────────────────────────┤  │  │  ├────────────────────────────────┤  │
│  │ • booking_count_1min       │  │  │  │ • events_raw (all events)      │  │
│  │ • search_volume_5min       │  │  │  │ • demand_hourly (aggregated)   │  │
│  │ • cancel_rate_15min        │  │  │  │ • pricing_decisions (audit)    │  │
│  │ • available_drivers        │  │  │  │ • model_performance (metrics)  │  │
│  │ • utilization_ratio        │  │  │  │ • user_segments (cohorts)      │  │
│  └────────────────────────────┘  │  │  └────────────────────────────────┘  │
│                                  │  │                                      │
│  ┌────────────────────────────┐  │  │  ┌────────────────────────────────┐  │ 
│  │ Enrichment & Joins         │  │  │  │ Query Patterns                 │  │
│  ├────────────────────────────┤  │  │  ├────────────────────────────────┤  │
│  │ • Join user + supply events│  │  │  │ • Time-series analysis         │  │
│  │ • Lookup region metadata   │  │  │  │ • Cohort analysis              │  │
│  │ • Correlate weather data   │  │  │  │ • Revenue reporting            │  │
│  └────────────────────────────┘  │  │  │ • A/B test analysis            │  │
│                                  │  │  └────────────────────────────────┘  │
│  ┌────────────────────────────┐  │  │                                      │
│  │ Feature Engineering        │  │  │  Retention: 2 years                  │
│  ├────────────────────────────┤  │  │  Compression: 10:1 (columnar)        │
│  │ • rolling_avg_demand_7d    │  │  │  Partitioning: By date + region      │
│  │ • demand_trend             │  │  │                                      │
│  │ • seasonality_score        │  │  └──────────────────────────────────────┘
│  │ • time_since_last_booking  │  │
│  │ • supply_demand_ratio      │  │
│  └────────────────────────────┘  │
│                                  │
│         Write Features ▼         │
│                                  │
└──────────────┬───────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          STORAGE LAYER                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────┐  ┌───────────────────────────────────┐   │
│  │   ONLINE FEATURE STORE        │  │   OFFLINE FEATURE STORE           │   │
│  │   (Redis Cluster)             │  │   (S3 + Parquet / Feast)          │   │
│  ├───────────────────────────────┤  ├───────────────────────────────────┤   │
│  │ Data Model:                   │  │ Data Model:                       │   │
│  │   Key: features:{region_id}   │  │   Path: s3://features/            │   │
│  │   Value: Hash                 │  │          {entity}/{date}/         │   │
│  │     {                         │  │          features.parquet         │   │
│  │       "hour_of_day": 14,      │  │                                   │   │
│  │       "day_of_week": 5,       │  │ Partitioning:                     │   │
│  │       "rolling_avg_7d": 1250, │  │   • By entity (region_id)         │   │
│  │       "demand_trend": 0.87,   │  │   • By date (YYYY-MM-DD)          │   │
│  │       "available_drivers": 45,│  │                                   │   │
│  │       "weather_temp": 28.5    │  │ Format:                           │   │
│  │     }                         │  │   • Parquet (columnar)            │   │
│  │   TTL: 120 seconds            │  │   • Snappy compression            │   │
│  │                               │  │                                   │   │
│  │ Configuration:                │  │ Usage:                            │   │
│  │   • Cluster Mode: 6 nodes     │  │   • Model training (batch)        │   │
│  │   • Replication: Master+Slave │  │   • Backfill historical features  │   │
│  │   • Eviction: LRU             │  │   • Feature validation            │   │ 
│  │   • Memory: 64GB per node     │  │   • Data science exploration      │   │
│  │   • QPS: 50K reads/s          │  │                                   │   │
│  │                               │  │ Retention: 2 years                │   │
│  └───────────────────────────────┘  └───────────────────────────────────┘   │
│                                                                             │
│  ┌───────────────────────────────┐  ┌───────────────────────────────────┐   │
│  │   PRICING RULES & METADATA    │  │   MODEL ARTIFACTS                 │   │
│  │   (PostgreSQL)                │  │   (S3 + MLflow)                   │   │ 
│  ├───────────────────────────────┤  ├───────────────────────────────────┤   │ 
│  │ Tables:                       │  │ Storage:                          │   │ 
│  │   • pricing_rules             │  │   • s3://models/                  │   │
│  │     - region_id               │  │     {model_name}/                 │   │
│  │     - base_price              │  │     {version}/                    │   │ 
│  │     - min_multiplier          │  │     model.onnx                    │   │
│  │     - max_multiplier          │  │                                   │   │
│  │     - effective_date          │  │ Metadata (MLflow):                │   │
│  │                               │  │   • Model version                 │   │
│  │   • audit_logs                │  │   • Training metrics (MAPE, etc.) │   │
│  │     - request_id              │  │   • Hyperparameters               │   │
│  │     - timestamp               │  │   • Feature importance            │   │
│  │     - region_id               │  │   • Training dataset reference    │   │
│  │     - computed_price          │  │   • Deployment status             │   │
│  │     - factors                 │  │                                   │   │
│  │     - model_version           │  │ Versioning:                       │   │
│  │                               │  │   • Semantic versioning           │   │
│  │   • ab_test_configs           │  │   • Immutable artifacts           │   │
│  │     - experiment_id           │  │   • Rollback capability           │   │
│  │     - variants                │  │                                   │   │
│  │     - traffic_split           │  │                                   │   │
│  │                               │  │                                   │   │
│  │ Replication:                  │  │                                   │   │
│  │   • 1 Primary + 2 Replicas    │  │                                   │   │
│  │   • Read-write splitting      │  │                                   │   │
│  │   • Auto-failover (RDS/GCS)   │  │                                   │   │
│  └───────────────────────────────┘  └───────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ Read Features & Models
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ML INFERENCE LAYER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                     ML Inference Service (ONNX Runtime)                │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                        │ │
│  │  Request Flow:                                                         │ │
│  │  1. Receive inference request (region_id, timestamp)                   │ │
│  │  2. Fetch features from Redis (online store)                           │ │
│  │     • Cache hit: ~2ms                                                  │ │
│  │     • Cache miss: Recompute from Kafka state (slower fallback)         │ │
│  │  3. Load model from memory (warm cache)                                │ │
│  │     • Model loaded on startup                                          │ │
│  │     • LRU cache: 3 recent versions                                     │ │
│  │  4. Run inference (XGBoost → ONNX)                                     │ │
│  │     • Input: 50-100 features                                           │ │
│  │     • Output: demand_score, confidence                                 │ │
│  │     • Latency: ~4ms P50, ~8ms P99                                      │ │
│  │  5. Return forecast with metadata                                      │ │
│  │                                                                        │ │
│  │  Model Management:                                                     │ │
│  │  • A/B Testing: Traffic split between model versions                   │ │
│  │  • Canary Rollout: Gradual traffic shift (10% → 50% → 100%)            │ │
│  │  • Shadow Mode: New model logs predictions without serving             │ │
│  │  • Rollback: Instant switch to previous version                        │ │
│  │                                                                        │ │
│  │  Optimization:                                                         │ │
│  │  • ONNX quantization (INT8) for faster inference                       │ │
│  │  • Batch inference (up to 32 requests) for throughput                  │ │
│  │  • Model warming: Pre-load on startup                                  │ │
│  │  • Feature preprocessing cache                                         │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   │ Forecast Result
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       BUSINESS LOGIC LAYER                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        Pricing Engine Service                          │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                        │ │
│  │  Inputs:                                                               │ │
│  │  • Demand forecast (from ML Inference)                                 │ │
│  │  • Supply metrics (from Feature Store)                                 │ │
│  │  • Pricing rules (from PostgreSQL)                                     │ │
│  │  • External factors (weather, events)                                  │ │
│  │                                                                        │ │
│  │  Pricing Algorithm:                                                    │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │ │
│  │  │ 1. Base Price Retrieval                                          │  │ │
│  │  │    base_price = get_base_price(region_id, product_type)          │  │ │
│  │  │                                                                  │  │ │
│  │  │ 2. Demand Factor Computation                                     │  │ │
│  │  │    demand_factor = sigmoid(demand_score - threshold)             │  │ │
│  │  │    # Maps 0-1 score to multiplier contribution                   │  │ │
│  │  │                                                                  │  │ │
│  │  │ 3. Supply Factor Computation                                     │  │ │
│  │  │    supply_factor = 1 / (1 + available_supply / demand)           │  │ │
│  │  │    # Scarcity increases price                                    │  │ │
│  │  │                                                                  │  │ │
│  │  │ 4. Time-of-Day Factor                                            │  │ │
│  │  │    time_factor = time_multiplier_map[hour_of_day]                │  │ │
│  │  │    # Peak hours (7-9am, 5-7pm) = 1.2x                            │  │ │
│  │  │                                                                  │  │ │
│  │  │ 5. External Event Factor                                         │  │ │
│  │  │    event_factor = 1.0 + (nearby_event_score * 0.5)               │  │ │
│  │  │    # Concert/sports → up to 1.5x                                 │  │ │
│  │  │                                                                  │  │ │
│  │  │ 6. Composite Surge Multiplier                                    │  │ │
│  │  │    surge_mult = 1.0 + (                                          │  │ │
│  │  │      demand_factor * 0.4 +                                       │  │ │
│  │  │      supply_factor * 0.3 +                                       │  │ │
│  │  │      time_factor * 0.2 +                                         │  │ │
│  │  │      event_factor * 0.1                                          │  │ │
│  │  │    )                                                             │  │ │
│  │  │                                                                  │  │ │
│  │  │ 7. Apply Business Constraints                                    │  │ │
│  │  │    surge_mult = clip(surge_mult, min_mult, max_mult)             │  │ │
│  │  │    # E.g., 1.0 - 3.0 range                                       │  │ │
│  │  │                                                                  │  │ │
│  │  │ 8. Final Price Computation                                       │  │ │
│  │  │    final_price = base_price * surge_mult                         │  │ │
│  │  │    final_price = round_to_nearest(final_price, 0.25)             │  │ │
│  │  └──────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                        │ │
│  │  Outputs:                                                              │ │
│  │  • final_price: Absolute price (e.g., $15.75)                          │ │
│  │  • surge_multiplier: Factor applied (e.g., 1.31x)                      │ │
│  │  • confidence: Model confidence score                                  │ │
│  │  • factors: Breakdown of contributing factors                          │ │
│  │    {                                                                   │ │
│  │      "demand": 0.45,                                                   │ │
│  │      "supply": -0.12,                                                  │ │
│  │      "time_of_day": 0.15,                                              │ │
│  │      "weather": 0.08,                                                  │ │
│  │      "events": 0.10                                                    │ │
│  │    }                                                                   │ │
│  │  • valid_until: Timestamp when price expires (e.g., 2min TTL)          │ │
│  │  • trace_id: Correlation ID for debugging                              │ │
│  │                                                                        │ │
│  │  Audit Trail:                                                          │ │
│  │  • Write to PostgreSQL (audit_logs table)                              │ │
│  │  • Publish to Kafka (pricing-decisions topic)                          │ │
│  │  • Include: request_id, region, price, factors, model_version          │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   │ Price Response
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API SERVING LAYER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       Pricing API (FastAPI)                            │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                        │ │
│  │  Endpoints:                                                            │ │
│  │  • POST /api/v1/pricing/compute                                        │ │
│  │    Compute dynamic price for a request                                 │ │
│  │                                                                        │ │
│  │  • GET /api/v1/forecast/demand?region_id=...&horizon=60                │ │
│  │    Retrieve demand forecast                                            │ │
│  │                                                                        │ │
│  │  • GET /api/v1/pricing/explain?request_id=...                          │ │
│  │    Explain a past pricing decision                                     │ │
│  │                                                                        │ │
│  │  • POST /api/v1/events                                                 │ │
│  │    Ingest user/supply events                                           │ │
│  │                                                                        │ │
│  │  Caching Strategy:                                                     │ │
│  │  ┌───────────────────────────────────────────────────────────────── ─┐ │ │
│  │  │ L1 Cache (In-Memory LRU):                                         │ │ │
│  │  │   • Per-pod cache                                                 │ │ │
│  │  │   • Size: 10K entries                                             │ │ │
│  │  │   • TTL: 30 seconds                                               │ │ │
│  │  │   • Hit Rate: ~60%                                                │ │ │
│  │  │                                                                   │ │ │
│  │  │ L2 Cache (Redis):                                                 │ │ │
│  │  │   • Shared across pods                                            │ │ │
│  │  │   • Key: price:{region}:{product}                                 │ │ │
│  │  │   • TTL: 60 seconds                                               │ │ │
│  │  │   • Hit Rate: ~95% (combined L1+L2)                               │ │ │
│  │  │                                                                   │ │ │
│  │  │ Cache Invalidation:                                               │ │ │
│  │  │   • TTL-based expiration (primary)                                │ │ │
│  │  │   • Proactive refresh (background workers)                        │ │ │
│  │  │   • Event-based invalidation (config changes)                     │ │ │
│  │  └───────────────────────────────────────────────────────────────── ─┘ │ │
│  │                                                                        │ │
│  │  Rate Limiting:                                                        │ │
│  │  • Per User: 1000 requests/minute                                      │ │
│  │  • Per IP: 5000 requests/minute                                        │ │
│  │  • Global: 50K requests/second (auto-scaling limit)                    │ │
│  │  • Algorithm: Token bucket                                             │ │
│  │                                                                        │ │
│  │  Response Format:                                                      │ │
│  │  {                                                                     │ │
│  │    "price": 15.75,                                                     │ │
│  │    "base_price": 12.00,                                                │ │
│  │    "surge_multiplier": 1.31,                                           │ │
│  │    "demand_score": 0.87,                                               │ │
│  │    "confidence": 0.94,                                                 │ │
│  │    "factors": {                                                        │ │
│  │      "demand": 0.45,                                                   │ │
│  │      "supply": -0.12,                                                  │ │
│  │      "time_of_day": 0.15,                                              │ │
│  │      "weather": 0.08,                                                  │ │
│  │      "events": 0.10                                                    │ │
│  │    },                                                                  │ │
│  │    "valid_until": "2026-01-11T10:32:00Z",                              │ │
│  │    "trace_id": "abc123def456",                                         │ │
│  │    "model_version": "v42",                                             │ │
│  │    "cached": false,                                                    │ │
│  │    "latency_ms": 28                                                    │ │
│  │  }                                                                     │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   │ Response
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA CONSUMERS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │   Mobile    │  │     Web     │  │  Internal   │  │   Monitoring     │    │
│  │    Apps     │  │   Portal    │  │  Dashboards │  │   & Analytics    │    │
│  │             │  │             │  │             │  │                  │    │
│  │ • Book ride │  │ • Admin     │  │ • Ops team  │  │ • Grafana        │    │
│  │ • View price│  │   config    │  │ • Pricing   │  │ • ClickHouse     │    │
│  │ • Track     │  │ • Reports   │  │   team      │  │ • Jupyter        │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Patterns

### Pattern 1: Hot Path (Real-Time Pricing)

```
User Request → API (2ms) → L1 Cache (hit: 60%)
                         ↓ (miss)
                         L2 Cache/Redis (hit: 95%)
                         ↓ (miss)
                         Pricing Engine (compute)
                           ├─ Forecast Service (8ms)
                           │  ├─ Feature Store (2ms)
                           │  └─ ML Inference (4ms)
                           └─ Rule Engine (3ms)
                         ↓
                         Response (total: ~28ms P50)

Data Volume: 5K requests/sec sustained, 12K peak
Latency Target: P99 < 50ms
```

---

### Pattern 2: Cold Path (Historical Analytics)

```
Kafka Events → Kafka Connect → ClickHouse
  (7-day retention)              (2-year retention)
                                      ↓
                                 Scheduled Queries
                                      ↓
                                 BI Dashboards
                                 (Grafana, Tableau)

Data Volume: 20K events/sec → 1.7B events/day
Query Latency: Seconds to minutes (not real-time)
```

---

### Pattern 3: Model Training Pipeline

```
Weekly Trigger (Airflow)
   ↓
Extract Features (S3 Offline Store)
   ↓
Train Models (Spark on Spot Instances)
   ↓
Validate & Register (MLflow)
   ↓
Canary Deployment (10% → 50% → 100%)
   ↓
Production Inference

Data Volume: 2 years × 1.7B events/day = 1.2TB training data
Pipeline Duration: 8 hours (automated)
```

---

## Data Volume & Growth Projections

| Metric | Current (Q1 2026) | 1-Year Projection | 2-Year Projection |
|--------|-------------------|-------------------|-------------------|
| **API Requests** | 5K/s | 15K/s (3x) | 50K/s (10x) |
| **Kafka Events** | 20K/s | 60K/s | 200K/s |
| **Redis Storage** | 50GB | 150GB | 500GB |
| **PostgreSQL** | 500GB | 1.5TB | 5TB |
| **ClickHouse** | 10TB | 30TB | 100TB |
| **S3 (Models + Features)** | 5TB | 15TB | 50TB |

---

## Related Documentation

- [High-Level Architecture](high-level.md) - Component overview
- [Sequence Diagrams](sequence-diagrams.md) - Interaction flows
- [Trade-offs](../docs/trade-offs.md) - Design decisions

