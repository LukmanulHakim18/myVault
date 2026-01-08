---
title: Setter Frequency Diagram
type: diagram
diagram_type: sequence
tags:
  - diagram
  - plantuml
  - setter
  - frequency
---
# Set Frequency Visited Point - Sequence Diagram

**Parent**: [[README|Optimalisasi Placement untuk Tim MRG]]  
**Type**: PlantUML Diagram

---

## Diagram

```plantuml
@startuml

title **Set frequency visited point**

queue KAFKA
control MapService
entity UserFrequencyRepo
entity GlobalFrequencyRepo
control GMO
database PostGISDB
database Redis


KAFKA -> MapService : Consume "endtrip" message

    MapService -> UserFrequencyRepo: Check is pickup frequency?
        UserFrequencyRepo ->  PostGISDB : Get to DB
        return  Data frequency pickup user
    MapService <- UserFrequencyRepo : Data frequency pickup user

    alt Data frequency != nil
        MapService -> UserFrequencyRepo: Update valid = time.now() + xTime
            UserFrequencyRepo ->  PostGISDB : Update valid = time.now() + xTime
            return ok
        MapService <- UserFrequencyRepo : ok
KAFKA <- MapService : Ack
    end 

    MapService -> Redis : Check user location history (ST_DWithin <= X meter)
    Redis --> MapService : Matching user pickup points
    alt Point not match
        MapService -> Redis : GEOADD
        return ok 
        MapService -> Redis : Set count 1 
        return ok 
    MapService -> KAFKA : Ack
    end

    MapService -> Redis : Get counter visited place
    return Data count of visited.

    alt Count < threshold frequency
        MapService -> Redis : INCR pickup_count:{user_id}:{lat}:{lon}
        return Current count
        alt count == threshold frequency
            MapService -> UserFrequencyRepo : INSERT location frequency
                UserFrequencyRepo -> PostGISDB : insert into user_location_frequency
                return ok
            MapService <- UserFrequencyRepo :ok
            MapService -> Redis : Delete GEOADD
            return ok 
            MapService -> Redis : Delete counter 
            return ok 
MapService -> KAFKA : Ack
        end
    end

@enduml
```

---

## Flow Description

### Trigger
- **Event**: `endtrip` message dari Kafka

### Processing Steps

1. **Consume** endtrip message dari Kafka
2. **Check** apakah sudah ada frequency data untuk user ini
3. **Jika ada**: Update `valid_until` timestamp
4. **Jika tidak ada**:
   - Check Redis untuk location history (ST_DWithin <= X meter)
   - Jika point tidak match: GEOADD ke Redis, set count = 1
   - Jika point match: INCR counter
5. **Jika count == threshold**: 
   - INSERT ke `user_location_frequency` table
   - Cleanup Redis (delete GEOADD dan counter)
6. **ACK** message ke Kafka

### Redis Keys Pattern

```
pickup_count:{user_id}:{lat}:{lon}
```

### Threshold

Default threshold: **>= 2 visits** untuk dianggap frequent location

---

## 🏷️ Tags

#diagram #plantuml #sequence #setter #frequency #kafka

---

*Last Updated*: 2025-01-05
