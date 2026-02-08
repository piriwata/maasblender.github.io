---
sidebar_position: 30
title: "Route Planner: Multi-modal Journey Planning"
---

This chapter specifies the Route Planner component in MaaS Blender.
The Route Planner converts a user's travel intention (`org`, `dst`, `dept`) into one or more route candidates.
Each route is composed of ordered legs representing different transportation modes and services.

The project currently ships two reference implementations under `maasblender/src/planner`:

- Simple In-Process Planner: deterministic routing over synthetic mobility networks.
- OpenTripPlanner-backed Service: real-world multi-modal routing using GTFS and OpenStreetMap data.

Both implementations provide the same interface: given an origin, destination, and departure time, they return a list of `Route` objects.

## Simple In-Process Planner (`SimplePlanner`)

- File: `maasblender/src/planner/simple/planner.py`
- Primary class: `SimplePlanner`

This planner provides lightweight routing for mobility networks constructed from GTFS or GBFS files.

### Behavior

- **Network Construction**: Builds a `MobilityNetwork` from input data:
  - Nodes represent locations (stops, stations, or points of interest).
  - Edges represent mobility services with associated travel times and schedules.
- **Route Calculation**:
  - Given `(org, dst, dept)`, the planner searches the network for feasible paths.
  - Routes are composed of one or more trips (legs), each using a specific service.
  - Walking is included as a default fallback service with distance-based travel time.
- **Multi-modal Support**: Combines different services in a single route (e.g., walking → bus → walking).
- **Determinism**: Produces consistent results for identical inputs (no randomness).
- **Output Format**: Returns a list of `Route` objects, each containing:
  - `dept`: overall departure time
  - `arrv`: overall arrival time
  - `trips`: list of individual legs, each with `org`, `dst`, `dept`, `arrv`, and `service`.

### Configuration

The Simple Planner is configured through the broker setup. It supports two primary network types: GTFS (public transit) and GBFS (bike share).

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

**Parameters:**
- `networks`: defines one or more mobility networks to use for planning.
  - `type`: `"gtfs"` indicates GTFS-based network construction.
  - `input_files`: list of GTFS zip files to load (upload separately via API).
  - `agency_id`: optional filter to use only specific transit agencies.
- `reference_time`: simulation reference date in YYYYMMDD format (8 digits).
- `walking_meters_per_minute`: assumed walking speed for pedestrian segments (default: 50.0 m/min ≈ 3 km/h).

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

:::tip
You can combine multiple networks in a single configuration by adding both `gtfs` and `gbfs` entries under `networks`. The planner will consider all networks when computing routes.
:::

## OpenTripPlanner-backed Service (`OTPPlanner`)

- File: `maasblender/src/planner/opentripplanner/planner.py`
- Primary class: `OTPPlanner`

This planner integrates with OpenTripPlanner (OTP), an open-source multi-modal journey planning engine with advanced routing capabilities.

### Behavior

- **External Service**: Communicates with an OTP server via REST API.
- **Data Sources**: OTP builds routing graphs from uploaded files:
  - GTFS for transit schedules
  - GBFS for bike share stations
  - OpenStreetMap (OSM) for walking, cycling, and road networks
  - OTP Configuration for graph build settings and routing preferences
- **Route Calculation**:
  - Sends a planning request to the OTP endpoint with origin, destination, and departure time.
  - OTP returns one or more itineraries, each composed of legs with different modes (walk, bus, train, bike, etc.).
  - The planner transforms OTP's response into MaaS Blender's `Route` format.
- **Advanced Features**: Supports complex multi-modal combinations like walk → bus → transfer → train → walk, with transfer timing and accessibility constraints.

### Configuration

OTP Planner is configured through the broker setup with GTFS/GBFS file inputs and OTP configuration.

:::warning
Ensure all required files (`otp-config.zip`, `gtfs.zip`, etc.) are uploaded before starting the broker setup.
The OTP graph building process requires these files and will fail if any are missing.
:::

#### Configuration Structure

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
      "walking_meters_per_minute": 50.0,        // optional: if null, read from router_config.json
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
- `walking_meters_per_minute`: Walking speed in meters per minute. If `null`, the value is read from OTP's `router-config.json`.
- `timezone`: Timezone offset in hours (default: `+9` for JST).

#### OTP Configuration File (otp-config.zip)

The `otp-config.zip` should contain:

1. **OpenStreetMap Data** (`map.osm.pbf` or similar):
   - Road network for walking and cycling routes
   - Building footprints and points of interest

2. **build-config.json** (optional):
   ```json
   {
     "areaVisibility": true,
     "platformEntriesLinking": true,
     "matchBusRoutesToStreets": true
   }
   ```
   - `areaVisibility`: enables visibility calculations for areas
   - `platformEntriesLinking`: links platform entrances to street network
   - `matchBusRoutesToStreets`: snaps bus stops to nearby streets

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
   - Defines default speeds (in m/s) for different transportation modes

:::info
For detailed OTP configuration options, refer to the [OpenTripPlanner documentation](http://docs.opentripplanner.org/).
:::

#### Example: Combined GTFS and GBFS

```json
{
  "planner": {
    "type": "planner",
    "endpoint": "http://planner",
    "details": {
      "otp_config": {
        "input_files": [{"filename": "otp-config.zip"}]
      },
      "networks": {
        "transit": {
          "type": "gtfs",
          "input_files": [{"filename": "transit.zip"}]
        },
        "bikes": {
          "type": "gbfs",
          "input_files": [{"filename": "bikes.json"}]
        }
      },
      "reference_time": "20251016",
      "modes": ["WALK", "TRANSIT", "BICYCLE"]
    }
  }
}
```

This configuration enables multi-modal routing across walking, public transit, and bike share.

---

### Common Operational Notes

- **Interface Consistency**: Both planners implement the same `plan(org, dst, dept)` interface, making them interchangeable.
- **Time Base**: All times are in minutes from simulation start.
- **Determinism**:
  - Simple Planner is fully deterministic.
  - OTP Planner is deterministic given the same graph and routing parameters.
- **Performance**: Simple Planner is faster for small synthetic networks; OTP is better suited for real-world, large-scale networks with complex schedules and constraints.
 