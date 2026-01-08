---
tags:
  - runbook
  - incident-response
  - database
  - postgresql
  - patroni
  - p0
type: runbook
severity: P0
created: '2026-01-08'
updated: '2026-01-08'
owner: platform-engineering
---
# 🚨 RUNBOOK: Database Failover

**Severity**: P0 - Critical  
**Blast Radius**: 100%  
**RTO Target**: < 5 minutes  
**RPO Target**: 0 (zero data loss)

---

## 📋 Quick Reference

| Item | Value |
|------|-------|
| Primary DB Host | `pg-master.mrg.internal:5432` |
| Replica 1 | `pg-replica-1.mrg.internal:5432` |
| Replica 2 | `pg-replica-2.mrg.internal:5432` |
| PgBouncer Write | `pgbouncer.mrg.internal:6432` |
| PgBouncer Read | `pgbouncer.mrg.internal:6433` |
| Patroni API | `http://pg-master:8008/patroni` |
| Grafana Dashboard | [DB Monitoring](https://grafana.bluebird.id/d/postgres) |
| On-Call | #mrg-oncall Slack |

---

## 🔍 Detection & Symptoms

### Alerts That Trigger This Runbook
```
- PostgresMasterDown: up{job="postgres-master"} == 0
- PatroniNoLeader: patroni_cluster_has_leader == 0
- DatabaseConnectionExhausted: pg_stat_activity_count > 450
- ReplicationLagCritical: pg_replication_lag_seconds > 30
```

### Symptoms
- [ ] All services returning database connection errors
- [ ] Order creation failing 100%
- [ ] Payment callbacks failing
- [ ] User authentication failing
- [ ] Grafana showing no metrics from PostgreSQL

---

## 🚀 Immediate Actions (First 5 Minutes)

### Step 1: Verify the Issue (30 seconds)
```bash
# Check Patroni cluster status
patronictl -c /etc/patroni/patroni.yml list

# Expected output (healthy):
# + Cluster: mrg-postgres (12345678) ----+----+-----------+
# | Member    | Host       | Role    | State   | Lag in MB |
# +-----------+------------+---------+---------+-----------+
# | pg-node-1 | 10.0.1.10  | Leader  | running |           |
# | pg-node-2 | 10.0.1.11  | Replica | running |         0 |
# | pg-node-3 | 10.0.1.12  | Replica | running |         0 |

# If no leader shown - FAILOVER NEEDED
```

### Step 2: Check if Automatic Failover Happened (30 seconds)
```bash
# Check Patroni logs for recent failover
journalctl -u patroni -n 50 --no-pager | grep -i "failover\|switchover\|leader"

# Check current leader
curl -s http://pg-node-1:8008/patroni | jq '.role'
curl -s http://pg-node-2:8008/patroni | jq '.role'
curl -s http://pg-node-3:8008/patroni | jq '.role'
```

### Step 3: If No Automatic Failover - Manual Intervention

#### Scenario A: Patroni Working, Just Need Failover
```bash
# Initiate manual failover to best candidate
patronictl -c /etc/patroni/patroni.yml failover mrg-postgres

# Follow prompts:
# - Select candidate (usually lowest lag)
# - Confirm failover

# Verify new leader
patronictl -c /etc/patroni/patroni.yml list
```

#### Scenario B: Patroni Not Responding
```bash
# Check etcd cluster health
etcdctl endpoint health --cluster

# If etcd down, restart etcd first
sudo systemctl restart etcd

# Then restart Patroni on all nodes
sudo systemctl restart patroni
```

#### Scenario C: Complete Cluster Failure (Disaster Recovery)
```bash
# 1. Identify node with latest data
for node in pg-node-1 pg-node-2 pg-node-3; do
    echo "=== $node ==="
    ssh $node "sudo -u postgres pg_controldata /var/lib/postgresql/data | grep 'Latest checkpoint'"
done

# 2. Start that node as standalone master
ssh $LATEST_NODE "sudo -u postgres pg_ctl start -D /var/lib/postgresql/data"

# 3. Update PgBouncer to point to this node
# Edit /etc/pgbouncer/pgbouncer.ini
# Restart pgbouncer

# 4. Rebuild cluster from this node
# ... (contact DBA team)
```

---

## 📊 Verification Steps

### Verify Database is Accepting Connections
```bash
# Test write operation
psql -h pgbouncer.mrg.internal -p 6432 -U app_user -d mrg -c "SELECT 1"

# Test from application
curl -s http://orderorchestrator:8080/health | jq '.checks.database'
```

### Verify Replication is Working
```bash
# Check replication status
psql -h $NEW_LEADER -U postgres -c "SELECT * FROM pg_stat_replication;"

# Check replica lag
patronictl -c /etc/patroni/patroni.yml list
```

### Verify Application Recovery
```bash
# Check all services health
for svc in orderorchestrator userservice paymentprocessor sessionmanager; do
    echo "=== $svc ==="
    curl -s http://$svc:8080/ready | jq '.status'
done
```

---

## 📢 Communication

### Stakeholder Notification Template
```
🚨 P0 INCIDENT: Database Failover in Progress

Impact: All MRG services affected
Start Time: [TIME] WIB
Current Status: Failover in progress / Recovered

Timeline:
- [TIME]: Database master failure detected
- [TIME]: Automatic/Manual failover initiated
- [TIME]: New master elected: [NODE]
- [TIME]: Services recovering

ETA to Resolution: [X] minutes
Next Update: [TIME]

On-Call: @[NAME]
```

### Slack Channels to Notify
- [ ] #mrg-incidents (primary)
- [ ] #platform-oncall
- [ ] #ops-war-room (if extended outage)

---

## 🔧 Post-Incident Tasks

### Immediate (Within 1 Hour)
- [ ] Verify all services healthy
- [ ] Check for stuck transactions
- [ ] Review failed jobs/messages in queues
- [ ] Replay failed payment webhooks if needed

### Within 24 Hours
- [ ] Rebuild failed node and rejoin cluster
- [ ] Review Patroni logs for root cause
- [ ] Check for data inconsistencies
- [ ] Update incident ticket with timeline

### Post-Mortem
- [ ] Schedule post-mortem meeting
- [ ] Document root cause
- [ ] Identify improvement actions
- [ ] Update this runbook if needed

---

## 📚 Related Documents

- [[MRG SPOF Assessment & Mitigation Strategy]]
- [[Database HA Architecture Design]]
- [[Patroni Configuration Guide]]
- [[PgBouncer Operations Guide]]

---

**Last Updated**: 2026-01-08  
**Owner**: Platform Engineering Team  
**Review Cycle**: Quarterly
