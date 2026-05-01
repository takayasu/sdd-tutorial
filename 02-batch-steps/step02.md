# Step 2: チャンク処理の基本

## 目的

### これは何か

大量データを一定件数（チャンク）ずつ読み取り→加工→書き込みする `processInChunks` 関数を実装する。「月次締め処理」を題材に、在庫ロットの一括状態遷移をバッチで実行する。

### なぜやるのか

- 10万件のロットを一度にメモリに載せると OutOfMemory になる。チャンク単位で処理すればメモリ使用量を一定に保てる
- チャンク単位でコミットすることで、途中で失敗しても処理済みのデータは確定される
- Spring Batch の `Reader → Processor → Writer` パイプラインに相当する仕組みを、関数3つの合成で実現する

### 何がうれしいのか

- 「10万件のロットを1000件ずつ処理して、全件の状態を更新する」バッチが動く
- 処理中にログで進捗が確認できる（「チャンク 5/100 完了」等）
- Reader / Processor / Writer を差し替えるだけで、別のバッチ処理にも使い回せる

## テスト用データの準備

バッチ処理を試すために、大量のテストデータを投入する:

```sql
-- 1万件のテストロットを作成（製造完了状態）
INSERT INTO lot (lot_number_year, lot_number_location, lot_number_seq,
                 division_code, department_code, section_code,
                 process_category, inspection_category, manufacturing_category,
                 status, manufacturing_completed_date)
SELECT 2026, 'A', seq,
       1, 1, 1, 1, 1, 1,
       'manufactured', '2026-04-01'
FROM generate_series(1, 10000) AS seq
ON CONFLICT DO NOTHING;
```

## 完了条件

1. バッチを実行すると、1万件のロットが1000件ずつチャンク処理されること:

```bash
# F#
dotnet run --project tools/BatchRunner -- --job=monthly-close --date=2026-04

# Kotlin
gradle run --args="--job=monthly-close --date=2026-04"
```

2. ログにチャンクごとの進捗が出力されること:

```
{"message":"Chunk 1/10 completed","processed":1000,"elapsed":"120ms"}
{"message":"Chunk 2/10 completed","processed":2000,"elapsed":"115ms"}
...
{"message":"Chunk 10/10 completed","processed":10000,"elapsed":"110ms"}
{"message":"Job completed","job":"monthly-close","totalProcessed":10000}
```

3. 全ロットの状態が更新されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT status, count(*) FROM lot WHERE lot_number_year = 2026 GROUP BY status;"
# → shipping_instructed | 10000（または期待する遷移先の状態）
```

4. メモリ使用量がデータ件数に比例して増加しないこと（チャンクサイズ分だけ使用）

### 確認のコツ

- 最初はチャンクサイズを小さく（100件）して動作を確認し、その後1000件に増やす
- 処理時間を計測して、チャンクサイズによる性能差を体感する
- `processInChunks` 関数は汎用的に作る。月次締め固有のロジックは Reader / Processor / Writer に閉じ込める

---

## 実装ガイド

### 構造

```
BatchRunner（エントリポイント）
  └── processInChunks(reader, processor, writer, chunkSize)
        ├── reader: DB から製造完了ロットを読み取る（WHERE id > lastId ORDER BY id LIMIT chunkSize）
        ├── processor: ドメインロジック適用（状態遷移）
        └── writer: DB に一括更新
```

### F#

```fsharp
// 汎用チャンク処理関数
let processInChunks
    (db: NpgsqlDataSource) (chunkSize: int)
    (reader: int64 -> 'a seq)
    (processor: 'a -> Result<'b, 'err>)
    (writer: NpgsqlConnection -> 'b list -> unit)
    (getId: 'a -> int64) =

    let mutable lastId = 0L
    let mutable chunkIndex = 0
    let mutable hasMore = true

    while hasMore do
        let chunk = reader lastId |> Seq.truncate chunkSize |> Seq.toArray
        if Array.isEmpty chunk then
            hasMore <- false
        else
            chunkIndex <- chunkIndex + 1
            use conn = db.OpenConnection()
            use tx = conn.BeginTransaction()
            let results = chunk |> Array.choose (fun x -> processor x |> Result.toOption) |> Array.toList
            writer conn results
            lastId <- getId (Array.last chunk)
            tx.Commit()
            printfn $"Chunk {chunkIndex} completed: {results.Length} items"

// 月次締めバッチ
let monthlyClose (db: NpgsqlDataSource) (date: string) =
    processInChunks db 1000
        (fun lastId -> queryManufacturedLots db lastId 1000)
        (fun lot -> instructShipping lot (DateOnly.Parse("2026-04-30")))
        (fun conn lots -> bulkUpdateLots conn lots)
        (fun lot -> lot.Id)
```

### Kotlin

```kotlin
fun <T, R> processInChunks(
    db: Database, chunkSize: Int,
    reader: (lastId: Long) -> List<T>,
    processor: (T) -> Either<Any, R>,
    writer: (List<R>) -> Unit,
    getId: (T) -> Long,
) {
    var lastId = 0L
    var chunkIndex = 0

    while (true) {
        val chunk = reader(lastId)
        if (chunk.isEmpty()) break

        chunkIndex++
        transaction(db) {
            val results = chunk.mapNotNull { processor(it).getOrNull() }
            writer(results)
            lastId = getId(chunk.last())
        }
        println("Chunk $chunkIndex completed: ${chunk.size} items")
    }
}
```

### バッチ用エントリポイント

API サーバーとは別に、コマンドライン引数でジョブを指定して実行するエントリポイントを作成する:

```
# F#: tools/BatchRunner/Program.fs
# Kotlin: src/main/kotlin/salesmanagement/batch/BatchMain.kt
```

---

## 02-advanced-steps を併用する場合の注意

02-advanced-steps と 02-batch-steps の両方を実施する場合、以下の点に注意する:

- **楽観的ロック（02-advanced-steps Step 2）**: バッチは大量のレコードを一括更新するため、API 経由の更新と version が競合する可能性がある。バッチ処理では楽観的ロックをスキップするか、バッチ専用の更新関数（version チェックなし）を用意する
- **Outbox（02-advanced-steps Step 7）**: バッチで1万件の状態遷移を行うと、1万件の outbox_events が INSERT される。バッチ処理ではイベント発行を抑制するか、バッチ完了後にサマリイベント1件だけ発行する設計を検討する
- **監査ログ（02-advanced-steps Step 8）**: バッチ実行時の `updated_by` には `batch:<job_name>` 等のシステムユーザー識別子を設定する（JWT からのユーザー ID は存在しない）

---

## 次のステップ

Step 2が完了したら [Step 3: ジョブ実行管理 + 二重実行防止](./step03.md) へ進む。
