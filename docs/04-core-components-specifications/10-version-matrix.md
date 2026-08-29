---
sidebar_position: 10
title: Version Compatibility
---

This page summarizes the version compatibility between MaaS Blender components and the external dependencies they rely on, including data format specifications.
Refer to this page when preparing input data or upgrading individual components.

## Version Tags

MaaS Blender uses tags to manage different versions, combining OTP versions with GTFS Flex specification variants.

| Release Tag | OTP Version | GTFS Flex Spec |
|-------------|:-----------:|:--------------:|
| v0.5.0      |     2.4     |     Draft      |
| v1.0.0      |     2.6     |    Standard    |
| v1.1.0      |     2.9     |    Standard    |

## GTFS Flex Compatibility

[GTFS Flex](https://gtfs.org/extensions/flex/) has undergone a significant structural change between its earlier draft specification and the version officially incorporated into the GTFS standard.
The two variants differ in how location group membership and stop times are represented.

### Specification Differences

| Field / File                                 |            GTFS Flex Draft             |            GTFS Flex Standard            |
|----------------------------------------------|:--------------------------------------:|:----------------------------------------:|
| Location group membership                    | `location_id` in `location_groups.txt` | Separate file `location_group_stops.txt` |
| Stop reference in `stop_times.txt`           |               `stop_id`                |           `location_group_id`            |
| Pickup/dropoff window start time culumn name |     `start_pickup_dropoff_window`      |      `start_pickup_drop_off_window`      |
| Pickup/dropoff window end time culumn name   |      `end_pickup_dropoff_window`       |       `end_pickup_drop_off_window`       |

### Component Support Matrix

| Component                 | Draft Spec | Standard Spec |
|---------------------------|:----------:|:-------------:|
| Route Deviation Simulator |     -¹     |   > v1.0.0    |
| On-Demand Simulator       |   v0.5.0   |   > v1.0.0    |
| Simple Planner            |   v0.5.0   |   > v1.0.0    |

¹ Route Deviation Simulator only supports GTFS Flex Standard specification.
