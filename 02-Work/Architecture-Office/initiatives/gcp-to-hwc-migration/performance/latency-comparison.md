# Performance Analysis: GCP vs Huawei Cloud Latency Comparison

**Status**: 🔴 Critical - Performance Degradation Detected  
**Analysis Date**: 2025-08-04  
**Analyzed By**: Lukmanul Hakim + Infrastructure Team

---

## 📊 Executive Summary

Post-migration performance analysis reveals **significant latency degradation** across multiple API endpoints when comparing Huawei Cloud (HWC) against the previous Google Cloud Platform (GCP) infrastructure.

### Key Findings
- **Average Degradation**: 2.8x to 42.5x slower
- **Worst Performer**: User favorite addresses endpoint (42.5x slower)
- **Best Performer**: Fleet list endpoint (2.8x slower, but still slow)
- **P95 Latency**: Several endpoints exceeding 1 second

### Severity Assessment
🔴 **Critical** - User experience significantly impacted

---

## 📈 Detailed Latency Comparison

### Summary Table

| Endpoint | GCP (ms) | HWC (ms) | Degradation | P95 HWC (ms) | Status |
|----------|----------|----------|-------------|--------------|--------|
| Navigation/User State | 17 | 472 | **27.8x** ⚠️ | 4xx | 🔴 Critical |
| Favorite Addresses | 23 | 977 | **42.5x** 🔴 | 446 | 🔴 Critical |
| Fleet List | 594 | 1,673 | **2.8x** ⚠️ | 2,606 | 🔴 Critical |
| Payment Methods | 140 | 502 | **3.6x** ⚠️ | 6xx | 🟡 High |
| User Profile (Me) | 53 | 440 | **8.3x** ⚠️ | 5xx | 🔴 Critical |

### Legend
- 🔴 **Critical**: > 10x degradation or P95 > 1000ms
- 🟡 **High**: 5-10x degradation or P95 500-1000ms
- ⚠️ **Moderate**: 2-5x degradation or P95 < 500ms

---

## 🔍 Endpoint-by-Endpoint Analysis

### 1. Navigation/User State
**Endpoint**: `GET /api/v6/navigation*user_state/:key_navigation`

| Metric | GCP | HWC | Change |
|--------|-----|-----|--------|
| Average Latency | 17ms | 472ms | +455ms ⬆️ |
| Degradation Factor | 1x | **27.8x** | 2,678% slower |
| P95 Latency | ~20-30ms | ~400ms | +370ms ⬆️ |

**Impact**:
- User navigation experience significantly affected
- Map loading delays
- Route calculation delays
- Poor responsiveness during trip planning

**Priority**: 🔴 **Critical** - Core navigation feature

**Potential Causes**:
- Database query performance
- Redis cache misses
- Network latency to external map services
- Service-to-service communication overhead

---

### 2. Favorite Addresses
**Endpoint**: `GET /api/user-service/v1/favorite-addresses`

| Metric | GCP | HWC | Change |
|--------|-----|-----|--------|
| Average Latency | 23ms | 977ms | +954ms ⬆️ |
| Degradation Factor | 1x | **42.5x** | 4,152% slower |
| P95 Latency | ~30-40ms | 446ms | +406ms ⬆️ |

**Impact**:
- Slow loading of user's saved addresses
- Booking flow delays
- Poor UX when selecting pickup/destination
- User frustration with app responsiveness

**Priority**: 🔴 **Critical** - Most degraded endpoint

**Potential Causes**:
- Database read performance (user data queries)
- Missing indexes on favorites table
- N+1 query problems
- No caching of favorite addresses
- Cross-region database calls

**Recommended Actions**:
1. Add Redis caching for favorite addresses
2. Review database query execution plans
3. Check for missing indexes
4. Consider materialized views
5. Profile query performance

---

### 3. Fleet List
**Endpoint**: `GET /api/v6/fleet_list/`

| Metric | GCP | HWC | Change |
|--------|-----|-----|--------|
| Average Latency | 594ms | 1,673ms | +1,079ms ⬆️ |
| Degradation Factor | 1x | **2.8x** | 182% slower |
| P95 Latency | ~800ms | 2,606ms | +1,806ms ⬆️ |

**Impact**:
- Slow vehicle availability display
- Delayed ETA calculations
- Poor user experience when booking
- Increased bounce rate

**Priority**: 🔴 **Critical** - P95 > 2.5 seconds unacceptable

**Notes**:
- Already slow on GCP (594ms baseline)
- Further degradation compounds the problem
- P95 approaching 3 seconds is very poor UX

**Potential Causes**:
- Complex geospatial queries (PostGIS)
- Large dataset scans
- No proper spatial indexing
- Real-time availability checks
- External BBD API calls

**Recommended Actions**:
1. Implement aggressive caching (5-10 second TTL)
2. Use spatial indexes properly (PostGIS GIST)
3. Paginate results
4. Pre-compute nearby fleet
5. Consider read replicas for fleet queries

---

### 4. Payment Methods
**Endpoint**: `GET /api/v6/me/payment_methods/?type=:id`

| Metric | GCP | HWC | Change |
|--------|-----|-----|--------|
| Average Latency | 140ms | 502ms | +362ms ⬆️ |
| Degradation Factor | 1x | **3.6x** | 259% slower |
| P95 Latency | ~200ms | ~600ms | +400ms ⬆️ |

**Impact**:
- Slow payment method selection
- Checkout flow delays
- User drop-off at payment stage

**Priority**: 🟡 **High** - Critical for conversion

**Potential Causes**:
- UPG service latency
- External payment gateway checks
- Token validation overhead
- Database query for payment methods
- Network latency to payment services

**Recommended Actions**:
1. Cache available payment methods
2. Lazy load payment method details
3. Optimize UPG API calls
4. Review payment gateway integration

---

### 5. User Profile (Me)
**Endpoint**: `GET /api/v6/me/`

| Metric | GCP | HWC | Change |
|--------|-----|-----|--------|
| Average Latency | 53ms | 440ms | +387ms ⬆️ |
| Degradation Factor | 1x | **8.3x** | 730% slower |
| P95 Latency | ~80ms | ~500ms | +420ms ⬆️ |

**Impact**:
- Slow app startup (profile loaded on launch)
- Delayed user authentication flow
- Poor initial app experience

**Priority**: 🔴 **Critical** - First impression endpoint

**Potential Causes**:
- Database query for user profile
- Multiple service calls to build profile
- Session validation overhead
- Missing user profile caching
- JWT token validation delays

**Recommended Actions**:
1. Implement aggressive user profile caching
2. Use Redis for session data
3. Reduce data returned in profile endpoint
4. Lazy load non-essential user data
5. Consider CDN for static user data

---

## 🎯 Root Cause Investigation

### Hypothesis Categories

#### 1. Network Latency 🌐
**Theory**: Geographic distance or routing issues

**Evidence**:
- Consistent degradation across all endpoints
- Similar degradation patterns regardless of complexity

**Investigation Needed**:
- Measure base network latency GCP ↔ HWC
- Check routing tables and peering arrangements
- Test inter-region latency
- Review CDN configuration

**Tests**:
```bash
# Network latency baseline
ping -c 100 hwc-endpoint.example.com
traceroute hwc-endpoint.example.com

# HTTP latency test
curl -w "@curl-format.txt" -o /dev/null -s https://hwc-endpoint.example.com/health
```

---

#### 2. Database Performance 🗄️
**Theory**: Database queries slower on HWC infrastructure

**Evidence**:
- Data-heavy endpoints most affected (favorites, fleet)
- Read queries particularly slow

**Investigation Needed**:
- Compare query execution plans (EXPLAIN ANALYZE)
- Check database server resources (CPU, Memory, I/O)
- Review connection pool settings
- Validate indexes exist and are used
- Check database server location/region

**Queries to Run**:
```sql
-- Check slow queries
SELECT * FROM pg_stat_statements 
WHERE mean_exec_time > 100 
ORDER BY mean_exec_time DESC 
LIMIT 20;

-- Check missing indexes
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
AND n_distinct > 100
ORDER BY n_distinct DESC;

-- Check table bloat
SELECT schemaname, tablename, 
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

#### 3. Redis/Caching Issues 🔴
**Theory**: Cache hit rate dropped or cache not working

**Evidence**:
- Endpoints that should be cached showing high latency
- Consistent slow performance (not intermittent)

**Investigation Needed**:
- Check Redis hit/miss ratio
- Verify cache keys being set correctly
- Confirm cache TTL values
- Test cache connectivity and latency
- Review cache eviction policies

**Redis Commands**:
```bash
# Check cache stats
redis-cli INFO stats | grep hit

# Check cache size
redis-cli INFO memory

# Monitor cache operations
redis-cli MONITOR

# Check specific keys
redis-cli --scan --pattern "user:*"
```

---

#### 4. Service Configuration ⚙️
**Theory**: Suboptimal service configuration on HWC

**Evidence**:
- All endpoints affected uniformly
- Degradation consistent across different service types

**Investigation Needed**:
- Compare service resource limits (CPU, memory)
- Check connection pool sizes
- Review timeout configurations
- Validate environment variables
- Check service mesh/load balancer config

**Configuration to Check**:
```yaml
# Database connection pool
max_connections: 100
idle_timeout: 30s
connection_lifetime: 3600s

# HTTP client timeouts
connect_timeout: 5s
request_timeout: 30s
idle_timeout: 90s

# Service resources
cpu_limit: "2000m"
memory_limit: "4Gi"
replicas: 3
```

---

#### 5. External Dependencies 🔌
**Theory**: Calls to external services slower from HWC

**Evidence**:
- Endpoints calling external APIs most affected
- Fleet list (BBD API) very slow

**Investigation Needed**:
- Test latency to external APIs from HWC
- Check firewall/security group rules
- Verify DNS resolution performance
- Review API gateway routing

---

## 📉 Performance Impact Analysis

### User Experience Impact

#### App Launch (< 2 seconds acceptable)
- GCP: Profile (53ms) + Payment (140ms) = **193ms** ✅ Excellent
- HWC: Profile (440ms) + Payment (502ms) = **942ms** ⚠️ Acceptable
- **Impact**: Still within acceptable range, but noticeably slower

#### Booking Flow (< 5 seconds acceptable)
- GCP: Favorites (23ms) + Fleet (594ms) + Payment (140ms) = **757ms** ✅ Excellent
- HWC: Favorites (977ms) + Fleet (1673ms) + Payment (502ms) = **3,152ms** 🔴 Poor
- **Impact**: 3+ seconds for booking flow is poor UX

#### Navigation (< 1 second acceptable)
- GCP: Navigation (17ms) ✅ Excellent
- HWC: Navigation (472ms) ⚠️ Acceptable but slow
- **Impact**: Noticeable delay in map interactions

---

### Business Impact Estimation

#### Conversion Rate Impact
Research shows:
- **100ms delay** = ~1% conversion drop
- **1 second delay** = ~7% conversion drop
- **3+ seconds** = ~40% bounce rate increase

**Estimated Impact on MyBluebird**:
- Booking flow 2.4 seconds slower → **~15-20% conversion drop risk**
- App startup ~450ms slower → **~3-5% engagement drop risk**

#### Monthly Impact (Estimated)
Assuming:
- 100,000 monthly active users
- 50,000 booking attempts per month
- Average booking value: 50,000 IDR

**Potential Revenue Impact**:
- 15% conversion drop on 50,000 bookings = 7,500 lost bookings
- 7,500 × 50,000 IDR = **375,000,000 IDR (~$25,000 USD) monthly loss**

---

## 🚨 Priority Action Plan

### Phase 1: Immediate (24-48 hours) 🔴

#### 1. Network Baseline Testing
**Owner**: Infrastructure Team + Huawei  
**Action**: Measure network latency between services
```bash
# From backend service to database
time curl -o /dev/null -s -w 'Total: %{time_total}s\n' https://db.hwc.internal/health

# From backend to Redis
redis-cli --latency -h redis.hwc.internal

# From backend to external APIs
time curl -o /dev/null -s -w 'Total: %{time_total}s\n' https://maps.googleapis.com/health
```

#### 2. Quick Wins - Caching
**Owner**: MRG & UPG Teams  
**Action**: Implement/fix caching for top 3 slow endpoints
- Favorite addresses: 5-minute cache
- User profile: 10-minute cache
- Payment methods: 5-minute cache

```go
// Example: Cache favorite addresses
func (s *UserService) GetFavoriteAddresses(userID string) ([]Address, error) {
    cacheKey := fmt.Sprintf("user:%s:favorites", userID)
    
    // Try cache first
    if cached, err := s.redis.Get(ctx, cacheKey).Result(); err == nil {
        var addresses []Address
        json.Unmarshal([]byte(cached), &addresses)
        return addresses, nil
    }
    
    // Cache miss, query database
    addresses, err := s.db.GetFavoriteAddresses(userID)
    if err != nil {
        return nil, err
    }
    
    // Set cache with 5-minute TTL
    data, _ := json.Marshal(addresses)
    s.redis.Set(ctx, cacheKey, data, 5*time.Minute)
    
    return addresses, nil
}
```

#### 3. Database Query Analysis
**Owner**: Database Team  
**Action**: Run EXPLAIN ANALYZE on slowest queries
```sql
-- Favorites query
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM favorite_addresses WHERE user_id = 'BB00215513';

-- Fleet query with spatial search
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM fleet 
WHERE ST_DWithin(location, ST_MakePoint(106.830, -6.201)::geography, 5000)
AND status = 'available';
```

---

### Phase 2: Short-term (1 week) 🟡

#### 1. Database Optimization
- Add missing indexes
- Optimize slow queries
- Review connection pool settings
- Consider read replicas for heavy-read endpoints

#### 2. Service Optimization
- Review and optimize service configurations
- Increase connection pool sizes if needed
- Tune timeouts and retry logic
- Profile application code for bottlenecks

#### 3. Load Testing
- Run load tests to identify breaking points
- Compare GCP vs HWC under load
- Identify resource constraints

#### 4. Monitoring Enhancement
- Set up detailed performance dashboards
- Add latency alerts (> 1 second)
- Track P50, P95, P99 metrics
- Monitor database query performance

---

### Phase 3: Medium-term (1 month) 🟢

#### 1. Architecture Review
- Consider CDN for static/cached content
- Evaluate database sharding if needed
- Review microservice communication patterns
- Consider async operations where appropriate

#### 2. Infrastructure Optimization
- Work with Huawei to optimize network routing
- Consider edge locations closer to users
- Evaluate database instance upgrades
- Review load balancer configuration

#### 3. Application Redesign
- Implement lazy loading strategies
- Reduce payload sizes
- Use GraphQL for flexible queries
- Consider Server-Sent Events for real-time data

---

## 📊 Monitoring & Metrics

### Key Metrics to Track

#### Latency Metrics
- **P50 (Median)**: Target < 100ms
- **P95**: Target < 500ms  
- **P99**: Target < 1000ms
- **Max**: Monitor for outliers

#### Success Metrics
- **Error Rate**: < 0.1%
- **Timeout Rate**: < 1%
- **Cache Hit Rate**: > 80%

#### Business Metrics
- **Booking Completion Rate**: Monitor for drops
- **User Retention**: Watch for churn
- **App Engagement**: Session duration, actions per session

### Alerting Rules

```yaml
# Example Prometheus alerting rules
groups:
  - name: api_latency
    rules:
      - alert: HighAPILatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High API latency detected"
          description: "P95 latency is {{ $value }}s for {{ $labels.endpoint }}"
      
      - alert: VeryHighAPILatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 2m
        labels:
          severity: emergency
        annotations:
          summary: "Very high API latency detected"
          description: "P95 latency is {{ $value }}s for {{ $labels.endpoint }}"
```

---

## 🔗 Related Documents

- [Main Migration README](../README.md)
- [Cut Over Simulation](../cut-over-simulation-30-july.md)
- [Infrastructure Architecture](../../architecture/)

---

## 📞 Contacts

**Performance Investigation**:
- **Lead**: Hudan (Infrastructure)
- **Support**: Huawei Infrastructure Team
- **Escalation**: Lukmanul Hakim

**Database Optimization**:
- **Lead**: Database Team
- **Support**: MRG & UPG Backend Engineers

---

## 🏷️ Tags

#performance #latency #optimization #critical #hwc #gcp #comparison

---

*Last Updated: 2025-08-04*  
*Status: Investigation Ongoing*  
*Priority: 🔴 Critical*  
*Next Review: Daily until resolved*
