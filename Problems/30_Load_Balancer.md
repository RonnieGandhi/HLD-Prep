# 30. Design a Load Balancer

---

## 1. Functional Requirements (FR)

- **Distribute traffic**: Route incoming requests across a pool of backend servers
- **Health checks**: Detect unhealthy servers and stop routing traffic to them
- **Multiple algorithms**: Support Round Robin, Weighted Round Robin, Least Connections, IP Hash, etc.
- **Session affinity (sticky sessions)**: Route requests from the same client to the same server
- **SSL/TLS termination**: Decrypt HTTPS at the load balancer (offload from backends)
- **Auto-scaling integration**: Dynamically add/remove backend servers
- **Rate limiting**: Limit requests per client/IP
- **Request routing**: Route by URL path, headers, or other request attributes (Layer 7)
- **Connection draining**: Gracefully drain connections before removing a server

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.999% — LB is the single entry point; if it's down, everything is down
- **Ultra-Low Latency**: Add < 1 ms overhead per request
- **High Throughput**: Handle millions of concurrent connections, 100K+ requests/sec
- **Scalability**: Scale with traffic; must not be a bottleneck
- **Fault Tolerant**: LB failure must not cause total outage (redundancy required)
- **No Single Point of Failure**: Active-passive or active-active LB pairs
- **Transparency**: Clients should be unaware of the load balancer (pass-through)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Requests / sec | 1M |
| Concurrent connections | 10M |
| Backend servers | 1,000 |
| Bandwidth | 100 Gbps |
| Health check interval | 5 seconds |
| Health checks / sec | 1,000 / 5 = 200 |

---

## 4. High-Level Design (HLD)

```
                         ┌──────────────┐
                         │    Clients   │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │     DNS      │  ← Returns VIP (Virtual IP)
                         └──────┬───────┘
                                │
                  ┌─────────────┼─────────────┐
                  │             │             │
           ┌──────▼──────┐     │      ┌──────▼──────┐
           │    LB 1     │◄───VRRP───▶│    LB 2     │
           │  (Active)   │  Keepalived │  (Passive)  │
           └──────┬──────┘             └─────────────┘
                  │
         ┌────────┼────────┬────────┐
         │        │        │        │
   ┌─────▼──┐ ┌───▼───┐ ┌──▼───┐ ┌──▼───┐
   │Server 1│ │Server 2│ │Srvr 3│ │Srvr N│
   └────────┘ └───────┘ └──────┘ └──────┘
```

### Layer 4 vs. Layer 7 Load Balancing

#### Layer 4 (Transport Layer — TCP/UDP)
- **How**: Makes routing decision based on IP address and TCP/UDP port
- **Does NOT** inspect packet content (no HTTP headers, no URL)
- **Mechanism**: Network Address Translation (NAT) or Direct Server Return (DSR)
- **Performance**: Extremely fast (kernel-level, hardware-accelerated)
- **Use cases**: High throughput TCP traffic, database connections, gaming servers
- **Examples**: AWS NLB, F5 BIG-IP, LVS (Linux Virtual Server)

**NAT-Based L4 Load Balancing**:
```
Client → LB (VIP: 10.0.0.1:80) → LB rewrites dst IP to backend (10.0.1.5:80)
Backend → LB → LB rewrites src IP back to VIP → Client

Client sees: 10.0.0.1 (doesn't know about backend)
```

**Direct Server Return (DSR)**:
```
Client → LB → LB forwards packet to backend (only changes MAC, not IP)
Backend → responds DIRECTLY to client (bypasses LB on return path!)

Advantage: LB only handles inbound traffic → 10× more throughput
           (outbound is much larger than inbound for web traffic)
```

#### Layer 7 (Application Layer — HTTP/HTTPS)
- **How**: Inspects HTTP headers, URL, cookies, body to make routing decisions
- **Can do**: URL-based routing, header-based routing, cookie-based sticky sessions, A/B testing, request rewriting, compression, caching
- **Performance**: Slower than L4 (must parse HTTP), but much more flexible
- **Use cases**: Web applications, microservices, API gateways
- **Examples**: AWS ALB, NGINX, HAProxy, Envoy

**Content-Based Routing (L7)**:
```
Rule 1: /api/v1/users/*    → User Service (servers 1-5)
Rule 2: /api/v1/orders/*   → Order Service (servers 6-10)
Rule 3: /static/*          → CDN / Static Server
Rule 4: /admin/*           → Admin Service (servers 11-12)
Default:                    → Frontend Service
```

### Component Deep Dive

#### Load Balancing Algorithms — Deep Dive

**1. Round Robin**
- Each request goes to the next server in rotation: S1 → S2 → S3 → S1 → ...
- **Pros**: Simple, even distribution when servers are identical
- **Cons**: Ignores server capacity and current load

**2. Weighted Round Robin**
- Each server has a weight. Server with weight 3 gets 3× more traffic than weight 1
- **Use case**: Different-sized servers (8-core vs 16-core)
```
Weights: S1=3, S2=1, S3=2
Sequence: S1, S1, S1, S2, S3, S3, S1, S1, S1, S2, S3, S3, ...
```

**3. Least Connections** ⭐ (recommended for long-lived connections)
- Route to the server with the fewest active connections
- **Why**: Some requests take longer than others. Least connections naturally avoids overloading slow servers
- **Weighted variant**: `effective_load = active_connections / weight`
```
Server 1: 150 connections, weight 3 → effective = 50
Server 2: 80 connections, weight 1 → effective = 80
Server 3: 90 connections, weight 2 → effective = 45  ← route here
```

**4. IP Hash (Consistent)**
- `server = hash(client_IP) % num_servers`
- Same client always goes to the same server → natural sticky sessions
- **With consistent hashing**: Adding/removing servers only affects minimal clients
- **Use case**: Caching servers where cache locality matters

**5. Least Response Time**
- Route to the server with the lowest average response time
- LB tracks response times per server (exponential moving average)
- **Best for**: Heterogeneous servers with different performance characteristics

**6. Random**
- Randomly pick a server. Statistically even distribution at scale
- **Power of Two Random Choices**: Pick 2 random servers → route to the one with fewer connections. Surprisingly effective — exponentially better than single random choice

**7. Consistent Hashing with Bounded Loads**
- Consistent hashing + cap: no server gets more than `(1 + ε) × average_load`
- If the target server is overloaded → spill to the next server on the ring
- Google's approach for their internal LB

#### Health Checks

**Active Health Checks**:
```
Every 5 seconds:
  For each backend server:
    Send HTTP GET /health
    If response 200 within 2 seconds → healthy
    If 3 consecutive failures → mark UNHEALTHY → stop routing
    If 2 consecutive successes after unhealthy → mark HEALTHY → resume routing
```

**Passive Health Checks**:
- Monitor actual request/response patterns
- If server returns 5xx for > 50% of requests in last 30 seconds → circuit-break → mark unhealthy
- **Advantage**: No extra health check traffic; detects issues faster

**Combination**: Use both active (guaranteed detection) and passive (faster detection)

#### High Availability for the Load Balancer Itself

**VRRP (Virtual Router Redundancy Protocol)**:
```
LB 1 (Active)  ←── VRRP heartbeat ──→  LB 2 (Passive)
     │                                       │
  VIP: 10.0.0.1                        VIP: (standby)
  
If LB 1 fails:
  LB 2 detects missed heartbeat (< 3 seconds)
  LB 2 takes over VIP: 10.0.0.1
  Clients see no interruption (same IP)
```

**Implementation**: Keepalived daemon on both LB instances

**Active-Active Setup**:
- DNS returns multiple LB IPs (DNS round robin)
- Both LBs handle traffic simultaneously
- If one fails, DNS health check removes it (slow: DNS TTL)
- Better: Anycast — both LBs announce same IP via BGP

#### Connection Draining (Graceful Shutdown)

When removing a server from the pool:
1. Stop sending NEW connections to the server
2. Allow EXISTING connections to complete (with a timeout, e.g., 30 seconds)
3. After timeout → forcefully close remaining connections
4. Remove server from the pool

---

## 5. APIs

### Backend Management
```http
POST /api/v1/backends
{
  "address": "10.0.1.5:8080",
  "weight": 3,
  "max_connections": 1000,
  "health_check": {
    "path": "/health",
    "interval_sec": 5,
    "timeout_sec": 2,
    "unhealthy_threshold": 3,
    "healthy_threshold": 2
  }
}

DELETE /api/v1/backends/{backend_id}?drain_timeout=30

GET /api/v1/backends
Response: 200 OK
{
  "backends": [
    {
      "address": "10.0.1.5:8080",
      "status": "healthy",
      "active_connections": 150,
      "weight": 3,
      "requests_total": 1500000,
      "avg_response_time_ms": 45
    }
  ]
}
```

### LB Configuration
```http
PUT /api/v1/config
{
  "algorithm": "least_connections",
  "sticky_sessions": {
    "enabled": true,
    "cookie_name": "SERVERID",
    "ttl_seconds": 3600
  },
  "ssl": {
    "certificate": "...",
    "key": "..."
  },
  "routing_rules": [
    {"path": "/api/*", "backend_pool": "api-servers"},
    {"path": "/static/*", "backend_pool": "static-servers"}
  ]
}
```

---

## 6. Data Model

### LB Configuration State (in-memory + persisted)

```
BackendPool:
  pool_name: "api-servers"
  algorithm: LEAST_CONNECTIONS
  backends:
    - {address: "10.0.1.5:8080", weight: 3, status: HEALTHY, connections: 150}
    - {address: "10.0.1.6:8080", weight: 3, status: HEALTHY, connections: 142}
    - {address: "10.0.1.7:8080", weight: 2, status: UNHEALTHY, connections: 0}

RoutingRules:
  - {pattern: "/api/v1/users/*", pool: "user-service"}
  - {pattern: "/api/v1/orders/*", pool: "order-service"}
  - {default: "frontend-service"}
```

### Connection Table (L4 LB — NAT)

```
┌──────────────────┬───────────────────┬──────────────────┐
│ Client Address   │ Backend Address   │ State            │
├──────────────────┼───────────────────┼──────────────────┤
│ 203.0.113.5:4321 │ 10.0.1.5:8080    │ ESTABLISHED      │
│ 198.51.100.2:5678│ 10.0.1.6:8080    │ ESTABLISHED      │
│ 192.0.2.10:9012  │ 10.0.1.5:8080    │ TIME_WAIT        │
└──────────────────┴───────────────────┴──────────────────┘
```

### Sticky Session Store (L7 LB)

```
Cookie: SERVERID=10.0.1.5_8080; Path=/; Max-Age=3600

Or in Redis (for LB cluster):
  Key:    sticky:{client_id_hash}
  Value:  "10.0.1.5:8080"
  TTL:    3600
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **LB failure** | Active-passive pair with VRRP/Keepalived; failover < 3 seconds |
| **Backend failure** | Health check detects → removed from pool; traffic redistributed |
| **Connection table loss** | L4 NAT: connections drop on LB failover (clients reconnect). DSR: unaffected (return path doesn't go through LB) |
| **Thundering herd on server recovery** | Slow start: gradually increase traffic to recovered server (start at 10%, ramp to 100% over 30 seconds) |
| **LB overloaded** | DNS-based multi-LB, Anycast, or hardware LB for extreme scale |

### Specific: Graceful Handling of Backend Crashes

```
Timeline:
  T=0:  Server 3 crashes
  T=0:  In-flight requests to Server 3 fail → LB retries on Server 1 or 2
  T=5:  Active health check fails (1st attempt)
  T=10: Active health check fails (2nd attempt)
  T=15: Active health check fails (3rd attempt) → Server 3 marked UNHEALTHY
  T=15: All new traffic routed to Server 1 and 2 only
  
  Passive health check would detect it faster:
  T=0-2: > 50% of requests to Server 3 return errors → circuit-break → immediate UNHEALTHY
```

---

## 8. Additional Considerations

### Global Server Load Balancing (GSLB)

For multi-datacenter deployments:
```
                    ┌──────────┐
                    │   DNS    │  ← GeoDNS: routes to closest DC
                    └────┬─────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
     ┌──────▼──────┐ ┌───▼───┐ ┌─────▼─────┐
     │  DC: US-East│ │DC: EU │ │DC: AP-South│
     │  (LB → BEs) │ │(LB→BEs│ │(LB → BEs) │
     └─────────────┘ └───────┘ └────────────┘
```

- DNS resolves to the LB IP in the nearest datacenter
- If a DC goes down, DNS removes it → traffic shifts to surviving DCs
- Also used for: disaster recovery, compliance (data sovereignty), latency optimization

### TLS/SSL Termination

```
Client ──── HTTPS (encrypted) ──── LB ──── HTTP (unencrypted) ──── Backend

Benefits:
- Backend servers don't need to handle TLS (CPU savings)
- Centralized certificate management
- LB can inspect HTTP headers for L7 routing
```

### Modern Load Balancers: Service Mesh

In microservices architectures, instead of a centralized LB:
```
Service A → Sidecar Proxy (Envoy) → Service B's Sidecar Proxy → Service B

Each service has a sidecar proxy that handles:
- Load balancing (client-side)
- Service discovery
- Circuit breaking
- Retry/timeout
- mTLS (mutual TLS)
- Observability
```

- **Istio, Linkerd**: Service mesh implementations
- **Envoy**: The sidecar proxy used by most service meshes
- Eliminates the centralized LB bottleneck for east-west (service-to-service) traffic

### Comparison of LB Solutions

| Solution | Type | Layer | Scale | Use Case |
|---|---|---|---|---|
| **HAProxy** | Software | L4/L7 | Millions of conns | General purpose |
| **NGINX** | Software | L7 | 100K+ req/s | Web, reverse proxy |
| **Envoy** | Software | L4/L7 | Service mesh | Microservices |
| **AWS ALB** | Managed | L7 | Auto-scales | Cloud-native HTTP |
| **AWS NLB** | Managed | L4 | Millions of conns | TCP/UDP, ultra-low latency |
| **F5 BIG-IP** | Hardware | L4/L7 | 100 Gbps+ | Enterprise, legacy |
| **Maglev** | Software (Google) | L3/L4 | Millions of pps | Google's internal LB |

### Monitoring
- **Requests/sec** per backend (detect imbalanced distribution)
- **Error rate** per backend (detect unhealthy servers)
- **Latency percentiles** (p50, p95, p99) per backend
- **Active connections** per backend
- **LB CPU and memory** (is the LB itself a bottleneck?)
- **Health check success rate**
- **Connection queue depth** (requests waiting for a backend connection)

---

## 9. Deep Dive: Engineering Trade-offs

### L4 vs L7: Packet Path Walkthrough

```
L4 Load Balancer (NAT mode) — what happens to each packet:

  Client sends:
    SRC: 198.51.100.5:4321  DST: 10.0.0.1:80 (VIP)
    
  LB receives, selects Server 2 (10.0.1.6:8080):
    Rewrites DST: 10.0.0.1:80 → 10.0.1.6:8080
    Stores mapping in connection table:
      {198.51.100.5:4321} → {10.0.1.6:8080}
    Forwards packet:
    SRC: 198.51.100.5:4321  DST: 10.0.1.6:8080
    
  Server 2 responds:
    SRC: 10.0.1.6:8080  DST: 198.51.100.5:4321
    
  LB receives, looks up connection table:
    Rewrites SRC: 10.0.1.6:8080 → 10.0.0.1:80
    Forwards to client:
    SRC: 10.0.0.1:80  DST: 198.51.100.5:4321
    
  Client sees response from 10.0.0.1:80 (VIP) → transparent
  
  Cost: 2 rewrites per packet (DNAT inbound, SNAT outbound)
  Throughput: kernel-level NAT → millions of packets/sec
  Limitation: LB sees ALL traffic (both directions) → bottleneck for 
              bandwidth-heavy responses (e.g., video streaming)


L4 Load Balancer (DSR mode) — bypasses LB on return:

  Client sends:
    SRC: 198.51.100.5:4321  DST: 10.0.0.1:80 (VIP)
    
  LB receives, selects Server 2:
    Changes L2 header (MAC address) to Server 2's MAC
    Does NOT rewrite IP addresses
    Packet arrives at Server 2 with DST still = 10.0.0.1 (VIP)
    (Server 2 must have VIP configured on loopback interface)
    
  Server 2 responds DIRECTLY to client:
    SRC: 10.0.0.1:80 (VIP)  DST: 198.51.100.5:4321
    (Response goes straight to client, NOT through LB)
    
  Advantage: LB only handles inbound traffic
    For web traffic: request = 1 KB, response = 100 KB
    LB handles 1% of total traffic → 100× more capacity than NAT mode
    
  Requirement: LB and backends must be in same L2 network (same VLAN)
    DSR doesn't work across subnets (L2 rewrite only)


L7 Load Balancer — full HTTP inspection:

  Client sends:
    TLS ClientHello → LB terminates TLS → decrypts HTTP
    LB reads: GET /api/v1/users/123  Host: api.example.com
    
  LB routing decision:
    Path matches "/api/v1/users/*" → backend pool "user-service"
    Algorithm: least connections → Server 3 (fewest active)
    
  LB opens NEW TCP connection to Server 3:
    SRC: LB_internal_IP  DST: 10.0.1.7:8080
    Sends proxied HTTP request (adds X-Forwarded-For header)
    
  Server 3 responds → LB receives → re-encrypts → sends to client
  
  Cost: full TCP termination + TLS + HTTP parse on EVERY request
  Throughput: ~100K req/sec per LB instance (vs millions for L4)
  Advantage: content-based routing, caching, compression, WAF
```

### Connection Table Scaling

```
Problem: L4 NAT LB maintains a connection table entry for EVERY active connection
  10M concurrent connections × 128 bytes per entry = 1.28 GB
  
  Connection table operations:
    Insert: O(1) hash table insert per new connection
    Lookup: O(1) per packet (hash by 5-tuple: src_ip, dst_ip, src_port, dst_port, proto)
    Delete: O(1) on connection close or timeout
    
  Memory pressure:
    10M entries fits in RAM easily
    But: hash table with 10M entries → cache misses for random access
    At 1M packets/sec → 1M hash lookups/sec → CPU cache thrashing
    
  Solution: Kernel bypass (DPDK, XDP)
    Bypass the kernel network stack entirely
    Process packets in user space with poll-mode drivers
    Pre-allocate connection table in huge pages (no TLB misses)
    Pin to dedicated CPU cores (no context switches)
    Result: 10M+ packets/sec on commodity hardware

  Connection timeout:
    TCP established: 300 seconds (default)
    TCP time_wait: 120 seconds
    UDP: 30 seconds
    Aggressive timeouts free table entries faster but may drop slow clients

  Connection table failover:
    Active LB syncs connection table to standby via multicast
    On failover: standby has ~99% of connections → most clients unaffected
    Missing entries: those connections reset → client reconnects
    For DSR: no connection table needed on return path → cleaner failover
```

### Consistent Hashing with Bounded Loads (Google Maglev)

```
Problem with standard consistent hashing for LB:
  Server S3 has hash position right after a large gap on the ring
  → S3 gets 40% of traffic while S1 and S2 get 30% each
  Virtual nodes help but don't guarantee bounds

Google's Maglev hashing:
  Build a lookup table of size M (prime, e.g., 65537)
  Each backend gets a "preference list" of table positions:
    preference[i] = (offset + i × skip) mod M
    offset = hash1(backend_name) mod M
    skip = hash2(backend_name) mod (M-1) + 1
    
  Fill the table round-robin by preference:
    Entry 0: Backend A (A's first preference is position 0)
    Entry 1: Backend B (B's first preference is position 1)
    Entry 2: Backend C (C's first preference is position 2)
    ...continue until all M entries filled
    
  Lookup: table[hash(5-tuple) mod M] → backend
  
  Properties:
    Disruption: adding/removing 1 of N backends moves only ~1/N of entries
    Uniformity: each backend gets exactly M/N ± 1 entries (near-perfect balance)
    Speed: single array lookup → O(1) per packet (cache-friendly)
    
  Why it's better than ring-based consistent hashing:
    Ring: 100-200 virtual nodes needed for reasonable balance → memory
    Maglev: one 65K-entry table → 512 KB → fits in L2 cache
    Ring: O(log N) binary search for closest node
    Maglev: O(1) array lookup

Bounded loads addition:
  Hard limit: no server gets more than (1 + ε) × average load
  ε = 0.25 → max 25% over average
  
  If selected server exceeds bound:
    Walk to next server in the table
    Repeat until finding one below bound
    
  Provides: strict load balancing guarantee + consistent hashing benefits
  Used by: Google (Maglev), Envoy (ring_hash with overprovisioning_factor)
```

### The LB as Single Point of Failure — Defense in Depth

```
Layer 1: Active-Passive (VRRP/Keepalived)
  Two LB instances share a VIP
  Active handles all traffic
  Passive monitors Active via heartbeat (every 1 second)
  Active dies → Passive claims VIP via gratuitous ARP (< 3 seconds)
  ✓ Simple, proven
  ✗ Passive wastes resources (idle), 50% hardware utilization
  ✗ Failover gap: 1-3 seconds of dropped connections

Layer 2: Active-Active (DNS round-robin)
  DNS returns multiple LB IPs: [10.0.0.1, 10.0.0.2]
  Both LBs handle traffic simultaneously
  If one fails → DNS health check removes it
  ✓ 100% hardware utilization
  ✗ DNS TTL means clients cache stale IP → traffic to dead LB for TTL duration
  ✗ Uneven distribution (DNS round-robin is not load-aware)

Layer 3: Anycast (BGP)
  Both LBs advertise the SAME IP via BGP
  Network routes packets to the NEAREST LB (by BGP path)
  If one LB fails → BGP withdraws route → traffic converges to other LB
  ✓ Automatic, network-level failover (< 30 seconds)
  ✓ Natural geographic load distribution
  ✗ BGP convergence can take 10-30 seconds
  ✗ Requires BGP-capable network infrastructure
  Used by: Cloudflare, Google, AWS NLB

Layer 4: Distributed LB (no single LB)
  Service mesh: every service has a sidecar proxy (Envoy)
  Each sidecar does its own load balancing (client-side LB)
  No centralized LB → no SPOF
  ✓ Eliminates LB as bottleneck entirely
  ✗ Every service needs sidecar → operational complexity
  ✗ Sidecar resource overhead (CPU/memory per pod)
  Used by: Kubernetes with Istio/Linkerd
```

