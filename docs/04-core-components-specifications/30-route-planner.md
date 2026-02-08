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

This planner integrates with OpenTripPlanner (OTP), an open-source multi-modal journey planning engine that uses real-world GTFS transit data and OpenStreetMap road networks.

### Behavior

- **External Service**: The planner communicates with an OTP server via REST API.
- **Data Sources**: OTP builds routing graphs from:
  - GTFS (General Transit Feed Specification) for transit schedules
  - OpenStreetMap (OSM) for walking, cycling, and road networks
- **Route Calculation**:
  - Sends a planning request to the OTP endpoint with origin, destination, and departure time.
  - OTP returns one or more itineraries, each composed of legs with different modes (walk, bus, train, etc.).
  - The planner transforms OTP's response into MaaS Blender's `Route` format.
- **Real-time Updates**: If OTP is configured with real-time feeds, routes reflect current delays and service changes.
- **Multi-modal Support**: Handles complex combinations like walk → bus → transfer → train → walk.

### Configuration

```json
{
  "endpoint": "http://localhost:8080/otp/routers/default/plan",
  "max_walk_distance": 1000,              // meters
  "modes": "WALK,TRANSIT",                // allowed travel modes
  "num_itineraries": 3,                   // number of alternative routes
  "walk_speed": 1.4,                      // m/s
  "timeout": 10.0                         // seconds
}
```

- `endpoint`: URL of the OTP planning API.
- `max_walk_distance`: maximum walking distance in meters.
- `modes`: comma-separated list of allowed modes (e.g., `"WALK,TRANSIT"`, `"WALK,BICYCLE,TRANSIT"`).
- `num_itineraries`: number of alternative routes to request.
- `walk_speed`: assumed walking speed in meters per second.
- `timeout`: maximum time to wait for OTP response.

#### Example Setup

1. **Start OTP Server**:
   ```bash
   # Download OTP and prepare graph data
   java -Xmx2G -jar otp-2.5.0-shaded.jar --build --save /path/to/graph
   java -Xmx2G -jar otp-2.5.0-shaded.jar --load /path/to/graph
   ```

2. **Configure MaaS Blender**:
   ```json
   {
     "planner": {
       "type": "opentripplanner",
       "endpoint": "http://localhost:8080/otp/routers/default/plan",
       "max_walk_distance": 1000,
       "modes": "WALK,TRANSIT",
       "num_itineraries": 3
     }
   }
   ```

3. **Use in Simulation**:
   ```python
   from planner.opentripplanner import OTPPlanner
   
   planner = OTPPlanner(
       endpoint="http://localhost:8080/otp/routers/default/plan",
       max_walk_distance=1000,
       modes="WALK,TRANSIT"
   )
   
   routes = await planner.plan(
       org=Location(location_id="Home", lat=35.6895, lng=139.6917),
       dst=Location(location_id="Office", lat=35.6812, lng=139.7671),
       dept=480.0  # 08:00
   )
   ```

### Output

- Returns a list of `Route` objects converted from OTP itineraries.
- Each `Route` contains:
  - `dept`: departure time (minutes from simulation start)
  - `arrv`: arrival time (minutes from simulation start)
  - `trips`: list of `Trip` objects representing legs with:
    - `org`, `dst`: origin and destination `Location` objects
    - `service`: mode identifier (e.g., `"walking"`, `"bus-line-123"`, `"train-A"`)
    - `dept`, `arrv`: leg-specific departure and arrival times
- If OTP returns no itineraries, returns an empty list or a default walking route.

:::warning
Ensure your OTP server is running and accessible before starting the simulation. Network issues or OTP downtime will cause planning failures.
:::

---

### Common Operational Notes

- **Interface Consistency**: Both planners implement the same `Planner` interface with an async `plan(org, dst, dept)` method.
- **Time Base**: All times are in minutes from the simulation start. The OTP Planner converts between simulation time and absolute timestamps.
- **Coordinate System**: Locations use latitude/longitude (WGS84).
- **Error Handling**:
  - If planning fails, planners should return an empty list or a walking-only route as fallback.
  - The Simple Planner always succeeds (with at least a walking route).
  - The OTP Planner may fail due to network issues or OTP errors; implement appropriate error handling and retries.
- **Performance**:
  - Simple Planner: Fast, in-memory graph search suitable for large-scale simulations with many concurrent users.
  - OTP Planner: Network latency and OTP processing time affect performance; consider caching for repeated queries.
- **Use Cases**:
  - Use Simple Planner for controlled experiments, testing, and synthetic scenarios.
  - Use OTP Planner for realistic simulations with real-world transit data and road networks.
