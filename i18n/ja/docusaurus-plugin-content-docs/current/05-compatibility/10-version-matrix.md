---
sidebar_position: 10
title: バージョン互換性マトリクス
---

このページでは、MaaS Blender の各コンポーネントと外部依存関係（データフォーマット仕様を含む）のバージョン互換性をまとめています。
入力データの準備や個々のコンポーネントのアップグレード時には、このページを参照してください。

## コンポーネント バージョン一覧

| コンポーネント | 採用バージョン | 備考 |
|-----------|----------------|------|
| OpenTripPlanner (OTP) | 2.6.0 | JDK 21 が必要 |
| オンデマンドシミュレーター | — | 最適化に OR-Tools 9.12 を使用 |
| シンプル内部プランナー | — | 軽量実装、外部ランタイム依存なし |
| Python ランタイム（全コンポーネント共通） | 3.11 | 必要最小バージョン |

## GTFS Flex 互換性

[GTFS Flex](https://gtfs.org/extensions/flex/) は、ドラフト仕様から GTFS 公式標準に正式採用されたバージョン（v2）の間で、構造的な仕様変更が行われています。
ロケーショングループのメンバーシップ情報やストップ時刻の表現方法が両仕様間で異なります。

### 仕様の違い

| フィールド / ファイル | 旧 GTFS Flex 仕様 | 新 GTFS Flex 仕様（v2） |
|---|---|---|
| ロケーショングループのメンバーシップ | `location_groups.txt` の `location_id` 列 | 専用ファイル `location_group_stops.txt` |
| `stop_times.txt` のストップ参照 | `stop_id` 列 | `location_group_id` 列 |

### コンポーネント別サポート状況

| コンポーネント | 旧 GTFS Flex 仕様 | 新 GTFS Flex 仕様（v2） |
|-----------|:-----------------:|:----------------------:|
| **OTP プランナー**（OTP 2.6.0） | ❌ | ✅ |
| **オンデマンドシミュレーター** | ✅ | ✅ |
| **シンプル内部プランナー** | ✅ | ❌ |

:::warning
**OTP プランナー** は、新 v2 形式の GTFS Flex データ（`location_group_stops.txt` と `stop_times.txt` の `location_group_id` 列）を必要とします。
旧形式の GTFS Flex データを使用した場合、グラフのビルド失敗または誤ったルーティング結果が発生します。
:::

:::warning
**シンプル内部プランナー** は、旧 GTFS Flex 形式（`location_groups.txt` の `location_id` 列、`stop_times.txt` の `stop_id` 列）のみをサポートしています。
新 v2 形式のファイルはこのプランナーではサポートされていません。
:::

:::info
**オンデマンドシミュレーター** は両方の形式を透過的に処理します。
`stop_times.txt` から `location_group_id` を優先して読み込み、列が存在しない場合は `stop_id` にフォールバックします。また、`location_group_stops.txt` が存在する場合はそちらからメンバーシップ情報を読み込み、存在しない場合は `location_groups.txt` 内の `location_id` を使用します。
:::

## OpenTripPlanner バージョン メモ

MaaS Blender は **OpenTripPlanner 2.6.0** を採用しており、OTP 2.x の GraphQL API（`/otp/routers/default/index/graphql`）経由で通信しています。

:::warning
OTP 2.x の GraphQL API は、**OTP 1.x の REST API と互換性がありません**。
別の OTP バージョンに置き換える場合は、同一の OTP 2.x GraphQL エンドポイントが提供されていることを確認してください。
OTP 2.0 より前のバージョンはサポートされていません。
:::

OTP 連携の主な制約事項：

- OTP 2.6.0 の実行には **JDK 21** が必要です。
- OTP のグラフビルド処理では、ブローカーのセットアップ確定**前**にすべての入力ファイル（`otp-config.zip`、GTFS/GTFS Flex の zip ファイルなど）のアップロードが完了している必要があります。
- GTFS Flex のルーティングには OTP の `FLEX` トランスポートモード修飾子を使用します。`otp_config` のルーター設定ファイルでフレキシブルトランジットが有効になっていることを確認してください。

OTP プランナーの設定詳細については、Route Planner の章を参照してください。
