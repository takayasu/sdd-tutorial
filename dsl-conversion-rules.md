# DSL → コード変換ルール表

DSLの構文を F# / Kotlin にどう変換するかのルール。

## データ構造

| DSL構文 | 意味 | F# | Kotlin |
|---|---|---|---|
| `data X = A AND B AND C` | 直積（全部そろっている） | `type X = { A: A; B: B; C: C }` | `data class X(val a: A, val b: B, val c: C)` |
| `data X = A OR B OR C` | 直和（どれか1つ） | `type X = \| A of A \| B of B \| C of C` | `sealed interface X` + 各 `data class` |
| `data X = 整数` | 単純値型（プリミティブのラッパー） | `type X = X of int` | `@JvmInline value class X(val value: Int)` |
| `data X = 文字列` | 単純値型 | `type X = X of string` | `@JvmInline value class X(val value: String)` |
| `data X = 数値` | 単純値型（小数） | `type X = X of decimal` | `@JvmInline value class X(val value: BigDecimal)` |
| `AND フィールド?` | オプショナル（あってもなくてもよい） | `フィールド: フィールド option` | `val フィールド: フィールド?` |
| `AND List<X>` | コレクション | `X list`（コメントに「1件以上」とあれば `X * X list`） | `List<X>`（1件以上なら `NonEmptyList<X>`） |

## 直和型（OR）の展開パターン

DSL:
```
data 在庫ロット = 製造中ロット OR 製造完了ロット OR 出荷指示済みロット OR 出荷完了ロット
```

F#:
```fsharp
type InventoryLot =
    | Manufacturing of ManufacturingLot
    | Manufactured of ManufacturedLot
    | ShippingInstructed of ShippingInstructedLot
    | Shipped of ShippedLot
```

Kotlin:
```kotlin
sealed interface InventoryLot {
    data class Manufacturing(val common: LotCommon) : InventoryLot
    data class Manufactured(val common: LotCommon, val manufacturingCompletedDate: LocalDate) : InventoryLot
    data class ShippingInstructed(...) : InventoryLot
    data class Shipped(...) : InventoryLot
}
```

## 共通部分の合成（AND による継承的構造）

DSL:
```
data ロット共通 = ロット番号 AND 事業部コード AND ...
data 製造中ロット = ロット共通
data 製造完了ロット = ロット共通 AND 製造完了日
```

F#:
```fsharp
type LotCommon = { LotNumber: LotNumber; DivisionCode: DivisionCode; ... }
type ManufacturingLot = { Common: LotCommon }
type ManufacturedLot = { Common: LotCommon; ManufacturingCompletedDate: DateOnly }
```

Kotlin:
```kotlin
data class LotCommon(val lotNumber: LotNumber, val divisionCode: DivisionCode, ...)
data class ManufacturingLot(val common: LotCommon)  // sealed interfaceのケースとして定義
data class ManufacturedLot(val common: LotCommon, val manufacturingCompletedDate: LocalDate)
```

## 振る舞い（behavior）

DSL:
```
behavior 製造完了を指示する = 製造中ロット AND 製造完了日 -> 製造完了ロット OR 製造完了指示エラー
```

F#:
```fsharp
// 入力型で不正な呼び出しを防ぐため、Result不要なケースもある
val completeManufacturing: ManufacturingLot -> DateOnly -> Result<ManufacturedLot, ManufacturingCompletionError>
```

Kotlin:
```kotlin
fun completeManufacturing(lot: ManufacturingLot, date: LocalDate): Either<ManufacturingCompletionError, ManufacturedLot>
```

### 変換ルール

| DSL | F# | Kotlin |
|---|---|---|
| `入力1 AND 入力2 ->` | 関数の引数（カリー化 or タプル） | 関数の引数 |
| `-> 出力 OR エラー` | `Result<出力, エラー>` | `Either<エラー, 出力>` |
| `-> 出力`（エラーなし） | 戻り値そのまま | 戻り値そのまま |

## エラー型

DSLの `OR エラー名` から sealed type を生成する。エラーの内部構造はDSLでは定義しないため、実装時に判断する。

F#:
```fsharp
type ManufacturingCompletionError = | LotNotInManufacturing | InvalidDate of string
```

Kotlin:
```kotlin
sealed interface ManufacturingCompletionError {
    data object LotNotInManufacturing : ManufacturingCompletionError
    data class InvalidDate(val reason: String) : ManufacturingCompletionError
}
```

## 命名変換

DSLは日本語、コードは英語。変換は以下の方針：

- 型名: DSLの日本語名を英訳（例: `在庫ロット` → `InventoryLot`）
- フィールド名: 同上（例: `製造完了日` → `manufacturingCompletedDate`）
- 関数名: behaviorの動詞を英訳（例: `製造完了を指示する` → `completeManufacturing`）
- エラー型: `〜エラー` → `〜Error`（例: `製造完了指示エラー` → `ManufacturingCompletionError`）
