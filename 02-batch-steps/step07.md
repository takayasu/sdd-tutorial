# Step 7: EventBridge + Step Functions スケジューリング (LocalStack)

## 目的

### これは何か

LocalStack 上の EventBridge Scheduler でバッチを定期実行し、Step Functions でジョブの起動・結果記録・通知を管理する。クラウドサービスを「薄いラッパー」として使い、バッチの核心ロジックはアプリ側に保つ設計を実践する。

### なぜやるのか

- Step 2-6 のバッチは手動でコマンドを叩いて実行していた。本番では cron のように定期実行する仕組みが必要
- Spring Batch 自体にスケジューラはなく、`@Scheduled` や Quartz で起動していた。ここでは EventBridge Scheduler で代替する
- Step Functions は「起動して、結果を記録して、失敗したら通知する」だけの薄いラッパー。バッチロジックには一切関与しない

### 何がうれしいのか

- `awslocal` コマンドで EventBridge ルールと Step Functions ステートマシンを作成し、ローカルで定期実行の動作を確認できる
- Step Functions のコンソール（LocalStack Dashboard）でジョブの実行履歴を視覚的に確認できる
- 本番では LocalStack を AWS に差し替えるだけ。アプリのコードは一切変更不要

## 事前準備

Step 1 で LocalStack は起動済み。`events` と `stepfunctions` サービスが有効であることを確認:

```bash
curl -s http://localhost:4566/_localstack/health | jq '.services | {events, stepfunctions}'
# → {"events":"running","stepfunctions":"running"}
```

## Step Functions ステートマシンの作成

### ステートマシン定義

バッチアプリをコンテナ（ECS Task）で起動する代わりに、LocalStack では Lambda で代替する。実際のバッチロジックは Lambda 内からアプリのバッチエントリポイントを呼び出す形。

```json
// localstack/state-machine.json
{
  "Comment": "Batch job orchestrator - thin wrapper",
  "StartAt": "RunBatch",
  "States": {
    "RunBatch": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "batch-runner",
        "Payload": {
          "jobName.$": "$.jobName",
          "jobParams.$": "$.jobParams"
        }
      },
      "Retry": [{
        "ErrorEquals": ["States.TaskFailed"],
        "IntervalSeconds": 60,
        "MaxAttempts": 2,
        "BackoffRate": 2.0
      }],
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "NotifyFailure"
      }],
      "Next": "NotifySuccess"
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "http://sqs.ap-northeast-1.localhost.localstack.cloud:4566/000000000000/batch-notifications",
        "MessageBody": {
          "status": "SUCCESS",
          "jobName.$": "$.jobName",
          "jobParams.$": "$.jobParams"
        }
      },
      "End": true
    },
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "http://sqs.ap-northeast-1.localhost.localstack.cloud:4566/000000000000/batch-notifications",
        "MessageBody": {
          "status": "FAILURE",
          "jobName.$": "$.jobName",
          "error.$": "$.Error"
        }
      },
      "End": true
    }
  }
}
```

### LocalStack にデプロイ

```bash
REGION=ap-northeast-1

# ステートマシン作成
awslocal stepfunctions create-state-machine \
  --name batch-orchestrator \
  --definition file://localstack/state-machine.json \
  --role-arn arn:aws:iam::000000000000:role/dummy \
  --region $REGION

# 確認
awslocal stepfunctions list-state-machines --region $REGION
# → {"stateMachines":[{"name":"batch-orchestrator",...}]}
```

## EventBridge Scheduler の作成

```bash
REGION=ap-northeast-1
STATE_MACHINE_ARN=$(awslocal stepfunctions list-state-machines --region $REGION --query 'stateMachines[0].stateMachineArn' --output text)

# 毎月1日 0:00 に月次締めバッチを実行するルール
awslocal events put-rule \
  --name monthly-close-schedule \
  --schedule-expression "cron(0 0 1 * ? *)" \
  --region $REGION

# ターゲット: Step Functions ステートマシン
awslocal events put-targets \
  --rule monthly-close-schedule \
  --targets "[{
    \"Id\": \"monthly-close\",
    \"Arn\": \"$STATE_MACHINE_ARN\",
    \"Input\": \"{\\\"jobName\\\":\\\"monthly-close\\\",\\\"jobParams\\\":\\\"$(date +%Y-%m)\\\"}\"
  }]" \
  --region $REGION

# 確認
awslocal events list-rules --region $REGION
awslocal events list-targets-by-rule --rule monthly-close-schedule --region $REGION
```

## 完了条件

### フェーズA: アプリ単体（バッチがコマンドラインから正しく動くこと）

このステップで新しいアプリコードは追加しない。Step 2-6 で作ったバッチがコマンドラインから正常に動作することを再確認する:

1. バッチをコマンドラインから実行し、正常完了すること:

```bash
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-04
# or
gradle run --args="--job=monthly-close --date=2026-04"

# → batch_job_execution.status = COMPLETED
```

2. `batch_job_execution` に実行履歴が記録されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT job_name, status, read_count, write_count FROM batch_job_execution ORDER BY started_at DESC LIMIT 1;"
# → monthly-close | COMPLETED | 10000 | 10000
```

フェーズA が通れば、アプリ側は問題ない。以降の確認で失敗した場合は LocalStack 側の問題。

---

### フェーズB: インフラ構築（LocalStack リソースの作成）

3. Step Functions ステートマシンが作成されていること:

```bash
awslocal stepfunctions list-state-machines --region ap-northeast-1
# → {"stateMachines":[{"name":"batch-orchestrator",...}]}
```

4. EventBridge ルールが作成されていること:

```bash
awslocal events list-rules --region ap-northeast-1 | jq '.Rules[] | {Name, ScheduleExpression, State}'
# → {"Name":"monthly-close-schedule","ScheduleExpression":"cron(0 0 1 * ? *)","State":"ENABLED"}
```

5. EventBridge ルールのターゲットが Step Functions に向いていること:

```bash
awslocal events list-targets-by-rule --rule monthly-close-schedule --region ap-northeast-1 | jq '.Targets[0].Arn'
# → "arn:aws:states:ap-northeast-1:000000000000:stateMachine:batch-orchestrator"
```

フェーズB が通れば、インフラリソースは正しく構築されている。

---

### フェーズC: 構造確認 + 手動バッチ実行

LocalStack 上のステートマシンは Lambda 関数（`batch-runner`）を呼び出す定義になっているが、LocalStack で Lambda を動かすには Docker in Docker + ランタイム同梱が必要で、セットアップが重い。本番では Lambda の代わりに ECS Task や k8s Job でバッチアプリを起動する構成になるため、ここでは Lambda 連携は行わず、以下を確認する:

- ステートマシンの実行フローが正しく定義されていること（構造確認）
- バッチ自体はコマンドラインから正常に動くこと（フェーズA で確認済み）
- Step Functions の二重実行防止が機能すること

6. ステートマシンの定義を取得し、フロー構造が正しいこと:

```bash
REGION=ap-northeast-1
STATE_MACHINE_ARN=$(awslocal stepfunctions list-state-machines --region $REGION --query 'stateMachines[0].stateMachineArn' --output text)

awslocal stepfunctions describe-state-machine \
  --state-machine-arn $STATE_MACHINE_ARN \
  --region $REGION | jq '.definition | fromjson | .States | keys'
# → ["NotifyFailure", "NotifySuccess", "RunBatch"]
```

7. ステートマシンの実行を開始できること（Lambda 未作成のため RunBatch で失敗し NotifyFailure に遷移するが、フロー自体は動く）:

```bash
EXECUTION_ARN=$(awslocal stepfunctions start-execution \
  --state-machine-arn $STATE_MACHINE_ARN \
  --name "monthly-close-2026-04" \
  --input '{"jobName":"monthly-close","jobParams":"2026-04"}' \
  --region $REGION \
  --query 'executionArn' --output text)

# 実行履歴を確認（Lambda 未作成のため FAILED になるが、ステートマシン自体は起動できている）
awslocal stepfunctions describe-execution \
  --execution-arn $EXECUTION_ARN \
  --region $REGION | jq '{status, startDate}'
```

8. 同一名で二重実行しようとすると拒否されること:

```bash
awslocal stepfunctions start-execution \
  --state-machine-arn $STATE_MACHINE_ARN \
  --name "monthly-close-2026-04" \
  --input '{"jobName":"monthly-close","jobParams":"2026-04"}' \
  --region $REGION
# → ExecutionAlreadyExists エラー
```

> **本番デプロイ時の課題**: ステートマシンの `RunBatch` ステートを Lambda invoke から ECS RunTask（`arn:aws:states:::ecs:runTask.sync`）や k8s Job に差し替える。バッチアプリのコードは変更不要。

### トラブルシューティング

| 症状 | 原因の切り分け |
|---|---|
| フェーズA で失敗 | アプリのコード or DB の問題。Step 2-6 を見直す |
| フェーズA は通るがフェーズB で失敗 | LocalStack の起動 or awslocal コマンドの問題。`curl localhost:4566/_localstack/health` を確認 |
| フェーズB は通るがフェーズC で失敗 | ステートマシン定義の問題。`state-machine.json` を見直す |

### 確認のコツ

- LocalStack の Step Functions は完全互換ではないが、基本的なステートマシン実行は動作する
- `awslocal` は `aws` CLI のラッパーで、`--endpoint-url http://localhost:4566` を自動付与する
- 初期化スクリプト（`localstack/init/setup.sh`）にステートマシンとEventBridgeルールの作成を追加すると、`docker compose up` だけで全て構築される

---

## 次のステップ

Step 7が完了したら [Step 8: CloudWatch 監視 + 完成](./step08.md) へ進む。
