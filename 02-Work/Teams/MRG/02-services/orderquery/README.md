# Order Query Service - Documentation Index

## 📋 Quick Navigation

### Core Documentation
1. [[01-overview|📖 Service Overview]] - Architecture, responsibilities, dan design patterns
2. [[02-api-reference|🔌 API Reference]] - Complete API documentation (30+ endpoints)
3. [[03-group-order-flows|🚐 Group Order Flows]] - Detailed flows untuk Shuttle/Cititrans
4. [[04-dependencies|🔗 External Dependencies]] - Services, databases, dan libraries

### Original Documentation
- [[_doc/README.md|📚 Full Documentation]] - Comprehensive documentation di _doc folder

---

## 🎯 Service at a Glance

**Service Name**: `orderquery` (mybb-order-query)  
**Team**: MRG (Meta Reservation Gateway)  
**Domain**: Order Management & Query  
**Status**: ⭐⭐⭐⭐ Production-Ready & Stable

### Primary Functions
- **Read-Only Order Queries**: Centralized access ke order data dari multiple sources
- **Trip History Management**: Manage trip history dengan smart status updates
- **Group Order Management**: Specialized handling untuk shuttle (Cititrans) orders
- **Address Intelligence**: Frequently used addresses & pickup instructions
- **Billing Information**: Comprehensive billing details & payment tracking
- **Real-time Status**: Active orders, outstanding payments, reschedule eligibility

### Technology Stack
- **Language**: Go 1.24
- **Architecture**: Clean Architecture + Layered Architecture
- **Databases**: PostgreSQL (Main + Legacy)
- **Protocol**: gRPC + REST Gateway
- **Observability**: Elastic APM + Prometheus

---

## 📚 Documentation by Category

### Architecture & Design
- [[01-overview#Architecture Overview|Clean Architecture Implementation]]
- [[01-overview#Key Design Patterns|Design Patterns (Repository, Interceptor, Singleton, Factory)]]
- [[01-overview#Integration Architecture|Integration with External Services]]
- [[01-overview#Data Model|Dual Database Strategy]]

### API & Integration
- [[02-api-reference#Order Queries|Order Queries (10+ endpoints)]]
- [[02-api-reference#Trip Management|Trip Management]]
- [[02-api-reference#Group Orders (Shuttle)|Group Orders (Shuttle/Cititrans)]]
- [[02-api-reference#Address & Location|Address & Location Features]]
- [[02-api-reference#Billing & Payment|Billing & Payment]]
- [[02-api-reference#Reschedule|Reschedule Operations]]

### Special Features
- [[03-group-order-flows#Status Lifecycle|Group Order Status Lifecycle]]
- [[03-group-order-flows#GetListOrder|GetListOrder Flow]]
- [[03-group-order-flows#UpsertGroupOrderDetail|Upsert Group Orders]]
- [[03-group-order-flows#RescheduleGroupOrderDetail|Reschedule Group Orders]]
- [[03-group-order-flows#Automatic Status Updates (Cronjob)|Automatic Status Updates]]

### Operations & DevOps
- [[04-dependencies#Health Checks|Health Check Configuration]]
- [[04-dependencies#Configuration|Environment Variables]]
- [[04-dependencies#Troubleshooting|Troubleshooting Guide]]
- [[04-dependencies#Circuit Breaker Configuration|Circuit Breaker & Retry]]

---

## 🔥 Quick Start

### Local Development
```bash
# Clone repository
git clone git@git.bluebird.id:mybb-ms/orderquery.git
cd orderquery

# Setup environment
cp .env.example .env
# Edit .env with your local config

# Install dependencies
go mod download

# Run tests
go test ./...

# Run service
go run main.go
```

### Docker
```bash
# Build image
docker build -t orderquery:latest .

# Run container
docker run -p 6005:6005 -p 8005:8005 \
  --env-file .env \
  orderquery:latest
```

### Kubernetes
```bash
# Deploy to cluster
kubectl apply -f k8s/huawei-application.yaml

# Check status
kubectl get pods -l app=orderquery

# View logs
kubectl logs -f deployment/orderquery
```

### Cron Job Mode
```bash
# Run as cron job for status updates
./orderquery -execUsecase=cronjoborderhistoryupdate

# Or schedule in Kubernetes
kubectl apply -f k8s/cronjob-status-update.yaml
```

---

## 🎓 Learning Path

### For New Team Members

**Week 1: Understanding the Basics**
1. Read [[01-overview|Service Overview]]
2. Understand [[02-api-reference#Order Queries|Core API Endpoints]]
3. Study [[04-dependencies|External Dependencies]]
4. Run service locally
5. Test basic queries via REST Gateway

**Week 2: Architecture Deep Dive**
1. Study [[01-overview#Clean Architecture Implementation|Clean Architecture]]
2. Understand [[01-overview#Key Design Patterns|Design Patterns]]
3. Review code structure (Transport → UseCase → Repository)
4. Understand [[01-overview#Dual Database Strategy|Dual DB Strategy]]
5. Write first unit test

**Week 3: Special Features**
1. Deep dive into [[03-group-order-flows|Group Order Flows]]
2. Understand shuttle-specific business logic
3. Study [[03-group-order-flows#Status Lifecycle|Status Lifecycle]]
4. Review [[03-group-order-flows#Automatic Status Updates (Cronjob)|Cronjob Implementation]]
5. Contribute to existing features

### For API Consumers

**Quick Integration Guide**:
1. Choose query type ([[02-api-reference#API Categories|API Categories]])
2. Review request/response format ([[02-api-reference|API Reference]])
3. Implement error handling ([[02-api-reference#Error Responses|Error Codes]])
4. Understand [[02-api-reference#Pagination|Cursor-Based Pagination]]
5. Test with staging environment
6. Monitor via [[04-dependencies#Health Checks|Health Checks]]

---

## 🏆 Key Features

### 1. Read-Only Query Pattern
```
Mobile App → OrderQuery (Read) → Multiple Data Sources
                                  ├─ Main DB
                                  ├─ Legacy DB
                                  ├─ User Service
                                  └─ Other Services
```

**Benefits**:
- ✅ Fast read operations
- ✅ No write contention
- ✅ Easy to scale horizontally
- ✅ Clear separation of concerns

**Learn More**: [[01-overview|Service Overview]]

---

### 2. Group Order Management (Shuttle)
Specialized handling untuk Cititrans shuttle orders:

```
Order Status Lifecycle:
SCHEDULE → ACTIVE → HISTORY
(automated via cronjob based on order_date)
```

**Features**:
- Batch upsert operations
- Smart status transitions
- Reschedule with new order creation
- Soft delete with audit trail
- Localized content (EN/ID)

**Learn More**: [[03-group-order-flows|Group Order Flows]]

---

### 3. Dual Protocol Support

**gRPC** (Primary):
```
grpc://orderquery:6005
```
- Binary protocol
- Type-safe via Protobuf
- Bidirectional streaming support
- Used by internal services

**REST** (via Gateway):
```
http://orderquery:8005
```
- JSON over HTTP
- Easy to test (curl, Postman)
- Auto-generated from gRPC
- Used by external clients

**Learn More**: [[02-api-reference|API Documentation]]

---

### 4. Clean Architecture
```
Transport → UseCase → Repository → Data
```

**Layer Responsibilities**:
- **Transport**: Request/response handling, interceptors
- **UseCase**: Business logic, orchestration
- **Repository**: Data access abstraction
- **Data**: PostgreSQL, gRPC clients

**Learn More**: [[01-overview#Clean Architecture Implementation|Architecture]]

---

## 📊 Service Metrics

### Key Performance Indicators
- **Availability**: 99.9% uptime
- **Latency P95**: <200ms
- **Request Rate**: ~500 req/s peak
- **Error Rate**: <0.1%
- **Database Connections**: 25 max (main), 10 max (legacy)

### Traffic Statistics
- **Daily Queries**: ~1M orders queried
- **Active Orders**: ~5K concurrent
- **Group Orders**: ~20K shuttle bookings/day
- **Peak Hour**: 07:00-09:00 WIB (morning commute)

### Health Dashboard
- [Service Metrics](http://grafana/d/orderquery)
- [Database Performance](http://grafana/d/orderquery-db)
- [External Service Calls](http://grafana/d/orderquery-external)

---

## 🚨 Important Notes

### Supported Service Categories

| Category | Product Types | Status |
|----------|---------------|--------|
| **RIDE** | BlueBird, SilverBird, GoldenBird | ✅ Full Support |
| **SHUTTLE** | Cititrans Executive, Super Executive, Suite, JAC | ✅ Full Support |
| **RENT** | BlueBird Rent, SilverBird Rent, GoldenBird Rent | ✅ Full Support |
| **DELIVERY** | Birdkirim | ✅ Basic Support |

### Database Strategy
- **Main DB**: New orders (microservices)
- **Legacy DB**: Old orders (monolith, read-only)
- **Migration**: Gradual migration ongoing

### Known Limitations
1. Legacy DB is read-only (no writes)
2. Group orders only for shuttle category
3. Some product types may have limited data
4. Reschedule depends on Fare Service availability

---

## 🤝 Contributing

### Code Standards
- Follow Go best practices
- Write tests (current coverage: ~70%, target: 85%)
- Document new features
- Update API reference if adding endpoints

### PR Checklist
- [ ] Tests pass (`go test ./...`)
- [ ] Linter clean (`golangci-lint run`)
- [ ] Documentation updated
- [ ] Changelog entry added
- [ ] Breaking changes documented (if any)

### Getting Help
- **Slack**: #mrg-platform-team
- **On-Call**: Use PagerDuty
- **Questions**: Ask in team standup

---

## 📞 Support

### Ownership
- **Team**: MRG Platform Team
- **Tech Lead**: [Name]
- **On-Call**: Rotation via PagerDuty
- **Escalation**: Platform Lead

### Incident Response
1. Check [[04-dependencies#Health Checks|Health Checks]]
2. Review [[04-dependencies#Troubleshooting|Troubleshooting Guide]]
3. Check service logs in Kibana
4. Check external service dependencies
5. Escalate if needed (P0/P1)

### Useful Links
- [Runbook](link-to-runbook)
- [Grafana Dashboard](http://grafana/d/orderquery)
- [Kibana Logs](http://kibana/...)
- [PagerDuty](https://bluebird.pagerduty.com/...)
- [Swagger UI](http://orderquery:8005/swagger)

---

## 🗺️ Roadmap

### Q1 2025
- [ ] Migrate more orders from legacy DB to main DB
- [ ] Implement caching layer (Redis)
- [ ] Add bulk operations for group orders
- [ ] Performance optimization (target P95 < 100ms)

### Q2 2025
- [ ] Complete legacy DB migration
- [ ] Add GraphQL endpoint (optional)
- [ ] Implement read replicas
- [ ] Advanced filtering & search

### Q3 2025
- [ ] Multi-region support
- [ ] Pagination improvements
- [ ] Enhanced analytics endpoints
- [ ] Archive old data strategy

---

## 📖 Related Resources

### Internal Documentation
- Service repository: `git.bluebird.id/mybb-ms/orderquery`
- Full documentation: [[_doc/README.md|_doc folder]]
- Architecture doc: [[_doc/02-ARCHITECTURE.md]]
- Project structure: [[_doc/03-PROJECT-STRUCTURE.md]]

### External Resources
- [Prometheus Metrics](http://prometheus/...)
- [Grafana Dashboards](http://grafana/...)
- [API Swagger UI](http://orderquery:8005/swagger)
- [Protobuf Definition](contract/order_query.proto)

### Team Documentation
- [[/02-Work/Teams/MRG/01-overview|MRG Team Overview]]
- [[/02-Work/Teams/MRG/02-services/|All MRG Services]]

---

## 📋 API Summary

### Most Used Endpoints

| Endpoint | Purpose | Usage |
|----------|---------|-------|
| `GetOrders` | Get paginated orders | Very High |
| `GetActiveOrders` | Get in-progress orders | Very High |
| `GetTripHistory` | Get completed trips | High |
| `GetSingleOrder` | Get order details | Very High |
| `GetListOrder` | Get group orders (shuttle) | High |
| `GetBillingDetails` | Get billing breakdown | Medium |
| `CheckEligibleReschedule` | Check reschedule eligibility | Medium |
| `GetChatRoomId` | Get chat room for driver | Medium |

### Query Performance

| Query Type | P50 | P95 | P99 |
|------------|-----|-----|-----|
| Get Single Order | 50ms | 150ms | 300ms |
| Get Active Orders | 80ms | 200ms | 400ms |
| Get Trip History | 100ms | 250ms | 500ms |
| Group Orders List | 120ms | 300ms | 600ms |

---

## 🔧 Error Codes Quick Reference

| Code | HTTP | Description |
|------|------|-------------|
| ODQR-4001 | 400 | Missing required parameter |
| ODQR-4002 | 400 | Invalid parameter value |
| ODQR-4003 | 400 | Not available for product type |
| ODQR-4004 | 400 | Cannot get room ID - order closed |
| ODQR-4005 | 400 | Product type not supported |
| ODQR-4041 | 404 | Order not found |
| ODQR-5000 | 500 | Internal server error |
| ODQR-5001 | 500 | Delete pickup instruction failed |

**Full Error Reference**: [[02-api-reference#Error Responses|API Error Codes]]

---

**Last Updated**: 2025-01-07  
**Service Version**: v1.x  
**Documentation Status**: ✅ Complete

---

**Need help?** Start with [[01-overview|Service Overview]] or ask in #mrg-platform-team Slack channel.