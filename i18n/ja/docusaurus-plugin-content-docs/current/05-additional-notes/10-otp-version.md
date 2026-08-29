---
sidebar_position: 10
title: OTP バージョン
---

OpenTripPlanner は、各 MaaS Blender リリースでサポートされている最新版の OTP をデフォルトで使用します。

あえて古い OTP を使いたい場合は、`opentripplanner` モジュールのコンテナをビルドする際に `build.args.otp_version` を変更してください。

```yaml
services:
  opentripplanner:
    build:
      args:
        otp_version: "2.5.0"
```
