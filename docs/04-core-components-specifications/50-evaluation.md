---
sidebar_position: 50
title: "Evaluation: Real-time Simulation Analysis"
---

This chapter specifies the Evaluation component in MaaS Blender.
An Evaluation component is an auxiliary component that runs alongside the simulation to observe and evaluate system behavior in real time.
It captures information that cannot be fully reconstructed from the event log alone, enabling richer post-simulation analysis.

While MaaS Blender is fundamentally event-driven, certain aspects of system behavior require observation at the moment they occur.
The Evaluation component exists to capture such information without affecting the simulation flow.

The project currently ships reference implementations under `maasblender/src/evaluation`:

- Route Choice Evaluator: tracks which route alternatives were available when users made their choices.
- Service Utilization Evaluator: monitors vehicle occupancy and capacity usage over time.
- User Experience Evaluator: measures journey quality metrics (travel time, waiting time, transfers).

## Event Contract

| Consumed Events | Emitted Events |
|-----------------|----------------|
| `DEMAND`        | _(none)_       |
| `RESERVE`       |                |
| `RESERVED`      |                |
| `DEPART`        |                |
| `DEPARTED`      |                |
| `ARRIVED`       |                |

An Evaluation component consumes all events in the system but does not emit events of its own.
It is strictly observational and must not influence simulation behavior.

## Purpose and Motivation

The primary purpose of the Evaluation component is to enable richer analysis than post-hoc event inspection alone can provide.

### Examples of Real-time Analysis

1. **Alternative Route Availability**:
   - When a user successfully reserves a mobility service, it may be important to know whether alternative route candidates were also reservable at that exact time.
   - This information is only available during simulation execution and cannot be reconstructed later.

2. **Counterfactual Scenarios**:
   - What would have happened if the user had chosen a different route?
   - Would other services have been available?
   - The evaluator can test these scenarios in real time.

3. **Temporal Snapshots**:
   - Vehicle locations and occupancy at specific moments
   - Queue lengths at bike share stations
   - Available capacity across all services at key decision points

## Route Choice Evaluator (`RouteChoiceEvaluator`)

- File: `maasblender/src/evaluation/route_choice/evaluator.py`
- Primary class: `RouteChoiceEvaluator`

This evaluator tracks which route alternatives were available and reservable when users made their route choices.

### Behavior

1. **On `DEMAND`**: Calls the Route Planner to obtain all available route candidates.
2. **On `RESERVE`**: Records which route the user selected and tests whether other routes would have been reservable.
3. **On `RESERVED`**: Records whether the selected route was successfully reserved.
4. **Output**: Produces a dataset showing, for each demand:
   - All available route alternatives
   - Which route was chosen
   - Reservation success/failure for each alternative

### Configuration

```json
{
  "evaluator": "route_choice",
  "output_file": "route_choices.csv",
  "test_all_alternatives": true
}
```

**Parameters:**
- `output_file`: Path to the output CSV file.
- `test_all_alternatives`: If true, tests reservation availability for all route alternatives (not just the chosen one).

:::warning
Testing all alternatives may add computational overhead, especially for scenarios with many users and routes.
Use this option when you need detailed alternative analysis; disable it for faster simulation runs.
:::

### Output Format

```csv
demand_id,user_id,time,chosen_route_id,num_alternatives,reservation_success,alternative_1_reservable,alternative_2_reservable
D_1,U001,10.0,route_1,3,true,true,false
D_2,U002,12.0,route_2,2,false,false,true
```

## Service Utilization Evaluator (`ServiceUtilizationEvaluator`)

- File: `maasblender/src/evaluation/utilization/evaluator.py`
- Primary class: `ServiceUtilizationEvaluator`

This evaluator monitors vehicle occupancy and capacity usage over time for each mobility service.

### Behavior

1. **On `DEPARTED`**: Increments the passenger count for the relevant vehicle/service.
2. **On `ARRIVED`**: Decrements the passenger count for the relevant vehicle/service.
3. **Periodic Sampling**: Records occupancy snapshots at regular intervals.
4. **Output**: Produces time-series data showing occupancy levels, capacity utilization, and peak load periods.

### Configuration

```json
{
  "evaluator": "service_utilization",
  "output_file": "utilization.csv",
  "sampling_interval_minutes": 5.0,
  "services": ["bus", "taxi", "bike_share"]
}
```

**Parameters:**
- `output_file`: Path to the output CSV file.
- `sampling_interval_minutes`: Time interval for periodic occupancy snapshots.
- `services`: List of services to monitor (if empty, monitors all services).

### Output Format

```csv
time,service,vehicle_id,current_occupancy,max_capacity,utilization_percent
15.0,bus,bus_1,25,50,50.0
15.0,taxi,taxi_3,1,4,25.0
20.0,bus,bus_1,38,50,76.0
```

## User Experience Evaluator (`UserExperienceEvaluator`)

- File: `maasblender/src/evaluation/experience/evaluator.py`
- Primary class: `UserExperienceEvaluator`

This evaluator measures journey quality metrics from the user's perspective.

### Behavior

1. **On `DEMAND`**: Records the demand creation time and desired departure time.
2. **On `RESERVE`/`RESERVED`**: Tracks reservation delays and failures.
3. **On `DEPART`/`DEPARTED`/`ARRIVED`**: Calculates travel time, waiting time, and total journey time.
4. **Output**: Produces a dataset of user journey metrics for analysis.

### Configuration

```json
{
  "evaluator": "user_experience",
  "output_file": "user_journeys.csv",
  "include_failed_trips": true
}
```

**Parameters:**
- `output_file`: Path to the output CSV file.
- `include_failed_trips`: If true, includes trips that failed to reserve or complete.

### Output Format

```csv
demand_id,user_id,request_time,desired_dept,actual_dept,actual_arrv,total_travel_time,waiting_time,num_transfers,success
D_1,U001,5.0,10.0,10.0,40.0,30.0,5.0,1,true
D_2,U002,8.0,15.0,null,null,null,null,0,false
```

**Metrics:**
- `total_travel_time`: Time from actual departure to arrival
- `waiting_time`: Time from demand creation to actual departure
- `num_transfers`: Number of service transfers in the journey
- `success`: Whether the journey was completed successfully

## Creating Custom Evaluators

To create a custom evaluator:

1. Implement a class that inherits from the base `Evaluator` interface.
2. Subscribe to relevant events by implementing event handler methods (e.g., `on_demand`, `on_reserve`).
3. Collect and store data in your desired format.
4. Implement an `export()` method to write results to a file or database.

### Example Structure

```python
class CustomEvaluator:
    def __init__(self, config):
        self.data = []
        self.config = config
    
    def on_demand(self, event):
        # Record demand information
        pass
    
    def on_reserved(self, event):
        # Record reservation outcome
        pass
    
    def export(self):
        # Write collected data to output file
        pass
```

:::tip
Keep evaluator logic simple and efficient. Complex calculations can slow down the simulation.
Store raw event data during simulation and perform heavy analysis in post-processing.
:::

## Design Rationale

By externalizing evaluation logic into a separate component:

- **Separation of Concerns**: Core simulation components remain simple and focused on their primary responsibilities.
- **Non-intrusive Observation**: Analytical concerns do not leak into execution logic.
- **Flexibility**: Multiple evaluation strategies can be developed and applied independently.
- **Extensibility**: New evaluation metrics can be added without modifying the core simulation.
- **Performance**: Evaluators can be enabled or disabled based on analysis needs, reducing overhead when not needed.

---

### Common Operational Notes

- **Event Observation**: Evaluators receive all events but must not modify event payloads or emit new events.
- **Time Synchronization**: Evaluators observe events at the same simulation time as other components.
- **State Tracking**: Evaluators may maintain internal state to correlate events across time (e.g., matching `DEPART` with `ARRIVED`).
- **Output Timing**: Most evaluators write output at the end of the simulation, but periodic exports during long simulations are also supported.
- **Performance Impact**: Evaluators add computational overhead proportional to the complexity of their analysis. Design evaluators to be efficient and consider disabling them for production runs where detailed analysis is not needed.
- **Multiple Evaluators**: Multiple evaluator instances can run simultaneously, each focusing on different analysis aspects.
