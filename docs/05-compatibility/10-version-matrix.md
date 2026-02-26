---
sidebar_position: 10
title: Version Compatibility Matrix
---

This page summarizes the version compatibility between MaaS Blender components and the external dependencies they rely on, including data format specifications.
Refer to this page when preparing input data or upgrading individual components.

## Component Version Overview

| Component | Implemented Version | Notes |
|-----------|-------------------|-------|
| OpenTripPlanner (OTP) | 2.6.0 | Requires JDK 21 |
| On-Demand Simulator | — | Uses OR-Tools 9.12 for optimization |
| Simple In-Process Planner | — | Lightweight, no external runtime dependency |
| Python Runtime (all components) | 3.11 | Minimum required version |

## GTFS Flex Compatibility

[GTFS Flex](https://gtfs.org/extensions/flex/) has undergone a significant structural change between its earlier draft specification and the version officially incorporated into the GTFS standard.
The two variants differ in how location group membership and stop times are represented.

### Specification Differences

| Field / File | Old GTFS Flex Spec | New GTFS Flex Spec (v2) |
|---|---|---|
| Location group membership | `location_id` column in `location_groups.txt` | Separate file `location_group_stops.txt` |
| Stop reference in `stop_times.txt` | `stop_id` column | `location_group_id` column |

### Component Support Matrix

| Component | Old GTFS Flex Spec | New GTFS Flex Spec (v2) |
|-----------|:-----------------:|:----------------------:|
| **OTP Planner** (OTP 2.6.0) | ❌ | ✅ |
| **On-Demand Simulator** | ✅ | ✅ |
| **Simple In-Process Planner** | ✅ | ❌ |

:::warning
The **OTP Planner** requires GTFS Flex data in the new v2 format (`location_group_stops.txt` and `location_group_id` in `stop_times.txt`).
Providing old-format GTFS Flex data will cause graph build failures or incorrect routing results.
:::

:::warning
The **Simple In-Process Planner** only supports the old GTFS Flex format (`location_id` in `location_groups.txt` and `stop_id` in `stop_times.txt`).
New v2 format files are not supported by this planner.
:::

:::info
The **On-Demand Simulator** handles both formats transparently.
It reads `location_group_id` first from `stop_times.txt`, falling back to `stop_id` if the column is absent, and reads membership from `location_group_stops.txt` when present or from `location_id` within `location_groups.txt` otherwise.
:::

## OpenTripPlanner Version Notes

MaaS Blender integrates **OpenTripPlanner 2.6.0** and communicates with it via the OTP 2.x GraphQL API at `/otp/routers/default/index/graphql`.

:::warning
The OTP 2.x GraphQL API is **not compatible** with the OTP 1.x REST API.
If you need to substitute a different OTP version, ensure it exposes the same OTP 2.x GraphQL endpoint.
OTP versions prior to 2.0 are not supported.
:::

Key constraints for the OTP integration:

- **JDK 21** is required to run OTP 2.6.0.
- The OTP graph build process requires all input files (`otp-config.zip`, GTFS/GTFS Flex zip, etc.) to be uploaded **before** the broker setup is finalized.
- GTFS Flex routing relies on OTP's `FLEX` transport mode qualifier. Ensure the `otp_config` files include a router configuration that enables flexible transit.

For more details on configuring the OTP planner, see the [Route Planner](../04-core-components-specifications/30-route-planner.md) chapter.
