# Step 3: ジョブ実行管理 + 二重実行防止

## 目的

### これは何か

バッチジョブの開始時にDBに実行レコードを作成し、完了/失敗時にステータスと処理件数を記録する。同一パラメータのジョブが同時に実行されることを防止する。

### なぜやるのか

- Step 2 のチャンク処理は「実行して終わり」。いつ実行されたか、何件処理したか、成功したかの記録がない
- 月次締めバッチを誤って2回実行すると、データが二重に処理される可能性がある
- Spring Batch の `JobRepository`（メタデータ5テーブル）に相当する機能を、`batch_job_execution` テーブル1つで実現する

### 何がうれしいのか

- `SELECT * FROM batch_job_execution` で全ジョブの実行履歴が一覧できる。「先月の月次締めはいつ実行されて、何件処理したか」がすぐわかる
- 同じバッチを2回実行しようとすると「Already running」「Already completed」で拒否される。誤操作による二重処理を防げる
- 失敗したジョブは `FAILED` ステータスで記録され、次のステップ（Step 4）でリスタートの起点になる

## 完了条件

### ジョブ実行記録の確認

1. バッチを実行すると、`batch_job_execution` にレコードが作成されること:

```bash
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-04
# or
gradle run --args="--job=monthly-close --date=2026-04"

docker compose exec db psql -U app -d sales_management \
  -c "SELECT job_name, job_params, status, read_count, write_count, started_at, completed_at FROM batch_job_execution;"
# → monthly-close | 2026-04 | COMPLETED | 10000 | 10000 | 2026-04-22 10:00:00 | 2026-04-22 10:00:15
```

### 二重実行防止の確認

2. 同一パラメータで再実行すると拒否されること:

```bash
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-04
# → "Job 'monthly-close' with params '2026-04' already completed"
# 終了コード: 0（エラーではない）
```

3. 異なるパラメータなら実行できること:

```bash
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-05
# → 正常に実行される（別のジョブインスタンス）
```

### 同時実行防止の確認

4. 2つのターミナルから同時に実行すると、片方が拒否されること:

```bash
# ターミナル1（先に開始）
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-06

# ターミナル2（ターミナル1の実行中に）
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-06
# → "Job 'monthly-close' with params '2026-06' is already running"
```

### 失敗時の記録確認

5. バッチ処理中にエラーが発生すると、`FAILED` ステータスで記録されること:

```bash
# 意図的にエラーを発生させる（存在しないテーブルを参照する等）
# → batch_job_execution.status = 'FAILED', error_message = '...'
```

### 確認のコツ

- `batch_job_execution` テーブルを頻繁に SELECT して状態遷移を観察する
- 二重実行防止は DB の PK 制約 + ステータスチェックで実現する。アプリレベルのロックではなく DB レベルなので、複数プロセスからの同時実行にも対応できる

---

## 実装ガイド

### tryStart 関数

```
起動時の判定ロジック:
  1. batch_job_execution に SELECT
  2. 既に RUNNING → "Already running" で拒否
  3. 既に COMPLETED → "Already completed" で拒否
  4. 既に FAILED → ステータスを RUNNING に更新（リスタート、Step 4 で活用）
  5. レコードなし → INSERT して開始
```

4 の排他制御: 2プロセスが同時に FAILED ジョブをリスタートしようとするレースコンディションを防ぐため、SELECT → UPDATE ではなく、1文で処理する:

```sql
UPDATE batch_job_execution
SET status = 'RUNNING', started_at = NOW(), error_message = NULL
WHERE job_name = @name AND job_params = @params AND status = 'FAILED'
RETURNING *;
```

affected rows = 0 なら「別プロセスが先にリスタート済み」と判定し、拒否する。5 の INSERT も `ON CONFLICT DO NOTHING` + affected rows チェックで同様に対処する。

### runBatch 関数（Step 2 の processInChunks をラップ）

```
runBatch:
  1. tryStart() → 二重実行チェック
  2. processInChunks() → チャンク処理（Step 2）
  3. 正常完了 → completeJob()（status=COMPLETED, 件数記録）
  4. 異常終了 → failJob()（status=FAILED, error_message記録）
```

### F# / Kotlin 共通

- `tryStart`, `completeJob`, `failJob` の3関数を実装
- Step 2 の `processInChunks` を `runBatch` でラップ
- バッチエントリポイントで `runBatch` を呼び出し、終了コードを返す

---

## 次のステップ

Step 3が完了したら [Step 4: チャンクリスタート](./step04.md) へ進む。
