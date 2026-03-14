# 84. Design a Circuit Breaker

---

## 1. Functional Requirements (FR)

- Monitor health of downstream service calls (success/failure rates)
- Automatically stop sending requests to unhealthy services (circuit OPEN)
- Periodically probe to check if service has recovered (HALF-OPEN state)
- Resume traffic when service recovers (circuit CLOSED)
- Configurable thresholds: failure rate, slow-call rate, minimum call volume
- Support per-service, per-endpoint, per-client circuit breakers
- Dashboard showing circuit breaker states across all services
- Integration with service mesh (Envoy/Istio sidecar) or application library
- Fallback responses when circuit is open (cached response, default value, degraded mode)
- Manual override: force-open or force-close circuits

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Overhead**: < 1µs per call decision (in-process check)
- **Fast Detection**: Detect failures within 5–10 seconds
- **Fast Recovery**: Resume traffic within seconds of downstream recovery
- **No SPOF**: Circuit breaker itself must not become a single point of failure
- **Consistency**: All instances of a service should have similar circuit state (eventual)
- **Observability**: Emit metrics for open/close transitions, fallback invocations

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Services in production | 500 |
| Inter-service RPC calls / sec | 10M |
| Circuit breakers (per service × endpoint) | ~5,000 |
| State transitions / min (normal) | < 10 |
| Metric data points / sec | 50K (10M calls × 5 dimensions sampled) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SERVICE A (Caller)                              │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  Application Code                                        │          │
│  │                                                          │          │
│  │  result = circuitBreaker.execute(() -> {                 │          │
│  │      return httpClient.call(serviceB, "/api/data");      │          │
│  │  }, fallback: () -> {                                    │          │
│  │      return cache.get("last_known_data");                │          │
│  │  });                                                     │          │
│  └──────────────────────┬───────────────────────────────────┘          │
│                         │                                              │
│  ┌──────────────────────▼───────────────────────────────────┐          │
│  │  Circuit Breaker (In-Process)                             │          │
│  │                                                          │          │
│  │  ┌─────────┐    ┌───────────┐    ┌─────────┐            │          │
│  │  │ CLOSED  │───►│   OPEN    │───►│HALF-OPEN│            │          │
│  │  │         │    │           │    │         │            │          │
│  │  │ Pass all│    │ Block all │    │ Allow   │            │          │
│  │  │ requests│    │ requests  │    │ N probe │            │          │
│  │  │         │    │ Return    │    │ requests│            │          │
│  │  │ Track   │◄───│ fallback  │◄───│         │            │          │
│  │  │ metrics │    │           │    │ If probe│            │          │
│  │  │         │    │ Wait for  │    │ succeeds│            │          │
│  │  │ If fail │    │ timeout   │    │ → CLOSED│            │          │
│  │  │ rate >  │    │ then →    │    │ If fails│            │          │
│  │  │ thresh  │    │ HALF-OPEN │    │ → OPEN  │            │          │
│  │  │ → OPEN  │    │           │    │         │            │          │
│  │  └─────────┘    └───────────┘    └─────────┘            │          │
│  │                                                          │          │
│  │  Sliding Window:                                         │          │
│  │  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐                       │          │
│  │  │✓ │✓ │✗ │✓ │✗ │✗ │✓ │✗ │✗ │✗ │  fail rate: 60%      │          │
│  │  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘  threshold: 50%      │          │
│  │  Last 10 calls (sliding window)     → TRIP OPEN!        │          │
│  └──────────────────────────────────────────────────────────┘          │
│                         │                                              │
│  ┌──────────────────────▼───────────────────────────────────┐          │
│  │  Metrics Emitter                                          │          │
│  │  - Circuit state transitions → Prometheus / StatsD        │          │
│  │  - Call outcomes (success/failure/timeout/rejected)        │          │
│  │  - Latency histogram                                      │          │
│  │  - Fallback invocations count                             │          │
│  └──────────────────────┬───────────────────────────────────┘          │
│                         │                                              │
└─────────────────────────│──────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   CONTROL PLANE (Optional)                             │
│                                                                        │
│  ┌────────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ Config Service     │  │ Metrics Aggregator│  │ Dashboard        │   │
│  │ (etcd / Consul)    │  │ (Prometheus)      │  │ (Grafana)        │   │
│  │                    │  │                   │  │                  │   │
│  │ - Threshold configs│  │ - Aggregate CB    │  │ - Service health │   │
│  │   per service      │  │   metrics from    │  │   map            │   │
│  │ - Manual overrides │  │   all instances   │  │ - Open circuits  │   │
│  │ - Feature flags    │  │ - Alert on open   │  │   alerts         │   │
│  │   for fallback     │  │   circuits        │  │ - Fallback rates │   │
│  │   behavior         │  │                   │  │ - Latency trends │   │
│  └────────────────────┘  └──────────────────┘  └──────────────────┘   │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Service Mesh Integration (Envoy / Istio)                      │    │
│  │                                                                │    │
│  │  Alternative to in-process CB: sidecar proxy handles CB logic  │    │
│  │  - outlier_detection in Envoy config                           │    │
│  │  - Ejection: remove unhealthy upstream from load balancer pool │    │
│  │  - Per-host circuit breaking (vs per-service in-process)       │    │
│  │  - Advantage: language-agnostic, centrally managed             │    │
│  │  - Disadvantage: less flexible fallback logic                  │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### State Machine Deep Dive

```
CLOSED → OPEN:
  Trigger: failure_rate > threshold (e.g., 50%) over sliding window
  AND: minimum_calls met (e.g., at least 20 calls in window)
  
  Sliding window types:
  1. Count-based: last N calls (e.g., last 100)
  2. Time-based: last T seconds (e.g., last 60 seconds)
  
  Failure types counted:
  - Exceptions / HTTP 5xx
  - Timeouts (call exceeds configured timeout)
  - Slow calls (call takes > slow_call_threshold, e.g., 5s)

OPEN → HALF-OPEN:
  After: wait_duration (e.g., 30 seconds)
  Purpose: give downstream time to recover

HALF-OPEN → CLOSED:
  If: permitted_calls_in_half_open succeed (e.g., 5 out of 5)
  Action: reset failure metrics, resume normal traffic

HALF-OPEN → OPEN:
  If: any probe call fails during half-open
  Action: restart wait_duration timer
  Optimization: exponential backoff on repeated failures
    30s → 60s → 120s → max 300s
```

---

## 5. APIs

### Configuration API

```
PUT /api/circuit-breakers/{service}/{endpoint}
{
  "failure_rate_threshold": 50,        // percentage
  "slow_call_rate_threshold": 80,      // percentage  
  "slow_call_duration_ms": 5000,       // what counts as slow
  "sliding_window_type": "COUNT",      // COUNT or TIME
  "sliding_window_size": 100,          // last 100 calls or last 100 seconds
  "minimum_calls": 20,                 // don't trip until this many calls
  "wait_duration_open_ms": 30000,      // how long to stay open
  "permitted_calls_half_open": 5,      // probe calls in half-open
  "fallback_type": "CACHE",           // CACHE | DEFAULT | DEGRADED | ERROR
  "manual_override": null              // FORCE_OPEN | FORCE_CLOSE | null
}

GET /api/circuit-breakers/status       → All circuit breaker states
GET /api/circuit-breakers/{service}/{endpoint}/status → Specific CB state
POST /api/circuit-breakers/{service}/{endpoint}/override
  { "state": "FORCE_OPEN" }           → Manual override
```

### Metrics API

```
GET /api/circuit-breakers/metrics?service=payment-service
{
  "circuit_state": "OPEN",
  "failure_rate": 67.5,
  "slow_call_rate": 45.0,
  "total_calls": 1500,
  "successful_calls": 487,
  "failed_calls": 1013,
  "not_permitted_calls": 2300,    // rejected while open
  "fallback_calls": 2300,
  "state_transitions": [
    {"from": "CLOSED", "to": "OPEN", "at": "2026-03-14T10:05:00Z"},
    {"from": "OPEN", "to": "HALF_OPEN", "at": "2026-03-14T10:05:30Z"},
    {"from": "HALF_OPEN", "to": "OPEN", "at": "2026-03-14T10:05:31Z"}
  ]
}
```

---

## 6. Data Models & Implementation

### In-Process Sliding Window (Count-Based)

```java
// Ring buffer implementation — O(1) per call, fixed memory
class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN }
    
    State state = CLOSED;
    int[] outcomes;          // ring buffer: 0=success, 1=failure, 2=slow
    int head = 0;
    int totalFailures = 0;
    int totalCalls = 0;
    long openedAt;
    Config config;
    
    Result execute(Supplier<Result> call, Supplier<Result> fallback) {
        if (state == OPEN) {
            if (System.nanoTime() - openedAt > config.waitDuration) {
                state = HALF_OPEN;
                halfOpenPermits = config.permittedCallsHalfOpen;
            } else {
                metrics.increment("not_permitted");
                return fallback.get();
            }
        }
        
        if (state == HALF_OPEN && halfOpenPermits <= 0) {
            return fallback.get();
        }
        
        try {
            long start = System.nanoTime();
            Result r = call.get();
            long duration = System.nanoTime() - start;
            recordSuccess(duration);
            return r;
        } catch (Exception e) {
            recordFailure();
            return fallback.get();
        }
    }
    
    void recordFailure() {
        // Update ring buffer, check threshold, possibly trip to OPEN
    }
}
```

### Metrics in Prometheus

```
# Circuit breaker state (0=closed, 1=open, 2=half-open)
circuit_breaker_state{service="payment", endpoint="/charge"} 1

# Call outcomes
circuit_breaker_calls_total{service="payment", outcome="success"} 487
circuit_breaker_calls_total{service="payment", outcome="failure"} 1013
circuit_breaker_calls_total{service="payment", outcome="not_permitted"} 2300

# State transitions
circuit_breaker_transitions_total{service="payment", from="closed", to="open"} 3

# Fallback invocations
circuit_breaker_fallback_total{service="payment", type="cache"} 2300
```

### Distributed Circuit Breaker State (Optional)

```
# Redis for cross-instance CB state sharing
HSET cb:payment:/charge state "OPEN" failure_rate 67.5 opened_at 1710410700

# Each instance publishes local metrics to Redis every 5 seconds
# Aggregation: if any instance sees > threshold → all instances open
# Trade-off: adds Redis latency to hot path
# Recommendation: keep CB in-process, share state for visibility only
```

---

## 7. Fault Tolerance & Deep Dives

### Cascading Failure Prevention

```
The core problem circuit breakers solve:

Without CB:
  Service A → Service B → Service C (down)
  
  C is down → B gets timeouts → B's thread pool exhausted →
  B starts timing out → A's thread pool exhausted →
  A starts timing out → Users see errors
  
  TOTAL SYSTEM DOWN because one leaf service failed

With CB:
  C is down → B's CB trips after 50% failures →
  B immediately returns fallback (no waiting for timeout) →
  B's thread pool stays healthy →
  A sees fast responses from B (with degraded data) →
  A remains healthy → Users see degraded but functional experience

Key insight: CB prevents resource exhaustion by failing fast
```

### Bulkhead Pattern (Complementary to CB)

```
Problem: Even with CB, a slow service can consume all threads

Solution: Bulkhead — isolate thread pools per downstream service
  Thread pool for Service B: 20 threads
  Thread pool for Service C: 10 threads
  Thread pool for Service D: 15 threads
  
  If Service C is slow → only 10 threads blocked
  Services B and D continue normally
  
  Implementation: 
  - Hystrix-style thread pool isolation
  - Semaphore isolation (lighter weight, no thread context switch)
  - Envoy: max_connections, max_pending_requests per upstream
```

### Timeout Strategy

```
Layered timeouts (defense in depth):
  1. Connection timeout: 1s (fail fast if can't connect)
  2. Read timeout: 5s (fail fast if service hangs)
  3. Circuit breaker slow-call threshold: 3s (count as slow)
  4. Retry budget: max 3 retries, total timeout 10s
  5. Deadline propagation: pass remaining budget downstream
  
  Without layered timeouts:
    Client timeout: 30s → waits 30s for each failing call → terrible UX
  
  With CB + timeouts:
    After 20 failures (each 5s timeout) → CB trips → instant fallback → good UX
```

### Service Mesh vs Library Implementation

```
Library (Resilience4j, Hystrix, Polly):
  ✅ Rich fallback logic (return cached data, degrade gracefully)
  ✅ Per-call configuration
  ✅ State visible in same process (zero latency check)
  ❌ Per-language implementation needed
  ❌ Configuration scattered across services

Service Mesh (Envoy/Istio):
  ✅ Language-agnostic (sidecar handles CB for any service)
  ✅ Centrally managed via control plane
  ✅ Outlier detection (per-host ejection, not just per-service)
  ❌ Limited fallback (can only return error, not cached data)
  ❌ Sidecar adds latency (~1ms per hop)

Recommendation: Use BOTH
  - Service mesh for basic outlier detection and connection-level CB
  - In-process library for application-level CB with rich fallbacks
```

### Distributed CB Coordination

```
Problem: 10 instances of Service A each have their own CB for Service B
  - Instance 1 sees 60% failure → trips CB
  - Instance 2 sees 30% failure → CB stays closed (different traffic pattern)
  - Inconsistent behavior

Solutions:
1. No coordination (simplest, recommended for most cases):
   - Each instance makes independent decisions
   - Eventually all converge (if B is truly down, all will trip)
   - Slight delay: instance 2 will trip 10-20s later

2. Shared metrics via Redis:
   - Aggregate failure rates across all instances
   - All instances read shared rate → consistent decisions
   - Adds Redis dependency to critical path (risky)

3. Control plane gossip:
   - Instances gossip CB state to each other
   - If > 50% of instances have tripped → all trip
   - Eventual consistency, no critical-path dependency

Recommendation: Option 1 for most cases. Option 3 for critical paths.
```

---

## 8. Additional Considerations

### Retry vs Circuit Breaker Interaction

```
Problem: Retries + CB can conflict

Bad pattern:
  retry(3 times, circuit_breaker(call_service_B))
  → Retries waste 3 attempts even when CB is open (CB returns fast)
  
Good pattern:
  circuit_breaker(retry(3 times, call_service_B))
  → CB wraps retries. When CB is open, no retries attempted.
  → During normal operation, retries handle transient failures.
  → When failures persistent → CB trips, stops all attempts.

Retry budget:
  Service mesh often enforces cluster-wide retry budget
  e.g., max 20% of calls can be retries → prevents retry storms
```

### Observability & Alerting

```
Alert rules:
  - Circuit opened: circuit_breaker_state == 1 for > 30s
  - High fallback rate: fallback_rate > 50% for > 5 min
  - Flapping: > 5 state transitions in 10 min (indicates partial outage)
  - All circuits open to one service: suggests service is down

Dashboard panels:
  1. Service dependency map with CB states (green/red/yellow)
  2. Failure rate trends per downstream service
  3. Latency heatmap (p50, p95, p99) 
  4. Fallback invocation rate
  5. Not-permitted call rate (requests rejected by open CB)
```

### Real-World Configurations

```
Low-risk service (product catalog):
  failure_rate_threshold: 70%     (tolerant)
  wait_duration: 10s              (quick retry)
  fallback: cached catalog data

High-risk service (payment):
  failure_rate_threshold: 30%     (conservative)
  wait_duration: 60s              (give it time)
  fallback: queue payment for later processing

Internal microservice:
  failure_rate_threshold: 50%
  slow_call_threshold: 2s
  wait_duration: 30s
  fallback: return error to caller (let caller decide)
```

### Testing Circuit Breakers

```
1. Chaos engineering: inject failures (Chaos Monkey, Gremlin)
   - Kill downstream service → verify CB trips
   - Inject latency → verify slow-call detection
   - Verify fallback returns correct degraded response

2. Load testing: simulate realistic traffic + failure patterns
   - Verify CB doesn't trip under normal load
   - Verify CB trips quickly under failure

3. Integration test: 
   - Configure CB with low thresholds (5 calls, 50% failure)
   - Send mix of success/failure
   - Assert CB state transitions
   - Assert metrics emitted correctly
```

