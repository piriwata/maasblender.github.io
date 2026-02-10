---
sidebar_position: 4
---

# Core Components Specifications

This section documents the core components that make up the MaaS Blender simulation platform. Each component has a specific role in the simulation pipeline and communicates via standardized REST APIs and event messages.

## Component Architecture

MaaS Blender follows a microservices architecture where each component:
- Runs as an independent service (typically in Docker containers)
- Implements a standard REST API for lifecycle management
- Produces and consumes events following the [Event Specification](../02-simulation-model/10-event.md)
- Advances simulation time via discrete `/step` calls

## Core Components

- **[Scenario](./10-scenario.md)** - Generates travel demand events based on configured user behavior patterns
- **User Model** - Models user decision-making and route selection behavior
- **Route Planner** - Provides multimodal route planning and journey alternatives
- **Mobility Service** - Simulates transportation services (buses, on-demand vehicles, etc.)
- **[Evaluation](./50-evaluation.md)** - Evaluates service usability and logs analysis data

## Standard Component API

All core components implement a standard REST API for simulation control:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/spec` | GET | Returns component specification and event triggers |
| `/setup` | POST | Configures component with settings |
| `/start` | POST | Initializes component for simulation start |
| `/peek` | GET | Returns next scheduled event time |
| `/step` | POST | Advances simulation by one time step |
| `/triggered` | POST | Receives events from other components |
| `/finish` | POST | Cleanup and shutdown |

## Simulation Flow

A typical simulation proceeds as follows:

1. **Setup Phase** - Configure all components via `/setup` endpoints
2. **Start Phase** - Initialize components via `/start` endpoints
3. **Simulation Loop**:
   - Query `/peek` to find next event time across all components
   - Call `/step` on components that have events at that time
   - Components emit events that trigger other components via `/triggered`
   - Repeat until simulation end time
4. **Finish Phase** - Cleanup and collect results via `/finish` endpoints

## Event-Driven Communication

Components communicate asynchronously through events:

```
┌──────────┐  DEMAND   ┌───────────┐  RESERVE   ┌──────────┐
│ Scenario │─────────>│User Model │──────────>│ Mobility │
└──────────┘           └───────────┘            │ Service  │
                             │                  └──────────┘
                             │ DEMAND                │
                             v                       │ RESERVED
                      ┌────────────┐                 │
                      │ Evaluation │<────────────────┘
                      └────────────┘
```

Each component processes specific event types and may produce new events in response.

## Configuration and Deployment

Components are typically deployed using Docker Compose with environment-specific configuration. See the [Quick Start](../01-quick-start/index.md) guide for deployment examples.
