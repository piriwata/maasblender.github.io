---
sidebar_position: 10
title: Scenario
---

# Scenario

**Generates demand events based on configured scenario settings.**

The Scenario component is responsible for creating travel demand events during simulation. It emits `DEMAND` events at scheduled times based on predefined user behavior patterns. These events represent user requests for travel and serve as the starting point for the simulation flow.

## Implementations

MaaS Blender provides the following scenario implementations:

- **commuter** - Generates commuter travel patterns with daily outbound and inbound trips
- **historical** - Replays historical demand data from past events or records
- **generator** - Creates demand based on configurable generation rules

## Commuter

The commuter scenario generates recurring daily demand patterns typical of commuters who travel between home and work locations.

**File:** `src/scenario/commuter`  
**Primary Class:** `CommuterScenario`

### Behavior

The commuter scenario creates two demand events per day for each configured commuter:
1. Outbound trip - departure from origin to destination at configured morning time
2. Inbound trip - return trip from destination back to origin at configured evening time

The scenario repeats this pattern daily (every 1440 time units) throughout the simulation period. Each demand is assigned a unique demand ID using the configured format string.

### Configuration

The commuter scenario is configured via the `/setup` endpoint with the following structure:

```json
{
  "commuters": {
    "user-001": {
      "deptOut": 480.0,  // Morning departure time (8:00 AM in minutes)
      "deptIn": 1020.0,  // Evening departure time (5:00 PM in minutes)
      "org": {
        "locationId": "home-001",
        "lat": 35.0,
        "lng": 135.0
      },
      "dst": {
        "locationId": "work-001",
        "lat": 35.1,
        "lng": 135.1
      },
      "user_type": "commuter",  // Optional: user type for classification
      "service": null  // Optional: preferred service (null = no preference)
    }
  },
  "demandIDFormat": "D_%d"  // Format string for demand IDs
}
```

**Key Configuration Parameters:**

- `deptOut` - Departure time for outbound trip (in simulation time units)
- `deptIn` - Departure time for inbound/return trip (in simulation time units)
- `org` - Origin location (typically home)
- `dst` - Destination location (typically workplace)
- `user_type` - Optional classification of user type
- `service` - Optional preferred mobility service
- `demandIDFormat` - Python format string for generating unique demand IDs (e.g., "D_%d" generates "D_1", "D_2", etc.)

:::info
The scenario uses SimPy's discrete event simulation environment to schedule and emit demand events. Time advances in discrete steps via the `/step` endpoint.
:::

### API Endpoints

The commuter scenario implements the standard component API:

- `GET /spec` - Returns component specification
- `POST /setup` - Configure scenario with commuter settings
- `GET /users` - Returns list of configured users
- `POST /start` - Starts the scenario generation
- `GET /peek` - Returns the next scheduled event time
- `POST /step` - Advances simulation and returns generated events
- `POST /triggered` - Receives triggered events (not used by scenario)
- `POST /finish` - Cleanup and shutdown

## Historical

The historical scenario replays demand events from historical data or logs.

**File:** `src/scenario/historical`  
**Primary Class:** `HistoricalScenario`

### Behavior

Reads demand events from historical data sources and replays them at their original timestamps. This is useful for:
- Validating simulation accuracy against real-world data
- Testing "what-if" scenarios with actual historical demand patterns
- Reproducing specific scenarios for debugging or analysis

## Generator

The generator scenario creates synthetic demand based on configurable rules and distributions.

**File:** `src/scenario/generator`  
**Primary Class:** `GeneratorScenario`

### Behavior

Generates demand events using statistical distributions and rules such as:
- Random origin-destination pairs from a set of locations
- Configurable demand rates and temporal patterns
- Various probability distributions for timing and spatial patterns

This is useful for stress testing and generating diverse traffic patterns.

## Common Operational Notes

### Event Format

All scenario implementations emit demand events following the standard [Event Specification](../02-simulation-model/10-event.md#demand-event) format with `eventType: "DEMAND"`.

### Simulation Time

Scenarios use discrete time units (typically minutes or seconds). The component advances time via the `/step` endpoint and returns events scheduled for the current simulation time.

### Integration

The Scenario component is typically the first component to emit events in the simulation pipeline. These `DEMAND` events trigger downstream components:
1. User model component receives demands and evaluates route options
2. Route planner generates candidate routes
3. User model selects a route and emits `RESERVE` events

:::tip
Use the `/peek` endpoint to check the next scheduled event time before calling `/step`. This allows efficient simulation advancement without unnecessary step calls.
:::
