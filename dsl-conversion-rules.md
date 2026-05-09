# DSL → コード変換ルール表

DSLの構文を Python / TypeScript にどう変換するかのルール。

## データ構造

| DSL構文 | 意味 | Python | TypeScript |
|---|---|---|---|
| `data X = A AND B AND C` | 直積（全部そろっている） | `@dataclass(frozen=True) class X` | `interface X { readonly a: A; readonly b: B; readonly c: C }` |
| `data X = A OR B OR C` | 直和（どれか1つ） | `X = Union[A, B, C]` + `Literal` discriminant | `type X = A \| B \| C` (discriminated union) |
| `data X = 整数` | 単純値型（プリミティブのラッパー） | `@dataclass(frozen=True) class X: value: int` | `type X = number & { readonly _brand: 'X' }` |
| `data X = 文字列` | 単純値型 | `@dataclass(frozen=True) class X: value: str` | `type X = string & { readonly _brand: 'X' }` |
| `data X = 数値` | 単純値型（小数） | `@dataclass(frozen=True) class X: value: Decimal` | `type X = number & { readonly _brand: 'X' }` |
| `AND フィールド?` | オプショナル（あってもなくてもよい） | `フィールド: 型 \| None = None` | `フィールド?: 型` |
| `AND List<X>` | コレクション | `list[X]`（1件以上なら `__post_init__` で検証） | `readonly X[]`（1件以上なら `[X, ...X[]]` タプル型） |

## 直和型（OR）の展開パターン

DSL:
```
data 在庫ロット = 製造中ロット OR 製造完了ロット OR 出荷指示済みロット OR 出荷完了ロット
```

Python:
```python
from __future__ import annotations
from dataclasses import dataclass
from datetime import date
from typing import Union, Literal

@dataclass(frozen=True)
class ManufacturingLot:
    type: Literal["manufacturing"]
    common: LotCommon

@dataclass(frozen=True)
class ManufacturedLot:
    type: Literal["manufactured"]
    common: LotCommon
    manufacturing_completed_date: date

@dataclass(frozen=True)
class ShippingInstructedLot:
    type: Literal["shipping_instructed"]
    common: LotCommon
    manufacturing_completed_date: date
    shipping_deadline_date: date

@dataclass(frozen=True)
class ShippedLot:
    type: Literal["shipped"]
    common: LotCommon
    manufacturing_completed_date: date
    shipping_deadline_date: date
    shipped_date: date

InventoryLot = Union[ManufacturingLot, ManufacturedLot, ShippingInstructedLot, ShippedLot]
```

TypeScript:
```typescript
interface ManufacturingLot {
  readonly type: 'manufacturing'
  readonly common: LotCommon
}

interface ManufacturedLot {
  readonly type: 'manufactured'
  readonly common: LotCommon
  readonly manufacturingCompletedDate: string  // ISO 8601
}

interface ShippingInstructedLot {
  readonly type: 'shipping_instructed'
  readonly common: LotCommon
  readonly manufacturingCompletedDate: string
  readonly shippingDeadlineDate: string
}

interface ShippedLot {
  readonly type: 'shipped'
  readonly common: LotCommon
  readonly manufacturingCompletedDate: string
  readonly shippingDeadlineDate: string
  readonly shippedDate: string
}

type InventoryLot =
  | ManufacturingLot
  | ManufacturedLot
  | ShippingInstructedLot
  | ShippedLot
```

## 共通部分の合成（AND による継承的構造）

DSL:
```
data ロット共通 = ロット番号 AND 事業部コード AND ...
data 製造中ロット = ロット共通
data 製造完了ロット = ロット共通 AND 製造完了日
```

Python:
```python
@dataclass(frozen=True)
class LotCommon:
    lot_number: LotNumber
    division_code: DivisionCode
    ...

@dataclass(frozen=True)
class ManufacturingLot:
    type: Literal["manufacturing"]
    common: LotCommon

@dataclass(frozen=True)
class ManufacturedLot:
    type: Literal["manufactured"]
    common: LotCommon
    manufacturing_completed_date: date
```

TypeScript:
```typescript
interface LotCommon {
  readonly lotNumber: LotNumber
  readonly divisionCode: DivisionCode
  ...
}

interface ManufacturingLot {
  readonly type: 'manufacturing'
  readonly common: LotCommon
}

interface ManufacturedLot {
  readonly type: 'manufactured'
  readonly common: LotCommon
  readonly manufacturingCompletedDate: string
}
```

## 振る舞い（behavior）

DSL:
```
behavior 製造完了を指示する = 製造中ロット AND 製造完了日 -> 製造完了ロット OR 製造完了指示エラー
```

Python:
```python
# ManufacturingLot を受け取るので、型レベルで不正な呼び出しを防ぐ
def complete_manufacturing(
    lot: ManufacturingLot,
    completed_date: date,
) -> ManufacturedLot:
    return ManufacturedLot(
        type="manufactured",
        common=lot.common,
        manufacturing_completed_date=completed_date,
    )
```

TypeScript:
```typescript
function completeManufacturing(
  lot: ManufacturingLot,
  completedDate: string,
): ManufacturedLot {
  return {
    type: 'manufactured',
    common: lot.common,
    manufacturingCompletedDate: completedDate,
  }
}
```

### 変換ルール

| DSL | Python | TypeScript |
|---|---|---|
| `入力1 AND 入力2 ->` | 関数の引数 | 関数の引数 |
| `-> 出力 OR エラー` | 例外（`raise LotError`）または `Result` 型 | 例外またはエラーオブジェクト返却 |
| `-> 出力`（エラーなし） | 戻り値そのまま | 戻り値そのまま |

## エラー型

DSLの `OR エラー名` から例外クラス群を生成する。

Python:
```python
class LotError(Exception):
    pass

class LotNotInManufacturing(LotError):
    pass

class InvalidDate(LotError):
    def __init__(self, reason: str) -> None:
        self.reason = reason
        super().__init__(reason)
```

TypeScript:
```typescript
type LotError =
  | { type: 'LotNotInManufacturing' }
  | { type: 'InvalidDate'; reason: string }
```

## 値型（スマートコンストラクタ）

Python では `@dataclass(frozen=True)` + `__post_init__` でバリデーションを実現する。

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Quantity:
    value: Decimal

    def __post_init__(self) -> None:
        if self.value < Decimal("0.001"):
            raise ValueError(f"Quantity must be >= 0.001, got {self.value}")

@dataclass(frozen=True)
class Count:
    value: int

    def __post_init__(self) -> None:
        if self.value < 1:
            raise ValueError(f"Count must be >= 1, got {self.value}")
```

TypeScript では Zod スキーマでランタイムバリデーションを行う:

```typescript
import { z } from 'zod'

const QuantitySchema = z.number().min(0.001)
const CountSchema = z.number().int().min(1)

type Quantity = z.infer<typeof QuantitySchema>
type Count = z.infer<typeof CountSchema>
```

## 命名変換

DSLは日本語、コードは英語。変換は以下の方針：

- 型名: DSLの日本語名を英訳（例: `在庫ロット` → `InventoryLot`）
- フィールド名: `snake_case`（Python）/ `camelCase`（TypeScript）
  - 例: `製造完了日` → `manufacturing_completed_date`（Python）/ `manufacturingCompletedDate`（TypeScript）
- 関数名: behaviorの動詞を英訳（例: `製造完了を指示する` → `complete_manufacturing`）
- エラー型: `〜エラー` → `〜Error`（例: `製造完了指示エラー` → `ManufacturingCompletionError`）

## Pydantic との使い分け（FastAPI）

ドメイン型は `@dataclass(frozen=True)` で定義し、API の入出力は Pydantic `BaseModel` で定義する。

```python
# ドメイン型（純粋・不変）
@dataclass(frozen=True)
class ManufacturingLot:
    type: Literal["manufacturing"]
    common: LotCommon

# APIレスポンス（Pydantic で JSON シリアライズ）
from pydantic import BaseModel

class LotResponse(BaseModel):
    lot_number: str
    status: str
    version: int
    manufacturing_completed_date: str | None = None
    shipping_deadline_date: str | None = None
    shipped_date: str | None = None
```

ドメイン型をAPIレスポンスに変換する関数を用意する:

```python
def to_lot_response(lot: InventoryLot, version: int) -> LotResponse:
    match lot:
        case ManufacturingLot(common=c):
            return LotResponse(
                lot_number=format_lot_number(c.lot_number),
                status="manufacturing",
                version=version,
            )
        case ManufacturedLot(common=c, manufacturing_completed_date=d):
            return LotResponse(
                lot_number=format_lot_number(c.lot_number),
                status="manufactured",
                version=version,
                manufacturing_completed_date=str(d),
            )
        case _:
            raise ValueError(f"Unexpected lot type: {lot}")
```

## pattern match（Python 3.10+）

Python の `match` 文は F# の判別共用体パターンマッチに近い書き方ができる:

```python
def describe_lot(lot: InventoryLot) -> str:
    match lot:
        case ManufacturingLot():
            return "製造中"
        case ManufacturedLot(manufacturing_completed_date=d):
            return f"製造完了（{d}）"
        case ShippingInstructedLot(shipping_deadline_date=d):
            return f"出荷指示済み（期限: {d}）"
        case ShippedLot(shipped_date=d):
            return f"出荷完了（{d}）"
```

TypeScript の discriminated union は `switch` で同様に扱う:

```typescript
function describeLot(lot: InventoryLot): string {
  switch (lot.type) {
    case 'manufacturing':
      return '製造中'
    case 'manufactured':
      return `製造完了（${lot.manufacturingCompletedDate}）`
    case 'shipping_instructed':
      return `出荷指示済み（期限: ${lot.shippingDeadlineDate}）`
    case 'shipped':
      return `出荷完了（${lot.shippedDate}）`
  }
}
```
