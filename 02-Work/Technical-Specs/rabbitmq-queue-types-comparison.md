---
created: '2026-01-30'
type: technical-reference
tags:
  - rabbitmq
  - messaging
  - infrastructure
  - benchmark
status: active
---
# RabbitMQ Queue Types: Quorum vs Classic

**Created:** 2026-01-30  
**Tags:** #rabbitmq #messaging #infrastructure #benchmark

---

## Overview

Perbandingan antara Classic Queue dan Quorum Queue di RabbitMQ untuk keperluan assessment infrastructure.

---

## Benchmark Summary

### Throughput

| Metric | Classic Queue | Quorum Queue |
|--------|---------------|--------------|
| Single queue (1KB msg) | ~10,000 msg/s | **~30,000 msg/s** |
| Sending rate (raw) | Higher | Slightly lower |
| Receiving rate | Higher | Lower (replication overhead) |
| Long backlog handling | Degraded | **Optimized (3.12+)** |

### Resource Usage

| Resource | Classic | Quorum |
|----------|---------|--------|
| RAM (idle, ~2000 queues) | Baseline | **+50-100%** |
| CPU | Baseline | **+~2x** |
| Disk I/O | Lower | **Higher** (all writes to disk) |
| Memory per message | Lower | **~32 bytes/msg** minimum |

### Latency

| Scenario | Classic | Quorum |
|----------|---------|--------|
| 99th percentile | Lower | Higher (~23ms) |
| Publish confirm | Faster | Slower (wait majority) |

---

## Data Safety & Availability

| Aspect | Classic | Quorum |
|--------|---------|--------|
| Replication | Async (bisa loss) | **Sync (Raft consensus)** |
| Failover | Manual sync, blocking | **Automatic, non-blocking** |
| Split-brain risk | **Ada** | Minimal |
| Poison message handling | Tidak ada | **Built-in (delivery limit)** |
| Single point of failure | **Ya** (single node) | **Tidak** (distributed) |

---

## Kapan Pakai Apa

### Classic Queue
- High throughput, low-priority tasks
- Transient/temporary messages
- Resource-constrained environment
- Non-critical data yang bisa di-replay

### Quorum Queue
- **Payment processing** (UPG)
- **Order orchestration** (MRG)
- Mission-critical messages
- Multi-node cluster dengan HA requirement
- Data yang tidak boleh hilang

---

## Key Takeaways

1. **Quorum lebih cepat** dari Classic Mirrored (dengan publisher confirms)
2. **Resource usage lebih tinggi** (~50-100% RAM, 2x CPU)
3. **Latency trade-off** karena consensus mechanism
4. **Message size matters** - throughput turun seiring size naik
5. **Cluster size matters** - lebih banyak node = throughput lebih rendah

---

## Recommendation untuk Bluebird

Untuk sistem **MRG** dan **UPG** yang handle order dan payment:
- **Gunakan Quorum Queue** untuk critical path
- Classic Queue boleh untuk logging/notification yang non-critical
- Minimum 3 nodes untuk Quorum Queue

---

## References

- [RabbitMQ Official - Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [RabbitMQ 3.12 Performance Improvements](https://www.rabbitmq.com/blog/2023/05/17/rabbitmq-3.12-performance-improvements)
- [CloudAMQP - Queue Types](https://www.cloudamqp.com/blog/rabbitmq-queue-types.html)
- [Quorum Queue Migration Guide](https://www.rabbitmq.com/blog/2023/03/02/quorum-queues-migration)
