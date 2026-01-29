---
tags:
  - trading
  - testing
  - qa
  - automation
  - ci-cd
created: '2026-01-21'
---
# Testing Strategy

## 1. Overview

```mermaid
flowchart TB
    subgraph PYRAMID["🔺 Testing Pyramid"]
        direction TB
        E2E["🔝 E2E Tests<br/>~10%"]
        INTEG["🔷 Integration Tests<br/>~20%"]
        UNIT["🔷 Unit Tests<br/>~70%"]
    end
    
    E2E --> INTEG --> UNIT
```

| Level | Coverage Target | Owner | Tools |
|-------|-----------------|-------|-------|
| Unit | 80%+ | BE/FE Agent | Go test, Jest |
| Integration | 70%+ | QA Agent | Go test, Supertest |
| E2E | Critical paths | QA Agent | Playwright Test |

---

## 2. Test Categories

```mermaid
mindmap
  root((Testing))
    Unit Tests
      Robot Engine
      Owner Server
      Dashboard Components
    Integration Tests
      API Endpoints
      Database
      WebSocket
    E2E Tests
      User Flows
      Real-time Updates
    Performance Tests
      Load Testing
      Stress Testing
    Security Tests
      Authentication
      Authorization
```

---

## 3. Unit Testing

### 3.1 Backend (Go)

#### Test Structure

```
robot-engine/
├── internal/
│   ├── engine/
│   │   ├── engine.go
│   │   └── engine_test.go
│   ├── worker/
│   │   ├── worker.go
│   │   └── worker_test.go
│   ├── state/
│   │   ├── state.go
│   │   └── state_test.go
│   └── adapter/
│       ├── stockbit/
│       │   ├── adapter.go
│       │   └── adapter_test.go
│       └── ipot/
│           ├── adapter.go
│           └── adapter_test.go

owner-server/
├── internal/
│   ├── handler/
│   │   ├── task_handler.go
│   │   └── task_handler_test.go
│   ├── service/
│   │   ├── task_service.go
│   │   └── task_service_test.go
│   └── repository/
│       ├── task_repo.go
│       └── task_repo_test.go
```

#### Example: State Machine Test

```go
// state_test.go
package state

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestOrderStateMachine_ValidTransitions(t *testing.T) {
    tests := []struct {
        name     string
        from     OrderState
        to       OrderState
        expected bool
    }{
        {"pending to submitting", StatePending, StateSubmitting, true},
        {"submitting to submitted", StateSubmitting, StateSubmitted, true},
        {"submitted to filled", StateSubmitted, StateFilled, true},
        {"filled to monitoring", StateFilled, StateMonitoring, true},
        {"monitoring to tp_hit", StateMonitoring, StateTPHit, true},
        {"monitoring to cl_hit", StateMonitoring, StateCLHit, true},
        
        // Invalid transitions
        {"pending to filled", StatePending, StateFilled, false},
        {"tp_hit to monitoring", StateTPHit, StateMonitoring, false},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            sm := NewOrderStateMachine(tt.from)
            err := sm.Transition(tt.to)
            
            if tt.expected {
                assert.NoError(t, err)
                assert.Equal(t, tt.to, sm.Current())
            } else {
                assert.Error(t, err)
                assert.Equal(t, tt.from, sm.Current())
            }
        })
    }
}

func TestOrderStateMachine_StateHistory(t *testing.T) {
    sm := NewOrderStateMachine(StatePending)
    
    sm.Transition(StateSubmitting)
    sm.Transition(StateSubmitted)
    sm.Transition(StateFilled)
    
    history := sm.History()
    
    assert.Len(t, history, 4)
    assert.Equal(t, StatePending, history[0].State)
    assert.Equal(t, StateFilled, history[3].State)
}
```

#### Example: Retry Logic Test

```go
// retry_test.go
package engine

import (
    "errors"
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestRetryPolicy_MaxRetries(t *testing.T) {
    policy := NewRetryPolicy(3, time.Second)
    
    attempts := 0
    err := policy.Execute(func() error {
        attempts++
        return errors.New("always fails")
    })
    
    assert.Error(t, err)
    assert.Equal(t, 3, attempts)
}

func TestRetryPolicy_SuccessOnSecondAttempt(t *testing.T) {
    policy := NewRetryPolicy(3, time.Millisecond)
    
    attempts := 0
    err := policy.Execute(func() error {
        attempts++
        if attempts < 2 {
            return errors.New("fail first time")
        }
        return nil
    })
    
    assert.NoError(t, err)
    assert.Equal(t, 2, attempts)
}

func TestRetryPolicy_ExponentialBackoff(t *testing.T) {
    policy := NewRetryPolicy(3, 100*time.Millisecond)
    policy.SetBackoffMultiplier(2.0)
    
    var delays []time.Duration
    startTime := time.Now()
    
    policy.Execute(func() error {
        delays = append(delays, time.Since(startTime))
        startTime = time.Now()
        return errors.New("fail")
    })
    
    // First attempt: immediate
    // Second attempt: ~100ms
    // Third attempt: ~200ms
    assert.True(t, delays[1] >= 90*time.Millisecond)
    assert.True(t, delays[2] >= 180*time.Millisecond)
}
```

#### Example: Element Finder Test (Mock)

```go
// element_finder_test.go
package worker

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

type MockPage struct {
    mock.Mock
}

func (m *MockPage) WaitForSelector(selector string, opts interface{}) (interface{}, error) {
    args := m.Called(selector, opts)
    return args.Get(0), args.Error(1)
}

func TestElementFinder_CSSPrimary(t *testing.T) {
    mockPage := new(MockPage)
    mockElement := &MockElement{}
    
    selector := Selector{
        CSS:   "#submit-btn",
        XPath: "//button[@type='submit']",
    }
    
    // CSS selector succeeds
    mockPage.On("WaitForSelector", "#submit-btn", mock.Anything).
        Return(mockElement, nil)
    
    finder := NewElementFinder(mockPage)
    element, err := finder.Find(selector)
    
    assert.NoError(t, err)
    assert.NotNil(t, element)
    
    // Verify CSS was tried, XPath was not
    mockPage.AssertCalled(t, "WaitForSelector", "#submit-btn", mock.Anything)
    mockPage.AssertNotCalled(t, "WaitForSelector", "//button[@type='submit']", mock.Anything)
}

func TestElementFinder_FallbackToXPath(t *testing.T) {
    mockPage := new(MockPage)
    mockElement := &MockElement{}
    
    selector := Selector{
        CSS:   "#submit-btn",
        XPath: "//button[@type='submit']",
    }
    
    // CSS selector fails
    mockPage.On("WaitForSelector", "#submit-btn", mock.Anything).
        Return(nil, errors.New("not found"))
    
    // XPath succeeds
    mockPage.On("WaitForSelector", "//button[@type='submit']", mock.Anything).
        Return(mockElement, nil)
    
    finder := NewElementFinder(mockPage)
    element, err := finder.Find(selector)
    
    assert.NoError(t, err)
    assert.NotNil(t, element)
}
```

### 3.2 Frontend (React/Next.js)

#### Test Structure

```
dashboard/
├── components/
│   ├── StatusBadge/
│   │   ├── StatusBadge.tsx
│   │   └── StatusBadge.test.tsx
│   ├── StatCard/
│   │   ├── StatCard.tsx
│   │   └── StatCard.test.tsx
│   └── OrderTable/
│       ├── OrderTable.tsx
│       └── OrderTable.test.tsx
├── hooks/
│   ├── useWebSocket.ts
│   └── useWebSocket.test.ts
└── utils/
    ├── formatters.ts
    └── formatters.test.ts
```

#### Example: Component Test

```typescript
// StatusBadge.test.tsx
import { render, screen } from '@testing-library/react';
import { StatusBadge } from './StatusBadge';

describe('StatusBadge', () => {
  it('renders online status correctly', () => {
    render(<StatusBadge status="online" />);
    
    expect(screen.getByText('Online')).toBeInTheDocument();
    expect(screen.getByTestId('status-badge')).toHaveClass('bg-green-500');
  });
  
  it('renders offline status correctly', () => {
    render(<StatusBadge status="offline" />);
    
    expect(screen.getByText('Offline')).toBeInTheDocument();
    expect(screen.getByTestId('status-badge')).toHaveClass('bg-red-500');
  });
  
  it('renders warning status correctly', () => {
    render(<StatusBadge status="warning" />);
    
    expect(screen.getByText('Warning')).toBeInTheDocument();
    expect(screen.getByTestId('status-badge')).toHaveClass('bg-yellow-500');
  });
});
```

#### Example: Hook Test

```typescript
// useWebSocket.test.ts
import { renderHook, act } from '@testing-library/react';
import { useWebSocket } from './useWebSocket';
import WS from 'jest-websocket-mock';

describe('useWebSocket', () => {
  let server: WS;
  
  beforeEach(() => {
    server = new WS('ws://localhost:8080/ws');
  });
  
  afterEach(() => {
    WS.clean();
  });
  
  it('connects to WebSocket server', async () => {
    const { result } = renderHook(() => useWebSocket('ws://localhost:8080/ws'));
    
    await server.connected;
    
    expect(result.current.isConnected).toBe(true);
  });
  
  it('receives and processes messages', async () => {
    const onMessage = jest.fn();
    const { result } = renderHook(() => 
      useWebSocket('ws://localhost:8080/ws', { onMessage })
    );
    
    await server.connected;
    
    act(() => {
      server.send(JSON.stringify({
        event: 'order_updated',
        data: { order_id: 'ord_001', status: 'tp_hit' }
      }));
    });
    
    expect(onMessage).toHaveBeenCalledWith({
      event: 'order_updated',
      data: { order_id: 'ord_001', status: 'tp_hit' }
    });
  });
  
  it('reconnects on disconnect', async () => {
    const { result } = renderHook(() => 
      useWebSocket('ws://localhost:8080/ws', { reconnect: true })
    );
    
    await server.connected;
    server.close();
    
    // Wait for reconnect
    await new Promise(resolve => setTimeout(resolve, 1100));
    
    expect(result.current.isConnected).toBe(true);
  });
});
```

---

## 4. Integration Testing

### 4.1 API Integration Tests

```go
// task_handler_integration_test.go
package handler_test

import (
    "net/http"
    "net/http/httptest"
    "testing"
    "encoding/json"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
)

type TaskHandlerSuite struct {
    suite.Suite
    server *httptest.Server
    db     *sql.DB
}

func (s *TaskHandlerSuite) SetupSuite() {
    // Setup test database
    s.db = setupTestDB()
    
    // Setup test server
    app := setupTestApp(s.db)
    s.server = httptest.NewServer(app.Handler())
}

func (s *TaskHandlerSuite) TearDownSuite() {
    s.server.Close()
    s.db.Close()
}

func (s *TaskHandlerSuite) SetupTest() {
    // Clean database before each test
    cleanDB(s.db)
    seedTestData(s.db)
}

func (s *TaskHandlerSuite) TestCreateTask_Success() {
    payload := map[string]interface{}{
        "account_id": "ACC_001",
        "emiten":     "BBCA",
        "action":     "BUY",
        "price":      9250,
        "lot":        10,
        "tp_price":   9500,
        "cl_price":   9100,
    }
    
    resp, err := s.postJSON("/api/v1/tasks", payload)
    
    s.NoError(err)
    s.Equal(http.StatusCreated, resp.StatusCode)
    
    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    
    s.True(result["success"].(bool))
    s.NotEmpty(result["data"].(map[string]interface{})["id"])
}

func (s *TaskHandlerSuite) TestCreateTask_InvalidAccount() {
    payload := map[string]interface{}{
        "account_id": "INVALID",
        "emiten":     "BBCA",
        "action":     "BUY",
        "price":      9250,
        "lot":        10,
    }
    
    resp, err := s.postJSON("/api/v1/tasks", payload)
    
    s.NoError(err)
    s.Equal(http.StatusBadRequest, resp.StatusCode)
}

func (s *TaskHandlerSuite) TestListTasks_WithFilters() {
    resp, err := http.Get(s.server.URL + "/api/v1/tasks?account=ACC_001&status=pending")
    
    s.NoError(err)
    s.Equal(http.StatusOK, resp.StatusCode)
    
    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    
    tasks := result["data"].(map[string]interface{})["tasks"].([]interface{})
    for _, t := range tasks {
        task := t.(map[string]interface{})
        s.Equal("ACC_001", task["account_id"])
        s.Equal("pending", task["status"])
    }
}

func TestTaskHandlerSuite(t *testing.T) {
    suite.Run(t, new(TaskHandlerSuite))
}
```

### 4.2 Database Integration Tests

```go
// task_repo_integration_test.go
package repository_test

import (
    "testing"
    "time"
    
    "github.com/stretchr/testify/suite"
)

type TaskRepoSuite struct {
    suite.Suite
    db   *sql.DB
    repo *TaskRepository
}

func (s *TaskRepoSuite) SetupSuite() {
    s.db = setupTestDB()
    s.repo = NewTaskRepository(s.db)
}

func (s *TaskRepoSuite) SetupTest() {
    cleanDB(s.db)
}

func (s *TaskRepoSuite) TestCreate_And_Find() {
    task := &Task{
        AccountID: "ACC_001",
        Emiten:    "BBCA",
        Action:    "BUY",
        Price:     9250,
        Lot:       10,
        TPPrice:   9500,
        CLPrice:   9100,
        Status:    "pending",
    }
    
    // Create
    err := s.repo.Create(task)
    s.NoError(err)
    s.NotEmpty(task.ID)
    
    // Find
    found, err := s.repo.FindByID(task.ID)
    s.NoError(err)
    s.Equal(task.Emiten, found.Emiten)
    s.Equal(task.Price, found.Price)
}

func (s *TaskRepoSuite) TestFindByDateAndAccount() {
    // Create test data
    s.createTask("ACC_001", "BBCA", "pending", time.Now())
    s.createTask("ACC_001", "TLKM", "completed", time.Now())
    s.createTask("ACC_002", "ASII", "pending", time.Now())
    s.createTask("ACC_001", "BMRI", "pending", time.Now().AddDate(0, 0, -1))
    
    // Query
    tasks, err := s.repo.FindByFilters(TaskFilter{
        AccountID: "ACC_001",
        Date:      time.Now().Format("2006-01-02"),
        Status:    "pending",
    })
    
    s.NoError(err)
    s.Len(tasks, 1)
    s.Equal("BBCA", tasks[0].Emiten)
}

func TestTaskRepoSuite(t *testing.T) {
    suite.Run(t, new(TaskRepoSuite))
}
```

### 4.3 WebSocket Integration Tests

```go
// websocket_integration_test.go
package handler_test

import (
    "testing"
    "time"
    
    "github.com/gorilla/websocket"
    "github.com/stretchr/testify/suite"
)

type WebSocketSuite struct {
    suite.Suite
    server *httptest.Server
    ws     *websocket.Conn
}

func (s *WebSocketSuite) SetupSuite() {
    app := setupTestApp()
    s.server = httptest.NewServer(app.Handler())
}

func (s *WebSocketSuite) SetupTest() {
    wsURL := "ws" + s.server.URL[4:] + "/ws?token=" + getTestToken()
    
    ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
    s.NoError(err)
    s.ws = ws
}

func (s *WebSocketSuite) TearDownTest() {
    s.ws.Close()
}

func (s *WebSocketSuite) TestSubscription() {
    // Subscribe
    s.ws.WriteJSON(map[string]interface{}{
        "type":     "subscribe",
        "channels": []string{"orders"},
    })
    
    // Read response
    var resp map[string]interface{}
    s.ws.ReadJSON(&resp)
    
    s.Equal("subscribed", resp["type"])
}

func (s *WebSocketSuite) TestOrderUpdateEvent() {
    // Subscribe to orders
    s.ws.WriteJSON(map[string]interface{}{
        "type":     "subscribe",
        "channels": []string{"orders"},
    })
    s.ws.ReadJSON(&map[string]interface{}{}) // consume subscription response
    
    // Trigger order update via API
    go func() {
        time.Sleep(100 * time.Millisecond)
        triggerOrderUpdate("ord_001", "tp_hit")
    }()
    
    // Wait for WebSocket event
    s.ws.SetReadDeadline(time.Now().Add(2 * time.Second))
    
    var event map[string]interface{}
    err := s.ws.ReadJSON(&event)
    
    s.NoError(err)
    s.Equal("order_updated", event["event"])
    s.Equal("tp_hit", event["data"].(map[string]interface{})["new_state"])
}

func TestWebSocketSuite(t *testing.T) {
    suite.Run(t, new(WebSocketSuite))
}
```

---

## 5. E2E Testing

### 5.1 Test Structure

```
e2e/
├── tests/
│   ├── dashboard.spec.ts
│   ├── accounts.spec.ts
│   ├── orders.spec.ts
│   └── reports.spec.ts
├── fixtures/
│   ├── accounts.json
│   └── orders.json
├── pages/
│   ├── DashboardPage.ts
│   ├── AccountsPage.ts
│   └── OrdersPage.ts
└── playwright.config.ts
```

### 5.2 Page Object Model

```typescript
// pages/DashboardPage.ts
import { Page, Locator } from '@playwright/test';

export class DashboardPage {
  readonly page: Page;
  readonly robotStatus: Locator;
  readonly accountCards: Locator;
  readonly activityFeed: Locator;
  readonly statsCards: Locator;

  constructor(page: Page) {
    this.page = page;
    this.robotStatus = page.getByTestId('robot-status');
    this.accountCards = page.getByTestId('account-card');
    this.activityFeed = page.getByTestId('activity-feed');
    this.statsCards = page.getByTestId('stat-card');
  }

  async goto() {
    await this.page.goto('/');
  }

  async getRobotStatus(): Promise<string> {
    return await this.robotStatus.textContent() || '';
  }

  async getAccountCount(): Promise<number> {
    return await this.accountCards.count();
  }

  async waitForWebSocketUpdate(eventType: string, timeout = 5000) {
    await this.page.waitForFunction(
      (type) => window.__lastWsEvent?.event === type,
      eventType,
      { timeout }
    );
  }
}
```

### 5.3 E2E Test Examples

```typescript
// tests/dashboard.spec.ts
import { test, expect } from '@playwright/test';
import { DashboardPage } from '../pages/DashboardPage';

test.describe('Dashboard', () => {
  let dashboard: DashboardPage;

  test.beforeEach(async ({ page }) => {
    // Login
    await page.goto('/login');
    await page.fill('[name="username"]', 'owner');
    await page.fill('[name="password"]', 'password');
    await page.click('button[type="submit"]');
    
    dashboard = new DashboardPage(page);
    await dashboard.goto();
  });

  test('shows robot status', async () => {
    const status = await dashboard.getRobotStatus();
    expect(['Running', 'Stopped']).toContain(status);
  });

  test('displays all accounts', async () => {
    const count = await dashboard.getAccountCount();
    expect(count).toBeGreaterThan(0);
  });

  test('receives real-time order updates', async ({ page }) => {
    // Setup WebSocket listener
    await page.evaluate(() => {
      window.__wsEvents = [];
      const origWs = WebSocket;
      window.WebSocket = class extends origWs {
        constructor(url: string) {
          super(url);
          this.addEventListener('message', (e) => {
            window.__wsEvents.push(JSON.parse(e.data));
          });
        }
      };
    });

    // Trigger order update via API
    await page.request.post('/api/test/trigger-order-update', {
      data: { order_id: 'ord_001', status: 'tp_hit' }
    });

    // Wait for UI update
    await expect(page.getByText('TP Hit')).toBeVisible({ timeout: 5000 });
  });

  test('navigates to account details', async ({ page }) => {
    await page.click('[data-testid="account-card"]:first-child');
    
    await expect(page).toHaveURL(/\/accounts\/ACC_/);
    await expect(page.getByText('Account Details')).toBeVisible();
  });
});
```

```typescript
// tests/orders.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Orders', () => {
  test.beforeEach(async ({ page }) => {
    await loginAsOwner(page);
    await page.goto('/orders');
  });

  test('filters orders by account', async ({ page }) => {
    await page.selectOption('[data-testid="account-filter"]', 'ACC_001');
    await page.click('button:has-text("Apply")');

    const rows = page.locator('tbody tr');
    const count = await rows.count();
    
    for (let i = 0; i < count; i++) {
      const accountCell = rows.nth(i).locator('td:nth-child(2)');
      await expect(accountCell).toHaveText('ACC_001');
    }
  });

  test('filters orders by status', async ({ page }) => {
    await page.selectOption('[data-testid="status-filter"]', 'tp_hit');
    await page.click('button:has-text("Apply")');

    const statusBadges = page.locator('[data-testid="status-badge"]');
    const count = await statusBadges.count();
    
    for (let i = 0; i < count; i++) {
      await expect(statusBadges.nth(i)).toHaveText('TP Hit');
    }
  });

  test('exports orders to CSV', async ({ page }) => {
    const [download] = await Promise.all([
      page.waitForEvent('download'),
      page.click('button:has-text("CSV")')
    ]);

    expect(download.suggestedFilename()).toMatch(/orders.*\.csv/);
  });

  test('shows order detail on click', async ({ page }) => {
    await page.click('tbody tr:first-child');
    
    await expect(page.getByText('Order Detail')).toBeVisible();
    await expect(page.getByText('State History')).toBeVisible();
  });
});
```

---

## 6. Performance Testing

### 6.1 Load Testing with k6

```javascript
// load-test/api-load.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },  // Ramp up
    { duration: '3m', target: 10 },  // Stay at 10 users
    { duration: '1m', target: 50 },  // Spike to 50
    { duration: '2m', target: 50 },  // Stay at 50
    { duration: '1m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% requests < 500ms
    http_req_failed: ['rate<0.01'],   // Error rate < 1%
  },
};

const BASE_URL = 'http://localhost:8080/api/v1';
const TOKEN = __ENV.AUTH_TOKEN;

export default function () {
  const headers = {
    'Authorization': `Bearer ${TOKEN}`,
    'Content-Type': 'application/json',
  };

  // Get robot status
  let res = http.get(`${BASE_URL}/robot/status`, { headers });
  check(res, {
    'robot status 200': (r) => r.status === 200,
    'robot status fast': (r) => r.timings.duration < 100,
  });

  sleep(1);

  // List accounts
  res = http.get(`${BASE_URL}/accounts`, { headers });
  check(res, {
    'accounts 200': (r) => r.status === 200,
    'accounts fast': (r) => r.timings.duration < 200,
  });

  sleep(1);

  // List orders with pagination
  res = http.get(`${BASE_URL}/orders?page=1&limit=20`, { headers });
  check(res, {
    'orders 200': (r) => r.status === 200,
    'orders fast': (r) => r.timings.duration < 300,
  });

  sleep(1);
}
```

### 6.2 WebSocket Load Test

```javascript
// load-test/ws-load.js
import ws from 'k6/ws';
import { check } from 'k6';

export const options = {
  vus: 100,
  duration: '5m',
};

export default function () {
  const url = `ws://localhost:8080/ws?token=${__ENV.AUTH_TOKEN}`;
  
  const res = ws.connect(url, {}, function (socket) {
    socket.on('open', () => {
      socket.send(JSON.stringify({
        type: 'subscribe',
        channels: ['robot', 'accounts', 'orders']
      }));
    });

    socket.on('message', (data) => {
      const msg = JSON.parse(data);
      check(msg, {
        'has event type': (m) => m.event !== undefined,
        'has timestamp': (m) => m.timestamp !== undefined,
      });
    });

    socket.setTimeout(() => {
      socket.close();
    }, 60000);
  });

  check(res, {
    'ws connected': (r) => r && r.status === 101,
  });
}
```

---

## 7. CI/CD Integration

### 7.1 GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      
      - name: Run Backend Unit Tests
        run: |
          cd robot-engine && go test -v -coverprofile=coverage.out ./...
          cd ../owner-server && go test -v -coverprofile=coverage.out ./...
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Run Frontend Unit Tests
        run: |
          cd dashboard
          npm ci
          npm run test:coverage

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: trading_test
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      
      - name: Run Integration Tests
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/trading_test
        run: |
          cd owner-server
          go test -v -tags=integration ./...

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install Playwright
        run: |
          cd e2e
          npm ci
          npx playwright install --with-deps
      
      - name: Start Services
        run: docker-compose -f docker-compose.test.yml up -d
      
      - name: Wait for Services
        run: |
          sleep 10
          curl --retry 10 --retry-delay 2 http://localhost:8080/health
      
      - name: Run E2E Tests
        run: |
          cd e2e
          npx playwright test
      
      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: e2e/playwright-report/
```

---

## 8. Test Coverage Goals

| Component | Target | Current |
|-----------|--------|---------|
| Robot Engine | 80% | - |
| Owner Server | 80% | - |
| Dashboard | 70% | - |
| API Integration | 90% | - |
| E2E Critical Paths | 100% | - |

---

## 9. Test Data Management

### 9.1 Fixtures

```json
// fixtures/accounts.json
{
  "accounts": [
    {
      "id": "ACC_001",
      "broker": "stockbit",
      "status": "online",
      "enabled": true
    },
    {
      "id": "ACC_002",
      "broker": "ipot",
      "status": "offline",
      "enabled": true
    }
  ]
}
```

### 9.2 Factory Pattern

```go
// testutil/factory.go
package testutil

func NewTestTask(overrides ...func(*Task)) *Task {
    task := &Task{
        ID:        generateID("task"),
        AccountID: "ACC_001",
        Emiten:    "BBCA",
        Action:    "BUY",
        Price:     9250,
        Lot:       10,
        TPPrice:   9500,
        CLPrice:   9100,
        Status:    "pending",
        CreatedAt: time.Now(),
    }
    
    for _, override := range overrides {
        override(task)
    }
    
    return task
}

// Usage
task := NewTestTask(func(t *Task) {
    t.Status = "completed"
    t.Result = "tp_hit"
})
```

---

## ✅ Status

| Item | Status |
|------|--------|
| Unit test strategy | ✅ Defined |
| Integration test strategy | ✅ Defined |
| E2E test strategy | ✅ Defined |
| Performance test strategy | ✅ Defined |
| CI/CD integration | ✅ Defined |
| Coverage goals | ✅ Defined |
