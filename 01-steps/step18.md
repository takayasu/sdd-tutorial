# Step 18: 予約・委託・品目変換のAPI実装

## 目的

### これは何か

予約販売案件・委託販売案件・品目変換のライフサイクルをREST APIとして実装する。これにより、domain-model-sales-management.md に定義された全23個のbehaviorがAPI化される。

### なぜやるのか

- ドメインモデル全体の実装を完成させる
- 予約と委託はそれぞれ独自のライフサイクルを持つ。直接販売案件とは異なるパターンを体験する
- 品目変換は「既存の在庫ロットに新しい状態遷移を追加する」例。モデルの拡張方法を学ぶ

### 何がうれしいのか

- 全behaviorが実装され、PoCの機能面が完成する
- 「DSL → 型 → API → PBT → CI」のサイクルを3回繰り返したことで、パターンが完全に身につく
- CIが全ステップ通ることで、23個のbehavior全てが正しく動作していることが保証される

本ステップも Step 14 で確立した「集約API完全パッケージ」と「URL 集約規約 (`/sales-cases/{id}/{caseType}/...`)」をそのまま継承する。**`/reservation-cases/...` / `/consignment-cases/...` という subtype 別 URL は新設しない**。

## 完了条件

### (a) 動作要件

```bash
# 1. 予約販売案件の査定作成 (URL は /sales-cases/{id}/reservation/...)
$ curl -sf -X POST http://localhost:5000/sales-cases/2024-04-002/reservation/appraisals \
  -H "Content-Type: application/json" \
  -d '{"appraisalDate":"2024-04-05","estimatedLotInfo":"A商品分類コード 10本","estimatedAmount":300000,"version":1}' \
  | jq -e '.status=="estimate_appraised" and .version' >/dev/null

# 2. 委託販売案件の指定 (URL は /sales-cases/{id}/consignment/...)
$ curl -sf -X POST http://localhost:5000/sales-cases/2024-04-003/consignment/designate \
  -H "Content-Type: application/json" \
  -d '{"consignorName":"委託業者A","consignorCode":"CN001","designatedDate":"2024-04-01","version":1}' \
  | jq -e '.status=="consignment_designated"' >/dev/null

# 3. 委託指定解除 → 指定前に戻る (往復性)
$ curl -sf -X DELETE http://localhost:5000/sales-cases/2024-04-003/consignment/designation \
  -H "Content-Type: application/json" \
  -d '{"version":2}' \
  | jq -e '.status=="before_consignment"' >/dev/null

# 4. 詳細 GET が caseType 別の追加フィールドを返す
$ curl -sf http://localhost:5000/sales-cases/2024-04-002 \
  | jq -e '.caseType=="reservation" and .reservationPrice' >/dev/null
$ curl -sf http://localhost:5000/sales-cases/2024-04-003 \
  | jq -e '.caseType=="consignment" and .consignmentInfo' >/dev/null

# 5. 一覧 GET の caseType フィルタ
$ curl -sf "http://localhost:5000/sales-cases?caseType=reservation&limit=10" \
  | jq -e '.items|all(.caseType=="reservation")' >/dev/null

# 6. 予約案件への販売契約締結 → 型レベルで不可能なので 404 (URL が存在しない)
$ curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:5000/sales-cases/2024-04-002/reservation/contracts
404

# 7. version 楽観ロック (各 caseType で動くこと)
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/sales-cases/2024-04-002/reservation/determine \
    -H "Content-Type: application/json" \
    -d '{"version":99}'
409 application/problem+json

# 8. レガシー URL は登録されていない
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/reservation-cases/x/appraisals
404
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/consignment-cases/x/designate
404

# 9. 品目変換 (Lot 拡張)
$ curl -sf -X POST http://localhost:5000/lots/2024-A-001/convert-item \
  -H "Content-Type: application/json" \
  -d '{"convertedItemCategory":"premium","version":2}' \
  | jq -e '.status=="conversion_instructed"' >/dev/null

# 10. PBTが通ること（全behavior分）
$ dotnet test --filter "Category=PBT"
$ gradle test --tests "*PropertyTest*"

# 11. OpenAPI 完全記述
$ python3 -c "
import yaml
y = yaml.safe_load(open('openapi.yaml'))
required = ['ReservationPriceResponse','ConsignmentInfoResponse','ConsignmentResultResponse','ConvertItemRequest']
missing = [s for s in required if s not in y['components']['schemas']]
assert not missing, f'missing: {missing}'
print('OK')
"
OK
```

### (b) `ci.sh` verify セクションへ追記

Step 14 / 15 の PASS 群に加え:

```
PASS reservation-appraisal-create
PASS consignment-designate
PASS consignment-designation-reverse
PASS sales-case-detail-polymorphic-fields
PASS sales-case-list-casetype-filter
PASS reservation-contract-impossible-404
PASS reservation-version-conflict-409
PASS no-legacy-subtype-urls
PASS lot-convert-item
PASS openapi-schemas-complete
```

---

## 実装するAPI

### 予約販売案件 (URL は `/sales-cases/{id}/reservation/...`)

| メソッド | パス | 対応するbehavior |
|---|---|---|
| POST | `/sales-cases/{id}/reservation/appraisals` | 予約価格を作成する |
| POST | `/sales-cases/{id}/reservation/determine` | 予約を確定する |
| DELETE | `/sales-cases/{id}/reservation/determination` | 予約確定を取り消す |
| POST | `/sales-cases/{id}/reservation/delivery` | 納品を指示する |

### 委託販売案件 (URL は `/sales-cases/{id}/consignment/...`)

| メソッド | パス | 対応するbehavior |
|---|---|---|
| POST | `/sales-cases/{id}/consignment/designate` | 委託販売案件を指定する |
| DELETE | `/sales-cases/{id}/consignment/designation` | 委託販売案件指定を解除する |
| POST | `/sales-cases/{id}/consignment/result` | 委託販売結果を入力する |

### 品目変換

| メソッド | パス | 対応するbehavior |
|---|---|---|
| POST | `/lots/{id}/convert-item` | 品目変換を指示する |
| DELETE | `/lots/{id}/convert-item` | 品目変換指示を取り消す |

すべての mutation は `version: int` body 必須、競合時 409 + problem+json (Step 7 / 14 と同じ規約)。

---

## F# ドメインロジック

```fsharp
// Domain/ReservationCaseWorkflows.fs
module SalesManagement.Domain.ReservationCaseWorkflows

type EstimateError = | CaseNotEstimateAppraised | CaseNotDetermined

let createReservationPrice (info: ReservationPriceCommon) (case: BeforeReservationPriceCase)
    : EstimateAppraisedCase =
    { Common = case.Common; Appraisal = Undetermined { Common = info } }

let determineEstimate (date: DateOnly) (amount: Amount) (case: EstimateAppraisedCase)
    : EstimateDeterminedCase =
    { Common = case.Common; Appraisal = case.Appraisal; DeterminedDate = date }

let cancelDetermination (case: EstimateDeterminedCase)
    : EstimateAppraisedCase =
    { Common = case.Common; Appraisal = case.Appraisal }

let deliverEstimate (date: DateOnly) (case: EstimateDeterminedCase)
    : EstimateDeliveredCase =
    { Common = case.Common; Appraisal = case.Appraisal; DeterminedDate = case.DeterminedDate; DeliveryDate = date }

// Domain/ConsignmentCaseWorkflows.fs
module SalesManagement.Domain.ConsignmentCaseWorkflows

let designateConsignment (info: ConsignorInfo) (case: BeforeConsignmentCase)
    : ConsignmentDesignatedCase =
    { Common = case.Common; ConsignorInfo = info }

let cancelDesignation (case: ConsignmentDesignatedCase)
    : BeforeConsignmentCase =
    { Common = case.Common }

let enterConsignmentResult (result: ConsignmentResult) (case: ConsignmentDesignatedCase)
    : ConsignmentResultEnteredCase =
    { Common = case.Common; ConsignorInfo = case.ConsignorInfo; Result = result }
```

---

## Kotlin ドメインロジック

```kotlin
// domain/ReservationCaseWorkflows.kt
package salesmanagement.domain

import arrow.core.Either
import arrow.core.right

fun createReservationPrice(
    case: ReservationSalesCase.BeforeReservationPrice,
    info: ReservationPriceCommon
): Either<SalesCaseError, ReservationSalesCase.EstimateAppraised> =
    ReservationSalesCase.EstimateAppraised(
        common = case.common,
        appraisal = EstimatePriceAppraisal.Undetermined(info)
    ).right()

fun cancelDesignation(
    case: ConsignmentSalesCase.ConsignmentDesignated
): Either<SalesCaseError, ConsignmentSalesCase.BeforeConsignment> =
    ConsignmentSalesCase.BeforeConsignment(common = case.common).right()
```

---

## PBT追加

```fsharp
// F#
testProperty "予約確定取消後は査定済み状態に戻る（往復性）" <|
    fun () ->
        let appraised = createEstimateAppraisedCase ()
        let determined = determineEstimate (DateOnly(2024, 5, 1)) (Amount 100000) appraised
        let cancelled = cancelDetermination determined
        cancelled.Common = appraised.Common

testProperty "委託指定解除後は指定前に戻る（往復性）" <|
    fun () ->
        let before = createBeforeConsignmentCase ()
        let designated = designateConsignment someConsignorInfo before
        let cancelled = cancelDesignation designated
        cancelled.Common = before.Common
```

```kotlin
// Kotlin
test("予約確定取消後は査定済み状態に戻る（往復性）") {
    forAll(salesCaseCommonArb, reservationPriceArb) { common, appraisal ->
        val appraised = ReservationSalesCase.EstimateAppraised(common, appraisal)
        val determined = determineEstimate(appraised, LocalDate.now(), Amount(100000)).getOrNull()!!
        val cancelled = cancelDetermination(determined).getOrNull()!!
        cancelled.common == common
    }
}
```

---

## 全behaviorの実装確認チェックリスト

domain-model-sales-management.md の全behaviorが実装されていることを確認：

- [x] 製造完了を指示する（Step 7）
- [x] 出荷を指示する（Step 7）
- [x] 出荷完了を指示する（Step 7）
- [x] 製造完了を取り消す（Step 7）
- [x] 販売案件を作成する（Step 14）
- [x] 販売案件を削除する（Step 14）
- [x] 価格査定を作成する（Step 14）
- [x] 価格査定を更新する（Step 15）
- [x] 価格査定を削除する（Step 14）
- [x] 販売契約を締結する（Step 14）
- [x] 販売契約を削除する（Step 14）
- [x] 出庫を指示する（Step 14）
- [x] 出庫完了を指示する（Step 14）
- [x] 出庫指示を取り消す（Step 14）
- [x] 予約価格を作成する（Step 18）
- [x] 予約を確定する（Step 18）
- [x] 予約確定を取り消す（Step 18）
- [x] 納品を指示する（Step 18）
- [x] 委託販売案件を指定する（Step 18）
- [x] 委託販売案件指定を解除する（Step 18）
- [x] 委託販売結果を入力する（Step 18）
- [x] 品目変換を指示する（Step 18）
- [x] 品目変換指示を取り消す（Step 18）

---

## 次のステップ

Step 18が完了したら [Step 19: 品質ダッシュボード構築](./step19.md) へ進む。
