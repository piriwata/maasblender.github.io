---
sidebar_position: 40
title: "モビリティサービス"
---

この章では、MaaS Blender のモビリティサービスコンポーネントのリファレンス実装を説明します。
各モビリティサービスは特定の交通手段をシミュレートし、予約から出発・到着に至るまでのユーザーの移動ライフサイクル全体を、シミュレーションイベントに応答することで処理します。

現在、`maasblender/src/base_simulators` 配下に 5 つのリファレンス実装が同梱されています。

- **オンデマンド**（`ondemand`）: 動的な配車最適化を伴うデマンド応答型交通。
- **ワンウェイ**（`oneway`）: 一方向の自由浮動型車両共有（例: 電動キックボードやカーシェア）。
- **ルート逸脱**（`routedeviation`）: オンデマンド停留所を柔軟に経由できる定時運行路線。
- **定時運行**（`scheduled`）: GTFS 時刻表に基づく固定スケジュール交通サービス。
- **徒歩**（`walking`）: 徒歩移動。フォールバックまたはファースト/ラストマイル接続として使用。

すべての実装は同じイベント駆動インターフェースを共有しており、`RESERVE` および `DEPART` のトリガーイベントを受け取り、`RESERVED`、`DEPARTED`、`ARRIVED` イベントを発行します。

## 徒歩（`walking`）

- ファイル: `maasblender/src/base_simulators/walking/`
- 主要クラス: `Simulation`（`simulation.py`）、API サーバー: `controller.py`

徒歩シミュレーターは任意の 2 地点間の歩行移動をモデル化します。
最もシンプルなモビリティサービスであり、他のサービスが利用できない場合のフォールバックや、マルチモーダル移動の徒歩区間として使用されます。

### 動作

- `RESERVE` 時:
  - 常に予約を受け付けます（このサービスは満員になりません）。
  - 設定された歩行速度（`walking_meters_per_minute`）を使って、`org` と `dst` の間の測地線距離から移動時間を計算します。
  - `arrv` が指定されていて `arrv > dept` の場合、指定された `arrv` をそのまま使用します。それ以外は `arrv = dept + 移動時間` として再計算します。
  - `success: true` と確定したルート（`dept`、`arrv`）を含む `RESERVED` イベントを即座に発行します。
- `DEPART` 時:
  - `dept` の時刻に `org` から `DEPARTED` を発行します。
  - `arrv` の時刻に `dst` で `ARRIVED` を発行します。
- `/reservable` エンドポイントは、起点・目的地に関わらず常に `true` を返します。

### 設定

徒歩シミュレーターは `/setup` エンドポイントで設定します。

```json
{
  "walking_meters_per_minute": 80.0   // 歩行速度（メートル/分）（デフォルト: 80.0）
}
```

- `walking_meters_per_minute`: 測地線距離から移動時間を計算する際に使用する歩行速度。
  デフォルト値の `80.0` m/分は約 4.8 km/h に相当します。

:::tip
測地線距離は `geopy.distance.geodesic` を使って計算されるため、地球の曲率が考慮されます。
短い都市距離では平面近似との差は無視できますが、長い歩行区間では精度が向上します。
:::

#### 例

```json
{
  "walking_meters_per_minute": 60.0
}
```

`60.0` m/分（3.6 km/h）に設定することで、ゆっくり歩く高齢者や交差点の多いルートをモデル化できます。

### 出力

- `RESERVED` イベント（`RESERVE` の直後に発行）:
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "route": [
        {
          "org": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StopB", "lat": 35.01, "lng": 139.01 },
          "dept": 480.0,
          "arrv": 492.3
        }
      ]
    }
  }
  ```
- `DEPARTED` イベント（`dept` の時刻に発行）:
  ```json
  {
    "eventType": "DEPARTED",
    "details": {
      "subjectId": "U001",
      "userId": "U001",
      "demandId": "D_1",
      "mobilityId": null,
      "location": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 }
    }
  }
  ```
- `ARRIVED` イベント（`arrv` の時刻に発行）:
  ```json
  {
    "eventType": "ARRIVED",
    "details": {
      "subjectId": "U001",
      "userId": "U001",
      "demandId": "D_1",
      "mobilityId": null,
      "location": { "locationId": "StopB", "lat": 35.01, "lng": 139.01 }
    }
  }
  ```

:::info
徒歩移動では物理的な車両が存在しないため、`mobilityId` は常に `null` になります。
:::

## 定時運行（`scheduled`）

- ファイル: `maasblender/src/base_simulators/scheduled/`
- 主要クラス: `Simulation`（`simulation.py`）、API サーバー: `controller.py`

定時運行シミュレーターは、バスや鉄道などの固定時刻表の交通サービスをモデル化します。
GTFS データを読み込み、事前定義された停留所シーケンスに従う車両をシミュレートし、ユーザーは路線上の任意の停留所で乗降できます。

### 動作

- `RESERVE` 時:
  - `org` 停留所に停車し、かつ `org` の後に `dst` にも停車する最も早い車両（便）を検索します。
  - その車両にまだ空き（`is_reservable`）があるか確認します。
  - 空きのある適切な車両が見つかった場合、座席を予約し、計算された `dept`（`org` からの定刻出発時刻）と `arrv`（`dst` への定刻到着時刻）を含む `RESERVED`（`success: true`）を発行します。
  - 適切な車両が見つからない場合、`RESERVED`（`success: false`）を発行します。
- `DEPART` 時:
  - ユーザーは `org` 停留所で待機します。予約した車両が `org` に到着するとユーザーが乗車し、`DEPARTED` が発行されます。
  - 車両が `dst` に到着するとユーザーが降車し、`ARRIVED` が発行されます。
- `/reservable` エンドポイントは、空きのある車両が `org` → `dst` を運行できるかをリアルタイムで確認します。

### 設定

定時運行シミュレーターは、GTFS zip ファイルをアップロードした後、`/setup` エンドポイントで設定します。

```json
{
  "reference_time": "20251016",          // シミュレーション基準日（YYYYMMDD、8 文字）
  "input_files": [
    { "filename": "gtfs.zip" }           // /upload 経由でアップロードした GTFS アーカイブ
  ],
  "mobility": {
    "capacity": 30                        // すべての便に共通で適用される座席数
  }
}
```

- `reference_time`: GTFS カレンダーの解決に使用する基準日（YYYYMMDD、ちょうど 8 文字）。
- `input_files`: GTFS zip ファイルを 1 つ指定します。`filename`（`/upload` でアップロード済み）または `fetch_url` で指定します。
- `mobility.capacity`: GTFS フィード内のすべての便に一律に適用される乗客定員。

:::info
GTFS の `block_id` をサポートしています。同じ `block_id` を持つ便はひとつの継続サービスとして連結され、乗客が降車・再乗車することなく複数便をまたがる直通サービスが可能になります。
:::

### 出力

- `RESERVED` イベント（成功）:
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "mobilityId": "trip-001",
      "route": [
        {
          "org": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StopC", "lat": 35.05, "lng": 139.05 },
          "dept": 480.0,
          "arrv": 495.0
        }
      ]
    }
  }
  ```
- `RESERVED` イベント（失敗）:
  ```json
  { "eventType": "RESERVED", "details": { "userId": "U001", "demandId": "D_1", "success": false } }
  ```
- `DEPARTED`・`ARRIVED` イベント: 車両が対応する停留所に到着したときに発行されます。
  `mobilityId` には GTFS の便 ID が格納されます。

## ルート逸脱（`routedeviation`）

- ファイル: `maasblender/src/base_simulators/routedeviation/`
- 主要クラス: `Simulation`（`simulation.py`）、API サーバー: `controller.py`

ルート逸脱シミュレーターは、定時運行モデルを拡張して、乗客を柔軟な（停留所外の）地点で乗降させるために車両が計画ルートから逸脱できるようにします。
デマンドレスポンシブ型フィーダーサービスや、おおまかな回廊に沿いながらもオンデマンド配車するシナリオに適しています。

### 動作

- `RESERVE` 時:
  - GTFS の固定停留所として定義されていない任意の座標（`TemporaryStop`）を含む `org` → `dst` のペアに対応できる最も早い車両を検索します。
  - 空きと実現可能性（`is_reservable`）を確認します。
  - 実現可能な場合、ユーザーを予約し、計画された乗車・降車時刻を含む `RESERVED`（`success: true`）を発行します。
  - 対応可能な車両がない場合、`RESERVED`（`success: false`）を発行します。
- `DEPART` 時:
  - ユーザーは柔軟な乗車地点で待機します。車両がそこに立ち寄ると `DEPARTED` が発行されます。
  - 車両が降車地点に到達すると `ARRIVED` が発行されます。
- `/reservable` エンドポイントは、いずれかの車両が指定した `org`/`dst` 座標に対応できるかをリアルタイムで確認します。

:::info
ルート逸脱では、起点と目的地に対して停留所 ID だけでなく完全な座標ペア（`lat`、`lng`）を受け付けます。
これが GTFS フィードに定義された停留所のみ対応する定時運行シミュレーターとの違いです。
:::

### 設定

ルート逸脱シミュレーターは定時運行と同じ設定構造を持ちます。

```json
{
  "reference_time": "20251016",          // シミュレーション基準日（YYYYMMDD、8 文字）
  "input_files": [
    { "filename": "gtfs.zip" }           // /upload 経由でアップロードした GTFS アーカイブ
  ],
  "mobility": {
    "capacity": 10                        // 車両あたりの乗客定員
  }
}
```

- `reference_time`: GTFS カレンダーの解決に使用する基準日（YYYYMMDD、ちょうど 8 文字）。
- `input_files`: GTFS zip ファイルを 1 つ指定します。
- `mobility.capacity`: すべての車両に適用される乗客定員。

### 出力

出力イベントは定時運行と同じ構造を持ちます。ただし、`RESERVED` 内の `org`/`dst` は GTFS 停留所リストに存在しない座標を参照する場合があります。

## オンデマンド（`ondemand`）

- ファイル: `maasblender/src/base_simulators/ondemand/`
- 主要クラス: `Simulation`（`simulation.py`）、API サーバー: `controller.py`

オンデマンドシミュレーターは、複数のユーザーを動的に乗降させるデマンド応答型バスサービスをモデル化します。
各車両は GTFS FLEX で定義されたサービスエリア内で運行し、新しい予約が入るたびに OR-Tools またはブルートフォース組み合わせ探索を使ってルートが継続的に最適化されます。

### 動作

- `RESERVE` 時:
  - すべての利用可能な車両を評価し、新しいユーザーを各車両の現在のスケジュールに最適に挿入する方法を計算します（全乗客の遅延合計を最小化）。
  - `enable_ortools` が `true`（デフォルト）の場合、**OR-Tools** 制約ソルバーを使用します。それ以外はブルートフォース列挙にフォールバックします。
  - 実現可能な割り当てが見つかった場合（遅延が `max_delay_time` 以内）、車両スケジュールを更新し、`dept`（予定乗車時刻）と `arrv`（予定降車時刻）を含む `RESERVED`（`success: true`）を発行します。
  - 実現可能な割り当てが存在しない場合、`RESERVED`（`success: false`）を発行します。
- `DEPART` 時:
  - ユーザーは起点停留所で出発準備（`ready_to_depart`）を通知します。
  - オンデマンド車両が到着してユーザーが乗車すると `DEPARTED` が発行されます。
  - 車両がユーザーを目的地停留所に届けると `ARRIVED` が発行されます。
- `/reservable` エンドポイントは、いずれかの車両が遅延制約内で `org` → `dst` に対応できるかをリアルタイムで確認します。

### 設定

オンデマンドシミュレーターは、GTFS FLEX zip ファイルと停留所間距離行列のアップロードが必要です。

```json
{
  "reference_time": "20251016",            // シミュレーション基準日（YYYYMMDD）
  "input_files": [
    { "filename": "gtfs_flex.zip" }        // GTFS FLEX アーカイブ（/upload 経由でアップロード）
  ],
  "network": {
    "fetch_url": "http://planner/network"  // 停留所間距離行列を取得する URL
    // または: "filename": "network.json"  // アップロード済み距離行列ファイル
  },
  "enable_ortools": true,                  // OR-Tools ソルバーを使用（デフォルト: true）
  "board_time": 1.0,                       // 停留所あたりの乗降時間（分）（enable_ortools=true 時は無視）
  "max_delay_time": 15.0,                  // 許容最大遅延時間（分）
  "mobility_speed": 333.33,                // 車両速度（m/分）（デフォルト: 20km/h）
  "max_calculation_seconds": 30,           // ソルバーの時間制限（秒）（デフォルト: 30）
  "max_calculation_stop_times_length": 10, // ブルートフォース用の最大停留所数（デフォルト: 10）
  "mobilities": [
    {
      "mobility_id": "car-1",             // 車両の一意識別子
      "trip_id": "trip-A",               // サービスエリアを定義する GTFS FLEX 便
      "capacity": 4,                      // 乗客定員
      "stop": "depot-stop"               // 初期停留所 ID
    }
  ]
}
```

- `reference_time`: GTFS FLEX カレンダーの解決に使用する基準日（YYYYMMDD、ちょうど 8 文字）。
- `input_files`: GTFS FLEX zip ファイルを 1 つ指定します（ローカルネットワークファイルと組み合わせる場合は最大 2 つ）。
- `network`: 停留所間の移動時間行列。`fetch_url`（プランナーからリアルタイムで取得）または `filename`（`stops` と `matrix` キーを持つ事前生成 JSON）を指定します。
- `enable_ortools`: `true` の場合、最適配車に OR-Tools を使用します。`board_time` は自動的に `0` に設定され、警告がログに出力されます。
- `board_time`: 停留所ごとの乗降追加時間（`enable_ortools` が `false` の場合のみ使用）。
- `max_delay_time`: ユーザーの希望出発時刻に対する許容乗車遅延の上限。この遅延を超える予約は拒否されます。
- `mobilities`: 車両定義のリスト。各車両は、許可されたサービスエリアと運行ウィンドウを定義する GTFS FLEX 便に従います。

:::warning
`enable_ortools` が `true` の場合、`board_time` は無視されて `0` に上書きされます。`board_time` はブルートフォースソルバーを使用する場合にのみ設定してください。
:::

### 出力

- `RESERVED` イベント（成功）:
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "mobilityId": "car-1",
      "route": [
        {
          "org": { "locationId": "StopA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StopB", "lat": 35.02, "lng": 139.02 },
          "dept": 485.0,
          "arrv": 498.0
        }
      ]
    }
  }
  ```
- `RESERVED` イベント（失敗）:
  ```json
  { "eventType": "RESERVED", "details": { "userId": "U001", "demandId": "D_1", "success": false } }
  ```
- `DEPARTED`・`ARRIVED` イベントには車両の現在停留所の位置と `mobilityId` が格納されます。

## ワンウェイ（`oneway`）

- ファイル: `maasblender/src/base_simulators/oneway/`
- 主要クラス: `Simulation`（`simulation.py`）、API サーバー: `controller.py`

ワンウェイシミュレーターは、GBFS データを使ったステーションベースの自由浮動型車両共有（例: 電動キックボードや共有自転車）をモデル化します。
ユーザーは起点ステーションで車両を予約し、目的地ステーションのドックを確保します。ユーザーが出発した後、車両はステーション間を自律的に移動します。
バックグラウンドのオペレータープロセスが定期的にステーション間で車両を再配置（リバランス）します。

### 動作

- `RESERVE` 時:
  - 起点ステーションに利用可能な（予約可能な）車両が少なくとも 1 台 **かつ** 目的地ステーションに利用可能なドックが少なくとも 1 つあることを確認します。
  - 両方の条件が満たされた場合、車両とドックを予約し、`mobility_speed` に基づいて計算した `arrv` を含む `RESERVED`（`success: true`）を発行します。
  - いずれかの条件が満たされない場合、`RESERVED`（`success: false`）を発行します。
- `DEPART` 時:
  - ユーザーが起点ステーションで予約した車両を引き取り、`DEPARTED` が発行されます。
  - 車両は `mobility_speed` で目的地ステーションへ移動し、到着時に駐車されて `ARRIVED` が発行されます。
- バッテリーシミュレーション: 車両の残充電量（SoC）は移動中に減少し（`discharging_speed`）、充電ステーションに停車中に回復します（`charging_speed`）。
- オペレーターによるリバランス: バックグラウンドのオペレータープロセスが `operator_start_time` から `operator_end_time` の間に稼働し、`operator_interval` 分ごとにステーションの均衡を確認して、1 回のトリップあたり最大 `operator_capacity` 台の車両を移動させます。
- `/reservable` エンドポイントは、`org` に利用可能な車両があり、`dst` に利用可能なドックがあるかをリアルタイムで確認します。

### 設定

ワンウェイシミュレーターは、GBFS zip ファイルをアップロードした後に設定します。

```json
{
  "input_files": [
    { "filename": "gbfs.zip" }              // GBFS アーカイブ（station_information + free_bike_status）
  ],
  "mobility_speed": 200.0,                  // 車両移動速度（m/分）（デフォルト: 200 ≈ 12km/h）
  "charging_speed": 0.003333,               // SoC 増加レート（/分）（デフォルト: 約 5 時間でフル充電）
  "discharging_speed": -0.004386,           // SoC 減少レート（/分）（デフォルト: 約 3 時間 38 分で完全放電）
  "operator_start_time": 360.0,             // オペレーター稼働開始（分）（デフォルト: 360 = 06:00）
  "operator_end_time": 720.0,               // オペレーター稼働終了（分）（デフォルト: 720 = 12:00）
  "operator_interval": 15.0,               // リバランス間隔（分）（デフォルト: 15）
  "operator_speed": 1000.0,                 // オペレーター車両速度（m/分）（デフォルト: 1000 = 60km/h）
  "operator_loading_time": 1,              // 車両 1 台の積み下ろし時間（分）（デフォルト: 1）
  "operator_capacity": 4                   // オペレーター 1 回のトリップあたりの最大積載台数（デフォルト: 4）
}
```

- `input_files`: `station_information.json` と `free_bike_status.json` を少なくとも含む GBFS zip ファイルを 1 つ指定します。
- `mobility_speed`: ステーション間の移動時間の計算に使用する速度。
- `charging_speed` / `discharging_speed`: バッテリー充電レート（正）および放電レート（負）。いずれも 1 分あたりの SoC 分率で指定します。
- `operator_*`: バックグラウンドで動作する自動リバランスオペレーターを制御するパラメーター。

:::tip
GBFS ファイル内の各自転車の `current_range_meters` フィールドを使って、シミュレーション開始時の各車両のバッテリー SoC が初期化されます。
:::

### 出力

- `RESERVED` イベント（成功）:
  ```json
  {
    "eventType": "RESERVED",
    "details": {
      "userId": "U001",
      "demandId": "D_1",
      "success": true,
      "mobilityId": "bike-42",
      "route": [
        {
          "org": { "locationId": "StationA", "lat": 35.0, "lng": 139.0 },
          "dst": { "locationId": "StationB", "lat": 35.01, "lng": 139.01 },
          "dept": 480.0,
          "arrv": 483.5
        }
      ]
    }
  }
  ```
- `RESERVED` イベント（失敗）:
  ```json
  { "eventType": "RESERVED", "details": { "userId": "U001", "demandId": "D_1", "success": false } }
  ```
- `DEPARTED`・`ARRIVED` イベントには `mobilityId`（特定の自転車/キックボードの ID）とステーションの位置情報が格納されます。

## 運用上の共通メモ

- 5 つのモビリティサービス実装すべてが同じ REST API コントラクトを共有しています: `/setup`、`/start`、`/peek`、`/step`、`/triggered`、`/reservable`、`/finish`。
- すべての実装は `/triggered` エンドポイント経由で `RESERVE` および `DEPART` イベントを受け取り、その応答として `RESERVED`、`DEPARTED`、`ARRIVED` イベントを発行します。
- `/reservable` エンドポイントは、ルートプランナーや評価コンポーネントが `RESERVE` を発行する前に、指定した起点・目的地ペアに対してサービスが新規予約を受け付けられるかどうかを確認するために使用されます。
