---
sidebar_position: 30
title: "Route Planner"
---

This chapter specifies the Route Planner component in MaaS Blender.
The Route Planner converts a user's travel intention (`org`, `dst`, `dept`) into one or more route candidates, each composed of ordered legs (trips).
Each route represents a possible way to travel from origin to destination, potentially involving multiple modes of transportation.

The project currently ships two reference implementations under `maasblender/src/planner`:
- Simple In-Process Planner: for deterministic routing over synthetic mobility networks.
- OpenTripPlanner-backed Service: for real-world multi-modal routing using GTFS and OpenStreetMap data.

Both implementations provide the same interface: given an origin, destination, and departure time, they return a list of `Route` objects.

## Simple In-Process Planner

- Files: `maasblender/src/planner/simple/`
- Primary class: `SimplePlanner`

This planner provides a lightweight routing solution for mobility networks, typically constructed from GTFS (General Transit Feed Specification) or GBFS (General Bikeshare Feed Specification) files.

### Behavior

- **Network Construction**: The planner constructs a `MobilityNetwork` from input data:
  - **GTFS Files**: Reads transit schedules, stops, routes, and trips from GTFS format for public transportation.
  - **GBFS Files**: Reads bikeshare station information and bike availability from GBFS format for bike-sharing systems.
  - Nodes represent stops, stations, or points of interest.
  - Edges represent mobility services with schedules and travel times derived from GTFS/GBFS data.
- **Route Calculation**:
  - Given `(org, dst, dept)`, the planner searches the network for feasible paths.
  - Routes are composed of one or more trips, each representing a leg using a specific service.
  - Walking is typically included as a default fallback service with distance-based travel time.
- **Determinism**: The planner produces consistent results for identical inputs (no randomness).
- **Multi-modal Support**: Can combine different services (e.g., walking → bus → walking).
- **Output Format**: Returns a list of `Route` objects, each containing:
  - `dept`: overall departure time
  - `arrv`: overall arrival time
  - `trips`: list of individual legs, each with `org`, `dst`, `dept`, `arrv`, and `service`.

### Configuration

The Simple Planner is typically configured through the broker setup with GTFS or GBFS file input:

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

- `type`: `"gbfs"` indicates GBFS-based network construction for bike-sharing systems.
- `input_files`: list of GBFS JSON files to load (station information, bike availability).

#### Multi-Modal Configuration (GTFS + GBFS)

Multiple network types can be combined for multi-modal routing:

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
          ]
        },
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

This configuration enables routes that combine transit and bike-sharing (e.g., bike → train → bike).

#### Example Setup

1. **Upload GTFS File** (for transit):
   ```python
   import httpx
   
   with httpx.Client() as client:
       with open("gtfs.zip", "rb") as gtfs_file:
           response = client.post(
               "http://localhost:3010/upload",
               files={
                   "upload_file": (
                       "gtfs.zip",
                       gtfs_file,
                       "application/x-zip-compressed"
                   )
               }
           )
   ```

2. **Upload GBFS File** (for bike share, optional):
   ```python
   with httpx.Client() as client:
       with open("gbfs_feed.json", "rb") as gbfs_file:
           response = client.post(
               "http://localhost:3010/upload",
               files={
                   "upload_file": (
                       "gbfs_feed.json",
                       gbfs_file,
                       "application/json"
                   )
               }
           )
   ```

3. **Configure Planner in broker_setup.json** (as shown above)

4. **Use Planner in Simulation**:
   The planner automatically loads the GTFS/GBFS network during broker setup and uses it for route planning requests.

#### Alternative: Programmatic Configuration

For testing or special cases, networks can also be constructed programmatically:

```python
from planner.simple import SimplePlanner
from core import Location, MobilityNetwork

# Create network
network = MobilityNetwork()

# Add locations
home = Location(location_id="Home", lat=35.0, lng=139.0)
station = Location(location_id="Station", lat=35.1, lng=139.1)

# Add services
network.add_service(
    service="walking",
    org=home,
    dst=station,
    duration=10.0  # minutes
)

# Initialize planner
planner = SimplePlanner(network=network)
```

### Output

- Returns a list of `Route` objects sorted by some criterion (e.g., earliest arrival, fewest transfers).
- Each `Route` contains:
  - `dept`: departure time (minutes)
  - `arrv`: arrival time (minutes)
  - `trips`: list of `Trip` objects representing individual legs with:
    - `org`, `dst`: origin and destination locations from GTFS stops or GBFS stations
    - `service`: transit service ID from GTFS (e.g., route name) or bike-sharing service from GBFS
    - `dept`, `arrv`: leg-specific departure and arrival times
- If no route is found, returns an empty list or a walking-only route.

:::tip
The Simple Planner with GTFS/GBFS input is ideal for simulating real-world multi-modal scenarios combining public transit and bike-sharing with actual schedule data, while maintaining lightweight computation and deterministic results.
:::

## OpenTripPlanner-backed Service

- Files: `maasblender/src/planner/opentripplanner/`
- Primary class: `OTPPlanner`

This planner integrates with OpenTripPlanner (OTP), an open-source multi-modal journey planning engine. In MaaS Blender, OTP is typically configured using GTFS (General Transit Feed Specification) and GBFS (General Bikeshare Feed Specification) files through the broker setup.

### Behavior

- **External Service**: The planner communicates with an OTP server via REST API.
- **Data Sources**: OTP builds routing graphs from uploaded files:
  - **GTFS Files**: Transit schedules, stops, routes, and trips for public transportation.
  - **GBFS Files**: Bike-sharing station information and vehicle availability (optional).
  - **OTP Configuration**: Graph build settings, routing preferences, and updater configurations.
  - **OpenStreetMap (OSM)**: Walking, cycling, and road networks (included in otp-config.zip).
- **Route Calculation**:
  - Sends a planning request to the OTP endpoint with origin, destination, and departure time.
  - OTP returns one or more itineraries, each composed of legs with different modes (walk, bus, train, bike, etc.).
  - The planner transforms OTP's response into MaaS Blender's `Route` format.
- **Real-time Updates**: If OTP is configured with real-time GTFS-RT feeds in the configuration, routes reflect current delays and service changes.
- **Multi-modal Support**: Handles complex combinations like walk → bus → transfer → train → bike → walk.

### Configuration

OTP Planner is configured through the broker setup with GTFS/GBFS file inputs and OTP configuration:

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
   - Road network for walking and cycling routes

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
     },
     "updaters": [
       {
         "type": "vehicle-rental",
         "sourceType": "gbfs",
         "url": "https://example.com/gbfs.json",
         "network": "bike-share-system"
       }
     ]
   }
   ```

#### GTFS Configuration

```json
{
  "networks": {
    "gtfs": {
      "type": "gtfs",
      "input_files": [
        {
          "filename": "gtfs.zip"
        }
      ],
      "agency_id": "7230001002032"
    }
  }
}
```

- `type`: Must be `"gtfs"` for transit data.
- `input_files`: List of GTFS zip files (uploaded separately).
- `agency_id`: Optional filter to use only specific transit agencies.

#### GBFS Configuration

```json
{
  "networks": {
    "bike_share": {
      "type": "gbfs",
      "input_files": [
        {
          "filename": "gbfs_feed.json"
        }
      ]
    }
  }
}
```

- `type`: Must be `"gbfs"` for bike-sharing data.
- `input_files`: List of GBFS JSON files with station and availability information.

#### Multi-Modal Configuration (GTFS + GBFS)

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
          ]
        },
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
      "modes": ["WALK", "TRANSIT", "BICYCLE"],
      "walking_meters_per_minute": 50.0,
      "timezone": 9
    }
  }
}
```

This enables multi-modal routing combining transit, bike-sharing, and walking.

#### Example Setup

1. **Upload OTP Configuration File**:
   ```python
   import httpx
   
   with httpx.Client() as client:
       # Upload OTP configuration (includes OSM data, build-config.json, router-config.json)
       with open("otp-config.zip", "rb") as otp_config_file:
           response = client.post(
               "http://localhost:3010/upload",
               files={
                   "upload_file": (
                       "otp-config.zip",
                       otp_config_file,
                       "application/x-zip-compressed"
                   )
               }
           )
   ```

2. **Upload GTFS File** (for transit):
   ```python
   with httpx.Client() as client:
       with open("gtfs.zip", "rb") as gtfs_file:
           response = client.post(
               "http://localhost:3010/upload",
               files={
                   "upload_file": (
                       "gtfs.zip",
                       gtfs_file,
                       "application/x-zip-compressed"
                   )
               }
           )
   ```

3. **Upload GBFS File** (for bike share, optional):
   ```python
   with httpx.Client() as client:
       with open("gbfs_feed.json", "rb") as gbfs_file:
           response = client.post(
               "http://localhost:3010/upload",
               files={
                   "upload_file": (
                       "gbfs_feed.json",
                       gbfs_file,
                       "application/json"
                   )
               }
           )
   ```

4. **Configure Planner in broker_setup.json** (as shown above)

5. **Setup and Start Simulation**:
   The OTP planner service automatically:
   - Extracts uploaded files
   - Builds the routing graph from GTFS data and OSM networks
   - Starts the OTP server
   - Becomes ready for route planning requests during simulation

:::info
The OTP graph building process may take several minutes depending on the size of GTFS data and OSM network. The broker setup step will wait for this process to complete before starting the simulation.
:::

### Output

- Returns a list of `Route` objects converted from OTP itineraries.
- Each `Route` contains:
  - `dept`: departure time (minutes from simulation start)
  - `arrv`: arrival time (minutes from simulation start)
  - `trips`: list of `Trip` objects representing legs with:
    - `org`, `dst`: origin and destination `Location` objects (from GTFS stops, GBFS stations, or OSM nodes)
    - `service`: mode identifier (e.g., `"walking"`, `"bus-line-123"`, `"train-A"`, `"bike-share"`)
    - `dept`, `arrv`: leg-specific departure and arrival times
- If OTP returns no itineraries, returns an empty list or a default walking route.

:::warning
Ensure all required files (otp-config.zip, gtfs.zip, etc.) are uploaded before starting the broker setup. The OTP graph building process requires these files and will fail if any are missing.
:::

:::tip
The OTP Planner with GTFS/GBFS input provides the most comprehensive multi-modal routing capabilities, supporting real-world transit schedules, bike-sharing systems, and detailed pedestrian/cycling networks from OpenStreetMap data.
:::

---

### Common Operational Notes

- **Interface Consistency**: Both planners implement the same `Planner` interface with an async `plan(org, dst, dept)` method.
- **Time Base**: All times are in minutes from the simulation start. The OTP Planner converts between simulation time and absolute timestamps using the `reference_time` and `timezone` settings.
- **Coordinate System**: Locations use latitude/longitude (WGS84).
- **File Upload**: Both planners require files to be uploaded via API before broker setup:
  - Simple Planner: Uploads GTFS/GBFS files to the planner service.
  - OTP Planner: Uploads otp-config.zip, GTFS files, and optionally GBFS files to the OTP service.
- **Network Construction**:
  - Simple Planner: Constructs in-memory network from GTFS/GBFS data during setup (fast, lightweight).
  - OTP Planner: Builds comprehensive routing graph including OSM street networks during setup (slower, but more detailed).
- **Error Handling**:
  - If planning fails, planners should return an empty list or a walking-only route as fallback.
  - The Simple Planner always succeeds (with at least a walking route).
  - The OTP Planner may fail due to graph building errors or OTP service issues; implement appropriate error handling.
- **Performance**:
  - Simple Planner: Fast, in-memory graph search suitable for large-scale simulations with many concurrent users.
  - OTP Planner: Network latency and OTP processing time affect performance; graph building can take several minutes; consider caching for repeated queries.
- **Use Cases**:
  - Use Simple Planner for lightweight simulations with GTFS/GBFS data where fast setup and deterministic results are priorities.
  - Use OTP Planner for comprehensive real-world simulations requiring detailed street networks, complex multi-modal routing, and real-time transit updates.
