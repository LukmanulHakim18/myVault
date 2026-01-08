# Group Order Flows (Shuttle / Cititrans)

## Overview
**GroupOrderDetail** adalah model khusus untuk menangani order grup dalam layanan **Shuttle (Cititrans)**. Model ini memungkinkan pengelolaan multiple order items dalam satu grup pemesanan.

## Data Model

### Entity Relationship
```
┌──────────────┐         ┌────────────────────────┐         ┌──────────────────────────┐
│  GroupOrder  │1────────*│ GroupOrderDetail      │1────────1│GroupOrderDetailMetadata │
│              │         │                        │         │                          │
│ - id         │         │ - group_order_id (FK)  │         │ - order_id (FK)          │
│ - bb_id      │         │ - internal_user_id     │         │ - category               │
│ - email      │         │ - order_id             │         │ - app_version            │
│ - total_price│         │ - price                │         │ - language               │
└──────────────┘         │ - status               │         │ - os                     │
                         │ - title                │         │ - device_model           │
                         │ - description          │         └──────────────────────────┘
                         │ - order_date           │
                         │ - outstanding          │
                         │ - product_type         │
                         │ - deleted_at           │
                         └────────────────────────┘
```

### GroupOrderDetail Model
```go
type GroupOrderDetail struct {
    GroupOrderID   int64        `db:"group_order_id"`    // FK to group_order
    InternalUserID string       `db:"internal_user_id"`  // User BBID
    Category       string       `db:"category"`          // "shuttle"
    OrderID        int64        `db:"order_id"`          // Unique order ID
    Price          float64      `db:"price"`             // Order price
    Status         string       `db:"status"`            // "active", "schedule", "history"
    Title          string       `db:"title"`             // JSON {en, id}
    Description    string       `db:"description"`       // JSON {en, id}
    OrderDate      sql.NullTime `db:"order_date"`        // Scheduled date
    Outstanding    float64      `db:"outstanding"`       // Unpaid amount
    Header         string       `db:"header"`            // JSON {en, id, color}
    ProductType    string       `db:"product_type"`      // cititrans_executive, etc.
    DeletedAt      sql.NullTime `db:"deleted_at"`        // Soft delete
    CreatedAt      time.Time    `db:"created_at"`
    UpdatedAt      sql.NullTime `db:"updated_at"`
}
```

## Status Lifecycle

### Status Transitions
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   SCHEDULE   │────>│    ACTIVE    │────>│   HISTORY    │
└──────────────┘     └──────────────┘     └──────────────┘
      │                     │                     │
      │  order_date         │  order_date         │  order_date
      │  > now              │  < now &&           │  < now - 2 days
      │                     │  > now - 2 days     │
      └─────────────────────┴─────────────────────┘
              (Managed by RoutineOrderHistoryUpdate cronjob)
```

### Status Rules

| Status | Condition | Description |
|--------|-----------|-------------|
| **SCHEDULE** | `order_date > now` | Future booking, not yet started |
| **ACTIVE** | `order_date <= now AND order_date > now - 2 days` | Current/recent trip (within 2 days) |
| **HISTORY** | `order_date <= now - 2 days` | Completed trip (older than 2 days) |

**Automatic Status Updates**: Cronjob `RoutineOrderHistoryUpdate` runs periodically to update statuses based on `order_date`.

## Core Operations

### API Summary

| Operation | Method | Purpose | Type |
|-----------|--------|---------|------|
| **GetListOrder** | Read | Query group orders with filters | READ |
| **UpsertGroupOrderDetail** | Write | Batch insert/update | CREATE/UPDATE |
| **RescheduleGroupOrderDetail** | Write | Reschedule order | CREATE/UPDATE |
| **DeleteGroupOrderDetail** | Write | Soft delete order | DELETE |
| **CreateGroupOrderDetailMetadata** | Write | Store device metadata | CREATE |
| **GetGroupOrderDetailMetadata** | Read | Retrieve metadata | READ |

---

## 1. GetListOrder

### Purpose
Query group order details untuk user dengan filter category, status, dan cursor-based pagination.

### Flow Diagram
```
Mobile App
    │
    ├─ POST /v1/orders/list
    │  {category: "shuttle", status: "active", cursor: "...", limit: 10}
    │
    ▼
Transport Layer (Interceptors)
    │
    ├─ InputValidator: Validate request structure
    ├─ MetadataValidator: Check headers (app_version, device_id, etc.)
    │
    ▼
UseCase Layer
    │
    ├─ Parse cursor (decode base64 → lastOrderID)
    ├─ Build filter {user_id, category, status, limit, cursor}
    │
    ├─ GetGroupOrderDetailFilter() → []GroupOrderDetail
    ├─ CountGroupOrderDetailWithFilter() → totalCount
    │
    ├─ Parallel: Get file resources from ContentProvider
    │  ├─ cititrans_executive icon
    │  ├─ cititrans_super_executive icon
    │  ├─ cititrans_suite icon
    │  ├─ cititrans_jac icon
    │  └─ status icons (active, schedule, history)
    │
    ├─ MapGroupOrderDetailToOrderResponse()
    │  ├─ Parse JSON fields (title, description, header)
    │  ├─ Localize strings (en/id based on user language)
    │  ├─ Format price → "Rp 150.000"
    │  ├─ Add product icons
    │  └─ Add action buttons (reschedule, cancel, etc.)
    │
    └─ CreatePagination(lastOrderID, count, total)
       └─ Encode next cursor (base64)
    │
    ▼
Response
    {
      "metadata": {
        "count_data": 10,
        "next_cursor": "base64_cursor",
        "total_data": 45
      },
      "records": [...]
    }
```

### Request
```json
{
  "category": "shuttle",
  "status": "active",  // or "schedule", "history"
  "cursor": "eyJvcmRlcl9pZCI6MTIzNDV9",  // base64 encoded
  "limit": 10
}
```

### Response
```json
{
  "metadata": {
    "count_data": 10,
    "next_cursor": "eyJvcmRlcl9pZCI6MTIzNTV9",
    "total_data": 45
  },
  "records": [
    {
      "order_id": 12345,
      "category": "shuttle",
      "order_detail_display": {
        "icon": "https://cdn.bluebird.id/icons/cititrans_executive.png",
        "status_icon": "https://cdn.bluebird.id/icons/status_active.png"
      },
      "header": {
        "en": "Executive Shuttle",
        "id": "Shuttle Eksekutif",
        "color": "#1E88E5"
      },
      "title": {
        "en": "Jakarta to Bandung",
        "id": "Jakarta ke Bandung"
      },
      "description": {
        "en": "Scheduled for Jan 10, 2025 08:00 AM",
        "id": "Dijadwalkan untuk 10 Jan 2025 08:00 WIB"
      },
      "price": {
        "formatted": "Rp 150.000",
        "value": 150000
      },
      "buttons": [
        {
          "action": "reschedule",
          "label": {"en": "Reschedule", "id": "Jadwal Ulang"}
        },
        {
          "action": "cancel",
          "label": {"en": "Cancel", "id": "Batalkan"}
        }
      ]
    }
  ]
}
```

### Database Query
```sql
SELECT * FROM group_order_detail
WHERE internal_user_id = ?
  AND category = ?
  AND status = ?
  AND order_id < ?  -- cursor pagination
  AND deleted_at IS NULL
ORDER BY order_id DESC
LIMIT ?;
```

---

## 2. UpsertGroupOrderDetail

### Purpose
Batch insert atau update group order details. Operation ini idempotent - jika order_id sudah ada, update; jika belum, insert baru.

### Flow Diagram
```
Upstream Service (e.g., Order Creator)
    │
    ├─ POST /v1/group-orders/upsert
    │  {group_order_details: [{order_id: 12345, ...}, ...]}
    │
    ▼
Transport Layer
    │
    ├─ InputValidator: Validate batch request
    │
    ▼
UseCase Layer
    │
    ├─ Initialize: groupOrderID = 0, response.success = true
    │
    ├─ Loop: For each detail in batch
    │  │
    │  ├─ GetByOrderId(order_id, category)
    │  │
    │  ├─ IF EXISTS:
    │  │  └─ UpdateGroupOrderDetail(existing.ID, detail)
    │  │     └─ UPDATE group_order_detail SET ... WHERE id = ?
    │  │
    │  └─ ELSE (not exists):
    │     ├─ IF groupOrderID == 0:
    │     │  └─ CreateGroupOrder(bb_id, email, 0)
    │     │     └─ INSERT INTO group_order ... RETURNING id
    │     │
    │     └─ CreateGroupOrderDetail(detail, groupOrderID)
    │        └─ INSERT INTO group_order_detail ...
    │
    └─ Return response {success: true/false, errors: [...]}
    │
    ▼
Response
    {
      "success": true,
      "message": "Group order details upserted successfully",
      "errors": []  // if any individual operations failed
    }
```

### Request
```json
{
  "group_order_details": [
    {
      "internal_user_id": "user_123",
      "category": "shuttle",
      "order_id": 12345,
      "price": 150000,
      "status": "schedule",
      "title": "{\"en\":\"Jakarta to Bandung\",\"id\":\"Jakarta ke Bandung\"}",
      "description": "{\"en\":\"Executive\",\"id\":\"Eksekutif\"}",
      "order_date": "2025-01-10T08:00:00Z",
      "customer_email": "user@example.com",
      "product_type": "cititrans_executive",
      "outstanding": 0
    },
    {
      "internal_user_id": "user_123",
      "category": "shuttle",
      "order_id": 12346,
      "price": 200000,
      "status": "schedule",
      "title": "{\"en\":\"Bandung to Jakarta\",\"id\":\"Bandung ke Jakarta\"}",
      "description": "{\"en\":\"Suite\",\"id\":\"Suite\"}",
      "order_date": "2025-01-12T15:00:00Z",
      "customer_email": "user@example.com",
      "product_type": "cititrans_suite",
      "outstanding": 0
    }
  ]
}
```

### Response
```json
{
  "success": true,
  "message": "2 group order details upserted successfully"
}
```

### Key Logic
```go
// Idempotent upsert logic
for _, detail := range request.GroupOrderDetails {
    existing, err := repo.GetByOrderId(detail.OrderID, detail.Category)
    
    if err == nil {
        // Update existing
        err = repo.UpdateGroupOrderDetail(existing.ID, detail)
    } else {
        // Create new
        if groupOrderID == 0 {
            // Create parent GroupOrder if not exists
            groupOrderID, err = repo.CreateGroupOrder(bbid, email, 0)
        }
        err = repo.CreateGroupOrderDetail(detail, groupOrderID)
    }
}
```

---

## 3. RescheduleGroupOrderDetail

### Purpose
Reschedule group order dengan membuat order baru dan mengupdate order lama ke status "history".

### Flow Diagram
```
Mobile App
    │
    ├─ POST /v1/group-orders/reschedule
    │  {order_id: 12345, new_order_date: "2025-01-15T08:00:00Z"}
    │
    ▼
UseCase Layer
    │
    ├─ GetByOrderId(order_id, category)
    │  └─ SELECT * FROM group_order_detail WHERE order_id = ?
    │
    ├─ IF NOT FOUND:
    │  └─ Return error ODQR-4041 (Order not found)
    │
    ├─ Create new order (copy from existing)
    │  ├─ newOrderID = GenerateNewOrderID()
    │  ├─ newDetail = CloneOrderDetail(existing)
    │  ├─ newDetail.OrderDate = new_order_date
    │  ├─ newDetail.Status = DetermineStatus(new_order_date)
    │  │
    │  └─ CreateGroupOrderDetail(newDetail, existing.GroupOrderID)
    │
    ├─ Update original order to history
    │  └─ UpdateGroupOrderDetail(existing.ID, {status: "history"})
    │
    └─ Return newOrderID
    │
    ▼
Response
    {
      "id": 12350  // new order ID
    }
```

### Request
```json
{
  "order_id": 12345,
  "internal_user_id": "user_123",
  "new_order_date": "2025-01-15T08:00:00Z"
}
```

### Response
```json
{
  "id": 12350  // new order ID created
}
```

### Status Determination for New Order
```go
func DetermineStatus(orderDate time.Time) string {
    now := time.Now()
    if orderDate.After(now) {
        return "schedule"  // Future
    } else if orderDate.After(now.Add(-2 * 24 * time.Hour)) {
        return "active"    // Within 2 days
    } else {
        return "history"   // Older than 2 days
    }
}
```

**Important**: Original order is updated to "history" status, NOT deleted. This maintains audit trail.

---

## 4. DeleteGroupOrderDetail

### Purpose
Soft delete group order detail (set `deleted_at` timestamp).

### Flow Diagram
```
Mobile App
    │
    ├─ POST /v1/group-orders/delete
    │  {order_id: 12345}
    │
    ▼
UseCase Layer
    │
    ├─ GetByOrderId(order_id, category)
    │  └─ SELECT * FROM group_order_detail WHERE order_id = ? AND deleted_at IS NULL
    │
    ├─ IF NOT FOUND:
    │  └─ Return error ODQR-4041 (Order not found)
    │
    ├─ SoftDeleteGroupOrderDetail(order_id, category)
    │  └─ UPDATE group_order_detail 
    │     SET deleted_at = NOW() 
    │     WHERE order_id = ? AND category = ?
    │
    └─ Return success {deleted_count: 1}
    │
    ▼
Response
    {
      "success": true,
      "deleted_count": 1
    }
```

### Request
```json
{
  "order_id": 12345,
  "internal_user_id": "user_123"
}
```

### Response
```json
{
  "success": true,
  "deleted_count": 1
}
```

### Database Operation
```sql
UPDATE group_order_detail
SET deleted_at = NOW()
WHERE order_id = ?
  AND category = ?
  AND internal_user_id = ?
  AND deleted_at IS NULL;
```

**Note**: Soft delete preserves data for audit/recovery. To query active orders, always filter `deleted_at IS NULL`.

---

## 5. Metadata Management

### CreateGroupOrderDetailMetadata
Store device and app metadata untuk analytics.

**Request**:
```json
{
  "order_id": 12345,
  "category": "shuttle",
  "app_version": "2.5.0",
  "language": "id",
  "os": "Android",
  "manufacturer": "Samsung",
  "device_model": "SM-G973F",
  "operating_system": "Android 13"
}
```

**Response**:
```json
{
  "id": 111  // metadata ID
}
```

### GetGroupOrderDetailMetadata
Retrieve metadata untuk order.

**Request**:
```json
{
  "order_id": 12345
}
```

**Response**:
```json
{
  "order_id": 12345,
  "category": "shuttle",
  "app_version": "2.5.0",
  "language": "id",
  "os": "Android",
  "manufacturer": "Samsung",
  "device_model": "SM-G973F",
  "operating_system": "Android 13"
}
```

---

## Automatic Status Updates (Cronjob)

### RoutineOrderHistoryUpdate

**Purpose**: Periodically update order statuses based on `order_date`.

**Schedule**: Runs daily (or configured interval)

**Logic**:
```sql
-- Move SCHEDULE → ACTIVE (when order_date <= now)
UPDATE group_order_detail
SET status = 'active'
WHERE status = 'schedule'
  AND order_date <= NOW()
  AND deleted_at IS NULL;

-- Move ACTIVE → HISTORY (when order_date <= now - 2 days)
UPDATE group_order_detail
SET status = 'history'
WHERE status = 'active'
  AND order_date <= NOW() - INTERVAL '2 days'
  AND deleted_at IS NULL;
```

**Execution**:
```bash
# Run as cronjob
./orderquery -execUsecase=cronjoborderhistoryupdate

# Or schedule in Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: orderquery-status-update
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: orderquery
            image: orderquery:latest
            args: ["-execUsecase=cronjoborderhistoryupdate"]
```

---

## Data Localization

### JSON Fields
Several fields store localized content as JSON:

#### Title
```json
{
  "en": "Jakarta to Bandung",
  "id": "Jakarta ke Bandung"
}
```

#### Description
```json
{
  "en": "Executive shuttle service",
  "id": "Layanan shuttle eksekutif"
}
```

#### Header
```json
{
  "en": "Upcoming Trip",
  "id": "Perjalanan Mendatang",
  "color": "#1E88E5"
}
```

### Response Localization
UseCase layer parses JSON and returns appropriate language:

```go
func LocalizeField(jsonStr, language string) string {
    var content map[string]string
    json.Unmarshal([]byte(jsonStr), &content)
    
    if val, ok := content[language]; ok {
        return val
    }
    return content["en"]  // fallback to English
}
```

---

## Error Handling

### Common Errors

| Error Code | Scenario | Solution |
|------------|----------|----------|
| ODQR-4001 | Missing required parameter | Check request payload |
| ODQR-4002 | Invalid parameter value | Validate order_date format, status enum |
| ODQR-4041 | Order not found | Verify order_id exists and not deleted |
| ODQR-5000 | Internal server error | Check logs, database connectivity |

### Example Error Response
```json
{
  "error_code": "ODQR-4041",
  "status_code": 404,
  "localized_message": {
    "english": "Order not found",
    "indonesia": "Pesanan tidak ditemukan"
  }
}
```

---

## Best Practices

### 1. Always Use Batch Operations
```go
// ✅ GOOD - Batch upsert
UpsertGroupOrderDetail([]GroupOrderDetail{detail1, detail2, detail3})

// ❌ BAD - Multiple single calls
for _, detail := range details {
    UpsertGroupOrderDetail([]GroupOrderDetail{detail})
}
```

### 2. Handle Soft Deletes
```sql
-- ✅ GOOD - Filter deleted records
SELECT * FROM group_order_detail 
WHERE deleted_at IS NULL;

-- ❌ BAD - May return deleted records
SELECT * FROM group_order_detail;
```

### 3. Use Cursor Pagination
```go
// ✅ GOOD - Cursor-based
GetListOrder(category, status, cursor, limit)

// ❌ BAD - Offset-based (slow for large datasets)
GetListOrder(category, status, offset, limit)
```

### 4. Validate Reschedule Eligibility
```go
// Before reschedule, check:
// - Order not already completed
// - New date is valid (not in past)
// - Order not already deleted
```

---

## Performance Considerations

### Indexing Strategy
```sql
-- Recommended indexes
CREATE INDEX idx_god_user_status ON group_order_detail(internal_user_id, status) 
    WHERE deleted_at IS NULL;

CREATE INDEX idx_god_order_date ON group_order_detail(order_date) 
    WHERE deleted_at IS NULL;

CREATE INDEX idx_god_category_status ON group_order_detail(category, status) 
    WHERE deleted_at IS NULL;
```

### Query Optimization
- Use `deleted_at IS NULL` in WHERE clause (not SELECT)
- Limit results with cursor pagination
- Parallel fetch icons from ContentProvider
- Cache product type icons

---

## Related Documentation
- [[01-overview|Service Overview]]
- [[02-api-reference|API Reference]]
- [[04-dependencies|Dependencies]]
- [[_doc/GROUP-ORDER-DETAIL-FLOWS.md|Full Flow Documentation]]

---
**Last Updated**: 2025-01-07
**Feature**: Group Order Management
**Service Category**: Shuttle (Cititrans)