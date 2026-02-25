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

## Common Operational Notes

- All five Mobility Service implementations share the same REST API contract: `/setup`, `/start`, `/peek`, `/step`, `/triggered`, `/reservable`, and `/finish`.
- All implementations receive `RESERVE` and `DEPART` events via the `/triggered` endpoint and produce `RESERVED`, `DEPARTED`, and `ARRIVED` events in response.
- The `/reservable` endpoint is used by route planners and evaluators to check whether a service can accept a new booking for a given origin–destination pair before issuing a `RESERVE`.
