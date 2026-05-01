# Batch Steps: Spring Batch 機能の代替実装

Spring Batch が業務バッチ処理に提供する機能を、F#（.NET）と Kotlin（JVM）で代替する。01-steps/ を全て完了した前提で、バッチ処理基盤として本番運用に耐えうるかを PoC する。

Web API の業務システム機能については [02-advanced-steps/](../02-advanced-steps/) を参照。

---

## 設計方針

### ベンダーロックインの境界線

バッチの核心ロジック（二重実行防止、チャンクリスタート、進捗管理）はアプリ内の DB 操作で自己完結させる。クラウドサービスには「起動」と「監視」だけを任せる。

| 役割 | どこに持つか | 理由 |
|---|---|---|
| チャンク処理・リスタート | アプリ側（DB） | バッチの核心。クラウドに渡すとロックイン |
| 二重実行防止 | アプリ側（DB ロック） | クラウド機能に依存すると移行時に再実装が必要 |
| リトライ（チャンク単位） | アプリ側 | チャンクリスタートがあればジョブ単位リトライで十分 |
| 並列処理 | アプリ側（言語の並列機構） | Map State 等に依存しない |
| 処理件数記録 | アプリ側（DB） | 運用監視に必要 |
| スケジューリング（Cron 起動） | クラウド（EventBridge 等） | cron 起動はどのクラウドにもある。k8s CronJob にも差し替え可 |
| ジョブ起動・結果通知 | クラウド（Step Functions 等） | 薄いラッパー。なくても動く |
| 監視・アラート | クラウド（CloudWatch 等） | Prometheus/Grafana にも差し替え可 |

### 前提

- 「Spring Batch の全機能再現」ではなく「業務バッチに必要な運用レベルの確保」が目的
- 外部ライブラリ（Hangfire, Jobrunr 等）には依存しない
- k8s Job 起動パターンでは Spring Batch の手動停止は実質無効、JobParameters の渡し方次第ではリスタートも無効の可能性がある

---

## 機能対応一覧

| Spring Batch 機能 | F# (.NET) | Kotlin (JVM) | 実装場所 | Step |
|---|---|---|---|---|
| チャンク処理 | `Seq.chunkBySize` + トランザクション | `Sequence.chunked` + Exposed transaction | アプリ | [Step 2](./step02.md) |
| ジョブ実行管理 | `batch_job_execution` テーブル + `runBatch` 関数 | 同左 | アプリ（DB） | [Step 3](./step03.md) |
| 二重実行防止 | DB の PK 制約 + ステータスチェック | 同左 | アプリ（DB） | [Step 3](./step03.md) |
| リスタート（オフセット再開） | `batch_chunk_progress` テーブル | 同左 | アプリ（DB） | [Step 4](./step04.md) |
| スキップ/リトライ | `Result` 型 + チャンクループ内 | `Either` + チャンクループ内 | アプリ | [Step 5](./step05.md) |
| リスナー/フック | 高階関数でラップ | 同左 | アプリ | [Step 5](./step05.md) |
| 並列処理 | `Async.Parallel` | `coroutineScope` | アプリ | [Step 6](./step06.md) |
| スケジューリング | EventBridge Scheduler | 同左 | クラウド（薄い） | [Step 7](./step07.md) |
| ダッシュボード | Step Functions コンソール + CloudWatch | 同左 | クラウド（薄い） | [Step 7](./step07.md) / [Step 8](./step08.md) |
| 監視・アラート | CloudWatch Logs + Metrics | 同左 | クラウド（薄い） | [Step 8](./step08.md) |
| CSV インポート | `FlatFileItemReader` | CsvHelper / Jackson CSV | アプリ | [Step 9](./step09.md) |

---

## 推奨アーキテクチャ

### 全体構成

```
┌─────────────────────────────────────────────────────────┐
│ クラウド層（薄い・差し替え可能）                            │
│                                                         │
│  EventBridge Scheduler ──→ Step Functions（薄いラッパー） │
│                              │  起動 → 結果記録 → 通知   │
│                              ▼                          │
│                           ECS Task / k8s Job            │
│  CloudWatch ← ログ・メトリクス                            │
└──────────────────────────────┬──────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────┐
│ アプリ層（自己完結・クラウド非依存）                         │
│                                                         │
│  runBatch()                                             │
│    ├── tryStart()        → 二重実行防止（DB）             │
│    ├── getLastProcessedId() → リスタート位置取得（DB）     │
│    ├── processInChunks() → チャンク処理ループ             │
│    │     └── 各チャンク: 業務処理 + 進捗更新（同一TX）     │
│    └── completeJob() / failJob() → ステータス更新（DB）   │
│                                                         │
│  依存: DB ライブラリ（Donald / Exposed）のみ              │
│  外部ライブラリ追加: なし                                  │
└──────────────────────────────┬──────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────┐
│ PostgreSQL                                              │
│  ├── 業務テーブル（ロット、販売案件等）                     │
│  ├── batch_job_execution（ジョブ管理）                    │
│  └── batch_chunk_progress（チャンク進捗）                 │
└─────────────────────────────────────────────────────────┘
```

### クラウド層の差し替え

| 現行（AWS） | 差し替え先 | 影響 |
|---|---|---|
| EventBridge Scheduler | k8s CronJob / Cloud Scheduler (GCP) / cron | アプリ変更なし |
| Step Functions | Cloud Run Jobs (GCP) / なし（直接起動） | アプリ変更なし |
| CloudWatch | Prometheus + Grafana | アプリ変更なし |
| ECS Task | k8s Job / Cloud Run Jobs | コンテナイメージそのまま |

### ユースケース別の構成

| ユースケース | 構成 | 備考 |
|---|---|---|
| 月次締め処理 | EventBridge → Step Functions → ECS Task → `runBatch` | リスタート必須。進捗テーブルで対応 |
| 大量ロット一括状態遷移 | 同上 | 冪等設計なら進捗テーブルなしでも可 |
| 棚卸・在庫計算 | 同上 | 並列パーティション推奨 |
| CSV インポート | 同上 | ファイル→DB。行番号でリスタート |

---

## まとめ

### 自前実装の総量

- テーブル: 2（`batch_job_execution` + `batch_chunk_progress`）
- コード: 約 100〜120 行（`tryStart` + `runBatch` + `processInChunks` + `upsertProgress` + リスナー型定義）
- 外部ライブラリ追加: なし（DB ライブラリは既存）

### 依存関係

```
アプリの依存:
  - DB ライブラリ（Donald / Exposed）← 既に使っている
  - 2 テーブル + runBatch 関数
  - 外部ライブラリ追加: なし

クラウドの依存（薄い・差し替え可能）:
  - EventBridge Scheduler → k8s CronJob に差し替え可
  - Step Functions → 起動 + 通知だけ。なくても動く
  - CloudWatch → Prometheus/Grafana に差し替え可
```

バッチの核心ロジック（二重実行防止、チャンクリスタート、進捗管理）は全てアプリ内の DB 操作で自己完結する。クラウドサービスは「起動」と「監視」だけを担い、ベンダーロックインの境界を明確に保つ。
