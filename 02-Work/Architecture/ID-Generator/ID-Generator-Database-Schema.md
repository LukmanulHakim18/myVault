# ID Generator - Database Schema

## Overview

Database schema untuk Two-Level ID Generation System dengan parent-child relationship antara bookings dan orders.

## Schema Architecture

```
bookings (parent)
    ├── id (internal)
    ├── exposed_booking_id (public)
    └── orders[] (children)
            ├── exposed_order_id (public)
            └── booking_id (FK)
```

## Bookings Table (Parent)

### Schema Definition

```sql
CREATE TABLE bookings (
    -- Primary identifiers
    id                  BIGSERIAL PRIMARY KEY,
    exposed_booking_id  VARCHAR(18) UNIQUE NOT NULL,
    
    -- Booking metadata
    channel             VARCHAR(2) NOT NULL,
    booking_type        CHAR(1) NOT NULL CHECK (booking_type IN ('S', 'M')),
    item_count          INTEGER NOT NULL DEFAULT 1,
    
    -- Financial
    total_amount        DECIMAL(15,2) NOT NULL,
    currency            CHAR(3) DEFAULT 'IDR',
    payment_status      VARCHAR(20) DEFAULT 'pending',
    
    -- Customer
    customer_id         BIGINT REFERENCES users(id),
    customer_name       VARCHAR(255),
    customer_phone      VARCHAR(20),
    customer_email      VARCHAR(255),
    
    -- Status tracking
    status              VARCHAR(20) DEFAULT 'pending',
    cancelled_at        TIMESTAMP,
    cancellation_reason TEXT,
    
    -- Timestamps
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW()
);
```

### Indexes

```sql
-- Primary lookup indexes
CREATE UNIQUE INDEX idx_exposed_booking_id ON bookings(exposed_booking_id);
CREATE INDEX idx_bookings_customer ON bookings(customer_id);
CREATE INDEX idx_bookings_created ON bookings(created_at);
CREATE INDEX idx_bookings_status ON bookings(status);

-- Analytics indexes
CREATE INDEX idx_bookings_channel ON bookings(channel);
CREATE INDEX idx_bookings_type ON bookings(booking_type);

-- Channel extraction for analytics
CREATE INDEX idx_bookings_channel_prefix ON bookings (
    LEFT(exposed_booking_id, 2)
);

-- Composite indexes for common queries
CREATE INDEX idx_bookings_customer_status ON bookings(customer_id, status, created_at);
CREATE INDEX idx_bookings_channel_date ON bookings(channel, created_at);
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| id | BIGSERIAL | Internal database ID (never exposed) |
| exposed_booking_id | VARCHAR(18) | Public-facing ID (CCT-TTTTTHHHHHHHHH) |
| channel | VARCHAR(2) | MY, WB, CR, AP, CT, WA |
| booking_type | CHAR(1) | S=Single, M=Multi |
| item_count | INTEGER | Number of orders in booking |
| total_amount | DECIMAL(15,2) | Total booking amount |
| payment_status | VARCHAR(20) | pending, paid, failed, refunded |
| status | VARCHAR(20) | pending, confirmed, completed, cancelled |

## Orders Table (Child)

### Schema Definition

```sql
CREATE TABLE orders (
    -- Primary identifiers
    id                  BIGSERIAL PRIMARY KEY,
    exposed_order_id    VARCHAR(18) UNIQUE NOT NULL,
    
    -- Parent relationship
    booking_id          BIGINT NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    exposed_booking_id  VARCHAR(18) NOT NULL,
    
    -- Order metadata
    channel             VARCHAR(2) NOT NULL,
    product_type        VARCHAR(2) NOT NULL,
    vehicle_type        CHAR(1) NOT NULL,  -- B, S, G, C
    service_type        CHAR(1) NOT NULL,  -- A, R, N, H, L, S, D
    
    -- Service details
    pickup_location     TEXT,
    pickup_lat          DECIMAL(10, 8),
    pickup_lng          DECIMAL(11, 8),
    dropoff_location    TEXT,
    dropoff_lat         DECIMAL(10, 8),
    dropoff_lng         DECIMAL(11, 8),
    pickup_time         TIMESTAMP,
    estimated_duration  INTEGER,  -- minutes
    
    -- Financial
    amount              DECIMAL(15,2) NOT NULL,
    base_fare           DECIMAL(15,2),
    distance_fare       DECIMAL(15,2),
    time_fare           DECIMAL(15,2),
    surcharge           DECIMAL(15,2),
    discount            DECIMAL(15,2),
    
    -- Driver assignment
    driver_id           BIGINT REFERENCES drivers(id),
    vehicle_id          BIGINT REFERENCES vehicles(id),
    assigned_at         TIMESTAMP,
    
    -- Status tracking
    status              VARCHAR(20) DEFAULT 'pending',
    started_at          TIMESTAMP,
    completed_at        TIMESTAMP,
    cancelled_at        TIMESTAMP,
    
    -- Timestamps
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP DEFAULT NOW()
);
```

### Indexes

```sql
-- Primary lookup indexes
CREATE UNIQUE INDEX idx_exposed_order_id ON orders(exposed_order_id);
CREATE INDEX idx_orders_booking ON orders(booking_id);
CREATE INDEX idx_orders_exposed_booking ON orders(exposed_booking_id);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_orders_status ON orders(status);

-- Analytics indexes
CREATE INDEX idx_orders_channel ON orders(channel);
CREATE INDEX idx_orders_product ON orders(product_type);
CREATE INDEX idx_orders_vehicle ON orders(vehicle_type);
CREATE INDEX idx_orders_service ON orders(service_type);

-- Product extraction for analytics
CREATE INDEX idx_orders_product_prefix ON orders (
    SUBSTRING(exposed_order_id, 1, 4)
);

-- Driver assignment indexes
CREATE INDEX idx_orders_driver ON orders(driver_id);
CREATE INDEX idx_orders_vehicle ON orders(vehicle_id);

-- Composite indexes for common queries
CREATE INDEX idx_orders_booking_status ON orders(booking_id, status, created_at);
CREATE INDEX idx_orders_driver_active ON orders(driver_id, status) 
    WHERE status IN ('assigned', 'picked_up', 'in_progress');
CREATE INDEX idx_orders_pickup_time ON orders(pickup_time) 
    WHERE status = 'pending';
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| id | BIGSERIAL | Internal database ID (never exposed) |
| exposed_order_id | VARCHAR(18) | Public-facing ID (CCPP-TTTTTHHHHHHHHH) |
| booking_id | BIGINT | FK to parent booking |
| exposed_booking_id | VARCHAR(18) | Parent booking public ID |
| product_type | VARCHAR(2) | BR, SN, GA, etc (vehicle+service) |
| vehicle_type | CHAR(1) | B=BlueBird, S=Silver, G=Golden, C=Citytrans |
| service_type | CHAR(1) | A=Airport, R=Ride, N=Rent, etc |
| status | VARCHAR(20) | pending, assigned, in_progress, completed, cancelled |

## Relationship Constraints

### Foreign Key Validation

```sql
-- Ensure booking-order relationship integrity
ALTER TABLE orders
ADD CONSTRAINT fk_orders_booking
FOREIGN KEY (booking_id, exposed_booking_id)
REFERENCES bookings(id, exposed_booking_id)
ON DELETE CASCADE;
```

### Validation Trigger

```sql
-- Trigger to validate booking-order relationship
CREATE OR REPLACE FUNCTION validate_order_booking()
RETURNS TRIGGER AS $$
BEGIN
    -- Verify booking exists
    IF NOT EXISTS (
        SELECT 1 FROM bookings 
        WHERE id = NEW.booking_id 
        AND exposed_booking_id = NEW.exposed_booking_id
    ) THEN
        RAISE EXCEPTION 'Invalid booking reference';
    END IF;
    
    -- Verify channel consistency
    IF NEW.channel != (
        SELECT channel FROM bookings WHERE id = NEW.booking_id
    ) THEN
        RAISE EXCEPTION 'Channel mismatch between order and booking';
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validate_order_booking
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION validate_order_booking();
```

## Common Queries

### Find all orders for a booking

```sql
SELECT * FROM orders 
WHERE exposed_booking_id = 'MYS-NQBMEK7P2QF01'
ORDER BY created_at;
```

### Get booking with all orders

```sql
SELECT 
    b.*,
    json_agg(o.*) as orders
FROM bookings b
LEFT JOIN orders o ON o.booking_id = b.id
WHERE b.exposed_booking_id = 'MYS-NQBMEK7P2QF01'
GROUP BY b.id;
```

### Analytics: Revenue by channel

```sql
SELECT 
    LEFT(exposed_booking_id, 2) as channel,
    COUNT(*) as total_bookings,
    SUM(total_amount) as revenue
FROM bookings
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY channel
ORDER BY revenue DESC;
```

### Analytics: Popular products

```sql
SELECT 
    SUBSTRING(exposed_order_id, 1, 4) as product,
    COUNT(*) as order_count,
    SUM(amount) as revenue
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY product
ORDER BY order_count DESC
LIMIT 10;
```

## Migration Strategy

### Phase 1: Add new columns
```sql
ALTER TABLE bookings ADD COLUMN exposed_booking_id VARCHAR(18);
ALTER TABLE orders ADD COLUMN exposed_order_id VARCHAR(18);
```

### Phase 2: Backfill existing data
```sql
-- Generate IDs for existing records
UPDATE bookings SET exposed_booking_id = generate_booking_id(id, channel, 'S');
UPDATE orders SET exposed_order_id = generate_order_id(id, channel, product_type);
```

### Phase 3: Add constraints
```sql
ALTER TABLE bookings ALTER COLUMN exposed_booking_id SET NOT NULL;
ALTER TABLE bookings ADD CONSTRAINT uq_exposed_booking_id UNIQUE (exposed_booking_id);

ALTER TABLE orders ALTER COLUMN exposed_order_id SET NOT NULL;
ALTER TABLE orders ADD CONSTRAINT uq_exposed_order_id UNIQUE (exposed_order_id);
```

## Performance Considerations

- **Index Size**: Exposed IDs index ~50% smaller than UUID
- **Query Performance**: Channel/product prefix queries use index efficiently
- **Partitioning**: Time hash enables time-based partitioning
- **Collision Prevention**: Unique constraint guarantees zero collisions

## Tags

#database #schema #postgresql #bluebird #id-generation