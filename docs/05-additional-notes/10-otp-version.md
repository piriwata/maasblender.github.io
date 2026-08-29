---
sidebar_position: 10
title: OTP Version
---

OpenTripPlanner uses the latest OTP version supported by each MaaS Blender release by default.

If you intentionally want to use an older OTP version, change `build.args.otp_version` when building the `opentripplanner` module container.

```yaml
services:
  opentripplanner:
    build:
      args:
        otp_version: "2.5.0"
```
