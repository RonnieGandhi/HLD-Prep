# 39. Design a Real-time Dashboard and Metrics System

---

## 1. Functional Requirements (FR)

- **Create dashboards**: Users create dashboards with multiple panels (charts, tables, gauges)
- **Widget types**: Line charts (time-series), bar charts, pie charts, heatmaps, tables, single-stat, alerts
- **Data sources**: Query from Prometheus, ClickHouse, Elasticsearch, PostgreSQL, custom APIs
- **Auto-refresh**: Dashboards auto-refresh at configurable intervals (5s, 15s, 1m)
- **Templating**: Dashboard variables (e.g., dropdown for service name, region)
- **Alerting**: Define alert rules on metrics; trigger notifications when thresholds breached
- **Sharing**: Share dashboards via link; embed in other tools
- **Annotations**: Mark events (deployments, incidents) on time-series charts

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Dashboard renders with all panels in < 3 seconds
- **High Availability**: 99.99% — dashboards used for incident response
- **Concurrent Users**: Support 10K users viewing dashboards simultaneously
- **Scalability**: Thousands of dashboards, each querying billions of data points
- **Responsive**: Work on desktop and mobile

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active dashboards | 50K |
| Panels per dashboard (avg) | 10 |
| Dashboard views / day | 2M |
| Queries / sec (peak) | 50K |
| Avg query response time | 500 ms |
| Dashboard definitions storage | < 1 GB (JSON configs) |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Dashboard Frontend (SPA)                        │
│                                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ Panel 1     │  │ Panel 2     │  │ Panel 3     │  │ Panel N    │ │
│  │ Time-series │  │ Stat/Gauge  │  │ Table       │  │ Heatmap    │ │
│  │ Chart       │  │             │  │             │  │            │ │
│  │             │  │             │  │             │  │            │ │
│  │ Auto-refresh│  │ Auto-refresh│  │ Auto-refresh│  │ Auto-ref.  │ │
│  │ every 15s   │  │ every 15s   │  │ every 60s   │  │ every 15s  │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬─────┘ │
│         │ independent     │              │              │         │
│         │ queries         │              │              │         │
└─────────┼────────────────┼──────────────┼──────────────┼─────────┘
          │                │              │              │
          ▼                ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   Dashboard API Server (Stateless)                   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                      Query Proxy                                │ │
│  │                                                                  │ │
│  │  Request: {datasource: "prometheus",                            │ │
│  │           query: "rate(http_requests_total{svc='api'}[5m])",    │ │
│  │           from: "now-6h", to: "now", step: "1m"}               │ │
│  │                                                                  │ │
│  │  Pipeline:                                                      │ │
│  │  1. Auth check (JWT → user → org → permissions)                │ │
│  │  2. Template variable substitution ($service → "api-gateway")  │ │
│  │  3. Query hash = SHA256(datasource + query + time_range)       │ │
│  │  4. Cache check (Redis) → if HIT → return cached result       │ │
│  │  5. Singleflight: if same query in-flight → wait for result   │ │
│  │  6. Execute against data source                                │ │
│  │  7. Transform: math expressions, unit conversion, renaming     │ │
│  │  8. Cache result (TTL = panel refresh interval)                │ │
│  │  9. Return JSON response                                       │ │
│  └──────────────────────────┬──────────────────────────────────────┘ │
│                              │                                       │
│  ┌────────────┐  ┌───────────┼───────────┐  ┌─────────────────────┐ │
│  │ Dashboard  │  │  Data Source Routing   │  │  Alert Evaluation   │ │
│  │ CRUD       │  │                        │  │  Engine             │ │
│  │            │  │  ┌──────────────────┐  │  │                     │ │
│  │ Create,    │  │  │ Prometheus       │  │  │  Every eval_interval│ │
│  │ Read,      │  │  │ (PromQL)         │  │  │  (10s-60s):        │ │
│  │ Update,    │  │  ├──────────────────┤  │  │  • Execute query    │ │
│  │ Delete,    │  │  │ ClickHouse       │  │  │  • Compare vs       │ │
│  │ Version    │  │  │ (SQL)            │  │  │    threshold        │ │
│  │            │  │  ├──────────────────┤  │  │  • If breached N    │ │
│  │            │  │  │ Elasticsearch    │  │  │    consecutive      │ │
│  │            │  │  │ (Lucene query)   │  │  │    evals → FIRE     │ │
│  │            │  │  ├──────────────────┤  │  │  • Notify via       │ │
│  │            │  │  │ PostgreSQL       │  │  │    PagerDuty/Slack  │ │
│  │            │  │  │ (SQL)            │  │  │                     │ │
│  │            │  │  └──────────────────┘  │  │                     │ │
│  └─────┬──────┘  └───────────────────────┘  └─────────────────────┘ │
└────────┼─────────────────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────┐
│          PostgreSQL (Dashboard Store)            │
│                                                  │
│  • dashboards (config JSONB, version, org_id)   │
│  • dashboard_versions (audit trail)             │
│  • users, orgs, permissions                     │
│  • alert_rules, alert_state                     │
│  • annotations (deployment markers on charts)    │
└────────┬────────────────────────────────────────┘
         │
┌────────▼──────┐
│  Redis Cache  │  Query result cache (TTL = refresh interval)
│               │  Dashboard config cache (TTL = 5 min)
│               │  Active alert state cache
└───────────────┘
```

### Component Deep Dive

#### Query Proxy — The Performance Layer

```
Client requests: GET /api/v1/panels/{panel_id}/data?from=now-6h&to=now

Query Proxy:
  1. Parse dashboard template variables (replace $service → "user-service")
  2. Route to correct data source (Prometheus, ClickHouse, etc.)
  3. Check query cache (Redis): same query within last 30s → return cached result
  4. If cache miss → execute query against data source
  5. Apply transformations (math, joins, renaming)
  6. Cache result with TTL = refresh_interval
  7. Return to frontend

Query caching is critical: 100 users viewing the same dashboard = 100 identical queries
With caching: 1 query to backend, 99 served from cache
```

#### Dashboard Definition Model

```json
{
  "dashboard_id": "dash-uuid",
  "title": "Production Overview",
  "variables": [
    {"name": "service", "type": "query", "query": "label_values(service_name)"}
  ],
  "panels": [
    {
      "panel_id": 1,
      "title": "Request Rate",
      "type": "timeseries",
      "datasource": "prometheus",
      "query": "rate(http_requests_total{service=\"$service\"}[5m])",
      "interval": "1m",
      "position": {"x": 0, "y": 0, "w": 12, "h": 8}
    },
    {
      "panel_id": 2,
      "title": "Error Rate",
      "type": "stat",
      "datasource": "prometheus",
      "query": "rate(http_errors_total{service=\"$service\"}[5m]) / rate(http_requests_total{service=\"$service\"}[5m])",
      "thresholds": [{"value": 0.01, "color": "green"}, {"value": 0.05, "color": "red"}]
    }
  ]
}
```

---

## 5. APIs

### Dashboard CRUD
```http
POST /api/v1/dashboards      ← Create dashboard
GET /api/v1/dashboards/{id}   ← Get dashboard config
PUT /api/v1/dashboards/{id}   ← Update dashboard
DELETE /api/v1/dashboards/{id}
```

### Query Data for Panel
```http
POST /api/v1/query
{
  "datasource": "prometheus",
  "query": "rate(http_requests_total{service='user-service'}[5m])",
  "from": "2026-03-14T04:00:00Z",
  "to": "2026-03-14T10:00:00Z",
  "interval": "1m"
}
→ { "data": [{"timestamp": 1710320000, "value": 342.5}, ...] }
```

---

## 6. Data Model

### PostgreSQL — Dashboard Definitions
```sql
CREATE TABLE dashboards (
    dashboard_id    UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    title           VARCHAR(256),
    config          JSONB NOT NULL,      -- full dashboard JSON
    version         INT DEFAULT 1,
    created_by      UUID,
    updated_at      TIMESTAMP,
    INDEX idx_org (org_id)
);

CREATE TABLE dashboard_versions (
    dashboard_id    UUID,
    version         INT,
    config          JSONB,
    updated_by      UUID,
    updated_at      TIMESTAMP,
    PRIMARY KEY (dashboard_id, version)
);
```

### Redis — Query Cache
```
Key:    query_cache:{hash(datasource + query + time_range)}
Value:  compressed JSON result
TTL:    30 seconds (or dashboard refresh interval)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Data source down** | Dashboard shows stale cached data with "Data source unavailable" warning |
| **Query timeout** | 30-second timeout; show partial results or error per panel |
| **Dashboard store down** | Read from PostgreSQL replica; dashboard configs are small and cacheable |
| **Concurrent dashboard edits** | Optimistic locking (version field); last save wins with conflict warning |
| **Query abuse** | Per-user query rate limiting; max query complexity limits |

### Race Conditions and Scale Challenges

#### 1. Dashboard Stampede — Incident Causes Everyone to Open Dashboards

```
Scenario: P0 incident → 500 engineers open the same "Production Health" dashboard
  Dashboard has 10 panels → 5000 queries fired simultaneously
  Each query hits Prometheus/ClickHouse → backend overwhelmed

  Without caching: 5000 identical queries → 5000 backend requests → CRUSHES the TSDB
  
  Solutions:
    1. Query result caching ⭐ (most important):
       Hash(datasource + query + time_range) → cached in Redis (TTL = refresh interval)
       First request: cache miss → query backend → cache result
       Remaining 4999: cache hit → return cached result
       Result: 1 backend query instead of 5000 → 5000× reduction
    
    2. Request coalescing (singleflight):
       If query is in-flight → don't send another → wait for the first result
       Prevents N identical queries from reaching the backend
    
    3. Auto-refresh stagger:
       Instead of all dashboards refreshing at exactly T=0, T=15, T=30...
       Add random jitter: T=0+rand(0,5s), T=15+rand(0,5s)
       Spreads load across the 15-second window
    
    4. Pre-computed dashboards for P0 incidents:
       "War room" dashboards pre-warmed and pre-computed continuously
       Served from cache even during backend degradation
```

#### 2. Dashboard Version Conflicts — Two Users Editing Simultaneously

```
Alice opens "Prod Dashboard" (version 5)
Bob opens same dashboard (version 5)
Alice saves changes (version 6)
Bob saves changes (still based on version 5) → OVERWRITES Alice's changes!

Solution: Optimistic locking
  Save request includes: {"dashboard_id": "...", "expected_version": 5, "config": {...}}
  
  Server:
    SELECT version FROM dashboards WHERE id = ? FOR UPDATE;
    If version ≠ expected_version → return 409 Conflict
    Else → UPDATE, increment version
  
  Bob's save: expected_version=5, current_version=6 → CONFLICT
  → UI shows: "Dashboard was modified by Alice. Reload and retry."
  → Both users' changes preserved (Bob can merge manually)
```

#### 3. Data Source Timeout — Panel Shows Error While Others Load

```
Dashboard with 10 panels, each from different data sources:
  Panel 1-8: Prometheus (responds in 200ms) ✓
  Panel 9: ClickHouse (slow query, 30-second timeout) ✗
  Panel 10: External API (down) ✗

Bad UX: Entire dashboard waits for the slowest panel → 30s to load anything

Solution: Independent panel loading ⭐
  Each panel queries independently (parallel)
  Panels 1-8 render in 200ms → user sees data immediately
  Panel 9: Shows "Loading..." for 30s → shows result or timeout error
  Panel 10: Shows "Data source unavailable" with last cached value (if any)
  
  Each panel is an independent React component with its own loading state.
```

---

## 8. Deep Dive: Engineering Trade-offs

### Server-Side vs Client-Side Rendering

```
Client-side rendering (Grafana approach) ⭐:
  Server sends raw data → browser renders charts (D3.js, Chart.js, uPlot)
  ✓ Rich interactivity (zoom, hover tooltips, pan)
  ✓ Reduced server load
  ✗ Large datasets slow down the browser
  
Server-side rendering (image-based):
  Server renders chart as PNG/SVG → sends image to client
  ✓ Fast for large datasets (rendering on powerful server)
  ✓ Easy embedding (just an image URL)
  ✗ No interactivity
  ✗ Higher server load
  
Best: Client-side with server-side fallback for PDF exports / email reports
```

### Alert Evaluation — Avoiding False Positives

```
Rule: "Alert if error rate > 5% for 5 minutes"

Implementation:
  Every eval_interval (e.g., 10 seconds):
    Execute: rate(http_errors_total{service="api"}[5m]) / rate(http_requests_total{service="api"}[5m])
    If result > 0.05 → mark this evaluation as "firing"
    
  Alert state machine:
    OK → (threshold breached) → PENDING → (breached for "for" duration) → FIRING
    FIRING → (threshold normal) → OK
  
  "for: 5m" means: threshold must be breached for 5 consecutive minutes
    This prevents: single spike of errors → transient false positive
    
  Hysteresis (avoiding flapping):
    If error rate oscillates around 5% (4.9%, 5.1%, 4.8%, 5.2%):
      Without hysteresis: FIRE → RESOLVE → FIRE → RESOLVE (alert fatigue)
      With hysteresis: fire at 5%, resolve at 3%
        → Once fired, must drop to 3% to resolve
        → No more flapping

Challenge: Eval frequency × number of rules = query load
  1000 alert rules × 10s eval interval = 100 queries/sec to Prometheus
  Solution: Batch evaluation (Grafana evaluates all rules in parallel)
  Or: Recording rules → pre-compute expensive queries, alert on pre-computed metric
```

### Dashboard Annotations — Correlating Events with Metrics

```
A chart shows a CPU spike at 10:15 AM. What caused it?
  → Annotation at 10:14: "Deployed v2.3.1 to production"
  → Immediate correlation: deploy caused the spike!

Implementation:
  POST /api/v1/annotations
  { "dashboard_id": "...", "time": 1710320040, "text": "Deploy v2.3.1", "tags": ["deploy"] }
  
  Sources:
    • CI/CD pipeline posts annotation on every deploy
    • PagerDuty webhook posts annotation on every incident
    • Feature flag service posts annotation on flag toggle
  
  Display: Vertical line on time-series charts with hover tooltip
  
  Storage: PostgreSQL (annotations table with dashboard_id + timestamp index)
  Query: SELECT * FROM annotations WHERE dashboard_id=? AND time BETWEEN ? AND ?
```

### Dashboard Sharing and Embedding

```
Sharing options:
  1. Share link: Anyone with the link can view (read-only)
     Implementation: Generate unique token, store in dashboard_shares table
     URL: /d/{dashboard_id}?token={share_token}
     
  2. Embed iframe: Embed in other tools (Confluence, internal wiki)
     <iframe src="/d/{dashboard_id}/embed?token={share_token}&panels=1,2,3">
     
  3. Snapshot: Freeze dashboard at current time → share as static snapshot
     All data saved at snapshot time → no live queries needed
     Useful for incident post-mortems: "here's what the dashboard looked like during the outage"
     
  4. PDF/image export: Server-side render (headless Chrome) → generate PDF
     Used for weekly reports, email digests

  5. RBAC: Role-based access control
     Viewer: see dashboard, can't edit
     Editor: edit panels, create new dashboards
     Admin: manage data sources, users, permissions
```

