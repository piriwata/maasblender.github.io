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

This planner provides a lightweight, dependency-free routing solution for synthetic mobility networks defined programmatically.

### Behavior

- **Network Construction**: The planner operates on a `MobilityNetwork` graph, where:
  - Nodes represent locations (stops, stations, or points of interest).
  - Edges represent mobility services with associated travel times and schedules.
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

The Simple Planner is typically configured via code rather than a configuration file. Network setup example:

```python
from planner.simple import SimplePlanner
from core import Location, MobilityNetwork

# Create network
network = MobilityNetwork()

# Add locations
home = Location(location_id="Home", lat=35.0, lng=139.0)
station = Location(location_id="Station", lat=35.1, lng=139.1)
office = Location(location_id="Office", lat=35.2, lng=139.2)

# Add services
network.add_service(
    service="walking",
    org=home,
    dst=station,
    duration=10.0  # minutes
)
network.add_service(
    service="train",
    org=station,
    dst=office,
    duration=15.0,
    frequency=10.0  # trains every 10 minutes
)

# Initialize planner
planner = SimplePlanner(network=network)
```

#### Example

```python
# Plan a route
routes = await planner.plan(
    org=Location(location_id="Home", lat=35.0, lng=139.0),
    dst=Location(location_id="Office", lat=35.2, lng=139.2),
    dept=480.0  # 08:00
)

# routes[0] might be:
# Route(
#   dept=480.0,
#   arrv=505.0,
#   trips=[
#     Trip(org=Home, dst=Station, dept=480.0, arrv=490.0, service="walking"),
#     Trip(org=Station, dst=Office, dept=490.0, arrv=505.0, service="train")
#   ]
# )
```

### Output

- Returns a list of `Route` objects sorted by some criterion (e.g., earliest arrival, fewest transfers).
- Each `Route` contains:
  - `dept`: departure time (minutes)
  - `arrv`: arrival time (minutes)
  - `trips`: list of `Trip` objects representing individual legs
- If no route is found, returns an empty list or a walking-only route.

:::tip
The Simple Planner is ideal for synthetic scenarios, controlled experiments, and unit testing where you need predictable, reproducible routing results.
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
