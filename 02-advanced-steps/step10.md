# Step 10: CSV/ファイルエクスポート

## 目的

### これは何か

ロットや販売案件のデータを CSV ファイルとしてダウンロードするエンドポイントを実装する。日本の業務システムでは Windows-31J（Shift_JIS）エンコーディングの CSV エクスポートが日常的に使われる。

### なぜやるのか

- 経理部門への報告、外部システムへのデータ連携、監査対応など、CSV エクスポートは業務システムの基本機能
- Spring Boot サンプルでは `CsvView`（Jackson CsvMapper + Windows-31J）、`ExcelView`（Apache POI）、`PdfView`（JasperReports）を実装していた
- 大量データのエクスポートではメモリに全件載せずにストリーミングで出力する必要がある

### 何がうれしいのか

- ブラウザや curl で `/lots/export?format=csv` を叩くと、CSV ファイルがダウンロードされる
- 日本語が文字化けしない（Windows-31J エンコーディング対応）
- 10万件のデータでもメモリを圧迫しない（ストリーミング出力）

## 完了条件

1. CSV エクスポートエンドポイントが動作すること:

```bash
TOKEN=$(./scripts/get-token.sh test-operator)

curl -H "Authorization: Bearer $TOKEN" \
  -o lots.csv \
  "http://localhost:8080/lots/export?format=csv"

# ファイルが生成されること
head -3 lots.csv
# → "ロット番号","事業部","状態","製造完了日"
# → "2024-A-001","1","manufactured","2024-04-01"
# → "2024-A-002","1","manufacturing",""
```

2. レスポンスヘッダが正しいこと:

```bash
curl -v -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/lots/export?format=csv" 2>&1 | grep -i content
# → Content-Type: text/csv; charset=windows-31j
# → Content-Disposition: attachment; filename="lots_20260422.csv"
```

3. 日本語が文字化けしないこと（Excel で開いて確認、または `nkf` コマンドで確認）:

```bash
nkf --guess lots.csv
# → Shift_JIS (or CP932/Windows-31J)
```

4. 大量データ（1万件以上）でもタイムアウトせずにダウンロードできること

5. 検索条件を指定してエクスポートできること:

```bash
curl -H "Authorization: Bearer $TOKEN" \
  -o manufactured_lots.csv \
  "http://localhost:8080/lots/export?format=csv&status=manufactured"
# → 製造完了ロットのみが出力される
```

---

## 実装ガイド

### F#

| 要素 | 実装方法 |
|---|---|
| CSV 生成 | `CsvHelper`（NuGet）— ストリーミング書き込み対応 |
| エンコーディング | `System.Text.Encoding.RegisterProvider(CodePagesEncodingProvider.Instance)` で Windows-31J を有効化 |
| ストリーミング | `ctx.Response.Body` に直接書き込み |

```fsharp
// NuGet: CsvHelper, System.Text.Encoding.CodePages

let exportLotsHandler: HttpHandler = fun next ctx -> task {
    let lots = queryLots db (parseFilters ctx)
    ctx.Response.ContentType <- "text/csv; charset=windows-31j"
    ctx.Response.Headers.ContentDisposition <- $"attachment; filename=\"lots_{DateTime.Now:yyyyMMdd}.csv\""
    let encoding = Encoding.GetEncoding("windows-31j")
    use writer = new StreamWriter(ctx.Response.Body, encoding)
    use csv = new CsvWriter(writer, CultureInfo.InvariantCulture)
    for lot in lots do
        csv.WriteRecord(lot)
        csv.NextRecord()
    return! next ctx
}
```

### Kotlin

| 要素 | 実装方法 |
|---|---|
| CSV 生成 | Jackson CSV (`jackson-dataformat-csv`) or `kotlinx-io` |
| エンコーディング | `Charset.forName("windows-31j")` |
| ストリーミング | Ktor `respondOutputStream` |

```kotlin
// Gradle: implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-csv")

get("/lots/export") {
    val lots = queryLots(db, call.parameters)
    call.response.header(HttpHeaders.ContentDisposition, "attachment; filename=\"lots_${LocalDate.now()}.csv\"")
    call.respondOutputStream(ContentType.Text.CSV.withCharset(Charset.forName("windows-31j"))) {
        val writer = OutputStreamWriter(this, Charset.forName("windows-31j"))
        val csvMapper = CsvMapper().apply { registerModule(KotlinModule.Builder().build()) }
        val schema = csvMapper.schemaFor(LotCsvRow::class.java).withHeader()
        csvMapper.writer(schema).writeValues(writer).use { seq ->
            lots.forEach { seq.write(it) }
        }
    }
}
```

### 確認のコツ

- Windows-31J で表現できない文字（一部の Unicode 文字）がデータに含まれる場合のエラーハンドリングも考慮する
- UTF-8 BOM 付き CSV が必要な場合は、エンコーディングを `utf-8` にして先頭に BOM (`\xEF\xBB\xBF`) を書き込む
- Excel 向けには CSV よりも TSV（タブ区切り）の方が文字化けしにくい場合がある

---

## 🎉 02-advanced-steps 完了

Step 10 が完了すると、01-steps/ で構築したドメインモデル + CI 基盤に加え、業務システムとして本番運用に必要な全機能が揃います:

| カテゴリ | 機能 | Step |
|---|---|---|
| 設定・ログ | 環境別設定、構造化JSONログ、リクエストID | Step 1 |
| API品質 | RFC 9457 Problem Details、Smart Constructor、楽観的ロック | Step 2 |
| セキュリティ | JWT認証（Keycloak）、RBAC | Step 3 |
| 運用 | ヘルスチェック、OpenAPI / Swagger UI | Step 4 |
| テスト | 統合テスト + TestContainers | Step 5 |
| 外部連携 | HTTPクライアント、リトライ、サーキットブレーカー | Step 6 |
| イベント駆動 | ドメインイベント、Outboxパターン | Step 7 |
| 監査・追跡 | 監査ログ、分散トレーシング（Jaeger） | Step 8 |
| 本番運用 | レート制限、キャッシュ、グレースフルシャットダウン | Step 9 |
| データ出力 | CSVエクスポート（Windows-31J対応） | Step 10 |

次のステップとして:
- [02-batch-steps/](../02-batch-steps/) でバッチ処理基盤を構築する
- `ci.sh` を更新し、統合テストをCIに組み込む
- 本番デプロイに向けて、Dockerfile と k8s マニフェストを整備する