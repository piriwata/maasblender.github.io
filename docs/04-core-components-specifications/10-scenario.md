---
sidebar_position: 10
title: Scenario
---

# Scenario Component

The **Scenario** component is responsible for generating travel demands that drive the simulation. It reads configuration data and emits `DEMAND` events at specified times, representing users who need to travel from one location to another.

## Overview

The Scenario component acts as the demand generator for the entire simulation. It determines:

- **Who** travels (user identifiers)
- **When** they want to travel (departure or arrival times)
- **Where** they travel from and to (origin and destination locations)
- **What** service constraints they may have (optional service preferences)

By configuring different scenario parameters, you can simulate various real-world situations such as commuter patterns, event-based travel, or historical trip data replay.

## Available Implementations

MaaS Blender provides multiple scenario implementations to support different use cases:

### 1. Commuter Scenario

**File:** `src/scenario/commuter/`  
**Primary class:** `Commuter` (in `commuter.py`)

The Commuter scenario generates regular commuter traffic patterns. It simulates people traveling between home and work locations at specified times, supporting typical morning and evening commute patterns.

#### Behavior

- Generates `DEMAND` events for users at configured times
- Supports both **leave-at** (departure time specified) and **arrive-by** (arrival time specified) demands
- Can generate return trips (e.g., evening commute back home)
- Emits events in chronological order during simulation initialization or at scheduled times

#### Configuration

The Commuter scenario is configured via a JSON configuration file. Key parameters include:

```json
{
  "users": [
    {
      "userId": "user-001",  // Unique identifier for the user
      "demandId": "dmd-001", // Unique identifier for this demand
      "org": {               // Origin location
        "locationId": "home-001",
        "lat": 35.681236,
        "lng": 139.767125
      },
      "dst": {               // Destination location
        "locationId": "office-001",
        "lat": 35.689487,
        "lng": 139.691711
      },
      "dept": 28800,         // Departure time (seconds since simulation start)
      "arrv": null           // Arrival time (null for leave-at demands)
    }
  ]
}
```

:::info Configuration Notes
- Either `dept` or `arrv` must be `null` - you cannot specify both
- Times are in seconds since the simulation start time (0)
- Location coordinates use standard WGS84 latitude/longitude
- The `service` field is optional and can be used to bind demands to specific mobility services
:::

### 2. Generator Scenario

**File:** `src/scenario/generator/`  
**Primary class:** `Generator` (in `generator.py`)

The Generator scenario creates synthetic travel demands based on statistical distributions and parameters. It can generate random or semi-random demand patterns for testing and experimentation.

#### Behavior

- Generates demands using probability distributions
- Supports various demand generation strategies (uniform, Poisson, time-based patterns)
- Can create large-scale demand sets for stress testing
- Allows randomization while maintaining reproducibility via seed values

#### Configuration

```json
{
  "seed": 42,                // Random seed for reproducibility
  "demandCount": 100,        // Total number of demands to generate
  "timeWindow": {
    "start": 0,              // Start time for demand generation
    "end": 86400             // End time (24 hours in seconds)
  },
  "locations": [             // Available locations for origin/destination
    {
      "locationId": "loc-001",
      "lat": 35.681236,
      "lng": 139.767125
    }
  ],
  "distribution": "uniform"  // Time distribution: "uniform", "poisson", "normal"
}
```

### 3. Historical Scenario

**File:** `src/scenario/historical/`  
**Primary class:** `Historical` (in `historical.py`)

The Historical scenario replays actual trip data from recorded logs or external data sources. This is useful for validating simulation results against real-world observations.

#### Behavior

- Reads demand data from external files (CSV, JSON, or database)
- Replays demands in their original temporal sequence
- Can apply time scaling (e.g., replay 24 hours in 1 hour of simulation time)
- Supports filtering and transformation of historical data

#### Configuration

```json
{
  "dataSource": "/path/to/historical/data.csv",  // Path to data file
  "timeScale": 1.0,          // Time scaling factor (1.0 = real-time)
  "startOffset": 0,          // Offset to add to all timestamps
  "filters": {
    "startTime": 0,          // Only include demands after this time
    "endTime": 86400,        // Only include demands before this time
    "locationIds": []        // Filter by specific locations (empty = all)
  }
}
```

## Component Interface

All scenario implementations expose a REST API for interacting with the simulation broker.

### Initialization Endpoint

**POST** `/initialize`

Initializes the scenario component with configuration parameters.

**Request Body:**
```json
{
  "config": {
    // Scenario-specific configuration object
  }
}
```

**Response:**
```json
{
  "status": "initialized"
}
```

### Event Streaming

The scenario component emits `DEMAND` events to the simulation broker as the simulation progresses. Events are sent via the broker's event ingestion API.

## Common Operational Notes

### Demand ID Management

Each demand must have a unique `demandId` within the simulation. If generating multiple demands for the same user, ensure each has a distinct identifier.

### Time Management

- All times are relative to the simulation start (time = 0)
- Times are expressed in seconds (floating-point values supported for sub-second precision)
- Negative times are not allowed

### Location Specifications

- Locations must include valid latitude and longitude coordinates
- The `locationId` field is used for referencing locations in logs and analysis
- Coordinates should match the geographic area covered by your mobility services

### Service Constraints

The optional `service` field in demands allows binding a request to a specific mobility service. This is useful for:
- Simulating user preferences (e.g., "I only want to use the bus")
- Testing service-specific scenarios
- Modeling loyalty programs or subscriptions

### Event Ordering

The Scenario component must emit `DEMAND` events in chronological order based on the event `time` field. The simulation broker may reject or mishandle out-of-order events.

### Performance Considerations

- Large-scale scenarios (>10,000 demands) should pre-load and index data during initialization
- Consider using batch initialization rather than dynamic generation for better performance
- Memory usage scales with the number of pending demands

### Integration with Other Components

The Scenario component only generates demands. It does not:
- Make routing decisions (handled by User Model and Route Planner)
- Manage reservations (handled by User Model and Mobility Services)
- Track user state or trip progress (handled by User Model)

After emitting a `DEMAND` event, the Scenario component's responsibility for that demand ends. The User Model takes over to manage the user's journey.
