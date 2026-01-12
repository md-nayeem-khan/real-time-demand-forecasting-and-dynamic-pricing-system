# Failure Scenarios & Resilience Strategies

## Overview

This document catalogs potential failure scenarios, their impact on system operations, and the mitigation strategies implemented to ensure high availability, data consistency, and graceful degradation. Each scenario includes detection mechanisms, automated responses, and manual intervention procedures.

---

## 1. Kafka Broker Failure

### Scenario Description
One or more Kafka brokers become unavailable due to hardware failure, network partition, or resource exhaustion.

### Impact Assessment

| Severity | Component | Impact |
|----------|-----------|--------|
| **High** | Event Ingestion | Producers cannot write to affected partitions |
| **Medium** | Stream Processing | Consumer rebalancing causes temporary lag |
| **Low** | Pricing API | No direct impact (operates on cached data) |

### Symptoms & Detection

**Monitoring Alerts:**
- Kafka broker down (Prometheus: `kafka_server_replica_manager_under_replicated_partitions > 0`)
- Producer send errors spike (Prometheus: `kafka_producer_send_errors_total`)
- Consumer lag increases (Prometheus: `kafka_consumer_lag > 10000`)

**Logs:**
```
ERROR [Producer] Connection refused to broker kafka-2:9092
WARN [Consumer] Rebalancing in progress
```

**Time to Detection:** < 30 seconds (automated alerting)

---

### Mitigation Strategies

#### Design-Time Protections

1. **Multi-Broker Replication**
   - Replication factor: 3 (tolerates 2 broker failures)
   - Min in-sync replicas (ISR): 2 (ensures durability)
   - Partitions distributed across availability zones

2. **Producer Configuration**

3. **Consumer Group Rebalancing**
   - Multiple consumers in group (auto-rebalance)
   - Static group membership (reduce rebalance storms)
   - Heartbeat interval: 3s, session timeout: 10s

#### Runtime Responses

**Automatic Recovery:**
1. Kafka controller triggers leader election for affected partitions
2. Consumers detect rebalance and reconnect to new leaders
3. Producers retry failed sends with exponential backoff

**Operational Procedures:**
1. **Immediate (0-5 min):**
   - PagerDuty alert to on-call engineer
   - Automated runbook execution (check broker logs, restart if crashed)
   - Traffic rerouted to healthy brokers

2. **Short-term (5-30 min):**
   - Restart failed broker with increased resources
   - Manually reassign partitions if auto-rebalancing stuck
   - Monitor consumer lag recovery

3. **Long-term (>30 min):**
   - Replace failed broker if hardware issue
   - Post-incident review and capacity planning

**Graceful Degradation:**
- Feature computation delayed by 2-5 minutes (acceptable per DA-3 assumption)
- Pricing API continues serving cached forecasts (TTL: 5 min)
- No user-facing impact if recovery within 5 minutes

---

### Validation & Testing

**Chaos Engineering Tests:**
- Monthly: Kill random broker during peak traffic
- Quarterly: Simulate full AZ failure

**Success Criteria:**
- Zero data loss (verified via offset tracking)
- Consumer lag recovery < 10 minutes
- No pricing API errors during failure

**Historical Incidents:**
- **2024-08-15**: Broker OOM killed → Auto-restart in 45s, no data loss
- **2024-11-23**: Network partition → Consumer rebalance in 12s, 3min lag recovery

---

## 2. Model Inference Timeout / Service Crash

### Scenario Description
ML inference service becomes slow (>10ms P99) or crashes due to memory leak, OOM, or high load.

### Impact Assessment

| Severity | Component | Impact |
|----------|-----------|--------|
| **Critical** | Pricing API | Cannot generate real-time forecasts |
| **High** | Revenue | Potential 5-10% revenue loss if prolonged |
| **Medium** | User Experience | Increased latency or fallback pricing |

### Symptoms & Detection

**Monitoring Alerts:**
- Inference latency spike (Prometheus: `inference_duration_seconds_p99 > 0.05`)
- Model service HTTP 5xx errors (Prometheus: `http_requests_total{status=~"5.."}`)
- Memory usage threshold (Prometheus: `container_memory_usage_bytes > 8GB`)

**Logs:**
```
ERROR [Inference] Model inference timeout after 100ms
ERROR [API] Failed to get forecast: connection refused
```

**Time to Detection:** < 15 seconds

---

### Mitigation Strategies

#### Design-Time Protections

1. **Circuit Breaker Pattern**

   **States:**
   - **Closed**: Normal operation, requests pass through
   - **Open**: After 5 failures, stop sending requests for 30s
   - **Half-Open**: After 30s, try 1 request to test recovery

2. **Timeout Configuration**
   - Inference service timeout: 50ms (P99 budget: 10ms inference + 40ms network/processing)
   - API timeout: 100ms (allows retries)
   - Client timeout: 200ms (user-facing)

3. **Bulkhead Pattern**
   - Separate thread pools for critical vs. non-critical requests
   - Resource isolation prevents cascading failures

#### Runtime Responses

**Fallback Hierarchy:**

**Automatic Recovery:**

1. **Health Checks & Auto-Restart**
   - Kubernetes liveness probe: `GET /health` every 10s
   - Restart pod if 3 consecutive failures
   - Readiness probe ensures pod ready before traffic routing

2. **Horizontal Pod Autoscaler (HPA)**
   ```yaml
   minReplicas: 3
   maxReplicas: 20
   metrics:
   - type: Resource
     resource:
       name: cpu
       target:
         type: Utilization
         averageUtilization: 70
   ```

3. **Memory Leak Protection**
   - Pod memory limit: 8GB
   - OOMKilled pods automatically restarted
   - Preemptive restart every 24 hours to clear leaks

**Operational Procedures:**

1. **Immediate (0-2 min):**
   - Circuit breaker opens, traffic uses fallbacks
   - Auto-scaling triggers if load-related
   - PagerDuty alert to on-call

2. **Short-term (2-15 min):**
   - Rollback to previous model version if recent deployment
   - Manually scale up replicas if auto-scaling insufficient
   - Analyze logs for error patterns

3. **Long-term (>15 min):**
   - Emergency model retraining if model degradation suspected
   - Capacity planning if infrastructure undersized
   - Post-incident root cause analysis

---

### Validation & Testing

**Load Testing:**
- Weekly: Sustained 10K req/s for 1 hour
- Monthly: Burst to 25K req/s for 5 minutes

**Chaos Engineering:**
- Quarterly: Terminate random inference pods during peak traffic
- Verify fallback activation and recovery time

**Success Criteria:**
- Fallback activation within 2 seconds of inference failure
- User-facing error rate < 0.1%
- Revenue impact < 2% during degraded mode

---

## 3. Feature Store / Redis Unavailability

### Scenario Description
Redis cluster becomes unavailable or experiences high latency due to network issues, memory exhaustion, or failover.

### Impact Assessment

| Severity | Component | Impact |
|----------|-----------|--------|
| **High** | Feature Retrieval | Cannot fetch real-time features |
| **High** | Pricing Cache | Increased latency, DB overload |
| **Medium** | Forecast Quality | Degraded accuracy without fresh features |

### Symptoms & Detection

**Monitoring Alerts:**
- Redis cluster down (Prometheus: `redis_up == 0`)
- High latency (Prometheus: `redis_command_duration_seconds_p99 > 0.01`)
- Memory evictions (Prometheus: `redis_evicted_keys_total > 1000`)

**Time to Detection:** < 30 seconds

---

### Mitigation Strategies

#### Design-Time Protections

1. **Multi-Layer Cache**
   ```
   L1 Cache: In-memory LRU (per API pod)
     ↓ miss
   L2 Cache: Redis cluster
     ↓ miss
   L3 Source: PostgreSQL + recompute
   ```

2. **Default Feature Values**

3. **Redis High Availability**
   - Redis Sentinel for automatic failover
   - 3-node cluster with replication
   - Read replicas for query offloading

#### Runtime Responses

**Automatic Recovery:**

1. **Connection Retry Logic**

2. **Circuit Breaker for Redis**
   - After 10 failures, stop querying Redis for 60s
   - Use L1 cache + default features during circuit open

3. **Sentinel Failover**
   - Sentinel detects master failure within 5s
   - Promotes replica to master automatically
   - Client library handles reconnection

**Graceful Degradation:**

**Operational Procedures:**

1. **Immediate:**
   - Sentinel triggers automatic failover
   - Monitor failover completion (<10s expected)

2. **Short-term:**
   - Investigate root cause (memory, network, config)
   - Scale Redis cluster if memory pressure
   - Manually failback after issue resolved

3. **Long-term:**
   - Capacity planning for feature volume growth
   - Review TTL policies to reduce memory usage

---

### Validation & Testing

**Chaos Engineering:**
- Monthly: Kill Redis master during peak traffic
- Verify sentinel failover time < 10s
- Check feature retrieval success rate >99%

---

## 4. Database (PostgreSQL) Failure

### Scenario Description
Primary PostgreSQL instance fails or experiences slow queries (>100ms P99).

### Impact Assessment

| Severity | Component | Impact |
|----------|-----------|--------|
| **Medium** | Pricing Rules | Cannot fetch dynamic pricing config |
| **Medium** | Audit Logs | Cannot write audit trail |
| **Low** | API (reads) | Cached data serves most requests |

### Mitigation Strategies

#### Design-Time Protections

1. **Primary-Replica Setup**
   - 1 primary (writes) + 2 replicas (reads)
   - Automatic failover with AWS RDS/Cloud SQL

2. **Read-Write Splitting**

3. **Connection Pooling**

#### Runtime Responses

1. **Failover (RDS):**
   - Automatic promotion of replica to primary (60-120s)
   - DNS update to point to new primary

2. **Fallback for Reads:**
   - Use cached pricing rules (TTL: 1 hour)
   - Reads can tolerate staleness

3. **Fallback for Writes:**
   - Buffer audit logs in Kafka
   - Replay after database recovery

**Degradation:**
- Non-critical writes (audit logs) can be delayed
- Critical writes (pricing rule updates) queue and retry

---

## 5. External API Failure (Weather, Events, Traffic)

### Scenario Description
Third-party APIs (weather, traffic, events) become unavailable or rate-limited.

### Impact Assessment

| Severity | Component | Impact |
|----------|-----------|--------|
| **Low** | Forecast Accuracy | 3-5% accuracy degradation |
| **Low** | User Experience | No user-facing impact |

### Mitigation Strategies

1. **Circuit Breaker**
   - After 5 failures, stop querying for 5 minutes
   - Use cached values during circuit open

2. **Default Values**

3. **Non-Blocking Enrichment**
   - External data enriches forecast but not required
   - Async calls with timeout (2s max)

---

## 6. Multi-Region Outage

### Scenario Description
Entire AWS region (e.g., us-east-1) becomes unavailable.

### Impact Assessment

| Severity | Component | Impact |
|----------|-----------|--------|
| **Critical** | All Services | Region-specific pricing unavailable |
| **High** | Revenue | 100% revenue loss for that region |

### Mitigation Strategies

1. **Multi-Region Deployment**
   - Active-Active: Each region operates independently
   - Regional data isolation (no cross-region dependencies)

2. **DNS Failover**
   - Route53 health checks (30s interval)
   - Automatic traffic rerouting to healthy regions

3. **Data Replication**
   - Kafka MirrorMaker for cross-region replication (async)
   - Database cross-region read replicas

**Recovery Time:**
- DNS propagation: 1-5 minutes
- Data synchronization: 10-30 minutes

---

## 7. DDoS Attack / Traffic Spike

### Scenario Description
Sudden traffic spike (10x normal) due to DDoS attack or legitimate viral event.

### Mitigation Strategies

1. **Rate Limiting**

2. **Auto-Scaling**
   - HPA scales pods from 5 to 50 within 2 minutes
   - Cluster autoscaler adds nodes within 5 minutes

3. **CDN / WAF**
   - CloudFlare / AWS CloudFront for DDoS protection
   - Rate limiting at edge (before hitting API)

4. **Priority Queuing**
   - Premium users get higher priority
   - Shed low-priority traffic during overload

---

## 8. Model Drift / Accuracy Degradation

### Scenario Description
Model accuracy degrades over time due to changing demand patterns.

### Detection

**Monitoring:**
- MAPE increases by >20% week-over-week
- Prediction vs. actual demand divergence

**Alerts:**
- Daily model performance report
- Alert if MAPE > 20% for 3 consecutive days

### Mitigation

1. **Automated Retraining**
   - Trigger emergency retraining if drift detected
   - Retraining completes within 4 hours

2. **A/B Testing**
   - New model tested on 10% traffic before full rollout
   - Automatic rollback if performance worse

3. **Manual Overrides**
   - Ops team can apply rule-based pricing during drift
   - Override lasts until new model deployed

---

## Failure Impact Matrix

| Failure Scenario | Detection Time | Recovery Time | User Impact | Revenue Impact | Mitigation Effectiveness |
|------------------|----------------|---------------|-------------|----------------|------------------------|
| Kafka Broker Failure | <30s | 2-5 min | None (if <5min) | 0% | ✅ Excellent |
| Inference Timeout | <15s | <2 min | Low (fallback) | <2% | ✅ Excellent |
| Redis Unavailability | <30s | <10s | Low (cached) | <1% | ✅ Excellent |
| Database Failure | <60s | 1-2 min | Low (cached) | <5% | ✅ Good |
| External API Failure | <60s | N/A (degraded) | None | <3% | ✅ Good |
| Multi-Region Outage | <5 min | 5-30 min | High (region) | 100% (region) | ⚠️ Moderate |
| DDoS Attack | <1 min | Varies | Medium (delays) | 10-30% | ⚠️ Moderate |
| Model Drift | 1-3 days | 4-24 hours | Medium | 5-15% | ⚠️ Moderate |

---

## Resilience Testing Strategy

### Chaos Engineering Schedule

**Weekly:**
- Random pod termination (pricing API, inference service)
- Network latency injection (100ms delay)

**Monthly:**
- Kafka broker failure
- Redis cluster failover
- Database replica lag simulation

**Quarterly:**
- Full AZ failure
- Multi-region failover
- DDoS simulation (load testing)

### Game Days

**Bi-annual:** Full disaster recovery drill
- Scenario: Primary region down
- Team practices failover procedures
- RTO/RPO validation

---

## Related Documentation

- [Problem Statement](problem-statement.md) - Requirements driving resilience needs
- [System Assumptions](assumptions.md) - Assumptions about failure rates
- [Trade-offs](trade-offs.md) - Design decisions favoring availability
