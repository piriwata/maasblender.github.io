---
sidebar_position: 30
title: "Route Planner"
---

This chapter specifies the Route Planner component in MaaS Blender.
The Route Planner converts a user's travel intention (`org`, `dst`, `dept`) into one or more route candidates, each composed of ordered legs.
Each route represents a possible way to travel from origin to destination, potentially involving multiple modes of transportation.

The project currently ships two reference implementations under `maasblender/src/planner`:
- Simple In-Process Planner: for deterministic routing over synthetic mobility networks.
- OpenTripPlanner-backed Service: for real-world multi-modal routing using GTFS and OpenStreetMap data.

Both implementations provide the same interface: given an origin, destination, and departure time, they return a list of `Route` objects.

## Simple In-Process Planner

- Files: `maasblender/src/planner/simple/`
- Primary class: `SimplePlanner`

This planner provides a lightweight routing solution for mobility networks, typically constructed from GTFS or GBFS files.

### Behavior

- **Network Construction**: The planner constructs a `MobilityNetwork` from input data
  - Nodes represent locations (stops, stations, or points of interest).
  - Edges represent mobility services with associated travel times and schedules.
- **Route Calculation**:
  - Given `(org, dst, dept)`, the planner searches the network for possible paths.
  - Routes are composed of one or more trips, each representing a leg using a specific service.
  - Walking is typically included as a default fallback service with distance-based travel time.
- **Determinism**: The planner produces consistent results for identical inputs (no randomness).
- **Multi-modal Support**: Can combine different services (e.g., walking → bus → walking).
- **Output Format**: Returns a list of `Route` objects, each containing:
  - `dept`: overall departure time
  - `arrv`: overall arrival time
  - `trips`: list of individual legs, each with `org`, `dst`, `dept`, `arrv`, and `service`.

### Configuration

The Simple Planner is typically configured through the broker setup with GTFS or GBFS file input

#### GTFS Configuration (Public Transit)

```json
{
  "planner": {
    "type": "planner",
    "endpoint": "http://planner",
    "details": {
      "networks": {
        "gtfs": {
          "type": "gtfs",
          "input_files": [
            {
              "filename": "gtfs.zip"
            }
          ],
          "agency_id": "7230001002032"          // optional: filter by agency
        }
      },
      "reference_time": "20251016",             // YYYYMMDD format
      "walking_meters_per_minute": 50.0         // walking speed
    }
  }
}
```

- `networks`: defines one or more mobility networks to use for planning.
  - `type`: `"gtfs"` indicates GTFS-based network construction.
  - `input_files`: list of GTFS zip files to load (uploaded separately via API).
  - `agency_id`: optional filter to use only specific transit agencies.
- `reference_time`: simulation reference date in YYYYMMDD format.
- `walking_meters_per_minute`: assumed walking speed for pedestrian segments.

#### GBFS Configuration (Bike Share)

```json
{
  "planner": {
    "type": "planner",
    "endpoint": "http://planner",
    "details": {
      "networks": {
        "bike_share": {
          "type": "gbfs",
          "input_files": [
            {
              "filename": "gbfs_feed.json"
            }
          ]
        }
      },
      "reference_time": "20251016",
      "walking_meters_per_minute": 50.0
    }
  }
}
```

## OpenTripPlanner-backed Service

- Files: `maasblender/src/planner/opentripplanner/`
- Primary class: `OTPPlanner`

This planner integrates with OpenTripPlanner (OTP), an open-source multi-modal journey planning engine.
In Maas Blender, OTP is typically configured using GTFS and GBFS files through the broker setup.

### Behavior

- **External Service**: The planner communicates with an OTP server via REST API.
- **Data Sources**: OTP builds routing graphs from uploaded files:
  - GTFS for transit schedules and GBFS for bike share stations.
  - OTP Configuration for graph build settings and routing preferences.
  - OpenStreetMap (OSM) for walking, cycling, and road networks
- **Route Calculation**:
  - Sends a planning request to the OTP endpoint with origin, destination, and departure time.
  - OTP returns one or more itineraries, each composed of legs with different modes (walk, bus, train, etc.).
  - The planner transforms OTP's response into MaaS Blender's `Route` format.
- **Multi-modal Support**: Handles complex combinations like walk → bus → transfer → train → walk.

### Configuration

OTP Planner is configured through the broker setup with GTFS/GBFS file inputs and OTP configuration:

:::warning
Ensure all required files (`otp-config.zip`, `gtfs.zip`, etc.) are uploaded before starting the broker setup.
The OTP graph building process requires these files and will fail if any are missing.
:::

#### Basic Configuration Structure

```json
{
  "planner": {
    "type": "planner",
    "endpoint": "http://planner",
    "details": {
      "otp_config": {
        "input_files": [
          {
            "filename": "otp-config.zip"
          }
        ]
      },
      "networks": {
        "gtfs": {
          "type": "gtfs",
          "input_files": [
            {
              "filename": "gtfs.zip"
            }
          ],
          "agency_id": "7230001002032"          // optional: filter by agency
        }
      },
      "reference_time": "20251016",             // YYYYMMDD format (required, 8 chars)
      "modes": ["WALK", "TRANSIT"],             // optional: allowed transport modes
      "walking_meters_per_minute": 50.0,        // optional: if None, read from router_config.json
      "timezone": 9                              // optional: timezone offset (default: +9)
    }
  }
}
```

**Required Parameters:**
- `otp_config`: OTP configuration files (includes OSM data, build settings, router configuration).
  - `input_files`: List of configuration zip files to upload.
- `networks`: Dictionary of network configurations (GTFS, GBFS, etc.).
- `reference_time`: Simulation reference date in YYYYMMDD format (must be exactly 8 characters).

**Optional Parameters:**
- `modes`: List of allowed transport modes (e.g., `["WALK", "TRANSIT", "BICYCLE"]`). If not specified, OTP uses all available modes.
- `walking_meters_per_minute`: Walking speed. If `null`, the value is read from OTP's `router_config.json`.
- `timezone`: Timezone offset in hours (default: `+9` for JST).

#### OTP Configuration File (otp-config.zip)

The `otp-config.zip` should contain:

1. **OpenStreetMap Data** (`map.osm.pbf` or similar):
  Road network for walking and cycling routes

2. **build-config.json** (optional):
   ```json
   {
     "areaVisibility": true,
     "platformEntriesLinking": true,
     "matchBusRoutesToStreets": true
   }
   ```

3. **router-config.json** (optional):
   ```json
   {
     "routingDefaults": {
       "walkSpeed": 1.4,
       "bikeSpeed": 5.0,
       "carSpeed": 15.0
     }
   }
   ```
 