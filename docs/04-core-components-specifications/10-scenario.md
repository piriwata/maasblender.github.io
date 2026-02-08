---
sidebar_position: 10
title: "Scenario: Demand Generator"
---

This chapter specifies the reference implementations for demand-generation components in MaaS Blender.
Each component emits `DEMAND` events that kick off user activity in the simulation.
They differ by how and when those demands are created.

The project currently ships two reference implementations under `maasblender/src/scenario`:

- Stochastic Window Generator: to simulate aggregate, probabilistic flows over time windows.
- Historical Replay: when you need exact timing and content of demands (replay/benchmark).
- Commuter Pattern: to model recurring daily commuter behavior at specified times.

## Stochastic Window Generator (`DemandGenerator`)

- File: `maasblender/src/scenario/generator/generator.py`
- Primary class: `DemandGenerator`

### Behavior
- You define one or more windows `[begin, end)` with an expected number of demands in that window.
- The generator discretizes the window into 1-minute slots and, for each slot,
  draws a Bernoulli trial with probability:
  `p = expected_demands / number_of_slots`.
- For each success, a demand is created at that slot’s time.
- If `resv` (reservation time) is provided, the `DEMAND` event is emitted at `resv`
  with `dept` set to the intended departure time (the minute-slot inside `[begin, end)`).

### Configuration

```json
{
  "seed": 123,                               // RNG seed for reproducibility
  "userIDFormat": "U%03d",                 // e.g., U001, U002, ...
  "demandIDFormat": "D_%d",                // e.g., D_1, D_2, ... 
  "demands": [
    // you can use multiple overlapping windows
    {
      "begin": 10.0,                        // window start (minutes)
      "end": 200.0,                         // window end (minutes)
      "expected_demands": 2.0,              // expected count within the window
      "resv": 5.0,                          // reservation time (minutes) — optional
      "org": {"locationId": "Org", "lat": 35.0, "lng": 139.0},
      "dst": {"locationId": "Dst", "lat": 35.7, "lng": 139.7},
      "user_type": "commuter",             // optional
      "service": "my-mobility-service"     // optional
    }
  ]
}
```

:::warning
If `p > 0.1` per minute, use a smaller unit time to better approximate a Poisson process.
:::

#### Example

```json
{
  "seed": 128,
  "userIDFormat": "U%03d",
  "demands": [
    {
      "begin": 10.0,
      "end": 200.0,
      "expected_demands": 2.0,
      "resv": 7.0,
      "org": {"locationId": "Org", "lat": 35.0, "lng": 139.0},
      "dst": {"locationId": "Dst", "lat": 35.7, "lng": 139.7},
      "service": "mobility-service-for-test"
    }
  ]
}
```
This configuration usually yields two reservation demands around minute 7, each with a different `dept` inside the `[10, 200)` window.

### Output
- `users()` returns the list of created users.
- `DEMAND` events carry `userId`, auto-assigned `demandId`, `org`, `dst`, optional `service`, and:
  - For advance reservations: `dept = intended_departure_time`.
  - For immediate travel: no `dept` field (or `null`), meaning “depart now.”

## Historical Replay (`HistoricalScenario`)

- File: `maasblender/src/scenario/historical/historical.py`
- Primary class: `HistoricalScenario`

### Behavior
- Each input record specifies exactly when to emit a `DEMAND` and its full payload.
- If a record lacks `user_id` or `demand_id`, the scenario fills them using provided formats (e.g., `U%03d`, `D_%d`).

### Configuration
The setup accepts a collection of `HistoricalDemandSetting` items, plus two ID formats:

```json
{
  "user_id_format": "U%03d",          // used when a record has no user_id
  "demand_id_format": "D_%d",         // used when a record has no demand_id
  "settings": [
    {
      "time": 480.0,                   // exact emission time (minutes since start)
      "org": {"locationId": "A", "lat": 35.1, "lng": 139.1},
      "dst": {"locationId": "B", "lat": 35.7, "lng": 139.7},
      "dept": 495.0,                   // intended departure time
      "user_id": "U123",              // optional; auto-assigned if missing
      "demand_id": "D_901",           // optional; auto-assigned if missing
      "user_type": "visitor",         // optional; associated to user_id
      "service": "bus-line-12",       // optional
      "actual_duration": 23.5          // optional metadata for analysis
    }
  ]
}
```

### Output
- `users()` returns all known users.
- A `DEMAND` is emitted exactly at `time` with the provided payload.

## Commuter Pattern (`CommuterScenario`)

- File: `maasblender/src/scenario/commuter/commuter.py`
- Primary class: `CommuterScenario`
- Purpose: Generate two daily demands per configured commuter: one outbound (home → work) and one inbound (work → home), repeating every day.

#### Behavior
- For each commuter, two `DEMAND`s are created daily:
  1) at `deptOut`: outbound trip (`org` → `dst`),
  2) at `deptIn`: return trip, automatically reversing the original direction.
- The pattern repeats every 1440 minutes (1 day).
- `demandId` values are generated from the provided `demand_id_format`.

#### Configuration

```json
{
  "demand_id_format": "D_%d",
  "commuters": {
    "U001": {
      "deptOut": 480.0,   // 08:00
      "deptIn": 1080.0,   // 18:00
      "org": {"locationId": "Home", "lat": 35.0, "lng": 139.0},
      "dst": {"locationId": "Work", "lat": 35.7, "lng": 139.7},
      "user_type": "office",
      "service": "rail"
    },
    "U002": {
      "deptOut": 510.0,   // 08:30
      "deptIn": 1110.0,   // 18:30
      "org": {"locationId": "Home2", "lat": 35.2, "lng": 139.2},
      "dst": {"locationId": "Work2", "lat": 35.6, "lng": 139.6}
    }
  }
}
```

#### Output
- `users()` returns all commuters.
- `DEMAND` events are emitted at `deptOut` and `deptIn` each day. The inbound event uses the reversed origin/destination automatically.

---

### Common Operational Notes

- All three components use `simpy` for event scheduling and produce `DEMAND` events carrying `DemandInfo`
- Time Base: All times are in minutes from the simulation start.
- Determinism and Seeding:
  - `DemandGenerator` uses a random seed to ensure reproducible stochastic outcomes.
  - `HistoricalScenario` and `CommuterScenario` are deterministic given their inputs.
