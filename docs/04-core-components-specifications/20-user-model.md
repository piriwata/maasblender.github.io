---
sidebar_position: 20
title: "User Model: Trip Orchestration"
---

This chapter specifies the reference implementations for User Model components in MaaS Blender.
Each User Model represents the decision-making and trip-orchestration logic of individual users.
It converts a travel demand into executable tasks by coordinating route planning, service selection, reservation, and trip execution through event-driven interactions.

The project currently ships two reference implementations under `maasblender/src/user_model`:

- Simple User Model: straightforward route selection with basic preference handling.
- Favorite-based User Model: sophisticated per-user preferences including favorite services and sorting criteria.

## Common Lifecycle

All User Model implementations follow this event-driven flow:

1. **Demand intake**: The simulator forwards a `DEMAND` event to the User Model via `UserManager.demand(...)`.
2. **Route planning**: The manager calls an external Route Planner with `plan(org, dst, dept)` to obtain route candidates.
3. **Candidate selection**: The User Model ranks and filters candidates according to its selection policy (varies by implementation).
4. **Task construction**: The selected route is converted into a sequence of executable tasks (`Wait`, `Reserve`, `Trip`).
5. **Task execution**:
   - The `User` entity executes tasks in order, emitting `RESERVE` and/or `DEPART` events at appropriate times.
   - On `RESERVED` failure, the User Model switches to a recovery sequence (fallback to alternative route or walking).
   - The User Model observes `DEPARTED` and `ARRIVED` events to orchestrate transfers and detect trip completion.

## Simple User Model (`UserManager`)

- File: `maasblender/src/user_model/simple/manager.py`
- Primary class: `UserManager`

This model implements straightforward routing with basic service preferences and two-plan fallback logic.

### Behavior

- **Route Planning**: Calls `Planner.plan(org, dst, dept)` to retrieve route candidates.
- **Route Selection**:
  - If the `DEMAND` specifies a `service`, applies the configured preference mode:
    - `PreferenceMode.fixed`: filters to keep only routes containing the specified service.
    - Other modes: sorts routes so that those containing the specified service come first.
  - If only 1 route is returned, builds tasks for that route with walking fallback if the mobility leg fails.
  - If 2+ routes are returned, uses the first as primary and the second as fallback; the fallback is activated if the primary mobility leg reservation fails.
- **Task Construction**:
  - **Waiting**: If `dept > now` and the first task is a `Trip`, inserts a `Wait(dept)` task before the trip.
  - **Reservation vs Direct Depart**: If a route matches the "reservation-required" pattern (typically a 3-leg route where the middle leg's service is in `confirmed_services`), a `Reserve` task is created for that leg. Otherwise, legs use direct `Trip` tasks.
- **Failure Recovery**:
  - If a mobility leg's reservation fails, the manager activates one of:
    - A walking fallback for that specific leg, or
    - The secondary route's task sequence (with a walking connector if needed), or
    - Walking as the final fallback.

### Configuration

```json
{
  "preference_mode": "fixed",                   // or any other value for prefer mode
  "confirmed_services": ["taxi", "bus"]         // services that require reservation
}
```

**Parameters:**
- `preference_mode` (from `jschema.query.PreferenceMode`):
  - `"fixed"`: filters routes to only those that include the user's specified service.
  - Other values: prefers routes containing the specified service but allows others.
- `confirmed_services`: list of service names that require reservation before use.

:::info
The Simple User Model is appropriate for scenarios with homogeneous user populations or when individual preferences are not critical to simulation outcomes.
:::

## Favorite-based User Model (`UserManager`)

- File: `maasblender/src/user_model/favorite/manager.py`
- Primary class: `UserManager`

This model provides rich, per-user routing preferences including favorite service, walking time limits, and candidate sorting criteria.

### Behavior

- **Per-user Routing Policy**: Each user is assigned a `RouteFilter` instance. If a `UserType` is provided, the filter becomes a `FavoriteSortedRouteFilter` that:
  - Sorts route candidates by the specified `SortType` (e.g., earliest arrival, least transfers, lowest cost).
  - Checks for favorite service presence in routes.
  - Enforces a maximum walking time limit when applicable.
- **Demand-level Override**:
  - If `fixed_service` is specified in the `DEMAND`, it takes priority: routes are filtered to include only those containing the given service.
  - If no routes contain the fixed service, a warning is logged and the original candidates are used.
- **Task Construction**: Similar to the Simple model, including waiting logic, reservation vs direct depart decisions, and failure recovery with walking fallbacks.

### Configuration

```json
{
  "user_params": {
    "U001": {
      "favorite_service": "rail",
      "walking_time_limit_min": 15.0,
      "sort_type": "earliest_arrival"
    },
    "U002": {
      "favorite_service": "bus",
      "walking_time_limit_min": 10.0,
      "sort_type": "least_transfers"
    },
    "U003": null
  },
  "confirmed_services": ["taxi", "shuttle"]
}
```

**Parameters:**
- `user_params`: dictionary mapping user IDs to their preferences (`UserType` or `null`):
  - `favorite_service`: preferred mobility service (e.g., `"rail"`, `"bus"`).
  - `walking_time_limit_min`: maximum acceptable walking time in minutes.
  - `sort_type`: sorting criterion from `jschema.query.SortType`:
    - `"earliest_arrival"`: prioritize routes with earliest arrival time.
    - `"least_transfers"`: prioritize routes with fewest transfers.
    - `"lowest_cost"`: prioritize routes with lowest cost (if cost data is available).
- `confirmed_services`: list of services that require reservation (same as Simple model).

:::tip
The Favorite-based User Model is ideal for simulations where heterogeneous user populations with diverse preferences significantly impact system behavior and outcomes.
:::

---

### Common Operational Notes

- **Event Contract**: Both models consume `DEMAND` events and emit `RESERVE` and `DEPART` events. They observe `RESERVED`, `DEPARTED`, and `ARRIVED` responses.
- **Time Base**: All times are in minutes from simulation start.
- **Determinism**: Given the same route candidates from the planner, both models produce deterministic task sequences.
- **Extensibility**: New user models can be created by implementing the same lifecycle and event contract while customizing the selection and task construction logic.
