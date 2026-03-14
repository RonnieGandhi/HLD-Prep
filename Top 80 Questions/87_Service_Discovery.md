# 87. Design a Service Discovery System

---

## 1. Functional Requirements (FR)

- Services register themselves on startup (name, address, port, health endpoint, metadata)
- Services deregister on shutdown (graceful) or get auto-deregistered on failure
- Clients look up healthy instances of a service by name
- Health checking: periodic checks (HTTP, TCP, gRPC) to detect unhealthy instances
- Support for multiple environments/namespaces (prod, staging, per-tenant)
- Service metadata: version, region, weight, canary flags
- Watch/subscribe: clients get notified of changes (new instances, removals)
- DNS-based and API-based discovery (some clients can only do DNS)
- Load balancing integration: return instances in weighted/round-robin order

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.999% — if discovery is down, no service can call another
- **Low Latency**: Lookup in < 1ms (client-side caching with server push)
- **Consistency**: Eventually consistent with < 5s propagation of changes
- **Scalability**: 100K+ service instances, 1M+ lookups/sec
- **Fault Tolerance**: Continue operating during network partitions (AP bias with consistency checks)
- **Zero Downtime**: Rolling updates, cluster resizing without disruption

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Services | 5,000 distinct service names |
| Instances (total) | 100,000 |
| Registrations / sec | 50 (deploys, autoscaling) |
| Deregistrations / sec | 50 |
| Health checks / sec | 100K instances × 1 check/10s = 10K/sec |
| Lookups / sec | 1M (with client caching, most served locally) |
| Watch subscribers | 50K (one per client instance) |
| Metadata per instance | ~500 bytes |
| Total metadata | 100K × 500B = 50 MB (easily fits in memory) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    SERVICE INSTANCES                                    │
│                                                                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐      │
│  │ payment-svc│  │ payment-svc│  │ order-svc  │  │ user-svc   │      │
│  │ instance-1 │  │ instance-2 │  │ instance-1 │  │ instance-1 │      │
│  │            │  │            │  │            │  │            │      │
│  │ On startup:│  │            │  │            │  │            │      │
│  │ Register   │  │            │  │            │  │            │      │
│  │ with SD    │  │            │  │            │  │            │      │
│  │            │  │            │  │            │  │            │      │
│  │ Heartbeat  │  │            │  │            │  │            │      │
│  │ every 10s  │  │            │  │            │  │            │      │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └──────┬─────┘      │
│        │               │               │                 │            │
└────────│───────────────│───────────────│─────────────────│────────────┘
         │               │               │                 │
         └───────────────┼───────────────┼─────────────────┘
                         │               │
┌────────────────────────▼───────────────▼──────────────────────────────┐
│                SERVICE DISCOVERY CLUSTER                               │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  Service Registry (Consul / etcd / ZooKeeper / Eureka)      │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────┐              │      │
│  │  │  Consensus Layer (Raft / ZAB)              │              │      │
│  │  │  - 3 or 5 nodes for HA                     │              │      │
│  │  │  - Leader handles writes (register/dereg)  │              │      │
│  │  │  - All nodes handle reads                  │              │      │
│  │  │  - Data replicated to all nodes            │              │      │
│  │  └────────────────────────────────────────────┘              │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────┐              │      │
│  │  │  Health Checker                             │              │      │
│  │  │  - HTTP GET /healthz every 10s per instance │              │      │
│  │  │  - 3 consecutive failures → mark unhealthy │              │      │
│  │  │  - Recovery: 2 consecutive passes → healthy│              │      │
│  │  │  - Gossip-based distributed health (Consul)│              │      │
│  │  └────────────────────────────────────────────┘              │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────┐              │      │
│  │  │  Watch / Push Manager                       │              │      │
│  │  │  - Long-poll / gRPC streaming to clients   │              │      │
│  │  │  - Push updates on service changes         │              │      │
│  │  │  - Blocking query with index (Consul style)│              │      │
│  │  └────────────────────────────────────────────┘              │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────┐              │      │
│  │  │  DNS Interface                              │              │      │
│  │  │  - payment-svc.service.consul → A records  │              │      │
│  │  │  - SRV records with port info              │              │      │
│  │  │  - TTL: 0 or very short (5s)               │              │      │
│  │  │  - For legacy clients that only speak DNS  │              │      │
│  │  └────────────────────────────────────────────┘              │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    CONSUMING SERVICES (Clients)                        │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  Client Library / Sidecar Proxy                              │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────┐              │      │
│  │  │  Local Cache                                │              │      │
│  │  │  - Cached service → instances mapping      │              │      │
│  │  │  - Updated via watch/push or periodic poll │              │      │
│  │  │  - If SD cluster unreachable → use cache   │              │      │
│  │  │    (stale but available > nothing)          │              │      │
│  │  └────────────────────────────────────────────┘              │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────┐              │      │
│  │  │  Client-Side Load Balancer                  │              │      │
│  │  │  - Round-robin / weighted / least-connections│             │      │
│  │  │  - Circuit breaker per instance              │             │      │
│  │  │  - Retry with different instance on failure  │             │      │
│  │  └────────────────────────────────────────────┘              │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Server-Side vs Client-Side Discovery

```
Server-Side Discovery (ELB, Kubernetes Service):
  Client → Load Balancer → Backend Instance
  Load Balancer queries registry for healthy instances
  ✅ Client is simple (just knows LB address)
  ❌ Extra network hop, LB is potential bottleneck

Client-Side Discovery (Consul, Eureka):
  Client queries registry → gets list of instances → picks one directly
  ✅ No extra hop, client can make smart decisions (locality-aware)
  ❌ Client needs discovery library, per-language implementation

Service Mesh (Istio/Envoy sidecar):
  Client → local Envoy sidecar → Backend Instance
  Sidecar handles discovery, load balancing, circuit breaking
  ✅ Language-agnostic, rich features
  ❌ Resource overhead (sidecar per pod)

Kubernetes DNS (kube-dns/CoreDNS):
  payment-svc.namespace.svc.cluster.local → ClusterIP
  kube-proxy / IPVS handles load balancing
  ✅ Built-in, zero setup
  ❌ Limited health checking, DNS caching issues
```

---

## 5. APIs

```
# Registration
PUT /v1/agent/service/register
{
  "id": "payment-svc-i-abc123",
  "name": "payment-svc",
  "address": "10.0.1.42",
  "port": 8080,
  "tags": ["v2.3.1", "canary"],
  "meta": {"region": "us-east", "weight": "100"},
  "check": {
    "http": "http://10.0.1.42:8080/healthz",
    "interval": "10s",
    "timeout": "3s",
    "deregister_critical_service_after": "60s"
  }
}

# Deregistration
PUT /v1/agent/service/deregister/{service_id}

# Lookup healthy instances
GET /v1/health/service/{service_name}?passing=true&near=_agent&tag=v2.3.1

# Watch for changes (long-poll / blocking query)
GET /v1/health/service/{service_name}?passing=true&index=42&wait=30s
  → Returns immediately if index > 42 (changes occurred)
  → Blocks up to 30s if no changes

# KV store (for config, feature flags)
PUT /v1/kv/{key}   → Store config value
GET /v1/kv/{key}   → Read config value

# DNS interface
dig payment-svc.service.consul SRV
  → 1 0 8080 i-abc123.node.dc1.consul.
  → 1 0 8080 i-def456.node.dc1.consul.
```

---

## 6. Data Models

### Service Registry (In-Memory + Raft-Replicated)

```json
{
  "services": {
    "payment-svc": {
      "instances": {
        "payment-svc-i-abc123": {
          "id": "payment-svc-i-abc123",
          "address": "10.0.1.42",
          "port": 8080,
          "tags": ["v2.3.1", "canary"],
          "meta": {"region": "us-east", "weight": "100"},
          "health": "passing",
          "last_heartbeat": "2026-03-14T10:05:00Z",
          "registered_at": "2026-03-14T08:00:00Z",
          "check": {
            "type": "http",
            "endpoint": "http://10.0.1.42:8080/healthz",
            "interval_sec": 10,
            "timeout_sec": 3,
            "consecutive_failures": 0,
            "last_check": "2026-03-14T10:05:00Z"
          }
        },
        "payment-svc-i-def456": { ... }
      }
    },
    "order-svc": { ... }
  }
}
```

### Consul vs etcd vs ZooKeeper Comparison

| Feature | Consul | etcd | ZooKeeper |
|---|---|---|---|
| **Consensus** | Raft | Raft | ZAB |
| **Health checks** | ✅ Built-in (HTTP, TCP, script, gRPC, TTL) | ❌ Must build (TTL leases) | ❌ Must build (ephemeral nodes) |
| **DNS interface** | ✅ Built-in | ❌ External (SkyDNS) | ❌ External |
| **Service mesh** | ✅ Consul Connect | ❌ (use with Envoy) | ❌ |
| **Multi-DC** | ✅ WAN gossip federation | ❌ Single cluster | ❌ Single cluster |
| **KV store** | ✅ | ✅ (primary use case) | ✅ (znodes) |
| **Watch** | Long-poll (blocking queries) | gRPC watch stream | ZooKeeper watches |
| **Used by** | HashiCorp stack, Nomad | Kubernetes (API server) | Kafka, HBase, Hadoop |

---

## 7. Fault Tolerance & Deep Dives

### SD Cluster Failure

```
Problem: All service discovery nodes are down

Defense layers:
1. Raft cluster with 5 nodes → tolerates 2 failures
2. If majority lost → cluster read-only (can't register new services)
3. Clients use local cache → continue routing to last-known instances
4. Client-side health checking as backup (ping instances directly)
5. On recovery: services re-register, health checks resume

Key principle: Service-to-service calls must NOT depend on SD availability
  → Client caches must be robust enough to last through SD outages
  → Graceful degradation: stale routing > no routing
```

### Anti-Entropy & Convergence

```
Problem: Different SD nodes may have slightly different views

Consul approach (gossip-based anti-entropy):
1. SERF gossip protocol: nodes share state updates peer-to-peer
2. Anti-entropy sync: every 30s, each agent syncs full state with servers
3. On conflict: latest write wins (LWW with Lamport timestamps)

Result: all nodes converge within seconds (typically < 5s)

Health check distribution (Consul):
  - Health checks NOT centralized on servers
  - Each local agent health-checks services on its node
  - Agent gossips health status to servers
  - Scales: 100K instances → 100K agents, each checking ~10 local services
```

### Graceful Deployment with Service Discovery

```
Blue-Green Deployment:
  1. Deploy v2 instances → register with tag "v2"
  2. Both v1 and v2 receive traffic
  3. Gradually shift weight: v1=90/v2=10 → v1=50/v2=50 → v1=0/v2=100
  4. Deregister v1 instances

Canary Deployment:
  1. Register 1 canary instance with tag "canary"
  2. Clients with canary routing rule → route 5% to canary
  3. Monitor error rates
  4. If healthy → promote to stable; if unhealthy → deregister canary

Connection Draining:
  1. Before deregistration → mark instance as "draining"
  2. SD stops sending new requests to draining instance
  3. Wait for in-flight requests to complete (30s grace)
  4. Then fully deregister
```

---

## 8. Additional Considerations

### Kubernetes Native Service Discovery
```
In K8s, service discovery is built-in:
  - Service object: payment-svc.namespace.svc.cluster.local
  - Endpoints: automatically updated when pods start/stop
  - kube-proxy: IPVS rules for load balancing
  - CoreDNS: DNS resolution of service names

When to use external SD (Consul) in K8s:
  - Multi-cluster discovery (cross K8s clusters)
  - VM + K8s hybrid environments
  - Advanced health checks beyond K8s readiness probes
  - Service mesh with non-K8s services
```

### Self-Registration vs Third-Party Registration

```
Self-Registration (Eureka, Consul agent):
  - Service registers itself on startup
  - Sends heartbeats to maintain registration
  ✅ Service knows its own state best
  ❌ Every service needs registration logic

Third-Party Registration (Kubernetes, Registrator):
  - External component watches for new service instances (container events)
  - Registers/deregisters on behalf of services
  ✅ Services don't need discovery-aware code
  ❌ Registrator is another component to manage
```

