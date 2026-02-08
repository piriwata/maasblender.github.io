---
sidebar_position: 20
title: "User Model"
---

This chapter specifies the User Model components in MaaS Blender.
The User Model represents the decision-making logic of individual users throughout their journey.
It orchestrates trip planning, route selection, reservation, and execution by coordinating with planners and mobility services through event-driven interactions.

The project currently ships two reference implementations under `maasblender/src/user_model`:
- Simple User Model: for straightforward route selection with basic preference handling.
- Favorite-based User Model: for more sophisticated per-user preferences including favorite services and sorting criteria.

Both implementations share the same event contract and lifecycle but differ in route-selection policies and configuration options.

### Common Lifecycle

All User Model implementations follow this general flow:

1. **Demand intake**: The simulator forwards a `DEMAND` to the User Model via `UserManager.demand(...)`.
2. **Route planning**: The manager calls an external route planner (`Planner`) with `(org, dst, dept)`.
3. **Candidate selection**: The User Model ranks/filters candidates according to its selection policy (varies by implementation).
4. **Task construction**: The selected route is converted into a sequence of tasks (`Reserve`, `Trip`, `Wait`).
5. **Execution and reactions**:
   - The `User` entity runs the tasks in order, emitting `RESERVE` and/or `DEPART` at appropriate times.
   - On `RESERVED` failure, the User Model switches to a recovery task sequence (fallback to alternative plan or walking).
   - The User Model continues to observe `DEPARTED` and `ARRIVED` to orchestrate transfers and detect leg completion.

## Simple User Model

- File: `maasblender/src/user_model/simple/user_manager.py`
- Primary class: `UserManager`

This model provides a minimal, easy-to-understand user decision policy with a small set of preferences.

### Behavior

- **Planning**: Calls `Planner.plan(org, dst, dept)` to retrieve route candidates.
- **Selection**:
  - If the `DEMAND` provides `service`, applies preference mode:
    - `PreferenceMode.fixed`: keeps only plans that contain the specified service.
    - Otherwise: sorts so that plans containing the service come first.
  - If only 1 plan is returned, builds tasks for that plan with walking fallback for mobility legs.
  - If 2+ plans are returned, builds a primary sequence and sets the secondary as recovery; recovery is used if the primary mobility leg fails.
- **Reservation vs Direct Depart**:
  - If a route pattern matches "reservation-required" (heuristic: 3-trip route with the middle leg's `service` in `confirmed_services`), the manager builds a `Reserve` task for that leg.
  - Otherwise, legs are built as direct `Trip` tasks (e.g., walking or services without booking).
- **Waiting**:
  - If `dept > now` and the first task is a `Trip`, a `Wait(dept)` task is inserted before the first leg.
- **Failure Handling**:
  - If a mobility leg's reservation fails, the manager either uses:
    - A walking fallback for that leg, or
    - The secondary plan's sequence (possibly prefixed with a walking connector), with walking as the last resort.

### Configuration

```json
{
  "preference_mode": "fixed",                   // or any other value for prefer mode
  "confirmed_services": ["taxi", "bus"]         // services that require reservation
}
```

- `preference_mode` (from `jschema.query.PreferenceMode`):
  - `"fixed"`: filters plans to only those that include the specified service per demand.
  - Otherwise: prefers plans containing the specified service, but allows others.
- `confirmed_services`: list of services that require advance reservation.

#### Example

```json
{
  "preference_mode": "fixed",
  "confirmed_services": ["taxi", "shuttle"]
}
```

With this configuration:
- If a user's `DEMAND` specifies `service: "taxi"`, only routes containing taxi will be considered.
- Routes with 3 trips where the middle trip uses taxi or shuttle will generate a `Reserve` task.

### Output

- Creates `User` instances that execute task sequences.
- Emits `RESERVE`, `DEPART` events through the event manager.
- Returns `triggered_events` containing events to be processed by mobility services.

## Favorite-based User Model

- File: `maasblender/src/user_model/favorite/user_manager.py`
- Primary class: `UserManager`

This model provides richer, per-user route preferences including favorite service, walking time limits, and candidate sorting.

### Behavior

- **Per-user routing policy**: Each user gets a `RouteFilter` instance. If a `UserType` is provided, the filter becomes a `FavoriteSortedRouteFilter` that:
  - Sorts plans by `SortType` (e.g., earliest arrival, least transfers, lowest cost).
  - Applies checks for favorite service presence and enforces a walking time cap when applicable.
- **Demand-level override**:
  - If `fixed_service` is provided in the `DEMAND`, it takes priority: plans are filtered so that the given service is included.
  - If no plans include the fixed service, a warning is logged and original plans are used.
- **Task construction**: Waiting and failure handling are analogous to the Simple model, including reservation vs direct depart and walking fallbacks.

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

- `user_params`: dictionary mapping user IDs to their preferences (`UserType` or `null`):
  - `favorite_service`: preferred mobility service (e.g., `"rail"`, `"bus"`).
  - `walking_time_limit_min`: maximum acceptable walking time in minutes.
  - `sort_type`: sorting criterion from `jschema.query.SortType` (e.g., `"earliest_arrival"`, `"least_transfers"`, `"lowest_cost"`).
- `confirmed_services`: list of services that require advance reservation (same as Simple model).

#### Example

```json
{
  "user_params": {
    "U001": {
      "favorite_service": "rail",
      "walking_time_limit_min": 20.0,
      "sort_type": "earliest_arrival"
    }
  },
  "confirmed_services": ["taxi"]
}
```

User U001 prefers rail, accepts up to 20 minutes of walking, and wants the earliest arrival route. If multiple rail routes exist, the earliest one is selected first.

### Output

- Creates `User` instances with personalized route preferences.
- Emits `RESERVE`, `DEPART` events through the event manager.
- Returns `triggered_events` containing events to be processed by mobility services.

---

### Common Operational Notes

- Both User Model implementations use `simpy` for task orchestration and event-driven execution.
- Time Base: All times are in minutes from the simulation start.
- Task Types:
  - `Wait`: waits until departure time.
  - `Trip`: executes a single leg of the journey (walking or direct mobility service).
  - `Reserve`: handles advance reservation for confirmed services, then executes the reserved mobility leg.
  - `ReservedTrip`: executes a leg that was already reserved.
- Failure Recovery: Both models provide fallback mechanisms, defaulting to walking when mobility services fail.
- Event Flow: User Models consume `DEMAND` events and emit `RESERVE`, `DEPART` events that mobility services respond to with `RESERVED`, `DEPARTED`, `ARRIVED`.
