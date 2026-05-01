# Step 8: PBT導入（在庫ロットの状態遷移）

## 目的

### これは何か

プロパティベーステスト（PBT）を導入する。PBTとは、「このコードはどんな入力に対しても、この性質（プロパティ）を満たすはず」という形でテストを書く手法。通常のテストが「入力Aを与えたら出力Bになる」と具体例で書くのに対し、PBTはランダムな入力を大量に生成して性質を検証する。

### なぜやるのか

- 通常のテストでは、テストケースに書いた具体例しか検証できない。PBTは100〜1000パターンのランダム入力で検証するため、人間が思いつかないエッジケースを発見できる
- 状態遷移の正しさを「性質」として表現できる：
  - 往復性：「製造完了→取消」で元に戻る
  - 順序性：製造中→製造完了→出荷指示→出荷完了の順でのみ遷移可能
  - 不変性：状態遷移してもロット番号は変わらない

### 何がうれしいのか

- AIが生成したコードに対して「どんな入力でも正しく動く」ことを自動検証できる
- CIに組み込むことで、コード変更のたびに自動的に大量のパターンで検証される
- 「CIが通った = ドメインロジックが正しい」と言い切れる度合いが大幅に上がる

## 完了条件

### F#

```bash
$ cd ../sales-management/apps/api-fsharp
$ dotnet test --filter "Category=PBT"
  Determining projects to restore...
  All projects are up-to-date for restore.
  Starting test execution, please wait...
  A total of 1 test files matched the specified pattern.

  Passed!  - Failed:     0, Passed:     3, Skipped:     0, Total:     3
$ echo $?
0
```

### Kotlin

```bash
$ cd kotlin
$ gradle test --tests "*PropertyTest*"
> Task :test

salesmanagement.domain.LotPropertyTest
  ✓ 製造完了→取消で元の製造中ロットに戻る（往復性）
  ✓ 製造中→製造完了→出荷指示→出荷完了の順序で遷移可能
  ✓ 状態遷移でロット共通情報は変わらない（不変性）

BUILD SUCCESSFUL in Xs
3 tests completed, 0 failed
```

---

## F#（xUnit + FsCheck.Xunit）

> ツール選定の経緯: 当初は Expecto + FsCheck を想定していたが、Expecto は最終 stable リリース（10.2.1）が 2023-03 から更新されておらず、後継の v11 系も alpha のまま。一方 FsCheck.Xunit は 3.x 系が安定リリースされ、xUnit は .NET テストの事実上の標準で `dotnet test --filter "Category=PBT"` がそのまま動く。本 PoC では **xUnit + FsCheck.Xunit + FsCheck 3** に統一する。

### 1. テストプロジェクト作成

```bash
mkdir -p fsharp/tests/SalesManagement.Tests
cd ../sales-management/apps/api-fsharp/tests/SalesManagement.Tests
dotnet new console -lang F#
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package FsCheck
dotnet add package FsCheck.Xunit
dotnet add reference ../../src/SalesManagement/SalesManagement.fsproj
```

`SalesManagement.Tests.fsproj` の `<PropertyGroup>` に以下を追加（自前 Program.fs を使うため）:

```xml
<IsPackable>false</IsPackable>
<GenerateProgramFile>false</GenerateProgramFile>
```

### 2. テストコード

```fsharp
// tests/SalesManagement.Tests/LotPropertyTests.fs
module SalesManagement.Tests.LotPropertyTests

open System
open Xunit
open FsCheck
open FsCheck.FSharp
open FsCheck.Xunit
open SalesManagement.Domain.Types
open SalesManagement.Domain.LotWorkflows

// 値型のスマートコンストラクタを test-only で必ず Ok にラップ
let private mustCount n =
    match Count.tryCreate n with
    | Ok c -> c
    | Error e -> failwithf "test setup: %s" e

let private mustQuantity v =
    match Quantity.tryCreate v with
    | Ok q -> q
    | Error e -> failwithf "test setup: %s" e

// ジェネレータ
let private lotDetailGen = gen {
    return
        { ItemCategory = Standard
          PremiumCategory = None
          ProductCategoryCode = "A分類"
          LengthSpecLower = 100m
          ThicknessSpecLower = 10m
          ThicknessSpecUpper = 20m
          QualityGrade = "A"
          Count = mustCount 10
          Quantity = mustQuantity 5.0m
          InspectionResultCategory = None }
}

let private lotCommonGen = gen {
    let! year = Gen.choose (2020, 2030)
    let! seq = Gen.choose (1, 9999)
    let! detail = lotDetailGen
    return
        { LotNumber = { Year = year; Location = "A"; Seq = seq }
          DivisionCode = DivisionCode 1
          DepartmentCode = DepartmentCode 1
          SectionCode = SectionCode 1
          ProcessCategory = 1
          InspectionCategory = 1
          ManufacturingCategory = 1
          Details = { Head = detail; Tail = [] } }
}

let private dateGen = gen {
    let! year = Gen.choose (2020, 2030)
    let! month = Gen.choose (1, 12)
    let! day = Gen.choose (1, 28)
    return DateOnly(year, month, day)
}

// FsCheck.Xunit が [<Property>] 引数に自動注入する Arbitrary 群
type Arbitraries =
    static member LotCommon() : Arbitrary<LotCommon> = Arb.fromGen lotCommonGen
    static member DateOnly() : Arbitrary<DateOnly> = Arb.fromGen dateGen

[<Properties(Arbitrary = [| typeof<Arbitraries> |])>]
module Tests =

    [<Property>]
    [<Trait("Category", "PBT")>]
    let ``製造完了→取消で元の製造中ロットに戻る（往復性）`` (date: DateOnly) (common: LotCommon) =
        let lot: ManufacturingLot = { Common = common }
        let manufactured = completeManufacturing date lot
        let cancelled = cancelManufacturingCompletion manufactured
        cancelled.Common = common

    [<Property>]
    [<Trait("Category", "PBT")>]
    let ``製造中→製造完了→出荷指示→出荷完了の順序で遷移可能`` (common: LotCommon) =
        let lot: ManufacturingLot = { Common = common }
        let d1 = DateOnly(2024, 4, 1)
        let d2 = DateOnly(2024, 4, 15)
        let d3 = DateOnly(2024, 4, 20)
        let manufactured = completeManufacturing d1 lot
        let instructed = instructShipping d2 manufactured
        let shipped = completeShipping d3 instructed
        shipped.Common = common
        && shipped.ManufacturingCompletedDate = d1
        && shipped.ShippingDeadlineDate = d2
        && shipped.ShippedDate = d3

    [<Property>]
    [<Trait("Category", "PBT")>]
    let ``状態遷移でロット共通情報は変わらない（不変性）`` (common: LotCommon) =
        let lot: ManufacturingLot = { Common = common }
        let manufactured = completeManufacturing (DateOnly(2024, 1, 1)) lot
        manufactured.Common = common
```

`Program.fs` は xUnit 経由の起動で参照されないが、`<OutputType>Exe</OutputType>` のため最小スタブを置く:

```fsharp
module SalesManagement.Tests.Program

[<EntryPoint>]
let main _ = 0
```

### 3. 実行

```bash
cd ../sales-management/apps/api-fsharp
dotnet test --filter "Category=PBT"
```

`[<Trait("Category", "PBT")>]` が xUnit のネイティブ Trait として認識されるため、`--filter "Category=PBT"` がそのまま機能する。

### 4. ポイント

- **値型の private コンストラクタ**: `Quantity` / `Count` / `Amount` は不正値の生成を型レベルで抑止するため `private` にしている。テスト側からも直接構築できないので `Quantity.tryCreate` を `mustQuantity` でラップして使う。
- **`Arbitraries` 静的クラス**: `[<Properties(Arbitrary = [| typeof<Arbitraries> |])>]` を付けたモジュール内では、`[<Property>]` の引数型 `LotCommon` / `DateOnly` に対応する `Arbitraries.LotCommon()` / `Arbitraries.DateOnly()` が自動で参照される。`Prop.forAll` を毎回書くより簡潔。
- **`NonEmptyList` のデフォルト Arb は無い**: 自前型なので `Arb.generate<LotCommon>` のような全自動生成は使えない。生成パスは必ず `lotCommonGen` を経由する。

---

## Kotlin（Kotest Property Testing）

### 1. 依存関係追加（build.gradle.kts）

```kotlin
dependencies {
    // 既存に追加
    testImplementation("io.kotest:kotest-runner-junit5:5.9.1")
    testImplementation("io.kotest:kotest-property:5.9.1")
    testImplementation("io.kotest:kotest-assertions-core:5.9.1")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### 2. テストコード

```kotlin
// src/test/kotlin/salesmanagement/domain/LotPropertyTest.kt
package salesmanagement.domain

import io.kotest.core.spec.style.FunSpec
import io.kotest.property.Arb
import io.kotest.property.arbitrary.*
import io.kotest.property.forAll
import io.kotest.matchers.shouldBe
import arrow.core.nonEmptyListOf
import java.time.LocalDate

class LotPropertyTest : FunSpec({

    val lotDetailArb = arbitrary {
        LotDetail(
            itemCategory = ItemCategory.Standard,
            premiumCategory = null,
            productCategoryCode = "A分類",
            lengthSpecLower = 100.0,
            thicknessSpecLower = 10.0,
            thicknessSpecUpper = 20.0,
            qualityGrade = "A",
            count = Count(10),
            quantity = Quantity(5.0),
            inspectionResultCategory = null
        )
    }

    val lotCommonArb = arbitrary {
        LotCommon(
            lotNumber = LotNumber(
                year = Arb.int(2020..2030).bind(),
                location = "A",
                seq = Arb.int(1..9999).bind()
            ),
            divisionCode = DivisionCode(1),
            departmentCode = DepartmentCode(1),
            sectionCode = SectionCode(1),
            processCategory = 1,
            inspectionCategory = 1,
            manufacturingCategory = 1,
            details = nonEmptyListOf(lotDetailArb.bind())
        )
    }

    test("製造完了→取消で元の製造中ロットに戻る（往復性）") {
        forAll(lotCommonArb) { common ->
            val lot = InventoryLot.Manufacturing(common)
            val manufactured = completeManufacturing(lot, LocalDate.of(2024, 4, 1))
            val cancelled = cancelManufacturingCompletion(manufactured.getOrNull()!!)
            cancelled.getOrNull()!!.common == common
        }
    }

    test("製造中→製造完了→出荷指示→出荷完了の順序で遷移可能") {
        forAll(lotCommonArb) { common ->
            val lot = InventoryLot.Manufacturing(common)
            val d1 = LocalDate.of(2024, 4, 1)
            val d2 = LocalDate.of(2024, 4, 15)
            val d3 = LocalDate.of(2024, 4, 20)

            val manufactured = completeManufacturing(lot, d1).getOrNull()!!
            val instructed = instructShipping(manufactured, d2).getOrNull()!!
            val shipped = completeShipping(instructed, d3).getOrNull()!!

            shipped.common == common &&
                shipped.manufacturingCompletedDate == d1 &&
                shipped.shippingDeadlineDate == d2 &&
                shipped.shippedDate == d3
        }
    }

    test("状態遷移でロット共通情報は変わらない（不変性）") {
        forAll(lotCommonArb) { common ->
            val lot = InventoryLot.Manufacturing(common)
            val manufactured = completeManufacturing(lot, LocalDate.of(2024, 1, 1))
            manufactured.getOrNull()!!.common == common
        }
    }
})
```

### 3. 実行

```bash
gradle test --tests "*PropertyTest*"
```

---

## ci.sh への追加

```bash
echo "=== テスト（PBT） ==="
# F#: dotnet test --filter "Category=PBT"
# Kotlin: gradle test --tests "*PropertyTest*"
```

---

## PBTで検証するプロパティの考え方

| プロパティ | 意味 |
|---|---|
| 往復性 | A→B→A の遷移で元に戻る |
| 不変性 | 遷移しても変わらないデータがある |
| 順序性 | 正しい順序でのみ遷移可能 |
| 冪等性 | 同じ操作を2回やっても結果が同じ |

---

## 次のステップ

Step 8が完了したら [Step 9: テストカバレッジ](./step09.md) へ進む。
