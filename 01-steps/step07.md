# Step 7: 在庫ロットの集約API完全パッケージ + DB永続化

## 目的

### これは何か

Step 6で定義した型を使って、在庫ロットの**集約APIを「完全パッケージ」として一気に実装する**。状態遷移の mutation だけでなく、後付けで漏れがちな以下を**最初から**含める:

| 要素 | 内容 |
|---|---|
| Mutation | 状態遷移 (create / complete-manufacturing / instruct-shipping / ...) |
| **詳細 GET** | `GET /lots/{id}` でロット 1 件取得 |
| **一覧 GET** | `GET /lots?status=...&limit=&offset=` でページング付き一覧取得 |
| **楽観ロック** | `version: int` カラム + `WHERE version = @expected` での更新。競合時 **409 Conflict** |
| **エラー形式** | 全レスポンスを `application/problem+json` (RFC 9457) で統一 |
| **OpenAPI 完全記述** | `components.schemas` に `LotResponse`, `LotSummary`, `LotsListResponse`, `CreateLotRequest`, `LotStatus` を全部定義し、各 path の `responses.content.schema` を `$ref` で埋める |
| **ci.sh verify** | Step 1 で導入した verify セクションに本ステップの curl を**追記**する |

これは Step 14 / 15 / 18 でも同じテンプレで繰り返す。**この時点で「型 + DB + API + ドキュメント + 検証」が揃った状態**を 1 step で確立し、以降の集約はテンプレ通りに作る。

### なぜやるのか

- DSLの `behavior`（振る舞い）をAPIエンドポイントとして実現する
- 「製造完了を指示する」「出荷を指示する」といった業務操作を、HTTPリクエストで呼び出せるようにする
- 状態をDBに永続化することで、サーバーを再起動してもデータが消えない
- **後付けは高コスト**: 詳細GET / 一覧GET / 楽観ロック / problem+json / openapi 完全記述 を後回しにすると、フロント連携時に必ず手戻りが発生する
- **CI の verify セクションに curl を残す**ことで、ralph が `[x]` をつけた後に実装が消えても次の `./ci.sh` で落ちる

### 何がうれしいのか

- 型定義のおかげで、不正な状態遷移（例：製造中ロットに出荷完了を指示する）がコンパイル時に防がれる
- APIとして公開することで、フロントエンドや他システムから呼び出せる
- 「DSL → 型 → API → DB」という一連の流れが体験できる。これがAI駆動開発の基本サイクル
- フロント側 (将来 React や別 SPA を生やす時) は `openapi-zod-client` 等で**手書きゼロ**で型が揃う

## 完了条件

以下の **(a) 動作要件** すべてが通ること、**かつ (b) ci.sh verify セクションへ追記** が完了していること。

### (a) 動作要件

```bash
# 1. ロット作成（製造中状態で作成される。version=1 が返る）
$ curl -sf -X POST http://localhost:5000/lots \
  -H "Content-Type: application/json" \
  -d '{"lotNumber": {"year": 2024, "location": "A", "seq": 1}, "divisionCode": 1, ...}'
{"status":"manufacturing","lotNumber":"2024-A-001","version":1}

# 2. 詳細 GET — 集約 ID から 1 件取得 (caseType ポリモーフィックの基礎)
$ curl -sf http://localhost:5000/lots/2024-A-001 | jq -e '.lotNumber and .status and .version' >/dev/null
$ echo $?
0

# 3. 一覧 GET — ページング (limit/offset) と status フィルタ
$ curl -sf "http://localhost:5000/lots?limit=20&offset=0" | jq -e '.items and .total and (.limit==20)' >/dev/null
$ curl -sf "http://localhost:5000/lots?status=manufacturing&limit=10" | jq -e '.items|all(.status=="manufacturing")' >/dev/null

# 4. 製造完了を指示（version を渡す。サーバは新 version を返す）
$ curl -sf -X POST http://localhost:5000/lots/2024-A-001/complete-manufacturing \
  -H "Content-Type: application/json" \
  -d '{"date": "2024-04-01", "version": 1}'
{"status":"manufactured","manufacturingCompletedDate":"2024-04-01","version":2}

# 5. 楽観ロック競合 — 古い version で更新すると 409 + problem+json
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/lots/2024-A-001/instruct-shipping \
    -H "Content-Type: application/json" \
    -d '{"deadlineDate":"2024-05-01","version":1}'
409 application/problem+json

# 6. 不正な遷移 — problem+json で 400 (`{ "error": ... }` ではない！)
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/lots/2024-A-001/complete-shipping \
    -H "Content-Type: application/json" \
    -d '{"date": "2024-04-10","version":2}'
400 application/problem+json

# 7. 存在しない ID — 404 + problem+json
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:5000/lots/9999-Z-999
404 application/problem+json

# 8. DB 永続化と version カラムの存在
$ docker compose exec db psql -U app -d sales_management \
  -c "SELECT lot_number_year, lot_number_location, lot_number_seq, status, version FROM lot"

# 9. OpenAPI 完全記述 — 必要なスキーマがすべて埋まっている
$ python3 -c "
import yaml
y = yaml.safe_load(open('openapi.yaml'))
required = ['LotResponse','LotSummary','LotsListResponse','CreateLotRequest','LotStatus']
missing = [s for s in required if s not in y['components']['schemas']]
assert not missing, f'missing schemas: {missing}'
# 各 path の 200 / 400 / 404 / 409 が \$ref を持つこと
for p, ops in y['paths'].items():
    for verb, op in ops.items():
        if verb not in ('get','post','put','delete','patch'): continue
        for code, resp in op.get('responses', {}).items():
            if code in ('400','404','409'):
                assert '\$ref' in str(resp), f'{p} {verb} {code}: not problem+json'
print('OK')
"
OK
```

### (b) `ci.sh` の verify セクションへ追記

Step 1 で `ci.sh` 末尾に作った verify ブロックに、上記 1-9 を `curl -sf` / `jq -e` の形で追加する。`./ci.sh` を実行して以下が出力されること:

```
=== verify (smoke) ===
PASS /health
PASS security-headers
PASS cors-preflight
PASS problem+json-on-404
PASS lot-create
PASS lot-detail-get
PASS lot-list-paged
PASS lot-list-status-filter
PASS lot-version-conflict-409
PASS lot-invalid-transition-400
PASS lot-not-found-404
PASS openapi-schemas-complete
=== CI完了 ===
$ echo $?
0
```

**重要**: ralph で本 step を `[x]` にする前に、必ず `./ci.sh` が緑になっていること。verify ブロックに curl が追加されていなければ false-positive completion とみなす。

---

## 実装するAPI

| メソッド | パス | 対応するbehavior / 役割 | version 必須 |
|---|---|---|---|
| POST | `/lots` | ロット作成（製造中状態で作成） | 不要 (新規) |
| **GET** | **`/lots`** | **一覧取得（`status` フィルタ、`limit`/`offset` ページング）** | 不要 |
| **GET** | **`/lots/{id}`** | **詳細取得（1 件）** | 不要 |
| POST | `/lots/{id}/complete-manufacturing` | 製造完了を指示する | **必須** |
| POST | `/lots/{id}/instruct-shipping` | 出荷を指示する | **必須** |
| POST | `/lots/{id}/complete-shipping` | 出荷完了を指示する | **必須** |
| POST | `/lots/{id}/cancel-manufacturing-completion` | 製造完了を取り消す | **必須** |

### 一覧 GET のレスポンス形 (Step 14 / 18 でも同じ規約を使う)

```json
{
  "items": [
    {"lotNumber":"2024-A-001","status":"manufactured","manufacturingCompletedDate":"2024-04-01","version":2}
  ],
  "total": 127,
  "limit": 20,
  "offset": 0
}
```

### 楽観ロック実装方針

- 全テーブルに `version INTEGER NOT NULL DEFAULT 1` を追加
- 状態遷移 mutation は `WHERE version = @expected_version` を含めた UPDATE
- 影響行 0 行 → 409 Conflict + problem+json (`type: "/errors/version-conflict"`)
- 成功時はレスポンス body の `version` をインクリメント後の値で返す

---

## F# 実装の構造

### ドメインロジック（純粋関数）

```fsharp
// Domain/LotWorkflows.fs
module SalesManagement.Domain.LotWorkflows

type ManufacturingCompletionError = | LotNotInManufacturing
type ShippingInstructionError = | LotNotManufactured
type ShippingCompletionError = | LotNotShippingInstructed
type CancellationError = | LotNotManufactured

let completeManufacturing (date: DateOnly) (lot: ManufacturingLot) : ManufacturedLot =
    { Common = lot.Common
      ManufacturingCompletedDate = date }

let instructShipping (deadline: DateOnly) (lot: ManufacturedLot) : ShippingInstructedLot =
    { Common = lot.Common
      ManufacturingCompletedDate = lot.ManufacturingCompletedDate
      ShippingDeadlineDate = deadline }

let completeShipping (date: DateOnly) (lot: ShippingInstructedLot) : ShippedLot =
    { Common = lot.Common
      ManufacturingCompletedDate = lot.ManufacturingCompletedDate
      ShippingDeadlineDate = lot.ShippingDeadlineDate
      ShippedDate = date }

let cancelManufacturingCompletion (lot: ManufacturedLot) : ManufacturingLot =
    { Common = lot.Common }
```

### DB永続化（Donald）

```fsharp
// Infrastructure/LotRepository.fs
module SalesManagement.Infrastructure.LotRepository

open Donald
open Npgsql

let save (conn: NpgsqlConnection) (lot: InventoryLot) =
    let (lotNumber, status, mfgDate, shipDeadline, shippedDate) =
        match lot with
        | Manufacturing l -> (l.Common.LotNumber, "manufacturing", None, None, None)
        | Manufactured l -> (l.Common.LotNumber, "manufactured", Some l.ManufacturingCompletedDate, None, None)
        | ShippingInstructed l -> (l.Common.LotNumber, "shipping_instructed", Some l.ManufacturingCompletedDate, Some l.ShippingDeadlineDate, None)
        | Shipped l -> (l.Common.LotNumber, "shipped", Some l.ManufacturingCompletedDate, Some l.ShippingDeadlineDate, Some l.ShippedDate)

    conn
    |> Db.newCommand """
        UPDATE lot SET status = @status,
            manufacturing_completed_date = @mfg_date,
            shipping_deadline_date = @ship_deadline,
            shipped_date = @shipped_date
        WHERE lot_number_year = @year
          AND lot_number_location = @location
          AND lot_number_seq = @seq"""
    |> Db.setParams [
        "status", SqlType.String status
        "mfg_date", SqlType.AnsiString (mfgDate |> Option.map string |> Option.defaultValue null)
        "ship_deadline", SqlType.AnsiString (shipDeadline |> Option.map string |> Option.defaultValue null)
        "shipped_date", SqlType.AnsiString (shippedDate |> Option.map string |> Option.defaultValue null)
        "year", SqlType.Int32 lotNumber.Year
        "location", SqlType.String lotNumber.Location
        "seq", SqlType.Int32 lotNumber.Seq ]
    |> Db.exec
```

### APIルーティング（Giraffe）

```fsharp
// Api/LotRoutes.fs
let lotRoutes : HttpHandler =
    choose [
        POST >=> routef "/lots/%s/complete-manufacturing" (fun id -> completeManufacturingHandler id)
        POST >=> routef "/lots/%s/instruct-shipping" (fun id -> instructShippingHandler id)
        POST >=> routef "/lots/%s/complete-shipping" (fun id -> completeShippingHandler id)
        POST >=> routef "/lots/%s/cancel-manufacturing-completion" (fun id -> cancelHandler id)
        GET  >=> routef "/lots/%s" (fun id -> getLotHandler id)
    ]
```

---

## Kotlin 実装の構造

### 依存関係追加（build.gradle.kts）

```kotlin
dependencies {
    // 既存に追加
    implementation("io.arrow-kt:arrow-core:1.2.4")
}
```

### ドメインロジック（純粋関数）

```kotlin
// domain/LotWorkflows.kt
package salesmanagement.domain

import arrow.core.Either
import arrow.core.left
import arrow.core.right
import java.time.LocalDate

sealed interface LotError {
    data object LotNotInManufacturing : LotError
    data object LotNotManufactured : LotError
    data object LotNotShippingInstructed : LotError
}

fun completeManufacturing(
    lot: InventoryLot.Manufacturing,
    date: LocalDate
): Either<LotError, InventoryLot.Manufactured> =
    InventoryLot.Manufactured(
        common = lot.common,
        manufacturingCompletedDate = date
    ).right()

fun instructShipping(
    lot: InventoryLot.Manufactured,
    deadline: LocalDate
): Either<LotError, InventoryLot.ShippingInstructed> =
    InventoryLot.ShippingInstructed(
        common = lot.common,
        manufacturingCompletedDate = lot.manufacturingCompletedDate,
        shippingDeadlineDate = deadline
    ).right()

fun completeShipping(
    lot: InventoryLot.ShippingInstructed,
    date: LocalDate
): Either<LotError, InventoryLot.Shipped> =
    InventoryLot.Shipped(
        common = lot.common,
        manufacturingCompletedDate = lot.manufacturingCompletedDate,
        shippingDeadlineDate = lot.shippingDeadlineDate,
        shippedDate = date
    ).right()

fun cancelManufacturingCompletion(
    lot: InventoryLot.Manufactured
): Either<LotError, InventoryLot.Manufacturing> =
    InventoryLot.Manufacturing(common = lot.common).right()
```

### DB永続化（Exposed）

```kotlin
// infrastructure/LotTable.kt
package salesmanagement.infrastructure

import org.jetbrains.exposed.sql.*

object LotTable : Table("lot") {
    val lotNumberYear = integer("lot_number_year")
    val lotNumberLocation = text("lot_number_location")
    val lotNumberSeq = integer("lot_number_seq")
    val status = text("status")
    val manufacturingCompletedDate = date("manufacturing_completed_date").nullable()
    val shippingDeadlineDate = date("shipping_deadline_date").nullable()
    val shippedDate = date("shipped_date").nullable()
    // ... 他のカラム

    override val primaryKey = PrimaryKey(lotNumberYear, lotNumberLocation, lotNumberSeq)
}
```

### APIルーティング（Ktor）

```kotlin
// api/LotRoutes.kt
fun Route.lotRoutes() {
    route("/lots") {
        post("/{id}/complete-manufacturing") { /* handler */ }
        post("/{id}/instruct-shipping") { /* handler */ }
        post("/{id}/complete-shipping") { /* handler */ }
        post("/{id}/cancel-manufacturing-completion") { /* handler */ }
        get("/{id}") { /* handler */ }
    }
}
```

---

## DB設計方針

### シングルテーブル + statusカラム方式を採用する理由

在庫ロットの各状態（製造中・製造完了・出荷指示済み・出荷完了）は、共通フィールド（ロット共通 + ロット明細）が同一で、差分は日付カラム数個のみ。この構造ではテーブル分割のメリットが薄い：

- 分割した場合、状態遷移のたびにINSERT + DELETEが必要になり複雑化する
- JOINが増えるだけで、カラムの重複削減効果がほぼない
- 状態ごとのテーブルに分けると、「ロット一覧取得」でUNION ALLが必要になる

### 型安全性の担保箇所

| 層 | 担保方法 |
|---|---|
| アプリケーション層 | DSLから生成した型（DU / sealed class）で不正な状態遷移をコンパイル時に防ぐ |
| DB層 | シングルテーブル。statusカラム + nullable日付カラムで永続化。整合性はアプリ層が保証 |

### テーブル設計

```sql
CREATE TABLE lot (
    lot_number_year       INTEGER NOT NULL,
    lot_number_location   TEXT NOT NULL,
    lot_number_seq        INTEGER NOT NULL,
    division_code         INTEGER NOT NULL,
    department_code       INTEGER NOT NULL,
    section_code          INTEGER NOT NULL,
    process_category      INTEGER NOT NULL,
    inspection_category   INTEGER NOT NULL,
    manufacturing_category INTEGER NOT NULL,
    status                TEXT NOT NULL,  -- 'manufacturing' | 'manufactured' | 'shipping_instructed' | 'shipped'
    manufacturing_completed_date DATE,
    shipping_deadline_date       DATE,
    shipped_date                 DATE,
    version               INTEGER NOT NULL DEFAULT 1,  -- 楽観ロック (Step 7 から導入)
    PRIMARY KEY (lot_number_year, lot_number_location, lot_number_seq)
);

CREATE TABLE lot_detail (
    lot_number_year       INTEGER NOT NULL,
    lot_number_location   TEXT NOT NULL,
    lot_number_seq        INTEGER NOT NULL,
    seq                   INTEGER NOT NULL,
    item_category         TEXT NOT NULL,
    premium_category      TEXT,
    product_category_code TEXT NOT NULL,
    length_spec_lower     NUMERIC NOT NULL,
    thickness_spec_lower   NUMERIC NOT NULL,
    thickness_spec_upper   NUMERIC NOT NULL,
    quality_grade         TEXT NOT NULL,
    count                 INTEGER NOT NULL,
    quantity              NUMERIC NOT NULL,
    pass_fail_category    TEXT,
    PRIMARY KEY (lot_number_year, lot_number_location, lot_number_seq, seq),
    FOREIGN KEY (lot_number_year, lot_number_location, lot_number_seq) REFERENCES lot
);
```

### 判断基準（他のエンティティにも適用）

- 状態間でカラムの重複が多い（8割以上共通） → シングルテーブル
- 状態間でカラム構成が大きく異なる → テーブル分割を検討
- このPoCでは全エンティティがシングルテーブル方式で十分

---

## 動作確認

```bash
# ロット作成
curl -X POST http://localhost:8080/lots -H "Content-Type: application/json" -d '{...}'

# 製造完了
curl -X POST http://localhost:8080/lots/2024-A-001/complete-manufacturing \
  -H "Content-Type: application/json" \
  -d '{"date": "2024-04-01"}'

# 不正な遷移（製造中に出荷完了 → エラー）
curl -X POST http://localhost:8080/lots/2024-A-001/complete-shipping \
  -H "Content-Type: application/json" \
  -d '{"date": "2024-04-10"}'
# → 400 Bad Request
```

---

## 次のステップ

Step 7が完了したら [Step 8: PBT導入](./step08.md) へ進む。
