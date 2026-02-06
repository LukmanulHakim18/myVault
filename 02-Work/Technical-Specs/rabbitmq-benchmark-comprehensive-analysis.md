# RabbitMQ Comprehensive Benchmark Analysis

**Test Date:** 2026-02-03 00:29 WIB  
**Environment:** Local Docker 3-Node Cluster  
**Test Duration:** 30 seconds per scenario  
**Author:** Architecture Team

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Test Results with Resource Usage](#test-results-with-resource-usage)
3. [Resource Usage Deep Dive](#resource-usage-deep-dive)
4. [Performance Analysis](#performance-analysis)
5. [Production Recommendations](#production-recommendations)

---

## Executive Summary

Comprehensive benchmark dengan **24 test scenarios** menunjukkan:

**🏆 Performance Champions:**
- **Classic T05** (small messages): **25,291 msg/s** - Absolute throughput winner
- **Classic T02** (with confirms): **22,611 msg/s** + latency 4.1ms - Production-ready champion
- **Classic L01** (low rate): **0.3ms p50 latency** - Ultra-low latency champion

**🔒 Reliability Champions:**
- **Quorum T12** (high concurrency): **21,024 msg/s** - Best Quorum throughput
- **Quorum T08** (with confirms): **12,029 msg/s** - Production-ready with HA

**💡 Key Insights:**
- Classic outperforms Quorum by **1.88x** pada production configs
- CPU usage spike hingga **99%** saat high-load tests (leader node bottleneck)
- Memory usage increase **60%** (10% → 16%) during sustained load
- Quorum overhead adalah trade-off untuk **data durability & HA**

---

## Test Results with Resource Usage

### Throughput Tests (T-Series) - 12 Scenarios

**Classic Queue Results:**

| Test ID | Description                | Send Rate    | Recv Rate    | Latency p50 | Latency p99  | CPU Leader | Memory Leader | Performance Rating |
|---------|----------------------------|--------------|--------------|-------------|--------------|-----------|--------------|-------------------|
| T01     | Baseline - no confirms     | 11,572 msg/s | 11,532 msg/s | 7,653 ms    | 16,719 ms    | 99%       | 16.5%        | ⚠️ Unstable       |
| **T02** | **With confirms (PROD)**   | **22,611 msg/s** | **22,611 msg/s** | **4.1 ms** | **15.0 ms** | **99%**   | **16.5%**    | ⭐⭐⭐⭐⭐ Excellent |
| T03     | Multi-producer (5P/5C)     | 19,986 msg/s | 19,986 msg/s | 13.1 ms     | 104.0 ms     | 99%       | 16.6%        | ⭐⭐⭐⭐ Good        |
| T04     | Large messages (10KB)      | 9,688 msg/s  | 9,675 msg/s  | 13.8 ms     | 122.0 ms     | 99%       | 16.7%        | ⭐⭐⭐ Fair         |
| **T05** | **Small messages (128B)**  | **25,291 msg/s** | **25,292 msg/s** | **6.5 ms** | **37.3 ms** | **99%**   | **16.5%**    | ⭐⭐⭐⭐⭐ Peak     |
| T06     | High concurrency (10P/10C) | 19,933 msg/s | 19,934 msg/s | 30.5 ms     | 145.5 ms     | 99%       | 16.5%        | ⭐⭐⭐⭐ Good        |

**Quorum Queue Results:**

| Test ID | Description                | Send Rate    | Recv Rate    | Latency p50 | Latency p99  | CPU Leader | Memory Leader | Performance Rating |
|---------|----------------------------|--------------|--------------|-------------|--------------|-----------|--------------|-------------------|
| T07     | Baseline - no confirms     | 13,194 msg/s | 13,194 msg/s | 318.6 ms    | 1,011 ms     | 99%       | 15.9%        | ⚠️ Unstable       |
| **T08** | **With confirms (PROD)**   | **12,029 msg/s** | **12,029 msg/s** | **6.7 ms** | **23.0 ms** | **99%**   | **16.0%**    | ⭐⭐⭐⭐ Good      |
| T09     | Multi-producer (5P/5C)     | 18,879 msg/s | 18,880 msg/s | 21.9 ms     | 84.0 ms      | 75%       | 15.9%        | ⭐⭐⭐⭐ Good        |
| T10     | Large messages (10KB)      | 9,266 msg/s  | 9,266 msg/s  | 17.7 ms     | 64.5 ms      | 30%       | 15.9%        | ⭐⭐⭐ Fair         |
| T11     | Small messages (128B)      | 21,219 msg/s | 21,219 msg/s | 12.7 ms     | 46.3 ms      | 75%       | 15.9%        | ⭐⭐⭐⭐ Good        |
| **T12** | **High concurrency (10P/10C)** | **21,024 msg/s** | **21,024 msg/s** | **41.7 ms** | **135.1 ms** | **75%**   | **16.0%**    | ⭐⭐⭐⭐ Good   |

### Latency Tests (L-Series) - 6 Scenarios

| Test ID | Queue Type  | Rate Limit    | Send Rate     | Latency p50 | Latency p99 | CPU Leader | Memory Leader | Rating     |
| ------- | ----------- | ------------- | ------------- | ----------- | ----------- | ---------- | ------------- | ---------- |
| **L01** | **Classic** | **500 msg/s** | **500 msg/s** | **0.3 ms**  | **1.0 ms**  | **92%**    | **21.3%**     | ⭐⭐⭐⭐⭐ Best |
| L02     | Classic     | 2K msg/s      | 2,001 msg/s   | 1.0 ms      | 3.8 ms      | 92%        | 21.3%         | ⭐⭐⭐⭐⭐      |
| L03     | Classic     | 5K msg/s      | 5,005 msg/s   | 0.6 ms      | 7.0 ms      | 92%        | 21.3%         | ⭐⭐⭐⭐⭐      |
| L04     | Quorum      | 500 msg/s     | 500 msg/s     | 2.2 ms      | 4.9 ms      | 92%        | 21.3%         | ⭐⭐⭐⭐       |
| L05     | Quorum      | 2K msg/s      | 2,001 msg/s   | 3.2 ms      | 13.0 ms     | 92%        | 21.3%         | ⭐⭐⭐⭐       |
| L06     | Quorum      | 5K msg/s      | 4,999 msg/s   | 3.9 ms      | 24.2 ms     | 92%        | 21.3%         | ⭐⭐⭐⭐       |

### Burst Pattern Tests (B-Series) - 4 Scenarios

| Test ID | Queue Type | Description          | Send Rate    | Latency p50 | Latency p99 | CPU Leader | Memory Leader |
| ------- | ---------- | -------------------- | ------------ | ----------- | ----------- | ---------- | ------------- |
| B01     | Classic    | Burst pattern        | 1,000 msg/s  | 0.3 ms      | 2.1 ms      | 40%        | 15.5%         |
| B02     | Quorum     | Burst pattern        | 1,000 msg/s  | 2.5 ms      | 10.8 ms     | 40%        | 15.5%         |
| B03     | Classic    | Consumer lag (5P/2C) | 21,518 msg/s | 12.2 ms     | 109.1 ms    | 97%        | 15.6%         |
| B04     | Classic    | Producer bottleneck  | 19,681 msg/s | 5.7 ms      | 50.2 ms     | 97%        | 15.6%         |

---

## Resource Usage Deep Dive

### CPU Usage Analysis - All Nodes

**Comprehensive Test (2026-02-03 00:29) - Cluster Load Distribution:**

| Test Phase | rabbitmq-0 (Leader) | rabbitmq-1 (Follower) | rabbitmq-2 (Follower) | Notes                    |
|------------|--------------------|-----------------------|-----------------------|--------------------------|
| T01-T06    | **99%**            | 28%                   | 35%                   | Classic tests - peak load |
| T07-T12    | **75-99%**         | 28%                   | 35%                   | Quorum tests - varied     |
| L01-L06    | **92%**            | 28%                   | 35%                   | Latency tests - sustained |
| B01-B04    | **40-97%**         | 28%                   | 35%                   | Burst tests - spiky       |

**Key Observation:**
- **Leader node (rabbitmq-0)** consistently at **75-99% CPU**
- **Follower nodes** underutilized at **28-35% CPU**
- **Imbalance**: Leader handles ~3x more load than followers
- ➡️ **Bottleneck**: Single-node processing limit pada Classic Queue

### Memory Usage Analysis - Node rabbitmq-0

| Time Period             | Memory Usage | Change   | Test Phase              | Notes                   |
|-------------------------|--------------|----------|-------------------------|-------------------------|
| Baseline (14:40)        | 16.6%        | -        | Idle state              | Initial state           |
| Pre-test cleanup (15:45)| 12.6%        | -24%     | Optimized               | Memory cleared          |
| Test ready (21:20)      | 10.4%        | -18%     | Benchmark ready         | Minimal footprint       |
| **T01-T06 (00:27-00:31)**   | **16.5%**    | **+59%** | **Classic throughput**  | **Stable during test**  |
| **T07-T12 (00:27-00:31)**   | **15.9-16.0%** | **-3%** | **Quorum throughput**   | **Slight decrease**     |
| **L01-L06 (21:48-22:08)**   | **21.3%**    | **+105%** | **Latency tests**      | **Peak memory**         |
| **B01-B04 (00:32-00:51)**   | **15.5-15.6%** | **-27%** | **Burst tests**        | **Memory released**     |
| Post-test spike (21:30) | 20.8%        | +100%    | After standard tests    | Temporary spike         |
| Steady state (07:00+)   | 18.5%        | +78%     | New baseline            | Higher than initial     |

**Memory Spike Pattern:**
```
Test Start → Stable during test → Spike after test → Gradual release → New baseline
```

**Analysis:**
- Latency tests consume **most memory** (21.3%)
- Throughput tests relatively efficient (15.9-16.7%)
- Burst tests **least memory** (15.5-15.6%)
- ➡️ Memory footprint proportional to **message rate consistency**

### Resource Correlation Matrix

| Test Type      | Throughput Range | CPU Usage | Memory Usage | Resource Efficiency |
|----------------|------------------|-----------|--------------|---------------------|
| Classic T-Series| 9,688-25,291 msg/s | 99%       | 16.5-16.7%   | ⭐⭐⭐               |
| Quorum T-Series | 9,266-21,024 msg/s | 30-99%    | 15.9-16.0%   | ⭐⭐⭐⭐              |
| Latency L-Series| 500-5,005 msg/s    | 92%       | 21.3%        | ⭐⭐                 |
| Burst B-Series  | 1,000-21,518 msg/s | 40-97%    | 15.5-15.6%   | ⭐⭐⭐⭐⭐             |

**Efficiency Insights:**
- **Burst tests**: Best resource efficiency (low memory, variable CPU)
- **Quorum tests**: Balanced resource usage across varying loads
- **Latency tests**: High CPU + high memory (rate-limited = queue buildup)
- **Classic tests**: Consistent high CPU, moderate memory

---

## Performance Analysis

### 1. Throughput Comparison

**Production Configs Head-to-Head:**

```
Classic T02:  22,611 msg/s | CPU 99% | Memory 16.5%
Quorum T08:   12,029 msg/s | CPU 99% | Memory 16.0%
──────────────────────────────────────────────────────
Performance:  1.88x faster  | Same CPU | -3% memory
```

**Message Size Impact:**

| Message Size | Classic        | Quorum        | CPU (Classic) | CPU (Quorum) | Memory Delta |
|--------------|----------------|---------------|---------------|--------------|--------------|
| 128B (small) | 25,291 msg/s   | 21,219 msg/s  | 99%           | 75%          | +0.6%        |
| 1KB (std)    | 22,611 msg/s   | 12,029 msg/s  | 99%           | 99%          | -0.5%        |
| 10KB (large) | 9,688 msg/s    | 9,266 msg/s   | 99%           | 30%          | +0.8%        |

**CPU Efficiency Analysis:**
- Classic small messages: **255 msg/s per 1% CPU**
- Quorum small messages: **283 msg/s per 1% CPU** ⭐ More efficient
- Quorum large messages: **309 msg/s per 1% CPU** ⭐ Best efficiency
- ➡️ Quorum more CPU-efficient, but lower absolute throughput

### 2. Latency vs Resource Usage

| Test | Queue  | Latency p50 | Latency p99 | CPU   | Memory | Latency per CPU% |
|------|--------|-------------|-------------|-------|--------|------------------|
| L01  | Classic| 0.3 ms      | 1.0 ms      | 92%   | 21.3%  | 0.0033 ms        |
| L04  | Quorum | 2.2 ms      | 4.9 ms      | 92%   | 21.3%  | 0.0239 ms        |
| T02  | Classic| 4.1 ms      | 15.0 ms     | 99%   | 16.5%  | 0.0414 ms        |
| T08  | Quorum | 6.7 ms      | 23.0 ms     | 99%   | 16.0%  | 0.0677 ms        |

**Latency Overhead per CPU%:**
- Quorum adds **7.3x latency overhead** at same CPU (L-series)
- Quorum adds **1.6x latency overhead** at same CPU (T-series)
- ➡️ Raft consensus cost visible in latency metrics

### 3. Concurrency Resource Impact

**Classic Queue Scaling:**

| Config      | Throughput   | CPU   | Memory | Throughput/CPU | Memory Growth |
|-------------|--------------|-------|--------|----------------|---------------|
| 1P/1C (T02) | 22,611 msg/s | 99%   | 16.5%  | 228 msg/s/%    | Baseline      |
| 5P/5C (T03) | 19,986 msg/s | 99%   | 16.6%  | 202 msg/s/%    | +0.6%         |
| 10P/10C (T06)| 19,933 msg/s| 99%   | 16.5%  | 201 msg/s/%    | 0%            |

**Quorum Queue Scaling:**

| Config      | Throughput   | CPU   | Memory | Throughput/CPU | Memory Growth |
|-------------|--------------|-------|--------|----------------|---------------|
| 1P/1C (T08) | 12,029 msg/s | 99%   | 16.0%  | 122 msg/s/%    | Baseline      |
| 5P/5C (T09) | 18,879 msg/s | 75%   | 15.9%  | 252 msg/s/%    | -0.6%         |
| 10P/10C (T12)| 21,024 msg/s| 75%   | 16.0%  | 280 msg/s/%    | 0%            |

**Scaling Insights:**
- Classic: **Degrades** with concurrency (-11% throughput, same CPU)
- Quorum: **Scales UP** with concurrency (+75% throughput, -24% CPU) ⭐
- Memory: Minimal impact from concurrency (<1% variance)
- ➡️ **Quorum handles multi-producer workloads more efficiently**

---

## Production Recommendations

### 🎯 Queue Selection Decision Tree

```
Production Workload Analysis
│
├─ Throughput > 20K msg/s?
│  ├─ YES → Classic Queue (T02 config)
│  │         CPU: 99% | Memory: 16.5% | Latency: 4.1ms
│  └─ NO → Continue evaluation...
│
├─ Data durability critical?
│  ├─ YES → Quorum Queue (T08 config)
│  │         CPU: 99% | Memory: 16.0% | Latency: 6.7ms
│  └─ NO → Continue evaluation...
│
├─ Latency < 5ms required?
│  ├─ YES → Classic Queue (L01-L03)
│  │         CPU: 92% | Memory: 21.3% | Latency: 0.3-1.0ms
│  └─ NO → Continue evaluation...
│
├─ High concurrency (10+ producers)?
│  ├─ YES → Quorum Queue (T12 config)
│  │         CPU: 75% | Memory: 16.0% | Throughput: 21K msg/s
│  └─ NO → Classic Queue (default)
│
└─ Burst traffic pattern?
   └─ Classic Queue (B01 config)
      CPU: 40% | Memory: 15.5% | Latency: 0.3ms
```

### 📊 Production Configuration Matrix

| Use Case                   | Queue Type | Config | Throughput   | Latency p99 | CPU | Memory | HA  |
| -------------------------- | ---------- | ------ | ------------ | ----------- | --- | ------ | --- |
| **High Throughput**        | Classic    | T02    | 22,611 msg/s | 15.0 ms     | 99% | 16.5%  | ❌   |
| **Financial Transactions** | Quorum     | T08    | 12,029 msg/s | 23.0 ms     | 99% | 16.0%  | ✅   |
| **Real-time Analytics**    | Classic    | L03    | 5,005 msg/s  | 7.0 ms      | 92% | 21.3%  | ❌   |
| **Order Processing**       | Quorum     | T08    | 12,029 msg/s | 23.0 ms     | 99% | 16.0%  | ✅   |
| **Event Streaming**        | Classic    | T05    | 25,291 msg/s | 37.3 ms     | 99% | 16.5%  | ❌   |
| **Microservices Mesh**     | Quorum     | T12    | 21,024 msg/s | 135.1 ms    | 75% | 16.0%  | ✅   |
| **IoT Telemetry**          | Classic    | B01    | Burst        | 2.1 ms      | 40% | 15.5%  | ❌   |
| **Payment Gateway**        | Quorum     | T08    | 12,029 msg/s | 23.0 ms     | 99% | 16.0%  | ✅   |

### ⚡ Resource Optimization Strategies

**CPU Optimization:**

1. **Vertical Scaling Leader Node**
   - Current: Single leader at 99% CPU
   - Recommendation: 2-4 vCPU untuk leader node
   - Expected: Support 40-80K msg/s throughput

2. **Distribute Load to Followers**
   - Current: Followers idle at 28-35% CPU
   - Option: Use Classic Queue sharding across nodes
   - Trade-off: Lose Raft benefits, gain parallelism

3. **Rate Limiting**
   - Use L-series config untuk <5K msg/s workloads
   - CPU drops to 92%, latency drops to <1ms
   - Benefit: Headroom untuk burst traffic

**Memory Optimization:**

1. **Message TTL & Max Length**
   - Current: Memory spikes +100% post-test
   - Recommendation: Set TTL 1-5 minutes, max-length 10K
   - Expected: Memory stable at 12-15%

2. **Lazy Queues**
   - Use untuk low-priority, high-volume queues
   - Offload memory to disk
   - Trade-off: +latency, -memory

3. **Message Size Limits**
   - Large messages (10KB) less efficient
   - Recommendation: Split >5KB messages atau use external storage
   - Benefit: +throughput, -memory pressure

### 🚨 Monitoring Thresholds

**Production Alerts:**

| Metric              | Warning | Critical | Action                           |
|---------------------|---------|----------|----------------------------------|
| CPU Leader          | > 85%   | > 95%    | Scale vertically or shard queues |
| CPU Followers       | > 60%   | > 80%    | Investigate replication lag      |
| Memory Leader       | > 70%   | > 85%    | Enable lazy queues, reduce TTL   |
| Queue Length        | > 50K   | > 100K   | Add consumers, rate limit        |
| Latency p99 Classic | > 50ms  | > 100ms  | Check network, scale resources   |
| Latency p99 Quorum  | > 100ms | > 200ms  | Check Raft lag, network issues   |
| Message Rate        | Varies  | > 25K/s  | Approaching single-node limit    |

---

**Document Version:** 1.0  
**Last Updated:** 2026-02-03  
**Related Documents:**
- [[rabbitmq-benchmark-comparison]]
- [[rabbitmq-queue-types-comparison]]
