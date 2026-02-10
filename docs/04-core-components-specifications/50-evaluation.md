---
sidebar_position: 50
title: "Evaluation: Analysis Sidecar"
---

This chapter specifies the Evaluation component in MaaS Blender.
The Evaluation component is an auxiliary sidecar that runs alongside the simulation to observe and analyze system behavior in real time.
Unlike other components that drive the simulation forward, the evaluation component is strictly observational and does not emit events or influence simulation execution.

The primary purpose of the evaluation component is to enable richer analysis than post-hoc event inspection alone can provide.
For example, when a user successfully reserves a mobility service, the evaluation component can capture whether alternative route candidates were also reservable at that exact moment—information that is only available during simulation execution.

## Design Philosophy

By externalizing evaluation logic into a sidecar component:

- Core simulation components remain simple and focused on their primary responsibilities.
- Analytical concerns do not leak into execution logic.
- Multiple evaluation strategies can be developed independently without affecting simulation behavior.

## Event Interface

The evaluation component consumes all events in the system but does not emit events of its own.

| Consumed Events | Emitted Events |
|-----------------|----------------|
| `DEMAND`        | _(none)_       |
| `RESERVE`       |                |
| `RESERVED`      |                |
| `DEPART`        |                |
| `DEPARTED`      |                |
| `ARRIVED`       |                |

## Available Implementations

The project currently provides a foundation for custom evaluation logic.
Users can implement their own evaluation strategies by creating components that observe simulation events and record custom metrics.

### Custom Evaluation Implementation

- Location: User-defined (can be placed in custom modules or scripts)
- Integration: Registers as an event observer in the simulation environment

#### Behavior

- **Event Observation**: The evaluation component receives all simulation events as they occur.
- **Non-intrusive**: Does not modify simulation state or emit events that would affect other components.
- **Real-time Analysis**: Can compute metrics and capture state information that would be difficult or impossible to reconstruct from event logs alone.
- **Flexible Metrics**: Can track any combination of:
  - User behavior patterns (e.g., route preferences, reservation success rates)
  - Service utilization (e.g., vehicle occupancy, wait times)
  - System-wide KPIs (e.g., average travel time, service coverage)
  - Alternative scenario analysis (e.g., "what if" comparisons for unselected routes)

#### Configuration

The evaluation component configuration is implementation-dependent. A typical structure might include:

```json
{
  "evaluation": {
    "enabled": true,                          // Enable/disable evaluation sidecar
    "output_file": "evaluation_results.json", // Output file for custom metrics
    "metrics": [
      "reservation_success_rate",             // Track reservation success/failure
      "average_travel_time",                  // Monitor trip durations
      "service_utilization"                   // Measure vehicle usage
    ],
    "sampling_interval": 60.0,                // Sample system state every N minutes (optional)
    "track_alternatives": true                // Record unselected route candidates (optional)
  }
}
```

**Common Configuration Parameters:**
- `enabled`: Whether to activate the evaluation component.
- `output_file`: Path for storing custom evaluation results (separate from `events.txt`).
- `metrics`: List of metrics to compute during simulation.
- `sampling_interval`: How frequently to sample system state (if applicable).
- `track_alternatives`: Whether to record information about routes not selected by users.

#### Example Use Cases

**1. Reservation Success Analysis**

Track which services have high or low reservation success rates:

```python
# Pseudocode example
class ReservationSuccessEvaluator:
    def __init__(self):
        self.success_count = {}
        self.failure_count = {}
    
    def on_reserved_event(self, event):
        service = event.source
        success = event.details.get("success", False)
        
        if success:
            self.success_count[service] = self.success_count.get(service, 0) + 1
        else:
            self.failure_count[service] = self.failure_count.get(service, 0) + 1
    
    def get_success_rates(self):
        # Calculate success rate per service
        rates = {}
        for service in set(self.success_count.keys()) | set(self.failure_count.keys()):
            total = self.success_count.get(service, 0) + self.failure_count.get(service, 0)
            rates[service] = self.success_count.get(service, 0) / total if total > 0 else 0.0
        return rates
```

**2. Alternative Route Analysis**

Capture information about routes that users considered but did not select:

```python
# Pseudocode example
class AlternativeRouteEvaluator:
    def __init__(self):
        self.alternatives = []
    
    def on_demand_event(self, event):
        # Store all candidate routes available to the user
        user_id = event.details.get("userId")
        demand_id = event.details.get("demandId")
        candidates = self.get_available_routes(event)
        
        self.alternatives.append({
            "time": event.time,
            "user_id": user_id,
            "demand_id": demand_id,
            "candidates": candidates
        })
    
    def on_reserved_event(self, event):
        # Mark which route was actually selected
        demand_id = event.details.get("demandId")
        selected_route = event.details.get("route")
        
        for alt in self.alternatives:
            if alt["demand_id"] == demand_id:
                alt["selected"] = selected_route
                break
```

**3. Service Utilization Monitoring**

Track real-time vehicle occupancy and service efficiency:

```python
# Pseudocode example
class ServiceUtilizationEvaluator:
    def __init__(self):
        self.vehicle_occupancy = {}  # mobilityId -> current passenger count
        self.utilization_history = []
    
    def on_departed_event(self, event):
        mobility_id = event.details.get("mobilityId")
        if mobility_id:
            self.vehicle_occupancy[mobility_id] = self.vehicle_occupancy.get(mobility_id, 0) + 1
    
    def on_arrived_event(self, event):
        mobility_id = event.details.get("mobilityId")
        if mobility_id:
            self.vehicle_occupancy[mobility_id] = max(0, self.vehicle_occupancy.get(mobility_id, 0) - 1)
    
    def sample_utilization(self, time):
        # Periodically record vehicle occupancy
        snapshot = {
            "time": time,
            "occupancy": dict(self.vehicle_occupancy)
        }
        self.utilization_history.append(snapshot)
```

---

## Post-Simulation Analysis

While the evaluation sidecar component enables real-time observation, most analysis is performed after simulation completion using the generated `events.txt` file.

### Event Log Format

After running a simulation, an `events.txt` file is generated in JSON Lines format:
- Each line represents a single event encoded as a JSON object.
- Events are written in chronological order.
- The structure and semantics are defined in the [Event Specification](../02-simulation-model/10-event.md).

### Analysis with Python and Pandas

The following examples demonstrate common analysis patterns:

#### User-Focused Analysis

Extract and analyze user reservation behavior:

```python
import pandas as pd

# Read line-delimited JSON
df = pd.read_json("events.txt", lines=True)

# Extract RESERVED events
df = df[df["eventType"] == "RESERVED"]

# Exclude walking-based reservations
df = df[df["source"] != "walking"]

# Expand relevant fields from 'details'
df["userId"] = df["details"].apply(lambda d: d.get("userId"))
df["demandId"] = df["details"].apply(lambda d: d.get("demandId"))
df["success"] = df["details"].apply(lambda d: d.get("success"))
df["org"] = df["details"].apply(lambda d: d["route"][0]["org"]["locationId"] if d.get("route") else None)
df["dst"] = df["details"].apply(lambda d: d["route"][0]["dst"]["locationId"] if d.get("route") else None)

# Select and sort columns
df["service"] = df["source"]
df = df[["time", "userId", "demandId", "service", "org", "dst", "success"]]
df = df.sort_values(["userId", "time"])

print(df)
```

#### Mobility-Focused Analysis

Analyze actual vehicle usage and passenger trips:

```python
import pandas as pd

# Read line-delimited JSON
df = pd.read_json("events.txt", lines=True)

# Keep only DEPARTED and ARRIVED events
df = df[df["eventType"].isin(["DEPARTED", "ARRIVED"])]

# Extract fields
df["mobilityId"] = df["details"].apply(lambda d: d.get("mobilityId"))
df["userId"] = df["details"].apply(lambda d: d.get("userId"))
df["demandId"] = df["details"].apply(lambda d: d.get("demandId"))
df["locationId"] = df["details"].apply(lambda d: d["location"].get("locationId") if d.get("location") else None)

# Keep only rows where both mobilityId and userId are present
df = df[df["mobilityId"].notna() & df["userId"].notna()]

# Select relevant columns
df = df[["time", "eventType", "userId", "mobilityId", "demandId", "locationId"]]

# Sort by userId, mobilityId, and time
df = df.sort_values(["userId", "mobilityId", "time"]).reset_index(drop=True)

print(df)
```

---

## Common Operational Notes

- **Non-interference**: The evaluation component must never affect simulation behavior. It is strictly observational.
- **Performance**: Event observation should be lightweight to avoid slowing down the simulation.
- **Time Base**: All event times are in minutes from simulation start, consistent with other components.
- **Extensibility**: Users can implement custom evaluation strategies by creating observers that register with the simulation environment.
- **Output Separation**: Custom evaluation results should be written to separate files (not `events.txt`) to maintain clear separation between simulation events and derived metrics.

:::info
For detailed information about event structures and fields, refer to the [Event Specification](../02-simulation-model/10-event.md) documentation.
For examples of post-simulation analysis, see the [Analyze Output Events](../01-quick-start/50_analyze.md) guide.
:::
