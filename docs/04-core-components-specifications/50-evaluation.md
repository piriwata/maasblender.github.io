---
sidebar_position: 50
title: Evaluation
---

# Evaluation

**Evaluates travel demand usability by querying route planners and analyzing service availability.**

The Evaluation component assesses the usability and quality of mobility services by analyzing travel demands. For each `DEMAND` event, it queries the route planner to obtain candidate routes, checks service reservability, and logs comprehensive evaluation results. These logs enable post-simulation analysis of service coverage, accessibility, and user experience.

## Implementations

MaaS Blender provides the following evaluation implementation:

- **simple** - Usability evaluator that logs route availability and reservability for each demand

## Simple

The simple evaluator performs usability evaluation by querying available routes and checking service reservability for each travel demand.

**File:** `src/evaluation/simple`  
**Primary Class:** `UsabilityEvaluator`

### Behavior

The evaluator operates as follows:

1. **Listens for DEMAND events** - Receives travel demand events via the `/triggered` endpoint
2. **Queues evaluation** - Schedules evaluation based on configured timing (at demand time or departure time)
3. **Queries route planner** - Requests all available routes for the origin-destination pair
4. **Checks reservability** - Verifies which services are actually reservable for the planned trips
5. **Logs evaluation results** - Writes comprehensive evaluation data including all candidate routes, actual user choice, and service availability

The evaluation timing can be configured to run either:
- `ON_DEMAND` - Evaluate immediately when demand is created (time of demand event)
- `ON_DEPARTURE` - Evaluate at the scheduled departure time

:::info
The evaluation component uses SimPy's discrete event simulation to schedule evaluations. It advances via the `/step` endpoint independently of other components.
:::

### Configuration

The evaluator is configured via the `/setup` endpoint:

```json
{
  "writer": {
    "endpoint": "http://jobmanager:8000"  // Optional: endpoint for centralized result collection
  },
  "planner": {
    "endpoint": "http://planner:8010/plan"  // Required: route planner endpoint
  },
  "reservable": {
    "endpoint": "http://broker:8000/reservable"  // Required: reservability checker endpoint
  },
  "evaluation_timing": "departure"  // "demand" or "departure"
}
```

**Key Configuration Parameters:**

- `writer.endpoint` - Optional URL for centralized result collection. If omitted, results are written to local file `evaluation.txt`
- `planner.endpoint` - URL of the route planner service
- `reservable.endpoint` - URL of the reservability checker service
- `evaluation_timing` - When to perform evaluation:
  - `"demand"` - Evaluate at the time the demand is created
  - `"departure"` - Evaluate at the scheduled departure time (default)

:::warning
The planner and reservable endpoints must be reachable and operational before starting evaluation. The component will fail if these services are unavailable.
:::

### Evaluation Output Format

The evaluator writes JSON log entries for each evaluated demand:

```json
{
  "demand_id": "D_1",
  "time": 480.0,
  "event_time": 120.5,
  "org": "home-001",
  "dst": "work-001",
  "actual_service": "bus-service",
  "plans": [
    [
      {
        "org": "home-001",
        "dst": "work-001",
        "dept": 480.0,
        "arrv": 510.0,
        "service": "bus-service",
        "reservable": true
      }
    ],
    [
      {
        "org": "home-001",
        "dst": "station-001",
        "dept": 480.0,
        "arrv": 490.0,
        "service": "walking",
        "reservable": true
      },
      {
        "org": "station-001",
        "dst": "work-001",
        "dept": 495.0,
        "arrv": 515.0,
        "service": "train-service",
        "reservable": false
      }
    ]
  ]
}
```

**Output Fields:**

- `demand_id` - Unique identifier for the demand
- `time` - Time when evaluation was performed (typically the departure time)
- `event_time` - Time when the original demand event was created (may differ from evaluation time when using ON_DEPARTURE timing)
- `org` - Origin location ID
- `dst` - Destination location ID
- `actual_service` - Service actually used by the user (if any)
- `plans` - Array of candidate routes, each containing an array of trip segments
  - `org` - Trip segment origin location ID
  - `dst` - Trip segment destination location ID
  - `dept` - Departure time for this segment
  - `arrv` - Arrival time for this segment
  - `service` - Service name for this segment
  - `reservable` - Whether this service was reservable at evaluation time

:::tip
Walking segments are always marked as `reservable: true` since they don't require reservation.
:::

### API Endpoints

The evaluation component implements the standard component API:

- `GET /spec` - Returns component specification (triggers on DEMAND events)
- `POST /setup` - Configure evaluation with planner/reservable endpoints and timing
- `POST /start` - Starts the evaluator
- `GET /peek` - Returns the next scheduled evaluation time
- `POST /step` - Advances simulation and processes queued evaluations
- `POST /triggered` - Receives DEMAND events for evaluation
- `POST /finish` - Cleanup, closes connections, and finalizes output
- `GET /evaluation` - Downloads evaluation results (only when using file writer)

### Analysis Use Cases

The evaluation logs support various post-simulation analyses:

**Service Coverage Analysis**
```python
import pandas as pd

# Load evaluation logs
df = pd.read_json("evaluation.txt", lines=True)

# Calculate percentage of demands with at least one reservable route
df['has_reservable'] = df['plans'].apply(
    lambda plans: any(
        all(trip['reservable'] for trip in route)
        for route in plans
    )
)
coverage = df['has_reservable'].mean() * 100
print(f"Service coverage: {coverage:.1f}%")
```

**Service Comparison**
```python
# Count how many routes include each service
from collections import Counter

service_counts = Counter()
for plans in df['plans']:
    for route in plans:
        services = {trip['service'] for trip in route if trip['service'] != 'walking'}
        service_counts.update(services)

print("Service availability in routes:")
for service, count in service_counts.most_common():
    print(f"  {service}: {count}")
```

**Temporal Availability**
```python
# Analyze service availability by time of day
df['hour'] = (df['time'] % 1440) // 60  # Convert to hour of day
availability_by_hour = df.groupby('hour')['has_reservable'].mean()
print(availability_by_hour)
```

## Common Operational Notes

### Performance Considerations

- The evaluator makes synchronous HTTP requests to the planner and reservability checker for each demand
- Evaluation can be time-consuming for large numbers of demands
- Consider using `evaluation_timing: "demand"` for real-time evaluation or `"departure"` to spread evaluation load over time

### Result Collection

Two output modes are supported:

1. **File Writer** - Writes results to local `evaluation.txt` file
   - Use when running standalone or for debugging
   - Results can be downloaded via `GET /evaluation` endpoint
   
2. **HTTP Writer** - Sends results to centralized job manager
   - Use in distributed deployments with centralized result collection
   - Configure via `writer.endpoint` parameter

### Integration with Route Planner

The evaluation component requires a route planner service that accepts:

**Request:**
```json
POST {planner.endpoint}?dept={departure_time}
{
  "org": {"id_": "loc-1", "lat": 35.0, "lng": 135.0},
  "dst": {"id_": "loc-2", "lat": 35.1, "lng": 135.1}
}
```

**Response:**
```json
[
  {
    "trips": [
      {
        "org": {"id_": "loc-1", "lat": 35.0, "lng": 135.0},
        "dst": {"id_": "loc-2", "lat": 35.1, "lng": 135.1},
        "dept": 480.0,
        "arrv": 510.0,
        "service": "bus"
      }
    ]
  }
]
```

### Integration with Reservability Checker

The reservability checker service must accept:

**Request:**
```
GET {reservable.endpoint}?service={service_name}&org={origin_id}&dst={dest_id}
```

**Response:**
```json
{
  "reservable": true
}
```

:::info
The reservability check is performed at evaluation time, which may differ from actual reservation time. Services may become full or unavailable between evaluation and actual reservation.
:::

### Validation and Assertions

The evaluator performs validation to ensure route consistency:
- Verifies that all returned routes have the same origin and destination (within a tolerance of 0.0001 degrees, approximately 11 meters at mid-latitudes)
- Logs warnings if routes are inconsistent
- Continues evaluation even if validation fails to avoid stopping the simulation

:::info
The coordinate tolerance of 0.0001 degrees corresponds to roughly 11 meters at the equator. At higher latitudes, the actual distance for the same degree difference will be less for longitude but remain similar for latitude.
:::

### Error Handling

- HTTP connection errors to planner or reservability services are propagated and may stop evaluation
- Use the `/finish` endpoint to ensure proper cleanup even if errors occur
- Check component logs for detailed error messages and stack traces
