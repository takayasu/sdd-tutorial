# Step 12: 直接販売案件 + 価格査定 + 販売契約の型定義

## 目的

### これは何か

domain-model-section2.md のDSLをAI（Kiro CLI）で型定義に変換する。Step 6では在庫ロットだけだったが、ここでは「直接販売案件」「価格査定」「販売契約」という、より複雑なビジネス概念を型で表現する。

### なぜやるのか

- 直接販売案件は5段階の状態（査定前→査定済み→契約済み→出荷指示→出荷完了）を持ち、各段階で保持するデータが増えていく
- 「査定前の案件に契約を締結する」といった業務上ありえない操作を、型レベルで防ぐ
- Step 6で学んだ「状態を型で表現する」パターンを、より複雑なドメインに適用する

### 何がうれしいのか

- ビジネスルールの複雑さが増しても、型が正しさを保証してくれる
- 既存のCI（フォーマッター、リンター、PBT）がそのまま動くことで、「新しいコードを追加しても既存が壊れていない」ことが自動確認される

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet build
  Build succeeded.
      0 Warning(s)
      0 Error(s)

# 既存のPBTも引き続き通ること
$ dotnet test --filter "Category=PBT"
  Passed!  - Failed:     0, Passed:     3, Skipped:     0, Total:     3
```

### Kotlin

```bash
$ cd kotlin
$ gradle build
BUILD SUCCESSFUL in Xs

# 既存のPBTも引き続き通ること
$ gradle test --tests "*PropertyTest*"
BUILD SUCCESSFUL in Xs
3 tests completed, 0 failed
```

---

## Kiro CLIでの変換

domain-model-section2.md をコンテキストとして渡し、型定義を生成する。Step 6で作成した型を参照する形で追加する。

---

## F# 期待される型定義の構造

```fsharp
// Domain/SalesCaseTypes.fs
module SalesManagement.Domain.SalesCaseTypes

open System
open SalesManagement.Domain.Types

// 追加の基本値型
type SalesCaseNumber = {
    Year: int
    Month: int
    Seq: int
}

type AppraisalNumber = {
    Year: int
    Month: int
    Seq: int
}

type ContractNumber = {
    Year: int
    Month: int
    Seq: int
}

// 販売案件共通
type SalesCaseCommon = {
    SalesCaseNumber: SalesCaseNumber
    DivisionCode: DivisionCode
    SalesDate: DateOnly
    Lots: InventoryLot list  // 1件以上
}

// 価格査定
type AppraisalCommon = {
    AppraisalNumber: AppraisalNumber
    AppraisalDate: DateOnly
    DeliveryDate: DateOnly
    SalesMarket: string
    BaseUnitPriceDate: string
    PeriodAdjustmentRateDate: string
    CounterpartyAdjustmentRateDate: string
    TaxExcludedEstimatedTotal: Amount
    LotAppraisals: LotAppraisal list  // 1件以上
}

type NormalAppraisal = { Common: AppraisalCommon }

type AgreementAppraisal = {
    Common: AppraisalCommon
    CustomerContractNumber: string
    ContractAdjustmentRate: decimal
}

type PriceAppraisal =
    | Normal of NormalAppraisal
    | Agreement of AgreementAppraisal

// 販売契約
type Buyer = {
    CustomerNumber: string
    AgentName: string option
}

type SalesInfo = {
    SalesType: int
    Item: string
    DeliveryMethod: string
    PaymentDeferralCondition: string option
    SalesMethod: int
    Reason: string option
    Usage: string option
    PaymentDeferralAmount: Amount option
}

type SalesPriceInfo = {
    TaxExcludedContractAmountTaxable: Amount
    ConsumptionTax: Amount
    TaxExcludedPaymentAmount: Amount
    PaymentConsumptionTax: Amount
}

type SalesContract = {
    ContractNumber: ContractNumber
    ContractDate: DateOnly
    Person: string
    Buyer: Buyer
    SalesInfo: SalesInfo
    SalesPriceInfo: SalesPriceInfo
    AppraisalNumber: AppraisalNumber
}

// 出荷指示情報
type ShippingInstructionInfo = {
    ShippingInstructionDate: DateOnly
}

// 直接販売案件（状態を型で表現）
type BeforeAppraisalCase = { Common: SalesCaseCommon }

type AppraisedCase = {
    Common: SalesCaseCommon
    Appraisal: PriceAppraisal
}

type ContractedCase = {
    Common: SalesCaseCommon
    Appraisal: PriceAppraisal
    Contract: SalesContract
}

type ShippingInstructedCase = {
    Common: SalesCaseCommon
    Appraisal: PriceAppraisal
    Contract: SalesContract
    ShippingInstruction: ShippingInstructionInfo
}

type ShippingCompletedCase = {
    Common: SalesCaseCommon
    Appraisal: PriceAppraisal
    Contract: SalesContract
    ShippingInstruction: ShippingInstructionInfo
    ShippingCompletedDate: DateOnly
}

type DirectSalesCase =
    | BeforeAppraisal of BeforeAppraisalCase
    | Appraised of AppraisedCase
    | Contracted of ContractedCase
    | ShippingInstructed of ShippingInstructedCase
    | ShippingCompleted of ShippingCompletedCase
```

---

## Kotlin 期待される型定義の構造

```kotlin
// domain/SalesCaseTypes.kt
package salesmanagement.domain

import java.time.LocalDate

data class SalesCaseNumber(val year: Int, val month: Int, val seq: Int)
data class AppraisalNumber(val year: Int, val month: Int, val seq: Int)
data class ContractNumber(val year: Int, val month: Int, val seq: Int)

data class SalesCaseCommon(
    val salesCaseNumber: SalesCaseNumber,
    val divisionCode: DivisionCode,
    val salesDate: LocalDate,
    val lots: List<InventoryLot>  // 1件以上
)

// 価格査定
sealed interface PriceAppraisal {
    val common: AppraisalCommon

    data class Normal(override val common: AppraisalCommon) : PriceAppraisal
    data class Agreement(
        override val common: AppraisalCommon,
        val customerContractNumber: String,
        val contractAdjustmentRate: Double
    ) : PriceAppraisal
}

// 販売契約
data class SalesContract(
    val contractNumber: ContractNumber,
    val contractDate: LocalDate,
    val person: String,
    val buyer: Buyer,
    val salesInfo: SalesInfo,
    val salesPriceInfo: SalesPriceInfo,
    val appraisalNumber: AppraisalNumber
)

// 出荷指示情報
data class ShippingInstructionInfo(val shippingInstructionDate: LocalDate)

// 直接販売案件
sealed interface DirectSalesCase {
    val common: SalesCaseCommon

    data class BeforeAppraisal(override val common: SalesCaseCommon) : DirectSalesCase
    data class Appraised(
        override val common: SalesCaseCommon,
        val appraisal: PriceAppraisal
    ) : DirectSalesCase
    data class Contracted(
        override val common: SalesCaseCommon,
        val appraisal: PriceAppraisal,
        val contract: SalesContract
    ) : DirectSalesCase
    data class ShippingInstructed(
        override val common: SalesCaseCommon,
        val appraisal: PriceAppraisal,
        val contract: SalesContract,
        val shippingInstruction: ShippingInstructionInfo
    ) : DirectSalesCase
    data class ShippingCompleted(
        override val common: SalesCaseCommon,
        val appraisal: PriceAppraisal,
        val contract: SalesContract,
        val shippingInstruction: ShippingInstructionInfo,
        val shippingCompletedDate: LocalDate
    ) : DirectSalesCase
}
```

---

## ポイント

- 直接販売案件は5段階の状態を持ち、各段階で保持するデータが累積的に増える
- 価格査定は通常/顧客契約の2種類（OR）で、顧客契約には追加フィールドがある
- 販売契約は査定番号を持つことで、どの査定に基づくかを追跡できる

---

## 次のステップ

Step 12が完了したら [Step 13: マイグレーション追加](./step13.md) へ進む。
