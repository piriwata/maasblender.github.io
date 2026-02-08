---
sidebar_position: 20
title: "Scenario: Demand Generator"
---

This section specifies the User Model components in MaaS Blender.
A User Model represents the decision-making logic of users and orchestrates trip planning, reservation, and execution via events.

The project currently ships two reference implementations under `maasblender/src/user_model`:
- Simple User Model: `user_model/simple/user_manager.py` (class `UserManager`)
- Favorite-based User Model: `user_model/favorite/user_manager.py` (class `UserManager`)

Both implementations share the same event contract and lifecycle but differ in route-selection policies and configuration knobs.

#### Common Lifecycle

1. Demand intake: The simulator forwards a `DEMAND` to the User Model via `UserManager.demand(...)`.
2. Route planning: The manager calls an external route planner (`Planner`) with `(org, dst, dept)`.
3. Candidate selection: The User Model ranks/filters candidates by its selection policy (varies by implementation).
4. Execution and reactions
- The `User` entity runs the tasks in order, emitting `RESERVE` and/or `DEPART` at appropriate times.
- On `RESERVED` failure, the User Model switches to a recovery task sequence (fallback to alternative plan or walking).
- The User Model continues to observe `DEPARTED` and `ARRIVED` to orchestrate transfers and detect leg completion.

## Simple User Model

File: `maasblender/src/user_model/simple/user_manager.py`

This model provides a minimal, easy-to-understand user decision policy with a small set of preferences.

### Behavior

- Planning: `Planner.plan(org, dst, dept)` returns candidates.
- Selection:
  - If the `DEMAND` provides `service`, apply preference:
    - `PreferenceMode.fixed`: keep only plans that contain the service.
    - Otherwise: sort so that plans containing the service come first.
  - If there is only 1 plan, build tasks for that plan with a walking fallback where needed.
  - If there are 2+ plans, build a primary sequence and a recovery sequence; the recovery is used if the primary mobility leg fails.
- Reservation vs direct depart:
  - If a route pattern matches “reservation-required” (heuristic: 3-trip route with the middle leg’s `service` in `confirmed_services`), the manager builds a `Reserve` task for that leg.
  - Otherwise, legs are built as direct `Trip` tasks (e.g., walking or services without booking).
- Waiting:
  - If `dept > now` and the first task is a `Trip`, a `Wait(dept)` task is inserted before the first leg.
- Failure handling:
  - If a mobility leg’s reservation fails, the manager attaches either a walking fallback for that leg or the secondary plan’s sequence (possibly prefixed with a walking connector), ending with walking as last resort.

### Configuration

- `PreferenceMode` (from `jschema.query`):
  - `fixed`: filter to plans that include the specified service per demand
  - otherwise: prefer plans containing the specified service
- `confirmed_services: list[str]`: services that require reservation

#### Favorite-based User Model

File: `maasblender/src/user_model/favorite/user_manager.py`

This model provides richer, per-user route preferences including favorite service, walking time limits, and candidate sorting.

### Behavior

- Per-user routing policy: each user gets a `RouteFilter` instance. If a `UserType` is provided, the filter becomes a `FavoriteSortedRouteFilter` that:
  - Sorts plans by `SortType` (e.g., earliest arrival, etc.).
  - Applies checks for favorite service presence and enforces a walking time cap when applicable.
- Demand-level override:
  - If `fixed_service` is provided in the `DEMAND`, it takes priority: plans are filtered so that the given service is included; if none include it, a warning is logged and original plans are used.
- Task construction, waiting, and failure handling are analogous to the Simple model, including reservation vs direct depart and walking fallbacks.

### Configuration

- `user_params: dict[userId, UserType | None]` where `UserType` may include:
  - `favorite_service: str | None`
  - `walking_time_limit_min: float | None`
  - `sort_type: SortType` (from `jschema.query`), e.g., earliest arrival, least transfers, lowest cost (implementation-specific)
- `confirmed_services: list[str]` — same semantics as in the Simple model.
