---
sidebar_position: 40
title: "Mobility Service: Integration Contract"
---

This chapter specifies the event-based integration contract for Mobility Service components in MaaS Blender.
A Mobility Service represents a transportation provider that can be integrated into the simulation platform.
Examples include public transit, ride-hailing, bike sharing, car sharing, and on-demand services.

The core value of MaaS Blender lies in its ability to integrate multiple heterogeneous mobility services.
To enable this, MaaS Blender defines a minimal, event-based integration contract.
Any service that complies with this contract can participate in the simulation, regardless of its internal implementation or data source.

The project currently ships several reference implementations under `maasblender/src/mobility_service`:

- Bus Service: scheduled fixed-route service based on GTFS data.
- Taxi/Ride-hailing Service: on-demand point-to-point service with vehicle fleet management.
- Bike Share Service: station-based shared bicycle system.
- Walking Service: pedestrian movement with distance-based travel time.

## Event Contract

| Consumed Events | Emitted Events |
|-----------------|----------------|
| `RESERVE`       | `RESERVED`     |
| `DEPART`        | `DEPARTED`     |
|                 | `ARRIVED`      |

All Mobility Services must consume `RESERVE` and `DEPART` events and emit the corresponding response events.

## Responsibilities of a Mobility Service

A Mobility Service is responsible for:

1. **Resource Management**: Managing operational resources such as vehicles, drivers, passenger capacity, and service-specific constraints.
2. **Reservation Handling**: Determining whether it can serve requested passengers based on availability and constraints.
3. **Trip Execution**: Transporting passengers from origin to destination and reporting trip status through events.

:::info
MaaS Blender does not impose internal modeling requirements.
Each service is free to implement its own logic, as long as it respects the event semantics.
:::

## Reservation Flow (`RESERVE` → `RESERVED`)

### Behavior

1. Upon receiving a `RESERVE` event, the mobility service determines whether it can serve the requested passenger.
2. This decision may consider factors such as:
   - Vehicle availability at the requested time
   - Driver availability
   - Current passenger capacity
   - Service-specific constraints (e.g., service area boundaries, operating hours)
3. The service emits a `RESERVED` event indicating success or failure.

### Event Payload

**RESERVE** (consumed):
```json
{
  "time": 10.0,
  "userId": "U001",
  "demandId": "D_1",
  "service": "bus",
  "org": {"locationId": "A", "lat": 35.0, "lng": 139.0},
  "dst": {"locationId": "B", "lat": 35.7, "lng": 139.7},
  "dept": 15.0
}
```

**RESERVED** (emitted):
```json
{
  "time": 10.0,
  "userId": "U001",
  "demandId": "D_1",
  "service": "bus",
  "reserved": true,           // or false if reservation failed
  "trip_id": "trip_123"       // optional: internal trip identifier
}
```

:::warning
Even for services that do not require reservations in reality (e.g., walking, always-available buses), the `RESERVE` / `RESERVED` interaction is mandatory.
This requirement exists to unify the interaction model across all mobility services and simplify orchestration at the User Model level.
:::

## Trip Execution Flow (`DEPART` → `DEPARTED` / `ARRIVED`)

### Behavior

1. Upon receiving a `DEPART` event, the mobility service initiates the actual movement of the passenger.
2. The service determines the actual travel duration according to its internal logic (e.g., scheduled time, distance-based calculation, real-time traffic).
3. When the trip starts, the service emits a `DEPARTED` event.
4. When the passenger reaches the destination, the service emits an `ARRIVED` event.

### Event Payload

**DEPART** (consumed):
```json
{
  "time": 15.0,
  "userId": "U001",
  "demandId": "D_1",
  "service": "bus",
  "org": {"locationId": "A", "lat": 35.0, "lng": 139.0},
  "dst": {"locationId": "B", "lat": 35.7, "lng": 139.7}
}
```

**DEPARTED** (emitted):
```json
{
  "time": 15.0,
  "userId": "U001",
  "demandId": "D_1",
  "service": "bus",
  "org": {"locationId": "A", "lat": 35.0, "lng": 139.0}
}
```

**ARRIVED** (emitted):
```json
{
  "time": 45.0,
  "userId": "U001",
  "demandId": "D_1",
  "service": "bus",
  "dst": {"locationId": "B", "lat": 35.7, "lng": 139.7},
  "trip_duration": 30.0       // optional: actual travel time
}
```

### Timing Constraints

:::warning
**Critical Timing Rules:**

1. If the `DEPART` event is emitted **before** the agreed departure time, the mobility service **must** transport the passenger.
2. If the `DEPART` event is **not** emitted by the agreed departure time, the mobility service **must not** transport the passenger and should consider the reservation cancelled.
:::

These rules ensure consistent behavior across all mobility services and prevent race conditions.

## Reference Implementations

### Bus Service (`BusService`)

- File: `maasblender/src/mobility_service/bus/bus.py`
- Primary class: `BusService`

A scheduled fixed-route service based on GTFS data.

**Characteristics:**
- Uses GTFS schedule data to determine trip times
- Checks vehicle capacity before accepting reservations
- Emits `DEPARTED` and `ARRIVED` based on scheduled times
- Supports route patterns with multiple stops

**Configuration:**
```json
{
  "service": "bus",
  "gtfs_file": "bus_routes.zip",
  "vehicle_capacity": 50
}
```

### Taxi/Ride-hailing Service (`TaxiService`)

- File: `maasblender/src/mobility_service/taxi/taxi.py`
- Primary class: `TaxiService`

An on-demand point-to-point service with vehicle fleet management.

**Characteristics:**
- Manages a fleet of vehicles with known locations
- Calculates pickup time based on nearest available vehicle
- Uses distance-based travel time estimation
- Supports dynamic pricing (optional)

**Configuration:**
```json
{
  "service": "taxi",
  "num_vehicles": 10,
  "initial_locations": [
    {"locationId": "depot", "lat": 35.5, "lng": 139.5}
  ],
  "speed_km_per_hour": 40.0
}
```

### Bike Share Service (`BikeShareService`)

- File: `maasblender/src/mobility_service/bikeshare/bikeshare.py`
- Primary class: `BikeShareService`

A station-based shared bicycle system.

**Characteristics:**
- Manages bike availability at stations
- Checks origin station for available bikes
- Checks destination station for available docks
- Uses cycling speed to calculate travel time

**Configuration:**
```json
{
  "service": "bike_share",
  "gbfs_file": "stations.json",
  "bikes_per_station": 20,
  "docks_per_station": 25,
  "cycling_speed_km_per_hour": 15.0
}
```

### Walking Service (`WalkingService`)

- File: `maasblender/src/mobility_service/walking/walking.py`
- Primary class: `WalkingService`

Pedestrian movement with distance-based travel time.

**Characteristics:**
- Always accepts reservations (walking is always available)
- Calculates travel time based on straight-line distance
- No capacity constraints

**Configuration:**
```json
{
  "service": "walking",
  "walking_speed_meters_per_minute": 50.0
}
```

## Design Rationale

By standardizing integration at the event level:

- **Heterogeneity**: MaaS Blender enables seamless integration of diverse mobility services with different operational models.
- **Simplicity**: Service developers can participate without deep knowledge of internal frameworks or data formats.
- **Extensibility**: New mobility services can be added by implementing the event contract, without modifying existing components.
- **Realism**: Services can model complex operational constraints while maintaining a simple external interface.

---

### Common Operational Notes

- **Event Timing**: All events include a `time` field representing simulation time in minutes from start.
- **Asynchronous Processing**: Services process events asynchronously using `simpy` process scheduling.
- **State Management**: Each service maintains its own internal state (vehicle locations, capacity, schedules).
- **Vehicle Modeling**: If the simulation requires explicit modeling of vehicle or driver movement, services may emit additional `DEPARTED` and `ARRIVED` events for those units.
- **Failure Handling**: Services should emit `RESERVED` with `reserved: false` rather than throwing exceptions, allowing the User Model to handle failures gracefully.
