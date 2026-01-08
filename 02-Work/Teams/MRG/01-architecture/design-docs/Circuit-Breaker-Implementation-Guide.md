---
tags:
  - architecture
  - circuit-breaker
  - resilience
  - go
  - implementation-guide
  - spof-mitigation
type: guide
created: '2026-01-08'
updated: '2026-01-08'
owner: platform-engineering
---
# Circuit Breaker Implementation Guide

**Purpose**: Panduan implementasi circuit breaker pattern untuk mencegah cascading failures  
**Target Services**: Order Orchestrator, TPG, Payment Processor, Session Manager  
**Library**: `github.com/sony/gobreaker` (recommended)

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [When to Use Circuit Breaker](#when-to-use-circuit-breaker)
3. [Implementation Pattern](#implementation-pattern)
4. [Service-Specific Configurations](#service-specific-configurations)
5. [Monitoring & Alerting](#monitoring--alerting)
6. [Testing Strategy](#testing-strategy)
7. [Rollout Plan](#rollout-plan)

---

## Overview

### What is Circuit Breaker?

Circuit breaker adalah pattern yang mencegah aplikasi terus mencoba operasi yang kemungkinan gagal, memberikan waktu untuk sistem downstream recover.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STATES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐         Failure threshold        ┌──────────┐    │
│   │  CLOSED  │ ────────── exceeded ──────────▶ │   OPEN   │    │
│   │          │                                  │          │    │
│   │ (Normal) │                                  │ (Reject) │    │
│   └────┬─────┘                                  └────┬─────┘    │
│        │                                             │          │
│        │ Success                          Timeout    │          │
│        │                                  expired    │          │
│        │                                             │          │
│        │         ┌─────────────┐                     │          │
│        │         │             │                     │          │
│        └─────────│ HALF-OPEN   │◀────────────────────┘          │
│                  │             │                                 │
│          Success │ (Test mode) │ Failure                        │
│                  │             │                                 │
│                  └──────┬──────┘                                 │
│                         │                                        │
│              Success: Back to CLOSED                             │
│              Failure: Back to OPEN                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why We Need It (From SPOF Assessment)

| Service | Current Issue | Impact Without CB |
|---------|---------------|-------------------|
| Order Orchestrator | No CB for dependencies | Cascading failure to all orders |
| TPG | No CB for BBD | Timeout accumulation, thread exhaustion |
| Payment Processor | No CB for gateways | Slow degradation, stuck payments |
| Session Manager | No CB for pricing | Search/pricing failures cascade |

---

## When to Use Circuit Breaker

### ✅ Use Circuit Breaker For:

1. **External Service Calls**
   - Payment gateways (Midtrans, Xendit, Nicepay)
   - BBD dispatch system
   - Map providers (Google Maps, HERE)
   - SMS/Email providers

2. **Internal Service Calls (Cross-service)**
   - Order Orchestrator → User Service
   - Order Orchestrator → Session Manager
   - TPG → Order Orchestrator callbacks

3. **Database Operations** (with caution)
   - Only for read replicas, not primary
   - For non-critical queries that can be skipped

### ❌ Don't Use Circuit Breaker For:

1. **Critical Path Without Fallback**
   - Primary database writes (order creation)
   - Core business logic

2. **Idempotent Retries Already Handled**
   - Message queue consumers with retry
   - Batch jobs with checkpointing

3. **Local Operations**
   - In-memory calculations
   - Local file operations

---

## Implementation Pattern

### Step 1: Install Library

```bash
go get github.com/sony/gobreaker
```

### Step 2: Create Circuit Breaker Manager

```go
// internal/circuitbreaker/manager.go
package circuitbreaker

import (
    "fmt"
    "sync"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/sony/gobreaker"
)

// Metrics for observability
var (
    circuitBreakerState = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "circuit_breaker_state",
            Help: "Current state of circuit breaker (0=closed, 1=half-open, 2=open)",
        },
        []string{"name"},
    )
    
    circuitBreakerRequests = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "circuit_breaker_requests_total",
            Help: "Total requests through circuit breaker",
        },
        []string{"name", "result"},
    )
)

// Manager holds all circuit breakers
type Manager struct {
    breakers map[string]*gobreaker.CircuitBreaker
    mu       sync.RWMutex
    alerter  Alerter
}

// Alerter interface for sending alerts
type Alerter interface {
    SendAlert(name, message string, severity string)
}

// Config for circuit breaker
type Config struct {
    Name                   string
    MaxRequests            uint32        // Requests allowed in half-open
    Interval               time.Duration // Interval to clear counts
    Timeout                time.Duration // Time to stay open before half-open
    FailureThreshold       uint32        // Consecutive failures to open
    FailureRatioThreshold  float64       // Failure ratio to open (0.0-1.0)
    MinRequests            uint32        // Min requests before ratio check
}

// DefaultConfigs for common services
var DefaultConfigs = map[string]Config{
    "bbd": {
        Name:                  "bbd-dispatch",
        MaxRequests:           3,
        Interval:              60 * time.Second,
        Timeout:               30 * time.Second,
        FailureThreshold:      5,
        FailureRatioThreshold: 0.5,
        MinRequests:           10,
    },
    "payment-midtrans": {
        Name:                  "payment-midtrans",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               15 * time.Second,
        FailureThreshold:      3,
        FailureRatioThreshold: 0.3,
        MinRequests:           10,
    },
    "payment-xendit": {
        Name:                  "payment-xendit",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               15 * time.Second,
        FailureThreshold:      3,
        FailureRatioThreshold: 0.3,
        MinRequests:           10,
    },
    "user-service": {
        Name:                  "user-service",
        MaxRequests:           10,
        Interval:              60 * time.Second,
        Timeout:               10 * time.Second,
        FailureThreshold:      5,
        FailureRatioThreshold: 0.4,
        MinRequests:           20,
    },
    "session-manager": {
        Name:                  "session-manager",
        MaxRequests:           10,
        Interval:              60 * time.Second,
        Timeout:               10 * time.Second,
        FailureThreshold:      5,
        FailureRatioThreshold: 0.4,
        MinRequests:           20,
    },
    "google-maps": {
        Name:                  "google-maps",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               20 * time.Second,
        FailureThreshold:      3,
        FailureRatioThreshold: 0.5,
        MinRequests:           10,
    },
}

// NewManager creates a new circuit breaker manager
func NewManager(alerter Alerter) *Manager {
    return &Manager{
        breakers: make(map[string]*gobreaker.CircuitBreaker),
        alerter:  alerter,
    }
}

// GetOrCreate returns existing or creates new circuit breaker
func (m *Manager) GetOrCreate(cfg Config) *gobreaker.CircuitBreaker {
    m.mu.RLock()
    if cb, ok := m.breakers[cfg.Name]; ok {
        m.mu.RUnlock()
        return cb
    }
    m.mu.RUnlock()

    m.mu.Lock()
    defer m.mu.Unlock()

    // Double-check after acquiring write lock
    if cb, ok := m.breakers[cfg.Name]; ok {
        return cb
    }

    settings := gobreaker.Settings{
        Name:        cfg.Name,
        MaxRequests: cfg.MaxRequests,
        Interval:    cfg.Interval,
        Timeout:     cfg.Timeout,
        ReadyToTrip: m.createTripFunc(cfg),
        OnStateChange: func(name string, from, to gobreaker.State) {
            m.handleStateChange(name, from, to)
        },
        IsSuccessful: func(err error) bool {
            return err == nil || isClientError(err)
        },
    }

    cb := gobreaker.NewCircuitBreaker(settings)
    m.breakers[cfg.Name] = cb
    
    return cb
}

// createTripFunc creates the ReadyToTrip function based on config
func (m *Manager) createTripFunc(cfg Config) func(counts gobreaker.Counts) bool {
    return func(counts gobreaker.Counts) bool {
        // Check consecutive failures
        if counts.ConsecutiveFailures >= cfg.FailureThreshold {
            return true
        }
        
        // Check failure ratio (only if min requests met)
        if counts.Requests >= cfg.MinRequests {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            if failureRatio >= cfg.FailureRatioThreshold {
                return true
            }
        }
        
        return false
    }
}

// handleStateChange handles circuit breaker state transitions
func (m *Manager) handleStateChange(name string, from, to gobreaker.State) {
    // Update metrics
    stateValue := map[gobreaker.State]float64{
        gobreaker.StateClosed:   0,
        gobreaker.StateHalfOpen: 1,
        gobreaker.StateOpen:     2,
    }
    circuitBreakerState.WithLabelValues(name).Set(stateValue[to])
    
    // Log state change
    log.Info("Circuit breaker state changed",
        "name", name,
        "from", from.String(),
        "to", to.String(),
    )
    
    // Alert on open
    if to == gobreaker.StateOpen {
        m.alerter.SendAlert(
            name,
            fmt.Sprintf("Circuit breaker %s is OPEN - service may be down", name),
            "warning",
        )
    }
    
    // Alert on recovery
    if from == gobreaker.StateOpen && to == gobreaker.StateClosed {
        m.alerter.SendAlert(
            name,
            fmt.Sprintf("Circuit breaker %s recovered to CLOSED", name),
            "info",
        )
    }
}

// Execute wraps a function with circuit breaker protection
func (m *Manager) Execute(name string, fn func() (interface{}, error)) (interface{}, error) {
    m.mu.RLock()
    cb, ok := m.breakers[name]
    m.mu.RUnlock()
    
    if !ok {
        return fn() // No circuit breaker configured, execute directly
    }
    
    result, err := cb.Execute(fn)
    
    // Record metrics
    resultLabel := "success"
    if err != nil {
        if err == gobreaker.ErrOpenState {
            resultLabel = "rejected"
        } else if err == gobreaker.ErrTooManyRequests {
            resultLabel = "limited"
        } else {
            resultLabel = "failure"
        }
    }
    circuitBreakerRequests.WithLabelValues(name, resultLabel).Inc()
    
    return result, err
}

// isClientError checks if error is a client error (4xx) - these shouldn't trip CB
func isClientError(err error) bool {
    // Implement based on your error types
    // Client errors (4xx) should not count as failures
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode >= 400 && httpErr.StatusCode < 500
    }
    return false
}
```

### Step 3: Create Service-Specific Wrappers

```go
// internal/clients/bbd_client.go
package clients

import (
    "context"
    "fmt"
    
    "github.com/sony/gobreaker"
    cb "myapp/internal/circuitbreaker"
)

type BBDClient struct {
    httpClient     *http.Client
    baseURL        string
    circuitBreaker *gobreaker.CircuitBreaker
}

func NewBBDClient(baseURL string, cbManager *cb.Manager) *BBDClient {
    return &BBDClient{
        httpClient: &http.Client{
            Timeout: 5 * time.Second,
        },
        baseURL:        baseURL,
        circuitBreaker: cbManager.GetOrCreate(cb.DefaultConfigs["bbd"]),
    }
}

// CreateOrder creates a taxi order with circuit breaker protection
func (c *BBDClient) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    result, err := c.circuitBreaker.Execute(func() (interface{}, error) {
        return c.doCreateOrder(ctx, req)
    })
    
    if err != nil {
        if err == gobreaker.ErrOpenState {
            // Circuit is open - BBD is known to be down
            return nil, &ServiceUnavailableError{
                Service: "BBD",
                Message: "Taxi dispatch service temporarily unavailable",
                Retry:   true,
            }
        }
        return nil, err
    }
    
    return result.(*Order), nil
}

func (c *BBDClient) doCreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // Actual HTTP call to BBD
    resp, err := c.httpClient.Post(
        c.baseURL+"/orders",
        "application/json",
        bytes.NewReader(reqBody),
    )
    // ... handle response
}

// GetVacantVehicles with circuit breaker and fallback
func (c *BBDClient) GetVacantVehicles(ctx context.Context, location Location, radius float64) ([]Vehicle, error) {
    result, err := c.circuitBreaker.Execute(func() (interface{}, error) {
        return c.doGetVacantVehicles(ctx, location, radius)
    })
    
    if err != nil {
        if err == gobreaker.ErrOpenState {
            // Return empty with flag indicating service down
            return nil, &ServiceUnavailableError{
                Service:      "BBD",
                Message:      "Vehicle search temporarily unavailable",
                Retry:        true,
                FallbackUsed: false, // No fallback for vacant vehicles
            }
        }
        return nil, err
    }
    
    return result.([]Vehicle), nil
}
```

### Step 4: Implement Fallback Patterns

```go
// internal/services/order_service.go
package services

type OrderService struct {
    userClient     *UserClient
    paymentClient  *PaymentClient
    cbManager      *circuitbreaker.Manager
    cache          *redis.Client
}

// GetUserWithFallback gets user with circuit breaker and cache fallback
func (s *OrderService) GetUserWithFallback(ctx context.Context, userID string) (*User, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("user:%s", userID)
    if cached, err := s.cache.Get(ctx, cacheKey).Result(); err == nil {
        var user User
        if json.Unmarshal([]byte(cached), &user) == nil {
            return &user, nil
        }
    }
    
    // Try user service with circuit breaker
    result, err := s.cbManager.Execute("user-service", func() (interface{}, error) {
        return s.userClient.GetUser(ctx, userID)
    })
    
    if err != nil {
        if errors.Is(err, gobreaker.ErrOpenState) {
            // Circuit open - try stale cache
            if cached, err := s.cache.Get(ctx, cacheKey+":stale").Result(); err == nil {
                var user User
                if json.Unmarshal([]byte(cached), &user) == nil {
                    user.IsStale = true
                    log.Warn("Using stale user data", "user_id", userID)
                    return &user, nil
                }
            }
            return nil, fmt.Errorf("user service unavailable and no cached data")
        }
        return nil, err
    }
    
    user := result.(*User)
    
    // Cache result
    userJSON, _ := json.Marshal(user)
    s.cache.Set(ctx, cacheKey, userJSON, 5*time.Minute)
    s.cache.Set(ctx, cacheKey+":stale", userJSON, 1*time.Hour) // Stale cache longer
    
    return user, nil
}

// CreatePaymentLinkWithFallback tries multiple gateways
func (s *OrderService) CreatePaymentLinkWithFallback(ctx context.Context, req PaymentRequest) (*PaymentLink, error) {
    gateways := []string{"payment-midtrans", "payment-xendit", "payment-nicepay"}
    
    var lastErr error
    for _, gateway := range gateways {
        result, err := s.cbManager.Execute(gateway, func() (interface{}, error) {
            return s.paymentClient.CreateLink(ctx, gateway, req)
        })
        
        if err == nil {
            return result.(*PaymentLink), nil
        }
        
        lastErr = err
        
        // If circuit open, skip to next immediately
        if errors.Is(err, gobreaker.ErrOpenState) {
            log.Warn("Payment gateway circuit open, trying next",
                "gateway", gateway)
            continue
        }
        
        // If other error, still try next
        log.Warn("Payment gateway failed, trying next",
            "gateway", gateway,
            "error", err)
    }
    
    return nil, fmt.Errorf("all payment gateways failed: %w", lastErr)
}
```

---

## Service-Specific Configurations

### Order Orchestrator

```go
// Dependencies requiring circuit breaker
var orderOrchestratorCBConfigs = map[string]circuitbreaker.Config{
    "user-service": {
        Name:             "user-service",
        MaxRequests:      10,
        Interval:         60 * time.Second,
        Timeout:          10 * time.Second,
        FailureThreshold: 5,
        MinRequests:      20,
    },
    "session-manager": {
        Name:             "session-manager",
        MaxRequests:      10,
        Interval:         60 * time.Second,
        Timeout:          10 * time.Second,
        FailureThreshold: 5,
        MinRequests:      20,
    },
    "payment-processor": {
        Name:             "payment-processor",
        MaxRequests:      5,
        Interval:         30 * time.Second,
        Timeout:          15 * time.Second,
        FailureThreshold: 3,
        MinRequests:      10,
    },
    "taxipartnergateway": {
        Name:             "taxipartnergateway",
        MaxRequests:      5,
        Interval:         60 * time.Second,
        Timeout:          30 * time.Second,
        FailureThreshold: 5,
        MinRequests:      10,
    },
}
```

### Taxi Partner Gateway

```go
// BBD is the critical dependency
var tpgCBConfigs = map[string]circuitbreaker.Config{
    "bbd": {
        Name:                  "bbd-dispatch",
        MaxRequests:           3,           // Very cautious in half-open
        Interval:              60 * time.Second,
        Timeout:               30 * time.Second,  // Stay open longer
        FailureThreshold:      5,
        FailureRatioThreshold: 0.5,
        MinRequests:           10,
    },
    "bbd-tracking": {
        Name:                  "bbd-tracking",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               15 * time.Second,
        FailureThreshold:      3,
        MinRequests:           10,
    },
}
```

### Payment Processor

```go
// Multiple gateways with individual circuit breakers
var paymentCBConfigs = map[string]circuitbreaker.Config{
    "midtrans": {
        Name:                  "payment-midtrans",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               15 * time.Second,
        FailureThreshold:      3,
        FailureRatioThreshold: 0.3,
        MinRequests:           10,
    },
    "xendit": {
        Name:                  "payment-xendit",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               15 * time.Second,
        FailureThreshold:      3,
        FailureRatioThreshold: 0.3,
        MinRequests:           10,
    },
    "nicepay": {
        Name:                  "payment-nicepay",
        MaxRequests:           5,
        Interval:              30 * time.Second,
        Timeout:               15 * time.Second,
        FailureThreshold:      3,
        FailureRatioThreshold: 0.3,
        MinRequests:           10,
    },
}
```

---

## Monitoring & Alerting

### Prometheus Metrics

```go
// Metrics exposed by circuit breaker manager
var (
    // State gauge: 0=closed, 1=half-open, 2=open
    circuit_breaker_state{name="bbd-dispatch"} 
    
    // Request counters
    circuit_breaker_requests_total{name="bbd-dispatch", result="success"}
    circuit_breaker_requests_total{name="bbd-dispatch", result="failure"}
    circuit_breaker_requests_total{name="bbd-dispatch", result="rejected"}
    circuit_breaker_requests_total{name="bbd-dispatch", result="limited"}
)
```

### Grafana Dashboard

```json
{
  "panels": [
    {
      "title": "Circuit Breaker States",
      "type": "stat",
      "targets": [
        {
          "expr": "circuit_breaker_state",
          "legendFormat": "{{name}}"
        }
      ],
      "fieldConfig": {
        "mappings": [
          {"value": 0, "text": "CLOSED", "color": "green"},
          {"value": 1, "text": "HALF-OPEN", "color": "yellow"},
          {"value": 2, "text": "OPEN", "color": "red"}
        ]
      }
    },
    {
      "title": "Request Success Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(circuit_breaker_requests_total{result=\"success\"}[5m]) / rate(circuit_breaker_requests_total[5m])",
          "legendFormat": "{{name}}"
        }
      ]
    },
    {
      "title": "Rejected Requests (Circuit Open)",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(circuit_breaker_requests_total{result=\"rejected\"}[5m])",
          "legendFormat": "{{name}}"
        }
      ]
    }
  ]
}
```

### Alert Rules

```yaml
groups:
  - name: circuit_breaker_alerts
    rules:
      - alert: CircuitBreakerOpen
        expr: circuit_breaker_state == 2
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Circuit breaker {{ $labels.name }} is OPEN"
          description: "Service {{ $labels.name }} may be experiencing issues"
          
      - alert: CircuitBreakerHighRejectionRate
        expr: |
          rate(circuit_breaker_requests_total{result="rejected"}[5m]) 
          / rate(circuit_breaker_requests_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High rejection rate on {{ $labels.name }}"
          description: "More than 10% of requests rejected by circuit breaker"
          
      - alert: CircuitBreakerFlapping
        expr: |
          changes(circuit_breaker_state[10m]) > 4
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Circuit breaker {{ $labels.name }} is flapping"
          description: "Service may be unstable - circuit breaker changing state frequently"
```

---

## Testing Strategy

### Unit Tests

```go
func TestCircuitBreaker_OpensAfterFailures(t *testing.T) {
    manager := circuitbreaker.NewManager(&mockAlerter{})
    cb := manager.GetOrCreate(circuitbreaker.Config{
        Name:             "test",
        FailureThreshold: 3,
        Timeout:          1 * time.Second,
    })
    
    // Simulate 3 failures
    for i := 0; i < 3; i++ {
        _, _ = cb.Execute(func() (interface{}, error) {
            return nil, errors.New("simulated failure")
        })
    }
    
    // Next call should be rejected
    _, err := cb.Execute(func() (interface{}, error) {
        return "should not reach", nil
    })
    
    assert.Equal(t, gobreaker.ErrOpenState, err)
}

func TestCircuitBreaker_RecoverAfterTimeout(t *testing.T) {
    manager := circuitbreaker.NewManager(&mockAlerter{})
    cb := manager.GetOrCreate(circuitbreaker.Config{
        Name:             "test",
        FailureThreshold: 3,
        Timeout:          100 * time.Millisecond, // Short for testing
        MaxRequests:      1,
    })
    
    // Open the circuit
    for i := 0; i < 3; i++ {
        cb.Execute(func() (interface{}, error) {
            return nil, errors.New("failure")
        })
    }
    
    // Wait for timeout
    time.Sleep(150 * time.Millisecond)
    
    // Should be in half-open, allow one request
    result, err := cb.Execute(func() (interface{}, error) {
        return "success", nil
    })
    
    assert.NoError(t, err)
    assert.Equal(t, "success", result)
}
```

### Integration Tests

```go
func TestBBDClient_CircuitBreaker_Integration(t *testing.T) {
    // Start mock BBD server that fails intermittently
    mockBBD := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if rand.Float32() < 0.8 { // 80% failure rate
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    }))
    defer mockBBD.Close()
    
    cbManager := circuitbreaker.NewManager(&mockAlerter{})
    client := NewBBDClient(mockBBD.URL, cbManager)
    
    // Make many requests
    var successCount, rejectedCount int
    for i := 0; i < 100; i++ {
        _, err := client.CreateOrder(context.Background(), CreateOrderRequest{})
        if err == nil {
            successCount++
        } else if errors.Is(err, &ServiceUnavailableError{}) {
            rejectedCount++
        }
        time.Sleep(10 * time.Millisecond)
    }
    
    // Circuit should have opened, preventing many failures
    t.Logf("Success: %d, Rejected: %d", successCount, rejectedCount)
    assert.True(t, rejectedCount > 0, "Circuit breaker should have opened")
}
```

### Chaos Testing

```go
// chaos/circuit_breaker_test.go
func TestChaos_BBDFailure(t *testing.T) {
    if os.Getenv("CHAOS_TEST") != "true" {
        t.Skip("Chaos testing disabled")
    }
    
    // Start the actual TPG service
    // Inject fault into BBD connection
    
    // Verify:
    // 1. Circuit opens after threshold
    // 2. Orders get queued (if queue implemented)
    // 3. Alert fires
    // 4. Recovery after BBD restored
}
```

---

## Rollout Plan

### Phase 1: TPG → BBD (Sprint 1)

**Why First**: Highest risk, no current protection

| Task | Owner | Duration |
|------|-------|----------|
| Implement CB manager | Platform | 2 days |
| Add CB to BBD calls | Platform | 2 days |
| Add manual dispatch queue | Platform | 3 days |
| Unit tests | Platform | 1 day |
| Integration tests | QA | 2 days |
| Staging rollout | Platform | 1 day |
| Production rollout | Platform | 1 day |
| Monitoring setup | Platform | 1 day |

### Phase 2: Order Orchestrator (Sprint 2)

| Task | Owner | Duration |
|------|-------|----------|
| Add CB to User Service calls | Platform | 1 day |
| Add CB to Session Manager calls | Platform | 1 day |
| Add CB to Payment Processor calls | Platform | 1 day |
| Add fallback logic | Platform | 2 days |
| Testing | QA | 2 days |
| Rollout | Platform | 2 days |

### Phase 3: Payment Processor (Sprint 2-3)

| Task | Owner | Duration |
|------|-------|----------|
| Add CB per payment gateway | UPG | 2 days |
| Implement auto-routing on CB open | UPG | 2 days |
| Testing | QA | 2 days |
| Rollout | UPG | 2 days |

### Phase 4: Remaining Services (Sprint 3-4)

- Session Manager → Map providers
- Notification Center → SMS/Email providers
- Config Service → Source systems

---

## 📚 References

- [[MRG SPOF Assessment & Mitigation Strategy]]
- [Sony gobreaker Documentation](https://github.com/sony/gobreaker)
- [Martin Fowler - Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Microsoft - Circuit Breaker Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)

---

**Last Updated**: 2026-01-08  
**Owner**: Platform Engineering Team  
**Review Cycle**: Quarterly
