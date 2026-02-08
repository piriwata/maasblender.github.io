---
sidebar_position: 30
title: "Route Planner"
---

The Route Planner is the component that converts a user’s travel intention (`org`, `dst`, `dept`) into 
one or more possible route candidates composed of ordered legs. 

The project currently ships two reference implementations under `planner/`:
- 
- A simple in-process planner for synthetic networks: `planner/simple`
- An OpenTripPlanner backed service: `planner/opentripplanner`

---

## Simple in-process planner

File: `planner/simple`

This planner provides a deterministic, dependency-light planner over known `MobilityNetwork` graphs.

### Behavior

### Configuration

---

## OpenTripPlanner-backed service

Files: `planner/opentripplanner`

### Behavior

### Configuration



