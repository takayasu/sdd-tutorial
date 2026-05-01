# Step 6: 在庫ロットの型定義

## 目的

### これは何か

ドメインモデルDSL（domain-model-section1.md）をAI（Kiro CLI）に渡して、プログラミング言語の「型定義」に変換する。型定義とは、データの構造を厳密に定義したもの。

### なぜやるのか

- 「在庫ロットには製造中・製造完了・出荷指示済み・出荷完了の4状態がある」というビジネスルールを、コードの構造そのもので表現する
- 型で表現することで、「製造中ロットに出荷日がある」といった不正な状態がそもそも作れなくなる（コンパイルエラーになる）
- DSLという仕様書から型定義を自動生成するため、仕様と実装がずれない

### 何がうれしいのか

- バグの多くは「ありえない状態」が発生することで起きる。型で防げば、テストを書く前にバグを潰せる
- 仕様変更時はDSLを変更 → 型を再生成するだけ。手動で整合性を取る必要がない
- F#の判別共用体 / Kotlinのsealed classが、DSLの `OR` にそのまま対応する

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet build
  SalesManagement -> /path/to/bin/Debug/net8.0/SalesManagement.dll
  Build succeeded.
      0 Warning(s)
      0 Error(s)

# 型が定義されていることを確認（REPLで）
$ dotnet fsi
> open SalesManagement.Domain.Types;;
> let lot = Manufacturing { Common = { LotNumber = { Year = 2024; Location = "A"; Seq = 1 }; ... } };;
val lot : InventoryLot = Manufacturing ...
```

### Kotlin

```bash
$ cd kotlin
$ gradle build
BUILD SUCCESSFUL in Xs

# 型が定義されていることを確認（テストで）
$ gradle test --tests "*TypesTest*"
BUILD SUCCESSFUL in Xs
```

---

## Kiro CLIでの変換

domain-model-section1.md をコンテキストとして渡し、型定義を生成する。

---

## F# 期待される型定義の構造

```fsharp
// Types.fs
module SalesManagement.Domain.Types

// NonEmptyList（1件以上を型で保証）
type NonEmptyList<'a> = { Head: 'a; Tail: 'a list }

// 基本値型
type DivisionCode = DivisionCode of int
type DepartmentCode = DepartmentCode of int
type SectionCode = SectionCode of int

type LotNumber = {
    Year: int
    Location: string
    Seq: int
}

type Amount = private Amount of int
type Quantity = private Quantity of decimal
type Count = private Count of int

// 品目区分
type ItemCategory =
    | Standard
    | Premium
    | Special

// ロット明細
type LotDetail = {
    ItemCategory: ItemCategory
    PremiumCategory: string option
    ProductCategoryCode: string
    LengthSpecLower: decimal
    ThicknessSpecLower: decimal
    ThicknessSpecUpper: decimal
    QualityGrade: string
    Count: Count
    Quantity: Quantity
    InspectionResultCategory: string option
}

// ロット共通
type LotCommon = {
    LotNumber: LotNumber
    DivisionCode: DivisionCode
    DepartmentCode: DepartmentCode
    SectionCode: SectionCode
    ProcessCategory: int
    InspectionCategory: int
    ManufacturingCategory: int
    Details: LotDetail NonEmptyList  // 1件以上を型で保証
}

// 在庫ロット（状態を型で表現）
type ManufacturingLot = { Common: LotCommon }

type ManufacturedLot = {
    Common: LotCommon
    ManufacturingCompletedDate: System.DateOnly
}

type ShippingInstructedLot = {
    Common: LotCommon
    ManufacturingCompletedDate: System.DateOnly
    ShippingDeadlineDate: System.DateOnly
}

type ShippedLot = {
    Common: LotCommon
    ManufacturingCompletedDate: System.DateOnly
    ShippingDeadlineDate: System.DateOnly
    ShippedDate: System.DateOnly
}

type InventoryLot =
    | Manufacturing of ManufacturingLot
    | Manufactured of ManufacturedLot
    | ShippingInstructed of ShippingInstructedLot
    | Shipped of ShippedLot
```

---

## Kotlin 期待される型定義の構造

### 依存関係追加（build.gradle.kts）

```kotlin
dependencies {
    // 既存に追加（NonEmptyList等を使用）
    implementation("io.arrow-kt:arrow-core:1.2.4")
}
```

### 型定義

```kotlin
// Types.kt
package salesmanagement.domain

import arrow.core.NonEmptyList
import java.time.LocalDate

// 基本値型
@JvmInline value class DivisionCode(val value: Int)
@JvmInline value class DepartmentCode(val value: Int)
@JvmInline value class SectionCode(val value: Int)

data class LotNumber(
    val year: Int,
    val location: String,
    val seq: Int
)

@JvmInline value class Amount(val value: Int) { init { require(value >= 0) } }
@JvmInline value class Quantity(val value: Double) { init { require(value >= 0.001) } }
@JvmInline value class Count(val value: Int) { init { require(value >= 1) } }

// 品目区分
sealed interface ItemCategory {
    data object Standard : ItemCategory
    data object Premium : ItemCategory
    data object Special : ItemCategory
}

// ロット明細
data class LotDetail(
    val itemCategory: ItemCategory,
    val premiumCategory: String?,
    val productCategoryCode: String,
    val lengthSpecLower: Double,
    val thicknessSpecLower: Double,
    val thicknessSpecUpper: Double,
    val qualityGrade: String,
    val count: Count,
    val quantity: Quantity,
    val inspectionResultCategory: String?
)

// ロット共通
data class LotCommon(
    val lotNumber: LotNumber,
    val divisionCode: DivisionCode,
    val departmentCode: DepartmentCode,
    val sectionCode: SectionCode,
    val processCategory: Int,
    val inspectionCategory: Int,
    val manufacturingCategory: Int,
    val details: NonEmptyList<LotDetail>  // 1件以上を型で保証
)

// 在庫ロット（状態を型で表現）
sealed interface InventoryLot {
    val common: LotCommon

    data class Manufacturing(override val common: LotCommon) : InventoryLot

    data class Manufactured(
        override val common: LotCommon,
        val manufacturingCompletedDate: LocalDate
    ) : InventoryLot

    data class ShippingInstructed(
        override val common: LotCommon,
        val manufacturingCompletedDate: LocalDate,
        val shippingDeadlineDate: LocalDate
    ) : InventoryLot

    data class Shipped(
        override val common: LotCommon,
        val manufacturingCompletedDate: LocalDate,
        val shippingDeadlineDate: LocalDate,
        val shippedDate: LocalDate
    ) : InventoryLot
}
```

---

## ポイント

- F#: 判別共用体（`type InventoryLot = Manufacturing of ... | Manufactured of ...`）でDSLのORを直接表現
- Kotlin: `sealed interface` + `data class` でORを表現。`@JvmInline value class` で基本値型をラップ
- 両言語とも、不正な状態（例：製造中ロットに出荷日がある）が型レベルで存在できない
- `NonEmptyList` により「1件以上」の制約を型レベルで保証。空リストを渡すコードがコンパイルエラーになる

---

## 次のステップ

Step 6が完了したら [Step 7: 在庫ロットの状態遷移API実装](./step07.md) へ進む。
