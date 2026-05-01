# Step 14: 直接販売案件の集約API完全パッケージ + URL 集約規約

## 目的

### これは何か

直接販売案件のライフサイクル（査定前→査定済み→契約済み→出荷指示→出荷完了）を **Step 7 の「集約API完全パッケージ」テンプレに従って** REST API 化する。すなわち以下を**1 step で揃える**:

| 要素 | 内容 |
|---|---|
| Mutation | 案件作成 / 査定作成 / 契約締結 / 出荷指示 / 出荷完了 / 取消系 |
| **詳細 GET** | `GET /sales-cases/{id}` — `caseType` を含むポリモーフィック (direct/reservation/consignment 共通 URL) |
| **一覧 GET** | `GET /sales-cases?caseType=&status=&limit=&offset=` |
| **楽観ロック** | `sales_case` / `appraisal` / `contract` テーブルに `version INTEGER NOT NULL DEFAULT 1` |
| **エラー形式** | 全レスポンス `application/problem+json` (Step 7 と同じ規約) |
| **OpenAPI 完全記述** | `SalesCaseResponse`, `SalesCaseSummary`, `SalesCasesListResponse`, `CaseType` 等を `components.schemas` に |
| **URL 集約規約** | mutation も含め全エンドポイントを `/sales-cases/{id}/...` 配下に集約。subtype 別 URL (`/reservation-cases/...`, `/consignment-cases/...`) は**作らない** (Step 18 でも踏襲) |
| **ci.sh verify** | Step 1 の verify セクションへ追記 |

### URL 集約規約 (Step 14 / 15 / 18 共通)

集約 ID `salesCaseNumber` は subtype に関わらず同じ採番空間で発行されるので、**URL prefix も統一**する:

| Before (避ける) | After (採用) |
|---|---|
| `POST /sales-cases/{id}/appraisals` | `POST /sales-cases/{id}/direct/appraisals` |
| `POST /reservation-cases/{id}/appraisals` | `POST /sales-cases/{id}/reservation/appraisals` |
| `POST /consignment-cases/{id}/designate` | `POST /sales-cases/{id}/consignment/designate` |
| `GET /reservation-cases/{id}` | `GET /sales-cases/{id}` (caseType でポリモーフィック) |

これにより「ID は分かっているが種類が分からない時にどの URL を叩くか」問題が消え、フロント側 SWR キーも `/sales-cases/${id}` プレフィックスで統一できる。

### なぜやるのか

- DSLの `behavior`（販売案件を作成する、価格査定を作成する、販売契約を締結する等）をAPIエンドポイントとして実現する
- 「査定前の案件に契約を締結する」操作が型レベルでエラーになることを体験する
- PBTで「往復性」（契約削除→査定済みに戻る）等を検証する
- **Step 7 と同じテンプレで揃える**ことで、後付けの不整合 (一覧なし / version 非対称 / URL バラバラ / problem+json 混在) を最初から排除する

### 何がうれしいのか

- ビジネスルールが複雑になっても、型 + PBT + CIの仕組みが正しさを保証してくれる
- Step 7と同じパターン（型定義→ドメインロジック→API→DB）を繰り返すことで、開発サイクルが身につく
- CIが全ステップ通ることで「新機能を追加しても既存が壊れていない」ことが自動確認される
- Section 3 (Step 18) で予約・委託を足すときも、URL 規約と一覧/詳細 GET のテンプレが共通なので増分が小さい

## 完了条件

### (a) 動作要件

```bash
# 1. 販売案件作成 (version=1 が返る)
$ curl -sf -X POST http://localhost:5000/sales-cases \
  -H "Content-Type: application/json" \
  -d '{"caseType":"direct","lots":["2024-A-1"],"divisionCode":1,"salesDate":"2024-04-01"}' \
  | jq -e '.salesCaseNumber and (.version==1) and (.caseType=="direct")' >/dev/null

# 2. 詳細 GET (caseType ポリモーフィック)
$ curl -sf http://localhost:5000/sales-cases/2024-04-001 \
  | jq -e '.caseType=="direct" and .status and .version' >/dev/null

# 3. 一覧 GET (caseType / status フィルタ + ページング)
$ curl -sf "http://localhost:5000/sales-cases?caseType=direct&limit=10&offset=0" \
  | jq -e '.items and .total and (.limit==10)' >/dev/null

# 4. 査定前案件への契約締結 → 400 + problem+json
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/sales-cases/2024-04-001/direct/contracts \
    -H "Content-Type: application/json" \
    -d '{"contractDate":"2024-04-15","version":1}'
400 application/problem+json

# 5. 楽観ロック競合 → 409 + problem+json (古い version で更新)
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" \
    -X POST http://localhost:5000/sales-cases/2024-04-001/direct/appraisals \
    -H "Content-Type: application/json" \
    -d '{"appraisalDate":"2024-04-05","version":99}'
409 application/problem+json

# 6. 存在しない ID → 404 + problem+json
$ curl -s -o /dev/null -w "%{http_code} %{content_type}\n" http://localhost:5000/sales-cases/9999-99-999
404 application/problem+json

# 7. URL 集約規約: 古い subtype 別 URL は登録されていない (404)
$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/reservation-cases/2024-04-001/appraisals
404

# 8. PBTが通ること（在庫ロット + 販売案件）
$ dotnet test --filter "Category=PBT"      # F#
$ gradle test --tests "*PropertyTest*"     # Kotlin

# 9. OpenAPI 完全記述
$ python3 -c "
import yaml
y = yaml.safe_load(open('openapi.yaml'))
required = ['SalesCaseResponse','SalesCaseSummary','SalesCasesListResponse','CaseType']
missing = [s for s in required if s not in y['components']['schemas']]
assert not missing, f'missing: {missing}'
# 旧 URL prefix が paths に残っていないこと
forbidden = [p for p in y['paths'] if p.startswith('/reservation-cases') or p.startswith('/consignment-cases')]
assert not forbidden, f'forbidden URL prefixes: {forbidden}'
print('OK')
"
OK
```

### (b) `ci.sh` verify セクションへ追記

Step 7 で確立した verify ブロックに上記 1-7, 9 を追加し、`./ci.sh` が exit 0:

```
=== verify (smoke) ===
... (Step 1 / 7 の PASS 群) ...
PASS sales-case-create
PASS sales-case-detail-get
PASS sales-case-list-paged
PASS sales-case-invalid-transition-400
PASS sales-case-version-conflict-409
PASS sales-case-not-found-404
PASS no-legacy-subtype-urls
PASS openapi-schemas-complete
=== CI完了 ===
```

---

## 実装するAPI

| メソッド | パス | 対応するbehavior |
|---|---|---|
| POST | `/sales-cases` | 販売案件を作成する (`caseType` で direct/reservation/consignment) |
| **GET** | **`/sales-cases`** | **一覧取得 (`caseType`, `status`, `limit`, `offset`)** |
| **GET** | **`/sales-cases/{id}`** | **詳細取得 (caseType ポリモーフィック)** |
| DELETE | `/sales-cases/{id}` | 販売案件を削除する |
| POST | `/sales-cases/{id}/direct/appraisals` | 価格査定を作成する |
| PUT | `/sales-cases/{id}/direct/appraisals` | 価格査定を更新する |
| DELETE | `/sales-cases/{id}/direct/appraisals` | 価格査定を削除する |
| POST | `/sales-cases/{id}/direct/contracts` | 販売契約を締結する |
| DELETE | `/sales-cases/{id}/direct/contracts` | 販売契約を削除する |
| POST | `/sales-cases/{id}/direct/shipping-instruction` | 出庫を指示する |
| POST | `/sales-cases/{id}/direct/shipping-completion` | 出庫完了を指示する |
| DELETE | `/sales-cases/{id}/direct/shipping-instruction` | 出庫指示を取り消す |

すべての mutation は body に **`version: int`** を必須化する。サーバは `WHERE version = @expected` の UPDATE を実行し、行数 0 で 409 Conflict + problem+json (`type: "/errors/version-conflict"`) を返す。

---

## F# ドメインロジック

```fsharp
// Domain/SalesCaseWorkflows.fs
module SalesManagement.Domain.SalesCaseWorkflows

type SalesCaseCreationError = | NoManufacturedLots
type AppraisalCreationError = | CaseNotBeforeAppraisal
type ContractError = | CaseNotAppraised
type ShippingInstructionError = | CaseNotContracted
type ShippingCompletionError = | CaseNotShippingInstructed

let createSalesCase (lots: ManufacturedLot list) (caseNumber: SalesCaseNumber) (divisionCode: DivisionCode) (salesDate: DateOnly)
    : Result<BeforeAppraisalCase, SalesCaseCreationError> =
    match lots with
    | [] -> Error NoManufacturedLots
    | _ ->
        Ok { Common = {
            SalesCaseNumber = caseNumber
            DivisionCode = divisionCode
            SalesDate = salesDate
            // ManufacturedLot → InventoryLot に変換して格納
            // DSLの「販売案件共通.List<在庫ロット>」に対応
            // 入力を ManufacturedLot に限定することで、製造完了ロットのみ受付可能にする
            Lots = lots |> List.map (fun l -> Manufactured l)
        }}

let createAppraisal (appraisal: PriceAppraisal) (case: BeforeAppraisalCase)
    : AppraisedCase =
    { Common = case.Common; Appraisal = appraisal }

let concludeContract (contract: SalesContract) (case: AppraisedCase)
    : ContractedCase =
    { Common = case.Common; Appraisal = case.Appraisal; Contract = contract }

let instructShipping (info: ShippingInstructionInfo) (case: ContractedCase)
    : ShippingInstructedCase =
    { Common = case.Common; Appraisal = case.Appraisal; Contract = case.Contract; ShippingInstruction = info }

let completeShipping (date: DateOnly) (case: ShippingInstructedCase)
    : ShippingCompletedCase =
    { Common = case.Common; Appraisal = case.Appraisal; Contract = case.Contract
      ShippingInstruction = case.ShippingInstruction; ShippingCompletedDate = date }

// 取消操作
let deleteAppraisal (case: AppraisedCase) : BeforeAppraisalCase =
    { Common = case.Common }

let deleteContract (case: ContractedCase) : AppraisedCase =
    { Common = case.Common; Appraisal = case.Appraisal }

let cancelShippingInstruction (case: ShippingInstructedCase) : ContractedCase =
    { Common = case.Common; Appraisal = case.Appraisal; Contract = case.Contract }
```

---

## Kotlin ドメインロジック

```kotlin
// domain/SalesCaseWorkflows.kt
package salesmanagement.domain

import arrow.core.Either
import arrow.core.left
import arrow.core.right

sealed interface SalesCaseError {
    data object NoManufacturedLots : SalesCaseError
    data object CaseNotBeforeAppraisal : SalesCaseError
    data object CaseNotAppraised : SalesCaseError
    data object CaseNotContracted : SalesCaseError
    data object CaseNotShippingInstructed : SalesCaseError
}

// 入力を ManufacturedLot に限定し、内部で InventoryLot に変換して格納
// DSLの「List<製造完了ロット> -> 販売案件」に対応
fun createSalesCase(
    lots: NonEmptyList<InventoryLot.Manufactured>,
    caseNumber: SalesCaseNumber,
    divisionCode: DivisionCode,
    salesDate: LocalDate
): Either<SalesCaseError, DirectSalesCase.BeforeAppraisal> =
    DirectSalesCase.BeforeAppraisal(
        common = SalesCaseCommon(
            salesCaseNumber = caseNumber,
            divisionCode = divisionCode,
            salesDate = salesDate,
            lots = lots.map { it as InventoryLot }
        )
    ).right()

fun createAppraisal(
    case: DirectSalesCase.BeforeAppraisal,
    appraisal: PriceAppraisal
): Either<SalesCaseError, DirectSalesCase.Appraised> =
    DirectSalesCase.Appraised(common = case.common, appraisal = appraisal).right()

fun concludeContract(
    case: DirectSalesCase.Appraised,
    contract: SalesContract
): Either<SalesCaseError, DirectSalesCase.Contracted> =
    DirectSalesCase.Contracted(
        common = case.common,
        appraisal = case.appraisal,
        contract = contract
    ).right()

fun deleteContract(
    case: DirectSalesCase.Contracted
): Either<SalesCaseError, DirectSalesCase.Appraised> =
    DirectSalesCase.Appraised(common = case.common, appraisal = case.appraisal).right()
```

---

## PBT追加

```fsharp
// F# PBT例
testProperty "査定削除後は査定前状態に戻る（往復性）" <|
    fun () ->
        let case = createBeforeAppraisalCase ()
        let appraised = createAppraisal someAppraisal case
        let deleted = deleteAppraisal appraised
        deleted.Common = case.Common

testProperty "査定前の案件に契約を締結できない（型レベルで防止）" <|
    // この操作はそもそもコンパイルエラーになるため、テスト不要
    // → 型安全性の証明
```

```kotlin
// Kotlin PBT例
test("契約削除後は査定済み状態に戻る（往復性）") {
    forAll(salesCaseCommonArb, appraisalArb, contractArb) { common, appraisal, contract ->
        val appraised = DirectSalesCase.Appraised(common, appraisal)
        val contracted = concludeContract(appraised, contract).getOrNull()!!
        val deleted = deleteContract(contracted).getOrNull()!!
        deleted.common == common && deleted.appraisal == appraisal
    }
}
```

---

## 次のステップ

Step 14が完了したら [Step 15: 価格査定・販売契約のAPI実装](./step15.md) へ進む。
