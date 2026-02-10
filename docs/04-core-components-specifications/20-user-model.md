---
sidebar_position: 20
title: User Model
---

# User Model Component

The **User Model** component simulates user behavior throughout their journey in the MaaS system. It receives `DEMAND` events from the Scenario component, evaluates routing options, makes reservation decisions, and manages the user's travel lifecycle until they reach their destination.

## Overview

The User Model component acts as the decision-making agent for each simulated user. It:

- **Receives** travel demands from the Scenario component
- **Requests** route options from the Route Planner component
- **Evaluates** available routes based on user preferences and constraints
- **Reserves** selected routes with Mobility Services
- **Manages** user state through the travel lifecycle (reserve, depart, arrive)
- **Emits** events that reflect user actions and state changes

This component bridges the gap between abstract travel demands and concrete trip reservations, simulating realistic user decision-making patterns.

## Available Implementations

MaaS Blender provides multiple user model implementations to represent different user behaviors:

### 1. Simple User Model

**File:** `src/user_model/simple/`  
**Primary class:** `UserManager` (in `user_manager.py`)

The Simple user model implements straightforward decision logic where users select the first available route that meets their time constraints.

#### Behavior

When a `DEMAND` event is received, the Simple user model:

1. **Queries the Route Planner** for available routes between origin and destination
2. **Selects the first acceptable route** from the returned options
3. **Emits a `RESERVE` event** to request a reservation with the mobility service
4. **Waits for a `RESERVED` response** from the mobility service
5. If reservation succeeds:
   - **Tracks the user's itinerary** and scheduled departure times
   - **Emits a `DEPART` event** at the appropriate time before each trip segment
   - **Monitors `DEPARTED` and `ARRIVED` events** to track progress
6. If reservation fails:
   - **May retry** with alternative routes (configurable)
   - **Marks the demand as unfulfilled** if no routes are available

The user model maintains state for each active user, tracking:
- Current location
- Scheduled trips
- Reservation status
- Progress through multi-segment routes

#### Configuration

```json
{
  "planner": {
    "endpoint": "http://localhost:5001",  // Route planner API endpoint
    "timeout": 30                         // Request timeout in seconds
  },
  "behavior": {
    "departureLeadTime": 300,      // Seconds before departure to emit DEPART event (default: 5 minutes)
    "maxRetries": 3,               // Maximum reservation retry attempts
    "retryDelay": 60,              // Seconds to wait between retries
    "routeSelectionStrategy": "first"  // Strategy: "first", "fastest", "cheapest"
  },
  "preferences": {
    "maxWalkingDistance": 1000,   // Maximum walking distance in meters (optional)
    "maxWaitTime": 900,            // Maximum waiting time in seconds (optional)
    "preferredModes": []           // Preferred mobility modes (empty = no preference)
  }
}
```

:::info Departure Lead Time
The `departureLeadTime` parameter controls when the `DEPART` event is emitted relative to the scheduled departure time. This simulates the time needed for a user to reach the departure point (e.g., walking to a bus stop). A typical value is 300 seconds (5 minutes).
:::

### 2. Favorite User Model

**File:** `src/user_model/favorite/`  
**Primary class:** `FavoriteUserManager` (in `favorite_user_manager.py`)

The Favorite user model simulates users who have preferences for specific services or routes based on past experience.

#### Behavior

This model extends the Simple user model with additional decision-making logic:

1. **Checks for favorite routes** matching the origin-destination pair
2. **Prioritizes familiar services** the user has used successfully before
3. **Falls back to general route search** if no favorites are applicable
4. **Updates preferences** based on successful trips (learning behavior)
5. **Applies loyalty or subscription constraints** if configured

The favorite model maintains per-user state including:
- History of successfully completed trips
- Preferred services and routes
- Service ratings or satisfaction scores
- Subscription or membership status

#### Configuration

```json
{
  "planner": {
    "endpoint": "http://localhost:5001",
    "timeout": 30
  },
  "behavior": {
    "departureLeadTime": 300,
    "maxRetries": 3,
    "retryDelay": 60,
    "routeSelectionStrategy": "favorite"  // Prioritize favorites
  },
  "preferences": {
    "maxWalkingDistance": 1000,
    "maxWaitTime": 900,
    "preferredModes": []
  },
  "favorites": {
    "learningEnabled": true,       // Learn from successful trips
    "initialFavorites": {},        // Pre-configured favorite routes per user
    "favoriteWeight": 2.0,         // Preference multiplier for favorites
    "minTripsToFavorite": 3        // Minimum successful trips to mark as favorite
  }
}
```

## Component Interface

All user model implementations expose a REST API for receiving events and managing user state.

### Event Processing Endpoint

**POST** `/event`

Processes incoming events from the simulation broker.

**Request Body:**
```json
{
  "eventType": "DEMAND",  // Event type: DEMAND, RESERVED, DEPARTED, ARRIVED
  "time": 120.5,
  "details": {
    // Event-specific details
  }
}
```

**Response:**
```json
{
  "status": "processed"
}
```

The user model processes different event types:

#### Processing DEMAND Events

1. Validates the demand (required fields, time constraints)
2. Queries the Route Planner for route options
3. Evaluates and selects a route
4. Emits a `RESERVE` event if a route is selected
5. Updates internal state to track the pending reservation

#### Processing RESERVED Events

1. Checks if the reservation succeeded or failed
2. If successful:
   - Stores the assigned route
   - Schedules future `DEPART` events
   - Updates user state to "reserved"
3. If failed:
   - May retry with an alternative route
   - Or marks the demand as unfulfilled

#### Processing DEPARTED Events

1. Verifies the departure matches expected trip segment
2. Updates user state to "in-transit"
3. Expects corresponding `ARRIVED` event

#### Processing ARRIVED Events

1. Confirms arrival at expected location
2. If final destination: Marks trip as complete
3. If intermediate stop: Prepares for next segment departure

### Query Endpoint

**GET** `/users/{userId}/status`

Retrieves current status of a specific user.

**Response:**
```json
{
  "userId": "user-001",
  "state": "in-transit",  // State: idle, planning, reserved, waiting, in-transit, arrived
  "currentLocation": {
    "locationId": "A",
    "lat": 35.0,
    "lng": 135.0
  },
  "activeDemand": "dmd-001",
  "route": [
    // Current route segments
  ]
}
```

## Event Emission Flow

The User Model emits several types of events during a user's journey:

### 1. RESERVE Event

Emitted when the user decides to book a specific route.

```json
{
  "eventType": "RESERVE",
  "time": 121.0,
  "service": "service-001",
  "details": {
    "userId": "user-001",
    "demandId": "dmd-001",
    "org": { "locationId": "A", "lat": 35.0, "lng": 135.0 },
    "dst": { "locationId": "B", "lat": 35.1, "lng": 135.1 },
    "dept": 130.0
  }
}
```

### 2. DEPART Event

Emitted when the user is ready to begin a trip segment.

```json
{
  "eventType": "DEPART",
  "time": 129.5,          // dept - departureLeadTime
  "service": "service-001",
  "details": {
    "userId": "user-001",
    "demandId": "dmd-001"
  }
}
```

:::warning Timing Consideration
The `DEPART` event is typically emitted **before** the scheduled departure time to allow the user to reach the departure location. The lead time is configured via `departureLeadTime`.
:::

### 3. DEPARTED Event

Emitted when the user actually leaves the origin location (may be emitted by the mobility service or user model, depending on configuration).

### 4. ARRIVED Event

Emitted when the user reaches a destination (may be emitted by the mobility service or user model).

## Route Selection Strategies

The user model must choose among multiple route options returned by the planner. Different strategies can be implemented:

### First Available Strategy

Selects the first valid route in the list. Simple but may not reflect realistic user preferences.

### Fastest Route Strategy

Selects the route with the shortest total travel time (arrival time - departure time).

### Cheapest Route Strategy

Selects the route with the lowest total cost (requires cost information in route data).

### Favorite Service Strategy

Prioritizes routes using services the user has successfully used before.

### Multi-Criteria Strategy

Evaluates routes based on weighted combination of factors:
- Travel time
- Cost
- Number of transfers
- Walking distance
- Service preferences

## Common Operational Notes

### State Management

The User Model must maintain accurate state for each user:
- **Idle**: No active travel demand
- **Planning**: Evaluating route options
- **Reserved**: Route booked, awaiting departure
- **Waiting**: At departure location, ready to depart
- **In-transit**: Currently traveling
- **Arrived**: Reached destination

State transitions must follow the expected event lifecycle. Unexpected events (e.g., `ARRIVED` when state is "idle") should be logged as errors.

### Handling Reservation Failures

When a reservation is rejected:
1. **Check retry budget**: Have we exceeded `maxRetries`?
2. **Request alternative routes**: Query the planner with updated constraints
3. **Select next best option**: If available
4. **Mark as unfulfilled**: If no alternatives exist

Failed reservations should be logged for analysis to identify capacity issues or planning problems.

### Multi-Segment Journeys

For routes with multiple segments (e.g., bus + train + bus):
1. User Model manages each segment as a separate sub-trip
2. `RESERVE` may be sent to multiple services (one per segment)
3. Each segment has its own `DEPART` → `DEPARTED` → `ARRIVED` cycle
4. User must complete segment N before starting segment N+1
5. Missing a connection requires re-planning

### Time Synchronization

The User Model must track simulation time accurately:
- Event emissions must respect chronological order
- `DEPART` events must account for lead time
- Don't emit events with timestamps in the past

### Performance Considerations

- **User state caching**: Keep active user state in memory for fast lookups
- **Event batching**: Process multiple events in a single transaction when possible
- **Asynchronous planning**: Query the planner asynchronously to avoid blocking
- **Memory management**: Clean up state for completed trips

### Integration with Route Planner

The User Model depends heavily on the Route Planner:
- Must handle planner timeouts gracefully (retry or fallback)
- Cache frequently requested routes to reduce planner load
- Validate planner responses before using them
- Report planner errors to the simulation broker

### Integration with Mobility Services

The User Model interacts with mobility services via `RESERVE` events:
- Services respond with `RESERVED` events (success/failure)
- Services emit `DEPARTED` and `ARRIVED` events for mobility tracking
- User Model should correlate these events with expected user journeys

### Debugging and Logging

Recommended logging for User Model:
- Log every demand received
- Log route queries and responses
- Log reservation attempts and outcomes
- Log state transitions for each user
- Log any error conditions or unexpected events

This helps diagnose issues like:
- Why did a reservation fail?
- Why did a user not depart on time?
- What routes were considered but rejected?
