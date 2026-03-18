---
title: 'OpenClaw Skill: ELK Log Analyzer'
type: skill-spec
version: 1.0.0
status: draft
tags:
  - openclaw
  - skill
  - elasticsearch
  - elk
  - logging
---
# OpenClaw Skills: ELK Log Analyzer Skill

## Metadata

| Field | Value |
|-------|-------|
| Skill Name | `elk-log-analyzer` |
| Version | 1.0.0 |
| Author | MRG Architecture Team |
| Dependencies | Elasticsearch 8.x |
| Based On | mcpmrg (existing tool) |

---

## Description

Skill untuk query dan analyze logs dari Elasticsearch cluster. Optimized untuk distributed tracing di microservices environment.

Skill ini dapat menggunakan `mcpmrg` yang sudah ada sebagai backend implementation.

---

## Capabilities

1. **Query by Trace ID** - Retrieve semua logs dengan trace_id yang sama
2. **Query by Order ID** - Retrieve logs terkait order tertentu
3. **Extract Errors** - Filter dan extract error messages
4. **Build Timeline** - Construct chronological timeline of events
5. **Identify Patterns** - Detect recurring error patterns

---

## Required Permissions

```yaml
permissions:
  - elasticsearch:read
  - filesystem:write  # untuk temporary files
```

---

## Configuration

### Environment Variables

```bash
# Required
ES_HOSTS=https://elasticsearch.bluebird.id:9200
ES_USERNAME=openclaw-reader
ES_PASSWORD=xxxxxxxxxxxx

# Optional
ES_CA_CERT=/path/to/ca.crt
ES_INDEX_PATTERN=mrg-*,upg-*
```

### config.yaml

```yaml
name: elk-log-analyzer
version: 1.0.0

environment:
  ES_HOSTS: ${ES_HOSTS}
  ES_USERNAME: ${ES_USERNAME}
  ES_PASSWORD: ${ES_PASSWORD}

settings:
  max_results: 1000
  default_time_range: "-2h"
  timezone: "Asia/Jakarta"
  scroll_timeout: "2m"

index_mapping:
  mrg:
    order-orchestrator: "mrg-order-orchestrator-*"
    session-manager: "mrg-session-manager-*"
    auth-service: "mrg-auth-service-*"
    notification-center: "mrg-notification-center-*"
    taxi-partner-gateway: "mrg-taxi-partner-gateway-*"
    
  upg:
    payment-service: "upg-payment-service-*"
    wallet-service: "upg-wallet-service-*"
    fds-service: "upg-fds-service-*"

field_mapping:
  trace_id: "trace_id"
  order_id: "order_id"
  timestamp: "@timestamp"
  level: "level"
  message: "message"
  service: "service.name"
  function: "function"
  error: "error.message"
  stack_trace: "error.stack_trace"
```

---

## Query Templates

### 1. Query by Trace ID

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "trace_id": "{{trace_id}}" } }
      ],
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "{{start_time}}",
              "lte": "{{end_time}}"
            }
          }
        }
      ]
    }
  },
  "sort": [
    { "@timestamp": "asc" }
  ],
  "size": 1000,
  "_source": [
    "@timestamp",
    "trace_id",
    "service.name",
    "function",
    "level",
    "message",
    "error",
    "duration_ms"
  ]
}
```

### 2. Query by Order ID

```json
{
  "query": {
    "bool": {
      "should": [
        { "match": { "order_id": "{{order_id}}" } },
        { "match": { "booking_id": "{{order_id}}" } },
        { "match": { "message": "{{order_id}}" } }
      ],
      "minimum_should_match": 1,
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "{{start_time}}",
              "lte": "{{end_time}}"
            }
          }
        }
      ]
    }
  },
  "sort": [
    { "@timestamp": "asc" }
  ],
  "size": 1000
}
```

### 3. Extract Errors Only

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "trace_id": "{{trace_id}}" } },
        {
          "bool": {
            "should": [
              { "match": { "level": "error" } },
              { "match": { "level": "fatal" } },
              { "exists": { "field": "error.message" } }
            ]
          }
        }
      ]
    }
  },
  "sort": [
    { "@timestamp": "asc" }
  ]
}
```

### 4. Service Hop Analysis

```json
{
  "query": {
    "match": { "trace_id": "{{trace_id}}" }
  },
  "aggs": {
    "services": {
      "terms": {
        "field": "service.name",
        "size": 20
      },
      "aggs": {
        "functions": {
          "terms": {
            "field": "function",
            "size": 50
          },
          "aggs": {
            "avg_duration": {
              "avg": { "field": "duration_ms" }
            },
            "errors": {
              "filter": {
                "term": { "level": "error" }
              }
            }
          }
        }
      }
    }
  },
  "size": 0
}
```

---

## Usage Examples

### 1. Basic Trace Lookup

**Prompt:**
```
Get all logs for trace_id "a1b2c3d4-e5f6-7890-abcd-ef1234567890" 
from the last 2 hours.
```

**Skill Action:**
```yaml
action: query_by_trace
trace_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
time_range: "-2h"
```

### 2. Error Extraction

**Prompt:**
```
Find all errors in trace "a1b2c3d4-e5f6-..." and identify the root cause.
```

**Skill Action:**
```yaml
action: extract_errors
trace_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
include:
  - error_message
  - stack_trace
  - preceding_logs: 5  # 5 logs before each error
```

### 3. Build Timeline

**Prompt:**
```
Create a timeline of events for order ORD-ABC123.
```

**Skill Action:**
```yaml
action: build_timeline
order_id: "ORD-ABC123"
granularity: "function"  # or "service"
include_duration: true
```

---

## Output Formats

### Timeline Format

```yaml
timeline:
  trace_id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  order_id: "ORD-ABC123"
  start_time: "2026-03-18T10:15:30.000+07:00"
  end_time: "2026-03-18T10:15:35.500+07:00"
  total_duration_ms: 5500
  
  events:
    - timestamp: "2026-03-18T10:15:30.000+07:00"
      service: order-orchestrator
      function: CreateOrder
      level: info
      message: "Creating order for user U123"
      duration_ms: 45
      
    - timestamp: "2026-03-18T10:15:30.100+07:00"
      service: session-manager
      function: AssignFleet
      level: info
      message: "Assigning fleet for zone Z001"
      duration_ms: 200
      
    - timestamp: "2026-03-18T10:15:31.500+07:00"
      service: payment-service
      function: ProcessPayment
      level: error
      message: "Payment failed: insufficient balance"
      duration_ms: 1200
      error:
        type: "InsufficientBalanceError"
        code: "PAY_ERR_001"
```

### Error Summary Format

```yaml
error_summary:
  total_errors: 1
  first_error_at: "2026-03-18T10:15:31.500+07:00"
  
  root_cause:
    service: payment-service
    function: ProcessPayment
    file: "internal/handler/payment.go"
    line: 234
    error_type: "InsufficientBalanceError"
    message: "Payment failed: insufficient balance"
    
  stack_trace: |
    github.com/bluebird/payment-service/internal/handler.ProcessPayment
        /app/internal/handler/payment.go:234
    github.com/bluebird/payment-service/internal/service.(*PaymentService).Execute
        /app/internal/service/payment.go:89
    github.com/bluebird/payment-service/internal/client.(*WalletClient).CheckBalance
        /app/internal/client/wallet.go:156
        
  preceding_events:
    - timestamp: "2026-03-18T10:15:31.000+07:00"
      service: payment-service
      message: "Checking user balance for amount 150000"
      
    - timestamp: "2026-03-18T10:15:31.200+07:00"
      service: wallet-service
      message: "Balance check: user=U123, available=100000, required=150000"
```

### Service Hop Summary

```yaml
service_hops:
  total_services: 4
  total_functions: 12
  
  services:
    - name: order-orchestrator
      functions_called: 3
      total_duration_ms: 150
      errors: 0
      
    - name: session-manager
      functions_called: 4
      total_duration_ms: 800
      errors: 0
      
    - name: payment-service
      functions_called: 3
      total_duration_ms: 1500
      errors: 1
      
    - name: wallet-service
      functions_called: 2
      total_duration_ms: 300
      errors: 0
```

---

## Integration with mcpmrg

Jika menggunakan mcpmrg sebagai backend:

```yaml
# Option A: Direct integration
implementation:
  type: mcpmrg
  binary: /usr/local/bin/mcpmrg
  
# Option B: Wrapper script
implementation:
  type: script
  command: |
    mcpmrg query \
      --trace-id={{trace_id}} \
      --time-range={{time_range}} \
      --format=json \
      --output=/tmp/openclaw/logs-{{trace_id}}.json
```

### mcpmrg Command Examples

```bash
# Query by trace ID
mcpmrg query --trace-id=a1b2c3d4-e5f6-7890-abcd-ef1234567890 --format=json

# Query by order ID
mcpmrg query --order-id=ORD-ABC123 --time-range="-2h" --format=json

# Extract errors only
mcpmrg query --trace-id=xxx --level=error --format=json

# Build timeline
mcpmrg timeline --trace-id=xxx --granularity=function
```

---

## Error Handling

```yaml
errors:
  ES_CONNECTION_FAILED:
    message: "Cannot connect to Elasticsearch"
    action: retry
    max_retries: 3
    backoff: exponential
    
  ES_TIMEOUT:
    message: "Elasticsearch query timeout"
    action: reduce_scope
    fallback:
      reduce_time_range: true
      reduce_max_results: true
      
  NO_LOGS_FOUND:
    message: "No logs found for given trace/order ID"
    action: expand_search
    fallback:
      expand_time_range: "+1h"
      try_alternate_fields: true
      
  TOO_MANY_RESULTS:
    message: "Query returned too many results"
    action: paginate
    page_size: 500
```

---

## Performance Considerations

1. **Index Pattern:** Gunakan specific index pattern, bukan wildcard yang terlalu broad
2. **Time Range:** Default -2h, jangan terlalu lebar tanpa alasan
3. **Field Selection:** Hanya retrieve fields yang diperlukan
4. **Pagination:** Gunakan scroll API untuk large result sets
5. **Caching:** Cache query results untuk repeated lookups

---

## Security Notes

1. **Read-Only Access:** Skill HANYA memiliki read permission ke ES
2. **Credential Storage:** Gunakan environment variables atau secret manager
3. **Query Sanitization:** Validate dan sanitize all input parameters
4. **Audit Trail:** Log all queries untuk compliance

---

## Related Files

- [[RFC-001-openclaw-incident-handler|Main RFC]]
- [[SKILL-gitlab-issue|GitLab Issue Skill]]
- [[SKILL-code-analyzer|Code Analyzer Skill]]
