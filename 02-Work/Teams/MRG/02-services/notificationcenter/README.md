# Notification Center - Documentation Index

## 📋 Quick Navigation

### Core Documentation
1. [[01-overview|📖 Service Overview]] - Architecture, capabilities, dan status service
2. [[02-email-strategy|📧 Email Strategy Pattern]] - Strategy pattern untuk email routing
3. [[03-rule-engine|⚙️ Rule Engine Architecture]] - Event-driven notification dengan factory pattern
4. [[04-api-reference|🔌 API Reference]] - Comprehensive API documentation
5. [[05-dependencies|🔗 External Dependencies]] - Database, cache, providers, dan libraries
6. [[06-legacy-migration|🚀 Legacy Migration]] - Technical debt dan migration plans

---

## 🎯 Service at a Glance

**Service Name**: `notificationcenter`  
**Team**: MRG (Meta Reservation Gateway)  
**Domain**: Notification & Communication  
**Status**: ⭐⭐⭐⭐ Production-Ready & Mature

### Primary Functions
- **Multi-Channel Notifications**: Email, SMS, Push, WhatsApp
- **Event-Driven Architecture**: Rule engine untuk booking/order/payment events
- **Strategy-Based Email Routing**: Channel × Service × Type routing
- **Multi-Protocol Support**: gRPC, REST, Message Broker, PubSub

### Technology Stack
- **Language**: Go 1.24
- **Architecture**: Layered Architecture + Strategy Pattern
- **Databases**: PostgreSQL + Redis
- **Message Broker**: Kafka / PubSub / RabbitMQ
- **Observability**: Elastic APM + Prometheus + Structured Logging

---

## 📚 Documentation by Category

### Architecture & Design
- [[01-overview#Architecture Highlights|Architecture Overview]]
- [[02-email-strategy|Email Strategy Pattern Deep Dive]]
- [[03-rule-engine|Rule Engine with Factory Pattern]]
- [[01-overview#Key Patterns|Design Patterns Used]]

### API & Integration
- [[04-api-reference#Core Notification APIs|Core Notification APIs]]
- [[04-api-reference#Email V2 API|Email V2 (Strategy-Based)]]
- [[04-api-reference#OTP|OTP Management]]
- [[04-api-reference#Order & Booking Events|Event Listeners]]

### Operations & DevOps
- [[05-dependencies#Health Checks|Health Check Configuration]]
- [[05-dependencies#Configuration|Environment Variables]]
- [[05-dependencies#Troubleshooting|Troubleshooting Guide]]
- [[05-dependencies#Fallback Strategies|Fallback & Recovery]]

### Development
- [[02-email-strategy#Implementation Pattern|Creating New Email Strategy]]
- [[03-rule-engine#Rule Implementation Pattern|Creating New Rule]]
- [[06-legacy-migration#Migration Runbook|Migration Procedures]]

---

## 🔥 Quick Start

### Local Development
```bash
# Clone repository
git clone git@git.bluebird.id:mybb-ms/notificationcenter.git
cd notificationcenter

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
docker build -t notificationcenter:latest .

# Run container
docker run -p 50051:50051 -p 8080:8080 \
  --env-file .env \
  notificationcenter:latest
```

### Kubernetes
```bash
# Deploy to cluster
kubectl apply -f k8s/huawei-application.yaml

# Check status
kubectl get pods -l app=notificationcenter

# View logs
kubectl logs -f deployment/notificationcenter
```

---

## 🎓 Learning Path

### For New Team Members

**Week 1: Understanding the Basics**
1. Read [[01-overview|Service Overview]]
2. Understand [[04-api-reference#Core Notification APIs|Core APIs]]
3. Study [[05-dependencies|External Dependencies]]
4. Run service locally

**Week 2: Architecture Deep Dive**
1. Study [[02-email-strategy|Email Strategy Pattern]]
2. Understand [[03-rule-engine|Rule Engine Architecture]]
3. Review code structure
4. Write first unit test

**Week 3: Hands-On Development**
1. Create new email strategy (following [[02-email-strategy#Implementation Pattern|guide]])
2. Add new rule (following [[03-rule-engine#Rule Implementation Pattern|guide]])
3. Review [[06-legacy-migration|Technical Debt]]
4. Contribute to migration efforts

### For API Consumers

**Quick Integration Guide**:
1. Choose notification channel ([[04-api-reference#API Categories|API Categories]])
2. Review request/response format ([[04-api-reference|API Reference]])
3. Implement error handling ([[04-api-reference#Error Codes|Error Codes]])
4. Test with staging environment
5. Monitor via [[05-dependencies#Health Checks|Health Checks]]

---

## 🏆 Key Features

### 1. Email Strategy Pattern
Routing berdasarkan **Channel × Service × Type**:
```
WebReservation × RIDE × ORDER_CREATED
→ WebReservationRideAirportTransferEmailStrategy
  → Specific template & business logic
```

**Benefits**:
- ✅ 60% code reduction via BaseEmailStrategy
- ✅ Type-safe via Go interfaces
- ✅ Easy to extend (3-step pattern)
- ✅ Centralized email + logging

**Learn More**: [[02-email-strategy|Email Strategy Documentation]]

---

### 2. Rule Engine (Factory Pattern)
Event-driven notification system:
```
Booking Event (APP, CREATED)
→ Registry["APP_CREATED"]
  → Factory creates rule with injected repo
    → Rule.SendNotif(ctx)
      → Multi-channel notification
```

**Benefits**:
- ✅ Zero timing issues
- ✅ Thread-safe by design
- ✅ Easy to add new rules
- ✅ Clear separation of concerns

**Learn More**: [[03-rule-engine|Rule Engine Documentation]]

---

### 3. Multi-Protocol Support

**gRPC** (Primary):
```go
grpc://notificationcenter:50051
```

**REST** (via Gateway):
```bash
http://notificationcenter:8080/api/v1/...
```

**Message Broker**:
- Kafka topics
- PubSub subscriptions
- RabbitMQ queues

**Learn More**: [[04-api-reference|API Documentation]]

---

## 📊 Service Metrics

### Key Performance Indicators
- **Availability**: 99.95% uptime (SLA: 99.9%)
- **Latency P95**: 300ms (Target: <500ms)
- **Email Success Rate**: 98%
- **SMS Success Rate**: 96%
- **Push Success Rate**: 98%

### Traffic Statistics
- **Daily Emails**: ~50,000
- **Daily SMS**: ~20,000
- **Daily Push**: ~100,000
- **Peak Hour**: 09:00-10:00 WIB

### Health Dashboard
- [Service Metrics](http://grafana/d/notificationcenter)
- [Error Tracking](http://grafana/d/notif-errors)
- [Migration Progress](http://grafana/d/migration)

---

## 🚨 Important Notes

### Current Migration Status
Service sedang dalam proses migrasi:
- ✅ Rule Engine Factory Pattern (DONE)
- 🔄 Legacy APIs → Unified APIs (60% complete)
- 🔄 Email Templates → Strategy Pattern (75% complete)
- 🔄 Unicush → Firebase Cloud Messaging (40% complete)

**Details**: [[06-legacy-migration|Legacy Migration Plan]]

### Known Issues
1. Some RENT email flows belum di-implement (non-blocking)
2. Legacy APIs masih aktif (akan di-deprecate Q2 2025)
3. Repository interfaces perlu segregation (planned Q1 2025)

**Track Progress**: [[06-legacy-migration#Technical Debt|Technical Debt Backlog]]

---

## 🤝 Contributing

### Code Standards
- Follow Go best practices
- Write tests (target: 85% coverage)
- Document new features
- Update API reference if adding endpoints

### PR Checklist
- [ ] Tests pass (`go test ./...`)
- [ ] Linter clean (`golangci-lint run`)
- [ ] Documentation updated
- [ ] Changelog entry added
- [ ] Breaking changes documented

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
1. Check [[05-dependencies#Health Checks|Health Checks]]
2. Review [[05-dependencies#Troubleshooting|Troubleshooting Guide]]
3. Check service logs in Kibana
4. Escalate if needed (P0/P1)

### Useful Links
- [Runbook](link-to-runbook)
- [Grafana Dashboard](http://grafana/d/notificationcenter)
- [Kibana Logs](http://kibana/...)
- [PagerDuty](https://bluebird.pagerduty.com/...)

---

## 🗺️ Roadmap

### Q1 2025
- [ ] Complete email strategy coverage (RENT flows)
- [ ] Repository interface segregation
- [ ] Configuration management refactoring
- [ ] Test coverage to 85%

### Q2 2025
- [ ] Deprecate legacy APIs
- [ ] Complete FCM migration
- [ ] Performance optimization (P95 < 200ms)
- [ ] C4 architecture diagrams

### Q3 2025
- [ ] Corporate channel email strategies
- [ ] Advanced analytics & reporting
- [ ] Rate limiting enhancements
- [ ] Multi-region support

**Full Details**: [[06-legacy-migration#Planned Migrations|Migration Roadmap]]

---

## 📖 Related Resources

### Internal Documentation
- Service repository: `git.bluebird.id/mybb-ms/notificationcenter`
- Architecture assessment: [[_docs/architecture_assessment.md]]
- Rule engine refactoring: [[_docs/RULE_ENGINE_REFACTORING.md]]

### External Resources
- [Prometheus Metrics](http://prometheus/...)
- [Grafana Dashboards](http://grafana/...)
- [API Swagger UI](http://notificationcenter:8080/swagger)

### Team Documentation
- [[/02-Work/Teams/MRG/01-overview|MRG Team Overview]]
- [[/02-Work/Teams/MRG/02-services/|All MRG Services]]

---

**Last Updated**: 2025-01-07  
**Service Version**: v2.x  
**Documentation Status**: ✅ Complete

---

**Need help?** Start with [[01-overview|Service Overview]] or ask in #mrg-platform-team Slack channel.