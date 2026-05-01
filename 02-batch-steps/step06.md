# Step 6: 並列処理（パーティショニング）

## 目的

### これは何か

大量データをID範囲で分割（パーティショニング）し、複数パーティションを並列で処理する。Spring Batch の `Partitioner` + `TaskExecutor` に相当する。

### なぜやるのか

- Step 2-5 のチャンク処理はシングルスレッド。10万件を1000件ずつ処理すると100チャンク分の時間がかかる
- パーティショニングにより、例えば10パーティション × 並列度5 で処理すれば、理論上5倍速くなる
- 棚卸・在庫計算のような大量データ集計では、並列処理が実用上必須

### 何がうれしいのか

- 処理時間が大幅に短縮される。10万件のバッチが15分 → 3分になるイメージ
- パーティションごとに独立してリスタートできる（Step 4 の進捗テーブルに `partition_id` を追加済み）
- 並列度を設定で変更できるため、DBの負荷に応じて調整可能

## テスト用データの準備

```bash
# 10万件のテストデータを投入
docker compose exec db psql -U app -d sales_management \
  -c "INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq,
      division_code, department_code, section_code,
      process_category, inspection_category, manufacturing_category,
      status, manufacturing_completed_date)
      SELECT 2026, 'P', seq, 1, 1, 1, 1, 1, 1, 'manufactured', '2026-04-01'
      FROM generate_series(1, 100000) AS seq ON CONFLICT DO NOTHING;"
```

## 完了条件

### 並列処理の確認

1. 10万件を5パーティションで並列処理し、シングルスレッドより速く完了すること:

```bash
# 並列実行
dotnet run --project tools/BatchRunner -- --job=parallel-test --date=2026-04 --partitions=5

# ログでパーティション分割が確認できること
# {"message":"Partitioning","totalCount":100000,"partitions":5,"ranges":["1-20000","20001-40000","40001-60000","60001-80000","80001-100000"]}
# {"message":"Partition 1 completed","processed":20000,"elapsed":"3200ms"}
# {"message":"Partition 3 completed","processed":20000,"elapsed":"3300ms"}
# {"message":"Partition 2 completed","processed":20000,"elapsed":"3400ms"}
# ...（完了順序は不定）
# {"message":"All partitions completed","totalProcessed":100000,"elapsed":"3500ms"}
```

2. 処理時間がシングルスレッドより短いこと:

```bash
# シングルスレッド（比較用）
time dotnet run --project tools/BatchRunner -- --job=single-test --date=2026-04 --partitions=1
# → real 0m15.000s（例）

# 5パーティション並列
time dotnet run --project tools/BatchRunner -- --job=parallel-test --date=2026-04 --partitions=5
# → real 0m4.000s（例、大幅に短縮）
```

### パーティション別リスタートの確認

3. パーティション2が途中で失敗した場合、パーティション2だけ再実行されること:

```bash
# 実行（パーティション2で意図的にエラー）
dotnet run --project tools/BatchRunner -- --job=partition-restart-test --date=2026-04 --partitions=5
# → パーティション0,1,3,4は完了、パーティション2はFAILED

# 進捗確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT partition_id, last_processed_id, processed_count FROM batch_chunk_progress WHERE job_name = 'partition-restart-test';"
# → partition_id=2 のみレコードが残る（他は完了済みで削除）

# 再実行 → パーティション2だけ再開
dotnet run --project tools/BatchRunner -- --job=partition-restart-test --date=2026-04 --partitions=5
# → {"message":"Partition 0 already completed, skipping"}
# → {"message":"Partition 1 already completed, skipping"}
# → {"message":"Partition 2 restarting from last_processed_id=45000"}
# → ...
```

### 確認のコツ

- 最初は2パーティションで動作を確認し、その後5パーティションに増やす
- `batch_chunk_progress` テーブルの `partition_id` カラムでパーティション別の進捗を確認する
- 並列度を上げすぎるとDBのコネクションプールが枯渇する。コネクション数 > 並列度 であることを確認する

---

## 実装ガイド

### パーティション分割

```fsharp
// F#
let partition (db: NpgsqlDataSource) (tableName: string) (partitionCount: int) =
    let minMax = queryMinMaxId db tableName  // (minId, maxId)
    let rangeSize = (snd minMax - fst minMax + 1L) / int64 partitionCount
    [ for i in 0 .. partitionCount - 1 ->
        let from = fst minMax + int64 i * rangeSize
        let to' = if i = partitionCount - 1 then snd minMax else from + rangeSize - 1L
        (i, from, to') ]
```

```kotlin
// Kotlin
fun partition(db: Database, partitionCount: Int): List<Triple<Int, Long, Long>> {
    val (minId, maxId) = queryMinMaxId(db)
    val rangeSize = (maxId - minId + 1) / partitionCount
    return (0 until partitionCount).map { i ->
        val from = minId + i * rangeSize
        val to = if (i == partitionCount - 1) maxId else from + rangeSize - 1
        Triple(i, from, to)
    }
}
```

### 並列実行

```fsharp
// F#: Async.Parallel
partitions
|> List.map (fun (partId, from, to') ->
    async { processPartition db partId from to' })
|> Async.Parallel
|> Async.RunSynchronously
```

```kotlin
// Kotlin: coroutineScope + async
coroutineScope {
    partitions.map { (partId, from, to) ->
        async { processPartition(db, partId, from, to) }
    }.awaitAll()
}
```

---

## 次のステップ

Step 6が完了したら [Step 7: EventBridge + Step Functions スケジューリング](./step07.md) へ進む。
