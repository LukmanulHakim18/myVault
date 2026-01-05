# ID Generator - Implementation Guide

## Migration Strategy

### Overview

Gradual migration dari sequential database ID ke Two-Level ID system tanpa downtime.

## Phase 1: Database Schema Changes

### Step 1.1: Add New Columns

```sql
-- Add exposed_id columns
ALTER TABLE bookings ADD COLUMN exposed_booking_id VARCHAR(18);
ALTER TABLE orders ADD COLUMN exposed_order_id VARCHAR(18);
ALTER TABLE orders ADD COLUMN exposed_booking_id VARCHAR(18);

-- Add metadata columns for tracking
ALTER TABLE bookings ADD COLUMN channel VARCHAR(2);
ALTER TABLE bookings ADD COLUMN booking_type CHAR(1);
ALTER TABLE orders ADD COLUMN product_type VARCHAR(2);
```

### Step 1.2: Create Indexes (Non-blocking)

```sql
-- Create indexes concurrently to avoid locking
CREATE INDEX CONCURRENTLY idx_exposed_booking_id ON bookings(exposed_booking_id);
CREATE INDEX CONCURRENTLY idx_exposed_order_id ON orders(exposed_order_id);
CREATE INDEX CONCURRENTLY idx_orders_exposed_booking ON orders(exposed_booking_id);
```

## Phase 2: Backfill Existing Data

### Step 2.1: Install ID Generator Package

```bash
go get github.com/bluebird/id-generator
```

### Step 2.2: Backfill Script

```go
package main

import (
    "database/sql"
    "log"
    idgen "github.com/bluebird/id-generator"
)

func BackfillBookings(db *sql.DB, client *idgen.IDGenerator) error {
    // Fetch bookings without exposed_id
    rows, err := db.Query(`
        SELECT id, channel, booking_type 
        FROM bookings 
        WHERE exposed_booking_id IS NULL
        ORDER BY id
        LIMIT 1000
    `)
    if err != nil {
        return err
    }
    defer rows.Close()
    
    for rows.Next() {
        var id int64
        var channel, bookingType string
        rows.Scan(&id, &channel, &bookingType)
        
        // Generate exposed ID
        exposedID, err := client.GenerateBookingID(id, channel, bookingType)
        if err != nil {
            log.Printf("Error generating ID for booking %d: %v", id, err)
            continue
        }
        
        // Update database
        _, err = db.Exec(`
            UPDATE bookings 
            SET exposed_booking_id = $1 
            WHERE id = $2
        `, exposedID, id)
        
        if err != nil {
            log.Printf("Error updating booking %d: %v", id, err)
        }
    }
    
    return nil
}

func BackfillOrders(db *sql.DB, client *idgen.IDGenerator) error {
    rows, err := db.Query(`
        SELECT o.id, o.channel, o.product_type, b.exposed_booking_id
        FROM orders o
        JOIN bookings b ON b.id = o.booking_id
        WHERE o.exposed_order_id IS NULL
        ORDER BY o.id
        LIMIT 1000
    `)
    if err != nil {
        return err
    }
    defer rows.Close()
    
    for rows.Next() {
        var id int64
        var channel, product, bookingID string
        rows.Scan(&id, &channel, &product, &bookingID)
        
        // Generate exposed ID
        exposedID, err := client.GenerateOrderID(id, channel, product)
        if err != nil {
            log.Printf("Error generating ID for order %d: %v", id, err)
            continue
        }
        
        // Update database
        _, err = db.Exec(`
            UPDATE orders 
            SET exposed_order_id = $1, exposed_booking_id = $2
            WHERE id = $3
        `, exposedID, bookingID, id)
        
        if err != nil {
            log.Printf("Error updating order %d: %v", id, err)
        }
    }
    
    return nil
}
```

### Step 2.3: Run Backfill

```bash
# Run in batches to avoid long transactions
./backfill-script --batch-size=1000 --delay=100ms
```

## Phase 3: Application Code Changes

### Step 3.1: Initialize Client

```go
// config/id_generator.go
package config

import (
    "os"
    "time"
    idgen "github.com/bluebird/id-generator"
)

var IDClient *idgen.IDGenerator

func InitIDGenerator() {
    location, _ := time.LoadLocation("Asia/Jakarta")
    
    IDClient = idgen.NewClient(idgen.Config{
        SecretKey:    os.Getenv("ID_GENERATOR_SECRET"),
        TimeLocation: location,
    })
}
```

### Step 3.2: Update Booking Creation

```go
// Before (old code)
func CreateBooking(db *gorm.DB, req *BookingRequest) (*Booking, error) {
    booking := &Booking{
        CustomerID:  req.CustomerID,
        TotalAmount: req.TotalAmount,
    }
    if err := db.Create(booking).Error; err != nil {
        return nil, err
    }
    return booking, nil
}

// After (with ID generator)
func CreateBooking(db *gorm.DB, req *BookingRequest) (*Booking, error) {
    booking := &Booking{
        Channel:      req.Channel,
        BookingType:  req.BookingType,
        CustomerID:   req.CustomerID,
        TotalAmount:  req.TotalAmount,
    }
    if err := db.Create(booking).Error; err != nil {
        return nil, err
    }
    
    // Generate exposed ID
    exposedID, err := config.IDClient.GenerateBookingID(
        booking.ID,
        booking.Channel,
        booking.BookingType,
    )
    if err != nil {
        return nil, err
    }
    
    booking.ExposedBookingID = exposedID
    db.Save(booking)
    
    return booking, nil
}
```

### Step 3.3: Update Order Creation

```go
func CreateOrder(db *gorm.DB, bookingID int64, req *OrderRequest) (*Order, error) {
    // Get parent booking
    var booking Booking
    if err := db.First(&booking, bookingID).Error; err != nil {
        return nil, err
    }
    
    order := &Order{
        BookingID:         bookingID,
        ExposedBookingID:  booking.ExposedBookingID,
        Channel:           booking.Channel,
        ProductType:       req.Product,
        Amount:            req.Amount,
    }
    
    if err := db.Create(order).Error; err != nil {
        return nil, err
    }
    
    // Generate exposed ID
    exposedID, err := config.IDClient.GenerateOrderID(
        order.ID,
        order.Channel,
        order.ProductType,
    )
    if err != nil {
        return nil, err
    }
    
    order.ExposedOrderID = exposedID
    db.Save(order)
    
    return order, nil
}
```

## Phase 4: API Response Changes

### Step 4.1: Update Response DTOs

```go
// Before
type BookingResponse struct {
    ID          int64   `json:"id"`
    CustomerID  int64   `json:"customer_id"`
    TotalAmount float64 `json:"total_amount"`
}

// After
type BookingResponse struct {
    ID          string  `json:"id"` // Now returns exposed_booking_id
    BookingID   string  `json:"booking_id"` // Alias for clarity
    CustomerID  int64   `json:"customer_id"`
    TotalAmount float64 `json:"total_amount"`
}

func ToBookingResponse(b *Booking) *BookingResponse {
    return &BookingResponse{
        ID:          b.ExposedBookingID,
        BookingID:   b.ExposedBookingID,
        CustomerID:  b.CustomerID,
        TotalAmount: b.TotalAmount,
    }
}
```

### Step 4.2: Support Both ID Formats (Transition Period)

```go
func GetBooking(exposedOrInternalID string) (*Booking, error) {
    var booking Booking
    
    // Try exposed ID first
    if err := db.Where("exposed_booking_id = ?", exposedOrInternalID).First(&booking).Error; err == nil {
        return &booking, nil
    }
    
    // Fallback to internal ID (for backward compatibility)
    if id, err := strconv.ParseInt(exposedOrInternalID, 10, 64); err == nil {
        if err := db.First(&booking, id).Error; err == nil {
            return &booking, nil
        }
    }
    
    return nil, ErrBookingNotFound
}
```

## Phase 5: Add Constraints

### Step 5.1: Add NOT NULL Constraints

```sql
-- After all data is backfilled
ALTER TABLE bookings ALTER COLUMN exposed_booking_id SET NOT NULL;
ALTER TABLE orders ALTER COLUMN exposed_order_id SET NOT NULL;
ALTER TABLE orders ALTER COLUMN exposed_booking_id SET NOT NULL;
```

### Step 5.2: Add Unique Constraints

```sql
-- Ensure uniqueness
ALTER TABLE bookings ADD CONSTRAINT uq_exposed_booking_id UNIQUE (exposed_booking_id);
ALTER TABLE orders ADD CONSTRAINT uq_exposed_order_id UNIQUE (exposed_order_id);
```

### Step 5.3: Add Validation Triggers

```sql
CREATE OR REPLACE FUNCTION validate_order_booking()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM bookings 
        WHERE id = NEW.booking_id 
        AND exposed_booking_id = NEW.exposed_booking_id
    ) THEN
        RAISE EXCEPTION 'Invalid booking reference';
    END IF;
    
    IF NEW.channel != (SELECT channel FROM bookings WHERE id = NEW.booking_id) THEN
        RAISE EXCEPTION 'Channel mismatch';
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validate_order_booking
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION validate_order_booking();
```

## Phase 6: Frontend Integration

### Step 6.1: Update UI Components

```typescript
// Before
interface Booking {
  id: number;
  customerName: string;
  totalAmount: number;
}

// After
interface Booking {
  id: string;           // exposed_booking_id
  bookingId: string;    // same as id
  customerName: string;
  totalAmount: number;
}

// Display booking ID with formatting
function formatBookingId(id: string): string {
  // MYS-NQBMEK7P2QF01 → MYS-NQBME K7P2QF01
  const parts = id.split('-');
  return `${parts[0]}-${parts[1].slice(0, 5)} ${parts[1].slice(5)}`;
}
```

### Step 6.2: Update Search/Filter

```typescript
// Support both ID formats during transition
async function searchBooking(query: string) {
  const response = await fetch(`/api/bookings/search?q=${query}`);
  return await response.json();
}

// Backend handles both:
// - "MYS-NQBMEK7P2QF01" (new format)
// - "123456" (old format, fallback)
```

## Phase 7: Customer Support Training

### Training Materials

**Document: ID System Quick Reference**

1. **ID Recognition**
   - 3 chars before dash = Booking ID
   - 4 chars before dash = Order ID

2. **Reading IDs over phone**
   - Spell character by character
   - Use phonetic alphabet for clarity
   - Example: "Mike Yankee Sierra dash..."

3. **Common Prefixes**
   - MY = MyBB app
   - WB = Web booking
   - CR = Corporate
   - First 2 chars always = channel

## Rollout Timeline

| Week | Phase | Activities |
|------|-------|------------|
| 1 | Schema | Add columns, create indexes |
| 2 | Backfill | Run backfill scripts (50% of data) |
| 3 | Backfill | Complete backfill (100% of data) |
| 4 | Code | Deploy new booking/order creation |
| 5 | API | Update API responses |
| 6 | Constraints | Add NOT NULL & UNIQUE constraints |
| 7 | Frontend | Update UI components |
| 8 | Training | Customer support training |
| 9 | Monitoring | Monitor for issues, collect feedback |
| 10 | Cleanup | Remove old ID fallback logic |

## Best Practices

### 1. Always Use Exposed IDs Externally

```go
// ❌ Bad: Exposing internal ID
type BookingResponse struct {
    ID int64 `json:"id"`
}

// ✅ Good: Using exposed ID
type BookingResponse struct {
    BookingID string `json:"booking_id"`
}
```

### 2. Keep Internal ID for Joins

```go
// Internal queries still use DB ID for performance
func GetBookingOrders(bookingID int64) ([]Order, error) {
    var orders []Order
    err := db.Where("booking_id = ?", bookingID).Find(&orders).Error
    return orders, err
}
```

### 3. Parse Before Querying

```go
// ✅ Good: Parse first, then query
info, err := idClient.ParseID(exposedID)
if err != nil {
    return nil, err
}
dbID, err := idClient.DecodeDBID(info.DBHash)
booking := db.First(&Booking{}, dbID)
```

### 4. Cache ID Generator Client

```go
// ✅ Good: Single instance, reused
var idClient *idgen.IDGenerator

func init() {
    idClient = idgen.NewClient(config)
}

// ❌ Bad: Creating new instance every time
func GenerateID() string {
    client := idgen.NewClient(config) // Don't do this!
    return client.GenerateBookingID(...)
}
```

## Monitoring & Alerts

### Metrics to Track

```go
// Track ID generation
metrics.Histogram("id_generation.duration", duration)
metrics.Counter("id_generation.total")
metrics.Counter("id_generation.errors")

// Track collisions (should be 0)
metrics.Counter("id_generation.collisions")
```

### Alerts

- Alert if ID generation > 1ms (p99)
- Alert if collision detected
- Alert if parse errors > 1%

## Troubleshooting

### Issue: ID Collision

```sql
-- Check for duplicates
SELECT exposed_booking_id, COUNT(*) 
FROM bookings 
GROUP BY exposed_booking_id 
HAVING COUNT(*) > 1;
```

**Solution**: This should never happen due to UNIQUE constraint. If it does, check secret key consistency.

### Issue: Cannot Decode ID

**Symptoms**: `DecodeDBID` returns wrong value

**Solution**: Verify secret key is same across all services:

```bash
# Check env variable
echo $ID_GENERATOR_SECRET

# Verify in code
fmt.Println(config.IDClient.GetSecretKey())
```

## Tags

#implementation #migration #best-practices #bluebird #id-generator