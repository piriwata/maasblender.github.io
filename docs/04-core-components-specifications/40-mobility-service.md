---
sidebar_position: 40
title: "Mobility Service"
---

This chapter specifies the reference implementations for Mobility Service components in MaaS Blender.
Each Mobility Service simulates a specific mode of transportation and handles the full lifecycle of a user's trip—from reservation through departure to arrival—by responding to simulation events.

The project currently ships five reference implementations under `maasblender/src/base_simulators`:

- **On-Demand** (`ondemand`): demand-responsive transport with dynamic dispatch optimization.
- **One-Way** (`oneway`): one-way free-floating vehicle sharing (e.g., e-scooters or carshare).
- **Route Deviation** (`routedeviation`): scheduled routes that flexibly deviate to serve on-demand stops.
- **Scheduled** (`scheduled`): fixed-schedule transit services based on GTFS timetables.
- **Walking** (`walking`): pedestrian travel, used as a fallback or first/last-mile connector.

All implementations share the same event-driven interface: they accept `RESERVE` and `DEPART` triggered events and emit `RESERVED`, `DEPARTED`, and `ARRIVED` events.

## Walking (`walking`)

- Files: `maasblender/src/base_simulators/walking/`
- Primary class: `Simulation` (in `simulation.py`), API server in `controller.py`

The Walking simulator models pedestrian travel between any two coordinates.
It is the simplest mobility service and is typically used as a fallback when no other service is available, or as a first/last-mile walking leg within a multi-modal trip.

### Behavior

- On `RESERVE`:
  - Always accepts the reservation (the service is never at capacity).
  - Computes travel duration from the geodesic distance between `org` and `dst` using the configured walking speed (`walking_meters_per_minute`).
  - If `arrv` is provided and `arrv > dept`, the provided `arrv` is used as-is; otherwise `arrv` is recalculated as `dept + duration`.
  - Immediately emits a `RESERVED` event with `success: true` and the confirmed route (`dept`, `arrv`).
- On `DEPART`:
  - Emits `DEPARTED` at time `dept` from `org`.
  - Emits `ARRIVED` at time `arrv` at `dst`.
- The `/reservable` endpoint always returns `true` regardless of origin or destination.

### Configuration

The Walking simulator is configured via the `/setup` endpoint:

```json
{
  "walking_meters_per_minute": 80.0   // walking speed in meters per minute (default: 80.0)
}
```

- `walking_meters_per_minute`: Walking speed used to compute travel time from geodesic distance.
  The default value of `80.0` m/min corresponds to approximately 4.8 km/h.

:::tip
The geodesic distance is computed using `geopy.distance.geodesic`, which accounts for the Earth's curvature.
For short urban distances the difference from a flat-Earth approximation is negligible, but it improves accuracy for longer walking legs.
:::

#### Example

```json
{
  "walking_meters_per_minute": 60.0
}
```

Setting `60.0` m/min (3.6 km/h) models a slower-paced elderly walker or a route with frequent crossings.

### Output

- `RESERVED` event (emitted immediately after `RESERVE`):
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "route": [
        {
          "org": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StopB", "lat": 35.01, "lng": 139.01 },
          "dept": 480.0,
          "arrv": 492.3
        }
      ]
    }
  }
  ```
- `DEPARTED` event (emitted at `dept`):
  ```json
  {
    "eventType": "DEPARTED",
    "details": {
      "subjectId": "U001",
      "userId": "U001",
      "demandId": "D_1",
      "mobilityId": null,
      "location": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 }
    }
  }
  ```
- `ARRIVED` event (emitted at `arrv`):
  ```json
  {
    "eventType": "ARRIVED",
    "details": {
      "subjectId": "U001",
      "userId": "U001",
      "demandId": "D_1",
      "mobilityId": null,
      "location": { "locationId": "StopB", "lat": 35.01, "lng": 139.01 }
    }
  }
  ```

:::info
`mobilityId` is always `null` for walking trips because there is no physical vehicle involved.
:::

## Scheduled (`scheduled`)

- Files: `maasblender/src/base_simulators/scheduled/`
- Primary class: `Simulation` (in `simulation.py`), API server in `controller.py`

The Scheduled simulator models fixed-timetable transit services such as buses and trains.
It loads GTFS data and simulates vehicles following pre-defined stop sequences, allowing users to board and alight at any stop along a route.

### Behavior

- On `RESERVE`:
  - Looks up the earliest vehicle (trip) that serves the `org` stop and also visits `dst` after `org`.
  - Checks whether that vehicle still has capacity (`is_reservable`).
  - If a suitable vehicle with capacity is found, reserves the seat and emits `RESERVED` with `success: true`, including computed `dept` (scheduled departure from `org`) and `arrv` (scheduled arrival at `dst`).
  - If no suitable vehicle is found, emits `RESERVED` with `success: false`.
- On `DEPART`:
  - The user waits at the `org` stop. When the reserved vehicle arrives at `org`, the user boards; `DEPARTED` is emitted.
  - When the vehicle arrives at `dst`, the user alights; `ARRIVED` is emitted.
- The `/reservable` endpoint checks in real time whether any vehicle can serve `org` → `dst` with available capacity.

### Configuration

The Scheduled simulator is configured via the `/setup` endpoint after uploading a GTFS zip file:

```json
{
  "reference_time": "20251016",          // simulation reference date (YYYYMMDD, 8 chars)
  "input_files": [
    { "filename": "gtfs.zip" }           // GTFS archive uploaded via /upload
  ],
  "mobility": {
    "capacity": 30                        // seat capacity shared across all trips
  }
}
```

- `reference_time`: Base date for GTFS calendar resolution (YYYYMMDD, must be exactly 8 characters).
- `input_files`: Exactly one GTFS zip file, either by `filename` (pre-uploaded via `/upload`) or `fetch_url`.
- `mobility.capacity`: Passenger capacity applied uniformly to every trip in the GTFS feed.

:::info
GTFS `block_id` is supported. Trips that share a `block_id` are chained into a continuous service, allowing through-service across multiple trips without passengers having to alight and re-board.
:::

### Output

- `RESERVED` event (success):
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "mobilityId": "trip-001",
      "route": [
        {
          "org": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StopC", "lat": 35.05, "lng": 139.05 },
          "dept": 480.0,
          "arrv": 495.0
        }
      ]
    }
  }
  ```
- `RESERVED` event (failure):
  ```json
  { "eventType": "RESERVED", "details": { "userId": "U001", "demandId": "D_1", "success": false } }
  ```
- `DEPARTED` and `ARRIVED` events: emitted when the vehicle reaches the respective stop.
  `mobilityId` contains the GTFS trip ID.

## Route Deviation (`routedeviation`)

- Files: `maasblender/src/base_simulators/routedeviation/`
- Primary class: `Simulation` (in `simulation.py`), API server in `controller.py`

The Route Deviation simulator extends the fixed-schedule model by allowing vehicles to deviate from their planned route to pick up or drop off passengers at flexible (off-stop) locations.
It is suitable for demand-responsive feeder services or dial-a-ride scenarios that still follow a general corridor.

### Behavior

- On `RESERVE`:
  - Looks up the earliest vehicle that can serve the given `org` → `dst` pair, including arbitrary coordinates not defined as fixed stops (`TemporaryStop`).
  - Checks capacity and feasibility (`is_reservable`).
  - If feasible, reserves the user and emits `RESERVED` with `success: true` and the planned pickup/dropoff times.
  - If no vehicle can serve the request, emits `RESERVED` with `success: false`.
- On `DEPART`:
  - The user waits at the flexible pickup point. When the vehicle deviates to serve the user, `DEPARTED` is emitted.
  - When the vehicle reaches the dropoff point, `ARRIVED` is emitted.
- The `/reservable` endpoint checks in real time whether any vehicle can reach the given `org`/`dst` coordinates.

:::info
Route Deviation accepts full coordinate pairs (`lat`, `lng`) for origin and destination, not only predefined stop IDs.
This distinguishes it from the Scheduled simulator, which only serves stops listed in the GTFS feed.
:::

### Configuration

The Route Deviation simulator shares the same configuration structure as Scheduled:

```json
{
  "reference_time": "20251016",          // simulation reference date (YYYYMMDD, 8 chars)
  "input_files": [
    { "filename": "gtfs.zip" }           // GTFS archive uploaded via /upload
  ],
  "mobility": {
    "capacity": 10                        // passenger capacity per vehicle
  }
}
```

- `reference_time`: Base date for GTFS calendar resolution (YYYYMMDD, exactly 8 characters).
- `input_files`: Exactly one GTFS zip file.
- `mobility.capacity`: Passenger capacity applied to all vehicles.

### Output

Output events have the same structure as Scheduled, with the difference that `org`/`dst` in `RESERVED` may reference coordinates not present in the GTFS stop list.

## On-Demand (`ondemand`)

- Files: `maasblender/src/base_simulators/ondemand/`
- Primary class: `Simulation` (in `simulation.py`), API server in `controller.py`

The On-Demand simulator models demand-responsive bus services that pick up and drop off multiple users dynamically.
Each vehicle operates within a defined service area (derived from GTFS FLEX) and its route is continuously re-optimized as new reservations arrive, using OR-Tools or brute-force combinatorial search.

### Behavior

- On `RESERVE`:
  - Evaluates all available vehicles and computes the optimal insertion of the new user into each vehicle's current schedule (minimizing total delay across all passengers).
  - Uses **OR-Tools** constraint solver when `enable_ortools` is `true` (default); falls back to brute-force enumeration otherwise.
  - If a feasible assignment is found (delay within `max_delay_time`), updates the vehicle's schedule and emits `RESERVED` with `success: true`, including `dept` (scheduled pickup) and `arrv` (scheduled dropoff).
  - If no feasible assignment exists, emits `RESERVED` with `success: false`.
- On `DEPART`:
  - The user signals readiness at the origin stop (`ready_to_depart`).
  - When the on-demand vehicle arrives and the user boards, `DEPARTED` is emitted.
  - When the vehicle delivers the user to the destination stop, `ARRIVED` is emitted.
- The `/reservable` endpoint checks in real time whether any vehicle can serve `org` → `dst` within delay constraints.

### Configuration

The On-Demand simulator requires uploading a GTFS FLEX zip and a stop-to-stop distance matrix:

```json
{
  "reference_time": "20251016",            // simulation reference date (YYYYMMDD)
  "input_files": [
    { "filename": "gtfs_flex.zip" }        // GTFS FLEX archive (uploaded via /upload)
  ],
  "network": {
    "fetch_url": "http://planner/network"  // URL to fetch the stop-stop distance matrix
    // or: "filename": "network.json"      // pre-uploaded distance matrix file
  },
  "enable_ortools": true,                  // use OR-Tools solver (default: true)
  "board_time": 1.0,                       // boarding/alighting time per stop [min] (ignored when enable_ortools=true)
  "max_delay_time": 15.0,                  // maximum acceptable delay [min]
  "mobility_speed": 333.33,                // vehicle speed [m/min] (default: 20km/h)
  "max_calculation_seconds": 30,           // solver time limit [sec] (default: 30)
  "max_calculation_stop_times_length": 10, // max stops per route for brute-force (default: 10)
  "mobilities": [
    {
      "mobility_id": "car-1",             // unique vehicle identifier
      "trip_id": "trip-A",               // GTFS FLEX trip defining the service area
      "capacity": 4,                      // passenger capacity
      "stop": "depot-stop"               // initial stop ID
    }
  ]
}
```

- `reference_time`: Base date for GTFS FLEX calendar resolution (YYYYMMDD, exactly 8 characters).
- `input_files`: Exactly one GTFS FLEX zip file (maximum two when combined with a local network file).
- `network`: Stop-to-stop travel time matrix. Provide either `fetch_url` (computed on-the-fly from a planner) or `filename` (pre-generated JSON with `stops` and `matrix` keys).
- `enable_ortools`: When `true`, uses OR-Tools for optimal dispatch; `board_time` is automatically set to `0` and a warning is logged.
- `board_time`: Extra time for boarding and alighting per stop (only used when `enable_ortools` is `false`).
- `max_delay_time`: Upper bound on acceptable pickup delay relative to user's desired departure. Reservations exceeding this delay are rejected.
- `mobilities`: List of vehicle definitions. Each vehicle follows a GTFS FLEX trip that defines its permitted service area and operating window.

:::warning
When `enable_ortools` is `true`, `board_time` is ignored and will be overridden to `0`. Set `board_time` only when using the brute-force solver.
:::

### Output

- `RESERVED` event (success):
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "mobilityId": "car-1",
      "route": [
        {
          "org": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StopB", "lat": 35.02, "lng": 139.02 },
          "dept": 485.0,
          "arrv": 498.0
        }
      ]
    }
  }
  ```
- `RESERVED` event (failure):
  ```json
  { "eventType": "RESERVED", "details": { "userId": "U001", "demandId": "D_1", "success": false } }
  ```
- `DEPARTED` and `ARRIVED` events carry the vehicle's current stop location and the `mobilityId`.

## One-Way (`oneway`)

- Files: `maasblender/src/base_simulators/oneway/`
- Primary class: `Simulation` (in `simulation.py`), API server in `controller.py`

The One-Way simulator models station-based free-floating vehicle sharing (e.g., e-scooters or shared bikes) using GBFS data.
Users reserve a vehicle at an origin station and a dock at a destination station; the vehicle travels between stations autonomously after the user departs.
An operator process periodically rebalances vehicles across stations.

### Behavior

- On `RESERVE`:
  - Checks that the origin station has at least one available (reservable) vehicle **and** the destination station has at least one available dock.
  - If both conditions are met, reserves the vehicle and dock; emits `RESERVED` with `success: true` and computed `arrv` based on `mobility_speed`.
  - If either condition fails, emits `RESERVED` with `success: false`.
- On `DEPART`:
  - The user picks up the reserved vehicle from the origin station; `DEPARTED` is emitted.
  - The vehicle travels (`mobility_speed`) to the destination station; on arrival the vehicle is parked and `ARRIVED` is emitted.
- Battery simulation: the vehicle's state of charge (SoC) decreases while moving (`discharging_speed`) and recovers while docked at a charging station (`charging_speed`).
- Operator rebalancing: a background operator process runs between `operator_start_time` and `operator_end_time`, checking station balance every `operator_interval` minutes and relocating up to `operator_capacity` vehicles per trip.
- The `/reservable` endpoint checks in real time whether `org` has an available vehicle and `dst` has an available dock.

### Configuration

The One-Way simulator is configured after uploading a GBFS zip file:

```json
{
  "input_files": [
    { "filename": "gbfs.zip" }              // GBFS archive (station_information + free_bike_status)
  ],
  "mobility_speed": 200.0,                  // vehicle travel speed [m/min] (default: 200 = ~12km/h)
  "charging_speed": 0.003333,               // SoC increase rate [/min] (default: full charge in ~5h)
  "discharging_speed": -0.004386,           // SoC decrease rate [/min] (default: full drain in ~3h38min)
  "operator_start_time": 360.0,             // operator workday start [min] (default: 360 = 06:00)
  "operator_end_time": 720.9,               // operator workday end [min] (default: ~720 = 12:00)
  "operator_interval": 15.0,               // rebalancing interval [min] (default: 15)
  "operator_speed": 1000.0,                 // operator vehicle speed [m/min] (default: 1000 = 60km/h)
  "operator_loading_time": 1,              // time to load/unload one vehicle [min] (default: 1)
  "operator_capacity": 4                   // max vehicles per operator trip (default: 4)
}
```

- `input_files`: Exactly one GBFS zip containing at least `station_information.json` and `free_bike_status.json`.
- `mobility_speed`: Speed used to compute travel time between stations.
- `charging_speed` / `discharging_speed`: Battery charge rate (positive) and discharge rate (negative), both in SoC fraction per minute.
- `operator_*`: Parameters governing the automatic rebalancing operator that runs in the background.

:::tip
The GBFS file's `current_range_meters` field per bike is used to initialize each vehicle's battery SoC at simulation start.
:::

### Output

- `RESERVED` event (success):
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "mobilityId": "bike-42",
      "route": [
        {
          "org": { "locationId": "StationA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StationB", "lat": 35.01, "lng": 139.01 },
          "dept": 480.0,
          "arrv": 483.5
        }
      ]
    }
  }
  ```
- `RESERVED` event (failure):
  ```json
  { "eventType": "RESERVED", "details": { "userId": "U001", "demandId": "D_1", "success": false } }
  ```
- `DEPARTED` and `ARRIVED` events carry `mobilityId` (the specific bike/scooter ID) and the station location.

## Common Operational Notes

- All five Mobility Service implementations share the same REST API contract: `/setup`, `/start`, `/peek`, `/step`, `/triggered`, `/reservable`, and `/finish`.
- All implementations receive `RESERVE` and `DEPART` events via the `/triggered` endpoint and produce `RESERVED`, `DEPARTED`, and `ARRIVED` events in response.
- The `/reservable` endpoint is used by route planners and evaluators to check whether a service can accept a new booking for a given origin–destination pair before issuing a `RESERVE`.
