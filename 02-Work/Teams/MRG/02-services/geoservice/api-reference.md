---
tags:
  - api
  - grpc
  - geoservice
  - mrg
  - reference
type: api-documentation
title: Geo Service - API Reference
parent: geoservice
---
# Geo Service - API Reference

**Service**: [[README|Geo Service]]  
**Type**: API Documentation

---

## 📡 Server Configuration

| Protocol | Port | Default |
|----------|------|---------|
| gRPC | `GRPC_PORT` | 6000 |
| REST (gRPC-Gateway) | `REST_PORT` | 8000 |

---

## 🔧 gRPC Methods

### Health & Utility

#### HealthCheck
```protobuf
rpc HealthCheck(EmptyRequest) returns (DefaultMessage);
```
- **Description**: Service health check
- **Output**: `{code: string, message: string}`

---

### Reverse Geocoding

#### ReverseGeocodeV2 (Legacy)
```protobuf
rpc ReverseGeocodeV2(ReverseGeocodeV2Request) returns (ReverseGeocodeV2Response);
```
- **Interceptors**: InputValidator, MetadataValidator
- **Input**:
  ```protobuf
  message ReverseGeocodeV2Request {
    string subplace_type = 1;      // Type of subplace filter
    string latlong = 2;            // "lat,long" format
    string trip_id = 3;            // Trip identifier
    string language = 4;           // Response language (en/id)
    bool disabled_auto_snap = 5;   // Disable auto snap feature
  }
  ```
- **Output**:
  ```protobuf
  message ReverseGeocodeV2Response {
    ReverseGeocodeV2 data = 1;
  }
  
  message ReverseGeocodeV2 {
    string id = 1;
    string external_id = 2;
    string name = 3;
    ReverseGeocodeV2_Coordinate coordinate = 4;
    string address = 5;
    bool subplace_required = 6;
    repeated ReverseGeocodeV2_Subplace subplaces = 7;
    repeated ReverseGeocodeV2_Coordinate polygon = 8;
    ReverseGeocodeV2_FavoriteAddress favorite = 9;
    repeated ReverseGeocodeV2_FavoriteAddress all_favorites = 10;
    string auto_snap_result = 11;
    string place_category = 12;
    double radius_from_current_location_in_meters = 13;
  }
  ```

#### ReverseGeocodeV3 (Latest)
```protobuf
rpc ReverseGeocodeV3(ReverseGeocodeV3Request) returns (ReverseGeocodeV3Response);
```
- **Interceptors**: InputValidator
- **Input**:
  ```protobuf
  message ReverseGeocodeV3Request {
    string type = 1;           // "pickup" or "dropoff"
    double latitude = 2;
    double longitude = 3;
    string trip_id = 4;
    string place_category = 5; // Optional category filter
  }
  ```
- **Output**:
  ```protobuf
  message ReverseGeocodeV3Response {
    string id = 1;
    string external_id = 2;
    string name = 3;
    ReverseGeocodeV3Coordinate coordinate = 4;
    string address = 5;
    bool subplace_required = 6;
    repeated ReverseGeocodeV3Subplace subplaces = 7;
    repeated ReverseGeocodeV3Coordinate polygon = 8;
    ReverseGeocodeV3FavoriteAddress favorite = 9;
    string place_category = 10;
    double radius_from_current_location_in_meters = 11;
    string timezone = 12;                                    // NEW: Timezone info
    repeated ReverseGeocodeV3GoldenbirdArea goldenbird_areas = 13; // NEW: Area zones
  }
  ```

**Key Differences V2 vs V3**:
- V3 uses separate latitude/longitude instead of "lat,long" string
- V3 returns timezone information
- V3 includes goldenbird_areas for operational zone mapping
- V3 uses cleaner type parameter ("pickup"/"dropoff")

---

### Location Search & Suggestions

#### AutoComplete
```protobuf
rpc AutoComplete(AutoCompleteRequest) returns (AutoCompleteResponse);
```
- **Interceptors**: InputValidator
- **Input**:
  ```protobuf
  enum PopularPlaceType {
    CITIES = 0;
    LOCATION = 1;
    POI = 2;
  }
  
  message AutoCompleteRequest {
    string type = 1;                      // "pickup" or "dropoff"
    double latitude = 2;                  // User current position
    double longitude = 3;
    string keyword = 4;                   // Search keyword
    string trip_id = 5;
    string place_category = 6;            // Optional category filter
    bool include_city = 7;                // Include city-level results
    bool show_popular_place = 8;          // Include popular places
    PopularPlaceType popular_place_type = 9;
    bool area_only = 10;                  // Return only area-type locations
  }
  ```
- **Output**:
  ```protobuf
  message AutoCompleteResponse {
    repeated AutoCompleteData autocomplete = 1;
  }
  
  message AutoCompleteData {
    string id = 1;
    string external_id = 2;
    repeated AutoCompleteGoldenbirdArea goldenbird_areas = 3;
    string name = 4;
    AutoCompleteDataCoordinate coordinate = 5;
    string address = 6;
    string place_category = 7;
    bool subplace_required = 8;
    string icon = 9;
    repeated AutoCompleteDataSubplace subplaces = 10;
    repeated AutoCompleteDataCoordinate polygon = 11;
    AutoCompleteDataFavoriteAddress favorite = 12;
    double radius_from_current_location_in_meters = 13;
    bool is_city = 14;
  }
  ```

**Use Cases**:
- Search "Bandara" → Returns airports with distance from user
- Search "Mall" → Returns nearby malls + popular shopping centers
- Search "Rumah" → Returns saved favorite addresses matching "Rumah"

#### HistoryPlace
```protobuf
rpc HistoryPlace(HistoryPlaceRequest) returns (HistoryPlaceResponse);
```
- **Interceptors**: InputValidator
- **Input**:
  ```protobuf
  message HistoryPlaceRequest {
    string type = 1;        // "pickup" or "dropoff"
    double latitude = 2;    // User current location
    double longitude = 3;
    string trip_id = 4;
  }
  ```
- **Output**:
  ```protobuf
  message HistoryPlaceResponse {
    repeated HistoryPlaceData history_places = 1;
  }
  
  message HistoryPlaceData {
    string id = 1;
    string external_id = 2;
    repeated HistoryPlaceGoldenbirdArea goldenbird_areas = 3;
    string name = 4;
    HistoryPlaceDataCoordinate coordinate = 5;
    string address = 6;
    string place_category = 7;
    bool subplace_required = 8;
    repeated HistoryPlaceDataSubplace subplaces = 9;
    repeated HistoryPlaceDataCoordinate polygon = 10;
    HistoryPlaceDataFavoriteAddress favorite = 11;
    double radius_from_current_location_in_meters = 12;
  }
  ```

**Business Logic**:
- Returns user's frequently visited places
- Based on trip history analysis
- Sorted by visit frequency & recency
- Limited by `HISTORY_PLACE_LIMIT` config (default: 2)

---

### Background Jobs & Event Listeners

#### OrderStateChangesListener
```protobuf
rpc OrderStateChangesListener(OrderStateChangeRequest) returns (DefaultMessage);
```
- **Description**: Listen to order state changes untuk track frequent locations
- **Input**:
  ```protobuf
  message OrderStateChangeRequest {
    int32 state = 1;
    string state_name = 2;
    OrderData data = 3;
  }
  
  message OrderData {
    RequestedData requested = 1;
    ActualData actual = 2;
    CustomerInfo customer_info = 3;
    VehicleInfo vehicle_info = 4;
    Payment payment = 5;
    string booking_type = 6;
    string trip_id = 7;
    string picked_up_at = 8;
    string dropoff_at = 9;
    int32 order_id = 10;
  }
  
  message RequestedData {
    TripLocation pickup = 1;
    TripLocation dropoff = 2;
  }
  
  message ActualData {
    TripLocation pickup = 1;
    TripLocation dropoff = 2;
  }
  
  message TripLocation {
    Coordinate coordinate = 1;
    string address = 2;
  }
  ```

**Trigger**: Message broker (via geo.broker.yaml config)
**States Tracked**: Completed orders untuk frequent location analysis

#### RemoveExpiredFrequentLocation
```protobuf
rpc RemoveExpiredFrequentLocation(EmptyRequest) returns (RemoveExpiredFrequentLocationResponse);
```
- **Description**: Cleanup expired frequent location entries
- **Output**:
  ```protobuf
  message RemoveExpiredFrequentLocationResponse {
    int32 total_deleted = 1;
  }
  ```

**Trigger**: Scheduled cron job or manual call
**Logic**: Remove entries older than `FREQUENT_LOCATION_LOOKBACK_PERIOD_DAYS`

---

## 📝 Common Types

### Coordinate
```protobuf
message Coordinate {
  double latitude = 1;
  double longitude = 2;
}
```

### FavoriteAddress
```protobuf
message FavoriteAddress {
  string id = 1;
  string driver_notes = 2;
  double latitude = 3;
  double longitude = 4;
  string address_short_name = 5;
  string address_long_name = 6;
  double radius_from_current_location_in_meters = 7;
}
```

### Subplace
```protobuf
message Subplace {
  string id = 1;
  string label = 2;
  string name = 3;
  bool selected = 4;
  Coordinate coordinate = 5;
  FavoriteAddress favorite = 6;
  double radius_from_current_location_in_meters = 7;
}
```

### GoldenbirdArea
```protobuf
message GoldenbirdArea {
  string id = 1;
  string name = 2;
  string zone_code = 3;
  string zone_number = 4;
  string city_code = 5;
  string city_name = 6;
}
```

---

## 🎯 Request/Response Examples

### Example 1: Reverse Geocode V3
**Request**:
```json
{
  "type": "pickup",
  "latitude": -6.2088,
  "longitude": 106.8456,
  "trip_id": "TRIP-12345"
}
```

**Response**:
```json
{
  "id": "place_123",
  "external_id": "ChIJyWEHuEMzaS4RwGkY0F0D",
  "name": "Gedung Sarinah",
  "coordinate": {
    "latitude": -6.2088,
    "longitude": 106.8456
  },
  "address": "Jl. M.H. Thamrin No.11, Jakarta Pusat",
  "subplace_required": true,
  "subplaces": [
    {
      "id": "sub_1",
      "label": "Main Entrance",
      "name": "Pintu Utama",
      "selected": true,
      "coordinate": {
        "latitude": -6.2087,
        "longitude": 106.8455
      }
    }
  ],
  "place_category": "mall",
  "timezone": "Asia/Jakarta",
  "goldenbird_areas": [
    {
      "id": "area_1",
      "name": "Jakarta Pusat",
      "zone_code": "JKT",
      "zone_number": "01",
      "city_code": "31",
      "city_name": "DKI Jakarta"
    }
  ]
}
```

### Example 2: AutoComplete
**Request**:
```json
{
  "type": "pickup",
  "latitude": -6.2088,
  "longitude": 106.8456,
  "keyword": "Bandara",
  "trip_id": "TRIP-12345",
  "show_popular_place": true,
  "popular_place_type": "POI"
}
```

**Response**:
```json
{
  "autocomplete": [
    {
      "id": "place_cgk",
      "external_id": "ChIJV0_CGK_faS4RsCE",
      "name": "Bandar Udara Internasional Soekarno-Hatta",
      "coordinate": {
        "latitude": -6.1256,
        "longitude": 106.6558
      },
      "address": "Tangerang, Banten",
      "place_category": "airport",
      "icon": "airport_icon_url",
      "radius_from_current_location_in_meters": 28500,
      "goldenbird_areas": [...]
    },
    {
      "id": "place_hlp",
      "name": "Bandar Udara Halim Perdanakusuma",
      "coordinate": {
        "latitude": -6.2666,
        "longitude": 106.8906
      },
      "address": "Jakarta Timur",
      "place_category": "airport",
      "radius_from_current_location_in_meters": 8200
    }
  ]
}
```

---

## 🔀 Method Flow Diagrams

### ReverseGeocodeV3 Flow
```
Client Request
    ↓
InputValidator (validate lat/long)
    ↓
Check Redis Cache (key: lat_long_type)
    ↓ (cache miss)
Map Service API (reverse geocode)
    ↓
User Service (get favorite addresses in radius)
    ↓
Area Management (get goldenbird zones)
    ↓
Auto Snap (if enabled & landmark found)
    ↓
Build Response + Cache
    ↓
Return to Client
```

### AutoComplete Flow
```
Client Request (keyword: "Mall")
    ↓
InputValidator
    ↓
Parallel Queries:
  - Map Service (search "Mall")
  - User Service (favorite addresses matching "Mall")
  - Content Provider (popular malls if enabled)
    ↓
Merge Results
    ↓
Deduplicate by external_id
    ↓
Sort by Distance from User
    ↓
Enrich with Goldenbird Areas
    ↓
Return Top Results
```

---

## 🏷️ Tags

#api #grpc #geoservice #mrg #reference

---

*Last Updated*: 2025-01-26
