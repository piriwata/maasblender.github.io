---
sidebar_position: 30
title: Route Planner
---

# Route Planner Component

The **Route Planner** component is responsible for calculating multi-modal routes between origin and destination locations. It evaluates available mobility services, transit schedules, and network connectivity to provide route options that satisfy user time constraints and preferences.

## Overview

The Route Planner serves as the core routing intelligence in the MaaS simulation. It:

- **Receives** route queries from the User Model component
- **Analyzes** available mobility services and their capabilities
- **Calculates** one or more viable routes using available services
- **Returns** route options with detailed timing and service information
- **Supports** various routing algorithms and optimization criteria

The planner must understand the characteristics of different mobility modes (fixed-route transit, on-demand services, shared mobility) and combine them into feasible multi-modal journeys.

## Available Implementations

MaaS Blender provides multiple route planner implementations supporting different routing approaches:

### 1. Simple Route Planner

**File:** `src/planner/simple/`  
**Primary class:** `RoutePlanner` (in `route_planner.py`)

The Simple route planner implements basic multi-modal routing using pre-loaded service data and simple graph algorithms.

#### Behavior

The Simple planner:

1. **Loads service definitions** during initialization:
   - GTFS data for fixed-route public transit (bus, train, tram)
   - GBFS data for bike-sharing and scooter-sharing services
   - GTFS-Flex data for demand-responsive transit
2. **Builds a transport network** graph with:
   - Stops/stations as nodes
   - Transit routes and walking paths as edges
   - Time-dependent edge weights (based on schedules)
3. **Processes route queries** by:
   - Finding nearby stops/stations to origin and destination
   - Running a time-dependent shortest path algorithm
   - Generating trip segments for each leg of the journey
   - Applying transfer penalties and waiting time constraints
4. **Returns multiple route options** ranked by:
   - Total travel time
   - Number of transfers
   - Departure time fit

#### Configuration

```json
{
  "services": [
    {
      "serviceId": "bus-service",      // Unique service identifier
      "type": "gtfs",                   // Service type: gtfs, gbfs, gtfs-flex
      "dataPath": "/data/gtfs/bus/"    // Path to service data
    },
    {
      "serviceId": "bike-share",
      "type": "gbfs",
      "dataPath": "/data/gbfs/bikes/",
      "maxDistance": 5000               // Maximum trip distance in meters
    },
    {
      "serviceId": "drt-service",
      "type": "gtfs-flex",
      "dataPath": "/data/gtfs-flex/drt/"
    }
  ],
  "routing": {
    "maxWalkDistance": 1000,      // Maximum walking distance between stops (meters)
    "walkingSpeed": 1.4,          // Walking speed in m/s (default: ~5 km/h)
    "maxTransfers": 3,            // Maximum number of transfers allowed
    "transferPenalty": 300,       // Time penalty for each transfer (seconds)
    "maxRoutesReturned": 5,       // Maximum number of route options to return
    "searchTimeWindow": 3600      // Time window to search for routes (seconds)
  },
  "network": {
    "distanceCalculation": "haversine",  // Method: "haversine", "euclidean"
    "coordinateSystem": "wgs84"          // Coordinate system for lat/lon
  }
}
```

:::info Service Types
- **GTFS**: General Transit Feed Specification for scheduled transit (buses, trains, ferries)
- **GBFS**: General Bikeshare Feed Specification for dockless bike/scooter sharing
- **GTFS-Flex**: Extension of GTFS for flexible/demand-responsive services
:::

### 2. OpenTripPlanner Integration

**File:** `src/planner/opentripplanner/`  
**Primary class:** `OTPPlanner` (in `otp_planner.py`)

This implementation integrates with [OpenTripPlanner](https://www.opentripplanner.org/), a mature open-source multi-modal route planner.

#### Behavior

The OTP integration:

1. **Delegates route planning** to an external OpenTripPlanner server
2. **Translates MaaS Blender requests** to OTP API format
3. **Parses OTP responses** and converts them to MaaS Blender route format
4. **Supports advanced OTP features**:
   - Real-time transit updates
   - Bike rental integration
   - Accessibility routing
   - Elevation-aware walking/cycling
5. **Handles OTP server failures** with appropriate error responses

#### Configuration

```json
{
  "otp": {
    "endpoint": "http://localhost:8080/otp/routers/default/",  // OTP server endpoint
    "timeout": 30,                 // Request timeout in seconds
    "retries": 3                   // Number of retry attempts
  },
  "routing": {
    "modes": "WALK,TRANSIT,BICYCLE",  // OTP mode string
    "maxWalkDistance": 1000,
    "maxTransfers": 3,
    "wheelchair": false,           // Accessibility requirement
    "optimize": "QUICK",           // Optimization: QUICK, SAFE, FLAT, GREENWAYS, TRIANGLE
    "numItineraries": 5            // Number of route options to request
  },
  "translation": {
    "serviceMapping": {            // Map OTP route IDs to MaaS service IDs
      "BUS:1": "bus-service",
      "RAIL:A": "train-service"
    }
  }
}
```

## Component Interface

All route planner implementations expose a REST API for route queries.

### Route Query Endpoint

**POST** `/plan`

Requests route options between two locations.

**Request Body:**
```json
{
  "org": {                       // Origin location
    "locationId": "A",
    "lat": 35.681236,
    "lng": 139.767125
  },
  "dst": {                       // Destination location
    "locationId": "B",
    "lat": 35.689487,
    "lng": 139.691711
  },
  "dept": 28800,                 // Desired departure time (seconds since simulation start)
  "arrv": null,                  // OR desired arrival time (one must be null)
  "preferences": {               // Optional user preferences
    "maxWalkDistance": 800,      // Override default maximum walking distance
    "maxWaitTime": 600,          // Maximum waiting time at stops
    "preferredModes": ["bus", "train"],  // Preferred mobility modes
    "avoidModes": ["bike"],      // Modes to avoid
    "wheelchair": false          // Accessibility requirement
  }
}
```

**Response:**
```json
{
  "routes": [                    // Array of route options
    {
      "trips": [                 // Array of trip segments
        {
          "org": {
            "locationId": "A",
            "lat": 35.681236,
            "lng": 139.767125
          },
          "dst": {
            "locationId": "stop-123",
            "lat": 35.682000,
            "lng": 139.768000
          },
          "dept": 28800,
          "arrv": 28860,         // 1-minute walk to stop
          "service": "walk",     // Special service ID for walking
          "mode": "walk",
          "distance": 120        // Distance in meters
        },
        {
          "org": {
            "locationId": "stop-123",
            "lat": 35.682000,
            "lng": 139.768000
          },
          "dst": {
            "locationId": "stop-456",
            "lat": 35.689000,
            "lng": 139.692000
          },
          "dept": 28900,         // 40-second wait at stop
          "arrv": 29400,         // 500-second bus ride (8:20)
          "service": "bus-service",
          "mode": "bus",
          "routeId": "route-1",  // Internal route identifier
          "tripId": "trip-123",  // Specific trip/run identifier
          "distance": 5000
        },
        {
          "org": {
            "locationId": "stop-456",
            "lat": 35.689000,
            "lng": 139.692000
          },
          "dst": {
            "locationId": "B",
            "lat": 35.689487,
            "lng": 139.691711
          },
          "dept": 29400,
          "arrv": 29520,         // 2-minute walk from stop
          "service": "walk",
          "mode": "walk",
          "distance": 200
        }
      ],
      "summary": {
        "totalTime": 720,        // Total travel time in seconds
        "totalDistance": 5320,   // Total distance in meters
        "walkDistance": 320,     // Total walking distance
        "transfers": 0,          // Number of transfers
        "modes": ["walk", "bus"] // Modes used in this route
      }
    }
  ]
}
```

:::warning Empty Routes
If no viable routes can be found, the `routes` array will be empty. The User Model must handle this case gracefully (e.g., mark the demand as unfulfilled or try different constraints).
:::

### Health Check Endpoint

**GET** `/health`

Returns the health status of the route planner.

**Response:**
```json
{
  "status": "healthy",
  "servicesLoaded": 3,
  "lastUpdate": "2024-02-10T10:30:00Z",
  "version": "1.0.0"
}
```

## Routing Algorithms

The route planner may use various algorithms depending on the implementation and requirements:

### Time-Dependent Shortest Path

For scheduled transit services, the planner uses time-dependent algorithms that account for:
- Service schedules and frequencies
- Transfer times and minimum connection times
- Service-specific boarding and alighting times

Common algorithms:
- **Dijkstra's algorithm** with time-dependent edge weights
- **RAPTOR** (Round-Based Public Transit Routing) for efficient multi-modal queries
- **CSA** (Connection Scan Algorithm) for fast schedule-based routing

### Spatial Indexing

To efficiently find nearby stops and services:
- **R-tree** or **Quadtree** for spatial indexing of stops
- **Geohashing** for quick proximity queries
- **Distance matrices** for precomputed walking distances

### Multi-Criteria Optimization

Routes may be evaluated on multiple criteria:
- **Pareto optimization**: Find routes that are not dominated on any criterion
- **Weighted sum**: Combine multiple objectives (time, cost, comfort) with weights
- **Lexicographic ordering**: Prioritize criteria in order (e.g., time first, then cost)

## Service Data Formats

The route planner supports standard mobility data formats:

### GTFS (General Transit Feed Specification)

Standard format for public transit schedules and geographic information. Key files:
- `stops.txt`: Stop locations and IDs
- `routes.txt`: Transit routes
- `trips.txt`: Specific trip instances
- `stop_times.txt`: Arrival/departure times at each stop
- `calendar.txt`: Service dates and days of operation

### GBFS (General Bikeshare Feed Specification)

JSON-based format for shared mobility services. Key endpoints:
- `system_information.json`: System metadata
- `station_information.json`: Station locations and capacities
- `station_status.json`: Real-time availability
- `free_bike_status.json`: Available vehicles for dockless systems

### GTFS-Flex

Extension of GTFS for demand-responsive and flexible transit services:
- `locations.geojson`: Service areas and zones
- `booking_rules.txt`: How to book trips
- `stop_times.txt` with `start_pickup_drop_off_window` and `end_pickup_drop_off_window`

## Common Operational Notes

### Data Loading and Updates

The route planner typically loads service data during initialization:
1. **Parse all service definitions** (GTFS, GBFS, etc.)
2. **Build internal graph representation** of the transport network
3. **Index stops and locations** for fast spatial queries
4. **Validate data integrity** (e.g., check for missing stops, invalid times)

For real-time simulations, consider:
- **Periodic data refreshes** to update GBFS availability
- **Service alert handling** for disruptions or delays
- **Hot-reload capabilities** to update data without restarting

### Handling Walking Segments

Walking is often part of multi-modal routes:
- **First/last mile**: Walking from origin to first stop, or from last stop to destination
- **Transfers**: Walking between stops during transfers
- **Default walking speed**: Typically 1.2-1.5 m/s (4.3-5.4 km/h)
- **Maximum walking distance**: Configurable, typically 500-1500 meters

Walking segments should be represented as special trip segments with `service: "walk"`.

### Transfer Handling

Transfers between services require careful handling:
- **Minimum transfer time**: Time needed to move between platforms/stops
- **Transfer penalties**: Additional time to discourage unnecessary transfers
- **Same-stop transfers**: May require no walking but still have minimum time
- **Cross-street transfers**: Require walking between nearby stops

### Time Windows and Flexibility

Route queries may specify:
- **Exact departure time**: Find routes leaving at or after this time
- **Exact arrival time**: Find routes arriving at or before this time (reverse search)
- **Time window**: Find all routes within a given time range

### Service Availability

The planner must respect service availability:
- **Operating days**: Services may not run every day (weekdays only, weekends, etc.)
- **Operating hours**: Services have start and end times
- **Special dates**: Holidays or special event schedules
- **Capacity constraints**: Some services may have limited capacity

### Coordinate Systems

- **Standard coordinates**: Latitude/Longitude in WGS84 (EPSG:4326)
- **Distance calculations**: Haversine formula for great-circle distance on Earth
- **Projection**: May use local projections for more accurate distance calculations

### Performance Optimization

For large-scale simulations:
- **Query caching**: Cache results for frequently requested routes
- **Precomputation**: Precompute common routes or distance matrices
- **Parallel processing**: Handle multiple route queries concurrently
- **Index optimization**: Use spatial and temporal indexes for fast lookups
- **Memory management**: Balance between memory usage and computation time

### Error Handling

The planner should handle various error conditions:
- **No routes found**: Origin/destination too far, no services available, time constraints too strict
- **Invalid inputs**: Missing coordinates, invalid times, malformed requests
- **Service data errors**: Missing stops, invalid schedules, inconsistent data
- **Timeout**: Complex queries that take too long to compute

Return meaningful error messages to help diagnose issues.

### Integration with User Model

The Route Planner is typically called by the User Model:
1. User Model receives a `DEMAND` event
2. User Model queries the Route Planner with origin, destination, and time constraints
3. Route Planner returns route options
4. User Model evaluates options and selects one
5. User Model emits `RESERVE` event for the selected route

The interface must be reliable and fast to avoid delaying the simulation.

### Integration with Mobility Services

The Route Planner suggests routes using specific mobility services:
- Service IDs in routes must match actual deployed services
- The planner doesn't reserve capacity - it only suggests possibilities
- Actual availability is confirmed by mobility services via `RESERVED` events
- Planner may use capacity hints if available (e.g., GBFS availability data)

### Testing and Validation

Recommended tests for route planners:
- **Correctness**: Do routes actually connect origin and destination?
- **Optimality**: Are routes reasonably efficient (not obviously suboptimal)?
- **Completeness**: Are all service modes considered?
- **Timing**: Are departure/arrival times realistic and consistent?
- **Edge cases**: Handle zero-distance trips, same origin/destination, unreachable locations
- **Performance**: Can handle expected query load without timeout?

### Debugging and Logging

Recommended logging for Route Planner:
- Log every route query received
- Log number of routes found and computation time
- Log when no routes are found (with reason if possible)
- Log service data loading and updates
- Log any errors or warnings during routing

This helps diagnose:
- Why are queries taking too long?
- Why are no routes being found?
- Are all services being considered?
- Is the service data loaded correctly?

### Known Limitations

Common limitations of route planners:
- **Static data**: Most planners use pre-loaded static schedules, not real-time data
- **Simplified walking**: Walking distances are often straight-line, not actual paths
- **Limited mode combinations**: Some mode combinations may not be supported
- **Computational complexity**: Finding optimal multi-modal routes is NP-hard
- **Capacity assumptions**: Planner assumes unlimited capacity unless explicitly modeled

Be aware of these limitations when interpreting simulation results.
