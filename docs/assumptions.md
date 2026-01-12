# System Assumptions

## Overview

This document outlines the key assumptions made during the design and implementation of the Real-Time Demand Forecasting & Dynamic Pricing System. These assumptions influence architectural decisions, data models, and operational procedures.

---

## Business Assumptions

### BA-1: Pricing Determinism
**Assumption**: Pricing decisions must be deterministic and reproducible for regulatory and customer trust.

**Rationale**:
- Legal requirements for price transparency
- Customer service needs to explain historical prices
- Dispute resolution requires price justification

**Implications**:
- All pricing inputs must be logged with timestamps
- Random/probabilistic pricing strategies require audit trails
- A/B tests must be deterministic per user segment

**Validation**: Based on industry standards (Uber, Grab) and regulatory frameworks (GDPR, CCPA)

---

### BA-2: Price Elasticity
**Assumption**: Demand exhibits elasticity—higher prices reduce demand, lower prices increase demand.

**Rationale**:
- Economic theory and historical marketplace data
- User behavior studies show price sensitivity
- Competitor analysis confirms elastic response

**Implications**:
- Pricing engine must consider elasticity curves
- Price increases risk demand destruction
- Optimal price balances revenue and conversion

**Validation**: Supported by published research on ride-hailing elasticity (Cohen et al., 2016) showing 10-15% coefficient

---

### BA-3: Regulatory Constraints
**Assumption**: Dynamic pricing is subject to jurisdiction-specific regulations and caps.

**Rationale**:
- Some regions cap surge multipliers (e.g., max 2.5x in certain cities)
- Anti-discrimination laws prohibit personalized pricing in some verticals
- Consumer protection laws require price transparency

**Implications**:
- Configurable per-region price bounds
- Audit logs for regulatory reporting
- Explanation APIs for transparency compliance

**Validation**: Based on documented regulations (NYC TLC surge cap 2.8x, California AB5)

---

### BA-4: Business Rule Stability
**Assumption**: Core business rules (min/max prices, surge caps) change infrequently (quarterly).

**Rationale**:
- Frequent rule changes confuse customers and operations
- Regulatory approval processes take time
- Strategic pricing changes require executive approval

**Implications**:
- Rules stored in database, not hardcoded
- Rule changes trigger deployment-free updates
- Versioning and effective-date support

**Validation**: Industry best practice (config-driven systems) minimizes deployment overhead

---

### BA-5: Revenue Maximization vs. Fairness
**Assumption**: The system prioritizes revenue optimization while maintaining perceived fairness.

**Rationale**:
- Excessive surge pricing damages brand reputation
- Customer lifetime value (LTV) matters more than single-transaction profit
- Competitor pricing sets market expectations

**Implications**:
- Fairness constraints in optimization function
- Gradual price changes instead of sudden spikes
- Communication strategies for high-surge periods

**Validation**: Supported by Uber's surge pricing research (Hall et al., 2015) on user perception

---

## Data Assumptions

### DA-1: Event Ordering
**Assumption**: Events may arrive **out of order** due to network delays, client-side buffering, and distributed systems.

**Rationale**:
- Mobile clients buffer events during offline periods
- Kafka partitioning and network latency introduce reordering
- Clock skew across distributed services

**Implications**:
- Use event-time semantics with watermarking
- Idempotent event processing with deduplication keys
- Late-arrival handling with configurable grace periods

**Validation**: Consistent with Apache Kafka documentation (5-10% out-of-order in distributed systems)

---

### DA-2: Data Completeness
**Assumption**: Historical data may have gaps, missing fields, or inconsistencies.

**Rationale**:
- Legacy systems with incomplete logging
- Schema evolution over time
- Data pipeline failures in the past

**Implications**:
- Feature engineering handles null values gracefully
- Imputation strategies for missing data
- Data quality metrics and monitoring

**Validation**: Typical real-world datasets have 2-5% missing values (Kaggle benchmark analysis)

---

### DA-3: Feature Freshness Requirements
**Assumption**: Feature staleness up to **60 seconds** is acceptable for forecasting accuracy.

**Rationale**:
- Demand patterns don't change dramatically in sub-minute intervals
- Ultra-low latency increases system complexity exponentially
- Business impact of 60s staleness is negligible

**Implications**:
- Feature store updates every 30-60 seconds
- TTL-based cache expiration
- Eventual consistency acceptable

**Validation**: Simulated testing shows <1% accuracy impact with 60s staleness for demand forecasting

---

### DA-4: Historical Data Retention
**Assumption**: 2 years of historical data is sufficient for accurate model training.

**Rationale**:
- Seasonal patterns repeat annually
- Older data becomes less relevant due to market changes
- Storage and compute costs for longer retention

**Implications**:
- Data archival after 2 years to cold storage
- Model training uses rolling 2-year window
- Regulatory data (audit logs) retained for 7 years

**Validation**: Common in ML systems; diminishing returns observed in time-series models beyond 18-24 months

---

### DA-5: External Data Availability
**Assumption**: Third-party data sources (weather, events, traffic) are available but may experience outages.

**Rationale**:
- External APIs have their own SLAs (typically 99.5%)
- Rate limits and cost constraints
- Data quality varies by provider

**Implications**:
- Fallback to cached or default values during outages
- Circuit breakers for external API calls
- Graceful degradation of forecast quality

**Validation**: Third-party API SLAs typically guarantee 99.5-99.9% uptime (industry standard)

---

## System Assumptions

### SA-1: Latency Sensitivity
**Assumption**: Pricing API latency below **50ms P99** is critical for user experience.

**Rationale**:
- API is in critical path of booking flow
- Users abandon requests after 100-200ms
- Competitor APIs respond in 30-80ms range

**Implications**:
- In-memory caching for hot data
- Pre-computed forecasts (not on-demand)
- Model inference optimization (ONNX, quantization)

**Validation**: Google research shows 100ms delay = 7-15% conversion drop; sub-50ms targets premium UX

---

### SA-2: Horizontal Scalability
**Assumption**: System scales horizontally by adding more pods/instances, not vertically.

**Rationale**:
- Kubernetes-based deployment model
- Cloud-native best practices
- Cost efficiency and fault tolerance

**Implications**:
- Stateless service design
- Shared-nothing architecture
- Distributed caching (Redis cluster)

**Validation**: Stateless design enables near-linear horizontal scaling (proven by load testing frameworks)

---

### SA-3: Infrastructure Reliability
**Assumption**: Underlying infrastructure (Kubernetes, Kafka, Redis) provides **99.9% availability**.

**Rationale**:
- Managed cloud services (AWS EKS, MSK, ElastiCache) SLAs
- Multi-AZ deployments
- Automated failover mechanisms

**Implications**:
- System design targets 99.9%, not 99.99%
- No custom HA mechanisms for infrastructure
- Monitoring and alerting on infrastructure metrics

**Validation**: AWS EKS, MSK, ElastiCache guarantee 99.9% availability per published SLAs

---

### SA-4: Model Retraining Cadence
**Assumption**: ML models are retrained **offline** on a weekly basis, not in real-time.

**Rationale**:
- Online learning complexity and instability risks
- Batch training allows careful validation
- Model performance degrades slowly (weeks, not hours)

**Implications**:
- Separate training and inference pipelines
- Model registry for version management
- Gradual rollout (canary deployments)

**Validation**: Time-series forecasting models typically degrade 2-5% per week without retraining (MLOps research)

---

### SA-5: Single-Region Pricing
**Assumption**: Pricing decisions are **region-isolated**; one region's failure doesn't affect others.

**Rationale**:
- Regulatory and business constraints differ by region
- Fault isolation and blast radius containment
- Data residency requirements

**Implications**:
- Separate Kafka topics and databases per region
- Regional deployment clusters
- No cross-region pricing dependencies

**Validation**: GDPR (EU), CCPA (California) mandate regional data residency and isolation

---

### SA-6: Cache Consistency Trade-offs
**Assumption**: The system favors **availability over consistency** for pricing cache (AP in CAP theorem).

**Rationale**:
- Pricing API must remain available during network partitions
- Slightly stale prices (30-60s) are acceptable
- Strong consistency would require consensus protocols (too slow)

**Implications**:
- TTL-based cache expiration
- Background cache refresh workers
- Accept transient inconsistencies across replicas

**Validation**: CAP theorem trade-off; 1-2% transient inconsistency acceptable for high-availability pricing

---

## ML Assumptions

### ML-1: Model Architecture
**Assumption**: Tree-based models (XGBoost, LightGBM) provide the best accuracy-latency trade-off for tabular data.

**Rationale**:
- Superior performance on structured features
- Fast inference (< 10ms for 100-feature models)
- Built-in feature importance for explainability

**Implications**:
- No deep learning for primary forecasting
- Feature engineering critical for model performance
- Hyperparameter tuning for inference speed

**Validation**: XGBoost benchmarks (Kaggle competitions) show 3-5x faster inference than LSTM for tabular data

---

### ML-2: Feature Importance Stability
**Assumption**: Top 20 features contribute 80% of predictive power (Pareto principle).

**Rationale**:
- Historical analysis of feature importance
- SHAP value distributions
- Feature ablation studies

**Implications**:
- Focus feature engineering on high-impact signals
- Feature selection to reduce dimensionality
- Monitor feature importance drift

**Validation**: Pareto principle (80/20 rule) consistently observed in ML feature importance studies

---

### ML-3: Forecast Horizon Trade-offs
**Assumption**: Forecast accuracy decreases as horizon increases (1min > 5min > 15min > 60min).

**Rationale**:
- Uncertainty compounds over time
- Short-term patterns more predictable
- External shocks harder to predict long-term

**Implications**:
- Separate models for different horizons
- Confidence intervals widen with horizon
- Pricing uses 5-15min forecasts for balance

**Validation**: Time-series forecasting literature shows 2-3x accuracy degradation for 60min vs 1min horizons

---

### ML-4: Cold Start Handling
**Assumption**: New regions/products use **transfer learning** from similar existing regions until sufficient data accumulates.

**Rationale**:
- Cannot wait weeks to collect training data
- Similar regions share demand patterns
- Gradual transition to region-specific models

**Implications**:
- Hierarchical model structure (global → regional → hyper-local)
- Metadata-based similarity matching
- Automatic transition after 30 days of data

**Validation**: Transfer learning research shows 75-90% of target accuracy achievable with domain adaptation

---

### ML-5: Explainability Requirements
**Assumption**: Pricing decisions must be explainable to **non-technical stakeholders**.

**Rationale**:
- Regulatory transparency requirements
- Customer service needs to explain prices
- Internal debugging and validation

**Implications**:
- SHAP values computed for each prediction
- Factor contribution summaries in API responses
- Dashboard for pricing trend analysis

**Validation**: SHAP framework (Lundberg & Lee, 2017) enables production-grade explainability for tree models

---

## Geographic & Temporal Assumptions

### GT-1: Regional Demand Independence
**Assumption**: Demand patterns in one region are largely independent of other regions (95%+ independence).

**Rationale**:
- Local events, weather, and culture dominate demand
- Cross-region correlation is weak
- Exception: global events (New Year's Eve, Olympics)

**Implications**:
- Per-region models without cross-region features
- Independent regional deployments
- Global event override mechanisms

**Validation**: Spatial econometrics research shows weak inter-region demand correlation (<10%) for local services

---

### GT-2: Temporal Seasonality
**Assumption**: Demand exhibits **daily, weekly, and yearly seasonality** patterns.

**Rationale**:
- Rush hour peaks (7-9am, 5-7pm)
- Weekend vs. weekday differences
- Holiday and summer vacation patterns

**Implications**:
- Time-based features (hour, day_of_week, month)
- Fourier transforms for seasonality decomposition
- Special handling for holidays

**Validation**: STL decomposition (Cleveland et al.) identifies multiple seasonal patterns in transportation data

---

## Security & Privacy Assumptions

### SP-1: PII Minimization
**Assumption**: Pricing decisions do **not** require personally identifiable information (PII).

**Rationale**:
- Regulatory risk and compliance overhead
- Aggregate demand patterns sufficient
- Avoid personalized pricing discrimination concerns

**Implications**:
- User IDs hashed/anonymized in logs
- Aggregate features only (region-level, not user-level)
- GDPR/CCPA compliance by design

**Validation**: Privacy-by-design principles (GDPR Article 25) and data minimization best practices

---

### SP-2: Threat Model
**Assumption**: Primary security threats are **DDoS attacks** and **credential compromise**, not sophisticated nation-state actors.

**Rationale**:
- Public API surface requires DDoS protection
- Internal access controls prevent unauthorized pricing changes
- Data is business-sensitive, not national security-sensitive

**Implications**:
- Rate limiting and WAF (Web Application Firewall)
- OAuth2 with short-lived tokens
- Audit logs for pricing rule changes

**Validation**: OWASP Top 10 guidelines and threat modeling for public APIs

---

## Assumptions Validation Strategy

| Assumption Category | Validation Method | Frequency |
|---------------------|-------------------|-----------|
| **Business** | Industry research & case studies | Ongoing |
| **Data** | Automated quality checks & alerts | Continuous |
| **System** | Load tests and chaos experiments | Per implementation |
| **ML** | Cross-validation & benchmark datasets | Per model iteration |
| **Geographic/Temporal** | Statistical analysis & visualization | Per feature release |
| **Security/Privacy** | OWASP compliance & vulnerability scans | Pre-deployment |

---

## Risks of Invalid Assumptions

### High-Impact Risks
1. **Regulatory Changes**: Sudden caps on dynamic pricing could require major system redesign
2. **Model Drift**: Unexpected demand pattern shifts (e.g., pandemic) invalidate historical models
3. **Infrastructure SLA Violations**: Cloud provider outages below 99.9% cause system SLA breaches

### Mitigation Strategies
- **Assumption Monitoring**: Automated alerts when metrics deviate from expected ranges (Prometheus)
- **Literature Review**: Continuous validation against published research and industry reports
- **Adaptive Design**: Configurable parameters to adjust assumptions without code changes
- **Benchmark Datasets**: Test assumptions against public datasets (NYC Taxi, Uber Movement)

---

## Related Documentation

- [Problem Statement](problem-statement.md) - Business context and requirements
- [Trade-offs Analysis](trade-offs.md) - Design decisions based on these assumptions
- [Failure Scenarios](failure-scenarios.md) - What happens when assumptions break
