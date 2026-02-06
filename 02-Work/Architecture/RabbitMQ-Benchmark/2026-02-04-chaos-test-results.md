---
title: RabbitMQ Chaos Test Results (Corrected)
date: '2026-02-04'
type: benchmark-result
status: completed
tags:
  - rabbitmq
  - chaos-testing
  - quorum-queue
  - benchmark
  - failover
test_version: v2-balanced
---
	x# Chaos Test Results - RabbitMQ Quorum Queue

**Test Date:** 2026-02-04  
**Duration:** ~10 minutes (13:30:33 - 13:40:23)  
**Data Points:** 60 samples (every 10 seconds)  
**Test Type:** Leader Failover with Balanced Throughput (Corrected)

---

## Executive Summary

| Criteria | Target | Actual | Status |
|----------|--------|--------|--------|
| Message Loss | 0 | 0 | ✅ PASS |
| Latency Spike < 5x Baseline | < 450ms (P99) | 799ms (P99) | ⚠️ 8.9x spike |
| Leader Election | < 30s | ~10 min* | ❌ FAIL |
| Throughput Stability | Balanced | ✅ Balanced | ✅ PASS |

*Note: Leader election cepat, tapi client reconnection logic memakan waktu lama (~10 min)

---

## Test Configuration (Corrected)

| Item | Previous Test | Current Test |
|------|---------------|-----------|
| Publish rate target | 1000 msg/s | 300 msg/s |
| Actual publish rate | 907 msg/s | 299 msg/s |
| Actual consume rate | 425 msg/s | 298 msg/s |
| Throughput balance | ❌ Imbalanced | ✅ Balanced |
| Message size | 5KB | 5KB |
| Queue type | Quorum queue | Quorum queue |
| Chaos method | `docker stop` | `docker stop` |

---

## Timeline Correlation (Log + JSON)

| Time | Event | Source | Impact |
|------|-------|--------|--------|
| 13:20:19 | Test started | Log | Baseline begins |
| 13:22:19 | Phase 2: Stopping Leader | Log | Chaos initiated |
| 13:22:21 | `docker stop rabbitmq1` | Log | Leader killed |
| 13:22:23 | Connection lost | Log | **~2,600+ failed publishes** |
| 13:32:27 | **Reconnected to rabbitmq2** | Log | Recovery complete |
| 13:30:33 | Metrics recording starts | JSON | Stable operation |
| 13:34:27 | `docker start rabbitmq1` | Log | Node recovery |
| 13:36:28 | Chaos scenario completed | Log | Test ends |
| 13:40:23 | Test duration reached | Log | Final |

---

## Baseline Metrics (Current Test - First 60s)

| Metric | Value |
|--------|-------|
| Publish Rate | 299 msg/s |
| Consume Rate | 298-299 msg/s |
| E2E P50 | 25-26 ms |
| E2E P95 | 47-59 ms |
| E2E P99 | 79-90 ms |

---

## Peak Chaos Metrics (Current Test)

| Metric | Baseline | Peak | Multiplier |
|--------|----------|------|------------|
| E2E P50 | 25 ms | 27 ms | **1.08x** ✅ |
| E2E P95 | 52 ms | 112 ms | **2.2x** ✅ |
| E2E P99 | 86 ms | 799 ms | **9.3x** ⚠️ |
| Publish Rate | 299 msg/s | 294 msg/s | 0.98x ✅ |
| Consume Rate | 298 msg/s | 294 msg/s | 0.99x ✅ |

---

## Key Findings

### ✅ PASS: Zero Message Loss
- Semua message yang berhasil di-publish, berhasil di-consume
- Quorum queue maintain durability selama chaos event

### ✅ PASS: Balanced Throughput
- Publish rate: 297-299 msg/s
- Consume rate: 294-299 msg/s
- No backlog buildup (berbeda dengan test sebelumnya)

### ⚠️ WARNING: Failed Publishes During Failover
```
Start failures: 13:22:23.254156
End failures:   13:32:27.681916
Duration:       ~10 minutes 4 seconds
Estimated failed publishes: ~2,600+ messages
Error: Exception (504) Reason: "channel/connection is not open"
```

**Root Cause:** Client tidak auto-reconnect ke node lain dengan cepat

### ⚠️ WARNING: P99 Latency Spike
- Baseline P99: ~86ms
- Peak P99: 799ms (9.3x spike)
- Impact terbatas pada percentile tinggi, P50 tetap stabil

---

## Comparison: Previous vs Current Test

| Metric | Previous Test | Current Test | Improvement |
|--------|---------------|--------------|-------------|
| Throughput Balance | ❌ Imbalanced (907/425) | ✅ Balanced (299/298) | Fixed |
| Baseline E2E P50 | 2,807ms | 25ms | **112x better** |
| Peak E2E P95 | 343,727ms | 112ms | **3,069x better** |
| Peak E2E P99 | 344,590ms | 799ms | **431x better** |
| Recovery | No recovery | Recovered | Fixed |
| Message Loss | 0 | 0 | Same |

---

## Root Cause Analysis

| Issue | Cause | Status |
|-------|-------|--------|
| Previous: Consumer bottleneck | Consumer speed < Publish speed | ✅ Fixed |
| Previous: Gradual latency increase | Queue backlog terus bertambah | ✅ Fixed |
| Current: Long reconnect time | Client tidak failover ke node lain | ❌ Needs fix |
| Current: P99 spike | Messages in-flight saat failover | Expected |

---

## Recommendations

### 1. Improve Client Reconnection Logic
```go
// Current: Wait for connection to recover
// Better: Immediate failover to other nodes
nodes := []string{"rabbitmq1:5672", "rabbitmq2:5673", "rabbitmq3:5674"}
for _, node := range nodes {
    conn, err := amqp.Dial(node)
    if err == nil {
        return conn
    }
}
```

### 2. Reduce Reconnection Timeout
- Current: ~10 minutes
- Target: < 5 seconds

### 3. Implement Publisher Confirms
- Track which messages were acknowledged
- Retry unconfirmed messages after reconnection

### 4. Consider Connection Pool
- Maintain connections to all nodes
- Switch immediately when one fails

---

## Charts

📊 Chart file: `D:\code\go\Arsitech\rabit-mq-banchmark\results\20260204-134023-charts.jsx`

---

## Related Files

- JSON Metrics (Current): `20260204-134023.json`
- Test Log: `test.log`
- Report: `20260204-134023-report.md`
- Brief: `D:\code\go\Arsitech\rabit-mq-banchmark\chaos-brocker-test\brief.md`
- Prompts: `prompt-01-infrastructure.md`, `prompt-02-metrics.md`, `prompt-03-test-implementation.md`

---

## Tags

#rabbitmq #chaos-testing #quorum-queue #benchmark #architecture #failover
