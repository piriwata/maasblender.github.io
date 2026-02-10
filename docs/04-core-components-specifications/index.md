---
sidebar_position: 4
---

# Core Components Specifications

This section documents the core components that make up the MaaS Blender simulation platform. These components work together to simulate multi-modal mobility services and evaluate Mobility-as-a-Service (MaaS) scenarios.

## Component Overview

MaaS Blender consists of several key components, each responsible for a specific aspect of the simulation:

### [Scenario Component](./10-scenario.md)

The **Scenario** component generates travel demands that drive the simulation. It determines who travels, when, where, and with what constraints.

- Supports multiple demand generation strategies (commuter patterns, synthetic generation, historical replay)
- Configurable demand parameters (times, locations, user preferences)
- Emits `DEMAND` events to initiate user journeys

### [User Model Component](./20-user-model.md)

The **User Model** component simulates user behavior and decision-making throughout their journey.

- Receives travel demands and evaluates routing options
- Makes reservation decisions based on user preferences
- Manages user state through the travel lifecycle
- Emits `RESERVE`, `DEPART`, and other user-action events

### [Route Planner Component](./30-route-planner.md)

The **Route Planner** component calculates multi-modal routes between locations.

- Analyzes available mobility services and schedules
- Computes viable routes combining multiple transport modes
- Returns route options with detailed timing and service information
- Supports various routing algorithms and data formats (GTFS, GBFS, GTFS-Flex)

## Component Interactions

These components interact through an event-driven architecture:

```
┌──────────────┐
│   Scenario   │
└──────┬───────┘
       │ DEMAND events
       ↓
┌──────────────┐      route queries     ┌──────────────┐
│  User Model  │ ←───────────────────→  │Route Planner │
└──────┬───────┘                        └──────────────┘
       │ RESERVE, DEPART events
       ↓
┌──────────────┐
│   Mobility   │
│   Services   │
└──────────────┘
```

1. **Scenario** generates `DEMAND` events representing user travel needs
2. **User Model** receives demands and queries the **Route Planner** for route options
3. **Route Planner** calculates and returns possible routes
4. **User Model** evaluates options and emits `RESERVE` events to book services
5. **Mobility Services** respond with `RESERVED` events (success/failure)
6. **User Model** emits `DEPART` events when users are ready to travel
7. **Mobility Services** emit `DEPARTED` and `ARRIVED` events as users travel

## Additional Components

While this section focuses on the core simulation components, MaaS Blender also includes:

- **Simulation Broker**: Coordinates event flow between components
- **Mobility Services**: Simulate specific mobility providers (buses, trains, shared bikes, on-demand vehicles)
- **Evaluation**: Analyzes simulation results and generates metrics

These additional components are documented in other sections.

## Common Patterns

All core components follow similar design patterns:

### REST API Interface

Each component exposes a REST API for:
- Initialization with configuration parameters
- Event processing (receiving and emitting events)
- Status queries and health checks

### Configuration via JSON

Components are configured using JSON files that specify:
- Behavior parameters (e.g., user preferences, routing constraints)
- Data sources (e.g., GTFS files, service definitions)
- Integration endpoints (e.g., URLs of other components)

### Event-Driven Communication

Components communicate asynchronously via events:
- Events are JSON objects with a standard structure
- Events include a timestamp and type identifier
- Events flow through the Simulation Broker
- Components process events and may emit new events in response

### Pluggable Implementations

Each component type supports multiple implementations:
- Different implementations offer different behaviors or capabilities
- Implementations can be swapped to test different scenarios
- Custom implementations can be added to extend functionality

## Development and Extension

To extend or customize MaaS Blender:

1. **Choose a component** to extend or replace
2. **Implement the component interface** (REST API endpoints)
3. **Follow the event specifications** for emitting and consuming events
4. **Configure the simulation** to use your custom component
5. **Test integration** with other components

Refer to the individual component documentation pages for detailed specifications and implementation guidelines.
