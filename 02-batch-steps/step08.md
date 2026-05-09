# Step 8: CloudWatch 監視 + 完成

## 目的

### これは何か

バッチの実行ログを LocalStack の CloudWatch Logs に送信し、メトリクス（処理件数、実行時間、エラー率）を CloudWatch Metrics に記録する。バッチ失敗時のアラート設定も行い、バッチ処理基盤を完成させる。

### なぜやるのか

- Step 1-7 でバッチの実行ロジックとスケジューリングは完成したが、「バッチが正常に動いているか」を継続的に監視する仕組みがない
- Spring Boot Actuator + CloudWatch の組み合わせに相当する監視基盤を構築する
- 「バッチが失敗したら通知する」「処理件数が異常に少なかったら通知する」といった運用アラートを設定する

### 何がうれしいのか

- `awslocal logs` でバッチのログを検索できる。「昨日の月次締めバッチのエラーログだけ見たい」が1コマンドで実現する
- メトリクスにより「先月と比べて処理件数が半分になっている」「実行時間が倍になっている」といった異常を検知できる
- アラートにより、バッチ失敗時に自動で SQS に通知が送られる。人間が毎朝ログを確認する運用から脱却できる
- これで Spring Batch が提供していた全機能の代替が完成する

## 完了条件

### フェーズA: アプリ単体（メトリクスデータの生成）

アプリがバッチ完了時にメトリクスデータを構造化ログとして出力できることを確認する。CloudWatch への送信はフェーズB。

1. バッチを実行し、完了時にメトリクスがログに出力されること:

```bash
uv run python -m batch.runner --job=monitoring-test --date=2026-04

# ログに以下のようなメトリクス情報が含まれること:
# {"event":"Job completed","job":"monitoring-test","metrics":{"processed_count":10000,"execution_time_ms":15000,"skip_count":3,"error_count":0}}
```

2. `batch_job_execution` テーブルに件数が正しく記録されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT job_name, status, read_count, write_count, skip_count FROM batch_job_execution WHERE job_name = 'monitoring-test';"
# → monitoring-test | COMPLETED | 10000 | 9997 | 3
```

3. アプリ内に CloudWatch にメトリクスを送信する関数が実装されていること（送信先の URL は設定ファイルで切り替え可能）

フェーズA が通れば、アプリのメトリクス生成ロジックは正しい。

---

### フェーズB: インフラ構築（CloudWatch リソースの作成）

4. CloudWatch Logs のロググループが作成されていること:

```bash
awslocal logs describe-log-groups --region ap-northeast-1 | jq '.logGroups[] | .logGroupName'
# → "/batch/sales-management"
```

5. CloudWatch Alarm が作成されていること:

```bash
awslocal cloudwatch describe-alarms --region ap-northeast-1 | jq '.MetricAlarms[] | {AlarmName, StateValue}'
# → {"AlarmName":"batch-failure-alarm","StateValue":"OK"}
```

6. 初期化スクリプトに全リソースの作成が含まれ、`docker compose up` だけで全て構築されること:

```bash
docker compose down -v
docker compose up -d
# 30秒待ってから
awslocal stepfunctions list-state-machines --region ap-northeast-1
awslocal events list-rules --region ap-northeast-1
awslocal sqs list-queues --region ap-northeast-1
awslocal logs describe-log-groups --region ap-northeast-1
awslocal cloudwatch describe-alarms --region ap-northeast-1
# → 全リソースが自動作成されている
```

---

### フェーズC: 一気通貫（アプリ → CloudWatch 連携）

7. バッチを実行し、CloudWatch Logs にログが送信されること:

```bash
REGION=ap-northeast-1

uv run python -m batch.runner --job=e2e-test --date=2026-04

# CloudWatch Logs を検索
awslocal logs filter-log-events \
  --log-group-name /batch/sales-management \
  --filter-pattern "e2e-test" \
  --region $REGION | jq '.events[] | .message' | head -5
# → バッチのログメッセージが表示される
```

8. バッチ完了後、CloudWatch Metrics にカスタムメトリクスが記録されること:

```bash
awslocal cloudwatch list-metrics --namespace "BatchProcessing" --region $REGION | jq '.Metrics[] | .MetricName'
# → "ProcessedCount", "ExecutionTime", "SkipCount", "ErrorCount"
```

9. エラーログだけをフィルタできること:

```bash
awslocal logs filter-log-events \
  --log-group-name /batch/sales-management \
  --filter-pattern '{ $.level = "error" }' \
  --region $REGION
```

### トラブルシューティング

| 症状 | 原因の切り分け |
|---|---|
| フェーズA で失敗 | バッチのコード or DB の問題。Step 2-6 を見直す |
| フェーズA は通るがフェーズB で失敗 | LocalStack の起動 or 初期化スクリプトの問題 |
| フェーズB は通るがフェーズC で失敗 | アプリの AWS SDK 設定（エンドポイント URL）の問題。`http://localhost:4566` に向いているか確認 |

---

## 実装ガイド

### アプリからの CloudWatch 連携

バッチの `run_batch` 関数の完了時に、CloudWatch Logs と Metrics にデータを送信する:

```
run_batch 完了時:
  1. batch_job_execution に記録（既存、Step 3）
  2. CloudWatch Logs にログ送信（新規）
  3. CloudWatch Metrics にメトリクス送信（新規）
```

### Python / boto3

```bash
uv add boto3
```

```python
# batch/cloudwatch.py
import os
from datetime import datetime, timezone

import boto3

_endpoint = os.environ.get("AWS_ENDPOINT_URL", "http://localhost:4566")
_region = os.environ.get("AWS_DEFAULT_REGION", "ap-northeast-1")

_logs = boto3.client("logs", endpoint_url=_endpoint, region_name=_region)
_cw = boto3.client("cloudwatch", endpoint_url=_endpoint, region_name=_region)

LOG_GROUP = "/batch/sales-management"


def put_metrics(job_name: str, processed: int, elapsed_ms: int, skipped: int, errors: int) -> None:
    now = datetime.now(timezone.utc)
    _cw.put_metric_data(
        Namespace="BatchProcessing",
        MetricData=[
            {"MetricName": "ProcessedCount", "Value": processed, "Unit": "Count", "Timestamp": now},
            {"MetricName": "ExecutionTime", "Value": elapsed_ms, "Unit": "Milliseconds", "Timestamp": now},
            {"MetricName": "SkipCount", "Value": skipped, "Unit": "Count", "Timestamp": now},
            {"MetricName": "ErrorCount", "Value": errors, "Unit": "Count", "Timestamp": now},
        ],
    )


def put_log(job_name: str, message: str) -> None:
    log_stream = job_name
    try:
        _logs.create_log_stream(logGroupName=LOG_GROUP, logStreamName=log_stream)
    except _logs.exceptions.ResourceAlreadyExistsException:
        pass

    _logs.put_log_events(
        logGroupName=LOG_GROUP,
        logStreamName=log_stream,
        logEvents=[{"timestamp": int(datetime.now(timezone.utc).timestamp() * 1000), "message": message}],
    )
```

### LocalStack 初期化スクリプトの最終版

```bash
#!/bin/bash
# localstack/init/setup.sh
REGION=ap-northeast-1

echo "=== Setting up LocalStack resources ==="

# SQS
awslocal sqs create-queue --queue-name batch-notifications --region $REGION

# CloudWatch Logs
awslocal logs create-log-group --log-group-name /batch/sales-management --region $REGION

# Step Functions
awslocal stepfunctions create-state-machine \
  --name batch-orchestrator \
  --definition file:///etc/localstack/init/ready.d/state-machine.json \
  --role-arn arn:aws:iam::000000000000:role/dummy \
  --region $REGION

# EventBridge
STATE_MACHINE_ARN=$(awslocal stepfunctions list-state-machines --region $REGION --query 'stateMachines[0].stateMachineArn' --output text)

awslocal events put-rule \
  --name monthly-close-schedule \
  --schedule-expression "cron(0 0 1 * ? *)" \
  --region $REGION

awslocal events put-targets \
  --rule monthly-close-schedule \
  --targets "[{\"Id\":\"monthly-close\",\"Arn\":\"$STATE_MACHINE_ARN\",\"Input\":\"{\\\"jobName\\\":\\\"monthly-close\\\"}\"}]" \
  --region $REGION

# CloudWatch Alarm
awslocal cloudwatch put-metric-alarm \
  --alarm-name batch-failure-alarm \
  --metric-name ErrorCount \
  --namespace BatchProcessing \
  --statistic Sum \
  --period 300 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sqs:$REGION:000000000000:batch-notifications" \
  --region $REGION

echo "=== LocalStack setup complete ==="
```

---

## おめでとうございます！

Step 8が完了すると、Spring Batch の全機能を代替するバッチ処理基盤が完成します:

| Spring Batch 機能 | 実装済み | 出典 |
|---|---|---|
| チャンク処理（Reader → Processor → Writer） | `process_in_chunks` 関数 | Step 2 |
| ジョブ実行管理 | `batch_job_execution` テーブル + `try_start` | Step 3 |
| 二重実行防止 | DB PK 制約 + ステータスチェック | Step 3 |
| チャンクリスタート | `batch_chunk_progress` テーブル + `upsert_progress` | Step 4 |
| スキップ/リトライ | `ChunkConfig` + `process_item_with_retry` | Step 5 |
| リスナー/フック | `BatchListeners` dataclass | Step 5 |
| 並列処理（パーティショニング） | `asyncio.gather` | Step 6 |
| スケジューリング | EventBridge Scheduler (LocalStack) | Step 7 |
| ジョブ起動・通知 | Step Functions (LocalStack) | Step 7 |
| 監視・アラート | CloudWatch Logs + Metrics (LocalStack) | Step 8 |

### 設計方針の振り返り

```
アプリ側（自己完結・クラウド非依存）:
  ✅ チャンク処理、リスタート、二重実行防止、スキップ/リトライ、並列処理
  → SQLAlchemy asyncio のみに依存。外部ライブラリ追加なし。

クラウド側（薄い・差し替え可能）:
  ✅ スケジューリング、ジョブ起動、監視・通知
  → LocalStack → AWS に差し替えるだけ。アプリ変更なし。
```

次のステップとして:
- [Step 9: CSV インポートバッチ](./step09.md) でファイルベースのバッチを実装する
- 本番デプロイに向けて、LocalStack の設定を AWS CDK / Terraform に変換する
- `ci.sh` にバッチのテスト（ジョブ管理テーブルの整合性チェック等）を追加する
- [README.md](./README.md) のユースケース（月次締め、棚卸、一括状態遷移）を実際のドメインロジックで実装する
