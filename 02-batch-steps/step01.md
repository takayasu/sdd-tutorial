# Step 1: LocalStack + ジョブ管理テーブル

## 目的

### これは何か

バッチ処理基盤の土台を作る。AWS のマネージドサービス（EventBridge, Step Functions, CloudWatch）をローカルで再現するために LocalStack を docker-compose に追加し、バッチのジョブ管理に必要なDBテーブルを作成する。

### なぜやるのか

- Spring Batch はメタデータテーブル5つでジョブの実行履歴・進捗・二重実行防止を管理していた。これを2テーブルに簡素化して自前で実装する
- 後続のステップで EventBridge（スケジューリング）や Step Functions（ジョブ起動）を使うが、AWS アカウントがなくても LocalStack でローカル検証できるようにする
- バッチの核心ロジックはアプリ内のDB操作で自己完結させ、クラウドサービスには「起動」と「監視」だけを任せる設計方針を最初に確立する

### 何がうれしいのか

- `docker compose up -d` だけで、PostgreSQL + LocalStack（AWS互換環境）が手元に揃う
- ジョブ管理テーブルにより「いつ・どのジョブが・どういう結果で終わったか」がDBに記録される。Spring Batch の管理画面に相当する情報がSQLで取得できる
- LocalStack により、AWS の課金を気にせずに EventBridge や Step Functions の動作を確認できる

## 事前準備: LocalStack を docker-compose に追加

### docker-compose.yml に追加

```yaml
  localstack:
    image: localstack/localstack:3.4
    ports:
      - "4566:4566"           # LocalStack Gateway
    environment:
      SERVICES: events,stepfunctions,logs,sqs
      DEFAULT_REGION: ap-northeast-1
      DEBUG: 0
    volumes:
      - ./localstack/init:/etc/localstack/init/ready.d  # 初期化スクリプト
```

### 初期化スクリプトを作成

```bash
mkdir -p localstack/init
```

```bash
#!/bin/bash
# localstack/init/setup.sh
# LocalStack 起動時に自動実行される

REGION=ap-northeast-1

echo "=== Setting up LocalStack resources ==="

# SQS キュー（Step 7 以降で使用）
awslocal sqs create-queue --queue-name batch-notifications --region $REGION

echo "=== LocalStack setup complete ==="
```

```bash
chmod +x localstack/init/setup.sh
```

### 起動と確認

```bash
docker compose up -d localstack

# LocalStack が起動したことを確認
curl http://localhost:4566/_localstack/health | jq .
# → {"services":{"events":"running","stepfunctions":"running","logs":"running","sqs":"running"}}

# awslocal コマンドのインストール（未インストールの場合）
pip install awscli-local

# SQS キューが作成されたことを確認
awslocal sqs list-queues --region ap-northeast-1
# → {"QueueUrls":["http://sqs.ap-northeast-1.localhost.localstack.cloud:4566/000000000000/batch-notifications"]}
```

## ジョブ管理テーブルのマイグレーション

### マイグレーションファイルを作成

Spring Batch のメタデータ5テーブルを2テーブルに簡素化する:

```sql
-- migrations/versions/XXXX_create_batch_tables.sql (Alembic で管理)

-- ジョブ実行管理（Spring Batch の JOB_INSTANCE + JOB_EXECUTION に相当）
CREATE TABLE batch_job_execution (
    job_name        TEXT NOT NULL,
    job_params      TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'RUNNING',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    read_count      INT NOT NULL DEFAULT 0,
    write_count     INT NOT NULL DEFAULT 0,
    skip_count      INT NOT NULL DEFAULT 0,
    error_message   TEXT,
    PRIMARY KEY (job_name, job_params)
);

-- チャンク進捗（Spring Batch の STEP_EXECUTION_CONTEXT に相当）
CREATE TABLE batch_chunk_progress (
    job_name          TEXT NOT NULL,
    job_params        TEXT NOT NULL,
    partition_id      INT NOT NULL DEFAULT 0,
    last_processed_id BIGINT NOT NULL,
    processed_count   INT NOT NULL DEFAULT 0,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (job_name, job_params, partition_id),
    FOREIGN KEY (job_name, job_params) REFERENCES batch_job_execution(job_name, job_params)
);
```

### マイグレーション実行

```bash
uv run alembic upgrade head
```

## 完了条件の確認方針

このチュートリアルでは、全ステップの完了条件を2フェーズに分けて確認する:

| フェーズ | 確認対象 | 失敗時の原因 |
|---|---|---|
| フェーズA: アプリ単体 | DB テーブル、アプリのロジック、コマンドライン実行 | コード、SQL、設定ファイルの問題 |
| フェーズB: インフラ連携 | LocalStack（EventBridge, Step Functions, CloudWatch, SQS） | LocalStack の設定、awslocal コマンド、ネットワークの問題 |

フェーズA が通らない状態でフェーズB に進まないこと。「アプリは正しく動くが、LocalStack との連携がうまくいかない」という切り分けができるようにする。

---

## 完了条件

### フェーズA: アプリ単体（DB テーブル）

1. マイグレーションが成功し、ジョブ管理テーブルが作成されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "\dt batch_*"
# → batch_job_execution, batch_chunk_progress の2テーブルが表示される
```

2. テーブルの構造が正しいこと:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "\d batch_job_execution"
# → job_name, job_params, status, started_at, ... が表示される

docker compose exec db psql -U app -d sales_management \
  -c "\d batch_chunk_progress"
# → job_name, job_params, partition_id, last_processed_id, ... が表示される
```

3. テーブルに手動で INSERT/SELECT できること（アプリから使う前にテーブル自体の動作を確認）:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "INSERT INTO batch_job_execution (job_name, job_params) VALUES ('test', '2026-04');
      SELECT * FROM batch_job_execution;
      DELETE FROM batch_job_execution WHERE job_name = 'test';"
# → 1行挿入 → 表示 → 削除 が成功する
```

### フェーズB: インフラ連携（LocalStack）

4. LocalStack が起動し、ヘルスチェックが通ること:

```bash
curl -s http://localhost:4566/_localstack/health | jq '.services.events'
# → "running"
```

5. `awslocal` コマンドで LocalStack のリソースにアクセスできること:

```bash
awslocal sqs list-queues --region ap-northeast-1
# → batch-notifications キューが表示される
```

フェーズA だけ通れば Step 2-6 に進める。フェーズB は Step 7 で必要になる。

---

## 次のステップ

Step 1が完了したら [Step 2: チャンク処理の基本](./step02.md) へ進む。
