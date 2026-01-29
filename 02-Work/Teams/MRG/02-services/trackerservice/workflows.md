---
tags:
  - trackerservice
  - workflow
  - architecture
  - mrg
type: workflow-documentation
title: Tracker Service - Workflows
parent: trackerservice
---
# Tracker Service - Workflows

**Service**: [[README|Tracker Service]]  
**Type**: Workflow Documentation

---

## 🔄 Workflow Overview

TrackerService mendukung berbagai workflows untuk real-time location tracking dan trip monitoring dalam MyBB ecosystem. Setiap workflow dirancang untuk provide accurate, real-time location data dengan minimal latency.

---

## 1️⃣ Real-Time Car Tracking Workflow

**Scenario**: Driver dalam perjalanan, location di-update setiap 20 detik

### Sequence Diagram

```mermaid
sequenceDiagram
    participant DriverApp
    participant TrackerService
    participant Firebase
    participant Redis
    participant CustomerApp

    Note over DriverApp: Trip started
    
    loop Every 20 seconds
        DriverApp->>DriverApp: Get current GPS location
        DriverApp->>TrackerService: CarTracking(car_number, order_id, location)
        
        TrackerService->>TrackerService: Validate order is active
        
        alt Order Active
            TrackerService->>Firebase: Write location to /car_tracking/{order_id}
            Firebase-->>TrackerService: Success
            
            TrackerService->>Redis: Cache location (TTL: 24h)
            Redis-->>TrackerService: Cached
            
            TrackerService-->>DriverApp: 200 OK
            
            Note over Firebase,CustomerApp: Real-time sync
            Firebase-->>CustomerApp: Location update event
            CustomerApp->>CustomerApp: Update map marker
        else Order Inactive
            TrackerService-->>DriverApp: 400 Error: Order not active
        end
    end
    
    Note over DriverApp: Trip ended
    DriverApp->>TrackerService: OrderChanges(TRIP_ENDED)
    TrackerService->>Firebase: Set tracking_active = false
    TrackerService->>Redis: Clear location cache
```

### Key Steps

1. **GPS Location Acquisition**
   - Driver app gets location from device GPS
   - Frequency: Every 20 seconds (configurable via `CAR_LOCATION_FREQUENCY`)
   - Include: latitude, longitude, timestamp, accuracy

2. **Location Update Request**
   ```go
   request := &CarLocationRequest{
       CarNumber: "B1234XYZ",
       OrderID: 12345,
       UserID: driverID,
       MetaData: metadata,
   }
   ```

3. **Order Validation**
   - Check order exists and is active
   - Verify driver is assigned to this order
   - Validate trip has started

4. **Firebase Write**
   ```
   Path: /car_tracking/{order_id}
   Data: {
     car_number: string,
     latitude: float,
     longitude: float,
     timestamp: int,
     driver_id: string,
     accuracy: float
   }
   ```

5. **Redis Caching**
   - Key: `car_location:{order_id}`
   - Hash fields:
     - `CAR_LOCATION_RESPONSE`: Current location
     - `CAR_LOCATION_IS_SHARE`: Share trip status
   - TTL: 24 hours

6. **Real-Time Sync to Customer**
   - Customer app subscribes to Firebase path
   - Receives instant updates when location changes
   - Updates map marker in real-time

7. **Location Change Detection**
   - Only write to Firebase if significant change (>10 meters)
   - Reduce unnecessary writes
   - Battery optimization

---

## 2️⃣ Nearby Cars Discovery Workflow

**Scenario**: User membuka app, perlu lihat available cars di sekitar

### Sequence Diagram

```mermaid
sequenceDiagram
    participant CustomerApp
    participant TrackerService
    participant Redis
    participant GbGateway
    participant TaxiPartner
    participant LocationCalc

    CustomerApp->>TrackerService: GetNearbyCars(location, service_type)
    TrackerService->>TrackerService: InputValidator
    
    TrackerService->>Redis: Check cache (nearby_cars:{lat}_{lng}_{type})
    
    alt Cache Hit
        Redis-->>TrackerService: Cached cars list
        TrackerService-->>CustomerApp: Return cached results
    else Cache Miss
        par Parallel Queries
            TrackerService->>GbGateway: GetAvailableCars(location, service_type)
            GbGateway-->>TrackerService: List of GB cars
        and
            TrackerService->>TaxiPartner: GetAvailableCars(location, service_type)
            TaxiPartner-->>TrackerService: List of partner cars
        end
        
        TrackerService->>TrackerService: Merge results
        
        TrackerService->>LocationCalc: Calculate distances
        LocationCalc-->>TrackerService: Distances calculated
        
        TrackerService->>TrackerService: Filter by radius (200m)
        TrackerService->>TrackerService: Sort by distance
        TrackerService->>TrackerService: Apply limit (10 cars)
        
        TrackerService->>Redis: Cache results (TTL: 30s)
        TrackerService-->>CustomerApp: Return nearby cars
        
        CustomerApp->>CustomerApp: Display cars on map
    end
```

### Key Steps

1. **User Location**
   - Get user's current GPS location
   - Request: latitude, longitude, service_type_id

2. **Cache Check**
   ```
   Key: nearby_cars:{lat_rounded}:{lng_rounded}:{service_type}
   Precision: 4 decimal places (±11m accuracy)
   TTL: 30 seconds
   ```

3. **Parallel Data Fetching**
   
   **From Goldenbird Gateway**:
   ```go
   gbCars := gbGateway.GetAvailableCars(ctx, &GetAvailableCarsRequest{
       Location: location,
       ServiceType: serviceType,
       Radius: config.NearbyCarsRadiusGB, // 200m
   })
   ```
   
   **From Taxi Partner Gateway**:
   ```go
   partnerCars := taxiPartner.GetAvailableCars(ctx, &GetAvailableCarsRequest{
       Location: location,
       ServiceType: serviceType,
       Status: "available",
   })
   ```

4. **Merge & Deduplicate**
   - Combine results from both sources
   - Remove duplicates by car_number
   - Tag with `taxi_category` (goldenbird/partner)

5. **Distance Calculation**
   ```go
   func calculateDistance(lat1, lng1, lat2, lng2 float64) float64 {
       // Haversine formula
       R := 6371.0 // Earth radius in km
       dLat := toRadians(lat2 - lat1)
       dLng := toRadians(lng2 - lng1)
       
       a := math.Sin(dLat/2) * math.Sin(dLat/2) +
            math.Cos(toRadians(lat1)) * math.Cos(toRadians(lat2)) *
            math.Sin(dLng/2) * math.Sin(dLng/2)
       
       c := 2 * math.Atan2(math.Sqrt(a), math.Sqrt(1-a))
       return R * c * 1000 // Convert to meters
   }
   ```

6. **Filtering & Sorting**
   ```go
   // Filter by radius
   filteredCars := filterByRadius(mergedCars, userLocation, 200.0)
   
   // Sort by distance (nearest first)
   sort.Slice(filteredCars, func(i, j int) bool {
       return filteredCars[i].Distance < filteredCars[j].Distance
   })
   
   // Apply limit
   if len(filteredCars) > 10 {
       filteredCars = filteredCars[:10]
   }
   ```

7. **Cache Result**
   - Store in Redis for 30 seconds
   - Next request within 30s returns cached data
   - Reduce load on GB Gateway & Taxi Partner

8. **Response to Client**
   ```json
   {
     "cars": [
       {
         "taxi_category": "goldenbird",
         "number": "B1234XYZ",
         "location": {"latitude": -6.2089, "longitude": 106.8457}
       }
     ]
   }
   ```

---

## 3️⃣ Share Trip Workflow

**Scenario**: User ingin share trip status ke family/friends

### Sequence Diagram

```mermaid
sequenceDiagram
    participant CustomerApp
    participant TrackerService
    participant Redis
    participant TaskManager
    participant OrderQuery
    participant ShareTripPage
    participant ExternalUser

    CustomerApp->>TrackerService: ShareTrip(order_id, trip_id)
    TrackerService->>TrackerService: Validate order active
    
    TrackerService->>TrackerService: Generate unique token (UUID)
    
    TrackerService->>Redis: Store share session
    Note over Redis: Key: share_trip:{token}<br/>Value: {order_id, trip_id, created_at}
    
    TrackerService->>TaskManager: Queue parse job
    TrackerService-->>CustomerApp: Return shareable URL
    
    Note over CustomerApp: User shares URL via WhatsApp/SMS
    
    par Background Processing
        TaskManager->>OrderQuery: Get order details
        OrderQuery-->>TaskManager: Order data
        
        TaskManager->>TaskManager: Parse order info
        TaskManager->>TaskManager: Generate public view data
        
        TaskManager->>Redis: Cache parsed data
        Note over Redis: Key: share_trip_data:{token}<br/>TTL: Until trip ends
    end
    
    ExternalUser->>ShareTripPage: Access URL (no auth required)
    ShareTripPage->>Redis: Get share session
    ShareTripPage->>Redis: Get parsed order data
    ShareTripPage->>Firebase: Subscribe to location updates
    
    loop Real-time updates
        Firebase-->>ShareTripPage: Location changed
        ShareTripPage->>ShareTripPage: Update map
    end
    
    Note over TrackerService: Trip ended
    TrackerService->>Redis: Expire share session
    ShareTripPage-->>ExternalUser: Show "Trip Completed"
```

### Key Steps

1. **Share Request**
   ```go
   request := &ShareTripRequest{
       OrderID: "ORDER123",
       TripID: "TRIP456",
   }
   ```

2. **Token Generation**
   ```go
   token := uuid.New().String()
   // Example: "550e8400-e29b-41d4-a716-446655440000"
   ```

3. **Session Storage**
   ```
   Key: share_trip:{token}
   Value: {
     "order_id": "ORDER123",
     "trip_id": "TRIP456",
     "created_at": 1706246400,
     "expires_at": 1706332800
   }
   TTL: Until trip ends (max 24h)
   ```

4. **URL Generation**
   ```
   Format: https://share.mybluebird.id/trip/{token}
   Example: https://share.mybluebird.id/trip/550e8400-e29b-41d4-a716-446655440000
   ```

5. **Background Parsing Job**
   - Queued via Task Manager
   - Priority: Low (non-blocking)
   - Timeout: 30 seconds

6. **Order Details Fetching**
   ```go
   orderDetails := orderQuery.GetOrderDetails(ctx, orderID)
   
   parsedData := ShareTripData{
       DriverName: orderDetails.DriverName,
       DriverPhoto: orderDetails.DriverPhoto,
       CarNumber: orderDetails.CarNumber,
       CarType: orderDetails.CarType,
       PickupAddress: orderDetails.PickupAddress,
       DropoffAddress: orderDetails.DropoffAddress,
       EstimatedArrival: orderDetails.ETA,
       TripStatus: "in_progress",
   }
   ```

7. **Public View Cache**
   ```
   Key: share_trip_data:{token}
   Value: Parsed order data (JSON)
   TTL: Until trip ends
   ```

8. **External User Access**
   - No authentication required
   - Access via unique token
   - Real-time location updates via Firebase listener

9. **Public View Features**
   - Driver info (name, photo, car details)
   - Real-time location on map
   - Trip route visualization
   - ETA updates
   - Pickup & dropoff markers

10. **Session Expiration**
    - Auto-expire when trip ends
    - Manual expire after 24 hours
    - Token becomes invalid after expiration

---

## 4️⃣ Order State Changes Workflow

**Scenario**: Order state berubah, tracking perlu di-update

### Sequence Diagram

```mermaid
sequenceDiagram
    participant OrderService
    participant PubSub
    participant TrackerService
    participant Firebase
    participant Redis
    participant PaymentProcessor

    OrderService->>PubSub: Publish order state change
    PubSub->>TrackerService: OrderChanges event
    
    TrackerService->>TrackerService: Parse order state
    
    alt State: TRIP_STARTED (4)
        TrackerService->>Firebase: Initialize tracking session
        TrackerService->>Redis: Set tracking_active = true
        TrackerService-->>PubSub: ACK
        
    else State: TRIP_ENDED (5)
        TrackerService->>TrackerService: Trigger end trip process
        
        TrackerService->>Firebase: Get final location
        Firebase-->>TrackerService: Last known location
        
        TrackerService->>Firebase: Set tracking_active = false
        TrackerService->>Firebase: Store final location
        
        TrackerService->>PaymentProcessor: Get payment status
        PaymentProcessor-->>TrackerService: Payment details
        
        TrackerService->>Redis: Store trip summary
        TrackerService->>Redis: Clear location cache
        TrackerService->>Redis: Expire share trip session
        
        TrackerService-->>PubSub: ACK
        
    else State: CANCELLED (8)
        TrackerService->>Firebase: Set status = cancelled
        TrackerService->>Redis: Clear all tracking data
        TrackerService->>Redis: Expire share session
        TrackerService-->>PubSub: ACK
        
    else Other States
        TrackerService->>Redis: Update order state
        TrackerService-->>PubSub: ACK
    end
```

### Key Steps

1. **PubSub Event Reception**
   ```go
   message := &OrderChangesReq{
       OrderState: 5,
       TripEnded: true,
       OrderID: 12345,
       UserID: "CUST123",
       PaymentMethod: "epay",
       FinalEpay: "150000",
   }
   ```

2. **State Parsing**
   ```go
   switch message.OrderState {
   case ORDER_CREATED:
       // Initialize order tracking
   case DRIVER_ASSIGNED:
       // Start monitoring driver availability
   case TRIP_STARTED:
       // Begin real-time tracking
   case TRIP_ENDED:
       // End trip process
   case CANCELLED:
       // Cleanup tracking data
   }
   ```

3. **Trip Started Handler**
   ```go
   func handleTripStarted(orderID int64) {
       // Initialize Firebase tracking path
       firebase.Set(ctx, fmt.Sprintf("/car_tracking/%d", orderID), map[string]interface{}{
           "tracking_active": true,
           "started_at": time.Now().Unix(),
       })
       
       // Set Redis flag
       redis.HSet(ctx, fmt.Sprintf("car_location:%d", orderID), 
           "tracking_active", true)
   }
   ```

4. **Trip Ended Handler**
   ```go
   func handleTripEnded(orderID int64) error {
       // Get final location from Firebase
       finalLocation := firebase.Get(ctx, fmt.Sprintf("/car_tracking/%d", orderID))
       
       // Stop tracking
       firebase.Set(ctx, fmt.Sprintf("/car_tracking/%d/tracking_active", orderID), false)
       firebase.Set(ctx, fmt.Sprintf("/car_tracking/%d/final_location", orderID), finalLocation)
       
       // Get payment status
       paymentStatus := paymentProcessor.GetStatus(ctx, orderID)
       
       // Store trip summary
       tripSummary := TripSummary{
           OrderID: orderID,
           FinalLocation: finalLocation,
           EndTime: time.Now(),
           PaymentStatus: paymentStatus,
       }
       redis.Set(ctx, fmt.Sprintf("trip_summary:%d", orderID), tripSummary, 24*time.Hour)
       
       // Clear real-time tracking cache
       redis.Del(ctx, fmt.Sprintf("car_location:%d", orderID))
       
       // Expire share trip session
       shareToken := getShareToken(orderID)
       if shareToken != "" {
           redis.Expire(ctx, fmt.Sprintf("share_trip:%s", shareToken), 5*time.Minute)
       }
       
       return nil
   }
   ```

5. **Cancelled Handler**
   ```go
   func handleCancelled(orderID int64) {
       // Update Firebase status
       firebase.Set(ctx, fmt.Sprintf("/car_tracking/%d/status", orderID), "cancelled")
       
       // Clear all tracking data
       redis.Del(ctx, 
           fmt.Sprintf("car_location:%d", orderID),
           fmt.Sprintf("order_change:%d", orderID))
       
       // Expire share session immediately
       shareToken := getShareToken(orderID)
       if shareToken != "" {
           redis.Del(ctx, fmt.Sprintf("share_trip:%s", shareToken))
       }
   }
   ```

6. **Payment Status Sync**
   - Query Payment Processor untuk final payment status
   - Update trip summary dengan payment details
   - Handle insufficient balance scenarios

7. **Cleanup Process**
   - Remove real-time tracking data from Redis
   - Keep trip summary for 24 hours (analytics)
   - Archive to cold storage if needed

---

## 5️⃣ Background Task Manager Workflow

**Scenario**: Background jobs untuk parsing, cleanup, monitoring

### Sequence Diagram

```mermaid
sequenceDiagram
    participant TaskManager
    participant ShareTripParser
    participant Redis
    participant OrderQuery
    participant Firebase

    Note over TaskManager: Service started
    TaskManager->>TaskManager: Initialize worker pool
    
    loop Every 1 minute
        TaskManager->>Redis: Get pending share trip jobs
        Redis-->>TaskManager: Job queue
        
        par Process Jobs (parallel workers)
            TaskManager->>ShareTripParser: Process job 1
            ShareTripParser->>OrderQuery: Get order details
            ShareTripParser->>Redis: Cache parsed data
            ShareTripParser-->>TaskManager: Job complete
        and
            TaskManager->>ShareTripParser: Process job 2
            ShareTripParser->>OrderQuery: Get order details
            ShareTripParser->>Redis: Cache parsed data
            ShareTripParser-->>TaskManager: Job complete
        end
    end
    
    loop Every 5 minutes
        TaskManager->>Redis: Get expired share sessions
        Redis-->>TaskManager: Expired sessions list
        TaskManager->>Redis: Cleanup expired data
        TaskManager->>Firebase: Archive tracking data
    end
    
    Note over TaskManager: Graceful shutdown signal
    TaskManager->>TaskManager: Stop accepting new jobs
    TaskManager->>TaskManager: Wait for running jobs (max 30s)
    TaskManager->>TaskManager: Shutdown complete
```

### Key Components

1. **Share Trip Parser Worker**
   ```go
   type ShareTripParserWorker struct {
       redis       RedisRepository
       orderQuery  OrderQuery
       workerCount int
   }
   
   func (w *ShareTripParserWorker) Start(ctx context.Context) {
       for i := 0; i < w.workerCount; i++ {
           go w.worker(ctx, i)
       }
   }
   
   func (w *ShareTripParserWorker) worker(ctx context.Context, id int) {
       ticker := time.NewTicker(1 * time.Minute)
       defer ticker.Stop()
       
       for {
           select {
           case <-ticker.C:
               w.processJobs(ctx)
           case <-ctx.Done():
               return
           }
       }
   }
   ```

2. **Job Processing**
   ```go
   func (w *ShareTripParserWorker) processJobs(ctx context.Context) {
       jobs := w.redis.GetPendingJobs(ctx, "share_trip_parse")
       
       for _, job := range jobs {
           // Parse order details
           orderDetails := w.orderQuery.GetOrderDetails(ctx, job.OrderID)
           
           // Transform to public view data
           publicData := transformToPublicData(orderDetails)
           
           // Cache parsed data
           w.redis.Set(ctx, 
               fmt.Sprintf("share_trip_data:%s", job.Token),
               publicData,
               24*time.Hour)
           
           // Mark job complete
           w.redis.RemoveJob(ctx, "share_trip_parse", job.ID)
       }
   }
   ```

3. **Cleanup Task**
   ```go
   func (tm *TaskManager) cleanupExpiredSessions(ctx context.Context) {
       // Get expired share sessions
       expiredSessions := tm.redis.GetExpiredSessions(ctx, "share_trip")
       
       for _, session := range expiredSessions {
           // Remove session data
           tm.redis.Del(ctx, fmt.Sprintf("share_trip:%s", session.Token))
           
           // Remove parsed data
           tm.redis.Del(ctx, fmt.Sprintf("share_trip_data:%s", session.Token))
           
           // Archive to Firebase (optional)
           tm.firebase.Archive(ctx, session)
       }
   }
   ```

4. **Graceful Shutdown**
   ```go
   func (tm *TaskManager) ShutDown(ctx context.Context) error {
       // Stop accepting new jobs
       tm.stopAcceptingJobs()
       
       // Wait for running jobs with timeout
       shutdownCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
       defer cancel()
       
       done := make(chan struct{})
       go func() {
           tm.waitForJobs()
           close(done)
       }()
       
       select {
       case <-done:
           return nil
       case <-shutdownCtx.Done():
           return errors.New("shutdown timeout")
       }
   }
   ```

---

## 📊 Workflow Comparison Matrix

| Workflow | Trigger | Frequency | External Calls | Caching |
|----------|---------|-----------|----------------|---------|
| **Car Tracking** | Driver app | Every 20s | Firebase, Redis | ✅ 24h |
| **Nearby Cars** | User action | On demand | GbGateway, TaxiPartner | ✅ 30s |
| **Share Trip** | User action | Once | OrderQuery, Redis | ✅ Until end |
| **Order Changes** | PubSub event | Real-time | PaymentProcessor | ✅ 24h |
| **Task Manager** | Background | Every 1-5min | OrderQuery, Firebase | ❌ |

---

## 🔒 Performance Optimizations

### Caching Strategy
| Data Type | TTL | Key Pattern |
|-----------|-----|-------------|
| Car Location | 24 hours | `car_location:{order_id}` |
| Nearby Cars | 30 seconds | `nearby_cars:{lat}_{lng}_{type}` |
| Share Session | Until trip ends | `share_trip:{token}` |
| Trip Summary | 24 hours | `trip_summary:{order_id}` |

### Firebase Optimization
- Write only on significant location changes (>10m)
- Use delta updates instead of full writes
- Index optimization for query performance
- Batch writes when possible

### Background Processing
- Worker pool untuk parallel job processing
- Job queue dengan priority levels
- Graceful shutdown dengan timeout protection

---

## 🏷️ Tags

#trackerservice #workflow #architecture #mrg

---

*Last Updated*: 2025-01-26
