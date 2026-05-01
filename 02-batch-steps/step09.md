# Step 9: CSV インポートバッチ

## 目的

### これは何か

CSV ファイルからデータを読み取り、行ごとにバリデーションしてDBに一括登録するバッチジョブを実装する。Step 2-6 で構築した `processInChunks` インフラを、ファイルベースの Reader で使う。

### なぜやるのか

- Step 2-8 のバッチは全て DB→DB 処理だった。業務システムでは「外部システムから受け取った CSV をインポートする」パターンが非常に多い
- Spring Batch の `FlatFileItemReader` に相当する機能を、言語標準のファイル読み取り + `processInChunks` で実現する
- 行ごとのバリデーションエラーを蓄積し、「100行中3行がエラー、97行が正常登録」という結果を返す

### 何がうれしいのか

- `processInChunks` の Reader を差し替えるだけで、DB→DB バッチと同じインフラ（リスタート、スキップ、リスナー）がファイルバッチにも使える
- バリデーションエラーの行番号と内容がログに記録されるため、データ提供元にフィードバックできる
- 大量の CSV（数万行）でもチャンク単位でコミットするため、途中で失敗しても処理済み分は確定される

## テスト用 CSV ファイルの準備

```bash
mkdir -p data
```

```csv
# data/import_lots.csv
ロット番号年度,ロット番号保管場所,ロット番号連番,事業部コード,部門コード,担当課コード,工程区分,検査区分,製造区分
2026,C,1,1,1,1,1,1,1
2026,C,2,1,1,1,1,1,1
2026,C,3,1,1,1,1,1,1
2026,C,invalid,1,1,1,1,1,1
2026,C,5,1,1,1,1,1,1
```

4行目の `invalid` はバリデーションエラーを発生させるためのテストデータ。

## 完了条件

### フェーズA: アプリ単体

1. CSV インポートバッチを実行し、正常行がDBに登録されること:

```bash
dotnet run --project tools/BatchRunner -- --job=import-lots --file=data/import_lots.csv
# or
gradle run --args="--job=import-lots --file=data/import_lots.csv"
```

2. `batch_job_execution` に結果が記録されること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT job_name, status, read_count, write_count, skip_count FROM batch_job_execution WHERE job_name = 'import-lots';"
# → import-lots | COMPLETED | 5 | 4 | 1
```

3. 正常行がDBに登録されていること:

```bash
docker compose exec db psql -U app -d sales_management \
  -c "SELECT lot_number_seq, status FROM lot WHERE lot_number_year = 2026 AND lot_number_location = 'C' ORDER BY lot_number_seq;"
# → 1, 2, 3, 5 が manufacturing 状態で登録（4行目の invalid はスキップ）
```

4. スキップされた行がログに記録されていること:

```
{"level":"Warning","message":"Row skipped","line":4,"reason":"lot_number_seq must be a positive integer","raw":"2026,C,invalid,1,1,1,1,1,1"}
```

5. エンコーディングが Windows-31J の CSV でも正しく読み取れること:

```bash
# Windows-31J の CSV を作成
nkf -s data/import_lots.csv > data/import_lots_sjis.csv

dotnet run --project tools/BatchRunner -- --job=import-lots --file=data/import_lots_sjis.csv --encoding=windows-31j
# → 正常に処理される
```

6. 大量データ（1万行）の CSV でもチャンク処理されること:

```bash
# 1万行の CSV を生成
python3 -c "
print('ロット番号年度,ロット番号保管場所,ロット番号連番,事業部コード,部門コード,担当課コード,工程区分,検査区分,製造区分')
for i in range(1, 10001):
    print(f'2026,D,{i},1,1,1,1,1,1')
" > data/import_lots_large.csv

dotnet run --project tools/BatchRunner -- --job=import-lots-large --file=data/import_lots_large.csv
# → チャンクごとにログが出力される
# → batch_job_execution.write_count = 10000
```

---

## 実装ガイド

### 構造

```
processInChunks(
  reader:    CSVファイルから行を読み取り、パース済みオブジェクトの Seq/Sequence を返す
  processor: 行ごとのバリデーション（Smart Constructor）
  writer:    DB に一括 INSERT
)
```

### F#

| 要素 | 実装方法 |
|---|---|
| CSV 読み取り | `CsvHelper`（NuGet）の `CsvReader` — ストリーミング読み取り |
| エンコーディング | `Encoding.GetEncoding("windows-31j")` |
| バリデーション | Smart Constructor（Step 2 で作成済み） |
| DB 書き込み | Donald `Db.exec` + バッチ INSERT |

```fsharp
// Reader: CSV → LotImportRow seq
let csvReader (filePath: string) (encoding: Encoding) (lastId: int64) : LotImportRow seq =
    seq {
        use reader = new StreamReader(filePath, encoding)
        use csv = new CsvReader(reader, CultureInfo.InvariantCulture)
        let mutable lineNo = 0L
        while csv.Read() do
            lineNo <- lineNo + 1L
            if lineNo > lastId then
                yield csv.GetRecord<LotImportRow>() |> withLineNumber lineNo
    }
```

### Kotlin

| 要素 | 実装方法 |
|---|---|
| CSV 読み取り | Jackson CSV (`jackson-dataformat-csv`) |
| エンコーディング | `Charset.forName("windows-31j")` |
| バリデーション | Arrow `Either` + Smart Constructor |
| DB 書き込み | Exposed `batchInsert` |

```kotlin
// Reader: CSV → Sequence<LotImportRow>
fun csvReader(filePath: String, charset: Charset, lastId: Long): Sequence<LotImportRow> {
    val csvMapper = CsvMapper().apply { registerModule(KotlinModule.Builder().build()) }
    val schema = csvMapper.schemaFor(LotImportRow::class.java).withHeader()
    return csvMapper.readerFor(LotImportRow::class.java)
        .with(schema)
        .readValues<LotImportRow>(File(filePath).reader(charset))
        .asSequence()
        .withIndex()
        .filter { it.index + 1 > lastId }
        .map { it.value.copy(lineNumber = it.index + 1L) }
}
```

### リスタートとの組み合わせ

CSV インポートのリスタートは「行番号」をオフセットとして使う。`batch_chunk_progress.last_processed_id` に最後に処理した行番号を記録し、再実行時はその行番号以降から読み取る。

---

## 🎉 バッチ処理基盤の完成

Step 9 が完了すると、DB→DB バッチ（Step 2-8）に加えて、ファイル→DB バッチも動作する状態になります。`processInChunks` の Reader を差し替えるだけで、リスタート・スキップ・リスナー・並列処理の全てが使い回せることが確認できました。
