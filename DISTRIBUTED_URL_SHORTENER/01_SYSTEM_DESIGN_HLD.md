# High-Level Design (HLD) - Distributed URL Shortener System

## 1. Overview

A distributed URL shortener service that converts long URLs into short, shareable links with analytics capabilities. Similar to Bitly, but designed for extreme scale with sub-100ms latency.

---

## 2. Functional Requirements

### Core Features
1. **URL Shortening**
   - Convert long URLs to short codes
   - Support custom aliases (7-30 characters)
   - Generate unique short codes algorithmically

2. **URL Redirection**
   - Redirect short URLs to original URLs
   - Support URL expiration
   - Track redirects in real-time

3. **Analytics**
   - Track clicks, geographic location, device type
   - Provide real-time and historical analytics
   - Support time-range queries

4. **User Management**
   - User registration and authentication
   - API key management
   - User preferences and settings

5. **URL Management**
   - Disable/enable URLs
   - Edit long URL
   - Set expiration dates
   - Bulk operations

---

## 3. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Availability** | 99.99% uptime (4 nines) |
| **Latency (Read)** | <50ms p99 latency |
| **Latency (Write)** | <200ms p99 latency |
| **Throughput** | 100K+ read QPS, 10K+ write QPS |
| **Consistency** | Eventual consistency |
| **Durability** | No data loss |
| **Scalability** | Horizontal scaling |

---

## 4. High-Level Architecture

### 4.1 Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                     Client Requests                              │
│         (Web Browser, Mobile App, API Clients)                   │
└────────────────────┬─────────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │   CDN / Global Load        │
        │   Balancer (Anycast)       │
        │   - Geographic routing     │
        │   - DDoS protection        │
        └────────────────┬───────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   Region 1          Region 2        Region 3
        │                │                │
        ▼                ▼                ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  Local  │      │  Local  │      │  Local  │
   │ Load    │      │ Load    │      │ Load    │
   │Balancer │      │Balancer │      │Balancer │
   └────┬────┘      └────┬────┘      └────┬────┘
        │                │                │
        ▼                ▼                ▼
    ┌─────────────────────────────────────────┐
    │         API Gateway Layer               │
    │  - Rate Limiting                        │
    │  - Authentication                       │
    │  - Request Routing                      │
    │  - Load Balancing                       │
    │  - Circuit Breaker                      │
    └──────────────┬──────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌─────────────────────────────────────────┐
│      Microservices Layer                │
│                                         │
│  ┌─────────────┐  ┌────────────────┐  │
│  │Auth Service │  │URL Service     │  │
│  │- Login      │  │- Shorten       │  │
│  │- Register   │  │- Redirect      │  │
│  │- Token Mgmt │  │- Validation    │  │
│  └─────────────┘  └────────────────┘  │
│                                         │
│  ┌─────────────────┐  ┌──────────────┐ │
│  │Analytics Service│  │Admin Service │ │
│  │- Click tracking │  │- Dashboard   │ │
│  │- Geo analytics  │  │- User mgmt   │ │
│  │- Device info    │  │- Reports     │ │
│  └─────────────────┘  └──────────────┘ │
└─────────────┬──────────────────────────┘
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
┌──────┐  ┌────────┐  ┌──────────┐
│Cache │  │Message │  │ Database │
│Layer │  │ Queue  │  │  Layer   │
│      │  │        │  │          │
│Redis │  │ Kafka  │  │PostgreSQL│
│      │  │        │  │+ Replicas│
└──────┘  └────────┘  └──────────┘
    ▲         │          ▲
    │         │          │
    └─────────┴──────────┘
       Data Sync
```

---

## 5. Component Details

### 5.1 API Gateway
**Responsibilities:**
- Request routing to appropriate service
- Rate limiting (token bucket algorithm)
- JWT token validation
- Load balancing across service instances
- Circuit breaker for downstream services
- Request/response logging
- CORS handling

**Technology:** Spring Cloud Gateway

### 5.2 Authentication Service
**Responsibilities:**
- User registration and login
- JWT token generation and refresh
- API key management
- OAuth2 integration (optional)
- Session management

**Technology:** Spring Security, JWT

### 5.3 URL Shortening Service
**Responsibilities:**
- Generate unique short codes
- Support custom aliases
- URL validation
- Expiration management
- CRUD operations on URLs
- Generate QR codes

**Technology:** Spring Boot, Spring Data JPA

### 5.4 Analytics Service
**Responsibilities:**
- Track click events (asynchronous)
- Aggregate analytics data
- Provide analytics APIs
- Geographic and device tracking
- Real-time and historical queries

**Technology:** Spring Boot, Kafka Consumer

### 5.5 Cache Layer
**Responsibilities:**
- Cache short URL → long URL mapping
- Cache analytics data
- Session caching
- Rate limit counters

**Technology:** Redis Cluster
**Strategy:** Write-through caching

### 5.6 Message Queue
**Responsibilities:**
- Asynchronous click event processing
- Analytics data pipeline
- Service communication
- Event sourcing

**Technology:** Apache Kafka

### 5.7 Database Layer
**Responsibilities:**
- Store URLs and metadata
- Store user information
- Store analytics data
- Persistence layer

**Technology:** PostgreSQL with Read Replicas

---

## 6. Data Flow

### 6.1 URL Shortening Flow

```
1. Client submits long URL
         │
         ▼
2. API Gateway validates request
   - Rate limit check
   - Token validation
         │
         ▼
3. URL Service receives request
   - Validate long URL
   - Check if already shortened
         │
         ▼
4. Generate short code (Algorithm)
   - Base62 encoding
   - Collision resolution
         │
         ▼
5. Store in Database
   - Primary write
   - Async replication
         │
         ▼
6. Cache short code → long URL mapping
   - Redis cache (Write-through)
   - TTL: 24 hours
         │
         ▼
7. Return short URL to client
   - Response time: <200ms p99
```

### 6.2 URL Redirection Flow

```
1. Client requests short URL
   http://short.url/abc123
         │
         ▼
2. API Gateway receives (minimal processing)
   - Route to redirect endpoint
         │
         ▼
3. Check Cache (Redis)
   - Cache hit (99% of requests)
         │
         ├─ YES ──▶ Return redirect (HTTP 301)
         │         Response time: <5ms
         │
         ├─ NO ──▶ Query Database
         │         │
         │         ▼
         │      Cache miss (1% of requests)
         │      Response time: <50ms
         │      │
         │      ▼
         │      Update cache with TTL
         │      │
         └──────▶ Return redirect
         │
         ▼
4. Emit click event to Kafka
   (Asynchronous, non-blocking)
   - Event schema: {shortCode, timestamp, geo, device}
         │
         ▼
5. Analytics Service consumes events
   - Aggregate in-memory
   - Flush to database periodically
```

### 6.3 Analytics Flow

```
1. Redirect generates click event
   - Sent to Kafka topic: "clicks"
   - Non-blocking (fire & forget)
         │
         ▼
2. Analytics Service Consumer
   - Consume events from Kafka
   - Parse event data
         │
         ▼
3. Aggregate in memory (sliding window)
   - Group by: hour, day, geography, device
   - Keep in-memory cache (Redis)
         │
         ▼
4. Periodic flush to database
   - Every 1 minute or 10K events
   - Batch write to PostgreSQL
   - Update Redis aggregates
         │
         ▼
5. Query Analytics
   - API endpoint: GET /api/v1/analytics/{shortCode}
   - Response from cache (99% hit rate)
   - Response time: <100ms
```

---

## 7. Database Architecture

### 7.1 Sharding Strategy

**Sharding Key:** shortCode (hash-based)
**Number of Shards:** 256 (2^8)
**Shard ID = hash(shortCode) % 256**

```
Short Code: "abc123"
     │
     ▼
Hash: 0xA3F2C1
     │
     ▼
0xA3F2C1 % 256 = 193
     │
     ▼
Route to Shard 193 (Database 193)
```

### 7.2 Read Replica Strategy

```
┌─────────────────────────────────────┐
│      Primary (Write Master)         │
│   - All writes go here              │
│   - Synchronous replication         │
└──────────────┬──────────────────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
    ▼          ▼          ▼
┌──────┐  ┌──────┐  ┌──────┐
│Read  │  │Read  │  │Read  │
│Rep1  │  │Rep2  │  │Rep3  │
└──────┘  └──────┘  └──────┘

Analytics queries distributed:
- Geo queries → Rep1
- Device queries → Rep2
- Time range → Rep3
```

---

## 8. Caching Strategy

### 8.1 Cache Layers

**Layer 1: HTTP Cache (CDN)**
- Cache-Control headers
- TTL: 24 hours
- Geographic distribution

**Layer 2: Application Cache (Redis)**
- Short URL → Long URL mapping
- TTL: 24 hours
- Cache hit rate: 99%+

**Layer 3: Database Cache (Query Results)**
- Analytics data
- User preferences
- TTL: 1 hour

### 8.2 Cache Invalidation

```
Event: URL deleted
  │
  ▼
Invalidate Cache:
- Delete from Redis
- Delete from CDN (purge)
- Mark as inactive in DB
```

---

## 9. Event-Driven Architecture

### 9.1 Event Topics

| Topic | Schema | Consumers |
|-------|--------|----------|
| clicks | {shortCode, timestamp, geo, device, userAgent} | Analytics Service, Archive Service |
| url.created | {shortCode, userId, longUrl, createdAt} | Notification Service, Audit Service |
| url.updated | {shortCode, updates, timestamp} | Audit Service, Cache Invalidation |
| url.deleted | {shortCode, deletedAt, reason} | Cache Invalidation, Audit Service |

### 9.2 Kafka Configuration

- **Partitions per topic:** 32 (scalability)
- **Replication factor:** 3 (durability)
- **Retention:** 7 days (operational window)
- **Consumer group:** analytics-service-group

---

## 10. Load Balancing

### 10.1 Global Load Balancing

```
Client
  │
  ▼
Geographic Routing (Anycast DNS)
  │
  ├─→ US Users → AWS us-east-1
  ├─→ EU Users → AWS eu-west-1
  └─→ APAC Users → AWS ap-southeast-1
```

### 10.2 Regional Load Balancing

```
Region: us-east-1
  │
  ▼
Nginx Load Balancer
  │
  ├─→ API Gateway 1 (weight: 33%)
  ├─→ API Gateway 2 (weight: 33%)
  └─→ API Gateway 3 (weight: 34%)
```

### 10.3 Service Load Balancing

```
API Gateway
  │
  ├─→ URL Service Instance 1
  ├─→ URL Service Instance 2
  └─→ URL Service Instance 3
  
Algorithm: Round-robin with health checks
```

---

## 11. Security Architecture

### 11.1 Authentication

```
Login
  │
  ▼
Validate credentials
  │
  ▼
Generate JWT token
  - Header: {alg: HS256, typ: JWT}
  - Payload: {userId, exp, iat, scopes}
  - Signature: HMAC-SHA256
  │
  ▼
Return token (expires in 1 hour)
```

### 11.2 Authorization

```
API Request with JWT
  │
  ▼
Validate JWT signature
  │
  ▼
Check expiration
  │
  ▼
Extract userId and scopes
  │
  ▼
Check permissions against resource
  │
  ▼
Grant or deny access
```

### 11.3 Rate Limiting

```
Request arrives
  │
  ▼
Extract rate limit key
- User ID (authenticated)
- IP address (anonymous)
  │
  ▼
Check Redis counter
- Key: rate_limit:{key}
- Value: request_count
  │
  ├─ Count < limit ──▶ Increment counter, process request
  └─ Count ≥ limit ──▶ Return 429 Too Many Requests
```

**Algorithm:** Token Bucket
- Tokens per minute: 1000
- Refill rate: 1 token every 60ms

---

## 12. Monitoring & Observability

### 12.1 Metrics

```
Application Metrics:
- Request latency (p50, p95, p99)
- Request throughput (QPS)
- Error rates (5xx, 4xx)
- Cache hit ratio
- Active connections

Business Metrics:
- URLs created per minute
- Total redirects per minute
- Unique users
- Geographic distribution
- Device distribution

Infrastructure Metrics:
- CPU utilization
- Memory usage
- Disk I/O
- Network bandwidth
- Database connections
```

### 12.2 Logging

```
ELK Stack (Elasticsearch, Logstash, Kibana)

Log Levels:
- ERROR: All exceptions and errors
- WARN: Rate limit exceeded, cache miss
- INFO: Request/response, service health
- DEBUG: Detailed flow (disabled in production)
```

### 12.3 Tracing

```
OpenTelemetry + Jaeger

Trace Flow:
Client Request
  ├─ API Gateway (Span 1)
  ├─ Auth Service (Span 2)
  ├─ URL Service (Span 3)
  │  ├─ Database Query (Span 3.1)
  │  └─ Cache Update (Span 3.2)
  └─ Response (Span 4)

Trace visualization in Jaeger UI
```

---

## 13. Failure Scenarios & Recovery

### 13.1 Database Failure

```
Scenario: Primary database down

Action:
1. Health check detects failure
2. Failover to read replica
3. Promote read replica to primary
4. Route all writes to new primary
5. Notify team for investigation
Recovery Time: <1 minute
```

### 13.2 Cache Layer Failure

```
Scenario: Redis cluster node down

Action:
1. Sentinel detects node failure
2. Redistribute data to other nodes
3. Read requests served from other nodes
4. Database used as fallback
Impact: Performance degradation, no data loss
```

### 13.3 Service Instance Failure

```
Scenario: API Gateway instance crashes

Action:
1. Health check fails
2. Load balancer removes from pool
3. Requests routed to healthy instances
4. Auto-scaler detects and replaces instance
Recovery Time: <30 seconds
```

---

## 14. Scalability Analysis

### 14.1 Horizontal Scaling

**Service Instances:**
- Current: 3 instances per service
- Auto-scale: 2-20 instances
- Trigger: CPU > 70% or memory > 80%

**Database Shards:**
- Current: 256 shards
- Can handle: 100B+ URLs
- Shard splitting: When a shard reaches capacity

**Cache Cluster:**
- Current: 6 nodes (3 masters, 3 replicas)
- Capacity: 500GB data
- Auto-scale based on eviction rate

### 14.2 Performance Under Load

```
Load: 100K QPS
Latency (p99): <50ms
Error Rate: <0.01%
Cache Hit Ratio: 99.5%
Database CPU: 45%
Database Memory: 65%

Load: 200K QPS (2x)
Action: Auto-scale services
Latency (p99): <80ms (degraded)
Error Rate: <0.05%
Cache Hit Ratio: 98.5%
Database CPU: 85% (bottleneck)

Solution: Add read replicas, scale cache
```

---

## 15. Cost Optimization

### 15.1 Resource Allocation

| Component | Instances | CPU | Memory | Cost/Month |
|-----------|-----------|-----|--------|------------|
| API Gateway | 3 | 2 cores | 4GB | $300 |
| URL Service | 3 | 4 cores | 8GB | $600 |
| Analytics Service | 2 | 2 cores | 4GB | $200 |
| Database (Primary) | 1 | 16 cores | 64GB | $2000 |
| Database (Read Replica) | 3 | 8 cores | 32GB | $3000 |
| Redis Cluster | 6 | 2 cores | 16GB | $900 |
| Kafka Cluster | 3 | 4 cores | 8GB | $600 |
| **Total** | | | | **$7,600** |

### 15.2 Cost Reduction Strategies

1. **Spot Instances** (Non-critical services): 70% cost reduction
2. **Reserved Instances** (3-year): 40% cost reduction
3. **Auto-scaling** (Scale down during off-peak): 30% cost reduction
4. **Compression** (Data and logs): 50% storage cost reduction

---

## 16. Deployment Strategy

### 16.1 Environment Setup

```
Development
  ├─ Single node cluster
  ├─ 1 PostgreSQL instance
  └─ 1 Redis instance

Staging
  ├─ 3-node cluster
  ├─ Replicated PostgreSQL
  └─ 3-node Redis cluster

Production
  ├─ Multi-region deployment
  ├─ 256 database shards
  └─ 6-node Redis cluster
```

### 16.2 Deployment Process

```
1. Code commit to main branch
   │
   ▼
2. CI/CD pipeline triggered
   - Unit tests
   - Integration tests
   - Security scans
   │
   ▼
3. Docker image built and pushed
   │
   ▼
4. Deploy to staging
   - Smoke tests
   - Performance tests
   │
   ▼
5. Manual approval for production
   │
   ▼
6. Blue-green deployment
   - Deploy new version (green)
   - Run health checks
   - Route traffic gradually
   - Monitor metrics
   │
   ▼
7. Rollback plan (if issues)
   - Revert to blue (old version)
   - Immediate rollback
```

---

## 17. Interview Perspective

### Key Talking Points

1. **Scalability:**
   - "How does the system handle 10x traffic?"
   - Auto-scaling via Kubernetes, database sharding, cache layers

2. **Latency:**
   - "How do you achieve <50ms p99?"
   - Redis caching, database indexing, CDN, read replicas

3. **Consistency:**
   - "What consistency model are you using?"
   - Eventual consistency for analytics, strong consistency for URLs

4. **Trade-offs:**
   - "Why not use a single database?"
   - Sharding necessary for scale, eventual consistency acceptable for analytics

5. **Failure Handling:**
   - "What if Redis goes down?"
   - Database fallback, Sentinel auto-failover, minimal latency impact

---

## 18. HFT Perspective (Low-Latency Optimization)

### Latency Optimization Techniques

1. **Cache Everything**
   - In-memory caches in application
   - Redis cluster near application
   - CDN for static content

2. **Reduce Network Hops**
   - Co-locate services in same datacenter
   - Use internal networks (10Gbps)
   - Minimize external API calls

3. **Database Optimization**
   - Index frequently queried columns
   - Use denormalization where appropriate
   - Connection pooling (HikariCP)

4. **Application Optimization**
   - Non-blocking I/O (Project Reactor)
   - Batch operations
   - Lock-free data structures where possible

5. **JVM Tuning**
   - GC optimization (G1GC)
   - Heap sizing
   - CPU affinity

### Target Metrics

| Component | Current | Target (HFT) |
|-----------|---------|-------------|
| Redirect (cache hit) | 2-5ms | <1ms |
| Redirect (cache miss) | 40-50ms | <20ms |
| Create URL | 180-200ms | <100ms |

---

## 19. Next Steps

1. Review LLD for detailed component design
2. Check database schema in `03_DATABASE_SCHEMA.md`
3. Study API specifications in `06_API_SPECIFICATION.md`
4. Review implementation in `10_SERVICE_IMPLEMENTATION.md`

---

**Continue to: `02_SYSTEM_DESIGN_LLD.md` for Low-Level Design details**
