# Step 16: 予約・委託販売案件の型定義

## 目的

### これは何か

domain-model-section3.md のDSLをAI（Kiro CLI）で型定義に変換する。予約販売案件・委託販売案件・品目変換を追加し、販売案件の全種別（直接/予約/委託）を統合した型を定義する。

### なぜやるのか

- 販売案件には3種類（直接/予約/委託）があり、それぞれ全く異なるライフサイクルを持つ
- 「予約販売案件に販売契約を締結する」といった業務上ありえない操作を、型レベルで不可能にする
- 全ステップ完了後、元のdomain-model-sales-management.mdと同一のドメインモデルになる

### 何がうれしいのか

- 3種類の販売案件を1つの `SalesCase` 型で統合することで、「どの種類の案件か」を型で判別できる
- 種類ごとに許可される操作が異なることが、コードの構造から明らか
- ドメインモデル全体が完成し、実業務の全体像がコードで表現される

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet build
  Build succeeded.
      0 Warning(s)
      0 Error(s)

# 既存のPBTが引き続き通ること
$ dotnet test --filter "Category=PBT"
  Passed!  - Failed:     0, Passed:    10, Skipped:     0, Total:    10
```

### Kotlin

```bash
$ cd kotlin
$ gradle build
BUILD SUCCESSFUL in Xs

# 既存のPBTが引き続き通ること
$ gradle test --tests "*PropertyTest*"
BUILD SUCCESSFUL in Xs
10 tests completed, 0 failed
```

---

## Kiro CLIでの変換

domain-model-section3.md をコンテキストとして渡し、型定義を生成する。

---

## F# 期待される型定義の構造

```fsharp
// Domain/ReservationCaseTypes.fs
module SalesManagement.Domain.ReservationCaseTypes

// 予約価格
type ReservationPriceCommon = {
    AppraisalNumber: AppraisalNumber
    AppraisalDate: DateOnly
    EstimatedLotInfo: string
    EstimatedAmount: Amount
}

type UndeterminedReservationPrice = { Common: ReservationPriceCommon }
type DeterminedReservationPrice = {
    Common: ReservationPriceCommon
    DeterminedDate: DateOnly
    DeterminedAmount: Amount
}

type EstimatePriceAppraisal =
    | Undetermined of UndeterminedReservationPrice
    | Determined of DeterminedReservationPrice

// 予約販売案件
type BeforeReservationPriceCase = { Common: SalesCaseCommon }
type EstimateAppraisedCase = { Common: SalesCaseCommon; Appraisal: EstimatePriceAppraisal }
type EstimateDeterminedCase = { Common: SalesCaseCommon; Appraisal: EstimatePriceAppraisal; DeterminedDate: DateOnly }
type EstimateDeliveredCase = { Common: SalesCaseCommon; Appraisal: EstimatePriceAppraisal; DeterminedDate: DateOnly; DeliveryDate: DateOnly }

type ReservationSalesCase =
    | BeforeReservationPrice of BeforeReservationPriceCase
    | EstimateAppraised of EstimateAppraisedCase
    | EstimateDetermined of EstimateDeterminedCase
    | EstimateDelivered of EstimateDeliveredCase

// 委託販売案件
type BeforeConsignmentCase = { Common: SalesCaseCommon }
type ConsignmentDesignatedCase = { Common: SalesCaseCommon; ConsignorInfo: ConsignorInfo }
type ConsignmentResultEnteredCase = { Common: SalesCaseCommon; ConsignorInfo: ConsignorInfo; Result: ConsignmentResult }

type ConsignmentSalesCase =
    | BeforeConsignment of BeforeConsignmentCase
    | ConsignmentDesignated of ConsignmentDesignatedCase
    | ConsignmentResultEntered of ConsignmentResultEnteredCase

// 販売案件（全種別統合）
type SalesCase =
    | Direct of DirectSalesCase
    | Reservation of ReservationSalesCase
    | Consignment of ConsignmentSalesCase
```

---

## Kotlin 期待される型定義の構造

```kotlin
// domain/ReservationCaseTypes.kt
package salesmanagement.domain

// 予約価格
sealed interface EstimatePriceAppraisal {
    val common: ReservationPriceCommon

    data class Undetermined(override val common: ReservationPriceCommon) : EstimatePriceAppraisal
    data class Determined(
        override val common: ReservationPriceCommon,
        val determinedDate: LocalDate,
        val determinedAmount: Amount
    ) : EstimatePriceAppraisal
}

// 予約販売案件
sealed interface ReservationSalesCase {
    val common: SalesCaseCommon

    data class BeforeReservationPrice(override val common: SalesCaseCommon) : ReservationSalesCase
    data class EstimateAppraised(override val common: SalesCaseCommon, val appraisal: EstimatePriceAppraisal) : ReservationSalesCase
    data class EstimateDetermined(override val common: SalesCaseCommon, val appraisal: EstimatePriceAppraisal, val determinedDate: LocalDate) : ReservationSalesCase
    data class EstimateDelivered(override val common: SalesCaseCommon, val appraisal: EstimatePriceAppraisal, val determinedDate: LocalDate, val deliveryDate: LocalDate) : ReservationSalesCase
}

// 委託販売案件
sealed interface ConsignmentSalesCase {
    val common: SalesCaseCommon

    data class BeforeConsignment(override val common: SalesCaseCommon) : ConsignmentSalesCase
    data class ConsignmentDesignated(override val common: SalesCaseCommon, val consignorInfo: ConsignorInfo) : ConsignmentSalesCase
    data class ConsignmentResultEntered(override val common: SalesCaseCommon, val consignorInfo: ConsignorInfo, val result: ConsignmentResult) : ConsignmentSalesCase
}

// 販売案件（全種別統合）
sealed interface SalesCase {
    data class Direct(val case: DirectSalesCase) : SalesCase
    data class Reservation(val case: ReservationSalesCase) : SalesCase
    data class Consignment(val case: ConsignmentSalesCase) : SalesCase
}
```

---

## ポイント

- `SalesCase` が全種別を統合するトップレベルの型。これにより「予約販売案件に販売契約を締結する」操作が型レベルで不可能になる
- 予約価格は確定前/確定後で型が分かれる（確定後は確定金額が必須）
- 品目変換は在庫ロットに新しい状態を追加する拡張

---

## 次のステップ

Step 16が完了したら [Step 17: マイグレーション追加](./step17.md) へ進む。
