# Step 5: スキップ/リトライ + リスナー

## 目的

### これは何か

チャンク処理中に一部のアイテムでエラーが発生した場合に、リトライ（再試行）やスキップ（飛ばして続行）する仕組みと、ジョブ/チャンクのライフサイクルにフックを差し込むリスナーを実装する。

### なぜやるのか

- 1万件中1件だけ不正データがあるとき、バッチ全体を止めるのは過剰。「N件まではスキップして続行」という柔軟性が必要
- 外部API呼び出しを含むバッチでは、一時的なネットワークエラーでリトライすれば成功するケースが多い
- リスナーにより、チャンクの開始/完了/エラーをログに記録したり、メトリクスを収集したりできる

### 何がうれしいのか

- 「1万件中3件がスキップされ、9997件が正常処理された」という結果が `batch_job_execution` に記録される
- スキップされたアイテムはログに記録されるため、後から手動で対処できる
- リスナーにより、バッチの実行状況をリアルタイムで監視できる

## 完了条件

### スキップの確認

1. 1万件中5件に不正データを仕込み、スキップ上限を10件に設定してバッチを実行:

```bash
# 不正データを5件作成
docker compose exec db psql -U app -d sales_management \
  -c "UPDATE lot SET status = 'invalid' WHERE lot_number_year = 2026 AND lot_number_location = 'C' AND lot_number_seq IN (100, 500, 2000, 5000, 9000);"

# バッチ実行（maxSkips=10）
dotnet run --project tools/BatchRunner -- --job=skip-test --date=2026-04
```

2. バッチが正常完了し、スキップ件数が記録されること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status, read_count, write_count, skip_count FROM batch_job_execution WHERE job_name = 'skip-test';"
# → COMPLETED | 10000 | 9995 | 5
```

3. スキップされたアイテムがログに記録されていること:

```
{"level":"Warning","message":"Item skipped","lotId":"2026-C-100","reason":"Invalid status: invalid"}
{"level":"Warning","message":"Item skipped","lotId":"2026-C-500","reason":"Invalid status: invalid"}
...
```

4. スキップ上限を超えるとバッチが失敗すること:

```bash
# 不正データを15件に増やし、maxSkips=10 で実行
# → "Skip limit exceeded: 11" で FAILED
```

### リトライの確認

5. 一時的なエラーが発生した場合、リトライにより成功すること。デッドロックの再現は困難なため、特定のアイテムで初回だけ例外をスローするテスト用ロジック（例: `if item.id == 3000 && attempt == 1 then throw`）を仕込んで確認する:

```
{"level":"Warning","message":"Retry attempt 1","item":"2026-C-3000","error":"Transient error (test)"}
{"level":"Information","message":"Retry succeeded","item":"2026-C-3000","attempt":2}
```

### リスナーの確認

6. ジョブの開始/完了、チャンクの開始/完了がログに記録されること:

```
{"level":"Information","message":"Job started","job":"skip-test","params":"2026-04"}
{"level":"Information","message":"Chunk 1 started"}
{"level":"Information","message":"Chunk 1 completed","processed":998,"skipped":2,"elapsed":"130ms"}
...
{"level":"Information","message":"Job completed","job":"skip-test","totalProcessed":9995,"totalSkipped":5}
```

---

## 実装ガイド

### スキップ/リトライの構造

```fsharp
// F#
type ChunkConfig = {
    MaxSkips: int
    MaxRetries: int
    IsRetryable: exn -> bool   // デッドロック等 → true
    IsSkippable: exn -> bool   // バリデーションエラー等 → true
}
```

```kotlin
// Kotlin
data class ChunkConfig(
    val maxSkips: Int = 10,
    val maxRetries: Int = 3,
    val isRetryable: (Throwable) -> Boolean = { it is PSQLException && it.message?.contains("deadlock") == true },
    val isSkippable: (Throwable) -> Boolean = { it is ValidationException },
)
```

### リスナーの構造

```fsharp
// F#: レコード型で定義、デフォルト値付き
type BatchListeners<'a> = {
    OnJobStart: string -> unit
    OnJobEnd: string -> unit
    OnChunkStart: int -> unit
    OnChunkEnd: int -> int -> int -> unit  // chunkIndex -> processed -> skipped
    OnChunkError: int -> exn -> unit
    OnItemSkipped: 'a -> exn -> unit
}
```

```kotlin
// Kotlin: data class で定義、デフォルト値付き
data class BatchListeners<T>(
    val onJobStart: (String) -> Unit = {},
    val onJobEnd: (String) -> Unit = {},
    val onChunkStart: (Int) -> Unit = {},
    val onChunkEnd: (Int, Int, Int) -> Unit = { _, _, _ -> },
    val onChunkError: (Int, Throwable) -> Unit = { _, _ -> },
    val onItemSkipped: (T, Throwable) -> Unit = { _, _ -> },
)
```

### processInChunks への組み込み

`processInChunks` の引数に `ChunkConfig` と `BatchListeners` を追加し、チャンクループ内でリスナーを呼び出す。

---

## 次のステップ

Step 5が完了したら [Step 6: 並列処理（パーティショニング）](./step06.md) へ進む。
