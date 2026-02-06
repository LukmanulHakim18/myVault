# RabbitMQ Chaos Broker Test - Setup Complete

**Created:** 2026-02-03  
**Location:** `D:\code\go\Arsitech\rabit-mq-banchmark\chaos-brocker-test`  
**Status:** ✅ Ready for Testing  

---

## 📦 Project Structure

```
chaos-brocker-test/
├── docker-compose.yml          ✅ 3-node cluster + monitoring
├── rabbitmq.conf               ✅ Quorum queue optimized
├── enabled_plugins             ✅ Management + Prometheus
├── prometheus.yml              ✅ Metrics collection
├── scripts/
│   ├── setup-cluster.sh        ✅ Cluster initialization
│   ├── generate-load.go        ✅ Load generator (Go)
│   └── go.mod                  ✅ Dependencies
├── tests/
│   ├── F01-leader-graceful.sh  ✅ Graceful shutdown test
│   ├── F02-leader-crash.sh     ✅ Hard kill test
│   └── F03-follower-failure.sh ✅ Follower down test
├── results/                    📁 Auto-generated
└── README.md                   ✅ Complete guide
```

## 🎯 Test Scenarios Implemented

### Priority High (P0)
- ✅ **F01**: Leader Graceful Shutdown - Normal failover
- ✅ **F02**: Leader Hard Kill (SIGKILL) - Crash scenario
- ✅ **F03**: Follower Failure - Quorum availability

### Priority Medium (P1) - To Implement
- ⏸️ **F04**: Network Partition - Split-brain test
- ⏸️ **F05**: Rolling Failures - Cascading scenario
- ⏸️ **F06**: Simultaneous Multi-node - Majority failure

## 🚀 Quick Commands

### Setup
```bash
cd D:\code\go\Arsitech\rabit-mq-banchmark\chaos-brocker-test
chmod +x scripts/*.sh tests/*.sh
./scripts/setup-cluster.sh
```

### Run Tests
```bash
# Test graceful failover
./tests/F01-leader-graceful.sh

# Test crash scenario
./tests/F02-leader-crash.sh

# Test follower impact
./tests/F03-follower-failure.sh
```

### Monitor
- Zone A: http://localhost:15672 (admin/admin123)
- Zone B: http://localhost:15673 (admin/admin123)
- Zone C: http://localhost:15674 (admin/admin123)
- Grafana: http://localhost:3000 (admin/admin123)
- Prometheus: http://localhost:9090

### Cleanup
```bash
docker-compose down -v
```

## 📊 Expected Metrics

**Leader Failover (F01, F02):**
- Election time: 5-15 seconds
- Message loss: 0% (committed)
- Throughput degradation: <50% during failover
- Recovery time: <60s to 90% capacity

**Follower Failure (F03):**
- No leader election
- Minimal latency impact: <10%
- Operations continue normally

## 📝 Test Execution Checklist

- [ ] Setup cluster (./scripts/setup-cluster.sh)
- [ ] Verify all 3 nodes healthy
- [ ] Verify quorum queue created
- [ ] Install Go dependencies (go mod download)
- [ ] Run F01 test (graceful shutdown)
- [ ] Analyze F01 results
- [ ] Run F02 test (hard kill)
- [ ] Analyze F02 results
- [ ] Run F03 test (follower failure)
- [ ] Analyze F03 results
- [ ] Document findings
- [ ] Create Grafana dashboards
- [ ] Write production recommendations

## 🔧 Key Features

**Load Generator (Go):**
- 3 producers, 3 consumers
- Configurable rate (default: 1000 msg/s)
- Automatic reconnection logic
- Publisher confirms enabled
- Real-time metrics collection
- CSV output for analysis

**Test Scripts (Bash):**
- Automatic leader detection
- Coordinated failure injection
- Leader election monitoring
- Pre/post failure snapshots
- Detailed logging

**Monitoring:**
- Prometheus metrics scraping (5s interval)
- Grafana dashboards
- Per-zone metrics collection
- RabbitMQ Management UI

## 💡 Next Actions

**Immediate (Week 1):**
1. Run baseline tests (F01-F03)
2. Analyze metrics and validate behavior
3. Document leader election times
4. Measure message durability

**Short-term (Week 2-3):**
1. Implement F04 (network partition)
2. Implement F05 (rolling failures)
3. Create automated test suite
4. Build comprehensive Grafana dashboards

**Long-term (Month 1-2):**
1. Production architecture design
2. Disaster recovery procedures
3. Runbook creation
4. Team training materials

## 📚 Related Documents

- [[rabbitmq-multi-zone-failover-test-plan]] - Complete test plan
- [[rabbitmq-failover-docker-setup]] - Detailed setup guide
- [[rabbitmq-benchmark-comprehensive-analysis]] - Performance baseline

## ✅ Validation Checklist

Before running tests, verify:
- [ ] Docker Desktop running
- [ ] At least 8GB RAM available
- [ ] Ports 5672-5674, 15672-15674, 3000, 9090 free
- [ ] Go 1.21+ installed
- [ ] Git Bash available (Windows)

---

**Status:** Ready for execution  
**Maintainer:** Architecture Team  
**Version:** 1.0
