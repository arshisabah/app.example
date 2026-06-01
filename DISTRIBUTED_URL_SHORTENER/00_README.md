# Production-Grade Distributed URL Shortener System

**A comprehensive, production-ready implementation similar to Bitly, designed for product-based companies and HFT firms.**

## 📚 Complete Documentation Index

This project contains comprehensive documentation across 15+ markdown files:

### 1. **System Design & Architecture**
- `01_SYSTEM_DESIGN_HLD.md` - High-Level Architecture
- `02_SYSTEM_DESIGN_LLD.md` - Low-Level Design
- `03_DATABASE_SCHEMA.md` - Database Design & Sharding
- `04_CAPACITY_PLANNING.md` - Load Calculation & Estimation
- `05_SECURITY_DESIGN.md` - Security Architecture

### 2. **API & Technical Specifications**
- `06_API_SPECIFICATION.md` - REST APIs with Examples
- `07_REDIS_STRATEGY.md` - Caching Architecture
- `08_KAFKA_EVENT_FLOW.md` - Event-Driven Design
- `09_MONITORING_OBSERVABILITY.md` - Monitoring Setup

### 3. **Implementation & Code**
- `10_SERVICE_IMPLEMENTATION.md` - Microservices Code
- `11_DOCKER_COMPOSE.md` - Docker Setup
- `12_KUBERNETES_DEPLOYMENT.md` - K8s Configuration
- `13_TESTING_STRATEGY.md` - Unit & Integration Tests

### 4. **Interview & Learning**
- `14_INTERVIEW_QUESTIONS.md` - System Design Q&A
- `15_PERFORMANCE_ANALYSIS.md` - Benchmarks & Optimization
- `16_DEPLOYMENT_GUIDE.md` - Production Deployment

---

## 🎯 Key Features

✅ **URL Shortening** - Generate short codes in ~100ms
✅ **Custom Aliases** - User-defined short codes
✅ **URL Expiration** - TTL-based URL management
✅ **Click Tracking** - Real-time analytics
✅ **Geo Analytics** - Location-based insights
✅ **Device Analytics** - Device type tracking
✅ **QR Code Generation** - Instant QR codes
✅ **User Accounts** - JWT-based authentication
✅ **Rate Limiting** - Token bucket algorithm
✅ **Admin Dashboard** - Management interface
✅ **API Keys** - Programmatic access
✅ **Auto-scaling** - Kubernetes horizontal scaling
✅ **Distributed Tracing** - OpenTelemetry integration
✅ **Monitoring** - Prometheus + Grafana

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (Nginx)                    │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
   ┌─────────────┐          ┌──────────────┐
   │ API Gateway │          │ CDN (cache)  │
   │ (Rate Limit)│          │ (edge nodes) │
   └──────┬──────┘          └──────────────┘
          │
    ┌─────┼─────────────────────┐
    ▼     ▼                     ▼
┌────────────┐  ┌──────────┐  ┌──────────┐
│Auth Service│  │  URL     │  │Analytics │
│  - JWT     │  │Shortener │  │ Service  │
│  - API Key │  │ Service  │  │- Clicks  │
└────────────┘  └──────────┘  │- Geo     │
                │              │- Device  │
                │              └──────────┘
        ┌───────┴────────┐
        ▼                ▼
    ┌────────┐    ┌──────────────┐
    │ Kafka  │    │ PostgreSQL   │
    │Cluster │    │ Primary      │
    └────────┘    │ + Read       │
                  │ Replicas     │
                  └──────────────┘
                        ▲
        ┌───────────────┴────────────────┐
        ▼                                ▼
   ┌──────────┐                    ┌──────────┐
   │  Redis   │                    │  Redis   │
   │ Cluster  │                    │ Cache    │
   │(Sessions)│                    │(Redirect)│
   └──────────┘                    └──────────┘
```

---

## 📊 Performance Targets

| Operation | P99 Latency | P95 Latency | Throughput |
|-----------|------------|------------|------------|
| Redirect (cache hit) | <5ms | <3ms | 100K+ QPS |
| Redirect (cache miss) | <50ms | <35ms | 50K+ QPS |
| Create URL | <200ms | <150ms | 10K+ QPS |
| Analytics Query | <500ms | <350ms | 5K+ QPS |
| Get User Stats | <300ms | <200ms | 10K+ QPS |

---

## 💾 Capacity Estimation

### Assumptions
- **DAU**: 100 Million
- **URLs Created/Day**: 10 Million
- **Reads/Day**: 1 Billion (100:1 read-write ratio)
- **Retention**: 2 years

### Storage Requirements
- **URLs Database**: ~50GB (100M URLs × 0.5KB)
- **Analytics**: ~500GB (2 years history)
- **Total**: ~550GB (with replicas: ~2.2TB)

### QPS Requirements
- **Peak Write**: 10M / 86400s = ~115 QPS
- **Peak Read**: 1B / 86400s = ~11,600 QPS
- **With spike factor (10x)**: ~116K read QPS

---

## 🔧 Tech Stack

| Component | Technology |
|-----------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.x |
| Security | Spring Security + JWT |
| Database | PostgreSQL / MySQL |
| Cache | Redis Cluster |
| Message Queue | Apache Kafka |
| API Gateway | Spring Cloud Gateway |
| Service Discovery | Eureka / Consul |
| Monitoring | Prometheus + Grafana |
| Tracing | OpenTelemetry + Jaeger |
| Container | Docker |
| Orchestration | Kubernetes |
| Load Balancing | Nginx / Spring Load Balancer |
| Resilience | Resilience4j |

---

## 📦 Project Structure

```
distributed-url-shortener/
├── services/
│   ├── config-server/
│   ├── service-registry/
│   ├── api-gateway/
│   ├── auth-service/
│   ├── url-service/
│   ├── analytics-service/
│   └── admin-dashboard/
├── common/
│   ├── models/
│   ├── exceptions/
│   ├── utils/
│   └── constants/
├── infrastructure/
│   ├── docker-compose.yml
│   └── kubernetes/
├── tests/
│   ├── unit-tests/
│   ├── integration-tests/
│   └── load-tests/
├── docs/
│   └── [All markdown documentation]
└── pom.xml
```

---

## 🚀 Quick Start

### Prerequisites
```bash
# Required
- Java 21
- Docker & Docker Compose
- Maven 3.8+
- Git

# Optional
- Kubernetes (kind/minikube/cloud)
- PostgreSQL client
- Redis CLI
```

### 1. Start Infrastructure
```bash
docker-compose -f infrastructure/docker-compose.yml up -d

# Wait for services to be healthy
docker-compose ps
```

### 2. Build Services
```bash
mvn clean package -DskipTests
```

### 3. Run Services
```bash
# Terminal 1
java -jar services/config-server/target/config-server.jar

# Terminal 2
java -jar services/service-registry/target/service-registry.jar

# Terminal 3
java -jar services/api-gateway/target/api-gateway.jar

# Terminal 4
java -jar services/auth-service/target/auth-service.jar

# Terminal 5
java -jar services/url-service/target/url-service.jar

# Terminal 6
java -jar services/analytics-service/target/analytics-service.jar
```

### 4. Test API
```bash
# Register user
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username": "test", "email": "test@example.com", "password": "Pass123!"}'

# Login
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username": "test", "password": "Pass123!"}'

# Create short URL (use token from login)
curl -X POST http://localhost:8080/api/v1/shorten \
  -H 'Authorization: Bearer {JWT_TOKEN}' \
  -H 'Content-Type: application/json' \
  -d '{"longUrl": "https://example.com/very/long/path", "customAlias": "mylink"}'

# Redirect
curl -L http://localhost:8080/r/mylink
```

### 5. Access Monitoring
- Grafana: http://localhost:3000 (admin/admin)
- Prometheus: http://localhost:9090
- Jaeger: http://localhost:16686

---

## 🧪 Testing

```bash
# Unit tests
mvn test

# Integration tests
mvn verify

# Load testing (requires k6)
k6 run tests/load-test.js --vus 100 --duration 60s
```

---

## 📈 Monitoring & Observability

### Metrics Tracked
- Request latency (p50, p95, p99)
- Error rates and types
- Cache hit/miss ratios
- Database connection pool
- Kafka consumer lag
- JVM metrics
- Custom business metrics

### Dashboards
- System Health Dashboard
- Service Latency Dashboard
- Error Analysis Dashboard
- Cache Performance Dashboard
- Database Performance Dashboard

---

## 🔐 Security Features

✅ JWT authentication with refresh tokens
✅ API key management
✅ Rate limiting (per user, per IP)
✅ Input validation & sanitization
✅ CORS configuration
✅ Encrypted sensitive data
✅ Audit logging
✅ SQL injection prevention
✅ HTTPS/TLS enforcement
✅ Token rotation & expiration

---

## 📚 Documentation Files

Each file covers a specific aspect in detail:

1. **HLD** - Understand the big picture
2. **LLD** - Deep dive into components
3. **Database Schema** - Table design & relationships
4. **Capacity Planning** - Load calculations
5. **API Spec** - All endpoints documented
6. **Redis Strategy** - Caching patterns
7. **Kafka Events** - Event schema & flow
8. **Security** - Authentication & authorization
9. **Monitoring** - Observability setup
10. **Interview Questions** - System design prep

---

## 🎓 Learning Outcomes

By studying this project, you'll learn:

✅ Microservices architecture patterns
✅ Distributed system design
✅ Database sharding & replication
✅ Caching strategies (Redis)
✅ Event-driven architecture (Kafka)
✅ API gateway implementation
✅ Service discovery & registration
✅ Distributed tracing
✅ Monitoring & alerting
✅ Load balancing strategies
✅ Circuit breaker patterns
✅ Rate limiting algorithms
✅ Kubernetes deployment
✅ Docker containerization
✅ System design interview skills

---

## 🎯 Interview Preparation

This project covers all major system design interview topics:

- **Scalability**: Horizontal scaling with K8s
- **Availability**: Replication & failover
- **Performance**: Caching & indexing
- **Consistency**: Database transactions
- **Reliability**: Circuit breakers & retries
- **Security**: Authentication & authorization
- **Cost Optimization**: Resource efficiency

---

## 📞 Support

For issues or questions:
1. Check documentation files
2. Review interview questions for context
3. Check test files for usage examples

---

## 📄 License

MIT License - Free for personal and commercial use

---

## 🙏 Acknowledgments

Inspired by:
- Bitly URL shortening service
- Twitter system design patterns
- Google infrastructure papers
- High-frequency trading optimization techniques

---

**Ready to dive in? Start with `01_SYSTEM_DESIGN_HLD.md`**
