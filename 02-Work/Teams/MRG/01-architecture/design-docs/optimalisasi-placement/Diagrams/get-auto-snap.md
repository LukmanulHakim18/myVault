---
title: Get Auto Snap Diagram
type: diagram
diagram_type: sequence
tags:
  - diagram
  - plantuml
  - autosnap
---
# Get Auto Snap - Sequence Diagram

**Parent**: [[README|Optimalisasi Placement untuk Tim MRG]]  
**Type**: PlantUML Diagram

---

## Diagram

```plantuml
@startuml

title **Get auto snap**

actor       MobileUser   as MU
boundary    APIGateway   as AG
control     AutoSnapUsecase as UC
entity      FavoriteRepo as FR
entity      FrequencyRepo as FQR
entity      PopularRepo  as PR
entity      MapService  as MAP
database    PostGISDB    as DB

MU -> AG : Request /autosnap (lat, lng, user_id, type)
   AG -> UC : Validate & process request
      UC -> FR : Get favorite locations (radius 100m)
         FR -> DB : SELECT ... FROM favorite_locations WHERE ...
         DB --> FR : Result
      FR --> UC : Favorite locations

      UC ----> UC : Append data favorite locations to list result location

      UC -> FQR : Get frequent locations (radius 100m)
         FQR -> DB : SELECT ... FROM user_location_frequency WHERE ...
         DB --> FQR : Result
      FQR --> UC : Frequent locations

      UC ----> UC : Append data frequent locations to list result location

alt Data list result location not nil
      UC ----> UC : select based on periority
   UC -> AG : Build ordered list of recommendations
AG --> MU : Return recommendation list
end
      UC -> MAP : Refers geo code
      MAP --> UC : Popular location

   UC -> AG : Build ordered list of recommendations
AG --> MU : Return recommendation list

@enduml
```

---

## Flow Description

### Request
- **Endpoint**: `/autosnap`
- **Parameters**: `lat`, `lng`, `user_id`, `type`

### Processing Steps

1. **API Gateway** validates dan forwards request ke AutoSnap Usecase
2. **FavoriteRepo** fetch favorite locations dalam radius 100m
3. **FrequencyRepo** fetch frequent locations dalam radius 100m
4. **Append** kedua hasil ke result list
5. **Jika ada data**: Select berdasarkan prioritas dan return
6. **Jika tidak ada**: Fallback ke MapService untuk geocode

### Priority Order
1. Favorite locations
2. Frequent locations
3. Popular locations (from geocode)

---

## 🏷️ Tags

#diagram #plantuml #sequence #autosnap #getter

---

*Last Updated*: 2025-01-05
