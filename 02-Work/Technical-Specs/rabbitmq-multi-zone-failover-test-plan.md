# RabbitMQ Quorum Queue Multi-Zone Failover Test Plan

**Test Date:** TBD  
**Environment:** Multi-Zone Docker Setup (3 nodes across 3 zones)  
**Queue Type:** Quorum Queue Only  
**Objective:** Validate failover behavior, data durability, and recovery in multi-zone failures

---

## Test Objectives

1. **Leader Failover**: Validate Raft leader election dan message continuity
2. **Data Durability**: Ensure zero message loss during node failures
3. **Client Resilience**: Test producer/consumer reconnection behavior
4. **Performance Impact**: Measure latency/throughput during failover
5. **Recovery Time**: Measure RTO (Recovery Time Objective) untuk different scenarios
6. **Network Partition**: Test split-brain scenarios dan conflict resolution

---

## Test Environment Setup

### Multi-Zone Topology

```
Zone A (Primary)     Zone B (Secondary)   Zone C (DR)
─────────────────    ──────────────────   ───────────────
rabbitmq-zone-a      rabbitmq-zone-b      rabbitmq-zone-c
  - Leader (initial)   - Follower           - Follower
  - IP: 172.20.1.10    - IP: 172.20.2.10    - IP: 172.20.3.10
  - CPU: 2 cores       - CPU: 2 cores       - CPU: 2 cores
  - RAM: 2GB           - RAM: 2GB           - RAM: 2GB
```

### Quorum Queue Configuration

```yaml
queue_name: test.quorum.failover
type: quorum
replication_factor: 3
min_quorum: 2
arguments:
  x-quorum-initial-group-size: 3
  x-max-length: 100000
  x-message-ttl: 3600000
```

### Client Configuration

```yaml
producers:
  count: 3
  rate_limit: 1000 msg/s per producer
  confirm_mode: true
  connection_strategy: round_robin
  endpoints:
    - rabbitmq-zone-a:5672
    - rabbitmq-zone-b:5672
    - rabbitmq-zone-c:5672

consumers:
  count: 3
  prefetch_count: 50
  ack_mode: manual
  connection_strategy: round_robin
```

---

## Test Scenarios

### F01: Leader Node Failure (Graceful Shutdown)

**Objective:** Test Raft leader election dengan graceful shutdown

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Monitor metrics for 30s (baseline)
3. Identify current leader node
4. Execute graceful shutdown: `docker stop rabbitmq-zone-a`
5. Monitor leader election process
6. Continue traffic for 60s
7. Verify message continuity
8. Check consumer lag

**Expected Results:**
- Leader election completes within 5-10 seconds
- Zero message loss
- Consumers reconnect automatically
- Throughput recovers to 90%+ within 30s

**Metrics to Capture:**
- Leader election time
- Messages published during failover
- Messages delivered during failover
- Consumer reconnection time
- Throughput degradation percentage
- Latency p50, p99 during failover

---

### F02: Leader Node Failure (Hard Kill)

**Objective:** Test behavior dengan sudden node death (crash scenario)

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Monitor metrics for 30s (baseline)
3. Identify current leader node
4. Execute hard kill: `docker kill -s SIGKILL rabbitmq-zone-a`
5. Monitor leader election process
6. Continue traffic for 60s
7. Verify message continuity

**Expected Results:**
- Leader election completes within 10-15 seconds (slower than graceful)
- Zero message loss (messages in quorum)
- Client timeout errors during election
- Automatic retry and recovery

**Metrics to Capture:**
- Leader election time
- Number of failed publish attempts
- Number of connection errors
- Time to full recovery
- Message loss (should be 0)

---

### F03: Follower Node Failure

**Objective:** Test impact of follower failure pada quorum availability

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Identify current follower nodes
3. Execute: `docker stop rabbitmq-zone-b` (follower)
4. Continue traffic for 60s
5. Monitor quorum status
6. Verify operations continue normally

**Expected Results:**
- No leader election triggered
- Operations continue with 2-node quorum
- Minimal latency impact (<10% increase)
- No message loss
- Replication continues to remaining follower

**Metrics to Capture:**
- Latency impact (p50, p99)
- Throughput impact
- Replication lag to remaining follower
- Client connection distribution

---

### F04: Network Partition (Zone Isolation)

**Objective:** Test split-brain scenario dengan network partition

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Create network partition:
   ```bash
   # Isolate Zone A from Zone B & C
   iptables -A INPUT -s 172.20.2.0/24 -j DROP
   iptables -A INPUT -s 172.20.3.0/24 -j DROP
   ```
3. Monitor cluster status (Zone A isolated)
4. Continue traffic for 60s
5. Heal partition:
   ```bash
   iptables -F
   ```
6. Monitor cluster reconciliation

**Expected Results:**
- Minority partition (Zone A) stops serving requests
- Majority partition (Zone B + C) elects new leader
- Clients connected to Zone A reconnect to Zone B/C
- After heal: Zone A rejoins as follower
- Zero message loss

**Metrics to Capture:**
- Partition detection time
- Leader election time in majority partition
- Client reconnection time
- Messages published to each partition
- Reconciliation time after heal

---

### F05: Rolling Node Failures

**Objective:** Test cascading failure scenario

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Stop Zone A (leader) - wait for election
3. After 30s, stop Zone B (new leader)
4. Monitor Zone C (last node standing)
5. Verify cluster cannot operate (no quorum)
6. Restart Zone A
7. Monitor cluster recovery
8. Restart Zone B

**Expected Results:**
- First failure: Leader election to Zone B/C
- Second failure: Cluster stops accepting writes (no quorum)
- After restart Zone A: Quorum restored, operations resume
- Message durability maintained

**Metrics to Capture:**
- Time without quorum (unavailability window)
- Messages in queue before/after failures
- Recovery time to operational state
- Message loss (should be 0)

---

### F06: Simultaneous Multi-Node Failure

**Objective:** Test data loss prevention dengan majority failure

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Simultaneously stop 2 nodes:
   ```bash
   docker stop rabbitmq-zone-a rabbitmq-zone-b
   ```
3. Monitor Zone C (minority node)
4. Verify cluster unavailable
5. Restart both nodes
6. Monitor cluster recovery
7. Verify message integrity

**Expected Results:**
- Cluster stops accepting writes (no quorum)
- Minority node (Zone C) enters read-only state
- After restart: Quorum restored
- All committed messages present
- In-flight messages may be lost (expected)

**Metrics to Capture:**
- Unavailability duration
- Messages committed before failure
- Messages recovered after restart
- Recovery time to operational state

---

### F07: Leader + Follower Failure (Quorum Loss)

**Objective:** Test worst-case scenario - quorum loss

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Stop leader + 1 follower (2 out of 3 nodes)
3. Monitor remaining node
4. Verify cluster unavailable
5. Restart 1 failed node (restore quorum)
6. Monitor recovery
7. Restart 2nd failed node

**Expected Results:**
- Immediate cluster unavailability
- Client connection failures
- After 1 node restart: Quorum restored, operations resume
- Message durability maintained for committed messages

---

### F08: Zone Failure Recovery (Full Zone Down)

**Objective:** Test complete zone failure dan recovery

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Simulate zone failure: stop all services in Zone A
3. Monitor failover to Zone B/C
4. Continue traffic for 5 minutes
5. Restore Zone A
6. Monitor node rejoin dan replication catch-up

**Expected Results:**
- Automatic failover to remaining zones
- Leader in Zone B or C
- Zone A rejoins as follower
- Replication catch-up completes
- Full cluster health restored

**Metrics to Capture:**
- Failover time
- Replication catch-up duration
- Data transferred during catch-up
- Throughput during degraded state

---

### F09: Flapping Node (Repeated Failures)

**Objective:** Test behavior dengan unstable node (crash loop)

**Steps:**
1. Start baseline traffic (1000 msg/s)
2. Simulate flapping node (Zone A):
   ```bash
   # Loop: stop 10s, start 20s, repeat 5x
   for i in {1..5}; do
     docker stop rabbitmq-zone-a
     sleep 10
     docker start rabbitmq-zone-a
     sleep 20
   done
   ```
3. Monitor cluster stability
4. Check leader election frequency
5. Verify message continuity

**Expected Results:**
- Cluster remains operational with 2 stable nodes
- Minimal impact on throughput
- Flapping node eventually stabilizes
- No data corruption

---

### F10: Client Failover Behavior

**Objective:** Test producer/consumer behavior during failures

**Steps:**
1. Start 5 producers, 5 consumers
2. All clients connect to Zone A initially
3. Stop Zone A
4. Monitor client reconnection behavior
5. Verify message continuity
6. Check message ordering

**Expected Results:**
- Clients detect connection loss within 5s
- Automatic reconnection to Zone B/C
- Message publishing resumes
- Consumer processing continues
- Message ordering maintained per queue

**Metrics to Capture:**
- Client connection error count
- Reconnection time (per client)
- Messages lost during reconnection
- Duplicate message rate

---

## Performance Benchmarks During Failover

### Key Metrics to Track

**Availability Metrics:**
- Leader election time (target: <10s)
- Time to quorum restoration (target: <15s)
- Total unavailability window (target: <30s)

**Durability Metrics:**
- Messages committed before failure
- Messages recovered after failure
- Message loss rate (target: 0%)

**Performance Metrics:**
- Baseline throughput: 3000 msg/s
- During failover throughput: >2000 msg/s (>66%)
- Recovery to 90% throughput: <60s
- Latency increase during failover: <2x baseline

**Client Metrics:**
- Connection error rate
- Reconnection success rate
- Retry attempts before success
- Duplicate detection rate

---

## Test Automation Script Structure

```go
// Pseudo-code structure
type FailoverTest struct {
    Cluster      *RabbitMQCluster
    Producers    []*Producer
    Consumers    []*Consumer
    Metrics      *MetricsCollector
    Scenario     TestScenario
}

func (t *FailoverTest) Run() {
    // Phase 1: Setup
    t.SetupCluster()
    t.StartProducers()
    t.StartConsumers()
    
    // Phase 2: Baseline
    t.CollectBaselineMetrics(30 * time.Second)
    
    // Phase 3: Inject Failure
    t.ExecuteFailureScenario()
    
    // Phase 4: Monitor Recovery
    t.MonitorRecovery(60 * time.Second)
    
    // Phase 5: Validate
    t.ValidateMessageIntegrity()
    t.GenerateReport()
}
```

---

## Success Criteria

### Must Have (P0)
- ✅ Zero message loss untuk committed messages
- ✅ Leader election completes within 15 seconds
- ✅ Automatic client reconnection successful
- ✅ Cluster recovers to operational state

### Should Have (P1)
- ✅ Throughput degradation <50% during failover
- ✅ Recovery to 90% throughput within 60s
- ✅ Latency increase <2x during failover
- ✅ No split-brain scenarios

### Nice to Have (P2)
- ✅ Leader election <10 seconds
- ✅ Zero duplicate messages
- ✅ Ordered message delivery maintained

---

## Monitoring & Observability

### Grafana Dashboards
- Cluster health status (per node)
- Leader election timeline
- Message flow during failure
- Client connection status
- Network partition detection

### Alerts to Configure
- Node down (critical)
- Leader election in progress (warning)
- Quorum lost (critical)
- High replication lag (warning)
- Client connection errors (warning)

---

## Risk Assessment

### High Risk Scenarios
1. **Majority node failure** → Cluster unavailable
2. **Network partition** → Split-brain risk
3. **Disk full on leader** → Cannot commit messages

### Mitigation Strategies
1. **Monitoring**: Real-time alerts untuk node health
2. **Automation**: Auto-recovery scripts
3. **Capacity**: Maintain 50% headroom pada each node
4. **Backup**: Regular queue backups
5. **Documentation**: Runbook untuk manual intervention

---

## Next Steps

1. [ ] Setup multi-zone Docker environment
2. [ ] Implement test automation framework
3. [ ] Configure monitoring dashboards
4. [ ] Execute baseline performance tests
5. [ ] Run all failover scenarios (F01-F10)
6. [ ] Generate comprehensive report
7. [ ] Document findings dan recommendations

---

**Document Version:** 1.0  
**Created:** 2026-02-03  
**Author:** Architecture Team  
**Status:** Draft - Pending Review
