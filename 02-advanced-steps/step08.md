# Step 8: 監査ログ + 分散トレーシング

## 目的

### これは何か

全てのデータ変更操作に「いつ・誰が・何をしたか」を自動記録する監査ログと、リクエストがシステム内をどう流れたかを可視化する分散トレーシングを導入する。トレースの可視化には Jaeger を docker-compose で起動する。

### なぜやるのか

- 業務システムでは「このデータを誰が変更したか」を追跡できることが必須。内部統制や監査対応で求められる
- 分散トレーシングは、1つのリクエストが「API → ドメインロジック → DB → 外部API」とどう流れたかを可視化する。障害調査で「どこで遅くなったか」を特定するのに不可欠

### 何がうれしいのか

- 「このロットの製造完了を指示したのは誰？いつ？」という問い合わせに、DBを1クエリ叩くだけで答えられる
- ブラウザで Jaeger を開くと、リクエストの流れが視覚的に確認できる。「このAPIが遅い原因はDBクエリ」「外部APIの呼び出しで3秒かかっている」が一目でわかる

## 事前準備: Jaeger を docker-compose に追加

### docker-compose.yml に追加

```yaml
  jaeger:
    image: jaegertracing/all-in-one:1.56
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
```

```bash
docker compose up -d jaeger

# Jaeger UI にアクセスできることを確認
# ブラウザで http://localhost:16686 を開く
```

## パート 1: 監査ログ

### やること

全てのテーブルに `created_at`, `created_by`, `updated_at`, `updated_by` カラムを追加し、データ変更時に自動的に記録する。`created_by` / `updated_by` にはJWTトークンから取得したユーザーIDが入る。

### マイグレーション

```sql
-- 新しいマイグレーションファイルを作成（V004 は Step 2 で使用済み）
-- F#: migrations/V005__add_audit_columns.sql
-- Kotlin: src/main/resources/db/migration/V005__add_audit_columns.sql

ALTER TABLE lot ADD COLUMN created_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE lot ADD COLUMN created_by TEXT NOT NULL DEFAULT 'system';
ALTER TABLE lot ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE lot ADD COLUMN updated_by TEXT NOT NULL DEFAULT 'system';

-- lot_detail, sales_case 等の他のテーブルにも同様に追加
```

### 「削除」はドメインの状態遷移で表現する

論理削除（`deleted_at` フラグ）は使わない。「削除」が必要なエンティティは、ドメインモデルに「取消済み」等の状態を追加して型で表現する。

```
-- DSL上の「削除」は状態遷移
behavior 販売案件を削除する = 査定前直接販売案件 -> 削除済み OR 削除エラー

-- DB上は status カラムの値で表現
UPDATE sales_case SET status = 'cancelled', updated_by = @userId WHERE ...
```

本当にレコードを物理削除する必要がある場合（個人情報の削除要求等）は、物理削除 + 監査ログテーブルへの記録で対応する。

### ユーザーIDの伝播

```
HTTPリクエスト
  → JWT認証ミドルウェア（Step 3）
    → JWTの "sub" クレームからユーザーIDを取得
      → ドメインロジック実行
        → DB保存時に created_by / updated_by にユーザーIDを設定
```

### 完了条件

1. ロットを作成すると、`created_at`, `created_by` が自動的に記録されること:

```bash
# operator ユーザーでロットを作成
TOKEN=$(./scripts/get-token.sh test-operator)
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8080/api/lots \
  -d '{"lotNumber":{"year":2024,"location":"A","seq":99}, ...}'

# DB確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT lot_number_seq, created_at, created_by, updated_at, updated_by FROM lot WHERE lot_number_seq = 99;"
# → created_by = "a1b2c3d4-..."（KeycloakのユーザーUUID）
```

2. ロットの状態を変更すると、`updated_at`, `updated_by` が更新されること（`created_at`, `created_by` は変わらない）:

```bash
# 別のユーザー（admin）で製造完了を指示
ADMIN_TOKEN=$(./scripts/get-token.sh test-admin)
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  http://localhost:8080/api/lots/2024-A-099/complete-manufacturing \
  -d '{"date":"2026-04-22"}'

# DB確認
docker compose exec db psql -U app -d sales_management \
  -c "SELECT created_by, updated_by FROM lot WHERE lot_number_seq = 99;"
# → created_by = "operator-uuid"（変わらない）
# → updated_by = "admin-uuid"（変わった）
```

---

## パート 2: 分散トレーシング

### やること

OpenTelemetry を導入し、HTTPリクエスト・DBクエリ・外部API呼び出しのトレースを Jaeger に送信する。

### F# での設定

```fsharp
// NuGet:
// OpenTelemetry.Extensions.Hosting
// OpenTelemetry.Instrumentation.AspNetCore
// OpenTelemetry.Instrumentation.Http
// OpenTelemetry.Exporter.OpenTelemetryProtocol
// Npgsql.OpenTelemetry

builder.Services.AddOpenTelemetry()
    .WithTracing(fun b ->
        b.SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("sales-management"))
         .AddAspNetCoreInstrumentation()   // HTTPリクエスト
         .AddHttpClientInstrumentation()   // 外部API呼び出し（Step 6）
         .AddNpgsql()                      // DBクエリ
         .AddOtlpExporter(fun opts ->
             opts.Endpoint <- Uri("http://localhost:4317")  // Jaeger OTLP
         ) |> ignore
    ) |> ignore
```

### Kotlin での設定

```kotlin
// Gradle:
// implementation("io.opentelemetry:opentelemetry-sdk")
// implementation("io.opentelemetry:opentelemetry-exporter-otlp")
// implementation("io.opentelemetry.instrumentation:opentelemetry-ktor-2.0")

// application.conf
ktor {
    opentelemetry {
        endpoint = "http://localhost:4317"
        serviceName = "sales-management"
    }
}
```

### 完了条件

3. アプリを起動し、APIリクエストを送った後、Jaeger UI でトレースが表示されること:

```bash
# APIリクエストを送る
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/lots/2024-A-001

# ブラウザで Jaeger UI を開く
# http://localhost:16686

# 左上の「Service」ドロップダウンで「sales-management」を選択
# 「Find Traces」をクリック
# → トレースが表示される
```

4. トレースをクリックすると、以下のスパン（処理の区間）が表示されること:

```
sales-management: GET /api/lots/2024-A-001  [15ms]
  ├── postgresql: SELECT * FROM lot WHERE ...  [3ms]
  └── postgresql: SELECT * FROM lot_detail WHERE ...  [2ms]
```

5. 外部API呼び出し（Step 6）を含むリクエストでは、外部APIのスパンも表示されること:

```
sales-management: GET /api/external/price-check  [120ms]
  ├── postgresql: SELECT * FROM lot WHERE ...  [3ms]
  └── HTTP GET http://wiremock:8080/api/pricing/...  [100ms]
```

6. ログにも `traceId` が含まれていること（Step 1の構造化ログと連携）:

```
{"traceId":"abc123def456","spanId":"789ghi","message":"GET /api/lots/2024-A-001"}
```

### 確認のコツ

- Jaeger UI の「Service」に `sales-management` が表示されない場合、OTLP Exporter の接続先（`http://localhost:4317`）が正しいか確認する
- トレースが表示されるまで数秒かかることがある。リクエスト送信後、少し待ってから「Find Traces」をクリックする
- Jaeger UI でトレースの各スパンをクリックすると、詳細情報（HTTPステータスコード、DBクエリ文等）が表示される

---

## 実装ガイド

### F#

| 要素 | 実装方法 |
|---|---|
| 監査カラム自動設定 | DB層の共通関数で `created_by`, `updated_at` 等を付与 |
| ユーザーID取得 | `HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)` |
| 分散トレーシング | `OpenTelemetry.Extensions.Hosting` + 各種 Instrumentation |
| トレースエクスポート | OTLP Exporter → Jaeger |

### Kotlin

| 要素 | 実装方法 |
|---|---|
| 監査カラム自動設定 | Exposed の `AuditedTable` 基底クラスで共通化 |
| ユーザーID取得 | `call.principal<JWTPrincipal>()?.payload?.subject` |
| 分散トレーシング | OpenTelemetry Java SDK + Ktor plugin |
| トレースエクスポート | OTLP Exporter → Jaeger |

---

## 次のステップ

Step 8が完了したら [Step 9: レート制限 + キャッシュ + グレースフルシャットダウン](./step09.md) へ進む。
