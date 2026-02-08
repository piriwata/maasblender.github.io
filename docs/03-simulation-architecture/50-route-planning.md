---
sidebar_position: 50
title: Route Planner
---

The Route Planner is a path-finding engine responsible for computing candidate routes between two locations.

It is primarily used by the User Model to evaluate travel options, 
but it may also be invoked by other components when route information is required.

## Interface and Communication Model

Unlike most components in MaaS Blender, the Route Planner does not communicate via events. 
Instead, it exposes its functionality through a REST API, 
allowing route queries such as origin and destination pairs, time constraints

This design reflects the Route Planner’s role as a computational service rather than a stateful simulation actor.

## Responsibilities

The Route Planner is responsible for:

- computing possible route candidates between locations,
- returning structured route descriptions suitable for evaluation by the User Model,
- supporting different planning strategies or algorithms, depending on the implementation.

The Route Planner does not initiate actions in the simulation and does not participate directly in the event flow.

### Data Model

All planners expose a `plan(org, dst, dept)` method returning a list of `Route`/`Path` candidates. Time units are minutes from the simulation’s reference start.

- `Location`: stop or coordinate identifier with `locationId`, `lat`, `lng`.
- `Trip`: a single leg with fields:
  - `org: Location`
  - `dst: Location`
  - `dept: float` (minutes since simulation start)
  - `arrv: float` (minutes since simulation start)
  - `service: string` (e.g., `walking`, `rail`, `bus`, `taxi`)
- `Route`: an ordered list of `Trip` legs representing a candidate route from `org` to `dst`.

## Assumptions and Limitations

To keep the planning process efficient and decoupled, the Route Planner operates under the following assumptions:

- No resource management
  The Route Planner does not manage vehicles, drivers, or passenger capacity.

- Unlimited availability assumption
  All mobility services and resources are assumed to be available when generating route candidates.

- Approximate service behavior
  The Route Planner is not required to fully reproduce the internal logic of mobility services, as doing so would require resource-level simulation.

As a result, the Route Planner provides possible routes, not guaranteed ones.

## Design Rationale

Route computation is intentionally separated from resource management so the Route Planner can remain stateless and reusable,
while routing algorithms can evolve independently without coupling to service execution.
