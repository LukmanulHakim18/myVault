# RabbitMQ Multi-Zone Failover Test - Docker Local Setup

**Environment:** Local Docker  
**Topology:** 3-node cluster simulating 3 zones  
**Test Focus:** Quorum Queue failover scenarios

---

## Directory Structure

```
rabit-mq-banchmark/
├── docker-compose.yml
├── rabbitmq.conf
├── enabled_plugins
├── scripts/
│   ├── setup-cluster.sh
│   ├── test-failover.sh
│   ├── monitor-metrics.sh
│   └── generate-load.go
├── tests/
│   ├── F01-leader-graceful.sh
│   ├── F02-leader-crash.sh
│   ├── F03-follower-failure.sh
│   └── F04-network-partition.sh
└── results/
    └── (test outputs)
```

---

## 1. Docker Compose Configuration

File: `docker-compose-failover.yml`

```yaml
version: '3.8'

services:
  # Zone A - Primary
  rabbitmq-zone-a:
    image: rabbitmq:3.13-management
    container_name: rabbitmq-zone-a
    hostname: rabbitmq-zone-a
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret-cookie-for-cluster'
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_NODENAME: rabbit@rabbitmq-zone-a
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins:ro
      - rabbitmq-zone-a-data:/var/lib/rabbitmq
    ports:
      - "5672:5672"      # AMQP
      - "15672:15672"    # Management UI
      - "15692:15692"    # Prometheus metrics
    networks:
      zone-a:
        ipv4_address: 172.20.1.10
      cluster-network:
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Zone B - Secondary
  rabbitmq-zone-b:
    image: rabbitmq:3.13-management
    container_name: rabbitmq-zone-b
    hostname: rabbitmq-zone-b
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret-cookie-for-cluster'
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_NODENAME: rabbit@rabbitmq-zone-b
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins:ro
      - rabbitmq-zone-b-data:/var/lib/rabbitmq
    ports:
      - "5673:5672"      # AMQP
      - "15673:15672"    # Management UI
      - "15693:15692"    # Prometheus metrics
    networks:
      zone-b:
        ipv4_address: 172.20.2.10
      cluster-network:
    depends_on:
      rabbitmq-zone-a:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Zone C - DR
  rabbitmq-zone-c:
    image: rabbitmq:3.13-management
    container_name: rabbitmq-zone-c
    hostname: rabbitmq-zone-c
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret-cookie-for-cluster'
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_NODENAME: rabbit@rabbitmq-zone-c
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins:ro
      - rabbitmq-zone-c-data:/var/lib/rabbitmq
    ports:
      - "5674:5672"      # AMQP
      - "15674:15672"    # Management UI
      - "15694:15692"    # Prometheus metrics
    networks:
      zone-c:
        ipv4_address: 172.20.3.10
      cluster-network:
    depends_on:
      rabbitmq-zone-a:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Prometheus for metrics collection
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus-failover
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - cluster-network
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=7d'

  # Grafana for visualization
  grafana:
    image: grafana/grafana:latest
    container_name: grafana-failover
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin123
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - cluster-network
    depends_on:
      - prometheus

networks:
  zone-a:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.1.0/24
  zone-b:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.2.0/24
  zone-c:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.3.0/24
  cluster-network:
    driver: bridge

volumes:
  rabbitmq-zone-a-data:
  rabbitmq-zone-b-data:
  rabbitmq-zone-c-data:
  prometheus-data:
  grafana-data:
```

---

## 2. RabbitMQ Configuration

File: `rabbitmq.conf`

```ini
# Cluster configuration
cluster_formation.peer_discovery_backend = classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq-zone-a
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq-zone-b
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq-zone-c

# Network settings
listeners.tcp.default = 5672
management.tcp.port = 15672

# Quorum queue settings
quorum_queue.target_group_size = 3
raft.segment_max_entries = 8192

# Memory & Disk
vm_memory_high_watermark.relative = 0.6
disk_free_limit.absolute = 2GB

# Heartbeat
heartbeat = 60

# Prometheus metrics
prometheus.return_per_object_metrics = true
```

File: `enabled_plugins`

```
[rabbitmq_management,rabbitmq_prometheus,rabbitmq_peer_discovery_common].
```

---

## 3. Prometheus Configuration

File: `prometheus.yml`

```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 5s

scrape_configs:
  - job_name: 'rabbitmq-zone-a'
    static_configs:
      - targets: ['rabbitmq-zone-a:15692']
        labels:
          zone: 'a'
          role: 'leader'

  - job_name: 'rabbitmq-zone-b'
    static_configs:
      - targets: ['rabbitmq-zone-b:15692']
        labels:
          zone: 'b'
          role: 'follower'

  - job_name: 'rabbitmq-zone-c'
    static_configs:
      - targets: ['rabbitmq-zone-c:15692']
        labels:
          zone: 'c'
          role: 'follower'
```

---

## 4. Cluster Setup Script

File: `scripts/setup-cluster.sh`

```bash
#!/bin/bash
set -e

echo "🚀 Starting RabbitMQ Multi-Zone Cluster..."

# Start all containers
docker-compose -f docker-compose-failover.yml up -d

echo "⏳ Waiting for RabbitMQ nodes to be healthy..."
sleep 30

# Join nodes to cluster
echo "🔗 Forming cluster..."

docker exec rabbitmq-zone-b rabbitmqctl stop_app
docker exec rabbitmq-zone-b rabbitmqctl reset
docker exec rabbitmq-zone-b rabbitmqctl join_cluster rabbit@rabbitmq-zone-a
docker exec rabbitmq-zone-b rabbitmqctl start_app

docker exec rabbitmq-zone-c rabbitmqctl stop_app
docker exec rabbitmq-zone-c rabbitmqctl reset
docker exec rabbitmq-zone-c rabbitmqctl join_cluster rabbit@rabbitmq-zone-a
docker exec rabbitmq-zone-c rabbitmqctl start_app

echo "⏳ Waiting for cluster formation..."
sleep 10

# Verify cluster status
echo "✅ Cluster Status:"
docker exec rabbitmq-zone-a rabbitmqctl cluster_status

# Create test quorum queue
echo "📦 Creating test quorum queue..."
docker exec rabbitmq-zone-a rabbitmqadmin declare queue \
  name=test.quorum.failover \
  durable=true \
  arguments='{"x-queue-type":"quorum"}'

# Set policy for all queues
docker exec rabbitmq-zone-a rabbitmqctl set_policy quorum-policy \
  "^test\\.quorum\\." \
  '{"ha-mode":"all"}' \
  --apply-to queues

echo "✅ Setup complete!"
echo "📊 Management UIs:"
echo "   Zone A: http://localhost:15672"
echo "   Zone B: http://localhost:15673"
echo "   Zone C: http://localhost:15674"
echo "   Grafana: http://localhost:3000"
echo "   Prometheus: http://localhost:9090"
```

---

## 5. Failover Test Scripts

### F01: Leader Graceful Shutdown

File: `tests/F01-leader-graceful.sh`

```bash
#!/bin/bash
set -e

TEST_NAME="F01-Leader-Graceful-Shutdown"
RESULT_DIR="results/F01-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RESULT_DIR"

echo "🧪 Starting Test: $TEST_NAME"
echo "📁 Results: $RESULT_DIR"

# Step 1: Identify leader
echo "🔍 Identifying current leader..."
LEADER=$(docker exec rabbitmq-zone-a rabbitmqctl eval \
  'rabbit_quorum_queue:get_leader(<<"/">>, <<"test.quorum.failover">>).')
echo "   Leader: $LEADER" | tee "$RESULT_DIR/test.log"

# Step 2: Start load generator (background)
echo "🚀 Starting load generator..."
go run scripts/generate-load.go \
  --rate 1000 \
  --duration 120 \
  --output "$RESULT_DIR/metrics.csv" &
LOAD_PID=$!

# Wait for baseline
sleep 30

# Step 3: Capture pre-failure metrics
echo "📊 Capturing pre-failure metrics..."
docker exec rabbitmq-zone-a rabbitmqctl list_queues \
  name messages consumers > "$RESULT_DIR/pre-failure.txt"

# Step 4: Execute failure (graceful stop)
echo "⚠️  Stopping leader node gracefully..."
FAILURE_TIME=$(date +%s)
docker stop rabbitmq-zone-a
echo "   Stopped at: $(date)" >> "$RESULT_DIR/test.log"

# Step 5: Monitor leader election
echo "⏳ Monitoring leader election..."
START_ELECTION=$(date +%s)

for i in {1..30}; do
  sleep 1
  NEW_LEADER=$(docker exec rabbitmq-zone-b rabbitmqctl eval \
    'rabbit_quorum_queue:get_leader(<<"/">>, <<"test.quorum.failover">>).' 2>/dev/null || echo "")
  
  if [ ! -z "$NEW_LEADER" ] && [ "$NEW_LEADER" != "$LEADER" ]; then
    END_ELECTION=$(date +%s)
    ELECTION_TIME=$((END_ELECTION - START_ELECTION))
    echo "✅ New leader elected: $NEW_LEADER"
    echo "   Election time: ${ELECTION_TIME}s" | tee -a "$RESULT_DIR/test.log"
    break
  fi
  
  echo "   Waiting for election... ($i/30)"
done

# Step 6: Continue monitoring
echo "📈 Monitoring recovery..."
sleep 60

# Step 7: Capture post-failure metrics
echo "📊 Capturing post-failure metrics..."
docker exec rabbitmq-zone-b rabbitmqctl list_queues \
  name messages consumers > "$RESULT_DIR/post-failure.txt"

# Stop load generator
kill $LOAD_PID 2>/dev/null || true

# Step 8: Analysis
echo "📊 Test Analysis:"
echo "   Leader election time: ${ELECTION_TIME}s"
echo "   Check $RESULT_DIR for detailed metrics"

# Optional: Restart failed node
echo "🔄 Restarting Zone A..."
docker start rabbitmq-zone-a
sleep 10

echo "✅ Test $TEST_NAME completed!"
```

### F02: Leader Hard Kill

File: `tests/F02-leader-crash.sh`

```bash
#!/bin/bash
set -e

TEST_NAME="F02-Leader-Hard-Kill"
RESULT_DIR="results/F02-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RESULT_DIR"

echo "🧪 Starting Test: $TEST_NAME"

# Identify leader
LEADER=$(docker exec rabbitmq-zone-a rabbitmqctl eval \
  'rabbit_quorum_queue:get_leader(<<"/">>, <<"test.quorum.failover">>).')
echo "   Leader: $LEADER" | tee "$RESULT_DIR/test.log"

# Start load
go run scripts/generate-load.go \
  --rate 1000 \
  --duration 120 \
  --output "$RESULT_DIR/metrics.csv" &
LOAD_PID=$!

sleep 30

# Capture pre-failure
docker exec rabbitmq-zone-a rabbitmqctl list_queues \
  name messages > "$RESULT_DIR/pre-failure.txt"

# Execute HARD KILL
echo "💥 HARD KILL: Sending SIGKILL to leader..."
docker kill rabbitmq-zone-a
echo "   Killed at: $(date)" >> "$RESULT_DIR/test.log"

# Monitor election
START_ELECTION=$(date +%s)
for i in {1..60}; do
  sleep 1
  NEW_LEADER=$(docker exec rabbitmq-zone-b rabbitmqctl eval \
    'rabbit_quorum_queue:get_leader(<<"/">>, <<"test.quorum.failover">>).' 2>/dev/null || echo "")
  
  if [ ! -z "$NEW_LEADER" ]; then
    END_ELECTION=$(date +%s)
    ELECTION_TIME=$((END_ELECTION - START_ELECTION))
    echo "✅ New leader: $NEW_LEADER (${ELECTION_TIME}s)"
    echo "   Election time: ${ELECTION_TIME}s" >> "$RESULT_DIR/test.log"
    break
  fi
done

sleep 60

# Capture post-failure
docker exec rabbitmq-zone-b rabbitmqctl list_queues \
  name messages > "$RESULT_DIR/post-failure.txt"

kill $LOAD_PID 2>/dev/null || true

# Restart
docker start rabbitmq-zone-a
sleep 10

echo "✅ Test completed. Results: $RESULT_DIR"
```

### F03: Follower Failure

File: `tests/F03-follower-failure.sh`

```bash
#!/bin/bash
set -e

TEST_NAME="F03-Follower-Failure"
RESULT_DIR="results/F03-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RESULT_DIR"

echo "🧪 Starting Test: $TEST_NAME"

# Start load
go run scripts/generate-load.go \
  --rate 1000 \
  --duration 90 \
  --output "$RESULT_DIR/metrics.csv" &
LOAD_PID=$!

sleep 30

# Stop FOLLOWER (Zone B - typically not leader)
echo "⚠️  Stopping follower (Zone B)..."
docker stop rabbitmq-zone-b
echo "   Stopped at: $(date)" | tee "$RESULT_DIR/test.log"

# Monitor - should NOT trigger leader election
echo "✅ Monitoring operations with 2-node quorum..."
sleep 60

# Check cluster status from Zone A
docker exec rabbitmq-zone-a rabbitmqctl cluster_status > "$RESULT_DIR/cluster-status.txt"

kill $LOAD_PID 2>/dev/null || true

# Restart
docker start rabbitmq-zone-b

echo "✅ Test completed. Results: $RESULT_DIR"
```

### F04: Network Partition

File: `tests/F04-network-partition.sh`

```bash
#!/bin/bash
set -e

TEST_NAME="F04-Network-Partition"
RESULT_DIR="results/F04-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$RESULT_DIR"

echo "🧪 Starting Test: $TEST_NAME"
echo "⚠️  This test requires 'pumba' tool for network chaos"

# Check if pumba installed
if ! command -v pumba &> /dev/null; then
    echo "❌ pumba not found. Install: go install github.com/alexei-led/pumba/cmd/pumba@latest"
    exit 1
fi

# Start load
go run scripts/generate-load.go \
  --rate 1000 \
  --duration 120 \
  --output "$RESULT_DIR/metrics.csv" &
LOAD_PID=$!

sleep 30

# Create network partition: Isolate Zone A
echo "🔥 Creating network partition (isolating Zone A)..."
pumba netem --duration 60s loss --percent 100 rabbitmq-zone-a &
PUMBA_PID=$!

echo "   Partition created at: $(date)" | tee "$RESULT_DIR/test.log"

# Monitor for 60s
sleep 60

# Partition heals automatically after 60s
echo "✅ Partition healed"

# Continue monitoring
sleep 30

kill $LOAD_PID 2>/dev/null || true
kill $PUMBA_PID 2>/dev/null || true

echo "✅ Test completed. Results: $RESULT_DIR"
```

---

## 6. Quick Start Commands

```bash
# 1. Setup environment
chmod +x scripts/*.sh tests/*.sh
./scripts/setup-cluster.sh

# 2. Run individual tests
./tests/F01-leader-graceful.sh
./tests/F02-leader-crash.sh
./tests/F03-follower-failure.sh

# 3. Monitor logs
docker logs -f rabbitmq-zone-a

# 4. Check cluster status
docker exec rabbitmq-zone-a rabbitmqctl cluster_status

# 5. View metrics
# Open: http://localhost:3000 (Grafana)
# Open: http://localhost:15672 (RabbitMQ Management)

# 6. Cleanup
docker-compose -f docker-compose-failover.yml down -v
```

---

## Next Steps

1. Create `generate-load.go` untuk traffic simulation
2. Build monitoring dashboards di Grafana
3. Run baseline tests
4. Execute failover scenarios
5. Analyze results

Mau saya buatkan Go script untuk load generator sekarang?
